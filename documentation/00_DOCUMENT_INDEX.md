# CrestFlow Consolidated Documentation Index

**Status:** Authoritative master index  
**Version:** 2.3  
**Last updated:** 20 November 2026  
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
├── 05_research_and_audits/            # Literature review, paper list & audit reports
│   ├── literature_survey.md           # Comprehensive academic literature review
│   ├── CrestFlow Literature Survey Papers List.csv # 54 indexed research papers
│   ├── AUDIT_REPORT.md                # Pre-remediation audit snapshot (historical findings)
│   ├── REMEDIATION_REPORT.md          # 39-finding disposition and re-audit evidence
│   └── integration_audit_report.md    # Pre-pivot cleanup & audit history
│
├── 06_uml_diagrams/                   # Current Mermaid source and rendered PNGs
│   ├── README.md                      # Render guide, notation mapping, traceability
│   ├── 01_component.mmd … 19_sdlc.mmd # 19 diagram sources; see §4
│   ├── puppeteer-mono.json            # Monochrome render launch config
│   └── png/                           # 19 rendered PNGs for the BE report
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
| **UML Source** | Current Mermaid diagram code rendered from canonical contracts. The README carries notation and traceability. |
| **Derived** | Summaries, cross-cutting indexes, orientations, and generated PNG exports. Must not define new contracts. |
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
| 24 | [`05_research_and_audits/literature_survey.md`](./05_research_and_audits/literature_survey.md) | Derived | Research | 2.0 | Academic literature review (54 papers) |
| 25 | [`05_research_and_audits/CrestFlow Literature Survey Papers List.csv`](./05_research_and_audits/CrestFlow%20Literature%20Survey%20Papers%20List.csv) | Derived | Research | 2.0 | Paper catalog with DOIs and verification status |
| 26 | [`05_research_and_audits/AUDIT_REPORT.md`](./05_research_and_audits/AUDIT_REPORT.md) | Historical | Architecture | 2.0 | Pre-remediation audit snapshot (39 findings) |
| 27 | [`05_research_and_audits/integration_audit_report.md`](./05_research_and_audits/integration_audit_report.md) | Historical | Docs | 1.1 | Pre-pivot cleanup report (historical reference) |
| 28 | [`assets/system-architecture-post-pivot.png`](./assets/system-architecture-post-pivot.png) | Component | Architecture | 2.0 | Current system architecture diagram |
| 29 | [`assets/system-architecture-pre-pivot.png`](./assets/system-architecture-pre-pivot.png) | Historical | Architecture | 1.0 | Pre-pivot architecture diagram |
| 30 | [`assets/system-architecture-before-pivot-legacy.png`](./assets/system-architecture-before-pivot-legacy.png) | Historical | Architecture | 1.0 | Legacy architecture graphic |
| 31 | [`06_uml_diagrams/README.md`](./06_uml_diagrams/README.md) | UML Source | Architecture | 1.2 | Render guide, monochrome notation, traceability matrix |
| 32 | [`06_uml_diagrams/`](./06_uml_diagrams/) | UML Source | Architecture | 1.2 | 19 current Mermaid diagrams (component, class, 5 sequences, 3 states, deployment, architecture, DFD 0/1, use case, activity, ER, Gantt, SDLC) |
| 33 | [`06_uml_diagrams/png/`](./06_uml_diagrams/png/) | Derived | Architecture | 1.1 | 19 rendered PNGs (2x scale, white background, strictly monochrome) |
| 34 | [`05_research_and_audits/REMEDIATION_REPORT.md`](./05_research_and_audits/REMEDIATION_REPORT.md) | Derived | Architecture | 1.0 | 39-finding disposition and re-audit evidence |

---

## 4. Current diagram pack

The current diagram source is `06_uml_diagrams/`. Each `.mmd` file has a rendered PNG in `06_uml_diagrams/png/` at 2x scale with a white background and strictly black-and-white linework. The pack is rendered through `puppeteer-mono.json` so no colored subpixel text fringes are written.

| # | Diagram | Type | PNG |
|---:|---|---|---|
| 01 | Component | UML component (flowchart approximation) | [PNG](./06_uml_diagrams/png/01_component.png) |
| 02 | Domain class | UML class | [PNG](./06_uml_diagrams/png/02_class_domain.png) |
| 03 | Onboarding and KYC | UML sequence | [PNG](./06_uml_diagrams/png/03_sequence_onboarding_kyc.png) |
| 04 | Policy evaluation | UML sequence | [PNG](./06_uml_diagrams/png/04_sequence_policy.png) |
| 05 | Execution pipeline | UML sequence | [PNG](./06_uml_diagrams/png/05_sequence_execution.png) |
| 06 | CCTP V2 bridge | UML sequence | [PNG](./06_uml_diagrams/png/06_sequence_cctp.png) |
| 07 | x402 payment | UML sequence | [PNG](./06_uml_diagrams/png/07_sequence_x402.png) |
| 08 | Execution saga | UML state machine | [PNG](./06_uml_diagrams/png/08_state_saga.png) |
| 09 | KYC lifecycle | UML state machine | [PNG](./06_uml_diagrams/png/09_state_kyc.png) |
| 10 | Autopilot delegation | UML state machine | [PNG](./06_uml_diagrams/png/10_state_delegation.png) |
| 11 | Deployment | UML deployment (flowchart approximation) | [PNG](./06_uml_diagrams/png/11_deployment.png) |
| 12 | System architecture | Layered architecture view | [PNG](./06_uml_diagrams/png/12_system_architecture.png) |
| 13 | Context | DFD level 0 | [PNG](./06_uml_diagrams/png/13_dfd_level0.png) |
| 14 | Subsystems | DFD level 1 | [PNG](./06_uml_diagrams/png/14_dfd_level1.png) |
| 15 | Use cases | UML use case (flowchart approximation) | [PNG](./06_uml_diagrams/png/15_use_case.png) |
| 16 | Execution activity | UML activity | [PNG](./06_uml_diagrams/png/16_activity_execution.png) |
| 17 | Core data model | ER diagram | [PNG](./06_uml_diagrams/png/17_er_core.png) |
| 18 | MVP schedule | Gantt | [PNG](./06_uml_diagrams/png/18_gantt.png) |
| 19 | SDLC | Iterative-incremental lifecycle view | [PNG](./06_uml_diagrams/png/19_sdlc.png) |

See [`06_uml_diagrams/README.md`](./06_uml_diagrams/README.md) for rendering, notation, and the diagram-to-decision traceability matrix. PNGs are generated exports; re-render them after changing a `.mmd` file.

---

## 5. Change log

| Version | Date | Change |
|---|---|---|
| 2.3 | 20 Nov 2026 | Reduced all 19 diagrams by one abstraction level; made the pack strictly monochrome; simplified the use-case view; added `puppeteer-mono.json` and refreshed diagram counts. |
| 2.2 | 19 Nov 2026 | Removed the superseded diagram specifications and all references; added 19 rendered PNG exports; updated diagram navigation and register. |
| 2.1 | 19 Nov 2026 | Added `06_uml_diagrams/` (19 Mermaid diagrams + README); added `REMEDIATION_REPORT.md`; marked `AUDIT_REPORT.md` historical; updated register and directory tree. |
| 2.0 | 18 Nov 2026 | Canonical decisions v1.2, canonical schemas v2.0, and 22 component/planning documents rewritten after the pivot. |
| 1.0 | Pre-pivot | Initial consolidated documentation index. |
