# CrestFlow — Technical Reference Index

**Status:** Derived index — does not define algorithms, policy, or schemas
**Version:** 2.0
**Last reviewed:** 18 November 2026
**Supersedes:** v1.x master reference, whose independent restatements of policy, risk, TOPSIS, cache, oracle, and audit rules conflicted with the component documents.
**Canonical decisions:** `CANONICAL_DECISIONS.md`
**Shared types:** `docs/canonical_schemas.md` (authoritative)
**Document precedence:** `docs/DOCUMENT_INDEX.md`

---

## Purpose

This document is a navigational and orientation index for the CrestFlow technical stack. It summarises what exists and where the authoritative definition lives. It intentionally does **not** restate formulas, thresholds, enums, or policy checks; those restatements were the source of the v1.x contradictions.

Rules:

1. If this index and a canonical source disagree, the canonical source wins.
2. Component algorithms are owned by component documents.
3. Shared types and enums are owned by `docs/canonical_schemas.md`.
4. Policy semantics are owned by `docs/policy_engine.md`.
5. Topology is owned by `core_documents/system_architecture.md`.
6. Runtime contracts are owned by `docs/implementation_spec.md`.

---

## Domain Map

| # | Domain | Authoritative document | Notes |
|---|---|---|---|
| 1 | System architecture & core patterns | `core_documents/system_architecture.md` | Topology only; execution order fixed at D7 |
| 2 | Authentication, keys, wallets | `docs/user_onboarding.md`, `docs/implementation_spec.md` §7 | Google `sub` anchor; Turnkey Path A; manual Path B |
| 3 | Identity, KYC, compliance | `docs/identity_kyc.md` | Six statuses, three tiers, HyperVerge conditional |
| 4 | Financial data infrastructure | `docs/financial_knowledge_layer.md` | Single data path; T6 cache table; T4 oracles |
| 5 | Portfolio intelligence (Engine 1) | `docs/engine1_portfolio_intelligence.md` | Snapshot, decomposition, HHI, PnL |
| 6 | Risk intelligence (Engine 2) | `docs/engine2_risk_intelligence.md` | `riskScore`, bands, persona multipliers |
| 7 | Yield & opportunity (Engine 3) | `docs/engine3_yield_opportunity.md` | Five-criteria TOPSIS, 7-day TWAP |
| 8 | Strategy optimization (Engine 4) | `docs/engine4_strategy_optimization.md` | Drift, cooldown, economic gate |
| 9 | User intelligence & Copilot (Engine 5) | `docs/engine5_user_intelligence.md` | Persona, `GoalProfile`, Gemini 3.8 Flash |
| 10 | Execution intelligence (Engine 6) | `docs/engine6_execution_intelligence.md` | `ExecutionIntent` DAG |
| 11 | Policy engine | `docs/policy_engine.md` | Canonical 9 checks, no short-circuit |
| 12 | Orchestration, simulation, saga | `docs/orchestrator_planner.md` | Build → simulate → policy → sign |
| 13 | Execution routing | `docs/execution_router.md` | Allowlisted venues; CCTP V2 parameters |
| 14 | Protocol adapters & cross-chain | `docs/protocol_layer.md`, `docs/cross_chain_ecosystem.md` | MVP roster; CCTP V2; CCE nonce owner |
| 15 | State synchronization | `docs/portfolio_state_manager.md` | Webhooks, deltas, `PARTIAL` quality |
| 16 | Audit & x402 | `docs/audit_layer_x402.md` | Six categories; x402 V2 on Base USDC |
| 17 | Data standards & API | `docs/canonical_schemas.md`, `docs/implementation_spec.md` | Types, units, REST/OpenAPI 3.1 |

---

## Shared Type Catalogue

All shared types are defined once in `docs/canonical_schemas.md`. This table is an index only.

| Type | Canonical section | Critical invariant |
|---|---|---|
| `Chain`, `Persona`, `PoolType`, `ActionType` | §2–§3 | Shared enums; no component-local variants |
| `User`, `AddressScope`, `AutopilotConfig` | §3 | `googleSub` unique; Path B is `MANUAL`; no delegated signing |
| `GoalProfile` | §4 | `allowedActionTypes` required and non-empty |
| `PortfolioSnapshot`, `Position` | §5 | One `dataQuality`; HHI on three axes |
| `RiskSnapshot`, `RiskAlert` | §6 | `riskScore` only — never `compositeRiskScore` |
| `YieldOpportunitySnapshot` | §7 | Exactly five TOPSIS criteria; `twapApy7dPercent` |
| `StrategyPlan` | §8 | Abstract deltas; no calldata |
| `ExecutionIntent`, `ExecutionAction` | §9 | Integer base units; route-critical fields present |
| `SimulationPassedPackage` | §11 | Policy input; only a fully passed package is accepted |
| `PolicyDecision`, `PolicyCheckResult` | §11 | All 9 checks recorded |
| `AuditEntry` | §13 | Category + action; `eventId` idempotency |
| `PaymentRequirements`, `PaymentPayload`, `PaymentRecord` | §15 | x402 V2 settlement fields; `$0.01` = `10000` base units |

---

## Policy Catalogue (names only)

`docs/policy_engine.md` owns the predicates. The nine checks, in order:

1. KYC status
2. Tier spend limit
3. Wallet authority / execution mode
4. Risk threshold and freshness
5. Chain allowlist
6. Protocol allowlist
7. Per-transaction value cap
8. Slippage cap
9. Allowed action types

All nine run. There is no short-circuit. Cooldown and gas/bridge payback are Engine 4 economic gates, **not** policy checks.

---

## Execution Order (D7)

```
Engine 6 → Execution Router → Orchestrator/Planner build → Simulation
         → Policy Engine → CCE → Portfolio State Manager
```

- Policy consumes `SimulationPassedPackage`.
- Policy runs on every execution, including autopilot.
- CCE owns per-chain nonce allocation, signing, broadcast, confirmation, and replacement.
- PSM reconciles state after confirmation.

---

## Stack (T15)

| Layer | Choice |
|---|---|
| Language / framework | TypeScript + NestJS |
| Database | PostgreSQL |
| Cache / queue | Redis + BullMQ |
| API | REST, OpenAPI 3.1 |
| Key management | Turnkey TEE |
| Environments | ETH/Base/Arb Sepolia + Solana Devnet |
| LLM | Gemini 3.8 Flash primary, Gemini Flash-Lite fallback (D8) |

---

## Superseded Statements

The following v1.x statements are removed and must not be reintroduced:

| Removed claim | Replacement |
|---|---|
| `riskSnapshot.compositeRiskScore` | `RiskSnapshot.riskScore` |
| Six-criteria TOPSIS including payback/gas | Five criteria; economic gate in Engine 4 |
| `twapApy30dPercent` | `twapApy7dPercent` |
| Fixed 0.5% / 1.0% slippage tiers | Stable 0.25%, correlated 0.5%, volatile 1.0%; persona may lower only; hard cap 1.0% |
| Policy checks for cooldown and payback | Engine 4 economic gates |
| `X-Payment` / `X-Payment-Authorization` | `PAYMENT-REQUIRED` / `PAYMENT-SIGNATURE` / `PAYMENT-RESPONSE` |
| x402 transaction-hash verification | EIP-3009 `transferWithAuthorization` via the `exact` scheme |
| CCTP V1 `depositForBurn` without V2 parameters | CCTP V2 with `destinationCaller`, `maxFee`, `minFinalityThreshold` |
| ERC-4337 session keys for MetaMask EOAs | Path B manual signing (D1) |
| SPL `delegate` for lending/LP execution | Path B manual signing (D1) |
| 11 flat audit event types | Six categories with category-scoped actions (T10) |
| Automated GMX routing/waterfall/policy branches | GMX read-only/manual (D2) |
| Non-USDC bridge aggregator | Out of MVP; `UNSUPPORTED_BRIDGE` (D3) |
| Fiat on-ramp in MVP | Out of MVP (T16) |
| Prices 60 s / historical indefinite / metadata 1 h | T6 cache table |
| CoinGecko 5% divergence as a signing gate | T4: on-chain oracle cross-check gates signing; CoinGecko is display-only |
| Engine 2 polling Aave/Kamino/MarginFi directly | FKL is the only external data path |
| Policy before build/simulation | D7 order |

---

## Change Log

| Version | Date | Change |
|---|---|---|
| 1.x | 2026 | Original master reference; independently restated algorithms and policy |
| 2.0 | 18 Nov 2026 | Converted to derived index; conflicts removed; canonical sources cited |
