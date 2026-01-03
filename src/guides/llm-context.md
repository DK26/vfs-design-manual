# Consumer Documentation Planning

> **This document specifies what the Context7-style consumer documentation should contain when the AnyFS library is implemented.**
> This is a planning/specification document, not actual API documentation.

---

## Purpose

When AnyFS is implemented, we need a **Context7-style reference document** that LLMs can use to correctly consume the AnyFS API. This document specifies what that reference should contain.

**Why Context7-style?**

- LLMs need quick decision trees to select the right components
- Copy-paste-ready patterns reduce hallucination
- Common mistakes section prevents known pitfalls
- Trait hierarchy helps understand what to implement

---

## Required Sections

The consumer documentation MUST include these sections:

### 1. Quick Decision Trees

Decision trees help LLMs quickly navigate to the right component. Include:

| Decision Tree      | Purpose                           |
| ------------------ | --------------------------------- |
| Which Crate?       | `anyfs-backend` vs `anyfs`        |
| Which Backend?     | Memory, SQLite, VRootFs, etc.     |
| Which Middleware?  | Quota, PathFilter, ReadOnly, etc. |
| Which Trait Level? | Fs, FsFull, FsFuse, FsPosix       |

**Format:** ASCII tree diagrams with terminal answers.

**Example structure (to be filled with actual API when implemented):**

```
Is data persistence required?
├─ NO → MemoryBackend
└─ YES → Is encryption needed?
         ├─ YES → SqliteCipherBackend
         └─ NO → [continue decision tree...]
```

### 2. Common Patterns

Provide copy-paste-ready code for these scenarios:

| Pattern                | Description                          |
| ---------------------- | ------------------------------------ |
| Simple File Operations | read, write, delete, check existence |
| Directory Operations   | create, list, remove                 |
| Sandboxed AI Agent     | Full middleware stack example        |
| Persistent Database    | SqliteBackend setup                  |
| Type-Safe Containers   | Marker types for compile-time safety |
| Streaming Large Files  | open_read/open_write usage           |

**Requirements for each pattern:**

- Complete, runnable code blocks
- All imports included
- Proper error handling (no `.unwrap()`)
- Minimal code that demonstrates the concept

### 3. Trait Hierarchy Diagram

Visual representation of the trait hierarchy:

```
FsPosix  ← Full POSIX (handles, locks, xattr)
    ↑
FsFuse   ← FUSE-mountable (+ inodes)
    ↑
FsFull   ← std::fs features (+ links, permissions, sync, stats)
    ↑
   Fs    ← Basic filesystem (90% of use cases)
    ↑
FsRead + FsWrite + FsDir  ← Core traits
```

With clear guidance: "Implement the lowest level you need. Higher levels include all below."

### 4. Backend Implementation Pattern

Template for implementing custom backends. The consumer docs should include:

| Level    | Traits to Implement                                | Result    |
| -------- | -------------------------------------------------- | --------- |
| Minimum  | `FsRead + FsWrite + FsDir`                         | `Fs`      |
| Extended | Add `FsLink`, `FsPermissions`, `FsSync`, `FsStats` | `FsFull`  |
| FUSE     | Add `FsInode`                                      | `FsFuse`  |
| POSIX    | Add `FsHandles`, `FsLock`, `FsXattr`               | `FsPosix` |

Each level should have a complete template showing all required method signatures.

### 5. Middleware Implementation Pattern

Template showing:

- How to wrap an inner backend with a generic type parameter
- Which methods to intercept vs delegate
- The `Layer` trait for `.layer()` syntax
- Common middleware patterns table:

| Pattern        | Intercept                  | Delegate        | Example      |
| -------------- | -------------------------- | --------------- | ------------ |
| Logging        | All (before/after)         | All             | `Tracing`    |
| Block writes   | Write methods → error      | Read methods    | `ReadOnly`   |
| Transform data | `read`/`write`             | Everything else | `Encryption` |
| Check access   | All (before)               | All             | `PathFilter` |
| Enforce limits | Write methods (check size) | Read methods    | `Quota`      |

### 6. Adapter Patterns

Templates for interoperability:

| Adapter Type  | Description                                          |
| ------------- | ---------------------------------------------------- |
| FROM external | Wrap external crate's filesystem as AnyFS backend    |
| TO external   | Wrap AnyFS backend to satisfy external crate's trait |

### 7. Error Handling Reference

All `FsError` variants with when to use each:

| Variant             | When to Return                         |
| ------------------- | -------------------------------------- |
| `NotFound`          | Path doesn't exist                     |
| `AlreadyExists`     | Path already exists (create conflict)  |
| `NotAFile`          | Expected file, got directory           |
| `NotADirectory`     | Expected directory, got file           |
| `DirectoryNotEmpty` | Can't remove non-empty directory       |
| `ReadOnly`          | Write blocked by ReadOnly middleware   |
| `AccessDenied`      | Blocked by PathFilter or permissions   |
| `QuotaExceeded`     | Size/count limit exceeded              |
| `NotSupported`      | Backend doesn't support this operation |
| `Backend`           | Backend-specific error                 |

### 8. Common Mistakes & Fixes

| Mistake                   | Fix                                |
| ------------------------- | ---------------------------------- |
| Using `unwrap()`          | Always use `?` or handle `FsError` |
| Assuming paths normalized | Use `canonicalize()` first         |
| Forgetting parent dirs    | Use `create_dir_all`               |
| Holding handles too long  | Drop promptly                      |
| Mixing backend types      | Use `FileStorage::boxed()`         |
| Testing with real files   | Use `MemoryBackend`                |

---

## Document Structure

When creating the actual consumer documentation, follow this structure:

```markdown
# AnyFS Implementation Patterns

## Quick Decision Trees
### Which Crate Do I Need?
### Which Backend Should I Use?
### Do I Need Middleware?
### Which Trait Level?

## Common Patterns
### Simple File Operations
### Directory Operations
### Sandboxed AI Agent
### Persistent Database
### Type-Safe Container Markers

## Trait Hierarchy (Pick Your Level)

## Pattern 1: Implement a Backend
### Minimum: Implement Fs
### Add Links/Permissions: Implement FsFull
### Add FUSE Support: Implement FsFuse

## Pattern 2: Implement Middleware
### Template
### Common Middleware Patterns

## Pattern 3: Implement an Adapter
### Adapter FROM another crate
### Adapter TO another crate

## Error Handling Reference

## Common Mistakes & Fixes

## Quick Reference: What to Implement
```

---

## Creation Guidelines

When creating the actual consumer documentation after implementation:

1. **Use actual tested code** - Every example must compile and run
2. **Include all imports** - LLMs need complete context
3. **Show error handling** - Never use `.unwrap()` in examples
4. **Keep examples minimal** - Shortest code that demonstrates the pattern
5. **Update with API changes** - This doc must stay in sync with implementation
6. **Validate against real usage** - Test each pattern before including it

---

## Quality Checklist

Before publishing the consumer documentation:

- [ ] All code examples compile
- [ ] All code examples run without panics
- [ ] Decision trees lead to correct answers
- [ ] Error variants match actual `FsError` enum
- [ ] Trait hierarchy matches actual trait definitions
- [ ] Common mistakes reflect actual issues found in testing

---

## Related Documents

| Document                                                      | Purpose                                                     |
| ------------------------------------------------------------- | ----------------------------------------------------------- |
| [LLM Development Methodology](llm-development-methodology.md) | For implementers: how to structure code for LLM development |
| This document                                                 | Specification for consumer documentation                    |
| [Backend Guide](../implementation/backend-guide.md)           | Design for backend implementation                           |
| [Middleware Tutorial](middleware-tutorial.md)                 | Design for middleware creation                              |

---

## Tracking

This planning document should be replaced with actual consumer documentation when:

1. **AnyFS is implemented** - The crates exist and compile
2. **API is stable** - No major breaking changes expected
3. **Examples are tested** - All patterns verified working

**GitHub Issue:** Create Context7-style consumer documentation
- **Status:** Blocked by AnyFS implementation
- **Template:** This planning document

