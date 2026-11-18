# CrestFlow UML & Architecture Documentation Pack — Master Index

**Status:** Specification Pack (Pre-diagram)  
**Version:** 1.0  
**Produced:** 18 November 2026  
**Source documents:** All authoritative and component documents as of v2.0 / CANONICAL_DECISIONS.md v1.2  
**Purpose:** Lucidchart-ready specifications for every required UML diagram. No existing planning documents were modified.

---

## How to Use This Pack

Each file in this pack is a self-contained specification for one diagram or diagram group.  
Read the file, then reproduce the diagram in Lucidchart using the element inventory, connection table, and layout guidance provided. PlantUML/Mermaid source is included at the end of each file as a supplementary check — it is **not** the primary deliverable.

**Construction order:** build diagrams in the order listed below. Later diagrams reference element names established in earlier ones.

---

## File Register

| # | File | Diagram | Category | Depends On |
|---|---|---|---|---|
| 01 | `01_component_diagram.md` | Component Diagram | Architecture | — |
| 02 | `02_class_diagram.md` | Class Diagram | Architecture | 01 |
| 03 | `03_seq_onboarding_kyc.md` | Sequence — Onboarding & KYC | Behaviour | 01, 02 |
| 04 | `04_seq_policy_evaluation.md` | Sequence — Policy Evaluation & Authorization | Behaviour | 01, 02 |
| 05 | `05_seq_transaction_execution.md` | Sequence — Transaction Execution | Behaviour | 01, 02, 04 |
| 06 | `06_seq_cctp_bridge.md` | Sequence — CCTP Cross-Chain Transfer | Behaviour | 01, 05 |
| 07 | `07_seq_x402_payment.md` | Sequence — x402 Payment & Settlement | Behaviour | 01, 02 |
| 08 | `08_state_saga_lifecycle.md` | State Machine — Execution Saga Lifecycle | State | 05 |
| 09 | `09_state_kyc_lifecycle.md` | State Machine — KYC Lifecycle | State | 03 |
| 10 | `10_state_delegation_lifecycle.md` | State Machine — Delegation (Autopilot) Lifecycle | State | 03 |
| 11 | `11_deployment_diagram.md` | Deployment Diagram | Infrastructure | 01 |
| 12 | `12_BE_system_architecture.md` | BE — System Architecture Overview | BE Report | 01 |
| 13 | `13_BE_dfd_level0.md` | BE — DFD Level 0 (Context) | BE Report | 12 |
| 14 | `14_BE_dfd_level1.md` | BE — DFD Level 1 (Subsystem) | BE Report | 13 |
| 15 | `15_BE_use_case.md` | BE — Use Case Diagram | BE Report | 01 |
| 16 | `16_BE_activity_diagram.md` | BE — Activity Diagram | BE Report | 05 |
| 17 | `17_BE_er_diagram.md` | BE — Entity-Relationship Diagram | BE Report | 02 |
| 18 | `18_project_gantt.md` | Project Planning — Gantt Chart | Planning | — |
| 19 | `19_project_sdlc.md` | Project Planning — SDLC Model | Planning | — |
| 20 | `20_unresolved_decisions.md` | Open Questions & Flags | Quality | All |

---

## Cross-Cutting Conventions

### Naming consistency (use exactly these names throughout all diagrams)

**Layers / Subsystems**
- `User Layer` — UA (Path A), UB (Path B), KYC
- `Intelligence Layer` — E1 through E6
- `Execution Layer` — ER (Execution Router), ORC (Orchestrator/Planner), SIM (Simulation), PE (Policy Engine), CCE (Cross-Chain Ecosystem)
- `Data Layer` — FKL (Financial Knowledge Layer), PSM (Portfolio State Manager), AL (Audit Layer), X402 (x402 Gateway)
- `Chain Layer` — ETH, BASE, ARB, SOL

**Component full names**
| Short | Full Name | NestJS Module |
|---|---|---|
| E1 | Portfolio Intelligence Engine | `portfolio` |
| E2 | Risk Intelligence Engine | `risk` |
| E3 | Yield & Opportunity Engine | `yield` |
| E4 | Strategy Optimization Engine | `strategy` |
| E5 | User Intelligence & Copilot Engine | `identity` + Copilot |
| E6 | Execution Intelligence Engine | `execution` |
| ER | Execution Router | `router` |
| ORC | Orchestrator / Planner | `orchestrator` + `planner` |
| SIM | Simulation | (sub-component of `planner`) |
| PE | Policy Engine | `policy` |
| CCE | Cross-Chain Ecosystem | `cce` |
| FKL | Financial Knowledge Layer | `fkl` |
| PSM | Portfolio State Manager | `psm` |
| AL | Audit Layer | `audit` |
| X402 | x402 Payment Gateway | `payments` |
| PL | Protocol Layer | (sub-component of `planner`) |

**Key canonical types (use exact names)**
`User`, `AddressScope`, `AuthSession`, `GoalProfile`, `AutopilotConfig`, `TopsisWeights`, `PortfolioSnapshot`, `Position`, `TokenPosition`, `LpPosition`, `LendingPosition`, `GmxPosition`, `HhiSet`, `AssetRef`, `RiskSnapshot`, `RiskAlert`, `YieldOpportunitySnapshot`, `YieldOpportunity`, `StrategyPlan`, `PlanAction`, `ExecutionIntent`, `ExecutionAction`, `ApprovalIntent`, `CctpBridgeIntent`, `ExecutionInstructions`, `ResolvedAction`, `SimulationPassedPackage`, `SimulatedAction`, `PolicyInput`, `PolicyDecision`, `PolicyCheckResult`, `AuditEntry`, `CctpTransfer`, `PaymentRequirements`, `PaymentPayload`, `PaymentRecord`

### UML shape guide for Lucidchart

| Element Type | Lucidchart Shape | Notes |
|---|---|---|
| Component | UML Component (box with two small rectangles on left side) | Use for engines, services, layers |
| Interface / Port | Small circle on component boundary | Named interfaces |
| Provided interface | Lollipop (circle on stick) | |
| Required interface | Socket (open arc on stick) | |
| Dependency | Dashed arrow with open arrowhead | |
| Association | Solid line | |
| Directed association | Solid line with open arrowhead | |
| Aggregation | Solid line with hollow diamond at owner | |
| Composition | Solid line with filled diamond at owner | |
| Inheritance / Generalization | Solid line with hollow triangle arrowhead | |
| Realization / Implementation | Dashed line with hollow triangle arrowhead | |
| Class | Rectangle with 3 compartments (name / attributes / methods) | |
| Actor | Stick figure | Use cases only |
| Use case | Oval | |
| State | Rounded rectangle | |
| Initial state | Filled circle | |
| Final state | Filled circle inside larger circle | |
| Transition | Solid arrow | |
| Swimlane | Vertical or horizontal lane | Sequence, activity |
| Node / Device | 3D box (cube face) | Deployment |
| Artifact | Rectangle with folded corner | Deployment |
| Database | Cylinder | Deployment |
| Boundary | Dashed rectangle | Trust / network zone |
| Note / Comment | Dog-eared rectangle | |

### Colour palette (recommended)

| Zone | Fill Colour | Border |
|---|---|---|
| User Layer | `#E3F2FD` (light blue) | `#1565C0` |
| Intelligence Layer | `#E8F5E9` (light green) | `#2E7D32` |
| Execution Layer | `#FFF3E0` (light orange) | `#E65100` |
| Data Layer | `#F3E5F5` (light purple) | `#6A1B9A` |
| Chain Layer | `#FCE4EC` (light pink) | `#880E4F` |
| External Provider | `#FAFAFA` (near white) | `#9E9E9E` |
| Trust boundary | No fill, dashed `#F44336` border | — |

---

## Source-of-Truth Cross-Reference

| Claim type | Authoritative source |
|---|---|
| Execution order | `CANONICAL_DECISIONS.md` D7 |
| Policy checks | `docs/policy_engine.md` |
| Canonical types | `docs/canonical_schemas.md` |
| API surface | `docs/implementation_spec.md` §2 |
| DB schema | `docs/implementation_spec.md` §3 |
| Saga states | `docs/implementation_spec.md` §7 |
| CCTP V2 parameters | `CANONICAL_DECISIONS.md` D5, §4.1 |
| x402 flow | `CANONICAL_DECISIONS.md` D4; `docs/audit_layer_x402.md` |
| Venue roster | `CANONICAL_DECISIONS.md` T12 |
| KYC statuses/tiers | `CANONICAL_DECISIONS.md` T8 |
| Risk score / bands | `CANONICAL_DECISIONS.md` T13 |
| Cache TTLs | `CANONICAL_DECISIONS.md` T6 |
| Oracle policy | `CANONICAL_DECISIONS.md` T4 |
| HHI formula | `CANONICAL_DECISIONS.md` T1 |
| Deployment stack | `CANONICAL_DECISIONS.md` D10, T15 |
