---
status: stable
---

# Rust Starter

Idiomatic Rust patterns and architectural guidance for Rust applications.

## What This Is

A reference repository documenting patterns, conventions, and architectural decisions for building Rust applications. Not a project template — a living guide that agents and developers consult when building.

## Scope

These patterns target **application** architecture (CLI tools, TUI apps, services with business logic). For library-only crates, some patterns (anyhow, clap, binary layer) don't apply. For async services, additional patterns (tokio runtime, async traits) come into play.

## Quick Start

- **Architecture overview:** [docs/architecture/architecture.md](docs/architecture/architecture.md)
- **Why we made each choice:** [docs/decisions/](docs/decisions/)
- **How to add features:** [docs/guides/development.md](docs/guides/development.md)
- **Project structure reference:** [docs/reference/project-reference.md](docs/reference/project-reference.md)

## Core Philosophy

1. **Library-first** — Business logic lives in library crates. Binaries are thin wrappers.
2. **Traits at boundaries** — Define capabilities as traits where modules meet. Implement concretely.
3. **Errors are values** — No panics in library code. `Result<T, E>` everywhere.
4. **Types encode intent** — Newtypes for domain identity. Enums for states. The compiler is your safety net.
5. **Test with fakes, not mocks** — Implement traits with controlled behavior. No mocking frameworks.
6. **Simple until proven otherwise** — Sync before async. Concrete before generic. Flat before nested.

## Architecture at a Glance

```
Binary (thin CLI wrapper)
    |
Engine (business logic library)
    |
Core (domain types, traits, errors)
    ^
Adapters (trait implementations for I/O)
```

Dependencies flow inward. Core has no dependencies on outer layers. Adapters implement core traits without core knowing about them.

## Patterns Covered

| Pattern | Summary | ADR |
|---|---|---|
| Cargo workspace | Shared deps, crate boundaries | [001](docs/decisions/001-cargo-workspace.md) |
| Library-first | Logic in libs, thin binaries | [002](docs/decisions/002-library-first.md) |
| Trait-based DI | Traits at boundaries, no dyn by default | [003](docs/decisions/003-trait-based-di.md) |
| Error handling | thiserror for libs, anyhow for bins | [004](docs/decisions/004-error-handling.md) |
| clap CLI | Derive-based, type-driven CLI | [005](docs/decisions/005-clap-cli.md) |
| Newtype pattern | Zero-cost domain type safety | [007](docs/decisions/007-newtype-pattern.md) |
| Testing without mocks | Trait fakes with builder setup | [008](docs/decisions/008-testing-without-mocks.md) |

## Sources

- [architecture.md](docs/architecture/architecture.md)
