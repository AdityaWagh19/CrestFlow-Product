# Policy Engine

**Status:** Authoritative  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D6  
**Canonical schemas:** `docs/canonical_schemas.md` §12  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Core Design Principle

**Fail-closed, default BLOCK.** No partial approval. A plan passes only if all nine checks pass. A plan that cannot be fully evaluated is `BLOCKED`, never treated as passing.

The Policy Engine evaluates the `PolicyInput` containing the resolved `SimulationPassedPackage`. It does not construct transactions, re-simulate, route, or optimize. It consumes the router's resolved instructions and the planner's simulation results.

**No short-circuit.** Every check runs for every plan. All failures are collected and returned as a complete set. Checks are ordered cheap → expensive for latency, but ordering never skips a check.

---

## The 9 Checks

### Check 1 — KYC Gate

```typescript
if (user.kycStatus !== 'APPROVED') → BLOCK (KYC_NOT_APPROVED)
```

Binary, chain-independent. `PENDING`, `REJECTED`, `RESUBMISSION_REQUESTED`, `SUSPENDED`, and `EXPIRED` all block. No partial KYC state allows partial execution.

### Check 2 — KYC Tier Daily Limit

```typescript
const spent24h = sum(
  auditEntries({
    userId,
    category: 'EXECUTION',
    last24h: true,
    chains: ['ethereum', 'base', 'arbitrum', 'solana'] // cross-chain sum
  })
)

if (spent24h + plan.totalValueUsd > tierLimit) → BLOCK (TIER_LIMIT_EXCEEDED)
```

| Tier | Daily execution limit |
|---|---:|
| `TIER_1` | $1,000 |
| `TIER_2` | $10,000 |
| `TIER_3` | Unlimited |

Summed across all four chains in a rolling 24-hour window. The audit ledger is the source of truth, not a mutable counter. `category='EXECUTION'` is defined in `CANONICAL_DECISIONS.md` T10; the canonical audit taxonomy includes it.

`plan.totalValueUsd` is the **gross notional** of new actions, excluding offsetting withdrawals that fund the same plan. The Planner computes it once and the Policy Engine does not re-derive it.

### Check 3 — Execution Authority Gate

```typescript
for (const action of plan.actions) {
  const scope = addressScopes.get(action.sourceAddress, action.chain)

  if (!scope || scope.executionMode === 'READ_ONLY') {
    → BLOCK (AUTHORITY_MISSING)
  }

  if (plan.source === 'AUTOPILOT') {
    if (scope.executionMode !== 'AUTOPILOT') → BLOCK (AUTHORITY_MISSING)
    if (!scope.turnkeySubOrgId) → BLOCK (AUTHORITY_MISSING)
    const status = checkTurnkeyDelegation(scope, freshness: '4h')
    if (status !== 'ACTIVE') → BLOCK (AUTHORITY_REVOKED | AUTHORITY_STALE)
  }

  if (plan.source === 'USER_APPROVED') {
    if (scope.executionMode !== 'MANUAL' && scope.executionMode !== 'AUTOPILOT') {
      → BLOCK (AUTHORITY_MISSING)
    }
    if (!scope.ownershipVerifiedAt) → BLOCK (AUTHORITY_MISSING)
    // The user's signature is verified by CCE at broadcast, not here.
  }
}
```

**Path B (external wallets) is manual.** There is no ERC-4337 session key and no SPL `delegate`. The user signs each transaction; the API verifies the signature at broadcast. `READ_ONLY` scopes can never execute.

**Path A (embedded Turnkey wallet) is autopilot.** The Turnkey sub-organisation is the only signer. Its delegation/policy must be active and no older than 4 hours.

### Check 4 — Risk Gate

```typescript
const riskSnapshot = getLatestRiskSnapshot(userId)
if (riskSnapshot.freshnessSeconds > 14_400) → BLOCK (RISK_SNAPSHOT_STALE)

if (riskSnapshot.riskScore > riskSnapshot.adjustedThreshold) {
  if (!plan.isRiskReducing) → BLOCK (RISK_TOO_HIGH)
}
```

`riskSnapshot.riskScore` is the only composite risk field. `compositeRiskScore` does not exist. `adjustedThreshold` is Engine 2's persona-multiplier-scaled baseline. There is no separate risk-tolerance field on `GoalProfile` or the Policy Engine.

`isRiskReducing` is true when the plan is designed to lower an already-elevated risk score (reduce a concentrated position, close a near-liquidation borrow). Blocking a risk-reducing plan because risk is high would be counterproductive.

### Check 5 — Chain Allowlist

```typescript
const CHAIN_ALLOWLIST = ['ethereum', 'base', 'arbitrum', 'solana']

for (const action of plan.actions) {
  if (!CHAIN_ALLOWLIST.includes(action.chain)) → BLOCK (CHAIN_NOT_ALLOWED)
}
```

### Check 6 — Protocol Allowlist

The allowlist must match `CANONICAL_DECISIONS.md` T12 and the Protocol Layer adapter roster exactly.

```typescript
const PROTOCOL_ALLOWLIST = {
  ethereum: ['aave-v3', 'uniswap-v3'],
  base:     ['aave-v3', 'uniswap-v3', 'aerodrome', 'moonwell'],
  arbitrum: ['aave-v3', 'uniswap-v3'],
  solana:   ['jupiter', 'raydium', 'kamino', 'marginfi']
}

const CROSS_CHAIN_PROTOCOL_ALLOWLIST = ['circle-cctp'] // CCTP V2 only

for (const action of plan.actions) {
  if (action.type === 'BRIDGE') {
    if (!CROSS_CHAIN_PROTOCOL_ALLOWLIST.includes(action.protocol)) {
      → BLOCK (PROTOCOL_NOT_ALLOWED)
    }
  } else {
    const allowedForChain = PROTOCOL_ALLOWLIST[action.chain] ?? []
    if (!allowedForChain.includes(action.protocol)) {
      → BLOCK (PROTOCOL_NOT_ALLOWED)
    }
  }
}
```

GMX is not in the automated allowlist; it is read-only/manual. 1inch, UniswapX, Curve, and Camelot are Phase 2 and not allowed in MVP.

**Allowlist curation includes sanctions screening.** Every protocol is screened against OFAC/Chainalysis/TRM before being added. Since only allowlisted contracts execute, this is the MVP's de facto per-transaction sanctions coverage. Live counterparty screening is Phase 2; Travel Rule is Phase 3.

### Check 7 — Per-Transaction Value Limit

```typescript
for (const action of plan.actions) {
  if (action.amountUsd > goalProfile.maxUsdPerTransaction) {
    → BLOCK (TX_VALUE_EXCEEDED)
  }
}
```

Per-action, not per-plan. Prevents limit-splitting across multiple smaller actions in the same plan.

**Tier 3 / institutional override:** an admin-set `PolicyOverride` can raise `maxUsdPerTransaction` above the persona ceiling. The override applies only to business-tunable limits. Checks 1, 3, 5, and 6 are never overridable by any tier.

### Check 8 — Slippage Tolerance (`poolType`-aware)

```typescript
const POOL_TYPE_CAP = {
  stable_swap: 0.25,
  correlated: 0.50,
  constant_product: 1.00
}

for (const action of plan.actions) {
  if (['SWAP', 'LP_ADD', 'LP_REMOVE'].includes(action.type)) {
    const cap = Math.min(goalProfile.maxSlippagePercent, POOL_TYPE_CAP[action.poolType])
    const actual = action.simulatedSlippagePercent ?? action.simulatedPriceImpactPercent
    if (actual > cap) → BLOCK (SLIPPAGE_EXCEEDED)
  }
}
```

The persona value may lower the cap but never raise it. The hard maximum is 1.00%. GMX is manual-only and is not evaluated here. The shared `@crestflow/pool-type` package owns the classification and the caps (T3, T11).

### Check 9 — Action Type Restrictions

```typescript
if (!goalProfile.allowedActionTypes.includes(action.type)) {
  → BLOCK (ACTION_NOT_ALLOWED)
}

if (plan.source === 'AUTOPILOT' && action.type === 'BRIDGE' && !autopilotConfig.bridgeAllowed) {
  → BLOCK (ACTION_NOT_ALLOWED)
}

if (action.protocol === 'gmx') {
  → BLOCK (UNSUPPORTED_VENUE) // GMX is manual-only in MVP
}
```

`allowedActionTypes` is a required field on `GoalProfile` (`docs/canonical_schemas.md` §4). `AutopilotConfig` may further restrict it, never expand it. There is no GMX persona branch because GMX actions are never generated.

---

## Check Numbering Reference

| # | Check | Type | Failure code |
|---:|---|---|---|
| 1 | KYC Gate | Binary compliance | `KYC_NOT_APPROVED` |
| 2 | KYC Tier Daily Limit | Rate limit | `TIER_LIMIT_EXCEEDED` |
| 3 | Execution Authority Gate | Binary eligibility | `AUTHORITY_MISSING` / `AUTHORITY_REVOKED` / `AUTHORITY_STALE` |
| 4 | Risk Gate | Risk score threshold | `RISK_TOO_HIGH` / `RISK_SNAPSHOT_STALE` |
| 5 | Chain Allowlist | Hard list | `CHAIN_NOT_ALLOWED` |
| 6 | Protocol Allowlist | Hard list | `PROTOCOL_NOT_ALLOWED` |
| 7 | Per-Transaction Value Limit | Dollar cap | `TX_VALUE_EXCEEDED` |
| 8 | Slippage Tolerance | Quality gate | `SLIPPAGE_EXCEEDED` |
| 9 | Action Type Restrictions | Policy gate | `ACTION_NOT_ALLOWED` / `UNSUPPORTED_VENUE` |

---

## Input and Output

`PolicyInput` and `PolicyDecision` are defined in `docs/canonical_schemas.md` §12. The decision always contains exactly nine `PolicyCheckResult` entries, even when blocked.

```typescript
PolicyDecision {
  decision: 'APPROVED' | 'BLOCKED',
  checksRun: PolicyCheckResult[],  // 9 entries
  failures: PolicyFailure[],
  policyVersion: string,
  simulationPackageId: UUID,
  decidedAt: IsoDateTime
}
```

Every decision — approval and rejection — is audit-logged as `category='EXECUTION'`, `action='POLICY_EVALUATION'`. The audit entry carries the KYC snapshot, `policyVersion`, `checksRun`, and the failure codes.

---

## What Policy Engine Does NOT Do

| Not Policy Engine's Job | Whose Job |
|---|---|
| Transaction construction | Planner / Protocol Layer |
| Simulation | Planner |
| Portfolio optimization | Engine 4 |
| Routing / venue selection | Execution Router |
| Cooldown enforcement | Engine 4 |
| Gas/bridge payback | Engine 4 economic gate |
| Risk score computation | Engine 2 |
| Live sanctions screening | Allowlist curation (MVP); Phase 2 counterparty screening |
| Travel Rule | Phase 3 |

---

## Resolved Items

1. **Curve's allowlist status** — Phase 2. Curve is not in the MVP venue roster (T12), so the question is deferred with the venue decision, not left open.
2. **Shared `poolType` library** — resolved by T11: one `@crestflow/pool-type` package, imported by Policy, Engines 1–4/6, Router, and Protocol Layer.
3. **Authority freshness mechanics** — resolved by T9/T11: Turnkey delegation is checked via the CCE adapter, cached for at most 4 hours, and re-checked on-chain before trusting a stale cache.
4. **RWA / leveraged protocols** — Phase 2. They will get allowlist rows when they graduate.

---

## Test Vectors

The pass/fail table, boundary values, and cross-check cases are in `docs/implementation_spec.md` §11. A policy change is not complete until the table is updated and all nine checks have at least one passing and one failing vector.
