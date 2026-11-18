# CrestFlow Implementation Specification

**Status:** Authoritative  
**Version:** 1.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md`  
**Canonical schemas:** `docs/canonical_schemas.md`  
**Purpose:** Fill the operational gaps identified by the audit: API surface, persistence, nonce management, gas/fee calculation, approvals, saga/compensation, security, observability, provider contracts, policy test vectors, and environments.

---

## 1. Conventions

- REST/JSON, OpenAPI 3.1, base path `/v1`.
- JSON field names are camelCase and match `docs/canonical_schemas.md`.
- All timestamps are ISO-8601 UTC.
- All monetary values follow the `BaseUnits` / `Usd` rules.
- Every request carries `X-Request-Id`; every mutating request carries `Idempotency-Key`.
- Errors use a single envelope:

```json
{
  "error": {
    "code": "POLICY_BLOCKED",
    "message": "Human-readable summary",
    "details": {},
    "requestId": "req_..."
  }
}
```

### 1.1 HTTP status mapping

| Status | Use |
|---:|---|
| 200 | Success |
| 201 | Created |
| 202 | Accepted (async job) |
| 400 | Validation error |
| 401 | Missing/invalid session |
| 403 | KYC, policy, or ownership failure |
| 404 | Not found |
| 409 | Idempotency conflict or state conflict |
| 422 | Semantically invalid (e.g., unsupported venue) |
| 429 | Rate limited |
| 500 | Internal error |
| 502/503 | Provider unavailable; retryable |

### 1.2 Idempotency

- `Idempotency-Key` is required for `POST` endpoints that create executions, payments, KYC sessions, or wallet links.
- Store `{key, userId, requestHash, response, createdAt}` for 24 hours.
- Same key + same body returns the stored response; same key + different body returns `409 IDEMPOTENCY_CONFLICT`.

### 1.3 Pagination

- Cursor-based: `?cursor=<opaque>&limit=<1..100>`.
- Response: `{ "items": [], "nextCursor": "..." }`.
- Default limit 25; max 100.

### 1.4 Rate limits

| Scope | Limit |
|---|---:|
| Auth endpoints | 10/min per IP |
| Portfolio/risk/yield reads | 60/min per user |
| Strategy generation | 10/min per user |
| Execution submission | 10/min per user |
| Copilot free tier | 10/day per user |
| Copilot paid tier | 60/min per user |
| Audit export | 2/hour per user |
| Webhooks | Provider-specific; no user limit |

Rate-limit responses include `Retry-After` and `X-RateLimit-Remaining`.

---

## 2. API surface

### 2.1 Auth and session

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/auth/google` | Exchange Google OAuth code; upsert by `googleSub`; return session |
| `POST` | `/v1/auth/refresh` | Rotate refresh token; return new access token |
| `POST` | `/v1/auth/logout` | Revoke refresh token |
| `GET` | `/v1/me` | Current user, KYC status/tier, persona summary |
| `PATCH` | `/v1/me` | Update mutable profile fields (email, display name) |

`POST /v1/auth/google` body:

```json
{ "code": "google-oauth-code", "redirectUri": "https://app.crestflow.xyz/auth/callback" }
```

Response:

```json
{
  "user": { "id": "...", "email": "...", "kycStatus": "PENDING", "kycTier": "TIER_1" },
  "accessToken": "...",
  "refreshToken": "...",
  "expiresAt": "2026-11-18T12:15:00.000Z"
}
```

The callback upserts on `googleSub`. Email changes never create a second user.

### 2.2 Wallets and address scopes

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/wallets` | Add an address; `executionMode` is `MANUAL` or `READ_ONLY` |
| `POST` | `/v1/wallets/:id/verify` | Submit wallet signature challenge; sets `ownershipVerifiedAt` |
| `GET` | `/v1/wallets` | List address scopes |
| `DELETE` | `/v1/wallets/:id` | Remove a non-autopilot address |
| `POST` | `/v1/wallets/:id/autopilot` | Provision/attach a Turnkey embedded wallet (Path A) |
| `POST` | `/v1/wallets/:id/autopilot/pause` | Pause autonomous execution |

Rules:

- `AUTOPILOT` is created only through the Turnkey provisioning flow.
- `MANUAL` requires a signature challenge before execution.
- `READ_ONLY` never executes.
- Deleting an address with an open execution returns `409 ADDRESS_IN_USE`.

### 2.3 KYC

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/kyc/session` | Create a HyperVerge session; return redirect URL |
| `GET` | `/v1/kyc/status` | Current status, tier, reasons, next action |
| `POST` | `/v1/kyc/webhook` | HyperVerge webhook (signature verified) |
| `POST` | `/v1/admin/kyc/:userId/review` | Manual Tier 2/3 review (admin only) |

`GET /v1/kyc/status` response:

```json
{
  "status": "RESUBMISSION_REQUESTED",
  "tier": "TIER_1",
  "reasons": ["DOCUMENT_GLARE"],
  "nextAction": "Re-upload the document in better lighting",
  "updatedAt": "2026-11-18T12:00:00.000Z"
}
```

### 2.4 Portfolio

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/portfolio` | Latest canonical snapshot |
| `GET` | `/v1/portfolio/snapshots` | Snapshot history |
| `GET` | `/v1/portfolio/snapshots/:id` | Snapshot detail |
| `POST` | `/v1/portfolio/scan` | Trigger a scan (async, 202) |
| `GET` | `/v1/portfolio/positions` | Positions with filters (`chain`, `protocol`, `kind`) |

Snapshot response uses `PortfolioSnapshot` and includes `dataQuality`.

### 2.5 Risk

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/risk` | Latest `RiskSnapshot` |
| `GET` | `/v1/risk/history` | Risk history |
| `GET` | `/v1/risk/alerts` | Open/resolved alerts |
| `POST` | `/v1/risk/alerts/:id/ack` | Acknowledge an alert |

### 2.6 Yield

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/yield/opportunities` | Ranked opportunities for the user's persona |
| `GET` | `/v1/yield/opportunities/:id` | Opportunity detail and TOPSIS breakdown |

### 2.7 Strategy

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/strategy/plans` | Generate a plan (async, 202) |
| `GET` | `/v1/strategy/plans` | List plans |
| `GET` | `/v1/strategy/plans/:id` | Plan detail with actions and rationale |
| `POST` | `/v1/strategy/plans/:id/approve` | User approval; triggers execution |
| `POST` | `/v1/strategy/plans/:id/reject` | Reject and record reason |

For `AUTOPILOT`, approval may be automatic if the plan is within `AutopilotConfig`. For `MANUAL`, approval returns unsigned transactions for the user to sign.

### 2.8 Execution

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/executions` | Create an execution from an approved plan |
| `GET` | `/v1/executions/:id` | Execution status and step summary |
| `GET` | `/v1/executions/:id/steps` | Per-step status, tx hashes, errors |
| `POST` | `/v1/executions/:id/cancel` | Cancel not-yet-started steps |
| `POST` | `/v1/executions/:id/retry` | Retry a failed retryable step |

Execution states follow `docs/implementation_spec.md` §7.

### 2.9 Policy

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/internal/policy/evaluate` | Internal; evaluate `PolicyInput` |
| `GET` | `/v1/policy/decisions/:id` | Decision detail (user-visible reasons) |

The endpoint is internal-only; user-facing surfaces show the decision summary, never raw internal payloads.

### 2.10 Audit

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/audit` | Paginated audit entries; filters `category`, `action`, `chain`, `from`, `to` |
| `GET` | `/v1/audit/:eventId` | Entry detail |
| `GET` | `/v1/audit/export` | CSV/JSON export (async, 202) |

### 2.11 Copilot and payments

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/copilot/query` | x402-protected Copilot query |
| `GET` | `/v1/copilot/usage` | Free/paid usage for the current user |
| `GET` | `/v1/payments` | Payment history |
| `POST` | `/v1/payments/webhook` | Facilitator settlement webhook (if used) |

`POST /v1/copilot/query` without payment returns `402` with `PAYMENT-REQUIRED`. With `PAYMENT-SIGNATURE`, the server verifies via the facilitator and returns `PAYMENT-RESPONSE` on success.

### 2.12 Admin

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/admin/users/:id/suspend` | Suspend a user (KYC/sanctions/manual) |
| `POST` | `/v1/admin/users/:id/autopilot/pause` | Pause autopilot |
| `POST` | `/v1/admin/policy/overrides` | Create a Tier 3 policy override |
| `GET` | `/v1/admin/health` | Provider and queue health |

---

## 3. Persistence schema

PostgreSQL 16. All tables use `uuid` primary keys unless noted (composite keys: `audit_entries`, `nonces`; natural key: `webhook_events`). Timestamps are `timestamptz`; money is `numeric`.

### 3.1 Core tables

| Table | Key columns | Notes |
|---|---|---|
| `users` | `id`, `google_sub` unique, `email`, `email_verified`, `kyc_status`, `kyc_tier`, `kyc_updated_at`, `created_at` | `email` is not a foreign key. |
| `auth_sessions` | `id`, `user_id`, `refresh_token_hash`, `expires_at`, `revoked_at` | Refresh tokens hashed. |
| `address_scopes` | `id`, `user_id`, `address`, `chain`, `execution_mode`, `ownership_verified_at`, `turnkey_sub_org_id`, `label`, `created_at` | Unique `(user_id, chain, address)`. |
| `kyc_records` | `id`, `user_id`, `vendor`, `vendor_reference`, `status`, `tier`, `decision_reason`, `payload_encrypted`, `created_at`, `updated_at` | Encrypted payload. |
| `goal_profiles` | `id`, `user_id`, `version`, `persona`, `constraints` JSONB, `allowed_action_types` text[], `topsis_weights` JSONB, `created_at` | Versioned; latest active. |
| `portfolio_snapshots` | `id`, `user_id`, `created_at`, `data_quality`, `total_value_usd`, `chain_exposure` JSONB, `asset_exposure` JSONB, `protocol_exposure` JSONB, `hhi` JSONB | Partition by month if volume grows. |
| `positions` | `id`, `snapshot_id`, `kind`, `chain`, `protocol`, `address`, `value_usd`, `data_quality`, `payload` JSONB | Payload shape by `kind`. |
| `risk_snapshots` | `id`, `user_id`, `portfolio_snapshot_id`, `analyzed_at`, `freshness_seconds`, `risk_score`, `risk_level`, `adjusted_threshold`, `payload` JSONB | `payload` holds component scores. |
| `risk_alerts` | `id`, `user_id`, `risk_snapshot_id`, `type`, `severity`, `message`, `created_at`, `resolved_at` | |
| `yield_snapshots` | `id`, `created_at`, `data_quality` | |
| `yield_opportunities` | `id`, `snapshot_id`, `chain`, `protocol`, `pool_id`, `pool_type`, `twap_apy_7d`, `net_apy`, `payload` JSONB | |
| `strategy_plans` | `id`, `user_id`, `created_at`, `urgency`, `total_value_usd`, `estimated_cost_usd`, `rationale`, `portfolio_snapshot_id`, `risk_snapshot_id`, `yield_snapshot_id` | |
| `plan_actions` | `id`, `plan_id`, `type`, `chain`, `protocol`, `amount_usd`, `payload` JSONB | |
| `execution_intents` | `id`, `user_id`, `plan_id`, `created_at`, `source`, `execution_mode`, `urgency`, `payload` JSONB | Mirrors canonical schema. |
| `execution_actions` | `id`, `intent_id`, `type`, `chain`, `protocol`, `amount_in`, `amount_usd`, `payload` JSONB | |
| `execution_instructions` | `id`, `intent_id`, `created_at`, `payload` JSONB | Router output. |
| `simulations` | `id`, `instructions_id`, `simulated_at`, `total_gas_usd`, `chains`, `payload` JSONB | |
| `simulated_actions` | `id`, `simulation_id`, `action_id`, `success`, `gas_used`, `payload` JSONB | |
| `policy_decisions` | `id`, `user_id`, `plan_id`, `decision`, `policy_version`, `decided_at`, `simulation_id` | |
| `policy_check_results` | `id`, `decision_id`, `check_no`, `name`, `passed`, `code`, `message` | 9 rows per decision. |
| `executions` | `id`, `user_id`, `plan_id`, `state`, `created_at`, `updated_at`, `error_code` | |
| `execution_steps` | `id`, `execution_id`, `action_id`, `step_no`, `state`, `tx_hash`, `chain`, `error_code`, `started_at`, `finished_at` | |
| `cctp_transfers` | `id`, `user_id`, `execution_id`, `action_id`, `source_chain`, `destination_chain`, `amount`, `max_fee`, `min_finality_threshold`, `status`, `burn_tx`, `attestation`, `mint_tx` | |
| `audit_entries` | `event_id` + `created_at` PK, `schema_version`, `user_id`, `category`, `action`, `status`, `execution_group_id`, `strategy_plan_id`, `intent_id`, `tx_id`, `chain`, `kyc_status`, `kyc_tier`, `wallet_addresses` (encrypted), `payload` JSONB (PII encrypted), `ip_address` (encrypted), `user_agent` (encrypted), `created_at` | Monthly partitions; append-only; action is category-scoped; PK includes partition key. |
| `audit_event_dedupe` | `event_id` PK, `created_at` | Global eventId idempotency; inserted in the same transaction as the audit row. |
| `payments` | `id`, `user_id`, `network`, `scheme`, `amount`, `asset`, `status`, `payer`, `facilitator_tx`, `payment_payload_hash`, `created_at`, `settled_at` | `payment_payload_hash` unique; ledger is authoritative for paid usage. |
| `nonces` | `chain`, `address`, `nonce`, `lease_owner`, `lease_expires_at`, `status`, `tx_hash`, `updated_at` | PK `(chain, address, nonce)`. |
| `webhook_events` | `id`, `provider`, `event_id`, `received_at`, `payload_hash`, `processed_at` | Unique `(provider, event_id)`. |
| `provider_health` | `provider`, `status`, `latency_ms`, `last_error`, `updated_at` | |

### 3.2 Indexes

- `users(google_sub)` unique.
- `address_scopes(user_id, chain, address)` unique.
- `portfolio_snapshots(user_id, created_at desc)`.
- `risk_snapshots(user_id, analyzed_at desc)`.
- `strategy_plans(user_id, created_at desc)`.
- `executions(user_id, created_at desc)`, `executions(state)`.
- `audit_entries(user_id, created_at desc)`, `audit_entries(category, created_at desc)`, `audit_entries(execution_group_id)`.
- `cctp_transfers(status, updated_at)`.
- `nonces(chain, address, status)`.
- `webhook_events(provider, event_id)` unique.

### 3.3 Retention and privacy

| Data | Retention |
|---|---|
| Audit entries | 7 years, monthly partitions |
| Audit event dedupe | 7 years; UUID rows only |
| KYC records | 7 years after account closure, encrypted |
| Portfolio snapshots | 90 days hot, 2 years cold |
| Risk snapshots | 1 year |
| Payments | 7 years |
| Webhook events | 30 days |
| Nonces | 30 days after finality |

- KYC and audit payloads use field-level encryption with a KMS-managed key.
- PII access is logged as `category='AUTH'` or `category='KYC'` audit entries.
- GDPR deletion requests anonymize PII but retain immutable execution/audit records with a pseudonymous user ID.

---

## 4. Nonce manager

**Owner:** CCE. The Orchestrator requests a nonce; it never allocates or signs.

### 4.1 EVM

- Key: `(chain, address)`.
- Allocate inside a transaction: select an available nonce row, set `lease_owner` and `lease_expires_at` (2 minutes), return it.
- If no row exists, read the on-chain nonce and insert a contiguous range.
- One in-flight transaction per nonce.
- On broadcast, store `tx_hash`, set `status='BROADCAST'`.
- On confirmation, set `status='CONFIRMED'`.
- Gap detection: if `nonce > onchainNonce`, wait; if `nonce < onchainNonce`, mark `REPLACED` and reconcile.
- Replacement: same nonce, higher `maxFeePerGas`/`maxPriorityFeePerGas` (minimum 10% bump), new tx hash.
- Cancellation: same nonce, zero-value self-transfer, higher gas.
- Lease expiry: a background job reclaims expired leases only if the nonce is not `BROADCAST`.

### 4.2 Solana

- No nonce lease by default; use a recent blockhash.
- Build, sign, and send within the blockhash validity window.
- On expiry, rebuild and re-sign; the CCE owns the re-sign handoff.
- Durable nonce is optional and only used for scheduled transfers; it is not the default.

### 4.3 Invariants

- The same nonce is never used for two different payloads concurrently.
- A broadcast transaction is never "cancelled" in the database; it is replaced or confirmed.
- Every nonce transition is audited (`category='EXECUTION'`, `action='BROADCAST'` or `'CONFIRMATION'`).

---

## 5. Gas and fee specification

### 5.1 Ethereum

- EIP-1559: `maxFeePerGas = baseFee * 2 + maxPriorityFeePerGas` (bounded by user cap).
- No blob/data fees in MVP.
- Estimate with `eth_estimateGas`, then add a 15% buffer.

### 5.2 Base

- `GasPriceOracle` at `0x420000000000000000000000000000000000000F`.
- Use `getL1Fee(bytes)` or `getL1FeeUpperBound(uint256)` for the L1 component.
- Include `getOperatorFee(gasUsed)` in the total.
- Total: `gasUsed * (baseFee + priorityFee) + l1Fee + operatorFee`.
- Do not hand-roll `l1BaseFee * size * scalar`.

### 5.3 Arbitrum

- `NodeInterface` at `0xc8`, accessible only over RPC.
- `gasEstimateL1Component(address to, bool contractCreation, bytes calldata data)` returns the L1 component.
- `ArbGasInfo` at `0x6c` provides `getL1BaseFeeEstimate()`.
- Total: L2 gas estimate plus L1 component; add a 15% buffer.

### 5.4 Solana

- Base fee: 5,000 lamports per signature.
- Priority fee: `computeUnitLimit * computeUnitPriceMicroLamports`.
- Set `computeUnitLimit` from simulation; add a 10% buffer.
- Re-sign on blockhash expiry; never reuse a stale blockhash.

### 5.5 Gas gate

- If `estimatedGasUsd > plan.estimatedGasCostUsd * 1.2`, re-plan or block.
- If the current gas price exceeds the user's cap, queue the transaction and notify.
- Bridge fees follow D5 (Standard free; Fast quoted and capped).

---

## 6. Approval specification

| Flow | Mechanism | When |
|---|---|---|
| Standard ERC-20 | `approve(spender, amount)` | Default for Uniswap V3, Aerodrome, Moonwell, and other EVM venues |
| Aave V3 supply | EIP-2612 `supplyWithPermit` | When the token supports EIP-2612 and the user has no allowance |
| Aave V3 fallback | `approve` then `supply` | Tokens without EIP-2612 |
| Permit2 | Phase 2 | Not in MVP; UniswapX and aggregator flows are Phase 2 |
| Solana ATA | `createAssociatedTokenAccountIdempotent` | Before SPL transfers if the ATA is missing |

Rules:

- Approval amount is the exact `amountIn` for the action, not unlimited, unless the user explicitly opts into unlimited approvals.
- USDT-like tokens that require zero-first: call `approve(0)` then `approve(amount)`.
- Every approval is simulated before broadcast.
- The approval action is a separate `ExecutionAction` with `type='APPROVE'` and a dependency edge to the action that needs it.
- `ApprovalInstruction.signatureRequired` is true for EIP-2612; the signature is collected and submitted with the supply transaction.
- Permit2 is not claimed for Aave V3. Aave uses EIP-2612.

---

## 7. Saga and compensation

### 7.1 Execution states

```
CREATED
  → ROUTING
  → BUILDING
  → SIMULATING
  → POLICY_CHECK
      → BLOCKED (terminal)
      → AWAITING_SIGNATURE (manual)
      → SIGNING (autopilot)
  → BROADCASTING
  → AWAITING_CONFIRMATION
  → AWAITING_ATTESTATION (CCTP)
  → MINTING (CCTP)
  → CONFIRMING
  → COMPLETED
```

Failure/exception states: `FAILED`, `COMPENSATING`, `COMPENSATED`, `PAUSED`.

### 7.2 Per-action compensation

| Action | Compensation | Limits |
|---|---|---|
| `SWAP` | Reverse swap within the original slippage cap | Only if the output is still held and the market is open |
| `LEND_SUPPLY` | `withdraw` the supplied amount | Only if no borrow depends on it |
| `LEND_WITHDRAW` | Re-supply the withdrawn amount | Only if the funds are still held |
| `BORROW` | `repay` the borrowed amount | Only if the debt is still open and the user has the funds |
| `REPAY` | No automatic compensation; manual review | Repayment is generally beneficial |
| `LP_ADD` | `decreaseLiquidity` + `collect` | Only if the NFT/LP position is still held |
| `LP_REMOVE` | Re-add liquidity | Only within the original range and slippage cap |
| `BRIDGE` (CCTP) | Wait for attestation and mint; retry mint | **No burn reversal**; if mint is permanently impossible, manual recovery |
| `GMX` | None in MVP | Manual-only |

### 7.3 In-flight rules

- Never cancel a broadcast transaction; wait for confirmation or replace it with the same nonce.
- If a CRITICAL risk alert fires, stop not-yet-broadcast actions, let in-flight actions resolve, disable autopilot, and notify the user.
- Compensation actions are planned and simulated like any other action; they pass through Policy.
- A compensated execution is terminal; it cannot resume automatically.

### 7.4 CCTP lifecycle

1. Router selects domain/speed and quotes `maxFee`.
2. Planner builds the burn transaction and simulates it.
3. Policy approves the bridge intent.
4. CCE broadcasts the burn (autopilot) or the user signs (manual).
5. CCE polls Iris for the attestation with exponential backoff (2s → 30s), timeout 60 minutes Standard / 5 minutes Fast.
6. Planner builds the mint transaction from the attestation; it is simulated again.
7. CCE broadcasts the mint (autopilot) or the user signs (manual).
8. PSM reconciles the destination balance.

The mint recipient is fixed at burn time. `destinationCaller` is `0x0` in MVP.

---

## 8. Security threat model

| Threat | Control |
|---|---|
| Webhook forgery | Alchemy HMAC `X-Alchemy-Signature`; Helius `authHeader` + secret URL; reject unsigned/unknown events |
| Webhook replay | Unique `(provider, event_id)`; idempotent processing; 30-day event retention |
| API replay | `Idempotency-Key`; JWT expiry; refresh rotation |
| x402 replay | EIP-3009 `validAfter`/`validBefore` + facilitator settlement tracking; no custom nonce |
| Prompt injection | Copilot has no direct execution tools; deterministic plan/policy path; numeric values come from DB; tool allowlist; output schema validation |
| Key compromise | Turnkey TEE; no keys in app memory; per-user sub-org; policy engine in enclave; incident pause |
| Delegated authority abuse | Path B has no delegation; Path A Turnkey policies enforce chain/protocol/value/slippage interfaces; user can pause |
| Malicious route/calldata | Venue allowlist; calldata decode/compare; simulation; `minAmountOut`; deadline; no arbitrary calldata |
| Oracle manipulation | Chainlink + Pyth cross-check; 1.5% divergence; 60s staleness; sequencer uptime; DEX TWAP cross-check |
| Stale data | `dataQuality` enum; execution re-quotes; stale cache fails closed |
| Supply chain | Lockfiles; pinned dependencies; SBOM; CI audit; signed images; least-privilege CI tokens |
| PII exposure | Field-level encryption; access logging; retention limits; GDPR anonymization |
| Nonce race | CCE lease; one in-flight per nonce; gap detection |
| Bridge stuck | Attestation polling; retry; manual recovery runbook; no burn reversal claim |
| Admin abuse | Tier 3 overrides require two-person approval; all admin actions audited |

### 8.1 Secrets

- Provider API keys, Turnkey credentials, KYC secrets, and database credentials live in a secrets manager/KMS.
- No secrets in the repository, logs, or client bundles.
- Rotation: quarterly and on incident.

---

## 9. Observability and SLOs

### 9.1 SLOs

| Signal | Target |
|---|---|
| API availability | 99.9% monthly |
| Portfolio scan p95 | < 5 s |
| Policy evaluation p95 | < 200 ms |
| Simulation p95 | < 3 s |
| Broadcast p95 | < 2 s |
| EVM confirmation p95 | < 60 s |
| Solana confirmation p95 | < 30 s |
| CCTP Standard completion p95 | < 25 min |
| CCTP Fast completion p95 | < 2 min |
| Webhook lag p95 | < 2 s |
| Audit write success | 100% (fail closed) |

### 9.2 Metrics

- Engine latency and error rate per engine.
- Provider error rate, latency, and rate-limit hits.
- Cache hit rate by data class.
- Policy block rate by failure code.
- Simulation failure rate by venue.
- Execution success rate by action type.
- Bridge completion time and stuck transfers.
- Webhook duplicates and processing lag.
- Nonce gaps and replacements.
- Copilot latency, token usage, fallback rate, and numeric-faithfulness violations.

### 9.3 Alerts

- Policy block rate > 30% over 15 minutes (possible misconfiguration).
- Simulation failure > 5% over 10 minutes.
- Provider error > 2% over 5 minutes.
- Webhook lag > 30 s.
- Nonce gap unresolved > 5 minutes.
- CCTP transfer stuck > 30 minutes.
- Audit write failure (page immediately).
- Turnkey signing failure or latency > 5 s.

### 9.4 Runbooks

- Provider outage: fail over to secondary provider or serve `dataQuality='STALE'`; block execution if pricing is stale.
- Nonce gap: inspect chain, replace or cancel, do not reuse.
- Bridge stuck: check Iris attestation, retry mint, escalate to manual recovery.
- Turnkey outage: pause autopilot, queue manual approvals, do not hold keys in app.
- DB failover: read replica promotion; audit writes fail closed until primary is restored.
- KYC vendor outage: queue sessions, do not auto-approve.

---

## 10. Provider contracts

| Provider | Plan / limit | Failure behaviour |
|---|---|---|
| CoinGecko | Demo 100 calls/min; paid plan for production; 515 IDs per `/simple/price` | Fall back to on-chain oracle/DEX for execution; display last known with `STALE` |
| DeFiLlama | Free `api.llama.fi` (no SLA); Pro `pro-api.llama.fi/{KEY}` for production | Use last known protocol score; if older than 48h, mark `PARTIAL` and lower confidence |
| Alchemy | Address Activity webhooks <100k wallets; HMAC `X-Alchemy-Signature`; retries 10 min (Free/PAYG), 1h (Enterprise) | Reconciliation sweep every 10 min; `PARTIAL` on lag |
| Helius | Webhooks 200–500ms; 1 credit/event; 100k addresses/webhook; 24h retry; static `authHeader` | Reconciliation sweep; `PARTIAL` on lag |
| Turnkey | TEE signing; policy engine; per-user sub-org | Pause autopilot; queue manual approvals |
| HyperVerge | Conditional; sandbox + webhook spec required | Queue KYC; do not auto-approve; fallback vendor |
| Circle Iris | Attestation API; poll 2s → 30s; 60 min Standard / 5 min Fast timeout | Retry; manual recovery if permanently stuck |
| x402 facilitator | `https://x402.org/facilitator`; verify/settle | Return 402/503; do not charge on failure |

### 10.1 Provider abstraction

- All providers implement a port in FKL or the relevant module.
- No engine calls a provider SDK directly.
- Provider responses are normalized into canonical schemas.
- Every provider call has a timeout, retry policy, and circuit breaker.
- Provider health is recorded in `provider_health`.

---

## 11. Policy test vectors

Each check has at least one passing and one failing case, including boundary values.

| # | Check | Passing case | Failing case |
|---:|---|---|---|
| 1 | KYC | `kycStatus=APPROVED` | `kycStatus=PENDING` → `KYC_NOT_APPROVED` |
| 2 | Tier limit | spend 900 + plan 100 = 1,000 at TIER_1 | spend 900 + plan 100.01 → `TIER_LIMIT_EXCEEDED` |
| 3 | Authority | AUTOPILOT active Turnkey delegation; MANUAL ownership verified | AUTOPILOT revoked → `AUTHORITY_REVOKED`; MANUAL unverified → `AUTHORITY_MISSING` |
| 4 | Risk | `riskScore=50`, `adjustedThreshold=50` | `riskScore=51` → `RISK_TOO_HIGH`; age 4h+1s → `RISK_SNAPSHOT_STALE` |
| 5 | Chain | `base` in allowlist | `optimism` → `CHAIN_NOT_ALLOWED` |
| 6 | Protocol | `aave-v3` in allowlist | `1inch` → `PROTOCOL_NOT_ALLOWED` |
| 7 | Per-tx value | action $1,000, cap $1,000 | action $1,000.01 → `TX_VALUE_EXCEEDED` |
| 8 | Slippage | stable 0.25%, cap 0.25% | stable 0.26% → `SLIPPAGE_EXCEEDED`; persona 0.10%, actual 0.15% → fail |
| 9 | Action types | `SWAP` allowed | `BORROW` not allowed → `ACTION_NOT_ALLOWED` |

### 11.1 Cross-check tests

- KYC fails and slippage fails → decision returns **both** failures, `checksRun.length = 9`.
- Risk snapshot stale + chain not allowed → both failures.
- Missing `allowedActionTypes` → `POLICY_INPUT_INCOMPLETE`, fail closed.
- Missing `riskScore` → `POLICY_INPUT_INCOMPLETE`, fail closed.
- Simulation failure → `SIMULATION_FAILED` before policy; no partial execution.
- GMX automated action → `UNSUPPORTED_VENUE`.
- `BRIDGE_AGGREGATOR` → `UNSUPPORTED_BRIDGE`.
- Path B `AUTOPILOT` request → `AUTHORITY_MISSING`.

---

## 12. Environments and feature flags

| Environment | Chains | Purpose |
|---|---|---|
| `local` | Anvil, Solana test validator | Unit/integration tests |
| `testnet` | Ethereum Sepolia, Base Sepolia, Arbitrum Sepolia, Solana Devnet | End-to-end validation |
| `mainnet` | Ethereum, Base, Arbitrum, Solana | Production, separate release gate |

Feature flags:

- `AUTOPILOT_ENABLED`
- `X402_ENABLED`
- `CCTP_FAST_ENABLED`
- `GMX_READ_ONLY` (always true in MVP)
- `NON_USDC_BRIDGE_ENABLED` (always false in MVP)
- `KYC_VENDOR` (`hyperverge` or `mock` in local)

### 12.1 Release gates

- Testnet: one swap, one Aave supply/withdraw, one Uniswap V3 LP add/remove, one CCTP Standard transfer, one x402 payment.
- Mainnet: all of the above plus monitoring, alerting, runbooks, and a rollback plan.

---

## 13. OpenAPI conventions

- `openapi: 3.1.0`.
- Security: `bearerAuth` for user endpoints; `internalAuth` for `/v1/internal/*`.
- Every error response uses the shared error envelope.
- Every list endpoint uses cursor pagination.
- Every mutating endpoint documents `Idempotency-Key`.
- Every monetary field is a string with a `description` stating base units or USD.
- Every chain field enumerates `ethereum`, `base`, `arbitrum`, `solana`.

---

## 14. Traceability

| Audit finding | Implementation section |
|---|---|
| C-01 / D7 execution order | §7 saga states; `docs/orchestrator_planner.md` |
| C-02 / D6 policy | `docs/policy_engine.md`; §11 test vectors |
| C-03 / D1 Path B | §2.2 wallets; §6 approvals; §8 delegation |
| C-04 / D5 CCTP | §7.4; `docs/protocol_layer.md` |
| C-05 / D4 x402 | §2.11; `docs/audit_layer_x402.md` |
| C-06 schema | `docs/canonical_schemas.md` |
| H-01 Router | `docs/execution_router.md` |
| H-02 venues | T12; `docs/protocol_layer.md` |
| H-03 approvals | §6 |
| H-04 GMX | D2; §7.2 |
| H-05 non-USDC | D3; §12 flags |
| H-06 HHI | T1 |
| H-07 TOPSIS/APY/slippage | T2/T3 |
| H-08 oracle/cache | T4/T6 |
| H-09 KYC | T8; `docs/identity_kyc.md` |
| H-10 identity | T9 |
| H-11 LLM | D8 |
| H-12 E3→E4 | `docs/engine3_yield_opportunity.md`, `docs/engine4_strategy_optimization.md` |
| H-13 audit | T10; `docs/audit_layer_x402.md` |
| H-14 indexer | T7 |
| M-05 gas | §5 |
| M-14 missing specs | This document |
| M-16 webhooks | §8, T7 |
| M-17 saga/compensation | §7 |

