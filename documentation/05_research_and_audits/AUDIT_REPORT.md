# CrestFlow Planning Repository — Independent Architecture & Documentation Audit

**Audit date:** 18 November 2026  
**Repository:** `D:\crestflow pivot`  
**Scope:** All 26 files in the repository (22 Markdown, 1 CSV, 3 PNG). No source code exists.  
**Audit type:** Cross-document consistency, external-integration validity, architectural feasibility, specification completeness, and readiness for a BE project report and UML documentation.  
**Change status:** No existing file was modified, deleted, renamed, or consolidated. This report is the only new file. No UML diagrams were produced.  
**Current status:** Pre-remediation audit snapshot. This document records the repository **as audited before remediation**; it is the finding register, not the current readiness verdict. The current disposition of every finding is in `REMEDIATION_REPORT.md`.

---

## 0. Verdict at a glance

**Overall verdict: NOT READY** to serve as an implementation contract, a BE project report source of truth, or a UML modelling baseline without remediation.

The repository has strong domain decomposition and unusually detailed financial/execution reasoning. The component boundaries are generally sensible: six intelligence engines, a Financial Knowledge Layer (FKL), a deterministic Policy Engine, an Orchestrator/Planner, an Execution Router, a Protocol Layer, a Cross-Chain Ecosystem (CCE), a Portfolio State Manager (PSM), and an Audit/x402 layer.

However, the documents do **not** agree on the core execution contract. The most serious problems are not spelling or formatting issues; they are incompatibilities that would cause:

- a policy gate that cannot be implemented because two different “9 checks” exist and two checks reference fields that do not exist;
- an execution sequence that is circular (Policy evaluates a simulation package that is produced after Policy in one version of the flow);
- delegated “Path B” execution that cannot sign for Aave/Uniswap/Solana positions as described;
- a CCTP adapter that uses legacy V1 calldata while claiming V2 support, and that hard-codes the mint recipient to the embedded wallet;
- an x402 payment flow that uses obsolete headers and a transaction-hash model incompatible with the official `exact` scheme; the named facilitator host is unreachable.

These are **P0 blockers**. Starting UML sequence/class modelling of execution before they are resolved would encode contradictions into the design.

### Finding counts

| Severity | Count | Meaning |
|---|---:|---|
| Critical (C) | 6 | Prevents a correct or safe implementation; must be decided before build/UML. |
| High (H) | 14 | Materially wrong integration, metric, schema, or dependency; likely rework. |
| Medium (M) | 15 | Ambiguity, stale documentation, unverified vendor claim, or operational gap. |
| Low (L) | 4 | Editorial, naming, link, or asset hygiene. |
| **Total** | **39** | |

### Top 10 findings

1. **C-01 — Execution order is circular and defined three ways.** `core_documents/system_architecture.md:60,137-142` places Policy before the Orchestrator, but Policy consumes a `SimulationPassedPackage` (`docs/policy_engine.md:11`) that the Orchestrator/Planner produces (`docs/orchestrator_planner.md:86,95-99`). `docs/technical_reference.md:456,478` says simulation is prior to Policy. The Execution Router also needs to run before transaction bytes are built, but the architecture places it after build/simulate.
2. **C-02 — The Policy Engine has two incompatible 9-check catalogs.** `docs/policy_engine.md:15-197` checks KYC, tier limit, authority, risk, chain allowlist, protocol allowlist, per-tx value, slippage, and action types. `docs/technical_reference.md:444-454` checks KYC, spend, allowlists, authority, risk freshness, persona volatility, slippage, gas/bridge payback, and cooldown. Two checks use undefined fields: `riskSnapshot.compositeRiskScore` and `goalProfile.allowedActionTypes`.
3. **C-03 — Delegated Path B is not feasible as written.** `docs/technical_reference.md:100-101`, `docs/protocol_layer.md:119-121`, and `docs/cross_chain_ecosystem.md:49` assume ERC-4337 session keys for MetaMask users and SPL `delegate` for Phantom users. MetaMask EOAs are not ERC-4337 smart accounts by default; EIP-7702 is omitted. Aave V3 has no general position delegation, Uniswap V3 LP NFTs are owner-held, and SPL token delegates cannot sign for lending/LP positions.
4. **C-04 — CCTP V2 calldata is wrong.** `docs/protocol_layer.md:134` encodes `depositForBurn(amount, destDomain, mintRecipient, burnToken)` — the V1 four-argument form. CCTP V2 requires `destinationCaller`, `maxFee`, and `minFinalityThreshold`. The destination mint is hard-coded to the embedded wallet (`:146`), which breaks delegated users.
5. **C-05 — x402 is incompatible with the current V2 protocol.** `docs/audit_layer_x402.md:277-287,319` and `docs/technical_reference.md:632` use `X-Payment` / `X-Payment-Authorization`, a transaction-hash model, and `https://api.goplusfable.xyz/facilitate`. V2 uses `PAYMENT-REQUIRED` / `PAYMENT-SIGNATURE` / `PAYMENT-RESPONSE`; the `exact` EVM scheme uses EIP-3009 `transferWithAuthorization` (or Permit2), not a tx hash; the official facilitator example is `https://x402.org/facilitator`.
6. **C-06 — `ExecutionIntent` cannot be executed or validated from its schema.** `docs/engine6_execution_intelligence.md:196-221` omits protocol, pool/market, source address, recipient/domain, approval spender/amount, and deadline. `amount` is a decimal string, while `docs/technical_reference.md:645` mandates integer base units. The Router and Policy Engine must infer or re-derive critical fields.
7. **H-01 — The Execution Router is both after simulation and “does not route.”** `docs/execution_router.md:9-13` says it produces route-resolved instructions, then says “Does not route.” `core_documents/system_architecture.md:139-140` runs Orchestrator build+simulate before Router, although bytes cannot be built before route selection.
8. **H-02 — Allowlist, adapter roster, and router venues disagree.** `docs/policy_engine.md:107-114` omits `1inch` and `UniswapX` but includes `camelot`; `docs/execution_router.md:23-25` routes through `1inch`/`UniswapX`; `docs/protocol_layer.md:29-36` has a combined `OneInchUniswapXAdapter`; no Camelot adapter exists.
9. **H-09/H-13 — KYC and audit taxonomies disagree.** `docs/identity_kyc.md:49` uses `DECLINED`; `docs/technical_reference.md:113` uses `REJECTED`. `docs/identity_kyc.md:97` writes `KYC_REVOKED` under `PROFILE_CHANGE`, but the canonical audit table has no KYC category and `docs/technical_reference.md:610` lists `PREFERENCE_UPDATE`, not `PROFILE_CHANGE`.
10. **H-11 — Both pinned LLMs are deprecated.** `docs/engine5_user_intelligence.md:138-139` pins `GPT-4.1-mini` and `Gemini 2.5 Flash`. As of 18 Nov 2026, OpenAI lists `gpt-4.1-mini` as deprecated, and Google lists `gemini-2.5-flash` as deprecated with automatic routing to `gemini-3.5-flash-lite` after 1 Dec 2026.

---

## 1. Scope, method, and evidence standard

### 1.1 What was audited

- Every Markdown file under `core_documents/` and `docs/` (22 files).
- The literature CSV (64 data rows).
- All three PNG architecture assets (visual inspection).
- No source code, build files, tests, API contracts, database migrations, or deployment manifests exist in the repository.

### 1.2 Method

1. **Full read** of each document, not keyword-only sampling.
2. **Contract extraction:** producers/consumers, schemas, event names, field names, ordering, thresholds, and ownership.
3. **Cross-document comparison** at field and step level.
4. **External verification** against official documentation for every proposed protocol, SDK, API, chain primitive, and vendor that materially affects execution.
5. **Feasibility check** against the actual authority model of Aave, Uniswap, SPL Token, ERC-4337, EIP-7702, CCTP, and x402.
6. **Citation audit** against Crossref/DOI metadata and the cited papers where accessible.

### 1.3 Evidence labels used in this report

| Label | Meaning |
|---|---|
| **Verified** | Confirmed against current official documentation or primary source. |
| **Partially verified** | Core capability exists, but a repo-specific claim, parameter, or contract detail is missing or inaccurate. |
| **Contradicted** | Official documentation or primary source contradicts the repo claim. |
| **Unverified** | Could not be confirmed because the official source was unreachable, rate-limited, or the claim is marketing-only. |
| **Internal conflict** | Two repo documents or two sections of the same document disagree. |

### 1.4 Limitations

- HyperVerge developer documentation (`docs.hyperverge.co`, `dev.hyperverge.co`) returned 502/transport errors; the webhook signature scheme and API version could not be verified.
- 1inch’s developer portal was unreachable; only the business documentation landing page was reachable.
- The attempted Chainlink price-feed address page returned 404; feed addresses and L2 sequencer-uptime handling remain unverified.
- Semantic Scholar returned 429/404 for some 2026 papers (ASRI/Farzulla & Maksakov, Qin et al.); those citations remain unverified rather than declared invalid.
- No runtime tests were possible because there is no implementation.
- “Current” external facts are as of **18 November 2026** and may change.

---

## 2. Complete document inventory

The repository contains **26 files**, not 24: **22 Markdown**, **1 CSV**, and **3 PNG**. The earlier working count of 24 was an inventory error; this report uses the verified filesystem count.

### 2.1 Core documents

| # | Path | Type | Lines | Bytes | Role | Audit status |
|---:|---|---|---:|---:|---|---|
| 1 | `core_documents/system_architecture.md` | Markdown | 228 | 10,017 | Post-pivot system overview, Mermaid component graph, chain topology, execution flow, event contracts, open items | Reviewed; contains critical ordering and asset-link issues |
| 2 | `core_documents/technical_reference.md` | Markdown | 222 | 18,970 | Condensed technology/pattern taxonomy across 17 domains | Reviewed; summary-level, conflicts with detailed docs |
| 3 | `core_documents/literature_survey.md` | Markdown | 604 | 70,809 | 10-domain literature review mapped to engines | Reviewed; citation errors confirmed |
| 4 | `core_documents/CrestFlow Literature Survey Papers List.csv` | CSV | 65 | 47,468 | 64 data rows; 54 URL-bearing rows | Reviewed; 53 unique titles, one duplicate |

### 2.2 Component and cross-cutting documents

| # | Path | Type | Lines | Bytes | Role | Audit status |
|---:|---|---|---:|---:|---|---|
| 5 | `docs/technical_reference.md` | Markdown | 688 | 74,237 | Master technical reference; 17 sections covering architecture through data standards | Reviewed; primary source of many conflicts |
| 6 | `docs/engine1_portfolio_intelligence.md` | Markdown | 290 | 10,436 | Portfolio snapshot, LP decomposition, exposure, HHI, WAC/PnL, health score | Reviewed; HHI band conflict |
| 7 | `docs/engine2_risk_intelligence.md` | Markdown | 299 | 10,600 | Five risk analysers, composite score, alerts, `RiskSnapshot` | Reviewed; field-name and HHI conflicts |
| 8 | `docs/engine3_yield_opportunity.md` | Markdown | 282 | 9,819 | Opportunity discovery, APY normalisation, IL, TOPSIS, idle capital | Reviewed; TOPSIS/APY conflicts |
| 9 | `docs/engine4_strategy_optimization.md` | Markdown | 290 | 10,875 | Optimisation model selection, constraints, drift/urgency, `StrategyPlan` | Reviewed; dependency/cooldown/GMX conflicts |
| 10 | `docs/engine5_user_intelligence.md` | Markdown | 229 | 9,838 | Persona model, `GoalProfile`, Copilot intents, LLM pipeline | Reviewed; undefined policy field and deprecated models |
| 11 | `docs/engine6_execution_intelligence.md` | Markdown | 241 | 10,110 | Authority filter, sourcing waterfall, typed DAG, `ExecutionIntent` | Reviewed; schema and GMX issues |
| 12 | `docs/orchestrator_planner.md` | Markdown | 209 | 10,151 | Build/simulate, `SimulationPassedPackage`, saga state machine, CCTP decomposition | Reviewed; central ordering conflict |
| 13 | `docs/policy_engine.md` | Markdown | 236 | 9,381 | Authoritative 9-check implementation spec, fail-closed behaviour | Reviewed; conflicts with master reference |
| 14 | `docs/execution_router.md` | Markdown | 205 | 9,184 | Aggregation, split routing, protocol selection, gas batching, CCTP parameters | Reviewed; ordering and allowlist conflicts |
| 15 | `docs/protocol_layer.md` | Markdown | 227 | 9,478 | Adapter roster, `UnsignedTransaction`, CCTP schema, Permit2, authority mode | Reviewed; CCTP V1/V2 and Permit2 errors |
| 16 | `docs/cross_chain_ecosystem.md` | Markdown | 243 | 9,648 | TEE signing, broadcast, gas strategy, confirmation, CCTP status | Reviewed; delegation and fee issues |
| 17 | `docs/portfolio_state_manager.md` | Markdown | 194 | 8,217 | Post-execution sync, webhook subscriptions, bridge monitoring | Reviewed; indexer and fee conflicts |
| 18 | `docs/portfolio_state_analysis.md` | Markdown | 143 | 6,952 | Address model, scan geometry, cross-chain price unification, cost basis | Reviewed; delegation overstatement |
| 19 | `docs/financial_knowledge_layer.md` | Markdown | 212 | 10,481 | Adapters, cache, normaliser, dual-source pricing, registry | Reviewed; TTL/oracle/rate-limit issues |
| 20 | `docs/identity_kyc.md` | Markdown | 106 | 6,248 | HyperVerge flow, tier assignment, KYC gate, AML phase | Reviewed; enum/tier/audit conflicts |
| 21 | `docs/user_onboarding.md` | Markdown | 154 | 9,287 | OAuth, Path A/B, Turnkey provisioning, deposit/delegate, sessions | Reviewed; identity schema and missing KYC step |
| 22 | `docs/audit_layer_x402.md` | Markdown | 358 | 16,114 | 11 audit categories, lifecycle events, schema, x402 gateway | Reviewed; taxonomy and x402 errors |
| 23 | `docs/integration_audit_report.md` | Markdown | 124 | 9,150 | Claims 17 docs audited with zero contradictions | Reviewed; claim is false against current files |
| 24 | `docs/assets/system-architecture after pivot.png` | PNG | — | 488,093 | Labelled post-pivot architecture | Inspected; stale pre-pivot content |
| 25 | `docs/assets/system-architecture before pivot.png` | PNG | — | 3,823,760 | Pre-pivot historical diagram | Inspected; matches pre-pivot scope |
| 26 | `docs/assets/Untitled-2026-06-22-2109.png` | PNG | — | 869,586 | Actual post-pivot diagram | Inspected; matches current textual flow |

### 2.3 Inventory-level observations

- The repository has **no declared document precedence**. `docs/technical_reference.md` reads as a master reference, but `docs/policy_engine.md` is more implementation-specific and contradicts it. `docs/integration_audit_report.md` asserts resolution without line-level evidence.
- The asset filenames are misleading: `system-architecture after pivot.png` is pre-pivot; `Untitled-2026-06-22-2109.png` is the real post-pivot diagram.
- `core_documents/system_architecture.md:9-11` links to a non-existent `.excalidraw` file and uses absolute `file:///C:/Users/omen/OneDrive/...` paths, so the links break outside the author’s machine.

---

## 3. Document-wise findings

Each subsection records what the document does well, the material findings, and the severity contribution. Finding IDs refer to the register in §8.

### 3.1 Core documents

#### 3.1.1 `core_documents/system_architecture.md`

**Strengths**

- Clear post-pivot scope: Ethereum, Base, Arbitrum, Solana; explicitly rejects the Algorand/Folks/Tinyman legacy stack.
- Good component decomposition and a readable Mermaid graph (`:14-70`).
- Useful chain/protocol matrix (`:91-94`) and event-contract summary (`:159-168`).
- Open-items table (`:220-226`) is honest about unresolved areas such as non-USDC bridging, nonce ownership, and GMX compensation.

**Findings**

1. **C-01 — Circular execution order.** The graph says `E6 --> PE --> ORC` (`:60`), but the Policy Engine is specified to evaluate a `SimulationPassedPackage` (`docs/policy_engine.md:11`) that is built by the Orchestrator/Planner (`docs/orchestrator_planner.md:86,95-99`). The prose flow at `:137-142` is `Engine 6 → Policy Engine → Orchestrator (build + simulate) → Execution Router`; this is incompatible with policy evaluating a completed simulation package. The same flow also puts route selection after transaction construction, which is impossible for aggregator routes.
2. **H-01 — Router ordering.** `:139-140` places `Orchestrator (build + simulate all steps)` before `Execution Router (route selection, CCTP params)`. A swap transaction cannot be built or simulated until the route, pool, and calldata are known.
3. **M-01 — Engine pipeline contradiction.** The graph is strictly sequential `E1 --> E2 --> E3 --> E4` (`:56`), but the prose says Engine 2 is “parallel with Engine 3” (`:128`). Engine 1 itself says it wakes Engine 2 and Engine 3 in parallel (`docs/engine1_portfolio_intelligence.md:268`).
4. **M-10 — `poolType` ownership.** `:113` recommends a shared library, `:222` says it is “currently reimplemented independently by Engines 1-4, Policy Engine, Execution Router, Protocol Layer,” while Engine 4 and Engine 6 both call themselves “the fourth engine requiring it” (`docs/engine4_strategy_optimization.md:289`, `docs/engine6_execution_intelligence.md:241`). The count and ownership are inconsistent.
5. **M-09 — Asset links.** `:9-11` links to `docs/assets/crestflow-architecture-overview.excalidraw` (does not exist), `system-architecture after pivot.png` (stale pre-pivot content), and an absolute `file:///C:/Users/omen/OneDrive/...` path (not portable).
6. **M-12 — Aggregator scope.** The protocol matrix (`:91-93`) lists `1inch/UniswapX`, but the Policy Engine allowlist omits both, and the Protocol Layer has a single combined adapter rather than the distinct UniswapX order flow.
7. **H-05 — Non-USDC bridge.** `:223` says non-USDC cross-chain is Phase 2, but runtime documents already branch on `BRIDGE_AGGREGATOR` (`docs/engine6_execution_intelligence.md:210`; `docs/orchestrator_planner.md:196`).
8. **M-33 (new) — Data-access contradiction.** `:185` states “Engines 1–3 and the Protocol Layer never make direct protocol RPC calls,” but `docs/technical_reference.md:232` says Engine 2 “continuously polls Aave V3 (and Kamino/MarginFi on Solana).” Either the engine polls directly or it consumes FKL; the documents disagree.
9. **Low — encoding artifact.** `:83` renders “Identical EVM address ? shared state” where the intended relation is “≠”. This is cosmetic but obscures an important invariant.

**Severity contribution:** C-01, H-01, H-05, M-01, M-09, M-10, M-12.

#### 3.1.2 `core_documents/technical_reference.md`

**Strengths**

- Compact, useful taxonomy of the technical surface: architecture, auth, KYC, FKL, six engines, Policy, orchestration, routing, adapters, PSM, audit/x402, numeric standards.
- Good high-level articulation of the fail-closed philosophy (`:25`) and the 4-chain, multi-VM substrate (`:41`).
- Useful as a navigation index, provided it is not treated as an implementation contract.

**Findings**

1. **C-02 — Different 9-gate list.** `:155` lists KYC, spend limits, allowlists, authority, staleness, bounds, slippage, payback, and cooldown. This matches `docs/technical_reference.md:444-454`, not `docs/policy_engine.md:15-197`. The repository therefore has two “authoritative” policy catalogs.
2. **H-08 — Oracle threshold.** `:60` says Chainlink/Pyth divergence above **1.5%** halts the pipeline; `docs/financial_knowledge_layer.md:186` says CoinGecko vs on-chain oracle divergence above **5%** halts execution. These may be two different guards, but no document reconciles scope, ordering, or which one actually blocks signing.
3. **H-03 — Permit2 overstatement.** `:189` claims “Permit2 & EIP-2612 Approvals” eliminate separate on-chain allowance transactions. Permit2 itself still requires a one-time on-chain approval of the Permit2 contract, and Aave V3 does not natively support Permit2.
4. **C-03 — Delegation overstatement.** `:41` and `:198-199` present ERC-4337 session keys and SPL `delegate` as general-purpose delegated execution. They are not sufficient for Aave/Uniswap positions or program-owned Solana positions.
5. **M-13 — FATF mislabel.** `:52` and `:157` label “FATF Travel Rule Compliance” / “FATF Travel Rule Curated Allowlist” as implemented mechanisms. `docs/identity_kyc.md:89` defers the Travel Rule to **Phase 3**, and the described allowlist is sanctions screening, not Travel Rule originator/beneficiary data exchange.
6. **M-04 — CCTP “1:1” claim.** `:200` calls CCTP “slippage-free 1:1.” Standard transfers are 1:1 with no protocol fee, but Fast transfers deduct a route-specific fee at mint, so the received amount can be less than the burned amount.
7. **M-14 — 11 categories without a list.** `:219` asserts 11 audit categories but does not enumerate them; the enumerations in `docs/audit_layer_x402.md:16-30` and `docs/technical_reference.md:610` do not match.
8. **H-07 — Single hard slippage cap.** `:159` imposes a hard `>0.5%` deviation cap, while `docs/technical_reference.md:452` allows 0.5% standard / 1.0% high-volatility, and `docs/policy_engine.md:155-158` uses `goalProfile.maxSlippagePercent`. Three different slippage rules coexist.

**Severity contribution:** C-02, C-03, H-03, H-07, H-08, M-04, M-13, M-14.

#### 3.1.3 `core_documents/literature_survey.md`

**Strengths**

- Broad and well-organised: ten domains, explicit mapping to CrestFlow engines, and a useful attempt to justify design choices with academic literature.
- The survey distinguishes foundational references from the 54-paper core and provides structured profiles.
- Several 2026 citations are real and technically relevant (see §11).

**Findings**

1. **M-07 — “54 papers” is not 54 unique papers.** The survey claims 54 core papers at `:14`, `:117`, `:148`, and `:162`. The CSV has 54 URL-bearing rows but only **53 unique titles**: “Optimal Portfolios Of Liquidity Positions” appears at `:544` and again at `:573` with a variant subtitle and the same Semantic Scholar URL. The count should be corrected or the duplicate justified as two distinct papers.
2. **M-07 — Kirişci journal is wrong.** `:542` cites “Complex & Intelligent Systems (Springer)” for DOI `10.1007/s00521-024-10839-2`. Crossref resolves that DOI to **Neural Computing and Applications**. The paper itself may be valid; the journal attribution is not.
3. **M-07 — Barber & Odean conflation.** `:581` cites “The Courage of Misguided Convictions: The Trading Behavior of Individual Investors” as *The Journal of Finance* 55(2), 773–806. That journal/volume/page range belongs to “Trading Is Hazardous to Your Wealth” (DOI `10.1112/0022-1082.00226`). “The Courage of Misguided Convictions” is *Financial Analysts Journal* 55(6), 1999 (DOI `10.2469/faj.v55.n6.2313`). The survey merges two papers.
4. **M-07 — Heimbach misattribution and overstatement.** `:226` says Heimbach, Schertenleib, & Wattenhofer (2022) “prove that over 50% of active LPs experience net negative returns relative to simple holding.” The “around 50% lose money” finding belongs to Loesch et al., which Heimbach cites; Heimbach’s own conclusion is more nuanced — with appropriate considerations, Uniswap V3 LP can be profitable. The survey turns a cited result into the paper’s conclusion and states it more strongly than the source.
5. **M-07 — Broken or unverified links.** Semantic Scholar links for Arora 2026 and TxLens 2026 return 404. ASRI/Farzulla & Maksakov 2026 and Qin et al. 2026 could not be verified due to 429/404 responses. These should be marked “unverified” rather than presented as confirmed metadata.
6. **M-07 — DOI/link hygiene.** Some entries provide only Semantic Scholar URLs, some provide publisher PDFs, and some provide no DOI. A consistent `DOI | publisher | open-access URL | verification date` column is needed.
7. **Low — overclaiming language.** Phrases such as “proves” and “direct mathematical justification” (`:171`, `:226`) overstate what individual papers establish and make the survey harder to defend in a BE report.

**Severity contribution:** M-07.

#### 3.1.4 `core_documents/CrestFlow Literature Survey Papers List.csv`

**Verified inventory**

- 65 lines total: 1 header + 64 data rows.
- 54 rows contain an `http(s)` URL.
- 10 rows are section headers without URLs.
- 53 unique paper titles; one duplicate (“Optimal Portfolios Of Liquidity Positions”).
- Header fields: `Title, Authors, URL, Published Year, Topic / Focus, Method / Approach, Relevance, Section / Subsection, Summary`.

**Findings**

1. **M-07/L-03 — Count and duplicate.** The “54 papers” claim counts URL-bearing rows, not unique papers. Either deduplicate to 53 or explain why the two Kroo et al. entries are distinct works.
2. **L-03 — Section headers are rows.** Ten rows are structural headings, not papers; any script that counts rows will overcount.
3. **M-07 — No DOI column.** URL-only identification is fragile; several links are already dead.
4. **M-07 — Inconsistent metadata.** Author-name order, year, venue, and title casing vary; this makes automated bibliography generation unreliable.

**Severity contribution:** M-07, L-03.

---

### 3.2 Engine specifications

#### 3.2.1 `docs/engine1_portfolio_intelligence.md`

**Strengths**

- Clear ten-step pipeline (`:13-39`) with explicit formulas for LP decomposition, true exposure, HHI, WAC, returns, and health score.
- Good treatment of cross-chain double counting and per-chain drill-down (`:133-177`).
- Explicitly excludes GMX GM pools from impermanent-loss math (`:106`), which is correct for the GMX model.
- Honest open item on the linear-vs-step HHI penalty curve (`:290`).

**Findings**

1. **H-06 — HHI bands conflict.** `:228-230` uses `<1,500` / `1,500-2,500` / `>2,500` for the chain-concentration penalty. Engine 2 uses `<1,500` / `1,500-2,500` / `2,500-5,000` / `>5,000` (`docs/engine2_risk_intelligence.md:90-93`). Engine 4 blocks placements above 2,500 (`docs/engine4_strategy_optimization.md:135-139`). A portfolio at HHI 3,000 is “high but below critical” in Engine 2, “highly concentrated” in Engine 1, and “blocked” in Engine 4.
2. **M-01 — Parallel emission.** `:268` says `PortfolioSnapshotCreated` wakes Engine 2 and Engine 3 in parallel; the system graph and master reference describe a sequential cascade.
3. **M-15 — Snapshot completeness.** The snapshot schema (`:250-266`) does not carry an explicit `dataQuality`/`partial` flag, although the PSM and technical reference rely on `dataQuality: 'PARTIAL'` after indexer lag (`docs/technical_reference.md:597`). Downstream engines cannot tell whether a snapshot is complete.
4. **M-03 — Tolerance duplication.** Engine 1 exposes a `hhi` block but no authoritative `driftTolerancePercent`; Engine 5 defines that field, Engine 4 hard-codes the bands. The ownership is unclear.

**Severity contribution:** H-06, M-01, M-15.

#### 3.2.2 `docs/engine2_risk_intelligence.md`

**Strengths**

- Five well-separated analysers (market, liquidation, concentration, protocol, liquidity) with explicit formulas and thresholds.
- Correct exclusion of GMX GM positions from the liquidation analyser (`:255`) and inclusion of correlated cross-chain liquidation risk.
- Clear alert lifecycle and a reasonably complete `RiskSnapshot` schema (`:230-289`).
- Persona multipliers are documented and tied to Engine 5 (`:186-204`).

**Findings**

1. **C-02 — `RiskSnapshot` has `riskScore`, not `compositeRiskScore`.** `:283` defines `riskScore`. `docs/engine4_strategy_optimization.md:25`, `docs/policy_engine.md:84`, and `docs/technical_reference.md:245,348` read `riskSnapshot.compositeRiskScore`. The field does not exist. This alone makes the defensive override and Policy Check 4 unimplementable without a rename or alias decision.
2. **H-06 — HHI bands conflict.** `:90-93` uses `2,500-5,000` as “high concentration → alert” and `>5,000` as “dangerous → critical alert.” `:98` claims these thresholds are consistent with Engine 1, but Engine 1 stops at 2,500, and the master reference blocks above 2,500. The `HIGH_CONCENTRATION` alert at `:212` fires only above 5,000, while Engine 4 may already be blocking at 2,500.
3. **C-02 — Check 4 semantics.** `:202` says `adjustedThreshold` (not `riskScore`) is what the Policy Engine reads. `docs/policy_engine.md:84` compares `compositeRiskScore > adjustedThreshold`. The intent is clear (score vs threshold), but the field name and the exact comparison contract are not.
4. **M-06 — Risk-score scale undefined.** The composite formula (`:170-176`) produces a weighted sum, but the document does not state the range (0–100?), the rounding rule, or the band boundaries for `LOW/MEDIUM/HIGH/CRITICAL`. That makes cross-engine consistency untestable.
5. **M-11 — Protocol-risk source.** The protocol safety score depends on DeFiLlama (`:102-129`), but the plan/tier, refresh cadence, and failure behaviour are not specified. The FKL open item admits this (`docs/financial_knowledge_layer.md:212`).
6. **M-01 — Trigger.** `:16` subscribes to `PortfolioSnapshotCreated`; `:291` emits `RiskAnalysisCompleted`. This is consistent, but Engine 3 is also supposed to run in parallel and read Engine 2’s “latest available” snapshot (`docs/engine3_yield_opportunity.md:15`). The fallback/default protocol scores are not defined.

**Severity contribution:** C-02, H-06, M-06, M-11.

#### 3.2.3 `docs/engine3_yield_opportunity.md`

**Strengths**

- Complete pipeline from discovery through APY normalisation, IL-adjusted yield, sustainability, TOPSIS, portfolio fit, idle capital, and output.
- Correct `poolType` dispatch for constant-product vs stable-swap IL (`:77-95`).
- Explicit 90-day payback and $25/year minimum (`:207-208`), which is a sensible economic gate.
- Good articulation of the difference between spot APY and 7-day TWAP (`:59-71`).

**Findings**

1. **H-07 — TOPSIS criteria and weights conflict.** `:140-146` defines **five** criteria: Net APY, Protocol Safety, Yield Consistency, Liquidity, IL Risk. `docs/technical_reference.md:279-286` defines **six**: Net APY, Protocol Safety, Liquidity, Net Switching Benefit, Gas/Bridge Cost, Payback Period. The weights also differ: Engine 3 gives Capital Preservation 0.50 to Safety and 0.05 to Net APY; the master reference gives Safety 45% and Yield Seeker Net Yield 50%, while Engine 3 gives Yield Seeker Net APY 0.35 and Safety 0.20. A ranking is not reproducible across documents.
2. **H-07 — APY window vs field name.** `:64-65` computes a TWAP with “minimum 7 daily data points”; `:227` names the output `twapApy30dPercent`. The master reference says “7-day TWAP” (`docs/technical_reference.md:254`). The field name implies 30 days; the formula implies 7. Either the window or the name must change.
3. **H-12 — `YieldOpportunitiesUpdated` does not wake Engine 4.** `:260` says the event wakes Engine 4. Engine 4 depends only on Engine 1, Engine 2, and Engine 5 (`docs/engine4_strategy_optimization.md:5`) and is triggered only by `RiskAnalysisCompleted` (`:18`). Engine 4 never consumes `YieldOpportunitySnapshot`. If E3 is meant to feed E4, the dependency is missing; if not, the event contract is wrong.
4. **M-12 — Cross-chain assessment ownership.** Engine 3 produces `crossChainRedeployment` (consumed by Engine 6 at `docs/engine6_execution_intelligence.md:156`), while Engine 4 independently computes a bridge cost-benefit gate (`docs/engine4_strategy_optimization.md:194-225`). There are two cross-chain economic gates with different inputs and thresholds.
5. **M-11 — Data freshness claim.** `:71` says the 300s FKL pool TTL guarantees opportunities are “never more than 5 minutes stale.” That is true only if the adapter refresh succeeds; a stale cache entry or failed refresh can be older. The claim needs a failure-mode qualifier.
6. **M-03 — Weight/portfolio-fit composition.** `:176` uses `0.70 × TOPSIS + 0.30 × portfolioFitScore`, but neither the master reference nor Engine 4 documents this blend or its interaction with persona weights.

**Severity contribution:** H-07, H-12, M-03, M-11.

#### 3.2.4 `docs/engine4_strategy_optimization.md`

**Strengths**

- Clear model-selection ladder by snapshot count (`:51-118`) and explicit constraints by persona.
- Good treatment of defensive override before optimisation (`:22-37`).
- Explicit drift/urgency tiers and cooldown (`:143-158`, `:229-239`).
- Honest open items on HHI derivation, cooldown tuning, and the shared `poolType` library (`:287-289`).

**Findings**

1. **C-02 — `compositeRiskScore` undefined.** `:25` reads `riskSnapshot.compositeRiskScore`; Engine 2 defines `riskScore` (`docs/engine2_risk_intelligence.md:283`). The defensive override cannot be implemented as written.
2. **H-12 — Missing Engine 3 dependency.** The header (`:5`) and trigger (`:18`) omit Engine 3 and `YieldOpportunitySnapshot`. Yet `docs/system_architecture.md:163` says Engine 3’s event is consumed by Engine 4, and Engine 3 says it wakes Engine 4. Either Engine 4 must consume E3, or the event graph must be corrected.
3. **H-06 — HHI blocking threshold.** `:135-139` blocks any placement that pushes chain HHI above 2,500. Engine 2 treats 2,500–5,000 as “high” but not critical, and its critical alert starts at 5,000. The same portfolio can be blocked by E4 while E2 says “high, not critical.”
4. **M-02 — Cooldown location.** `:232-239` suppresses plan generation for 7 days unless HIGH urgency. `docs/technical_reference.md:454` says the **Policy Engine’s Check 9** rejects LOW/MEDIUM plans within 7 days. `docs/policy_engine.md:166-197` defines Check 9 as action-type restrictions, not cooldown. The cooldown is specified in two places and is absent from the “authoritative” policy spec.
5. **M-03 — Drift vs tolerance overlap.** `:149-158` sets LOW urgency at 2–7% drift and persona tolerance at ±3% for CAPITAL_PRESERVATION/CONSERVATIVE. A 2.5% drift is simultaneously “within tolerance” and “LOW urgency → routine maintenance.” The precedence is undefined.
6. **H-04 — GMX dead path.** `:174-186` says GMX GM positions produce `MANUAL_REVIEW`, not `LP_ADD`/`LP_REMOVE`. But Engine 6’s sourcing waterfall still draws from GMX as a last resort (`docs/engine6_execution_intelligence.md:71-73`), and the Execution Router still selects GM pools (`docs/execution_router.md:75-83`). There is no automated GMX withdrawal action to satisfy the waterfall.
7. **M-04 — CCTP cost/timing prose.** `:198-199` says Standard is `$0` with ~13–19 min finality and Fast is a “small fixed fee, near-instant.” Official CCTP v2 Standard finality is roughly 15–19 min on ETH/ARB/Base and ~17s on Solana; Fast fees vary by route and are deducted at mint. “Fixed fee” and the 13-minute lower bound are inaccurate.
8. **M-12 — Aggregator/venue assumptions.** Engine 4 emits abstract deltas and does not choose venues, but its cross-chain gate assumes CCTP is always the bridge and that HIGH urgency justifies Fast. The router later allows `BRIDGE_AGGREGATOR` for non-USDC, which Engine 4’s cost model does not cover.

**Severity contribution:** C-02, H-04, H-06, H-12, M-02, M-03, M-04.

#### 3.2.5 `docs/engine5_user_intelligence.md`

**Strengths**

- Clear two-score persona model and a deterministic mapping to four personas plus a yield tilt (`:29-65`).
- `GoalProfile` is compact and mostly well-scoped (`:93-108`).
- Good Copilot intent table and a sensible free/paid split (`:155-165`).
- Honest note that there is no `maxRiskScore` on `GoalProfile` (`:111`), which correctly avoids a second risk-tolerance source.

**Findings**

1. **C-02 — `allowedActionTypes` does not exist on `GoalProfile`.** `docs/policy_engine.md:177` checks `goalProfile.allowedActionTypes.includes(action.type)`. The `GoalProfile` schema (`:96-108`) has no such field. `allowedActionTypes` exists only in the Engine 6 `AutopilotConfig` (`docs/engine6_execution_intelligence.md:228-234`). Policy Check 9 cannot run.
2. **H-11 — Both pinned LLMs are deprecated.** `:138-139` pins `GPT-4.1-mini` as primary and `Gemini 2.5 Flash` as fallback. As of 18 Nov 2026, OpenAI marks `gpt-4.1-mini` as deprecated/retiring, and Google marks `gemini-2.5-flash` as deprecated with API access limited to existing users until 1 Dec 2026, after which requests auto-route to `gemini-3.5-flash-lite`. A production design must not pin either.
3. **M-14 — No trigger/ownership section.** Unlike Engines 1–4 and 6, Engine 5 has no “Trigger” section. It is both (a) the onboarding persona profiler that produces `GoalProfile` and (b) the Copilot that presents plans after Engine 4. The master pipeline places E5 after E4 (`docs/technical_reference.md:36`), but E3/E4 require `GoalProfile` earlier. This is resolvable only if `GoalProfile` is treated as a persisted onboarding artifact, which no document states.
4. **M-03 — `driftTolerancePercent` / `rebalancingFrequencyCap` are not consumed.** Engine 4 hard-codes ±3/5/7% tolerance and a 7-day cooldown (`docs/engine4_strategy_optimization.md:155-158,232-239`). The `GoalProfile` fields (`:105-106`) are therefore dead or duplicated.
5. **M-12 — Copilot `STRATEGY_QUERY` triggers E3/E4.** `:159` says a strategy query triggers Engine 3/4 analysis. There is no idempotency, deduplication, or cost-control contract for repeated queries, and x402 charges per query.
6. **M-06 — Prompt-injection/guardrail spec absent.** The Copilot has tools and execution intents but no documented prompt-injection boundary, tool-call allowlist, or “never fabricate numbers” enforcement beyond prompt instructions.

**Severity contribution:** C-02, H-11, M-03, M-12, M-14.

#### 3.2.6 `docs/engine6_execution_intelligence.md`

**Strengths**

- The macro-to-micro split from Engine 4 to Engine 6 is well explained.
- Explicit authority filter (`:36-59`) and cross-position sourcing waterfall (`:61-83`).
- Clear action taxonomy (`:88-100`) and dependency-injection rules (`:129-149`).
- Good treatment of autopilot qualification and emergency pause (`:174-190`).
- Honest open items on partial plans and the shared `poolType` library (`:236-241`).

**Findings**

1. **C-06 — `ExecutionIntent` is not execution-complete.** `:196-221` includes `type, chain, asset, amount, amountUsd, poolType, bridgeMechanism, cctpTransferSpeed, fulfillment, dependsOn, estimatedCostUsd, overrideReason`. It lacks:
   - `protocol` / adapter identifier;
   - `sourceAddress` / position address;
   - `pool` / `market` / `tokenId` for LP positions;
   - CCTP `destinationCaller`, `mintRecipient`, `destinationDomain`, `maxFee`, `minFinalityThreshold`;
   - ERC-20 `approve` spender and amount;
   - `deadline` / `validUntil`;
   - slippage tolerance per action;
   - nonce / transaction ordering key.
   The Router and Policy Engine cannot validate or build from this schema without re-deriving fields.
2. **C-06 — Amount units.** `:207` uses `amount: string // decimal string`. `docs/technical_reference.md:645` mandates integer base units (wei/lamports) or scaled arbitrary-precision values. Decimal strings are acceptable only if the scale/decimals are carried; they are not.
3. **C-01 — “Hands to Orchestrator” bypasses Policy.** `:9` and `:31` say the `ExecutionIntent` goes to the Orchestrator and stops. Policy is not in that sentence. This contradicts `docs/technical_reference.md:456` and `core_documents/system_architecture.md:60`.
4. **H-04 — GMX waterfall inconsistency.** `:71-73` orders GMX last in the sourcing waterfall, but `:93-100` says GMX positions generate `MANUAL_REVIEW`, not `LP_REMOVE`. The waterfall cannot actually drain a GMX position without a manual step.
5. **H-05 — `BRIDGE_AGGREGATOR` is a runtime branch with no MVP adapter.** `:210` allows `bridgeMechanism: 'BRIDGE_AGGREGATOR'`; `docs/protocol_layer.md:225` says non-USDC cross-chain has no MVP adapter and needs a product decision.
6. **M-14 — No explicit approval graph for Permit2/EIP-2612.** `:134` injects `APPROVE(token → protocol)` if allowance is insufficient. It does not distinguish standard `approve`, EIP-2612 `permit`, Permit2 `approve`, or Permit2 `permit`; the Router/Planner cannot know which transaction to build.
7. **M-12 — Aggregator/order lifecycle absent.** For UniswapX, the intent must be signed, broadcast to a filler auction, and awaited; Engine 6 has no order lifecycle state. It treats swaps as atomic actions.

**Severity contribution:** C-01, C-06, H-04, H-05, M-12, M-14.

---

### 3.3 Execution stack

#### 3.3.1 `docs/orchestrator_planner.md`

**Strengths**

- Clear separation between Planner (build + simulate) and Orchestrator (state machine, retries, compensation) (`:7-16`).
- Good fail-closed simulation discipline and explicit revert reasons (`:11-14`).
- Well-defined saga states, including GMX `AWAITING_FULFILLMENT` (`:90-130`).
- CCTP decomposition into burn / attest / mint is conceptually correct (`:50-70`).
- Honest open items on GMX compensation and simulation concurrency (`:204-208`).

**Findings**

1. **C-01 — Simulation-before-Policy conflicts with the master flow.** `:86` passes `SimulationPassedPackage` directly to the Policy Engine; `:95-99` runs `PLANNING → SIMULATING → POLICY_CHECK`. `core_documents/system_architecture.md:137-142` runs Policy before the Orchestrator builds/simulates. `docs/technical_reference.md:456` calls Policy a gate “between Engine 6 and the Orchestrator,” while `:478` says simulation is prior to Policy. The repository cannot decide whether Policy sees a simulated package or a pre-simulation intent.
2. **H-01 — The Execution Router is absent.** The Planner builds and simulates transactions directly from the `ExecutionIntent` (`:20-31`). There is no step that calls the Execution Router to resolve pools, split routes, aggregator calldata, or CCTP parameters before building bytes. The Router document claims it produces `ExecutionInstructions` “ready for the Protocol Layer to build transaction bytes” (`docs/execution_router.md:9`), but the Orchestrator never invokes it.
3. **C-06 — `SimulationPassedPackage` is too thin.** `:77-83` carries only `userId`, `executionIntentId`, simulated actions, total gas, and chains. It does not carry the resolved route, pool, calldata, approvals, destination caller, or per-action slippage. Policy cannot validate what it cannot see.
4. **H-05 — Non-USDC bridge is a runtime branch.** `:72` requires the bridge-aggregator adapter’s sub-steps to follow a burn/attest/mint structure, but `docs/protocol_layer.md:225` says no MVP adapter exists and the product decision is open.
5. **M-16 — Simulation method is underspecified.** `:25-28` says EVM uses `eth_call`; `eth_call` alone does not validate nonce, gas payment, signature validity, or full transaction envelope. The doc does not specify state overrides, `eth_estimateGas`, `debug_traceCall`, or how approvals/balances are pre-funded for simulation.
6. **M-16 — No nonce manager.** The saga executes multi-step, potentially parallel same-chain transactions, but no component owns nonce allocation. `core_documents/system_architecture.md:226` explicitly lists “Per-chain nonce tracking home” as unresolved. This is an execution blocker.
7. **M-04 — CCTP timing.** `:62` says Fast Transfer is “seconds (near-instant).” Official Fast finality is roughly 8s on Arbitrum/Base, ~20s on Ethereum, and sub-second on Solana; “near-instant” is acceptable marketing but not a timeout/SLA specification. `:185-186` uses a 60-minute Standard timeout and leaves Fast timeout “determined at build time.”
8. **M-17 — Compensation semantics for GMX unresolved.** `:206` admits the `AWAITING_FULFILLMENT` compensation policy is unresolved. A keeper-fulfilled action can complete after the saga decides to compensate, producing double exposure.

**Severity contribution:** C-01, C-06, H-01, H-05, M-04, M-16, M-17.

#### 3.3.2 `docs/policy_engine.md`

**Strengths**

- The clearest statement of fail-closed intent in the repository (`:7-17`).
- Each check has a concrete predicate and failure code.
- Check 3’s delegation-freshness window and Check 4’s risk-snapshot freshness are sensible.
- Check 8’s `poolType`-aware slippage/price-impact split is technically appropriate.
- Good open items on Curve, the shared `poolType` library, and authority-check mechanics (`:231-236`).

**Findings**

1. **C-02 — Check catalog conflicts with the master reference.** The nine checks (`:15-197`) are KYC, tier limit, authority, risk, chain allowlist, protocol allowlist, per-tx value, slippage, action types. `docs/technical_reference.md:444-454` has KYC, spend, allowlists, authority, risk freshness, persona volatility, slippage, gas/bridge payback, cooldown. Only five checks (KYC, spend, allowlists, authority, slippage) overlap conceptually; the catalogs are not the same.
2. **C-02 — Check 4 reads a non-existent field.** `:84` uses `riskSnapshot.compositeRiskScore`. `RiskSnapshot` defines `riskScore` (`docs/engine2_risk_intelligence.md:283`).
3. **C-02 — Check 9 reads a non-existent field.** `:177` uses `goalProfile.allowedActionTypes`. `GoalProfile` has no such field (`docs/engine5_user_intelligence.md:96-108`).
4. **H-13 — Check 2 queries an audit category the canonical list omits.** `:30-36` sums `auditEntries({ category: 'EXECUTION' })`. `docs/technical_reference.md:610` lists `AUTH, POLICY_EVALUATION, SIMULATION, SIGNING, BROADCAST, CONFIRMATION, FULFILLMENT, COMPENSATION, CIRCUIT_BREAKER, PREFERENCE_UPDATE, PAYMENT` — no `EXECUTION`. `docs/audit_layer_x402.md:28` does define `EXECUTION`, so the audit layer and master reference disagree on the category enum.
5. **Internal contradiction — short-circuiting.** `:17` says every check runs and failures are not short-circuited. `:65` says Check 3 “should short-circuit expensive downstream checks.” The two statements cannot both be true for the same request. If the intent is “run all checks but order cheap ones first,” the wording should say so.
6. **H-02 — Allowlist omits router venues.** `:107-114` allows `aave-v3`, `uniswap-v3`, `marginfi`, `kamino`, `jupiter`, `raydium`, `aerodrome`, `moonwell`, `gmx`, `camelot`. It omits `1inch` and `UniswapX`, which the Router uses (`docs/execution_router.md:23-25`). It includes `camelot`, for which no adapter exists (`docs/protocol_layer.md:23-58`). It also omits `curve`, which the Policy doc itself flags as a routing-quality problem (`:233`).
7. **H-02 — Protocol allowlist validates before routing.** Policy checks `action.protocol`, but the Router selects the actual venue/pool after Policy in the architecture (`core_documents/system_architecture.md:138-140`). If the action carries only a high-level protocol, the allowlist cannot validate the aggregator route legs the Router will choose. The Router has its own post-hoc filter (`docs/execution_router.md:30-40`), which is a second, differently implemented allowlist.
8. **H-04 — GMX branch is dead.** `:170-173` blocks GMX GM pools for Conservative/Capital Preservation. Engine 4/6 never generate `LP_ADD`/`LP_REMOVE` for GMX (`docs/engine4_strategy_optimization.md:174-186`; `docs/engine6_execution_intelligence.md:93-100`), so the branch can only be reached by a malformed or manual action.
9. **M-02 — No cooldown check.** `docs/technical_reference.md:454` says Check 9 is the 7-day cooldown. `:166-197` defines Check 9 as action-type restrictions and contains no cooldown predicate. The cooldown exists only in Engine 4 (`docs/engine4_strategy_optimization.md:232-239`).
10. **M-06 — Tier limit uses `plan.totalValueUsd`.** `:39` compares the 24h spend plus `plan.totalValueUsd` to the tier limit. If the plan includes a withdraw-and-redeposit loop, the total value can double-count economic exposure. The doc does not define whether `totalValueUsd` is gross notional, net new capital, or sum of action notionals.
11. **M-13 — Sanctions coverage is indirect.** `:128` relies on allowlist curation for OFAC/Chainalysis/TRM screening. This is reasonable for MVP but is not a live per-transaction sanctions check; `docs/identity_kyc.md:81-89` defers counterparty screening to Phase 2 and the Travel Rule to Phase 3. The security posture should be stated consistently.

**Severity contribution:** C-02, H-02, H-04, H-13, M-02, M-06, M-13.

#### 3.3.3 `docs/execution_router.md`

**Strengths**

- Good articulation of micro-routing responsibilities: aggregation, split orders, protocol selection, multicall batching, CCTP speed/domain selection.
- The allowlist filter for aggregator route legs is the right idea (`:30-40`).
- Job 3’s `poolType` dispatch is consistent with the rest of the system.
- Open items on Curve, GMX liquidity in aggregator graphs, and the shared library are useful (`:200-204`).

**Findings**

1. **H-01 — Self-contradiction.** `:9` says the Router produces route-resolved instructions; `:13` says “Does not route, sign, simulate, or check policy.” The intended meaning may be “does not execute/sign,” but the sentence contradicts the entire document.
2. **H-01 — Ordering.** The Router must run before the Planner builds bytes, because the bytes depend on the selected pool, split, aggregator calldata, and CCTP parameters. `core_documents/system_architecture.md:139-140` places it after build+simulate. `docs/orchestrator_planner.md` never calls it.
3. **H-02 — Venue/allowlist mismatch.** `:23-25` uses `1inch` / `UniswapX` on all three EVM chains. `docs/policy_engine.md:107-114` does not allow either. `:30` says any non-allowlisted leg is discarded, so the Router would discard every 1inch/UniswapX route, leaving only direct Uniswap V3 — which contradicts the stated aggregation strategy.
4. **H-02 — Camelot adapter missing.** Policy allows `camelot` on Arbitrum (`docs/policy_engine.md:111`); the Protocol Layer roster has no Camelot adapter (`docs/protocol_layer.md:23-58`). A policy-approved venue cannot be executed.
5. **H-04 — GMX Job 3 is unreachable.** `:75-83` selects a GM pool for `gmx_gm_pool` actions, but Engine 4/6 never emit automated GMX LP actions. The job is dead code for the current product scope.
6. **M-12 — UniswapX lifecycle missing.** `:23-25` treats UniswapX as an aggregator. UniswapX is an intent auction: the swapper signs a Permit2 order, broadcasters/fillers compete, and settlement is asynchronous. The Router has no order signing, auction broadcast, filler wait, or failure/expiry state.
7. **M-14 — `ExecutionInstructions` schema gaps.** `:163-186` includes steps with `protocol, tokenIn, tokenOut, amountIn, amountOut`, but not `poolAddress`, `routerCalldata`, `approvalTarget`, `deadline`, `value`, `nonce`, or `gasLimit`. The Protocol Layer still has to infer or re-derive them.
8. **M-04 — CCTP fee/timing.** `:125-129` says Standard has no fee and ~13–19 min, Fast is instant, and domains are assigned. The domain IDs are correct. The fee/timing prose is simplified; Fast fees vary by route and are deducted at mint.

**Severity contribution:** H-01, H-02, H-04, M-12, M-14.

#### 3.3.4 `docs/protocol_layer.md`

**Strengths**

- Clean adapter pattern and a useful roster by category (`:23-72`).
- Correct high-level treatment of Uniswap V3 NFT positions, Aerodrome stable pools, and GMX async deposits (`:75-105`).
- `UnsignedTransaction` schema is a good boundary object (`:163-194`).
- Config-driven adapter manifest is the right extensibility model (`:196-200`).
- Open items on non-USDC bridging and multi-chain instances are honest (`:222-227`).

**Findings**

1. **C-04 — CCTP V1 calldata.** `:134` uses `encodeDepositForBurn(amount, destDomain, mintRecipient, burnToken)`. CCTP V2 `depositForBurn` requires `destinationCaller`, `maxFee`, and `minFinalityThreshold` in addition. A V2 TokenMessenger will reject or mis-handle this call.
2. **C-04 — Mint recipient hard-coded.** `:146` says `executionMode: 'embedded' // minting always goes to the embedded wallet`. For a delegated Path B user, the burned USDC may come from the external wallet; minting to the embedded wallet changes custody and breaks the user’s expectation. The recipient must be a parameter, with an explicit policy decision.
3. **C-04/M-04 — Fee claims.** `:159` says “CCTP v1/v2 Standard Transfer has no protocol-level fee. CCTP v2 Fast Transfer carries a small flat fee.” Standard has no protocol fee; Fast fees are **route-specific**, not flat, and are deducted from the minted amount. The adapter must quote `maxFee` and expose the expected received amount.
4. **C-03 — Solana delegate overstatement.** `:121` says the instruction’s signer/authority references the delegate and that “SPL delegate authority is expressed at the instruction level.” SPL Token `delegate` authorizes transfer/burn of tokens up to a delegated amount; it does not authorize the delegate to sign lending/LP instructions on behalf of the owner. Aave/Kamino/MarginFi positions are program-owned accounts whose authority is the user’s wallet, not a token delegate.
5. **C-03 — ERC-4337 assumption.** `:119` wraps calldata in a `UserOperation` targeting the user’s ERC-4337 smart account/session-key executor. MetaMask and Phantom EOAs are not ERC-4337 accounts by default. EIP-7702 (Final) is not mentioned, and no smart-account provisioning step exists for Path B.
6. **H-03 — Permit2 claim.** `:204` says Aave V3 and Uniswap-ecosystem contracts support Permit2 and can avoid separate `approve()` transactions. Aave V3 supports EIP-2612 `supplyWithPermit`, not Permit2. Permit2 requires a one-time on-chain `approve(Permit2)` per token before signature-based transfers work.
7. **H-02 — Adapter roster gaps.** `:29-36` has a combined `OneInchUniswapXAdapter`, which conflates an aggregator API with an intent auction. No Camelot adapter exists despite the policy allowlist. No Curve adapter exists, which the Policy doc flags.
8. **H-05 — Non-USDC bridge.** `:225` says no MVP adapter; `docs/orchestrator_planner.md:196` already lists a Phase 2 bridge aggregator, and `docs/engine6_execution_intelligence.md:210` allows the branch at runtime.
9. **M-14 — No token decimals/rounding contract.** `UnsignedTransaction` carries amounts, but the doc does not state whether amounts are base units, how decimals are resolved, or how round-down is enforced. This interacts with the decimal-string issue in `ExecutionIntent`.

**Severity contribution:** C-03, C-04, H-02, H-03, H-05, M-04, M-14.

#### 3.3.5 `docs/cross_chain_ecosystem.md`

**Strengths**

- Clear TEE signing and broadcast responsibilities (`:9-61`).
- Good gas-cap congestion model and explicit `queued` state (`:74-130`).
- Correct distinction between practical L2 finality and native rollup withdrawal finality (`:141`).
- Good CCTP status/result schema (`:173-223`).
- Honest open items on Solana re-sign handoff and queued-state handling (`:238-243`).

**Findings**

1. **C-03 — Delegation model.** `:49` says delegated authority is provided by a TEE-managed session key authorized on-chain via ERC-4337 session key (EVM) or SPL `delegate` (Solana). As above, this does not cover Aave/Uniswap positions or program-owned Solana positions.
2. **M-05 — Base gas formula omits operator fee.** `:87-89` computes `l1DataFee = GasPriceOracle.l1BaseFee × txCalldataSizeEstimate × scalar`. The current Base formula includes `l1Fee` **and** `operatorFee`; the recommended approach is `GasPriceOracle.getL1Fee(bytes)` / `getL1FeeUpperBound(uint256)` plus `getOperatorFee(gasUsed)`, not a hand-rolled scalar multiplication.
3. **M-05 — Arbitrum API.** `:97-99` uses `ArbGasInfo.getL1BaseFeeEstimate() × calldataDerivedGasUnits`, which is valid, but the repo also claims use of `NodeInterface` (`docs/technical_reference.md:504`) without specifying the required `gasEstimateL1Component(address to, bool contractCreation, bytes calldata data)` parameters.
4. **M-04 — CCTP fee reporting.** `:187` says fee fields reflect CCTP’s actual schedule. The `ChainExecutionResult` schema (`:202-223`) has `bridgeFeeUsd` but no `maxFee`, `minFinalityThreshold`, `destinationCaller`, or expected received amount. The fee cannot be audited from the result alone.
5. **H-13/C-04 — CCTP ownership overlap.** `:185` says the target-chain mint is confirmed before returning to the Orchestrator and that the PSM handles only externally initiated bridges. `docs/orchestrator_planner.md:50-70` says the Planner polls Iris and builds the mint tx. `docs/portfolio_state_manager.md:149-151` says CCE tracks CrestFlow-initiated bridges. Three components claim ownership.
6. **M-16 — Re-sign handoff unresolved.** `:169,241` admits the Turnkey re-sign interface for Solana blockhash expiry is unspecified. This is a real operational gap: a delayed Solana transaction cannot be re-signed without a defined API contract.
7. **M-17 — `queued` downstream handling.** `:243` admits the Orchestrator and Copilot handling for `queued` is undefined. A transaction waiting on gas is not the same as a stuck broadcast; the user-facing and retry semantics differ.

**Severity contribution:** C-03, C-04, H-13, M-04, M-05, M-16, M-17.

#### 3.3.6 `docs/portfolio_state_manager.md`

**Strengths**

- Clear four-job scope and correct post-execution scoping to `event.chains` (`:15-99`).
- Good distinction between CrestFlow-initiated and externally initiated bridges (`:147-168`).
- Explicit retention and reconciliation sweep (`:101-143`).
- Correct statement that `poolType` is not applicable to PSM (`:164-168`).

**Findings**

1. **H-14 — Indexer polling mismatch.** `:26-28` uses `{ solana: 500ms, evm: 1500ms }`, `maxWait: 10000ms`, and logs `INDEXER_LAG_WARNING` on timeout. `docs/technical_reference.md:597` polls every 1 second, times out after 15 seconds, and sets `dataQuality: 'PARTIAL'`. The timeout and degradation behaviour differ.
2. **M-04 — CCTP fee claim is wrong for Standard.** `:172` says CCTP is “a 1:1 burn-and-mint at the burned amount minus Circle’s flat protocol fee.” Standard transfers have no protocol fee; Fast transfers have a route-specific fee. The sentence is wrong for the MVP Standard path and contradicts `docs/cross_chain_ecosystem.md:219` (“zero for Standard Transfer”).
3. **M-06/M-16 — Webhook signature verification not specified.** `:113-129` subscribes to Alchemy Notify and Helius but does not require HMAC/signature verification. Alchemy provides `X-Alchemy-Signature` (HMAC SHA-256); Helius provides a static `authHeader`. Without verification, an attacker who learns the webhook URL can forge balance events.
4. **M-16 — Webhook idempotency/ordering.** The doc does not specify event IDs, deduplication, or out-of-order handling. At-least-once webhook delivery plus a 10-minute reconciliation sweep can produce duplicate scans.
5. **H-13 — Bridge ownership overlap.** `:149-151` says PSM monitors external bridges only, but `docs/orchestrator_planner.md:50-70` and `docs/cross_chain_ecosystem.md:185` assign internal bridge monitoring to other components. The boundary is not consistent.
6. **M-15 — `dataQuality` is not defined.** The technical reference introduces `dataQuality: 'PARTIAL'`, but PSM’s snapshot/sync contract does not define the enum, when it is set, or how downstream engines consume it.

**Severity contribution:** H-13, H-14, M-04, M-06, M-15, M-16.

#### 3.3.7 `docs/portfolio_state_analysis.md`

**Strengths**

- Clear address model: two address strings, four chain scans (`:13-25`).
- Correct per-chain treatment of the shared EVM address (`:21`).
- Good cross-chain price unification via canonical symbol + CoinGecko ID (`:63-76`).
- Explicit scan triggers and output schema (`:101-131`).

**Findings**

1. **C-03 — Execution-authority claim is wrong.** `:23` says execution authority is established by “ERC-20 approve, ERC-4337 session key, SPL delegate instruction.” An ERC-20 `approve` only lets a spender transfer that token; it does not authorize management of an Aave position or a Uniswap V3 LP NFT. SPL `delegate` is similarly token-scoped. This sentence overstates what onboarding achieves.
2. **M-11 — Indexer strategy unresolved.** `:97,141` explicitly leaves chain-native vs unified multi-chain indexing open. This affects scan latency, cost, and data consistency; it should be decided before implementation.
3. **M-15 — Snapshot overlap.** The output schema (`:112-131`) overlaps `PortfolioSnapshot` in Engine 1 and the PSM’s canonical state. The document does not state which is authoritative or how conflicts are resolved.
4. **M-06 — Address ownership proof.** `:23` says no separate ownership-proof signature is required because proof occurred on-chain at delegation. For read-only tracking addresses, there is no on-chain proof; the system needs a user-signed challenge to prevent adding someone else’s address. The doc does not distinguish read-only tracking from execution authority.

**Severity contribution:** C-03, M-06, M-11, M-15.

---

### 3.4 Data, identity, audit, and governance

#### 3.4.1 `docs/financial_knowledge_layer.md`

**Strengths**

- Excellent port/adapter decomposition and a clear “engines never call external APIs directly” principle (`:9-11`).
- Correct per-chain treatment of Alchemy for Ethereum/Base/Arbitrum and Helius for Solana (`:33-45`).
- Good normaliser and asset-registry design (`:126-180`).
- Dual-source pricing with a divergence guard is the right safety pattern (`:182-187`).
- Honest open items on Aerodrome and provider rate limits (`:208-212`).

**Findings**

1. **H-08 — Cache TTL conflict.** `:101-109` sets live wallet holdings 30s, open positions 30s, pool APYs/rates 300s, token prices 60s, historical daily prices indefinite. `docs/technical_reference.md:142-144` sets live prices 30s, pool data 300s, historical OHLCV 24h. `core_documents/system_architecture.md:189-191` sets live prices 60s, pool APY/TVL 300s, historical daily cron. Three different TTL tables.
2. **H-08 — Oracle divergence conflict.** `:186` halts execution if CoinGecko and the on-chain oracle diverge by more than **5%**. `core_documents/technical_reference.md:60` halts if Chainlink and Pyth diverge by more than **1.5%**. These may be different guards, but the docs do not define which one gates signing, what happens when both fire, or which oracle is “the on-chain oracle.”
3. **M-11 — CoinGecko batch size.** `:184` says batch up to **250** tokens per call. CoinGecko’s `/simple/price` currently accepts up to **515** IDs per request. The repo is under the limit, so this is not fatal, but the stated number is stale and should be derived from the current provider limit.
4. **M-11 — DeFiLlama tier.** The protocol-safety pipeline depends on DeFiLlama, but the doc does not state Free vs Pro, the key mechanism, the rate limit, or the SLA. DeFiLlama Free has no published hard limit/SLA; Pro is a paid tier with higher limits. This is an operational risk for daily scoring.
5. **M-33 — Direct-RPC contradiction.** `:9` says engines never call external APIs; `docs/technical_reference.md:232` says Engine 2 continuously polls Aave/Kamino/MarginFi. The FKL must be the only data path, or the architecture must explicitly allow Engine 2 read-only RPC for liquidation monitoring.
6. **M-10 — `poolType` library claim.** `:152` says the tag is consumed by a shared library across Engines 1–4, Engine 6, the Router, and Policy. `core_documents/system_architecture.md:222` says it is currently reimplemented independently. The library does not exist yet.
7. **M-12 — Jupiter version.** The adapter table (`:88`) names Jupiter but not the API version. Jupiter’s current Swap API uses `/order` and `/build` with a Developer Platform API key; “Jupiter V6 Aggregator API” is outdated nomenclature in the ecosystem.

**Severity contribution:** H-08, M-10, M-11, M-12, M-33.

#### 3.4.2 `docs/identity_kyc.md`

**Strengths**

- Clear compliance purpose and a sensible “KYC as an execution gate” model (`:7-11`, `:70-79`).
- Tiered limits and cross-chain summation are well explained (`:60-67`).
- Good audit-snapshot requirement: capture `kycStatus`/`kycTier` at action time (`:77`).
- Honest open items on in-flight suspension and counterparty AML timing (`:102-106`).

**Findings**

1. **H-09 — Status enum mismatch.** `:49` uses `APPROVED`, `DECLINED`, `RESUBMISSION_REQUESTED`. `docs/technical_reference.md:113` uses `APPROVED`, `REJECTED`, `PENDING`. The system needs one canonical state machine; `DECLINED` vs `REJECTED` and the missing `RESUBMISSION_REQUESTED` will break webhook handling and audit queries.
2. **H-09 — Tier basis mismatch.** `:62` puts “Basic document + liveness” at TIER_1 and “additional docs + source of funds” at TIER_2. `docs/technical_reference.md:119-120` puts “Basic document scan” at Tier 1 and “Full document + facial liveness” at Tier 2. Liveness moves from Tier 1 to Tier 2 depending on the document.
3. **Internal contradiction — KYC short-circuit.** `:75` says “no other check runs if this one fails.” `docs/policy_engine.md:17` and `docs/technical_reference.md:457` say all nine checks run with no short-circuit. This is not just wording: it changes whether a user sees one or all failure reasons, and whether expensive downstream checks consume resources on a KYC-failed plan.
4. **H-13 — `KYC_REVOKED` under `PROFILE_CHANGE`.** `:97` says the Audit Layer emits a `KYC_REVOKED` event under `PROFILE_CHANGE`. `docs/audit_layer_x402.md:26` defines `PROFILE_CHANGE` as Engine 5 goal/persona changes; there is no KYC category. `docs/technical_reference.md:610` has `PREFERENCE_UPDATE` instead. A sanctions revocation is not a preference change.
5. **M-13 — FATF timing contradiction.** `:89` defers the FATF Travel Rule to Phase 3. `core_documents/technical_reference.md:52,157` presents FATF Travel Rule compliance as an implemented security mechanism. The BE report cannot claim both.
6. **M-06 — Vendor claims unverified.** `:15` cites WazirX’s 10-million-user base and ~30,000 checks/day as proof of HyperVerge scale. Those are vendor/customer marketing claims; no independent source was found. HyperVerge developer documentation was unreachable (502), so the webhook HMAC scheme, API version, and sandbox details could not be verified.
7. **M-06 — Webhook verification detail missing.** `:47` says HMAC-SHA256 against a shared secret, which is good, but does not specify the header name, canonicalization, timestamp/replay protection, or key rotation. Without those, the implementation is underspecified.
8. **M-14 — Sanctions rescreen cadence.** `:93-100` covers post-approval sanctions hits but does not specify periodic rescreening, list-update cadence, or how a `SUSPENDED` user’s open positions are handled.

**Severity contribution:** H-09, H-13, M-06, M-13, M-14.

#### 3.4.3 `docs/user_onboarding.md`

**Strengths**

- Clear OAuth + JWT session model (`:13-23`).
- Good Path A/Path B distinction and an honest explanation of why existing Web3 users still get an embedded wallet (`:25-93`).
- Turnkey sub-organisation model and two-layer authority model are well described (`:42-66`).
- Address scope schema is explicit (`:131-148`).

**Findings**

1. **H-10 — Identity schema mismatch.** `:15` extracts `email` + `googleId`; `:21` stores `User { id, email, googleId }`. `:19` says the OAuth callback is keyed on the Google `sub` claim. `docs/technical_reference.md:63-65` says `sub` is the stable identity anchor and email is never the foreign key. The schema has no `sub`/`googleSub` field, so the stated idempotency key cannot be stored.
2. **C-02/H-09 — No KYC in onboarding.** The onboarding steps are authentication, Path A/B fork, embedded wallet provisioning, deposit vs delegate, first portfolio scan, and fiat on-ramp. There is **no KYC step** and no `kycStatus`/`kycTier` field in the `User` schema. Yet `core_documents/system_architecture.md:121` says the flow is Turnkey → KYC → TIER assignment, and Policy Check 1 reads `user.kycStatus`. The user record has no such field.
3. **C-03 — Path B delegation overstatement.** `:32-40,68-80` describes external wallet delegation but relies on the same ERC-4337/SPL delegate assumptions. It does not explain how a MetaMask EOA becomes an ERC-4337 account or how Aave/Uniswap positions are made operable.
4. **H-03 — Turnkey policy capability.** `:63` says the Turnkey-native policy layer enforces “spend limits, destination allowlists, permitted method types.” Turnkey’s policy language can enforce transaction constraints, but slippage is not a native policy field; it requires an uploaded contract interface/ABI/IDL. The doc should not imply that all CrestFlow policy checks are duplicated in the enclave.
5. **M-14 — Fiat on-ramp vendor unspecified.** `:107-115` lists a fiat on-ramp as optional but names no provider, jurisdiction, KYC reuse, or settlement flow.
6. **M-14 — Delegation revocation UX absent.** There is no flow for revoking a session key/SPL delegate, checking residual allowances, or handling a revoked delegation mid-plan. The Policy doc flags stale delegation (`docs/policy_engine.md:67`), but onboarding does not define the user-facing revocation path.
7. **M-15 — Address scope schema duplicates identity.** `:131-148` repeats `userId`, addresses, and scope; it is not clear whether this is a separate table or a view over `User`. The canonical persistence model is undefined.

**Severity contribution:** C-02, C-03, H-03, H-09, H-10, M-14, M-15.

#### 3.4.4 `docs/audit_layer_x402.md`

**Strengths**

- Clear immutability and completeness principles (`:7-14`).
- Good 11-category taxonomy and execution-lifecycle event list (`:16-66`).
- `AuditEntry` schema is detailed, with `executionGroupId`, `relatedTxId`, `kycStatus`, and encryption notes (`:68-129`).
- Useful compliance queries, including cross-chain reconstruction and rolling tier spend (`:193-229`).
- Honest open items (`:352-358`).

**Findings**

1. **C-05 — x402 headers are obsolete.** `:277-287` uses `X-Payment` and decodes it; `docs/technical_reference.md:632` uses `X-Payment-Authorization` and attaches a transaction hash. x402 V2 uses `PAYMENT-REQUIRED` (server → client), `PAYMENT-SIGNATURE` (client → server), and `PAYMENT-RESPONSE` (server → client). The repo’s header names will not interoperate with current facilitators.
2. **C-05 — `exact` scheme semantics are wrong.** `:314-319` advertises `scheme: "exact"`, but the flow verifies a payment transaction hash. The EVM `exact` scheme uses an EIP-3009 `transferWithAuthorization` signature (or Permit2), submitted by the facilitator; the client does not send a tx hash. The repo’s replay model is built around a nonce/timestamp that does not match the scheme’s `validAfter`/`validBefore` authorization window.
3. **C-05 — Facilitator host unreachable.** `:319,327` uses `https://api.goplusfable.xyz/facilitate`. That host returned a transport error during verification and is not the official x402 facilitator example (`https://x402.org/facilitator`). The repository should not depend on an unverified facilitator.
4. **C-05 — Network identifiers are not CAIP-2.** `:315,323` uses `network: "ethereum"` / `"base"`. x402 V2 uses CAIP-2 identifiers such as `eip155:1` and `eip155:8453` (and `solana:<genesisHash>` for Solana). The `accepts` object also puts `facilitator` inside each payment option; V2 separates facilitator/resource server configuration from `PaymentRequirements`.
5. **H-13 — Category taxonomy conflicts with the master reference.** `:16-30` defines 11 categories including `EXECUTION` and `PROFILE_CHANGE`. `docs/technical_reference.md:610` defines 11 event types including `POLICY_EVALUATION`, `SIMULATION`, `SIGNING`, `BROADCAST`, `CONFIRMATION`, `FULFILLMENT`, `COMPENSATION`, `CIRCUIT_BREAKER`, `PREFERENCE_UPDATE`, and `PAYMENT` — a different taxonomy. Policy Check 2 relies on `EXECUTION`, which the master list omits.
6. **H-13 — KYC events have no category.** `docs/identity_kyc.md:97` emits `KYC_REVOKED` under `PROFILE_CHANGE`; the audit table has no `KYC` category. KYC approval/rejection/revocation should have a first-class category or a documented subcategory.
7. **M-13 — Travel Rule query is premature.** `:221-229` provides a FATF Travel Rule originator package query, while `docs/identity_kyc.md:89` defers the Travel Rule to Phase 3. The query may be useful for future planning, but it should be labelled Phase 3 and not presented as an MVP compliance control.
8. **M-16 — No event schema/versioning.** The audit schema has no `schemaVersion`, event version, or idempotency key. At-least-once delivery plus retries can duplicate records; the doc says “immutable” but not “deduplicated.”
9. **M-15 — No retention/partition policy.** Monthly partitioning is mentioned in the system architecture (`core_documents/system_architecture.md:205`), but this doc does not define retention, archival, legal hold, or deletion (GDPR) semantics for encrypted PII.

**Severity contribution:** C-05, H-13, M-13, M-15, M-16.

#### 3.4.5 `docs/integration_audit_report.md`

**Strengths**

- Useful historical record of the pre-pivot cleanup categories (legacy chains, “EVM = one chain,” CCTP promotion).
- The issue categories themselves are sensible.

**Findings**

1. **M-08 — The central claim is false against the current repository.** `:11-13` says “The 17 final documents in `docs/` are internally consistent, architecturally aligned, and free of cross-document bugs” and “17 documents audited. 0 unresolved architectural contradictions.” The repository currently has 19 documents in `docs/` (22 Markdown overall) and numerous contradictions documented in this report. The report also predates the current engine/execution documents.
2. **M-08 — No line-level evidence.** The tables describe resolutions but do not cite file:line or show the before/after text. It cannot be used as an audit trail.
3. **M-08 — Status/version not marked.** The file is not labelled “historical” or “superseded,” so a reader may treat it as current assurance. It should be marked historical or removed from the authoritative set.
4. **M-08 — It creates false confidence.** A BE report or reviewer consulting this file would conclude the design is consistent when it is not. This is a governance risk, not just an editorial one.

**Severity contribution:** M-08.

---

### 3.5 Assets

#### 3.5.1 `docs/assets/system-architecture after pivot.png`

**Finding (M-09): stale/mislabeled.** The image is labelled “after pivot” but still shows the pre-pivot stack: Veriff KYC, Algorand, Folks Finance, Tinyman, Pact, Gora Oracle, a GoPlusfable facilitator, and Haystack Router. It does not match the current four-chain, HyperVerge, CCTP, Turnkey design. This is the most dangerous asset because its filename implies it is current.

#### 3.5.2 `docs/assets/system-architecture before pivot.png`

**Finding (L-04): correct historical reference.** It shows the pre-pivot architecture and is consistent with the “before” label. It should remain, but only as a historical reference.

#### 3.5.3 `docs/assets/Untitled-2026-06-22-2109.png`

**Finding (M-09): actual post-pivot diagram, poor filename.** It shows Orchestrator → Policy → Execution Router and the 9-check list matching `docs/technical_reference.md`. It should be renamed to a descriptive, versioned filename and linked from `core_documents/system_architecture.md`. The current filename is not discoverable and is not referenced by any document.

---

## 4. Cross-document inconsistencies (exact references)

The table below is the core consistency audit. Each row identifies the exact conflicting locations and the implementation consequence. IDs are referenced in the finding register (§8).

| ID | Topic | Document A (reference) | Document B (reference) | Nature of conflict | Implementation impact | Severity |
|---|---|---|---|---|---|---|
| X-01 | Policy check catalog | `docs/policy_engine.md:15-197` — KYC, tier limit, authority, risk, chain allowlist, protocol allowlist, per-tx value, slippage, action types | `docs/technical_reference.md:444-454` — KYC, spend, allowlists, authority, risk freshness, persona volatility, slippage, gas/bridge payback, cooldown; `core_documents/technical_reference.md:155` repeats this second set | Two different “9 checks” | No single policy contract; tests cannot be written; two implementations would behave differently | **C** |
| X-02 | Policy short-circuit | `docs/policy_engine.md:17` — every check runs, no short-circuit | `docs/policy_engine.md:65` — “should short-circuit expensive downstream checks”; `docs/identity_kyc.md:75` — “no other check runs if this one fails” | Same document and KYC doc contradict the all-checks rule | Either failures are incomplete or expensive checks run unnecessarily; UX and resource behaviour undefined | **C** |
| X-03 | Risk score field | `docs/engine2_risk_intelligence.md:283` — `riskScore: number` | `docs/engine4_strategy_optimization.md:25`, `docs/policy_engine.md:84`, `docs/technical_reference.md:245,348` — `riskSnapshot.compositeRiskScore` | Field does not exist in `RiskSnapshot` | Defensive override and Policy Check 4 cannot compile; silent `undefined > threshold` is false in JS | **C** |
| X-04 | Action-type field | `docs/engine5_user_intelligence.md:96-108` — `GoalProfile` has no `allowedActionTypes` | `docs/policy_engine.md:177` — `goalProfile.allowedActionTypes.includes(action.type)` | Field does not exist | Check 9 cannot run; either throws or passes/fails unpredictably | **C** |
| X-05 | Audit category for spend | `docs/policy_engine.md:30-36` — sums `category: 'EXECUTION'` | `docs/technical_reference.md:610` — 11-type list omits `EXECUTION`; `docs/audit_layer_x402.md:28` includes it | Category enum conflict | Tier-limit query returns zero or errors; AML limit unenforceable | **C** |
| X-06 | Execution order | `core_documents/system_architecture.md:60,137-142` — `E6 → PE → ORC → ER` | `docs/policy_engine.md:11` — PE evaluates `SimulationPassedPackage`; `docs/orchestrator_planner.md:86,95-99` — Planner builds/simulates then Policy | Circular dependency: Policy needs simulation; simulation is placed after Policy | The pipeline cannot be scheduled; either Policy sees no package or build happens before approval | **C** |
| X-07 | Router position/role | `docs/execution_router.md:9` — route-resolved instructions before bytes | `docs/execution_router.md:13` — “Does not route”; `core_documents/system_architecture.md:139-140` — build/simulate before Router | Router both routes and does not; placed after the step that needs it | No component actually invokes the Router; bytes built without route | **H** |
| X-08 | Allowlist vs venues | `docs/policy_engine.md:107-114` — no 1inch/UniswapX; includes camelot | `docs/execution_router.md:23-25,30` — routes via 1inch/UniswapX and filters against allowlist; `docs/protocol_layer.md:23-58` — no camelot adapter | Allowlist, router, and adapter roster disagree | Router discards all aggregator routes or bypasses policy; camelot actions unexecutable | **H** |
| X-09 | CCTP calldata | `docs/protocol_layer.md:134` — 4-arg `depositForBurn` | CCTP V2 requires `destinationCaller`, `maxFee`, `minFinalityThreshold` | V1 calldata in a V2 design | Burn reverts or is misencoded; cross-chain transfer fails | **C** |
| X-10 | CCTP mint recipient | `docs/protocol_layer.md:146` — mint always to embedded wallet | `docs/protocol_layer.md:114-121` — delegated Path B execution from external wallet | Custody/recipient contradiction | Delegated user’s USDC is minted to a different wallet; accounting and ownership break | **C** |
| X-11 | CCTP ownership | `docs/orchestrator_planner.md:50-70,86` — Planner polls Iris/builds mint | `docs/cross_chain_ecosystem.md:185` — CCE confirms mint; `docs/portfolio_state_manager.md:149-151` — PSM external-only; `core_documents/system_architecture.md:152` — Orchestrator receives both txIds | Three owners for one bridge lifecycle | Duplicate polling, double mint attempts, or no owner; audit trail ambiguous | **C** |
| X-12 | CCTP fees | `docs/protocol_layer.md:159` — Fast “small flat fee”; `docs/portfolio_state_manager.md:172` — “minus Circle’s flat protocol fee” | `docs/cross_chain_ecosystem.md:219` — zero for Standard; official docs: Standard zero, Fast route-specific fee deducted at mint | Fee model wrong/imprecise | Cost-benefit math and user quotes are wrong; received amount not predicted | **H** |
| X-13 | Delegated execution | `docs/technical_reference.md:100-101`, `docs/protocol_layer.md:119-121`, `docs/cross_chain_ecosystem.md:49`, `docs/portfolio_state_analysis.md:23` — ERC-4337 session keys + SPL delegate for all positions | Aave V3 `withdraw` is `msg.sender`-only; Uniswap V3 LP NFT owner-only; SPL delegate is transfer/burn-only | Authority model does not match protocol reality | Path B cannot rebalance/withdraw Aave or Uniswap positions; product promise is undeliverable | **C** |
| X-14 | Permit2 | `docs/protocol_layer.md:204` — Aave V3/Uniswap support Permit2, no separate approve | Aave V3 supports EIP-2612 `supplyWithPermit`, not Permit2; Permit2 requires one-time on-chain approval | Wrong approval model | First-time flows still need an on-chain approval; Aave path reverts or falls back | **H** |
| X-15 | KYC status/tier | `docs/identity_kyc.md:49` — `DECLINED`, `RESUBMISSION_REQUESTED`; `:62` — liveness at Tier 1 | `docs/technical_reference.md:113` — `REJECTED`, `PENDING`; `:119-120` — liveness at Tier 2 | Enums and tier basis differ | Webhook mapping and tier assignment inconsistent; wrong limits | **H** |
| X-16 | KYC gate semantics | `docs/identity_kyc.md:75` — first wall, no other check runs | `docs/policy_engine.md:17`, `docs/technical_reference.md:457` — all checks run | Short-circuit contradiction | Different failure payloads and resource use; audit `checksRun` count ambiguous | **H** |
| X-17 | User identity anchor | `docs/user_onboarding.md:21` — `User { id, email, googleId }`; `:19` — callback keyed on `sub` | `docs/technical_reference.md:63-65` — `sub` is the anchor; email is not a foreign key | No `sub` field; idempotency key cannot be stored | Duplicate users on retried OAuth; identity joins unreliable | **H** |
| X-18 | Audit taxonomy | `docs/audit_layer_x402.md:16-30` — 11 categories incl. `EXECUTION`, `PROFILE_CHANGE` | `docs/technical_reference.md:610` — 11 event types incl. `PREFERENCE_UPDATE`, no `EXECUTION`; `docs/identity_kyc.md:97` — `KYC_REVOKED` under `PROFILE_CHANGE` | Category vs action taxonomy conflated | Tier-limit query and compliance reports target non-existent categories | **H** |
| X-19 | FATF Travel Rule | `core_documents/technical_reference.md:52,157` — implemented FATF Travel Rule | `docs/identity_kyc.md:89` — Phase 3; `docs/audit_layer_x402.md:221-229` — query exists | Compliance status overstated | BE report may claim non-existent compliance; legal/reporting risk | **M** |
| X-20 | HHI bands | `docs/engine1_portfolio_intelligence.md:228-230` — 1,500/2,500; `docs/engine2_risk_intelligence.md:90-93` — adds 2,500–5,000/>5,000; `docs/engine4_strategy_optimization.md:135-139` — blocks >2,500 | `docs/technical_reference.md:188-190` — blocks >2,500 | Three band sets | Same portfolio is “high,” “critical,” and “blocked” depending on component | **H** |
| X-21 | TOPSIS criteria/weights | `docs/engine3_yield_opportunity.md:140-146` — 5 criteria; CP Safety 0.50, YS Net APY 0.35 | `docs/technical_reference.md:279-286` — 6 criteria incl. switching benefit, gas/bridge cost, payback; CP Safety 45%, YS Net Yield 50% | Different model and weights | Rankings not reproducible; persona behaviour differs by implementation | **H** |
| X-22 | APY window | `docs/engine3_yield_opportunity.md:64-65` — 7 daily points; `:227` — `twapApy30dPercent` | `docs/technical_reference.md:254` — 7-day TWAP | Name implies 30d, formula says 7d | UI/analytics mislabel; model input ambiguous | **H** |
| X-23 | Cache TTL | `docs/financial_knowledge_layer.md:103-109` — prices 60s, holdings 30s, historical indefinite | `docs/technical_reference.md:142-144` — prices 30s, pool 300s, historical 24h; `core_documents/system_architecture.md:189-191` — prices 60s, historical daily cron | Three TTL tables | Stale execution pricing or unnecessary RPC load; historical refresh undefined | **H** |
| X-24 | Oracle divergence | `docs/financial_knowledge_layer.md:186` — CoinGecko vs on-chain >5% halts | `core_documents/technical_reference.md:60` — Chainlink vs Pyth >1.5% halts; `docs/technical_reference.md:53` — 60s staleness fatal | Different guards/thresholds, no precedence | Unclear which condition blocks signing; false halts or missed stale prices | **H** |
| X-25 | Indexer sync | `docs/portfolio_state_manager.md:26-28` — 500/1500ms, 10s, warning | `docs/technical_reference.md:597` — 1s, 15s, `dataQuality: 'PARTIAL'` | Different poll/timeout/degradation | Race conditions or premature partial snapshots; downstream behaviour differs | **H** |
| X-26 | Engine pipeline | `docs/technical_reference.md:36-37` — sequential E1→E2→E3→E4→E5→E6; `core_documents/system_architecture.md:56` — sequential | `docs/engine1_portfolio_intelligence.md:268` — E2 and E3 parallel; `core_documents/system_architecture.md:128` — E2 parallel with E3; `docs/engine3_yield_opportunity.md:15` — parallel with E2 | Sequential vs parallel graph | Scheduling, idempotency, and data-freshness assumptions differ | **M** |
| X-27 | E3→E4 dependency | `core_documents/system_architecture.md:163` — E3 event consumed by E4; `docs/engine3_yield_opportunity.md:260` — wakes E4 | `docs/engine4_strategy_optimization.md:5,18` — depends on E1/E2/E5 only; triggered by `RiskAnalysisCompleted` | Event contract vs actual dependency | E4 never consumes yield data; E3 wake is a no-op; stale opportunities | **H** |
| X-28 | Cooldown ownership | `docs/engine4_strategy_optimization.md:232-239` — suppress plan if `<7 days` | `docs/technical_reference.md:454,362` — Policy Check 9 rejects `<7 days`; `docs/policy_engine.md:166-197` — Check 9 is action types, no cooldown | Duplicate/incorrect attribution | Cooldown may be enforced twice or not at all; policy tests wrong | **M** |
| X-29 | Drift/tolerance | `docs/engine4_strategy_optimization.md:149-152` — LOW 2–7%; `:155-158` — CP/CONS ±3% | Same document | 2–3% is both within tolerance and LOW urgency | Non-deterministic plan generation; user sees unnecessary rebalances | **M** |
| X-30 | GMX automation | `docs/engine4_strategy_optimization.md:174-186`, `docs/engine6_execution_intelligence.md:93-100` — GMX = `MANUAL_REVIEW`, no LP actions | `docs/engine6_execution_intelligence.md:71-73` — GMX last-resort source; `docs/execution_router.md:75-83` — GMX pool selection; `docs/policy_engine.md:170-173` — GMX LP branch | Dead/infeasible path | Waterfall cannot drain GMX; Router/Policy branches unreachable; manual exception unhandled | **H** |
| X-31 | Non-USDC bridge | `core_documents/system_architecture.md:223`, `docs/protocol_layer.md:225` — Phase 2, no MVP adapter | `docs/engine6_execution_intelligence.md:210` — `BRIDGE_AGGREGATOR` allowed; `docs/orchestrator_planner.md:196` — adapter listed | Runtime branch vs no adapter | Plan can emit an action no adapter can build | **H** |
| X-32 | Intent schema/units | `docs/engine6_execution_intelligence.md:196-221` — no protocol/pool/recipient/approval/deadline; `amount` decimal string | `docs/technical_reference.md:645` — integer base units; `docs/execution_router.md:163-186` — needs route fields | Schema incomplete and unit-inconsistent | Router/Policy/Planner must infer fields; precision bugs | **C** |
| X-33 | `poolType` library | `core_documents/system_architecture.md:113,222` — recommended shared, currently reimplemented | `docs/engine4_strategy_optimization.md:289` and `docs/engine6_execution_intelligence.md:241` — both “fourth engine”; `docs/financial_knowledge_layer.md:152` — already shared across 1–4, 6, Router, Policy | Ownership and count conflict | Duplicate classification logic; inconsistent GMX/stable-swap behaviour | **M** |
| X-34 | Direct RPC | `core_documents/system_architecture.md:185` — Engines 1–3 never direct RPC | `docs/technical_reference.md:232` — Engine 2 polls Aave/Kamino/MarginFi | Data-access boundary conflict | Rate limits, inconsistent state reads, cache bypass | **M** |
| X-35 | Audit/governance claim | `docs/integration_audit_report.md:11-13` — 17 docs, zero contradictions | Actual 22 Markdown files and the contradictions in this table | False assurance | Reviewers may skip real issues; BE report credibility risk | **M** |
| X-36 | Assets/links | `core_documents/system_architecture.md:9-11` — links to `.excalidraw` and “after pivot” PNG | Actual assets: no `.excalidraw`; “after pivot” PNG is stale; `Untitled...png` is current | Mislabeled/missing assets | Readers use the wrong architecture; broken docs | **M** |
| X-37 | LLM lifecycle | `docs/engine5_user_intelligence.md:138-139` — GPT-4.1-mini / Gemini 2.5 Flash | Official deprecation pages (Nov 2026) — both deprecated | Pinned models are obsolete | Runtime failures after shutdown; cost/quality drift | **H** |
| X-38 | Policy bypass | `docs/engine6_execution_intelligence.md:9,31` — `ExecutionIntent` to Orchestrator | `docs/technical_reference.md:456`, `core_documents/system_architecture.md:60` — Policy gate before Orchestrator | Policy stage omitted in Engine 6 contract | A literal implementation bypasses all safety checks | **C** |
| X-39 | Slippage rules | `docs/policy_engine.md:155-158` — `goalProfile.maxSlippagePercent`; `docs/technical_reference.md:452` — 0.5%/1.0% fixed; `core_documents/technical_reference.md:159` — hard >0.5% reject | Same system | Three slippage policies | A 0.8% swap passes one rule, fails another; policy cannot be certified | **H** |
| X-40 | Audit event count/type | `docs/audit_layer_x402.md:16` — 11 categories; `docs/technical_reference.md:610` — 11 action types; `core_documents/technical_reference.md:219` — 11 categories | Same system | Category vs action conflation | Schema and queries disagree; reporting unreliable | **H** |

### 4.1 Root causes behind the inconsistency pattern

1. **No canonical contract owner.** Each document re-states shared concepts (9 checks, HHI bands, TTLs, audit categories) instead of importing a single definition.
2. **Summary documents were edited independently of component documents.** `core_documents/technical_reference.md` and `docs/technical_reference.md` contain “best of” statements that were not propagated to the detailed specs.
3. **Policy was specified twice.** `docs/policy_engine.md` is implementation-level; the master reference describes a different policy generation. Neither is marked authoritative.
4. **Execution ordering was never fixed after the post-pivot rewrite.** The Mermaid graph, execution prose, Planner state machine, and Router document each describe a different sequence.
5. **External integrations were written from memory, not from current vendor contracts.** CCTP, x402, Permit2, ERC-4337, and the LLM pins all show signs of stale or assumed APIs.

---

## 5. External integration and technology verification

**Verification date:** 18 November 2026. “Verified” means the official documentation or primary source was inspected; “Unverified” means the source was unreachable or the claim is marketing-only. This section does not treat vendor marketing as technical evidence.

| ID | Integration / claim in repo | What the official source says | Status | Official reference |
|---|---|---|---|---|
| E-01 | CCTP adapter is “v1/v2” and uses 4-arg `depositForBurn` (`docs/protocol_layer.md:134,159`) | CCTP V2 is the current version; V1 is legacy. V2 `depositForBurn` requires `destinationCaller`, `maxFee`, and `minFinalityThreshold` in addition to amount, domain, recipient, and burn token. | **Contradicted** | https://developers.circle.com/cctp/references/contract-interfaces |
| E-02 | Standard ~13–19 min; Fast “small flat fee / near-instant” (`docs/execution_router.md:125-129`, `docs/engine4_strategy_optimization.md:198-199`) | Standard has no protocol fee; finality is roughly 15–19 min on Ethereum/Arbitrum/Base and ~17s on Solana. Fast fees vary by route and are deducted at mint; Fast finality is roughly 8s on Arbitrum/Base, ~20s on Ethereum, and sub-second on Solana. | **Partially verified** | https://developers.circle.com/cctp |
| E-03 | Destination mint always goes to embedded wallet (`docs/protocol_layer.md:146`); three components claim bridge ownership | CCTP mint recipient is a parameter (`mintRecipient` / `destinationCaller` in V2). The protocol does not decide custody; the integrator does. | **Contradicted** | https://developers.circle.com/cctp/references/contract-interfaces |
| E-04 | x402 uses `X-Payment` / `X-Payment-Authorization` (`docs/audit_layer_x402.md:277-287`, `docs/technical_reference.md:632`) | x402 V2 uses `PAYMENT-REQUIRED` (server response), `PAYMENT-SIGNATURE` (client request), and `PAYMENT-RESPONSE` (server settlement). The old `X-Payment` header is obsolete. | **Contradicted** | https://docs.x402.org/core-concepts/http-402 |
| E-05 | `scheme: "exact"` verifies a payment transaction hash (`docs/audit_layer_x402.md:314-319`, `docs/technical_reference.md:632`) | The EVM `exact` scheme uses an EIP-3009 `transferWithAuthorization` signature (or Permit2), submitted by the facilitator. The buyer authorizes the exact amount; the client does not send a tx hash. | **Contradicted** | https://docs.x402.org/schemes/exact |
| E-06 | Facilitator `https://api.goplusfable.xyz/facilitate`; networks `ethereum` / `base` (`docs/audit_layer_x402.md:315-327`) | Official facilitator example is `https://x402.org/facilitator`. Networks use CAIP-2 (`eip155:1`, `eip155:8453`, `solana:<genesisHash>`). The GoPlusfable host returned a transport error and is not an official x402 facilitator. | **Contradicted / Unverified** | https://docs.x402.org/core-concepts/network-and-token-support |
| E-07 | Turnkey policy layer enforces spend limits, destination allowlists, permitted methods (`docs/user_onboarding.md:63`); slippage gating implied | Turnkey policy language can enforce `eth.tx.to`, `value`, `chain_id`, EIP-7702 authorizations, Solana `program_keys`/SPL transfers, and uploaded ABI/IDL interfaces. Slippage is not a native policy primitive; it requires an uploaded contract interface. Enforcement occurs in the TEE. | **Partially verified** | https://docs.turnkey.com/ |
| E-08 | Permit2 signatures replace separate `approve()` for Aave V3/Uniswap (`docs/protocol_layer.md:204`; `core_documents/technical_reference.md:189`) | Permit2 requires a one-time on-chain approval of the Permit2 contract per token; after that, signature-based allowances work. Aave V3 does not natively support Permit2; it supports EIP-2612 `supplyWithPermit`. | **Contradicted** | https://docs.uniswap.org/contracts/permit2/overview |
| E-09 | Delegated EVM execution can withdraw/rebalance Aave positions | Aave V3 `withdraw()` burns aTokens from `msg.sender`; there is no general position delegation or credit delegation for supplied positions. Only the position owner can withdraw/rebalance. `supplyWithPermit` is EIP-2612, not a position delegation. | **Contradicted** | https://aave.com/docs/aave-v3/ |
| E-10 | Delegated execution can remove/rebalance Uniswap V3 LP positions | Uniswap V3 LP positions are NFTs; the owner (or approved operator) controls `decreaseLiquidity`/`collect`. A token `delegate`/session key cannot act on an owner-held NFT unless it is an approved operator or the NFT is held by a smart account. | **Contradicted** | https://docs.uniswap.org/contracts/v3/ |
| E-11 | SPL Token `delegate` enables “full delegated execution” (`docs/technical_reference.md:101`, `docs/protocol_layer.md:121`) | SPL Token `delegate` authorizes transfer or burn up to the delegated token amount. It cannot sign lending/LP instructions for positions held in program-owned accounts; those authorities are the user’s wallet or a program-specific authority. | **Contradicted** | https://solana.com/docs/core/tokens |
| E-12 | MetaMask/Phantom users can use ERC-4337 session keys (`docs/technical_reference.md:100`, `docs/protocol_layer.md:119`) | A MetaMask EOA is not an ERC-4337 smart account by default. ERC-4337 requires a smart account and bundler/EntryPoint flow. EIP-7702 (Final) allows an EOA to delegate to code and is the relevant path for EOA-based permissions, but the repo does not mention it. | **Contradicted / Incomplete** | https://eips.ethereum.org/EIPS/eip-7702 |
| E-13 | GMX v2 deposits/withdrawals are async, keeper-fulfilled (`docs/protocol_layer.md:95-105`, `docs/technical_reference.md:495-499`) | Confirmed: GMX v2 uses `ExchangeRouter` request transactions fulfilled by keepers in a later transaction. The repo’s async `AWAITING_FULFILLMENT` model is appropriate. The error is using GMX as an automated liquidity source without a withdrawal action. | **Verified (model) / Internal conflict (use)** | https://docs.gmx.io/ |
| E-14 | Arbitrum gas uses `NodeInterface` / `ArbGasInfo` (`docs/technical_reference.md:504`, `docs/cross_chain_ecosystem.md:97-99`) | `NodeInterface` is a virtual contract at `0xc8`, accessible only over RPC; `gasEstimateL1Component(address to, bool contractCreation, bytes calldata data)` exists. `ArbGasInfo` is at `0x6c` with `getL1BaseFeeEstimate()`. | **Partially verified** — API exists; repo elides required parameters | https://docs.arbitrum.io/build-decentralized-apps/nodeinterface |
| E-15 | Base gas uses `GasPriceOracle.l1BaseFee × size × scalar` (`docs/cross_chain_ecosystem.md:87-89`) | Base `GasPriceOracle` is at `0x420000000000000000000000000000000000000F`. Current fee formula is `gasUsed × (baseFee + priorityFee) + l1Fee + operatorFee`, with `getL1Fee(bytes)`, `getL1FeeUpperBound(uint256)`, `getOperatorFee(gasUsed)`. | **Contradicted (formula)** | https://docs.base.org/ |
| E-16 | Helius webhooks provide sub-second updates, 10-minute sweep fallback (`docs/portfolio_state_manager.md:113-143`) | Helius reports 200–500ms latency from confirmation, 1 credit/event, up to 100k addresses per webhook, 24h retry, and a static `authHeader` (no signed payload). “Sub-second” is supported. | **Verified** | https://www.helius.dev/docs/webhooks |
| E-17 | Alchemy Notify webhooks; no signature verification specified (`docs/portfolio_state_manager.md:113-129`) | Address Activity webhooks support <100k wallets; HMAC SHA-256 `X-Alchemy-Signature` with a signing key; retries 10 min (Free/PAYG) or 1 hour (Enterprise). The repo does not specify signature verification. | **Partially verified** | https://www.alchemy.com/docs/notify |
| E-18 | CoinGecko Pro batches up to 250 tokens per call (`docs/financial_knowledge_layer.md:184`) | `/simple/price` accepts up to 515 IDs per request; Demo plan is 100 calls/min; paid-plan limits vary; all errors count toward the limit. 250 is under the cap, not a blocker, but stale. | **Partially verified** | https://docs.coingecko.com/reference/simple-price |
| E-19 | DeFiLlama provides protocol TVL/safety data; daily pull (`docs/financial_knowledge_layer.md:212`) | Separate Free (`api.llama.fi`, no auth, standard rate limit) and Pro (`pro-api.llama.fi/{KEY}`, $300/mo, higher limits). Free has no published hard limit or SLA; TVL/protocol/historical endpoints are available free. | **Partially verified** | https://docs.llama.fi/ |
| E-20 | “Jupiter V6 Aggregator API” (`docs/engine3_yield_opportunity.md:49`, `docs/protocol_layer.md:36`) | Current product is “Jupiter Swap API” with `/order` and `/build`, requiring a Developer Platform API key. “V6 Aggregator API” is outdated nomenclature. | **Contradicted (naming/API)** | https://dev.jup.ag/docs/swap-api |
| E-21 | `OneInchUniswapXAdapter` handles aggregator-selected route execution (`docs/protocol_layer.md:30-36`) | UniswapX is a permissionless auction-based intent protocol: swappers sign orders using Permit2, fillers compete to settle, gas-free for swappers, no cost for failed swaps. It is not an aggregator API and needs order signing/broadcast/filler-wait states. | **Contradicted (model)** | https://docs.uniswap.org/contracts/uniswapx/overview |
| E-22 | 1inch integration via developer API (`docs/execution_router.md:23-25`) | 1inch documentation has moved to `https://business.1inch.com/portal/documentation/overview/welcome`; `portal.1inch.dev`/`docs.1inch.dev` were unreachable. API version, auth, and endpoint contract are not specified in the repo. | **Unverified** | https://business.1inch.com/portal/documentation/overview/welcome |
| E-23 | Primary Copilot model `GPT-4.1-mini` (`docs/engine5_user_intelligence.md:138`) | OpenAI lists `gpt-4.1-mini` as **Deprecated** (retirement announced). Pinning it in a new design is not acceptable. | **Contradicted (lifecycle)** | https://platform.openai.com/docs/deprecations |
| E-24 | Fallback Copilot model `Gemini 2.5 Flash` (`docs/engine5_user_intelligence.md:139`) | Google lists `gemini-2.5-flash` as **deprecated**; API access is limited to existing users until 1 Dec 2026, after which requests auto-route to `gemini-3.5-flash-lite`. | **Contradicted (lifecycle)** | https://ai.google.dev/gemini-api/docs/deprecations |
| E-25 | HyperVerge handles KYC at WazirX scale (~30k checks/day) and provides webhook HMAC (`docs/identity_kyc.md:15,45-47`) | HyperVerge’s site confirms document verification, passive liveness/face auth, video KYC, AML screening/ongoing monitoring, Aadhaar/PAN/NSDL, and a crypto vertical. Developer docs (`docs.hyperverge.co`, `dev.hyperverge.co`) returned 502/transport errors; webhook signature scheme, API version, and sandbox details could not be verified. WazirX/30k claims are marketing. | **Partially verified / Unverified** | https://hyperverge.co/ |
| E-26 | Chainlink/Pyth cross-validation at 1.5% (`core_documents/technical_reference.md:60`); oracle addresses | Chainlink and Pyth both provide price feeds, but the repo does not specify feed addresses, decimals, heartbeat/staleness thresholds, or L2 sequencer-uptime handling. The attempted Chainlink feed-address page returned 404. | **Unverified / Incomplete** | https://docs.chain.link/data-feeds |
| E-27 | LI.FI / Socket bridge aggregator for non-USDC (`docs/orchestrator_planner.md:196`) | Both are real bridge aggregators, but no MVP adapter exists in the Protocol Layer; the repo itself marks it Phase 2. The runtime branch should not be reachable. | **Verified (vendor) / Internal conflict (scope)** | https://li.fi/ |

### 5.1 External verification conclusions

1. **CCTP and x402 are the two hard integration failures.** Both are core to the product (cross-chain execution and paid Copilot), and both are specified against obsolete APIs. These are not “nice to fix later”; they would fail at runtime.
2. **Delegation is a product-model failure, not just an API bug.** The docs promise non-custodial automated management of external wallets, but the underlying protocols do not grant that authority through token delegates or session keys alone. This needs a product/architecture decision (embedded custody, smart-account migration, EIP-7702, or restricted Path B scope).
3. **Data providers are broadly viable.** Alchemy, Helius, CoinGecko, and DeFiLlama can support the design, but the repo must specify plan tiers, rate limits, webhook signatures, and failure behaviour. The current “sub-second” and “batch 250” claims are directionally fine but not contracts.
4. **Gas models are directionally right but formula-level wrong on Base.** Arbitrum’s API exists; Base’s current formula includes an operator fee and should use the precompile’s `getL1Fee` methods, not a hand-rolled scalar.
5. **LLM pins are obsolete.** This is easy to fix but must be fixed before build; otherwise the Copilot will break on the provider’s shutdown date.
6. **HyperVerge remains a vendor-risk item.** Capabilities are plausible, but the developer contract is unverified. Do not put “HMAC verified” or “30k checks/day” in a BE report without a sandbox test and a vendor document.

---

## 6. Architectural and technical feasibility

### 6.1 The execution pipeline is not schedulable as written

The repository describes four different orderings:

1. `E6 → Policy → Orchestrator (build+simulate) → Router` (`core_documents/system_architecture.md:60,137-142`).
2. `E6 → Orchestrator/Planner (build+simulate) → Policy → execute` (`docs/orchestrator_planner.md:86,95-99`).
3. `E6 → Policy` as a gate “between Engine 6 and the Orchestrator,” with simulation “prior to Policy” (`docs/technical_reference.md:456,478`).
4. `E6 → Router → Protocol Layer → Orchestrator?` (`docs/execution_router.md:9`; `docs/engine6_execution_intelligence.md:31` says intent goes to Orchestrator).

No ordering satisfies all four. The minimal coherent order that satisfies the actual data dependencies is:

```
Engine 6 (macro intent)
  → Execution Router (venue/pool/split/CCTP parameters)
  → Planner (build unsigned bytes)
  → Simulation (eth_call / simulateTransaction)
  → Policy Engine (evaluate SimulationPassedPackage + intent)
  → CCE (sign, broadcast, confirm)
  → PSM (sync)
```

If policy must run before any simulation (for cost/latency), it can run twice: a cheap pre-check on the intent, then a final authoritative check on the simulated package. That is a legitimate design, but it must be stated explicitly and the check catalog must distinguish “pre-check” from “final check.”

**Feasibility:** feasible, but only after a single canonical sequence is chosen. The current documents cannot be implemented without an arbitrary choice.

### 6.2 Delegated execution is the largest product-feasibility risk

The docs claim Path B users can connect MetaMask/Phantom and let CrestFlow execute autonomously. The actual authority model is:

| Position type | Who can move it? | Does the repo’s delegation work? |
|---|---|---|
| Embedded Turnkey wallet (Path A) | CrestFlow’s TEE key | Yes — full custody/execution. |
| EVM EOA token balance | EOA, or a spender with ERC-20 allowance | Partial — allowance can move tokens, but not manage positions. |
| Aave V3 supply position | The supplier (`msg.sender`), via aTokens | **No** — no general position delegation; `withdraw` burns the supplier’s aTokens. |
| Uniswap V3 LP NFT | NFT owner or approved operator | **No** — session key/token delegate is not an NFT operator unless explicitly approved or the NFT is held by a smart account. |
| GMX GM position | Account that holds the GM token / request owner | **No** — async keeper model; no token-delegate path. |
| Solana SPL token balance | Owner or SPL delegate (transfer/burn only) | Partial — delegate can move tokens, not sign lending/LP instructions. |
| Kamino/MarginFi position | Position owner / program authority | **No** — token delegate does not confer program authority. |

**Feasible options:**

1. **Restrict Path B to “deposit and delegate tokens, not positions.”** The user delegates USDC/ETH; CrestFlow opens positions in its own embedded wallet or a vault. This is a custody change and must be disclosed.
2. **Require a smart account.** The user migrates to an ERC-4337 smart account (or EIP-7702 delegation) that holds the positions and enforces scoped permissions. This is a real onboarding step, not a signature.
3. **Use protocol-specific position transfer/approval.** Some protocols support position NFTs or operator approvals; Aave V3 does not have a general delegation primitive. This would be protocol-by-protocol and is not a general solution.
4. **Keep Path B read-only.** The user gets analysis and manual execution; autonomous rebalancing is Path A only.

The repo must choose one and update `docs/technical_reference.md:95-105`, `docs/protocol_layer.md:109-123`, `docs/cross_chain_ecosystem.md:49`, `docs/portfolio_state_analysis.md:23`, and `docs/user_onboarding.md:68-80`.

### 6.3 CCTP is feasible but the adapter is wrong

CCTP V2 is a good fit for USDC-only cross-chain moves. The required corrections are mechanical but non-optional:

- Use the V2 `depositForBurn` signature with `destinationCaller`, `maxFee`, and `minFinalityThreshold`.
- Quote the Fast fee per route; do not assume a flat fee. Deduct it from the expected minted amount.
- Make `mintRecipient` a parameter. For Path A it can be the embedded wallet; for Path B it must be the external wallet or the smart account, by explicit policy.
- Set `destinationCaller` deliberately: zero allows any caller to mint; a specific address restricts minting to CrestFlow. The repo must state which and why.
- Do not claim “1:1” for Fast transfers; the received amount is net of fee.
- Poll the correct Iris attestation API/version and handle the “attestation ready but mint delayed” state.
- Clarify ownership: Router selects speed/domain; Planner builds/simulates; CCE signs/broadcasts; PSM reconciles. One owner per step.

**Feasibility:** high, once corrected. The current 4-arg call is a hard failure.

### 6.4 x402 is feasible but the integration is a different protocol

The current design is closer to a “pay with a tx hash” gateway than to x402. To use x402 V2:

- Return `402` with a `PAYMENT-REQUIRED` header carrying `PaymentRequirements` (CAIP-2 network, asset address, amount, payTo, scheme, timeout).
- Accept `PAYMENT-SIGNATURE` with the signed EIP-3009 authorization (or Permit2 payload), not a transaction hash.
- Use a facilitator (official example `https://x402.org/facilitator`) to verify/settle; or implement the facilitator role yourself if you want custody of settlement.
- Return `PAYMENT-RESPONSE` with the settlement result.
- Use CAIP-2 network IDs and token contract addresses, not `"ethereum"`/`"base"`/`"USDC"`.
- Model replay with the authorization’s `validAfter`/`validBefore` and the facilitator’s nonce, not a custom 15-minute/24-hour Redis nonce.

**Feasibility:** high, but it is a rewrite of the middleware, not a header rename.

### 6.5 Policy is feasible but must be one catalog

The policy engine is the right idea: deterministic, fail-closed, externalized from the LLM. The implementation is blocked by:

- two catalogs;
- undefined fields (`compositeRiskScore`, `allowedActionTypes`);
- a category that may not exist (`EXECUTION`);
- allowlist/routing order;
- a dead GMX branch;
- missing gas/bridge payback and cooldown checks if the master reference is authoritative;
- the short-circuit contradiction.

**Recommendation:** make `docs/policy_engine.md` the single source of truth for check predicates, and make the master reference link to it rather than restating a different list. Add a typed `PolicyInput` that contains every field the checks read. Add a `policyVersion` and a golden-test table (input → expected decision) for each check.

### 6.6 GMX automation is not feasible in the current scope

GMX v2’s async keeper model is real and the repo handles it correctly for *manual* positions. The problem is the contradiction:

- Engine 4/6 say GMX positions generate `MANUAL_REVIEW`, not automated LP actions.
- Engine 6 still lists GMX as the last-resort source in the liquidation waterfall.
- The Router still selects GM pools.
- Policy still has GMX-specific branches.

**Recommendation:** remove GMX from the automated waterfall and Router Job 3 for MVP, or add a first-class async `GMX_WITHDRAW` action with request/fulfillment states and compensation. Do not leave dead branches that suggest automation exists.

### 6.7 Data/oracle/cache design is feasible but underspecified

- **Prices:** CoinGecko + on-chain oracle is reasonable; the repo must define which oracle (Chainlink/Pyth), feed addresses, decimals, staleness/heartbeat, and L2 sequencer-uptime handling.
- **Divergence:** 5% CoinGecko-vs-oracle and 1.5% Chainlink-vs-Pyth are different guards. Define both, their order, and which one blocks signing.
- **Cache:** pick one TTL table. For execution safety, 30s prices and 300s pool data are defensible; 60s prices may be too stale for slippage checks. Historical data should have an explicit retention/refresh policy, not “indefinite.”
- **Indexer:** 500ms/1500ms vs 1s is an implementation detail, but the timeout (10s vs 15s) and `dataQuality` behaviour are contracts. Pick one and propagate it to Engine 1.
- **Rate limits:** CoinGecko 515 IDs/request is fine; DeFiLlama Free has no SLA; Alchemy/Helius webhook limits and retries must be in the operational plan.

### 6.8 Gas accounting is mostly feasible, with two corrections

- **Arbitrum:** `NodeInterface` at `0xc8` and `ArbGasInfo` at `0x6c` are real. The repo must pass `(to, contractCreation, data)` to `gasEstimateL1Component` and handle RPC-only access.
- **Base:** use `GasPriceOracle` at `0x420...000F`; include `operatorFee`; prefer `getL1Fee(bytes)`/`getL1FeeUpperBound` over a hand-rolled scalar formula.
- **Ethereum:** EIP-1559 estimate is fine; include blob/data fees only if the transaction uses blobs (it does not).
- **Solana:** base fee + priority fee + compute-unit limit is fine; the repo should specify the priority-fee strategy and re-sign path.

### 6.9 LLM/Copilot is feasible but the model pins are obsolete

Replace `GPT-4.1-mini` / `Gemini 2.5 Flash` with a current small model and a current fallback, and add:

- a model-version configuration (not hard-coded);
- an evaluation set for intent classification and numeric faithfulness;
- a cost/latency budget per intent;
- a provider-failure fallback that does not change the user’s financial result;
- a guardrail against prompt injection via portfolio metadata.

### 6.10 Scale, security, and compliance

- **Scale:** 4 chains × per-user addresses; Alchemy Address Activity supports <100k wallets per webhook, Helius up to 100k addresses per webhook, CoinGecko 515 IDs/request. These are workable for an MVP but need sharding/batching and a queue.
- **Security:** TEE signing + Turnkey policies is strong, but webhook signature verification, replay protection, idempotency, nonce management, and delegation revocation are missing or inconsistent.
- **Compliance:** KYC state machine, tier basis, audit categories, sanctions rescreening, and Travel Rule timing must be one coherent model. The current docs overstate FATF compliance and under-specify the KYC persistence fields.

---

## 7. Missing specifications and unresolved decisions

The following are not “nice to have” documentation gaps. Each one prevents a correct BE implementation or a defensible UML model. They are ordered by dependency.

### 7.1 Execution and policy contracts

1. **Canonical execution sequence and ownership.** Decide whether the order is `E6 → Router → Planner/simulate → Policy → CCE` or a two-pass policy (`pre-check → simulate → final check`). Assign one owner to each transition: intent, route, build, simulate, approve, sign, broadcast, confirm, reconcile. Today, four documents give four orders.
2. **Single policy catalog.** Make `docs/policy_engine.md` the source of truth or replace it with a new canonical spec. Define each check’s typed input, predicate, failure code, whether it is pre- or post-simulation, and whether it short-circuits. Reconcile the 9-check list with the master reference.
3. **Typed `PolicyInput`.** The policy engine needs a single object containing `user.kycStatus`, `user.kycTier`, `addressRegistry`, `riskSnapshot`, `goalProfile`, `plan.actions`, `plan.totalValueUsd`, `plan.source`, `autopilotConfig`, and the simulated outputs. Several of these fields do not exist today.
4. **Canonical domain schemas.** Reconcile and version: `RiskSnapshot` (`riskScore` vs `compositeRiskScore`), `GoalProfile` (add `allowedActionTypes` or remove Check 9), `ExecutionIntent` (protocol, pool, source address, recipient, approvals, deadline, units), `SimulationPassedPackage`, `ExecutionInstructions`, `StrategyPlan`, `YieldOpportunitySnapshot`, `AuditEntry`.
5. **Policy/audit error taxonomy.** Define category, action, severity, and user-facing message for every failure. Ensure the audit category enum includes every category the policy queries (`EXECUTION` at minimum).
6. **Policy test vectors.** Add a table of at least one passing and one failing case per check, including boundary values (tier limit exactly, slippage exactly at cap, risk exactly at threshold, cooldown exactly 7 days).

### 7.2 Custody, delegation, and onboarding

7. **Delegation/custody decision.** Choose Path B scope: read-only, token-only delegation, smart-account migration, EIP-7702, or protocol-specific position transfer. Document which position types are automatable per path.
8. **Smart-account provisioning.** If ERC-4337 or EIP-7702 is chosen, specify account creation, bundler/paymaster, session-key issuance, revocation, and recovery. MetaMask/Phantom EOA onboarding is insufficient.
9. **Solana authority model.** Specify which program authorities are needed for Kamino/MarginFi/Raydium, and why SPL `delegate` is or is not sufficient. Add the exact instructions and account metas.
10. **KYC state machine.** Canonical statuses (`PENDING`, `APPROVED`, `REJECTED`, `RESUBMISSION_REQUESTED`, `SUSPENDED`, `EXPIRED`), tier basis, transition triggers, webhook mapping, and rescreen cadence.
11. **User persistence model.** Decide `sub` vs `googleId`, add `kycStatus`/`kycTier`, define the address-scope table, and specify encryption/retention for PII.
12. **Onboarding sequence.** Add the KYC step and tier assignment to `docs/user_onboarding.md`; reconcile it with `core_documents/system_architecture.md:121`. Define the failed-KYC and resubmission UX.

### 7.3 Cross-chain and payments

13. **CCTP V2 adapter spec.** Exact ABI, `destinationCaller` policy, `maxFee` quoting, `minFinalityThreshold` per speed, `mintRecipient` policy, attestation API version, timeout/retry, and failure/compensation states.
14. **Bridge ownership.** One owner for burn, attestation, mint, and reconciliation. Remove duplicate ownership from Orchestrator/CCE/PSM.
15. **Non-USDC bridge decision.** Either remove `BRIDGE_AGGREGATOR` from MVP and fail closed, or specify the aggregator adapter, supported assets/chains, quote model, slippage, and recovery.
16. **x402 V2 spec.** `PaymentRequirements` schema, CAIP-2 networks, asset addresses, `PAYMENT-REQUIRED`/`PAYMENT-SIGNATURE`/`PAYMENT-RESPONSE`, EIP-3009/Permit2 payload, facilitator URL, settlement/replay model, pricing, and refund/failure behaviour.
17. **Facilitator decision.** Use the official facilitator, a reputable third-party facilitator, or self-host. Do not depend on an unreachable host with no published contract.

### 7.4 Metrics and data

18. **HHI definition and bands.** One formula, one band set, one blocking rule. State whether bands apply to asset, protocol, and chain HHI identically, and whether they are persona-scaled.
19. **TOPSIS definition.** One criteria set, one weight matrix per persona, one normalization/ideal-point method, and one treatment of missing criteria (e.g., IL for GMX).
20. **APY definition.** One window (7-day vs 30-day), one weighting scheme, one field name, and one rule for insufficient history.
21. **Drift/cooldown definition.** One urgency table, one tolerance table, one cooldown owner, and one rule for the overlap at 2–3%.
22. **Oracle spec.** Feed addresses, decimals, heartbeat/staleness, deviation threshold, L2 sequencer-uptime feed, fallback oracle, and the exact condition that blocks signing.
23. **Cache/refresh policy.** One TTL table, stale-while-revalidate behaviour, failure behaviour, and historical retention.
24. **Indexer/webhook contract.** Signature verification, event IDs, idempotency, ordering, retry, `dataQuality` enum, and reconciliation cadence.
25. **Provider plan/SLA.** CoinGecko plan and rate limit, DeFiLlama Free/Pro and key handling, Alchemy/Helius plan limits, and RPC provider failover.

### 7.5 Execution mechanics and operations

26. **Nonce manager.** One component owns per-chain nonce allocation, gap detection, replacement, and cancellation. `core_documents/system_architecture.md:226` admits this is unresolved.
27. **Gas/fee spec.** Base operator fee, Arbitrum NodeInterface parameters, Solana priority-fee strategy, EIP-1559 bump policy, and the exact `gasPriceCap` comparison.
28. **Approval spec.** Standard `approve`, EIP-2612 `permit`, Permit2 `approve`, and Permit2 `permit` — when each is used, who submits it, and how it is simulated.
29. **Saga/compensation spec.** Per-action compensation for swaps, bridges, lending, LP, and GMX; what is reversible, what is not; how in-flight keeper actions are handled.
30. **API surface.** OpenAPI/REST contract for auth, portfolio, risk, yield, strategy, execution, audit, and x402 endpoints; idempotency keys; pagination; error taxonomy; rate limits.
31. **Persistence schema.** Tables for users, identities, wallets, address scopes, positions, snapshots, risk snapshots, opportunities, plans, executions, policy decisions, audit entries, payments; indexes, retention, partitioning, and PII encryption.
32. **Observability.** SLOs for scan latency, policy latency, simulation success, broadcast confirmation, bridge completion, and webhook lag; dashboards, alerts, and runbooks.
33. **Security threat model.** Webhook forgery, replay, prompt injection, key compromise, delegated-authority abuse, malicious aggregator routes, oracle manipulation, and supply-chain risk.
34. **Regulatory decisions.** Custody classification, sanctions screening, Travel Rule timing, jurisdiction, terms of service, and user disclosures. These are legal/product decisions, not documentation fixes.

### 7.6 Documentation governance

35. **Source-of-truth hierarchy.** Declare which document wins for each domain. Suggested: `docs/policy_engine.md` for policy; `docs/technical_reference.md` for cross-cutting contracts; component docs for their own schemas; `core_documents/system_architecture.md` for topology only.
36. **Versioning/deprecation.** Add `status` (draft/authoritative/superseded), `version`, `last-reviewed`, and `supersedes` headers. Mark `docs/integration_audit_report.md` as historical.
37. **Traceability matrix.** Map each requirement/decision to the documents and tests that implement it. Without this, the same contradictions will reappear.

---

## 8. Severity and priority register

**Priority definitions**

- **P0:** Resolve before any implementation or execution/UML modelling. A wrong choice propagates into contracts.
- **P1:** Resolve before building the affected module; otherwise expect rework or unsafe behaviour.
- **P2:** Resolve before launch or before finalizing the BE report; operational/quality risk.
- **P3:** Documentation hygiene; can be batched.

| ID | Finding | Severity | Priority | Blocks | Primary evidence |
|---|---|---|---|---|---|
| C-01 | Execution order is circular/defined four ways | Critical | P0 | Execution UML, Orchestrator, Policy | `system_architecture.md:60,137-142`; `policy_engine.md:11`; `orchestrator_planner.md:86,95-99`; `technical_reference.md:456,478` |
| C-02 | Two policy catalogs; undefined fields | Critical | P0 | Policy implementation, tests, BE report | `policy_engine.md:15-197`; `technical_reference.md:444-454`; `engine2:283`; `engine5:96-108` |
| C-03 | Delegated Path B infeasible for positions | Critical | P0 | Custody model, onboarding, execution | `technical_reference.md:100-101`; `protocol_layer.md:119-121`; Aave/Uniswap/SPL docs |
| C-04 | CCTP V2 calldata/recipient wrong | Critical | P0 | Cross-chain execution | `protocol_layer.md:134,146`; CCTP V2 docs |
| C-05 | x402 V2 incompatible | Critical | P0 | Paid Copilot/agent access | `audit_layer_x402.md:277-327`; `technical_reference.md:632`; x402 docs |
| C-06 | `ExecutionIntent` schema insufficient/units | Critical | P0 | Router, Policy, Planner, class model | `engine6:196-221`; `technical_reference.md:645` |
| H-01 | Router placed after build/sim; “does not route” | High | P0 | Execution sequence/UML | `execution_router.md:9-13`; `system_architecture.md:139-140` |
| H-02 | Allowlist/adapter/router venue mismatch | High | P1 | Router, Policy, adapters | `policy_engine.md:107-114`; `execution_router.md:23-25,30`; `protocol_layer.md:23-58` |
| H-03 | Permit2/Aave approval claim wrong | High | P1 | Approval flow, onboarding | `protocol_layer.md:204`; Aave/Permit2 docs |
| H-04 | GMX dead paths/waterfall | High | P1 | Engine 6, Router, Policy | `engine4:174-186`; `engine6:71-73,93-100`; `execution_router.md:75-83` |
| H-05 | Non-USDC bridge runtime vs no adapter | High | P1 | Execution scope | `system_architecture.md:223`; `protocol_layer.md:225`; `engine6:210` |
| H-06 | HHI thresholds/blocking conflict | High | P1 | Engines 1/2/4, Policy | `engine1:228-230`; `engine2:90-93`; `engine4:135-139`; `technical_reference.md:188-190` |
| H-07 | TOPSIS criteria/weights/APY conflict | High | P1 | Engine 3, rankings, BE report | `engine3:64-65,140-146,227`; `technical_reference.md:254,279-286` |
| H-08 | Oracle/cache/TTL conflicts | High | P1 | FKL, execution safety | `FKL:103-109,186`; `technical_reference.md:142-144`; `system_architecture.md:189-191` |
| H-09 | KYC enum/tier basis mismatch | High | P1 | Onboarding, Policy, audit | `identity_kyc.md:49,62`; `technical_reference.md:113,119-120` |
| H-10 | Identity anchor schema mismatch | High | P1 | Auth, DB schema | `user_onboarding.md:19,21`; `technical_reference.md:63-65` |
| H-11 | Both LLM pins deprecated | High | P1 | Copilot | `engine5:138-139`; OpenAI/Gemini deprecation pages |
| H-12 | E3→E4 dependency/event mismatch | High | P1 | Engine 3/4 scheduling | `system_architecture.md:163`; `engine3:260`; `engine4:5,18` |
| H-13 | Audit taxonomy/ownership conflicts | High | P1 | Audit schema, compliance queries | `audit_layer_x402.md:16-30`; `technical_reference.md:610`; `identity_kyc.md:97` |
| H-14 | Indexer sync timeout/degradation mismatch | High | P2 | PSM, Engine 1 | `PSM:26-28`; `technical_reference.md:597` |
| M-01 | E2/E3 parallel vs sequential pipeline | Medium | P2 | Scheduler, event graph | `technical_reference.md:36-37`; `engine1:268`; `system_architecture.md:56,128` |
| M-02 | Cooldown duplicated/mislabelled Check 9 | Medium | P2 | Engine 4, Policy | `engine4:232-239`; `technical_reference.md:454,362`; `policy_engine.md:166-197` |
| M-03 | Drift/tolerance overlap at 2–3% | Medium | P2 | Engine 4 | `engine4:149-158` |
| M-04 | CCTP timing/fee prose imprecise | Medium | P2 | UX/cost model | `engine4:198-199`; `execution_router.md:125-129`; `PSM:172` |
| M-05 | Base gas formula/Arbitrum params wrong | Medium | P2 | CCE, fee estimation | `CCE:87-99`; Base/Arbitrum docs |
| M-06 | HyperVerge contract/claims unverified | Medium | P2 | KYC integration/BE report | `identity_kyc.md:15,45-47`; vendor docs unreachable |
| M-07 | Literature citation errors | Medium | P2 | Literature chapter/report | `literature_survey.md:226,542,581`; CSV counts |
| M-08 | Integration report falsely claims zero contradictions | Medium | P2 | Governance/report credibility | `integration_audit_report.md:11-13` |
| M-09 | Stale/mislabeled architecture assets | Medium | P3 | Documentation/UML inputs | `system_architecture.md:9-11`; PNG inspection |
| M-10 | Shared `poolType` ownership/count conflict | Medium | P2 | Engines/Router/Policy | `system_architecture.md:113,222`; `engine4:289`; `engine6:241`; `FKL:152` |
| M-11 | Provider plan/rate-limit/SLA unspecified | Medium | P2 | FKL/operations | `FKL:184,212`; CoinGecko/DeFiLlama docs |
| M-12 | Aggregator API/order lifecycle outdated | Medium | P2 | Router/adapters | `execution_router.md:23-25`; `protocol_layer.md:30-36`; Jupiter/UniswapX docs |
| M-13 | FATF Travel Rule overstatement | Medium | P2 | Compliance/BE report | `core_documents/technical_reference.md:52,157`; `identity_kyc.md:89` |
| M-14 | API/DB/nonce/observability specs missing | Medium | P2 | Backend design/UML | System-wide; `system_architecture.md:226` |
| M-15 | No source-of-truth/versioning | Medium | P2 | All docs/UML traceability | Repository-wide |
| L-01 | Router says “Does not route” | Low | P3 | Readability | `execution_router.md:13` |
| L-02 | Absolute/broken asset links | Low | P3 | Docs | `system_architecture.md:9-11` |
| L-03 | CSV duplicate/section-header rows | Low | P3 | Literature count | CSV rows; `literature_survey.md:544,573` |
| L-04 | Encoding/typos (e.g. “? shared state”) | Low | P3 | Readability | `system_architecture.md:83`; various |

### 8.1 Priority summary

- **P0 (7):** C-01 through C-06 and H-01. These are the "do not build/UML yet" set.
- **P1 (12):** H-02 through H-13. These affect module design and external integrations.
- **P2 (15):** H-14 plus M-01 through M-15, excluding M-09 (P3). These affect correctness/operations/report quality.
- **P3 (5):** M-09 and L-01 through L-04. Documentation hygiene.

> Note: the P0 count treats H-01 as P0 because the Router ordering is inseparable from C-01. The critical/high labels are about consequence; priority is about sequencing.

---

## 9. Prioritized remediation plan

The plan is sequenced by dependency, not by document order. Do not start with copy-editing; the first phase is decision-making.

### Phase 0 — Decide and freeze (3–5 person-days)

| Task | Output | Owner | Exit criterion |
|---|---|---|---|
| R0.1 Create `CANONICAL_DECISIONS.md` | A numbered decision log (D1–D12) with rationale and status | Architect + PM | Every P0 issue has an owner and a decision date |
| R0.2 Declare document precedence | A short “source of truth” table: policy, schemas, topology, metrics, operations | Architect | No document is allowed to restate a contract it does not own |
| R0.3 Freeze MVP scope | Path B scope, GMX automation, non-USDC bridge, x402 paid endpoints | Product + Architect | Scope statement signed off |
| R0.4 Mark stale material | `integration_audit_report.md` = historical; “after pivot” PNG = deprecated; link the real diagram | Docs owner | No stale file is presented as current |
| R0.5 Create a traceability matrix | Requirement → document → test/decision | PM | Every P0/P1 finding has a row |

**Key decisions to make in Phase 0**

- **D1:** Canonical execution order.
- **D2:** Policy catalog and whether it runs once or twice.
- **D3:** Path B custody/delegation model.
- **D4:** CCTP V2 parameters and mint-recipient policy.
- **D5:** x402 V2 facilitator and settlement model.
- **D6:** GMX in or out of automated MVP.
- **D7:** Non-USDC bridge in or out of MVP.
- **D8:** HHI band/blocking rule.
- **D9:** TOPSIS criteria/weights and APY window.
- **D10:** Oracle set and divergence/staleness gates.
- **D11:** KYC state machine/tier basis and Travel Rule timing.
- **D12:** LLM provider/model and fallback.

### Phase 1 — Fix the P0 execution contract (6–9 person-days)

| Task | What to change | Depends on | Exit criterion |
|---|---|---|---|
| R1.1 Execution sequence | Rewrite the flow in `system_architecture.md`, `technical_reference.md`, `orchestrator_planner.md`, `execution_router.md`, `engine6` to one order. Add a step-ownership table. | D1 | One sequence; every document points to it |
| R1.2 Policy catalog | Make `policy_engine.md` canonical; add `PolicyInput`; define all 9 checks, predicates, failure codes, pre/post-simulation stage, and short-circuit semantics. | D2 | A developer can implement each check without reading another doc |
| R1.3 Delegation model | Choose Path B scope; document per-position authority; add smart-account/EIP-7702 or restrict to read-only/token-only. | D3 | A position-type authority matrix exists |
| R1.4 CCTP V2 adapter | Replace the 4-arg call; add `destinationCaller`, `maxFee`, `minFinalityThreshold`, recipient policy, fee quote, attestation API, timeout/retry, ownership. | D4 | A CCTP transfer can be built/simulated on testnet |
| R1.5 x402 V2 | Replace headers with `PAYMENT-REQUIRED`/`PAYMENT-SIGNATURE`/`PAYMENT-RESPONSE`; use CAIP-2 + EIP-3009/Permit2; choose facilitator. | D5 | A 402 → pay → retry flow works on testnet |
| R1.6 Schema reconciliation | Rename/add fields: `riskScore`, `allowedActionTypes`, `ExecutionIntent` fields, integer units, `SimulationPassedPackage`, `ExecutionInstructions`. | D1, D2 | TypeScript interfaces compile and round-trip test data |

**Phase 1 exit gate:** no P0 finding open; a single end-to-end happy-path sequence (one swap + one Aave deposit + one CCTP bridge) can be described with exact fields and calls.

### Phase 2 — Fix P1 integrations and metrics (8–12 person-days)

| Task | What to change | Depends on | Exit criterion |
|---|---|---|---|
| R2.1 Protocol/venue matrix | One table: chain × action × allowlisted protocol × adapter × router venue. Add/remove 1inch, UniswapX, Camelot, Curve. | R1.1 | Policy, Router, Protocol Layer reference the same matrix |
| R2.2 GMX scope | Remove GMX from automated waterfall/Router/Policy, or add async withdraw with compensation. | D6 | No dead branch; manual path documented |
| R2.3 Non-USDC bridge | Remove runtime branch or specify adapter + recovery. | D7 | Plan cannot emit an unbuildable action |
| R2.4 HHI reconciliation | One formula/band/blocking rule across Engines 1/2/4/Policy. | D8 | Same portfolio gets the same status everywhere |
| R2.5 TOPSIS/APY | One criteria set/weights/window/field name; define missing-criterion handling. | D9 | Reproducible ranking with a worked example |
| R2.6 Oracle/cache | One TTL table; feed addresses/decimals/heartbeat; divergence precedence; `dataQuality` enum. | D10 | Signing gate conditions are testable |
| R2.7 KYC/audit | One status enum/tier basis; add KYC category; align audit taxonomy; label Travel Rule Phase 3. | D11 | Policy, identity, audit, and onboarding use the same enums |
| R2.8 LLM | Replace deprecated models; config-driven model IDs; eval set; fallback. | D12 | Copilot runs on a supported model with a tested fallback |

### Phase 3 — Operationalize P2 (6–8 person-days)

| Task | Output |
|---|---|
| R3.1 Nonce manager | Component spec: per-chain allocation, gap detection, replacement, cancellation. |
| R3.2 Gas/fee spec | Base `getL1Fee` + operator fee; Arbitrum `gasEstimateL1Component` params; Solana priority fee; EIP-1559 bump. |
| R3.3 Webhook contracts | HMAC/auth header, timestamp/replay, event IDs, idempotency, ordering, retries, reconciliation. |
| R3.4 Provider contracts | CoinGecko/DeFiLlama/Alchemy/Helius plan, limits, keys, failover, cost. |
| R3.5 API/OpenAPI | Auth, portfolio, risk, yield, strategy, execution, audit, x402; idempotency; errors; pagination. |
| R3.6 Persistence | Tables, indexes, retention, partitioning, PII encryption, migrations. |
| R3.7 Observability/security | SLOs, dashboards, alerts, runbooks, threat model, incident response. |
| R3.8 Governance | Doc status/version headers, deprecation policy, traceability maintenance. |

### Phase 4 — Documentation and UML readiness (4–6 person-days)

| Task | Output |
|---|---|
| R4.1 Rewrite affected docs | Remove duplicate contracts; link to canonical sources. |
| R4.2 Fix citations | Correct Kirişci, Barber & Odean, Heimbach; dedupe CSV; add DOI/verification column. |
| R4.3 Rename/link assets | Rename the true post-pivot diagram; delete/deprecate the stale “after pivot” image; fix `.excalidraw` link. |
| R4.4 Produce the UML pack | Component, class, sequence (onboarding, policy, execution, CCTP, x402), state (saga, KYC, delegation), deployment — only after Phase 1. |
| R4.5 Re-audit | Re-run this checklist; require zero P0/P1 and no new contradictions. |

### 9.1 Effort and sequencing summary

| Phase | Effort | Gate |
|---|---:|---|
| Phase 0 — Decide/freeze | 3–5 pd | Decisions D1–D12 recorded |
| Phase 1 — P0 execution contract | 6–9 pd | Happy-path sequence works on testnet |
| Phase 2 — P1 integrations/metrics | 8–12 pd | Protocol matrix + metrics + KYC/audit consistent |
| Phase 3 — P2 operations | 6–8 pd | Nonce/gas/webhooks/API/DB/observability specified |
| Phase 4 — Docs/UML | 4–6 pd | Re-audit clean; UML pack approved |
| **Total** | **27–40 pd** | |

This is a planning estimate, not a commitment. The important point is ordering: **Phase 0 and Phase 1 must precede UML and implementation.**

---

## 10. Readiness for the BE project report and UML documentation

### 10.1 What can be written now, with caveats

| BE report chapter | Readiness | Caveat |
|---|---|---|
| Introduction, problem statement, objectives | **Ready** | No major contradictions. |
| Literature review | **Ready after citation fixes** | Fix Kirişci, Barber & Odean, Heimbach, duplicate count, dead links. |
| System overview and component responsibilities | **Mostly ready** | Component boundaries are coherent; do not present the execution order as final. |
| Technology stack | **Ready with corrections** | CCTP V2, x402 V2, EIP-7702, Base gas, LLM pins need updates. |
| Data sources and FKL | **Mostly ready** | Resolve TTL/oracle/provider tier conflicts. |
| Risk and yield models | **Partially ready** | HHI/TOPSIS/APY definitions conflict; use one canonical set. |
| Security and compliance | **Partially ready** | KYC enums/tiers, FATF timing, audit taxonomy, delegation authority need reconciliation. |
| Execution architecture | **Not ready** | C-01/C-06/H-01; do not write sequence diagrams yet. |
| Cross-chain and payments | **Not ready** | C-04/C-05; current specs would fail. |
| Testing/validation plan | **Not ready** | Needs policy test vectors, simulation harness, testnet plan. |
| Deployment/operations | **Not ready** | Nonce, RPC, webhook, observability, provider SLAs missing. |

### 10.2 UML readiness by diagram type

| UML diagram | Can it be drawn now? | Why / what must precede it |
|---|---|---|
| **Component diagram** | **Yes, at a high level** | The component names and responsibilities are stable. Mark Policy/Router/Orchestrator order as “TBD” rather than drawing a false sequence. |
| **Package/class diagram** | **No** | `RiskSnapshot`, `GoalProfile`, `ExecutionIntent`, `SimulationPassedPackage`, and `ExecutionInstructions` are inconsistent or incomplete. Wait for R1.6. |
| **Sequence — onboarding/KYC** | **No** | KYC is missing from onboarding; status enums and tier basis conflict. Wait for R2.7. |
| **Sequence — policy evaluation** | **No** | Two catalogs, undefined fields, short-circuit contradiction. Wait for R1.2. |
| **Sequence — execution/rebalance** | **No** | Circular order, Router missing, delegation infeasible. Wait for R1.1/R1.3/R1.6. |
| **Sequence — CCTP bridge** | **No** | V1 calldata, recipient hard-coded, ownership overlap. Wait for R1.4. |
| **Sequence — x402 payment** | **No** | Wrong headers/scheme/facilitator. Wait for R1.5. |
| **State machine — execution saga** | **Partially** | The saga states are good, but the transition order and compensation for GMX are unresolved. |
| **State machine — KYC** | **No** | Enums/transitions conflict. |
| **State machine — delegation** | **No** | No canonical delegation model. |
| **Deployment diagram** | **No** | RPC/indexer/TEE/provider topology and nonce ownership are unspecified. |

### 10.3 Recommended UML pack after remediation

1. **Component diagram:** Client → API Gateway → six engines → FKL; execution path E6 → Router → Planner → Policy → CCE → PSM; Audit passive; x402 at the gateway.
2. **Class diagram:** canonical schemas with field types, units, enums, and relationships (`PortfolioSnapshot`, `RiskSnapshot`, `GoalProfile`, `YieldOpportunitySnapshot`, `StrategyPlan`, `ExecutionIntent`, `SimulationPassedPackage`, `ExecutionInstructions`, `AuditEntry`).
3. **Sequence diagrams:** onboarding + KYC + delegation; portfolio scan E1→E2/E3→E4→E5; policy evaluation (pre/post); execution saga; CCTP burn/attest/mint; x402 402/pay/settle.
4. **State machines:** execution saga (including GMX), KYC, delegation, payment.
5. **Deployment diagram:** API/engines, workers/queues, Redis, Postgres, TEE/Turnkey, RPC providers, indexers/webhooks, chains.
6. **Traceability matrix:** requirement → component → schema → test → document section.

### 10.4 What to tell the BE report supervisor

A defensible framing:

> “The repository is a strong domain design, but the execution contract is still being decided. The audit identified six critical and fourteen high-severity issues, concentrated in policy, delegated custody, CCTP, x402, and schema completeness. The BE report can present the domain architecture, data sources, and financial models now; the execution, policy, cross-chain, and payment chapters should be written after the Phase 1 decisions. UML sequence/class diagrams for those areas would currently encode unresolved contradictions.”

---

## 11. Literature and citation audit

### 11.1 Confirmed issues

| # | Location | Claim | Verification | Action |
|---:|---|---|---|---|
| 1 | `literature_survey.md:14,117,148,162` | “54 papers” | CSV has 54 URL-bearing rows but **53 unique titles**; one duplicate at `:544` and `:573`. | Change to 53 unique, or justify two distinct papers and add a unique ID. |
| 2 | `literature_survey.md:542` | Kirişci (2025), *Complex & Intelligent Systems* | DOI `10.1007/s00521-024-10839-2` resolves to **Neural Computing and Applications**. | Correct the journal. |
| 3 | `literature_survey.md:581` | Barber & Odean (2000), “The Courage of Misguided Convictions,” *Journal of Finance* 55(2), 773–806 | That journal/volume/page range is “Trading Is Hazardous to Your Wealth” (DOI `10.1112/0022-1082.00226`). “Courage” is *Financial Analysts Journal* 55(6), 1999 (DOI `10.2469/faj.v55.n6.2313`). | Split into two citations or correct the metadata. |
| 4 | `literature_survey.md:226` | Heimbach et al. “prove over 50% of active LPs experience net negative returns” | The ~50% finding belongs to Loesch et al., cited by Heimbach; Heimbach’s own conclusion is that Uniswap V3 LP can be profitable with appropriate considerations. | Rewrite the attribution and soften “prove.” |
| 5 | `literature_survey.md` Arora/TxLens entries | Semantic Scholar links | Both links returned 404. DOIs/publisher metadata are valid (`10.1016/j.bcra.2026.100496`; `10.1109/icbc67748.2026.11575464`). | Replace dead links with DOI/publisher URLs. |
| 6 | `literature_survey.md` ASRI/Farzulla & Maksakov, Qin et al. | 2026 citations | Could not verify due to 429/404. | Mark “unverified” and re-check before submission. |
| 7 | `integration_audit_report.md:11-13` | “0 unresolved contradictions” | False against current files. | Mark historical; do not cite as assurance. |

### 11.2 Verified citations

| Citation | Verification | Notes |
|---|---|---|
| Pindza (2026) | DOI `10.1007/s42521-026-00213-3`, *Digital Finance* | Verified via Crossref. |
| Ginavar (2026) | DOI `10.2478/picbe-2026-0017` | Abstract confirms 10,000 Monte Carlo paths, Aave V3, CIR + Poisson-exponential jumps. |
| Arora (2026) | DOI `10.1016/j.bcra.2026.100496`, *Blockchain: Research and Applications* | DOI/journal verified; Semantic Scholar link dead. |
| TxLens (2026) | DOI `10.1109/icbc67748.2026.11575464` | DOI verified; Semantic Scholar link dead. |
| Heimbach et al. (2022) | arXiv `2205.08904v2` | Paper is real; the survey’s attribution/conclusion is wrong (see above). |
| Kroo et al. (2024) | Semantic Scholar URL in CSV | Real paper, but duplicated under two titles. |

### 11.3 Citation cleanup checklist

- [ ] Add a `DOI` column and a `verified_on` column to the CSV.
- [ ] Deduplicate the Kroo et al. entry or split it with distinct subtitles/IDs.
- [ ] Correct Kirişci journal.
- [ ] Split Barber & Odean into two references.
- [ ] Rewrite the Heimbach paragraph with the correct attribution.
- [ ] Replace dead Semantic Scholar links with DOI/publisher links.
- [ ] Mark ASRI/Qin as unverified until checked.
- [ ] Recalculate the “54 papers” claims.
- [ ] Add a short “citation verification method” note to the survey.

---

## 12. Appendix A — Official references used

| Area | Reference |
|---|---|
| CCTP overview | https://developers.circle.com/cctp |
| CCTP contract interfaces | https://developers.circle.com/cctp/references/contract-interfaces |
| x402 HTTP 402 | https://docs.x402.org/core-concepts/http-402 |
| x402 exact scheme | https://docs.x402.org/schemes/exact |
| x402 networks/tokens | https://docs.x402.org/core-concepts/network-and-token-support |
| Turnkey policies | https://docs.turnkey.com/ |
| Permit2 | https://docs.uniswap.org/contracts/permit2/overview |
| Uniswap V3 | https://docs.uniswap.org/contracts/v3/ |
| UniswapX | https://docs.uniswap.org/contracts/uniswapx/overview |
| Aave V3 | https://aave.com/docs/aave-v3/ |
| SPL Token | https://solana.com/docs/core/tokens |
| EIP-7702 | https://eips.ethereum.org/EIPS/eip-7702 |
| GMX v2 | https://docs.gmx.io/ |
| Arbitrum NodeInterface | https://docs.arbitrum.io/build-decentralized-apps/nodeinterface |
| Base docs | https://docs.base.org/ |
| Helius webhooks | https://www.helius.dev/docs/webhooks |
| Alchemy Notify | https://www.alchemy.com/docs/notify |
| CoinGecko simple price | https://docs.coingecko.com/reference/simple-price |
| DeFiLlama API | https://docs.llama.fi/ |
| Jupiter Swap API | https://dev.jup.ag/docs/swap-api |
| 1inch business docs | https://business.1inch.com/portal/documentation/overview/welcome |
| OpenAI deprecations | https://platform.openai.com/docs/deprecations |
| Gemini deprecations | https://ai.google.dev/gemini-api/docs/deprecations |
| HyperVerge | https://hyperverge.co/ |
| Chainlink data feeds | https://docs.chain.link/data-feeds |
| LI.FI | https://li.fi/ |

---

## 13. Appendix B — Audit attestation

- **Files read:** all 22 Markdown files, the CSV, and all three PNGs.
- **Files modified:** none.
- **Files deleted/renamed/consolidated:** none.
- **UML diagrams produced:** none.
- **Report created:** `AUDIT_REPORT.md` (this file), assembled from section drafts in the session temp directory. No repository document was changed.
- **External verification date:** 18 November 2026.
- **Unverified items:** HyperVerge developer contract/scale claims, 1inch developer portal, Chainlink feed addresses, ASRI/Qin citations.
- **Confidence:** High on internal contradictions and the CCTP/x402/Aave/Uniswap/SPL/LLM findings; medium on vendor-specific operational claims that could not be verified; low on any statement that depends on unavailable vendor documentation.

### 13.1 Recommended next action

Do **not** start UML or implementation. Convene a 1–2 day architecture decision session covering D1–D12, then execute Phase 1. Re-run this audit after Phase 1; the P0 set should be zero before any sequence/class diagram is treated as final.

