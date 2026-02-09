---
status: stable
---

# Development Guide

Step-by-step recipes for common development tasks.

## Add a Domain Type

**When:** You need a new concept in the core crate (entity, value object, identifier).

### Steps

1. **Define the type** in `crates/project-core/src/`:

```rust
// domain.rs (or a new file if large)

/// A repository URL pointing to a git remote.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct RepoUrl(String);

impl RepoUrl {
    pub fn new(url: impl Into<String>) -> Result<Self, ValidationError> {
        let url = url.into();
        if !url.contains("://") && !url.starts_with("git@") {
            return Err(ValidationError::invalid_format(
                "repository URL",
                "must be a valid git URL",
            ));
        }
        Ok(Self(url))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl fmt::Display for RepoUrl {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(&self.0)
    }
}
```

2. **Re-export from lib.rs:**

```rust
pub use domain::RepoUrl;
```

3. **Write tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_https_url() {
        let url = RepoUrl::new("https://github.com/user/repo").unwrap();
        assert_eq!(url.as_str(), "https://github.com/user/repo");
    }

    #[test]
    fn accepts_ssh_url() {
        assert!(RepoUrl::new("git@github.com:user/repo").is_ok());
    }

    #[test]
    fn rejects_bare_path() {
        assert!(RepoUrl::new("just/a/path").is_err());
    }
}
```

4. **Verify:** `cargo test -p project-core`

## Add a Trait (Port)

**When:** The engine needs a capability that requires I/O or external resources.

### Steps

1. **Define the trait** in the core crate:

```rust
// In crates/project-core/src/lib.rs (or a ports.rs file)

/// Fetch repository metadata from a remote.
pub trait RepoFetcher {
    fn fetch_metadata(&self, url: &RepoUrl) -> Result<RepoMetadata, GitError>;
}
```

2. **Use the trait** in the engine:

```rust
// In crates/project-engine/src/lib.rs

pub fn check_status(
    fetcher: &impl RepoFetcher,
    deps: &[Dependency],
) -> Result<Vec<StatusReport>, EngineError> {
    deps.iter()
        .map(|dep| {
            let metadata = fetcher.fetch_metadata(&dep.url)?;
            Ok(StatusReport {
                name: dep.name.clone(),
                up_to_date: metadata.head == dep.locked_commit,
            })
        })
        .collect()
}
```

3. **Write a fake** for testing:

```rust
#[cfg(test)]
mod tests {
    struct FakeFetcher {
        metadata: HashMap<RepoUrl, RepoMetadata>,
    }

    impl FakeFetcher {
        fn new() -> Self { Self { metadata: HashMap::new() } }

        fn with_repo(mut self, url: RepoUrl, metadata: RepoMetadata) -> Self {
            self.metadata.insert(url, metadata);
            self
        }
    }

    impl RepoFetcher for FakeFetcher {
        fn fetch_metadata(&self, url: &RepoUrl) -> Result<RepoMetadata, GitError> {
            self.metadata.get(url).cloned()
                .ok_or(GitError::NotFound { url: url.clone() })
        }
    }
}
```

4. **Implement concretely** (adapter) when ready:

```rust
// In crates/project-engine/src/git.rs (or a separate adapter crate)

pub struct GitRepoFetcher;

impl RepoFetcher for GitRepoFetcher {
    fn fetch_metadata(&self, url: &RepoUrl) -> Result<RepoMetadata, GitError> {
        // Real git operations here
    }
}
```

5. **Wire in the binary:**

```rust
// src/main.rs
let fetcher = GitRepoFetcher;
let status = check_status(&fetcher, &config.dependencies)?;
```

## Add a CLI Subcommand

**When:** Exposing a new operation to users.

### Steps

1. **Add a variant** to the `Command` enum:

```rust
#[derive(Subcommand)]
enum Command {
    // ... existing commands ...

    /// Check if dependencies are up to date
    Check {
        /// Show details for each dependency
        #[arg(long)]
        verbose: bool,
    },
}
```

2. **Add a handler function:**

```rust
fn cmd_check(json: bool, verbose: bool) -> anyhow::Result<()> {
    let reader = FileConfigReader::new();
    let config = reader.read_config(Path::new("graft.yaml"))
        .context("failed to load config")?;

    let fetcher = GitRepoFetcher;
    let status = check_status(&fetcher, &config.dependencies)
        .context("status check failed")?;

    if json {
        println!("{}", serde_json::to_string_pretty(&status)?);
    } else {
        for report in &status {
            let icon = if report.up_to_date { "ok" } else { "outdated" };
            println!("{}: {}", report.name, icon);
        }
    }

    Ok(())
}
```

3. **Wire in main:**

```rust
match cli.command {
    // ...
    Command::Check { verbose } => cmd_check(cli.json, verbose),
}
```

4. **Test:** `cargo run -- check --help` to verify help text.

## Add an Error Variant

**When:** A new failure case arises in library code.

### Steps

1. **Add the variant** to the relevant error enum:

```rust
#[derive(Debug, Error)]
pub enum GitError {
    // ... existing variants ...

    #[error("authentication failed for {url}")]
    AuthFailed { url: RepoUrl },
}
```

2. **Add a constructor** if the variant has multiple fields:

```rust
impl GitError {
    pub fn auth_failed(url: &RepoUrl) -> Self {
        Self::AuthFailed { url: url.clone() }
    }
}
```

3. **Handle it** where it can occur. The compiler will tell you about non-exhaustive match arms.

## Run Quality Checks

```bash
# Full check suite
cargo fmt --all -- --check          # Formatting
cargo clippy --all-targets -- -D warnings   # Linting
cargo test --all                    # Tests
cargo doc --no-deps                 # Documentation builds

# Quick iteration
cargo test -p project-engine        # Test one crate
cargo clippy -p project-core        # Lint one crate
```

## Dependency Management

### Add a dependency

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
