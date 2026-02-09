---
status: accepted
---

# ADR 003: Trait-Based Dependency Injection

## Context

Business logic needs to work with external resources (filesystems, network, databases) without being coupled to specific implementations. We need a dependency injection strategy.

## Decision

Define traits in the crate that needs the capability. Accept trait bounds on functions. Use concrete types only at the binary level.

```rust
// Core defines what it needs
pub trait Store {
    fn lookup(&self, url: &str, reference: &str) -> Result<RecordId, StoreError>;
}

// Engine uses the trait
pub fn process(store: &impl Store, url: &str, reference: &str) -> Result<Report, ProcessError> {
    let record = store.lookup(url, reference)?;
    // ...
}
```

## Rationale

- **Testability** — Swap real implementations for fakes in tests.
- **Decoupling** — Core logic doesn't know about specific libraries, file systems, or network.
- **Rust idiom** — Traits are Rust's native abstraction mechanism.
- **Zero cost** — `impl Trait` is monomorphized at compile time. No vtable overhead.

## Static vs Dynamic Dispatch

**Default: Static dispatch (`impl Trait` / generics)**

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

## See Also

- [architecture.md — Traits (Ports)](../architecture/architecture.md#traits-ports) for trait definition patterns
- [architecture.md — Engine Crate Patterns](../architecture/architecture.md#engine-crate-patterns) for usage with trait bounds
