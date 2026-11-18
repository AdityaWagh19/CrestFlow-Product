# CrestFlow Document Index

**Status:** Authoritative index  
**Version:** 1.0  
**Last reviewed:** 18 November 2026  
**Purpose:** Declare the class, owner, version, and canonical scope of every planning document.

---

## 1. Document classes

| Class | Rule |
|---|---|
| **Authoritative** | Defines a contract. Other documents must link to it and must not restate it. |
| **Component** | Owns a component's algorithm and local schema extensions. Shared types come from `docs/canonical_schemas.md`. |
| **Derived** | Summary, index, or historical record. Must not define new contracts. |
| **Historical** | Kept for context only. Not evidence of current state. |

---

## 2. Document register

| # | Document | Class | Owner | Version | Canonical scope | Notes |
|---:|---|---|---|---|---|---|
| 1 | `CANONICAL_DECISIONS.md` | Authoritative | Architecture | 1.2 | D1–D10, T1–T17 | Top of the hierarchy; T10 action set completed. |
| 2 | `docs/canonical_schemas.md` | Authoritative | Architecture | 2.0 | Types, enums, units, invariants | Feeds UML class diagrams. |
| 3 | `docs/policy_engine.md` | Authoritative | Risk/Policy | 2.0 | Nine policy checks and predicates | Cooldown/payback moved to Engine 4. |
| 4 | `docs/implementation_spec.md` | Authoritative | Backend | 1.0 | API, DB, nonce, gas, approvals, saga, security, SLOs, providers, test vectors | Replaces scattered operational notes. |
| 5 | `docs/DOCUMENT_INDEX.md` | Authoritative | Docs | 1.0 | Document classes and status | This file. |
| 6 | `core_documents/system_architecture.md` | Component | Architecture | 2.0 | Topology, component boundaries, chain scope | No execution contract. |
| 7 | `docs/orchestrator_planner.md` | Component | Execution | 2.0 | Planner build/simulate, Orchestrator saga | Router runs before Planner. |
| 8 | `docs/execution_router.md` | Component | Execution | 2.0 | Venue/pool/split/CCTP routing | Produces `ExecutionInstructions`. |
| 9 | `docs/protocol_layer.md` | Component | Execution | 2.0 | Adapter contracts, `UnsignedTransaction` | CCTP V2; Path A only signing. |
| 10 | `docs/cross_chain_ecosystem.md` | Component | Execution | 2.0 | TEE signing, broadcast, gas, CCTP status | CCE owns nonce and bridge broadcast. |
| 11 | `docs/portfolio_state_manager.md` | Component | Data | 2.0 | Post-execution sync, webhooks, reconciliation | Source for T7. |
| 12 | `docs/portfolio_state_analysis.md` | Component | Data | 2.0 | Address model, scans, price unification | Path B read-only/manual. |
| 13 | `docs/financial_knowledge_layer.md` | Component | Data | 2.0 | Adapters, cache, normaliser, oracle pricing | Source for T4/T6. |
| 14 | `docs/identity_kyc.md` | Component | Compliance | 2.0 | HyperVerge flow, statuses, tiers, sanctions | Source for T8. |
| 15 | `docs/user_onboarding.md` | Component | Product/Identity | 2.0 | OAuth, Path A/B, Turnkey provisioning, KYC step | Source for T9. |
| 16 | `docs/audit_layer_x402.md` | Component | Audit/Payments | 2.0 | Audit taxonomy, lifecycle, x402 gateway | Source for T10 and D4. |
| 17 | `docs/engine1_portfolio_intelligence.md` | Component | Engines | 2.0 | Portfolio snapshot, HHI, WAC, health | T1 bands. |
| 18 | `docs/engine2_risk_intelligence.md` | Component | Engines | 2.0 | Risk analysers, `riskScore`, alerts | T13 scale. |
| 19 | `docs/engine3_yield_opportunity.md` | Component | Engines | 2.0 | Opportunity discovery, APY, TOPSIS | T2 five criteria. |
| 20 | `docs/engine4_strategy_optimization.md` | Component | Engines | 2.0 | Optimisation, drift, cooldown, plans | T5; economic gates. |
| 21 | `docs/engine5_user_intelligence.md` | Component | Engines | 2.0 | Persona, `GoalProfile`, Copilot | D8 models. |
| 22 | `docs/engine6_execution_intelligence.md` | Component | Engines | 2.0 | Authority filter, waterfall, `ExecutionIntent` | C-06 schema. |
| 23 | `docs/technical_reference.md` | Derived | Architecture | 2.0 | Cross-cutting index | Links to canonical sources. |
| 24 | `core_documents/technical_reference.md` | Derived | Architecture | 2.0 | Condensed taxonomy | Must not define contracts. |
| 25 | `core_documents/literature_survey.md` | Derived | Research | 2.0 | Literature review | 54 unique papers; duplicate removed, Barber & Odean split. |
| 26 | `core_documents/CrestFlow Literature Survey Papers List.csv` | Derived | Research | 2.0 | Paper index | 54 rows; DOI and verification columns. |
| 27 | `docs/integration_audit_report.md` | Historical | Docs | 1.1 | Pre-pivot cleanup record | Marked historical; zero-contradiction claim withdrawn. |
| 28 | `docs/assets/system-architecture-pre-pivot.png` | Historical | Docs | 1.0 | Pre-pivot diagram | Archived. |
| 29 | `docs/assets/system-architecture-post-pivot.png` | Component | Docs | 2.0 | Current topology diagram | Renamed from `Untitled-...png`. |
| 30 | `docs/assets/system-architecture-before-pivot-legacy.png` | Historical | Docs | 1.0 | Legacy pre-pivot diagram | Archived. |

---

## 3. Status header convention

Every authoritative or component document starts with:

```markdown
> **Status:** Authoritative | Component | Derived | Historical
> **Version:** x.y
> **Last reviewed:** YYYY-MM-DD
> **Canonical references:** CANONICAL_DECISIONS.md, docs/canonical_schemas.md
> **Supersedes:** previous version / conflicting section
```

Documents that do not yet carry the block are brought into compliance during remediation. The index remains the fallback source of truth for class and version.

---

## 4. Change control

1. A decision change updates `CANONICAL_DECISIONS.md` first.
2. A schema change updates `docs/canonical_schemas.md` second.
3. Affected component documents are updated in the same change set.
4. `docs/DOCUMENT_INDEX.md` version is bumped.
5. The re-audit checklist is run; no Critical/High finding may remain.
