---
status: accepted
---

# ADR 006: clippy + rustfmt for Code Quality

## Context

We need consistent code formatting and lint enforcement across all Rust crates.

## Decision

Use `rustfmt` for formatting and `clippy` for linting. Both are built into the Rust toolchain.

### rustfmt Configuration

```toml
# rustfmt.toml (workspace root)
edition = "2021"
max_width = 100
use_field_init_shorthand = true
```

Keep configuration minimal. The default rustfmt style is well-designed — override only what genuinely improves readability.

### clippy Configuration

```toml
# Cargo.toml or .cargo/config.toml
[lints.clippy]
# Deny these categories
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }

# Allow specific pedantic lints that conflict with our patterns
module_name_repetitions = "allow"     # We prefer explicit module::TypeName
must_use_candidate = "allow"          # Too noisy for our codebase
missing_errors_doc = "allow"          # Error types are self-documenting
```

### CI Enforcement

```bash
# Check formatting (fails on diff)
cargo fmt --all -- --check

# Run clippy (fails on warnings)
cargo clippy --all-targets --all-features -- -D warnings

# Run tests
cargo test --all
```

## Rationale

- **Built-in tools** — No external dependencies. Ship with `rustup`.
- **Zero debate** — rustfmt is opinionated by design. Teams don't argue about style.
- **Catch bugs early** — clippy catches common mistakes, performance issues, and unidiomatic code.
- **Pedantic is worth it** — The `pedantic` lint group catches subtle issues. Allow-list the few that don't fit.
- **CI enforcement** — Format and lint checks in CI prevent style drift.

## Alternatives Considered

**No linting:** Faster CI but accumulates technical debt. Not acceptable for long-lived projects.

**Custom lint rules:** `dylint` or `rust-analyzer` custom lints. Too complex for our needs. clippy covers >99% of useful lint cases.
