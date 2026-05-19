# Verification Report: Stablecoin CDP System

Three open-source verification tools and manual code review were used to analyze this codebase. All findings have been patched or acknowledged.

## Summary

| Metric | Value |
|--------|-------|
| Tools used | 3 (daml-lint, daml-props, daml-verify) |
| Findings identified | 3 (production) + 8 (test, false positives) |
| Patched | 1 (L1 -- oracle price guard) |
| Acknowledged | 2 (L2, L3 -- false positives on guarded divisions) |
| Test-only findings | 8 (all false positives on comment arithmetic) |
| Formal properties proved | 14/14 (9 simple-token + 5 vault) |
| Property-based tests passed | 5/5 (200 random sequences each) |
| Functional tests passed | 22/22 V1 CDP tests |
| Experimental CIP-0112 script probes passed | 17/17 non-release probes |

## 2026-05-19 Live V2 Settlement Closeout

This closeout reconciles the audit notes with the compiled experimental live
V2 settlement prototype. It is non-release, non-production, and
non-conformant.

### Behavior Checked

| Surface | Closeout status |
|---|---|
| Close settlement | `Vault_Cip112CloseWithSettlement` is controlled by the vault owner, requires `settlementConfig.executor == admin`, creates accepted owner repayment and burn-account allocations, exercises live `SettlementFactory_SettleBatch`, returns collateral, and consumes the vault on success. |
| Liquidation settlement | `Vault_Cip112LiquidateWithSettlement` is controlled by `admin, liquidator`, requires admin as executor, materializes temporary seized collateral, creates accepted debt-burn and collateral-seizure allocations, settles them in one live batch, reports liquidator stablecoin change, owner collateral remainder, and bad debt. |
| Burn sink | Positive credits to the special `cip-112/burn` account are not materialized as admin-owned stablecoin holdings; the live close/liquidation tests assert admin stablecoin balance remains `0.0`. |
| Overpayment and change | Close overpayment returns authorizer V2 change and a V1 stablecoin balance for the owner; liquidation overpayment returns liquidator stablecoin change. |
| Locked inputs | Unexpired locked repayment inputs are rejected before settlement and leave the vault active. Expired-lock and stale allocation recovery remain covered by simple-token V2 tests, not by new stablecoin release claims. |
| Stale state | Successful close consumes the vault and stale withdraw fails. Failed underpayment and locked-input paths roll back and leave the vault active. |
| Provider-managed rejection | Provider-managed transfer-leg accounts are rejected by the local `SimpleTokenRules` settlement account guard. The current CDP path supports providerless basic accounts and special admin mint/burn accounts only. |
| Metadata | Experimental flow metadata uses `oz.cdp/cip112-flow`; prototype outputs also carry `oz.cdp/non-conformance = experimental`. |
| Privacy and disclosure | The prototype keeps private-asset defaults and uses `submitMulti`/limited rules disclosure in tests. It does not define a production disclosure policy, reduced-privacy mode, off-ledger publication, or Wallet SDK authority. |
| Archival | Successful settlement consumes the vault, input holdings used by allocation accept, locked allocation holdings, V2 allocations, and returns V1 compatibility outputs for collateral/remainder. Failed paths roll back atomically. |
| Upgrade | Preview V2 DARs and utility defaults remain volatile experimental inputs from the Splice preview branch. Re-baselining is required before any release or conformance work. |

### Tool Availability

| Tool/check | Closeout result |
|---|---|
| DPM | Available at `/Users/x/.dpm/bin/dpm`; `dpm --version` reported `1.0.10`; active SDK reported `3.4.11`; package configs use SDK `3.4.11`, LF `2.1`. |
| `scripts/verify.sh` | Passed with `2 passed, 0 failed`; it ran `simple-token-test` and `stablecoin-test`. It skipped `daml-lint` because neither `daml-lint` on `PATH` nor the stablecoin-local release binary was present, and skipped `daml-verify` because no stablecoin-local `tools/daml-verify/.venv/bin/python` exists. |
| Manual root `daml-lint` | `tools/daml-lint/target/debug/daml-lint tools/canton-stablecoin/stablecoin/daml --format markdown` was available and reported 2 high `unguarded-division` findings: the constant denominator in `accrueDebt` and the `collateralRatio` division guarded by the preceding `if debt == 0.0` branch. These are the existing L2/L3 false positives. |
| Manual root `daml-verify` | `tools/daml-verify/.venv/bin/python main.py` proved 14/14 properties. |

Remaining gaps: allocation requests, iterated settlement, provider-managed
account authority, cross-admin settlement, receipt/reporting standards,
production disclosure policy, LocalNet/client smoke paths, source-of-record and
public package pins, Wallet SDK authority, ChainSafe alignment, and formal
CIP-112 conformance tests.

---

## Tools Used

### daml-lint (Static Analysis)

Static analyzer for DAML that catches security anti-patterns through AST pattern matching. 6 detectors covering missing ensure clauses, unguarded division, missing positive-amount checks, archive-before-execute, head-of-list on queries, and unbounded fields.

**How it was run:**

```bash
cd daml-lint && cargo build --release

# Production code
daml-lint stablecoin/daml/ --format markdown

# Test code
daml-lint stablecoin-test/daml/ --format markdown
```

**Production code results (stablecoin):**

| Detector | Severity | File | Finding |
|----------|----------|------|---------|
| `unguarded-division` | HIGH | Vault.daml:232 | Division by `oracle.price` in `Vault_Liquidate` |
| `unguarded-division` | HIGH | Vault.daml:332 | Division by `intToDecimal microsPerYear` in `accrueDebt` |
| `unguarded-division` | HIGH | Vault.daml:340 | Division by `debt` in `collateralRatio` |

No MEDIUM or LOW findings. No `missing-ensure-decimal` (all Decimal fields have ensure clauses). No `unbounded-fields` (no unbounded List/Text fields on templates).

**Test code results (stablecoin-test):**

8 HIGH `unguarded-division` findings -- all false positives triggered by arithmetic in comments describing expected ratio calculations (e.g., `-- ratio: (10 * 2000) / 5000 = 4.0`). Test harness only. No production impact.

### daml-verify (Formal Verification)

Lightweight formal verification using Z3 SMT solver. Proves critical invariants hold for **all possible inputs**, not just sampled test cases. Extended from 9 to 14 properties with 5 vault-specific proofs.

**How it was run:**

```bash
cd daml-verify
source .venv/bin/activate
python main.py
```

**Results:**

```
daml-verify: 14 properties, 14 proved, 0 disproved

  [PROVED] C1: conservation total
  [PROVED] C2: receiver amount
  [PROVED] C3: sender change
  [PROVED] D1: scaleFees safety
  [PROVED] D2: issuance safety
  [PROVED] D3: ensure sufficient
  [PROVED] T1: transfer temporal
  [PROVED] T2: allocation temporal
  [PROVED] T3: lock expiry
  [PROVED] V1: fee monotonicity
  [PROVED] V2: collateral ratio guard
  [PROVED] V3: liquidation conservation
  [PROVED] V4: division safety (ratio)
  [PROVED] V5: division safety (seize)
```

### daml-props (Property-Based Testing)

Pure DAML property-based testing library with shrinking. Generates random action sequences, checks invariants hold for all, and shrinks failing cases to minimal counterexamples.

**How it was run:**

```bash
cd stablecoin-test && dpm build && dpm test
```

**Results:** All 5 vault property tests passed (200 random sequences each, up to 15-20 actions per sequence).

### Experimental CIP-0112 Live Settlement Probes

This audit now also records a non-release CIP-0112 discovery slice. The
prototype vendors preview V2 DARs from the Splice
`origin/token-standard-v2-daml-preview` branch at
`b91de5d4b910ded598151981654dce2acc6f84ba`, ports the local simple-token V2
allocation/finalization/settlement/event dependency, and exercises live CDP
close/liquidation settlement through `SimpleTokenRules`.

**Authority and visibility notes:**

| Surface | Current experimental behavior |
|---|---|
| Signatories | V1 `Vault` remains signed by `admin, owner`; V1 `SimpleHolding` outputs remain signed by `admin` and the holder. Local V2 allocation instructions/allocations are signed by `admin` and the authorizer principal. |
| Observers | V1 oracle/vault/holding visibility is unchanged. Local V2 allocations observe settlement executors; provider-managed observers are not introduced. |
| Controllers | `Vault_Cip112CloseWithSettlement` is controlled by `owner`; `Vault_Cip112LiquidateWithSettlement` by `admin, liquidator`; settlement actors are `[admin]`. |
| Disclosures | Tests use same-transaction `submitMulti` paths and limited rule-contract disclosure helpers. A production app disclosure policy is not implemented. |
| Privacy | Only providerless basic accounts and special admin mint/burn accounts are supported. Provider-managed settlement legs are rejected. |
| Archival | Successful live settlement consumes the vault, archives input/locked allocation holdings through the V2 dependency, consumes the V2 allocations, suppresses materialization of positive `cip-112/burn` credits, and creates compatibility V1 collateral/remainder outputs. Failed underpayment or unexpired-lock paths roll back. |
| Upgrade | Preview V2 DARs and helper defaults are volatile. This is not a package pin or conformance baseline. |

**Script coverage added:** live close overpayment/change/collateral return,
liquidation collateral seizure/change, bad-debt liquidation, underpayment
rollback, unexpired locked holding contention, provider-managed account
rejection, settlement metadata naming, and live-settlement support markers.
The full `stablecoin-test` package now passes 44 scripts: 22 V1 CDP tests, 5
vault property tests, and 17 experimental CIP-0112 probes.

---

## Findings

### L1: Missing explicit oracle price guard in Vault_Liquidate (Patched)

**Severity:** HIGH (per daml-lint), LOW (actual risk)
**Detector:** daml-lint `unguarded-division`
**Location:** Vault.daml:232
**Description:** `collateralToSeize = (accruedDebt * (1 + params.liquidationBonus)) / oracle.price` divides by `oracle.price` without a local guard. The `PriceOracle` template's `ensure price > 0.0` clause prevents zero prices at creation, but the linter cannot see cross-template guards.
**Fix:** Added explicit `assertMsg "Oracle price must be positive" (oracle.price > 0.0)` as defense-in-depth before the division.
**Status:** Patched. All 22 functional tests pass.

### L2: Division by constant in accrueDebt (Acknowledged)

**Severity:** HIGH (per daml-lint), FALSE POSITIVE (actual)
**Detector:** daml-lint `unguarded-division`
**Location:** Vault.daml:332
**Description:** `elapsedYears = intToDecimal elapsedMicros / intToDecimal microsPerYear` where `microsPerYear = 365 * 24 * 3600 * 1_000_000`. The denominator is a compile-time constant that evaluates to `31,536,000,000,000` -- always positive.
**Status:** Acknowledged. False positive. No fix needed.

### L3: Division guarded by if-then-else in collateralRatio (Acknowledged)

**Severity:** HIGH (per daml-lint), FALSE POSITIVE (actual)
**Detector:** daml-lint `unguarded-division`
**Location:** Vault.daml:340
**Description:** `else (collateral * price) / debt` is in the else branch of `if debt == 0.0 then 999999.0`. The linter cannot see the guard in the preceding line.
**Status:** Acknowledged. False positive. The guard on line 339 prevents division by zero.

---

## Vault Proofs (daml-verify)

These proofs establish that vault arithmetic is safe for all possible inputs.

| Property | Statement | Result |
|----------|-----------|--------|
| V1: fee monotonicity | `accrueDebt(d, r, t) >= d` when `d > 0, r >= 0, t >= 0` | PROVED |
| V2: collateral ratio guard | `collateral * price >= minRatio * debt` implies `collateralRatio >= minRatio` | PROVED |
| V3: liquidation conservation | `collateralToSeize + remainder == totalCollateral` (partial liquidation) | PROVED |
| V4: division safety (ratio) | `debt > 0` guards `collateralRatio` division | PROVED |
| V5: division safety (seize) | `price > 0` guards `collateralToSeize` division | PROVED |

**Model fidelity:** Symbolic models in `daml_verify/model/vault.py` were compared line-by-line to `Vault.daml:325-341` (helper functions) and `Vault.daml:212-291` (liquidation logic). All arithmetic relationships are exact.

## Conservation Proofs (daml-verify, inherited from simple-token)

| Property | Statement | Result |
|----------|-----------|--------|
| C1: conservation total | `totalInput == receiverAmount + senderChange` | PROVED |
| C2: receiver amount | Receiver gets exactly `transfer.amount` | PROVED |
| C3: sender change | `senderChange == totalInput - transfer.amount` | PROVED |

These apply to stablecoin transfers via `SimpleTokenRules` (proven by `test_mintAndTransfer`).

## Property-Based Testing Results (daml-props)

Pure state-machine model of the vault system. 7 action types (Deposit, Withdraw, Mint, Burn, CloseVault, UpdatePrice, Liquidate). Each test runs 200 random sequences.

| Test | Property | Sequences | Max Length | Result |
|------|----------|-----------|------------|--------|
| `test_vaultCollateralNonNegative` | Collateral never goes negative | 200 | 15 | PASS |
| `test_vaultDebtNonNegative` | Debt never goes negative | 200 | 15 | PASS |
| `test_vaultPricePositive` | Oracle price always > 0 | 200 | 15 | PASS |
| `test_vaultFullLifecycle` | All invariants hold under random operations | 200 | 20 | PASS |
| `test_vaultDebtImpliesCollateral` | Debt > 0 implies collateral > 0 | 200 | 15 | PASS |

**Methodology:** Executors return `Right state` (no-op) for invalid preconditions (insufficient funds, ratio breach), matching Echidna/Foundry semantics. `Left` is reserved for true invariant violations.

---

## Methodology

### Model Fidelity

All symbolic models and pure state-machine models were validated against actual DAML source code through line-by-line comparison:

- **daml-verify models** (`vault.py`): 3 symbolic functions modeling `accrueDebt`, `collateralRatio`, and `collateralToSeize`. All arithmetic relationships verified exact against Vault.daml.
- **daml-props models** (`VaultModel.daml`, ~130 LOC): Pure executor faithfully reproduces all 7 action types. Invalid preconditions return `Right state` (no-op). Liquidation model covers both partial (seize < total) and full (seize >= total) paths.

### Tool Versions

- daml-lint: built from source (`cargo build --release`)
- daml-props: v0.1.0 (pure DAML, SDK 3.4.10, target 2.1)
- daml-verify: Python 3.10+, z3-solver >= 4.12
- DAML SDK: 3.4.10
