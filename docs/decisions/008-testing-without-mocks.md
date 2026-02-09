---
status: accepted
---

# ADR 008: Testing Without Mocks

## Context

Testing business logic that depends on traits (I/O, network, filesystem) requires providing implementations in tests. We need a strategy for test doubles.

## Decision

Write trait implementations by hand ("fakes") with builder-style setup. No mocking frameworks.

```rust
struct FakeStore {
    records: HashMap<String, RecordId>,
    should_fail: bool,
}

impl FakeStore {
    fn new() -> Self {
        Self { records: HashMap::new(), should_fail: false }
    }

    fn with_record(mut self, reference: &str, record: RecordId) -> Self {
        self.records.insert(reference.to_string(), record);
        self
    }

    fn failing(mut self) -> Self {
        self.should_fail = true;
        self
    }
}
```

## Rationale

**Why fakes over mocking frameworks:**

- **Readable** — Fakes are regular Rust code. No macro magic, no DSL to learn.
- **Debuggable** — Step through fake implementations like any other code.
- **Reusable** — The same fake works across multiple test functions. Extract to a shared module when used across test files.
- **Refactor-friendly** — When a trait changes, the compiler tells you exactly which fakes to update. Mock setups silently become stale.
- **Builder pattern** — `.with_record("main", record).failing()` reads as a specification of the test scenario.

**Why not mockall:**

- `mockall` generates code via procedural macros — harder to read and debug.
- Mock expectations (`expect_lookup().times(1).returning(...)`) test implementation details, not behavior.
- Mocks encourage testing *how* code interacts with dependencies rather than *what* it produces.
- When a trait changes, mock setups may compile but silently test the wrong thing.

## Shared Fakes

When a fake is used across multiple test files, extract it:

```rust
// tests/common/mod.rs
pub struct FakeStore { ... }

// tests/process_test.rs
mod common;
use common::FakeStore;
```

Or for in-crate sharing:

```rust
// src/testing.rs (behind cfg(test))
#[cfg(test)]
pub(crate) mod fakes {
    pub struct FakeStore { ... }
}
```

## Test Organization

- **Unit tests** — `#[cfg(test)] mod tests` in the same file as the code. Test individual functions.
- **Integration tests** — `tests/` directory. Test multiple components wired together.
- **Test fixtures** — `tests/fixtures/` for sample config files, etc.
- **Shared fakes** — `tests/common/mod.rs` or `src/testing.rs` behind `#[cfg(test)]`.

## See Also

- [architecture.md — Testing Patterns](../architecture/architecture.md#testing-patterns) for full fake and test examples
