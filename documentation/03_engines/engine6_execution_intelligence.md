# Engine 6 — Execution Intelligence Engine

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D1, D2, D3, D5, D7, T3, T12  
**Canonical schemas:** `docs/canonical_schemas.md` §9  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

Routing and sequencing brain. Translates Engine 4's `StrategyPlan` into a fully resolved, dependency-aware `ExecutionIntent` with typed actions, chain-level sourcing, CCTP parameters, and authority checks. Does not build bytes, simulate, check policy, sign, or broadcast.

**Handoff:** `ExecutionIntent` goes to the **Execution Router**, which resolves venues, pools, splits, calldata, approvals, and CCTP parameters. The Planner builds from the Router's `ExecutionInstructions`. Engine 6 is not the last step before the Orchestrator.

---

## Trigger

Two paths:

1. **User-Approved:** Copilot presents the plan; the user approves; Engine 6 receives the `strategyPlanId`.
2. **Autopilot:** drift or risk qualifies for autopilot; Engine 6 receives the trigger automatically. Autopilot is **not** a bypass — Policy runs on every execution.

Path B (external wallets) is **manual**: Engine 6 still produces the intent, but the Plan/Policy/CCE path returns unsigned transactions for the user to sign. No delegation or session key exists (D1).

---

## Pipeline

```
[Step 1]   Authority Filter
[Step 1.5] Cross-Chain and Cross-Position Sourcing
[Step 2]   Intent Translation + Bridge Parameter Assignment
[Step 3]   Dependency Graph Construction
[Step 4]   Cross-Chain Cost-Benefit (consume Engine 3/4)
[Step 5]   Autopilot Qualification
[Output]   ExecutionIntent → Execution Router
```

---

## Step 1 — Authority Filter

Filter candidate source addresses by `AddressScope.executionMode`:

```typescript
const executable = portfolioState.addresses.filter(
  a => a.chain === targetChain &&
       (a.executionMode === 'AUTOPILOT' || a.executionMode === 'MANUAL')
)
```

- `AUTOPILOT` — Turnkey embedded wallet; CCE signs.
- `MANUAL` — external wallet; the user signs; CCE verifies and broadcasts.
- `READ_ONLY` — never a source of execution.

If the only holdings of a needed asset sit in read-only addresses, Engine 6 does **not** emit an action for them. It reports the shortfall:

```typescript
insufficientAuthorizedBalance: {
  asset: 'ETH',
  requiredUsd: Usd,
  authorizedUsd: Usd,
  suggestion: 'You have $1,300 of ETH in a read-only address. Add it as a MANUAL scope and sign the transaction yourself, or adjust the target.'
}
```

There is no "delegate it" path in MVP.

---

## Step 1.5 — Cross-Chain and Cross-Position Sourcing

Engine 4 produces aggregate asset-level deltas (e.g., "REDUCE ETH by 15%"). ETH exposure may exist as a wallet balance on Ethereum, an Aave deposit on Base, and a GMX GM position on Arbitrum. This step resolves which chain-level, position-level holdings to draw from.

**Liquidation waterfall (least disruptive first):**

```
Priority 1: Idle wallet balances (no unwind cost)
Priority 2: Single-sided lending deposits (clean withdraw, no slippage, no IL)
Priority 3: LP positions, ordered by unwind cost:
    a. constant_product pools (Uniswap V3, Raydium) — single-tx removal
    b. stable_swap / correlated pools (Aerodrome) — single-tx removal
```

**GMX is not in the waterfall (D2).** A GM position is never an automated funding source. If the only liquidity for a delta is a GM position, Engine 6 emits:

```typescript
manualReviewItems: [{
  id, kind: 'GMX_POSITION', chain: 'arbitrum', protocol: 'gmx',
  description: 'The plan needs liquidity currently held in a GMX GM pool.',
  suggestedAction: 'Reduce the GM position manually, then re-run the plan.'
}]
```

and does not generate a `BRIDGE`, `SWAP`, or `LP_REMOVE` that depends on it.

Within each tier, prefer the chain with the lowest current withdrawal gas cost (Protocol Layer estimate). Only reach into the next tier or chain if the current source cannot cover the full delta. Every candidate is filtered by Step 1 first.

The output is a set of resolved `(asset, chain, positionType, amount)` tuples.

---

## Step 2 — Intent Translation + Bridge Parameters

Each resolved tuple becomes a typed action node:

| Action Type | Description | Applies To |
|---|---|---|
| `SWAP` | Exchange asset A for asset B | All chains |
| `LEND_SUPPLY` | Supply asset to a lending protocol | All chains |
| `LEND_WITHDRAW` | Withdraw supply | All chains |
| `BORROW` | Borrow against collateral | All chains |
| `REPAY` | Repay debt | All chains |
| `LP_ADD` | Add liquidity | `constant_product`, `stable_swap`, `correlated` |
| `LP_REMOVE` | Remove liquidity | `constant_product`, `stable_swap`, `correlated` |
| `BRIDGE` | USDC CCTP V2 transfer | Cross-chain |
| `APPROVE` | ERC-20 allowance | EVM |
| `ATA_CREATE` | Associated Token Account creation | Solana |
| `MANUAL_REVIEW` | Human decision required | GMX or unsupported route |

There is no `BRIDGE_AGGREGATOR` value and no `LEND_DEPOSIT` alias. The canonical enum is `LEND_SUPPLY` / `LEND_WITHDRAW`.

### Bridge assignment

```typescript
if (action.type === 'BRIDGE') {
  if (action.assetIn.symbol !== 'USDC') {
    // D3: non-USDC cross-chain is out of MVP
    return manualReview('UNSUPPORTED_BRIDGE')
  }

  action.bridge = {
    mechanism: 'CCTP_V2',
    sourceDomain: CCTP_DOMAIN_IDS[sourceChain],
    destinationDomain: CCTP_DOMAIN_IDS[destinationChain],
    destinationChain,
    mintRecipient: resolveRecipient(action),
    destinationCaller: '0x0',
    burnToken: USDC[sourceChain],
    amount: action.amountIn,
    maxFee: speed === 'FAST' ? quotedMaxFee : 0n,
    minFinalityThreshold: speed === 'FAST' ? 1000 : 2000,
    speed
  }
}
```

**Speed:** Standard is the default. Fast is eligible only when `plan.urgency === 'HIGH'`, `CCTP_FAST_ENABLED` is true, and the quoted fee is at or below **0.10% of the transfer or $5, whichever is lower**. If the quote exceeds the cap, fall back to Standard and record the reason. The Router performs the final quote; Engine 6 records the preference.

**Recipient:**

- Path A: the embedded Turnkey wallet.
- Path B: the user's external wallet, because the user signs the mint.

`destinationCaller` is `0x0` in MVP. Domain IDs: Ethereum `0`, Arbitrum `3`, Solana `5`, Base `6`.

### LP fulfillment

```typescript
if (['constant_product', 'stable_swap', 'correlated'].includes(action.poolType)) {
  action.fulfillment = 'atomic'
}
```

GMX LP actions never reach here; they are `MANUAL_REVIEW`.

### Approvals

Every action that needs an allowance carries an explicit `ApprovalIntent`:

| Flow | Mechanism |
|---|---|
| Standard ERC-20 | `ERC20_APPROVE` |
| Aave V3 supply | `EIP2612_PERMIT` when supported; otherwise `ERC20_APPROVE` |
| Permit2 | Phase 2; not used in MVP |
| Solana ATA | `ATA_CREATE` action, not an approval |

Approval amount is the exact `amountIn`, not unlimited, unless the user explicitly opts into unlimited. USDT-like tokens use `approve(0)` then `approve(amount)`.

---

## Step 3 — Dependency Graph Construction

DAG where nodes are actions and edges are "must complete before":

**EVM:**

- `LEND_SUPPLY` or `LP_ADD` against an ERC-20 → inject `APPROVE(token → protocol)` if allowance is insufficient.
- `BRIDGE(USDC)` → inject `APPROVE(USDC → TokenMessengerV2)` if needed.

**Solana:**

- `LEND_SUPPLY` or `LP_ADD` for an SPL token with no ATA → inject `ATA_CREATE(token)`.
- Example: `ATA_CREATE(USDC SPL)` → `LEND_SUPPLY(USDC → Kamino)`.

**Ordering:**

1. Prerequisites (`APPROVE`, `ATA_CREATE`)
2. Sell / withdraw
3. Bridge
4. Buy / deposit on the target chain

**Partial plans:** the Copilot passes the requested subset; Engine 6 re-runs Step 3 on that subset. If it is dependency-complete, it emits a scoped intent. If not, it rejects the subset with a plain-language reason. The Copilot never mutates DAG edges.

---

## Step 4 — Cross-Chain Cost-Benefit (consume Engine 3/4)

Engine 6 does not recompute the break-even formula. It reads the pre-computed assessment from `YieldOpportunitySnapshot` / `StrategyPlan`:

```typescript
const crossChain = {
  qualifies: boolean,
  netSwitchingBenefitUsd: Usd,
  paybackDays: number,
  sourceChain, targetChain
}

if (crossChain.qualifies) {
  // include BRIDGE
} else if (plan.urgency === 'HIGH' && plan.isRiskReducing) {
  action.overrideReason = 'HIGH_URGENCY_RISK_GATE'
  // include BRIDGE despite a negative economic gate
} else {
  // remove cross-chain actions; keep within the source chain
}
```

The urgency override applies only to risk-reducing plans. It never bypasses Policy. Non-USDC cross-chain is always `UNSUPPORTED_BRIDGE`, even at HIGH urgency.

---

## Step 5 — Autopilot Qualification

```typescript
AutopilotConfig {
  enabled:            boolean,
  maxUsdPerExecution: Usd,
  allowedActionTypes: ActionType[],
  allowedChains:      Chain[],
  driftThreshold:     number,
  bridgeAllowed:      boolean
}
```

| Execution mode | `ExecutionIntent.source` | Who signs | Policy Check 3 |
|---|---|---|---|
| `AUTOPILOT` | `AUTOPILOT` | Turnkey TEE | Delegation active |
| `MANUAL` | `USER_APPROVED` | User's wallet | Ownership verified; signature checked at broadcast |

`AutopilotConfig` may further restrict `GoalProfile`, never expand it. Policy is never skipped for autopilot.

**Emergency pause:** if a CRITICAL risk alert fires mid-plan, the Orchestrator halts not-yet-started actions, lets in-flight actions resolve, disables autopilot for the user, and notifies them with what completed, what was skipped, and why. The `AWAITING_FULFILLMENT` saga state is retained for future GMX automation but is unreachable in MVP.

---

## `ExecutionIntent` Output Schema

Canonical type: `docs/canonical_schemas.md` §9.

```typescript
ExecutionIntent {
  id: UUID,
  userId: UUID,
  strategyPlanId: UUID,
  createdAt: IsoDateTime,
  source: 'USER_APPROVED' | 'AUTOPILOT',
  executionMode: 'AUTOPILOT' | 'MANUAL',
  urgency: 'NONE' | 'LOW' | 'MEDIUM' | 'HIGH' | 'EMERGENCY',
  actions: ExecutionAction[],
  manualReviewItems: ManualReviewItem[],
  estimatedTotalCostUsd: Usd,
  estimatedGasCostUsd: Usd
}

ExecutionAction {
  id: string,
  type: ActionType,
  chain: Chain,
  protocol: ProtocolId,
  poolId?: string,
  poolAddress?: string,
  tokenId?: string,
  assetIn: AssetRef,
  assetOut?: AssetRef,
  amountIn: BaseUnits,              // integer base units, never a decimal string
  amountUsd: Usd,
  sourceAddress: string,
  recipientAddress: string,
  poolType: PoolType,
  slippageCapPercent: Percent,      // min(goalProfile.maxSlippagePercent, poolType cap)
  deadline: IsoDateTime,
  approval?: ApprovalIntent,
  bridge?: CctpBridgeIntent,        // BRIDGE actions only; CCTP_V2 only
  dependsOn: string[],
  estimatedCostUsd: Usd,
  overrideReason?: string
}
```

**Invariants:**

- Every executable action has `protocol`, `sourceAddress`, `recipientAddress`, `poolType`, `slippageCapPercent`, and `deadline`.
- `amountIn` is base units; no `amount` decimal string exists.
- `bridge` is present only for `BRIDGE` and only `CCTP_V2`.
- `approval` is present when an allowance is required.
- `READ_ONLY` addresses never appear as `sourceAddress`.
- `manualReviewItems` covers GMX, unsupported bridges, unsupported venues, and complex actions.

The Router consumes this object and produces `ExecutionInstructions`; the Planner then builds and simulates. Engine 6 does not assign nonces, gas prices, calldata, or signatures.

---

## Resolved Items

1. **Bridge aggregator** — out of MVP (D3). Non-USDC fails closed with `UNSUPPORTED_BRIDGE`; no LI.FI/Socket branch.
2. **Mid-DAG authority revocation** — Orchestrator/CCE check authority at signing and broadcast; Engine 6 only pre-filters at planning time. Path B has no delegated key to revoke.
3. **Partial-plan contract** — same `ExecutionIntent` shape with a subset of actions and dependency validation; no separate type.
4. **Shared `poolType` library** — one `@crestflow/pool-type` package (T11).
5. **GMX** — removed from sourcing waterfall, automated LP actions, and Policy branches; manual review only.
6. **`ExecutionIntent` gaps** — closed by the canonical schema: base units, source/recipient, pool type, slippage cap, deadline, approval, bridge, dependencies.
