# 13 — BE Report: DFD Level 0 (Context Diagram)

**Pack:** CrestFlow UML & Architecture Documentation Pack  
**Version:** 1.0 | **Source docs:** `system_architecture.md` v2.0, `implementation_spec.md` v1.0, `CANONICAL_DECISIONS.md` v1.2 (D1–D10)  
**Primary deliverable:** Lucidchart-ready DFD specification. DFD notation, not UML.

---

## 1. Purpose & Scope

Specifies the **Context Diagram (DFD Level 0)** for the CrestFlow DeFi Portfolio Management System. Shows CrestFlow as a single central process (black box) surrounded by all external entities with labeled data flows in and out.

**Purpose:** Establish system boundary, identify all external actors and systems, and characterize the nature of data exchanged.

**Audience:** BE project report — academic/professional audience. Clear, unambiguous, suitable for Chapter 3 (System Architecture) of a final-year BE project.

**DFD notation (not UML):**
- **External entities:** Rectangles (sharp corners)
- **Central process:** Rounded rectangle or circle labeled with process number and name
- **Data stores:** Open-ended rectangles (like a box with two open sides) — not used at Level 0
- **Data flows:** Directed arrows with descriptive labels

---

## 2. DFD Notation Guide

| DFD Element | Shape | Lucidchart Shape Name | Label Convention |
|---|---|---|---|
| External Entity | Rectangle (sharp corners) | "Rectangle" | Noun phrase: entity name |
| Central Process | Rounded rectangle / Circle | "Rounded Rectangle" or "Circle" | "0 / CrestFlow System" |
| Data Flow | Directed arrow | "Arrow" connector | Verb-noun phrase: e.g., "Portfolio Analysis Results" |
| Data Store | Open-ended rectangle | "Data Store (DFD)" | Not used at Level 0 |
| System Boundary | Dashed rectangle surrounding process | "Dashed Rectangle" group | Optional; demarcates system |

**Labeling convention:**
- Data flows: use gerunds or noun phrases describing the data content, NOT the protocol (e.g., "KYC Application Data" not "HTTPS POST")
- External entity names: use common names, not technical identifiers

---

## 3. Central Process

| Attribute | Value |
|---|---|
| Process Number | 0 |
| Process Name | CrestFlow DeFi Portfolio Management System |
| Description | Collects user identity and portfolio data; applies AI-driven analysis; evaluates policy; executes or routes DeFi transactions; manages cross-chain USDC transfers; tracks audit and payment events |
| Lucidchart Shape | Rounded rectangle, centered on diagram, larger than external entities |
| Label Style | Bold, two-line: "0" (top), "CrestFlow DeFi Portfolio Management System" (bottom) |

---

## 4. External Entity Inventory

| # | Entity Name | Direction | Description | Lucidchart Label |
|---|---|---|---|---|
| E1 | Path A User | Bidirectional | Authenticated user with Turnkey embedded wallet. Uses AUTOPILOT and MANUAL modes. Provides goal inputs; receives portfolio analysis and execution results. | "Path A User\n(Embedded Wallet)" |
| E2 | Path B User | Bidirectional | Authenticated user with external wallet. Uses MANUAL and READ_ONLY modes. Provides wallet address + signatures; receives portfolio views. | "Path B User\n(External Wallet)" |
| E3 | Google OAuth | Bidirectional | Provides OAuth 2.0 identity authentication (token exchange). | "Google OAuth" |
| E4 | HyperVerge KYC | Bidirectional | Provides KYC verification sessions; delivers webhook callbacks with KYC decision. (Conditional — D9) | "HyperVerge KYC\n(Conditional)" |
| E5 | Turnkey TEE | Bidirectional | Secure enclave for key management and transaction signing. Provides wallet addresses and signed transactions. | "Turnkey TEE\n(Key Management)" |
| E6 | Alchemy | Bidirectional | EVM RPC provider (Ethereum, Base, Arbitrum); delivers blockchain webhooks for transaction confirmation. | "Alchemy\n(EVM RPC + Webhooks)" |
| E7 | Helius | Bidirectional | Solana RPC provider; delivers Solana transaction webhooks. | "Helius\n(Solana RPC + Webhooks)" |
| E8 | Circle (CCTP) | Bidirectional | CCTP V2 attestation service (Iris API); provides cross-chain USDC transfer attestations. | "Circle CCTP\n(Iris Attestation API)" |
| E9 | x402.org Facilitator | Bidirectional | x402 HTTP payment protocol facilitator; verifies and settles EIP-3009 USDC micropayments. | "x402.org\n(Payment Facilitator)" |
| E10 | HyperVerge KYC | — | (See E4) | — |
| E11 | Gemini LLM API | Bidirectional | AI inference provider for portfolio analysis, natural language responses, and strategy explanations. | "Gemini LLM API\n(Google)" |
| E12 | CoinGecko | Input to CrestFlow | Market price feed provider (HTTPS API). | "CoinGecko\n(Price Data)" |
| E13 | DeFiLlama | Input to CrestFlow | DeFi protocol TVL, yield, and safety data provider. | "DeFiLlama\n(Protocol Data)" |
| E14 | Chainlink | Input to CrestFlow | On-chain EVM price oracle (read-only smart contract calls). | "Chainlink\n(On-chain Oracle)" |
| E15 | Pyth Network | Input to CrestFlow | On-chain Solana and EVM price oracle (read-only). | "Pyth Network\n(On-chain Oracle)" |
| E16 | Blockchain Networks | Bidirectional | Ethereum, Base, Arbitrum, Solana — target execution environments. CrestFlow submits transactions; receives confirmations and state. | "Blockchain Networks\n(ETH/BASE/ARB/SOL)" |
| E17 | Admin | Bidirectional | Internal system administrator. Can suspend users, pause autopilot, create policy overrides, view system health. | "Admin" |

---

## 5. Data Flow Inventory

| Flow ID | From | To | Data Flow Label | Data Content Description |
|---|---|---|---|---|
| DF-01 | Path A/B User | CrestFlow | Authentication Credentials | Google OAuth authorization code, JWT access/refresh tokens on subsequent calls |
| DF-02 | Google OAuth | CrestFlow | OAuth Identity Token | Google `id_token` with `sub` (googleSub), email, display name |
| DF-03 | CrestFlow | Google OAuth | OAuth Token Request | Authorization code + client credentials for token exchange |
| DF-04 | Path A/B User | CrestFlow | KYC Application Data | Personal identity documents, selfie, government ID (via HyperVerge SDK embedded in browser) |
| DF-05 | HyperVerge KYC | CrestFlow | KYC Decision Webhook | Verification result: APPROVED / REJECTED / RESUBMISSION_REQUESTED; transaction ID |
| DF-06 | CrestFlow | HyperVerge KYC | KYC Session Request | Session creation request with workflow ID and redirect URL |
| DF-07 | HyperVerge KYC | CrestFlow | KYC SDK Token | Session ID and SDK token for browser-embedded KYC flow |
| DF-08 | CrestFlow | Turnkey TEE | Wallet Provisioning Request | Sub-org creation + wallet key derivation requests (secp256k1 + Ed25519) |
| DF-09 | Turnkey TEE | CrestFlow | Wallet Addresses | Derived EVM and Solana wallet addresses; sub-org ID |
| DF-10 | CrestFlow | Turnkey TEE | Transaction Signing Request | Unsigned transaction hex; signing key ID |
| DF-11 | Turnkey TEE | CrestFlow | Signed Transaction | Signed transaction hex ready for broadcast |
| DF-12 | Path A User | CrestFlow | Goal and Preference Settings | Investment goals, risk tolerance, autopilot preferences, allowed action types |
| DF-13 | Path B User | CrestFlow | Wallet Verification Signature | Signed challenge proving ownership of external wallet address |
| DF-14 | CrestFlow | Path A/B User | Portfolio Analysis Dashboard | Portfolio value, positions, risk scores, yield opportunities, strategy recommendations |
| DF-15 | CrestFlow | Path A/B User | Execution Status Updates | Transaction hash, status (PENDING/CONFIRMING/COMPLETED/FAILED), gas cost |
| DF-16 | Path A User | CrestFlow | Manual Execution Approval | Plan approval or rejection; signed transactions (MANUAL mode) |
| DF-17 | CrestFlow | Blockchain Networks | Signed Transactions | Encoded and signed transaction data for EVM or Solana |
| DF-18 | Blockchain Networks | CrestFlow | Transaction Receipts | Block number, gas used, status, event logs confirming on-chain execution |
| DF-19 | Alchemy | CrestFlow | EVM Webhooks | Transaction confirmation notifications (HMAC-verified) |
| DF-20 | Helius | CrestFlow | Solana Webhooks | Transaction confirmation notifications |
| DF-21 | CrestFlow | Alchemy | EVM RPC Calls | eth_call, eth_sendRawTransaction, eth_getTransactionReceipt, price/pool queries |
| DF-22 | CrestFlow | Helius | Solana RPC Calls | simulateTransaction, sendTransaction, getTransaction, slot data |
| DF-23 | Circle CCTP | CrestFlow | CCTP Attestation | Message attestation (hex) + message bytes authorizing USDC mint on destination chain |
| DF-24 | CrestFlow | Circle CCTP | Attestation Query | Burn transaction hash used to query Iris API for attestation status |
| DF-25 | Path A/B User | CrestFlow | Copilot Query (Paid) | Natural language financial query with EIP-3009 PAYMENT-SIGNATURE header |
| DF-26 | CrestFlow | x402.org | Payment Verification Request | EIP-3009 authorization object + payment requirements |
| DF-27 | x402.org | CrestFlow | Payment Settlement Confirmation | Settlement transaction hash, confirmed payment |
| DF-28 | CrestFlow | Path A/B User | Copilot Response | Natural language answer; PAYMENT-RESPONSE header if paid query |
| DF-29 | CoinGecko | CrestFlow | Market Price Data | Asset prices in USD; historical OHLCV data |
| DF-30 | DeFiLlama | CrestFlow | Protocol TVL / Yield Data | TVL per protocol, APY data, protocol safety indicators |
| DF-31 | Chainlink | CrestFlow | On-chain Price Feed | Real-time asset price from Chainlink price aggregator contracts |
| DF-32 | Pyth Network | CrestFlow | On-chain Solana Price Feed | Real-time Solana and cross-chain asset prices |
| DF-33 | CrestFlow | Gemini LLM API | AI Analysis Prompt | Portfolio context, user query, strategy request |
| DF-34 | Gemini LLM API | CrestFlow | AI Analysis Response | Portfolio insights, strategy recommendations, natural language explanations |
| DF-35 | Admin | CrestFlow | Administrative Commands | User suspension, autopilot pause, policy override creation |
| DF-36 | CrestFlow | Admin | System Health Reports | Service status, error rates, provider health, audit log summaries |

---

## 6. Layout Recommendations

**Central positioning:**
- Place the CrestFlow central process (rounded rectangle) in the center of the diagram.
- Size: approximately 2× larger than external entity rectangles to visually anchor the diagram.

**External entity placement (radial / clock-face layout):**

| Position | Entity | Rationale |
|---|---|---|
| Top-left | Path A User | Primary user; most data flows |
| Top-center | Path B User | Secondary user |
| Top-right | Admin | Administrative actor |
| Right-top | Google OAuth | Auth flow: input from users → CrestFlow → Google |
| Right-middle | Turnkey TEE | Critical signing dependency |
| Right-bottom | HyperVerge KYC | KYC dependency |
| Bottom-right | Gemini LLM API | AI backend |
| Bottom-center | Blockchain Networks | Execution target |
| Bottom-left | Circle CCTP | Bridge attestation |
| Left-bottom | x402.org Facilitator | Payment |
| Left-middle | Alchemy + Helius | RPC providers (can be grouped) |
| Left-top | CoinGecko + DeFiLlama + Chainlink + Pyth | Data providers (can be grouped) |

**Data flow arrow tips:**
- Use directional arrows (open arrowhead).
- Keep labels short (2–5 words): "OAuth Identity Token", "Signed Transaction", "Portfolio Dashboard".
- Group bidirectional flows as two parallel arrows or a single arrow with two labels.
- Color code by flow category: Auth (blue), Execution (green), Data (yellow), Payments (purple).

---

## 7. Lucidchart Element Table

| Element Type | Lucidchart Shape | Label | Style / Color |
|---|---|---|---|
| Central Process | Rounded Rectangle | "0 / CrestFlow DeFi Portfolio Management System" | White fill, bold border, center |
| Path A User | Rectangle | "Path A User (Embedded Wallet)" | Light blue fill |
| Path B User | Rectangle | "Path B User (External Wallet)" | Light blue fill |
| Admin | Rectangle | "Admin" | Light grey fill |
| Google OAuth | Rectangle | "Google OAuth" | Light grey fill |
| Turnkey TEE | Rectangle | "Turnkey TEE (Key Management)" | Light orange fill |
| HyperVerge KYC | Rectangle | "HyperVerge KYC (Conditional)" | Light yellow fill |
| Alchemy | Rectangle | "Alchemy (EVM RPC + Webhooks)" | Light grey fill |
| Helius | Rectangle | "Helius (Solana RPC + Webhooks)" | Light grey fill |
| Circle CCTP | Rectangle | "Circle CCTP (Iris API)" | Light grey fill |
| x402.org | Rectangle | "x402.org Facilitator" | Light purple fill |
| Gemini LLM | Rectangle | "Gemini LLM API (Google)" | Light green fill |
| CoinGecko | Rectangle | "CoinGecko (Price Data)" | Light grey fill |
| DeFiLlama | Rectangle | "DeFiLlama (Protocol Data)" | Light grey fill |
| Chainlink | Rectangle | "Chainlink (On-chain Oracle)" | Light grey fill |
| Pyth Network | Rectangle | "Pyth Network (On-chain Oracle)" | Light grey fill |
| Blockchain Networks | Rectangle | "Blockchain Networks (ETH/BASE/ARB/SOL)" | Light blue fill, bold border |
| Data Flow arrows | Arrow connector | Per DF-XX label above | 1.5px, directional arrowhead |

---

## 8. Flags & Open Questions

| Flag ID | Type | Description | Impact |
|---|---|---|---|
| F-13-01 | **Conditional** | HyperVerge KYC (E4) is conditional (D9). Label as "(Conditional)" to reflect that this entity may change to Persona or Sumsub. | Label accuracy |
| F-13-02 | **Grouping** | CoinGecko, DeFiLlama, Chainlink, and Pyth can be grouped as "Market Data Providers" in Lucidchart if the diagram becomes crowded. Each is a separate entity logically. | Aesthetic choice |
| F-13-03 | **Out of MVP** | Fiat on-ramp is excluded (T16). No external entity for a fiat payment provider is included. | Confirmed omission |
| F-13-04 | **Phase 2** | LI.FI / Socket bridge providers are Phase 2. Only Circle CCTP appears as an external entity. | Confirmed omission |
| F-13-05 | **Assumption** | "Blockchain Networks" is treated as a single external entity at Level 0. Level 1 (DFD) will decompose by chain. | Appropriate for Level 0 abstraction |
