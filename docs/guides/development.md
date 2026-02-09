---
status: stable
---

# Development Guide

Step-by-step recipes for common development tasks. Each recipe shows the key steps with brief snippets — see [architecture.md](../architecture/architecture.md) for full patterns.

## Add a Domain Type

**When:** You need a new concept in the core module (entity, value object, identifier).

### Steps

1. **Define the type** with validation in the constructor. For identifiers, use the newtype pattern — see [architecture.md#domain-types](../architecture/architecture.md#domain-types) for the full example.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct RepoUrl(String);

impl RepoUrl {
    pub fn new(url: impl Into<String>) -> Result<Self, ValidationError> {
        let url = url.into();
        if !url.contains("://") && !url.starts_with("git@") {
            return Err(ValidationError::invalid_format("URL", "must be a valid URL"));
        }
        Ok(Self(url))
    }

    pub fn as_str(&self) -> &str { &self.0 }
}
```

2. **Re-export from lib.rs:** `pub use domain::RepoUrl;`

3. **Write tests** — validation of valid/invalid inputs.

4. **Verify:** `cargo test` (or `cargo test -p my-tool-core` for workspaces).

## Add a Trait (Port)

**When:** The engine needs a capability that requires I/O or external resources.

### Steps

1. **Define the trait** in the core module — see [architecture.md#traits-ports](../architecture/architecture.md#traits-ports).

```rust
pub trait MetadataFetcher {
    fn fetch(&self, url: &RepoUrl) -> Result<Metadata, StoreError>;
}
```

2. **Use the trait** in engine code via `&impl MetadataFetcher`.

3. **Write a fake** for testing — see [architecture.md#testing-patterns](../architecture/architecture.md#testing-patterns) for the full fake pattern.

4. **Implement concretely** when ready (adapter in engine crate or dedicated crate).

5. **Wire in the binary** — construct the concrete type and pass it.

## Add a CLI Subcommand

**When:** Exposing a new operation to users.

### Steps

1. **Add a variant** to the `Command` enum:

```rust
/// Check if items are up to date
Check {
    /// Show details for each item
    #[arg(long)]
    verbose: bool,
},
```

2. **Add a handler function** that constructs adapters, calls engine, and formats output — see [architecture.md#binary-patterns](../architecture/architecture.md#binary-patterns) for the full pattern.

3. **Wire in main:** add a match arm.

4. **Test:** `cargo run -- check --help` to verify help text.

## Add an Error Variant

**When:** A new failure case arises in library code.

### Steps

1. **Add the variant** to the relevant error enum:

```rust
#[error("authentication failed for {url}")]
AuthFailed { url: String },
```

2. **Add a constructor** if the variant has multiple fields.

3. **Handle it** — the compiler will tell you about non-exhaustive match arms.

See [architecture.md#error-types](../architecture/architecture.md#error-types) for full error design patterns.

## Run Quality Checks

```bash
# Full check suite
cargo fmt --all -- --check          # Formatting
cargo clippy --all-targets -- -D warnings   # Linting
cargo test --all                    # Tests
cargo doc --no-deps                 # Documentation builds

# Quick iteration (workspace projects)
cargo test -p my-tool-engine        # Test one crate
cargo clippy -p my-tool-core        # Lint one crate
```

## Dependency Management

### Add a dependency

For single-crate projects, add directly to `[dependencies]`.

For workspaces:

```toml
# 1. Add to workspace dependencies (Cargo.toml root)
[workspace.dependencies]
tokio = { version = "1", features = ["full"] }

# 2. Use in specific crate (crates/my-crate/Cargo.toml)
[dependencies]
tokio.workspace = true
```

### Update dependencies

```bash
cargo update                        # Update all deps within semver bounds
cargo update -p specific-crate      # Update one dependency
```

## Sources

- [architecture.md](../architecture/architecture.md)
