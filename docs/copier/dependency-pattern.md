---
title: Template as Dependency Pattern
status: stable
---

# Template as Dependency Pattern

Generated projects treat the template as a **dependency** rather than duplicating its documentation.

## Problem

Traditional templates copy all docs into generated projects, creating duplication, drift, and maintenance burden across N projects.

## Solution

1. Template docs stay in the template (single source of truth)
2. Generated projects **link** to template docs via relative paths (`.graft/rust-starter/docs/...`)
3. Knowledge base imports make docs discoverable to agents
4. `copier update` pushes improvements downstream

## File Categories

**Never generate** (template-only):
- `docs/architecture/`, `docs/decisions/`, `docs/guides/`, `docs/reference/`, `docs/copier/`

**Generate once, protect** (`_skip_if_exists`):
- `README.md`, `docs/agents.md`, project `knowledge-base.yaml`
- Domain code: `domain.rs`, `error.rs`, `traits.rs`, `service.rs`
- Tests: `tests/common/mod.rs`, `tests/integration_test.rs`
- Binary: `src/main.rs`

**Always update**:
- `Cargo.toml`, `rustfmt.toml`, infrastructure files

## Agent Discovery Flow

1. Agent reads project `knowledge-base.yaml`
2. Follows import to `.graft/rust-starter/knowledge-base.yaml`
3. Discovers template docs (architecture, decisions, guides)
4. Applies patterns to project-specific code

## When Not to Use

- Projects deployed without template access
- Template is unstable with frequent breaking changes
- Projects that significantly fork the patterns

## Sources

- [Template design](template-design.md) — Copier configuration
- [copier.yml](../../copier.yml) — template configuration
- [Copier documentation](https://copier.readthedocs.io/)
