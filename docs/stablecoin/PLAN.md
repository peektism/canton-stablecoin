# Plan: MakerDAO-esque Stablecoin CDP System

## Context

The `simple-token` module is a minimal CIP-056 token standard implementation with 7 templates, 36 passing tests, and 24 security invariants. It handles UTXO token holdings, transfers (3-way dispatch), allocations (DvP), and all 6 CIP-056 interfaces.

The `stablecoin` module extends that base to support MakerDAO-style Collateralized Debt Positions (CDPs). Users lock collateral tokens to mint stablecoins. The stablecoins are standard `SimpleHolding` contracts (same CIP-056 Holding interface), so they work with the existing transfer infrastructure out of the box.

**Design goal:** Smallest possible addition that captures the core MakerDAO mechanics (overcollateralization, minting, liquidation, stability fees) while reusing existing templates and patterns from `simple-token`.

## Project Structure

```
stablecoin/                           production contracts
  daml.yaml                           depends on simple-token DAR + splice DARs
  daml/Stablecoin/
    Oracle.daml                       PriceOracle template
    Vault.daml                        VaultParams, VaultFactory, Vault + helpers
    Experimental/Cip112/              non-release account-policy/settlement markers
stablecoin-test/                      test suite
  daml.yaml                           depends on stablecoin + simple-token + daml-props
  daml/Stablecoin/Test/
    Cdp.daml                          all CDP tests (happy path + negative)
    Cip112AccountPolicy.daml          experimental account-policy probes
    Cip112LiveSettlement.daml         experimental live V2 settlement probes
    Cip112SettlementDependency.daml   experimental V2 dependency probes
    VaultModel.daml                   pure state-machine model
    VaultProperties.daml              property-based tests
```

### Build Configuration

**`stablecoin/daml.yaml`:**
```yaml
sdk-version: 3.4.11
name: stablecoin
version: 0.1.0
source: daml
build-options:
  - --target=2.1
dependencies:
  - daml-prim
  - daml-stdlib
data-dependencies:
  - ../dars/splice-api-token-metadata-v1-1.0.0.dar
  - ../dars/splice-api-token-holding-v1-1.0.0.dar
  - ../dars/splice-api-token-transfer-instruction-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-instruction-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-request-v1-1.0.0.dar
  - ../simple-token/.daml/dist/simple-token-0.1.0.dar
```

**`stablecoin-test/daml.yaml`:**
```yaml
sdk-version: 3.4.11
name: stablecoin-test
version: 0.1.0
source: daml
build-options:
  - --target=2.1
dependencies:
  - daml-prim
  - daml-stdlib
  - daml-script
data-dependencies:
  - ../dars/splice-api-token-metadata-v1-1.0.0.dar
  - ../dars/splice-api-token-holding-v1-1.0.0.dar
  - ../dars/splice-api-token-transfer-instruction-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-instruction-v1-1.0.0.dar
  - ../dars/splice-api-token-allocation-request-v1-1.0.0.dar
  - ../simple-token/.daml/dist/simple-token-0.1.0.dar
  - ../stablecoin/.daml/dist/stablecoin-0.1.0.dar
  - ../dars/daml-props-0.1.0.dar
```

---

## What We Reuse from simple-token (no changes to that module)

- `SimpleHolding` for both collateral tokens and stablecoins (different `InstrumentId`)
- `SimpleTokenRules` for stablecoin transfers (works for any registered instrument)
- `archiveAndSumInputs` validation pattern (replicated as `archiveAndSumCollateral`)
- `emptyMetadata`, UTXO consume-and-create pattern, `InstrumentId` type
- Test helpers: `SimpleRegistry`, `WalletClient`, `TestParties` (imported from simple-token-test DAR)

---

## Templates

### 1. PriceOracle (`Stablecoin/Oracle.daml`)

Admin-controlled price feed. Price = stablecoin units per 1 unit of collateral (e.g., 2000.0 means 1 collateral = 2000 stablecoin).

```
template PriceOracle
  with
    admin : Party
    collateralInstrumentId : InstrumentId
    price : Decimal
    updatedAt : Time
  where
    signatory admin
    ensure price > 0.0

    choice PriceOracle_UpdatePrice : ContractId PriceOracle
      with newPrice : Decimal
      controller admin
      -- validates newPrice > 0.0, sets updatedAt = now
```

### 2. VaultParams (data type in `Stablecoin/Vault.daml`)

```
data VaultParams = VaultParams with
    minCollateralRatio : Decimal    -- e.g., 1.5 (150%). Required for minting/withdrawing.
    liquidationRatio : Decimal      -- e.g., 1.3 (130%). Below this, vault is liquidatable.
    liquidationBonus : Decimal      -- e.g., 0.13 (13%). Extra collateral to liquidator.
    stabilityFeeRate : Decimal      -- e.g., 0.02 (2% annual). Linear accrual on debt.
```

### 3. VaultFactory (`Stablecoin/Vault.daml`)

Nonconsuming factory for creating vaults. Follows the `SimpleTokenRules` pattern.

```
template VaultFactory
  with
    admin : Party
    collateralInstrumentId : InstrumentId
    stablecoinInstrumentId : InstrumentId
    params : VaultParams
  where
    signatory admin
    ensure minCollateralRatio > 1.0
        && liquidationRatio > 1.0
        && liquidationRatio <= minCollateralRatio
        && liquidationBonus >= 0.0
        && stabilityFeeRate >= 0.0
        && collateralInstrumentId.admin == admin
        && stablecoinInstrumentId.admin == admin

    nonconsuming choice VaultFactory_OpenVault : ContractId Vault
      with
        owner : Party
        collateralHoldingCids : [ContractId Holding]
      controller admin, owner
      -- archives collateral holdings, creates Vault with debt=0
```

### 4. Vault (`Stablecoin/Vault.daml`)

The CDP itself. All mutations are consuming (UTXO). Embeds its own params.

```
template Vault
  with
    admin : Party
    owner : Party
    collateralInstrumentId : InstrumentId
    stablecoinInstrumentId : InstrumentId
    collateralAmount : Decimal
    debtAmount : Decimal
    params : VaultParams
    lastAccrualTime : Time
  where
    signatory admin, owner
    ensure collateralAmount > 0.0 && debtAmount >= 0.0
```

**Choices (all consuming):**

| Choice | Controller | What it does |
|--------|-----------|-------------|
| `Vault_DepositCollateral` | `owner` | Archives collateral inputs, accrues fees, creates new vault with higher collateral |
| `Vault_WithdrawCollateral` | `owner` | Fetches oracle, accrues fees, checks ratio >= `minCollateralRatio`, creates new vault + collateral holding for owner |
| `Vault_MintStablecoin` | `owner` | Fetches oracle, accrues fees, checks ratio >= `minCollateralRatio`, creates new vault + stablecoin holding for owner |
| `Vault_BurnStablecoin` | `owner` | Archives stablecoin inputs, accrues fees, creates new vault with lower debt |
| `Vault_Close` | `owner` | Requires full debt repayment (with accrued fees), returns all collateral to owner |
| `Vault_Liquidate` | `admin, liquidator` | Validates vault is undercollateralized, liquidator provides stablecoin, receives collateral at discount, remaining collateral returned to owner |

**Experimental non-release CIP-0112 choices:**

| Choice | Controller | What it does |
|--------|-----------|-------------|
| `Vault_Cip112CloseWithSettlement` | `owner` | Requires admin as settlement executor, creates and accepts V2 allocations for owner repayment and admin burn receipt, exercises live `SettlementFactory_SettleBatch`, returns collateral, and reports V2 repayment change. |
| `Vault_Cip112LiquidateWithSettlement` | `admin, liquidator` | Requires admin as settlement executor, materializes temporary seized collateral, creates and accepts V2 allocations for debt burn and collateral seizure, exercises live batch settlement, reports liquidator change and bad debt, and returns any owner collateral remainder. |

These choices are local discovery code only. They keep V1 behavior intact,
support providerless basic accounts plus special `cip-112/mint` and
`cip-112/burn` admin accounts, reject provider-managed settlement legs, and do
not make CIP-0112 conformance or release-package claims.

### Authorization Analysis

| Choice | Authorizers | Can create holdings for |
|--------|------------|------------------------|
| Owner choices | admin (vault sig) + owner (controller) | owner: yes |
| `Vault_Liquidate` | admin (sig+controller) + owner (sig) + liquidator (controller) | liquidator: yes, owner: yes |

The liquidation authorization works because DAML's authorizer set = vault signatories {admin, owner} + choice controllers {admin, liquidator} = {admin, owner, liquidator}. This lets us create `SimpleHolding` for both the liquidator (needs admin+liquidator) and the owner (needs admin+owner).

---

## Helper Functions

```
-- Replicates archiveAndSumInputs from Rules.daml for vault context
archiveAndSumCollateral : Party -> InstrumentId -> [ContractId Holding] -> Update Decimal

-- Linear stability fee: newDebt = oldDebt * (1 + rate * elapsedYears)
accrueDebt : Decimal -> Time -> Time -> Decimal -> Decimal

-- ratio = (collateralAmount * price) / debt. Returns 999999.0 if debt == 0.
collateralRatio : Decimal -> Decimal -> Decimal -> Decimal
```

### Stability Fee

Linear accrual on-demand at each vault operation. Every choice calls `accrueDebt` first, then sets `lastAccrualTime = now`.

```
elapsedYears = microseconds(now - lastAccrualTime) / microseconds(365 days)
newDebt = oldDebt * (1 + annualRate * elapsedYears)
```

### Liquidation Flow

1. Price drops so `collateralRatio < liquidationRatio`
2. Liquidator calls `Vault_Liquidate` providing stablecoin to cover debt
3. `collateralToSeize = (accruedDebt * (1 + liquidationBonus)) / price`
4. If `collateralToSeize >= collateralAmount`: liquidator gets all collateral, `badDebt` recorded
5. Otherwise: liquidator gets seized portion, owner gets remainder
6. Vault is always fully consumed

### VaultLiquidationResult

```
data VaultLiquidationResult = VaultLiquidationResult with
    liquidatorCollateralCid : ContractId SimpleHolding
    liquidatorChangeCid : Optional (ContractId SimpleHolding)  -- excess stablecoin
    ownerCollateralCid : Optional (ContractId SimpleHolding)   -- remaining collateral
    badDebt : Decimal                                          -- unrecoverable debt
```

---

## Test Plan

Core V1 CDP tests live in `stablecoin-test/daml/Stablecoin/Test/Cdp.daml`.
Experimental non-release CIP-0112 probes live in
`Cip112AccountPolicy.daml`, `Cip112SettlementDependency.daml`, and
`Cip112LiveSettlement.daml`.

### Setup

```
data CdpTestEnv -- parties, instrumentIds, factoryCid, oracleCid
setupCdpTestEnv : Script CdpTestEnv
-- VaultFactory: minRatio=1.5, liqRatio=1.3, bonus=0.13, fee=0.02
-- PriceOracle: price=2000.0
```

### Happy Path (12 tests)

| # | Test | Validates |
|---|------|-----------|
| 1 | `test_openVault` | Deposit 10 collateral, vault created with collateral=10, debt=0 |
| 2 | `test_mintStablecoin` | Mint 10000 stablecoin (safe at 150% ratio) |
| 3 | `test_burnStablecoin` | Mint then burn 5000, debt reduced |
| 4 | `test_depositCollateral` | Add 5 more collateral |
| 5 | `test_withdrawCollateral` | Mint 5000, withdraw 2 collateral (ratio stays safe) |
| 6 | `test_closeVaultZeroDebt` | Open vault, close immediately, all collateral returned |
| 7 | `test_closeVaultWithDebt` | Mint, close by repaying full debt |
| 8 | `test_liquidation` | Mint near max, drop price, liquidator liquidates |
| 9 | `test_liquidationBadDebt` | Catastrophic price drop, liquidator gets all, badDebt > 0 |
| 10 | `test_stabilityFeeAccrual` | Mint 10000, advance 1 year, debt ~10200 (2% fee) |
| 11 | `test_oracleUpdate` | Update oracle price |
| 12 | `test_mintAndTransfer` | Mint stablecoin, transfer via existing SimpleTokenRules |

### Negative Tests (9 tests)

| # | Test | Invariant |
|---|------|-----------|
| 13 | `test_mintExceedsRatio` | Can't mint below 150% ratio |
| 14 | `test_withdrawExceedsRatio` | Can't withdraw below 150% ratio |
| 15 | `test_burnExceedsDebt` | Can't burn more than debt |
| 16 | `test_liquidateHealthyVault` | Can't liquidate above liquidation ratio |
| 17 | `test_liquidateUnauthorized` | Non-admin can't trigger liquidation alone |
| 18 | `test_wrongOracleAdmin` | Oracle from different admin rejected |
| 19 | `test_wrongOracleInstrument` | Oracle for wrong collateral rejected |
| 20 | `test_wrongOwnerCollateral` | Can't deposit someone else's collateral |
| 21 | `test_closeInsufficientRepayment` | Can't close without covering full debt |

### Experimental CIP-0112 Probes (17 tests)

These tests do not claim conformance. They document the local prototype
boundary and keep V1 behavior covered while exercising preview V2 dependencies.

| Group | Tests | Validates |
|---|---|---|
| Account policy | 9 | Withdrawn, minted, burn, close-change, provider-visibility, and liquidation outputs remain basic-account by default; mint/burn use special admin account ids; stale V1 vault exercises fail. |
| Dependency markers | 2 | Liquidation settle-batch arguments compile and the local live-settlement support markers are enabled while non-release blockers remain explicit. |
| Live settlement | 6 | Close overpayment/change/collateral return, liquidation collateral seizure/change, bad-debt liquidation, underpayment rollback, unexpired-lock contention, provider-managed account rejection, and metadata naming. |

---

## Implementation Sequence (TDD)

Following RED-GREEN-REFACTOR:

1. **Project setup**: Create `daml.yaml` files, verify `dpm build` works with empty modules
2. **Oracle**: `test_oracleUpdate` (RED) -> `PriceOracle` (GREEN) -> refactor
3. **Helpers + OpenVault**: `test_openVault` (RED) -> `VaultParams`, `VaultFactory`, `Vault`, `archiveAndSumCollateral` (GREEN) -> refactor
4. **Mint**: `test_mintStablecoin` + `test_mintExceedsRatio` (RED) -> `Vault_MintStablecoin` + `collateralRatio` (GREEN) -> refactor
5. **Burn**: `test_burnStablecoin` + `test_burnExceedsDebt` (RED) -> `Vault_BurnStablecoin` (GREEN) -> refactor
6. **Deposit**: `test_depositCollateral` + `test_wrongOwnerCollateral` (RED) -> `Vault_DepositCollateral` (GREEN) -> refactor
7. **Withdraw**: `test_withdrawCollateral` + `test_withdrawExceedsRatio` (RED) -> `Vault_WithdrawCollateral` (GREEN) -> refactor
8. **Close**: `test_closeVaultZeroDebt` + `test_closeVaultWithDebt` + `test_closeInsufficientRepayment` (RED) -> `Vault_Close` (GREEN) -> refactor
9. **Liquidation**: `test_liquidation` + `test_liquidationBadDebt` + `test_liquidateHealthyVault` + `test_liquidateUnauthorized` (RED) -> `Vault_Liquidate` (GREEN) -> refactor
10. **Integration**: `test_stabilityFeeAccrual` + `test_mintAndTransfer` + oracle negative tests (RED) -> verify (GREEN) -> final refactor

---

## Security Invariants

| # | Invariant | Location | Enforced By |
|---|-----------|----------|-------------|
| S1 | `collateralAmount > 0.0` | Vault ensure | Template ensure clause |
| S2 | `debtAmount >= 0.0` | Vault ensure | Template ensure clause |
| S3 | `price > 0.0` | PriceOracle ensure | Template ensure clause |
| S4 | `newPrice > 0.0` | PriceOracle_UpdatePrice | assertMsg |
| S5 | `minCollateralRatio > 1.0` | VaultFactory ensure | Template ensure clause |
| S6 | `liquidationRatio > 1.0` | VaultFactory ensure | Template ensure clause |
| S7 | `liquidationRatio <= minCollateralRatio` | VaultFactory ensure | Template ensure clause |
| S8 | `liquidationBonus >= 0.0` | VaultFactory ensure | Template ensure clause |
| S9 | `stabilityFeeRate >= 0.0` | VaultFactory ensure | Template ensure clause |
| S10 | Mint/withdraw maintain `ratio >= minCollateralRatio` | Vault_MintStablecoin, Vault_WithdrawCollateral | assertMsg |
| S11 | Liquidation requires `ratio < liquidationRatio` | Vault_Liquidate | assertMsg |
| S12 | Input holding owner match | archiveAndSumCollateral | assertMsg per-input |
| S13 | Input holding instrumentId match | archiveAndSumCollateral | assertMsg per-input |
| S14 | Input holding admin match | archiveAndSumCollateral | assertMsg per-input |
| S15 | Unexpired lock rejection | archiveAndSumCollateral | assertMsg per-input |
| S16 | Oracle admin match | Vault choices (Withdraw, Mint, Liquidate) | assertMsg |
| S17 | Oracle instrument match | Vault choices (Withdraw, Mint, Liquidate) | assertMsg |
| S18 | Oracle price > 0 | Vault_Liquidate | assertMsg (defense-in-depth) |
| S19 | Burn amount <= debt | Vault_BurnStablecoin | assertMsg |
| S20 | Close requires full debt repayment | Vault_Close | assertMsg |
| S21 | Admin+owner auth on VaultFactory_OpenVault | VaultFactory | controller admin, owner |
| S22 | Admin+liquidator auth on Vault_Liquidate | Vault | controller admin, liquidator |

## Acceptance Criteria

| Template/Choice | Criterion |
|---|---|
| PriceOracle | Admin can create with positive price; update replaces price and timestamp |
| VaultFactory_OpenVault | Archives input holdings, creates vault with correct collateral and debt=0 |
| Vault_DepositCollateral | Increases collateral, accrues fees, archives inputs |
| Vault_WithdrawCollateral | Decreases collateral, creates holding, maintains ratio when debt > 0 |
| Vault_MintStablecoin | Creates stablecoin holding, increases debt, maintains ratio |
| Vault_BurnStablecoin | Archives stablecoin inputs, decreases debt, debt >= 0 |
| Vault_Close | Returns all collateral, requires full repayment, handles overpayment |
| Vault_Liquidate | Validates undercollateralization, seizes correct collateral, handles partial and full cases, returns change and remainder |
| Vault_Cip112CloseWithSettlement | Experimental only: live V2 repayment burn settles through `SettlementFactory_SettleBatch`, overpayment change is returned, collateral is returned, and failed underpayment/lock paths roll back |
| Vault_Cip112LiquidateWithSettlement | Experimental only: live V2 debt burn and collateral seizure settle in one batch, change/remainder/bad-debt reporting is explicit, and provider-managed accounts remain rejected |
| Integration | Minted stablecoins transfer via SimpleTokenRules |

---

## What This Deliberately Excludes (V2 candidates)

See [SCOPE.md](SCOPE.md) for detailed analysis.

- **Governance / multi-sig**: Admin unilaterally controls params and oracle
- **Auction mechanism**: Liquidation is direct, not Dutch/English auction
- **Global debt ceiling**: No system-wide cap on total stablecoin supply
- **Emergency shutdown**: No mechanism to freeze all vaults
- **Savings rate (DSR)**: No interest paid on stablecoin holders
- **Oracle staleness check**: No TTL on oracle prices
- **Multi-collateral per vault**: One VaultFactory per collateral type

## Verification

1. `dpm build` from `stablecoin/` (new modules compile)
2. `JAVA_HOME=... dpm test` from `stablecoin-test/` (44 scripts pass -- 22 V1 CDP, 5 property-based, 17 experimental CIP-0112)
3. `JAVA_HOME=... dpm test` from `simple-token-test/` (existing V1 tests plus experimental V2 transfer/allocation probes pass, no regressions)
4. Verify stablecoin interop: minted stablecoin transfers via `SimpleTokenRules` in `test_mintAndTransfer`
5. `daml-lint` static analysis: historical report has 1 finding patched and
   2 false positives acknowledged; 2026-05-19 closeout manual root
   `daml-lint` still reports only those 2 stablecoin false positives, while
   `scripts/verify.sh` skips lint when it is not installed on its own search
   path (see [AUDIT.md](AUDIT.md)).
6. `daml-verify` formal proofs: 14/14 proved (9 simple-token + 5 vault) when
   run manually from the workspace root tool venv; `scripts/verify.sh` skips
   formal verification unless a stablecoin-local `tools/daml-verify` venv is
   installed.
7. `daml-props` property tests: 5/5 passed (200 random sequences each)
