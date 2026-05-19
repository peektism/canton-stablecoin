# CIP-0112 Extension Plan: canton-stablecoin

Date: 2026-05-19
Status: Discovery / experimental implementation; narrow Daml source changes
are allowed when clearly marked non-release
Owner: OpenZeppelin technical lead
Reviewer: Digital Asset technical contact, CIP-0112 authors, OpenZeppelin
security lead (for CDP authorization model implications)

## Scope

This is the per-tool extension plan for adding CIP-0112 (Token Standard V2)
V2 interfaces to the MakerDAO-style CDP stablecoin in
`tools/canton-stablecoin/stablecoin/`, on top of the underlying
CIP-0056 token registry reused from `simple-token/`.

Parents:

- Workspace plan: `/Users/x/canton/docs/architecture/cip-112-extension-plan.md`
- Sibling per-tool plan:
  `/Users/x/canton/tools/canton-token-template/docs/CIP-0112-EXTENSION-PLAN.md`

This plan is **discovery plus experimental implementation**. Daml source in
`stablecoin/` or `stablecoin-test/` may be modified in small, reversible
prototype slices when the local change can either compile against the current
simple-token base or record the exact missing token-template dependency it
needs. Prototype code must not be presented as CIP-0112-conformant library or
RI code.

## Why the stablecoin is special

Per `/Users/x/canton/CLAUDE.md` and the existing
`canton-stablecoin/docs/SCOPE.md`:

> `canton-stablecoin` reuses `simple-token` unchanged: minted stablecoins
> are CIP-056 holdings that transfer through `SimpleTokenRules`. Don't
> fork the token module to add CDP behavior.

The CDP layer therefore does not own the on-ledger transfer/allocation
flows; those flow through the simple-token `SimpleTokenRules`. The CDP
layer adds vault creation, mint, burn, repay, withdraw, and liquidation
choices. **All V2 changes to transfer/allocation semantics are inherited
from the simple-token extension plan; the CDP plan focuses on what the
CDP layer must add or change to remain consistent.**

## CIP-0112 surfaces with direct CDP impact

| CIP-0112 surface | CDP impact |
| :--- | :--- |
| `Account` replaces `Party` (§4.3.2) | The vault keeper / admin is a natural `provider` for vault-collateralised holdings. The CDP must decide: is the vault owner's stablecoin Holding `account.provider = admin` (admin co-authorises movements) or `account.provider = None` (basic account with no V2 provider authority; existing V1 admin visibility remains a signatory consequence)? |
| Mint/burn account-id conventions (§4.3.2.1) | Stablecoin mint and burn flows in the CDP MUST use `account-id = cip-112/mint` and `cip-112/burn` for the admin-side transfer leg so wallets present consistent UX. |
| Configurable `actors` on choices (§4.3.2.3) | CDP-specific choices (`CreateVault`, `Mint`, `Burn`, `Repay`, `Withdraw`, `Liquidate`) need `actors : [Party]` if they exercise V2 `TransferFactory_Transfer` or `AllocationFactory_Allocate`. |
| `availableActions` map on instructions (§4.3.2) | CDP-issued mint transfers (vault → owner) carry the new map; CDP-issued burn transfers (owner → admin with `cip-112/burn` account-id) similarly. |
| `SettlementFactory_SettleBatch` (§4.3.1) | Liquidation flows are natural candidates for multi-leg settlement (collateral -> liquidator, debt -> burn/admin, fees -> admin in one transaction). This slice now exercises live preview V2 settlement for close and liquidation through the locally ported `SimpleTokenRules` V2 prototype. |
| `splice-token-standard-utils` upcast/downcast (§5.3) | Stablecoin now vendors the preview utils DAR and uses the local simple-token V2 helper shape for upcasts, finalized allocations, event logging, and settlement defaults. This is still a volatile experimental dependency, not a release pin. |

## Module-by-module V2 mapping

Current source inventory:

| Module | Templates/helpers | CIP-112 account impact |
| :--- | :--- | :--- |
| `stablecoin/daml/Stablecoin/Oracle.daml` | `PriceOracle`, `PriceOracle_UpdatePrice` | No token holding is created or consumed. Oracle visibility remains the `observers` field; this slice adds no Account semantics. |
| `stablecoin/daml/Stablecoin/Vault.daml` | `VaultFactory`, `Vault`, V1 helpers, `Cip112SettlementConfig`, `Vault_Cip112CloseWithSettlement`, `Vault_Cip112LiquidateWithSettlement`, live V2 allocation helpers | Existing V1 CDP token movement is unchanged. The new choices are explicitly experimental and create/accept local V2 allocations, call live `SettlementFactory_SettleBatch`, then create V1 collateral/remainder outputs. |
| `stablecoin/daml/Stablecoin/Experimental/Cip112/AccountPolicy.daml` | `Cip112Account` projection, mint/burn account-id constants, CDP account-policy helpers, live-settlement support markers | Experimental policy surface. It mirrors the preview `Account` shape (`owner : Optional Party`, `provider : Optional Party`, `id : Text`) and records that live batch settlement is available locally while release/conformance/provider decisions remain blocked. |
| `stablecoin/daml/Stablecoin/Experimental/Cip112/LiquidationSettlement.daml` | Preview V2 `HoldingV2.Account` helpers, liquidation debt/collateral `TransferLeg` builders, `SettlementFactory_SettleBatch` argument builder, cancellation/withdrawal support markers | Experimental shape and blocker surface. It imports the real preview V2 DARs and records the live-settlement path, while iterated settlement, receipts, provider-managed accounts, and conformance remain blocked. |

`tools/canton-stablecoin/stablecoin-test/daml/Stablecoin/Test/` now includes
`Cip112AccountPolicy.daml`, which exercises existing V1 CDP flows and asserts
the account-policy projection, `Cip112SettlementDependency.daml`, which keeps
the preview V2 settle-batch argument shape compiled, and
`Cip112LiveSettlement.daml`, which exercises the local live V2 close and
liquidation paths. These tests do **not** claim CIP-0112 conformance.

## Current V2 Dependency Boundary

Stablecoin now vendors six preview V2 DARs copied from the staged
`tools/canton-token-template` prototype, whose provenance is documented in the
sibling extension plan as `/Users/x/excanton/CN/splice`
`origin/token-standard-v2-daml-preview` commit
`b91de5d4b910ded598151981654dce2acc6f84ba`:

| Local DAR | Why it is present in stablecoin | SHA-256 |
| :--- | :--- | :--- |
| `dars/splice-api-token-holding-v2-1.0.0.dar` | Provides the preview `Account` type used by V2 transfer legs. | `156a5d78659abbf9664cac3ece97338afa9fd1c2a4a72588ea89fc09e78ebea9` |
| `dars/splice-api-token-allocation-v2-1.0.0.dar` | Provides `TransferLeg`, `SettlementInfo`, `FinalizedAllocation`, and `SettlementFactory_SettleBatch` argument/result types. | `b023bd40cdf0ed9189e6819f94225fa05ed86b4276b647cb450aa6979b65b801` |
| `dars/splice-api-token-allocation-instruction-v2-1.0.0.dar` | Provides allocation instruction factory and accept/withdraw choices used to create finalized V2 allocations before settlement. | `e1de9d448bd3c67d68bb50c73e51d4e0bbe53d6e8f5f9662c992f7c609cfd9aa` |
| `dars/splice-api-token-transfer-instruction-v2-1.0.0.dar` | Required by the local simple-token V2 dependency graph and future transfer probes; stablecoin does not yet exercise V2 transfer instructions directly. | `b52837469d6aae7d7bddb9ca3edc4b8ebe6021c5a841880666aab2fc913b42fc` |
| `dars/splice-api-token-transfer-events-v2-1.0.0.dar` | Provides the V2 event log interface used by the local settlement factory defaults. | `b865117fc61b1bdb46756dcaa2d2330822e62e7a925c9bec49805e71dd0bddda` |
| `dars/splice-token-standard-utils-2.0.0.dar` | Supplies preview utility/default implementations such as upcasts and allocation/settlement helper logic. | `e8735eb88f9557a3438098a6608dfa675ac957cac9038567ab790fdea83b2e07` |

The stablecoin package still does **not** vendor
`splice-api-token-allocation-request-v2` because this slice creates
authorizer-side allocations directly and does not expose a V2 allocation
request workflow. That is the remaining narrow boundary: stablecoin can
create/accept/finalize local V2 allocations, exercise live
`SettlementFactory_SettleBatch`, cancel/withdraw simple-token V2 allocations
through the ported dependency, and emit settlement events through the local
event log. It does not implement allocation requests, iterated settlement,
provider-managed accounts, public package pins, or conformance tests.

CDP choice mapping:

| Choice/helper | Underlying token/simple-token flow | Account-policy result in this slice |
| :--- | :--- | :--- |
| `VaultFactory_OpenVault` | Consumes owner collateral `Holding` CIDs through `archiveAndSumCollateral`; creates a `Vault` with `collateralAmount`. | Collateral is no longer a live Holding while in the vault. If projected, the default stays owner `basicAccount`. Admin/provider authority is not introduced because the `Vault` is already signed by `admin, owner`. |
| `Vault_DepositCollateral` | Consumes additional owner collateral `Holding` CIDs through `archiveAndSumCollateral`; creates replacement `Vault`. | Same as open-vault collateral: no live V2 Holding exists while collateral is in the vault. |
| `Vault_WithdrawCollateral` | Creates a new owner `SimpleHolding` for withdrawn collateral. | Withdrawn holdings project to owner `basicAccount` by default; provider remains `None`. Covered by `test_cip112WithdrawnHoldingsStayBasicAccount`. |
| `Vault_MintStablecoin` | Creates a new owner `SimpleHolding` for minted stablecoin. | Minted stablecoin output projects to owner `basicAccount`; the admin-side mint source leg is represented by special account id `cip-112/mint`. Covered by `test_cip112MintBurnUseSpecialAdminAccounts`. |
| `Vault_BurnStablecoin` | Consumes owner stablecoin `Holding` CIDs through `archiveAndSumCollateral`; creates replacement `Vault`. | Burn inputs project to owner `basicAccount`; the admin-side burn sink leg is represented by special account id `cip-112/burn`. Covered by `test_cip112MintBurnUseSpecialAdminAccounts`. |
| `Vault_Close` | Optionally consumes stablecoin repayment, may create stablecoin change, and creates owner collateral `SimpleHolding`. | V1 behavior is preserved. Repayment burn uses the same `cip-112/burn` convention in the V2 prototype; collateral and change outputs stay owner `basicAccount`. `test_cip112CloseOverpaymentReturnsBasicAccountChange` covers V1 overpayment change. `test_cip112CloseArchivesVaultAndStaleWithdrawFails` covers stale-state archival. |
| `Vault_Cip112CloseWithSettlement` | Experimental close path. Reads owner stablecoin input holdings, creates and accepts a sender allocation from owner basic account to burn account plus a burn receiver allocation, exercises live `SettlementFactory_SettleBatch`, then creates owner collateral output. | Controller is `owner`; the settlement executor must be `admin`. Input authorizer is owner basic account; burn authorizer is special `cip-112/burn` account whose principal resolves to admin, but settlement treats that special account as a sink and does not materialize admin-owned stablecoin holdings. Overpayment change is returned through V2 allocation accept results; the V1 `Vault` is consumed only if the whole transaction succeeds. Covered by `test_cip112CloseWithLiveSettlementReturnsCollateralAndChange`, `test_cip112CloseUnderpaymentRollsBackVault`, and `test_cip112LiveSettlementRejectsCollateralLockedRepayment`. |
| `Vault_Liquidate` | Consumes liquidator stablecoin, creates liquidator collateral, optional liquidator stablecoin change, optional owner collateral remainder. | V1 behavior is preserved. Liquidation outputs stay basic-account in the current V1 flow. Covered by `test_cip112LiquidationOutputsStayBasicAndBatchSettlementLive`, `test_cip112LiquidationOverpaymentChangeAndStaleVault`, and the V1 CDP suite. |
| `Vault_Cip112LiquidateWithSettlement` | Experimental liquidation path. Materializes seized vault collateral as a temporary owner holding, creates/accepts four V2 allocations for debt repayment, burn receipt, collateral sender, and collateral receiver, exercises live `SettlementFactory_SettleBatch`, then creates any owner remainder. | Controllers are `admin, liquidator`; the consumed `Vault` contributes owner authority for the temporary collateral and the settlement executor must be `admin`. Debt leg is `basicAccount liquidator -> cip-112/burn`; collateral leg is `basicAccount owner -> basicAccount liquidator`. Bad debt and change are reported in the choice result. Covered by `test_cip112LiquidationWithLiveSettlementSeizesCollateralAndReturnsChange` and `test_cip112LiquidationWithLiveSettlementRecordsBadDebt`. |
| `archiveAndSumCollateral` | Fetches V1 `Holding`, validates owner/instrument/admin, rejects unexpired locks, archives inputs, sums amounts. | This is the only CDP helper that accepts token inputs. It does not understand V2 `Account` or provider-managed holdings in this repo. |

## Settlement Assumptions

`Stablecoin.Experimental.Cip112.LiquidationSettlement` remains a lightweight
dependency/shape marker, while `Vault_Cip112CloseWithSettlement` and
`Vault_Cip112LiquidateWithSettlement` exercise live local V2 allocation and
settlement paths. The following assumptions are explicit for this non-release
prototype:

| Surface | Current assumption |
| :--- | :--- |
| Signatories | Existing V1 `Vault` contracts remain signed by `admin, owner`; existing `SimpleHolding` outputs remain signed by `admin` and the affected owner/liquidator. Local V2 allocation instructions and allocations are signed by `admin` and the allocation authorizer principal: owner/liquidator for basic accounts and admin for special mint/burn accounts. |
| Observers | Existing V1 visibility is unchanged: `PriceOracle.observers` controls oracle reads, `Vault` visibility comes from `admin, owner`, and `SimpleHolding` visibility comes from `admin` plus the holder. Local V2 allocation instructions/allocations observe settlement executors. The event log is the `SimpleTokenRules` contract itself, visible to admin. |
| Controllers | Existing V1 CDP controllers are unchanged. Experimental `Vault_Cip112CloseWithSettlement` is controlled by `owner` and requires `settlementConfig.executor == admin`. Experimental `Vault_Cip112LiquidateWithSettlement` is controlled by `admin, liquidator` and also requires admin as settlement executor. Allocation accept is performed by the authorizer principal; settlement actors are `[admin]`. |
| Disclosures | Tests pass the rules/factory disclosure to submitters for ergonomics, but successful paths use `submitMulti` because the participants act together in the script. A real multi-party app still needs an explicit disclosure plan for the rules contract and any allocation CIDs visible across submitters. |
| Privacy | The live prototype keeps providerless basic accounts for owners and liquidators and special admin mint/burn accounts. It does not opt into reduced-privacy/public-asset observers and rejects provider-managed settlement legs through the local account guard. V1 admin visibility on holdings remains a signatory consequence, not a V2 provider authority claim. |
| Authorization | Close repayment is represented as `basicAccount owner -> cip-112/burn`. Liquidation repayment is `basicAccount liquidator -> cip-112/burn`; collateral seizure is `basicAccount owner -> basicAccount liquidator`. Owner-side seizure authority comes from the consumed `Vault` signed by owner plus the experimental creation of a temporary owner collateral holding inside the liquidation transaction. |
| Archival | Successful live settlement consumes the `Vault`, consumes accepted V2 allocations, archives locked allocation holdings, archives input holdings used by allocation accept, and creates V2 authorizer change holdings as needed. Positive credits to the special `cip-112/burn` account are intentionally not materialized as admin-owned holdings. The CDP choices then create V1 `SimpleHolding` collateral/remainder outputs for compatibility. Failed underpayment or locked-input paths roll back the whole transaction and leave the `Vault` active. |
| Stale state | V1 stale-vault probes still prove consumed vault CIDs cannot be re-exercised. Live close and negative-path tests also confirm failed V2 settlement preparation leaves the CDP state active. Direct stale V2 allocation-CID replay/cancel/withdraw tests remain delegated to the local simple-token V2 tests. |
| Upgrade | The six preview V2 DARs and the copied simple-token V2 source are experimental inputs from the token-template prototype. Any source-of-record, package-name, field-shape, or utility-default churn requires re-baselining this module before release work. |
| Blockers | Allocation requests, iterated settlement, provider-managed account authority, wallet/off-ledger metadata publication, receipt/reporting standards, cross-admin settlement, and public package pins are not implemented. |
| Non-conformance | The module is not a CIP-0112 implementation and is not a conformance test. It is a local compile/test proof that the CDP can exercise live V2 allocation and batch settlement under conservative basic-account assumptions. |

## Verification Closeout

2026-05-19 closeout confirms the compiled live-settlement behavior and keeps
the prototype non-release and non-conformant:

- `scripts/verify.sh` passed the DPM test phases for `simple-token-test` and
  `stablecoin-test` with `2 passed, 0 failed`.
- `scripts/verify.sh` did not run optional `daml-lint` or `daml-verify`
  because those tools were not available in its stablecoin-local search path.
- Manual workspace-root `daml-lint` was available only as
  `tools/daml-lint/target/debug/daml-lint`; it reported the acknowledged
  stablecoin false positives and the simple-token V2 unbounded-list findings
  recorded in `docs/AUDIT.md` and `docs/stablecoin/AUDIT.md`.
- Manual workspace-root `daml-verify` was available at
  `tools/daml-verify/.venv/bin/python` and proved 14/14 properties.
- The behavior checked includes close, liquidation, burn sink suppression,
  overpayment/change, unexpired locked input rollback, stale vault handling,
  provider-managed account rejection, metadata naming, privacy/disclosure
  assumptions, archival behavior, upgrade volatility, and unresolved gaps.

## Unblocked CDP prototype path

The CDP work can proceed ergonomically in small slices while the token-template
V2 transfer/allocation work is also moving:

1. Enumerate stablecoin modules and map every vault, mint, burn, repay,
   withdraw, and liquidation choice to the underlying token-template contract
   it consumes. **Done for the current `Stablecoin/` modules above.**
2. Add documentation and focused tests for the account policy defaults:
   `basicAccount` for withdrawn holdings, and `provider = Some admin` only for
   CDP-controlled or collateral-locked holdings if the implementation proves
   that authority model is needed. **Done for current V1 CDP outputs with
   `Stablecoin.Experimental.Cip112.AccountPolicy` and the
   `Cip112AccountPolicy` script probes.**
3. Prototype mint/burn account-id projection using `cip-112/mint` and
   `cip-112/burn` in metadata or transfer-leg shapes. If the required V2
   transfer or allocation type is not available in the stablecoin dependency
   graph yet, record that exact dependency as the blocker and keep the test
   narrow. **Done as a compile-proof projection only; V1 holding metadata is
   unchanged.**
4. Prototype liquidation as a V2 batch-settlement scenario against the compiled
   token-template `SettlementFactory_SettleBatch` path when it exists; before
   then, write a focused blocker test that captures the missing interface
   dependency. **Done for the local live boundary:** stablecoin now vendors the
   preview V2 DARs needed for allocations, finalization, settlement, events,
   and utils, ports the local simple-token V2 dependency, and exercises live
   close/liquidation settlement. **Current blocker:** allocation requests,
   iterated settlement, provider-managed account authority, cross-admin
   settlement, receipt/reporting standards, and conformance/release pins remain
   unresolved.
5. Re-baseline CDP invariants over Account-keyed positions once the first
   provider/account code compiles.

Reference `/Users/x/excanton/CN/splice`
`origin/token-standard-v2-daml-preview` only as evidence. The most relevant
paths are `token-standard/examples/splice-test-token-v2/`,
`token-standard/splice-token-standard-v2-test/`, and
`daml/splice-amulet/daml/Splice/AmuletAllocationV2.daml`. They inform the CDP
tradeoffs but do not require the stablecoin to adopt Splice's full account
configuration machinery.

## CDP-specific design decisions

These extend the workspace stakeholder packet with CDP-specific rows:

| ID | Decision | Owner Lane | Reviewer | Default Until Decided |
| :--- | :--- | :--- | :--- | :--- |
| SC-Q-001 | Vault keeper as `Account.provider`. Does the CDP admin (vault keeper) gain `provider` authority over collateral- and debt-side stablecoin holdings? Required-for-all-movements, opt-in delegation, or basic-account-only? | Technical architecture, security and assurance | OpenZeppelin security lead, Digital Asset technical contact | Owner `basicAccount` by default for withdrawn holdings, minted holdings, liquidation outputs, and vault-held collateral projections. `provider = Some admin` remains a candidate only for a future provider-managed collateral flow that proves the need. |
| SC-Q-002 | Burn-to-repay account-id: should repay-flow burns use the standard `cip-112/burn` account-id, or a more specific `cip-112/burn/repay-debt` to distinguish from non-CDP burns? | Technical architecture | Wallet maintainers | Use the standard `cip-112/burn`; pass `meta` for CDP-specific repay context. |
| SC-Q-003 | Liquidation as multi-leg settlement: should liquidation flows model the collateral seizure + debt write-off + fee transfer as a single `SettlementFactory_SettleBatch` call, or remain as separate choices? | Technical architecture, security and assurance | OpenZeppelin security lead | The live prototype uses a single batch with a liquidator-stablecoin debt leg into `cip-112/burn` and an owner-collateral seizure leg to the liquidator. Release work must still decide provider authority, receipt/reporting, cross-admin settlement, and conformance requirements. |
| SC-Q-004 | Cross-instrument liquidation: if collateral and debt are different assets with *different* admins, does the CDP still settle in one batch? Per §4.3.4 this is forbidden. | Technical architecture | OpenZeppelin technical lead | If different admins, liquidation requires the V2 cross-admin pattern (an `executor` co-validates atomicity between two `SettlementFactory_SettleBatch` calls). The simplification is to require collateral and debt admin to match for the initial M1 discovery slice and document the cross-admin pattern as a future enhancement. |
| SC-Q-005 | CDP invariants under the `Account` substitution: which of the 24 simple-token invariants gain a CDP corollary? E.g., "no debt without locked collateral" must hold over `Account`-keyed positions. | Security and assurance | OpenZeppelin security lead | Re-baseline the CDP invariant catalog after `Account` substitution; preserve the V1 list under `daml-verify` for backward-compat proof. |
| SC-Q-006 | `availableActions` for vault flows: does the CDP wallet present `Mint`, `Burn`, `Repay`, `Withdraw`, `Liquidate` actions through the standard map, or via CDP-specific extension? | Technical architecture, ecosystem adoption | OpenZeppelin technical lead, wallet maintainers | Use the standard `TransferInstructionAction.TIA_Custom` with namespaced id `oz.cdp/mint`, `oz.cdp/burn`, etc., aligned with `Metadata` DNS prefix convention from `docs/SCOPE.md` Q9. |

## Test impact

`stablecoin-test/` mirrors `simple-token-test/` plus CDP-specific tests
for the vault lifecycle. The V2 expansion:

- All CIP-0112 §5.4 transfer matrix variants apply to stablecoin Holdings
  because the CDP routes through simple-token transfers.
- CDP-specific V2 tests:
  - Current compile-proof probes:
    - withdrawn collateral outputs stay owner `basicAccount`;
    - minted stablecoin outputs stay owner `basicAccount`;
    - mint source account id is `cip-112/mint`;
    - burn sink account id is `cip-112/burn`;
    - burn/repay archives the stablecoin input and clears debt;
    - close overpayment returns stablecoin change as a basic-account output;
    - close archives the vault and stale withdraw against the consumed vault
      fails;
    - vault-held collateral stays basic-account by default, while a
      provider-managed candidate explicitly records `provider = Some admin`
      and is not the default;
    - unrelated third parties do not gain provider visibility, while V1 admin
      signatory visibility remains explicit;
    - liquidation outputs and overpayment change stay basic-account;
    - liquidation consumes the vault and stale follow-up exercise fails;
    - real preview V2 `SettlementFactory_SettleBatch` arguments compile for
      the liquidation debt and collateral legs;
    - live V2 close settlement burns debt through `cip-112/burn`, returns
      overpayment change, returns collateral, consumes the vault, and rejects
      stale follow-up exercises;
    - live V2 liquidation settlement burns debt, seizes collateral in the same
      batch, returns stablecoin change, reports bad debt for full seizure, and
      returns owner collateral remainder for partial seizure;
    - underpayment and unexpired locked-holding contention roll back without
      consuming the vault;
    - provider-managed account transfer legs are rejected by the local
      settlement factory account guard while metadata keeps the experimental
      flow name visible.
  - Future V2 DAR-backed tests:
    - V2 mint flow with `cip-112/mint` account-id on admin leg.
    - V2 burn flow with `cip-112/burn` account-id.
    - Allocation-request based V2 liquidation and close flows.
    - Iterated settlement, stale allocation replay, and direct cancel/withdraw
      recovery beyond the simple-token V2 dependency tests.
    - `Account.provider` authorization tests only after a real
      provider-managed CDP flow proves the need.
    - V1/V2 dual-compat mint/burn (a V1 wallet must still be able to repay
      a V2-issued vault loan).

## Verification pipeline impact

The CDP currently runs the same `scripts/verify.sh` orchestration. V2
changes propagate:

- `daml-lint`: no CDP-specific detector additions identified in
  discovery; revisit during implementation.
- `daml-props`: the existing transfer/conservation invariants for the
  CDP layer (debt/collateral conservation) must be restated over
  `Account` positions; provider authority becomes a precondition for
  liquidation generators.
- `daml-verify`: add a CDP-specific proof class covering
  "liquidation seizes collateral iff debt exceeds collateral ratio" over
  `Account`-keyed positions.

## Open questions specific to CDP

In addition to SC-Q-001 through SC-Q-006 above:

- Does the CDP need to advertise `showAccountInputFields: true` (because
  vault operations *can* be performed by a provider on behalf of an
  owner) while the underlying simple-token advertises `false`?
- How does the CDP interact with CIP-0103 (dApp Standard) when the
  wallet supports V2 signing flows but the CDP UI is V1?
- For partial repayments through multi-leg V2 allocations, what is the
  CDP-side authority check on the repayment leg?

## License note

`canton-stablecoin` is AGPL-3.0. Extension work inside this repo is fine
under AGPL. M1-PF-004 must resolve before any extended source is imported
into `repos/oz-canton-ri-lending/` (the future MIT-licensed lending RI).

## Non-goals

- No promotion of prototype Daml into a released RI or `repos/oz-daml-contracts/`.
- No release packaging; the V2 DAR source-of-record is unresolved.
- No claim of CIP-0112 conformance for the CDP layer.
- No new SDK pins unless the prototype proves the exact need and an ADR records
  it.
