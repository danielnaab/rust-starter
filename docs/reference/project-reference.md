---
status: stable
---

# Project Reference

Quick reference for project structure, configuration, and tooling.

## Directory Structure

### Single Crate

```
my-tool/
  Cargo.toml              # Package definition
  Cargo.lock              # Lockfile (committed for binaries)
  rustfmt.toml            # rustfmt config
  src/
    main.rs               # Binary entry point (thin)
    lib.rs                # Public API re-exports
    domain.rs             # Domain model types
    error.rs              # Error type definitions
    store.rs              # Trait definitions + adapters
  tests/
    integration.rs        # Integration tests
    fixtures/             # Test data files
```

### Workspace

```
my-tool/
  Cargo.toml              # Workspace root, binary definition
  Cargo.lock              # Lockfile (committed for binaries)
  rustfmt.toml            # rustfmt config
  src/
    main.rs               # Binary entry point (thin)
  crates/
    my-tool-core/         # Domain types, traits, errors
      Cargo.toml
      src/
        lib.rs            # Public API re-exports
        domain.rs         # Domain model types
        error.rs          # Error type definitions
    my-tool-engine/       # Business logic, orchestration
      Cargo.toml
      src/
        lib.rs            # Public API
        process.rs        # Processing logic
        store.rs          # Storage adapter
  tests/                  # Integration tests
    common/
      mod.rs              # Shared test helpers and fakes
    integration_test.rs   # Cross-crate integration tests
  tests/fixtures/         # Test data files
```

## Cargo.toml Patterns

### Single Crate

```toml
[package]
name = "my-tool"
version = "0.1.0"
edition = "2021"

[dependencies]
thiserror = "2"
anyhow = "1"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }

[lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
module_name_repetitions = "allow"
must_use_candidate = "allow"
missing_errors_doc = "allow"
```

### Workspace Root

```toml
[workspace]
resolver = "2"
members = ["crates/*"]

[workspace.package]
edition = "2021"
rust-version = "1.75"

[workspace.dependencies]
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

### Binary Crate (workspace)

```toml
[package]
name = "my-tool"
version = "0.1.0"
edition.workspace = true

[dependencies]
my-tool-core = { path = "crates/my-tool-core" }
my-tool-engine = { path = "crates/my-tool-engine" }
anyhow.workspace = true
clap.workspace = true

[lints]
workspace = true
```

### Library Crate (workspace)

```toml
[package]
name = "my-tool-core"
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
| `thiserror` | Derive error types | Library code |
| `anyhow` | Contextual error chains | Binary code |
| `serde` | Serialization framework | All modules |
| `serde_yaml` | YAML parsing | Engine/binary |
| `serde_json` | JSON output | Binary |
| `clap` | CLI framework | Binary |
| `tracing` | Structured logging | All modules |

## Code Quality

### rustfmt Configuration

```toml
# rustfmt.toml (project root)
edition = "2021"
max_width = 100
use_field_init_shorthand = true
```

Keep configuration minimal. The default rustfmt style is well-designed — override only what genuinely improves readability.

### clippy Configuration

```toml
# In Cargo.toml [lints.clippy] or [workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }

# Allow specific pedantic lints that conflict with common patterns
module_name_repetitions = "allow"     # We prefer explicit module::TypeName
must_use_candidate = "allow"          # Too noisy for most codebases
missing_errors_doc = "allow"          # Error types are self-documenting
```

The `pedantic` lint group catches subtle issues. Allow-list the few that don't fit rather than disabling the whole group.

### CI Commands

```bash
# Check formatting (fails on diff)
cargo fmt --all -- --check

# Run clippy (fails on warnings)
cargo clippy --all-targets --all-features -- -D warnings

# Run tests
cargo test --all
```

## Tooling Quick Reference

| Task | Command |
|---|---|
| Build all | `cargo build --all` |
| Test all | `cargo test --all` |
| Test one crate | `cargo test -p my-tool-core` |
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
  depends on: engine, core, anyhow, clap

crates/my-tool-engine/ (or src/lib.rs in single-crate)
  depends on: core, thiserror

crates/my-tool-core/ (or core modules in single-crate)
  depends on: thiserror, serde
  depends on nothing from this workspace
```

**Rule:** Core never depends on engine. Engine never depends on the binary. Dependencies flow inward only.

## Naming Conventions

Follow the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) naming patterns:

### Conversion Methods

| Prefix | Cost | Ownership | Example |
|---|---|---|---|
| `as_` | Free | Borrows `&self`, returns reference | `as_str() -> &str` |
| `to_` | Expensive | Borrows `&self`, returns owned | `to_string() -> String` |
| `into_` | Variable | Consumes `self`, returns owned | `into_inner() -> String` |

### Predicates

| Prefix | Returns | Example |
|---|---|---|
| `is_` | `bool` | `is_empty()`, `is_valid()` |
| `has_` | `bool` | `has_children()`, `has_key()` |

### Constructors and Factories

| Pattern | When | Example |
|---|---|---|
| `new()` | Primary constructor | `ItemId::new("x")` |
| `with_*()` | Constructor variant | `Vec::with_capacity(10)` |
| `from_*()` | Named conversion (when `From` trait is insufficient) | `Config::from_path(p)` |
| `try_new()` | Fallible primary constructor (alternative to `new() -> Result`) | `ItemId::try_new("x")` |

### Iterators

| Method | Returns | Example |
|---|---|---|
| `iter()` | `Iterator<Item = &T>` | `items.iter()` |
| `iter_mut()` | `Iterator<Item = &mut T>` | `items.iter_mut()` |
| `into_iter()` | `Iterator<Item = T>` (consumes) | `items.into_iter()` |

### Attributes

**`#[must_use]`** — Add to functions where ignoring the return value is almost certainly a bug. Particularly useful on builder methods and pure functions:

```rust
#[must_use]
pub fn with_timeout(mut self, secs: u64) -> Self {
    self.timeout = secs;
    self
}
```

The `must_use_candidate` clippy lint is suppressed (too noisy), so add `#[must_use]` explicitly where it matters.

See [architecture.md — Naming Conventions](../architecture/architecture.md#naming-conventions) for design rationale.

## Import Conventions

```rust
// Group imports: std, external crates, workspace crates, this crate
use std::path::Path;

use anyhow::{Context, Result};
use clap::Parser;

use my_tool_core::{ItemId, Config};
use my_tool_engine::process_all;

use crate::output::format_table;
```

rustfmt handles import sorting automatically with `merge_imports`.

## Commit Lockfile

For binary projects, commit `Cargo.lock`. This ensures reproducible builds.

For library-only projects, don't commit `Cargo.lock` — let downstream consumers resolve versions.
