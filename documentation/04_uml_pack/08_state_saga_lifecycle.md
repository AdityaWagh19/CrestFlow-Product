# 08 — State Machine: Execution Saga Lifecycle

> **Diagram type:** UML State Machine (stateDiagram-v2)
> **Lucidchart template:** UML State Machine
> **Owner:** CrestFlow Platform Engineering
> **Relates to:** `implementation_spec §7`

---

## 1. Purpose & Scope

This specification defines the state machine for the **Execution Saga** — the top-level aggregate that tracks a cross-chain or single-chain transaction from user intent through on-chain finality.

**In scope:**
- All saga lifecycle states from `CREATED` through terminal states `COMPLETED`, `BLOCKED`, `COMPENSATED`
- Compensation / rollback sub-path
- CCTP (Circle Cross-Chain Transfer Protocol) sub-path
- Emergency pause path triggered by CRITICAL alerts
- GMX Phase 2 reserved state (`AWAITING_FULFILLMENT`)
- Policy Check gates (Policy Check 1, 2, 3)

**Out of scope:**
- KYC lifecycle (see File 09)
- Delegation / Autopilot lifecycle (see File 10)
- Internal step-level state machines (e.g., individual BridgeStep, SwapStep)

**Primary entity:** `ExecutionSaga` (database table `execution_sagas`)
**Primary key:** `saga_id` (UUID)
**Status field:** `ExecutionSaga.status` (enum `SagaStatus`)

---

## 2. States Inventory

| State | Type | Description | Entry Action | Exit Action |
|---|---|---|---|---|
| *(initial pseudo-state)* | initial | UML filled circle; no DB row yet | — | Create `ExecutionSaga` row, status=CREATED |
| `CREATED` | intermediate | Saga persisted; intent validated; not yet routed | Emit `SAGA_CREATED` audit event; assign `saga_id` | — |
| `ROUTING` | intermediate | Route-finding in progress; best path being selected across DEX/bridge adapters | Invoke RoutingEngine; start routing timeout | Cancel routing timeout on exit |
| `BUILDING` | intermediate | Transaction(s) being constructed: calldata assembled, gas estimated | Invoke TxBuilder; populate `steps[]` | — |
| `SIMULATING` | intermediate | Simulation of execution path; slippage/revert checks | Invoke SimulationService (Tenderly or equiv.) | — |
| `POLICY_CHECK` | choice/intermediate | Three sequential policy gates evaluated | Evaluate Policy Check 1 (KYC), Policy Check 2 (limits), Policy Check 3 (delegation) | — |
| `BLOCKED` | terminal | Policy gate rejected execution; no further processing | Emit `SAGA_BLOCKED` audit event; notify user; record `block_reason` | — |
| `AWAITING_SIGNATURE` | intermediate | Manual signing path: unsigned tx payload delivered to user wallet for signing | Emit `SIGNATURE_REQUESTED` audit event; start signature timeout | Cancel signature timeout |
| `SIGNING` | intermediate | Autopilot path: Turnkey sub-org signing in progress | Submit signing request to Turnkey API; start signing timeout | — |
| `BROADCASTING` | intermediate | Signed transaction(s) submitted to mempool | Submit to RPC endpoint(s); emit `TX_BROADCAST` audit event | — |
| `AWAITING_CONFIRMATION` | intermediate | Waiting for on-chain inclusion and sufficient block confirmations | Start confirmation poller; set confirmation threshold | Stop poller |
| `AWAITING_ATTESTATION` | intermediate | **CCTP path only.** Waiting for Circle attestation service to sign the burn proof | Start Circle attestation polling | Stop Circle poller |
| `MINTING` | intermediate | **CCTP path only.** Submitting mint tx on destination chain using Circle attestation | Invoke destination-chain mint call | — |
| `CONFIRMING` | intermediate | Final confirmation count accumulating (post-mint for CCTP, or post-broadcast for non-CCTP) | Start final-confirmation poller | Stop poller |
| `COMPLETED` | terminal | Saga fully settled on-chain; all steps confirmed | Emit `SAGA_COMPLETED` audit event; update portfolio balances; release fee | — |
| `FAILED` | intermediate | Irrecoverable on-chain failure (revert, timeout) detected | Emit `SAGA_FAILED` audit event; trigger compensation decision | — |
| `COMPENSATING` | intermediate | Rollback actions being executed (reverse swaps, refunds) | Emit `COMPENSATION_STARTED` audit event; execute compensating steps | — |
| `COMPENSATED` | terminal | All compensating actions confirmed; funds returned to origin | Emit `SAGA_COMPENSATED` audit event; notify user | — |
| `PAUSED` | intermediate | Emergency pause: CRITICAL alert fired; not-yet-broadcast actions halted | Emit `SAGA_PAUSED` audit event; freeze pending actions; disable autopilot for saga | — |

---

## 3. Transitions Table

| From State | To State | Trigger / Event | Guard Condition | Action | Notes |
|---|---|---|---|---|---|
| *(initial)* | `CREATED` | `POST /v1/executions` received | Request authenticated; wallet exists | Persist `ExecutionSaga`; validate intent schema | — |
| `CREATED` | `ROUTING` | Intent validation passes | `saga.intent` well-formed | Dispatch to RoutingEngine | — |
| `ROUTING` | `BUILDING` | Route selected | At least one viable route found | Store `saga.route`; populate adapter config | — |
| `ROUTING` | `FAILED` | Routing timeout / no route | No viable route within timeout | Record `failure_reason=NO_ROUTE` | — |
| `BUILDING` | `SIMULATING` | Tx construction complete | All steps built; calldata non-null | — | — |
| `BUILDING` | `FAILED` | Build error | Calldata assembly fails | Record `failure_reason=BUILD_ERROR` | — |
| `SIMULATING` | `POLICY_CHECK` | Simulation passes | Simulation result = success; slippage within tolerance | — | — |
| `SIMULATING` | `FAILED` | Simulation reverts | Simulation result = revert | Record `failure_reason=SIMULATION_REVERT` | — |
| `POLICY_CHECK` | `BLOCKED` | Policy Check 1 fails | `kyc.status ≠ APPROVED` | Record `block_reason=KYC_NOT_APPROVED` | Policy Check 1 |
| `POLICY_CHECK` | `BLOCKED` | Policy Check 2 fails | Daily volume limit exceeded for user tier | Record `block_reason=LIMIT_EXCEEDED` | Policy Check 2 |
| `POLICY_CHECK` | `BLOCKED` | Policy Check 3 fails | Autopilot `REVOKED` or `PAUSED`; execution mode=AUTOPILOT | Record `block_reason=AUTHORITY_REVOKED` | Policy Check 3 |
| `POLICY_CHECK` | `BLOCKED` | Policy Check 3 fails (stale) | Autopilot `ACTIVE` but delegation timestamp > 4h | Record `block_reason=AUTHORITY_STALE` | Policy Check 3 |
| `POLICY_CHECK` | `AWAITING_SIGNATURE` | All policy checks pass | `wallet.executionMode = MANUAL` | Deliver unsigned tx payload to client | Manual path |
| `POLICY_CHECK` | `SIGNING` | All policy checks pass | `wallet.executionMode = AUTOPILOT`; delegation ACTIVE + fresh (≤ 4h) | Submit to Turnkey signing API | Autopilot path |
| `AWAITING_SIGNATURE` | `BROADCASTING` | User signs & submits | Valid signature received; nonce correct | Attach signature to tx; enqueue broadcast | — |
| `AWAITING_SIGNATURE` | `FAILED` | Signature timeout | No signature within configured window | Record `failure_reason=SIGNATURE_TIMEOUT` | — |
| `SIGNING` | `BROADCASTING` | Turnkey signing complete | Signed tx returned from Turnkey | Attach signature; enqueue broadcast | — |
| `SIGNING` | `FAILED` | Turnkey error | Turnkey API error or timeout | Record `failure_reason=SIGNING_ERROR` | — |
| `BROADCASTING` | `AWAITING_CONFIRMATION` | Tx accepted by mempool | `txHash` returned from RPC | Store `saga.txHash`; start confirmation poller | Non-CCTP path |
| `BROADCASTING` | `AWAITING_ATTESTATION` | Burn tx confirmed on source chain | CCTP burn event detected; `messageHash` available | Store `saga.messageHash`; start Circle poller | CCTP path only |
| `BROADCASTING` | `FAILED` | RPC rejection / gas failure | RPC returns error; not retryable | Record `failure_reason=BROADCAST_FAILED` | — |
| `AWAITING_CONFIRMATION` | `CONFIRMING` | Sufficient confirmations reached | Block confirmations ≥ threshold | — | Non-CCTP path |
| `AWAITING_CONFIRMATION` | `FAILED` | Confirmation timeout | Confirmation not reached within window | Record `failure_reason=CONFIRMATION_TIMEOUT` | — |
| `AWAITING_ATTESTATION` | `MINTING` | Circle attestation received | `attestation.status = complete`; attestation signature valid | Submit mint tx on destination chain | CCTP path |
| `AWAITING_ATTESTATION` | `FAILED` | Attestation timeout | Circle attestation not received within window | Record `failure_reason=ATTESTATION_TIMEOUT` | CCTP path |
| `MINTING` | `CONFIRMING` | Mint tx confirmed | Destination-chain mint tx included + confirmations ≥ threshold | — | CCTP path |
| `MINTING` | `FAILED` | Mint failure | Mint tx reverts or times out | Record `failure_reason=MINT_FAILED` | CCTP path |
| `CONFIRMING` | `COMPLETED` | Final confirmation reached | All steps confirmed; total confirmations ≥ finality threshold | Emit `SAGA_COMPLETED`; update balances | — |
| `CONFIRMING` | `FAILED` | Final confirmation timeout | — | Record `failure_reason=FINAL_CONFIRMATION_TIMEOUT` | — |
| `FAILED` | `COMPENSATING` | Compensation decision made | Compensatable steps exist; compensation policy = auto | Emit `COMPENSATION_STARTED` | — |
| `FAILED` | `COMPENSATED` | No compensating steps needed | No broadcast steps executed | Emit `SAGA_COMPENSATED` with zero-step note | Shortcut path |
| `COMPENSATING` | `COMPENSATED` | All compensating actions confirmed | Compensating txs included on-chain | Emit `SAGA_COMPENSATED` | — |
| `COMPENSATING` | `COMPENSATED` | Partial rollback accepted | Some steps non-reversible; partial funds returned | Record `compensation_type=PARTIAL_ROLLBACK` | — |
| `COMPENSATING` | `COMPENSATED` | Manual recovery required | Compensation itself fails; ops intervenes | Record `compensation_type=MANUAL_RECOVERY`; page on-call | — |
| `PAUSED` | `BROADCASTING` | Emergency resolved; operator resumes | CRITICAL alert cleared; operator triggers resume | Re-enable pending actions | Only for sagas not yet broadcast |
| `PAUSED` | `FAILED` | Operator decides to abort paused saga | Operator explicit abort | — | — |
| *(any non-terminal)* | `PAUSED` | CRITICAL alert fires | `alert.severity = CRITICAL`; saga not yet broadcast | Freeze pending actions; disable autopilot for saga | Emergency path |

> **Note:** `AWAITING_FULFILLMENT` state is **reserved but unreachable in MVP** (GMX Phase 2). Do not add transitions to/from it in MVP diagrams. Reserve a shape with a "GMX Phase 2" annotation.

---

## 4. Special Paths

### 4.1 CCTP Sub-Path
Applies when `saga.bridgeProtocol = CCTP` (Circle Cross-Chain Transfer Protocol).

```
BROADCASTING → AWAITING_ATTESTATION → MINTING → CONFIRMING → COMPLETED
```

- After `BROADCASTING`, the system monitors for a Circle burn event rather than standard confirmation.
- `AWAITING_ATTESTATION`: polls Circle Attestation API until `status=complete`.
- `MINTING`: submits the Circle-signed mint calldata to the destination chain.
- Diagram note: branch the CCTP path off `BROADCASTING` with a guard label `[bridgeProtocol=CCTP]`.

### 4.2 Compensation Path
Triggered when a FAILED saga has executable compensating steps.

```
FAILED → COMPENSATING → COMPENSATED (terminal)
```

Three sub-outcomes all resolve to `COMPENSATED`:
| Sub-outcome | `compensation_type` |
|---|---|
| Full rollback confirmed | `ROLLED_BACK` |
| Some steps non-reversible | `PARTIAL_ROLLBACK` |
| Compensation itself fails | `MANUAL_RECOVERY` |

### 4.3 Emergency Pause Path
Triggered by a CRITICAL-severity monitoring alert (e.g., price oracle manipulation, smart contract exploit).

```
(any non-terminal, pre-broadcast) → PAUSED
```

- Actions **already broadcast** are allowed to resolve normally (not recalled).
- Actions **not yet broadcast** are frozen.
- Autopilot is disabled for the affected saga.
- Resume path: `PAUSED → BROADCASTING` (operator-triggered).
- Abort path: `PAUSED → FAILED → (compensation as normal)`.

### 4.4 GMX Reserved State
`AWAITING_FULFILLMENT` — reserved for GMX limit-order fulfillment polling (Phase 2). **Not reachable in MVP.** Diagram must include the shape with a dashed border and the annotation: *"GMX Phase 2 — not reachable in MVP"*.

---

## 5. Lucidchart Shape Guide

| Element | UML Shape | Lucidchart Shape Name | Style Notes |
|---|---|---|---|
| Initial pseudo-state | Filled circle | `UML Initial State` | Black fill, no border |
| Terminal state | Circle-in-circle (bullseye) | `UML Final State` | Black inner circle |
| Intermediate state | Rounded rectangle | `UML State` | White fill, 1px border |
| Choice pseudo-state | Diamond | `UML Choice` | White fill; use for POLICY_CHECK gate |
| Transition | Arrow | `Connection` with open arrowhead | Label: `event [guard] / action` |
| PAUSED state | Rounded rectangle | `UML State` | Yellow fill (#FFF3CD) to indicate suspended |
| Terminal error states | Rounded rectangle | `UML State` | Light red fill (#FDECEA) for BLOCKED, COMPENSATED |
| COMPLETED | Rounded rectangle | `UML State` | Light green fill (#E6F4EA) |
| Reserved state | Rounded rectangle | `UML State` | Dashed border, grey fill (#F5F5F5) |
| Sub-path label | Swimlane or note | `UML Note` | Attach note shape to CCTP path branch |
| Guard condition | Bracket notation on arrow | Inline label | `[bridgeProtocol=CCTP]` style |

---

## 6. Layout Recommendations

- **Orientation:** Top-to-bottom (TD) for the happy path.
- **Happy path spine:** Center column, top-to-bottom:
  `CREATED → ROUTING → BUILDING → SIMULATING → POLICY_CHECK → SIGNING/AWAITING_SIGNATURE → BROADCASTING → AWAITING_CONFIRMATION → CONFIRMING → COMPLETED`
- **CCTP branch:** Offset to the right of `BROADCASTING`; re-join at `CONFIRMING`.
- **Policy Check fork:** Use a diamond (choice) shape for `POLICY_CHECK`; fan `BLOCKED` to the right.
- **Compensation path:** Place below `FAILED`, flowing left-to-right: `FAILED → COMPENSATING → COMPENSATED`.
- **PAUSED:** Place to the left of the happy path with a horizontal arrow from any pre-broadcast state.
- **Reserved state:** Bottom-right corner with dashed box.
- **Approximate canvas size:** 1800×2400px at 100% zoom.
- **Group CCTP states** (`AWAITING_ATTESTATION`, `MINTING`) inside a composite state box labeled "CCTP Sub-Path".

---

## 7. Supplementary Mermaid

```mermaid
stateDiagram-v2
    [*] --> CREATED : POST /v1/executions

    CREATED --> ROUTING : intent valid
    ROUTING --> BUILDING : route found
    ROUTING --> FAILED : no route / timeout

    BUILDING --> SIMULATING : tx built
    BUILDING --> FAILED : build error

    SIMULATING --> POLICY_CHECK : simulation passes
    SIMULATING --> FAILED : simulation reverts

    state POLICY_CHECK <<choice>>
    POLICY_CHECK --> BLOCKED : KYC not approved [PC1]
    POLICY_CHECK --> BLOCKED : limit exceeded [PC2]
    POLICY_CHECK --> BLOCKED : authority revoked/stale [PC3]
    POLICY_CHECK --> AWAITING_SIGNATURE : all checks pass [MANUAL]
    POLICY_CHECK --> SIGNING : all checks pass [AUTOPILOT]

    AWAITING_SIGNATURE --> BROADCASTING : signed tx received
    AWAITING_SIGNATURE --> FAILED : signature timeout

    SIGNING --> BROADCASTING : Turnkey signing complete
    SIGNING --> FAILED : Turnkey error

    state BROADCASTING <<choice>>
    BROADCASTING --> AWAITING_CONFIRMATION : txHash received [non-CCTP]
    BROADCASTING --> AWAITING_ATTESTATION : burn confirmed [CCTP]
    BROADCASTING --> FAILED : RPC rejection

    AWAITING_CONFIRMATION --> CONFIRMING : confirmations >= threshold
    AWAITING_CONFIRMATION --> FAILED : confirmation timeout

    state CCTP_SUB {
        AWAITING_ATTESTATION --> MINTING : attestation received
        AWAITING_ATTESTATION --> FAILED : attestation timeout
        MINTING --> CONFIRMING : mint confirmed
        MINTING --> FAILED : mint failed
    }

    CONFIRMING --> COMPLETED : finality reached
    CONFIRMING --> FAILED : final confirmation timeout

    FAILED --> COMPENSATING : compensatable steps exist
    FAILED --> COMPENSATED : no steps to compensate

    COMPENSATING --> COMPENSATED : rollback complete
    COMPENSATING --> COMPENSATED : partial rollback
    COMPENSATING --> COMPENSATED : manual recovery

    note right of COMPENSATED
        compensation_type:
        ROLLED_BACK | PARTIAL_ROLLBACK | MANUAL_RECOVERY
    end note

    PAUSED --> BROADCASTING : operator resumes
    PAUSED --> FAILED : operator aborts

    note left of AWAITING_FULFILLMENT
        GMX Phase 2 — not reachable in MVP
    end note

    COMPLETED --> [*]
    BLOCKED --> [*]
    COMPENSATED --> [*]
```

---

## 8. Flags & Open Questions

| # | Flag | Detail | Owner | Status |
|---|---|---|---|---|
| F1 | `AWAITING_FULFILLMENT` scope | GMX Phase 2: define entry/exit events before enabling | Protocol team | Open |
| F2 | Compensation policy | Auto-compensate vs. manual decision threshold not finalized | Risk/Ops | Open |
| F3 | CCTP confirmation threshold | How many destination-chain confirmations constitute finality? | Engineering | Open |
| F4 | Signature timeout window | Configurable? Per-chain? Default value? | Product | Open |
| F5 | PAUSED resume authorization | Who can trigger resume — ops only, or also user? | Compliance | Open |
| F6 | Partial rollback definition | Exact criteria for `PARTIAL_ROLLBACK` vs `MANUAL_RECOVERY` | Engineering | Open |
| F7 | Multi-step saga | Does `CONFIRMING` wait for ALL steps, or just the final step? | Engineering | Open |
