# canton-stablecoin

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)

> [!WARNING]
> This software is experimental and not intended for production use. Use at your own risk.

A DAML template for building CIP-056 token registries and MakerDAO-esque CDP stablecoins on the Canton Network. Clone it, run the setup script, start writing contracts.

**simple-token**: 800 lines of production code implement all 6 CIP-056 on-ledger interfaces. 36 tests cover transfer lifecycle, allocation/DvP, defragmentation, and 20 security invariants.

**stablecoin**: 340 lines extend simple-token with a CDP vault system (overcollateralization, minting, liquidation, stability fees). 27 tests cover the full vault lifecycle and 22 security invariants. Minted stablecoins are standard CIP-056 holdings that transfer via the existing SimpleTokenRules.

Three verification tools (static analysis, property testing, formal proofs) validate both modules.

## Quick Start

```sh
git clone https://github.com/OpenZeppelin/canton-stablecoin.git && cd canton-stablecoin
scripts/setup.sh            # checks tools, builds, runs tests
```

Or manually:

```sh
cd simple-token && dpm build
cd ../stablecoin && dpm build
cd ../simple-token-test && dpm build && dpm test
cd ../stablecoin-test && dpm build && dpm test
```

`dpm test` requires Java 21. The setup script detects your JAVA_HOME automatically.

## What You Get

```
simple-token/          7 templates implementing 6 CIP-056 interfaces
simple-token-test/     36 tests (9 transfer, 5 allocation, 2 defrag, 20 security)
stablecoin/            3 templates (PriceOracle, VaultFactory, Vault) + 3 helpers
stablecoin-test/       27 tests (12 happy path, 9 negative, 1 integration, 5 property-based)
dars/                  Splice interface DARs + daml-props (committed, no setup needed)
scripts/               setup.sh (bootstrap) and verify.sh (run all verification tools)
docs/                  Design docs for simple-token and stablecoin
```

## Customizing

Rename the token: edit `simple-token/daml.yaml` and `SimpleTokenRules.supportedInstruments`.

Add behavior: the factory's 3-way dispatch (self-transfer, direct, two-step) lives in `Rules.daml`. Allocation/DvP logic is in `Allocation.daml`. Each template maps to one lifecycle state.

CDP vaults: `stablecoin/daml/Stablecoin/Vault.daml` contains VaultFactory, Vault, and all choices. Oracle price feeds live in `Oracle.daml`. Risk parameters (collateral ratio, liquidation bonus, stability fee) are in `VaultParams`.

Write tests first. The TDD skill enforces RED-GREEN-REFACTOR gates.

## Verification

```sh
scripts/verify.sh     # runs all three tools
```

- [daml-lint](https://github.com/OpenZeppelin/daml-lint) -- static analysis (6 detectors, <1s)
- [daml-props](https://github.com/OpenZeppelin/daml-props) -- property-based testing with shrinking (~30s)
- [daml-verify](https://github.com/OpenZeppelin/daml-verify) -- formal verification via Z3 (~2s)

Install verification tools: `scripts/setup.sh` (without `--skip-verification`).

## Agent Skills

For AI-assisted development with Claude Code or Codex, see [daml-skills](https://github.com/OpenZeppelin/daml-skills.git).

## Prerequisites

- [dpm](https://docs.daml.com) (Digital Asset Package Manager)
- Daml SDK 3.4.10
- Java 21

## Docs

### simple-token (CIP-056)
- [docs/SCOPE.md](docs/SCOPE.md) -- what's in scope, what's not, differences from Splice
- [docs/PLAN.md](docs/PLAN.md) -- implementation plan, security invariants, test criteria
- [docs/AUDIT.md](docs/AUDIT.md) -- verification report from three tools

### stablecoin (CDP)
- [docs/stablecoin/SCOPE.md](docs/stablecoin/SCOPE.md) -- scope, omitted MakerDAO features, architectural decisions
- [docs/stablecoin/PLAN.md](docs/stablecoin/PLAN.md) -- templates, choices, security invariants, test plan
- [docs/stablecoin/AUDIT.md](docs/stablecoin/AUDIT.md) -- verification report (14 proofs, 5 property tests, 3 findings)

