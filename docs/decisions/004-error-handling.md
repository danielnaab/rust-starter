---
status: stable
---

# ADR 004: Error Handling Strategy

## Context

Rust has two error handling paradigms: `Result<T, E>` for recoverable errors and `panic!` for unrecoverable ones. We need conventions for when to use each, and what error types to define.

## Decision

- **Libraries:** Custom error enums with `thiserror`. Never panic.
- **Binaries:** `anyhow::Result` with `.context()` for error chains. Panic only in truly unreachable code.

## Rationale

**Why thiserror for libraries:**
- Callers can match on specific error variants
- Structured fields enable programmatic error handling
- `#[from]` automates error conversion
- Zero runtime overhead (derive macro generates at compile time)

**Why anyhow for binaries:**
- Errors are displayed to humans, not matched programmatically
- `.context()` builds readable error chains: `"failed to load config: config file not found: config.yaml"`
- Erases type complexity at the boundary where it no longer matters

**Why never panic in libraries:**
- Panics are unrecoverable. Callers can't handle them gracefully.
- A library panic kills the entire process — unacceptable when used in a daemon or TUI.
- `Result` makes failure explicit in the function signature.

## Error Design Guidelines

1. **One error enum per domain concern** — `RepoError`, `StoreError`, `ValidationError`. Not one giant `Error` enum.

2. **Structured fields over formatted messages** — `NotFound { path }` not `NotFound(String)`. Callers can extract the path.

3. **Use `#[from]` sparingly** — Only when the conversion is unambiguous. If an `io::Error` could mean different things in different contexts, wrap it explicitly.

4. **Constructor methods for ergonomics:**
   ```rust
   impl ValidationError {
       pub fn empty_field(name: &'static str) -> Self {
           Self::EmptyField { field: name }
       }
   }
   ```

5. **Convert at boundaries:**
   ```rust
   #[derive(Debug, Error)]
   pub enum ProcessError {
       #[error("configuration error")]
       Config(#[from] RepoError),

       #[error("store operation failed")]
       Store(#[from] StoreError),
   }
   ```

## Alternatives Considered

**`anyhow` everywhere:** Simpler but library callers can't match on specific errors. Inappropriate for reusable code.

**`std::error::Error` manually:** Works but verbose. `thiserror` generates the same code with less boilerplate.

**`eyre` instead of `anyhow`:** Similar capabilities. `anyhow` has larger ecosystem adoption and simpler API.

## Sources

- [architecture.md — Error Types](../architecture/architecture.md#error-types) for full error type examples
- [architecture.md — Conversions and Builders](../architecture/architecture.md#from-and-error-conversion) for how `From` powers `?`
- [thiserror](https://docs.rs/thiserror)
- [anyhow](https://docs.rs/anyhow)
