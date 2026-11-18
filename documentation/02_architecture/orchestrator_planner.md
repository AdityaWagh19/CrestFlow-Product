# Orchestrator / Planner

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D7, D5  
**Canonical schemas:** `docs/canonical_schemas.md` §9–§12  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

Two roles in one component:

- **Planner:** Build valid unsigned transactions from `ExecutionInstructions` (the Execution Router's output). Simulate every transaction against actual chain state. Fail closed on any simulation failure.
- **Orchestrator:** Own the workflow state machine for every execution plan. Coordinate multi-step, cross-chain sequences. Manage retries, compensation, and failure modes.

Core principle: **route first, build second, simulate third, policy fourth.** The Planner never builds from a bare `ExecutionIntent`; it builds from resolved instructions that contain the venue, pool, calldata, approvals, and CCTP parameters.

---

## Position in the pipeline

```
Engine 6 ExecutionIntent
  → Execution Router (ExecutionInstructions)
  → Planner (build + simulate)
  → Policy Engine (final gate)
  → CCE (sign, broadcast, confirm)
  → PSM (sync)
```

The Planner does not route, choose venues, or evaluate policy. The Router does not build bytes. The Policy Engine does not re-simulate. The CCE does not build or route.

---

## Planner — Build + Simulate

For each resolved action in `ExecutionInstructions`:

1. **Build** unsigned transaction bytes via the Protocol Layer adapter.
2. **Simulate** against real chain state:
   - EVM: `eth_call` with state overrides where needed; `eth_estimateGas` for gas.
   - Solana: `simulateTransaction` with `sigVerify: false`, `replaceRecentBlockhash: true`.
3. **Extract** simulated output amounts and balance deltas. Step N's simulated output feeds step N+1's input where the DAG has a dependency.
4. **Validate the envelope** separately: nonce (CCE), gas payment, chain ID, deadline, and signature requirements are not proven by `eth_call`. The CCE validates them before broadcast.

**Quote-staleness TTL:** if more than 30 seconds elapse between simulation passing and signing, re-simulate. Never sign on a stale result.

**Concurrency:** independent DAG branches simulate concurrently. Default cap: 8 concurrent simulations per plan; configurable by provider limits.

**Compensation simulation:** compensating transactions use the same fail-closed discipline. A compensation that fails to simulate is a critical, human-escalated alert — not retried automatically.

### State overrides

- EVM simulations may override balances/allowances to model approvals that will exist at execution time, but the override set must be recorded in `SimulationPassedPackage.stateOverridesUsed` and reverted for the real transaction.
- Solana simulations use the real account state; missing ATAs are injected as prerequisite actions, not overridden.

### Planner — Output

`SimulationPassedPackage` is defined in `docs/canonical_schemas.md` §11. It carries:

- the simulation ID and instruction ID;
- per-action success, gas used, gas cost, simulated slippage/price impact, minimum output, and balance deltas;
- total gas in USD;
- chains and block numbers;
- whether state overrides were used;
- warnings.

A package with any `success=false` action is a simulation failure; it is never passed to Policy as approved.

---

## Planner — Gas Accounting

| Chain | Method |
|---|---|
| Ethereum | EIP-1559: `maxFeePerGas`, `maxPriorityFeePerGas`, `eth_estimateGas` + 15% buffer |
| Base | EIP-1559 + `GasPriceOracle` at `0x420000000000000000000000000000000000000F`: `getL1Fee(bytes)` / `getL1FeeUpperBound(uint256)` + `getOperatorFee(gasUsed)` |
| Arbitrum | L2 `eth_estimateGas` + `NodeInterface` at `0xc8`: `gasEstimateL1Component(address to, bool contractCreation, bytes data)`; `ArbGasInfo` at `0x6c` for `getL1BaseFeeEstimate()` |
| Solana | 5,000 lamports/signature + `computeUnitLimit × computeUnitPriceMicroLamports`; simulation-derived CU limit + 10% buffer |

The Base formula is:

```
totalFee = gasUsed × (baseFee + priorityFee) + l1Fee + operatorFee
```

Do not hand-roll `l1BaseFee × size × scalar`. The detailed spec is in `docs/implementation_spec.md` §5.

---

## Planner — CCTP V2 Decomposition

A `BRIDGE` action decomposes into:

```
1. Burn (source chain)
   - Planner builds depositForBurn(
       amount,
       destinationDomain,
       mintRecipient,
       burnToken,
       destinationCaller,
       maxFee,
       minFinalityThreshold
     )
   - Simulated and policy-gated like any action.

2. Attestation (off-chain)
   - CCE polls Circle Iris with exponential backoff (2s → 30s).
   - Standard timeout: 60 minutes. Fast timeout: 5 minutes.
   - Standard finality is ~15–19 min on ETH/ARB/Base and ~17s on Solana.
   - Fast finality is ~8s on ARB/Base, ~20s on ETH, sub-second on Solana.

3. Mint (destination chain)
   - On attestation, CCE asks the Planner to build receiveMessage().
   - Planner simulates the mint, then CCE signs/broadcasts (Path A) or returns it to the user (Path B).
```

The burn and mint are separate transactions. The mint recipient is fixed at burn time. `destinationCaller` is `0x0` in MVP. Fast fees are route-specific and deducted at mint; the expected received amount is `amount - maxFee`.

Bridges cannot be cancelled once the burn is submitted. If the mint cannot complete, the Orchestrator enters `AWAITING_ATTESTATION` / manual recovery; it never claims a burn reversal.

Non-USDC bridging is out of MVP. The Planner emits `UNSUPPORTED_BRIDGE` and a manual-review item.

---

## Orchestrator — Workflow State Machine

```
RECEIVED
  → ROUTING            (Execution Router)
  → BUILDING           (Planner builds unsigned txs)
  → SIMULATING         (Planner simulates)
  → POLICY_CHECK       (Policy Engine, all 9 checks)
      → POLICY_REJECTED (terminal)
      → EXECUTING
  → SIGNING            (CCE for AUTOPILOT; user for MANUAL)
  → BROADCASTING
  → PENDING_CONFIRMATION
  → CONFIRMED
  → AWAITING_ATTESTATION  (CCTP only)
  → MINTING               (CCTP only)
  → CONFIRMING
  → COMPLETED
```

Terminal abort states:

- `SIMULATION_FAILED` — with the specific revert reason.
- `POLICY_REJECTED` — with the full set of failed checks.
- `UNSUPPORTED_VENUE` / `UNSUPPORTED_BRIDGE` — with a manual-review item.

Failure/compensation path:

```
STEP_FAILED
  → COMPENSATING (simulate + execute compensating actions for completed prior steps)
  → ROLLED_BACK | PARTIAL_ROLLBACK | MANUAL_RECOVERY
```

**`queued` state:** a transaction not yet broadcast because `maxFee > gasPriceCap` enters `queued`. It is distinct from `PENDING_CONFIRMATION` (broadcast, awaiting confirmation). The Orchestrator tracks both separately and notifies the user.

---

## Retry Logic

### Transient (retry)

| Failure | Chain | Fix |
|---|---|---|
| Nonce conflict | EVM | CCE nonce manager reallocates; rebuild, re-sign, rebroadcast |
| Gas underpriced | EVM | Bump gas ≥10%, same nonce, rebroadcast |
| Blockhash expired | Solana | CCE re-signs with a fresh blockhash |
| Priority fee too low | Solana | Increase compute-unit price, rebroadcast |
| Provider 5xx / timeout | All | Retry with exponential backoff; circuit breaker |

### Permanent (abort)

| Failure | Reason |
|---|---|
| Simulation reverted | State changed materially; re-plan, do not blindly retry |
| Policy rejected | Business-rule block; retrying will not change the outcome |
| Authority revoked | Turnkey delegation invalid; user must re-authorize |
| Missing prerequisite | `APPROVE` or `ATA_CREATE` not injected; rebuild the DAG |
| Unsupported venue/bridge | Not in MVP scope; manual review |

**Path A authority:** the Turnkey policy engine rejects an unauthorized call during signing. **Path B:** the user's wallet rejects the transaction; the API verifies the signature before broadcast. There is no SPL delegate or ERC-4337 session key to check.

---

## Saga Compensation

Per-action compensation is defined in `docs/implementation_spec.md` §7.2.

- Swaps: reverse swap within the original slippage cap.
- Lending: withdraw/re-supply, only if no dependent borrow.
- LP: remove/add liquidity, only if the position is still held.
- CCTP: no burn reversal; wait for attestation, retry mint, escalate to manual recovery.
- GMX: manual-only; no automated compensation in MVP.

Compensating actions are planned, routed, simulated, and policy-checked exactly like forward actions. A compensation that fails to simulate escalates to a human alert.

---

## Bridge Timeout Policy

| Transfer | Timeout | Behaviour |
|---|---:|---|
| CCTP Standard | 60 min | Poll Iris; then "delayed — awaiting confirmation"; manual recovery if still stuck |
| CCTP Fast | 5 min | Poll Iris; retry; then escalate |
| Non-USDC | — | Not supported in MVP; `UNSUPPORTED_BRIDGE` |

---

## Adapters

| Role | Adapter |
|---|---|
| USDC cross-chain | `CircleCCTPAdapter` (V2 only) |
| EVM lending | `AaveV3Adapter` (Ethereum, Base, Arbitrum) / `MoonwellAdapter` (Base) |
| Solana lending | `MarginFiAdapter` / `KaminoAdapter` |
| EVM LP | `UniswapV3LPAdapter` / `AerodromeAdapter` (Base) |
| Solana LP | `RaydiumAdapter` |
| Solana swap | `JupiterSwapAdapter` (current Swap API: `/order` + `/build`) |
| GMX | Read-only position reader (no execution adapter in MVP) |
| Non-USDC bridge | None in MVP; Phase 2 LI.FI/Socket |

---

## Resolved Items

1. **GMX compensation** — resolved by D2: GMX is manual-only; the `AWAITING_FULFILLMENT` state is reserved for future automation and unreachable in MVP.
2. **Solana re-sign handoff** — resolved by T11: CCE owns the blockhash refresh and re-sign. The Planner rebuilds the transaction; the CCE signs it. Interface: `CCE.reSign(executionStepId, freshBlockhash)`.
3. **Simulation concurrency cap** — default 8 per plan, configurable by provider limits; tracked in `provider_health`.
4. **RPC providers** — Alchemy for Ethereum/Base/Arbitrum, Helius for Solana, with failover providers configured per chain.
