# Cross-Chain Ecosystem

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D1, D3, D5, T4, T11  
**Canonical schemas:** `docs/canonical_schemas.md` §9, §14  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

**Layer role:** Sign → Broadcast → Confirm. No decisions, no routing, no business rules.

---

## Purpose

The final execution layer. Takes unsigned transaction bytes from the Protocol Layer, signs them in the TEE (Path A only), broadcasts to chain RPCs, and monitors for finality. Returns `ChainExecutionResult` to the Orchestrator.

CrestFlow's servers never see a private key at any point — creation, signing, or otherwise. Turnkey is the TEE provider.

**Execution modes:** Path A (embedded Turnkey wallet) is signed here. Path B (external wallet) is **manual** — CrestFlow returns an unsigned transaction and the user signs in their own wallet; this component verifies and broadcasts the user-signed transaction only after the user submits it back. There is no session key, ERC-4337 delegation, EIP-7702 delegation, or SPL `delegate` for Path B in MVP (D1).

---

## Signing Key Model

**Two keys, not N-per-chain.**

| Account | Curve | Derivation Path | Address Scope |
|---|---|---|---|
| EVM account | secp256k1 | `m/44'/60'/0'/0/0` | **Same address** on Ethereum, Base, and Arbitrum |
| Solana account | Ed25519 | `m/44'/501'/0'/0'` | Solana only |

`chain` is still a required input to the signer — it determines chain ID, transaction encoding rules, and which RPC target broadcast uses — but it does not imply a distinct key or address for the three EVM chains.

**Nonce management:** CCE owns nonce allocation per `(chain, address)` inside a Postgres/Redis lease. The same EVM address on Ethereum, Base, and Arbitrum has three independent nonce sequences. The Orchestrator requests a nonce; it never allocates or signs. See `docs/implementation_spec.md` §4.

### Signing Pipeline (Path A)

```typescript
// Sign request to TEE:
{ walletId, userId, unsignedTxBytes, chain }

// Inside Turnkey enclave:
account    = deriveAccount(subOrgId, walletId, curveFor(chain))
             // curveFor(Ethereum | Base | Arbitrum) → EVM account (secp256k1)
             // curveFor(Solana)                      → Solana account (Ed25519)
privateKey = account.privateKey
signature  = sign(unsignedTxBytes, privateKey)

// Returns to CrestFlow:
{ signedTxBytes }
```

### Manual Execution (Path B)

Path B is **read-only plus manual execution** (D1):

1. Planner builds and simulates the transaction.
2. Policy approves it.
3. CCE returns the unsigned transaction (EVM) or serialized message (Solana) with a `MANUAL_EXECUTION_REQUIRED` package.
4. The user signs in MetaMask/Phantom or another wallet and submits the signed transaction back.
5. CCE verifies the signature against the verified owner, broadcasts, and confirms.

No WalletConnect session key is used for autonomous execution. Wallet-connect may be used only as a transport for the user to sign manually. There is no TEE-resident delegate for external wallets.

---

## Broadcast

| Chain | RPC Method | Provider |
|---|---|---|
| Ethereum | `eth_sendRawTransaction` | Alchemy |
| Base | `eth_sendRawTransaction` | Alchemy |
| Arbitrum | `eth_sendRawTransaction` | Alchemy |
| Solana | `sendTransaction` (versioned) | Helius |

Autopilot broadcasts go directly through the provider. Manual broadcasts accept only a transaction whose signature recovers to the verified `AddressScope` owner; mismatches are rejected with `SIGNATURE_MISMATCH`.

---

## Gas Strategy

All formulas and fee-oracle addresses are authoritative in `docs/implementation_spec.md` §5. Summary:

### EVM — Ethereum

```typescript
maxFeePerGas = baseFee × 2 + maxPriorityFeePerGas
// Cap against GoalProfile.gasPriceCap
// gasPriceCap checked against: maxFeePerGas × gasLimit (L1 execution cost only)
```

### EVM — Base (OP Stack)

```typescript
l2ExecutionFee = (baseFee + priorityFee) × gasUsed
l1DataFee      = GasPriceOracle(0x420000000000000000000000000000000000000F).getL1Fee(bytes)
operatorFee    = GasPriceOracle.getOperatorFee(gasUsed)
totalFee       = l2ExecutionFee + l1DataFee + operatorFee

// gasPriceCap checked against totalFee
```

Do not hand-roll `l1BaseFee × size × scalar`.

### EVM — Arbitrum (Nitro)

```typescript
l2ExecutionFee = (baseFee + priorityFee) × gasUsed
l1DataFee      = NodeInterface(0xc8).gasEstimateL1Component(to, false, data)
totalFee       = l2ExecutionFee + l1DataFee
```

`ArbGasInfo(0x6c).getL1BaseFeeEstimate()` may be used as a cross-check, not as the primary L1 estimate.

### Solana

```typescript
fee = 5_000 lamports × signatures + computeUnitLimit × computeUnitPriceMicroLamports
```

`computeUnitLimit` comes from simulation plus a 10% buffer.

`GoalProfile.gasPriceCap` is checked against the **total fee** including the L1 data component on L2s, not just the L2 execution component.

---

## Gas-Cap Congestion Handling

```typescript
if (totalFee > GoalProfile.gasPriceCap) {
  // Hold in 'queued' state — do NOT broadcast yet
  // Re-check baseFee on each polling interval

  if (totalFee falls below gasPriceCap) → broadcast normally

  if (queuedDuration > notifyThreshold (default: 5 min)) {
    → notify user via Copilot: "waiting on gas, offer manual override to proceed or continue waiting"
  }

  if (queuedDuration > hardTimeout (default: 30 min)) {
    → escalate to Orchestrator as 'GAS_CAP_EXCEEDED' status
  }
}
```

`queued` is distinct from `timeout` (broadcast but not confirmed). The Orchestrator tracks both.

---

## Confirmation Monitoring

| Chain | Practical Finality | Polling Interval |
|---|---|---|
| Ethereum | ~24 seconds (2 blocks post-merge) | Poll receipt every 6s |
| Base | ~2 seconds (sequencer soft-confirmation) | Poll receipt every 1s |
| Arbitrum | ~1–2 seconds (sequencer soft-confirmation) | Poll receipt every 1s |
| Solana | ~0.8 seconds | Poll signature status every 1s |

**L2 finality:** "practical finality" for Base and Arbitrum is sequencer soft-confirmation, not the ~7-day L1 challenge period. CCTP uses its own attestation, not the native rollup bridge.

### Solana Commitment Level

Reports `confirmed` (supermajority cluster confirmation) to the Orchestrator as success, unblocking the next DAG step. Continues polling to `finalized` (~32 slots) for the audit trail. `finalizedAt` is written asynchronously without holding up execution.

### Terminal States

| State | Meaning |
|---|---|
| `confirmed` | Included at practical finality |
| `failed` | Reverted on-chain |
| `dropped` | Not found in mempool after timeout |
| `timeout` | Broadcast but not confirmed within the wait window |
| `queued` | Fee above cap; not yet broadcast |

### Dropped Transaction Re-broadcast (EVM)

Re-broadcast with the same nonce and a gas bump of at least 10–20%, under the CCE nonce lease. Never allocate a new nonce for a replacement.

### Solana Blockhash Expiry Retry

On expiry (~150 slots), CCE fetches a fresh blockhash, asks the Planner to rebuild, re-signs (Path A) or requests a new signature (Path B), and rebroadcasts. Interface: `CCE.reSign(executionStepId, freshBlockhash)`.

---

## Bridge — CCTP V2 Status

CCTP bridge flows return two txIds for Path A, and a burn tx plus a user-signed mint for Path B:

```typescript
{
  sourceTxId: string,       // burn tx on source chain
  targetTxId?: string,      // mint tx on destination chain
  cctpAttestationId: string // Circle attestation identifier, for audit trail
}
```

**Ownership (D5):**

- Router selects speed, domains, and fee parameters.
- Planner builds the burn and post-attestation mint template.
- CCE broadcasts the burn, polls Iris, builds the final mint after attestation, broadcasts it (Path A), and confirms.
- For Path B, the user signs both the burn and the mint; CCE verifies and broadcasts.
- PSM reconciles balances after execution; it does not own the bridge lifecycle.

**Fees:** Standard has no protocol fee (`maxFee = 0`). Fast is opt-in for `HIGH` urgency with a route-specific `maxFee` capped at 0.10% of the transfer or $5, whichever is lower. Expected received amount is `amount - maxFee`. There is no flat fee.

Bridges cannot be cancelled after the burn. If the mint cannot complete, the Orchestrator enters manual recovery; no burn reversal is claimed.

---

## Bridge Provider Reference

| Provider | Mechanism | Status |
|---|---|---|
| CCTP V2 (Circle) | Native USDC burn-and-mint via Iris attestation | **MVP — the only bridge** |
| Non-USDC aggregators (LI.FI/Socket) | Various | Phase 2; out of MVP (D3) |
| Wormhole | Guardian network VAA | Phase 2 — non-USDC only |
| LayerZero | Ultra Light Node messaging | Phase 2 — non-USDC only |
| CCIP (Chainlink) | DON-based messaging | Phase 2 — non-USDC only |

Any non-USDC cross-chain intent fails closed with `UNSUPPORTED_BRIDGE`.

---

## `ChainExecutionResult` Schema

```typescript
ChainExecutionResult {
  userId: UUID,
  executionIntentId: UUID,
  actionId: string,
  chain: Chain,
  txId: string,
  status: 'confirmed' | 'failed' | 'dropped' | 'timeout' | 'queued',
  confirmedAt: DateTime,
  finalizedAt?: DateTime,         // Solana: filled async after 'finalized' commitment
  gasUsedUsd: string,
  // Bridge-specific
  sourceTxId?: string,
  targetTxId?: string,
  cctpAttestationId?: string,
  bridgeFeeUsd?: string           // zero for Standard; quoted maxFee for Fast
}
```

This result schema is a component-local extension; shared fields follow `docs/canonical_schemas.md`.

---

## What Cross-Chain Ecosystem Does NOT Do

| Not Cross-Chain Ecosystem's Job | Whose Job |
|---|---|
| Routing / chain selection | Engine 6 |
| Pool and CCTP parameter selection | Execution Router |
| Transaction byte construction | Protocol Layer / Planner |
| Business rule checks | Policy Engine |
| Simulation | Planner |
| Portfolio state updates | Portfolio State Manager |
| Pool valuation / `poolType` math | Engines 1–3 |
| Nonce allocation | CCE (this component) |

---

## Resolved Items

1. **Path B authority** — D1: manual-only. No session key, no ERC-4337, no SPL delegate. Removed from MVP.
2. **Solana re-sign handoff** — T11: CCE owns the blockhash refresh and re-sign; the Planner rebuilds.
3. **Per-chain nonce tracking home** — T11: CCE owns it; lease key `(chain, address)` in Postgres/Redis.
4. **`queued` status downstream handling** — Orchestrator tracks `queued` separately and notifies via Copilot; the state is included in `ChainExecutionResult`.
5. **L2 native-bridge slow path** — not in MVP; non-USDC bridges are Phase 2 and would need a long-lived finality state.
