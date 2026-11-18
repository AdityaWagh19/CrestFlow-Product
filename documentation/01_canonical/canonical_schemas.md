# CrestFlow Canonical Schemas

**Status:** Authoritative  
**Version:** 2.0  
**Last reviewed:** 18 November 2026  
**Canonical decisions:** `CANONICAL_DECISIONS.md`  
**Purpose:** Single source of truth for shared TypeScript types, enums, units, and invariants. Component documents may extend these types but must not rename or duplicate canonical fields.

---

## 1. Conventions

```typescript
type UUID = string;
type IsoDateTime = string;   // ISO-8601 UTC, e.g. "2026-11-18T12:00:00.000Z"
type BaseUnits = string;     // integer string in token base units, e.g. "1000000" = 1 USDC
type Usd = string;           // decimal string, max 8 dp, e.g. "1234.56789012"
type Percent = number;       // 0..100, never a fraction
type Bps = number;           // basis points, 0..10000
type Hex = `0x${string}`;
```

**Hard rules**

1. Money is never a JavaScript `number`. Token amounts use `BaseUnits`; USD estimates use `Usd`.
2. Every token amount carries its `decimals` and `symbol` through `AssetRef`.
3. Amounts are rounded **down** for user-protective values (minimum received, maximum spend).
4. Timestamps are UTC ISO-8601 strings.
5. On-chain addresses are stored in their canonical chain format; EVM addresses are lowercase hex, Solana addresses are base58.
6. No field may be inferred by a downstream component. If a component needs it, it is in the schema.

---

## 2. Enums

```typescript
export type Chain = 'ethereum' | 'base' | 'arbitrum' | 'solana';

export type EvmChainId = 1 | 8453 | 42161;
export type CctpDomain = 0 | 3 | 5 | 6; // Ethereum=0, Arbitrum=3, Solana=5, Base=6

export type ProtocolId =
  | 'aave-v3'
  | 'uniswap-v3'
  | 'aerodrome'
  | 'moonwell'
  | 'jupiter'
  | 'raydium'
  | 'kamino'
  | 'marginfi'
  | 'gmx'; // read-only in MVP

export type ActionType =
  | 'SWAP'
  | 'LP_ADD'
  | 'LP_REMOVE'
  | 'LEND_SUPPLY'
  | 'LEND_WITHDRAW'
  | 'BORROW'
  | 'REPAY'
  | 'BRIDGE'
  | 'APPROVE'
  | 'ATA_CREATE'
  | 'MANUAL_REVIEW';

export type PoolType =
  | 'constant_product'
  | 'stable_swap'
  | 'correlated'
  | 'gmx_gm_pool'   // classification/reporting only; manual execution
  | 'lending_market'
  | 'perps'
  | 'none';

export type BridgeMechanism = 'CCTP_V2' | 'NONE';
export type CctpTransferSpeed = 'STANDARD' | 'FAST';

export type ExecutionMode = 'AUTOPILOT' | 'MANUAL' | 'READ_ONLY';

export type Persona =
  | 'CAPITAL_PRESERVATION'
  | 'CONSERVATIVE'
  | 'MODERATE'
  | 'AGGRESSIVE'
  | 'YIELD_SEEKER';

export type KycStatus =
  | 'PENDING'
  | 'APPROVED'
  | 'REJECTED'
  | 'RESUBMISSION_REQUESTED'
  | 'SUSPENDED'
  | 'EXPIRED';

export type KycTier = 'TIER_1' | 'TIER_2' | 'TIER_3';

export type RiskLevel = 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
export type DataQuality = 'COMPLETE' | 'PARTIAL' | 'STALE';
export type DriftUrgency = 'NONE' | 'LOW' | 'MEDIUM' | 'HIGH' | 'EMERGENCY';

export type AuditCategory =
  | 'AUTH'
  | 'KYC'
  | 'EXECUTION'
  | 'PREFERENCE'
  | 'PAYMENT'
  | 'SYSTEM';

export type AuditAction =
  // EXECUTION
  | 'POLICY_EVALUATION'
  | 'SIMULATION'
  | 'SIGNING'
  | 'BROADCAST'
  | 'CONFIRMATION'
  | 'FULFILLMENT'
  | 'COMPENSATION'
  | 'CIRCUIT_BREAKER'
  // KYC
  | 'KYC_SUBMITTED'
  | 'KYC_APPROVED'
  | 'KYC_REJECTED'
  | 'KYC_RESUBMISSION_REQUESTED'
  | 'KYC_REVOKED'
  | 'KYC_EXPIRED'
  // PREFERENCE
  | 'GOAL_UPDATED'
  | 'PREFERENCE_UPDATED'
  | 'AUTOPILOT_ENABLED'
  | 'AUTOPILOT_DISABLED'
  // PAYMENT
  | 'PAYMENT_REQUIRED'
  | 'PAYMENT_SETTLED'
  | 'PAYMENT_FAILED'
  | 'PAYMENT_REPLAY_BLOCKED'
  // AUTH
  | 'LOGIN'
  | 'LOGOUT'
  | 'TOKEN_REFRESH'
  | 'SESSION_REVOKED'
  | 'WALLET_PROVISIONED'
  | 'WALLET_LINKED'
  | 'WALLET_VERIFIED'
  | 'WALLET_REMOVED'
  // SYSTEM
  | 'CONFIG_CHANGED'
  | 'ADAPTER_REGISTERED'
  | 'ADAPTER_REMOVED'
  | 'AUDIT_RECOVERED';

export type PaymentScheme = 'exact';
export type PaymentNetwork = 'eip155:8453' | 'eip155:84532';

export type PolicyDecisionStatus = 'APPROVED' | 'BLOCKED';

export type PolicyFailureCode =
  | 'KYC_NOT_APPROVED'
  | 'KYC_TIER_INSUFFICIENT'
  | 'TIER_LIMIT_EXCEEDED'
  | 'AUTHORITY_MISSING'
  | 'AUTHORITY_REVOKED'
  | 'AUTHORITY_STALE'
  | 'RISK_TOO_HIGH'
  | 'RISK_SNAPSHOT_STALE'
  | 'CHAIN_NOT_ALLOWED'
  | 'PROTOCOL_NOT_ALLOWED'
  | 'TX_VALUE_EXCEEDED'
  | 'SLIPPAGE_EXCEEDED'
  | 'ACTION_NOT_ALLOWED'
  | 'SIMULATION_FAILED'
  | 'STALE_QUOTE'
  | 'UNSUPPORTED_VENUE'
  | 'UNSUPPORTED_BRIDGE'
  | 'POLICY_INPUT_INCOMPLETE';

export type CctpTransferStatus =
  | 'PENDING'
  | 'BURNED'
  | 'ATTESTED'
  | 'MINTED'
  | 'FAILED';
```

---

## 3. Identity and access

```typescript
export interface User {
  id: UUID;
  googleSub: string;         // stable identity anchor; unique
  email: string;             // mutable profile attribute; never a foreign key
  emailVerified: boolean;
  kycStatus: KycStatus;
  kycTier: KycTier;
  kycUpdatedAt: IsoDateTime;
  createdAt: IsoDateTime;
}

export interface AddressScope {
  id: UUID;
  userId: UUID;
  address: string;
  chain: Chain;
  executionMode: ExecutionMode;
  ownershipVerifiedAt: IsoDateTime | null;
  turnkeySubOrgId?: string;  // present only for AUTOPILOT embedded wallets
  label?: string;
  createdAt: IsoDateTime;
}

export interface AuthSession {
  userId: UUID;
  accessToken: string;
  refreshToken: string;
  accessExpiresAt: IsoDateTime;
  refreshExpiresAt: IsoDateTime;
}
```

**Invariants**

- `googleSub` is unique. Email changes never create a new user.
- `AUTOPILOT` requires `turnkeySubOrgId`.
- `MANUAL` requires `ownershipVerifiedAt`.
- `READ_ONLY` may have `ownershipVerifiedAt = null`; it can never execute.

---

## 4. Goals and autopilot

```typescript
export interface GoalProfile {
  persona: Persona;
  maxVolatilePercent: Percent;
  minStablecoinPercent: Percent;
  maxSingleAssetPercent: Percent;
  maxSingleProtocolPercent: Percent;
  maxSingleChainPercent: Percent;
  maxUsdPerTransaction: Usd;
  maxSlippagePercent: Percent;
  driftTolerancePercent: Percent;
  rebalancingFrequencyCap: number; // days
  allowedActionTypes: ActionType[];
  topsisWeights: TopsisWeights;
}

export interface TopsisWeights {
  netApy: number;
  protocolSafety: number;
  yieldConsistency: number;
  liquidity: number;
  ilRisk: number;
}

export interface AutopilotConfig {
  enabled: boolean;
  maxUsdPerExecution: Usd;
  allowedActionTypes: ActionType[];
  allowedChains: Chain[];
  driftThreshold: number;
  bridgeAllowed: boolean;
}
```

**Invariants**

- `allowedActionTypes` is required and non-empty for any user who can execute.
- `maxSlippagePercent` may lower a pool-type cap but never raise it (see T3).
- `GoalProfile` has **no** `maxRiskScore`; the risk threshold comes from `RiskSnapshot.adjustedThreshold`.
- `AutopilotConfig` may further restrict `GoalProfile`, never expand it.

---

## 5. Portfolio

```typescript
export interface AssetRef {
  symbol: string;
  address: string | null; // null for native assets
  chain: Chain;
  decimals: number;
  coingeckoId?: string;
}

export interface PortfolioSnapshot {
  id: UUID;
  userId: UUID;
  createdAt: IsoDateTime;
  dataQuality: DataQuality;
  totalValueUsd: Usd;
  positions: Position[];
  chainExposureUsd: Record<Chain, Usd>;
  assetExposureUsd: Record<string, Usd>;
  protocolExposureUsd: Record<string, Usd>;
  hhi: HhiSet;
}

export interface HhiSet {
  asset: number;
  protocol: number;
  chain: number;
}

export type Position =
  | TokenPosition
  | LpPosition
  | LendingPosition
  | GmxPosition;

interface PositionBase {
  id: UUID;
  chain: Chain;
  protocol: ProtocolId | 'wallet';
  address: string;          // account that holds the position
  valueUsd: Usd;
  dataQuality: DataQuality;
  fetchedAt: IsoDateTime;
}

export interface TokenPosition extends PositionBase {
  kind: 'TOKEN';
  asset: AssetRef;
  amount: BaseUnits;
}

export interface LpPosition extends PositionBase {
  kind: 'LP';
  asset: AssetRef;          // LP share token
  amount: BaseUnits;
  poolId: string;
  poolAddress: string;
  poolType: PoolType;
  token0: AssetRef;
  token1: AssetRef;
  tokenId?: string;         // Uniswap V3 NFT
  underlyingAmounts: { asset: AssetRef; amount: BaseUnits }[];
  inRange?: boolean;
}

export interface LendingPosition extends PositionBase {
  kind: 'LENDING';
  marketId: string;
  supplied: { asset: AssetRef; amount: BaseUnits; valueUsd: Usd }[];
  borrowed: { asset: AssetRef; amount: BaseUnits; valueUsd: Usd }[];
  healthFactor: number | null;
  collateralUsd: Usd;
  debtUsd: Usd;
}

export interface GmxPosition extends PositionBase {
  kind: 'GMX';
  market: string;
  collateralAsset: AssetRef;
  collateralAmount: BaseUnits;
  sizeUsd: Usd;
  isLong: boolean;
  entryPriceUsd: Usd;
  liquidationPriceUsd: Usd | null;
  executionMode: 'MANUAL_ONLY';
}
```

**Invariants**

- `PortfolioSnapshot.dataQuality` is `COMPLETE`, `PARTIAL`, or `STALE`.
- `COMPLETE` when every included position is `COMPLETE`.
- `PARTIAL` when at least one included position is `PARTIAL`, or a chain scan failed/timed out, but fresh data exists for at least one position.
- `STALE` when every included position is `STALE`, or the snapshot itself is served beyond the stale-while-revalidate window.
- If any position is `PARTIAL`, the snapshot is `PARTIAL` unless no position has fresh data, in which case it is `STALE`.
- GMX positions are `MANUAL_ONLY` and never produce automated actions.
- HHI is computed with the T1 formula and bands.

---

## 6. Risk

```typescript
export interface RiskSnapshot {
  id: UUID;
  userId: UUID;
  portfolioSnapshotId: UUID;
  analyzedAt: IsoDateTime;
  freshnessSeconds: number; // must be <= 14400 for policy

  cvar95Percent: number | null;
  sortinoRatio: number | null;
  maxDrawdownPercent: number | null;
  calmarRatio: number | null;
  realizedVol7d: number | null;
  realizedVol30d: number | null;
  insufficientHistory: boolean;

  liquidationPositions: LiquidationPosition[];
  minHealthFactor: number | null;
  crossChainCollateral: CrossChainCollateral;
  liquidationRiskScore: number;

  assetHhi: number;
  protocolHhi: number;
  chainHhi: number;
  concentrationScore: number;

  protocolScores: Record<string, number>;
  weightedProtocolScore: number;
  protocolRiskScore: number;

  exitRiskPositions: ExitRiskPosition[];
  maxExitImpactPercent: number;
  liquidityRiskScore: number;

  riskScore: number;              // 0..100, higher = riskier
  riskLevel: RiskLevel;
  adjustedThreshold: number;      // persona-scaled threshold
  appliedProfileMultiplier: number;
  scoreComponents: Record<string, number>;
}

export interface LiquidationPosition {
  chain: Chain;
  protocol: ProtocolId;
  collateralUsd: Usd;
  debtUsd: Usd;
  healthFactor: number;
  distancePercent: number;
}

export interface CrossChainCollateral {
  totalCollateralUsd: Usd;
  totalDebtUsd: Usd;
  weightedHealthFactor: number | null;
  correlatedLiquidationRisk: boolean;
}

export interface ExitRiskPosition {
  chain: Chain;
  protocol: ProtocolId;
  poolType: PoolType;
  exitImpactPercent: Percent;
  liquidityTier: 'HIGHLY_LIQUID' | 'ACCEPTABLE' | 'ILLIQUID' | 'SEVERELY_ILLIQUID';
}

export type RiskAlertType =
  | 'LIQUIDATION_IMMINENT'
  | 'LIQUIDATION_WARNING'
  | 'HIGH_VOLATILITY'
  | 'SIGNIFICANT_DRAWDOWN'
  | 'LOW_PROTOCOL_SCORE'
  | 'HIGH_CHAIN_CONCENTRATION'
  | 'HIGH_ASSET_CONCENTRATION'
  | 'HIGH_PROTOCOL_CONCENTRATION'
  | 'ORACLE_DIVERGENCE'
  | 'DATA_QUALITY_DEGRADED';

export interface RiskAlert {
  id: UUID;
  userId: UUID;
  riskSnapshotId: UUID;
  type: RiskAlertType;
  severity: 'INFO' | 'WARNING' | 'CRITICAL';
  message: string;
  createdAt: IsoDateTime;
  resolvedAt?: IsoDateTime;
}
```

**Invariants**

- `riskScore` is the only composite risk field. `compositeRiskScore` does not exist.
- Bands: 0–39 LOW, 40–59 MEDIUM, 60–79 HIGH, 80–100 CRITICAL.
- Persona multipliers: 0.5 / 0.7 / 1.0 / 1.3 / 1.15.
- Policy Check 4 compares `riskScore` to `adjustedThreshold`; it never compares to a fixed constant.
- `freshnessSeconds > 14,400` blocks signing.

---

## 7. Yield

```typescript
export interface YieldOpportunitySnapshot {
  id: UUID;
  createdAt: IsoDateTime;
  dataQuality: DataQuality;
  opportunities: YieldOpportunity[];
  topsisCriteria: TopsisCriterion[];
  weightsByPersona: Record<Persona, TopsisWeights>;
}

export interface YieldOpportunity {
  id: string;
  chain: Chain;
  protocol: ProtocolId;
  poolId: string;
  poolAddress: string;
  poolType: PoolType;
  assetSymbols: string[];
  tvlUsd: Usd;
  spotApyPercent: Percent;
  twapApy7dPercent: Percent;      // 7-day TWAP, at least 7 daily points
  yieldConsistencyCv: number;
  ilRiskScore: number;            // 0..100, higher = riskier
  liquidityScore: number;         // 0..100
  protocolSafetyScore: number;    // 0..100
  netApyPercent: Percent;
  fetchedAt: IsoDateTime;
  dataQuality: DataQuality;
}

export interface TopsisCriterion {
  key: 'netApy' | 'protocolSafety' | 'yieldConsistency' | 'liquidity' | 'ilRisk';
  direction: 'MAXIMISE' | 'MINIMISE';
  weight: number;
}
```

**Invariants**

- The APY field is `twapApy7dPercent`; there is no `twapApy30dPercent`.
- Five criteria only. Gas/bridge/payback are filters in Engine 4, not TOPSIS criteria.
- GMX opportunities are excluded from TOPSIS in MVP.

---

## 8. Strategy

```typescript
export interface StrategyPlan {
  id: UUID;
  userId: UUID;
  createdAt: IsoDateTime;
  goalProfileVersion: string;
  portfolioSnapshotId: UUID;
  riskSnapshotId: UUID;
  yieldSnapshotId: UUID;
  urgency: DriftUrgency;
  actions: PlanAction[];
  totalValueUsd: Usd;
  estimatedCostUsd: Usd;
  rationale: string;
  manualReviewItems: ManualReviewItem[];
}

export interface PlanAction {
  id: string;
  type: ActionType;
  chain: Chain;
  protocol: ProtocolId;
  poolId?: string;
  poolAddress?: string;
  tokenId?: string;
  assetIn: AssetRef;
  assetOut?: AssetRef;
  amountUsd: Usd;
  targetWeightPercent?: Percent;
  dependsOn: string[];
}

export interface ManualReviewItem {
  id: string;
  kind: 'GMX_POSITION' | 'UNSUPPORTED_BRIDGE' | 'UNSUPPORTED_VENUE' | 'COMPLEX_ACTION';
  chain: Chain;
  protocol?: ProtocolId;
  description: string;
  suggestedAction?: string;
}
```

---

## 9. Execution

```typescript
export interface ExecutionIntent {
  id: UUID;
  userId: UUID;
  strategyPlanId: UUID;
  createdAt: IsoDateTime;
  source: 'USER_APPROVED' | 'AUTOPILOT';
  executionMode: ExecutionMode;
  urgency: DriftUrgency;
  actions: ExecutionAction[];
  manualReviewItems: ManualReviewItem[];
  estimatedTotalCostUsd: Usd;
  estimatedGasCostUsd: Usd;
}

export interface ExecutionAction {
  id: string;
  type: ActionType;
  chain: Chain;
  protocol: ProtocolId;
  poolId?: string;
  poolAddress?: string;
  tokenId?: string;
  assetIn: AssetRef;
  assetOut?: AssetRef;
  amountIn: BaseUnits;            // integer base units of assetIn
  amountUsd: Usd;
  sourceAddress: string;          // position/owner address
  recipientAddress: string;       // where output lands
  poolType: PoolType;
  slippageCapPercent: Percent;    // effective cap after min(goal, poolType)
  deadline: IsoDateTime;
  approval?: ApprovalIntent;
  bridge?: CctpBridgeIntent;
  dependsOn: string[];
  estimatedCostUsd: Usd;
  overrideReason?: string;
}

export interface ApprovalIntent {
  token: AssetRef;
  spender: string;
  amount: BaseUnits;
  mechanism: 'ERC20_APPROVE' | 'EIP2612_PERMIT' | 'PERMIT2_APPROVE' | 'PERMIT2_PERMIT';
  deadline?: IsoDateTime;
}

export interface CctpBridgeIntent {
  mechanism: 'CCTP_V2';
  sourceDomain: CctpDomain;
  destinationDomain: CctpDomain;
  destinationChain: Chain;
  mintRecipient: string;          // 32-byte padded EVM address or Solana pubkey
  destinationCaller: Hex;         // 0x0 in MVP
  burnToken: string;              // USDC on source chain
  amount: BaseUnits;
  maxFee: BaseUnits;              // 0 for Standard
  minFinalityThreshold: 1000 | 2000;
  speed: CctpTransferSpeed;
}
```

**Invariants**

- `amountIn` is base units, never a decimal string.
- `protocol`, `poolId`/`poolAddress`, `sourceAddress`, `recipientAddress`, `slippageCapPercent`, and `deadline` are mandatory for every executable action.
- `bridge` is present only for `BRIDGE` actions and only `CCTP_V2`.
- `approval` is present when an allowance is required; the mechanism is explicit.
- No `BRIDGE_AGGREGATOR` value exists.

---

## 10. Router and planner outputs

```typescript
export interface ExecutionInstructions {
  id: UUID;
  intentId: UUID;
  createdAt: IsoDateTime;
  actions: ResolvedAction[];
  totalGasEstimateUsd: Usd;
  warnings: string[];
}

export interface ResolvedAction {
  actionId: string;
  type: ActionType;
  chain: Chain;
  protocol: ProtocolId;
  poolAddress?: string;
  tokenId?: string;
  routerAddress: string;
  calldata: Hex;
  value: BaseUnits;
  gasLimit: string;
  maxFeePerGas?: BaseUnits;
  maxPriorityFeePerGas?: BaseUnits;
  deadline: IsoDateTime;
  approval?: ApprovalInstruction;
  bridge?: CctpBridgeInstruction;
  routeLegs?: RouteLeg[];
}

export interface RouteLeg {
  venue: ProtocolId;
  tokenIn: AssetRef;
  tokenOut: AssetRef;
  amountIn: BaseUnits;
  minAmountOut: BaseUnits;
  poolAddress?: string;
  feeTier?: number;
  sharePercent: Percent;
}

export interface ApprovalInstruction {
  token: AssetRef;
  spender: string;
  amount: BaseUnits;
  mechanism: 'ERC20_APPROVE' | 'EIP2612_PERMIT' | 'PERMIT2_APPROVE' | 'PERMIT2_PERMIT';
  calldata?: Hex;                 // for approve/permit transactions
  signatureRequired: boolean;     // true for permit-based flows
  deadline?: IsoDateTime;
}

export interface CctpBridgeInstruction {
  sourceDomain: CctpDomain;
  destinationDomain: CctpDomain;
  mintRecipient: string;
  destinationCaller: Hex;
  burnToken: string;
  amount: BaseUnits;
  maxFee: BaseUnits;
  minFinalityThreshold: 1000 | 2000;
  speed: CctpTransferSpeed;
  burnCalldata: Hex;
  mintCalldataTemplate?: Hex;     // finalized after attestation
}
```

---

## 11. Simulation

```typescript
export interface SimulationPassedPackage {
  id: UUID;
  intentId: UUID;
  instructionsId: UUID;
  userId: UUID;
  simulatedAt: IsoDateTime;
  actions: SimulatedAction[];
  totalGasUsd: Usd;
  chains: Chain[];
  blockNumbers: Partial<Record<Chain, string>>;
  stateOverridesUsed: boolean;
  warnings: string[];
}

export interface SimulatedAction {
  actionId: string;
  success: boolean;
  gasUsed: string;
  gasCostUsd: Usd;
  simulatedSlippagePercent?: Percent;
  simulatedPriceImpactPercent?: Percent;
  minAmountOut?: BaseUnits;
  balanceDeltas: BalanceDelta[];
  revertReason?: string;
}

export interface BalanceDelta {
  account: string;
  asset: AssetRef;
  delta: BaseUnits; // signed integer string
}
```

**Invariants**

- A package with any `success = false` action cannot be passed to Policy as approved; the Planner returns a simulation failure.
- `totalGasUsd` is an estimate in USD; the authoritative gas fields remain in base units on the transaction.
- `blockNumbers` pins the simulation to a chain state.

---

## 12. Policy

```typescript
export interface PolicyInput {
  userId: UUID;
  policyVersion: string;
  user: User;
  goalProfile: GoalProfile;
  riskSnapshot: RiskSnapshot;
  addressScopes: AddressScope[];
  plan: StrategyPlan;
  simulation: SimulationPassedPackage;
  auditSpend24hUsd: Usd;
  autopilotConfig?: AutopilotConfig;
  now: IsoDateTime;
}

export interface PolicyCheckResult {
  check: 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
  name: string;
  passed: boolean;
  code?: PolicyFailureCode;
  message?: string;
  details?: Record<string, unknown>;
}

export interface PolicyFailure {
  check: 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
  code: PolicyFailureCode;
  message: string;
  actionId?: string;
}

export interface PolicyDecision {
  id: UUID;
  userId: UUID;
  planId: UUID;
  decision: PolicyDecisionStatus;
  checksRun: PolicyCheckResult[];  // always 9 entries
  failures: PolicyFailure[];
  policyVersion: string;
  decidedAt: IsoDateTime;
  simulationPackageId: UUID;
}
```

**Invariants**

- `checksRun` always has exactly 9 entries, even when the decision is `BLOCKED`.
- No short-circuit: every check executes.
- `AUTHORITY_MISSING` applies to `AUTOPILOT` when the Turnkey delegation is not active; `MANUAL` requires `ownershipVerifiedAt` and the signature is verified later at broadcast.
- `POLICY_INPUT_INCOMPLETE` blocks if any required field is missing; never default to pass.

---

## 13. Audit

```typescript
export interface AuditEntry {
  eventId: UUID;
  schemaVersion: '2.0';
  userId: UUID | null;              // null for unauthenticated agents and system events
  category: AuditCategory;
  action: AuditAction;
  status: 'SUCCESS' | 'FAILURE' | 'BLOCKED';
  executionGroupId?: UUID;
  strategyPlanId?: UUID;
  intentId?: UUID;
  txId?: string;
  chain?: Chain;
  kycStatus: KycStatus | null;      // snapshot; null only when there is no user context
  kycTier: KycTier | null;
  walletAddresses: Record<string, unknown> | null; // encrypted JSON
  payload: Record<string, unknown>; // encrypted where it contains PII
  ipAddress: string | null;         // AUTH/PAYMENT only; encrypted
  userAgent: string | null;         // AUTH/PAYMENT only; encrypted
  createdAt: IsoDateTime;
}
```

**Invariants**

- Append-only; no updates or deletes.
- `category = 'EXECUTION'` is the source for tier-spend queries.
- KYC events use `category = 'KYC'`.
- `eventId` is idempotent and unique; duplicate webhook deliveries must not create duplicate entries.
- `kycStatus` and `kycTier` are snapshotted, not referenced by foreign key.

---

## 14. CCTP transfers

```typescript
export interface CctpTransfer {
  id: UUID;
  userId: UUID;
  intentId: UUID;
  actionId: string;
  sourceChain: Chain;
  destinationChain: Chain;
  sourceDomain: CctpDomain;
  destinationDomain: CctpDomain;
  amount: BaseUnits;
  mintRecipient: string;
  destinationCaller: Hex;
  burnToken: string;
  maxFee: BaseUnits;
  minFinalityThreshold: 1000 | 2000;
  speed: CctpTransferSpeed;
  burnTxId?: string;
  attestation?: string;
  mintTxId?: string;
  status: CctpTransferStatus;
  expectedReceivedAmount: BaseUnits;
  actualReceivedAmount?: BaseUnits;
  createdAt: IsoDateTime;
  updatedAt: IsoDateTime;
}
```

**Invariants**

- `expectedReceivedAmount = amount - maxFee` for Fast; `amount` for Standard.
- `actualReceivedAmount` is filled from the destination mint/transfer.
- `status` transitions: `PENDING → BURNED → ATTESTED → MINTED`, or `FAILED` from any state.

---

## 15. x402 payments

```typescript
export interface PaymentRequirements {
  scheme: PaymentScheme;
  network: PaymentNetwork;
  maxAmountRequired: BaseUnits;
  resource: string;
  description: string;
  mimeType: string;
  payTo: string;
  maxTimeoutSeconds: number;
  asset: string;             // USDC contract address
  extra?: {
    name?: string;
    version?: string;
  };
}

export interface PaymentPayload {
  x402Version: 2;
  scheme: PaymentScheme;
  network: PaymentNetwork;
  payload: {
    signature: Hex;
    authorization: {
      from: string;
      to: string;
      value: BaseUnits;
      validAfter: string;    // unix seconds
      validBefore: string;   // unix seconds
      nonce: Hex;
    };
  };
}

export interface PaymentResponse {
  success: boolean;
  transaction?: string;
  network?: PaymentNetwork;
  payer?: string;
  errorReason?: string;
}

export interface PaymentRecord {
  id: UUID;
  userId: UUID | null;            // null for anonymous agents
  network: PaymentNetwork;
  scheme: PaymentScheme;
  amount: BaseUnits;              // 10000 = $0.01 at 6 decimals
  asset: string;                  // USDC contract address
  status: 'REQUIRED' | 'SETTLED' | 'FAILED' | 'REPLAY_BLOCKED';
  payer?: string;
  facilitatorTx?: string;
  paymentPayloadHash: string;
  createdAt: IsoDateTime;
  settledAt?: IsoDateTime;
}
```

**Invariants**

- Headers are `PAYMENT-REQUIRED`, `PAYMENT-SIGNATURE`, `PAYMENT-RESPONSE`.
- The `exact` scheme uses EIP-3009 authorization; no transaction hash is sent by the client.
- Replay protection is the authorization window plus facilitator settlement tracking.
- `PaymentRecord.status` is `REQUIRED → SETTLED`, or `FAILED` / `REPLAY_BLOCKED`; `paymentPayloadHash` is unique per settlement attempt.
- 5xx responses are not settled.

---

## 16. UML mapping

| Canonical type | UML class | Key relationships |
|---|---|---|
| `User` | `User` | 1 → many `AddressScope`, 1 → 0..1 `GoalProfile` |
| `GoalProfile` | `GoalProfile` | 1 → 1 `User`; contains `TopsisWeights` |
| `PortfolioSnapshot` | `PortfolioSnapshot` | 1 → many `Position`; 1 → many `RiskSnapshot` |
| `RiskSnapshot` | `RiskSnapshot` | many → 1 `PortfolioSnapshot`; contains `RiskAlert` |
| `YieldOpportunitySnapshot` | `YieldOpportunitySnapshot` | 1 → many `YieldOpportunity` |
| `StrategyPlan` | `StrategyPlan` | 1 → many `PlanAction`; links to three snapshots |
| `ExecutionIntent` | `ExecutionIntent` | 1 → many `ExecutionAction`; 1 → 1 `StrategyPlan` |
| `ExecutionInstructions` | `ExecutionInstructions` | 1 → many `ResolvedAction` |
| `SimulationPassedPackage` | `SimulationPassedPackage` | 1 → many `SimulatedAction` |
| `PolicyInput` | `PolicyInput` | aggregates user, goal, risk, plan, simulation |
| `PolicyDecision` | `PolicyDecision` | 1 → 9 `PolicyCheckResult` |
| `AuditEntry` | `AuditEntry` | many → 1 `User`; optional links to execution objects |
| `CctpTransfer` | `CctpTransfer` | 1 → 1 `ExecutionAction` |
| `PaymentRequirements` | `PaymentRequirements` | used by `PAYMENT-REQUIRED` |
| `PaymentPayload` | `PaymentPayload` | used by `PAYMENT-SIGNATURE` |
| `PaymentRecord` | `PaymentRecord` | settlement ledger; 0..1 → `User` |

---

## 17. Validation checklist

Before any schema change is accepted:

- [ ] Every monetary field uses `BaseUnits` or `Usd`, never `number`.
- [ ] Every execution action has `protocol`, `sourceAddress`, `recipientAddress`, `deadline`, and `slippageCapPercent`.
- [ ] Only `CCTP_V2` appears as a bridge mechanism.
- [ ] `riskScore` is used everywhere; no `compositeRiskScore`.
- [ ] `twapApy7dPercent` is used everywhere; no `twapApy30dPercent`.
- [ ] `allowedActionTypes` exists on `GoalProfile`.
- [ ] `dataQuality` is present on portfolio, yield, and position objects.
- [ ] Audit entries use `category` + `action`, not a single flat enum.
- [ ] Every canonical KYC status transition has a category-scoped audit action.
- [ ] No `BRIDGE_AGGREGATOR`, `gmx_gm_pool` automated action, or `PROFILE_CHANGE` remains.
