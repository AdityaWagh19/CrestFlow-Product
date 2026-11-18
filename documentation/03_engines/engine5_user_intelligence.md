# Engine 5 — User Intelligence Engine

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D8, T2, T13, T17  
**Canonical schemas:** `docs/canonical_schemas.md` §3–§4  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Part 1 — User Intelligence (Persona & Goal Profiling)

### Purpose

Derive a stable behavioural persona from onboarding questionnaire responses and ongoing behavioural signals. Produce a `GoalProfile` that constrains every downstream optimisation, strategy, and execution decision.

---

### 7-Question Onboarding Questionnaire

| # | Question | Dimension |
|---|---|---|
| 1 | Investment time horizon | Aggression |
| 2 | Drawdown tolerance | Aggression |
| 3 | Volatility comfort | Aggression |
| 4 | Capital importance | Aggression |
| 5 | DeFi experience level | Aggression |
| 6 | Yield vs. growth preference | Yield Tilt |
| 7 | Primary financial goal | Yield Tilt |

---

### Two-Score Persona Model

```typescript
aggressionScore = Σ(answer[i].score × weight[i])   // questions 1–5
yieldTiltScore  = Σ(answer[i].score × weight[i])   // questions 6–7

aggressionDrift = Σ(ACTED_ON_REBALANCE, DISMISSED_REBALANCE signals)
yieldDrift      = Σ(DEPLOYED_IDLE_CAPITAL, DISMISSED_IDLE_CAPITAL signals)

adjustedAggressionScore = clamp(aggressionScore + aggressionDrift,
                                aggressionScore − 20, aggressionScore + 20)
adjustedYieldTiltScore  = clamp(yieldTiltScore + yieldDrift,
                                yieldTiltScore − 20, yieldTiltScore + 20)
```

**Persona resolution:**

```typescript
function resolvePersona(aggressionScore: number, yieldTiltScore: number): Persona {
  let persona: Persona
  if      (aggressionScore <= 20) persona = 'CAPITAL_PRESERVATION'
  else if (aggressionScore <= 40) persona = 'CONSERVATIVE'
  else if (aggressionScore <= 60) persona = 'MODERATE'
  else                            persona = 'AGGRESSIVE'

  if (yieldTiltScore >= 70 && persona !== 'CAPITAL_PRESERVATION') {
    persona = 'YIELD_SEEKER'
  }
  return persona
}
```

**Boundary-crossing trigger:** if either adjusted score crosses a bucket boundary, surface a "your investment style may have changed" prompt and route to the questionnaire refresh. A `GOAL_CHANGE` Copilot intent routes here — never to a silent persona update.

**Launch strategy:** rule-based scoring at launch. Phase 2 (500+ users, 30+ days of signal): HDBSCAN clustering on the two-score vector, with rule-based assignment as fallback.

---

### Engine 2 Persona Multipliers

| Persona | Multiplier |
|---|---:|
| `CAPITAL_PRESERVATION` | 0.5× |
| `CONSERVATIVE` | 0.7× |
| `MODERATE` | 1.0× |
| `AGGRESSIVE` | 1.3× |
| `YIELD_SEEKER` | 1.15× |

These are canonical in T13 and are consumed by Engine 2 to produce `adjustedThreshold`. Engine 5 does not compute risk thresholds.

---

### `GoalProfile` Schema

Canonical type: `docs/canonical_schemas.md` §4.

```typescript
GoalProfile {
  persona:                  Persona,
  maxVolatilePercent:       Percent,
  minStablecoinPercent:     Percent,
  maxSingleAssetPercent:    Percent,
  maxSingleProtocolPercent: Percent,
  maxSingleChainPercent:    Percent,
  maxUsdPerTransaction:     Usd,          // per-action cap read by Policy Check 7
  maxSlippagePercent:       Percent,      // may lower, never raise, the T3 cap
  driftTolerancePercent:    Percent,      // Engine 4 gate
  rebalancingFrequencyCap:  number,       // days; Engine 4 cooldown
  allowedActionTypes:       ActionType[], // required, non-empty; Policy Check 9
  topsisWeights:            TopsisWeights
}
```

**Required fields:** `allowedActionTypes` is mandatory and non-empty for any user who can execute. `maxSlippagePercent` must be at or below the hard 1.00% cap; the effective cap is `min(goalProfile.maxSlippagePercent, poolTypeCap)`.

There is **no** `maxRiskScore` on `GoalProfile`. Engine 4's defensive override reads Engine 2's `adjustedThreshold`.

---

## Part 2 — AI Copilot

### Model configuration (D8)

| Role | Model ID |
|---|---|
| Primary | `google/gemini-3.8-flash` |
| Fallback | `google/gemini-flash-lite-latest` |

- Model IDs live in configuration, never in code.
- The deprecated `GPT-4.1-mini` and `Gemini 2.5 Flash` pins are removed.
- A provider abstraction allows adding OpenAI later without changing the engine.
- Model failure never changes a financial result; the Copilot falls back to deterministic templates.
- An evaluation set covers intent classification, numeric faithfulness, and refusal behaviour.
- Prompt-injection boundaries and tool-call allowlists are specified in `docs/implementation_spec.md` §8.

---

### Architecture

```
User message
    ↓
[1] Session load (Redis, last 10 turns)
    + persistent preferences load (DB, per userId)
    ↓
[2] Intent classification (LLM call #1, Flash-Lite)
    → 15 intent types
    ↓
[3] Context assembly (data pull from Engines 1–4)
    BRIDGE_REQUEST: resolve visibility vs. transfer here
    ↓
[4] x402 check (free-tier counter, then $0.01 per query)
    ↓
[5] Prompt construction
    language mirrored to the user's input language
    persistentPreferences in a pinned context block
    ↓
[6] LLM call #2 (response generation)
    primary:  google/gemini-3.8-flash
    fallback: google/gemini-flash-lite-latest
    ↓
[7] Structured output validation
    ↓
[8] SSE stream to frontend
    ↓
[9] Session update
    append turn to Redis
    persist explicit durable preference statements to DB
    ↓
[10] Telemetry + payment audit
    model, latency, tokens, numeric-faithfulness check
    PAYMENT/PAYMENT_SETTLED only when a paid query is charged
```

Chat content is not written to `audit_entries`. Paid queries produce `PAYMENT` entries; model usage and latency go to observability metrics. This is a privacy decision and is consistent with the T10 taxonomy.

---

### Intent Table (15)

| Intent | Pipeline | Notes |
|---|---|---|
| `PORTFOLIO_QUERY` | Engine 1 read | Balance, allocation, value |
| `RISK_QUERY` | Engine 2 read | Risk score, alerts, HF |
| `GOAL_PROGRESS_QUERY` | Engine 4 read | Progress against goals |
| `STRATEGY_QUERY` | Engine 3/4 read | Existing plan explanation |
| `YIELD_QUERY` | Engine 3 read | Ranked opportunities |
| `REBALANCE_REQUEST` | Engine 3/4 read | Narrates pre-computed benefit; no inline math |
| `YIELD_DEPLOY_REQUEST` | Engine 3/4 read | Same |
| `CROSS_CHAIN_COMPARISON` | Engine 3 read | Cross-chain yield comparison |
| `BRIDGE_REQUEST` | Engine 6 handoff | Disambiguates visibility vs. actual transfer |
| `GOAL_CHANGE` | Questionnaire refresh | Never a silent persona update |
| `MARKET_QUESTION` | General context | No portfolio mutation |
| `CHAIN_QUERY` | Portfolio read | Chain-specific info |
| `GLOSSARY_REQUEST` | Static | DeFi term explanations |
| `GENERAL_FINANCIAL` | General context | Educational, not personalised advice |
| `UNKNOWN` | Free | Never charged; rate-limited |

**Pricing (T17):** every Copilot query consumes the user's free tier; the first **10 queries per user per day** are free, then each query costs **$0.01**. `UNKNOWN` is never charged and does not consume the free tier. The intent table does not carry per-intent prices — the old $0.05/$0.10 schedule is removed.

---

### `BRIDGE_REQUEST` Disambiguation

The embedded wallet's EVM address is shared across Ethereum, Base, and Arbitrum, so a user asking "can you move my USDC to Base?" may already have USDC on Base:

```typescript
if (chainAllocation[destinationChain]?.[requestedAsset] > 0) {
  // visibility answer: "You already have $X USDC on Base, no bridge needed"
} else if (requestedAsset !== 'USDC') {
  // UNSUPPORTED_BRIDGE — non-USDC cross-chain is out of MVP
} else {
  // propose CCTP V2 bridge via Engine 6 / Router / Planner
}
```

Path B users receive an unsigned transaction and sign manually; the Copilot must say so explicitly.

---

### Switching-Benefit Narration

`REBALANCE_REQUEST` and `YIELD_DEPLOY_REQUEST` narrate pre-computed fields from `YieldOpportunitySnapshot` — `netSwitchingBenefitUsd`, `paybackDays`, `switchingBenefitUsd`. The Copilot does not recompute break-even math inline. Numeric values come from the database, not from the model.

---

### Persistent Preferences

```typescript
pinnedContext: {
  userName: string,
  currentPortfolioValueUsd: Usd,
  activeAlerts: RiskAlert[],
  goalPersona: Persona,
  chains: Chain[],
  persistentPreferences: string[]   // DB-backed, cap ~10, explicit statements only
}
```

`persistentPreferences` are written only on an explicit user statement (e.g., "don't use bridges for me"). Explicit removal or questionnaire refresh clears them.

**Downstream binding:** a preference that restricts execution must be mapped to a machine-enforced field, not only to prompt text. "Don't use bridges" sets `AutopilotConfig.bridgeAllowed = false` and removes `BRIDGE` from `GoalProfile.allowedActionTypes`; it is then enforced by Policy Check 9. The Copilot never relies on the model to remember a restriction.

---

### Language Support

The system prompt instructs the model to detect the user's input language and respond in kind. Only human-facing JSON fields (`response`, `disclaimer`, `suggestedFollowUps`) localise; `intent`, `confidence`, `sources`, and machine-facing enums stay in English.

---

### Proactive Alerts

CRITICAL-severity Engine 2 alerts trigger a templated push message, independent of whether a chat session is open. MEDIUM/LOW alerts remain passive.

```
Engine 2 fires CRITICAL alert
    ↓
Copilot renders a deterministic template with the alert data
    ↓
Push via notification channel
    ↓
Notification telemetry (not an audit entry)
```

The push is template-generated, not model-generated, so a model outage cannot suppress a critical alert.

---

### Safety and Numeric Faithfulness

- The Copilot has **no direct execution tool**. It can propose a plan; Engine 6, Router, Planner, Simulation, and Policy remain the only path to execution.
- Financial numbers in responses come from typed query results, not model arithmetic.
- Output is schema-validated; unsupported claims fall back to a deterministic template.
- Prompt injection is contained by a tool allowlist, no arbitrary URL fetch, and no direct data mutation.

---

## Resolved Items

1. **Model pins** — D8: Gemini 3.8 Flash primary, Gemini Flash-Lite fallback; provider abstraction retained.
2. **Copilot pricing** — T17: 10 free queries/day, then $0.01/query; per-intent price table removed.
3. **Copilot audit** — chat content is not audited; payments and execution are. Model telemetry is observability data.
4. **Preference binding** — execution restrictions are written to `GoalProfile`/`AutopilotConfig` and enforced by Policy.
5. **Engine 4 alignment** — `GoalProfile` has no `maxRiskScore`; Engine 4 reads `RiskSnapshot.adjustedThreshold`.
6. **HDBSCAN persona model** — Phase 2, 500+ users and 30+ days of signal.
