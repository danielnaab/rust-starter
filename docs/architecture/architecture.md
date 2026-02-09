---
status: stable
---

# Architecture

Patterns for structuring Rust applications.

## Scope

These patterns target **application** architecture: CLI tools, TUI apps, services with business logic. For library-only crates, some patterns (anyhow, clap, the binary layer) don't apply. For async services, additional patterns (tokio runtime, async traits) come into play — those are not covered here.

## Layers

```
+------------------------------------------+
|              Binary (CLI)                |
|  Parses args, wires dependencies, exits  |
+------------------------------------------+
          |                    |
          v                    v
+-------------------+  +---------------+
|      Engine       |  |   Adapters    |
| Business logic,   |  | Trait impls   |
| orchestration     |  | for I/O       |
+-------------------+  +---------------+
          |                    |
          v                    v
+------------------------------------------+
|               Core                       |
|  Domain types, traits, error types       |
|  No I/O. No side effects.               |
+------------------------------------------+
```

**Dependency rule:** Dependencies point inward. Core depends on nothing. Engine depends on core. Adapters depend on core (to implement its traits). Binary depends on everything.

**Adapter placement:** Adapters implementing core traits live in the engine crate (or a dedicated adapter crate for larger projects), not in core. Core defines *what* is needed; adapters provide *how*.

## Crate Structure

### Starting Point

Start with a single crate. This is the right choice for most new projects:

```
my-tool/
  Cargo.toml
  src/
    main.rs         # Entry point
    lib.rs          # Public API, re-exports
    domain.rs       # Domain types
    store.rs        # Trait definitions + adapter implementations
    error.rs        # Error types
  tests/
    integration.rs  # Integration tests
```

A single crate with `src/lib.rs` + `src/main.rs` gives you testable library code and a thin binary in one package. The module system provides logical boundaries without the overhead of a workspace.

### When to Evolve into a Workspace

Split into a workspace when:
- Two components have genuinely different dependency trees
- Compile time matters and changes are localized to one area
- A library is reusable across projects (e.g., a shared protocol crate)

Don't split when:
- The project is small and boundaries would be ceremony
- You're splitting speculatively "in case we need it later"

### Workspace Layout

When a workspace is warranted, use `crates/` for libraries and root-level `src/` for the primary binary:

```
my-tool/
  Cargo.toml                    # [workspace] root
  crates/
    my-tool-core/               # Types, traits, errors
      Cargo.toml
      src/
        lib.rs                  # pub use re-exports
        config.rs               # Configuration types
        domain.rs               # Domain model
        error.rs                # Error types
    my-tool-engine/             # Business logic
      Cargo.toml                # depends on my-tool-core
      src/
        lib.rs
        process.rs              # Processing logic
        store.rs                # Storage adapter (implements core traits)
  src/
    main.rs                     # Binary entry point
  tests/
    process_integration.rs      # Integration tests
```

## Core Crate Patterns

Core contains domain types, trait definitions, and error types. It has minimal dependencies (typically just `serde` and `thiserror`).

### Type Design Principles

**Make invalid states unrepresentable.** Use the type system to eliminate impossible combinations at compile time:

```rust
// BAD: boolean flag — caller must know which combinations are valid
struct Item {
    url: Option<String>,
    path: Option<PathBuf>,
    is_remote: bool,  // must be consistent with url/path
}

// GOOD: enum makes exactly one variant possible
enum ItemKind {
    Remote { url: String },
    Local { path: PathBuf },
}
```

Apply this at every level: enums over boolean flags, newtypes over raw strings, `Result` over sentinel values. If a function can't fail, don't return `Result`. If a value can't be empty, validate at construction and carry the proof in the type.

**Parse, don't validate.** Validate data once at the boundary (parsing input, reading config), then carry the proof in the type. Code that receives an `ItemId` knows it's already valid — no re-checking needed.

### Domain Types

```rust
use std::path::PathBuf;

/// A unique identifier for an item.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct ItemId(String);

impl ItemId {
    pub fn new(id: impl Into<String>) -> Result<Self, ValidationError> {
        let id = id.into();
        if id.is_empty() {
            return Err(ValidationError::empty_field("item ID"));
        }
        if id.contains(' ') {
            return Err(ValidationError::invalid_format(
                "item ID",
                "must not contain spaces",
            ));
        }
        Ok(Self(id))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

/// Project configuration loaded from config.yaml.
#[derive(Debug, Clone)]
pub struct Config {
    pub name: String,
    pub items: Vec<Item>,
}

/// A single item declaration.
#[derive(Debug, Clone)]
pub struct Item {
    pub id: ItemId,
    pub kind: ItemKind,
}

/// Where an item comes from.
#[derive(Debug, Clone)]
pub enum ItemKind {
    Remote { url: String, reference: String },
    Local { path: PathBuf },
}
```

**Key patterns:**
- Newtypes for identifiers (`ItemId`). Validation in constructor.
- Plain structs for data containers. Public fields when there's no invariant to protect.
- Enums for closed sets of alternatives (`ItemKind`).
- Derive `Debug, Clone` on everything. Add `PartialEq, Eq, Hash` when needed for collections.
- Implement `Display` for types with a natural human-readable form (identifiers, errors). For complex structs, `Debug` is sufficient.
- Derive or implement `Default` when a type has a meaningful zero/empty state. Use `..Default::default()` for partial struct construction in tests.

### Traits (Ports)

```rust
/// Read project configuration from a source.
pub trait Repository {
    fn load_config(&self, path: &Path) -> Result<Config, RepoError>;
}

/// Look up a record by reference.
pub trait Store {
    fn lookup(&self, url: &str, reference: &str) -> Result<RecordId, StoreError>;
}
```

**Key patterns:**
- Traits live in the core crate — the crate that *needs* the capability.
- One capability per trait. Small interfaces are easier to fake in tests.
- Choose the receiver that matches the operation's semantics: `&self` for reads, `&mut self` for mutations, `self` for consumption.
- Return the core crate's error type, not adapter-specific errors.

### Error Types

```rust
use std::path::PathBuf;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum RepoError {
    #[error("config file not found: {path}")]
    NotFound { path: PathBuf },

    #[error("invalid configuration: {reason}")]
    Invalid { reason: String },

    #[error("failed to read config file")]
    Io(#[from] std::io::Error),
}

#[derive(Debug, Error)]
pub enum ValidationError {
    #[error("{field} cannot be empty")]
    EmptyField { field: &'static str },

    #[error("{field} has invalid format: {reason}")]
    InvalidFormat {
        field: &'static str,
        reason: &'static str,
    },
}

impl ValidationError {
    pub fn empty_field(field: &'static str) -> Self {
        Self::EmptyField { field }
    }

    pub fn invalid_format(field: &'static str, reason: &'static str) -> Self {
        Self::InvalidFormat { field, reason }
    }
}
```

**Key patterns:**
- One error enum per domain concern (not one giant enum).
- Structured fields, not formatted strings — callers can match and extract.
- `#[from]` for automatic conversion from underlying errors.
- Constructor methods on error types for ergonomic creation.
- All library errors derive `Debug` and `thiserror::Error`.
- Use `#[error(transparent)]` to wrap an error without adding a new message layer — the inner error's display is used directly:
  ```rust
  #[error(transparent)]
  Validation(#[from] ValidationError),  // displays as the inner error
  ```

## Engine Crate Patterns

Engine contains business logic. It depends on core and uses trait bounds to stay decoupled from adapters.

```rust
use my_tool_core::{
    Store, Config, Item, ItemKind, RecordId, StoreError,
};

/// Result of processing all items.
#[derive(Debug)]
pub struct Report {
    pub entries: Vec<ReportEntry>,
}

#[derive(Debug)]
pub struct ReportEntry {
    pub id: ItemId,
    pub record: RecordId,
}

/// Process all remote items in a configuration.
pub fn process_all(
    config: &Config,
    store: &impl Store,
) -> Result<Report, ProcessError> {
    let mut entries = Vec::new();

    for item in &config.items {
        match &item.kind {
            ItemKind::Remote { url, reference } => {
                let record = store.lookup(url, reference)?;
                entries.push(ReportEntry {
                    id: item.id.clone(),
                    record,
                });
            }
            ItemKind::Local { .. } => {
                // Local items don't need processing
            }
        }
    }

    Ok(Report { entries })
}
```

**Key patterns:**
- Free functions for stateless operations. `impl` blocks when state is involved.
- Take trait bounds as `&impl Trait` for simple cases. Use generic `<S: Store>` when the type parameter is used in multiple positions.
- Return domain-specific result types, not raw `Result<_, Box<dyn Error>>`.
- Logic is pure where possible — given these inputs, produce this output.
- Prefer iterator chains when the transformation is straightforward. The `collect::<Result<Vec<_>, _>>()` pattern short-circuits on the first error:
  ```rust
  let entries = config.items.iter()
      .filter_map(|item| match &item.kind {
          ItemKind::Remote { url, reference } => Some((item, url, reference)),
          ItemKind::Local { .. } => None,
      })
      .map(|(item, url, reference)| {
          let record = store.lookup(url, reference)?;
          Ok(ReportEntry { id: item.id.clone(), record })
      })
      .collect::<Result<Vec<_>, ProcessError>>()?;
  ```
  Use manual loops when the logic involves complex accumulation across iterations or would be harder to read as a chain.

## Binary Patterns

The binary is thin. It parses arguments, constructs concrete types, calls engine functions, and formats output.

```rust
use anyhow::{Context, Result};
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "my-tool", about = "A project management tool")]
struct Cli {
    #[command(subcommand)]
    command: Command,

    /// Output as JSON
    #[arg(long, global = true)]
    json: bool,
}

#[derive(Subcommand)]
enum Command {
    /// Process all items
    Process,
    /// Show item status
    Status,
}

fn main() -> Result<()> {
    let cli = Cli::parse();

    match cli.command {
        Command::Process => cmd_process(cli.json),
        Command::Status => cmd_status(cli.json),
    }
}

fn cmd_process(json: bool) -> Result<()> {
    let repo = FileRepository::new();
    let config = repo
        .load_config(Path::new("config.yaml"))
        .context("failed to load project configuration")?;

    let store = HttpStore::new();
    let report = process_all(&config, &store)
        .context("failed to process items")?;

    if json {
        println!("{}", serde_json::to_string_pretty(&report)?);
    } else {
        for entry in &report.entries {
            println!("{}: {}", entry.id.as_str(), entry.record);
        }
    }

    Ok(())
}
```

**Key patterns:**
- `anyhow::Result` in binary, never in library code.
- `.context()` on every `?` that crosses an abstraction boundary — it produces readable error chains.
- Match on commands, delegate to handler functions.
- `--json` flag for machine-readable output.
- No business logic in the binary — just wiring and formatting.

## Testing Patterns

### Unit Tests (in-file)

```rust
// In src/process.rs (or crates/my-tool-engine/src/process.rs)

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashMap;

    struct FakeStore {
        records: HashMap<(String, String), RecordId>,
    }

    impl FakeStore {
        fn new() -> Self {
            Self { records: HashMap::new() }
        }

        fn with_record(
            mut self,
            url: &str,
            reference: &str,
            record: RecordId,
        ) -> Self {
            self.records.insert((url.to_string(), reference.to_string()), record);
            self
        }
    }

    impl Store for FakeStore {
        fn lookup(&self, url: &str, reference: &str) -> Result<RecordId, StoreError> {
            self.records
                .get(&(url.to_string(), reference.to_string()))
                .cloned()
                .ok_or(StoreError::NotFound {
                    reference: reference.to_string(),
                })
        }
    }

    #[test]
    fn processes_remote_item() {
        let record = RecordId::new("abc123");
        let store = FakeStore::new()
            .with_record("https://example.com/repo", "main", record.clone());

        let config = Config {
            name: "test".into(),
            items: vec![Item {
                id: ItemId::new("my-item").unwrap(),
                kind: ItemKind::Remote {
                    url: "https://example.com/repo".into(),
                    reference: "main".into(),
                },
            }],
        };

        let report = process_all(&config, &store).unwrap();
        assert_eq!(report.entries.len(), 1);
        assert_eq!(report.entries[0].record, record);
    }

    #[test]
    fn skips_local_items() {
        let store = FakeStore::new();
        let config = Config {
            name: "test".into(),
            items: vec![Item {
                id: ItemId::new("local-item").unwrap(),
                kind: ItemKind::Local {
                    path: PathBuf::from("../local"),
                },
            }],
        };

        let report = process_all(&config, &store).unwrap();
        assert!(report.entries.is_empty());
    }
}
```

**Key patterns:**
- Fakes are structs that implement traits with builder-style setup methods.
- Tests read as specifications: setup, act, assert.
- Fakes live in the test module next to the code they test.
- No external mocking framework.

### Integration Tests

```rust
// In tests/process_integration.rs
use my_tool_engine::process_all;

#[test]
fn process_from_real_config_file() {
    // Uses real file I/O against test fixtures
    let repo = FileRepository::new();
    let config = repo.load_config(Path::new("tests/fixtures/config.yaml")).unwrap();
    // ...
}
```

**Key patterns:**
- Integration tests live in `tests/` at workspace root.
- They use real adapters against test fixtures.
- They test the wiring — do the pieces fit together?

## Module Organization

### Flat by Default

```rust
// lib.rs — re-export public API
pub mod config;
pub mod domain;
pub mod error;

pub use config::Config;
pub use domain::{Item, ItemId, ItemKind};
pub use error::{RepoError, ValidationError};
```

Split when a module covers multiple distinct concepts. A 500-line module about one concept is fine; a 200-line module mixing two concepts should split.

```
src/
  lib.rs
  config.rs         -> src/config/mod.rs (or config.rs + config/ dir)
  config/
    parse.rs
    validate.rs
```

### Visibility

- `pub` — Part of the crate's public API. Stable.
- `pub(crate)` — Used across modules within the crate. Not exposed to dependents.
- `pub(super)` — Useful for internal sub-module organization, e.g. when a module is split into sub-files and sub-modules need access to shared helpers.
- Private (default) — Implementation details.

**Rule of thumb:** Start private. Promote to `pub(crate)` when another module needs it. Promote to `pub` only when an external crate needs it.

## Ownership and API Design

### Function Signatures

Choose parameter types based on what the function needs:

```rust
// Borrows — the caller keeps ownership. Use for read-only access.
fn validate(id: &ItemId) -> Result<(), ValidationError> { ... }

// Owned — the function takes ownership. Use when the value will be stored.
fn register(id: ItemId, record: RecordId) -> Entry { ... }

// Mutable borrow — temporary exclusive access.
fn update(store: &mut Vec<Entry>, entry: Entry) { ... }
```

**Guidelines:**
- **Prefer borrowing** (`&T`) when the function only reads the value.
- **Take ownership** (`T`) when the function stores the value in a struct or collection.
- **Use `&mut T`** when the function modifies the value in place.
- **Accept `&str`** instead of `&String`, and `&[T]` instead of `&Vec<T>` — broader types accept more callers.

### When to Clone

Clone when it simplifies the API. Don't fight the borrow checker for negligible performance gains in application code:

```rust
// Fine — clarity over micro-optimization
let name = item.id.clone();
entries.push(ReportEntry { id: name, ... });

// Unnecessary — restructure instead of cloning in a loop
// BAD: data.iter().map(|x| x.clone()).collect()
// GOOD: data.into_iter().collect()  // move instead of clone
```

**Rule of thumb:** Prefer borrowing first. If lifetimes become tangled, clone. Profile before optimizing clones away.

### Lifetime Patterns

For application code, prefer owned types in structs. Lifetimes in structs complicate APIs and often aren't worth it:

```rust
// Prefer: owned types in structs
struct Entry {
    id: ItemId,
    label: String,
}

// Avoid unless performance-critical:
struct EntryRef<'a> {
    id: &'a ItemId,
    label: &'a str,
}
```

Use lifetimes in function signatures when the borrow is straightforward:

```rust
// Fine — clear input-to-output relationship
fn find<'a>(items: &'a [Item], id: &ItemId) -> Option<&'a Item> {
    items.iter().find(|item| &item.id == id)
}

// Lifetime elision handles the common case automatically:
fn first(items: &[Item]) -> Option<&Item> {
    items.first()
}
```

### Naming Conventions

Follow the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) naming patterns. The most common:

| Pattern | Meaning | Example |
|---|---|---|
| `as_str()`, `as_bytes()` | Free borrowed view, no allocation | `fn as_str(&self) -> &str` |
| `to_string()`, `to_vec()` | Expensive conversion, new allocation | `fn to_string(&self) -> String` |
| `into_inner()`, `into_vec()` | Consumes self, returns owned data | `fn into_inner(self) -> String` |
| `is_empty()`, `has_children()` | Boolean predicate, no mutation | `fn is_empty(&self) -> bool` |
| `with_capacity()` | Constructor variant | `fn with_capacity(n: usize) -> Self` |

See [project-reference.md — Naming Conventions](../reference/project-reference.md#naming-conventions) for the full quick-reference table.

## Conversions and Builders

### `From` / `Into`

Implement `From` for infallible conversions between types:

```rust
impl From<RawConfig> for Config {
    fn from(raw: RawConfig) -> Self {
        Config {
            name: raw.name,
            items: Vec::new(), // defaults for optional fields
        }
    }
}

// Now you can use .into():
let config: Config = raw_config.into();
```

Use `TryFrom` when conversion can fail:

```rust
impl TryFrom<RawConfig> for Config {
    type Error = ValidationError;

    fn try_from(raw: RawConfig) -> Result<Self, Self::Error> {
        let items = raw.items
            .into_iter()
            .map(Item::try_from)
            .collect::<Result<Vec<_>, _>>()?;
        Ok(Config { name: raw.name, items })
    }
}
```

### `From` and Error Conversion

`From` powers `?` for error conversion. When you write `#[from]` in a thiserror enum, it generates a `From` impl, which lets `?` automatically convert:

```rust
#[derive(Debug, Error)]
pub enum ProcessError {
    #[error("store error")]
    Store(#[from] StoreError),

    #[error("config error")]
    Config(#[from] RepoError),
}

// Now ? converts automatically:
fn process(store: &impl Store, repo: &impl Repository) -> Result<(), ProcessError> {
    let config = repo.load_config(path)?;  // RepoError -> ProcessError via From
    let record = store.lookup(url, r)?;     // StoreError -> ProcessError via From
    Ok(())
}
```

### Builder Pattern

Use builders for constructing types with many optional fields:

```rust
pub struct ServerConfig {
    host: String,
    port: u16,
    max_connections: usize,
    timeout_secs: u64,
}

pub struct ServerConfigBuilder {
    host: String,
    port: u16,
    max_connections: usize,
    timeout_secs: u64,
}

impl ServerConfigBuilder {
    pub fn new(host: impl Into<String>, port: u16) -> Self {
        Self {
            host: host.into(),
            port,
            max_connections: 100,
            timeout_secs: 30,
        }
    }

    pub fn max_connections(mut self, n: usize) -> Self {
        self.max_connections = n;
        self
    }

    pub fn timeout_secs(mut self, secs: u64) -> Self {
        self.timeout_secs = secs;
        self
    }

    pub fn build(self) -> ServerConfig {
        ServerConfig {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
            timeout_secs: self.timeout_secs,
        }
    }
}
```

Use builders when a type has more than 2-3 optional fields. For simpler types, a `new()` constructor with required fields and `Default` is sufficient.

## Documentation

**Use `cargo doc --open` as a design feedback tool.** If the generated docs look confusing, the API is confusing. Review them before stabilizing public interfaces.

### Doc Comments

Use `///` for public items, `//!` for module-level docs:

```rust
//! # Item processing
//!
//! This module handles processing items from remote sources.

/// Process all items and return a summary report.
///
/// Skips local items — only remote items are processed.
///
/// # Errors
///
/// Returns `ProcessError` if any remote lookup fails.
pub fn process_all(config: &Config, store: &impl Store) -> Result<Report, ProcessError> {
    // ...
}
```

### Doc Examples

Code blocks in doc comments run as tests with `cargo test`:

```rust
/// Parse an item ID from a string.
///
/// ```
/// use my_tool_core::ItemId;
///
/// let id = ItemId::new("widget-1").unwrap();
/// assert_eq!(id.as_str(), "widget-1");
/// ```
pub fn new(id: impl Into<String>) -> Result<Self, ValidationError> {
    // ...
}
```

### `#[non_exhaustive]`

Use on public enums that may gain variants:

```rust
#[derive(Debug, Error)]
#[non_exhaustive]
pub enum StoreError {
    #[error("not found: {reference}")]
    NotFound { reference: String },

    #[error("connection failed")]
    ConnectionFailed,
}
```

This prevents downstream code from matching exhaustively, allowing you to add variants without a breaking change.

## Configuration Patterns

```rust
use serde::Deserialize;
use std::path::PathBuf;

/// Raw configuration as deserialized from YAML.
#[derive(Debug, Deserialize)]
#[serde(rename_all = "kebab-case")]
struct RawConfig {
    name: String,
    #[serde(default)]
    items: Vec<RawItem>,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "kebab-case")]
struct RawItem {
    name: String,
    url: Option<String>,
    #[serde(rename = "ref")]
    reference: Option<String>,
    local: Option<PathBuf>,
}

impl TryFrom<RawConfig> for Config {
    type Error = RepoError;

    fn try_from(raw: RawConfig) -> Result<Self, Self::Error> {
        let items = raw.items
            .into_iter()
            .map(Item::try_from)
            .collect::<Result<Vec<_>, _>>()?;

        Ok(Config {
            name: raw.name,
            items,
        })
    }
}
```

**Key patterns:**
- Separate raw (serde) types from domain types. Deserialize into raw, validate via `TryFrom` into domain.
- `rename_all = "kebab-case"` for YAML/TOML conventions.
- `#[serde(default)]` for optional fields.
- Validation happens at the boundary between raw and domain, not during deserialization.

## Sources

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — Naming conventions, type design, interoperability
- [The Rust Programming Language](https://doc.rust-lang.org/book/) — Ownership, traits, error handling fundamentals
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) — Community pattern catalog
- [Rust for Rustaceans](https://rust-for-rustaceans.com/) — Intermediate API design, error handling trade-offs
