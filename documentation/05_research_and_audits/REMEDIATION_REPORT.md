# CrestFlow Remediation Report

**Status:** Complete — re-audit passed  
**Version:** 1.0  
**Date:** 19 November 2026  
**Audit baseline:** `AUDIT_REPORT.md` (pre-remediation snapshot, 39 findings)  
**Scope:** All 26 files audited on 18 November 2026, plus the new canonical decision record, implementation specification, and diagram pack  
**Verdict:** **39/39 register findings closed; 0 Critical/High open; 0 Medium/Low open.** The three narrative cross-reference IDs (M-16, M-17, M-33) are also closed. One Medium finding (M-06) is closed as a gated external dependency, not left open.

---

## 1. Executive summary

The repository was remediated decision-first, then re-audited. The work did not copy-edit around contradictions; it removed the duplicate contracts and declared a single owner for each one.

The material changes are:

1. **Decision record.** `01_canonical/CANONICAL_DECISIONS.md` v1.2 records D1–D10 and T1–T17. Every P0/P1 finding maps to a decision or rule.
2. **Canonical schemas.** `01_canonical/canonical_schemas.md` v2.0 owns shared types, enums, units, and invariants. `compositeRiskScore`, `twapApy30dPercent`, `SimulationResult`, `BRIDGE_AGGREGATOR`, `KYC_PENDING`, and `PROFILE_CHANGE` are removed or explicitly mapped.
3. **One policy catalog.** `02_architecture/policy_engine.md` owns exactly nine checks. Cooldown and gas/bridge payback moved to Engine 4. The technical reference links to the policy document instead of restating a second catalog.
4. **One execution order.** Engine 6 → Execution Router → Planner/build → Simulation → Policy → CCE → PSM. The Router now runs before bytes are built, and Policy runs once on `SimulationPassedPackage`.
5. **CCTP V2 and x402 V2.** CCTP uses `destinationCaller`, `maxFee`, and `minFinalityThreshold`, with USDC only. x402 uses the official facilitator, Base USDC, CAIP-2 `eip155:8453`, EIP-3009 `exact`, and `PAYMENT-REQUIRED` / `PAYMENT-SIGNATURE` / `PAYMENT-RESPONSE`.
6. **Path B scope.** External wallets are read-only plus manual signing. Autonomous execution is Path A only. ERC-4337, EIP-7702, and SPL `delegate` claims are removed from the MVP.
7. **Metrics and data.** HHI uses `Σ(share²) × 10,000` with the approved bands and a new-placement block above 2,500. TOPSIS uses five criteria, persona weights, and `twapApy7dPercent`. Oracle, cache, indexer, and data-quality rules are specified.
8. **KYC and audit.** KYC statuses, tiers, and actions are canonical. Audit uses `category` + `action`, six categories, `eventId` idempotency, and append-only partitions.
9. **Documentation governance.** `T14` assigns ownership, all current documents have status/version headers, the master index is updated, and the old UML pack is historical.
10. **Diagram pack.** `documentation/06_uml_diagrams/` contains 19 concise Mermaid diagrams, a README with notation/rendering/traceability, and 19 rendered PNG exports in `png/`.

---

## 2. Finding disposition summary

| Severity | Total | Fixed | Closed-gated | Open |
|---|---:|---:|---:|---:|
| Critical | 6 | 6 | 0 | **0** |
| High | 14 | 14 | 0 | **0** |
| Medium | 15 | 14 | 1 | **0** |
| Low | 4 | 4 | 0 | **0** |
| **Total** | **39** | **38** | **1** | **0** |

**No Critical or High finding remains open.** M-06 is closed as a gated vendor dependency: the documentation no longer asserts unverified HyperVerge specifics, and D9 requires a two-week sandbox/webhook spike before any build commitment.

---

## 3. Detailed disposition

### 3.1 Critical findings

| ID | Finding | Disposition | Evidence |
|---|---|---|---|
| **C-01** | Execution order was circular and defined four ways. | **Fixed.** One order is canonical: Engine 6 → Router → Planner/build → Simulation → Policy → CCE → PSM. Policy evaluates a completed `SimulationPassedPackage`; it does not run before build. All architecture, planner, router, engine, and technical-reference text points to D7. | `CANONICAL_DECISIONS.md` D7; `02_architecture/system_architecture.md`; `02_architecture/implementation_spec.md` §7; `03_engines/engine6_execution_intelligence.md` |
| **C-02** | Two incompatible nine-check policy catalogs, with undefined fields. | **Fixed.** `policy_engine.md` is the only policy catalog: KYC, tier spend, authority, risk, chain allowlist, protocol allowlist, per-tx value, slippage, action types. `riskScore` and `GoalProfile.allowedActionTypes` are canonical. Cooldown and payback are not policy checks. | `CANONICAL_DECISIONS.md` D6, T13; `02_architecture/policy_engine.md`; `01_canonical/canonical_schemas.md`; `02_architecture/technical_reference.md` |
| **C-03** | Delegated Path B was infeasible for Aave, Uniswap, and Solana positions. | **Fixed.** D1 limits Path B to read-only plus manual signing. Path A is the only autonomous mode. ERC-4337 session keys, EIP-7702 delegation, and SPL `delegate` are removed from MVP and labelled Phase 2 options. | `CANONICAL_DECISIONS.md` D1; `02_architecture/protocol_layer.md`; `02_architecture/identity_kyc.md`; `02_architecture/user_onboarding.md`; `02_architecture/cross_chain_ecosystem.md` |
| **C-04** | CCTP used V1 four-argument calldata and hard-coded the embedded wallet as mint recipient. | **Fixed.** CCTP V2 uses `destinationCaller`, `maxFee`, and `minFinalityThreshold`; the mint recipient is a parameter; USDC only; non-USDC fails closed. Standard and Fast parameters follow D5. | `CANONICAL_DECISIONS.md` D5; `02_architecture/protocol_layer.md`; `02_architecture/cross_chain_ecosystem.md`; `02_architecture/implementation_spec.md` §7.4 |
| **C-05** | x402 used incompatible headers, a transaction-hash model, and a dead facilitator. | **Fixed.** Official `https://x402.org/facilitator`; Base USDC; CAIP-2 `eip155:8453`; `exact` scheme; EIP-3009 authorization; V2 headers; `$0.01/query`; 10 free authenticated queries per day; no subscription. | `CANONICAL_DECISIONS.md` D4, T17; `02_architecture/audit_layer_x402.md` |
| **C-06** | `ExecutionIntent` omitted execution-critical fields and used decimal strings without scale. | **Fixed.** The schema carries protocol, pool/market, source address, recipient/domain, approval target and amount, deadline, and base-unit amounts. `ExecutionInstructions`, `SimulationPassedPackage`, and `PolicyDecision` carry the resolved route and per-action data. | `01_canonical/canonical_schemas.md`; `03_engines/engine6_execution_intelligence.md`; `02_architecture/technical_reference.md`; `02_architecture/implementation_spec.md` §3 |

### 3.2 High findings

| ID | Finding | Disposition | Evidence |
|---|---|---|---|
| **H-01** | Router was placed after build/simulation and claimed it “does not route.” | **Fixed.** Router selects venue, pool, split, aggregator calldata, and CCTP parameters before the Planner builds bytes. The self-contradictory sentence is replaced with an explicit responsibility boundary. | `CANONICAL_DECISIONS.md` D7; `02_architecture/execution_router.md`; `02_architecture/orchestrator_planner.md` |
| **H-02** | Policy allowlist, adapter roster, and Router venues disagreed. | **Fixed.** T12 defines one MVP matrix: Aave V3, Uniswap V3, Aerodrome, Moonwell, Jupiter `/order` + `/build`, Raydium, Kamino, MarginFi. GMX is read-only. 1inch, UniswapX, Curve, and Camelot are Phase 2. | `CANONICAL_DECISIONS.md` T12; `02_architecture/policy_engine.md`; `02_architecture/execution_router.md`; `02_architecture/protocol_layer.md` |
| **H-03** | Permit2/Aave approval claims were wrong. | **Fixed.** Aave V3 uses EIP-2612 `supplyWithPermit`; Permit2 is not claimed for Aave. One-time approvals are acknowledged. Permit2 is not part of MVP x402 settlement. | `02_architecture/protocol_layer.md`; `02_architecture/implementation_spec.md` §6; `CANONICAL_DECISIONS.md` D4 |
| **H-04** | GMX had dead automated paths in the waterfall, Router, and Policy. | **Fixed.** D2 makes GMX manual-only. Automated GMX waterfall, router, policy, and compensation branches are removed; read-only monitoring remains. | `CANONICAL_DECISIONS.md` D2; `03_engines/engine6_execution_intelligence.md`; `02_architecture/execution_router.md`; `02_architecture/policy_engine.md` |
| **H-05** | Non-USDC bridge was a runtime branch with no MVP adapter. | **Fixed.** D3 restricts MVP bridging to USDC/CCTP. Non-USDC or unsupported bridges fail closed with `UNSUPPORTED_BRIDGE`; `BRIDGE_AGGREGATOR` is removed from the runtime enum. | `CANONICAL_DECISIONS.md` D3; `03_engines/engine6_execution_intelligence.md`; `02_architecture/protocol_layer.md`; `02_architecture/orchestrator_planner.md` |
| **H-06** | HHI bands and blocking rules conflicted across Engines 1, 2, and 4. | **Fixed.** T1 fixes `Σ(share²) × 10,000`, the `<1,500` / `1,500–2,500` / `2,500–5,000` / `>5,000` bands, and blocks new placements above 2,500. Engine 2 retains its 0–39/40–59/60–79/80–100 risk bands; HHI is a separate concentration metric. | `CANONICAL_DECISIONS.md` T1; `03_engines/engine1_portfolio_intelligence.md`; `03_engines/engine2_risk_intelligence.md`; `03_engines/engine4_strategy_optimization.md` |
| **H-07** | TOPSIS criteria, weights, and APY window conflicted. | **Fixed.** T2 fixes five criteria, Engine 3 persona weights, a 7-day TWAP, and `twapApy7dPercent`. The technical reference no longer defines a competing six-criterion model. | `CANONICAL_DECISIONS.md` T2; `03_engines/engine3_yield_opportunity.md`; `02_architecture/technical_reference.md` |
| **H-08** | Oracle, cache, and TTL rules conflicted. | **Fixed.** T4 sets Chainlink primary plus Pyth fallback on EVM, Pyth primary on Solana, 60-second staleness blocks, 1.5% cross-check, CoinGecko display only, and 5% divergence as a warning. T6 sets prices 30s, positions 30s, pools 300s, protocol safety 24h, historical 24h, metadata 7d, with stale-while-revalidate. | `CANONICAL_DECISIONS.md` T4, T6; `02_architecture/financial_knowledge_layer.md`; `02_architecture/technical_reference.md` |
| **H-09** | KYC enums and tier basis differed between identity, policy, and audit. | **Fixed.** T8 fixes `PENDING`, `APPROVED`, `REJECTED`, `RESUBMISSION_REQUESTED`, `SUSPENDED`, `EXPIRED`; tiers $1,000/$10,000/unlimited; all checks run; Travel Rule is Phase 3. | `CANONICAL_DECISIONS.md` T8; `02_architecture/identity_kyc.md`; `01_canonical/canonical_schemas.md` |
| **H-10** | Identity anchor and user schema mismatched. | **Fixed.** T9 makes Google `sub` the unique identity anchor, email mutable, and adds `kycStatus`/`kycTier`. JWT sessions and wallet-signature challenges are specified. | `CANONICAL_DECISIONS.md` T9; `02_architecture/user_onboarding.md`; `02_architecture/implementation_spec.md` §2–3 |
| **H-11** | Both pinned LLMs were deprecated. | **Fixed.** D8 selects Gemini 3.8 Flash as primary and Gemini Flash-Lite as fallback, with config-driven model IDs. | `CANONICAL_DECISIONS.md` D8; `03_engines/engine5_user_intelligence.md` |
| **H-12** | Engine 3 → Engine 4 dependency and event contract disagreed. | **Fixed.** Engine 4 declares `YieldOpportunitySnapshot` as a dependency and waits for both risk and yield outputs; the scheduler and event graph are aligned. | `03_engines/engine4_strategy_optimization.md`; `02_architecture/system_architecture.md`; `03_engines/engine3_yield_opportunity.md` |
| **H-13** | Audit taxonomy and ownership conflicted. | **Fixed.** T10 splits `category` and `action`; categories are AUTH, KYC, EXECUTION, PREFERENCE, PAYMENT, SYSTEM; KYC actions are complete; `eventId` is idempotent through `audit_event_dedupe`; audit ownership is explicit. | `CANONICAL_DECISIONS.md` T10; `02_architecture/audit_layer_x402.md`; `01_canonical/canonical_schemas.md` §13; `02_architecture/implementation_spec.md` §3 |
| **H-14** | Indexer polling, timeout, and degradation behaviour differed. | **Fixed.** T7 sets Solana 500ms, EVM 1500ms, 10-second maximum wait, `PARTIAL` on lag, provider signature verification, deduplication, and a 10-minute reconciliation sweep. | `CANONICAL_DECISIONS.md` T7; `02_architecture/portfolio_state_manager.md`; `02_architecture/technical_reference.md` |

### 3.3 Medium findings

| ID | Finding | Disposition | Evidence |
|---|---|---|---|
| **M-01** | Engine 2/3 were described as both parallel and sequential. | **Fixed.** Engine 1 wakes Engine 2 and Engine 3 in parallel; Engine 4 consumes both outputs. The architecture graph and engine triggers agree. | `02_architecture/system_architecture.md`; `03_engines/engine1_portfolio_intelligence.md`; `03_engines/engine2_risk_intelligence.md`; `03_engines/engine3_yield_opportunity.md` |
| **M-02** | Cooldown was duplicated and mislabelled as policy Check 9. | **Fixed.** D6 makes the nine policy checks canonical and moves cooldown to Engine 4 as an economic gate. Policy Check 9 is action-type restriction only. | `CANONICAL_DECISIONS.md` D6, T5; `02_architecture/policy_engine.md`; `03_engines/engine4_strategy_optimization.md` |
| **M-03** | Drift tolerance and urgency overlapped at 2–3%. | **Fixed.** T5 sets tolerance at 3% for Capital Preservation/Conservative, 5% for Balanced, 7% for Growth/Yield Seeker; Engine 4 owns drift and cooldown; HIGH urgency bypasses cooldown. Engine 4 retains its 2–7%/7–15%/>15% urgency tiers as a separate classification. | `CANONICAL_DECISIONS.md` T5; `03_engines/engine4_strategy_optimization.md`; `03_engines/engine5_user_intelligence.md` |
| **M-04** | CCTP timing and fee prose was imprecise. | **Fixed.** D5 describes Standard with `maxFee=0` and threshold 2000, Fast with a quoted fee and threshold 1000, and a fee cap of 0.10% or $5, whichever is lower. Fees are deducted at mint; expected received amount is exposed. | `CANONICAL_DECISIONS.md` D5; `03_engines/engine4_strategy_optimization.md`; `02_architecture/execution_router.md`; `02_architecture/portfolio_state_manager.md` |
| **M-05** | Base gas formula omitted operator fee; Arbitrum parameters were underspecified. | **Fixed.** `implementation_spec.md` §5 uses Base `getL1Fee`/`getL1FeeUpperBound` plus `getOperatorFee`, and Arbitrum `NodeInterface.gasEstimateL1Component` plus `ArbGasInfo`. CCE references the same formulas. | `02_architecture/implementation_spec.md` §5; `02_architecture/cross_chain_ecosystem.md` |
| **M-06** | HyperVerge contract and claims were unverified. | **Closed (gated).** D9 makes HyperVerge conditional on a two-week sandbox/webhook spike. The docs no longer assert unverified payload or signature specifics; Persona and Sumsub are recorded fallbacks. The spike is a pre-build condition, not an open audit finding. | `CANONICAL_DECISIONS.md` D9; `02_architecture/identity_kyc.md`; `02_architecture/user_onboarding.md`; `02_architecture/implementation_spec.md` §10 |
| **M-07** | Literature survey contained count, citation, and attribution errors. | **Fixed.** 54 unique papers; duplicate removed; Barber & Odean split into two works; Kirişci journal corrected; Heimbach attribution corrected; ASRI/Qin marked unverified; bibliography replaced; CSV rebuilt with 54 unique titles and DOI/identifier/verification columns. | `05_research_and_audits/literature_survey.md`; `05_research_and_audits/CrestFlow Literature Survey Papers List.csv` |
| **M-08** | Integration report falsely claimed zero unresolved contradictions. | **Fixed.** The report is marked Historical; the claim is withdrawn; it points to this report for disposition. | `05_research_and_audits/integration_audit_report.md` |
| **M-09** | Architecture assets were stale, mislabelled, or linked by absolute paths. | **Fixed.** Current image is `system-architecture-post-pivot.png`; pre-pivot and legacy images are explicitly historical; links are relative; no `file:///` path remains. | `02_architecture/system_architecture.md`; `assets/` |
| **M-10** | `poolType` ownership and implementation count conflicted. | **Fixed.** T11 establishes one `@crestflow/pool-type` package used by engines, Router, Policy, and the Protocol Layer. | `CANONICAL_DECISIONS.md` T11; `02_architecture/system_architecture.md`; `03_engines/engine4_strategy_optimization.md`; `03_engines/engine6_execution_intelligence.md` |
| **M-11** | Provider plans, rate limits, and SLA behaviour were unspecified. | **Fixed.** `implementation_spec.md` §10 defines CoinGecko, DeFiLlama, Alchemy, Helius, Turnkey, HyperVerge, and Circle Iris limits and failure behaviour; FKL links to it. | `02_architecture/implementation_spec.md` §10; `02_architecture/financial_knowledge_layer.md` |
| **M-12** | Aggregator API and order-lifecycle assumptions were outdated. | **Fixed.** T12 removes 1inch, UniswapX, Curve, and Camelot from MVP. Jupiter uses `/order` + `/build`; UniswapX order lifecycle is Phase 2. Adapter roster and allowlist agree. | `CANONICAL_DECISIONS.md` T12; `02_architecture/execution_router.md`; `02_architecture/protocol_layer.md` |
| **M-13** | FATF Travel Rule was overstated as implemented. | **Fixed.** Travel Rule is Phase 3. Allowlist curation is labelled sanctions screening, not Travel Rule data exchange. The technical reference is corrected. | `CANONICAL_DECISIONS.md` T8; `02_architecture/identity_kyc.md`; `02_architecture/technical_reference.md` |
| **M-14** | API, database, nonce, and observability specifications were missing. | **Fixed.** `implementation_spec.md` adds API/OpenAPI 3.1, persistence schema, nonce manager, gas/fee formulas, saga and compensation, CCTP lifecycle, security threat model, provider contracts, and SLOs. | `02_architecture/implementation_spec.md` |
| **M-15** | No source-of-truth or versioning discipline. | **Fixed.** T14 defines precedence; canonical documents have status/version headers; `00_DOCUMENT_INDEX.md` is updated; the old UML pack is historical. | `CANONICAL_DECISIONS.md` T14; `00_DOCUMENT_INDEX.md`; document headers |

### 3.4 Low findings

| ID | Finding | Disposition | Evidence |
|---|---|---|---|
| **L-01** | Router said “Does not route.” | **Fixed.** The router role is now “select venue, pool, split, calldata route, and CCTP parameters before byte construction.” The contradictory sentence is gone. | `02_architecture/execution_router.md` |
| **L-02** | Absolute or broken asset links. | **Fixed.** Only relative links remain; assets are renamed; the stale `.excalidraw` and absolute OneDrive path are removed; three `../docs/assets` links were corrected to `../assets`. | `02_architecture/system_architecture.md`; `assets/` |
| **L-03** | CSV contained duplicate and section-header rows. | **Fixed.** 54 paper rows, no section headers, 54 unique titles, DOI/identifier and verification columns; 2 entries marked unverified. | `05_research_and_audits/CrestFlow Literature Survey Papers List.csv`; `literature_survey.md` §15.3 |
| **L-04** | Encoding artifacts and typos. | **Fixed.** No U+FFFD replacement characters remain; the “? shared state” artifact is corrected to “≠ shared state.” | Repository-wide encoding scan; `02_architecture/system_architecture.md` |

### 3.5 Narrative cross-reference IDs

The audit register has 39 rows. Its detailed narrative also uses **M-16**, **M-17**, and **M-33** as grouped labels for related operational findings. They are cross-referenced here so no audited issue is lost.

| ID | Grouped finding | Disposition | Evidence |
|---|---|---|---|
| **M-16** | Simulation method, nonce manager, Solana re-sign handoff, queued-state handling, webhook signature/idempotency, and audit event versioning. | **Fixed under M-14, H-14, T7, and T10.** The Planner specifies `eth_call` with state overrides plus `eth_estimateGas`, and separately validates nonce/gas/chain ID/deadline/envelope. CCE owns nonce allocation and re-sign. `queued` is distinct from `PENDING_CONFIRMATION`. Webhooks require provider signatures, dedupe, and sweep. Audit has `schemaVersion` and `eventId`. | `02_architecture/orchestrator_planner.md`; `02_architecture/implementation_spec.md` §4, §7, §8; `02_architecture/portfolio_state_manager.md`; `02_architecture/audit_layer_x402.md` |
| **M-17** | GMX async compensation and queued downstream handling. | **Fixed under D2 and M-14.** GMX is manual-only; `AWAITING_FULFILLMENT` is retained only for future automation and is unreachable in MVP. Queued transactions are tracked separately and notified; CCTP timeout/manual recovery is specified. | `CANONICAL_DECISIONS.md` D2; `02_architecture/orchestrator_planner.md`; `03_engines/engine6_execution_intelligence.md` |
| **M-33** | Engines were said to never call RPC, while Engine 2 was said to poll Aave/Kamino/MarginFi. | **Fixed under M-11 and T4.** FKL is the only external data path. Engine 2 makes zero external adapter calls and consumes FKL-supplied protocol-safety scores. | `02_architecture/system_architecture.md`; `03_engines/engine2_risk_intelligence.md`; `02_architecture/financial_knowledge_layer.md` |

---

## 4. Re-audit evidence

### 4.1 Structural checks

| Check | Method | Result |
|---|---|---|
| Decision coverage | Map every C/H/M/L finding to a decision or canonical document | 39/39 mapped; no finding without an owner |
| Critical/High closure | Re-read each disposition against the current canonical text | **0 open Critical; 0 open High** |
| Policy catalog | Search for a second nine-check list outside `policy_engine.md` | Only policy_engine defines the checks; other docs link or summarise |
| Execution order | Search for Policy-before-build and Router-after-build statements | One order: E6 → Router → Planner/build → Simulation → Policy → CCE → PSM |
| Schema names | Search for `compositeRiskScore`, `twapApy30dPercent`, `SimulationResult`, `KYC_PENDING`, `PROFILE_CHANGE` | Remaining occurrences are removal statements, compatibility mappings, or historical audit findings |
| Bridge scope | Search for `BRIDGE_AGGREGATOR` and non-USDC runtime branches | Only removal statements and fail-closed mappings remain |
| x402 V2 | Check headers, network, asset, scheme, facilitator | V2 headers, Base USDC, `eip155:8453`, `exact`, official facilitator |
| CCTP V2 | Check calldata parameters and recipient handling | `destinationCaller`, `maxFee`, `minFinalityThreshold`; recipient parameter; USDC only |
| KYC/audit enums | Compare identity, policy, audit, schemas | Same statuses, tiers, categories, and actions |
| HHI/TOPSIS/oracle/cache | Compare engines and technical reference | One formula, one criteria set, one TTL table, one oracle precedence |
| Links and encoding | Scan 62 relative Markdown links in 31 Markdown files, plus replacement characters | 0 broken links; 0 U+FFFD |
| Diagram syntax | Render every `.mmd` with Mermaid CLI | 19/19 render successfully with `@mermaid-js/mermaid-cli` 11.17.1; 19/19 PNG exports present |

### 4.2 Stale-token scan

The following terms still appear only where the document is explicitly recording a removal, a compatibility mapping, or a historical audit finding:

- `compositeRiskScore` — canonical replacement notice and mapping table.
- `twapApy30dPercent` — canonical replacement notice and mapping table.
- `X-Payment` / `goplusfable` — removal notice and old-to-new mapping.
- `BRIDGE_AGGREGATOR` — removal notice and fail-closed mapping.
- `ERC-4337` / `SPL delegate` — explicit exclusion from MVP and Phase 2 note.
- `PROFILE_CHANGE` — removal notice.
- `Does not route` — only “the Planner does not route,” which is the correct responsibility boundary.

No current contract still uses those terms as an active design.

### 4.3 Diagram validation

All 19 `.mmd` files render without a parser error. The pack was checked for stale pre-remediation details:

- Policy sequence uses nine checks, not three.
- Onboarding uses canonical KYC states, not `KYC_PENDING`.
- Execution sequence uses the D7 order, not Policy-before-build.
- Class diagram omits `SimulationResult`.
- CCTP sequence uses V2 parameters.
- x402 sequence uses V2 headers and the official facilitator.
- Saga states match `implementation_spec.md` §7.1.

---

## 5. Residual conditions and honest limitations

These are conditions, not open audit findings.

1. **HyperVerge vendor verification.** D9 requires a two-week sandbox/webhook spike before integration commitment. If it fails, the KYC adapter changes but the canonical states, tiers, and policy checks do not.
2. **External documentation drift.** Vendor APIs and model names can change. The decision record pins the current choices; production code should make model IDs and provider endpoints configurable.
3. **No source code.** This repository is a planning and architecture repository. The remediation verifies document consistency, schema contracts, and diagram syntax, not a compiled implementation.
4. **Mainnet is not approved.** Testnet release gates are defined. Mainnet remains a separate decision and security-audit gate.
5. **Gantt dates are indicative.** The chart is a planning aid, not a delivery commitment.
6. **Mermaid approximations.** Component, deployment, use-case, and DFD views use flowchart shapes. The notation mapping in `06_uml_diagrams/README.md` is part of the contract.
7. **Historical audit report.** `AUDIT_REPORT.md` intentionally retains the pre-remediation findings. It is the audit trail; this report is the disposition.

---

## 6. Readiness statement

The repository is ready to serve as the **BE report source and UML baseline**. The audit's P0 and P1 gates are closed. The current diagram pack is abstract, code-first, and traceable to canonical decisions.

It is **not** a claim that the system is ready for mainnet. Implementation, testnet execution, security review, and the mainnet release gate remain future work.

**Recommended next action:** review the 19 diagrams in `documentation/06_uml_diagrams/` (Mermaid sources and PNG exports), approve or request changes, then commit the remediation and diagram pack together. Do not commit or push before that approval.

---

## 7. Change log

| Date | Change |
|---|---|
| 19 Nov 2026 | Initial remediation report. Closes all 39 audit findings; records M-06 as a gated vendor dependency; validates and exports 19 Mermaid diagrams as PNG. |
