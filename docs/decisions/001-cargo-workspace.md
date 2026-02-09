---
status: accepted
---

# ADR 001: Cargo Workspace Layout

## Context

Rust projects can be a single crate or a workspace of multiple crates. We need to decide how to structure multi-component projects (core types + engine + CLI).

## Decision

Start with a single crate. Evolve into a cargo workspace with a `crates/` directory when genuine boundaries emerge.

For workspaces, use `[workspace.dependencies]` for shared versions and `resolver = "2"` for correct feature unification.

## Rationale

- **Shared dependency versions** — `[workspace.dependencies]` prevents version skew across crates.
- **Independent compilation** — Changes to the CLI don't recompile core logic.
- **Clear boundaries** — Each crate has an explicit public API and dependency list.
- **Consistent metadata** — `[workspace.package]` shares edition, license, MSRV.
- **`resolver = "2"`** — Required for correct feature unification in workspaces.

## Alternatives Considered

**Single crate with modules:** Simpler for small projects, but boundaries become suggestions rather than enforced. This is the recommended starting point — only add workspace structure when you outgrow it.

**Separate repositories:** Maximum isolation but painful for coordinated changes. Only warranted for truly independent libraries.

## When to Split Crates

Split when:
- Two components have genuinely different dependency trees
- Compile time matters and changes are localized
- A library is reusable across projects

Don't split when:
- The project is small and boundaries would be ceremony
- You're splitting speculatively "in case we need it later"

## See Also

- [architecture.md — Crate Structure](../architecture/architecture.md#crate-structure) for full layout examples
