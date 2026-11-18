# 18 — Project Planning: Gantt Chart

**Pack:** CrestFlow UML & Architecture Documentation Pack  
**Version:** 1.0 | **Source docs:** `DOCUMENT_INDEX.md`, `implementation_spec.md` v1.0, `CANONICAL_DECISIONS.md` v1.2  
**Primary deliverable:** Lucidchart-ready Gantt chart specification. Mermaid gantt is supplementary.

> [!WARNING]
> **Actual project dates are NOT documented in the planning files.** The timeline below is a **proposed structure** derived from module dependencies and documented release gates. Use as a template — insert actual dates from your project schedule. All durations are estimates and must be validated against your team capacity.

---

## 1. Purpose & Scope

Specifies the **Gantt Chart** for CrestFlow project planning. Presents phases, milestones, and task dependencies structured around:
- The documented module dependency order (engines, execution layer, protocol layer)
- The documented release gate criteria (testnet before mainnet)
- The documented Phase 2 / out-of-MVP boundaries

---

## 2. Release Gate Milestones (from planning docs)

| Milestone | Criteria (from CANONICAL_DECISIONS.md) | Gate Type |
|---|---|---|
| **Testnet Release Gate** | ≥1 successful swap on Sepolia/Base Sepolia; ≥1 successful Aave V3 lend/withdraw; ≥1 successful CCTP transfer; ≥1 successful x402 payment | Testnet complete → eligible for mainnet |
| **Mainnet Release Gate** | Separate release decision (not yet triggered); security audit passing | Mainnet launch |

---

## 3. Project Phases & Tasks

### Phase 1: Requirements & Planning
**Estimated duration: 2–3 weeks**

| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T1.1 | Stakeholder Requirements Gathering | — | Requirements document |
| T1.2 | Technology Stack Finalisation | T1.1 | Stack decision (D10): TypeScript/NestJS/PostgreSQL 16/Redis |
| T1.3 | Canonical Decisions Documentation | T1.2 | CANONICAL_DECISIONS.md (D1–D10, T1–T17) |
| T1.4 | Third-Party Integration Research | T1.1 | Turnkey, HyperVerge (D9 spike), CCTP V2, x402 evaluated |
| T1.5 | HyperVerge KYC Integration Spike | T1.4 | D9 decision: proceed with HyperVerge or pivot |
| T1.6 | MVP Scope Definition | T1.3 | MVP scope; Phase 2 backlog established |

---

### Phase 2: System Design
**Estimated duration: 3–4 weeks**

| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T2.1 | System Architecture Design (5-layer) | T1.3 | system_architecture.md v2.0 |
| T2.2 | Canonical Schema Design | T2.1 | canonical_schemas.md v2.0 |
| T2.3 | Database Schema Design (29 tables) | T2.2 | implementation_spec.md §3.1 |
| T2.4 | Engine Specifications (E1–E6) | T2.1 | engine1–engine6 docs v2.0 |
| T2.5 | Execution Pipeline Specification | T2.1 | orchestrator_planner.md, execution_router.md, protocol_layer.md |
| T2.6 | Policy Engine Specification | T2.5 | policy_engine.md v2.0 (9 checks defined) |
| T2.7 | Identity & KYC Specification | T2.1 | identity_kyc.md, user_onboarding.md v2.0 |
| T2.8 | Cross-Chain & x402 Specification | T2.1 | cross_chain_ecosystem.md, audit_layer_x402.md v2.0 |
| T2.9 | UML & Architecture Documentation Pack | T2.1–T2.8 | This pack (21 files) |
| T2.10 | API Design (OpenAPI 3.1) | T2.3 | OpenAPI specification |

---

### Phase 3: Backend Development
**Estimated duration: 8–12 weeks**  
*(Parallelised across sub-teams where possible)*

#### Sub-Phase 3A: Foundation (Weeks 1–3)
| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T3A.1 | NestJS Project Setup + CI/CD | T2.1 | Monorepo structure, GitHub Actions |
| T3A.2 | PostgreSQL 16 Schema Migrations | T2.3 | All 29 tables migrated |
| T3A.3 | Redis Setup + BullMQ Configuration | T3A.1 | Queue infrastructure |
| T3A.4 | Identity Module: Google OAuth + JWT | T3A.2 | UC-01, UC-02, UC-03 working |
| T3A.5 | Turnkey TEE Integration (Path A) | T3A.4 | UC-04: embedded wallet provisioning |
| T3A.6 | External Wallet Connection (Path B) | T3A.4 | UC-05, UC-06 |
| T3A.7 | HyperVerge KYC Integration | T3A.5, T1.5 | UC-10, UC-11, UC-12 (conditional) |
| T3A.8 | Audit Layer Implementation | T3A.2 | audit_entries partitioned table + service |
| T3A.9 | `@crestflow/pool-type` Shared Library | T2.2 | NPM package (T11); imported by E1–E4, E6, ER, PL, PE |

#### Sub-Phase 3B: Intelligence Engines (Weeks 2–6)
| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T3B.1 | Financial Knowledge Layer | T3A.9 | Protocol definitions, pool type registry |
| T3B.2 | Engine 1: Portfolio Intelligence | T3A.2, T3B.1 | Portfolio scanning, position tracking |
| T3B.3 | Engine 2: Risk Intelligence | T3B.2, T3A.9 | Risk scoring, HHI bands, risk alerts |
| T3B.4 | Engine 3: Yield Opportunity | T3B.1, T3A.9 | TOPSIS ranking, yield_opportunities |
| T3B.5 | Engine 4: Strategy Optimization | T3B.3, T3B.4 | Strategy plan generation |
| T3B.6 | Engine 5: User Intelligence (Copilot) | T3B.2, T3A.4 | Natural language query; Gemini LLM integration |
| T3B.7 | Engine 6: Execution Intelligence | T3B.3, T3B.4, T3B.5 | Drift detection, execution trigger |

#### Sub-Phase 3C: Execution Pipeline (Weeks 4–9)
| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T3C.1 | Execution Router | T3B.7, T3A.9 | Route selection, venue determination |
| T3C.2 | Orchestrator/Planner | T3C.1 | Instruction building, 30s re-quote TTL |
| T3C.3 | Simulation Engine | T3C.2 | eth_call / simulateTransaction dry-run |
| T3C.4 | Policy Engine (9 checks) | T3C.3, T3A.5–T3A.7 | All 9 checks; policy_decisions table |
| T3C.5 | CCE: Nonce Management | T3C.4 | Nonce lease/allocate/confirm |
| T3C.6 | CCE: Turnkey Signing (AUTOPILOT) | T3C.5, T3A.5 | AUTOPILOT transaction signing |
| T3C.7 | CCE: Broadcast (EVM) | T3C.5–T3C.6 | Alchemy RPC integration, EIP-1559 gas |
| T3C.8 | CCE: Broadcast (Solana) | T3C.5–T3C.6 | Helius RPC integration, compute units |
| T3C.9 | Webhook Consumer (Alchemy/Helius) | T3C.7, T3C.8 | HMAC-verified confirmation webhooks |
| T3C.10 | Portfolio State Manager (PSM) | T3C.9 | Reconciliation, snapshot updates |

#### Sub-Phase 3D: Protocol Adapters (Weeks 5–10)
| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T3D.1 | Uniswap V3 Adapter (ETH/BASE/ARB) | T3A.9 | SWAP action encoding |
| T3D.2 | Aerodrome Adapter (BASE) | T3A.9 | SWAP on Base |
| T3D.3 | Aave V3 Adapter (ETH/BASE/ARB) | T3A.9 | LEND_SUPPLY / LEND_WITHDRAW |
| T3D.4 | Moonwell Adapter (BASE) | T3A.9 | LEND_SUPPLY / LEND_WITHDRAW on Base |
| T3D.5 | Jupiter Adapter (Solana) | T3A.9 | /order + /build endpoints |
| T3D.6 | Raydium Adapter (Solana) | T3A.9 | SWAP on Solana |
| T3D.7 | Kamino Adapter (Solana) | T3A.9 | Solana lending |
| T3D.8 | MarginFi Adapter (Solana) | T3A.9 | Solana lending |
| T3D.9 | CCTP V2 Bridge Adapter | T3C.7, T3C.8 | depositForBurn + receiveMessage + Iris polling |

#### Sub-Phase 3E: Payments (Weeks 7–9)
| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T3E.1 | x402 Middleware | T3A.4 | 402 response, free tier check |
| T3E.2 | x402.org Facilitator Integration | T3E.1 | /verify + /settle calls; PaymentRecord |
| T3E.3 | EIP-3009 Authorization Verification | T3E.2 | Replay protection, timing window |

---

### Phase 4: Testing
**Estimated duration: 4–6 weeks** (overlaps with Phase 3)

| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T4.1 | Unit Tests (NestJS services) | T3A–T3E (each module) | >80% coverage per module |
| T4.2 | Integration Tests (DB + Redis + BullMQ) | T3A.2, T3A.3 | Integration test suite |
| T4.3 | Policy Engine Test Suite (9 checks × scenarios) | T3C.4 | All check combinations tested |
| T4.4 | Testnet Deployment (Sepolia + Base Sepolia) | T3D.1–T3D.9 | Testnet environment live |
| T4.5 | **Testnet Release Gate: Swap** | T4.4 | ≥1 successful swap on Sepolia |
| T4.6 | **Testnet Release Gate: Aave V3** | T4.4 | ≥1 successful lend/withdraw |
| T4.7 | **Testnet Release Gate: CCTP** | T4.4, T3D.9 | ≥1 successful cross-chain USDC transfer |
| T4.8 | **Testnet Release Gate: x402** | T4.4, T3E.2 | ≥1 successful paid Copilot query |
| T4.9 | Load + Stress Testing | T4.4 | Performance benchmarks |
| T4.10 | Security Audit | T4.4 | External security review |

---

### Phase 5: Documentation
**Estimated duration: 2–3 weeks** (overlaps with Phase 4)

| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T5.1 | BE Project Report (Core) | T2.9 | Report chapters 1–4 |
| T5.2 | API Documentation (OpenAPI) | T2.10, T3A–T3E | Swagger/Redoc published |
| T5.3 | UML Diagram Finalisation in Lucidchart | T2.9 | All 20 Lucidchart diagrams |
| T5.4 | BE Project Report (Testing + Results) | T4.5–T4.10 | Report chapters 5–7 |
| T5.5 | Deployment & Operations Guide | T6.1 | Runbook |

---

### Phase 6: Deployment
**Estimated duration: 1–2 weeks**

| Task ID | Task Name | Dependencies | Key Deliverable |
|---|---|---|---|
| T6.1 | Production Infrastructure Setup | T4.9 | NestJS cluster, PostgreSQL primary+replica, Redis cluster |
| T6.2 | KMS / Secrets Manager Configuration | T6.1 | All API keys in KMS |
| T6.3 | **Mainnet Release Gate Decision** | T4.10, T6.2 | Separate release decision (not yet triggered in planning docs) |
| T6.4 | Mainnet Launch (Phase A — ETH, BASE, ARB) | T6.3 | Live mainnet deployment |
| T6.5 | Mainnet Launch (Phase B — Solana) | T6.4 | Solana mainnet enabled |

---

## 4. Lucidchart Gantt Specification

**Lucidchart does not have a native Gantt chart shape.** Use one of these approaches:

**Option A (Recommended): Table + Swimlanes**
- Create a horizontal table with phases as rows and time (weeks/months) as columns.
- Color each task bar by phase: Planning (blue), Design (green), Development (orange), Testing (red), Docs (purple), Deployment (teal).
- Use Lucidchart's "Timeline" template under Charts.

**Option B: Import Mermaid and screenshot**
- Use the Mermaid Gantt below; render it and import as an image into Lucidchart.
- Label each bar with Task ID.

**Column headers (time scale):** Use relative weeks (Week 1, Week 2… Week 24) or absolute months.

**Key visual elements:**
- Double-line separator between phases
- Diamond shapes at milestone/gate points (T4.5–T4.8, T6.3)
- Critical path highlighted: T1.5 → T3A.7 (HyperVerge) is a dependency gate
- Parallel bars for T3B + T3C + T3D (intelligence, execution, adapters)

---

## 5. Critical Path

```
T1.5 (KYC spike) → T3A.7 (KYC integration) → T4.6 (Testnet: Aave Gate)
T3A.9 (pool-type lib) → T3C.4 (Policy Engine) → T4.3 (PE test suite) → T4.5–T4.8
T3D.9 (CCTP adapter) → T4.7 (Testnet: CCTP Gate)
T3E.2 (x402 facilitator) → T4.8 (Testnet: x402 Gate)
All gates (T4.5–T4.8) → T6.3 (Mainnet Gate) → T6.4 (Mainnet Launch)
```

---

## 6. Supplementary Mermaid

```mermaid
gantt
    title CrestFlow MVP Development Timeline (Relative Weeks)
    dateFormat  YYYY-MM-DD
    axisFormat  Week %W

    section Phase 1: Requirements
    Requirements Gathering     :T1_1, 2026-01-01, 1w
    Tech Stack Finalisation    :T1_2, after T1_1, 1w
    Canonical Decisions        :T1_3, after T1_2, 1w
    HyperVerge KYC Spike       :crit, T1_5, after T1_3, 2w
    MVP Scope Definition       :T1_6, after T1_3, 1w

    section Phase 2: Design
    System Architecture        :T2_1, after T1_6, 1w
    Schema Design              :T2_2, after T2_1, 1w
    Engine Specs (E1-E6)       :T2_4, after T2_1, 2w
    Execution Pipeline Spec    :T2_5, after T2_1, 2w
    UML Pack (this document)   :T2_9, after T2_5, 1w

    section Phase 3A: Foundation
    NestJS Setup + CI/CD       :T3A_1, after T2_1, 1w
    DB Migrations (29 tables)  :T3A_2, after T2_2, 1w
    Identity Module + OAuth    :T3A_4, after T3A_2, 1w
    Turnkey Integration        :T3A_5, after T3A_4, 1w
    HyperVerge KYC Integration :crit, T3A_7, after T1_5, 2w
    pool-type Shared Library   :T3A_9, after T3A_1, 1w

    section Phase 3B: Engines
    Engine 1: Portfolio        :T3B_2, after T3A_2, 2w
    Engine 2: Risk             :T3B_3, after T3B_2, 1w
    Engine 3: Yield            :T3B_4, after T3B_2, 1w
    Engine 4: Strategy         :T3B_5, after T3B_3, 1w
    Engine 5: Copilot          :T3B_6, after T3B_2, 2w
    Engine 6: Execution Intel  :T3B_7, after T3B_5, 1w

    section Phase 3C: Execution Pipeline
    Execution Router           :T3C_1, after T3B_7, 1w
    Orchestrator/Planner       :T3C_2, after T3C_1, 1w
    Simulation Engine          :T3C_3, after T3C_2, 1w
    Policy Engine (9 checks)   :crit, T3C_4, after T3C_3, 1w
    CCE (Nonce + Sign + Bcast) :T3C_7, after T3C_4, 2w
    Webhook Consumer           :T3C_9, after T3C_7, 1w
    PSM Reconciliation         :T3C_10, after T3C_9, 1w

    section Phase 3D: Protocol Adapters
    Uniswap V3 (EVM)           :T3D_1, after T3A_9, 2w
    Aave V3 (EVM)              :T3D_3, after T3A_9, 1w
    Jupiter + Raydium (Sol)    :T3D_5, after T3A_9, 2w
    Kamino + MarginFi (Sol)    :T3D_7, after T3A_9, 1w
    CCTP V2 Adapter            :crit, T3D_9, after T3C_7, 1w

    section Phase 3E: Payments
    x402 Middleware            :T3E_1, after T3A_4, 1w
    x402 Facilitator           :T3E_2, after T3E_1, 1w

    section Phase 4: Testing
    Unit Tests (all modules)   :T4_1, after T3C_10, 2w
    Testnet Deployment         :T4_4, after T4_1, 1w
    Gate: Swap                 :milestone, T4_5, after T4_4, 0d
    Gate: Aave V3              :milestone, T4_6, after T4_4, 0d
    Gate: CCTP                 :milestone, T4_7, after T4_4, 0d
    Gate: x402                 :milestone, T4_8, after T4_4, 0d
    Security Audit             :crit, T4_10, after T4_4, 2w

    section Phase 5: Documentation
    BE Project Report          :T5_1, after T2_9, 4w
    Lucidchart Diagrams        :T5_3, after T2_9, 2w

    section Phase 6: Deployment
    Production Infra Setup     :T6_1, after T4_10, 1w
    Mainnet Gate Decision      :milestone, crit, T6_3, after T6_1, 0d
    Mainnet Launch             :T6_4, after T6_3, 1w
```

---

## 7. Flags & Open Questions

| Flag ID | Type | Description |
|---|---|---|
| F-18-01 | **No actual dates** | Actual project dates are not documented in the planning files. All durations above are estimates. Insert real dates from your project schedule. |
| F-18-02 | **D9 critical path** | HyperVerge KYC integration (T1.5, T3A.7) is a critical path item because D9 decision ("selected pending integration validation") could require pivoting to Persona or Sumsub, adding 1–2 weeks. |
| F-18-03 | **Mainnet gate** | The mainnet release gate (T6.3) is a separate decision point that requires all testnet gates (T4.5–T4.8) plus security audit (T4.10) to pass. The gate has not been triggered in the planning docs. |
| F-18-04 | **Parallelisation** | Sub-phases 3B, 3C, 3D, and 3E can be parallelised across sub-teams. Single-developer projects must sequence them. |
| F-18-05 | **Phase 2 scope** | Black-Litterman optimizer, GMX automation, ERC-4337, fiat on-ramp, and other Phase 2 items are excluded from this Gantt. |
