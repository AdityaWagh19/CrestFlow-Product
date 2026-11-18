# Portfolio State Manager

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` T6, T7, D5  
**Canonical schemas:** `docs/canonical_schemas.md` §5, §14  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

**Layer role:** Sync, Tracking, Monitoring, Updates — closes the execution loop back into the Core Intelligence Layer.

---

## Purpose

The final component in the execution stack. After every execution or scheduled scan, it updates the canonical portfolio state, propagates changes to downstream engines, and monitors for urgent conditions that warrant an early rescan.

The PSM owns post-execution synchronization and reconciliation. It does **not** own the CCTP bridge lifecycle (D5) and does not compute valuation or risk.

---

## The Four Jobs

### Job 1 — Post-Execution Sync

**Trigger:** `ExecutionConfirmed` event from the Orchestrator.

**Scope:** target `event.chains` only — not every chain the user has configured. A CCTP bridge event includes both the source and destination chain.

```typescript
on ExecutionConfirmed({ userId, txIds, chains }) {
  // Wait for the indexer to reflect the confirmed transactions (bounded poll)
  await pollUntilIndexed(txIds, chains, {
    intervals: { solana: 500, evm: 1500 },   // ms
    maxWait: 10000                             // ms; log INDEXER_LAG_WARNING if exceeded
  })
  // Trigger an Engine 1 scan scoped to event.chains
  await triggerScan(userId, chains)
}
```

On `maxWait` timeout, mark the affected positions `dataQuality = 'PARTIAL'`, emit `INDEXER_LAG_WARNING`, and schedule a re-sync after 30 seconds. Do not block the execution result on indexing.

### Job 2 — Delta Tracking

```typescript
// Every 30s while the user has active positions
displayedValue = Σ(holding.quantity × priceCache[holding.symbol])
// Uses the 30s-TTL live price cache (T6)
// Does NOT write a new snapshot
// Does NOT trigger engines
```

### Job 3 — Monitoring (Watchdog)

**Emergency rescan triggers:**

| Trigger | Condition | Scope |
|---|---|---|
| Liquidation proximity | HF < 1.2 on any lending position | That chain only |
| Significant price move | >8% in 1 hour for any held asset | All chains |
| New undetected transaction | Transaction not in any open Execution plan lands on a monitored address | That chain only |
| Protocol TVL incident | TVL drop >20% in 24h for a protocol the user is exposed to | That chain only |

**Health-factor thresholds are absolute.** The protocol liquidation point (HF = 1.0) is fixed by the protocol, not the user's risk profile. Persona scaling stays in Engine 2's threshold system.

**Scheduled scan frequency by position type:**

| Position Type / HF Band | Scan Frequency |
|---|---|
| Lending (HF 1.5+) | Every 1 hour |
| Lending (HF 1.3–1.5) | Every 15 minutes |
| Lending (HF < 1.3) | Every 5 minutes (CRITICAL) |
| LP positions | Every 1 hour |
| Idle wallet balance | Every 4 hours |

**Scan scheduling (multi-user staggering):**

```typescript
offsetSeconds = hash(userId) % scanIntervalSeconds
nextScanTime  = lastScanTime + scanIntervalSeconds + offsetSeconds
// Stable per-user offset spreads users across each interval window
// Recalculated on tier change
```

### Job 4 — Downstream Propagation

After a scan completes:

```typescript
const change = computeChange(newSnapshot, priorSnapshot)

if (Math.abs(change.percentChange) < 1) {
  // Suppress Engine 4 (no strategy reoptimization needed)
  // Still run Engine 2 (risk rescoring) and Engine 3 (yield opportunity update)
} else {
  // Trigger Engine 1 → Engine 2 / Engine 3 in parallel → Engine 4
}

// Always emit PortfolioStateUpdated
// Always write the POST_EXECUTION snapshot (immutable audit record)
```

Engine 2 and Engine 3 run in parallel after Engine 1; Engine 4 consumes both. The topology is canonical in `core_documents/system_architecture.md`.

---

## Snapshot Retention

| Snapshot Type | Retention |
|---|---|
| Daily | Last 90 days hot, 2 years cold |
| `POST_EXECUTION` | Permanent immutable audit record |
| First-of-month | Long-term tracking |

Pruning runs via the `crestflow:maintenance` BullMQ queue. The append-only constraint on `audit_entries` is respected — pruning only removes `portfolio_snapshots` rows within retention windows, never audit entries.

Canonical retention values are in `docs/implementation_spec.md` §3.3.

---

## Webhook Subscriptions

Address-activity webhooks fire within seconds of on-chain confirmation for normal-case external transaction detection.

```typescript
// EVM: one subscription per chain per address
// (the same address on Ethereum/Base/Arbitrum requires 3 separate Alchemy Notify subscriptions)
subscribe(Alchemy, network: 'ethereum',  address: embeddedEVMAddress)
subscribe(Alchemy, network: 'base',      address: embeddedEVMAddress)
subscribe(Alchemy, network: 'arbitrum',  address: embeddedEVMAddress)

// Solana
subscribe(Helius, address: embeddedSolanaAddress)

// External addresses (Path B, read-only/manual tracking), per active chain
subscribe(...)
```

**Signature verification (T7):**

- Alchemy Notify: verify `X-Alchemy-Signature` (HMAC SHA-256) with the signing key.
- Helius: verify the static `authHeader` and keep the webhook URL secret; Helius does not sign payloads.
- Reject unsigned or unknown events.
- Deduplicate by `(provider, event_id)`; order by block height and log index.

**10-minute reconciliation sweep (fallback):**

```typescript
// Every 10 minutes, per monitored address:
const latest = indexer.getLatestActivity(address, chain)
const recorded = db.getLastRecordedActivity(address, chain)
if (latest.blockHeight > recorded.blockHeight) {
  triggerCatchUpScan(address, chain)
  log('WEBHOOK_MISS')
}
```

Worst-case external-transaction detection latency is ~10 minutes on total webhook failure. Normal-case latency remains single-digit seconds.

---

## Bridge Completion Monitoring

**Scope: externally initiated bridges only.** CrestFlow-initiated bridges are tracked by CCE end-to-end; the PSM does not duplicate that watch.

```typescript
if (detected CCTP mint on monitoredAddress AND no matchingOpenExecution) {
  // User bridged in manually, outside any CrestFlow-planned action
  triggerScan(userId, targetChain)
}
```

"Open execution" is any execution in a non-terminal state for the same user, source chain, destination chain, and amount within the last 2 hours. This window is a configuration value.

---

## `poolType` Non-Applicability

The PSM does not perform valuation math. The shared `@crestflow/pool-type` package classifies positions for Engines 1–3. GMX GM positions are covered by the price-movement and TVL watchdogs; no special case is needed here.

---

## Bridge Reconciliation

CCTP V2 is a 1:1 burn-and-mint of USDC. Standard transfers have no protocol fee; Fast transfers deduct the quoted `maxFee`. Reconciliation compares:

```
expectedReceivedAmount = amount - maxFee
actualReceivedAmount   = destination balance delta
```

If `actualReceivedAmount < expectedReceivedAmount`, open a `BRIDGE_AMOUNT_VARIANCE` item and audit it; do not silently adjust the snapshot. There is no AMM slippage in CCTP.

Phase 2 wrapped-asset bridges would use the actual on-chain amount as ground truth and flag variances above 0.5%.

---

## What Portfolio State Manager Does NOT Do

| Not Portfolio State Manager's Job | Whose Job |
|---|---|
| Metric/risk computation | Engine 2 |
| Yield opportunity scoring | Engine 3 |
| Strategy optimization | Engine 4 |
| CCTP burn/attest/mint lifecycle | CCE |
| Price fetching | FKL live-price TTL cache |
| Valuation / IL math | Engines 1–3 |

---

## Resolved Items

1. **Indexer-lag `maxWait`** — T7: 10 seconds; on timeout, `PARTIAL` plus a 30-second re-sync.
2. **Reconciliation sweep cost at scale** — batched per provider; cost review is an implementation task, not an open design question.
3. **Externally initiated bridge false positives** — the 2-hour matching window and non-terminal execution lookup are specified above.
4. **CCTP fee wording** — corrected: Standard is free; Fast uses a quoted fee capped at 0.10%/$5.
