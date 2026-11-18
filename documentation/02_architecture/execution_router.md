# Execution Router

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D7, D5, T12  
**Canonical schemas:** `docs/canonical_schemas.md` §10  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

Micro-execution optimisation. Takes a DAG-ordered `ExecutionIntent` from Engine 6 and produces `ExecutionInstructions`: route-resolved, pool-resolved, bridge-parameterised instructions that the Planner can turn into unsigned transaction bytes.

The Router **routes**, but it does **not** build bytes, simulate, sign, or check policy.

- Engine 6: macro routing (which chain, which protocol, which action sequence).
- Execution Router: micro routing (which pool, which split, which venue, which CCTP parameters).
- Planner: builds and simulates.
- Policy: approves or blocks.

**Ordering:** the Router runs before the Planner. Bytes cannot be built until the route, pool, calldata, approvals, and CCTP parameters are known.

---

## Job 1 — Swap Pathfinding Within the Allowlisted Venues

For every `SWAP` action, select the best route among the **MVP venues only**:

| Chain | Venues |
|---|---|
| Ethereum | Uniswap V3 |
| Base | Uniswap V3, Aerodrome |
| Arbitrum | Uniswap V3 |
| Solana | Jupiter Swap API (`/order` + `/build`), Raydium |

There is no 1inch or UniswapX integration in MVP. UniswapX is an intent auction, not an aggregator, and is Phase 2. Any route leg outside the allowlist is discarded.

```typescript
const routes = await venue.quote(fromToken, toToken, amount, chain)
const allowedRoutes = routes.filter(
  r => r.legs.every(leg => PROTOCOL_ALLOWLIST[chain].includes(leg.protocol))
)
const bestRoute = selectBest(allowedRoutes)
```

If no allowlisted route exists, return `NO_ROUTE_WITHIN_ALLOWLIST`. Never silently route through a non-allowlisted venue.

**Always re-quote at route-selection time.** Simulation confirms technical feasibility; it does not lock a price. Quotes are valid for seconds to a few blocks. The Router does not reuse the Planner's simulated price.

---

## Job 2 — Split Order Routing

For swaps above the size threshold, split across multiple allowlisted pools:

```typescript
if (swapValueUsd > SPLIT_THRESHOLD) {
  const splitRoutes = venue.quoteWithSplits(fromToken, toToken, amount, chain)
  const allowed = splitRoutes.filter(r => r.legs.every(isAllowlisted))
  const best = selectBestSplit(allowed, gasEstimate)
  if (best.netOutput > singleRoute.netOutput) use(best)
}
```

`SPLIT_THRESHOLD` is per chain. Each split leg must pass the allowlist. On Solana, Jupiter's split routing is used within the same allowlist.

---

## Job 3 — Protocol Selection for Non-Swap Actions

For `LEND_SUPPLY` / `LEND_WITHDRAW`: compare APY, utilisation, and TVL across allowlisted lending markets for the same asset on the target chain.

| Chain | Lending venues |
|---|---|
| Ethereum | Aave V3 |
| Base | Aave V3, Moonwell |
| Arbitrum | Aave V3 |
| Solana | Kamino, MarginFi |

For `LP_ADD` / `LP_REMOVE`, branch by `poolType`:

```typescript
switch (action.poolType) {
  case 'constant_product': return selectByFeeAndVolume(candidates) // Uniswap V3, Raydium
  case 'stable_swap':      return selectByPegDepth(candidates)      // Aerodrome stable pools
  case 'correlated':       return selectByPegDepth(candidates)
  case 'gmx_gm_pool':      return reject('UNSUPPORTED_VENUE')       // manual-only
}
```

GMX is **not** selectable. If a `gmx_gm_pool` action arrives, the Router returns `UNSUPPORTED_VENUE` and a manual-review item. The shared `@crestflow/pool-type` package owns the classification.

---

## Job 4 — Gas Optimisation via Multicall Batching (EVM)

Where a protocol supports atomic multicall execution, batch sequential same-chain actions:

```typescript
// Before: APPROVE(USDC → Aave), LEND_SUPPLY(USDC, Aave)
// After:  multicall([APPROVE(...), LEND_SUPPLY(...)])
```

Chain-specific cost:

| Chain | Total gas cost |
|---|---|
| Ethereum | L1 execution gas |
| Base | L2 execution gas + `GasPriceOracle.getL1Fee(bytes)` + `getOperatorFee(gasUsed)` |
| Arbitrum | L2 execution gas + `NodeInterface.gasEstimateL1Component(to, contractCreation, data)` |
| Solana | Native multi-instruction transactions; no batching job |

Batching-benefit calculations use the chain-specific total. On L2s, L1 data cost can dominate; a multicall that saves L2 execution gas may not save on the L1 component. Detailed formulas are in `docs/implementation_spec.md` §5.

---

## Job 5 — CCTP V2 Path Selection

For every `BRIDGE` action, select CCTP V2 parameters from the plan's urgency:

```typescript
const speed = (plan.urgency === 'HIGH' && CCTP_FAST_ENABLED) ? 'FAST' : 'STANDARD'

const params = {
  mechanism: 'CCTP_V2',
  sourceDomain: CCTP_DOMAIN_IDS[sourceChain],
  destinationDomain: CCTP_DOMAIN_IDS[destinationChain],
  mintRecipient: resolveRecipient(action),      // Path A embedded; Path B external
  destinationCaller: '0x0',                     // permissionless mint in MVP
  burnToken: USDC[sourceChain],
  amount: action.amountIn,
  maxFee: speed === 'FAST' ? await quoteFastFee(...) : 0n,
  minFinalityThreshold: speed === 'FAST' ? 1000 : 2000,
  speed
}
```

**CCTP domain IDs:**

| Chain | Domain |
|---|---:|
| Ethereum | 0 |
| Arbitrum | 3 |
| Base | 6 |
| Solana | 5 |

**Fee cap (Fast only):** 0.10% of the transfer or $5, whichever is lower. If the quote exceeds the cap, fall back to Standard and notify the user. Standard has no protocol fee. Fast fees are route-specific and deducted at mint; the expected received amount is `amount - maxFee`.

Non-USDC bridging is out of MVP: return `UNSUPPORTED_BRIDGE`.

---

## MEV Protection

For swaps above the split threshold:

- **Ethereum:** private-mempool relay (Flashbots Protect or equivalent).
- **Arbitrum:** evaluate chain-specific private-RPC options; the sequencer model changes but does not eliminate MEV.
- **Base:** evaluate separately; do not assume Ethereum's Flashbots integration applies.
- **Solana:** Jito bundles, not Flashbots.

MEV protection is a routing option, not a policy decision. It must not change the allowlisted venue or the user's minimum output.

---

## `ExecutionInstructions` Schema

The canonical schema is in `docs/canonical_schemas.md` §10. Each `ResolvedAction` carries:

- `actionId`, `type`, `chain`, `protocol`;
- `poolAddress` / `tokenId`;
- `routerAddress`, `calldata`, `value`, `gasLimit`;
- `maxFeePerGas` / `maxPriorityFeePerGas` where applicable;
- `deadline`;
- `approval` (mechanism, spender, amount, calldata, signature requirement);
- `bridge` (CCTP V2 parameters);
- `routeLegs` for split routes.

The Planner must be able to build and simulate from this object without re-deriving venue, pool, or calldata.

---

## Resolved Items

1. **Curve** — Phase 2; not in the MVP allowlist. The venue decision is T12, not an open router question.
2. **1inch/UniswapX/GMX routing** — out of MVP. 1inch and UniswapX are Phase 2; GMX is manual-only. The Router never sees them in the automated path.
3. **Shared `poolType` library** — T11: one `@crestflow/pool-type` package.
4. **MEV relays for Base/Arbitrum** — implementation research; the default is a private RPC per chain, with a public fallback only if the private relay is unavailable.
