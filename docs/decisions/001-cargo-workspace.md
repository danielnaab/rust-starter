---
status: accepted
---

# ADR 001: Cargo Workspace Layout

## Context

Rust projects can be a single crate or a workspace of multiple crates. We need to decide how to structure multi-component projects like graft (core types + engine + CLI) and grove (core + engine + TUI).

## Decision

Use cargo workspaces with a `crates/` directory for library crates and root-level `src/` for the primary binary.

```toml
# Cargo.toml (workspace root)
[workspace]
resolver = "2"
members = ["crates/*"]

[workspace.package]
edition = "2021"
license = "MIT"
rust-version = "1.75"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
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

## Rationale

- **Shared dependency versions** — `[workspace.dependencies]` prevents version skew across crates.
- **Independent compilation** — Changes to the CLI don't recompile core logic.
- **Clear boundaries** — Each crate has an explicit public API and dependency list.
- **Consistent metadata** — `[workspace.package]` shares edition, license, MSRV.
- **`resolver = "2"`** — Required for correct feature unification in workspaces.

## Alternatives Considered

**Single crate with modules:** Simpler for small projects, but boundaries become suggestions rather than enforced. Acceptable for projects under ~2000 lines.

**Separate repositories:** Maximum isolation but painful for coordinated changes. Only warranted for truly independent libraries.

## When to Split Crates

Split when:
- Two components have genuinely different dependency trees
- Compile time matters and changes are localized
- A library is reusable across projects (e.g., git operations)

Don't split when:
- The project is small and boundaries would be ceremony
- You're splitting speculatively "in case we need it later"
