# Engine 1 — Portfolio Intelligence Engine

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` T1, T6, T11, D2  
**Canonical schemas:** `docs/canonical_schemas.md` §5  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

Pure computation engine. Takes raw canonical data from the Financial Knowledge Layer and produces financial intelligence — the single source of truth every other engine reads from. No adapter calls, no external APIs.

---

## Processing Pipeline

```
PortfolioScanTriggered { userId, addresses, chains, trigger }
        ↓
[Step 1]  Fetch raw data from FKL (canonical types)
        ↓
[Step 2]  LP Token Decomposition
        ↓
[Step 3]  Asset Classification
        ↓
[Step 4]  True Exposure Computation
        ↓
[Step 5]  Allocation Analysis
        ↓
[Step 6]  Concentration Measurement (HHI)
        ↓
[Step 7]  Cost Basis + PnL
        ↓
[Step 8]  Performance Returns
        ↓
[Step 9]  Health Score
        ↓
[Step 10] Write PortfolioSnapshot (INSERT-only) + Emit
```

---

## Step 1 — Fetch from FKL

```typescript
Promise.allSettled([
  fkl.getHoldings(addresses),
  fkl.getProtocolPositions(addresses),
  fkl.getTransactionHistory(addresses),
  fkl.getPrices(allDetectedTokens)
])
```

`Promise.allSettled` — one adapter failing never crashes the scan. Canonical `DataQuality` is `COMPLETE`, `PARTIAL`, or `STALE`.

The snapshot carries one top-level `dataQuality` plus a component extension that records per-chain detail:

```typescript
dataQuality: 'PARTIAL',
chainQuality: {
  ethereum: 'COMPLETE',
  base:     'COMPLETE',
  arbitrum: 'COMPLETE',
  solana:   'STALE'
}
```

**Partial chain failure:** when a chain's scan fails, the last known data for that chain is blended in as `STALE` rather than omitted. Stale chain data is excluded from freshness-sensitive calculations (e.g., return computations for that window) but contributes to total value and allocation so the portfolio view stays coherent. If any included position is `STALE`, the snapshot is `PARTIAL` when fresher data exists for other positions, and `STALE` when no included position has fresh data. Execution pricing never uses this snapshot; the FKL re-quotes at build time (T6).

---

## Step 2 — LP Token Decomposition

### Reserve-based decomposition (`constant_product`, `stable_swap`, `correlated`)

For any reserve-based AMM (Uniswap V3, Raydium, Aerodrome):

```
userShare    = userLpTokens / totalLpTokenSupply
asset1Amount = poolReserve1 × userShare
asset2Amount = poolReserve2 × userShare
```

This reads the proportional claim on current reserves — a directly queryable fact independent of the pricing curve. It applies to `constant_product`, `stable_swap`, and `correlated` pools.

**Impermanent loss for `constant_product`:**

```
k  = currentPriceRatio / entryPriceRatio
IL = 2√k / (1+k) − 1
ilUsd = (asset1ValueAtEntry + asset2ValueAtEntry) × IL
```

**Impermanent loss for `stable_swap` / `correlated`:** the constant-product formula overstates IL for pegged pairs because StableSwap curves are much flatter near peg. A StableSwap-aware curve is used: near-zero IL while the pair trades near peg, falling back to the standard formula if a significant depeg occurs.

**Uniswap V3 tick ranges:** `UniswapV3LPAdapter` pre-computes effective reserves within the active tick range before passing canonical `Position` data to Engine 1. Engine 1 receives already-normalised data.

### GMX GM positions (`gmx_gm_pool`) — display only

GMX GM pools are not reserve-based. LPs are the counterparty to leveraged traders; pool value is collateral plus or minus trader PnL, priced from an oracle. The `userShare × poolReserve` formula is wrong for GM positions.

GMX is **read-only in MVP** (D2). Engine 1 values GM positions for display and risk context, but never produces an automated action from them.

```typescript
// poolType === 'gmx_gm_pool':
const { gmTokenPrice, longTokenPercent, shortTokenPercent } = await gmxReader.getGmPoolData(poolAddress)
const positionValueUsd = userGmTokens * gmTokenPrice
const longExposureUsd  = positionValueUsd * longTokenPercent
const shortExposureUsd = positionValueUsd * shortTokenPercent
```

"Impermanent loss" is not meaningful for GM pools — their downside comes from counterparty PnL, not asset-ratio drift. IL is not computed for `gmx_gm_pool` positions. Their risk contribution flows through Engine 2's market and liquidity analysers, not the liquidation analyser.

### Decomposition dispatch

```typescript
switch (position.poolType) {
  case 'constant_product': return reserveBasedDecomposition(position)
  case 'stable_swap':      return reserveBasedDecomposition(position, { ilModel: 'stableSwap' })
  case 'correlated':       return reserveBasedDecomposition(position, { ilModel: 'stableSwap' })
  case 'gmx_gm_pool':      return gmxReaderDecomposition(position) // display only
  case 'lending_market':   return position.supplied // supplied/borrowed breakdown
  case 'none':             return directHolding(position)
}
```

The `PoolType` enum and classification live in the shared `@crestflow/pool-type` package (T11). Engine 1 does not re-classify pools.

---

## Step 3 — Asset Classification

| Category | What Qualifies | Risk Weight |
|---|---|---|
| `VOLATILE` | BTC, ETH, SOL, governance tokens | High |
| `STABLECOIN` | USDC, USDT, DAI, PYUSD | Low |
| `LENDING` | Yield-bearing deposit tokens (aUSDC, mUSDC) | Low |
| `RWA` | Tokenised gold, bonds, real estate (Phase 2) | Medium |

Classification uses the canonical symbol from the Asset Registry, not the chain-specific contract address. USDC on Ethereum, Base, and Arbitrum classifies identically.

---

## Step 4 — True Exposure Computation

```
trueExposure[symbol] = directHoldings[symbol] + Σ(lpDecomposedAmounts[symbol])
```

The embedded wallet reuses the same EVM address across Ethereum, Base, and Arbitrum. Holdings on each chain are separate values, fetched as distinct `(address, chain)` pairs. Summing them is correct aggregation, not triple counting.

Per-chain breakdown is preserved beneath the aggregate:

```typescript
trueExposure['ETH'] = {
  total: 3.3,
  perChain: {
    ethereum: 2.5,
    base:     0.5,
    arbitrum: 0.3,
    solana:   0.0
  }
}
```

---

## Step 5 — Allocation Analysis

```typescript
assetAllocation: {
  'ETH':  { valueUsd: 7500, percent: 37.5 },
  'USDC': { valueUsd: 5000, percent: 25.0 },
  'BTC':  { valueUsd: 5000, percent: 25.0 },
  'SOL':  { valueUsd: 2500, percent: 12.5 }
}

chainAllocation: {
  ethereum: 50.0,
  base:     30.0,
  arbitrum: 15.0,
  solana:    5.0
}
```

Aggregate at the top level; per-chain breakdown is available as drill-down from `trueExposure[symbol].perChain`. Engine 2's chain-concentration work and the UI read the per-chain breakdown.

---

## Step 6 — Concentration Measurement (HHI)

```
HHI = Σ(shareᵢ²) × 10,000
Range: 0 (perfect diversification) → 10,000 (single position)
```

Computed across three axes: asset HHI, protocol HHI, chain HHI.

**Canonical bands (T1):**

| Band | Range | Action |
|---|---|---|
| Low | `< 1,500` | Diversified |
| Moderate | `1,500–2,500` | Monitor |
| High | `2,500–5,000` | HIGH alert; block new placements that would push the axis above 2,500 |
| Critical | `> 5,000` | CRITICAL alert |

Engine 1 reports the HHI set. Engine 2 raises alerts. Engine 4 blocks placements that would breach 2,500. No component forces an unwind solely because HHI is high; a forced reduction needs a separate risk trigger.

---

## Step 7 — Cost Basis + PnL

```
avgCostUsd        = totalCostUsd / totalQuantity
unrealizedPnlUsd  = (currentPriceUsd − avgCostUsd) × currentQuantity
```

`AssetCostBasis` is the one mutable record in the system.

**GMX positions:** PnL uses the Reader-sourced valuation from Step 2, not an IL formula. IL is not computed for `gmx_gm_pool`.

---

## Step 8 — Performance Returns

```
return7d  = (currentValue − value7dAgo)  / value7dAgo  × 100
return30d = (currentValue − value30dAgo) / value30dAgo × 100
return90d = (currentValue − value90dAgo) / value90dAgo × 100
```

Minimum two snapshots before any return is computable. Stale-window data is excluded from the numerator and denominator.

---

## Step 9 — Health Score (0–100)

```
healthScore = weighted_average(
  diversificationScore × 0.25,
  liquidityScore       × 0.20,
  yieldQualityScore    × 0.20,
  sustainabilityScore  × 0.20,
  protocolHealthScore  × 0.15
)
```

**Chain concentration penalty** (inside `diversificationScore`) uses the T1 bands:

- Chain HHI `< 1,500` → no penalty
- Chain HHI `1,500–2,500` → linear penalty
- Chain HHI `> 2,500` → full penalty weight

**Out-of-range and non-earning LP treatment** (inside `yieldQualityScore`): any LP not generating yield relative to its exposure — out-of-range Uniswap V3 ticks, or GMX GM positions with negligible/negative funding relative to directional exposure — reduces the Yield Quality component. Exposure is present; yield is not.

**Protocol Health component:** Engine 1 reads the FKL `protocolSafetyScore` directly from the 24-hour protocol-safety cache. It does **not** read Engine 2's output, avoiding a circular dependency (Engine 2 consumes Engine 1's snapshot). Engine 2 independently scores protocol risk from the same FKL source.

---

## Step 10 — Write PortfolioSnapshot + Emit

The canonical type is `docs/canonical_schemas.md` §5. Engine 1 writes that shape and adds component-local fields:

```typescript
PortfolioSnapshot {
  // canonical
  id, userId, createdAt,
  dataQuality: 'COMPLETE' | 'PARTIAL' | 'STALE',
  totalValueUsd,
  positions: Position[],
  chainExposureUsd,
  assetExposureUsd,
  protocolExposureUsd,
  hhi: { asset, protocol, chain },

  // Engine 1 extension
  trigger: 'ONBOARDING' | 'MANUAL' | 'POST_EXECUTION' | 'SCHEDULED' | 'EMERGENCY',
  chainQuality: Record<Chain, DataQuality>,
  assetAllocation, categoryAllocation, protocolAllocation, chainAllocation,
  directExposure, indirectExposure, trueExposure,
  unrealizedPnlUsd, realizedPnlUsd, yieldEarnedUsd, impermanentLossUsd,
  return7d, return30d, return90d,
  healthScore, healthComponents
}
```

INSERT-only. Emits `PortfolioSnapshotCreated`, which wakes Engine 2 and Engine 3 **in parallel**. Engine 4 waits for both.

---

## Responsibility Boundary

| Not Engine 1's Job | Whose Job |
|---|---|
| Fetching blockchain data | Financial Knowledge Layer |
| Scoring risk | Engine 2 |
| Ranking yield opportunities | Engine 3 |
| Recommending allocation changes | Engine 4 |
| Profiling the user | Engine 5 |
| Executing transactions | Engine 6 → Router → Planner → Policy → CCE |
| Deciding when to scan | Portfolio State Manager |
| Classifying pool types | Shared `@crestflow/pool-type` package |

---

## Resolved Items

1. **GMX Reader/subgraph integration** — read-only position and valuation adapter; no execution adapter (D2).
2. **StableSwap IL curve parameters** — owned by the shared `@crestflow/pool-type` package and validated against Aerodrome data; Engines 1–3 import it.
3. **Chain concentration penalty curve** — T1 bands; the between-band interpolation is a shared function, not per-engine.
4. **`dataQuality` shape** — canonical single enum plus per-chain `chainQuality` extension; the old `ok|failed|stale` map is not the snapshot-level contract.
5. **Protocol health circularity** — Engine 1 reads FKL directly, not Engine 2.
