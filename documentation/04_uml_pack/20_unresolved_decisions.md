# 20 — Open Questions, Flags & Unresolved Decisions

**Pack:** CrestFlow UML & Architecture Documentation Pack  
**Version:** 1.0 | **Source docs:** All CrestFlow planning documents as of CANONICAL_DECISIONS.md v1.2 and all v2.0 component docs  
**Purpose:** Consolidated registry of all unresolved decisions, contradictions, assumptions, and Phase 2 deferrals identified during the production of this documentation pack.

> [!IMPORTANT]
> Review this file before finalising any diagram in Lucidchart. Flags marked **Decision Pending** require a human decision before the affected diagram element can be treated as authoritative.

---

## 1. Unresolved Architectural Decisions

| Flag ID | Priority | Decision Area | Status | Detail | Affects |
|---|---|---|---|---|---|
| **F-D01** | 🔴 HIGH | KYC Vendor (D9) | **Decision Pending** | HyperVerge labeled "selected pending integration validation". Two-week spike required before build commitment. Fallback providers: Persona or Sumsub. Webhook payload format, session creation API, and HMAC signing mechanism differ between vendors. | Files 03, 09, 15 — participant names and flow steps |
| **F-D02** | 🔴 HIGH | Mainnet Activation | **Not triggered** | Mainnet release gate is a separate decision not yet triggered in planning docs. All testnet gates must pass + security audit before gate opens. No actual mainnet go-live date documented. | File 11 (deployment), File 18 (Gantt) |
| **F-D03** | 🟡 MEDIUM | CCTP Fast Mode Flag | **Conditionally enabled** | `CCTP_FAST_ENABLED` feature flag may be `false` in production. Fast mode is documented and designed but operational enablement is a deployment decision. | File 06 — Fast mode branch |
| **F-D04** | 🟡 MEDIUM | KYC Expiry Configuration | **Jurisdiction-specific** | MVP KYC approvals do not auto-expire by default. Auto-expiry timing is jurisdiction-specific and not yet configured. `kyc_records.expires_at` defaults to NULL. | File 09, File 17 — `expires_at` column |
| **F-D05** | 🟡 MEDIUM | TIER_2 / TIER_3 KYC Upgrade | **Not documented** | TIER_2 (\$10k/day) and TIER_3 (unlimited) upgrade path is not documented. Manual admin operation is assumed. No endpoint or workflow specified. | File 09 (KYC lifecycle) |
| **F-D06** | 🟡 MEDIUM | Protocol Safety Score Threshold | **Not specified** | The threshold value triggering a `LOW_PROTOCOL_SAFETY` alert (Policy Check 6, Engine 2) is a configuration value not documented in planning files. | Files 04, 09 — risk alert generation |
| **F-D07** | 🟡 MEDIUM | SPLIT_THRESHOLD (Order Routing) | **Not specified** | Per-chain threshold for split order routing in the Execution Router is noted as per-chain but specific numeric values are not documented. Single-route diagrams are shown for simplicity. | File 04, File 05 |
| **F-D08** | 🟢 LOW | Solana Durable Nonces | **Optional** | Solana durable nonces are optional only for scheduled transfers; not the default. Standard Solana blockhash-based nonces are the default for all MVP executions. | File 05, File 11 |
| **F-D09** | 🟢 LOW | Counterparty AML Screening | **Phase 2** | Real-time counterparty AML screening at execution time is deferred to Phase 2. MVP only checks KYC status of the initiating user. | File 04 — policy check scope |
| **F-D10** | 🟢 LOW | Anonymous Agent Free Tier | **Not specified** | For unauthenticated API agents (userId=null), the grouping key for the free tier counter is not specified. IP-based or API-key-based grouping is assumed. | File 07 |

---

## 2. Phase 2 Items (Explicitly Out of MVP)

The following are documented as Phase 2 or post-MVP in the planning files. They must NOT appear in MVP diagrams.

| Feature | Phase | Source | Diagram Impact |
|---|---|---|---|
| Black-Litterman portfolio optimizer | Phase 2 | engine4_strategy_optimization.md | Requires 90+ portfolio snapshots as prerequisite |
| RWA (Real World Asset) adapters | Phase 2 | cross_chain_ecosystem.md | No RWA protocol in MVP venue list |
| Permit2 / 1inch / UniswapX routing | Phase 2 | execution_router.md | MVP uses Uniswap V3 direct only |
| Curve Finance adapter | Phase 2 | protocol_layer.md | Not in MVP DEX list |
| Camelot DEX adapter (Arbitrum) | Phase 2 | protocol_layer.md | Not in MVP DEX list |
| Non-USDC bridge (LI.FI / Socket) | Phase 2 | cross_chain_ecosystem.md | CCTP V2 USDC-only is MVP |
| Travel Rule compliance | Phase 3 | identity_kyc.md | Deferred |
| Fiat on-ramp | Out of MVP | CANONICAL_DECISIONS.md T16 | No fiat entity in diagrams |
| GMX automated execution | Out of MVP (D2) | CANONICAL_DECISIONS.md D2 | GMX = read-only display only |
| HDBSCAN persona clustering | Phase 2 | engine5_user_intelligence.md | Static persona assignment in MVP |
| Pre-funded credit balances | Phase 2 | audit_layer_x402.md | x402 per-query only in MVP |
| Agent credit ledger | Phase 2 | audit_layer_x402.md | No credit ledger tables in MVP ER diagram |
| ERC-4337 account abstraction | Phase 2 (D1) | CANONICAL_DECISIONS.md D1 | Path B = READ_ONLY + MANUAL only |
| EIP-7702 delegation | Phase 2 | cross_chain_ecosystem.md | Not in MVP |
| Session keys (EVM) | Phase 2 (D1) | CANONICAL_DECISIONS.md D1 | Excluded |
| SPL delegate (Solana) | Phase 2 (D1) | CANONICAL_DECISIONS.md D1 | Excluded |
| Turnkey key-export (self-custody exit) | Post-MVP | identity_kyc.md | No REVOKED→export state in delegation lifecycle |
| GMX keeper tracking / AWAITING_FULFILLMENT | Phase 2 (D2) | CANONICAL_DECISIONS.md D2 | State is reserved but unreachable in MVP |
| Kafka event bus | Phase 2 | implementation_spec.md | BullMQ/Redis only in MVP |
| Cross-chain LP positions | Phase 2 | cross_chain_ecosystem.md | MVP: single-chain LP only |
| Arbitrum MEV relay | Research item | protocol_layer.md | Not implemented in MVP |
| Base MEV relay | Research item | protocol_layer.md | Not implemented in MVP (Flashbots model doesn't directly apply) |

---

## 3. Assumptions Made During Pack Production

The following assumptions were made where planning docs did not provide explicit detail. Each should be validated.

| Assumption ID | Assumption | Basis | Affects |
|---|---|---|---|
| A-01 | Refresh token is stored as bcrypt hash in `auth_sessions`. Exact algorithm not specified — "hashed" is confirmed, bcrypt assumed. | General best practice | File 03, File 17 |
| A-02 | Outbox polling interval for `PortfolioScanTriggered` uses BullMQ cron cadence (not specified in docs). | BullMQ architecture | File 03 |
| A-03 | Redis is NOT consulted for policy inputs (risk snapshots, goal profiles). Policy Engine fetches fresh from PostgreSQL. | Correctness requirement | File 04 |
| A-04 | `adjustedThreshold` = base risk threshold × persona multiplier. The base threshold default numeric value is not specified. | Persona multiplier table (T13) is specified | File 04 |
| A-05 | `delegatedAt` is reset on each successful CCE signing call (keeps delegation fresh without explicit user action). | TEE integration behavior | File 10 |
| A-06 | Admin endpoints follow `/v1/admin/users/:id/...` pattern. Not specified in planning docs — consistent with NestJS REST conventions. | Convention | Files 10, 15 |
| A-07 | Free tier counter computed by `COUNT(payments WHERE status=SETTLED AND created_at > NOW()-24h)`. Not a separate counter column. | Anti-counter-drift design | File 07 |
| A-08 | For CCTP: attestation polling is CCE-driven (CCE polls Iris, not webhook-driven). No Iris webhook endpoint documented. | Iris API documentation (D5) | File 06 |
| A-09 | Compensation for multi-step executions attempts step reversal in reverse order (LIFO). Specific reversal strategy not documented. | General saga pattern | File 05, File 08 |
| A-10 | `portfolio_snapshots` created after every execution reconciliation (not only on scheduled scans). | PSM design | File 05, File 17 |
| A-11 | Gantt chart durations are estimates only — no actual project dates are documented in planning files. | Explicit gap in docs | File 18 |
| A-12 | Lucidchart is the target diagramming tool (confirmed by user brief). Mermaid/PlantUML are supplementary. | User request | All files |

---

## 4. Naming Convention Critical Reference

The following naming conventions are **explicitly mandated** in the planning documents and must be enforced in all diagrams:

| ✅ Correct | ❌ Incorrect | Source |
|---|---|---|
| `riskScore` | `compositeRiskScore` | canonical_schemas.md |
| `twapApy7dPercent` | `twapApy30dPercent` | canonical_schemas.md |
| `amountIn: BaseUnits` (integer string) | `amountIn: 1.5` (decimal) | canonical_schemas.md |
| `LEND_SUPPLY` | `LEND_DEPOSIT` | canonical_schemas.md |
| `LEND_WITHDRAW` | `LEND_REDEEM` | canonical_schemas.md |
| `KYC_RESUBMISSION_REQUESTED` | `KYC_RESUBMIT` | implementation_spec.md T10 |
| `execution_mode: AUTOPILOT` | `mode: auto` | canonical_schemas.md |
| `allowedActionTypes` (required on GoalProfile) | `actionTypes` | canonical_schemas.md |
| `destinationCaller = 0x0` | `destinationCaller = null` | CANONICAL_DECISIONS.md D5 |
| `minFinalityThreshold` | `minFinality` | CANONICAL_DECISIONS.md D5 |

---

## 5. External Service Dependency Flags

| Service | Flag | Detail | Production Mitigation |
|---|---|---|---|
| DeFiLlama | 🟡 SLA risk | Free tier has no SLA. API may be rate-limited or unavailable without warning. | Production requires paid plan |
| CoinGecko | 🟡 Rate limit | Demo tier: 100 calls/min. Insufficient for production load. | Production requires paid plan |
| HyperVerge | 🔴 Conditional | D9 — vendor not yet confirmed. | Spike result determines vendor |
| Circle Iris API | 🟢 Stable | CCTP V2 is production-grade; Iris API is documented and stable. | Standard retry with backoff |
| Turnkey TEE | 🟢 Stable | Used in production by multiple projects; documented API. | Standard error handling |
| x402.org Facilitator | 🟡 Emerging | x402 protocol is relatively new. Facilitator uptime and SLA not publicly guaranteed at time of writing. | Fallback: service degrade (return 503 on payment failure) |
| Gemini LLM API | 🟢 Stable | google/gemini-3.8-flash primary; google/gemini-flash-lite-latest fallback. | Automatic model fallback (D8) |

---

## 6. Contradictions Identified

No direct factual contradictions were found between planning documents during pack production. However, the following **potential inconsistencies** are flagged for clarification:

| Contradiction ID | Description | Documents Involved | Resolution Needed |
|---|---|---|---|
| C-01 | `risk_snapshots.is_stale` boolean column vs freshness check at runtime (14,400s since `computed_at`). It is unclear whether `is_stale` is set eagerly (by a background job) or computed lazily (at policy evaluation time). | implementation_spec.md §3.1, policy_engine.md | Clarify update timing of `is_stale` |
| C-02 | `users.kyc_status` (denormalized column) vs `kyc_records.status` (source of truth). If a webhook updates `kyc_records.status` before `users.kyc_status`, there is a brief window of inconsistency. | implementation_spec.md §3.1 | Confirm atomic update pattern (transaction wrapping both UPDATE statements) |
| C-03 | Delegation freshness: policy engine checks `NOW() - delegatedAt ≤ 14,400s` for AUTOPILOT. But `delegatedAt` is assumed to be reset on each CCE signing call (A-05). If not reset, all AUTOPILOT plans would become stale within 4h of initial provisioning even during active use. | policy_engine.md, implementation_spec.md | Confirm `delegatedAt` reset mechanism |

---

## 7. Pack Completeness Status

| File | Status | Notes |
|---|---|---|
| `00_INDEX.md` | ✅ Complete | Master index |
| `01_component_diagram.md` | ✅ Complete | 47 elements, 40 dependencies |
| `02_class_diagram.md` | ✅ Complete | 30+ classes, 36 relationships |
| `03_seq_onboarding_kyc.md` | ✅ Complete | 5 flows, 41 steps |
| `04_seq_policy_evaluation.md` | ✅ Complete | 9 checks, APPROVED + BLOCKED paths |
| `05_seq_transaction_execution.md` | ✅ Complete | Full D7 pipeline, all stages |
| `06_seq_cctp_bridge.md` | ✅ Complete | Standard + Fast, Path A + B |
| `07_seq_x402_payment.md` | ✅ Complete | Free tier + paid path + replay protection |
| `08_state_saga_lifecycle.md` | ✅ Complete | 18 states, compensation paths |
| `09_state_kyc_lifecycle.md` | ✅ Complete | 6 KycStatus states |
| `10_state_delegation_lifecycle.md` | ✅ Complete | 5 delegation states |
| `11_deployment_diagram.md` | ✅ Complete | All nodes, trust boundaries |
| `12_BE_system_architecture.md` | ✅ Complete | 5-layer view, tech stack, D1–D10 |
| `13_BE_dfd_level0.md` | ✅ Complete | Context diagram, 36 data flows |
| `14_BE_dfd_level1.md` | ✅ Complete | 8 subsystems, 43 data flows |
| `15_BE_use_case.md` | ✅ Complete | 40 use cases, 9 actors, 14 relationships |
| `16_BE_activity_diagram.md` | ✅ Complete | 5 swimlanes, fork/join, compensation |
| `17_BE_er_diagram.md` | ✅ Complete | 29 tables, all columns + types + FKs |
| `18_project_gantt.md` | ✅ Complete | 6 phases, all tasks, release gates |
| `19_project_sdlc.md` | ✅ Complete | Iterative Incremental model |
| `20_unresolved_decisions.md` | ✅ Complete | This file |

---

## 8. Recommended Review Order

Before finalizing diagrams in Lucidchart, review and resolve in this order:

1. **F-D01 (KYC Vendor)** — resolve D9 spike; update Files 03, 09, 15 participant names if vendor changes.
2. **C-01, C-02, C-03** — verify with implementation team; update affected DB and sequence diagrams.
3. **F-D05 (KYC Tier Upgrade)** — document the admin upgrade endpoint; add to File 15 (Use Cases) and File 09 (KYC lifecycle).
4. **F-D06 (Protocol Safety Threshold)** — configure the numeric threshold; add to File 04 (policy_check_results).
5. **F-D03 (CCTP Fast Flag)** — confirm production value; add conditional note to File 06.
6. **A-05 (delegatedAt reset)** — verify with Turnkey integration code; resolve C-03 if confirmed.
7. **F-D02 (Mainnet dates)** — insert actual dates into File 18 (Gantt) once project schedule is established.
