# sovereign-sdk threat model

The Sovereign SDK is a Rust toolkit for building rollups with real-time soft confirmations. A rollup built with it reads batches and proofs from a data availability (DA) layer such as Celestia, executes them in a deterministic state transition function (STF) composed of modules, and commits the result to a NOMT-backed authenticated state. The same STF runs natively in full nodes and the sequencer, and inside a zkVM (risc0 or SP1) or under optimistic attestation for light clients and bridges. Protect STF determinism and agreement between native and zk execution, funds and supply, the sequencing rules, the soundness of DA, proof, and attestation verification, and the availability of nodes and the sequencer.

## Existing project guidance

Read the checkout's [AGENTS.md](https://github.com/Sovereign-Labs/sovereign-sdk/blob/dev/AGENTS.md), especially "Scanning for state breakage", [module-system AGENTS.md](https://github.com/Sovereign-Labs/sovereign-sdk/blob/dev/crates/module-system/AGENTS.md), the AGENTS.md files in `crates/full-node/sov-ethereum` and `crates/module-system/module-implementations/sov-evm`, the [rollup-interface specs](https://github.com/Sovereign-Labs/sovereign-sdk/tree/dev/crates/rollup-interface/specs), the [state machine README](https://github.com/Sovereign-Labs/sovereign-sdk/blob/dev/crates/rollup-interface/src/state_machine/README.md), the [Celestia adapter README](https://github.com/Sovereign-Labs/sovereign-sdk/blob/dev/crates/adapters/celestia/README.md), and the READMEs of each crate under `crates/full-node` and `crates/module-system`. The repository has no published audit reports.
The determinism, safe-arithmetic, and state-breakage rules in `AGENTS.md` state what the maintainers treat as security properties. Use them to frame findings.
Use the scanned checkout for technical instructions, since linked documentation follows `dev`.

## Actors and entry points

Treat transaction senders, anyone who posts blobs to the rollup's DA namespace, provers, attesters and challengers, and clients of the node and sequencer APIs as adversarial. Treat a registered sequencer, including the preferred sequencer, as adversarial towards the STF: it can reorder, withhold, or forge batches, but it must not be able to steal funds, mint, or make honest full nodes diverge. Assume the DA layer orders and publishes data honestly, but treat every byte it returns as attacker-controlled. Distinguish input any account can submit from input that needs a bonded sequencer, prover, or attester, a module admin, or the rollup operator.

Data enters from the DA layer through the `DaService` and `DaVerifier` traits in `crates/rollup-interface/src/state_machine/da.rs` and `crates/rollup-interface/src/node/da.rs`. The Celestia adapter in `crates/adapters/celestia` must reject any omission, duplication, reordering, or insertion of blobs and any tampered sender: review `verifier/mod.rs`, `verifier/proofs.rs`, `shares.rs`, `envelope.rs`, and `types/` as parsers and verifiers of namespace inclusion and completeness proofs. Its README marks it as an unaudited prototype; it is in scope. `crates/adapters/mock-da` is a test DA and matters only where production code relies on its behaviour.

Batches and proofs are selected and executed by `crates/module-system/sov-modules-stf-blueprint` (`stf_blueprint.rs`, `sequencer_mode/registered.rs` and `unregistered.rs`, `proof_processing.rs`) with the kernels in `crates/module-system/sov-kernels` (`basic.rs`, `soft_confirmations.rs`) and the blob selector in `sov-blob-storage`. Check that based (unregistered) blobs and deferred blobs cannot bypass the preferred sequencer's rules, censor or replay transactions, or skip fees and penalties. Transactions are decoded with borsh and authenticated in `crates/module-system/sov-modules-api/src/transaction`, `sov-capabilities`, `sov-eip712-auth`, `sov-solana-offchain-auth`, and `sov-uniqueness` (nonces and generations). Gas metering lives in `sov-modules-api/src/gas` and `sov-state`: any state access, decoding, or signature check that is not charged, or charged differently in native and zk mode, is in scope.

Review the modules under `crates/module-system/module-implementations`: `sov-bank` (supply and transfers), `sov-accounts`, `sov-sequencer-registry` (bonding and slashing), `sov-prover-incentives` and `sov-attester-incentives` (zk proofs, optimistic attestations and challenges, bonding and unbonding), `sov-paymaster`, `sov-revenue-share`, `sov-timelock`, `sov-chain-state`, `sov-evm` (revm execution), and the Hyperlane modules in `crates/module-system/hyperlane` (mailbox, ISMs, merkle hook, IGP, warp routes). Check supply conservation, bond accounting, replay protection, and authorization of admin calls. The zkVM adapters in `crates/adapters/risc0` and `crates/adapters/sp1` verify proofs and code commitments; a proof or attestation accepted for the wrong code, state root, or DA range is in scope. Code under `#[cfg(feature = "native")]` does not run in the zkVM, so a check that exists only there is a native/zk divergence.

The sequencer in `crates/full-node/sov-sequencer` (`rest_api.rs`, `preferred/`, `standard/`) accepts transactions over REST and streams confirmations over WebSocket, executes them speculatively, and publishes batches. The full node runner in `crates/full-node/sov-stf-runner` exposes REST and JSON-RPC (`http/`), `crates/full-node/sov-ledger-apis` and `sov-rollup-apis` serve ledger and state queries, and `crates/full-node/sov-ethereum` serves the Ethereum JSON-RPC. Storage is `sov-state` and `sov-db` (NOMT and RocksDB through rockbound). Treat the sequencer's transaction submission as public. Treat other node APIs as operator-exposed: report resource exhaustion through them only when it is reachable with default configuration and exceeds the cost an operator would reasonably expect.

## Scope and ownership

The whole repository is in scope for source review. The priorities below guide investigation rather than exclude other code.

Prioritize, in order: non-determinism in the STF, or divergence between native and zk execution, that splits honest nodes or makes a valid proof impossible; theft, inflation, or loss of funds and bonds; accepting invalid DA data, proofs, or attestations, or rejecting valid ones; censorship or bypassing of sequencer rules and fees; gas metering bypass; and remotely triggerable crashes or denial of service against full nodes and the sequencer.

`examples/` (including `demo-rollup`), `crates/bench`, `crates/utils` load-testing and soak-testing crates, `scripts/`, `docker/`, `typescript/`, and `python/` are reference, test, and tooling code. Report them only for a concrete security failure, or when they show a flaw in an SDK crate. risc0, SP1, NOMT, revm, the Celestia client crates (celestia-types, celestia-rpc, celestia-client), and rockbound are separate projects pinned in `Cargo.lock`. They remain in scope when the SDK reaches the affected code, but identify the upstream owner before proposing a fix. Do not report a library contract violation without a path from the SDK.

## Build and bounded validation

The checkout is `/src`, and the build output is in `/src/target/debug`. The demo rollup node `sov-demo-rollup`, built with both mock and Celestia DA support, and `sov-cli` are there. Every workspace member's test binaries are compiled with `--all-features` before network access is removed; a later `cargo test --workspace --all-features` only relinks `sov-demo-rollup` and two dependents, whose build scripts always rerun. The image sets `CARGO_NET_OFFLINE=true`, so a missing crate fails immediately instead of hanging. Do not edit `Cargo.toml` or `Cargo.lock`, and do not run `cargo switcheroo`.

The image builds and tests the way CI does for unit tests: `SKIP_GUEST_BUILD=1` and `SP1_SKIP_PROGRAM_BUILD=true` skip the risc0 and SP1 guest builds, `RISC0_DEV_MODE=1` and `SP1_PROVER=mock` run the provers in mock mode, and `SOV_TEST_SKIP_DOCKER=1` skips tests that need Docker. The guest toolchains are not installed, so real zk proving and tests that need guest ELFs cannot run; review guest code and zk-mode paths by source and through the native `Zk` execution mode. Tests that need Docker (the `da_service` and `test_helper::docker` tests of the Celestia adapter, postgres-backed sequencer storage, Hyperlane agents, toxiproxy) or a Celestia node cannot run. NOMT storage uses `io_uring`. Where the sandbox's seccomp policy blocks it, every test that opens rollup storage panics with `Error building io_uring: ... Operation not permitted` in `nomt/src/io/linux.rs`; that is an environment limit, not a finding. These are execution limits, not source-review exclusions. Keep the environment variables and `/usr/local/cargo/config.toml` unchanged: changing them, or the feature set, recompiles large parts of the workspace.

The scanner VM has two CPUs and 8 GB of memory. Keep `--build-jobs 2` (`-j 2` for `cargo test`) and two test threads, and bound every run with `timeout`. Keep `--workspace --all-features` and select tests with a filter so nothing is rebuilt; `cargo test -p <crate>` works but recompiles around 150 dependencies with that crate's feature set first. Tests under `examples/demo-rollup/tests` and `crates/full-node/sov-sequencer/tests` start in-process rollups and are slow, so run single tests.

```sh
timeout 1200 cargo nextest run --workspace --all-features --build-jobs 2 --test-threads 2 -E 'package(sov-bank)'
timeout 1200 cargo nextest run --workspace --all-features --build-jobs 2 --test-threads 2 \
  -E 'package(sov-celestia-adapter) & not test(/da_service::tests|test_helper::docker/)'
timeout 1800 cargo nextest run --workspace --all-features --build-jobs 2 --test-threads 2 -E 'package(sov-attester-incentives)'
timeout 1200 cargo test -j 2 -p sov-rollup-interface -- --test-threads=2
```

Use `sov-test-utils` (`TestRunner`, `TestRollup`, generators) for in-process rollups and transactions, and `PROPTEST_CASES` to bound property tests. The libFuzzer targets in `crates/fuzz/fuzz_targets` (accounts and bank call messages, address parsing) build offline in a few minutes with `cargo build --all-features -j 2 -p sovereign-sdk-fuzz`. Coverage instrumentation needs a nightly toolchain that is not installed, so these binaries only mutate inputs blindly; prefer property tests and targeted unit tests. Bound any fuzz run with a run count and an external timeout:

```sh
timeout --kill-after=5 300 /src/target/debug/bank_parse_call_message -runs=200000 -max_total_time=240
```

Reproduce only against in-process test rollups, mock DA, and temporary directories. Never connect to Celestia Mainnet Beta, Mocha, Arabica, or other public networks or rollups, and never use real keys or funds.

## Evidence and severity

Record the exact revision, crate and feature set, execution mode (native or zk), DA adapter, attacker capability (any account, blob poster on the namespace, registered or preferred sequencer, prover, attester or challenger, module admin, operator), and the observed consequence. Keep source-only hypotheses separate from tests that reproduce the behavior. For determinism findings, show two executions of the same input producing different state roots, or a divergence between native and `Zk` execution. For resource findings, compare the work performed with the gas charged or the protocol bound, and state the measured cost.

Rate severity by impact on a production rollup:

- Critical: a state root divergence or chain halt triggered by one transaction or blob; theft or inflation of funds or bonds; acceptance of an invalid DA proof, zk proof, or attestation that lets a wrong state root be finalized; or bypassing authentication to act as another account.
- High: a remotely triggerable crash or sustained denial of service against full nodes or the sequencer with default configuration; a gas metering bypass that makes unbounded work cheap; censorship or bypassing of sequencer rules by an unprivileged party; or permanent loss of funds or bonds that needs a privileged but non-malicious actor.
- Medium: limited-scope degradation, bounded resource amplification, or attacks needing a malicious registered sequencer, prover, or attester, or an unusual configuration.
- Low: issues needing local access, operator or module-admin collusion, or other strong preconditions, and defense-in-depth hardening.

A panic, sanitizer report, or bug-class label does not determine severity on its own. A panic caught and reverted for one transaction is not a chain halt. Keep confidence, remediation urgency, and disclosure timing separate.

Check Git history and existing tests for the same root cause. Keep patches minimal and follow the Minimal Patch Mode and state-breakage rules in `AGENTS.md`, and add a test that fails before the fix and passes after it.

Report findings privately to `security@celestia.org` or through a GitHub security advisory on the repository, as described in the repository's [security policy](https://github.com/Sovereign-Labs/sovereign-sdk/blob/dev/SECURITY.md). Do not put suspected vulnerabilities, reproducer details, or unreviewed findings into public issues or PRs. Include the smallest bounded reproducer, actual output, expected result, and evidence limits in each report. Disclosure decisions belong to the Sovereign SDK maintainers and the Celestia security team.
