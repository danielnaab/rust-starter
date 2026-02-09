---
status: accepted
---

# ADR 003: Trait-Based Dependency Injection

## Context

Business logic needs to work with external resources (git repos, filesystems, network) without being coupled to specific implementations. We need a dependency injection strategy.

## Decision

Define traits in the crate that needs the capability. Accept trait bounds on functions. Use concrete types only at the binary level.

```rust
// Core defines what it needs
pub trait ConfigReader {
    fn read_config(&self, path: &Path) -> Result<ProjectConfig, ConfigError>;
}

// Engine uses the trait
pub fn load_and_validate(
    reader: &impl ConfigReader,
    path: &Path,
) -> Result<ValidatedConfig, EngineError> {
    let config = reader.read_config(path)?;
    validate(config)
}

// Binary provides the concrete type
fn main() -> anyhow::Result<()> {
    let reader = FileConfigReader::new();
    let config = load_and_validate(&reader, Path::new("graft.yaml"))?;
    // ...
}
```

## Rationale

- **Testability** — Swap real implementations for fakes in tests.
- **Decoupling** — Core logic doesn't know about git libraries, file systems, or network.
- **Rust idiom** — Traits are Rust's native abstraction mechanism.
- **Zero cost** — `impl Trait` is monomorphized at compile time. No vtable overhead.

## Static vs Dynamic Dispatch

**Default: Static dispatch (`impl Trait` / generics)**

```rust
fn resolve(resolver: &impl RefResolver) -> Result<Resolution, Error> { ... }
// or
fn resolve<R: RefResolver>(resolver: &R) -> Result<Resolution, Error> { ... }
```

Use generics when the concrete type is known at compile time (which is almost always).

**Exception: Dynamic dispatch (`dyn Trait`)**

```rust
fn process_all(handlers: &[Box<dyn Handler>]) -> Result<(), Error> { ... }
```

Use `dyn Trait` only when you need heterogeneous collections — storing different concrete types together. This is rare.

## Alternatives Considered

**No abstraction:** Pass concrete types everywhere. Simpler but untestable without real resources. Only acceptable for scripts or prototypes.

**DI frameworks:** Crates like `shaku` or `inject`. Add complexity and magic. Not worth it for the project sizes we work with.

**Function pointers:** `fn(path: &Path) -> Result<Config>`. Works but loses access to state and is harder to compose.
