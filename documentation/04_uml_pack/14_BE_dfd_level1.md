# 14 — BE Report: DFD Level 1 (Subsystem Decomposition)

**Pack:** CrestFlow UML & Architecture Documentation Pack  
**Version:** 1.0 | **Source docs:** `system_architecture.md` v2.0, `implementation_spec.md` v1.0, all engine docs v2.0  
**Primary deliverable:** Lucidchart-ready DFD Level 1 specification.

---

## 1. Purpose & Scope

Decomposes the CrestFlow central process (Level 0) into **8 major subsystems** with data flows between them and to/from external entities. Each subsystem corresponds to a coherent architectural layer or functional domain.

**DFD Level 1 scope:** Shows internal process decomposition. External entities from Level 0 appear on the boundary. Data stores (PostgreSQL, Redis) appear between subsystems.

---

## 2. Subsystem Inventory

| Process # | Process Name | Key Responsibilities | Key Internal Components |
|---|---|---|---|
| 1.0 | User & Identity Subsystem | Google OAuth integration, JWT issuance, wallet provisioning (Path A: Turnkey; Path B: challenge-response), KYC session management, session lifecycle | Identity Module (NestJS), Auth Guards, KYC Controller, Turnkey Service |
| 2.0 | Intelligence Engine Pipeline | AI-driven portfolio analysis: drift detection, risk scoring, yield opportunity discovery, strategy optimization, user intelligence, execution intelligence | Engines 1–6, Financial Knowledge Layer, Engine Bus (BullMQ) |
| 3.0 | Execution Pipeline | Builds and executes transactions: routing, instruction building, simulation, policy enforcement, CCE signing and broadcasting, nonce management | Execution Router, Orchestrator/Planner, Simulation Engine, Policy Engine, CCE |
| 4.0 | Protocol Layer | Chain-specific and venue-specific adapters: DEX swaps, lending, CCTP bridge; gas estimation per chain; protocol safety module | Protocol Layer adapters (EVM/Solana), CCTP Bridge Adapter, Shared `@crestflow/pool-type` library |
| 5.0 | Portfolio State Management | Maintains real-time portfolio view: snapshots, positions, PSM reconciliation after execution | Portfolio State Manager (PSM), Portfolio State Analysis module |
| 6.0 | Audit & Compliance | Immutable audit trail for all system events; deduplication; partitioned by time | Audit Layer (NestJS service), `audit_entries` (partitioned), `audit_event_dedupe` |
| 7.0 | Payments (x402) | x402 HTTP payment middleware; free tier enforcement; EIP-3009 authorization verification and settlement | x402 Middleware, Payment Controller, `payments` table |
| 8.0 | External Integration Hub | Price feeds, provider health, webhook ingestion, oracle reads | Price Feed Service, Webhook Consumer, Provider Health Monitor |

---

## 3. Data Store Inventory

| Store ID | Store Name | Technology | Contents | Accessed By |
|---|---|---|---|---|
| DS-1 | Primary Database | PostgreSQL 16 | All persistent data: users, wallets, KYC, executions, audit, payments, snapshots | Processes 1.0–7.0 |
| DS-2 | Cache / Queue Store | Redis | Price cache (30s TTL), holdings cache, BullMQ job queues, nonce locks, session data | Processes 2.0, 3.0, 8.0 |

---

## 4. Data Flow Table

| Flow ID | From | To | Data Flow Label | Trigger | Notes |
|---|---|---|---|---|---|
| L1-01 | Path A/B User (ext) | 1.0 User & Identity | Authentication Request | User initiates sign-in | OAuth code, wallet address, signatures |
| L1-02 | Google OAuth (ext) | 1.0 User & Identity | OAuth Token + Identity | OAuth token exchange | googleSub, email |
| L1-03 | HyperVerge KYC (ext) | 1.0 User & Identity | KYC Webhook Decision | HyperVerge delivers webhook | APPROVED/REJECTED/RESUBMISSION_REQUESTED |
| L1-04 | Turnkey TEE (ext) | 1.0 User & Identity | Wallet Addresses | After sub-org + key derivation | evmAddress, solanaAddress |
| L1-05 | 1.0 User & Identity | Turnkey TEE (ext) | Sub-org + Key Derivation Request | Path A wallet provisioning | SECP256K1 + ED25519 key requests |
| L1-06 | 1.0 User & Identity | HyperVerge KYC (ext) | KYC Session Request | User submits KYC | workflowId, transactionId |
| L1-07 | 1.0 User & Identity | DS-1 | User + KYC + Session Records | On auth/wallet/KYC events | users, kyc_records, auth_sessions, address_scopes |
| L1-08 | 1.0 User & Identity | 6.0 Audit & Compliance | Auth + KYC Audit Events | All auth/KYC state changes | LOGIN, WALLET_PROVISIONED, KYC_APPROVED etc. |
| L1-09 | 1.0 User & Identity | Path A/B User (ext) | JWT Tokens + Onboarding Status | After successful auth | accessToken, refreshToken, onboardingStatus |
| L1-10 | DS-1 | 2.0 Intelligence Engines | User + Portfolio State | Engine pipeline triggered | goal_profiles, positions, risk_snapshots |
| L1-11 | DS-2 | 2.0 Intelligence Engines | Cached Price + Pool Data | Cache hit for analysis | prices, APYs, pool data (various TTLs) |
| L1-12 | 8.0 External Integrations | 2.0 Intelligence Engines | Live Market Data | Cache miss; on-demand fetch | prices, TVL, protocol safety, APY |
| L1-13 | 2.0 Intelligence Engines | Gemini LLM API (ext) | AI Analysis Prompt | Per-engine AI call | Structured context + query |
| L1-14 | Gemini LLM API (ext) | 2.0 Intelligence Engines | AI Analysis Response | LLM response | Insights, strategy text, scores |
| L1-15 | 2.0 Intelligence Engines | DS-1 | Engine Output Records | After each engine run | risk_snapshots, yield_opportunities, strategy_plans, plan_actions |
| L1-16 | 2.0 Intelligence Engines | DS-2 | Updated Cache | After fresh data fetch | Refreshed prices, APYs |
| L1-17 | 2.0 Intelligence Engines | 3.0 Execution Pipeline | Execution Trigger + Plan | AUTOPILOT drift detected OR manual approval | executionIntentId, planId, routes |
| L1-18 | 3.0 Execution Pipeline | DS-1 | Execution Records | Throughout pipeline | execution_intents, instructions, simulations, policy_decisions, executions, nonces |
| L1-19 | DS-1 | 3.0 Execution Pipeline | Policy Input Data | Policy check assembly | users, kyc_records, risk_snapshots, goal_profiles, address_scopes, audit_entries |
| L1-20 | 3.0 Execution Pipeline | 4.0 Protocol Layer | Execution Instructions | Per action | action type, chain, venue, amountIn (BaseUnits) |
| L1-21 | 4.0 Protocol Layer | 3.0 Execution Pipeline | Encoded Transaction Calldata | Venue adapter output | ABI-encoded calldata, gas estimates |
| L1-22 | 3.0 Execution Pipeline | Turnkey TEE (ext) | Transaction Signing Request | AUTOPILOT signing | unsigned tx hex, key ID |
| L1-23 | Turnkey TEE (ext) | 3.0 Execution Pipeline | Signed Transaction | AUTOPILOT signing | signed tx hex |
| L1-24 | 3.0 Execution Pipeline | Blockchain Networks (ext) | Signed Transactions | CCE broadcasts | eth_sendRawTransaction / sendTransaction |
| L1-25 | Blockchain Networks (ext) | 3.0 Execution Pipeline | Transaction Receipts + Webhooks | After on-chain confirmation | Alchemy / Helius webhooks |
| L1-26 | 4.0 Protocol Layer | Circle CCTP (ext) | Burn Transaction | CCTP cross-chain | depositForBurn calldata |
| L1-27 | Circle CCTP (ext) | 4.0 Protocol Layer | CCTP Attestation | Iris polling response | attestation hex + messageBytes |
| L1-28 | 3.0 Execution Pipeline | 5.0 Portfolio State Mgmt | Reconciliation Request | After confirmed execution | executedActions[], txReceipt |
| L1-29 | 5.0 Portfolio State Mgmt | DS-1 | Updated Positions + Snapshots | After reconciliation | positions, portfolio_snapshots |
| L1-30 | 5.0 Portfolio State Mgmt | 2.0 Intelligence Engines | Fresh Portfolio State | After reconciliation | Triggers re-analysis |
| L1-31 | 3.0 Execution Pipeline | 6.0 Audit & Compliance | Execution Audit Events | Throughout execution | POLICY_EVALUATED, SIGNED, BROADCAST, CONFIRMED |
| L1-32 | 6.0 Audit & Compliance | DS-1 | Audit Entries | All events | audit_entries (partitioned), audit_event_dedupe |
| L1-33 | Path A/B User (ext) | 7.0 Payments (x402) | Copilot Query + Payment Auth | POST /v1/copilot/query | query + EIP-3009 PAYMENT-SIGNATURE |
| L1-34 | 7.0 Payments (x402) | x402.org Facilitator (ext) | Payment Verification + Settlement | Per paid query | EIP-3009 authorization |
| L1-35 | x402.org Facilitator (ext) | 7.0 Payments (x402) | Settlement Confirmation | After settlement | txHash, settledAt |
| L1-36 | 7.0 Payments (x402) | 2.0 Intelligence Engines | Authorized Copilot Query | After free/paid check | Forwarded query to Copilot Engine (E5/Engine 1) |
| L1-37 | 7.0 Payments (x402) | DS-1 | Payment Records | Per query | payments table |
| L1-38 | 7.0 Payments (x402) | 6.0 Audit & Compliance | Payment Audit Events | Per payment | PAYMENT_REQUIRED, PAYMENT_SETTLED, PAYMENT_FAILED |
| L1-39 | 8.0 External Integrations | DS-2 | Cached Market Data | After external fetch | prices, APY, protocol safety |
| L1-40 | 8.0 External Integrations | DS-1 | Provider Health Records | Periodic health checks | provider_health, webhook_events |
| L1-41 | Alchemy / Helius (ext) | 8.0 External Integrations | RPC Data + Webhooks | On-demand + push | Blockchain state, tx confirmations |
| L1-42 | CoinGecko / DeFiLlama / Chainlink / Pyth (ext) | 8.0 External Integrations | Market + Protocol Data | On-demand | Prices, TVL, APY, oracle reads |
| L1-43 | 5.0 Portfolio State Mgmt | Path A/B User (ext) | Portfolio Dashboard Data | On user request | Portfolio value, positions, risk, yield |

---

## 5. External Entity Connections

| External Entity | Connects To Subsystem(s) | Flow Direction | Key Data |
|---|---|---|---|
| Path A/B User | 1.0 (auth/wallet), 7.0 (Copilot), 5.0 (read portfolio) | Bidirectional | Credentials, queries, portfolio view |
| Admin | 1.0 (user management) | Bidirectional | Admin commands, health reports |
| Google OAuth | 1.0 | Bidirectional | Token exchange |
| Turnkey TEE | 1.0 (provisioning), 3.0 (signing) | Bidirectional | Wallet addresses, signed txs |
| HyperVerge KYC | 1.0 | Bidirectional | KYC session, webhook |
| Alchemy | 3.0 (broadcast/confirm), 8.0 (RPC + webhooks) | Bidirectional | Signed txs, receipts, webhooks |
| Helius | 3.0 (broadcast/confirm), 8.0 (RPC + webhooks) | Bidirectional | Solana txs, webhooks |
| Circle CCTP | 4.0 (bridge), 3.0 (via Protocol Layer) | Bidirectional | Burn tx, attestation |
| x402.org | 7.0 | Bidirectional | Payment verify/settle |
| Gemini LLM | 2.0 | Bidirectional | AI prompts, responses |
| CoinGecko | 8.0 | Input | Prices |
| DeFiLlama | 8.0 | Input | Protocol data |
| Chainlink / Pyth | 8.0 (via RPC) | Input | On-chain price oracles |
| Blockchain Networks | 3.0 (broadcast), 4.0 (simulate) | Bidirectional | Signed txs, receipts |

---

## 6. Layout Recommendations

**Suggested Lucidchart layout:**

```
[External entities — top row]
Path A User   Path B User   Admin   Google OAuth   HyperVerge   Turnkey TEE
                     ↓           ↓          ↓             ↓           ↓
              [1.0 User & Identity]
                     ↓
         [2.0 Intelligence Engine Pipeline] ←→ Gemini LLM API
                 ↙              ↘
[8.0 External Integrations]  [7.0 Payments (x402)] ←→ x402.org
        ↓                           ↓
[CoinGecko/DeFiLlama/Chainlink/Pyth]  [Path A/B User]

              [3.0 Execution Pipeline]
                 ↓               ↘
[4.0 Protocol Layer]        [5.0 Portfolio State Mgmt]
        ↓                           ↓
[Blockchain Networks]        [Path A/B User]
Circle CCTP
                     ↓
              [6.0 Audit & Compliance]

[DS-1 PostgreSQL] — shared store spanning bottom
[DS-2 Redis] — shared cache spanning middle
```

**Practical tips:**
- Place all 8 process boxes in a 3×3 grid with DS-1 and DS-2 running as horizontal bands behind them.
- External entities on the diagram perimeter.
- Data flows between internal processes should be clearly labeled but can be abbreviated if the diagram gets crowded — use abbreviated labels with a legend table on the same page.
- Color code subsystems: Identity (blue), Intelligence (green), Execution (orange), Protocol (brown), PSM (teal), Audit (red), Payments (purple), Integration Hub (grey).

---

## 7. Lucidchart Element Table

| Element | Shape | Label | Color |
|---|---|---|---|
| Subsystem 1.0 | Rounded rectangle | "1.0 / User & Identity" | Light blue |
| Subsystem 2.0 | Rounded rectangle | "2.0 / Intelligence Engine Pipeline" | Light green |
| Subsystem 3.0 | Rounded rectangle | "3.0 / Execution Pipeline" | Light orange |
| Subsystem 4.0 | Rounded rectangle | "4.0 / Protocol Layer" | Tan / Light brown |
| Subsystem 5.0 | Rounded rectangle | "5.0 / Portfolio State Management" | Teal |
| Subsystem 6.0 | Rounded rectangle | "6.0 / Audit & Compliance" | Light red |
| Subsystem 7.0 | Rounded rectangle | "7.0 / Payments (x402)" | Light purple |
| Subsystem 8.0 | Rounded rectangle | "8.0 / External Integration Hub" | Light grey |
| DS-1 PostgreSQL | Open-ended rectangle (DFD) | "DS-1 / PostgreSQL 16" | White, grey border |
| DS-2 Redis | Open-ended rectangle (DFD) | "DS-2 / Redis" | White, dashed border |
| External entities | Sharp rectangle | Per entity name | Light grey |
| Data flow arrows | Directional arrow | Per L1-XX label | Black, 1.5px |

---

## 8. Flags & Open Questions

| Flag ID | Type | Description |
|---|---|---|
| F-14-01 | **Conditional** | HyperVerge KYC connector (L1-03, L1-06) may change to Persona or Sumsub depending on D9 spike outcome. |
| F-14-02 | **Phase 2** | Kafka event bus (Phase 2) would replace some BullMQ internal flows. DS-2 diagram would add Kafka as a third store. |
| F-14-03 | **Assumption** | The Intelligence Engine Pipeline (2.0) includes Copilot Engine (Engine 5 — User Intelligence). The x402 payment gateway (7.0) forwards the Copilot query to 2.0 after payment validation. |
| F-14-04 | **Grouping** | Alchemy and Helius appear in both 3.0 (direct RPC in CCE) and 8.0 (webhook consumer). This dual-connection is correct and should be shown as two separate arrow groups to each subsystem. |
