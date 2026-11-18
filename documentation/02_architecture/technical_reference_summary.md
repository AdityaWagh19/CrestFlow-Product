# CrestFlow — Technical Reference (Core Summary)

**Status:** Derived index — orientation only
**Version:** 2.0
**Last reviewed:** 18 November 2026
**Supersedes:** v1.x condensed reference, which restated policy, slippage, KYC, oracle, and compliance rules independently of the component documents.
**Canonical decisions:** `CANONICAL_DECISIONS.md`
**Full index:** `docs/technical_reference.md`
**Shared types:** `docs/canonical_schemas.md`

---

## How This Document Is Used

This is a one-page orientation map for reviewers. It names each subsystem and points to the authoritative source. It does not define thresholds, enums, formulas, or policy predicates; where a detail matters, follow the link.

If any statement here conflicts with a canonical document, the canonical document wins.

---

## Subsystem Map

| Layer | Components | Authoritative source |
|---|---|---|
| User | Google identity, KYC, Path A embedded wallet, Path B external wallet | `docs/user_onboarding.md`, `docs/identity_kyc.md` |
| Intelligence | Engines 1–6 | `docs/engine1_portfolio_intelligence.md` … `engine6_execution_intelligence.md` |
| Execution | Execution Router, Orchestrator/Planner, Simulation, Policy Engine, CCE | `docs/execution_router.md`, `docs/orchestrator_planner.md`, `docs/policy_engine.md`, `docs/cross_chain_ecosystem.md` |
| Data | FKL, PSM, Audit Layer, x402 Gateway | `docs/financial_knowledge_layer.md`, `docs/portfolio_state_manager.md`, `docs/audit_layer_x402.md` |
| Protocols | Aave V3, Uniswap V3, Aerodrome, Moonwell, Jupiter, Raydium, Kamino, MarginFi; GMX read-only | `docs/protocol_layer.md` |
| Chains | Ethereum, Base, Arbitrum, Solana (testnets in MVP) | `core_documents/system_architecture.md` |

---

## Canonical Invariants

| Topic | Canonical position | Owner |
|---|---|---|
| Execution order | Engine 6 → Router → build → simulate → Policy → CCE → PSM | D7 |
| Policy catalogue | Nine checks, all run, no short-circuit | `docs/policy_engine.md` |
| Risk field | `RiskSnapshot.riskScore` (0–100) | `docs/canonical_schemas.md` §6 |
| Risk bands | 0–39 / 40–59 / 60–79 / 80–100 | T13 |
| Persona multipliers | 0.5 / 0.7 / 1.0 / 1.3 / 1.15 | T13 |
| TOPSIS | Exactly five criteria | T2 |
| APY field | `twapApy7dPercent` | T2 |
| Slippage | Stable 0.25%, correlated 0.5%, volatile 1.0%; persona may lower only | T3 |
| HHI | `Σ(share²) × 10,000`; block new placements above 2,500 | T1 |
| Drift tolerance | 3% CP/CONS, 5% BALANCED, 7% GROWTH/YS; Engine 4 owns | T5 |
| Cooldown | 7 days, Engine 4; HIGH urgency bypasses | T5 |
| Cache TTLs | Prices 30 s, positions 30 s, pools 300 s, safety 24 h, historical 24 h, metadata 7 d | T6 |
| Oracles | Chainlink + Pyth on EVM; Pyth on Solana; 60 s staleness; 1.5% cross-check | T4 |
| KYC statuses | PENDING, APPROVED, REJECTED, RESUBMISSION_REQUESTED, SUSPENDED, EXPIRED | T8 |
| KYC tiers | $1,000 / $10,000 / unlimited | T8 |
| Audit taxonomy | Categories AUTH, KYC, EXECUTION, PREFERENCE, PAYMENT, SYSTEM + scoped actions | T10 |
| Bridge | USDC via CCTP V2 only; non-USDC `UNSUPPORTED_BRIDGE` | D3, D5 |
| GMX | Read-only/manual; no automated branch | D2 |
| Path B | Manual signing only; no ERC-4337 or SPL delegate | D1 |
| x402 | Official facilitator, Base USDC, CAIP-2, EIP-3009 exact, $0.01/query + 10 free/day | D4, T17 |
| Stack | TypeScript/NestJS, Postgres, Redis, BullMQ, REST/OpenAPI 3.1, Turnkey TEE | T15 |

---

## What Changed From v1.x

| v1.x claim | v2.0 position |
|---|---|
| Policy includes cooldown, gas/bridge payback, persona volatility | Those are Engine 4/Engine 2 responsibilities; Policy has nine checks |
| Hard 0.5% slippage cap | T3 tiered caps with persona-lowering only |
| FATF Travel Rule implemented | Phase 3 (T8); not claimed as implemented |
| Permit2 replaces `approve()` | Permit2 is Phase 2; Aave uses EIP-2612 where supported |
| ERC-4337 / SPL delegate for Path B | Removed; Path B is manual (D1) |
| 11 flat audit event types | Six categories + scoped actions (T10) |
| Chainlink/Pyth 1.5% is the only oracle guard | T4 separates staleness, cross-check, and display-only CoinGecko |
| `compositeRiskScore` | `riskScore` |
| Six TOPSIS criteria | Five criteria |
| `twapApy30dPercent` | `twapApy7dPercent` |
| GMX automated LP management | Read-only/manual |

---

## Reading Order

1. `CANONICAL_DECISIONS.md`
2. `docs/DOCUMENT_INDEX.md`
3. `docs/canonical_schemas.md`
4. `docs/policy_engine.md`
5. `docs/implementation_spec.md`
6. Component documents in engine order
7. `core_documents/system_architecture.md`
