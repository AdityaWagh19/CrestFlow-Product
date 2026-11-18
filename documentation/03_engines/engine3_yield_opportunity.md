# Engine 3 — Yield & Opportunity Engine

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` T2, T3, T6, D2, D3, D5, T12  
**Canonical schemas:** `docs/canonical_schemas.md` §7  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

Discovers, evaluates, and ranks yield opportunities across every allowlisted protocol and chain, matched to the user's profile. Does not execute. Does not manage the portfolio. Engine 4 consumes the resulting `YieldOpportunitySnapshot`.

---

## Trigger

Runs in parallel with Engine 2, off `PortfolioSnapshotCreated`. Reads the user's `GoalProfile` from Engine 5 and the latest available `RiskSnapshot` if one exists, but never blocks on Engine 2. Protocol safety scores come from FKL's 24h cache, not from Engine 2's output.

---

## Pipeline

```
[Step 1]  Opportunity Discovery
[Step 2]  APY Normalisation (7-day TWAP)
[Step 3]  IL-Adjusted True Yield (LP positions)
[Step 4]  Sustainability Classification
[Step 5]  TOPSIS Multi-Criteria Ranking (5 criteria)
[Step 6]  Portfolio Fit Scoring
[Step 7]  Idle Capital and Switching Signals
[Step 8]  Write YieldOpportunitySnapshot + Emit
```

---

## Step 1 — Opportunity Discovery

```typescript
fkl.getLendingPools(chains)
fkl.getLpPools(chains)
fkl.getRwaPools(chains)   // Phase 2
```

Discovery table:

| Source | Chain | Type | Data |
|---|---|---|---|
| Aave V3 | Ethereum | Lending | Supply APY, borrow rate, LTV, TVL |
| Aave V3 / Moonwell | Base | Lending | Supply APY, borrow rate, LTV, TVL |
| Aave V3 | Arbitrum | Lending | Supply APY, borrow rate, LTV, TVL |
| MarginFi / Kamino | Solana | Lending | Supply APY, utilisation, TVL |
| Uniswap V3 | Ethereum | LP | Fee APR by tier, TVL, tick range |
| Aerodrome | Base | LP | Fee APR (stable/volatile), TVL, pool reserves |
| Uniswap V3 | Arbitrum | LP | Fee APR, TVL |
| Raydium | Solana | LP | Fee APR, TVL |

**GMX GM pools are excluded from TOPSIS in MVP** (D2, T2). They are read-only/manual; they have no meaningful IL model, and ranking them would imply an automated route that does not exist. Their yield may be displayed separately as reference data.

Scoped to assets the user already holds — no recommendations for assets they would need to acquire first.

---

## Step 2 — APY Normalisation

```
APY = (1 + APR/365)^365 − 1

TWAP_APY_7D = Σ(APY[day_i] × weight[day_i]) / Σ(weight[day_i])
              minimum 7 daily data points; equally weighted by default
```

- The canonical field is `twapApy7dPercent`. `twapApy30dPercent` is removed.
- `spotApyPercent = organicAPY + incentivisedAPY`
- `excessYield = netApy − riskFreeBaseline`

APY freshness: the FKL pool-data TTL is 300 seconds, so opportunities are at most 5 minutes stale in normal operation. `dataAsOf` is propagated to the output. If data is `STALE`, the opportunity is ranked but marked and cannot trigger execution without a re-quote.

If fewer than 7 daily points exist, the opportunity is excluded from TOPSIS rather than ranked on a shorter window. A new pool with no history is not comparable on consistency.

---

## Step 3 — IL-Adjusted True Yield

### `constant_product` (Uniswap V3, Raydium)

```
IL = 2√k / (1+k) − 1    where k = currentPriceRatio / entryPriceRatio
annualIL  = IL_per_period × (365 / holdingDays)
trueYield = tradingFeeAPY + rewardAPY − annualIL
```

### `stable_swap` / `correlated` (Aerodrome)

The constant-product formula overstates IL for pegged pairs. A StableSwap-aware model applies — near-zero IL while the pair trades near peg:

```
trueYield = tradingFeeAPY + rewardAPY − stableSwapAdjustedIL
```

### `gmx_gm_pool` — excluded

GM pools are not evaluated by TOPSIS in MVP. No synthetic IL score is assigned. If displayed, GM yield is reference-only and cannot become a plan action.

### `poolType` dispatch

```typescript
switch (pool.poolType) {
  case 'constant_product': return constantProductTrueYield(pool)
  case 'stable_swap':      return stableSwapTrueYield(pool)
  case 'correlated':       return stableSwapTrueYield(pool)
  case 'gmx_gm_pool':      return null            // excluded from ranking
  case 'lending_market':   return lendingTrueYield(pool)
}
```

The shared `@crestflow/pool-type` package owns classification, IL branching, and slippage caps (T3, T11). Engine 3 does not reimplement them.

---

## Step 4 — Sustainability Classification

```
organicRatio = organicAPY / totalAPY

organicRatio > 0.80    → ORGANIC
organicRatio 0.40–0.80 → MIXED
organicRatio < 0.40    → INCENTIVIZED

tvlChange7d > 0%        → GROWING
tvlChange7d −10% to 0%  → STABLE
tvlChange7d < −10%      → DECLINING
tvlChange7d < −30%      → CRITICAL
```

---

## Step 5 — TOPSIS Multi-Criteria Ranking

**Exactly five criteria** (T2):

| Criterion | Direction | Capital Preservation | Conservative | Moderate | Aggressive | Yield Seeker |
|---|---|---|---|---|---|---|
| Net APY | Maximise | 0.05 | 0.10 | 0.25 | 0.40 | 0.35 |
| Protocol Safety | Maximise | 0.50 | 0.40 | 0.25 | 0.15 | 0.20 |
| Yield Consistency (1/CV) | Maximise | 0.30 | 0.25 | 0.20 | 0.15 | 0.20 |
| Liquidity | Maximise | 0.10 | 0.15 | 0.15 | 0.15 | 0.15 |
| IL Risk | Minimise | 0.05 | 0.10 | 0.15 | 0.15 | 0.10 |

```
// Normalise
r[i][j] = x[i][j] / √(Σ x[k][j]²)

// Weight
v[i][j] = w[j] × r[i][j]

// Ideal and anti-ideal
A+[j] = max(v[i][j]) for 'Maximise', min(v[i][j]) for 'Minimise'
A−[j] = min(v[i][j]) for 'Maximise', max(v[i][j]) for 'Minimise'

// Distance
d+[i] = √(Σ (v[i][j] − A+[j])²)
d−[i] = √(Σ (v[i][j] − A−[j])²)

// Closeness coefficient (higher = better)
C[i] = d−[i] / (d+[i] + d−[i])
```

**No sixth criterion.** Gas, bridge cost, and payback are post-TOPSIS filters. Net switching benefit is handled by Engine 4's drift/cooldown and economic gate, not by TOPSIS.

**Missing values:** if a row lacks a criterion (e.g., no IL model), the row is excluded from ranking; the weight is not redistributed to fabricate a score. This is the GMX case.

**Post-TOPSIS gates (both must pass):**

1. `netSwitchingBenefit >= $25/year` (configurable floor).
2. Payback period `switchingCost / (annualBenefit / 365) < 90 days`.

**RWA yields:** separate ranking track with its own criteria (Net Yield, Issuer/Counterparty Risk, Redemption Liquidity, Regulatory Clarity). Phase 2.

---

## Step 6 — Portfolio Fit Scoring

```
finalScore = 0.70 × topsisScore + 0.30 × portfolioFitScore
```

Fit factors:

- concentration penalty (would this push asset, protocol, or chain HHI above 2,500?),
- diversification reward (cross-chain / cross-protocol spread),
- alignment with `GoalProfile.maxSingleChainPercent`,
- protocol HHI concern.

A candidate that would push any HHI axis above 2,500 is down-ranked and cannot be selected by Engine 4 as a new placement. It is not forced to zero; existing exposure can remain.

---

## Step 7 — Idle Capital and Switching Signals

```
IDLE            → earning 0%
UNDERPERFORMING → earning < 50% of best available
SUBOPTIMAL      → earning 50–80% of best available

opportunityCostUsd/year = idleCapitalUsd × bestAvailableAPY
```

Idle capital is detected **per chain**; the same EVM address on Ethereum, Base, and Arbitrum carries independent balances.

**Cross-chain signal:**

```
netSwitchingBenefit = (targetAPY − currentAPY) × capitalUsd × timeHorizon
                      − bridgeCostUsd − gasEstimateUsd
```

`bridgeCostUsd` uses CCTP V2:

- Standard: `0` protocol fee; expected received = amount.
- Fast: quoted `maxFee`, capped at 0.10% of the transfer or $5, whichever is lower; expected received = `amount − maxFee`.
- Non-USDC cross-chain: `UNSUPPORTED_BRIDGE`; no signal.

Engine 3 emits a **signal**, not a plan. Engine 4 decides whether drift, cooldown, and net benefit justify a `StrategyPlan`. This prevents Engine 3 from bypassing Engine 4's economic gate.

---

## Step 8 — Write + Emit

The canonical type is `docs/canonical_schemas.md` §7:

```typescript
YieldOpportunitySnapshot {
  id, createdAt, dataQuality,
  opportunities: YieldOpportunity[],
  topsisCriteria: TopsisCriterion[],       // exactly 5
  weightsByPersona: Record<Persona, TopsisWeights>
}

YieldOpportunity {
  id, chain, protocol, poolId, poolAddress, poolType,
  assetSymbols, tvlUsd,
  spotApyPercent,
  twapApy7dPercent,                        // canonical; 7 daily points minimum
  yieldConsistencyCv,
  ilRiskScore,                             // 0..100, higher = riskier; TOPSIS minimises
  liquidityScore,
  protocolSafetyScore,
  netApyPercent,
  fetchedAt, dataQuality
}
```

Component-local additions may include `organicApyPercent`, `incentivisedApyPercent`, `excessYieldPercent`, `sustainabilityTier`, `tvlTrend`, `topsisClosenessCoeff`, `portfolioFitScore`, `finalRank`, and `switchingBenefitUsd`. They do not rename canonical fields.

Emits `YieldOpportunitiesUpdated`, consumed by Engine 4 alongside `RiskAnalysisCompleted`.

---

## Responsibility Boundary

| Not Engine 3's Job | Whose Job |
|---|---|
| Fetching pool data | FKL |
| Computing portfolio risk | Engine 2 |
| Deciding final allocation | Engine 4 |
| Drift tolerance and cooldown | Engine 4 |
| Executing rebalances | Engine 6 → Router → Planner → Policy → CCE |
| Profiling the user | Engine 5 |
| Bridging non-USDC assets | Not in MVP; `UNSUPPORTED_BRIDGE` |

Leveraged/borrow-to-deploy yield strategies are out of default MVP scope — Phase 2 as an explicit opt-in strategy type with its own risk labeling, not a TOPSIS adjustment.

---

## Resolved Items

1. **APY field name** — `twapApy7dPercent`; 7-day window with at least 7 daily points.
2. **TOPSIS criteria** — five, exactly as tabled; no payback criterion.
3. **GMX** — excluded from ranking; no synthetic IL score; read-only display only.
4. **CCTP fee source** — Router/FKL quotes Standard and Fast fees at selection time; Standard is zero, Fast is capped.
5. **Shared `poolType` library** — one `@crestflow/pool-type` package (T11).
6. **RWA weights** — Phase 2.
7. **E3→E4 handoff** — `YieldOpportunitySnapshot` plus `YieldOpportunitiesUpdated`; Engine 4 consumes the snapshot, not Engine 3 internals.
