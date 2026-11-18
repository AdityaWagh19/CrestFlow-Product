# Portfolio State Analysis

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D1, T6, T7, T9  
**Canonical schemas:** `docs/canonical_schemas.md` §5  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

Sits between Identity & KYC and the Core Intelligence Layer. Answers one question: "What does this user actually own, and where, right now, across all four chains?" This is a data-collection layer, not an intelligence layer. It gathers raw truth and hands it to Engine 1.

---

## Address Model

The address model is defined by onboarding (`docs/user_onboarding.md`) and `docs/canonical_schemas.md` §3. Portfolio State Analysis consumes it and does not re-derive it.

**Path A (embedded Turnkey wallet):** one embedded wallet with two address strings — one EVM address shared across Ethereum, Base, and Arbitrum, and one Solana address. Execution mode is `AUTOPILOT`.

**Path B (external wallets):** N external addresses per chain, each `MANUAL` (read-only tracking plus user-signed execution) or `READ_ONLY` (watch-only). Path B never receives delegated authority; there is no ERC-4337 session key and no SPL `delegate` in MVP (D1).

**Ownership proof:** an external address is added only after a wallet-signature challenge succeeds; `AddressScope.ownershipVerifiedAt` records it. This is a signed message, not an on-chain delegation.

**Embedded wallet scan geometry:** two unique address strings, four separate chain-scans. The same EVM address on Ethereum, Base, and Arbitrum holds three independent balance and position sets. The scan fires three separate adapter calls against that address.

---

## What It Covers

| What | Means |
|---|---|
| Assets | Tokens and coins the user holds |
| Holdings | Actual quantities and current USD values |
| Positions | Open DeFi positions — lending, LP, staking, borrowing |
| History | Past transactions — cost basis, PnL, performance |

---

## The Scan

For each registered `(address, chain)` pair, all adapters fire in parallel:

```
BlockchainDataPort.getHoldings(address, chain)       → raw token balances
BlockchainDataPort.getTransactionHistory(address)    → transaction history for cost basis
LendingPort.getPositions(address, chain)             → open supply/borrow positions
LPPort.getPositions(address, chain)                  → open LP positions
RWAPort.getPositions(address, chain)                 → tokenized asset positions (Phase 2)
```

All fired with `Promise.allSettled` — one adapter failing does not crash the scan. Per-adapter data quality is `COMPLETE`, `PARTIAL`, or `STALE`.

**Partial chain failure:** if one chain-scan fails, the scan proceeds with the remaining chains. The failed chain's last known data is blended in as `STALE` rather than omitted. Stale data is excluded from freshness-sensitive calculations (e.g., return computations for that window) but still contributes to total value and allocation so the portfolio view stays coherent.

The Financial Knowledge Layer handles:

- calling the correct adapter per chain;
- Redis caching (T6: positions 30s, pool data 300s, metadata 7d);
- normalising raw responses into canonical types.

If a chain scan exceeds the 10-second `maxWait`, the snapshot is `PARTIAL`, not complete.

---

## Cross-Chain Price Unification

WETH has distinct contract addresses on each chain. The Asset Registry maintains per-chain entries per symbol:

```
Ethereum (chain 1):     0xC02aaA...   → symbol: ETH → coingeckoId: ethereum
Base (chain 8453):      0x4200...06   → symbol: ETH → coingeckoId: ethereum
Arbitrum (chain 42161): 0x82aF49...   → symbol: ETH → coingeckoId: ethereum
Solana:                 7vfCXTUXx5... → symbol: ETH → coingeckoId: ethereum
```

The canonical symbol + CoinGecko ID resolves cross-chain double-counting at the price layer. The Asset Registry keeps four real entries per multi-chain asset, not one aggregated entry.

**Execution pricing is oracle-only** (T4). CoinGecko is for discovery, display, and ranking.

---

## Position Types Across Chains

| Position Type | Ethereum | Base | Arbitrum | Solana |
|---|---|---|---|---|
| Lending Supply | Aave V3 aToken | Moonwell mToken / Aave V3 aToken | Aave V3 aToken | MarginFi / Kamino deposit |
| Lending Borrow | Aave V3 variable debt | Moonwell borrow / Aave V3 debt | Aave V3 variable debt | MarginFi / Kamino borrow |
| LP Position | Uniswap V3 NFT | Aerodrome LP / Uniswap V3 NFT | Uniswap V3 NFT | Raydium LP token |
| Perps / GM | — | — | GMX GM (read-only) | — |
| RWA | Backed Finance | Phase 2 | Phase 2 | Phase 2 |

Each LendingPort and LPPort adapter normalises into the same canonical `Position` union (`docs/canonical_schemas.md` §5) regardless of chain. GMX positions use `GmxPosition` with `executionMode: 'MANUAL_ONLY'` and never generate automated actions.

---

## Transaction History and Cost Basis

- **Required fields:** acquisition price/time per asset (cost basis), LP entry price ratio (for IL calculation), and yield earned (performance attribution).
- **Caching:** past transactions are immutable — full history is cached and only transactions since the last cached timestamp are fetched.
- **Scale:** four chain-scans, each requiring its own indexer call. Alchemy covers Ethereum/Base/Arbitrum; Helius covers Solana.

---

## Scan Triggers

| Trigger | When |
|---|---|
| `ONBOARDING` | First scan after wallet/address setup completes |
| `MANUAL` | User selects "Refresh" |
| `POST_EXECUTION` | Immediately after any on-chain transaction confirms |
| `SCHEDULED` | Background periodic refresh |
| `EMERGENCY` | Watchdog condition (PSM) |

---

## Output

The scan produces a canonical `PortfolioSnapshot` (`docs/canonical_schemas.md` §5):

```typescript
PortfolioSnapshot {
  id, userId, createdAt,
  dataQuality: 'COMPLETE' | 'PARTIAL' | 'STALE',
  totalValueUsd,
  positions: Position[],
  chainExposureUsd, assetExposureUsd, protocolExposureUsd,
  hhi: { asset, protocol, chain }
}
```

The legacy shape `dataQuality: { ethereum: 'ok', ... }` is replaced by the canonical single enum plus per-position quality. Per-chain detail lives on each `Position.dataQuality`; the snapshot is `PARTIAL` if any included position is `PARTIAL` while fresh data exists elsewhere, and `STALE` only when no included position has fresh data. Execution fails closed when pricing is stale.

---

## Connection to Portfolio State Manager

The PSM owns sync, tracking, monitoring, and updates. The consistency mechanism is: `ExecutionConfirmed` → `POST_EXECUTION` scan → new Engine 1 snapshot → downstream engine pipeline.

---

## Resolved Items

1. **Indexer strategy per chain** — Alchemy (EVM) plus Helius (Solana), with failover providers configured per chain; see `docs/implementation_spec.md` §10.
2. **Scheduled scan frequency** — per-chain scheduling is allowed; frequencies are in `docs/portfolio_state_manager.md`.
3. **Cross-chain LP positions** — Phase 2. Not handled by the MVP LPPort model.
4. **Path B ownership proof** — wallet-signature challenge, recorded as `ownershipVerifiedAt`; no on-chain delegation (D1/T9).
