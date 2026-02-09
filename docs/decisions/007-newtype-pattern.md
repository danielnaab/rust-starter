---
status: accepted
---

# ADR 007: Newtype Pattern for Domain Identity

## Context

Domain concepts like dependency names, commit IDs, and repository paths are often represented as bare `String` or `PathBuf`. This loses type safety — the compiler can't distinguish a dependency name from a repository URL.

## Decision

Wrap domain identifiers in newtype structs. Validate in the constructor. Expose the inner value via `as_str()` or `as_path()`.

```rust
/// A dependency name as declared in graft.yaml.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct DependencyName(String);

impl DependencyName {
    /// Create a new dependency name, validating format.
    pub fn new(name: impl Into<String>) -> Result<Self, ValidationError> {
        let name = name.into();
        if name.is_empty() {
            return Err(ValidationError::empty_field("dependency name"));
        }
        Ok(Self(name))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl fmt::Display for DependencyName {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(&self.0)
    }
}
```

## Rationale

- **Compile-time safety** — `fn resolve(name: DependencyName)` can't accidentally receive a URL or path. The compiler enforces this.
- **Validation at construction** — Invalid states are impossible. If you have a `DependencyName`, it's valid.
- **Self-documenting APIs** — Function signatures reveal intent: `resolve(name: DependencyName, url: RepoUrl)` vs `resolve(name: String, url: String)`.
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

**Type aliases:** `type Name = String` — provides documentation but no compile-time safety. The compiler still accepts any `String`.

**Validated smart constructors without newtypes:** Validates but doesn't prevent passing raw strings to functions that expect validated ones.
