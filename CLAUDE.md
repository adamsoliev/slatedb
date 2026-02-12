# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is SlateDB

Cloud-native embedded key-value storage engine built on LSM-tree principles. Writes data to object storage (S3, GCS, ABS) instead of local disk. Rust core with FFI bindings for Go, Java, and Python.

## Build & Test Commands

```bash
# Build
cargo build --workspace --all-features

# Run all tests
cargo test --workspace --all-features

# Run a single test by name
cargo test --all-features -p slatedb -- test_name_here

# Run doc tests
cargo test --doc --all-features

# Linting (CI runs with RUSTFLAGS="-Dwarnings")
cargo clippy --workspace --all-features
cargo clippy --workspace --all-features --tests

# Format check
cargo fmt -- --check

# Benchmarks
cargo bench

# C bindings
cargo build --release -p slatedb-c

# Python bindings (from slatedb-py/)
maturin develop && pytest -xvs
```

CI uses `cargo nextest` and `cargo hack` (install: `cargo install cargo-nextest cargo-hack`). CI-equivalent commands:

```bash
cargo hack check --each-feature --no-dev-deps --workspace
cargo hack clippy --each-feature --no-dev-deps --workspace
cargo nextest run --workspace --all-features --profile ci
RUSTFLAGS="--cfg dst --cfg tokio_unstable" cargo nextest run -p slatedb-dst --all-features --profile dst
```

## Workspace Structure

| Crate | Purpose |
|-------|---------|
| `slatedb` | Core database engine |
| `slatedb-common` | Shared utilities (clock abstraction) |
| `slatedb-c` | C FFI bindings (cdylib + staticlib, uses cbindgen) |
| `slatedb-py` | Python bindings (PyO3) |
| `slatedb-cli` | CLI tool |
| `slatedb-dst` | Deterministic simulation testing |
| `slatedb-bencher` | Benchmarking |
| `slatedb-txn-obj` | Transaction object abstractions |
| `examples` | Example programs |

Language bindings (Go in `slatedb-go/`, Java in `slatedb-java/`) use the C FFI library.

## Architecture

### Write Path

`Db::put()` → `WritableKVTable` (in-memory skiplist memtable) → flush → WAL SST on object storage → L0 SSTs → compaction into sorted runs.

Each entry is a `RowEntry` with key, value (`ValueDeletable`: Value | Merge | Tombstone), sequence number, and create/expire timestamps.

### Read Path

`Db::get()` → memtable → immutable memtables → L0 SSTs → sorted runs. Bloom filters skip irrelevant SSTs. Block cache (`foyer` or `moka`) and disk cache (`cached_object_store`) reduce object store reads.

`DbIterator` for range scans. `DbSnapshot` for point-in-time reads. `DbTransaction` for MVCC transactions.

### Key Types

- `Db` / `DbBuilder` — main entry point (`db.rs`)
- `DbState` — manifest-backed state tracking all SSTs (`db_state.rs`)
- `Manifest` / `ManifestStore` — durable metadata (`manifest/`)
- `SsTableHandle` / `SsTableId` (`Wal(u64)` | `Compacted(Ulid)`) — SST references (`db_state.rs`)
- `SortedRun` — non-overlapping SSTs at one compaction level (`db_state.rs`)
- `WritableKVTable` / `ImmutableMemtable` — memtable lifecycle (`mem_table.rs`)
- `Settings` — configuration loaded from TOML/JSON/YAML/env vars with `SLATEDB_` prefix (`config.rs`)

### Key Traits

- `DbRead` — object-safe read interface (get, scan, scan_prefix) (`db_read.rs`)
- `ObjectStore` — re-exported from `object_store` crate
- `CompactionScheduler` / `CompactionSchedulerSupplier` — compaction policy (`compactor.rs`)
- `MergeOperator` — custom merge semantics (`merge_operator.rs`)
- `KeyValueIterator` — generic iteration (`iter.rs`)

### Serialization

FlatBuffers for on-disk format. Schemas in `schemas/`. Generated code in `slatedb/src/generated/`. CI validates schema evolution and regenerated code matches checked-in files. Uses `flatc` v24.3.25.

## Code Conventions

- Rust 1.91.1 (pinned in `rust-toolchain.toml`)
- `RUSTFLAGS="-Dwarnings"` in CI — all warnings are errors
- `#![deny(clippy::disallowed_types, clippy::disallowed_methods)]` in production code to enforce determinism
- `unreachable_pub = "warn"` workspace-wide
- Custom cfg attributes: `dst`, `slow`, `tokio_unstable`
- Test infrastructure auto-initializes via `#[ctor]` (deadlock detector, tracing)
- Default features: `aws`, `foyer`

## Contribution Process

- Issue-first for substantial changes; RFC required for large features (template: `rfcs/0000-template.md`)
- Incremental PRs preferred; large PRs submitted as drafts with breakdown plan
- AI-generated PRs must be disclosed with tool/model specified
- CLA required
