# CrestFlow — UML File 12: BE System Architecture Overview

> **Document type:** Backend Project Report — System Architecture Overview  
> **Audience:** Academic / professional reviewers; engineering team  
> **Standard:** Descriptive architecture specification with Lucidchart-ready diagram inventory  
> **Status:** Draft — Testnet / Pre-Mainnet  
> **Companion files:** 01_component_diagram.md, 11_deployment_diagram.md

---

## 1. Purpose & Scope

This document provides a comprehensive, high-level description of the CrestFlow backend system architecture. CrestFlow is an AI-augmented DeFi portfolio management platform enabling users to monitor, analyse, and execute cross-chain DeFi strategies across Ethereum, Base, Arbitrum, and Solana.

The backend is a **TypeScript/NestJS monolith** deployed as horizontally scalable microservice-style processes (API server + BullMQ worker pool), backed by PostgreSQL 16 and Redis, integrating with a rich ecosystem of external providers for blockchain RPC, AI inference, price data, identity, KYC, signing, and cross-chain bridging.

**Scope of this document:**
- Five-layer architecture description and component roles
- Lucidchart-ready high-level diagram element inventory
- Full technology stack table with rationale
- Key architectural decisions (D1–D10)
- End-to-end data flow narrative
- Security architecture
- Scalability and reliability design

---

## 2. Architecture Overview

The CrestFlow backend is organised into **five logical layers**. Each layer has clearly defined responsibilities and communicates with adjacent layers through defined interfaces.

### Layer 1 — Interface Layer

The Interface Layer is the entry point for all client interactions. It encompasses the API Gateway/Load Balancer (which handles HTTPS termination, TLS 1.3, and rate limiting), the NestJS REST controller layer (OpenAPI 3.1-compliant endpoints), and the x402 Payment Middleware.

The x402 Middleware intercepts requests to paid Copilot endpoints. It enforces a 10 free queries/day allowance per user and, once exhausted, requires a valid EIP-3009-signed Base USDC micropayment (`$0.01/query`) delivered via the `PAYMENT-SIGNATURE` HTTP header. The middleware calls the x402.org Facilitator to verify and settle payments before forwarding the request to the Intelligence Layer. The anonymous agent path (no authenticated user) is also supported, with `userId = null` in the payment record.

Webhooks from external providers (Alchemy, Helius, HyperVerge) arrive at dedicated controller endpoints within the Interface Layer and are dispatched to the appropriate domain services.

### Layer 2 — Intelligence Layer

The Intelligence Layer is the analytical core of CrestFlow. It contains six AI-powered engines and the Financial Knowledge Layer:

- **Engine 1 — Portfolio Analyser:** Aggregates multi-chain portfolio positions, computes USD valuations via price feeds (CoinGecko, Chainlink, Pyth), and maintains the Portfolio State Manager.
- **Engine 2 — Market Intelligence:** Fetches and normalises price data, liquidity depth, and market conditions from on-chain oracles (Chainlink for EVM, Pyth for Solana) and off-chain APIs (CoinGecko).
- **Engine 3 — Opportunity Scanner:** Identifies yield opportunities, arbitrage paths, and rebalancing triggers across DEX and lending venues. Checks DeFiLlama for protocol TVL/safety scores.
- **Engine 4 — Risk Assessor:** Computes per-action risk scores incorporating slippage, liquidity, protocol safety, and portfolio concentration. Outputs a `riskScore` used by the Policy Engine.
- **Engine 5 — Strategy Optimiser:** Generates ranked strategy recommendations given portfolio state, market data, and risk constraints.
- **Engine 6 — Copilot (LLM Interface):** Accepts natural-language user queries, calls the Gemini LLM API (`google/gemini-3.8-flash` primary; `google/gemini-flash-lite-latest` fallback), interprets intent, and routes structured action requests to the Execution Layer.

The **Financial Knowledge Layer** provides shared domain logic: venue-specific trade/lending calculations, `@crestflow/pool-type` pool classification (stable, correlated, constant_product), gas estimation, and cross-chain value normalisation.

### Layer 3 — Execution Layer

The Execution Layer translates intelligence outputs into atomic, policy-checked blockchain operations. It contains four key components:

- **Orchestrator / Planner:** Decomposes complex strategies (e.g., bridge + swap + deposit) into ordered execution steps. Manages the saga state machine (`CREATED → ROUTING → BUILDING → SIMULATING → POLICY_CHECK → SIGNING → BROADCASTING → AWAITING_CONFIRMATION → COMPLETED`). Coordinates compensation flows for failed sagas.
- **Execution Router:** Selects the optimal execution path given the user's `ExecutionMode` (AUTOPILOT, MANUAL, READ_ONLY), wallet type (Path A: Turnkey embedded; Path B: external), and target chain.
- **Policy Engine:** Runs all 9 policy checks in order (no short-circuit) before any transaction is signed or broadcast: KYC Gate, KYC Tier Daily Limit, Execution Authority, Risk Gate, Chain Allowlist, Protocol Allowlist, Per-Transaction Value Limit, Slippage Tolerance, Action Type Restrictions. Any failed check terminates the saga with `BLOCKED` status.
- **Cross-Chain Coordinator (CCE):** Manages CCTP V2 bridge operations. Encodes `depositForBurn` calls with the correct domain mapping (ETH=0, ARB=3, SOL=5, BASE=6), polls Circle Iris API for attestation (`BURNED → ATTESTED`), and triggers `receiveMessage` on the destination chain (`ATTESTED → MINTED`).

### Layer 4 — Protocol Layer

The Protocol Layer provides chain-specific and venue-specific adapters that abstract blockchain heterogeneity from the layers above:

- **Chain Adapters:** Separate adapters for EVM chains (Ethereum/Base/Arbitrum — EIP-1559, GasPriceOracle, NodeInterface) and Solana (compute budget, versioned transactions). Adapters interact with Alchemy (EVM RPC) and Helius (Solana RPC).
- **Venue Adapters:** One adapter per MVP venue — Uniswap V3 (ETH/BASE/ARB), Aerodrome (BASE), Jupiter (Solana — `/order` + `/build`), Raydium (Solana), Aave V3 (ETH/BASE/ARB), Moonwell (BASE), Kamino (Solana), MarginFi (Solana). GMX is read-only display only (D2 — no automated execution).
- **CCTP Bridge Adapter:** Encodes and decodes CCTP V2 messages; supports Standard mode (maxFee=0, minFinalityThreshold=2000, 60min timeout) and Fast mode (quoted maxFee capped at 0.10% or $5, minFinalityThreshold=1000, 5min timeout — opt-in for HIGH urgency only).
- **Protocol Safety Module (PSM):** Pre-execution simulation, slippage validation, liquidity checks, and circuit-breaker logic.
- **Turnkey Integration:** All transaction signing is delegated to the Turnkey TEE via HTTPS API. For Path A AUTOPILOT, the app holds delegated signing permissions; for MANUAL mode, the user approves signing in-session. Private keys never exist in application memory.

### Layer 5 — Data Layer

The Data Layer manages all persistent and ephemeral state:

- **PostgreSQL 16:** Primary datastore for users, wallets, portfolios, audit logs, saga states, payment records, KYC records, and all entity relationships. Streaming replication to a read replica for portfolio and audit queries.
- **Redis:** Dual-purpose: (1) application cache for price feeds, portfolio snapshots, and engine outputs; (2) BullMQ backing store for async job queue between the NestJS API and Worker Pool.
- **Portfolio State Manager:** Service that maintains real-time and cached portfolio state, reconciles on-chain data with DB records, and publishes `PortfolioScanTriggered` events via the outbox pattern.
- **Audit Layer:** Append-only audit log in PostgreSQL with structured `AuditCategory` entries (AUTH, KYC, EXECUTION, PREFERENCE, PAYMENT, SYSTEM). All sensitive operations emit audit records. Used for compliance, debugging, and replay protection verification.

---

## 3. Architecture Diagram Specification

### 3.1 Element Inventory Table

> This table defines every box that should appear in the high-level Lucidchart architecture diagram.

| Layer | Element Name | Responsibility |
|---|---|---|
| **Interface** | API Gateway / Load Balancer | HTTPS termination, TLS 1.3, rate limiting, routing to NestJS |
| **Interface** | REST API Controllers | OpenAPI 3.1 endpoint handlers; request validation; auth guard |
| **Interface** | x402 Payment Middleware | Micropayment gating for Copilot: free tier check → 402 → verify → settle → pass-through |
| **Interface** | Webhook Receiver | Inbound webhook dispatch: Alchemy, Helius, HyperVerge |
| **Intelligence** | Engine 1 — Portfolio Analyser | Multi-chain portfolio aggregation and USD valuation |
| **Intelligence** | Engine 2 — Market Intelligence | Price + liquidity data aggregation from oracles and APIs |
| **Intelligence** | Engine 3 — Opportunity Scanner | Yield/arbitrage opportunity detection; DeFiLlama TVL checks |
| **Intelligence** | Engine 4 — Risk Assessor | Per-action risk scoring (riskScore + freshness validation) |
| **Intelligence** | Engine 5 — Strategy Optimiser | Ranked strategy recommendation generation |
| **Intelligence** | Engine 6 — Copilot (LLM) | NL query → Gemini LLM → structured action intent |
| **Intelligence** | Financial Knowledge Layer | Pool-type classification, gas estimation, value normalisation |
| **Execution** | Orchestrator / Planner | Saga decomposition, state machine, compensation flows |
| **Execution** | Execution Router | Path A/B routing; ExecutionMode selection; chain targeting |
| **Execution** | Policy Engine | 9-check policy validation (no short-circuit) before every tx |
| **Execution** | Cross-Chain Coordinator (CCE) | CCTP V2 encode/decode, Circle Iris attestation polling, mint trigger |
| **Protocol** | EVM Chain Adapters (ETH/BASE/ARB) | EIP-1559 / GasPriceOracle / NodeInterface RPC abstraction |
| **Protocol** | Solana Chain Adapter | Compute budget, versioned transactions, Helius RPC |
| **Protocol** | Venue Adapters (DEX) | Uniswap V3, Aerodrome, Jupiter, Raydium trade encoding |
| **Protocol** | Venue Adapters (Lending) | Aave V3, Moonwell, Kamino, MarginFi supply/borrow encoding |
| **Protocol** | CCTP Bridge Adapter | Standard + Fast burn/mint encoding; domain mapping |
| **Protocol** | Protocol Safety Module (PSM) | Simulation, slippage validation, circuit-breaker |
| **Protocol** | Turnkey Integration | TEE delegated signing; sub-org management |
| **Data** | PostgreSQL 16 Primary | Authoritative relational store; write path |
| **Data** | PostgreSQL 16 Read Replica | Read-optimised clone; portfolio + audit queries |
| **Data** | Redis Cluster | Cache + BullMQ backing |
| **Data** | Portfolio State Manager | Real-time state reconciliation; outbox pattern |
| **Data** | Audit Layer | Append-only structured audit log |
| **Data** | BullMQ Worker Pool | Async job execution (saga steps, portfolio scans, etc.) |
| **External** | Turnkey TEE | Cryptographically isolated key signing environment |
| **External** | Alchemy | EVM RPC + Notify webhooks |
| **External** | Helius | Solana RPC + webhooks |
| **External** | Circle Iris API | CCTP V2 attestation |
| **External** | x402.org Facilitator | Payment verify + settle |
| **External** | HyperVerge | KYC session + status webhooks [conditional D9] |
| **External** | Google OAuth | OIDC authentication |
| **External** | Gemini LLM API | AI inference for Copilot |
| **External** | CoinGecko | Off-chain price feeds |
| **External** | DeFiLlama | Protocol TVL/safety data |
| **External** | Blockchain Networks | ETH / BASE / ARB / SOL (mainnet + testnet) |
| **External** | Chainlink Oracle | On-chain EVM price feeds |
| **External** | Pyth Oracle | On-chain Solana price feeds |

### 3.2 Connection Table

| From | To | Flow Description | Protocol |
|---|---|---|---|
| Client | API Gateway | User HTTP request | HTTPS :443 |
| API Gateway | REST API Controllers | Routed internal request | HTTP (VPC) |
| REST API Controllers | x402 Payment Middleware | Copilot endpoint interception | In-process |
| x402 Payment Middleware | x402.org Facilitator | Payment verify + settle | HTTPS :443 |
| REST API Controllers | Engine 6 — Copilot | Copilot query dispatch | In-process |
| Engine 6 — Copilot | Gemini LLM API | NL inference request | HTTPS :443 |
| Engine 6 — Copilot | Orchestrator / Planner | Structured action intent | In-process |
| Engine 1–5 | Financial Knowledge Layer | Shared calculation helpers | In-process |
| Engine 2 — Market Intelligence | CoinGecko | Price feed poll | HTTPS :443 |
| Engine 3 — Opportunity Scanner | DeFiLlama | TVL/safety data poll | HTTPS :443 |
| Orchestrator / Planner | Policy Engine | Pre-execution 9-check validation | In-process |
| Orchestrator / Planner | Execution Router | Execution path selection | In-process |
| Execution Router | Turnkey Integration | Delegated signing request | In-process → HTTPS :443 |
| Turnkey Integration | Turnkey TEE | Sign transaction payload | HTTPS :443 |
| Execution Router | EVM Chain Adapters | Build + broadcast EVM tx | In-process |
| Execution Router | Solana Chain Adapter | Build + broadcast Solana tx | In-process |
| EVM Chain Adapters | Alchemy | EVM RPC calls | HTTPS :443 |
| Solana Chain Adapter | Helius | Solana RPC calls | HTTPS :443 |
| Alchemy | Blockchain Networks (EVM) | On-chain read + broadcast | RPC |
| Helius | Blockchain Networks (Solana) | On-chain read + broadcast | RPC |
| Orchestrator / Planner | Cross-Chain Coordinator (CCE) | CCTP bridge orchestration | In-process |
| CCE | CCTP Bridge Adapter | Burn + mint encoding | In-process |
| CCE | Circle Iris API | Attestation polling | HTTPS :443 |
| REST API Controllers | Google OAuth | Auth code token exchange | HTTPS :443 |
| REST API Controllers | HyperVerge | KYC session creation | HTTPS :443 |
| HyperVerge | Webhook Receiver | KYC status webhook | HTTPS :443 |
| Alchemy | Webhook Receiver | On-chain event notification | HTTPS :443 |
| Helius | Webhook Receiver | Solana event notification | HTTPS :443 |
| REST API Controllers | Portfolio State Manager | Portfolio read/write | In-process |
| Portfolio State Manager | PostgreSQL 16 Primary | Persistent state write | TCP/PostgreSQL :5432 |
| Portfolio State Manager | PostgreSQL 16 Read Replica | Portfolio state read | TCP/PostgreSQL :5432 |
| Audit Layer | PostgreSQL 16 Primary | Audit record write | TCP/PostgreSQL :5432 |
| REST API Controllers | Redis Cluster | Cache read/write | TCP/Redis :6379 |
| REST API Controllers | BullMQ Worker Pool | Async job enqueue | BullMQ/Redis |
| BullMQ Worker Pool | PostgreSQL 16 Primary | Saga state persist | TCP/PostgreSQL :5432 |
| BullMQ Worker Pool | Turnkey TEE | Signing (AUTOPILOT worker) | HTTPS :443 |
| REST API Controllers | KMS / Secrets Manager | Secret retrieval at startup | HTTPS :443 |
| Blockchain Networks (EVM) | Chainlink Oracle | On-chain oracle read | Blockchain read |
| Blockchain Networks (Solana) | Pyth Oracle | On-chain oracle read | Blockchain read |

---

## 4. Technology Stack Table

| Category | Technology | Version / Detail | Rationale |
|---|---|---|---|
| **Runtime** | Node.js | 20 LTS | LTS stability; large async I/O ecosystem; TypeScript native |
| **Framework** | NestJS | Latest stable | Modular DI framework; decorator-based; OpenAPI integration via `@nestjs/swagger` |
| **Language** | TypeScript | 5.x | Type safety across API, worker, and shared packages; enables `@crestflow/pool-type` sharing |
| **API specification** | OpenAPI | 3.1 | Industry standard; enables client SDK generation; aligns with x402 spec requirements |
| **Database** | PostgreSQL | 16 | ACID compliance; JSONB for flexible metadata; streaming replication; TypeORM integration |
| **ORM** | TypeORM | Latest stable | Entity-first migrations; PostgreSQL-native features; compatible with NestJS DI |
| **Cache / Queue backing** | Redis | 7.x (cluster mode) | High-throughput cache; BullMQ native backing; supports pub/sub for event dispatch |
| **Job queue** | BullMQ | Latest stable | Redis-backed; reliable job delivery; retry semantics; saga step execution |
| **Shared package** | `@crestflow/pool-type` | Internal NPM | Pool-type classification shared between API and worker; prevents duplication |
| **Signing / Key management** | Turnkey TEE | Cloud API | Cryptographically isolated; private keys never in app memory; supports secp256k1 + Ed25519 |
| **Secret management** | Cloud KMS + Secrets Manager | Managed | Centralised secret retrieval; rotation support; no plaintext secrets in env files |
| **EVM RPC** | Alchemy | Managed node | Reliability SLAs; Notify webhooks; supports ETH/BASE/ARB mainnet + testnet |
| **Solana RPC** | Helius | Managed node | Reliability SLAs; webhooks; devnet support |
| **AI / LLM** | Google Gemini API | `gemini-3.8-flash` (primary), `gemini-flash-lite-latest` (fallback) | Latency-cost balance; fallback for quota/capacity resilience (D8) |
| **Price feeds (off-chain)** | CoinGecko | Free tier: 100 req/min; production: paid plan | Broad token coverage; free tier sufficient for early stage |
| **Protocol safety (off-chain)** | DeFiLlama | Free tier: no SLA; production: paid plan | TVL + protocol risk data; free tier sufficient for early stage |
| **Price feeds (on-chain EVM)** | Chainlink | On-chain contracts | Decentralised; tamper-resistant; native EVM integration |
| **Price feeds (on-chain Solana)** | Pyth | On-chain program | Low-latency; first-class Solana support |
| **Bridge** | CCTP V2 | Circle — TokenMinter + MessageTransmitter | USDC-native; non-custodial; Standard + Fast modes (D3/D5) |
| **Payment protocol** | x402 / EIP-3009 | `https://x402.org/facilitator`; Base USDC; `exact` scheme | Micropayment standard; Base USDC; replay-resistant; anonymous agent support (D4) |
| **KYC** | HyperVerge | Conditional integration (D9) | Webhook-driven; tier-based limit enforcement; pending integration validation |
| **Authentication** | Google OAuth 2.0 / OIDC | Google cloud | Zero-friction onboarding; upsert by `googleSub`; JWT access + refresh tokens |
| **DEX venues** | Uniswap V3, Aerodrome, Jupiter, Raydium | Per-chain adapters | Highest liquidity per chain; Jupiter `/order` + `/build` for Solana aggregation |
| **Lending venues** | Aave V3, Moonwell, Kamino, MarginFi | Per-chain adapters | Leading lending protocols per chain; audited; large TVL |
| **EVM testnets** | Sepolia, Base Sepolia, Arbitrum Sepolia | Public testnets | Official testnets for each supported chain |
| **Solana testnet** | Solana Devnet | Public testnet | Standard Solana development environment |
| **Local EVM fork** | Anvil | Foundry toolchain | Deterministic fork; fast block production; no gas cost |

---

## 5. Key Architectural Decisions

| Decision ID | Decision | Rationale | Status |
|---|---|---|---|
| **D1** | Two wallet paths: Path A (Turnkey embedded, AUTOPILOT/MANUAL) and Path B (external wallet, READ_ONLY/MANUAL only). No ERC-4337, no session keys. | Simplifies key management; ERC-4337 adds infrastructure complexity without clear MVP benefit; session keys deferred | Decided |
| **D2** | GMX is read-only display only — no automated execution | GMX UI complexity and liquidation risk; MVP scope reduction; avoids regulatory ambiguity around perpetuals | Decided |
| **D3** | Bridge: CCTP V2, USDC-only, Standard mode default | Non-custodial; no wrapped tokens; Circle-attested; USDC native; simplest trust model for MVP | Decided |
| **D4** | x402 micropayment for Copilot: Base USDC, `exact` scheme, $0.01/query, 10 free/day, EIP-3009, anonymous agent support | Enables monetisation without subscription friction; Base USDC minimises gas cost; EIP-3009 avoids on-chain approval UX | Decided |
| **D5** | CCTP V2 Fast mode opt-in for HIGH urgency only (minFinalityThreshold=1000, 5min timeout, capped fee) | Fast mode carries a fee; most users prefer Standard (60min); Fast mode reserved for time-sensitive rebalancing | Decided |
| **D6** | NestJS monolith deployed as two horizontally scalable process types (API server + BullMQ worker pool) | Avoids microservice distribution overhead at MVP scale; BullMQ provides natural async boundary; can be split later | Decided |
| **D7** | Execution order: `Engine 6 → Execution Router → Orchestrator/Planner → Simulation → Policy Engine → CCE → PSM` | Enforces policy check after simulation but before signing; prevents policy bypass via ordering; aligns with saga model | Decided |
| **D8** | LLM: `gemini-3.8-flash` primary, `gemini-flash-lite-latest` fallback | Cost-performance balance; fallback reduces downtime risk from quota exhaustion or model unavailability | Decided |
| **D9** | KYC via HyperVerge — conditional, labeled "selected pending integration validation" | KYC adds regulatory compliance for higher-tier users; conditional status reflects integration maturity | Conditional / Open |
| **D10** | Technology stack: TypeScript/NestJS, PostgreSQL 16, Redis, BullMQ, REST/OpenAPI 3.1 | Team expertise; strong ecosystem; TypeORM + PostgreSQL mature pairing; Redis dual-purpose (cache + queue) | Decided |

---

## 6. Data Flow Summary

A complete user interaction — from natural-language Copilot query to confirmed on-chain execution — flows through all five layers as follows:

**Step 1 — Ingress (Interface Layer):**  
The user submits a natural-language query to the `/copilot/query` endpoint via HTTPS. The API Gateway terminates TLS and forwards the request to the NestJS API server. The auth guard validates the JWT access token (signed via KMS). The x402 Payment Middleware checks whether the user has remaining free-tier queries (10/day tracked in PostgreSQL). If exhausted, it inspects the `PAYMENT-SIGNATURE` header; if absent, it returns HTTP 402 with a `PAYMENT-REQUIRED` response containing the payment details. If present, it decodes the EIP-3009 authorization, verifies the network (Base, `eip155:8453`), asset (Base USDC), and timing window, then calls x402.org Facilitator `/verify` and `/settle`. On success, a `PaymentRecord` is written to PostgreSQL with a `PAYMENT_SETTLED` audit entry, and the request is forwarded.

**Step 2 — Intelligence (Intelligence Layer):**  
Engine 6 (Copilot) receives the query. It calls the Gemini LLM API (`gemini-3.8-flash`) with a structured prompt including the user's portfolio context (from Engine 1) and market data (from Engines 2–3). The LLM returns a structured action intent (e.g., "swap 500 USDC on Base for ETH via Uniswap V3 with 0.5% slippage"). Engine 4 (Risk Assessor) computes a `riskScore` for the proposed action against current market conditions. Engine 5 (Strategy Optimiser) may augment with alternative suggestions. The structured action plan is passed to the Execution Layer.

**Step 3 — Execution Planning (Execution Layer):**  
The Orchestrator/Planner decomposes the action into ordered saga steps, assigns each a state starting at `CREATED`, and transitions to `ROUTING`. The Execution Router determines the user's `ExecutionMode` (AUTOPILOT or MANUAL based on wallet path) and target chain. The Planner transitions to `BUILDING`, calling the relevant Venue Adapter to encode the transaction calldata. The saga moves to `SIMULATING` — the Protocol Safety Module runs a simulation call against the RPC node to validate expected outputs and gas costs. After simulation, the saga transitions to `POLICY_CHECK`.

**Step 4 — Policy Validation (Execution Layer):**  
The Policy Engine runs all 9 checks sequentially. If any check fails, the saga terminates at `BLOCKED` with a structured error and an `EXECUTION` audit record. If all checks pass, the saga transitions to `AWAITING_SIGNATURE` (MANUAL) or `SIGNING` (AUTOPILOT). For AUTOPILOT, the Turnkey Integration service calls the Turnkey TEE API with the encoded transaction payload; the TEE signs and returns the signed transaction bytes — private keys never transit the application. For MANUAL, the signed transaction is delivered to the client via the API response.

**Step 5 — Broadcast & Confirmation (Protocol Layer):**  
The signed transaction is broadcast to the target chain via Alchemy (EVM) or Helius (Solana). The saga transitions to `BROADCASTING` → `AWAITING_CONFIRMATION`. On-chain confirmation is detected via Alchemy Notify webhook or Helius webhook (delivered to the Webhook Receiver in the Interface Layer). For CCTP cross-chain operations, the saga enters `AWAITING_ATTESTATION` while the CCE polls Circle Iris API; on attestation, it triggers `receiveMessage` on the destination chain, transitioning to `MINTING` → `CONFIRMING` → `COMPLETED`.

**Step 6 — State & Audit (Data Layer):**  
Throughout execution, every state transition writes to PostgreSQL (saga state table). The Audit Layer appends structured `EXECUTION` audit records for every significant event. Portfolio State Manager reconciles the confirmed on-chain changes with the user's DB-persisted portfolio, publishing a `PortfolioScanTriggered` event (outbox pattern) to trigger an async portfolio refresh via BullMQ.

---

## 7. Security Architecture

### 7.1 Private Key Security — Turnkey TEE
CrestFlow holds **zero private keys** in application memory, environment variables, or persistent storage. All transaction signing is delegated to the Turnkey TEE via HTTPS API. For Path A (embedded wallet) in AUTOPILOT mode, the application holds a delegated signing permission scoped to the user's Turnkey sub-organisation (`AddressScope: AUTOPILOT`). For MANUAL mode, the user's in-session approval is required. The TEE boundary is cryptographically attested — no code outside the TEE can access raw key material.

### 7.2 Authentication — JWT + Refresh Token Hashing
User sessions use short-lived JWT access tokens (signed via KMS). Refresh tokens are **hashed before storage** in PostgreSQL — the plaintext token is never persisted. On refresh, the plaintext token submitted by the client is hashed and compared against the stored hash. Access tokens carry standard JWT claims (`sub`, `iat`, `exp`) plus CrestFlow-specific claims (`executionMode`, `kycTier`).

Google OAuth onboarding uses the standard authorization code flow. The backend exchanges the code for tokens, validates the `id_token`, and upserts the user record by `googleSub` — never by email, to prevent account takeover if a user changes their Google email.

### 7.3 Audit Trail
All sensitive operations emit structured audit records to the append-only `audit_log` PostgreSQL table, categorised by `AuditCategory`: AUTH, KYC, EXECUTION, PREFERENCE, PAYMENT, SYSTEM. Audit records are immutable once written (no UPDATE or DELETE operations on audit tables). The audit trail is used for compliance review, debugging saga failures, and replay protection verification.

### 7.4 x402 Replay Protection
The x402 Payment Middleware implements multi-layer replay protection:
1. **`validAfter` / `validBefore`:** EIP-3009 timing window enforced server-side — requests outside the window are rejected.
2. **EIP-3009 nonce:** The on-chain nonce embedded in the authorization prevents the same signed payload from being used twice on-chain.
3. **`paymentPayloadHash`:** A hash of the full payment payload is stored in the `PaymentRecord` table. Duplicate hashes are rejected before calling the facilitator.

The x402 middleware also handles the 5xx path: if a downstream error occurs after payment settlement, the `PaymentRecord.status` is set to `FAILED` and an audit record is written — the user is not charged for failed requests.

### 7.5 Input Validation & Rate Limiting
All REST endpoints are protected by NestJS `ValidationPipe` (class-validator + class-transformer). The API Gateway enforces rate limiting per IP and per authenticated user before requests reach the application. Webhook callbacks from Alchemy, Helius, and HyperVerge are validated via HMAC signature verification before processing.

### 7.6 KYC Tier Enforcement
The Policy Engine enforces KYC tier daily spending limits as check #2 (rolling 24h audit-based spend aggregation). KYC tier caps: TIER_1 $1k/day, TIER_2 $10k/day, TIER_3 unlimited. All execution is blocked for users with `kycStatus` other than `APPROVED`.

---

## 8. Scalability & Reliability Notes

### 8.1 Horizontal Scaling
The NestJS API Server is stateless (all session state in PostgreSQL/Redis; no in-process state) and horizontally scalable. Additional instances are added behind the API Gateway/Load Balancer without coordination. The BullMQ Worker Pool is independently scalable — additional worker instances compete for jobs from the Redis-backed queue; BullMQ's atomic job claiming prevents duplicate processing.

### 8.2 BullMQ Async Decoupling
Long-running operations (saga execution steps, portfolio scans, on-chain confirmation polling) are dispatched as BullMQ jobs rather than blocking the API request. This decouples API response latency from blockchain confirmation latency (which can range from seconds to minutes for CCTP Standard mode). BullMQ provides configurable retry semantics, dead-letter queues, and job prioritisation.

### 8.3 Redis Caching Strategy
Price feeds (CoinGecko, Chainlink, Pyth) are cached in Redis with TTL-based expiration to reduce redundant external API calls and to stay within CoinGecko free-tier rate limits (100 req/min). Engine outputs (portfolio snapshots, risk scores) are cached with staleness thresholds (Risk Assessor freshness: ≤ 14,400 seconds for valid policy check). Cache invalidation is event-driven via portfolio scan triggers.

### 8.4 PostgreSQL Replication
Write operations (saga state, audit records, portfolio updates) target the primary PostgreSQL 16 instance. Read-heavy operations (portfolio display, audit log queries, price history) target the read replica, reducing primary load. Streaming replication lag is monitored — the Policy Engine reads risk scores from the primary to avoid stale replica data for safety-critical checks.

### 8.5 Provider Health Checks & Fallback
- **Gemini LLM:** Primary `gemini-3.8-flash` with automatic fallback to `gemini-flash-lite-latest` on quota exhaustion or error (D8).
- **EVM RPC:** Alchemy is the sole managed provider; circuit-breaker logic in the EVM Chain Adapter prevents cascade failures.
- **Solana RPC:** Helius sole managed provider; same circuit-breaker pattern.
- **CoinGecko / DeFiLlama:** Free-tier rate limits managed via Redis-backed request throttling and cache-first strategy. Upgrade triggers to paid plans are defined (see F11-05).
- **Circle Iris API:** CCTP attestation polling uses exponential backoff with configurable timeout (60min Standard, 5min Fast).

### 8.6 Saga Compensation
The Orchestrator/Planner implements saga compensation flows for failed multi-step operations. If a step fails after a prior step succeeded (e.g., bridge burn succeeded but destination mint failed), the saga transitions to `COMPENSATING`, attempts reversal or manual intervention, and settles at `COMPENSATED` or `FAILED`. All compensation events are audited. The `PAUSED` state supports manual operator intervention for stuck sagas.

---

## 9. Lucidchart Shape Guide

| Element Type | UML / Diagram Shape | Lucidchart Shape Name | Colour / Style |
|---|---|---|---|
| Layer container | Horizontal swim lane or bordered rectangle | Group / Container | Each layer a distinct pastel colour |
| Application component | Rectangle with component name | General → Rectangle | White fill, dark border |
| External service | Rectangle with `«external»` label | General → Rectangle | Light orange fill |
| Database | Cylinder | Entity Relationship → Entity or General → Database | Light blue fill |
| Cache / Queue | Cylinder or queue shape | General → Database | Light green fill |
| Blockchain network | Hexagon or labelled rectangle | General → Hexagon | Light grey fill |
| TEE (Turnkey) | Rectangle with lock icon label | General → Rectangle | Light purple fill, dashed border |
| Synchronous connection | Solid arrow | Connector → Line with Arrow | Label: protocol |
| Async / Queue connection | Dashed arrow | Connector → Dashed Line with Arrow | Label: "BullMQ" or "webhook" |
| In-process call | Thin solid arrow (no label or brief label) | Connector → Line | Used within same layer |
| Trust boundary | Rounded dashed border | Group / Container (dashed) | Colour-coded by boundary type |

---

## 10. Layout Recommendations

### High-Level Architecture Diagram Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — INTERFACE                                                │
│  [API Gateway]  [REST Controllers]  [x402 Middleware]  [Webhooks]  │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│  LAYER 2 — INTELLIGENCE                                             │
│  [E1 Portfolio] [E2 Market] [E3 Opportunity] [E4 Risk] [E5 Strat]  │
│  [E6 Copilot (LLM)]                   [Financial Knowledge Layer]  │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│  LAYER 3 — EXECUTION                                                │
│  [Orchestrator/Planner]  [Execution Router]  [Policy Engine]  [CCE]│
└───────────────────────────┬─────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│  LAYER 4 — PROTOCOL                                                 │
│  [EVM Adapters]  [Solana Adapter]  [Venue Adapters DEX/Lending]    │
│  [CCTP Bridge Adapter]  [PSM]  [Turnkey Integration]               │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│  LAYER 5 — DATA                                                     │
│  [PostgreSQL Primary]  [PostgreSQL Replica]  [Redis]  [BullMQ Pool]│
│  [Portfolio State Manager]  [Audit Layer]                           │
└─────────────────────────────────────────────────────────────────────┘

EXTERNAL SERVICES (right-side vertical band):
  ↕ Turnkey TEE | Alchemy | Helius | Gemini LLM | Google OAuth
  ↕ Circle Iris | x402 Facilitator | HyperVerge | CoinGecko | DeFiLlama

BLOCKCHAIN NETWORKS (bottom band):
  ETH / BASE / ARB (via Alchemy) | SOL (via Helius)
  Chainlink | Pyth | CCTP V2 | DEX / Lending Contracts
```

### Grouping Rules
1. **Layer swimlanes:** horizontal bands, top-to-bottom, Interface → Data
2. **External services:** right-side vertical band; arrow connections from relevant layer components
3. **Blockchain networks:** bottom band; grouped EVM (left) and Solana (right)
4. **Colour coding:** align with deployment diagram trust boundary colours for visual consistency across documents
5. **Avoid crossing arrows:** route external service connections from the layer closest to that integration (Interface for auth/payment/KYC; Protocol for RPC/signing; Intelligence for LLM/price feeds)

---

## 11. Flags & Open Questions

| ID | Flag / Question | Owner | Status |
|---|---|---|---|
| F12-01 | D9 (HyperVerge KYC) labeled "conditional pending integration validation" — final go/no-go decision needed before mainnet | Product / BE | Open |
| F12-02 | CoinGecko and DeFiLlama free-tier upgrade criteria not formally specified — define request volume thresholds | Product | Open |
| F12-03 | Redis cluster mode vs. Sentinel selection pending — affects HA topology and failover behaviour | Infra | Open |
| F12-04 | Saga `PAUSED` state: operator intervention workflow not fully specified — tooling needed | BE / Ops | Open |
| F12-05 | LLM prompt versioning strategy not defined — changes to Copilot prompts could alter Engine 6 output format | BE | Open |
| F12-06 | `@crestflow/pool-type` NPM package versioning and publication strategy (private registry vs. monorepo) | BE | Open |
| F12-07 | MANUAL mode signing UX — how the signed transaction is returned to and submitted by the external wallet (Path B) not fully specified | Product / FE | Open |
| F12-08 | GMX read-only display scope: which GMX data points are shown and how they are fetched (direct RPC or API) not specified | BE | Open |
| F12-09 | Outbox pattern for `PortfolioScanTriggered` — transactional outbox implementation (polling vs. CDC) not decided | BE | Open |
| F12-10 | Mainnet release gate criteria — specific checklist for triggering mainnet deployment not documented | Product / Infra | Open |
