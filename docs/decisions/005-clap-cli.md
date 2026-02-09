---
status: accepted
---

# ADR 005: clap for CLI

## Context

We need a CLI framework for command-line applications. The framework should support subcommands, help generation, and type-safe argument parsing.

## Decision

Use clap with the derive macro for CLI definitions.

## Rationale

- **Type-driven** — CLI structure is defined in types. The compiler catches structural errors.
- **Auto-generated help** — Doc comments become help text. No manual strings.
- **Derive macro** — Minimal boilerplate. Add a field, get an argument.
- **Ecosystem standard** — Most Rust CLI tools use clap. Familiar to contributors.
- **Rich features** — Subcommands, global args, value validation, shell completions, all built in.

## Conventions

1. **Global flags** (`--json`, `--verbose`) use `global = true` so they work with any subcommand.
2. **Doc comments** on structs and variants become help text. Write them as user-facing descriptions.
3. **Required args** are positional. **Optional args** use `--flag` syntax.
4. **Boolean flags** use `--flag` (present = true, absent = false). No `--no-flag`.
5. **Exit codes:** 0 = success, 1 = application error, 2 = usage error (clap handles this).

## Alternatives Considered

**argh:** Simpler, Google-backed. Fewer features, smaller ecosystem. Good for very simple CLIs.

**structopt:** Predecessor to clap derive. Merged into clap 3+. No reason to use separately.

**Manual argument parsing:** Maximum control but tedious. Only justified for extremely performance-sensitive tools.

## See Also

- [architecture.md — Binary Patterns](../architecture/architecture.md#binary-patterns) for the full CLI struct and wiring example
