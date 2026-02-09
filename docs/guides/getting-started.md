---
status: stable
---

# Getting Started

How to set up a new Rust project following rust-starter patterns.

## Prerequisites

- Rust toolchain via [rustup](https://rustup.rs/) (stable channel)
- `cargo` (included with rustup)
- `clippy` and `rustfmt` (included with rustup)

## Path A: Single-Crate Project (Recommended Start)

Most projects should start here. You get a testable library and a thin binary in one crate.

### 1. Initialize

```bash
mkdir my-tool && cd my-tool
cargo init
touch src/lib.rs
```

### 2. Set up Cargo.toml

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

### 3. Add tooling config

```toml
# rustfmt.toml
edition = "2021"
max_width = 100
use_field_init_shorthand = true
```

### 4. Create initial structure

```
my-tool/
  Cargo.toml
  rustfmt.toml
  src/
    main.rs          # Binary entry point
    lib.rs           # Public API re-exports
    domain.rs        # Domain types
    error.rs         # Error types
```

### 5. Verify

```bash
cargo build
cargo test
cargo clippy --all-targets -- -D warnings
cargo fmt --all -- --check
```

## Path B: Workspace Project

When a project outgrows a single crate (see [when to evolve](../architecture/architecture.md#when-to-evolve-into-a-workspace)), split into a workspace.

### 1. Initialize the workspace

```bash
mkdir my-tool && cd my-tool
cargo init
mkdir -p crates/my-tool-core
mkdir -p crates/my-tool-engine
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
thiserror = "2"
anyhow = "1"
clap = { version = "4", features = ["derive"] }

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
module_name_repetitions = "allow"
must_use_candidate = "allow"
missing_errors_doc = "allow"

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

### 3. Initialize library crates

```bash
cd crates/my-tool-core && cargo init --lib && cd ../..
cd crates/my-tool-engine && cargo init --lib && cd ../..
```

Update each crate's `Cargo.toml` to use workspace dependencies:

```toml
# crates/my-tool-core/Cargo.toml
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

### 4. Add tooling config

```toml
# rustfmt.toml
edition = "2021"
max_width = 100
use_field_init_shorthand = true
```

### 5. Verify

```bash
cargo build
cargo test --all
cargo clippy --all-targets -- -D warnings
cargo fmt --all -- --check
```

## Next Steps

- Read the [architecture overview](../architecture/architecture.md)
- Follow the [development guide](development.md) to add your first feature
- Review the [ADRs](../decisions/) to understand the reasoning behind each pattern
