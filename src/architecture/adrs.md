# AnyFS - Architecture Decision Records

This file captures the decisions for the current AnyFS design.

---

## ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Path-based `VfsBackend` trait | Accepted |
| ADR-002 | Three-crate structure | Accepted |
| ADR-003 | `impl AsRef<Path>` for all path parameters | Accepted |
| ADR-004 | Tower-style middleware pattern | Accepted |
| ADR-005 | `std::fs`-aligned method names | Accepted |
| ADR-006 | Quota for quota enforcement | Accepted |
| ADR-007 | FeatureGuard for least-privilege | Accepted |
| ADR-008 | FilesContainer as thin ergonomic wrapper | Accepted |
| ADR-009 | Built-in backends are feature-gated | Accepted |
| ADR-010 | Sync-first, async-ready design | Accepted |
| ADR-011 | Layer trait for standardized composition | Accepted |
| ADR-012 | Tracing for instrumentation | Accepted |
| ADR-013 | VfsBackendExt for extension methods | Accepted |
| ADR-014 | Optional Bytes support | Accepted |
| ADR-015 | Contextual VfsError | Accepted |
| ADR-016 | PathFilter for path-based access control | Accepted |
| ADR-017 | ReadOnly for preventing writes | Accepted |
| ADR-018 | RateLimit for operation throttling | Accepted |
| ADR-019 | DryRun for testing and debugging | Accepted |
| ADR-020 | Cache for read performance | Accepted |
| ADR-021 | Overlay for union filesystem | Accepted |

---

## ADR-001: Path-based `VfsBackend` trait

**Decision:** Backends implement a path-based trait aligned with `std::fs` method naming.

**Why:** Filesystem operations are naturally path-oriented; a single, familiar trait surface is easier to implement and adopt than graph-store or inode models.

---

## ADR-002: Three-crate structure

**Decision:**

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Minimal contract: `VfsBackend` trait, `Layer` trait, `VfsBackendExt`, types |
| `anyfs` | Backends + middleware (Quota, FeatureGuard, Tracing) |
| `anyfs-container` | Ergonomic wrapper: `FilesContainer<B>`, `BackendStack` builder |

**Why:**
- Backend authors only need `anyfs-backend` (no heavy dependencies).
- Middleware is composable and lives with backends in `anyfs`.
- `FilesContainer` is purely ergonomic - no policy logic.

---

## ADR-003: `impl AsRef<Path>` for all path parameters

**Decision:** Both `VfsBackend` and `FilesContainer` accept `impl AsRef<Path>` for all path parameters.

**Why:**
- Aligned with `std::fs` API conventions.
- Works across all platforms (not limited to UTF-8).
- Ergonomic: accepts `&str`, `String`, `&Path`, `PathBuf`.

---

## ADR-004: Tower-style middleware pattern

**Decision:** Use composable middleware (decorator pattern) for cross-cutting concerns like limits, logging, and feature gates. Each middleware implements `VfsBackend` by wrapping another `VfsBackend`.

**Why:**
- Complete separation of concerns - each layer has one job.
- Composable - use only what you need.
- Familiar pattern (Axum/Tower use the same approach).
- No code duplication - middleware written once, works with any backend.
- Testable - each layer can be tested in isolation.

**Example:**
```rust
let backend = Tracing::new(
    FeatureGuard::new(
        Quota::new(SqliteBackend::open("data.db")?)
    )
);
```

---

## ADR-005: `std::fs`-aligned method names

**Decision:** Prefer `read_dir`, `create_dir_all`, `remove_file`, etc.

**Why:** Familiarity and reduced cognitive overhead.

---

## ADR-006: Quota for quota enforcement

**Decision:** Quota/limit enforcement is handled by `Quota<B>` middleware, not by backends or FilesContainer.

**Configuration:**
- `with_max_total_size(bytes)` - total storage limit
- `with_max_file_size(bytes)` - per-file limit
- `with_max_node_count(count)` - max files/directories
- `with_max_dir_entries(count)` - max entries per directory
- `with_max_path_depth(depth)` - max directory nesting

**Why:**
- Limits are policy, not storage semantics.
- Written once, works with any backend.
- Optional - users who don't need limits skip this middleware.

**Implementation notes:**
- On construction, scan existing backend to initialize usage counters.
- Wrap `open_write` streams with `CountingWriter` to track streamed bytes.
- Check limits before operations, update usage after successful operations.

---

## ADR-007: FeatureGuard for least-privilege

**Decision:** Dangerous features (symlinks, hard links, permission mutation) are disabled by default via `FeatureGuard<B>` middleware.

**Configuration:**
- `.with_symlinks()` - enable symlink creation/following
- `.with_hard_links()` - enable hard link creation
- `.with_permissions()` - enable `set_permissions`
- `.with_max_symlink_resolution(n)` - limit symlink hops (default: 40)

When disabled, operations return `VfsError::FeatureNotEnabled`.

**Why:**
- Reduces attack surface by default.
- Explicit opt-in for dangerous features.
- Separate from backend - works with any backend.

---

## ADR-008: FilesContainer as thin ergonomic wrapper

**Decision:** `FilesContainer<B>` is a thin wrapper that provides std::fs-aligned ergonomics only. It contains NO policy logic.

**What it does:**
- Provides familiar method names
- Accepts `impl AsRef<Path>` for convenience
- Delegates all operations to the wrapped backend

**What it does NOT do:**
- Quota enforcement (use Quota)
- Feature gating (use FeatureGuard)
- Instrumentation (use Tracing)
- Any other policy

**Why:**
- Single responsibility - ergonomics only.
- Users who don't need ergonomics can use backends directly.
- Policy is composable via middleware, not hardcoded.

---

## ADR-009: Built-in backends are feature-gated

**Decision:** `anyfs` uses Cargo features so users only pull the dependencies they need.

- `memory` (default)
- `sqlite` (optional)
- `vrootfs` (optional)

**Why:** Minimizes binary size and compile time for users who don't need all backends.

---

## ADR-010: Sync-first, async-ready design

**Decision:** VfsBackend is synchronous for v1. The API is designed to allow adding `AsyncVfsBackend` later without breaking changes.

**Rationale:**
- All built-in backends are naturally synchronous:
  - `MemoryBackend` - in-memory, instant
  - `SqliteBackend` - rusqlite is sync
  - `VRootFsBackend` - std::fs is sync
- Sync is simpler - no runtime dependency (tokio/async-std)
- Users can wrap sync backends in `spawn_blocking` if needed

**Async-ready design principles:**
- Trait requires `Send` - compatible with async executors
- Return types are `Result<T, VfsError>` - works with async
- No internal blocking assumptions
- Methods are stateless per-call - no hidden blocking state

**Future async path (Option 2):**
When async is needed (e.g., network-backed storage), add a parallel trait:

```rust
// In anyfs-backend
pub trait AsyncVfsBackend: Send + Sync {
    async fn read(&self, path: impl AsRef<Path> + Send) -> Result<Vec<u8>, VfsError>;
    async fn write(&mut self, path: impl AsRef<Path> + Send, data: &[u8]) -> Result<(), VfsError>;
    // ... mirrors VfsBackend with async

    // Streaming uses AsyncRead/AsyncWrite
    async fn open_read(&self, path: impl AsRef<Path> + Send)
        -> Result<Box<dyn AsyncRead + Send + Unpin>, VfsError>;
}
```

**Migration notes:**
- `AsyncVfsBackend` would be a separate trait, not replacing `VfsBackend`
- Blanket impl possible: `impl<T: VfsBackend> AsyncVfsBackend for T` using `spawn_blocking`
- Middleware would need async variants: `AsyncQuota<B>`, etc.
- No breaking changes to existing sync API

**Why not async now:**
- Complexity without benefit - all current backends are sync
- Rust 1.75 makes async traits easy, so adding later is low-cost
- Better to wait for real async backend requirements

---

## ADR-011: Layer trait for standardized composition

**Decision:** Provide a `Layer` trait (inspired by Tower) that standardizes middleware composition.

```rust
pub trait Layer<B: VfsBackend> {
    type Backend: VfsBackend;
    fn layer(self, backend: B) -> Self::Backend;
}
```

**Why:**
- Standardized composition pattern familiar to Tower/Axum users.
- IDE autocomplete for available layers.
- Enables `BackendStack` fluent builder in anyfs-container.
- Each middleware provides a corresponding `*Layer` type.

**Example:**
```rust
let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::new().max_total_size(100_000))
    .layer(TracingLayer::new());
```

---

## ADR-012: Tracing for instrumentation

**Decision:** Use `Tracing<B>` integrated with the `tracing` ecosystem instead of a custom logging solution.

**Why:**
- Works with existing tracing infrastructure (tracing-subscriber, OpenTelemetry, Jaeger).
- Structured logging with spans for each operation.
- Users choose their subscriber - no logging framework lock-in.
- Consistent with modern Rust ecosystem practices.

**Configuration:**
```rust
Tracing::new(backend)
    .with_target("anyfs")
    .with_level(tracing::Level::DEBUG)
```

---

## ADR-013: VfsBackendExt for extension methods

**Decision:** Provide `VfsBackendExt` trait with convenience methods, auto-implemented for all backends.

```rust
pub trait VfsBackendExt: VfsBackend {
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, VfsError>;
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), VfsError>;
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
}

impl<B: VfsBackend> VfsBackendExt for B {}
```

**Why:**
- Adds convenience without bloating `VfsBackend` trait.
- Blanket impl means all backends get these methods for free.
- Users can define their own extension traits for domain-specific operations.
- Follows Rust convention (e.g., `IteratorExt`, `StreamExt`).

---

## ADR-014: Optional Bytes support

**Decision:** Support the `bytes` crate via an optional feature for zero-copy efficiency.

```toml
anyfs = { version = "0.1", features = ["bytes"] }
```

**Why:**
- `Bytes` provides O(1) slicing via reference counting.
- Beneficial for large file handling, network backends, streaming.
- Optional - users who don't need it avoid the dependency.
- Default remains `Vec<u8>` for simplicity.

**Implementation:** Use a type alias to avoid breaking API:

```rust
// In anyfs-backend/src/types.rs

#[cfg(feature = "bytes")]
pub type FileContent = bytes::Bytes;

#[cfg(not(feature = "bytes"))]
pub type FileContent = Vec<u8>;

// In trait definition
pub trait VfsBackend: Send {
    fn read(&self, path: impl AsRef<Path>) -> Result<FileContent, VfsError>;
    // ...
}
```

**Middleware compatibility:** Middleware passes `FileContent` through unchanged. No special handling needed - both `Vec<u8>` and `Bytes` implement `AsRef<[u8]>` and `Deref<Target=[u8]>`.

---

## ADR-015: Contextual VfsError

**Decision:** `VfsError` variants include context for better debugging.

```rust
VfsError::NotFound {
    path: PathBuf,
    operation: &'static str,  // "read", "metadata", etc.
}

VfsError::QuotaExceeded {
    limit: u64,
    requested: u64,
    usage: u64,
}
```

**Why:**
- Error messages include enough context to understand what failed.
- No need for separate error context crate (like anyhow) for basic usage.
- Operation field helps distinguish "file not found during read" vs "during metadata".
- Quota errors include all relevant numbers for debugging.

---

## ADR-016: PathFilter for path-based access control

**Decision:** Provide `PathFilter<B>` middleware for glob-based path access control.

**Configuration:**
```rust
PathFilter::new(backend)
    .allow("/workspace/**")    // Allow workspace access
    .deny("**/.env")           // Deny .env files anywhere
    .deny("**/secrets/**")     // Deny secrets directories
```

**Semantics:**
- Rules are evaluated in order; first match wins.
- If no rules match, access is denied (deny by default).
- Uses glob patterns (e.g., `**` for recursive, `*` for single segment).
- Returns `VfsError::AccessDenied` for denied paths.

**Why:**
- Essential for AI agent sandboxing - restrict to specific directories.
- Prevents access to sensitive files (.env, secrets, credentials).
- Separate from backend - works with any backend.
- Inspired by AgentFS and similar AI sandbox patterns.

**Implementation notes:**
- Use `globset` crate for efficient glob pattern matching.
- `read_dir` filters out denied entries from results (don't expose existence of denied files).
- Check path at operation start, then delegate to inner backend.

---

## ADR-017: ReadOnly for preventing writes

**Decision:** Provide `ReadOnly<B>` middleware that blocks all write operations.

**Usage:**
```rust
let readonly_fs = ReadOnly::new(backend);
```

**Semantics:**
- All read operations pass through to inner backend.
- All write operations return `VfsError::ReadOnly`.
- Simple, no configuration needed.

**Why:**
- Safe browsing of container contents without modification risk.
- Useful for debugging, inspection, auditing.
- Simpler than configuring FeatureGuard for read-only use case.

---

## ADR-018: RateLimit for operation throttling

**Decision:** Provide `RateLimit<B>` middleware to limit operations per time window.

**Configuration:**
```rust
RateLimit::new(backend)
    .max_ops(1000)
    .per_second()
```

**Semantics:**
- Tracks operation count in sliding time window.
- Returns `VfsError::RateLimitExceeded` when limit exceeded.
- Counter resets after window expires.

**Why:**
- Protects against runaway processes consuming resources.
- Essential for multi-tenant environments.
- Prevents denial-of-service from misbehaving code.

**Implementation notes:**
- Use `std::time::Instant` for timing.
- Store window start time and counter; reset when window expires.
- Count operation calls (including `open_read`/`open_write`), not bytes transferred.
- Return error immediately when limit exceeded (no blocking/waiting).

---

## ADR-019: DryRun for testing and debugging

**Decision:** Provide `DryRun<B>` middleware that logs write operations without executing them.

**Usage:**
```rust
let mut dry_run = DryRun::new(backend);
dry_run.write("/test.txt", b"hello")?;  // Logged but not written
let ops = dry_run.operations();         // ["write /test.txt (5 bytes)"]
```

**Semantics:**
- Read operations execute normally against inner backend.
- Write operations are logged but return `Ok(())` without executing.
- Operations log can be inspected for verification.

**Why:**
- Test code paths without side effects.
- Debug complex operation sequences.
- Audit what would happen before committing.

**Implementation notes:**
- Read operations delegate to inner backend (test against real state).
- Write operations log and return `Ok(())` without executing.
- `open_write` returns `std::io::sink()` - writes are discarded.
- Useful for: "What would this code do?" not "Run this in isolation."

---

## ADR-020: Cache for read performance

**Decision:** Provide `Cache<B>` middleware with LRU caching for read operations.

**Configuration:**
```rust
Cache::new(backend)
    .max_entries(1000)
    .max_entry_size(1024 * 1024)  // 1MB max per entry
    .ttl(Duration::from_secs(60))
```

**Semantics:**
- Read operations check cache first, populate on miss.
- Write operations invalidate relevant cache entries.
- LRU eviction when max entries exceeded.
- TTL-based expiration for freshness.

**Why:**
- Improves performance for repeated reads.
- Reduces load on underlying backend (especially for SQLite/network).
- Configurable to balance memory vs performance.

**Implementation notes:**
- Cache bulk reads only: `read()`, `read_to_string()`, `read_range()`, `metadata()`, `exists()`.
- Do NOT cache `open_read()` - streams are for large files that shouldn't be cached.
- Invalidate cache entry on any write to that path.
- Use `lru` crate or similar for LRU eviction.
- Check TTL on cache hits; evict expired entries.

---

## ADR-021: Overlay for union filesystem

**Decision:** Provide `Overlay<B1, B2>` middleware for copy-on-write layered filesystems.

**Usage:**
```rust
let base = SqliteBackend::open("base.db")?;  // Read-only base
let upper = MemoryBackend::new();             // Writable upper layer

let overlay = Overlay::new(base, upper);
```

**Semantics:**
- Read: check upper layer first, fall back to base if not found.
- Write: always to upper layer (copy-on-write).
- Delete: create whiteout marker in upper layer (file appears deleted but base unchanged).
- Directory listing: merge results from both layers.

**Why:**
- Docker-like layered filesystem for containers.
- Base image with per-instance modifications.
- Testing with isolated changes over shared baseline.
- Inspired by OverlayFS and VFS crate patterns.

**Implementation notes:**
- Whiteout convention: `.wh.<filename>` marks deleted files from base layer.
- `read_dir` must merge results from both layers, excluding whiteouts and whited-out files.
- `exists` checks upper first, then base (respecting whiteouts).
- All writes go to upper layer; base is never modified.
- Consider `opaque` directories (`.wh..wh..opq`) to hide entire base directories.
