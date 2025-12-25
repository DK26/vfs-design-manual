# Implementation Plan

This plan describes a phased rollout of the AnyFS ecosystem:

- `anyfs-backend`: Core trait (`VfsBackend`, `Layer`) + types
- `anyfs`: Built-in backends + middleware (feature-gated)
- `anyfs-container`: `FilesContainer<B>` ergonomic wrapper

---

## Implementation Guidelines

These guidelines apply to ALL implementation work. Derived from analysis of issues in similar projects (`vfs`, `agentfs`).

### 1. No Panic Policy

**NEVER panic in library code.** Always return `Result<T, VfsError>`.

- Audit all `.unwrap()` and `.expect()` calls - replace with `?` or proper error handling
- Use `ok_or_else(|| VfsError::...)` instead of `.unwrap()`
- Edge cases must return errors, not panic
- Test in constrained environments (WASM) to catch hidden panics

```rust
// BAD
let entry = self.entries.get(&path).unwrap();

// GOOD
let entry = self.entries.get(&path)
    .ok_or_else(|| VfsError::NotFound { path: path.to_path_buf() })?;
```

### 2. Thread Safety Requirements

All backends must be safe for concurrent access:

- `MemoryBackend`: Use `Arc<RwLock<...>>` for internal state
- `SqliteBackend`: Use WAL mode, handle `SQLITE_BUSY`
- `VRootFsBackend`: File operations are inherently concurrent-safe

**Required:** Concurrent stress tests in conformance suite.

### 3. Consistent Path Handling

Normalize paths in ONE place (FilesContainer or dedicated normalizer):

- Always absolute paths internally
- Always `/` separator (even on Windows)
- Normalize `..` and `.` components
- Handle edge cases: `//`, trailing `/`, empty string

### 4. Error Type Design

`VfsError` must be:
- Easy to pattern match
- Include context (path, operation)
- Derive `thiserror` for good messages

```rust
#[derive(Debug, thiserror::Error)]
pub enum VfsError {
    #[error("not found: {path}")]
    NotFound { path: PathBuf },

    #[error("already exists: {path}")]
    AlreadyExists { path: PathBuf },

    #[error("quota exceeded: limit {limit}, attempted {attempted}")]
    QuotaExceeded { limit: u64, attempted: u64 },

    #[error("feature not enabled: {feature}")]
    FeatureNotEnabled { feature: &'static str },

    #[error("permission denied: {path} ({operation})")]
    PermissionDenied { path: PathBuf, operation: &'static str },

    // ... etc
}
```

### 5. Documentation Requirements

Every backend and middleware must document:
- Thread safety guarantees
- Performance characteristics
- Which operations are O(1) vs O(n)
- Any platform-specific behavior

---

## Phase 1: `anyfs-backend` (core contract)

**Goal:** Define the stable backend interface and composition traits.

- Define `VfsBackend` trait (25 `std::fs`-aligned methods)
  - Include `const NEEDS_PATH_RESOLUTION: bool = true` (opt-out for real FS backends)
- Define `Layer` trait (Tower-style middleware composition)
- Define `VfsBackendExt` trait (extension methods)
- Define core types (`Metadata`, `Permissions`, `FileType`, `DirEntry`, `StatFs`)
- Define `VfsError` with contextual variants (see guidelines above)

**Exit criteria:** `anyfs-backend` stands alone with minimal dependencies (`thiserror`).

---

## Phase 2: `anyfs` (backends + middleware)

**Goal:** Provide reference backends and core middleware.

### Path Resolution Utility

- `resolve_path(backend, path, follow_symlinks)` - works on any `VfsBackend`
  - Walks path component by component using `metadata()` and `read_link()`
  - Handles `..` correctly (requires knowing what each component resolves to)
  - Detects circular symlinks (max depth or visited set)
  - Returns canonical resolved path
- Applied automatically by `FilesContainer` when `B::NEEDS_PATH_RESOLUTION == true`

### Backends (feature-gated)

- `memory` (default): `MemoryBackend`
  - `NEEDS_PATH_RESOLUTION = true` (uses path resolution utility)
  - `set_follow_symlinks(bool)` - control symlink resolution
- `sqlite` (optional): `SqliteBackend`
  - `NEEDS_PATH_RESOLUTION = true` (uses path resolution utility)
  - `set_follow_symlinks(bool)` - control symlink resolution
- `vrootfs` (optional): `VRootFsBackend` using `strict-path` for containment
  - `NEEDS_PATH_RESOLUTION = false` (OS handles resolution)
  - `strict-path` prevents symlink escapes

### Middleware

- `Quota<B>` + `QuotaLayer` - Resource limits
- `Restrictions<B>` + `RestrictionsLayer` - Opt-in restrictions (`.deny_symlinks()`, `.deny_hard_links()`, etc.)
- `PathFilter<B>` + `PathFilterLayer` - Path-based access control
- `ReadOnly<B>` + `ReadOnlyLayer` - Block writes
- `RateLimit<B>` + `RateLimitLayer` - Operation throttling
- `Tracing<B>` + `TracingLayer` - Instrumentation
- `DryRun<B>` + `DryRunLayer` - Log without executing
- `Cache<B>` + `CacheLayer` - LRU read cache
- `Overlay<B1,B2>` + `OverlayLayer` - Union filesystem

**Exit criteria:** Each backend implements `VfsBackend` and passes conformance suite. Each middleware wraps any `VfsBackend`.

---

## Phase 3: `anyfs-container` (ergonomics)

**Goal:** Provide user-facing ergonomic wrapper.

- `FilesContainer<B>` - Thin wrapper with `std::fs`-aligned API
- Accepts `impl AsRef<Path>` for convenience
- Delegates all operations to wrapped backend

**Note:** `FilesContainer` contains NO policy logic. Policy is handled by middleware.

**Exit criteria:** Applications can use `FilesContainer` as drop-in for `std::fs` patterns.

---

## Phase 4: Conformance test suite

**Goal:** Prevent backend divergence and validate middleware behavior.

### Backend conformance tests

#### Basic Operations
- `read`/`write`/`append` semantics
- Directory operations (`create_dir*`, `read_dir`, `remove_dir*`)
- `rename` and `copy` semantics
- Link behavior (`symlink`, `read_link`, `hard_link`)
- Streaming I/O (`open_read`, `open_write`)
- `truncate`, `sync`, `fsync`, `statfs`

#### Path Resolution Tests (virtual backends only)
- `/foo/../bar` resolves correctly when `foo` is a regular directory
- `/foo/../bar` resolves correctly when `foo` is a symlink (follows symlink, then `..`)
- Symlink chains resolve correctly (A → B → C → target)
- Circular symlink detection (A → B → A returns error, not infinite loop)
- Max symlink depth enforced (prevent deep chains)
- `set_follow_symlinks(true)`: reading symlink follows to target
- `set_follow_symlinks(false)`: reading symlink returns symlink metadata, not target

#### Path Edge Cases (learned from `vfs` issues)
- `//double//slashes//` normalizes correctly
- Note: `/foo/../bar` requires resolution (see above), not simple normalization
- Trailing slashes handled consistently
- Empty path returns error (not panic)
- Root path `/` works correctly
- Very long paths (near OS limits)
- Unicode paths
- Paths with spaces and special characters

#### Thread Safety Tests (learned from `vfs` #72, #47)
- Concurrent `read` from multiple threads
- Concurrent `write` to different files
- Concurrent `create_dir_all` to same path (must not race)
- Concurrent `read_dir` while modifying directory
- Stress test: 100 threads, 1000 operations each

#### Error Handling Tests (learned from `vfs` #8, #23)
- Missing file returns `NotFound`, not panic
- Missing parent directory returns error, not panic
- Invalid UTF-8 in path returns error, not panic
- All error variants are matchable

#### Platform Tests
- Windows path separators (`\` vs `/`)
- Case sensitivity differences
- Symlink behavior differences

### Middleware tests

- `Quota`: Limit enforcement, usage tracking, streaming writes
- `Restrictions`: Operation blocking via `.deny_*()` methods, error messages
- `PathFilter`: Glob pattern matching, deny-by-default
- `RateLimit`: Throttling behavior, burst handling
- `ReadOnly`: All write operations blocked
- `Tracing`: Operations logged correctly
- Middleware composition order (inner to outer)
- Middleware with streaming I/O (wrappers work correctly)

### No-Panic Tests

```rust
#[test]
fn no_panic_on_missing_file() {
    let backend = create_backend();
    let result = backend.read("/nonexistent");
    assert!(matches!(result, Err(VfsError::NotFound { .. })));
}

#[test]
fn no_panic_on_invalid_operation() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"data").unwrap();
    // Try to read directory on a file
    let result = backend.read_dir("/file.txt");
    assert!(matches!(result, Err(VfsError::NotADirectory { .. })));
}
```

### WASM Compatibility Tests (learned from `vfs` #68)

```rust
#[cfg(target_arch = "wasm32")]
#[wasm_bindgen_test]
fn memory_backend_works_in_wasm() {
    let mut backend = MemoryBackend::new();
    backend.write("/test.txt", b"hello").unwrap();
    // Should not panic
}
```

**Exit criteria:** All backends pass same suite; middleware tests are backend-agnostic; zero panics in any test.

---

## Phase 5: Documentation + examples

- Keep `AGENTS.md` and `src/architecture/design-overview.md` authoritative
- Provide example per backend
- Provide backend implementer guide
- Provide middleware implementer guide
- Document performance characteristics per backend
- Document thread safety guarantees per backend
- Document platform-specific behavior

---

## Phase 6: CI/CD Pipeline

**Goal:** Ensure quality across platforms and prevent regressions.

### Cross-Platform Testing

```yaml
# .github/workflows/ci.yml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    rust: [stable, beta]
```

Required CI checks:
- `cargo test` on all platforms
- `cargo clippy -- -D warnings`
- `cargo fmt --check`
- `cargo doc --no-deps`
- WASM build test: `cargo build --target wasm32-unknown-unknown`

### Additional CI Jobs

- **Miri** (undefined behavior detection): `cargo +nightly miri test`
- **Address Sanitizer**: Detect memory issues
- **Thread Sanitizer**: Detect data races
- **Coverage**: Minimum 80% line coverage

### Release Checklist

- [ ] All CI checks pass
- [ ] No new `clippy` warnings
- [ ] CHANGELOG updated
- [ ] Version bumped appropriately
- [ ] Documentation builds without warnings

---

## Future work (post-MVP)

- Async API (`AsyncVfsBackend` trait)
- Import/export helpers (host path <-> container)
- Extended attributes
- Encryption middleware
- Compression middleware
- `no_std` support (learned from `vfs` #38)
- Batch operations for performance (learned from `agentfs` #130)

### `anyfs-vfs-compat` - Interop with `vfs` crate

Adapter crate for bidirectional compatibility with the [`vfs`](https://github.com/manuel-woelker/rust-vfs) crate ecosystem.

**Why not adopt their trait?** The `vfs::FileSystem` trait is too limited:
- No symlinks, hard links, or permissions
- No `sync`/`fsync` for durability
- No `truncate`, `statfs`, or `read_range`
- No middleware composition pattern

**Our trait is a superset** - we support everything they do, plus more.

**Adapters:**

```rust
// Wrap a vfs::FileSystem to use as AnyFS backend
// Missing features (symlinks, permissions, etc.) return VfsError::NotSupported
pub struct VfsCompat<F: vfs::FileSystem>(F);
impl<F: vfs::FileSystem> VfsBackend for VfsCompat<F> { ... }

// Wrap an AnyFS backend to use as vfs::FileSystem
// Only exposes the subset that vfs supports
pub struct AnyFsCompat<B: VfsBackend>(B);
impl<B: VfsBackend> vfs::FileSystem for AnyFsCompat<B> { ... }
```

**Use cases:**
- Migrate from `vfs` to AnyFS incrementally
- Use existing `vfs` backends (EmbeddedFS) in AnyFS
- Use AnyFS backends in projects that depend on `vfs`

---

## Lessons Learned (Reference)

This plan incorporates lessons from issues in similar projects:

| Source | Issue | Lesson Applied |
|--------|-------|----------------|
| vfs #72 | RwLock panic | Thread safety tests |
| vfs #47 | `create_dir_all` race | Concurrent stress tests |
| vfs #8, #23 | Panics instead of errors | No-panic policy |
| vfs #24, #42 | Path inconsistencies | Path edge case tests |
| vfs #33 | Hard to match errors | Ergonomic `VfsError` design |
| vfs #68 | WASM panics | WASM compatibility tests |
| vfs #66 | `'static` confusion | Minimal trait bounds |
| agentfs #130 | Slow file deletion | Performance documentation |
| agentfs #129 | Signal handling | Proper `Drop` implementations |

See [Lessons from Similar Projects](./lessons-learned.md) for full analysis.
