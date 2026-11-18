# CrestFlow Consolidated Documentation Index

**Status:** Authoritative master index  
**Version:** 2.0  
**Last updated:** 18 November 2026  
**Structure:** Unified Documentation Repository (`documentation/`)  
**Purpose:** Declare the class, owner, version, canonical scope, and relative path of every planning and architectural document.

---

## 1. Directory Structure

```text
documentation/
├── 00_DOCUMENT_INDEX.md               # Master navigation index (this file)
│
├── 01_canonical/                      # Authoritative decision records & shared schemas
│   ├── CANONICAL_DECISIONS.md         # Top of hierarchy: D1–D10 decisions, T1–T17 rules
│   └── canonical_schemas.md          # Shared domain types, enums, units, and invariants
│
├── 02_architecture/                   # System & component technical specifications
│   ├── system_architecture.md         # Overall 5-layer system architecture & topology
│   ├── implementation_spec.md         # Backend API, DB, nonce, gas, saga, and security spec
│   ├── policy_engine.md               # 9 policy evaluation checks and predicates
│   ├── orchestrator_planner.md        # Planner instruction build/simulation, saga lifecycle
│   ├── execution_router.md            # Venue, pool type, split order, and CCTP routing
│   ├── protocol_layer.md              # Venue adapters and unsigned transaction generation
│   ├── cross_chain_ecosystem.md       # CCE, TEE signing, broadcast, and bridge tracking
│   ├── identity_kyc.md                # Identity, Turnkey TEE sub-orgs, and KYC statuses
│   ├── user_onboarding.md             # Path A/B user onboarding and wallet creation flows
│   ├── audit_layer_x402.md            # Audit taxonomy, immutability, and x402 payment gateway
│   ├── financial_knowledge_layer.md   # Market data adapters, price unification, and oracles
│   ├── portfolio_state_manager.md     # Post-execution sync, webhooks, and reconciliation
│   ├── portfolio_state_analysis.md    # Portfolio aggregation, address scopes, and valuation
│   ├── technical_reference.md         # Full cross-cutting technical reference index
│   └── technical_reference_summary.md # One-page orientation summary
│
├── 03_engines/                        # The 6 core intelligence engines
│   ├── engine1_portfolio_intelligence.md # Portfolio snapshots, HHI, WAC, health score
│   ├── engine2_risk_intelligence.md      # Risk scoring, persona multipliers, risk alerts
│   ├── engine3_yield_opportunity.md      # Yield discovery, APY normalisation, TOPSIS ranking
│   ├── engine4_strategy_optimization.md  # Strategy optimization, drift, cooldown, plan building
│   ├── engine5_user_intelligence.md      # Persona clustering, GoalProfile, Copilot AI
│   └── engine6_execution_intelligence.md # Authority filter, execution waterfall, ExecutionIntent
│
├── 04_uml_pack/                       # Lucidchart-ready UML & BE project diagram specifications
│   ├── 00_INDEX.md                    # UML Pack master index, shape guides, and color palette
│   ├── 01_component_diagram.md        # Component Diagram (47 elements, 19 interfaces)
│   ├── 02_class_diagram.md            # Class Diagram (30+ domain entities, 36 relationships)
│   ├── 03_seq_onboarding_kyc.md       # Sequence Diagram: User Onboarding & KYC
│   ├── 04_seq_policy_evaluation.md    # Sequence Diagram: Policy Evaluation & Authorization
│   ├── 05_seq_transaction_execution.md # Sequence Diagram: Transaction Execution Pipeline
│   ├── 06_seq_cctp_bridge.md          # Sequence Diagram: CCTP V2 Cross-Chain Transfer
│   ├── 07_seq_x402_payment.md         # Sequence Diagram: x402 Micropayment & Settlement
│   ├── 08_state_saga_lifecycle.md     # State Machine: Execution Saga Lifecycle (18 states)
│   ├── 09_state_kyc_lifecycle.md      # State Machine: KYC Lifecycle (6 states)
│   ├── 10_state_delegation_lifecycle.md # State Machine: Autopilot Delegation Lifecycle
│   ├── 11_deployment_diagram.md       # Deployment Diagram (infrastructure & trust boundaries)
│   ├── 12_BE_system_architecture.md   # BE Report: System Architecture Overview
│   ├── 13_BE_dfd_level0.md            # BE Report: DFD Level 0 Context Diagram (36 flows)
│   ├── 14_BE_dfd_level1.md            # BE Report: DFD Level 1 Subsystem Decomposition
│   ├── 15_BE_use_case.md              # BE Report: Use Case Diagram (40 UCs, 9 actors)
│   ├── 16_BE_activity_diagram.md      # BE Report: Activity Diagram (5 swimlanes)
│   ├── 17_BE_er_diagram.md            # BE Report: Entity-Relationship Diagram (29 tables)
│   ├── 18_project_gantt.md            # Project Planning: Gantt Chart & Release Gates
│   ├── 19_project_sdlc.md             # Project Planning: Iterative Incremental SDLC Model
│   └── 20_unresolved_decisions.md     # Consolidated Open Questions, Flags & Assumptions
│
├── 05_research_and_audits/            # Literature review, paper list & audit reports
│   ├── literature_survey.md           # Comprehensive academic literature review
│   ├── CrestFlow Literature Survey Papers List.csv # 54 indexed research papers
│   ├── AUDIT_REPORT.md                # Architecture remediation audit report
│   └── integration_audit_report.md    # Pre-pivot cleanup & audit history
│
└── assets/                            # Architecture diagrams & graphic assets
    ├── system-architecture-post-pivot.png           # Current post-pivot system topology
    ├── system-architecture-pre-pivot.png            # Historical pre-pivot architecture
    └── system-architecture-before-pivot-legacy.png  # Historical legacy architecture diagram
```

---

## 2. Document Classes

| Class | Rule |
|---|---|
| **Authoritative** | Defines an immutable contract or rule. All other documents must link to it and must not restate its content. |
| **Component** | Owns a specific subsystem or engine algorithm and local schemas. Shared types must reference `01_canonical/canonical_schemas.md`. |
| **UML Specification** | Lucidchart-ready architectural and behavioral blueprints for visual modeling and BE project documentation. |
| **Derived** | Summaries, cross-cutting indexes, and orientations. Must not define new contracts. |
| **Historical** | Retained for historical reference, audit trails, and context. |

---

## 3. Master Document Register

| # | Document Path | Class | Owner | Version | Scope |
|---:|---|---|---|---|---|
| 1 | [`01_canonical/CANONICAL_DECISIONS.md`](./01_canonical/CANONICAL_DECISIONS.md) | Authoritative | Architecture | 1.2 | Highest priority: D1–D10 decisions, T1–T17 rules |
| 2 | [`01_canonical/canonical_schemas.md`](./01_canonical/canonical_schemas.md) | Authoritative | Architecture | 2.0 | Shared types, enums, units, invariants |
| 3 | [`02_architecture/system_architecture.md`](./02_architecture/system_architecture.md) | Component | Architecture | 2.0 | Overall 5-layer topology, trust boundaries |
| 4 | [`02_architecture/implementation_spec.md`](./02_architecture/implementation_spec.md) | Authoritative | Backend | 1.0 | DB schema, APIs, saga, nonce, gas formulas |
| 5 | [`02_architecture/policy_engine.md`](./02_architecture/policy_engine.md) | Authoritative | Risk/Policy | 2.0 | 9 policy checks, predicates, evaluation rules |
| 6 | [`02_architecture/orchestrator_planner.md`](./02_architecture/orchestrator_planner.md) | Component | Execution | 2.0 | Instruction building, simulation, saga control |
| 7 | [`02_architecture/execution_router.md`](./02_architecture/execution_router.md) | Component | Execution | 2.0 | Venue selection, pool types, split routing |
| 8 | [`02_architecture/protocol_layer.md`](./02_architecture/protocol_layer.md) | Component | Execution | 2.0 | Protocol adapters, calldata encoding |
| 9 | [`02_architecture/cross_chain_ecosystem.md`](./02_architecture/cross_chain_ecosystem.md) | Component | Execution | 2.0 | TEE signing, CCE broadcast, CCTP V2 tracking |
| 10 | [`02_architecture/identity_kyc.md`](./02_architecture/identity_kyc.md) | Component | Compliance | 2.0 | HyperVerge flow, KYC tiers, sanctions rules |
| 11 | [`02_architecture/user_onboarding.md`](./02_architecture/user_onboarding.md) | Component | Product/Identity | 2.0 | OAuth, Path A/B provisioning, address scopes |
| 12 | [`02_architecture/audit_layer_x402.md`](./02_architecture/audit_layer_x402.md) | Component | Audit/Payments | 2.0 | Audit event schema, x402 payment flow |
| 13 | [`02_architecture/financial_knowledge_layer.md`](./02_architecture/financial_knowledge_layer.md) | Component | Data | 2.0 | Market data feeds, caching TTLs, normalisation |
| 14 | [`02_architecture/portfolio_state_manager.md`](./02_architecture/portfolio_state_manager.md) | Component | Data | 2.0 | PSM post-execution sync, webhooks, reconciliation |
| 15 | [`02_architecture/portfolio_state_analysis.md`](./02_architecture/portfolio_state_analysis.md) | Component | Data | 2.0 | Position scanning, valuation, data quality |
| 16 | [`02_architecture/technical_reference.md`](./02_architecture/technical_reference.md) | Derived | Architecture | 2.0 | Navigational index across all technical specs |
| 17 | [`02_architecture/technical_reference_summary.md`](./02_architecture/technical_reference_summary.md) | Derived | Architecture | 2.0 | One-page subsystem orientation summary |
| 18 | [`03_engines/engine1_portfolio_intelligence.md`](./03_engines/engine1_portfolio_intelligence.md) | Component | Engines | 2.0 | Portfolio snapshots, HHI concentration bands |
| 19 | [`03_engines/engine2_risk_intelligence.md`](./03_engines/engine2_risk_intelligence.md) | Component | Engines | 2.0 | Risk score computation, persona multipliers |
| 20 | [`03_engines/engine3_yield_opportunity.md`](./03_engines/engine3_yield_opportunity.md) | Component | Engines | 2.0 | TOPSIS 5-criteria ranking, APY tracking |
| 21 | [`03_engines/engine4_strategy_optimization.md`](./03_engines/engine4_strategy_optimization.md) | Component | Engines | 2.0 | Strategy planning, cooldown gates, rebalancing |
| 22 | [`03_engines/engine5_user_intelligence.md`](./03_engines/engine5_user_intelligence.md) | Component | Engines | 2.0 | Persona profiling, GoalProfile, Copilot AI |
| 23 | [`03_engines/engine6_execution_intelligence.md`](./03_engines/engine6_execution_intelligence.md) | Component | Engines | 2.0 | Execution intent generation, waterfall logic |
| 24 | [`04_uml_pack/00_INDEX.md`](./04_uml_pack/00_INDEX.md) | UML Specification | Architecture | 1.0 | UML pack register, visual standards, color scheme |
| 25 | [`04_uml_pack/01_component_diagram.md`](./04_uml_pack/01_component_diagram.md) | UML Specification | Architecture | 1.0 | Component Diagram specification |
| 26 | [`04_uml_pack/02_class_diagram.md`](./04_uml_pack/02_class_diagram.md) | UML Specification | Architecture | 1.0 | Class Diagram specification |
| 27 | [`04_uml_pack/03_seq_onboarding_kyc.md`](./04_uml_pack/03_seq_onboarding_kyc.md) | UML Specification | Architecture | 1.0 | Sequence: User Onboarding & KYC |
| 28 | [`04_uml_pack/04_seq_policy_evaluation.md`](./04_uml_pack/04_seq_policy_evaluation.md) | UML Specification | Architecture | 1.0 | Sequence: Policy Evaluation (9 checks) |
| 29 | [`04_uml_pack/05_seq_transaction_execution.md`](./04_uml_pack/05_seq_transaction_execution.md) | UML Specification | Architecture | 1.0 | Sequence: Full Execution Pipeline |
| 30 | [`04_uml_pack/06_seq_cctp_bridge.md`](./04_uml_pack/06_seq_cctp_bridge.md) | UML Specification | Architecture | 1.0 | Sequence: CCTP V2 Cross-Chain USDC Transfer |
| 31 | [`04_uml_pack/07_seq_x402_payment.md`](./04_uml_pack/07_seq_x402_payment.md) | UML Specification | Architecture | 1.0 | Sequence: x402 Micropayment & Settlement |
| 32 | [`04_uml_pack/08_state_saga_lifecycle.md`](./04_uml_pack/08_state_saga_lifecycle.md) | UML Specification | Architecture | 1.0 | State Machine: Execution Saga (18 states) |
| 33 | [`04_uml_pack/09_state_kyc_lifecycle.md`](./04_uml_pack/09_state_kyc_lifecycle.md) | UML Specification | Architecture | 1.0 | State Machine: KYC Status Transitions |
| 34 | [`04_uml_pack/10_state_delegation_lifecycle.md`](./04_uml_pack/10_state_delegation_lifecycle.md) | UML Specification | Architecture | 1.0 | State Machine: Autopilot Delegation Lifecycle |
| 35 | [`04_uml_pack/11_deployment_diagram.md`](./04_uml_pack/11_deployment_diagram.md) | UML Specification | Architecture | 1.0 | Deployment Diagram & Trust Boundaries |
| 36 | [`04_uml_pack/12_BE_system_architecture.md`](./04_uml_pack/12_BE_system_architecture.md) | UML Specification | Architecture | 1.0 | BE Report: System Architecture Narrative |
| 37 | [`04_uml_pack/13_BE_dfd_level0.md`](./04_uml_pack/13_BE_dfd_level0.md) | UML Specification | Architecture | 1.0 | BE Report: DFD Level 0 Context Diagram |
| 38 | [`04_uml_pack/14_BE_dfd_level1.md`](./04_uml_pack/14_BE_dfd_level1.md) | UML Specification | Architecture | 1.0 | BE Report: DFD Level 1 Subsystem Decomposition |
| 39 | [`04_uml_pack/15_BE_use_case.md`](./04_uml_pack/15_BE_use_case.md) | UML Specification | Architecture | 1.0 | BE Report: Use Case Diagram & Actor Matrix |
| 40 | [`04_uml_pack/16_BE_activity_diagram.md`](./04_uml_pack/16_BE_activity_diagram.md) | UML Specification | Architecture | 1.0 | BE Report: Activity Diagram (5 swimlanes) |
| 41 | [`04_uml_pack/17_BE_er_diagram.md`](./04_uml_pack/17_BE_er_diagram.md) | UML Specification | Architecture | 1.0 | BE Report: ER Diagram (29 PostgreSQL tables) |
| 42 | [`04_uml_pack/18_project_gantt.md`](./04_uml_pack/18_project_gantt.md) | UML Specification | Architecture | 1.0 | Project Planning: Gantt Schedule & Gates |
| 43 | [`04_uml_pack/19_project_sdlc.md`](./04_uml_pack/19_project_sdlc.md) | UML Specification | Architecture | 1.0 | Project Planning: Iterative SDLC Model |
| 44 | [`04_uml_pack/20_unresolved_decisions.md`](./04_uml_pack/20_unresolved_decisions.md) | UML Specification | Architecture | 1.0 | Open Questions, Decisions & Phase 2 Scope |
| 45 | [`05_research_and_audits/literature_survey.md`](./05_research_and_audits/literature_survey.md) | Derived | Research | 2.0 | Academic literature review (54 papers) |
| 46 | [`05_research_and_audits/CrestFlow Literature Survey Papers List.csv`](./05_research_and_audits/CrestFlow%20Literature%20Survey%20Papers%20List.csv) | Derived | Research | 2.0 | Paper catalog with DOIs and verification status |
| 47 | [`05_research_and_audits/AUDIT_REPORT.md`](./05_research_and_audits/AUDIT_REPORT.md) | Derived | Architecture | 2.0 | Master audit report and remediation trail |
| 48 | [`05_research_and_audits/integration_audit_report.md`](./05_research_and_audits/integration_audit_report.md) | Historical | Docs | 1.1 | Pre-pivot cleanup report (historical reference) |
| 49 | [`assets/system-architecture-post-pivot.png`](./assets/system-architecture-post-pivot.png) | Component | Architecture | 2.0 | Current system architecture diagram |
| 50 | [`assets/system-architecture-pre-pivot.png`](./assets/system-architecture-pre-pivot.png) | Historical | Architecture | 1.0 | Pre-pivot architecture diagram |
| 51 | [`assets/system-architecture-before-pivot-legacy.png`](./assets/system-architecture-before-pivot-legacy.png) | Historical | Architecture | 1.0 | Legacy architecture graphic |
