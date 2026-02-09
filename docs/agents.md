---
status: stable
---

# Agent Reference — Rust Starter Patterns

## Project Type

Guidance repository for idiomatic Rust patterns. No compilable code — documentation with inline examples.

## Core Pattern

```rust
// Domain types in core crate (no I/O, no side effects)
pub struct DependencyName(String);

pub trait Repository {
    fn get(&self, name: &DependencyName) -> Result<Config, CoreError>;
}

// Business logic in engine crate (takes trait bounds)
pub fn resolve(repo: &impl Repository, name: &DependencyName) -> Result<Resolution, EngineError> {
    let config = repo.get(name)?;
    // Pure logic here
    Ok(Resolution::from(config))
}

// Binary wires concrete types and calls engine
fn main() -> anyhow::Result<()> {
    let repo = GitRepository::open(".")?;
    let name = DependencyName::new("my-dep")?;
    let result = resolve(&repo, &name)?;
    println!("{result}");
    Ok(())
}
```

## Key Rules

1. **Library code never panics** — Return `Result<T, E>`. Use `thiserror` for error types.
2. **Binaries use anyhow** — Add `.context()` for human-readable error chains.
3. **Traits define boundaries** — Core defines what it needs. Adapters provide it.
4. **Newtypes for domain identity** — `DependencyName(String)` not bare `String`.
5. **Generics by default** — Use `impl Trait` or `T: Trait`. Use `dyn Trait` only for heterogeneous collections.
6. **Fakes over mocks** — Implement traits with controlled test behavior. No mockall.
7. **Flat modules** — Split files only when they exceed ~300 lines.
8. **Workspace dependencies** — Declare versions once in root `Cargo.toml`.

## File Organization

```
project/
  Cargo.toml              # Workspace root
  crates/
    project-core/         # Domain types, traits, errors
      src/lib.rs
    project-engine/       # Business logic, orchestration
      src/lib.rs
  src/main.rs             # Binary entry point (thin)
  tests/                  # Integration tests
```

## Common Operations

### Add a new domain type
1. Define struct in `crates/project-core/src/`
2. Derive `Debug, Clone, PartialEq, Eq` at minimum
3. Use newtype wrapper if it represents an identity
4. Add validation in constructor, return `Result`

### Add a new trait (port)
1. Define trait in `crates/project-core/src/`
2. Keep it small — one capability per trait
3. Use `&self` methods (no `&mut self` unless mutation is the point)
4. Error type should be the core error enum

### Add a new adapter
1. Create struct implementing the trait
2. Place in engine crate or a dedicated adapter crate
3. Write integration test in `tests/`

### Add a CLI subcommand
1. Add variant to the `Command` enum (clap derive)
2. Add match arm in dispatch function
3. Call engine function, format output

## Authority

- **Canonical:** ADRs in `docs/decisions/` and architecture in `docs/architecture/`
- **Interpretive:** Guides in `docs/guides/` and this file
- **When in doubt:** Follow the pattern shown in the architecture doc
