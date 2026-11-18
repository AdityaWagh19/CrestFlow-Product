# User Onboarding

**Status:** Component  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md` D1, D9, T8, T9, T16  
**Canonical schemas:** `docs/canonical_schemas.md` §3  
**Chain scope:** Ethereum, Base, Arbitrum, Solana.

---

## Purpose

Move a user from first landing on CrestFlow to an active, fully analysed portfolio state, with either autonomous execution (Path A) or read-only tracking plus manual execution (Path B). The two paths are structurally different, not cosmetic UX.

---

## Step 1 — Authentication

**Google OAuth 2.0.** The user selects "Sign in with Google"; Google returns an ID token; CrestFlow verifies the signature against Google's public keys and reads `sub`, `email`, and `email_verified`.

**Identity anchor.** `googleSub` is the unique, stable identity. `email` is mutable profile data and is never a foreign key. The OAuth callback upserts by `googleSub`; changing an email never creates a second user.

**Session.** Short-lived JWT access token plus a rotating refresh token. The access token carries `userId`, `sessionId`, and expiry. Refresh tokens are hashed in Postgres and can be revoked per device.

**Idempotency.** The callback is idempotent on `googleSub` plus a request nonce. A double-click or retried redirect cannot create duplicate users or duplicate wallet-provisioning attempts.

**Stored:** `User { id, googleSub, email, emailVerified, kycStatus, kycTier, kycUpdatedAt, createdAt }`.

---

## Step 2 — Path Fork

### Path A — New Web3 User

No existing portfolio. Provision an embedded Turnkey wallet, complete KYC, fund the wallet, and use the product. Execution mode is `AUTOPILOT`; Turnkey is the only signer.

### Path B — Existing Web3 User

Existing assets across chains and wallets. CrestFlow:

1. Registers each provided address per chain as `MANUAL` (read-only tracking plus user-signed execution) or `READ_ONLY` (watch-only).
2. Requires a wallet-signature challenge before the address is trusted for tracking.
3. Does **not** require asset migration.
4. Does **not** take delegated authority. Every transaction is signed by the user in their own wallet (D1).

Path B is a first-class **read-only + manual** experience. There is no ERC-4337 session key, no EIP-7702 delegation, and no SPL `delegate` in MVP. Those are Phase 2 options, not MVP behaviour.

---

## Step 3 — Embedded Wallet Provisioning (Path A)

Path A receives one embedded CrestFlow wallet. Path B does not require an embedded wallet; it may optionally create one for new capital, but that wallet is a separate `AUTOPILOT` scope, not a delegation target for existing assets.

**TEE provider: Turnkey.** One isolated sub-organisation per user. Keys are generated inside the enclave and never leave it. CrestFlow stores only `walletId` and `subOrgId` references, never key material.

**Key derivation.** Two curves cover four chains:

| Account | Curve | Derivation Path | Chains |
|---|---|---|---|
| EVM account | secp256k1 | `m/44'/60'/0'/0/0` | Ethereum, Base, Arbitrum (single shared address) |
| Solana account | Ed25519 | `m/44'/501'/0'/0'` | Solana only |

Lazy derivation is not used; two keys are derived eagerly at wallet creation.

**Stored:** `AddressScope { userId, address, chain, executionMode: 'AUTOPILOT', turnkeySubOrgId, ownershipVerifiedAt }` for each of the two address strings, plus the wallet reference on the user.

### Two-Layer Authority Model (Path A)

1. **Turnkey-native policy layer** — attached to the sub-org/key inside the enclave: spend limits, destination allowlists, permitted method types. This is the hard floor and holds even if the CrestFlow app server is compromised.
2. **CrestFlow Policy Engine** — nine runtime checks (KYC, tier limit, execution authority, risk, chain allowlist, protocol allowlist, per-transaction value, slippage, action types). All business-logic configurability lives here.

Path B has only the second layer plus the user's own signature; it has no delegated key in the enclave.

---

## Step 4 — Deposit, Track, or Sign Manually

**Path A** deposits into the embedded wallet. No ambiguity.

**Path B** keeps custody and signs manually:

- **EVM:** the user signs each transaction in MetaMask or another wallet. CrestFlow returns an unsigned transaction plus a deep link or wallet-connect payload. The user submits the signed transaction; CCE verifies the signature against the verified owner before broadcast.
- **Solana:** the user signs the serialized message in Phantom or another wallet; CCE verifies and broadcasts.

**Why no migration:** migration costs gas for no functional gain, some positions are non-transferable, consolidation increases blast radius, and moving VDAs may have tax treatment implications in India. Path B exists precisely so users do not have to migrate.

**Product limitation, stated plainly:** Path B cannot automatically rebalance Aave/Uniswap/Kamino/MarginFi positions. It produces plans, simulations, and unsigned transactions; the user executes them. This is a deliberate MVP boundary, not an implementation gap.

---

## Step 5 — KYC

KYC is required before any execution, on both paths. The flow is in `docs/identity_kyc.md`.

- `PENDING` users can browse and plan but cannot execute.
- `APPROVED` users receive a tier and a daily execution limit.
- `SUSPENDED` / `EXPIRED` / `REJECTED` users cannot execute; the Policy Engine returns `KYC_NOT_APPROVED`.
- KYC is completed once and enforced on all four chains.

**Fiat on-ramp (T16):** out of MVP. Onboarding may link to a third-party on-ramp as an optional external step, with no CrestFlow custody, no embedded widget contract, and no shared KYC. Phase 2.

---

## Step 6 — First Portfolio Scan

Fires a `PortfolioScanTriggered` event using the **outbox pattern**: the event is written in the same database transaction as the address configuration change, then published asynchronously. This guarantees no scan is silently dropped if the process crashes between state change and publish.

**Trigger type:** `ONBOARDING`.

- Path A: near-no-op scan — one embedded wallet, empty until funded.
- Path B: parallel scan across all registered external addresses via the FKL chain adapters.

If a chain scan exceeds the 10-second `maxWait`, the snapshot is `PARTIAL` and a re-sync is scheduled.

---

## Address Scope Schema

The canonical schema is `docs/canonical_schemas.md` §3. Summary:

```typescript
AddressScope {
  id: UUID,
  userId: UUID,
  address: string,
  chain: Chain,
  executionMode: 'AUTOPILOT' | 'MANUAL' | 'READ_ONLY',
  ownershipVerifiedAt: IsoDateTime | null,
  turnkeySubOrgId?: string,   // AUTOPILOT only
  label?: string,
  createdAt: IsoDateTime
}
```

**Invariants:**

- `AUTOPILOT` requires `turnkeySubOrgId`.
- `MANUAL` requires `ownershipVerifiedAt`; the user signs each transaction.
- `READ_ONLY` never executes and may have no ownership proof.
- An address is unique per `(userId, chain, address)`.
- Deleting an address with an open execution returns `409 ADDRESS_IN_USE`.

---

## Session Model

**Onboarding state machine on the user record:**

```
PENDING_WALLET → WALLET_CREATED → PENDING_KYC → KYC_APPROVED
```

Wallet creation involves an external call to Turnkey that can partially fail; the user resumes from the actual stuck state rather than restarting.

**Per-session revocation.** The `auth_sessions` table is keyed by session/JTI. Revoking one compromised device does not sign out every device.

**Idempotency throughout the onboarding saga.** Onboarding spans CrestFlow's database, Turnkey, and the KYC vendor. Every external call is safe to retry without double-creating state.

---

## Resolved Items

1. **Path B authority** — D1: read-only plus manual signing. No delegation, no session key, no SPL delegate. The old "delegate-don't-migrate" model is removed from MVP.
2. **Embedded wallet for Path B** — optional, separate `AUTOPILOT` scope; never a delegation target for external assets.
3. **Fiat on-ramp** — T16: out of MVP; optional external link only.
4. **Self-custody exit path** — Turnkey key-export flow is a post-MVP requirement; it does not change the MVP authority model.
5. **Step-up authorization** — available as a user setting for large transactions; the Policy Engine's per-transaction cap and tier limit remain the hard controls.
