# Audit Layer & x402 Payment Gateway

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D4, T10, T17  
**Canonical schemas:** `docs/canonical_schemas.md` §13, §15  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Design Principles

- **INSERT-only.** Once written, no audit record is modified or deleted. Enforced at the database-role level: `crestflow_app` has `INSERT` + `SELECT` on `audit_entries`; no application role has `UPDATE` or `DELETE`.
- **Passive.** No engine calls the Audit Layer directly. It consumes domain events. Zero coupling. Engines continue working if the Audit Layer restarts; events queue and are processed on recovery.
- **Complete.** Every significant financial action, KYC event, system change, payment, and execution lifecycle step produces an immutable record.
- **Point-in-time snapshots.** `kycStatus`, `kycTier`, and wallet addresses are snapshotted at INSERT time, not referenced from a mutable user record.
- **Two-field taxonomy.** Every entry has a `category` and an `action`. They are not interchangeable.

**Scope note:** engines also emit operational telemetry (scan durations, opportunity counts, risk alerts). That is observability data and belongs in metrics/logs, not `audit_entries`. The audit ledger records compliance and financial lifecycle events. Risk alerts and portfolio snapshots remain queryable in their own tables and are linked from audit payloads where relevant.

---

## Canonical Taxonomy

### Categories (6)

| Category | Source | What Is Recorded |
|---|---|---|
| `AUTH` | Auth / wallet module | Login, logout, token refresh, session revocation, wallet provisioned/linked/verified/removed |
| `KYC` | KYC module | Submitted, approved, rejected, resubmission requested, revoked/suspended, expired |
| `EXECUTION` | Engine 6 / Router / Planner / Policy / CCE / Orchestrator | Policy evaluation, simulation, signing, broadcast, confirmation, fulfillment, compensation, circuit breaker |
| `PREFERENCE` | Engine 5 / settings | Goal updated, preference updated, autopilot enabled/disabled |
| `PAYMENT` | x402 middleware | Payment required, settled, failed, replay blocked |
| `SYSTEM` | All services | Config changed, adapter registered/removed, audit recovery |

### Actions (category-scoped)

| Category | Actions |
|---|---|
| `AUTH` | `LOGIN`, `LOGOUT`, `TOKEN_REFRESH`, `SESSION_REVOKED`, `WALLET_PROVISIONED`, `WALLET_LINKED`, `WALLET_VERIFIED`, `WALLET_REMOVED` |
| `KYC` | `KYC_SUBMITTED`, `KYC_APPROVED`, `KYC_REJECTED`, `KYC_RESUBMISSION_REQUESTED`, `KYC_REVOKED`, `KYC_EXPIRED` |
| `EXECUTION` | `POLICY_EVALUATION`, `SIMULATION`, `SIGNING`, `BROADCAST`, `CONFIRMATION`, `FULFILLMENT`, `COMPENSATION`, `CIRCUIT_BREAKER` |
| `PREFERENCE` | `GOAL_UPDATED`, `PREFERENCE_UPDATED`, `AUTOPILOT_ENABLED`, `AUTOPILOT_DISABLED` |
| `PAYMENT` | `PAYMENT_REQUIRED`, `PAYMENT_SETTLED`, `PAYMENT_FAILED`, `PAYMENT_REPLAY_BLOCKED` |
| `SYSTEM` | `CONFIG_CHANGED`, `ADAPTER_REGISTERED`, `ADAPTER_REMOVED`, `AUDIT_RECOVERED` |

An action outside its category is rejected at insert time by a `CHECK` constraint or the event consumer's schema validation.

---

## Execution Lifecycle Events

Every event in one execution sequence shares an `executionGroupId`:

| Lifecycle step | `category` | `action` | `status` |
|---|---|---|---|
| Policy decision | `EXECUTION` | `POLICY_EVALUATION` | `SUCCESS` or `BLOCKED` |
| Simulation result | `EXECUTION` | `SIMULATION` | `SUCCESS` or `FAILURE` |
| Signing (Path A TEE or Path B user signature verified) | `EXECUTION` | `SIGNING` | `SUCCESS` or `FAILURE` |
| Broadcast (burn, mint, or ordinary tx) | `EXECUTION` | `BROADCAST` | `SUCCESS` or `FAILURE` |
| Confirmation (including CCTP attestation) | `EXECUTION` | `CONFIRMATION` | `SUCCESS` or `FAILURE` |
| Fulfillment (reserved; GMX automation is not in MVP) | `EXECUTION` | `FULFILLMENT` | `SUCCESS` or `FAILURE` |
| Compensation | `EXECUTION` | `COMPENSATION` | `SUCCESS` or `FAILURE` |
| Emergency pause / autopilot halt | `EXECUTION` | `CIRCUIT_BREAKER` | `SUCCESS` |

**CCTP bridge trail:**

```
chain: 'ethereum', txId: '0xBURN...', action: 'BROADCAST'      // burn tx
chain: null,       txId: null,        action: 'CONFIRMATION'   // attestation received; payload.stage = 'ATTESTATION'
chain: 'base',     txId: '0xMINT...', action: 'CONFIRMATION'   // mint confirmed
```

All three share the same `executionGroupId`. The trail is reconstructed with:

```sql
SELECT * FROM audit_entries WHERE execution_group_id = $id ORDER BY created_at ASC;
```

Plan creation, approval, and rejection are stored on the execution/plan aggregate. The audit ledger starts at the authoritative policy decision. A blocked decision is recorded (`POLICY_EVALUATION`, `BLOCKED`).

---

## `AuditEntry` Schema

The canonical type is in `docs/canonical_schemas.md` §13.

```typescript
interface AuditEntry {
  eventId:            UUID;             // idempotency key; unique
  schemaVersion:      '2.0';
  userId:             UUID | null;      // null for unauthenticated agents and system events
  category:           AuditCategory;
  action:             AuditAction;
  status:             'SUCCESS' | 'FAILURE' | 'BLOCKED';

  executionGroupId:   UUID | null;
  strategyPlanId:     UUID | null;
  intentId:           UUID | null;
  txId:               string | null;
  chain:              Chain | null;

  kycStatus:          KycStatus | null;
  kycTier:            KycTier | null;
  walletAddresses:    Json | null;      // AES-256-GCM encrypted

  payload:            Json;             // action-specific fields; PII encrypted
  ipAddress:          string | null;    // AUTH/PAYMENT only; encrypted
  userAgent:          string | null;    // AUTH/PAYMENT only; encrypted
  createdAt:          DateTime;         // DB server default now()
}
```

`createdAt` is set by the database server, not the application, to prevent clock skew or backdating. `payload` carries action-specific fields such as `policyVersion`, `checksRun`, `failureCodes`, `cctpAttestationId`, `stage`, and provider references.

---

## Field-Level Encryption

Encrypted fields: `walletAddresses`, `ipAddress`, `userAgent`, and any PII inside `payload`. All other fields remain plaintext for compliance queries.

```
Algorithm: AES-256-GCM
Keys: per-row symmetric keys derived from a master key in AWS KMS / GCP Cloud KMS / HashiCorp Vault
Encryption: applied by the Audit Layer consumer before INSERT
TDE: baseline at the DB host
Key rotation: annual minimum; existing records re-encrypted by a background job
```

---

## Database Schema

```sql
CREATE TYPE audit_category_enum AS ENUM
  ('AUTH', 'KYC', 'EXECUTION', 'PREFERENCE', 'PAYMENT', 'SYSTEM');

CREATE TYPE audit_status_enum AS ENUM ('SUCCESS', 'FAILURE', 'BLOCKED');

CREATE TABLE audit_entries (
  event_id            UUID NOT NULL,
  schema_version      VARCHAR(8) NOT NULL,
  user_id             UUID REFERENCES users(id) ON DELETE SET NULL,
  category            audit_category_enum NOT NULL,
  action              VARCHAR(64) NOT NULL,
  status              audit_status_enum NOT NULL,
  execution_group_id  UUID,
  strategy_plan_id    UUID,
  intent_id           UUID,
  tx_id               VARCHAR(128),
  chain               chain_enum,
  kyc_status          VARCHAR(32),
  kyc_tier            VARCHAR(16),
  wallet_addresses    JSONB,
  payload             JSONB NOT NULL DEFAULT '{}',
  ip_address          VARCHAR(64),
  user_agent          TEXT,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (event_id, created_at)
) PARTITION BY RANGE (created_at);

-- PostgreSQL requires the partition key in a partitioned table's primary key.
-- Global eventId idempotency is enforced here, in the same transaction as the insert.
CREATE TABLE audit_event_dedupe (
  event_id   UUID PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_entries_2026_11
  PARTITION OF audit_entries
  FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');

CREATE INDEX idx_audit_user_created      ON audit_entries(user_id, created_at DESC);
CREATE INDEX idx_audit_category_created  ON audit_entries(category, created_at DESC);
CREATE INDEX idx_audit_action_created    ON audit_entries(action, created_at DESC);
CREATE INDEX idx_audit_execution_group   ON audit_entries(execution_group_id) WHERE execution_group_id IS NOT NULL;
CREATE INDEX idx_audit_tx_id             ON audit_entries(tx_id) WHERE tx_id IS NOT NULL;
CREATE INDEX idx_audit_chain_created     ON audit_entries(chain, created_at DESC) WHERE chain IS NOT NULL;

REVOKE UPDATE, DELETE ON audit_entries, audit_event_dedupe FROM crestflow_app;
GRANT INSERT, SELECT ON audit_entries, audit_event_dedupe TO crestflow_app;
```

Retention: 7 years, monthly partitions for `audit_entries`; `audit_event_dedupe` follows the same retention window. See `docs/implementation_spec.md` §3.3.

---

## Compliance Queries

### All executions for a user in a date range

```sql
SELECT event_id, action, status, chain, tx_id,
       (payload->>'valueUsd')::numeric AS value_usd,
       payload->>'assetSymbol' AS asset_symbol,
       kyc_status, kyc_tier, created_at
FROM audit_entries
WHERE user_id = $userId AND category = 'EXECUTION'
  AND created_at BETWEEN $start AND $end
ORDER BY created_at ASC;
```

### Cross-chain execution reconstruction

```sql
SELECT event_id, action, status, chain, tx_id, created_at
FROM audit_entries
WHERE execution_group_id = $executionGroupId
ORDER BY created_at ASC;
```

### KYC tier daily limit (Policy Check 2)

```sql
SELECT COALESCE(SUM((payload->>'valueUsd')::numeric), 0) AS daily_total
FROM audit_entries
WHERE user_id = $userId
  AND category = 'EXECUTION'
  AND action = 'CONFIRMATION'
  AND status = 'SUCCESS'
  AND created_at >= now() - INTERVAL '24 hours';
```

`valueUsd` may instead be promoted to a generated numeric column for indexing. The rule is that spend is counted from `EXECUTION`/`CONFIRMATION`/`SUCCESS` entries only.

### FATF Travel Rule originator package (Phase 3)

```sql
SELECT user_id, execution_group_id, chain, tx_id,
       payload->>'valueUsd' AS value_usd,
       payload->>'assetSymbol' AS asset_symbol,
       kyc_status, kyc_tier, wallet_addresses, created_at
FROM audit_entries
WHERE execution_group_id = $executionGroupId AND category = 'EXECUTION'
ORDER BY created_at ASC;
```

Travel Rule messaging itself is **Phase 3** (T8). The query exists so the data is available when the capability ships.

---

## x402 Payment Gateway

### Scope and pricing

- **Network:** Base only — `eip155:8453` (mainnet) and `eip155:84532` (Base Sepolia).
- **Asset:** native USDC on Base.
- **Scheme:** `exact` with EIP-3009 `transferWithAuthorization`.
- **Facilitator:** the official `https://x402.org/facilitator`.
- **Price:** **$0.01 per paid Copilot query**.
- **Free tier:** **10 queries per authenticated user per day**.
- **Subscription:** none in MVP.

Other API endpoints are free and rate-limited. Only Copilot queries consume the free tier or require payment. Phase 2 may add Ethereum, Arbitrum, and Solana payment networks, but they are not accepted in MVP.

### Headers (x402 V2)

| Header | Direction | Content |
|---|---|---|
| `PAYMENT-REQUIRED` | server → client | Base64-encoded `PaymentRequirements` array and error context |
| `PAYMENT-SIGNATURE` | client → server | Base64-encoded `PaymentPayload` with the EIP-3009 authorization and signature |
| `PAYMENT-RESPONSE` | server → client | Settlement result, network, payer, and transaction reference |

The client **does not** send a transaction hash. It signs an EIP-3009 authorization; the facilitator submits `transferWithAuthorization`.

### Middleware flow

```
Incoming request
    ↓ Rate limiter (per IP, per user)
    ↓ Auth middleware (JWT validation)
    ↓ x402 middleware
      Is the endpoint paid (Copilot query)?
        NO  → pass through
        YES → Is the user within the 10/day free tier?
                YES → decrement the free counter in Postgres, pass through
                NO  → Is PAYMENT-SIGNATURE present?
                        NO  → HTTP 402 + PAYMENT-REQUIRED
                        YES → 1. Decode and schema-validate the payload
                              2. Verify network = eip155:8453 or eip155:84532
                              3. Verify asset = Base USDC, scheme = exact
                              4. Verify payTo, amount, and resource
                              5. Verify validAfter <= now < validBefore
                              6. Verify the EIP-3009 nonce has not been used (facilitator + local payment record)
                              7. Call facilitator /verify
                              8. Call facilitator /settle
                              9. On success: write PAYMENT/PAYMENT_SETTLED, return PAYMENT-RESPONSE
                             10. On failure: write PAYMENT/PAYMENT_FAILED, return 402
```

**Replay protection** is the authorization's `validAfter`/`validBefore` window plus the EIP-3009 nonce and the facilitator's settlement record. There is **no custom 15-minute freshness check and no 24-hour Redis nonce**. The old custom nonce scheme is removed.

**Charging rule:** settlement occurs only for a successful (2xx) response. A 5xx response is not charged; the payment record is marked `FAILED` and the authorization is not settled. The free-tier counter is decremented only on a successful response.

**Free-tier counter:** per authenticated user, stored in Postgres, reset at 00:00 UTC, reconciled with `payments` records. The counter is not trusted as the only source of truth; the payment ledger is authoritative for paid usage.

### HTTP 402 response

```http
HTTP/1.1 402 Payment Required
PAYMENT-REQUIRED: <base64-encoded PaymentRequirements[]>
Content-Type: application/json

{
  "error": {
    "code": "PAYMENT_REQUIRED",
    "message": "Payment required for this Copilot query",
    "requestId": "req_..."
  }
}
```

Decoded `PAYMENT-REQUIRED` content:

```json
[
  {
    "scheme": "exact",
    "network": "eip155:8453",
    "maxAmountRequired": "10000",
    "resource": "https://api.crestflow.xyz/v1/copilot/query",
    "description": "CrestFlow Copilot query",
    "mimeType": "application/json",
    "payTo": "0xCrestFlowTreasuryBase",
    "maxTimeoutSeconds": 60,
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "extra": { "name": "USD Coin", "version": "2" }
  }
]
```

`10000` base units = $0.01 at 6 decimals.

### Agent access

- **Anonymous:** no account; every request is paid at $0.01. `userId = null` in audit entries; rate-limited by IP.
- **Registered API key:** tied to a user; receives the 10/day free tier and standard rate limits. API keys are hashed at rest.

Pre-funded credit balances are Phase 2. There is no subscription or credit ledger in MVP.

---

## Passive Event Bus Architecture

```
Engines, Router, Planner, Policy, CCE, Orchestrator, Auth, KYC, x402, System
    ↓ (domain events — durable, at-least-once delivery)
Message bus (BullMQ / Redis Streams)
    ↓
Audit Layer event consumer
(consume → validate category/action pair → encrypt PII → INSERT)
    ↓
PostgreSQL audit_entries (INSERT-only, monthly partitioned)
```

**Idempotency:** `eventId` is unique and is the dedup key. At-least-once delivery cannot create duplicate entries. The consumer inserts into `audit_event_dedupe` in the same transaction as the audit row; a primary-key conflict means the event was already processed and is acknowledged and dropped.

**Recovery:** if the consumer falls behind, it emits `SYSTEM`/`AUDIT_RECOVERED` with `{ queueDepth, missedEventCount, recoveryDuration }` so time gaps are self-documenting. Deletions are impossible at the database-role level.

---

## Resolved Items

1. **KYC revocation** — `category = 'KYC'`, `action = 'KYC_REVOKED'`, status `SUSPENDED`, and the Orchestrator circuit-breaker rule in `docs/identity_kyc.md`.
2. **Message bus** — BullMQ on Redis for MVP; the audit consumer is a dedicated worker. Kafka is a Phase 2 option if volume requires it.
3. **Regulatory export format** — CSV/JSON export endpoint exists; regulator-specific mapping remains a legal/compliance deliverable.
4. **Agent credit ledger** — Phase 2. MVP supports per-request payment and API keys.
5. **GMX fulfillment events** — the `FULFILLMENT` action is reserved but unreachable in MVP because GMX is manual-only (D2).
6. **x402 version and facilitator** — V2 headers, official facilitator, Base USDC, EIP-3009, $0.01/query, 10 free/day (D4/T17).
