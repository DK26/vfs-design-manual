# Lessons from Similar Projects

**Analysis of issues from `vfs` and `agentfs` to inform AnyFS design.**

This chapter documents problems encountered by similar projects and how AnyFS addresses them. These lessons are incorporated into our [Implementation Plan](./plan.md) and [Backend Guide](./backend-guide.md).

---

## Summary

| Priority | Issue | AnyFS Response |
|----------|-------|----------------|
| 1 | Panics instead of errors | No-panic policy, always return `Result` |
| 2 | Thread safety problems | Concurrent stress tests required |
| 3 | Inconsistent path handling | Normalize in one place, test edge cases |
| 4 | Poor error ergonomics | `FsError` with context fields |
| 5 | Missing documentation | Performance & thread safety docs required |
| 6 | Platform issues | Cross-platform CI pipeline |

---

## 1. Thread Safety Issues

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| vfs | [#72](https://github.com/manuel-woelker/rust-vfs/issues/72) | RwLock panic in production |
| vfs | [#47](https://github.com/manuel-woelker/rust-vfs/issues/47) | `create_dir_all` races with itself |

**Root cause:** Insufficient synchronization in concurrent access patterns.

### AnyFS Response

- **Test concurrent operations explicitly** - stress test with multiple threads
- **Document thread safety guarantees** per backend
- **`Fs: Send`** bound is intentional
- **`MemoryBackend` uses `Arc<RwLock<...>>`** for interior mutability

**Required tests:**
```rust
#[test]
fn test_concurrent_create_dir_all() {
    let backend = Arc::new(RwLock::new(create_backend()));
    let handles: Vec<_> = (0..10).map(|_| {
        let backend = backend.clone();
        std::thread::spawn(move || {
            let mut backend = backend.write().unwrap();
            let _ = backend.create_dir_all("/a/b/c/d");
        })
    }).collect();
    for handle in handles {
        handle.join().unwrap();
    }
}
```

---

## 2. Panics Instead of Errors

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| vfs | [#8](https://github.com/manuel-woelker/rust-vfs/issues/8) | AltrootFS panics when file doesn't exist |
| vfs | [#23](https://github.com/manuel-woelker/rust-vfs/issues/23) | Unhandled edge cases cause panics |
| vfs | [#68](https://github.com/manuel-woelker/rust-vfs/issues/68) | MemoryFS panics in WebAssembly |

**Root cause:** Using `.unwrap()` or `.expect()` on fallible operations.

### AnyFS Response

**No-panic policy:** Never use `.unwrap()` or `.expect()` in library code.

```rust
// BAD - will panic
let entry = self.entries.get(&path).unwrap();

// GOOD - returns error
let entry = self.entries.get(&path)
    .ok_or_else(|| FsError::NotFound { path: path.to_path_buf() })?;
```

**Edge cases that must return errors (not panic):**
- File doesn't exist
- Directory doesn't exist
- Path is empty string
- Invalid UTF-8 in path
- Parent directory missing
- Type mismatch (file vs directory)
- Concurrent access conflicts

---

## 3. Path Handling Inconsistencies

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| vfs | [#24](https://github.com/manuel-woelker/rust-vfs/issues/24) | Inconsistent path definition across backends |
| vfs | [#42](https://github.com/manuel-woelker/rust-vfs/issues/42) | Path join doesn't behave Unix-like |
| vfs | [#22](https://github.com/manuel-woelker/rust-vfs/issues/22) | Non-UTF-8 path support questions |

**Root cause:** Each backend implemented path handling differently.

### AnyFS Response

- **Normalize paths in ONE place** (FileStorage or dedicated normalizer)
- **Consistent semantics:** always absolute, always `/` separator
- **Use `impl AsRef<Path>`** but normalize internally

**Required conformance tests:**

| Input | Expected Output |
|-------|-----------------|
| `/foo/../bar` | `/bar` |
| `/foo/./bar` | `/foo/bar` |
| `//double//slash` | `/double/slash` |
| `/` | `/` |
| `` (empty) | Error |
| `/foo/bar/` | `/foo/bar` |

---

## 4. Static Lifetime Requirements

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| vfs | [#66](https://github.com/manuel-woelker/rust-vfs/issues/66) | Why does filesystem require `'static`? |

**Root cause:** Design decision that confused users and limited flexibility.

### AnyFS Response

- **Avoid `'static` bounds** unless necessary
- **Our design:** `Fs: Send` (not `'static`)
- **Document why bounds exist** when needed

---

## 5. Missing Symlink Support

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| vfs | [#81](https://github.com/manuel-woelker/rust-vfs/issues/81) | Symlink support missing entirely |

**Root cause:** Symlinks are complex and were deferred indefinitely.

### AnyFS Response

- **Symlinks are opt-in** via `Restrictions` middleware
- **Default deny is correct** - most use cases don't need symlinks
- **When enabled, bound resolution depth** (default: 40 hops)
- **`strict-path` prevents symlink escapes** in `VRootFsBackend`

---

## 6. Error Type Ergonomics

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| vfs | [#33](https://github.com/manuel-woelker/rust-vfs/issues/33) | Error type hard to match programmatically |

**Root cause:** Error enum wasn't designed for pattern matching.

### AnyFS Response

`FsError` includes context and is easy to match:

```rust
#[derive(Debug, thiserror::Error)]
pub enum FsError {
    #[error("{operation}: not found: {path}")]
    NotFound { path: PathBuf, operation: &'static str },

    #[error("{operation}: already exists: {path}")]
    AlreadyExists { path: PathBuf, operation: &'static str },

    #[error("quota exceeded: limit {limit}, attempted {attempted}")]
    QuotaExceeded { limit: u64, attempted: u64 },

    #[error("feature not enabled: {feature}")]
    FeatureNotEnabled { feature: &'static str },

    #[error("permission denied: {path} ({operation})")]
    PermissionDenied { path: PathBuf, operation: &'static str },

    // ...
}
```

---

## 7. Seek + Write Operations

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| vfs | [#35](https://github.com/manuel-woelker/rust-vfs/issues/35) | Missing file positioning features |

**Root cause:** Initial API was too simple.

### AnyFS Response

- **Streaming I/O:** `open_read`/`open_write` return `Box<dyn Read/Write + Send>`
- **Seek support varies by backend** - document which support it
- **Consider future:** `open_read_seek` variant or capability query

---

## 8. Read-Only Filesystem Request

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| vfs | [#58](https://github.com/manuel-woelker/rust-vfs/issues/58) | Request for immutable filesystem |

**Root cause:** No built-in way to enforce read-only access.

### AnyFS Response

**Already solved:** `ReadOnly<B>` middleware blocks all writes.

```rust
let readonly_fs = FileStorage::new(
    ReadOnly::new(SqliteBackend::open("archive.db")?)
);
// All write operations return FsError::ReadOnly
```

This validates our middleware approach.

---

## 9. Performance Issues

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| agentfs | [#130](https://github.com/tursodatabase/agentfs/issues/130) | File deletion is slow |
| agentfs | [#135](https://github.com/tursodatabase/agentfs/issues/135) | Benchmark hangs |

**Root cause:** SQLite operations not optimized, FUSE overhead.

### AnyFS Response

- **Batch operations** where possible in `SqliteBackend`
- **Use transactions** for multi-file operations
- **Document performance characteristics** per backend
- **Avoid FUSE** - we're a library, not a mount

**Documentation requirement:**
```rust
/// # Performance Characteristics
///
/// | Operation | Complexity | Notes |
/// |-----------|------------|-------|
/// | `read` | O(1) | Single DB query |
/// | `write` | O(n) | n = data size |
/// | `remove_dir_all` | O(n) | n = descendants |
pub struct SqliteBackend { ... }
```

---

## 10. Signal Handling / Shutdown

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| agentfs | [#129](https://github.com/tursodatabase/agentfs/issues/129) | Doesn't shutdown on SIGTERM |

**Root cause:** FUSE mount cleanup issues.

### AnyFS Response

- **Not our problem** - we're a library, not a daemon
- **Ensure `Drop` implementations clean up properly**
- **`SqliteBackend` flushes on drop**

```rust
impl Drop for SqliteBackend {
    fn drop(&mut self) {
        if let Err(e) = self.sync() {
            eprintln!("Warning: failed to sync on drop: {}", e);
        }
    }
}
```

---

## 11. Platform Compatibility

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| agentfs | [#132](https://github.com/tursodatabase/agentfs/issues/132) | FUSE-T support (macOS) |
| agentfs | [#138](https://github.com/tursodatabase/agentfs/issues/138) | virtio-fs support |

**Root cause:** Platform-specific FUSE variants.

### AnyFS Response

- **We avoid this** - no FUSE, pure library
- **Cross-platform by design** - Memory and SQLite work everywhere
- **`VRootFsBackend` uses `strict-path`** which handles Windows/Unix

**CI requirement:**
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
```

---

## 12. Multiple Sessions / Concurrent Access

### What Happened

| Project | Issue | Problem |
|---------|-------|---------|
| agentfs | [#126](https://github.com/tursodatabase/agentfs/issues/126) | Can't have multiple sessions on same filesystem |

**Root cause:** Locking/concurrency design.

### AnyFS Response

- **`SqliteBackend` uses WAL mode** for concurrent readers
- **Document concurrency model** per backend
- **`MemoryBackend` uses `Arc<RwLock<...>>`** for sharing

---

## Issues We Already Avoid

Our design decisions already prevent these problems:

| Problem in Others | AnyFS Solution |
|-------------------|----------------|
| No middleware pattern | Tower-style composable middleware |
| No quota enforcement | `Quota<B>` middleware |
| No read-only mode | `ReadOnly<B>` middleware |
| Symlink complexity | `Restrictions<B>` (opt-in) |
| Path escape via symlinks | `strict-path` canonicalization |
| FUSE complexity | Pure library, no mount |
| SQLite-only | Multiple backends |
| Monolithic features | Composable middleware |

---

## References

- [rust-vfs Issues](https://github.com/manuel-woelker/rust-vfs/issues)
- [agentfs Issues](https://github.com/tursodatabase/agentfs/issues)
- [Implementation Plan](./plan.md) - incorporates these lessons
- [Backend Guide](./backend-guide.md) - implementation requirements
