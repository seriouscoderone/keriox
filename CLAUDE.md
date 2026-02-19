# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

KERIOX is a Rust implementation of the Key Event Receipt Infrastructure (KERI) protocol — a decentralized key management system providing blockchain-equivalent cryptographic security without global ordering. Developed by the Human Colossus Foundation.

## Build and Test Commands

```bash
# Build entire workspace
cargo build
cargo build --all-features

# Run all workspace tests (CI uses this)
cargo test --all-features --verbose

# Test a specific crate
cargo test --package keri-core
cargo test --package keri-controller
cargo test --package witness
cargo test --package watcher
cargo test --package teliox
cargo test --package keri-sdk
cargo test --package keri-tests        # Integration tests (spins up witnesses/watchers)

# Run a single test
cargo test test_name
cargo test --package keri-core test_name

# Check compilation (faster than build)
cargo check --all-features

# Build specific binaries
cargo build --package witness --release
cargo build --package watcher --release

# Run witness/watcher locally (uses YAML config)
cd components/witness && cargo run -- -c witness.yml
cd components/watcher && cargo run -- -c watcher.yml
```

Note: CI runs `cargo check --all-features` and `cargo test --all-features` on stable Rust. Always pass `--all-features` when checking complete compilation.

## Workspace Architecture

```
keriox_core  (keri-core)        ← Core protocol: events, database traits, processor, signer, prefixes
├── keriox_sdk  (keri-sdk)      ← Simplified SDK wrapping core for external consumers
├── support/teliox              ← Transaction Event Log (TEL) for credential issuance/revocation
├── support/gossip              ← Gossip protocol (standalone, no keri-core dependency)
├── components/controller (keri-controller) ← High-level client: identifier management, OOBI resolution, mailbox
├── components/witness          ← KERI Witness node (actix-web HTTP server)
└── components/watcher          ← KERI Watcher node (actix-web HTTP server)
    keriox_tests (keri-tests)   ← Integration tests using all components
```

**Important:** Crate names differ from directory names. The `keriox_core/` directory produces the `keri-core` crate, `keriox_sdk/` produces `keri-sdk`, etc. Use the crate names with `--package`.

### Dependency Flow

`keri-core` → `teliox` → `keri-controller` / `witness` / `watcher` / `keri-sdk`

- `keri-core` has no internal workspace dependencies
- `teliox` depends on `keri-core` with `query` feature
- `witness` and `watcher` depend on `keri-core` with `oobi-manager` + `mailbox` features
- `keri-controller` depends on `keri-core` + `teliox`
- `keri-sdk` depends on `keri-core` + `teliox`
- `keri-tests` depends on all components

### Feature Flags (keri-core)

Feature flags gate significant functionality and affect which modules are compiled:

| Feature | Enables | Used by |
|---------|---------|---------|
| `storage-redb` | `RedbDatabase`, redb dependency (default) | witness, watcher, controller, keri-tests |
| `query` | `query` module, `serde_cbor` | teliox, keri-sdk, controller |
| `oobi` | `oobi` module, URL/strum deps | keri-sdk, controller, witness, watcher |
| `oobi-manager` | `oobi_manager` + `transport` modules (implies `oobi` + `query` + `storage-redb`) | controller, witness, watcher |
| `mailbox` | `mailbox` module (implies `query` + `storage-redb`) | witness, watcher |

Many `pub` items are gated behind `#[cfg(feature = "...")]`. When adding code that uses OOBI, query, or mailbox types, ensure the appropriate feature is enabled. When verifying compilation without redb, use: `cargo check --package keri-core --no-default-features --features query`.

## Core Abstractions

### Database Layer

Storage is **trait-based and feature-flagged**. The `storage-redb` feature (enabled by default) provides the concrete `RedbDatabase` implementation backed by the redb embedded key-value store. Without this feature, only trait-based code and the in-memory `MemoryDatabase` are available, enabling alternative storage backends (e.g. DynamoDB for serverless).

- **`EventDatabase`** (`database/mod.rs`) — Primary trait for KEL storage: finalized events, receipts, key state, replies
- **`LogDatabase`** (`database/mod.rs`) — Lower-level log storage with transaction support
- **`EscrowDatabase`** / **`SequencedEventDatabase`** — Escrow storage for events awaiting completion
- **`EscrowCreator`** — Factory trait for creating escrow database instances
- **`RedbDatabase`** (`database/redb/mod.rs`) — Concrete redb implementation (gated behind `storage-redb`)
- **`MemoryDatabase`** (`database/memory/mod.rs`) — In-memory implementation for testing the trait abstraction

All processor and escrow code is generic over `D: EventDatabase`, so swapping storage backends requires no changes to event processing logic. For TEL storage, `teliox` defines its own `TelEventDatabase` trait with a `RedbTelDatabase` implementation (also gated behind `storage-redb`).

### Event Processing Pipeline

1. Raw bytes → `parse_event_stream()` / `parse_notice_stream()` (in `actor/mod.rs`) → `Message` / `Notice`
2. `BasicProcessor` receives `Notice` and runs validation via `EventValidator`
3. Valid events → stored in database, `NotificationBus` emits `Notification::KeyEventAdded`
4. Invalid/incomplete events → routed to appropriate escrow via notifications (out-of-order, partially signed, partially witnessed, delegation pending)
5. Escrows re-process events when blocking conditions resolve

Key types in the pipeline:
- **`Notice`** — Event, NontransferableRct, or TransferableRct
- **`Message`** — Notice or Op (query/reply)
- **`SignedEventMessage`** — Event with signatures, optional witness receipts, optional delegator seal
- **`Notification`** / **`NotificationBus`** — Observer pattern for escrow routing

### NotificationBus (Swappable Dispatch)

`NotificationBus` (`processor/notification.rs`) is a `Clone`-able wrapper around `Arc<dyn NotificationDispatch>`. It uses an internal dispatch trait to allow swapping how notifications are delivered without adding generic type parameters anywhere in the codebase.

- **`NotificationDispatch`** trait — `dispatch(&self, &Notification)` and `register_observer(&self, ...)`. Implement this for custom notification delivery (e.g. SQS for serverless).
- **`InProcessDispatch`** (private) — Default implementation preserving the original HashMap-based in-process observer pattern. Uses `RwLock` for interior mutability and `OnceLock<NotificationBus>` as a back-reference for `Notifier::notify()` callbacks.
- **`NotificationBus::new()`** — Creates a bus with `InProcessDispatch` (default behavior).
- **`NotificationBus::from_dispatch(Arc<dyn NotificationDispatch>)`** — Creates a bus backed by a custom dispatch implementation.
- **`Notifier`** trait — Unchanged: `fn notify(&self, &Notification, &NotificationBus) -> Result<(), Error>`. Escrows implement this to react to notifications.

All `register_observer` methods take `&self` (not `&mut self`) thanks to interior mutability in the dispatch layer.

### Processor Trait (`Processor`)

Defined in `processor/mod.rs`. Implement this to customize event processing. `BasicProcessor` is the standard implementation. The `process_notice` method is the main entry point.

### Key Management (`signer/mod.rs`)

- **`KeyManager`** trait — `sign()`, `public_key()`, `next_public_key()`, `rotate()`
- **`CryptoBox`** — Ed25519 implementation with pre-rotation support
- **`Signer`** — Lower-level signing (used directly by witness/watcher)

### Identifier Prefixes (`prefix/mod.rs`)

`IdentifierPrefix` enum: `Basic(BasicPrefix)` | `SelfAddressing(SaidValue)` | `SelfSigning(SelfSigningPrefix)`

- `BasicPrefix` — Public key-based (e.g., Ed25519 with "D" prefix in CESR)
- `SelfAddressingIdentifier` — Content-addressed (digest-based, "E" prefix)
- `SeedPrefix` — Encoded private key seeds used to derive key pairs

### Controller Component

Two levels of controller abstraction exist:

1. **`keri-controller`** (`components/controller/`) — Full-featured: manages `KnownEvents`, `Communication` (HTTP transport), OOBI resolution, mailbox queries, identifier lifecycle. Uses `RedbDatabase` internally.
2. **`keri-sdk`** (`keriox_sdk/`) — Simplified wrapper generic over database types. Exposes `Controller<D, T>` and `Identifier<D>` with basic incept/process/state operations.

### Witness and Watcher

Both are actix-web HTTP servers configured via YAML + CLI args + env vars (using `figment`):

- **Witness** (`components/witness/`): Signs receipts for events, stores KEL, supports mailbox queries. Config prefix: `WITNESS_`
- **Watcher** (`components/watcher/`): Monitors KELs, resolves OOBIs, provides TEL data. Config prefix: `WATCHER_`

Both use `clap` for CLI, `figment` for layered config (YAML → env → CLI args).

## Integration Tests

`keriox_tests/` contains integration tests that spin up real witness and watcher instances using `test-context`. The `InfrastructureContext` (in `src/settings.rs`) sets up:
- Two witnesses on ports 3232, 3233
- One watcher on port 3236

Tests use `actix_rt::test` and `#[test_context(InfrastructureContext)]`. These tests bind to fixed ports, so they cannot run in parallel with each other. Run them with:

```bash
cargo test --package keri-tests -- --test-threads=1
```

## External Crate Dependencies

Key external crates to understand:
- **`cesrox`** — CESR encoding/decoding, parsing (`parse_many`), `CesrPrimitive` trait
- **`said`** — Self-Addressing Identifier derivation, `SelfAddressingIdentifier`, versioning, serialization formats
- **`redb`** — Embedded database (replaced earlier `sled` usage)
- **`rkyv`** — Zero-copy deserialization, used on core types (`KeyEvent`, `IdentifierState`, `IdentifierPrefix`)

## Error Handling

`keri-core` uses a central `Error` enum in `error/mod.rs` with `thiserror`. It derives both `Serialize` and `Deserialize`. Component crates define their own error types (e.g., `ControllerError`, `ActorError`, `MechanicsError`) that wrap `keri_core::error::Error`.

## Serialization

KERI events use a custom serialization format. The `event_message/serializer.rs` handles KERI-specific field ordering. Events support JSON, CBOR, and MessagePack formats via `said::version::format::SerializationFormats`. The `serde_hex` crate is used for hex-encoded sequence numbers (`sn` fields).
