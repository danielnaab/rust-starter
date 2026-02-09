---
title: Copier Template Design
status: stable
---

# Copier Template Design

This repository is a [Copier](https://copier.readthedocs.io/) template. Users run `copier copy` to generate projects from it.

## Template Variables

### User Inputs

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `project_name` | str | "My Rust Project" | Human-readable name |
| `crate_name` | str | (derived) | Cargo crate name (kebab-case) |
| `author_name` | str | "Your Name" | Author |
| `author_email` | str | "you@example.com" | Email (validated) |
| `include_example_code` | bool | true | Full working examples |
| `include_github_actions` | bool | true | CI workflow |
| `license` | choice | "MIT" | MIT, Apache-2.0, BSD-3-Clause, GPL-3.0, Proprietary |

### Computed

- `module_name` — snake_case version of `crate_name`
- `core_crate` / `engine_crate` — workspace crate names
- `core_module` / `engine_module` — Rust module names (underscored)
- `current_year` — auto-generated

## What Stays Fixed

- Architecture pattern (workspace, library-first, trait-based DI)
- Crate structure (`crates/*` for libraries, root `src/` for binary)
- Tooling choices (clippy, rustfmt, thiserror, anyhow, clap)
- Testing strategy (fakes over mocks)

## Conditional Content

Files use `{% if include_example_code %}` to include or omit working examples. When disabled, files contain skeleton code with comments pointing to architecture documentation.

Conditional features:
- `include_github_actions` — `.github/workflows/ci.yml` workflow

## Post-Generation

Copier runs automatically after generation:
1. `git init` — initialize repository
2. `git add .` && `git commit` — initial commit

No `cargo build` step — first build happens when the user runs it.

## Template Updates

Generated projects can pull template updates:

```bash
copier update --trust
```

Protected files (`_skip_if_exists`): `src/main.rs`, core domain/error/traits files, engine service file, tests, `README.md`, `docs/agents.md`, `knowledge-base.yaml`

## Sources

- [copier.yml](../../copier.yml) — template configuration
- [Copier documentation](https://copier.readthedocs.io/)
- [Dependency pattern](dependency-pattern.md) — template-as-dependency approach
