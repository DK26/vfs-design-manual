# AnyFS - Architecture Decision Records

This file captures the decisions for the current AnyFS design.

---

## Decision Map

Primary docs are where each decision is explained in narrative form. ADRs remain the source of truth for the decision itself.

| ADR     | Primary doc                                                                 |
| ------- | --------------------------------------------------------------------------- |
| ADR-001 | [Design Overview](./design-overview.md)                                     |
| ADR-002 | [Project Structure](../overview/project-structure.md)                       |
| ADR-003 | [Layered Traits](../traits/layered-traits.md)                               |
| ADR-004 | [Design Overview](./design-overview.md)                                     |
| ADR-005 | [API Quick Reference](../getting-started/api-reference.md)                  |
| ADR-006 | [Middleware Implementation](../implementation/middleware-implementation.md) |
| ADR-007 | [Middleware Implementation](../implementation/middleware-implementation.md) |
| ADR-008 | [FileStorage](../traits/files-container.md)                                 |
| ADR-009 | [Project Structure](../overview/project-structure.md)                       |
| ADR-010 | [Implementation Plan](../implementation/plan.md)                            |
| ADR-011 | [Design Overview](./design-overview.md)                                     |
| ADR-012 | [Middleware Implementation](../implementation/middleware-implementation.md) |
| ADR-013 | [Layered Traits](../traits/layered-traits.md)                               |
| ADR-014 | [Design Overview](./design-overview.md)                                     |
| ADR-015 | [Design Overview](./design-overview.md)                                     |
| ADR-016 | [Security Considerations](../comparisons/security.md)                       |
| ADR-017 | [Middleware Implementation](../implementation/middleware-implementation.md) |
| ADR-018 | [Middleware Implementation](../implementation/middleware-implementation.md) |
| ADR-019 | [Middleware Implementation](../implementation/middleware-implementation.md) |
| ADR-020 | [Middleware Implementation](../implementation/middleware-implementation.md) |
| ADR-021 | [Middleware Implementation](../implementation/middleware-implementation.md) |
| ADR-022 | [API Quick Reference](../getting-started/api-reference.md)                  |
| ADR-023 | [Layered Traits](../traits/layered-traits.md)                               |
| ADR-024 | [Implementation Plan](../implementation/plan.md)                            |
| ADR-025 | [Zero-Cost Alternatives](./zero-cost-alternatives.md)                       |
| ADR-026 | [Implementation Plan](../implementation/plan.md)                            |
| ADR-027 | [Design Overview](./design-overview.md)                                     |
| ADR-028 | [Design Overview](./design-overview.md)                                     |
| ADR-029 | [Two-Layer Path Handling](./two-layer-design.md)                            |
| ADR-030 | [Layered Traits](../traits/layered-traits.md)                               |
| ADR-031 | [Indexing Middleware](./indexed-realfs-backend.md)                          |

---

## ADR Index

| ADR     | Title                                       | Status            |
| ------- | ------------------------------------------- | ----------------- |
| ADR-001 | Path-based `Fs` trait                       | Accepted          |
| ADR-002 | Two-crate structure                         | Accepted          |
| ADR-003 | Object-safe path parameters                 | Accepted          |
| ADR-004 | Tower-style middleware pattern              | Accepted          |
| ADR-005 | `std::fs`-aligned method names              | Accepted          |
| ADR-006 | Quota for quota enforcement                 | Accepted          |
| ADR-007 | Restrictions for least-privilege            | Accepted          |
| ADR-008 | FileStorage as thin ergonomic wrapper       | Accepted          |
| ADR-009 | Built-in backends are feature-gated         | Accepted          |
| ADR-010 | Sync-first, async-ready design              | Accepted          |
| ADR-011 | Layer trait for standardized composition    | Accepted          |
| ADR-012 | Tracing for instrumentation                 | Accepted          |
| ADR-013 | FsExt for extension methods                 | Accepted          |
| ADR-014 | Optional Bytes support                      | Accepted          |
| ADR-015 | Contextual FsError                          | Accepted          |
| ADR-016 | PathFilter for path-based access control    | Accepted          |
| ADR-017 | ReadOnly for preventing writes              | Accepted          |
| ADR-018 | RateLimit for operation throttling          | Accepted          |
| ADR-019 | DryRun for testing and debugging            | Accepted          |
| ADR-020 | Cache for read performance                  | Accepted          |
| ADR-021 | Overlay for union filesystem                | Accepted          |
| ADR-022 | Builder pattern for configurable middleware | Accepted          |
| ADR-023 | Interior mutability for all trait methods   | Accepted          |
| ADR-024 | Async Strategy                              | Accepted          |
| ADR-025 | Strategic Boxing (Tower-style)              | Accepted          |
| ADR-026 | Companion shell (anyfs-shell)               | Accepted (Future) |
| ADR-027 | Permissive core; security via middleware    | Accepted          |
| ADR-028 | Linux-like semantics for virtual backends   | Accepted          |
| ADR-029 | Path resolution in FileStorage              | Accepted          |
| ADR-030 | Layered trait hierarchy                     | Accepted          |
| ADR-031 | Indexing as middleware                      | Accepted (Future) |
| ADR-032 | Path Canonicalization via FsPath Trait      | Accepted          |
| ADR-033 | PathResolver Trait for Pluggable Resolution | Accepted          |
| ADR-034 | LLM-Oriented Architecture (LOA)             | Accepted          |

---

## ADR-001: Path-based `Fs` trait

**Decision:** Backends implement a path-based trait aligned with `std::fs` method naming.

**Why:** Filesystem operations are naturally path-oriented; a single, familiar trait surface is easier to implement and adopt than graph-store or inode models.

---

## ADR-002: Two-crate structure

**Decision:**

| Crate           | Purpose                                                                  |
| --------------- | ------------------------------------------------------------------------ |
| `anyfs-backend` | Minimal contract: `Fs` trait, `Layer` trait, `FsExt`, types              |
| `anyfs`         | Backends + middleware + ergonomics (`FileStorage<B, M>`, `BackendStack`) |

**Why:**
- Backend authors only need `anyfs-backend` (no heavy dependencies).
- Middleware is composable and lives with backends in `anyfs`.
- `FileStorage` provides ergonomics plus centralized path resolution for virtual backends - no policy logic - included in `anyfs` for convenience.

---

## ADR-003: Object-safe path parameters

**Decision:** Core `Fs` traits take `&Path` so they remain object-safe (`dyn Fs` works). For ergonomics, `FileStorage` and `FsExt` accept `impl AsRef<Path>` and forward to the core traits.

**Why:**
- Object safety enables opt-in type erasure (`FileStorage::boxed()`).
- Keeps hot-path calls zero-cost; dynamic dispatch is explicit and optional.
- Ergonomics preserved via `FileStorage`/`FsExt` (`&str`, `String`, `PathBuf`).

---

## ADR-004: Tower-style middleware pattern

**Decision:** Use composable middleware (decorator pattern) for cross-cutting concerns like limits, logging, and feature gates. Each middleware implements `Fs` by wrapping another `Fs`.

**Why:**
- Complete separation of concerns - each layer has one job.
- Composable - use only what you need.
- Familiar pattern (Axum/Tower use the same approach).
- No code duplication - middleware written once, works with any backend.
- Testable - each layer can be tested in isolation.

**Example:**
```rust
let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .build())
    .layer(TracingLayer::new());
```

---

## ADR-005: `std::fs`-aligned method names

**Decision:** Prefer `read_dir`, `create_dir_all`, `remove_file`, etc.

**Why:** Familiarity and reduced cognitive overhead.

---

## ADR-006: Quota for quota enforcement

**Decision:** Quota/limit enforcement is handled by `Quota<B>` middleware, not by backends or FileStorage.

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

## ADR-007: Capability via Trait Bounds

**Decision:** Symlink and hard-link capability is determined by whether the backend implements `FsLink`. There is NO runtime toggle.

### The Rule

| Backend implements `FsLink`? | Symlinks work?     |
| ---------------------------- | ------------------ |
| Yes (`B: FsLink`)            | Yes                |
| No                           | No (won't compile) |

### Examples

```rust
// MemoryBackend implements FsLink
let fs = FileStorage::new(MemoryBackend::new());
fs.symlink("/target", "/link")?;  // ✅ Works

// Custom backend that doesn't implement FsLink
let fs = FileStorage::new(MySimpleBackend::new());
fs.symlink("/target", "/link")?;  // ❌ Won't compile - no FsLink impl
```

### Why Not Runtime Blocking?

A hypothetical `deny_symlinks()` middleware would create type/behavior mismatch:
- Type says "I implement FsLink"
- Runtime says "but symlink() errors"

This is confusing and defeats the purpose of Rust's type system. Instead, symlink capability is determined at compile time by trait bounds.

### Restrictions Middleware

`Restrictions<B>` is limited to operations where runtime policy makes sense:

```rust
let backend = RestrictionsLayer::builder()
    .deny_permissions()  // Prevent metadata changes
    .build()
    .layer(backend);
```

**Symlink following:** Part of backend semantics. FileStorage resolves paths for non-`SelfResolving` backends; OS-backed backends delegate to the OS (strict-path prevents escapes). There is no runtime toggle.

---

## ADR-008: FileStorage as thin ergonomic wrapper

**Decision:** `FileStorage<B, M>` is a thin wrapper that provides std::fs-aligned ergonomics and centralized path resolution for virtual backends. It contains NO policy logic.

**What it does:**
- Provides familiar method names
- Accepts `impl AsRef<Path>` for convenience and forwards to the core `&Path` traits
- Supports an optional marker type `M` for compile-time segregation of containers
- Delegates all operations to the wrapped backend

**What it does NOT do:**
- Quota enforcement (use Quota)
- Feature gating (use Restrictions)
- Instrumentation (use Tracing)
- Any other policy

**Why:**
- Single responsibility - ergonomics + path resolution (no policy).
- Users who don't need ergonomics can use backends directly.
- Policy is composable via middleware, not hardcoded.
- Marker types prevent accidental mixing of different storage domains without runtime tags.

---

## ADR-009: Built-in backends are feature-gated

**Decision:** `anyfs` uses Cargo features so users only pull the dependencies they need.

- `memory` (default)
- `sqlite` (optional)
- `vrootfs` (optional)

**Why:** Minimizes binary size and compile time for users who don't need all backends.

---

## ADR-010: Sync-first, async-ready design

**Decision:** `Fs` traits are synchronous. The API is designed to allow adding `AsyncFs` later without breaking changes.

**Rationale:**
- All built-in backends are naturally synchronous:
  - `MemoryBackend` - in-memory, instant
  - `SqliteBackend` - rusqlite is sync
  - `VRootFsBackend` - std::fs is sync
- Sync is simpler - no runtime dependency (tokio/async-std)
- Users can wrap sync backends in `spawn_blocking` if needed

**Async-ready design principles:**
- Traits require `Send` - compatible with async executors
- Return types are `Result<T, FsError>` - works with async
- No internal blocking assumptions
- Methods are stateless per-call - no hidden blocking state

**Future async path (Option 2):**
When async is needed (e.g., network-backed storage), add a parallel trait:

```rust
// In anyfs-backend
pub trait AsyncFs: Send + Sync {
    async fn read(&self, path: &Path) -> Result<Vec<u8>, FsError>;
    async fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError>;
    // ... mirrors Fs with async

    // Streaming uses AsyncRead/AsyncWrite
    async fn open_read(&self, path: &Path)
        -> Result<Box<dyn AsyncRead + Send + Unpin>, FsError>;
}
```

**Migration notes:**
- `AsyncFs` would be a separate trait, not replacing `Fs`
- Blanket impl possible: `impl<T: Fs> AsyncFs for T` using `spawn_blocking`
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
pub trait Layer<B: Fs> {
    type Backend: Fs;
    fn layer(self, backend: B) -> Self::Backend;
}
```

**Why:**
- Standardized composition pattern familiar to Tower/Axum users.
- IDE autocomplete for available layers.
- Enables `BackendStack` fluent builder in anyfs.
- Each middleware provides a corresponding `*Layer` type.

**Example:**
```rust
let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(100_000)
        .build())
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
backend.layer(TracingLayer::new()
    .with_target("anyfs")
    .with_level(tracing::Level::DEBUG))
```

---

## ADR-013: FsExt for extension methods

**Decision:** Provide `FsExt` trait with convenience methods, auto-implemented for all backends.

```rust
pub trait FsExt: Fs {
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;

    // JSON methods require `serde` feature
    #[cfg(feature = "serde")]
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, FsError>;
    #[cfg(feature = "serde")]
    fn write_json<T: Serialize>(&self, path: impl AsRef<Path>, value: &T) -> Result<(), FsError>;
}

impl<B: Fs> FsExt for B {}
```

**Feature gating:**
- `is_file()` and `is_dir()` are always available.
- `read_json()` and `write_json()` require `anyfs-backend = { features = ["serde"] }`.

**Why:**
- Adds convenience without bloating `Fs` trait.
- Blanket impl means all backends get these methods for free.
- Users can define their own extension traits for domain-specific operations.
- Follows Rust convention (e.g., `IteratorExt`, `StreamExt`).
- Serde is optional - users who don't need JSON avoid the dependency.

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
pub trait FsRead: Send {
    fn read(&self, path: &Path) -> Result<FileContent, FsError>;
    // ...
}
```

**Middleware compatibility:** Middleware passes `FileContent` through unchanged. No special handling needed - both `Vec<u8>` and `Bytes` implement `AsRef<[u8]>` and `Deref<Target=[u8]>`.

---

## ADR-015: Contextual FsError

**Decision:** `FsError` variants include context for better debugging.

```rust
FsError::NotFound {
    path: PathBuf,
}

FsError::QuotaExceeded {
    limit: u64,
    requested: u64,
    usage: u64,
}
```

**Why:**
- Error messages include enough context to understand what failed.
- No need for separate error context crate (like anyhow) for basic usage.
- Path is sufficient for NotFound - the call site knows the operation.
- Quota errors include all relevant numbers for debugging.

---

## ADR-016: PathFilter for path-based access control

**Decision:** Provide `PathFilter<B>` middleware for glob-based path access control.

**Configuration:**
```rust
PathFilterLayer::builder()
    .allow("/workspace/**")    // Allow workspace access
    .deny("**/.env")           // Deny .env files anywhere
    .deny("**/secrets/**")     // Deny secrets directories
    .build()
    .layer(backend)
```

**Semantics:**
- Rules are evaluated in order; first match wins.
- If no rules match, access is denied (deny by default).
- Uses glob patterns (e.g., `**` for recursive, `*` for single segment).
- Returns `FsError::AccessDenied` for denied paths.

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
- All write operations return `FsError::ReadOnly`.
- Simple, no configuration needed.

**Why:**
- Safe browsing of container contents without modification risk.
- Useful for debugging, inspection, auditing.
- Simpler than configuring Restrictions for read-only use case.

---

## ADR-018: RateLimit for operation throttling

**Decision:** Provide `RateLimit<B>` middleware to limit operations per time window.

**Configuration:**
```rust
RateLimitLayer::builder()
    .max_ops(1000)
    .per_second()
    .build()
    .layer(backend)
```

**Semantics:**
- Tracks operation count in sliding time window.
- Returns `FsError::RateLimitExceeded` when limit exceeded.
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
let dry_run = DryRun::new(backend);
let fs = FileStorage::new(dry_run);

fs.write("/test.txt", b"hello")?;  // Logged but not written
// To inspect recorded operations, keep the DryRun handle before wrapping it.
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
CacheLayer::builder()
    .max_entries(1000)
    .max_entry_size(1024 * 1024)  // 1MB max per entry
    .ttl(Duration::from_secs(60))
    .build()
    .layer(backend)
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

---

## ADR-022: Builder pattern for configurable middleware

**Decision:** Middleware that requires configuration MUST use a builder pattern that prevents construction without meaningful values. `::new()` constructors are NOT allowed for middleware where a default configuration is nonsensical.

**Problem:** A constructor like `QuotaLayer::new()` raises the question: "What quota?" An unlimited quota is pointless - you wouldn't use `QuotaLayer` at all. Similarly, `RestrictionsLayer::new()` with no restrictions, `PathFilterLayer::new()` with no rules, and `RateLimitLayer::new()` with no rate limit are all nonsensical.

**Solution:** Use builders that enforce at least one meaningful configuration:

```rust
// QuotaLayer - requires at least one limit
let quota = QuotaLayer::builder()
    .max_total_size(100 * 1024 * 1024)
    .build();

// Can also set multiple limits
let quota = QuotaLayer::builder()
    .max_total_size(1_000_000)
    .max_file_size(100_000)
    .max_node_count(1000)
    .build();

// RestrictionsLayer - requires at least one restriction
let restrictions = RestrictionsLayer::builder()
    .deny_permissions()
    .build();

// PathFilterLayer - requires at least one rule
let filter = PathFilterLayer::builder()
    .allow("/workspace/**")
    .deny("**/.env")
    .build();

// RateLimitLayer - requires rate limit parameters
let rate_limit = RateLimitLayer::builder()
    .max_ops(1000)
    .per_second()
    .build();

// CacheLayer - requires cache configuration
let cache = CacheLayer::builder()
    .max_entries(1000)
    .build();
```

**Middleware that MAY keep `::new()`:**

| Middleware      | Rationale                                                           |
| --------------- | ------------------------------------------------------------------- |
| `TracingLayer`  | Default (global tracing subscriber) is meaningful                   |
| `ReadOnlyLayer` | No configuration needed                                             |
| `DryRunLayer`   | No configuration needed                                             |
| `OverlayLayer`  | Takes two backends as required params: `Overlay::new(lower, upper)` |

**Implementation:**

```rust
// Builder with typestate pattern for compile-time enforcement
pub struct QuotaLayerBuilder<State = Unconfigured> {
    max_total_size: Option<u64>,
    max_file_size: Option<u64>,
    max_node_count: Option<u64>,
    _state: PhantomData<State>,
}

pub struct Unconfigured;
pub struct Configured;

impl QuotaLayerBuilder<Unconfigured> {
    pub fn max_total_size(mut self, bytes: u64) -> QuotaLayerBuilder<Configured> {
        self.max_total_size = Some(bytes);
        QuotaLayerBuilder {
            max_total_size: self.max_total_size,
            max_file_size: self.max_file_size,
            max_node_count: self.max_node_count,
            _state: PhantomData,
        }
    }

    pub fn max_file_size(mut self, bytes: u64) -> QuotaLayerBuilder<Configured> {
        // Similar transition to Configured state
    }

    pub fn max_node_count(mut self, count: u64) -> QuotaLayerBuilder<Configured> {
        // Similar transition to Configured state
    }

    // Note: NO build() method on Unconfigured state!
}

impl QuotaLayerBuilder<Configured> {
    // Additional configuration methods stay in Configured state
    pub fn max_total_size(mut self, bytes: u64) -> Self {
        self.max_total_size = Some(bytes);
        self
    }

    // Only Configured state has build()
    pub fn build(self) -> QuotaLayer {
        QuotaLayer { /* ... */ }
    }
}

impl QuotaLayer {
    pub fn builder() -> QuotaLayerBuilder<Unconfigured> {
        QuotaLayerBuilder {
            max_total_size: None,
            max_file_size: None,
            max_node_count: None,
            _state: PhantomData,
        }
    }
}
```

**Why:**
- **Compile-time safety:** Invalid configurations don't compile.
- **Self-documenting API:** Users must explicitly choose configuration.
- **No meaningless defaults:** Eliminates "what does this default to?" confusion.
- **IDE guidance:** Autocomplete shows required methods before `build()`.
- **Familiar pattern:** Rust builders are idiomatic and widely understood.

**Error prevention:**
```rust
// This won't compile - no build() on Unconfigured
let quota = QuotaLayer::builder().build();  // ❌ Error!

// This compiles - at least one limit set
let quota = QuotaLayer::builder()
    .max_total_size(1_000_000)
    .build();  // ✅ OK
```

---

## ADR-023: Interior mutability for all trait methods

**Decision:** All `Fs` trait methods use `&self`, not `&mut self`. Backends manage their own synchronization internally (interior mutability).

**Previous design:**
```rust
pub trait FsRead: Send {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError>;
}

pub trait FsWrite: Send {
    fn write(&mut self, path: &Path, data: &[u8]) -> Result<(), FsError>;
}
```

**New design:**
```rust
pub trait FsRead: Send + Sync {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError>;
}

pub trait FsWrite: Send + Sync {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError>;
}
```

**Why:**

1. **Filesystems are conceptually always mutable.** A filesystem doesn't become "borrowed" when you write to it - the underlying storage manages concurrency itself.

2. **Enables concurrent access patterns.** With `&mut self`, you cannot have concurrent readers and writers even when the backend supports it (e.g., SQLite with WAL mode, real filesystems).

3. **Matches real-world filesystem semantics.** `std::fs::write()` takes a path, not a mutable reference to some filesystem object. Files are shared resources.

4. **Simplifies middleware implementation.** Middleware no longer needs to worry about propagating mutability - all operations use `&self`.

5. **Common pattern in Rust.** Many I/O abstractions use interior mutability: `std::io::Write` for `File` (via OS handles), `tokio::fs`, database connection pools, etc.

**Implementation:**

Backends use appropriate synchronization primitives:

```rust
pub struct MemoryBackend {
    // Interior mutability via Mutex/RwLock
    data: RwLock<HashMap<PathBuf, Vec<u8>>>,
}

impl FsWrite for MemoryBackend {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        let mut guard = self.data.write().unwrap();
        guard.insert(path.as_ref().to_path_buf(), data.to_vec());
        Ok(())
    }
}

pub struct SqliteBackend {
    // SQLite handles its own locking
    conn: Connection,  // rusqlite::Connection is internally synchronized
}
```

**Trade-offs:**

| Aspect              | &mut self              | &self (interior mutability)    |
| ------------------- | ---------------------- | ------------------------------ |
| Compile-time safety | Single writer enforced | Runtime synchronization        |
| Concurrent access   | Not possible           | Backend decides                |
| API simplicity      | Simple                 | Slightly more complex backends |
| Real-world match    | Poor                   | Good                           |

**Backend implementer responsibility:**

Backends MUST use interior mutability (`RwLock`, `Mutex`, etc.) to ensure thread-safe concurrent access. This guarantees:
- Memory safety (no data corruption)
- Atomic operations (a single `write()` won't produce partial results)

This does NOT guarantee:
- Order of concurrent writes to the same path (last write wins - standard FS behavior)

**Conclusion:** The benefits of matching filesystem semantics and enabling concurrent access outweigh the loss of compile-time single-writer enforcement. Backends are responsible for their own thread safety via interior mutability.

---

## ADR-024: Async Strategy

**Status:** Accepted
**Context:** Async/await is prevalent in Rust networking and I/O. While AnyFS is primarily sync-focused (matching `std::fs`), we may need async support in the future for:
- Network-backed storage (S3, WebDAV, etc.)
- High-concurrency scenarios
- Integration with async runtimes (tokio, async-std)

**Decision:** Plan for a **parallel async trait hierarchy** that mirrors the sync traits.

**Strategy:**

```
Sync Traits          Async Traits
-----------          ------------
FsRead        →      AsyncFsRead
FsWrite       →      AsyncFsWrite
FsDir         →      AsyncFsDir
Fs            →      AsyncFs
FsFull        →      AsyncFsFull
FsFuse        →      AsyncFsFuse
FsPosix       →      AsyncFsPosix
```

**Design principles:**

1. **Separate crate:** Async traits live in `anyfs-async` to avoid pulling async dependencies into the core.

2. **Method parity:** Each async trait method corresponds 1:1 with its sync counterpart:
   ```rust
   // Sync (anyfs-backend)
   pub trait FsRead: Send + Sync {
       fn read(&self, path: &Path) -> Result<Vec<u8>, FsError>;
   }

   // Async (anyfs-async)
   #[async_trait]
   pub trait AsyncFsRead: Send + Sync {
       async fn read(&self, path: &Path) -> Result<Vec<u8>, FsError>;
   }
   ```

3. **Layer trait compatibility:** The `Layer` trait works for both sync and async:
   ```rust
   pub trait Layer<B> {
       type Backend;
       fn layer(self, backend: B) -> Self::Backend;
   }

   // Middleware can implement for both:
   impl<B: Fs> Layer<B> for QuotaLayer {
       type Backend = Quota<B>;
       fn layer(self, backend: B) -> Self::Backend { ... }
   }

   impl<B: AsyncFs> Layer<B> for QuotaLayer {
       type Backend = AsyncQuota<B>;
       fn layer(self, backend: B) -> Self::Backend { ... }
   }
   ```

4. **Sync-to-async bridge:** Provide adapters for using sync backends in async contexts:
   ```rust
   // Wraps sync backend for use in async code (uses spawn_blocking)
   pub struct SyncToAsync<B>(B);

   impl<B: Fs> AsyncFsRead for SyncToAsync<B> {
       async fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
           let path = path.as_ref().to_path_buf();
           let backend = self.0.clone(); // requires Clone
           tokio::task::spawn_blocking(move || backend.read(&path)).await?
       }
   }
   ```

5. **No async-to-sync bridge:** We intentionally don't provide async-to-sync adapters (would require blocking on async runtime, which is problematic).

**Implementation phases:**

| Phase | Scope               | Dependency  |
| ----- | ------------------- | ----------- |
| 1     | Sync traits stable  | Now         |
| 2     | Design async traits | When needed |
| 3     | `anyfs-async` crate | When needed |
| 4     | Async middleware    | When needed |

**Why parallel traits (not feature flags):**

- **No conditional compilation complexity** - sync and async are separate, clean codebases
- **No trait object issues** - async traits have different object safety requirements
- **Clear dependency boundaries** - sync code doesn't pull in tokio/async-std
- **Ecosystem alignment** - mirrors how `std::io` vs `tokio::io` work

**Trade-offs:**

| Approach        | Pros                                    | Cons                                       |
| --------------- | --------------------------------------- | ------------------------------------------ |
| Parallel traits | Clean separation, no async deps in core | Code duplication in middleware             |
| Feature flags   | Single codebase                         | Complex conditional compilation            |
| Async-only      | Modern, no duplication                  | Forces async runtime on sync users         |
| Sync-only       | Simple                                  | Can't support network backends efficiently |

**Conclusion:** Parallel async traits provide the best balance of simplicity now (sync-only core) with a clear migration path for async support later. The `Layer` trait design already accommodates this pattern.

---

## ADR-025: Strategic Boxing (Tower-style)

**Status:** Accepted

**Context:** Dynamic dispatch (`Box<dyn Trait>`) adds heap allocation and vtable indirection. We need to decide where boxing is acceptable vs. where zero-cost abstractions are required.

**Decision:** Follow Tower/Axum's battle-tested strategy: **zero-cost on the hot path, box at boundaries where flexibility is needed and I/O cost dominates.**

**Principle:** Avoid heap allocations and dynamic dispatch unless they buy real flexibility with negligible performance impact. Box only at cold boundaries (streams/iterators), and make type erasure explicit and opt-in.

**DX stance:** Application code uses `FileStorage`/`FsExt` (std::fs-style paths). Core traits stay object-safe for `dyn Fs`. For hot loops on known concrete backends, we provide a typed streaming extension as the first-class zero-alloc fast path.

### Boxing Strategy

```
HOT PATH (many calls per operation - must be zero-cost):
┌─────────────────────────────────────────────────────┐
│  read(), write(), metadata(), exists()              │  ← Returns concrete types
│  Read::read() / Write::write() on streams           │  ← Vtable dispatch only
│  Iterator::next() on ReadDirIter                    │  ← Vtable dispatch only
│  Middleware composition                             │  ← Generics, monomorphized
└─────────────────────────────────────────────────────┘

COLD PATH (once per operation - boxing acceptable):
┌─────────────────────────────────────────────────────┐
│  open_read(), open_write()                          │  ← Box<dyn Read/Write>
│  read_dir()                                         │  ← ReadDirIter (boxed inner)
└─────────────────────────────────────────────────────┘

SETUP (once at startup - zero-cost):
┌─────────────────────────────────────────────────────┐
│  Middleware stacking: Quota<Tracing<B>>             │  ← Generics, no boxing
│  FileStorage::new(backend)                          │  ← Zero-cost wrapper
└─────────────────────────────────────────────────────┘

OPT-IN TYPE ERASURE (when explicitly needed):
┌─────────────────────────────────────────────────────┐
│  FileStorage::boxed() -> FileStorage<Box<dyn Fs>>   │  ← Like Tower's BoxService
└─────────────────────────────────────────────────────┘
```

### What Gets Boxed and Why

| API                               | Boxed?      | Rationale                                              |
| --------------------------------- | ----------- | ------------------------------------------------------ |
| `read()` → `Vec<u8>`              | No          | Hot path, most common operation                        |
| `write(data)` → `()`              | No          | Hot path, most common operation                        |
| `metadata()` → `Metadata`         | No          | Hot path, frequently called                            |
| `exists()` → `bool`               | No          | Hot path, frequently called                            |
| `open_read()` → `Box<dyn Read>`   | Yes         | Cold path (once per file), enables middleware wrappers |
| `open_write()` → `Box<dyn Write>` | Yes         | Cold path (once per file), enables `QuotaWriter`       |
| `read_dir()` → `ReadDirIter`      | Yes (inner) | Enables filtering in PathFilter, merging in Overlay    |
| Middleware stack                  | No          | Generics compose at compile time                       |
| `FileStorage::boxed()`            | Opt-in      | Explicit type erasure when needed                      |

### Why This Works

**1. Bulk operations are the common case:**
Most code uses `read()` and `write()`, not streaming. These are zero-cost.

**2. Streaming is for large files:**
`open_read()` / `open_write()` are for files too large to load into memory. For large files, I/O time (1-100ms) dwarfs box allocation (~50ns).

**3. Box once, vtable many:**
After `open_read()` allocates once, subsequent `Read::read()` calls are just vtable dispatch - no further allocations.

**4. Middleware needs flexibility:**
- `Quota` wraps streams with `QuotaWriter` to count bytes
- `PathFilter` filters `ReadDirIter` to hide denied entries
- `Overlay` merges directory listings from two backends
Boxing enables this without type explosion.

### Comparison to Tower/Axum

| AnyFS                  | Tower/Axum                       | Purpose                          |
| ---------------------- | -------------------------------- | -------------------------------- |
| `Quota<Tracing<B>>`    | `Timeout<RateLimit<S>>`          | Zero-cost middleware composition |
| `Box<dyn Read>`        | `Pin<Box<dyn Future>>`           | Flexibility at boundaries        |
| `ReadDirIter`          | `BoxedIntoRoute`                 | Type erasure for storage         |
| `FileStorage::boxed()` | `BoxService` / `BoxCloneService` | Opt-in type erasure              |

Tower's Timeout middleware [uses `Pin<Box<dyn Future>>`](https://docs.rs/tower/latest/tower/trait.Service.html) in practice. Axum's Router [uses `BoxedIntoRoute`](https://github.com/tokio-rs/axum/discussions/1438) to store handlers. We follow the same pattern.

### Cost Analysis

| Operation                   | Box Allocation | Actual I/O   | Box % of Total |
| --------------------------- | -------------- | ------------ | -------------- |
| Open + read 4KB file        | ~50ns          | ~10,000ns    | 0.5%           |
| Open + read 1MB file        | ~50ns          | ~1,000,000ns | 0.005%         |
| List directory (10 entries) | ~50ns          | ~5,000ns     | 1%             |

**The boxing cost is negligible relative to actual I/O.**

### Alternatives Considered

**1. Associated types everywhere:**
```rust
pub trait FsRead {
    type Reader: Read + Send;
    fn open_read(&self, path: &Path) -> Result<Self::Reader, FsError>;
}
```
Rejected: Causes type explosion. `QuotaReader<PathFilterReader<TracingReader<Cursor<Vec<u8>>>>>` is unwieldy and every middleware needs a custom wrapper type.

**2. RPITIT (Rust 1.75+):**
```rust
fn open_read(&self, path: &Path) -> Result<impl Read + Send, FsError>;
```
Rejected as default: Loses object safety. Can't use `dyn Fs` for runtime backend selection.

**3. Always box everything:**
Rejected: Unnecessary overhead on hot path operations like `read()`.

### Future Considerations

If profiling shows stream boxing is a bottleneck (unlikely), we can add:

```rust
/// Extension trait for zero-cost streaming when backend type is known
pub trait FsReadTyped: FsRead {
    type Reader: Read + Send;
    fn open_read_typed(&self, path: &Path) -> Result<Self::Reader, FsError>;
}
```

This follows Tower's pattern of providing both `Service` (with associated types) and `BoxService` (with type erasure).

### Conclusion

Our boxing strategy mirrors Tower/Axum's production-proven approach:
- **Zero-cost where it matters** (hot path bulk operations, middleware composition)
- **Box where flexibility is needed** (streaming I/O, iterator filtering)
- **Opt-in type erasure** (explicit `boxed()` method)

The performance cost is negligible (<1% of I/O time), while the ergonomic and flexibility benefits are substantial.

---

## ADR-026: Companion shell (anyfs-shell)

**Status:** Accepted (Future)

**Context:** Users want a low-friction way to explore how different backends and middleware behave without writing a full application.

**Decision:** Provide a separate companion crate (e.g., `anyfs-shell`) that exposes a bash-style navigation and file management interface built on `FileStorage`.

**Scope:**
- Commands: `ls`, `cd`, `cat`, `cp`, `mv`, `rm`, `mkdir`, `stat`.
- Navigation and file management only; no full bash scripting, pipes, or job control.
- All operations route through `FileStorage` to exercise middleware and backend composition.

**Why:**
- Demonstrates backend neutrality and middleware effects in a tangible way.
- Useful for docs, demos, and quick validation.
- Keeps the core crates free of CLI/UI dependencies.

---

## ADR-027: Permissive core; security via middleware

**Status:** Accepted

**Context:** We need predictable filesystem semantics across backends. Some use cases require strict sandboxing, while others expect full filesystem behavior. Baking security restrictions into core traits would make behavior surprising and backend-dependent.

**Decision:** Core traits are permissive: all operations supported by a backend are allowed by default. Security controls (limits, access control, read-only, rate limiting, audit) are applied via middleware such as `Restrictions`, `PathFilter`, `ReadOnly`, `Quota`, `RateLimit`, and `Tracing`.

**Why:**
- Predictability: core behavior matches `std::fs` expectations.
- Backend-agnostic: virtual and host backends share the same contract.
- Separation of concerns: policy lives in middleware, not storage semantics.
- Explicit security posture: applications opt in to the protections they need.

---

## ADR-028: Linux-like semantics for virtual backends

**Status:** Accepted

**Context:** Cross-platform filesystems differ in case sensitivity, separators, reserved names, and path length limits. Virtual backends need a consistent model that does not inherit OS quirks.

**Decision:** Virtual backends use Linux-like semantics by default:
- Case-sensitive paths
- `/` as the internal separator
- No reserved names
- No max path length
- No ADS (`:stream`) support

**Why:**
- Cross-platform consistency for the same data.
- Fewer surprises and reduced security footguns.
- Simplifies backend implementation and testing.
- Custom semantics remain possible via middleware or custom backends.

---

## ADR-029: Path resolution in FileStorage

**Status:** Accepted

**Context:** Path normalization (`//`, `.`, `..`) and symlink resolution must be consistent across backends. Implementing this logic in every backend is error-prone and leads to divergent behavior.

**Decision:** `FileStorage` performs canonicalization and normalization for virtual backends. Backends receive resolved paths. Real filesystem backends (e.g., `VRootFsBackend`) delegate to OS resolution plus `strict-path` containment. `FileStorage` exposes `canonicalize`, `soft_canonicalize`, and `anchored_canonicalize` for explicit use.

**Why:**
- Consistent semantics across all backends.
- Centralizes security-critical path handling.
- Simplifies backend implementations.
- Makes conformance testing straightforward.

---

## ADR-030: Layered trait hierarchy

**Status:** Accepted

**Context:** Not all backends can or should implement full POSIX behavior. Forcing a single large trait would make simple backends harder to implement and would obscure capabilities.

**Decision:** Split the API into layered traits:
- Core: `FsRead`, `FsWrite`, `FsDir` (combined as `Fs`)
- Extensions: `FsLink`, `FsPermissions`, `FsSync`, `FsStats`
- FUSE: `FsInode`
- POSIX: `FsHandles`, `FsLock`, `FsXattr`
- Convenience supertraits: `Fs`, `FsFull`, `FsFuse`, `FsPosix`

**Why:**
- Implement the lowest level you need.
- Clear capability boundaries and trait bounds.
- Avoids forcing unsupported features on backends.
- Enables middleware to target specific capabilities.

---

## ADR-031: Indexing as middleware

**Status:** Accepted (Future)

**Context:** We want a durable, queryable index of file activity and metadata (for audit trails, drive management tools, and statistics). This indexing should be optional, configurable, and work across all backends.

**Decision:** Indexing is implemented as middleware (`Indexing<B>` with `IndexLayer`), not as a specialized backend. The middleware writes to a sidecar index (SQLite by default) and can evolve to support alternate index engines.

**Naming:** Use `IndexLayer` (builder) and `Indexing<B>` (middleware), consistent with existing layer naming.

**Why:**
- **Separation of concerns:** Indexing is policy/analytics, not storage semantics.
- **Backend-agnostic:** Works with Memory, SQLite, VRootFs, and custom backends.
- **Composability:** Users opt in and configure it like other middleware (Quota, Tracing).
- **Flexibility:** Allows future index engines without changing core traits.
- **DX consistency:** Keeps std::fs-style usage via `FileStorage` with no API changes.

**Trade-offs:**
- **External OS changes:** Not captured unless a future watcher/scan helper is added.
- **Index failures:** Choose between strict mode (fail the op) and best-effort mode.

**Implementation sketch:**
- `IndexLayer::builder().index_file("index.db").consistency(IndexConsistency::Strict)...`
- Wraps `open_write()` with a counting writer to record final size on close.
- Updates a `nodes` table and logs `ops` entries per operation.

---

## ADR-032: Path Canonicalization via FsPath Trait

**Status:** Accepted

**Context:** Path canonicalization (resolving `..`, `.`, and symlinks) is needed for consistent path handling. The naive approach of baking this into `FileStorage` has issues:
- It's not testable in isolation
- It can't be optimized per-backend
- N+1 queries for paths like `/a/b/c/d/e` (each component = separate call)

**Decision:** Introduce an `FsPath` trait with `canonicalize()` and `soft_canonicalize()` methods that have **default implementations** but allow **backend-specific optimizations**.

**The Pattern:**

```rust
pub trait FsPath: FsRead + FsLink {
    /// Resolve all symlinks and normalize path components.
    /// Returns error if final path doesn't exist.
    fn canonicalize(&self, path: &Path) -> Result<PathBuf, FsError> {
        default_canonicalize(self, path)
    }

    /// Like canonicalize, but allows non-existent final component.
    fn soft_canonicalize(&self, path: &Path) -> Result<PathBuf, FsError> {
        default_soft_canonicalize(self, path)
    }
}

// Auto-implement for all FsLink implementors
impl<T: FsRead + FsLink> FsPath for T {}
```

**Default Implementation:**

```rust
fn default_canonicalize<F: FsRead + FsLink>(fs: &F, path: &Path) -> Result<PathBuf, FsError> {
    let mut resolved = PathBuf::from("/");
    for component in path.components() {
        match component {
            Component::RootDir => resolved = PathBuf::from("/"),
            Component::ParentDir => { resolved.pop(); },
            Component::CurDir => {},
            Component::Normal(name) => {
                resolved.push(name);
                if let Ok(meta) = fs.symlink_metadata(&resolved) {
                    if meta.file_type.is_symlink() {
                        let target = fs.read_link(&resolved)?;
                        resolved.pop();
                        resolved = resolve_relative(&resolved, &target);
                    }
                }
            },
            _ => {},
        }
    }
    // Verify final path exists
    if !fs.exists(&resolved)? {
        return Err(FsError::NotFound { path: resolved, operation: "canonicalize" });
    }
    Ok(resolved)
}
```

**Backend Optimization Examples:**

| Backend          | Optimization                                               |
| ---------------- | ---------------------------------------------------------- |
| `SqliteBackend`  | Single recursive CTE query resolves entire path            |
| `VRootFsBackend` | Delegates to `std::fs::canonicalize()` + containment check |
| `MemoryBackend`  | Uses default (in-memory is fast anyway)                    |

**SQLite Optimized Implementation:**

```rust
impl FsPath for SqliteBackend {
    fn canonicalize(&self, path: &Path) -> Result<PathBuf, FsError> {
        // Single query with recursive CTE
        self.conn.query_row(
            r#"
            WITH RECURSIVE path_resolve(segment, remaining, resolved, depth) AS (
                -- Initial: split path into segments
                SELECT ..., 0
                UNION ALL
                -- Recursive: resolve each segment, following symlinks
                SELECT ...
                FROM path_resolve
                JOIN nodes ON ...
                WHERE depth < 40  -- Loop protection
            )
            SELECT resolved FROM path_resolve 
            WHERE remaining = '' 
            ORDER BY depth DESC LIMIT 1
            "#,
            params![path.to_string_lossy()],
            |row| Ok(PathBuf::from(row.get::<_, String>(0)?))
        ).map_err(|e| FsError::NotFound { path: path.into(), operation: "canonicalize" })
    }
}
```

**Why This Design:**

| Benefit              | Explanation                                            |
| -------------------- | ------------------------------------------------------ |
| **Portable default** | Works with any `Fs` backend out of the box             |
| **Optimizable**      | Backends can override for O(1) queries vs O(n)         |
| **Testable**         | Canonicalization logic is separate, can be unit tested |
| **Composable**       | Middleware can wrap/intercept canonicalization         |

**FileStorage Integration:**

```rust
impl<B: Fs + FsPath, M> FileStorage<B, M> {
    pub fn canonicalize(&self, path: impl AsRef<Path>) -> Result<PathBuf, FsError> {
        self.backend.canonicalize(path.as_ref())
    }

    pub fn soft_canonicalize(&self, path: impl AsRef<Path>) -> Result<PathBuf, FsError> {
        self.backend.soft_canonicalize(path.as_ref())
    }
}
```

**Trade-offs:**

| Approach      | Queries            | Complexity | Best For                   |
| ------------- | ------------------ | ---------- | -------------------------- |
| Default impl  | O(n) per component | Simple     | Memory, small files        |
| SQLite CTE    | O(1) single query  | Moderate   | Large trees, many symlinks |
| OS delegation | O(1) syscall       | Simple     | Real filesystem            |

**Conclusion:** The `FsPath` trait provides a clean abstraction that works everywhere but can be optimized where it matters. This follows Rust's "zero-cost abstractions" philosophy: you don't pay for what you don't use, and you can optimize hot paths when needed.

---

## ADR-033: PathResolver Trait for Pluggable Resolution

**Status:** Accepted

**Context:** Path resolution (normalizing `..`, `.`, and following symlinks) is currently handled in two places:
1. `FsPath` trait methods (`canonicalize`, `soft_canonicalize`) with backend-specific optimizations
2. `FileStorage` performs pre-resolution for non-`SelfResolving` backends
3. `SelfResolving` marker trait opts out of FileStorage resolution

This works, but the resolution **algorithm** is not a first-class, testable unit. The logic is spread across components, making it harder to:
- Test path resolution in isolation
- Benchmark/profile resolution performance
- Provide third-party custom resolvers
- Explore alternative resolution strategies (case-insensitive, caching, etc.)

**Decision:** Introduce a `PathResolver` trait that encapsulates the path resolution algorithm as a standalone, pluggable component.

**The Pattern:**

```rust
// In anyfs-backend (trait definition)
/// Strategy trait for path resolution algorithms.
///
/// Encapsulates path normalization, `..`/`.` resolution, and optionally symlink following.
/// Symlink resolution requires the backend to implement `FsLink`. If the backend only
/// implements `Fs`, the resolver will normalize paths and resolve `.`/`..` but cannot
/// follow symlinks (they are treated as regular files/directories).
pub trait PathResolver: Send + Sync {
    /// Resolve path to canonical form.
    /// If backend implements `FsLink`, symlinks are followed up to max depth.
    /// If backend only implements `Fs`, symlinks are not followed.
    fn canonicalize(&self, path: &Path, fs: &dyn Fs) -> Result<PathBuf, FsError>;
    
    /// Like canonicalize, but allows non-existent final component.
    fn soft_canonicalize(&self, path: &Path, fs: &dyn Fs) -> Result<PathBuf, FsError>;
}
```

**Note on symlink handling:** The trait accepts `&dyn Fs` for object safety, but implementations can attempt to downcast to `&dyn FsLink` when symlink awareness is needed. All built-in virtual backends implement `FsLink`, so this works seamlessly. For backends without `FsLink`, resolution still works but treats all entries as non-symlinks.

**Built-in Implementations (in `anyfs` crate):**

```rust
/// Default iterative resolver - walks path component by component.
/// Follows symlinks if the backend implements FsLink.
pub struct IterativeResolver {
    max_symlink_depth: usize,  // Default: 40
}

/// No-op resolver for SelfResolving backends (OS handles resolution).
pub struct NoOpResolver;

/// LRU cache wrapper around another resolver.
pub struct CachingResolver<R: PathResolver> {
    inner: R,
    cache: Cache<PathBuf, PathBuf>,  // LRU cache, bounded size
}

// Case-folding resolver is NOT built-in. Users can implement one via PathResolver
// trait if needed, but real-world demand is minimal since VRootFsBackend on
// Windows/macOS already gets case-insensitivity from the OS.
```

**Integration with FileStorage:**

```rust
impl<B: Fs, M> FileStorage<B, IterativeResolver, M> {
    pub fn new(backend: B) -> Self { ... }
}

impl<B: Fs, R: PathResolver, M> FileStorage<B, R, M> {
    pub fn with_resolver(backend: B, resolver: R) -> Self {
        Self { backend, resolver, _marker: PhantomData }
    }
}

// Usage
let fs = FileStorage::new(backend);  // Uses IterativeResolver
let fs = FileStorage::with_resolver(backend, CachingResolver::new(IterativeResolver::default()));
```

**Relationship with FsPath Trait:**

| Component       | Responsibility                                                                  |
| --------------- | ------------------------------------------------------------------------------- |
| `PathResolver`  | Algorithm for resolution (first-class, testable, swappable)                     |
| `FsPath`        | Backend-level optimization hook (can delegate to resolver or override entirely) |
| `SelfResolving` | Remains as marker OR becomes `NoOpResolver` assignment                          |

`FsPath` can delegate to the resolver:

```rust
pub trait FsPath: FsRead + FsLink {
    fn resolver(&self) -> &dyn PathResolver {
        static DEFAULT: IterativeResolver = IterativeResolver::new();
        &DEFAULT
    }
    
    fn canonicalize(&self, path: &Path) -> Result<PathBuf, FsError> {
        self.resolver().canonicalize(path, self)
    }
}
```

Or backends can override entirely for optimized implementations (e.g., SQLite CTE).

**Why This Design:**

| Benefit                    | Explanation                                                    |
| -------------------------- | -------------------------------------------------------------- |
| **Testable in isolation**  | Unit test resolvers without full backend setup                 |
| **Benchmarkable**          | Profile resolution algorithms independently                    |
| **Third-party extensible** | Custom resolvers without touching `Fs` traits                  |
| **Maintainable**           | Path resolution is one focused, isolated component             |
| **New capabilities**       | Case-insensitive, caching, Windows-style resolvers become easy |
| **Backwards compatible**   | Existing FsPath overrides still work; resolver is additive     |

**Crate Placement:**

| Component               | Crate           | Rationale                        |
| ----------------------- | --------------- | -------------------------------- |
| `PathResolver` trait    | `anyfs-backend` | Core contract, minimal deps      |
| `IterativeResolver`     | `anyfs`         | Default impl, needs Fs methods   |
| `NoOpResolver`          | `anyfs`         | For SelfResolving backends       |
| `CachingResolver`       | `anyfs`         | Optional, needs cache impl       |
| FileStorage integration | `anyfs`         | Uses resolvers for path handling |

> **Note:** Case-folding resolvers are NOT built-in. The `PathResolver` trait allows users to implement custom resolvers if needed, but we don't ship speculative features.

**Example Use Cases:**

```rust
// Default: case-sensitive, symlink-aware (IterativeResolver is ZST, zero-cost)
let fs = FileStorage::new(MemoryBackend::new());

// Caching for read-heavy workloads
let fs = FileStorage::with_resolver(
    backend,
    CachingResolver::new(IterativeResolver::default())
);

// Custom resolver (user-implemented)
let fs = FileStorage::with_resolver(backend, MyCustomResolver::new());

// Testing: verify resolution behavior in isolation
#[test]
fn test_symlink_loop_detection() {
    let resolver = IterativeResolver::new();
    let mock_fs = MockFs::with_symlink_loop();
    let result = resolver.canonicalize(Path::new("/loop"), &mock_fs);
    assert!(matches!(result, Err(FsError::InvalidData { .. })));
}
```

**Conclusion:** The `PathResolver` trait provides clean separation of concerns, making path resolution testable, benchmarkable, and extensible. It complements `FsPath` (backend optimization hook) and can replace or work alongside `SelfResolving` (via `NoOpResolver`).

---

## ADR-034: LLM-Oriented Architecture (LOA)

**Status:** Accepted

**Context:** AnyFS is being developed with significant LLM assistance (GitHub Copilot, Claude, etc.). Traditional software architecture prioritizes maintainability, testability, and extensibility for **human developers**. However, when LLMs are part of the development workflow, additional constraints become essential:

1. LLMs work best with **limited context windows** - they can't "understand" an entire codebase
2. LLMs excel at **pattern matching** - consistent structure enables better assistance
3. LLMs need **clear contracts** - well-documented interfaces reduce hallucination
4. LLMs benefit from **isolated components** - fixing one thing shouldn't require understanding everything

These same properties also benefit:
- Open source contributors (quick onboarding)
- Code review (focused changes)
- Parallel development (independent components)
- AI-generated tests and documentation

**Decision:** Structure AnyFS using **LLM-Oriented Architecture (LOA)** - a methodology where every component is independently understandable, testable, and fixable with only local context.

**The Five Pillars:**

| Pillar                    | Description            | Implementation                     |
| ------------------------- | ---------------------- | ---------------------------------- |
| **Single Responsibility** | One file = one concept | `quota.rs`, `iterative.rs`, etc.   |
| **Contract-First**        | Traits define the spec | Documented trait invariants        |
| **Isolated Testing**      | Tests use mocks only   | No real backends in unit tests     |
| **Rich Errors**           | Errors explain the fix | Context in every `FsError` variant |
| **Boundary Docs**         | Examples at every API  | Usage example in every doc comment |

**File Structure Convention:**

```rust
//! # Component Name
//!
//! ## Responsibility
//! - Single bullet point
//!
//! ## Dependencies  
//! - Traits/types only
//!
//! ## Usage
//! ```rust
//! // Minimal example
//! ```

// ============================================================================
// Types
// ============================================================================

// ============================================================================
// Trait Implementations
// ============================================================================

// ============================================================================
// Public API
// ============================================================================

// ============================================================================
// Private Helpers
// ============================================================================

// ============================================================================
// Tests
// ============================================================================
```

**Component Isolation Checklist:**

- [ ] Single file per component
- [ ] Implements a trait with documented invariants
- [ ] Dependencies are traits/types, not implementations
- [ ] Tests use mocks, not real backends
- [ ] Error messages explain what went wrong and how to fix
- [ ] Doc comment shows standalone usage example
- [ ] No global state
- [ ] `Send + Sync` where required

**LLM Prompting Patterns:**

The architecture enables these clean prompts:

```
# Implement (user-provided resolver example)
Implement a case-folding resolver in your project.
Contract: Implement `PathResolver` trait from anyfs-backend.
Test: "/Foo/BAR" → "/foo/bar"

# Fix
Bug: Quota<B> doesn't account for existing file size.
File: src/middleware/quota.rs
Error: QuotaExceeded writing 50 bytes to 30-byte file with 100-byte limit.

# Review
Does this change maintain the PathResolver contract?
Are edge cases handled?
Are error messages informative?
```

**Deliverables:**

1. **AGENTS.md** - Instructions for LLMs contributing to the codebase
2. **LLM Development Methodology Guide** - Full methodology documentation
3. **llm-context.md** - Context7-style API guide for LLMs *using* the library

**Why This Design:**

| Benefit              | For LLMs               | For Humans           |
| -------------------- | ---------------------- | -------------------- |
| Isolated components  | Fits in context window | Easy to understand   |
| Clear contracts      | Reduces hallucination  | Self-documenting     |
| Consistent structure | Pattern matching works | Predictable codebase |
| Rich errors          | Can suggest fixes      | Quick debugging      |
| Focused tests        | Can verify changes     | Fast CI              |

**Trade-offs:**

| Approach          | Pros                        | Cons                         |
| ----------------- | --------------------------- | ---------------------------- |
| Deep abstraction  | Maximum isolation           | More files, more indirection |
| Monolithic design | Fewer files                 | LLMs can't reason about it   |
| LOA (chosen)      | LLM-friendly + maintainable | Requires discipline          |

**Relationship to Other ADRs:**

- **ADR-030 (Layered traits):** LOA extends this with per-file isolation
- **ADR-033 (PathResolver):** Example of LOA - resolver is isolated, testable, replaceable
- **ADR-025 (Strategic Boxing):** LOA prefers simplicity over micro-optimization

**Conclusion:** LLM-Oriented Architecture is not just about AI. It's about creating a codebase where **any component can be understood, tested, fixed, or replaced with only local context**. This benefits LLMs, open source contributors, code reviewers, and future maintainers equally. As AI-assisted development becomes standard, LOA positions AnyFS as a reference implementation for sustainable human-AI collaboration.

**See Also:** [LLM Development Methodology Guide](../guides/llm-development-methodology.md)