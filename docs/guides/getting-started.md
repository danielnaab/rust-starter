---
status: stable
---

# Getting Started

How to set up a new Rust project following rust-starter patterns.

## Prerequisites

- Rust toolchain via [rustup](https://rustup.rs/) (stable channel)
- `cargo` (included with rustup)
- `clippy` and `rustfmt` (included with rustup)

## Create a New Project

### 1. Initialize the workspace

```bash
mkdir my-project && cd my-project
cargo init                          # Creates src/main.rs and Cargo.toml
mkdir -p crates/my-project-core
mkdir -p crates/my-project-engine
```

### 2. Set up workspace Cargo.toml

```toml
[workspace]
resolver = "2"
members = ["crates/*"]

[workspace.package]
edition = "2021"
rust-version = "1.75"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
serde_yaml = "0.9"
thiserror = "2"
anyhow = "1"
clap = { version = "4", features = ["derive"] }

[package]
name = "my-project"
version = "0.1.0"
edition.workspace = true

[dependencies]
my-project-core = { path = "crates/my-project-core" }
my-project-engine = { path = "crates/my-project-engine" }
anyhow.workspace = true
clap.workspace = true
```

### 3. Initialize library crates

```bash
cd crates/my-project-core
cargo init --lib
cd ../my-project-engine
cargo init --lib
cd ../..
```

Update each crate's `Cargo.toml` to use workspace dependencies:

```toml
# crates/my-project-core/Cargo.toml
[package]
name = "my-project-core"
version = "0.1.0"
edition.workspace = true

[dependencies]
thiserror.workspace = true
serde.workspace = true
```

```toml
# crates/my-project-engine/Cargo.toml
[package]
name = "my-project-engine"
version = "0.1.0"
edition.workspace = true

[dependencies]
my-project-core = { path = "../my-project-core" }
thiserror.workspace = true
```

### 4. Add tooling configuration

```toml
# rustfmt.toml
edition = "2021"
max_width = 100
use_field_init_shorthand = true
```

```toml
# Add to workspace Cargo.toml
[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
module_name_repetitions = "allow"
must_use_candidate = "allow"
missing_errors_doc = "allow"
```

### 5. Verify

```bash
cargo build                         # Compiles
cargo test --all                    # Tests pass
cargo clippy --all-targets          # No warnings
cargo fmt --all -- --check          # Formatting clean
```

## Project Layout

After setup:

```
my-project/
  Cargo.toml                # Workspace root + binary
  rustfmt.toml              # Format config
  src/
    main.rs                  # Binary entry point
  crates/
    my-project-core/
      Cargo.toml
      src/lib.rs             # Domain types, traits, errors
    my-project-engine/
      Cargo.toml
      src/lib.rs             # Business logic
  tests/                     # Integration tests (created as needed)
```

## Next Steps

- Read the [architecture overview](../architecture/architecture.md)
- Follow the [development guide](development.md) to add your first feature
- Review the [ADRs](../decisions/) to understand the reasoning behind each pattern
