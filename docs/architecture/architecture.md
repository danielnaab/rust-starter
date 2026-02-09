---
status: stable
---

# Architecture

Patterns for structuring Rust applications in the graft ecosystem.

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

## Crate Structure

A typical project has 2-3 crates in a cargo workspace:

```
project/
  Cargo.toml                    # [workspace] root
  crates/
    project-core/               # Types, traits, errors
      Cargo.toml
      src/
        lib.rs                  # pub use re-exports
        config.rs               # Configuration types
        domain.rs               # Domain model
        error.rs                # Error types
    project-engine/             # Business logic
      Cargo.toml                # depends on project-core
      src/
        lib.rs
        resolve.rs              # Resolution logic
        git.rs                  # Git adapter (implements core traits)
  src/
    main.rs                     # Binary entry point
  tests/
    resolve_integration.rs      # Integration tests
```

**When to split crates:** Start with core + engine. Add crates when a clear boundary emerges (e.g., `project-git` if git operations become substantial and reusable).

**When not to split:** Don't split preemptively. A single `lib.rs` with modules is fine for small projects. Split when compile times matter or when you need independent versioning.

## Core Crate Patterns

Core contains domain types, trait definitions, and error types. It has minimal dependencies (typically just `serde` and `thiserror`).

### Domain Types

```rust
use std::path::PathBuf;

/// A dependency name as declared in configuration.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct DependencyName(String);

impl DependencyName {
    pub fn new(name: impl Into<String>) -> Result<Self, ValidationError> {
        let name = name.into();
        if name.is_empty() {
            return Err(ValidationError::empty_field("dependency name"));
        }
        if name.contains(' ') {
            return Err(ValidationError::invalid_format(
                "dependency name",
                "must not contain spaces",
            ));
        }
        Ok(Self(name))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

/// Project configuration loaded from graft.yaml.
#[derive(Debug, Clone)]
pub struct ProjectConfig {
    pub name: String,
    pub dependencies: Vec<Dependency>,
}

/// A single dependency declaration.
#[derive(Debug, Clone)]
pub struct Dependency {
    pub name: DependencyName,
    pub source: DependencySource,
}

/// Where a dependency comes from.
#[derive(Debug, Clone)]
pub enum DependencySource {
    Git { url: String, reference: String },
    Local { path: PathBuf },
}
```

**Key patterns:**
- Newtypes for identifiers (`DependencyName`). Validation in constructor.
- Plain structs for data containers. Public fields when there's no invariant to protect.
- Enums for closed sets of alternatives (`DependencySource`).
- Derive `Debug, Clone` on everything. Add `PartialEq, Eq, Hash` when needed for collections.

### Traits (Ports)

```rust
/// Read project configuration from a source.
pub trait ConfigReader {
    fn read_config(&self, path: &Path) -> Result<ProjectConfig, ConfigError>;
}

/// Resolve a git reference to a concrete commit.
pub trait RefResolver {
    fn resolve_ref(&self, url: &str, reference: &str) -> Result<CommitId, GitError>;
}
```

**Key patterns:**
- Traits live in the core crate — the crate that *needs* the capability.
- One capability per trait. Small interfaces are easier to fake in tests.
- `&self` by default. Use `&mut self` only when the operation genuinely mutates internal state.
- Return the core crate's error type, not adapter-specific errors.

### Error Types

```rust
use std::path::PathBuf;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
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

## Engine Crate Patterns

Engine contains business logic. It depends on core and uses trait bounds to stay decoupled from adapters.

```rust
use project_core::{
    ConfigReader, RefResolver, ProjectConfig, Dependency,
    DependencySource, CommitId, ConfigError,
};

/// Resolved state of all dependencies.
#[derive(Debug)]
pub struct Resolution {
    pub resolved: Vec<ResolvedDependency>,
}

#[derive(Debug)]
pub struct ResolvedDependency {
    pub name: DependencyName,
    pub commit: CommitId,
}

/// Resolve all dependencies in a project configuration.
pub fn resolve_all(
    config: &ProjectConfig,
    resolver: &impl RefResolver,
) -> Result<Resolution, ResolveError> {
    let mut resolved = Vec::new();

    for dep in &config.dependencies {
        match &dep.source {
            DependencySource::Git { url, reference } => {
                let commit = resolver.resolve_ref(url, reference)?;
                resolved.push(ResolvedDependency {
                    name: dep.name.clone(),
                    commit,
                });
            }
            DependencySource::Local { .. } => {
                // Local deps don't need resolution
            }
        }
    }

    Ok(Resolution { resolved })
}
```

**Key patterns:**
- Free functions for stateless operations. `impl` blocks when state is involved.
- Take trait bounds as `&impl Trait` for simple cases. Use generic `<R: Trait>` when the type parameter is used in multiple positions.
- Return domain-specific result types, not raw `Result<_, Box<dyn Error>>`.
- Logic is pure where possible — given these inputs, produce this output.

## Binary Patterns

The binary is thin. It parses arguments, constructs concrete types, calls engine functions, and formats output.

```rust
use anyhow::{Context, Result};
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "graft", about = "Semantic dependency management")]
struct Cli {
    #[command(subcommand)]
    command: Command,

    /// Output as JSON
    #[arg(long, global = true)]
    json: bool,
}

#[derive(Subcommand)]
enum Command {
    /// Resolve all dependencies
    Resolve,
    /// Show dependency status
    Status,
}

fn main() -> Result<()> {
    let cli = Cli::parse();

    match cli.command {
        Command::Resolve => cmd_resolve(cli.json),
        Command::Status => cmd_status(cli.json),
    }
}

fn cmd_resolve(json: bool) -> Result<()> {
    let reader = FileConfigReader::new();
    let config = reader
        .read_config(Path::new("graft.yaml"))
        .context("failed to load project configuration")?;

    let resolver = GitRefResolver::new();
    let resolution = resolve_all(&config, &resolver)
        .context("failed to resolve dependencies")?;

    if json {
        println!("{}", serde_json::to_string_pretty(&resolution)?);
    } else {
        for dep in &resolution.resolved {
            println!("{}: {}", dep.name.as_str(), dep.commit);
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
// In crates/project-engine/src/resolve.rs

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashMap;

    struct FakeResolver {
        refs: HashMap<(String, String), CommitId>,
    }

    impl FakeResolver {
        fn new() -> Self {
            Self { refs: HashMap::new() }
        }

        fn with_ref(
            mut self,
            url: &str,
            reference: &str,
            commit: CommitId,
        ) -> Self {
            self.refs.insert((url.to_string(), reference.to_string()), commit);
            self
        }
    }

    impl RefResolver for FakeResolver {
        fn resolve_ref(&self, url: &str, reference: &str) -> Result<CommitId, GitError> {
            self.refs
                .get(&(url.to_string(), reference.to_string()))
                .cloned()
                .ok_or(GitError::RefNotFound {
                    reference: reference.to_string(),
                })
        }
    }

    #[test]
    fn resolves_git_dependency() {
        let commit = CommitId::new("abc123");
        let resolver = FakeResolver::new()
            .with_ref("https://example.com/repo", "main", commit.clone());

        let config = ProjectConfig {
            name: "test".into(),
            dependencies: vec![Dependency {
                name: DependencyName::new("dep").unwrap(),
                source: DependencySource::Git {
                    url: "https://example.com/repo".into(),
                    reference: "main".into(),
                },
            }],
        };

        let resolution = resolve_all(&config, &resolver).unwrap();
        assert_eq!(resolution.resolved.len(), 1);
        assert_eq!(resolution.resolved[0].commit, commit);
    }

    #[test]
    fn skips_local_dependencies() {
        let resolver = FakeResolver::new();
        let config = ProjectConfig {
            name: "test".into(),
            dependencies: vec![Dependency {
                name: DependencyName::new("local-dep").unwrap(),
                source: DependencySource::Local {
                    path: PathBuf::from("../local"),
                },
            }],
        };

        let resolution = resolve_all(&config, &resolver).unwrap();
        assert!(resolution.resolved.is_empty());
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
// In tests/resolve_integration.rs
use project_engine::resolve_all;

#[test]
fn resolve_from_real_config_file() {
    // Uses real file I/O, real git operations against test fixtures
    let reader = FileConfigReader::new();
    let config = reader.read_config(Path::new("tests/fixtures/graft.yaml")).unwrap();
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

pub use config::ProjectConfig;
pub use domain::{Dependency, DependencyName, DependencySource};
pub use error::{ConfigError, ValidationError};
```

Each module is a single file until it grows beyond ~300 lines, then split into a directory:

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
- `pub(super)` — Rarely needed. Used for test helpers.
- Private (default) — Implementation details.

**Rule of thumb:** Start private. Promote to `pub(crate)` when another module needs it. Promote to `pub` only when an external crate needs it.

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
    dependencies: Vec<RawDependency>,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "kebab-case")]
struct RawDependency {
    name: String,
    git: Option<String>,
    #[serde(rename = "ref")]
    reference: Option<String>,
    local: Option<PathBuf>,
}

impl TryFrom<RawConfig> for ProjectConfig {
    type Error = ConfigError;

    fn try_from(raw: RawConfig) -> Result<Self, Self::Error> {
        let dependencies = raw.dependencies
            .into_iter()
            .map(Dependency::try_from)
            .collect::<Result<Vec<_>, _>>()?;

        Ok(ProjectConfig {
            name: raw.name,
            dependencies,
        })
    }
}
```

**Key patterns:**
- Separate raw (serde) types from domain types. Deserialize into raw, validate via `TryFrom` into domain.
- `rename_all = "kebab-case"` for YAML/TOML conventions.
- `#[serde(default)]` for optional fields.
- Validation happens at the boundary between raw and domain, not during deserialization.
