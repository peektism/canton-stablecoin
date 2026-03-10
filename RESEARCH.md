# DAML Patterns for Stablecoins and Payment Infrastructure

Patterns extracted from five FOSS reference projects for building DAML stablecoins and infrastructure corresponding to existing payment systems used by US financial institutions.

**Source projects:**
- [canton-dex-platform](https://github.com/0xalberto/canton-dex-platform)
- [canton-erc20](https://github.com/ChainSafe/canton-erc20)
- [canton-vault](https://github.com/ted-gc/canton-vault)
- [daml-finance](https://github.com/digital-asset/daml-finance)
- [splice](https://github.com/hyperledger-labs/splice)

**Date:** 2026-03-04
**Audit tools:** [daml-lint](https://github.com/4meta5/daml-lint), [daml-props](https://github.com/4meta5/daml-props), [daml-verify](https://github.com/4meta5/daml-verify)

---

## Table of Contents

1. [Token Holding Patterns](#1-token-holding-patterns)
2. [Transfer Dispatch (Three-Way)](#2-transfer-dispatch-three-way)
3. [Atomic Settlement / DvP](#3-atomic-settlement--dvp)
4. [Mint and Burn](#4-mint-and-burn)
5. [Fee Calculation](#5-fee-calculation)
6. [Lock Semantics](#6-lock-semantics)
7. [Subscription and Recurring Payments](#7-subscription-and-recurring-payments)
8. [Transfer Pre-Approvals](#8-transfer-pre-approvals)
9. [Vault / Yield-Bearing Tokens](#9-vault--yield-bearing-tokens)
10. [Cross-Chain Bridge](#10-cross-chain-bridge)
11. [Governance and Configuration](#11-governance-and-configuration)
12. [UTXO Consume-and-Create](#12-utxo-consume-and-create)
13. [Multi-Party Authorization](#13-multi-party-authorization)
14. [Audit Logging](#14-audit-logging)
15. [Settlement Lifecycle (daml-finance)](#15-settlement-lifecycle-daml-finance)
16. [Async Request Queuing (EIP-7540)](#16-async-request-queuing-eip-7540)
17. [Payment Application Integration](#17-payment-application-integration)
18. [Mapping to US Payment Infrastructure](#18-mapping-to-us-payment-infrastructure)

---

## 1. Token Holding Patterns

### CIP-056 Holding (canton-erc20, canton-vault, splice)

The Canton Token Standard (CIP-056) defines holdings as UTXO contracts with dual signatories:

```daml
template SimpleHolding
  with
    admin : Party          -- issuer / registry operator
    owner : Party          -- holder of the tokens
    instrumentId : InstrumentId
    amount : Decimal
    lock : Optional Lock
    meta : Metadata
  where
    signatory admin, owner
    ensure amount > 0.0
```

**Why dual-sig:** Prevents either party from unilaterally archiving the holding. Admin cannot steal; owner cannot forge.

**Stablecoin application:** Stablecoin issuer is `admin`, holder is `owner`. Neither can move funds alone — transfers require a factory contract that both trust.

### Issuer-Controlled Holding (canton-erc20)

For retail stablecoins where users don't manage Canton keys:

```daml
template CIP56Holding
  with
    issuer : Party         -- sole signatory
    owner : Party          -- observer only
    instrumentId : InstrumentId
    amount : Decimal
    lock : Optional Lock
    meta : Metadata
  where
    signatory issuer
    observer owner
```

**Trade-off:** Simpler key management (issuer holds all keys) but issuer is single point of failure. Suitable for custodial stablecoin models where the issuer is a regulated bank.

### Expiring Holding (splice)

For holdings that decay over time (storage fees, demurrage):

```daml
data ExpiringAmount = ExpiringAmount
  with
    initialAmount : Decimal
    createdAt : Round
    ratePerRound : RatePerRound

getValueAsOfRound : Round -> ExpiringAmount -> Decimal
getValueAsOfRound currentRound ea =
  ea.initialAmount - ea.ratePerRound.rate * intToDecimal (currentRound.number - ea.createdAt.number)
```

**Payment application:** Implements "inactivity fees" on dormant stablecoin balances. Encourages velocity and reclaims ledger storage.

---

## 2. Transfer Dispatch (Three-Way)

### Pattern from splice token standard

A single `TransferFactory` handles three transfer modes:

```
case (optPreapprovalCid, sender == receiver) of
  (_, True)      -> SELF-TRANSFER: merge/defragment holdings
  (Some cid, _)  -> DIRECT: exercise TransferPreapproval_Accept
  (None, _)      -> TWO-STEP: lock funds, create TransferInstruction
```

**Self-transfer:** Archives multiple input holdings, creates single merged output. Used for defragmentation.

**Direct transfer:** Receiver has pre-approved. Factory reads `TransferPreapproval` contract ID from `ChoiceContext`. Settlement is atomic — single transaction.

**Two-step transfer:** No pre-approval exists. Factory locks sender's funds (creates `LockedSimpleHolding`), creates `TransferInstruction`. Receiver later exercises Accept, Reject, or Withdraw.

**Stablecoin application:** Maps directly to payment infrastructure:
- Self-transfer = account consolidation / sweep
- Direct transfer = pre-authorized ACH / standing order
- Two-step transfer = wire transfer requiring beneficiary confirmation

### Validation Pipeline

Every transfer factory choice runs 9 checks in order:

1. `requireExpectedAdminMatch` — admin identity
2. `assertDeadlineExceeded` — `requestedAt` in the past
3. `assertWithinDeadline` — `executeBefore` in the future
4. `requireDeadlineOrdering` — `allocateBefore <= settleBefore`
5. `requirePositiveAmount` — `amount > 0`
6. `requireInstrumentIdMatch` — `instrumentId.admin == admin`
7. `requireNonEmptyInputs` — at least 1 input holding
8. `fetchAndValidateHolding` — per-input owner/admin check
9. `requireSufficientFunds` — `sum(inputs) >= amount`

---

## 3. Atomic Settlement / DvP

### Pattern from canton-dex-platform

Delivery-versus-Payment with 4-party signature:

```daml
template Settlement
  with
    buyer : Party; seller : Party
    securitiesIssuer : Party; cashProvider : Party
    quantity : Decimal; cashAmount : Decimal
    status : Text
  where
    signatory buyer, seller, securitiesIssuer, cashProvider
    ensure quantity > 0.0 && cashAmount > 0.0

    choice ExecuteDeliveryVsPayment : ContractId SettledDeliveryVsPayment
      controller buyer, seller
      do create SettledDeliveryVsPayment with ...
```

**Key properties:**
- All 4 parties must sign the settlement contract at creation
- Execution requires both buyer and seller
- Creates immutable `SettledDeliveryVsPayment` record (no modification choices)
- Settlement is atomic — succeeds fully or fails fully

**Stablecoin application:** Maps to PvP (Payment-versus-Payment) for stablecoin/FX settlement. Issuer and cash provider co-sign to ensure both legs execute atomically.

### Pattern from daml-finance (V4 Settlement)

More sophisticated multi-leg settlement:

```daml
template Batch
  with instructions : [ContractId Instruction]
  -- All instructions in batch settle atomically

template Instruction
  with
    requestors : Set Party
    settlers : Set Party
    sender : Party; receiver : Party
    instrumentId : InstrumentId
    amount : Decimal
```

**Hierarchical routing:** `RouteProvider` determines intermediary chain. Settlement instructions can be routed through custodians and intermediaries.

---

## 4. Mint and Burn

### Issuer-Controlled Mint (canton-erc20)

```daml
-- On TokenConfig template:
choice IssuerMint : ContractId CIP56Holding
  with recipient : Party; amount : Decimal; meta : Metadata
  controller issuer
  do
    assertMsg "amount > 0" (amount > 0.0)
    create CIP56Holding with issuer; owner = recipient; instrumentId; amount; lock = None; meta
```

### Factory-Based Mint (canton-vault)

Vault mints share tokens proportional to deposited assets:

```daml
-- In Vault.Deposit choice:
let sharesToMint = calculateShares depositAmount state
newShareCid <- create VaultShareHolding with
  vault; owner = depositor; amount = sharesToMint; holdingLock = None; meta
let newState = state with
  totalAssets = state.totalAssets + depositAmount
  totalShares = state.totalShares + sharesToMint
create this with state = newState
```

### Burn on Redemption

```daml
-- In Vault.Redeem choice:
let assetsToReturn = calculateAssets sharesToRedeem state
exercise shareHoldingCid Shares.Burn
-- Transfer underlying assets back to redeemer
```

**Stablecoin application:**
- Mint = stablecoin issuance against collateral deposit
- Burn = stablecoin redemption returning collateral
- Factory pattern ensures mint/burn is always paired with collateral movement

---

## 5. Fee Calculation

### Stepped Rate (splice)

Piecewise linear fee schedule (like US tax brackets):

```daml
data SteppedRate = SteppedRate
  with
    initialRate : Decimal    -- rate for first bracket
    steps : [(Decimal, Decimal)]  -- [(threshold, rate_above_threshold)]

chargeSteppedRate : SteppedRate -> Decimal -> Decimal
-- Example: 1% on first $100, 0.1% on $100-$1000, 0.01% above $1000
```

### USD-to-Token Scaling (splice)

All fees configured in USD, converted at transfer time:

```daml
scaleFees : Decimal -> TransferConfig USD -> TransferConfig Amulet
scaleFees priceMultiplier usdConfig = TransferConfig with
    createFee = scaleFixedFee priceMultiplier usdConfig.createFee
    holdingFee = scaleRatePerRound priceMultiplier usdConfig.holdingFee
    transferFee = scaleSteppedRate priceMultiplier usdConfig.transferFee
    ...
```

**Stablecoin application:** Configure fees in USD. For USD stablecoins, multiplier is 1.0. For non-USD stablecoins, use oracle price at transaction time. Stepped rates model Fedwire/ACH fee tiers.

### Receiver Fee Splitting (splice)

```daml
data TransferOutput = TransferOutput
  with
    receiver : Party
    amount : Decimal
    receiverFeeRatio : Decimal  -- 0.0 = sender pays all, 1.0 = receiver pays all
    lock : Optional TimeLock
```

**Payment application:** Models who pays the wire fee. Sender-pays (OUR), receiver-pays (BEN), or split (SHA) — standard SWIFT MT103 fee conventions.

---

## 6. Lock Semantics

### TimeLock with Expiry (splice)

```daml
data TimeLock = TimeLock
  with
    holders : [Party]           -- must cooperate to unlock
    expiresAt : Time            -- auto-unlock after this time
    optContext : Optional Text  -- "amulet-subscription: monthly"
```

**Three unlock paths:**
1. Owner + all holders cooperate (normal unlock)
2. Owner unilaterally after expiry (timeout fallback)
3. DSO/admin cleanup of expired locks (garbage collection)

### HoldingLock with Context (canton-vault)

```daml
data HoldingLock = HoldingLock
  with
    lockHolder : Party
    context : Text        -- "DVP settlement #12345"
    expiresAt : Optional Time
```

**Stablecoin application:** Locks implement:
- Payment holds (pending ACH debits)
- Escrow for DvP settlement
- Regulatory freezes (compliance officer as lock holder)
- Subscription pre-authorization

---

## 7. Subscription and Recurring Payments

### Five-State Machine (splice)

```
SubscriptionRequest -> SubscriptionInitialPayment -> Subscription
                                                   -> SubscriptionIdleState
                                                   -> SubscriptionPayment
                                                   -> SubscriptionIdleState (cycle)
```

**Key fields:**

```daml
data SubscriptionPayData = SubscriptionPayData
  with
    paymentAmount : PaymentAmount   -- amount per interval
    paymentInterval : RelTime       -- e.g., 30 days
    paymentDuration : RelTime       -- total subscription duration
```

**Payment cycle:**
1. `SubscriptionIdleState` waits until `nextPaymentDueAt`
2. Sender exercises `MakePayment` — locks amulet, creates `SubscriptionPayment`
3. Receiver exercises `Collect` — unlocks and transfers to receiver
4. New `SubscriptionIdleState` created with `nextPaymentDueAt += paymentInterval`

**Lock mechanism:** Each payment locks exact amount + fees with provider and receiver as co-holders. Lock expires if receiver doesn't collect within round window.

**Stablecoin application:** Direct model for:
- Recurring bill payments (utility, rent, insurance)
- Subscription services (SaaS billing)
- Loan repayments (mortgage, auto, student)
- Standing orders / direct debits (UK/EU equivalent)

---

## 8. Transfer Pre-Approvals

### Pattern from splice

Receiver creates a standing authorization before any transfer:

```daml
template TransferPreapproval
  with
    receiver : Party; provider : Party; dso : Party
    lastRenewedAt : Time; expiresAt : Time
  where
    signatory receiver, provider, dso

    -- Non-consuming: allows discovery without archival
    nonconsuming choice TransferPreapproval_Send : TransferResult
      with sender : Party; inputs : [TransferInput]; ...
      controller sender
      do ...
```

**Renewal:**
- Provider renews by paying fee proportional to duration
- `feeAmulet = (relTimeToDays duration) * transferPreapprovalFee / amuletPrice`

**Stablecoin application:** Maps to:
- ACH pre-authorization (receiver's bank pre-approves debits)
- Direct debit mandates (SEPA DD)
- Standing payment instructions
- Payroll direct deposit authorization

---

## 9. Vault / Yield-Bearing Tokens

### ERC-4626 Pattern (canton-vault)

```daml
template Vault
  with
    id : VaultId
    config : VaultConfig       -- depositLimit, minDeposit, fees
    state : VaultState         -- totalAssets, totalShares, lastFeeAccrual

-- Synchronous operations (ERC-4626):
choice Deposit  : ...  -- assets -> shares
choice Mint     : ...  -- exact shares -> required assets
choice Redeem   : ...  -- shares -> assets
choice Withdraw : ...  -- exact assets -> required shares
```

**Share price calculation:**

```daml
sharePrice state =
    if state.totalShares == 0.0 then 1.0
    else state.totalAssets / state.totalShares

calculateShares assets state =
    if state.totalShares == 0.0 then assets  -- 1:1 for first deposit
    else roundDown (assets * state.totalShares / state.totalAssets)
```

**Rounding:** Always round DOWN (favor vault). Prevents share holders from extracting more than deposited.

**First-deposit protection:** When `totalShares == 0`, shares = assets (1:1). Prevents ERC-4626 inflation attack.

**Stablecoin application:**
- Money market fund shares (USDC -> yield-bearing vault shares)
- Certificate of deposit (CD) representation
- Treasury bill tokenization
- Sweep account implementation

---

## 10. Cross-Chain Bridge

### Fingerprint-Based Bridge (canton-erc20)

**Deposit flow (EVM -> Canton):**

```
1. User sends ERC-20 to bridge contract with fingerprint as bytes32
2. Middleware detects event, creates PendingDeposit on Canton
3. Issuer processes: PendingDeposit -> DepositReceipt
4. Issuer mints CIP56Holding for resolved userParty
```

**Withdrawal flow (Canton -> EVM):**

```
1. Issuer creates WithdrawalRequest on behalf of user
2. ProcessWithdrawal burns tokens, creates WithdrawalEvent
3. Middleware detects event, releases ERC-20 on EVM
4. CompleteWithdrawal marks event as done
```

**Fingerprint resolution:**

```daml
-- Canton Party format: "hint::fingerprint"
-- Issuer allocates Party via AllocateParty API
-- Creates FingerprintMapping { userParty, fingerprint, evmAddress }
```

**Stablecoin application:**
- Multi-chain stablecoin issuance (mint on Canton, mirror on Ethereum/Solana)
- Cross-ledger settlement (FedNow -> Canton bridge)
- SWIFT message -> Canton contract bridge

---

## 11. Governance and Configuration

### Byzantine Fault Tolerant Voting (splice)

```daml
-- Voting threshold: 2/3 Byzantine fault tolerance
numRequiredVotes = ceiling ((intToDecimal (numSvs + f + 1)) / 2.0)
  where f = floor ((intToDecimal (numSvs - 1)) / 3.0)
```

**Three-layer governance:**
1. `VoteRequest` — SV proposes action, collects votes
2. `Confirmation` — Created once threshold met
3. `Execution` — Action executed atomically

**Configurable parameters:**
- Fee rates (create, transfer, holding)
- Transfer limits (max inputs, max outputs, max lock holders)
- Issuance schedule (reward rates per coupon type)
- Amulet price (oracle feed)

**Stablecoin application:**
- Multi-bank governance consortium (similar to USDF consortium)
- Parameter changes require majority vote (fee changes, issuance caps)
- Maps to Federal Reserve governance model for rate changes

---

## 12. UTXO Consume-and-Create

### Core Pattern (all projects)

```daml
-- Archive ALL inputs, create ALL outputs in single transaction
holdings <- forA inputHoldingCids \cid -> do
  h <- fetchAndValidate admin sender cid
  archive (fromInterfaceContractId @SimpleHolding cid)
  pure h

let totalInput = sum (map (.amount) holdings)
let change = totalInput - requestedAmount

receiverCid <- create SimpleHolding with
  admin; owner = receiver; amount = requestedAmount; meta

when (change > 0.0) do
  void $ create SimpleHolding with
    admin; owner = sender; amount = change; meta
```

**Critical rules:**
- Archive ALL inputs or NONE (contention-based double-spend prevention)
- Create change output for sender if `totalInput > requestedAmount`
- Client retries with fresh holdings from ACS query on contention conflict

**Stablecoin application:** Every stablecoin transfer follows this pattern. No balance tables — each "balance" is a set of UTXO holdings that sum to the total.

---

## 13. Multi-Party Authorization

### Signatory Patterns by Use Case

| Use Case | Signatories | Example |
|----------|------------|---------|
| Token holding | admin + owner | SimpleHolding (splice) |
| Locked holding | admin + owner + lock.holders | LockedSimpleHolding (splice) |
| Transfer instruction | admin + sender | TransferInstruction (splice) |
| Settlement | buyer + seller + issuer + provider | Settlement (canton-dex) |
| Factory | admin only | TransferFactory (canton-erc20) |
| Governance | DSO (collective) | DsoRules (splice) |
| Custodial holding | issuer only | CIP56Holding (canton-erc20) |

### Controller Patterns

```daml
-- View-based controller (from interface):
controller (view this).transfer.receiver

-- Multi-party controller:
controller [v.settlement.executor, v.transferLeg.sender]

-- Admin + flexible extras:
controller instrumentId.admin, extraActors
```

---

## 14. Audit Logging

### Immutable Event Trail (canton-dex-platform)

```daml
template AuditLog
  with
    actor : Party; logId : Text; accountId : Text
    action : Text; details : Text; timestamp : Time
    observers : [Party]
  where
    signatory actor
    observer observers
    -- No choices: completely immutable
```

### Token Transfer Events (canton-erc20)

```daml
template TokenTransferEvent
  with
    issuer : Party
    fromParty : Optional Party   -- None for mints
    toParty : Optional Party     -- None for burns
    amount : Decimal
    instrumentId : InstrumentId
    meta : Metadata
    auditObservers : [Party]
  where
    signatory issuer
    observer fromParty, toParty, auditObservers
```

**Stablecoin application:**
- BSA/AML transaction reporting (auto-generated per transfer)
- SAR filing support (compliance officers as observers)
- SWIFT gpi tracking equivalent (metadata carries end-to-end reference)

---

## 15. Settlement Lifecycle (daml-finance)

### Full Financial Instrument Lifecycle

```
Instrument Creation -> Lifecycle Event -> Effect Calculation -> Holding Settlement
```

**Components:**

| Template | Purpose |
|----------|---------|
| `Instrument` | Financial instrument definition (bond, equity, swap, etc.) |
| `Event` | Lifecycle event (coupon date, maturity, corporate action) |
| `Effect` | Computed impact of event on holdings |
| `Instruction` | Settlement instruction for each affected holding |
| `Batch` | Atomic batch of settlement instructions |

**Key pattern:** Interface-based design separates instrument definition from settlement mechanics. Any instrument that implements the `Lifecycle` interface can be settled using the same `Batch` + `Instruction` infrastructure.

**Stablecoin application:**
- Interest accrual on yield-bearing stablecoins (lifecycle events)
- Collateral rebalancing (corporate action equivalent)
- Maturity handling for fixed-term deposits

---

## 16. Async Request Queuing (EIP-7540)

### Pattern from canton-vault

```daml
-- Phase 1: User submits request
template DepositRequest
  with
    vault : Vault; owner : Party
    requestId : Text; reqController : Party
    underlyingAmount : Decimal
    status : RequestStatus  -- Pending -> Claimable -> Claimed

-- Phase 2: Admin makes request claimable (off-chain computation)
choice MakeDepositClaimable : ...
  with sharesToMint : Decimal
  controller vault.id.admin
  do create this with status = Claimable; sharesToMint

-- Phase 3: User claims result
choice ClaimDeposit : ContractId VaultShareHolding
  controller owner
  do create VaultShareHolding with amount = sharesToMint; ...
```

**Stablecoin application:**
- Large redemption queuing (prevents bank run)
- Batch settlement processing (collect requests, settle in batch)
- Maps to ACH batch processing model (collect during day, settle at cutoff)

---

## 17. Payment Application Integration

### Wallet Install Pattern (splice)

```daml
template WalletAppInstall
  with dso : Party; validator : Party; endUserParty : Party

-- Orchestrates batched operations:
data CompletedSenderChangeOperation
  = CO_AppPayment { ... }
  | CO_SubscriptionAcceptAndMakeInitialPayment { ... }
  | CO_SubscriptionMakePayment { ... }
  | CO_MergeTransferInputs { ... }
  | CO_BuyMemberTraffic { ... }
  | CO_TransferPreapprovalSend { ... }
```

**Execution pattern:** Sequential operation execution with result threading. Each operation consumes the sender change amulet from the previous step, enabling complex multi-step payment workflows in a single submission.

### Multi-Receiver Payments (splice)

```daml
template AppPaymentRequest
  with
    sender : Party
    receiverAmounts : [ReceiverAmount]  -- multiple receivers
    provider : Party; dso : Party
    expiresAt : Time; description : Text
  where
    signatory sender, receivers, provider
```

**Flow:** Accept -> Lock funds -> Collect (per receiver) or Reject/Expire

**Stablecoin application:**
- Payroll disbursement (single sender, multiple employee receivers)
- Dividend distribution
- Batch vendor payments

---

## 18. Mapping to US Payment Infrastructure

### Pattern-to-Infrastructure Mapping

| US Payment System | DAML Pattern | Source Project |
|-------------------|-------------|---------------|
| **Fedwire** (real-time gross) | Direct transfer via TransferPreapproval | splice |
| **ACH** (batch) | Async request queuing + batch settlement | canton-vault, daml-finance |
| **ACH pre-auth** | TransferPreapproval (standing mandate) | splice |
| **SWIFT MT103** | Two-step transfer with receiver confirmation | splice |
| **SWIFT gpi** | Metadata tracking on TokenTransferEvent | canton-erc20 |
| **FedNow** (instant) | Self-transfer / direct transfer (atomic) | splice |
| **DTCC settlement** | Batch settlement with RouteProvider | daml-finance |
| **DvP / T+1** | Settlement with 4-party signatory | canton-dex-platform |
| **Wire transfer** | Two-step transfer (lock -> accept) | splice |
| **Direct deposit** | TransferPreapproval + recurring | splice |
| **Bill pay** | Subscription state machine | splice |
| **Standing order** | SubscriptionIdleState cycle | splice |
| **Escrow** | LockedSimpleHolding with TimeLock | splice |
| **CD / Money market** | Vault deposit/redeem with yield | canton-vault |
| **Treasury management** | Multi-instrument settlement lifecycle | daml-finance |
| **BSA/AML reporting** | AuditLog + TokenTransferEvent observers | canton-dex, canton-erc20 |
| **Multi-bank consortium** | DSO governance with BFT voting | splice |
| **Cross-border** | Bridge pattern (fingerprint resolution) | canton-erc20 |

### Implementation Priority for Stablecoin MVP

**Phase 1 — Core Token:**
1. SimpleHolding with dual-sig (Pattern 1)
2. TransferFactory with 9-check validation pipeline (Pattern 2)
3. Mint/Burn with collateral pairing (Pattern 4)

**Phase 2 — Payments:**
4. TransferPreapproval for pre-authorized transfers (Pattern 8)
5. Subscription state machine for recurring payments (Pattern 7)
6. Multi-receiver payments for batch disbursement (Pattern 17)

**Phase 3 — Settlement:**
7. DvP settlement for PvP cross-currency (Pattern 3)
8. Async request queuing for batch processing (Pattern 16)
9. Settlement lifecycle for instrument management (Pattern 15)

**Phase 4 — Infrastructure:**
10. Governance for multi-bank consortium (Pattern 11)
11. Bridge for cross-chain/cross-ledger (Pattern 10)
12. Audit logging for regulatory compliance (Pattern 14)

---

*Extracted from audit of 5 DAML reference projects — 2026-03-04*
