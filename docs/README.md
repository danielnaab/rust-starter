---
status: stable
---

# Rust Starter Documentation

## Navigation

### For Humans

Start with the [architecture overview](architecture/architecture.md) to understand the structural patterns, then browse [decisions](decisions/) for the reasoning behind each choice.

When building features, follow the [development guide](guides/development.md).

### For Agents

See [agents.md](agents.md) for structured operational guidance.

## Document Index

### Architecture
- [Architecture Overview](architecture/architecture.md) — Layers, patterns, dependency flow

### Decisions (ADRs)
- [001: Cargo Workspace](decisions/001-cargo-workspace.md) — Why workspace layout
- [002: Library-First](decisions/002-library-first.md) — Why logic in libs
- [003: Trait-Based DI](decisions/003-trait-based-di.md) — Why traits at boundaries
- [004: Error Handling](decisions/004-error-handling.md) — Why thiserror + anyhow
- [005: clap CLI](decisions/005-clap-cli.md) — Why clap derive
- [006: clippy + rustfmt](decisions/006-clippy-rustfmt.md) — Why both, which lints
- [007: Newtype Pattern](decisions/007-newtype-pattern.md) — Why wrapped domain types
- [008: Testing Without Mocks](decisions/008-testing-without-mocks.md) — Why trait fakes

### Guides
- [Getting Started](guides/getting-started.md) — New project setup
- [Development](guides/development.md) — Adding features step-by-step

### Reference
- [Project Reference](reference/project-reference.md) — Structure, config, tooling
