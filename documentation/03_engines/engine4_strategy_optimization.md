# Engine 4 — Strategy & Optimization Engine

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` T1, T5, D2, D3, D5  
**Canonical schemas:** `docs/canonical_schemas.md` §8  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

**Depends on:** Engine 1 (`PortfolioSnapshot`), Engine 2 (`RiskSnapshot`), Engine 3 (`YieldOpportunitySnapshot`), Engine 5 (`GoalProfile`).  
**Consumed by:** Engine 6.

---

## Purpose

Translates portfolio analysis, risk, and ranked opportunities into an actionable rebalancing plan. Runs quantitative portfolio optimization models scaled to available data history. Does not execute, simulate, route, or select pools.

Engine 4 owns the **economic gates**: drift tolerance, rebalancing cooldown, gas/bridge payback, and net switching benefit. These are not Policy Engine checks.

---

## Trigger

BullMQ, after **both** `RiskAnalysisCompleted` and `YieldOpportunitiesUpdated` for the same `PortfolioSnapshot`. Engine 4 needs the risk score for the defensive override and the yield snapshot for opportunity ranking.

---

## Defensive Override (runs first)

```typescript
if (riskSnapshot.riskScore > riskSnapshot.adjustedThreshold) {
  // Skip the optimizer entirely
  return defensiveShiftPlan({
    increaseStablecoinAllocationTo: computeStablecoinFloor(goalProfile),
    reduceVolatileAllocationTo:     computeVolatileCeiling(goalProfile),
    flagForManualReview:            highRiskPositions  // includes gmx_gm_pool positions
  })
}
```

**Input:** Engine 2's `riskScore` and `adjustedThreshold` — the baseline scaled by the persona multiplier. `compositeRiskScore` does not exist. There is no `maxRiskScore` field on `GoalProfile`; Engine 2 is the single source of truth for "is this portfolio too risky."

A defensive plan is risk-reducing and is marked `isRiskReducing = true` for Policy Check 4.

---

## Asset Identity Convention

Engine 4 optimizes over **aggregated asset-level exposure** — one node per economic asset (ETH, SOL, USDC) combined across chains. This matches Engine 1's top-level aggregation; per-chain detail is a placement constraint applied afterward.

ETH on Ethereum and ETH on Base has 1.0 return correlation; treating them as separate covariance nodes is incorrect. Chain allocation is a derived placement decision, not an optimizer output.

---

## Optimization Model Selection (by snapshot count)

| History Available | Model |
|---|---|
| < 14 snapshots | Equal Weight + Goal Tilt |
| 14–29 snapshots | Inverse Volatility |
| 30–89 snapshots | HRP + Mean-CVaR Ensemble |
| 90+ snapshots | Black-Litterman (Phase 2 stub) |

### Model 1 — Equal Weight + Goal Tilt

```typescript
baseWeight[i] = 1 / n

const bounds = {
  CAPITAL_PRESERVATION: { volatile: [0.0, 0.20], stable: [0.60, 1.0] },
  CONSERVATIVE:         { volatile: [0.0, 0.30], stable: [0.40, 0.80] },
  MODERATE:             { volatile: [0.0, 0.55], stable: [0.20, 0.60] },
  AGGRESSIVE:           { volatile: [0.0, 0.80], stable: [0.0, 0.40] },
  YIELD_SEEKER:         { volatile: [0.0, 0.60], stable: [0.10, 0.50] }
}
```

### Model 2 — Inverse Volatility

```typescript
sigma[i]  = historical_std(returns[i])  // annualised, last 14 days
invVol[i] = 1 / sigma[i]
weight[i] = invVol[i] / Σ(invVol)
```

Uses only the covariance diagonal; no off-diagonal estimation at this history depth.

### Model 3 — HRP + Mean-CVaR Ensemble (50/50)

**HRP:**

```typescript
d[i][j] = √(0.5 × (1 − ρ[i][j]))
// Ward linkage → quasi-diagonalisation → recursive bisection
// Weights by inverse cluster variance
```

**Mean-CVaR (Rockafellar & Uryasev 2000):**

```typescript
minimize: CVaR_95
subject to:
  Σweight[i] = 1
  weight[i] >= minPositionSize (e.g., 2%)
  goal-profile hard constraints
```

**Covariance:** Ledoit-Wolf shrinkage toward a scaled-identity target, applied to both HRP and Mean-CVaR.

**Ensemble:**

```typescript
finalWeight[i] = 0.50 × hrpWeight[i] + 0.50 × cvarWeight[i]
```

### Model 4 — Black-Litterman (Phase 2)

Placeholder at 90+ snapshots. Requires market equilibrium weights and analyst views; not in MVP.

---

## Constraint Application

```typescript
weight[i] = clamp(weight[i], minAllocation[i], maxAllocation[i])
weight[i] = weight[i] / Σ(weight)
if weight[i] < minPositionSize: weight[i] = 0; redistribute proportionally
```

**HHI hard cap (T1):**

- A new placement that would push asset, protocol, or chain HHI above **2,500** → reject.
- HHI 1,500–2,500 → warning, non-blocking.
- HHI above 5,000 → critical alert; no new placement on that axis.
- No forced unwind solely because HHI is high.

Same bands as Engine 1 and Engine 2. One concentration metric across the system.

---

## Drift and Tolerance

```
drift[i] = |targetWeight[i] − currentWeight[i]|
maxDrift = max(drift[i])
```

**Tolerance precedence (T5):**

1. If `maxDrift <= goalProfile.driftTolerancePercent`, no plan is generated. This check runs first.
2. If drift exceeds tolerance, the urgency tier applies:

| Urgency | Drift | Action |
|---|---|---|
| `HIGH` | `> 15%` | Plan allowed; cooldown bypassed; Fast CCTP eligible |
| `MEDIUM` | `7–15%` | Plan allowed if cooldown elapsed; Standard CCTP |
| `LOW` | `2–7%` | Plan allowed if cooldown elapsed; Standard CCTP |
| `NONE` | within tolerance | No plan |

**Tolerance bands:**

| Persona | Tolerance |
|---|---:|
| `CAPITAL_PRESERVATION` | ±3% |
| `CONSERVATIVE` | ±3% |
| `MODERATE` | ±5% |
| `AGGRESSIVE` | ±7% |
| `YIELD_SEEKER` | ±7% |

This removes the old 2–3% overlap: a 2.5% drift with a 3% tolerance is "no action," not "LOW urgency." `GoalProfile.driftTolerancePercent` is consumed, not duplicated.

---

## Action Generation

Engine 4 emits abstract delta instructions; Engine 6 converts them into canonical `ActionType` values (`SWAP`, `LEND_SUPPLY`, `LEND_WITHDRAW`, `LP_ADD`, `LP_REMOVE`, `BRIDGE`, `APPROVE`, `ATA_CREATE`, `BORROW`, `REPAY`, `MANUAL_REVIEW`).

### `poolType` handling

- `constant_product`, `stable_swap`, `correlated`: reserve-based valuation; StableSwap-aware IL for Aerodrome.
- `gmx_gm_pool`: GMX Reader valuation for display and risk only.

**GMX is manual-only (D2).** Engine 4 does not generate `LP_ADD` / `LP_REMOVE` for GM pools. If drift requires adjusting a GM position, it emits:

```typescript
{
  type: 'MANUAL_REVIEW',
  kind: 'GMX_POSITION',
  protocol: 'gmx',
  chain: 'arbitrum',
  currentWeight, targetWeight,
  description: string,
  suggestedAction: 'REDUCE' | 'INCREASE' | 'EXIT'
}
```

The user sees this through the Copilot and acts in their own wallet. Automated GMX management is Phase 2.

Aerodrome stable/correlated LP actions remain automated and use the StableSwap-aware impact formula for cost estimation.

---

## Cross-Chain Rebalancing

### Bridge cost model

```typescript
bridgeCost = cctpFeeUsd + destinationGasUsd

// Standard: cctpFeeUsd = 0; expected received = amount; ~15 min finality
// Fast:     cctpFeeUsd = quoted maxFee, capped at 0.10% or $5; ~seconds-to-minutes
```

`preBridgeSwapCost = swapSlippageUsd + swapGasUsd + swapFeeUsd`, tracked separately.

No slippage term in the bridge leg: CCTP is a 1:1 USDC burn-and-mint. Non-USDC cross-chain is `UNSUPPORTED_BRIDGE`; no bridge-aggregator path (D3).

### Transfer speed

- `urgency === HIGH` → Fast eligible, only if `CCTP_FAST_ENABLED` and the quoted fee is within the cap. Otherwise Standard.
- `urgency` LOW/MEDIUM → Standard (zero protocol fee).

The Router selects the final speed and quotes the fee; Engine 4 records the preference in the plan.

### Economic gate

```typescript
rebalancingBenefit = (targetReturn − currentReturn) × portfolioValueUsd
netBenefit = rebalancingBenefit − totalBridgeLegCost

recommend = netBenefit > 0
            AND paybackDays < 90
            AND netBenefit >= $25/year
```

The $25/year floor and 90-day payback are filters, not TOPSIS criteria (T2). A HIGH-urgency risk-reducing plan may override the economic gate with `overrideReason = 'HIGH_URGENCY_RISK_GATE'`; it may not override Policy.

---

## Rebalancing Cooldown

```typescript
const cooldownDays = goalProfile.rebalancingFrequencyCap ?? 7
if (daysSinceLastExecution < cooldownDays && urgency !== 'HIGH') {
  // suppress new plan generation
  // log suppressed-drift event; Copilot can explain on request
  return null
}
```

- Cooldown is owned by Engine 4, not Policy.
- Default is 7 days; `GoalProfile.rebalancingFrequencyCap` may lengthen it.
- HIGH urgency bypasses the cooldown.
- An emergency risk override may also bypass it; the override is recorded on the plan.

---

## Action Ordering

1. Prerequisite actions (`APPROVE`, `ATA_CREATE`)
2. Sell / withdraw
3. Bridge
4. Buy / deposit on the target chain

---

## `StrategyPlan` Output

Canonical fields are in `docs/canonical_schemas.md` §8:

```typescript
StrategyPlan {
  id, userId, createdAt,
  goalProfileVersion,
  portfolioSnapshotId, riskSnapshotId, yieldSnapshotId,
  urgency: 'NONE' | 'LOW' | 'MEDIUM' | 'HIGH' | 'EMERGENCY',
  actions: PlanAction[],
  totalValueUsd, estimatedCostUsd,
  rationale,
  manualReviewItems,

  // Engine 4 extension
  modelUsed: 'EW_GOAL_TILT' | 'INV_VOL' | 'HRP_CVAR' | 'BLACK_LITTERMAN',
  defensiveOverrideApplied: boolean,
  isRiskReducing: boolean,
  targetAllocation: Record<string, number>,
  expectedHealthScoreAfter: number,
  expectedRiskScoreAfter: number,
  cctp: {
    transferSpeed: 'STANDARD' | 'FAST' | null,
    estimatedBridgeTimeMinutes: number | null
  }
}
```

Emits `StrategyPlanCreated` or `StrategyPlanRevised`.

---

## Tax-Aware Ordering Heuristic

Full jurisdiction-aware tax-lot rebalancing is Phase 2. For MVP: when new capital is available (on-ramp proceeds, harvested yield, swap proceeds), direct it to underweight positions before selling overweight appreciated positions. This reduces unnecessary taxable events without lot-level tracking beyond Engine 1's WAC/PnL.

---

## Resolved Items

1. **HHI-to-per-chain cap** — the constraint is applied directly to HHI (reject new placements above 2,500), so no separate `maxChainConcentration` derivation is needed.
2. **Cooldown default** — 7 days, configurable via `GoalProfile.rebalancingFrequencyCap`; HIGH urgency bypasses.
3. **Drift/tolerance overlap** — tolerance gate runs first; urgency tiers apply only above tolerance.
4. **Shared `poolType` library** — one `@crestflow/pool-type` package (T11).
5. **Yield dependency** — Engine 4 consumes `YieldOpportunitySnapshot`; `RiskAnalysisCompleted` alone is not sufficient.
6. **GMX** — manual-only; no automated LP actions, no bridge-aggregator fallback.
7. **Tax-lot rebalancing** — Phase 2.
