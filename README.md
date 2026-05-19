# canton-stablecoin

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)

> [!WARNING]
> This software is experimental and not intended for production use. Use at your own risk.

A DAML template for building CIP-056 token registries and MakerDAO-esque CDP stablecoins on the Canton Network. Clone it, run the setup script, start writing contracts.

**simple-token**: the CIP-056 implementation remains the token engine for the stablecoin. It now also carries non-release CIP-0112 V2 allocation, finalization, settlement, cancellation, withdrawal, and event prototypes used by the CDP discovery tests. The original V1 transfer lifecycle, allocation/DvP, defragmentation, and security tests are preserved.

**stablecoin**: the core V1 CDP vault system (overcollateralization, minting, liquidation, stability fees) is preserved. The repo now adds experimental CIP-0112 live settlement helpers for close and liquidation over the local SimpleTokenRules V2 prototype. 27 core V1 tests still cover the vault lifecycle and security invariants, with 17 additional non-release CIP-0112 account-policy, dependency, and live-settlement probes. Minted stablecoins remain standard CIP-056 holdings.

The DPM test suites validate both modules. Optional verification tools
(`daml-lint` and `daml-verify`) can be run when installed; `scripts/verify.sh`
reports them as skipped rather than pretending they ran.

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
simple-token/          CIP-056 templates plus non-release CIP-0112 V2 prototypes
simple-token-test/     V1 regression tests plus CIP-0112 transfer/allocation probes
stablecoin/            PriceOracle, VaultFactory, Vault, and experimental CIP-0112 helpers
stablecoin-test/       27 core V1 tests + 17 experimental CIP-0112 probes
dars/                  Splice V1 DARs, preview V2 token DARs, daml-props
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
scripts/verify.sh     # runs DPM tests and optional tools when available
```

- [daml-lint](https://github.com/OpenZeppelin/daml-lint) -- static analysis (6 detectors, <1s)
- [daml-props](https://github.com/OpenZeppelin/daml-props) -- property-based testing with shrinking (~30s)
- [daml-verify](https://github.com/OpenZeppelin/daml-verify) -- formal verification via Z3 (~2s)

Install optional verification tools: `scripts/setup.sh` (without
`--skip-verification`). In the 2026-05-19 closeout workspace,
`scripts/verify.sh` found DPM and ran the two Daml test packages, but did not
find `daml-lint` or the stablecoin-local `tools/daml-verify` venv on its own
search path.

## Agent Skills

For AI-assisted development with Claude Code or Codex, see [daml-skills](https://github.com/OpenZeppelin/daml-skills.git).

## Prerequisites

- [dpm](https://docs.daml.com) (Digital Asset Package Manager)
- Daml SDK 3.4.11
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
- [docs/CIP-0112-EXTENSION-PLAN.md](docs/CIP-0112-EXTENSION-PLAN.md) -- non-release CIP-0112 CDP account-policy, live settlement probes, vendored DAR checksums, authority/privacy notes, and blockers
