# SCOPE: Stablecoin CDP System

## 1. Mission

This module delivers a minimal MakerDAO-esque Collateralized Debt Position (CDP) system built on the simple-token CIP-056 implementation. Users lock CIP-056-compliant collateral tokens to mint stablecoins. Stablecoins are `SimpleHolding` contracts with a stablecoin `InstrumentId`, so they work with the existing transfer infrastructure (SimpleTokenRules, TransferPreapproval, TransferInstruction) out of the box.

**Design goal:** Smallest possible addition that captures the core MakerDAO mechanics (overcollateralization, minting, liquidation, stability fees) while reusing existing templates and patterns from `simple-token`.

SDK: 3.4.11, LF target: 2.1.

## 2. Omitted Feature Analysis

After running three open-source verification tools ([daml-lint](https://github.com/OpenZeppelin/daml-lint), [daml-verify](https://github.com/OpenZeppelin/daml-verify), [daml-props](https://github.com/OpenZeppelin/daml-props)) and performing manual code review, several MakerDAO features were identified as deliberately excluded. This section examines each and asks whether it is required.

### 2.1 Governance (MKR Voting)

**What it is:** In MakerDAO, MKR token holders vote on risk parameters (stability fees, liquidation ratios, debt ceilings, collateral types). Governance changes are executed via time-locked proposals.

**Why MakerDAO needs it:** MakerDAO is a decentralized protocol with no single admin. Parameter changes must be approved by token-weighted voting.

**Why we left it out:** Our system has a single admin party who controls the VaultFactory and PriceOracle. Parameter changes are made by creating a new VaultFactory with updated `VaultParams`. No voting needed because there's only one decision-maker.

**Critical for production?** No, for a single-admin system. Yes, if you need consortium governance. If needed later, wrap VaultFactory creation with a vote-gated admin choice.

**Complexity to add:** ~1,500-2,000 LOC for proposal/vote/execute templates + time locks.

### 2.2 Dutch Auction Liquidation

**What it is:** MakerDAO v2 uses Dutch auctions for liquidation. Collateral starts at a high price and decreases over time until a bidder takes it. This achieves better price discovery than fixed-discount liquidation.

**Why MakerDAO needs it:** In a permissionless DeFi system, MEV and front-running mean fixed-discount liquidations are inefficient. Dutch auctions let the market discover the fair liquidation price.

**Why we left it out:** Our liquidation uses a fixed `liquidationBonus` (e.g., 13%) discount. This is simpler and sufficient for a permissioned Canton network where MEV and front-running don't exist (transactions are mediated, not broadcast).

**Critical for production?** No, on Canton. Canton's privacy model and mediated transactions eliminate the MEV problem that motivates Dutch auctions. Fixed-discount liquidation is the correct choice for this execution environment.

**Complexity to add:** ~800-1,200 LOC for auction templates, time decay, keeper infrastructure.

### 2.3 Global Debt Ceiling

**What it is:** MakerDAO caps the total DAI supply per collateral type. Each `ilk` (collateral type) has a `line` (ceiling) and there's a global `Line` across all collateral types.

**Why MakerDAO needs it:** Prevents over-exposure to any single collateral type. Risk management for the entire protocol.

**Why we left it out:** Our VaultFactory creates individual vaults with independent debt. There's no shared state tracking total debt across vaults. Adding a debt ceiling requires either a shared mutable contract or a read-only aggregate.

**Critical for production?** Recommended for production. This is the highest-priority omitted feature. A simple `totalDebt : Decimal` field on VaultFactory (consumed and recreated on each mint/burn) would suffice, at the cost of increased contention.

**Complexity to add:** ~100-200 LOC. Main challenge is contention — all mints/burns would serialize through the factory.

### 2.4 Dai Savings Rate (DSR)

**What it is:** MakerDAO pays interest to DAI holders who deposit into the DSR contract. The rate is set by governance and funded by stability fee revenue.

**Why we left it out:** The DSR is an incentive mechanism for DAI demand. Our stablecoins are standard `SimpleHolding` contracts — adding interest would require wrapping them in a deposit contract that accrues value over time. This is orthogonal to the CDP mechanics.

**Critical for production?** No. The DSR is a monetary policy tool, not a core CDP feature.

**Complexity to add:** ~400-600 LOC for deposit/withdraw templates + interest accrual.

### 2.5 Emergency Shutdown

**What it is:** MakerDAO has an emergency shutdown mechanism that freezes all vaults, allows vault owners to claim excess collateral, and allows DAI holders to redeem at the last oracle price.

**Why we left it out:** Our admin can effectively freeze the system by archiving the VaultFactory (preventing new vaults) and not exercising liquidation choices. Full emergency shutdown with pro-rata collateral distribution adds significant complexity.

**Critical for production?** Recommended for production. Without it, a catastrophic oracle failure or black swan event has no clean resolution path.

**Complexity to add:** ~500-800 LOC for shutdown trigger, claim templates, pro-rata distribution.

### 2.6 Oracle Staleness / Multi-Oracle Medianizer

**What it is:** MakerDAO uses a medianizer that takes the median of multiple oracle feeds and rejects stale prices. Each oracle feed has a TTL.

**Why we left it out:** Our PriceOracle is a single admin-controlled feed with an `updatedAt` timestamp but no TTL check. The oracle `ensure price > 0.0` prevents zero prices, and the admin is trusted.

**Critical for production?** Recommended for production. Oracle manipulation is the primary attack vector for CDP systems. At minimum, add a `maxStaleness : RelTime` check. Multi-oracle medianizer is optional.

**Complexity to add:** ~50 LOC for staleness check, ~300-500 LOC for multi-oracle medianizer.

### 2.7 Multi-Collateral Vaults

**What it is:** MakerDAO supports multiple collateral types (ETH, WBTC, USDC, etc.) with per-type risk parameters.

**Why we left it out:** Our system already supports multiple collateral types — each `VaultFactory` is parameterized with a `collateralInstrumentId`. Creating multiple VaultFactory contracts with different instruments achieves multi-collateral support without additional templates.

**Critical for production?** Already supported via multiple VaultFactory instances. No additional code needed.

**Complexity to add:** 0 LOC. Already supported.

### 2.8 Summary

| Feature | MakerDAO LOC | Our LOC | Required | Verdict |
|---------|-------------|---------|----------|---------|
| Governance (MKR voting) | ~5,000 | 0 | No (single admin) | Correctly omitted |
| Dutch auction liquidation | ~2,000 | 0 | No (Canton has no MEV) | Correctly omitted |
| Global debt ceiling | ~200 | 0 | Recommended | V2 candidate |
| Dai Savings Rate | ~800 | 0 | No | Correctly omitted |
| Emergency shutdown | ~1,500 | 0 | Recommended | V2 candidate |
| Oracle staleness | ~50 | 0 | Recommended | V2 candidate |
| Multi-oracle medianizer | ~500 | 0 | No | V2 candidate |
| Multi-collateral vaults | ~0 | 0 | Yes | Already supported |
| **Total omitted** | **~10,050** | **0** | | |
| **Core CDP logic** | | **~340** | **Yes** | **Implemented** |

## 3. In Scope

- 3 on-ledger templates: `PriceOracle`, `VaultFactory`, `Vault`
- 1 nonconsuming factory choice: `VaultFactory_OpenVault`
- 7 consuming choices: `PriceOracle_UpdatePrice`, `Vault_DepositCollateral`, `Vault_WithdrawCollateral`, `Vault_MintStablecoin`, `Vault_BurnStablecoin`, `Vault_Close`, `Vault_Liquidate`
- 3 helper functions: `archiveAndSumCollateral`, `accrueDebt`, `collateralRatio`
- Linear stability fee accrual
- Full liquidation with partial/full paths and bad debt tracking
- CIP-056 interop: minted stablecoins transfer via existing `SimpleTokenRules`
- 16 security invariants (see [PLAN.md](PLAN.md))
- 22 functional tests: 12 happy path + 9 negative + 1 integration
- 5 property-based tests (daml-props, 200 random sequences each)
- 5 formal proofs (daml-verify, Z3 SMT solver)

## 4. Out of Scope

| Feature | Reason | Where It Would Live |
|---|---|---|
| Governance / multi-sig | Single-admin architecture | Wrapper around VaultFactory |
| Auction liquidation | Canton has no MEV | Replace `Vault_Liquidate` |
| Debt ceiling | Contention tradeoff | VaultFactory mutable state |
| Emergency shutdown | Complexity vs. MVP | New template + admin choice |
| Oracle staleness | Trusted admin oracle | `maxStaleness` field on PriceOracle |
| DSR / savings rate | Monetary policy, not CDP | Separate deposit template |
| Off-ledger HTTP API | Deployment concern | Separate service project |
| CLI tooling | Out of scope | N/A |

## 5. Differences from MakerDAO

| # | Feature | MakerDAO | This Project | Justification |
|---|---|---|---|---|
| 1 | Liquidation mechanism | Dutch auction | Fixed-bonus direct | Canton has no MEV; simpler |
| 2 | Fee accrual | Compound (per-second) | Linear (per-year) | Simpler arithmetic; sufficient accuracy |
| 3 | Oracle | Multi-feed medianizer | Single admin feed | Trusted admin; add medianizer if needed |
| 4 | Debt ceiling | Per-ilk and global | None | MVP scope; V2 candidate |
| 5 | Governance | MKR token voting | Admin-controlled | Single admin; no voting needed |
| 6 | Collateral types | Multi-ilk via `vat` | Multi-VaultFactory instances | Same capability, simpler architecture |
| 7 | Stablecoin type | ERC-20 DAI | CIP-056 SimpleHolding | Canton-native; standard-compliant |
| 8 | Authorization | Permissionless | `controller admin, owner`/`admin, liquidator` | Canton signatory model |
| 9 | Emergency shutdown | Global shutdown module | Archive factory | Simple but less graceful |
| 10 | Savings rate | DSR contract | None | Monetary policy out of scope |

## 6. Architectural Decisions

| Decision | Resolution | Rationale |
|---|---|---|
| Stablecoin representation | `SimpleHolding` with stablecoin `InstrumentId` | Reuses existing CIP-056 transfer infrastructure |
| Vault mutation model | UTXO (consume and recreate) | Matches Canton/DAML execution model |
| Fee accrual model | Linear on-demand | Simple, no background scheduler needed |
| Liquidation authorization | `controller admin, liquidator` | Gives authorizer set {admin, owner, liquidator} for creating holdings for both parties |
| Oracle visibility | `observers` field on PriceOracle | Allows vault owners to fetch oracle in their choices |
| VaultFactory reusability | Nonconsuming `VaultFactory_OpenVault` | Factory can create unlimited vaults without contention |
| Params embedding | `VaultParams` stored on each Vault | Vault carries its own risk parameters (params can diverge across vaults) |

## 7. Post-MVP

Ordered by priority.

**P0 -- Verification Hardening**
1. Oracle price guard in `Vault_Liquidate` (patched, see [AUDIT.md](AUDIT.md) L1)

**P1 -- Recommended for Production**
2. Oracle staleness check (`maxStaleness : RelTime`)
3. Global debt ceiling per VaultFactory
4. Emergency shutdown mechanism

**P2 -- Extensions**
5. Multi-oracle medianizer
6. Compound fee accrual (per-second)
7. Partial liquidation (liquidate portion of debt, not all)
8. Vault migration (move to new VaultFactory with different params)
9. Dai Savings Rate equivalent
10. Dutch auction liquidation

## 8. References

- MakerDAO whitepaper: https://makerdao.com/whitepaper/
- MakerDAO technical docs: https://docs.makerdao.com/
- CIP-0056 Final (created 2025-03-07, approved 2025-03-31)
