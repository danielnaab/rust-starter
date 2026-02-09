---
status: accepted
---

# ADR 007: Newtype Pattern for Domain Identity

## Context

Domain concepts like item IDs, record IDs, and paths are often represented as bare `String` or `PathBuf`. This loses type safety — the compiler can't distinguish an item ID from a URL.

## Decision

Wrap domain identifiers in newtype structs. Validate in the constructor. Expose the inner value via `as_str()` or `as_path()`.

```rust
/// A unique identifier for an item.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct ItemId(String);

impl ItemId {
    pub fn new(id: impl Into<String>) -> Result<Self, ValidationError> {
        let id = id.into();
        if id.is_empty() {
            return Err(ValidationError::empty_field("item ID"));
        }
        Ok(Self(id))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl fmt::Display for ItemId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(&self.0)
    }
}
```

## Rationale

- **Compile-time safety** — `fn process(id: ItemId)` can't accidentally receive a URL or path. The compiler enforces this.
- **Validation at construction** — Invalid states are impossible. If you have an `ItemId`, it's valid.
- **Self-documenting APIs** — Function signatures reveal intent: `process(id: ItemId, url: RepoUrl)` vs `process(id: String, url: String)`.
- **Zero runtime cost** — Newtypes have the same representation as the inner type. The wrapper exists only at compile time.

## When to Use Newtypes

**Use for:**
- Identifiers (names, IDs, hashes)
- Validated strings (URLs, paths with constraints)
- Domain quantities that shouldn't be confused (line numbers vs column numbers)

**Don't use for:**
- Strings that are just passed through without interpretation
- Types that would need constant unwrapping in every function
- Internal implementation details

## Derive Guidelines

```rust
// Minimum: Debug, Clone
#[derive(Debug, Clone)]

// For collection keys: add Eq + Hash
#[derive(Debug, Clone, PartialEq, Eq, Hash)]

// For serialization: add Serialize/Deserialize
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(transparent)]  // Serialize as the inner type
```

Use `#[serde(transparent)]` so newtypes serialize as their inner value, not as a struct wrapper.

## Alternatives Considered

**Bare primitives:** Simpler but error-prone. Common source of bugs in larger codebases.

**Type aliases:** `type Id = String` — provides documentation but no compile-time safety. The compiler still accepts any `String`.

**Validated smart constructors without newtypes:** Validates but doesn't prevent passing raw strings to functions that expect validated ones.

## See Also

- [architecture.md — Domain Types](../architecture/architecture.md#domain-types) for full newtype examples in context
