---
status: stable
---

# Agent Reference — Rust Starter Patterns

## Project Type

Guidance repository for idiomatic Rust patterns. No compilable code — documentation with inline examples.

## Key Rules

1. **Library code never panics** — Return `Result<T, E>`. Use `thiserror` for error types.
2. **Binaries use anyhow** — Add `.context()` for human-readable error chains.
3. **Traits define boundaries** — Core defines what it needs. Adapters provide it.
4. **Newtypes for domain identity** — `ItemId(String)` not bare `String`.
5. **Generics by default** — Use `impl Trait` or `T: Trait`. Use `dyn Trait` only for heterogeneous collections.
6. **Fakes over mocks** — Implement traits with controlled test behavior. No mockall.
7. **Flat modules** — Split when a module covers multiple distinct concepts, not at an arbitrary line count.
8. **Workspace dependencies** — Declare versions once in root `Cargo.toml`.

## Where to Find What

| Need | Go to |
|---|---|
| Full code patterns and architecture | [architecture.md](architecture/architecture.md) |
| Why a decision was made | [decisions/](decisions/) (ADRs 001-008, excluding 006) |
| Step-by-step recipes | [development.md](guides/development.md) |
| Project setup | [getting-started.md](guides/getting-started.md) |
| Cargo.toml templates, tooling commands | [project-reference.md](reference/project-reference.md) |
| Type design (invalid states, newtypes) | [architecture.md — Type Design Principles](architecture/architecture.md#type-design-principles) |
| Ownership, lifetimes, `From`/`Into` | [architecture.md — Ownership and API Design](architecture/architecture.md#ownership-and-api-design) |
| Naming conventions (`as_`/`to_`/`into_`) | [project-reference.md — Naming Conventions](reference/project-reference.md#naming-conventions) |
| Error conversion with `?` | [architecture.md — Conversions and Builders](architecture/architecture.md#conversions-and-builders) |

## Common Operations

### Add a new domain type
1. Define struct in core module — see [architecture.md#domain-types](architecture/architecture.md#domain-types)
2. Derive `Debug, Clone, PartialEq, Eq` at minimum
3. Use newtype wrapper if it represents an identity
4. Add validation in constructor, return `Result`

### Add a new trait (port)
1. Define trait in core module — see [architecture.md#traits-ports](architecture/architecture.md#traits-ports)
2. Keep it small — one capability per trait
3. Choose receiver by semantics: `&self` for reads, `&mut self` for mutations
4. Error type should be the core error enum

### Add a new adapter
1. Create struct implementing the trait
2. Place in engine crate or a dedicated adapter crate (not in core)
3. Write integration test in `tests/`

### Add a CLI subcommand
1. Add variant to the `Command` enum (clap derive)
2. Add match arm in dispatch function
3. Call engine function, format output
4. See [architecture.md#binary-patterns](architecture/architecture.md#binary-patterns)

## Authority

- **Canonical:** ADRs in `docs/decisions/` and architecture in `docs/architecture/`
- **Interpretive:** Guides in `docs/guides/` and this file
- **When in doubt:** Follow the pattern shown in the architecture doc
