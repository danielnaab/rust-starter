---
status: stable
---

# Project Reference

Quick reference for project structure, configuration, and tooling.

## Directory Structure

```
project/
  Cargo.toml              # Workspace root, binary definition
  Cargo.lock              # Lockfile (committed for binaries)
  rustfmt.toml            # rustfmt config
  src/
    main.rs               # Binary entry point (thin)
  crates/
    project-core/         # Domain types, traits, errors
      Cargo.toml
      src/
        lib.rs            # Public API re-exports
        domain.rs         # Domain model types
        error.rs          # Error type definitions
    project-engine/       # Business logic, orchestration
      Cargo.toml
      src/
        lib.rs            # Public API
        resolve.rs        # Resolution logic
        git.rs            # Git adapter
  tests/                  # Integration tests
    common/
      mod.rs              # Shared test helpers and fakes
    integration_test.rs   # Cross-crate integration tests
  tests/fixtures/         # Test data files
```

## Cargo.toml Patterns

### Workspace Root

```toml
[workspace]
resolver = "2"
members = ["crates/*"]

[workspace.package]
edition = "2021"
rust-version = "1.75"

[workspace.dependencies]
# Declare all dependency versions here
serde = { version = "1", features = ["derive"] }
thiserror = "2"
anyhow = "1"
clap = { version = "4", features = ["derive"] }

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
module_name_repetitions = "allow"
must_use_candidate = "allow"
missing_errors_doc = "allow"
```

### Binary Crate

```toml
[package]
name = "my-project"
version = "0.1.0"
edition.workspace = true

[dependencies]
my-project-core = { path = "crates/my-project-core" }
my-project-engine = { path = "crates/my-project-engine" }
anyhow.workspace = true
clap.workspace = true

[lints]
workspace = true
```

### Library Crate

```toml
[package]
name = "my-project-core"
version = "0.1.0"
edition.workspace = true

[dependencies]
thiserror.workspace = true
serde.workspace = true

[lints]
workspace = true
```

## Standard Dependencies

| Crate | Purpose | Used In |
|---|---|---|
| `thiserror` | Derive error types | Library crates |
| `anyhow` | Contextual error chains | Binary crate |
| `serde` | Serialization framework | All crates |
| `serde_yaml` | YAML parsing | Engine/binary |
| `serde_json` | JSON output | Binary |
| `clap` | CLI framework | Binary |
| `tracing` | Structured logging | All crates |

## Tooling Quick Reference

| Task | Command |
|---|---|
| Build all | `cargo build --all` |
| Test all | `cargo test --all` |
| Test one crate | `cargo test -p my-project-core` |
| Lint | `cargo clippy --all-targets -- -D warnings` |
| Format | `cargo fmt --all` |
| Format check | `cargo fmt --all -- --check` |
| Build docs | `cargo doc --no-deps --open` |
| Update deps | `cargo update` |
| Check MSRV | `cargo msrv` (install via `cargo install cargo-msrv`) |
| Unused deps | `cargo machete` (install via `cargo install cargo-machete`) |

## Dependency Flow

```
src/main.rs
  depends on: project-engine, project-core, anyhow, clap

crates/project-engine/
  depends on: project-core, thiserror

crates/project-core/
  depends on: thiserror, serde
  depends on nothing from this workspace
```

**Rule:** Core never depends on engine. Engine never depends on the binary. Dependencies flow inward only.

## Import Conventions

```rust
// Group imports: std, external crates, workspace crates, this crate
use std::path::Path;

use anyhow::{Context, Result};
use clap::Parser;

use project_core::{DependencyName, ProjectConfig};
use project_engine::resolve_all;

use crate::output::format_table;
```

rustfmt handles import sorting automatically with `merge_imports`.

## Commit Lockfile

For binary projects, commit `Cargo.lock`. This ensures reproducible builds.

For library-only projects, don't commit `Cargo.lock` — let downstream consumers resolve versions.
