# Financial Knowledge Layer

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` T4, T6, T12  
**Canonical schemas:** `docs/canonical_schemas.md`  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

Pure infrastructure. Sits left of the Core Intelligence Layer and feeds market, protocol, research, and risk data to every engine. No intelligence, no computation — fetches, caches, normalises. Engines never call external APIs directly; they interact only with canonical types produced here.

---

## Structure: Four Sub-Layers

```
External World (APIs, blockchains, oracles)
        ↓
[1] Adapter Layer        ← chain/protocol-specific fetchers
        ↓
[2] Cache Layer          ← Redis TTL, shields external APIs
        ↓
[3] Normaliser Layer     ← raw → canonical types
        ↓
[4] Price Service        ← unified pricing across all chains
        ↓
Engines (consume canonical types only)
```

---

## Sub-Layer 1 — Adapters

Ethereum, Base, and Arbitrum are distinct rows in every table. They share a VM but not a protocol ecosystem, and Alchemy covers them as distinct networks. Each chain requires separate adapter calls even for shared protocols such as Aave V3.

### BlockchainDataPort — On-Chain State

| Chain | Adapter | What It Fetches |
|---|---|---|
| Ethereum | Alchemy | ERC-20 balances, NFTs, transaction history, event logs |
| Base | Alchemy | ERC-20 balances, NFTs, transaction history, event logs |
| Arbitrum | Alchemy | ERC-20 balances, NFTs, transaction history, event logs |
| Solana | Helius | SPL token balances, transaction history, account data |
| Any EVM (fallback) | Viem + RPC | Direct RPC calls if Alchemy is degraded |

### LendingPort — Lending/Borrowing Positions and Rates

| Protocol | Chain | Status | What It Fetches |
|---|---|---|---|
| Aave V3 | Ethereum | MVP | aToken balances, debt positions, pool APY, liquidation threshold |
| Aave V3 | Base | MVP | Supply/borrow positions, pool APY, liquidation threshold |
| Moonwell | Base | MVP | Supply/borrow positions, pool APY, liquidation threshold |
| Aave V3 | Arbitrum | MVP | Supply/borrow positions, pool APY, liquidation threshold |
| MarginFi | Solana | MVP | Lending accounts, APYs, collateral ratios |
| Kamino | Solana | MVP | Lending accounts, APYs, collateral ratios |

### LPPort — Liquidity Pool Positions and Rates

| Protocol | Chain | Pool Type | What It Fetches |
|---|---|---|---|
| Uniswap V3 | Ethereum | `constant_product` | NFT LP positions, tick ranges, pool reserves, fee APR |
| Uniswap V3 | Base | `constant_product` | NFT LP positions, tick ranges, pool reserves, fee APR |
| Aerodrome | Base | `stable_swap` / `correlated` / `constant_product` | LP positions, pool reserves, fee/vote-incentive APR |
| Uniswap V3 | Arbitrum | `constant_product` | NFT LP positions, tick ranges, pool reserves, fee APR |
| GMX v2 | Arbitrum | `gmx_gm_pool` | GM positions for display only; manual execution |
| Raydium | Solana | `constant_product` | LP positions, pool reserves, fee APR |

GMX is read-only in MVP (D2). The reader feeds portfolio and risk display; it never feeds automated routing.

### PriceOraclePort — Token Pricing

| Adapter | Type | Role |
|---|---|---|
| Chainlink | On-chain (EVM) | **Primary execution price on EVM** |
| Pyth | On-chain (EVM + Solana) | **Fallback on EVM; primary on Solana** |
| CoinGecko | Off-chain REST | Discovery, display, and ranking only — never an execution price |
| DEX TWAP | On-chain | Cross-check when only one oracle feed exists |

**Execution pricing rules (T4):**

- Execution uses an on-chain oracle, never CoinGecko.
- Chainlink and Pyth are cross-checked on EVM; divergence above **1.5%** blocks signing.
- A feed older than **60 seconds** blocks signing.
- If only one feed exists, a DEX TWAP cross-check above **1%** blocks signing.
- CoinGecko vs oracle divergence above **5%** is a data-quality warning, not an execution halt.
- L2 sequencer uptime is checked before signing; a down sequencer blocks signing.

### DEXPort — Swap Quotes and Routing Data

| Adapter | Chain | Role |
|---|---|---|
| Uniswap V3 quoter | Ethereum, Base, Arbitrum | Direct pool quotes |
| Aerodrome router | Base | Stable/correlated pool quotes |
| Jupiter Swap API (`/order` + `/build`) | Solana | Route and transaction build |
| Raydium | Solana | Direct pool quotes |

There is no 1inch or UniswapX quote path in MVP (T12). They are Phase 2.

### BridgePort — Cross-Chain Transfer Parameters

| Adapter | Mechanism | Chains | Status |
|---|---|---|---|
| Circle CCTP V2 | Native USDC burn-and-mint | Ethereum ↔ Base ↔ Arbitrum ↔ Solana | MVP — the only bridge |
| Non-USDC aggregators | LI.FI / Socket | — | Phase 2; out of MVP |

Any non-USDC cross-chain request fails closed with `UNSUPPORTED_BRIDGE`.

---

## Sub-Layer 2 — Cache Layer

TTLs are canonical in T6.

| Data Type | TTL | Reason |
|---|---:|---|
| Token prices (display) | 30s | Rate-limit-bound; display freshness |
| Wallet holdings | 30s | Changes after every transaction |
| Open DeFi positions | 30s | Liquidation proximity changes fast |
| Pool APYs / rates | 300s | Meaningful changes happen over minutes |
| Protocol safety scores | 24h | Computed daily from DeFiLlama |
| Historical daily prices | 24h | Underlying table is append-only; cache refreshes daily |
| Static asset metadata | 7d | Name, symbol, decimals never change |

**Stale-while-revalidate:** allowed up to 2× TTL with `dataQuality = 'STALE'`. Beyond 2× TTL, execution fails closed. A stale cache never supplies an execution price; the Price Service re-quotes at build time.

**Cache key structure:**

```
crestflow:{chain}:{adapter}:{resource}:{identifier}
e.g. crestflow:ethereum:alchemy:holdings:0xABCD...
     crestflow:base:alchemy:holdings:0xABCD...
     crestflow:arbitrum:alchemy:holdings:0xABCD...
     crestflow:price:chainlink:ethereum:usdc
```

The embedded wallet reuses the same address across Ethereum, Base, and Arbitrum. `chain` is a key segment, so the three holdings sets never collide.

**Graceful degradation:** Redis unreachable → bypass cache → fetch live → log warning → tag `dataQuality = 'STALE'` if the live fetch also fails. Never crash. Engines always receive data; execution blocks if the data is not fresh enough.

---

## Sub-Layer 3 — Normaliser Layer

Raw provider responses are chain-specific and heterogeneous. The Normaliser converts everything into canonical types (`docs/canonical_schemas.md`).

```
Alchemy (Ethereum) raw:
  { contractAddress: "0xA0b8...", tokenBalance: "0x4C4B40" }

Helius (Solana) raw:
  { mint: "EPjFWdd5...", amount: "5000000", decimals: 6 }

After normalisation (both become):
  AssetRef + BaseUnits + Usd
  {
    asset: { symbol: "USDC", chain: "ethereum" | "solana", decimals: 6 },
    amount: "5000000",
    valueUsd: "5.00000000"
  }
```

**`bigint` / `decimal.js` rule:** token amounts are integer base units; USD values are decimal strings computed with `decimal.js`, never native IEEE 754 arithmetic. No monetary field is a JavaScript `number`.

**`poolType`:** every pool record carries the canonical `PoolType`. Classification is owned by the shared `@crestflow/pool-type` package (T11), not reimplemented per adapter. The adapter supplies raw pool metadata; the shared package classifies it.

---

## Sub-Layer 4 — Price Service

### Asset Registry

Keyed by `(chainId, tokenAddress)`, one entry per chain per symbol.

```typescript
Symbol: "ETH"
  entries: [
    { chainId: 1,        address: "0x0000...0000",  decimals: 18 },  // Ethereum, native
    { chainId: 8453,     address: "0x4200...0006",  decimals: 18 },  // Base, WETH
    { chainId: 42161,    address: "0x82aF49...",    decimals: 18 },  // Arbitrum, WETH
    { chainId: "solana", address: "7vfCXTUXx5...",  decimals: 8  }   // Solana, Wrapped ETH
  ]
  coingeckoId: "ethereum"

Symbol: "USDC"
  entries: [
    { chainId: 1,        address: "0xA0b86991...", decimals: 6 },
    { chainId: 8453,     address: "0x8335...",     decimals: 6 },
    { chainId: 42161,    address: "0xaf88d0...",   decimals: 6 },
    { chainId: "solana", address: "EPjFWdd5...",   decimals: 6 }
  ]
  coingeckoId: "usd-coin"
```

The registry also stores oracle feed addresses, decimals, and heartbeats per asset per chain.

### Pricing paths

- **Display / analysis:** CoinGecko Pro (`/simple/price`, up to 515 IDs per call), 30s cache.
- **Execution:** Chainlink primary + Pyth fallback on EVM; Pyth primary on Solana; read on-chain at build time.
- **Historical:** CoinGecko `market_chart`, daily cron, append-only table, 24h cache.
- **Divergence:** oracle-vs-oracle 1.5% blocks; CoinGecko-vs-oracle 5% warns.

### Adapter Registry

New adapters are added via a config-driven manifest per `(port, chain)` pair naming the implementation and credentials. Adding a chain means adding manifest entries and implementing the interface. Auto-discovery is not used.

---

## Responsibility Boundary

| Not FKL's Job | Whose Job |
|---|---|
| Computing IL | Engine 1 |
| Calculating risk scores | Engine 2 |
| Ranking yield opportunities | Engine 3 |
| Deciding what to buy/sell | Engine 4 |
| Storing portfolio snapshots | Engine 1 |
| Executing transactions | Engine 6 → Router → Planner → Policy → CCE |
| Enforcing policy | Policy Engine |
| Allocating nonces | CCE |

---

## Resolved Items

1. **Aerodrome adapter build-out** — MVP venue, dedicated adapter work; not a generic EVM LP adapter.
2. **CoinGecko limits** — Demo 100 calls/min; production uses a paid plan; `/simple/price` supports 515 IDs. Provider contracts are in `docs/implementation_spec.md` §10.
3. **DeFiLlama terms** — free `api.llama.fi` has no SLA; production uses `pro-api.llama.fi/{KEY}`. Protocol safety is cached 24h.
4. **Oracle roles** — corrected by T4: Chainlink primary on EVM, Pyth primary on Solana, CoinGecko display only.
