# LLM-Optimized Development Methodology

**Purpose:** This document defines the methodology for structuring AnyFS code so that each component is independently testable, reviewable, replaceable, and fixable—by both humans and LLMs—without requiring full project context.

---

## Core Principle: Context-Independent Components

Every component in AnyFS should be understandable and modifiable with **only local context**. An LLM (or human contributor) should be able to:

1. **Understand** a component by reading only its file + trait definition
2. **Test** a component in isolation without the rest of the system
3. **Fix** a bug by looking at only the failing component + error message
4. **Review** changes without understanding the entire architecture
5. **Replace** a component with an alternative implementation

This is achieved through strict separation of concerns, clear contracts (traits), and self-documenting structure.

---

## The Five Pillars

### 1. Single Responsibility per File

Each file implements **exactly one concept**:

| File           | Implements            | Dependencies          |
| -------------- | --------------------- | --------------------- |
| `fs_read.rs`   | `FsRead` trait        | `FsError`, `Metadata` |
| `quota.rs`     | `Quota<B>` middleware | `Fs` trait            |
| `memory.rs`    | `MemoryBackend`       | `Fs`, `FsLink`, etc.  |
| `iterative.rs` | `IterativeResolver`   | `PathResolver` trait  |

**Why:** An LLM can be given just the file + its dependencies. No need for "the big picture."

### 2. Contract-First Design (Traits as Contracts)

Every component implements a well-defined trait. The trait IS the specification:

```rust
/// Read operations for a virtual filesystem.
/// 
/// # Contract
/// - All methods use `&self` (interior mutability)
/// - Thread-safe: `Send + Sync` required
/// - Errors are always `FsError`, never panic
/// 
/// # Implementor Checklist
/// - [ ] Handle non-existent paths with `FsError::NotFound`
/// - [ ] Handle non-UTF8 content in `read_to_string` with `FsError::InvalidData`
/// - [ ] `metadata()` follows symlinks; use `symlink_metadata()` for link info
pub trait FsRead: Send + Sync {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError>;
    // ...
}
```

**LLM Instruction:** "Implement `FsRead` for `MyBackend`. Follow the contract in the trait doc."

### 3. Isolated Testing (No Integration Dependencies)

Each component has tests that run without external dependencies:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    // Mock only what's needed
    struct MockFs {
        files: HashMap<PathBuf, Vec<u8>>,
    }
    
    #[test]
    fn quota_rejects_oversized_write() {
        let mock = MockFs::new();
        let quota = Quota::new(mock).max_file_size(100);
        
        let result = quota.write(Path::new("/big.txt"), &[0u8; 200]);
        assert!(matches!(result, Err(FsError::FileSizeExceeded { .. })));
    }
}
```

**Why:** LLM can run tests for just the component being fixed. No database, no filesystem, no network.

### 4. Error Messages as Documentation

Errors must contain enough context to fix the problem without reading other code:

```rust
// ❌ Bad: Requires context to understand
Err(FsError::NotFound { path: path.to_path_buf() })

// ✅ Good: Self-explanatory
Err(FsError::NotFound { 
    path: path.to_path_buf(),
    operation: "read",
    context: "file does not exist or is a directory".into(),
})
```

**LLM Instruction:** "The error says 'quota exceeded: limit 100MB, requested 150MB, usage 80MB'. Fix the code that's writing 150MB."

### 5. Documentation at Every Boundary

Every public item has documentation explaining:
- **What** it does (one line)
- **When** to use it (use case)
- **How** to use it (example)
- **Why** it exists (rationale, if non-obvious)

```rust
/// Path resolution strategy using iterative component-by-component traversal.
///
/// # When to Use
/// - Default resolver for virtual backends (MemoryBackend, SqliteBackend)
/// - When you need standard POSIX-like symlink resolution
///
/// # Example
/// ```rust
/// let resolver = IterativeResolver::new();
/// let canonical = resolver.canonicalize(Path::new("/a/b/../c"), &fs)?;
/// ```
///
/// # Performance
/// O(n) where n = number of path components. For deep paths with many symlinks,
/// consider `CachingResolver` wrapper.
pub struct IterativeResolver { /* ... */ }
```

---

## File Structure Convention

Every implementation file follows this structure:

```rust
//! # Component Name
//!
//! Brief description of what this component does.
//!
//! ## Responsibility
//! - Single bullet point describing THE responsibility
//!
//! ## Dependencies
//! - List of traits/types this depends on
//!
//! ## Usage
//! ```rust
//! // Minimal working example
//! ```

use crate::{...}; // Minimal imports

// ============================================================================
// Types
// ============================================================================

/// Primary type for this component.
pub struct ComponentName { /* ... */ }

// ============================================================================
// Trait Implementations
// ============================================================================

impl SomeTrait for ComponentName {
    // Implementation
}

// ============================================================================
// Public API
// ============================================================================

impl ComponentName {
    /// Constructor with sensible defaults.
    pub fn new() -> Self { /* ... */ }
    
    /// Builder-style configuration.
    pub fn with_option(self, value: T) -> Self { /* ... */ }
}

// ============================================================================
// Private Helpers
// ============================================================================

impl ComponentName {
    fn internal_helper(&self) { /* ... */ }
}

// ============================================================================
// Tests
// ============================================================================

#[cfg(test)]
mod tests {
    use super::*;
    
    // Tests that verify the contract
}
```

---

## LLM Prompting Patterns

### Pattern 1: Implement a Component

```
Implement `CaseFoldingResolver` in `src/resolvers/case_folding.rs`.

Contract: Implement `PathResolver` trait (see src/path_resolver.rs).

Requirements:
- Wrap another resolver
- Normalize path components to lowercase during lookup
- Preserve original casing in returned paths

Test: Write a test showing "/Foo/BAR" resolves to "/foo/bar" on a case-insensitive filesystem.
```

### Pattern 2: Fix a Bug

```
Bug: `Quota<B>` doesn't account for existing file size when checking write limits.

File: src/middleware/quota.rs
Error: QuotaExceeded when writing 50 bytes to a 30-byte file with 100-byte limit.
Expected: Should succeed (30 + 50 = 80 < 100).

Fix the `check_write_quota` method.
```

### Pattern 3: Add a Feature

```
Add `max_path_depth` limit to `Quota<B>` middleware.

File: src/middleware/quota.rs
Contract: Reject operations that would create paths deeper than the limit.

Example:
```rust
let quota = Quota::new(backend).max_path_depth(5);
quota.create_dir_all("/a/b/c/d/e/f")?; // Err: depth 6 > limit 5
```
```

### Pattern 4: Review a Change

```
Review this change to IterativeResolver:

- Does it maintain the PathResolver contract?
- Are edge cases handled (empty path, root path, circular symlinks)?
- Are error messages informative?
- Are tests sufficient?

[diff]
```

---

## Component Isolation Checklist

Before considering a component complete, verify:

- [ ] **Single file** - Component lives in one file (or one module with mod.rs)
- [ ] **Clear contract** - Implements a trait with documented invariants
- [ ] **Minimal dependencies** - Only depends on traits/types, not other implementations
- [ ] **Self-contained tests** - Tests use mocks, not real backends
- [ ] **Informative errors** - Error messages explain what went wrong and how to fix
- [ ] **Usage example** - Doc comment shows how to use in isolation
- [ ] **No global state** - All state is in the struct instance
- [ ] **Thread-safe** - `Send + Sync` where required
- [ ] **Documented edge cases** - What happens with empty input, None, errors?

---

## Open Source Contribution Benefits

This methodology directly enables:

| Benefit                     | How                                                             |
| --------------------------- | --------------------------------------------------------------- |
| **First-time contributors** | Can understand one component without reading the whole codebase |
| **Focused PRs**             | Changes stay in one file, easy to review                        |
| **Parallel development**    | Multiple contributors work on different components              |
| **Quick onboarding**        | Read the trait, implement the trait, done                       |
| **CI efficiency**           | Test just the changed component                                 |

---

## Anti-Patterns to Avoid

### ❌ Spaghetti Dependencies

```rust
// Bad: Middleware knows about specific backends
impl<B: Fs> Quota<B> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        if let Some(sqlite) = self.inner.downcast_ref::<SqliteBackend>() {
            // Special case for SQLite
        }
    }
}
```

### ❌ Hidden Context Requirements

```rust
// Bad: Requires knowing about global configuration
impl FsRead for MyBackend {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let config = CONFIG.lock().unwrap(); // Hidden global!
        // ...
    }
}
```

### ❌ Tests That Require Setup

```rust
// Bad: Requires database, filesystem, network
#[test]
fn test_sqlite_backend() {
    let db = SqliteBackend::open("test.db").unwrap(); // Creates real file!
    // ...
}
```

### ❌ Vague Errors

```rust
// Bad: No context
Err(FsError::Backend("operation failed".into()))
```

---

## Integration with Context7-style Documentation

When the project is complete, we will provide a **consumer-facing LLM context document** that:

1. Explains the surface API (what to import, what to call)
2. Provides decision trees (which backend? which middleware?)
3. Shows complete, runnable examples
4. Lists common mistakes and how to avoid them

This is separate from AGENTS.md (for contributors) and lives in [Implementation Patterns](./llm-context.md).

---

## Summary

| Principle      | Implementation                       |
| -------------- | ------------------------------------ |
| **Isolated**   | One file, one concept                |
| **Contracted** | Traits define the spec               |
| **Testable**   | Mock-based unit tests                |
| **Debuggable** | Rich error context                   |
| **Documented** | Examples at every boundary           |
| **LLM-Ready**  | Promptable patterns for common tasks |

By following this methodology, AnyFS becomes a codebase where any component can be understood, tested, fixed, or replaced by an LLM (or human) with only local context. This is the foundation for sustainable AI-assisted development.
