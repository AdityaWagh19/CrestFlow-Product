# CrestFlow — Integration Audit Report (Historical)

**Status:** Historical — superseded, not an assurance document  
**Version:** 1.1 (historical annotation)  
**Original date:** 2026  
**Last annotated:** 18 November 2026  
**Superseded by:** `AUDIT_REPORT.md` (39 findings: 6 Critical, 14 High, 15 Medium, 4 Low) and the remediation set indexed in `docs/DOCUMENT_INDEX.md`.

> **Do not cite this report as evidence that the repository is consistent.** It records an earlier internal pass and its “zero unresolved contradictions” claim was false against the files as they then existed. The authoritative audit is `AUDIT_REPORT.md`; the remediation status is recorded in `REMEDIATION_REPORT.md`.

**Scope (historical):** All 17 `docs/` documents as they existed at the time  
**Chain scope verified:** Ethereum, Solana, Base, Arbitrum  
**Methodology:** Cross-document consistency audit against every component interface, data contract, and cross-cutting assumption.

---

## Executive Summary (Corrected)

The original version of this report claimed that 17 documents were internally consistent and free of contradictions. That claim is withdrawn. A subsequent line-level audit found **39 findings**, including six Critical issues in the policy catalogue, execution order, Path B authority model, CCTP calldata, x402 protocol version, and `ExecutionIntent` schema.

This document is retained only as a record of the earlier integration pass. It must not be used as a readiness signal, a test plan, or a source of truth for any component contract.

**Historical count:** 17 documents audited; 8 cross-cutting open items at the time.  
**Post-audit reality:** 22 Markdown documents and one CSV were present; 39 findings were raised; remediation is tracked in `REMEDIATION_REPORT.md`.

---

## Resolved Bugs — Category Summary

### A. Legacy Chain References (Cosmetic → Structural)

| Document | Issue | Resolution |
|---|---|---|
| All 17 | Algorand/Folks Finance/Tinyman/Wormhole/algod references | Removed entirely. Replaced with Ethereum/Solana/Base/Arbitrum equivalents throughout. |
| Policy Engine | `CHAIN_ALLOWLIST` missing Arbitrum | Added. |
| Policy Engine | `PROTOCOL_ALLOWLIST` missing Arbitrum row | Added with GMX/Camelot. Raydium added to Solana. |
| Execution Router | Aggregator table missing Base/Arbitrum, EVM marked Phase 2 | All four chains populated; Phase 2 tags removed. |
| Cross-Chain Ecosystem | Wormhole treated as primary bridge; CCTP absent | CCTP promoted as MVP default for all USDC moves. |

### B. "EVM = One Chain" Bug (Structural)

This bug recurred in every component that listed protocols, adapters, gas models, webhook subscriptions, or allowlists at the "EVM" level without distinguishing Ethereum, Base, and Arbitrum.

| Document | Manifestation | Fix |
|---|---|---|
| Portfolio State Analysis | Ethereum/Base/Arbitrum described with one shared balance state | Per-chain balance tracking with stale-blend behavior |
| FKL | Single "EVM" row in adapter table | Three separate chain rows |
| Engine 3 | Aerodrome stable pools treated as constant-product | StableSwap-aware IL formula; poolType dispatch |
| Engine 6 | No distinction for GMX keeper-fulfilled actions | `fulfillment: 'async'` tag on gmx_gm_pool LP actions |
| Orchestrator | Single bridge transaction implied | CCTP 3-sub-step decomposition (burn / attest / mint) |
| Orchestrator | `gasEstimate` applied uniformly to all EVM chains | Arbitrum: L2 exec + L1 calldata via `NodeInterface` |
| Cross-Chain Ecosystem | EIP-1559 formula applied as single component on all EVM | Base: OP Stack L2+L1 formula; Arbitrum: Nitro L2+L1 formula |
| Portfolio State Manager | Webhook subscription described per address, not per chain | Explicit: 3 separate Alchemy Notify subscriptions for shared EVM address |
| Protocol Layer | "EVM — Aave V3" adapter implied one adapter for all three EVM chains | Per-chain adapter instances with distinct contract-address configs |
| Execution Router | Gas savings calculation assumed L1 formula on L2s | Chain-specific total cost formula in Job 4 |

### C. `poolType` Framework — Missing or Incorrect (Structural)

| Document | Issue | Fix |
|---|---|---|
| Engine 1 | No poolType-aware LP valuation | poolType decomposition paths added |
| Engine 2 | IL formula applied to GMX positions | GMX explicitly excluded; `poolType === 'gmx_gm_pool'` branching added |
| Engine 3 | Aerodrome stable pools run through constant-product IL | StableSwap-aware IL path; GMX uses Reader-sourced net yield |
| Engine 4 | No poolType-aware LP action handling | GMX positions → MANUAL_REVIEW; stable-swap aware cost for Aerodrome |
| Engine 6 | `LP_ADD`/`LP_REMOVE` assumed atomic universally | `fulfillment: 'atomic' \| 'async'` on all LP nodes |
| Policy Engine | Slippage check assumed AMM slippage universally | GMX → price-impact check via GMX Reader |
| Policy Engine | No action restriction for GMX exposure by persona | Conservative/Capital Preservation blocked from gmx_gm_pool |
| Execution Router | Venue selection logic constant-product only | Job 3 poolType-aware: stable_swap depth-weighted; gmx_gm_pool → direct lookup |
| Protocol Layer | No async execution model for GMX | `isAsyncExecution`, `executionFee`, keeper-request pattern documented |
| Orchestrator | No state for keeper-fulfilled actions | `STEP_N_AWAITING_FULFILLMENT` state added |
| Audit Layer | No audit events for async fulfillment | `ExecutionFulfillmentPending`, `ExecutionFulfillmentComplete` added |

### D. Duplicate-Model Bugs (Structural — Same calculation in two components)

| Pair | Bug | Fix |
|---|---|---|
| Engine 3 ↔ Engine 6 | Engine 6 recomputed its own cross-chain break-even formula (dimensionally malformed) | Engine 6 consumes Engine 3's `netSwitchingBenefitUsd` / `paybackDays` directly |
| Engine 2 ↔ Engine 4 | Engine 4 read `goalProfile.maxRiskTolerance` — a second, potentially-out-of-sync risk scalar | Engine 4 consumes Engine 2's `adjustedThreshold` directly; no separate tolerance field on GoalProfile |
| Engine 3 ↔ Engine 5 | Risk-reducing narration used different switching-benefit model than Engine 3 computed | Engine 5 Copilot narrates Engine 3's pre-computed fields; never recalculates inline |

### E. Authority / Security Gaps (Structural)

| Document | Gap | Fix |
|---|---|---|
| Policy Engine | No check for execution authority (read scope ≠ execution scope) | New Check 3: Execution Authority Gate |
| Engine 6 | Source address not filtered to `executionScope: true` | Step 1: authority filter before DAG construction |
| Policy Engine | No allowlist slot for BRIDGE/CCTP actions | `CROSS_CHAIN_PROTOCOL_ALLOWLIST = ['circle-cctp']`; BRIDGE checked against it |
| Audit Layer | x402 nonce only checked for existence (slow-replay possible) | 15-minute payment-header freshness check added |
| Audit Layer | No PAYMENT audit category | `PAYMENT` category added with 5 action types |
| Audit Layer | Replay attempt not audited | `ReplayAttemptDetected` as first-class PAYMENT audit event |

### F. Structural Gaps — Missing Components

| Document | Missing Element | Fix |
|---|---|---|
| Engine 6 | No sourcing logic for aggregate cross-chain deltas | Step 1.5: liquidation-waterfall sourcing |
| Execution Router | No job for BRIDGE execution | Job 5: CCTP transfer-speed selection + domain ID resolution |
| Execution Router | Route legs not filtered against Policy Engine allowlist | Pre-selection allowlist filter applied in Jobs 1–3 |
| Orchestrator | Bridge treated as single transaction | Decomposed to burn / attest / mint sub-steps |
| Portfolio State Manager | Post-execution sync targeted all chains regardless of event scope | Scoped to `event.chains`; bridge scope already included |
| Portfolio State Manager | Fixed sleep after execution confirmation | Poll-until-indexed with chain-appropriate interval |

---

## Cross-Document Interface Consistency — Verified Contracts

| Interface | Verified |
|---|---|
| Engine 3 `YieldOpportunitySnapshot.crossChainRedeployment` consumed by Engine 6 | ✅ |
| Engine 2 `adjustedThreshold` consumed by Engine 4 (no duplicate tolerance field) | ✅ |
| Engine 5 intent classification consumed by x402 middleware (no re-classification) | ✅ |
| `poolType` tag shared across Engines 1–4, Policy Engine, Execution Router, Protocol Layer | ✅ |
| Engine 4 `MANUAL_REVIEW` action narrated by Engine 5, not auto-executed by Engine 6 | ✅ |
| Engine 6 `urgencyFlag` consumed by Execution Router (Job 5) for CCTP transfer speed | ✅ |
| CCTP `executionGroupId` linking all multi-leg audit entries | ✅ |
| `executionScope` flag checked at Engine 6 Step 1 AND Policy Engine Check 3 | ✅ |
| Bridge completion: Cross-Chain Ecosystem monitors CrestFlow-initiated; PSM monitors external-only | ✅ |
| `poolType === 'gmx_gm_pool'` excluded from: IL formula (E2, E3), HF (E2), automated LP actions (E4, E6), slippage check (Policy Engine) | ✅ |

---

## Historical Cross-Cutting Open Items — Post-Audit Disposition

These were the eight deferred items recorded by the original report. They are listed here with their current canonical disposition so the historical record does not read as a live blocker list.

| # | Historical item | Current disposition |
|---|---|---|
| 1 | Shared `poolType`-aware library — recommendation, not built | Mandatory single package `@crestflow/pool-type` (T11); no component re-implements classification |
| 2 | Non-USDC cross-chain transfer — CCTP covers USDC only | Out of MVP; fail closed `UNSUPPORTED_BRIDGE` (D3) |
| 3 | GM pool Saga compensation semantics | Not applicable; GMX is read-only/manual, never an automated saga leg (D2) |
| 4 | Solana blockhash re-sign handoff | CCE owns blockhash refresh/re-sign for Path A; Path B re-prompts the user |
| 5 | Per-chain nonce tracking home for shared EVM address | CCE owns nonce allocation, gaps, replacement, cancellation (T11) |
| 6 | Curve allowlist status | Curve is Phase 2; MVP uses allowlisted venues only (T12) |
| 7 | Bridge-aggregator vendor selection | Phase 2; no MVP aggregator branch (D3) |
| 8 | KYC post-approval revocation event | First-class `KYC` audit category with scoped actions (T10); canonical KYC statuses include `SUSPENDED` and `EXPIRED` (T8) |

---

## Historical Record Notice

Later sections of this document describe resolutions that may use superseded terminology (for example, `compositeRiskScore`, 11 flat audit categories, ERC-4337/SPL delegation, CCTP V1, or x402 `X-Payment`). Those statements are part of the historical record and are **not** current contracts. Consult `docs/canonical_schemas.md`, `docs/policy_engine.md`, and the relevant component document for the current position.

