---
status: accepted
---

# ADR 008: Testing Without Mocks

## Context

Testing business logic that depends on traits (I/O, git, filesystem) requires providing implementations in tests. We need a strategy for test doubles.

## Decision

Write trait implementations by hand ("fakes") with builder-style setup. No mocking frameworks.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// Fake git resolver that returns pre-configured results.
    struct FakeResolver {
        refs: HashMap<String, CommitId>,
        should_fail: bool,
    }

    impl FakeResolver {
        fn new() -> Self {
            Self {
                refs: HashMap::new(),
                should_fail: false,
            }
        }

        fn with_ref(mut self, reference: &str, commit: CommitId) -> Self {
            self.refs.insert(reference.to_string(), commit);
            self
        }

        fn failing(mut self) -> Self {
            self.should_fail = true;
            self
        }
    }

    impl RefResolver for FakeResolver {
        fn resolve_ref(&self, _url: &str, reference: &str) -> Result<CommitId, GitError> {
            if self.should_fail {
                return Err(GitError::NetworkError("fake failure".into()));
            }
            self.refs
                .get(reference)
                .cloned()
                .ok_or(GitError::RefNotFound {
                    reference: reference.to_string(),
                })
        }
    }

    #[test]
    fn resolves_known_ref() {
        let resolver = FakeResolver::new()
            .with_ref("main", CommitId::new("abc123"));

        let result = resolve(&resolver, "https://example.com", "main");
        assert_eq!(result.unwrap(), CommitId::new("abc123"));
    }

    #[test]
    fn returns_error_for_unknown_ref() {
        let resolver = FakeResolver::new();

        let result = resolve(&resolver, "https://example.com", "nonexistent");
        assert!(matches!(result, Err(GitError::RefNotFound { .. })));
    }

    #[test]
    fn propagates_network_errors() {
        let resolver = FakeResolver::new().failing();

        let result = resolve(&resolver, "https://example.com", "main");
        assert!(matches!(result, Err(GitError::NetworkError(_))));
    }
}
```

## Rationale

**Why fakes over mocking frameworks:**

- **Readable** — Fakes are regular Rust code. No macro magic, no DSL to learn.
- **Debuggable** — Step through fake implementations like any other code.
- **Reusable** — The same fake works across multiple test functions. Extract to a shared module when used across test files.
- **Refactor-friendly** — When a trait changes, the compiler tells you exactly which fakes to update. Mock setups silently become stale.
- **Builder pattern** — `.with_ref("main", commit).failing()` reads as a specification of the test scenario.

**Why not mockall:**

- `mockall` generates code via procedural macros — harder to read and debug.
- Mock expectations (`expect_resolve_ref().times(1).returning(...)`) test implementation details, not behavior.
- Mocks encourage testing *how* code interacts with dependencies rather than *what* it produces.
- When a trait changes, mock setups may compile but silently test the wrong thing.

## Shared Fakes

When a fake is used across multiple test files, extract it:

```rust
// tests/common/mod.rs (or tests/fakes.rs)
pub struct FakeResolver { ... }
impl RefResolver for FakeResolver { ... }

// tests/resolve_test.rs
mod common;
use common::FakeResolver;
```

Or for in-crate sharing:

```rust
// src/testing.rs (behind cfg(test))
#[cfg(test)]
pub(crate) mod fakes {
    pub struct FakeResolver { ... }
}
```

## Test Organization

- **Unit tests** — `#[cfg(test)] mod tests` in the same file as the code. Test individual functions.
- **Integration tests** — `tests/` directory. Test multiple components wired together.
- **Test fixtures** — `tests/fixtures/` for sample config files, git repos, etc.
- **Shared fakes** — `tests/common/mod.rs` or `src/testing.rs` behind `#[cfg(test)]`.
