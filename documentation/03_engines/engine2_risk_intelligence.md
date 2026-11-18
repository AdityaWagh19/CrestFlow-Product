# Engine 2 — Risk Intelligence Engine

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` T1, T4, T13  
**Canonical schemas:** `docs/canonical_schemas.md` §6  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

Pure computation engine. Zero external adapter calls. Reads Engine 1's `PortfolioSnapshot` and FKL-supplied protocol-safety data, then runs quantitative risk analysis. Risk is a computation problem, not a data problem.

**Canonical risk field:** `RiskSnapshot.riskScore`. The old `compositeRiskScore` does not exist.

---

## Trigger

```
Engine 1 emits → PortfolioSnapshotCreated
                        ↓
          Engine 2 picks up from BullMQ queue
                        ↓
          Loads full PortfolioSnapshot by snapshotId
                        ↓
          Loads last N historical snapshots for the user
                        ↓
          Loads protocol-safety snapshot from FKL (24h cache)
                        ↓
          Runs 5 analysers in parallel
```

Engine 2 and Engine 3 run concurrently after Engine 1. Engine 4 waits for both.

---

## Analyser 1 — Market Risk

```
returns[i] = (value[i] − value[i-1]) / value[i-1]

CVaR_95     = mean(sorted_returns[0..floor(0.05 × n)])

Sortino     = (annualisedReturn − riskFreeRate) / (std(downsideReturns) × √365)

MDD         = min((value[i] − peak[i]) / peak[i])

Calmar      = annualisedReturn / |MDD|

vol_7d      = std(last_7_daily_returns) × √365
vol_30d     = std(last_30_daily_returns) × √365
```

Minimum history:

- Volatility: 7 snapshots
- Sortino / CVaR: 30 snapshots
- MDD / Calmar: 14 snapshots

Below threshold → metric is `null`, `insufficientHistory: true`. The missing metric's weight is redistributed proportionally across the available metrics (see §Risk score).

**CVaR model:** historical-simulation CVaR (5th-percentile cutoff of daily returns) for MVP. Cornish-Fisher adjustment is Phase 2, gated behind materially higher minimum history.

---

## Analyser 2 — Liquidation Risk

```
HF              = Σ(collateralValueUsd × liquidationThreshold) / Σ(debtValueUsd)
distancePercent = (HF − 1.0) / HF × 100

HF > 2.0      → SAFE
HF 1.5–2.0    → WARNING
HF 1.1–1.5    → DANGER
HF < 1.1      → CRITICAL
```

**GMX GM positions are excluded from HF scoring.** A GM LP is not borrowing against collateral; there is no LP-side liquidation event in the HF sense. GM risk flows through Analyser 1 (value swings) and Analyser 5 (exit impact).

**Cross-chain collateral correlation:**

```typescript
crossChainCollateral = {
  totalCollateralUsd: Usd,
  totalDebtUsd: Usd,
  weightedHealthFactor: number | null,
  correlatedLiquidationRisk: boolean
}
```

Algorithm:

1. Collect every lending position with `debtUsd > 0`.
2. Group by canonical asset symbol (not contract address) across all chains and protocols.
3. If a symbol appears as collateral in two or more positions, set `correlatedLiquidationRisk = true` and aggregate collateral and debt.
4. Emit `CROSS_CHAIN_COLLATERAL_RISK` with the affected positions.

Because the embedded EVM address is identical across Ethereum, Base, and Arbitrum, posting the same asset as collateral on multiple Aave deployments is expected normal use, not an edge case.

---

## Analyser 3 — Concentration Risk

**HHI (T1):**

| Band | Range | Meaning |
|---|---|---|
| Low | `< 1,500` | Diversified |
| Moderate | `1,500–2,500` | Acceptable but monitored |
| High | `2,500–5,000` | Concentrated; HIGH alert |
| Critical | `> 5,000` | Dangerous; CRITICAL alert |

```
concentrationScore = max(assetHhiScore, protocolHhiScore, chainHhiScore)
```

Engine 2 raises alerts; it does not force an unwind. Engine 4 blocks new placements that would push any axis above 2,500.

---

## Analyser 4 — Protocol Risk

```
protocolRiskScore = Σ(protocolSafetyScore[p] × allocationWeight[p])
```

Protocol safety scores are computed by the FKL protocol-safety pipeline from DeFiLlama data (TVL, TVL trend, time-in-market, audit/exploit history) and cached 24 hours. Engine 2 consumes the cached canonical scores; it does not call DeFiLlama directly.

Per-chain scoring:

```typescript
protocolScores: {
  'aave:ethereum':   90,
  'aave:base':       87,
  'aave:arbitrum':   85,
  'moonwell:base':   80,
  'aerodrome:base':  78,
  'marginfi:solana': 80,
  'kamino:solana':   76,
  'gmx:arbitrum':    82   // display/risk only; never automated
}
```

Same protocol, different chain, different maturity/audit/TVL profile → scored independently.

TVL alert trigger: a >30% TVL drop in 7 days fires a protocol risk alert regardless of chain.

---

## Analyser 5 — Liquidity Risk

Liquidity risk is the slippage cost of exiting the full position at current market conditions.

**`poolType`-aware price impact:**

```typescript
// constant_product (Uniswap V3, Raydium):
priceImpact = positionSize / (2 × poolReserve) × 100

// stable_swap / correlated (Aerodrome):
priceImpact = stableSwapImpact(positionSize, poolDepthAtPeg)

// gmx_gm_pool (GMX, Arbitrum) — display only:
priceImpact = gmxReader.getExitImpact(positionSize, gmPool)
```

Applying the constant-product formula to stable pools overstates impact; applying it to GM pools is undefined. The shared `@crestflow/pool-type` package owns the branch selection and formulas.

**Liquidity tiers:**

```
priceImpact < 0.1%    → HIGHLY_LIQUID
priceImpact 0.1–0.5%  → ACCEPTABLE
priceImpact 0.5–2%    → ILLIQUID → flag
priceImpact > 2%      → SEVERELY_ILLIQUID → alert
```

Vesting/time-locked exit is Phase 2.

---

## Risk Score

```
riskScore = (
  marketRiskScore      × 0.30 +
  liquidationRiskScore × 0.25 +
  concentrationScore   × 0.20 +
  protocolRiskScore    × 0.15 +
  liquidityRiskScore   × 0.10
)
```

- Scale: `0–100`, higher = riskier.
- If a metric is unavailable, its weight is redistributed proportionally across the available metrics.
- `scoreComponents` records each component and its effective weight.

**Bands (existing Engine 2 bands, retained by T13):**

| Range | Level |
|---|---|
| `0–39` | LOW |
| `40–59` | MEDIUM |
| `60–79` | HIGH |
| `80–100` | CRITICAL |

**Persona-adjusted threshold:**

| Persona | Multiplier |
|---|---:|
| `CAPITAL_PRESERVATION` | 0.5× |
| `CONSERVATIVE` | 0.7× |
| `MODERATE` | 1.0× |
| `AGGRESSIVE` | 1.3× |
| `YIELD_SEEKER` | 1.15× |

```
adjustedThreshold = baselineThreshold × profileMultiplier
baselineThreshold = 60 (configuration; MODERATE default)
```

With the default baseline, thresholds are: Capital Preservation 30, Conservative 42, Moderate 60, Aggressive 78, Yield Seeker 69. The formula is canonical; the baseline is tunable configuration and is recorded in `RiskSnapshot`.

`adjustedThreshold` is what Policy Check 4 compares against `riskScore`. Engine 4 may also use it for defensive overrides. If Engine 5's persona is unavailable, default to `MODERATE` (1.0×) and record `appliedProfileMultiplier = 1.0`.

**Freshness:** `freshnessSeconds > 14,400` (4 hours) blocks signing. Policy Check 4 returns `RISK_SNAPSHOT_STALE`; it does not silently use an old snapshot.

---

## Alert System

| Alert Type | Trigger | Severity |
|---|---|---|
| `LIQUIDATION_IMMINENT` | HF < 1.1 (lending only; GMX excluded) | CRITICAL |
| `LIQUIDATION_WARNING` | HF 1.1–1.5 | WARNING |
| `HIGH_ASSET_CONCENTRATION` | Asset HHI 2,500–5,000 / >5,000 | WARNING / CRITICAL |
| `HIGH_PROTOCOL_CONCENTRATION` | Protocol HHI 2,500–5,000 / >5,000 | WARNING / CRITICAL |
| `HIGH_CHAIN_CONCENTRATION` | Chain HHI 2,500–5,000 / >5,000 | WARNING / CRITICAL |
| `HIGH_VOLATILITY` | 7D vol > persona-adjusted threshold | WARNING |
| `SIGNIFICANT_DRAWDOWN` | MDD crosses persona-adjusted threshold | WARNING / CRITICAL |
| `LOW_PROTOCOL_SCORE` | Weighted protocol score below threshold | WARNING |
| `ORACLE_DIVERGENCE` | Oracle cross-check above tolerance | CRITICAL |
| `DATA_QUALITY_DEGRADED` | Snapshot `dataQuality != COMPLETE` | WARNING |

**Alert lifecycle:**

```
New breach → create ACTIVE alert (if no ACTIVE alert of same type)
Breach clears → auto-resolve → RESOLVED
User dismisses → DISMISSED (survives re-runs; never re-triggers for the same event)
```

Alerts are stored in `risk_alerts`. They are not audit entries; the audit layer records compliance and execution events.

---

## Output Schema

The canonical type is `docs/canonical_schemas.md` §6. Required semantics:

```typescript
RiskSnapshot {
  id, userId, portfolioSnapshotId, analyzedAt,
  freshnessSeconds,                 // <= 14,400 for policy
  cvar95Percent, sortinoRatio, maxDrawdownPercent, calmarRatio,
  realizedVol7d, realizedVol30d, insufficientHistory,
  liquidationPositions, minHealthFactor, crossChainCollateral, liquidationRiskScore,
  assetHhi, protocolHhi, chainHhi, concentrationScore,
  protocolScores, weightedProtocolScore, protocolRiskScore,
  exitRiskPositions, maxExitImpactPercent, liquidityRiskScore,
  riskScore,                        // 0..100, higher = riskier
  riskLevel,                        // LOW | MEDIUM | HIGH | CRITICAL
  adjustedThreshold,                // baselineThreshold × persona multiplier
  appliedProfileMultiplier,
  scoreComponents
}
```

Emits `RiskAnalysisCompleted`, which is consumed by Engine 4 together with `YieldOpportunitiesUpdated`.

---

## Resolved Items

1. **`crossChainCollateral` algorithm** — specified above: group lending debt positions by canonical symbol; 2+ positions → correlated.
2. **Persona multipliers** — 0.5 / 0.7 / 1.0 / 1.3 / 1.15 are canonical (T13); tuning is post-launch.
3. **GMX leveraged-trader risk** — out of scope; GMX is read-only and LP-only in MVP.
4. **Risk field name** — `riskScore` only; `compositeRiskScore` removed.
5. **Threshold bands** — T13 retains 0–39 / 40–59 / 60–79 / 80–100.
6. **Protocol data path** — FKL supplies 24h-cached safety scores; Engine 2 makes no external calls.
