# Protocol Layer

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D1, D2, D3, D5, T12  
**Canonical schemas:** `docs/canonical_schemas.md` §9–§10  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

**Convert `ExecutionInstructions` into unsigned chain-native transactions.** No routing, no compliance checks, no simulation, no signing. Build bytes and stop.

---

## Adapter Pattern

One adapter class per protocol per chain, implementing a port interface. The dual role is preserved:

- FKL uses adapters for **reads** (`getPositions`, `getPoolAPY`, `getHoldings`).
- Protocol Layer uses the **same adapter class** for **writes** (`buildDepositTx`, `buildWithdrawTx`, `buildSwapTx`).

No duplication between FKL and Protocol Layer adapters.

---

## Adapter Roster

### Category 1 — DEX Adapters

| Chain | Adapter | Notes |
|---|---|---|
| Ethereum | `UniswapV3Adapter` | Direct pool swap |
| Base | `UniswapV3Adapter` | Direct pool swap |
| Base | `AerodromeAdapter` | Stable/correlated pools |
| Arbitrum | `UniswapV3Adapter` | Direct pool swap |
| Solana | `JupiterSwapAdapter` | Current Swap API (`/order` + `/build`) |

No 1inch or UniswapX adapter in MVP. UniswapX is an intent auction, not an aggregator, and is Phase 2.

### Category 2 — Lending Adapters

| Chain | Adapter |
|---|---|
| Ethereum | `AaveV3Adapter` |
| Base | `AaveV3Adapter`, `MoonwellAdapter` |
| Arbitrum | `AaveV3Adapter` |
| Solana | `MarginFiAdapter`, `KaminoAdapter` |

Aave V3 is deployed on Ethereum, Base, and Arbitrum. The adapter code is shared; each chain is a config instance with its own addresses.

### Category 3 — LP Adapters

| Chain | Adapter | `poolType` |
|---|---|---|
| Ethereum | `UniswapV3LPAdapter` | `constant_product` |
| Base | `UniswapV3LPAdapter` | `constant_product` |
| Base | `AerodromeLPAdapter` | `stable_swap` / `correlated` |
| Arbitrum | `UniswapV3LPAdapter` | `constant_product` |
| Solana | `RaydiumLPAdapter` | `constant_product` |

**GMX is read-only.** There is no `GMXGmPoolAdapter` execution adapter in MVP. The read-only `GmxPositionReader` supports portfolio/risk display.

### Category 4 — Bridge Adapters

| Adapter | Mechanism | Chains | Status |
|---|---|---|---|
| `CircleCCTPAdapter` | CCTP V2 native USDC burn-and-mint | Ethereum ↔ Base ↔ Arbitrum ↔ Solana | MVP |
| Non-USDC bridge | — | — | Out of MVP; Phase 2 LI.FI/Socket |

### Category 5 — RWA Adapters

Phase 2. Not registered in MVP.

---

## `poolType`-Aware Transaction Building

### `constant_product` — Uniswap V3, Raydium

Uniswap V3 requires NFT position management (tick range, `mint` with tick bounds). Raydium uses standard AMM LP deposit.

### `stable_swap` / `correlated` — Aerodrome (Base)

Call Aerodrome's router `addLiquidity`/`removeLiquidity` with `stable=true`. ABI and addresses differ from Uniswap V3, hence a separate adapter. StableSwap-aware IL/price-impact math stays in Engines 1–3; the Protocol Layer only builds bytes.

### `gmx_gm_pool` — GMX v2 (Arbitrum): read-only

GMX v2 deposits/withdrawals are asynchronous request/keeper operations. In MVP, CrestFlow **does not build or submit** GMX deposit/withdraw transactions. The GMX reader returns positions for display and `MANUAL_REVIEW_REQUIRED` items.

The async model is documented for Phase 2:

1. `createDeposit` / `createWithdrawal` queues a request with a native-token execution fee.
2. A keeper executes later using oracle prices at execution time.
3. The final LP amount is not knowable at build time; the request can be cancelled/refunded if constraints fail.

No `UnsignedTransaction` is produced for GMX in MVP.

---

## Execution Authority Mode

`ExecutionMode` is `AUTOPILOT`, `MANUAL`, or `READ_ONLY`. There is no ERC-4337 `UserOperation` and no SPL `delegate` path.

| Mode | Who signs | Adapter behaviour |
|---|---|---|
| `AUTOPILOT` | Turnkey TEE (embedded wallet) | Build a plain transaction from the embedded wallet; CCE signs |
| `MANUAL` | User's external wallet | Return the unsigned transaction (EVM) or serialized message (Solana) for the user to sign |
| `READ_ONLY` | Nobody | Never reaches the Protocol Layer |

The adapter's calldata construction does not change by mode. Only the outer signing envelope changes. `UnsignedTransaction.from` is the source address; `executionMode` tells the caller who must sign.

---

## CCTP V2 — Transaction Schema

```typescript
// Source chain: depositForBurn (CCTP V2, 7 arguments)
UnsignedTransaction {
  chain: sourceChain,
  from: sourceAddress,
  to: TOKEN_MESSENGER_V2[sourceChain],
  data: encodeDepositForBurn({
    amount,
    destinationDomain,
    mintRecipient,
    burnToken,
    destinationCaller,       // 0x0 in MVP
    maxFee,                  // 0 for Standard; quoted for Fast
    minFinalityThreshold     // 2000 Standard, 1000 Fast
  }),
  value: 0n,
  executionMode: 'AUTOPILOT' | 'MANUAL'
}

// After Circle attestation (off-chain):
UnsignedTransaction {
  chain: destChain,
  from: sourceAddress,
  to: MESSAGE_TRANSMITTER_V2[destChain],
  data: encodeReceiveMessage(message, attestation),
  value: 0n,
  executionMode: 'AUTOPILOT' | 'MANUAL'
}
```

**Recipient policy:** `mintRecipient` is the embedded Turnkey wallet for Path A and the user's external wallet for Path B. It is never hard-coded to the embedded wallet.

**Domain IDs:**

| Chain | Domain |
|---|---:|
| Ethereum | 0 |
| Arbitrum | 3 |
| Base | 6 |
| Solana | 5 |

**Fees:** Standard has no protocol fee. Fast fees are route-specific and deducted at mint; the adapter exposes `maxFee` and `expectedReceivedAmount = amount - maxFee`. There is no flat fee.

---

## `UnsignedTransaction` Schema

```typescript
UnsignedTransaction {
  chain: Chain,
  type: 'evm' | 'solana',
  executionMode: 'AUTOPILOT' | 'MANUAL',

  // Common
  from: string,
  to?: string,
  data?: Hex,
  value?: BaseUnits,
  gasLimit?: BaseUnits,
  nonce?: number,              // assigned by CCE, not the adapter
  chainId?: number,

  // EVM
  maxFeePerGas?: BaseUnits,
  maxPriorityFeePerGas?: BaseUnits,

  // Solana
  instructions?: TransactionInstruction[],
  computeUnits?: number,
  recentBlockhash?: string,

  // Approval
  approval?: ApprovalInstruction,

  // CCTP
  bridge?: CctpBridgeInstruction,

  // Output
  expectedOutputAmount: BaseUnits
}
```

The adapter does not set the nonce; CCE owns it. The adapter does not sign. The adapter does not simulate.

---

## Approval Model

MVP uses:

1. Standard ERC-20 `approve(spender, amount)` for Uniswap V3, Aerodrome, Moonwell, and other EVM venues.
2. EIP-2612 `supplyWithPermit` for Aave V3 when the token supports it; otherwise `approve` + `supply`.
3. Solana ATA creation via `createAssociatedTokenAccountIdempotent` before SPL transfers.

**Permit2 is Phase 2.** It is not used for Aave V3 (Aave supports EIP-2612, not Permit2). It is not used for UniswapX in MVP. The first Permit2 use always requires a one-time on-chain `approve(Permit2)`; the docs must not claim it eliminates approvals.

Details and edge cases are in `docs/implementation_spec.md` §6.

---

## Protocol Version Management

Adapters are registered via a config-driven manifest keyed by `(port, chain, protocolVersion)`. Adding a chain or protocol version means adding manifest entries and implementing the adapter interface. New adapters are reviewed and merged before production.

---

## What Protocol Layer Does NOT Do

| Not Protocol Layer's Job | Whose Job |
|---|---|
| Routing decisions | Execution Router |
| Price simulation | Planner |
| Business-rule checks | Policy Engine |
| Signing | CCE for AUTOPILOT; user for MANUAL |
| Broadcasting | CCE |
| Nonce allocation | CCE |
| Post-confirmation state updates | PSM |
| Valuation / IL math | Engines 1–3 |
| GMX keeper tracking | Manual in MVP; Phase 2 CCE |

---

## Resolved Items

1. **GMX async tracking** — out of MVP. GMX is read-only/manual; no request tracking is needed.
2. **Non-USDC cross-chain** — out of MVP by D3. CCTP V2 USDC only; Phase 2 LI.FI/Socket.
3. **Permit2 capability** — not used in MVP. Aave uses EIP-2612; Uniswap V3 uses standard approvals. Phase 2 will populate a `supportsPermit2` flag per protocol after ABI verification.
4. **Multi-chain instance vs chain-native** — the manifest records `deploymentKind: 'shared_code' | 'chain_native'` so consumers do not assume 1:1 mapping.
