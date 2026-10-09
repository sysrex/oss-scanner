FROM rust:1.93.0-bookworm

# SKIP_GUEST_BUILD and friends are how the maintainers' CI compiles and tests the workspace without the risc0 and SP1
# guest toolchains: the zkVM guests are not built, the risc0 and SP1 provers run in dev/mock mode, and
# Docker-backed tests (postgres, hyperlane agents) are skipped. They stay set in the image so offline runs hit the
# same build fingerprints.
ENV DEBIAN_FRONTEND=noninteractive \
    CARGO_TERM_COLOR=never \
    CARGO_INCREMENTAL=0 \
    CARGO_PROFILE_DEV_DEBUG=line-tables-only \
    CARGO_NET_GIT_FETCH_WITH_CLI=true \
    RUSTFLAGS="-C link-arg=-fuse-ld=lld" \
    SKIP_GUEST_BUILD=1 \
    SP1_SKIP_PROGRAM_BUILD=true \
    RISC0_SKIP_BUILD=1 \
    RISC0_DEV_MODE=1 \
    SP1_PROVER=mock \
    SOV_TEST_SKIP_DOCKER=1 \
    PROPTEST_CASES=50 \
    RUST_BACKTRACE=1

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git make clang libclang-dev lld cmake pkg-config protobuf-compiler libprotobuf-dev \
      python3 python3-dev \
    && rm -rf /var/lib/apt/lists/*

# cargo-nextest is the maintainers' test runner (`make test`).
RUN case "$(uname -m)" in aarch64) p=linux-arm ;; *) p=linux ;; esac \
    && curl -fsSL "https://get.nexte.st/0.9.100/$p" | tar -xzf - -C /usr/local/cargo/bin \
    && cargo nextest --version

# Line tables for workspace crates only: full debug info for ~1300 dependencies makes the test binaries several
# times larger and the links slower.
RUN printf '[profile.dev.package."*"]\ndebug = false\n' >> /usr/local/cargo/config.toml

COPY . /src
WORKDIR /src

# Image layers drop sub-second file times, which makes cargo rebuild crates whose build scripts track source mtimes.
# Whole-second times on the sources keep the fingerprints valid after export.
RUN find /src -path /src/target -prune -o -exec touch -h -d '2020-01-01T00:00:00Z' {} +

# Install the exact toolchain and components pinned in rust-toolchain.toml.
RUN rustup toolchain install && rustc --version && cargo --version

RUN for i in 1 2 3; do cargo fetch --locked && break; sleep 10; done && cargo fetch --locked

# Keep parallel rustc and linker jobs within memory (about 2 GB each).
RUN echo "$(( $(nproc) < $(awk '/MemTotal/ {print int($2 / 2097152)}' /proc/meminfo) ? $(nproc) : $(awk '/MemTotal/ {print int($2 / 2097152)}' /proc/meminfo) ))" > /tmp/jobs \
    && cat /tmp/jobs

# Compile every test binary of every workspace member with all features, as CI's `make test-all` does, so offline
# runs only relink. This also builds the sov-demo-rollup node (mock and Celestia DA) and sov-cli in target/debug.
RUN cargo test --locked --workspace --all-features --no-run -j "$(cat /tmp/jobs)"

# Run a cheap subset (core interfaces, bank, the Celestia adapter, sequencer registry). Selecting it with a filter
# rather than -p keeps the same feature set as the build above, so nothing is recompiled.
RUN (timeout 1200 cargo nextest run --locked --workspace --all-features --no-fail-fast \
        --build-jobs "$(cat /tmp/jobs)" --test-threads "$(cat /tmp/jobs)" \
        -E 'package(sov-rollup-interface) | package(sov-bank) | package(sov-celestia-adapter) | package(sov-sequencer-registry)' \
        || echo "WARNING: cargo nextest reported failures") \
    && find /src -path /src/target -prune -o -exec touch -h -d '2020-01-01T00:00:00Z' {} + \
    && git status --short

ENV CARGO_NET_OFFLINE=true
