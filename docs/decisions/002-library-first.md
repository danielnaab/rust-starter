---
status: stable
---

# ADR 002: Library-First Architecture

## Context

Rust applications can put logic directly in `main.rs` or factor it into library crates that binaries consume. We need a standard approach.

## Decision

All business logic lives in library code (`lib.rs`). Binary crates (`main.rs`) are thin wrappers that parse arguments, construct concrete types, call library functions, and format output.

```
Binary responsibility:
  1. Parse CLI arguments (clap)
  2. Construct concrete adapter instances
  3. Call engine functions
  4. Format and print results
  5. Map errors to exit codes

Everything else is in the library.
```

## Rationale

- **Testable without CLI** — Library code is tested directly with unit tests. No need to spawn processes or parse stdout.
- **Reusable** — The same engine can serve a CLI, a TUI, a daemon, or a language server.
- **Faster feedback** — `cargo test -p my-engine` runs instantly without building the full binary.
- **Clean separation** — Forces you to think about what's a "decision" (binary) vs what's "logic" (library).

## What Belongs Where

| Binary | Library |
|---|---|
| Argument parsing | Business logic |
| Output formatting | Data transformation |
| Exit code mapping | Error creation |
| Concrete type construction | Trait definitions |
| Signal handling | State management |
| Environment variable reading | Configuration parsing |

## Anti-Patterns

- **Logic in `main.rs`** — If you're writing `if`/`match` on domain concepts in main, it belongs in the library.
- **Library depending on clap** — The library should never know it's being called from a CLI.
- **Formatting in the library** — The library returns data. The binary decides how to present it.

## Sources

- [architecture.md — Binary Patterns](../architecture/architecture.md#binary-patterns) for the full binary example
