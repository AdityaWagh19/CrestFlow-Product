# 15 — BE Report: Use Case Diagram

**Pack:** CrestFlow UML & Architecture Documentation Pack  
**Version:** 1.0 | **Source docs:** `system_architecture.md` v2.0, `user_onboarding.md` v2.0, `implementation_spec.md` v1.0, `CANONICAL_DECISIONS.md` v1.2 (D1, D2, D4)  
**Primary deliverable:** Lucidchart-ready specification. PlantUML is supplementary only.

---

## 1. Purpose & Scope

Specifies the **Use Case Diagram** for the CrestFlow DeFi Portfolio Management System. Shows all actors, use cases within the system boundary, and include/extend relationships.

**Scope:** MVP use cases only. Phase 2 items (GMX automation, fiat on-ramp, ERC-4337) are excluded.

---

## 2. Actor Inventory

| # | Actor | Type | Description | Stereotype |
|---|---|---|---|---|
| A1 | Path A User | Primary (Human) | Authenticated user with Turnkey embedded wallet. Full access: AUTOPILOT + MANUAL + READ_ONLY views. | «user» |
| A2 | Path B User | Primary (Human) | Authenticated user with external wallet. READ_ONLY + MANUAL only (D1). | «user» |
| A3 | Admin | Secondary (Human) | Internal system administrator. Manage users, overrides, system health. | «admin» |
| A4 | System (Automated) | System | Background processes: scheduler, webhooks, BullMQ workers. No human interaction. | «system» |
| A5 | Google OAuth | External System | Provides OAuth 2.0 authentication tokens. Participates in sign-in use cases. | «external» |
| A6 | HyperVerge KYC | External System | Delivers KYC decision webhooks. Participates in KYC use cases. (Conditional — D9) | «external» |
| A7 | Turnkey TEE | External System | Signs transactions and provisions wallets for Path A. | «external» |
| A8 | x402.org Facilitator | External System | Verifies and settles micropayments for Copilot paid queries. | «external» |
| A9 | Blockchain Networks | External System | Receives transactions; confirms execution. | «external» |

---

## 3. Use Case Inventory

| UC ID | Domain | Use Case Name | Primary Actor | Brief Description |
|---|---|---|---|---|
| UC-01 | Authentication | Sign In with Google | A1, A2 | Authenticate using Google OAuth 2.0; receive JWT tokens |
| UC-02 | Authentication | Refresh Session | A1, A2 | Exchange refresh token for new access token |
| UC-03 | Authentication | Sign Out | A1, A2 | Invalidate current session and refresh token |
| UC-04 | Wallet | Add Embedded Wallet (Path A) | A1 | Provision Turnkey sub-org, derive EVM + Solana keys, register AddressScope (AUTOPILOT) |
| UC-05 | Wallet | Add External Wallet (Path B) | A2 | Challenge-response wallet connection, register AddressScope (MANUAL/READ_ONLY) |
| UC-06 | Wallet | Verify Wallet Ownership | A2 | Sign a challenge string to prove ownership of external wallet address |
| UC-07 | Wallet | Remove Wallet | A1, A2 | Deactivate wallet; revoke delegation if AUTOPILOT |
| UC-08 | Wallet | Pause Autopilot | A1 | Temporarily suspend autopilot execution without removing wallet |
| UC-09 | Wallet | Resume Autopilot | A1 | Re-enable previously paused autopilot |
| UC-10 | KYC | Submit KYC Application | A1, A2 | Create HyperVerge KYC session; complete SDK flow |
| UC-11 | KYC | View KYC Status | A1, A2 | Check current KYC status (PENDING/APPROVED/REJECTED/etc.) |
| UC-12 | KYC | Resubmit KYC Documents | A1, A2 | Provide updated documents after RESUBMISSION_REQUESTED |
| UC-13 | Portfolio | View Portfolio Overview | A1, A2 | View current portfolio value, positions, allocations |
| UC-14 | Portfolio | Trigger Manual Portfolio Scan | A1, A2 | Manually initiate a fresh portfolio data scan |
| UC-15 | Portfolio | View Position Detail | A1, A2 | View detailed data for a single position (chain, protocol, APY, etc.) |
| UC-16 | Risk | View Risk Score | A1, A2 | View current portfolio risk score (0–100) and breakdown |
| UC-17 | Risk | View Risk Alerts | A1, A2 | View active risk alerts (HHI, protocol safety, etc.) |
| UC-18 | Risk | Acknowledge Alert | A1, A2 | Mark a risk alert as acknowledged |
| UC-19 | Yield | View Yield Opportunities | A1, A2 | Browse ranked yield opportunities from Engine 3 |
| UC-20 | Yield | Compare Yield Opportunities | A1, A2 | Side-by-side comparison of TOPSIS-ranked yield options |
| UC-21 | Strategy | Generate Strategy Plan | A4 | System generates a rebalancing strategy plan (Engines 4+6) |
| UC-22 | Strategy | Approve Strategy Plan | A1, A2 | User manually approves a pending strategy plan |
| UC-23 | Strategy | Reject Strategy Plan | A1, A2 | User rejects a pending strategy plan |
| UC-24 | Execution | Execute Plan — AUTOPILOT | A4 | System auto-executes approved plan via CCE + Turnkey TEE |
| UC-25 | Execution | Sign & Execute Plan — MANUAL | A1, A2 | User reviews unsigned tx, signs externally, submits signed tx |
| UC-26 | Execution | View Execution Status | A1, A2 | View current saga state, tx hash, gas used |
| UC-27 | Execution | Cancel Execution | A1, A2 | Cancel a queued execution intent (pre-broadcast only) |
| UC-28 | Execution | Retry Failed Execution | A1, A2 | Re-trigger a FAILED execution intent |
| UC-29 | Copilot | Query Copilot — Free Tier | A1, A2 | Natural language query within 10 free queries/day |
| UC-30 | Copilot | Query Copilot — Paid (x402) | A1, A2 | Paid natural language query using x402/EIP-3009 (\$0.01) |
| UC-31 | Copilot | Change Goal Profile | A1, A2 | Update investment goals, risk tolerance, allowed action types |
| UC-32 | Admin | Suspend User Account | A3 | Deactivate user; revoke autopilot; halt in-flight executions |
| UC-33 | Admin | Pause Autopilot (Admin) | A3 | Administratively pause a user's autopilot |
| UC-34 | Admin | Create Policy Override | A3 | Temporarily modify a policy parameter for a user |
| UC-35 | Admin | View System Health Dashboard | A3 | View provider health, error rates, audit log summaries |
| UC-36 | System Background | Process KYC Webhook | A4, A6 | Receive and process HyperVerge KYC decision webhook |
| UC-37 | System Background | Process Transaction Webhook | A4, A9 | Receive and process Alchemy/Helius confirmation webhooks |
| UC-38 | System Background | Run Scheduled Portfolio Scan | A4 | Periodic Engine 1 portfolio data refresh |
| UC-39 | System Background | Run Risk Rescore | A4 | Periodic Engine 2 risk score computation |
| UC-40 | System Background | Refresh Yield Data | A4 | Periodic Engine 3 yield opportunity refresh |

---

## 4. Actor-to-Use-Case Matrix

| Use Case | A1 Path A | A2 Path B | A3 Admin | A4 System | A5 Google | A6 HyperVerge | A7 Turnkey | A8 x402.org | A9 Blockchain |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| UC-01 Sign In | ✓ | ✓ | | | ✓ | | | | |
| UC-02 Refresh | ✓ | ✓ | | | | | | | |
| UC-03 Sign Out | ✓ | ✓ | | | | | | | |
| UC-04 Add Embedded Wallet | ✓ | | | | | | ✓ | | |
| UC-05 Add External Wallet | | ✓ | | | | | | | |
| UC-06 Verify Wallet | | ✓ | | | | | | | |
| UC-07 Remove Wallet | ✓ | ✓ | | | | | | | |
| UC-08 Pause Autopilot | ✓ | | | | | | | | |
| UC-09 Resume Autopilot | ✓ | | | | | | | | |
| UC-10 Submit KYC | ✓ | ✓ | | | | ✓ | | | |
| UC-11 View KYC Status | ✓ | ✓ | | | | | | | |
| UC-12 Resubmit KYC | ✓ | ✓ | | | | ✓ | | | |
| UC-13 View Portfolio | ✓ | ✓ | | | | | | | |
| UC-14 Manual Scan | ✓ | ✓ | | | | | | | |
| UC-15 View Position | ✓ | ✓ | | | | | | | |
| UC-16 View Risk Score | ✓ | ✓ | | | | | | | |
| UC-17 View Alerts | ✓ | ✓ | | | | | | | |
| UC-18 Acknowledge Alert | ✓ | ✓ | | | | | | | |
| UC-19 View Yield Opps | ✓ | ✓ | | | | | | | |
| UC-20 Compare Yield | ✓ | ✓ | | | | | | | |
| UC-21 Generate Plan | | | | ✓ | | | | | |
| UC-22 Approve Plan | ✓ | ✓ | | | | | | | |
| UC-23 Reject Plan | ✓ | ✓ | | | | | | | |
| UC-24 Execute AUTOPILOT | ✓ | | | ✓ | | | ✓ | | ✓ |
| UC-25 Sign & Execute MANUAL | ✓ | ✓ | | ✓ | | | | | ✓ |
| UC-26 View Exec Status | ✓ | ✓ | | | | | | | |
| UC-27 Cancel Execution | ✓ | ✓ | | | | | | | |
| UC-28 Retry Execution | ✓ | ✓ | | | | | | | |
| UC-29 Query Copilot Free | ✓ | ✓ | | | | | | | |
| UC-30 Query Copilot Paid | ✓ | ✓ | | | | | | ✓ | |
| UC-31 Change Goal Profile | ✓ | ✓ | | | | | | | |
| UC-32 Suspend User | | | ✓ | | | | | | |
| UC-33 Pause Autopilot (Admin) | | | ✓ | | | | | | |
| UC-34 Policy Override | | | ✓ | | | | | | |
| UC-35 View Health | | | ✓ | | | | | | |
| UC-36 Process KYC Webhook | | | | ✓ | | ✓ | | | |
| UC-37 Process Tx Webhook | | | | ✓ | | | | | ✓ |
| UC-38 Portfolio Scan | | | | ✓ | | | | | |
| UC-39 Risk Rescore | | | | ✓ | | | | | |
| UC-40 Yield Refresh | | | | ✓ | | | | | |

---

## 5. Include & Extend Relationships

| Relationship # | Type | Base Use Case | Relationship | Related Use Case | Condition / Note |
|---|---|---|---|---|---|
| R-01 | «include» | UC-01 Sign In with Google | includes | UC-05-EXT: Google Token Exchange | Always — OAuth code must be exchanged |
| R-02 | «include» | UC-04 Add Embedded Wallet | includes | UC-TK: Turnkey Sub-Org Provisioning | Always — Turnkey is the only wallet provider |
| R-03 | «include» | UC-05 Add External Wallet | includes | UC-06 Verify Wallet Ownership | Always — challenge-response required to link wallet |
| R-04 | «include» | UC-25 Sign & Execute (MANUAL) | includes | UC-06 Verify Wallet Ownership | Ownership must be confirmed before MANUAL signing |
| R-05 | «include» | UC-24 Execute AUTOPILOT | includes | UC-PE: Policy Evaluation | Every execution includes policy gate (all 9 checks) |
| R-06 | «include» | UC-25 Sign & Execute MANUAL | includes | UC-PE: Policy Evaluation | Same as R-05 |
| R-07 | «include» | UC-24 Execute AUTOPILOT | includes | UC-TK: Turnkey Signing | Autopilot always uses Turnkey TEE for signing |
| R-08 | «extend» | UC-30 Query Copilot Paid | extends | UC-29 Query Copilot Free | Guard: free tier exhausted (dailyCount ≥ 10) |
| R-09 | «extend» | UC-12 Resubmit KYC | extends | UC-10 Submit KYC | Guard: kycStatus = RESUBMISSION_REQUESTED |
| R-10 | «extend» | UC-28 Retry Failed Execution | extends | UC-24 or UC-25 Execute | Guard: execution_intents.status = FAILED |
| R-11 | «extend» | UC-33 Pause Autopilot (Admin) | extends | UC-08 Pause Autopilot | Guard: admin role; extends user's ability to pause |
| R-12 | «include» | UC-21 Generate Strategy Plan | includes | UC-39 Risk Rescore | Plan generation uses latest risk score |
| R-13 | «include» | UC-21 Generate Strategy Plan | includes | UC-40 Yield Refresh | Plan generation uses latest yield data |
| R-14 | «generalize» | A1 Path A User | generalizes | A1/A2 (Authenticated User) | Both share all portfolio/copilot/kyc UCs; Path A adds AUTOPILOT-specific |

---

## 6. Use Case Boundaries

**System boundary box** contains: UC-01 through UC-40.

**Outside system boundary (external actors):** Google OAuth, HyperVerge KYC, Turnkey TEE, x402.org Facilitator, Blockchain Networks.

**Actor placement (Lucidchart):**
- Path A User: left side of system boundary
- Path B User: left side, below Path A User
- Admin: left side, below Path B User
- System (Automated): bottom-left, inside or on boundary edge
- Google OAuth: right side, top (connected to UC-01)
- HyperVerge KYC: right side, middle (connected to UC-10, UC-12, UC-36)
- Turnkey TEE: right side (connected to UC-04, UC-24)
- x402.org Facilitator: right side, bottom (connected to UC-30)
- Blockchain Networks: right side, bottom (connected to UC-24, UC-25, UC-37)

---

## 7. Lucidchart Shape Guide

| Element | Shape | Lucidchart Name | Style |
|---|---|---|---|
| Actor (human) | Stick figure | "Actor (UML)" | Label below figure |
| Actor (system) | Stick figure with «system» stereotype | "Actor (UML)" | Label with «system» in chevrons |
| External Actor | Stick figure outside boundary | "Actor (UML)" | Greyed out / dashed outline |
| Use Case | Oval / Ellipse | "Use Case (UML)" | Label inside oval |
| System boundary | Rectangle (dashed) | "System Boundary (UML)" | Dashed border; label at top |
| «include» | Dashed arrow, open arrowhead | Dashed connector | Label: «include» |
| «extend» | Dashed arrow, open arrowhead | Dashed connector | Label: «extend»; arrowhead points to base UC |
| «generalize» | Solid arrow, triangle arrowhead | "Generalization" connector | Child → parent direction |
| Association | Solid line, no arrowhead | "Association" connector | Actor ↔ Use Case |

---

## 8. Layout Recommendations

**Suggested Lucidchart layout:**

**Left column (actors):**
- Path A User (top)
- Path B User
- Admin
- System (Automated, bottom)

**Right column (external actors):**
- Google OAuth
- Turnkey TEE
- HyperVerge KYC
- x402.org Facilitator
- Blockchain Networks

**Inside boundary (use cases grouped by domain):**

```
[Authentication cluster — top]       UC-01, UC-02, UC-03

[Wallet cluster]                      UC-04, UC-05, UC-06, UC-07, UC-08, UC-09

[KYC cluster]                         UC-10, UC-11, UC-12

[Portfolio / Risk / Yield cluster]    UC-13, UC-14, UC-15, UC-16, UC-17, UC-18, UC-19, UC-20

[Strategy cluster]                    UC-21, UC-22, UC-23

[Execution cluster]                   UC-24, UC-25, UC-26, UC-27, UC-28

[Copilot cluster]                     UC-29, UC-30, UC-31

[Admin cluster]                       UC-32, UC-33, UC-34, UC-35

[System background — small, bottom]   UC-36, UC-37, UC-38, UC-39, UC-40
```

**Tips:**
- Use Lucidchart "Groups" (containers) for each domain cluster with a light background color and domain label.
- Avoid crossing lines by routing «include» arrows within domain groups and «extend» arrows between adjacent clusters.
- Use `«system»` stereotype annotation on the System actor and all external actors to distinguish from human actors.

---

## 9. Supplementary PlantUML

```plantuml
@startuml CrestFlow_UseCaseDiagram
!theme plain
left to right direction
skinparam actorStyle awesome

actor "Path A User" as UA
actor "Path B User" as UB
actor "Admin" as ADM
actor "System\n(Automated)" as SYS

actor "Google OAuth" as GOAUTH
actor "HyperVerge KYC" as HV
actor "Turnkey TEE" as TK
actor "x402.org" as X402
actor "Blockchain" as BC

rectangle "CrestFlow System Boundary" {
  package "Authentication" {
    usecase "Sign In with Google" as UC01
    usecase "Refresh Session" as UC02
    usecase "Sign Out" as UC03
  }
  package "Wallet" {
    usecase "Add Embedded Wallet (Path A)" as UC04
    usecase "Add External Wallet (Path B)" as UC05
    usecase "Verify Wallet Ownership" as UC06
    usecase "Remove Wallet" as UC07
    usecase "Pause Autopilot" as UC08
    usecase "Resume Autopilot" as UC09
  }
  package "KYC" {
    usecase "Submit KYC Application" as UC10
    usecase "View KYC Status" as UC11
    usecase "Resubmit KYC Documents" as UC12
  }
  package "Portfolio / Risk / Yield" {
    usecase "View Portfolio Overview" as UC13
    usecase "View Risk Score" as UC16
    usecase "View Yield Opportunities" as UC19
  }
  package "Strategy" {
    usecase "Generate Strategy Plan" as UC21
    usecase "Approve Strategy Plan" as UC22
  }
  package "Execution" {
    usecase "Execute Plan — AUTOPILOT" as UC24
    usecase "Sign & Execute — MANUAL" as UC25
    usecase "View Execution Status" as UC26
    usecase "Policy Evaluation" as UCPE
    usecase "Turnkey Signing" as UCtk
  }
  package "Copilot" {
    usecase "Query Copilot — Free" as UC29
    usecase "Query Copilot — Paid (x402)" as UC30
    usecase "Change Goal Profile" as UC31
  }
  package "Admin" {
    usecase "Suspend User" as UC32
    usecase "View System Health" as UC35
  }
}

UA --> UC01 ; UB --> UC01 ; UC01 --> GOAUTH
UA --> UC04 ; UC04 --> TK
UB --> UC05 ; UC05 ..> UC06 : «include»
UA --> UC08 ; UA --> UC09
UA --> UC10 ; UB --> UC10 ; UC10 --> HV
UC12 ..> UC10 : «extend»\n[RESUBMISSION_REQUESTED]
UA --> UC13 ; UB --> UC13
UA --> UC22 ; UB --> UC22
SYS --> UC21
UA --> UC24 ; UC24 ..> UCPE : «include» ; UC24 ..> UCtk : «include» ; UC24 --> BC
UA --> UC25 ; UB --> UC25 ; UC25 ..> UCPE : «include» ; UC25 --> BC
UA --> UC29 ; UB --> UC29
UC30 ..> UC29 : «extend»\n[free tier exhausted]
UC30 --> X402
UA --> UC31 ; UB --> UC31
ADM --> UC32 ; ADM --> UC35
@enduml
```

---

## 10. Flags & Open Questions

| Flag ID | Type | Description |
|---|---|---|
| F-15-01 | **D2 / Confirmed** | GMX is READ-ONLY display only. No "Execute on GMX" use case exists in MVP. GMX data is shown through UC-13/UC-15 (view portfolio/position). |
| F-15-02 | **D1 / Confirmed** | Path B Users cannot access UC-04 (Add Embedded Wallet) or UC-08/UC-09 (Autopilot pause/resume) — these are Path A only. |
| F-15-03 | **Conditional** | UC-10 and UC-36 both depend on HyperVerge KYC (D9 — conditional). If vendor changes, the external actor name changes. |
| F-15-04 | **Out of MVP** | Fiat on-ramp use case (T16) is excluded. No "Buy Crypto" or "Deposit Fiat" use case. |
| F-15-05 | **Assumption** | "Policy Evaluation" (UCPE) is modeled as an internal use case to show «include» relationships. It is not directly triggered by any actor — it is always included by execution use cases. |
