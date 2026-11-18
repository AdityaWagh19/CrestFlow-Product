# Identity & KYC

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` T8, T9, D9  
**Canonical schemas:** `docs/canonical_schemas.md` §2–§3, §13  
**Chain scope:** Ethereum, Base, Arbitrum, Solana. KYC is chain-independent — completed once, enforced uniformly.

---

## Purpose

Prove a user is a real, non-sanctioned person before they execute financial transactions, and give the Policy Engine a fast, reliable internal check of that fact at execution time. KYC status is internal platform state — no portable credentials, no blockchain-anchored identity, no external verifiers in scope.

---

## KYC Provider: HyperVerge (conditional)

**Selected pending integration validation (D9).** HyperVerge is the primary candidate because it is a crypto-platform KYC vendor with document and liveness coverage relevant to CrestFlow's initial market.

**A two-week integration spike must confirm, before build commitment:**

1. sandbox access and test identities;
2. webhook signature scheme and replay protection;
3. API version and rate limits;
4. status-to-tier mapping;
5. per-completed-KYC pricing in writing.

If any item fails, switch to **Persona or Sumsub**. The fallback evaluation runs in parallel from day one.

**Do not state as fact:** WazirX scale, 30,000 checks/day, or a verified HMAC scheme. Those are vendor claims or unverified assumptions. The adapter treats the signature scheme as a configuration value verified during the spike.

---

## Step 1 — KYC Initiation

The user selects "Verify Identity" in the dashboard.

1. CrestFlow creates a verification session through the vendor adapter.
2. The frontend opens the vendor's hosted SDK flow.
3. The vendor handles document capture and liveness detection.

**Stored:** `KycRecord { userId, vendor, vendorReference, status: PENDING, tier, attemptNumber }`.

---

## Step 2 — Vendor Processing

Three checks run on the vendor side:

- **Document verification** — OCR and authenticity (Aadhaar, PAN, passport, national ID).
- **Liveness detection** — anti-spoofing biometric check.
- **AML screening** — OFAC, PEP, and global sanctions lists at onboarding.

The vendor delivers the decision via webhook to `POST /v1/kyc/webhook`.

**Webhook security:** every payload is verified with the vendor's documented scheme before any state change. A spoofed approval callback is the highest-risk attack surface in this flow. Until the spike confirms the scheme, the adapter must reject unsigned payloads and fail closed. Replay protection uses the vendor event ID plus the canonical `eventId` idempotency key.

**Decision outcomes** map to the canonical enum:

| Vendor concept | Canonical status |
|---|---|
| Approved | `APPROVED` |
| Declined | `REJECTED` |
| More information needed | `RESUBMISSION_REQUESTED` |
| Under review | `PENDING` |
| Post-approval sanctions hit | `SUSPENDED` |
| Re-verification required | `EXPIRED` |

The adapter owns the mapping. Vendor-specific strings never enter the domain model.

Every status transition is auditable under `category = 'KYC'`: `PENDING` creation → `KYC_SUBMITTED`; `APPROVED` → `KYC_APPROVED`; `REJECTED` → `KYC_REJECTED`; `RESUBMISSION_REQUESTED` → `KYC_RESUBMISSION_REQUESTED`; `SUSPENDED` → `KYC_REVOKED`; `EXPIRED` → `KYC_EXPIRED`. The action set is category-scoped per T10 and defined in `docs/canonical_schemas.md` §13.

---

## Step 3 — Status Update and Tier Assignment

On `APPROVED`:

- `User.kycStatus` → `APPROVED`
- `KycRecord.status` → `APPROVED`
- `User.kycTier` is assigned from verification depth.

| Tier | Basis | Daily execution limit |
|---|---|---:|
| `TIER_1` | Basic document + liveness | $1,000 |
| `TIER_2` | Enhanced verification (additional docs, source of funds) | $10,000 |
| `TIER_3` | Institutional — manual review | Unlimited |

**Cross-chain summation.** Tier limits are enforced across all four chains in aggregate. A `TIER_1` user cannot execute $800 on Ethereum and $600 on Solana in the same 24-hour window. Policy sums `audit_entries` with `category = 'EXECUTION'` and `status = 'SUCCESS'` over the rolling window regardless of chain.

---

## Step 4 — KYC as an Execution Gate

Policy Check 1 verifies `User.kycStatus === 'APPROVED'`.

- **Binary** — no partial KYC state allows partial execution.
- **Chain-independent** — the same status governs all four chains.
- **Not short-circuiting** — Check 1 failing does **not** stop Checks 2–9. All nine checks run and all failures are returned. This is deliberate: the user and Copilot get a complete explanation, and the audit record shows every independently failing condition.

`KYC_NOT_APPROVED` is the failure code.

**KYC snapshot in audit.** Every audit entry captures `kycStatus` and `kycTier` at the moment of the action, snapshotted rather than referenced by foreign key. If status changes later, the audit record still proves the status at execution time.

---

## Step 5 — AML and Travel Rule

MVP AML coverage is at KYC time (vendor screening) plus protocol-allowlist curation, which includes OFAC/Chainalysis/TRM screening of every protocol before it is added. Since only allowlisted contracts execute, this is the MVP's de facto per-transaction sanctions coverage.

**Deferred:**

1. **Counterparty screening at execution time** — Phase 2.
2. **FATF Travel Rule** — **Phase 3**. The audit schema retains the fields needed to assemble an originator package, but no Travel Rule message is sent in MVP.

---

## KYC Expiry and Post-Approval Events

- **Sanctions hit (post-approval):** `User.kycStatus` → `SUSPENDED`; Policy Check 1 blocks all new execution; the Audit Layer writes `category = 'KYC'`, `action = 'KYC_REVOKED'`; the Orchestrator pauses not-yet-broadcast actions and disables autopilot.
- **Rescreening:** daily and on every sanctions-list update.
- **Expiry:** `EXPIRED` exists in the canonical enum. MVP approvals do not auto-expire by default; jurisdiction-specific re-verification cadences are configured when multi-jurisdiction support is added.

**In-flight execution rule:** a mid-execution suspension stops not-yet-broadcast actions, lets in-flight transactions resolve, disables autopilot, and opens a manual review. It does not attempt to reverse confirmed transactions.

---

## Data and Retention

- `KycRecord.payloadEncrypted` uses field-level encryption with a KMS-managed key.
- KYC records are retained 7 years after account closure.
- Access to KYC data is audited under `category = 'KYC'` or `'AUTH'`.
- `User.email` is mutable and is never a foreign key; `googleSub` is the identity anchor (T9).

---

## Resolved Items

1. **In-flight execution during suspension** — specified above: stop not-yet-broadcast actions, resolve in-flight, disable autopilot, manual review.
2. **Multi-jurisdiction** — design keeps the vendor adapter and tier mapping jurisdiction-configurable; MVP is India-first.
3. **Counterparty AML timing** — Phase 2; allowlist curation is the MVP control. Legal counsel sign-off is a release gate.
4. **HyperVerge claims** — removed as unverified; conditional on the two-week spike (D9).
5. **Travel Rule** — Phase 3, explicitly out of MVP.
