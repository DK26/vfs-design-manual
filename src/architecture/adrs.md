# AnyFS - Architecture Decision Records

This file captures the decisions for the current AnyFS design.

---

## ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Path-based `Fs` trait | Accepted |
| ADR-002 | Two-crate structure | Accepted |
| ADR-003 | `impl AsRef<Path>` for all path parameters | Accepted |
| ADR-004 | Tower-style middleware pattern | Accepted |
| ADR-005 | `std::fs`-aligned method names | Accepted |
| ADR-006 | Quota for quota enforcement | Accepted |
| ADR-007 | Restrictions for least-privilege | Accepted |
| ADR-008 | FileStorage as thin ergonomic wrapper | Accepted |
| ADR-009 | Built-in backends are feature-gated | Accepted |
| ADR-010 | Sync-first, async-ready design | Accepted |
| ADR-011 | Layer trait for standardized composition | Accepted |
| ADR-012 | Tracing for instrumentation | Accepted |
| ADR-013 | FsExt for extension methods | Accepted |
| ADR-014 | Optional Bytes support | Accepted |
| ADR-015 | Contextual FsError | Accepted |
| ADR-016 | PathFilter for path-based access control | Accepted |
| ADR-017 | ReadOnly for preventing writes | Accepted |
| ADR-018 | RateLimit for operation throttling | Accepted |
| ADR-019 | DryRun for testing and debugging | Accepted |
| ADR-020 | Cache for read performance | Accepted |
| ADR-021 | Overlay for union filesystem | Accepted |
| ADR-022 | Builder pattern for configurable middleware | Accepted |
| ADR-023 | Interior mutability for all trait methods | Accepted |

---

## ADR-001: Path-based `Fs` trait

**Decision:** Backends implement a path-based trait aligned with `std::fs` method naming.

**Why:** Filesystem operations are naturally path-oriented; a single, familiar trait surface is easier to implement and adopt than graph-store or inode models.

---

## ADR-002: Two-crate structure

**Decision:**

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Minimal contract: `Fs` trait, `Layer` trait, `FsExt`, types |
| `anyfs` | Backends + middleware + ergonomics (`FileStorage<M>`, `BackendStack`) |

**Why:**
- Backend authors only need `anyfs-backend` (no heavy dependencies).
- Middleware is composable and lives with backends in `anyfs`.
- `FileStorage` is purely ergonomic - no policy logic - included in `anyfs` for convenience.

---

## ADR-003: `impl AsRef<Path>` for all path parameters

**Decision:** Both `Fs` traits and `FileStorage` accept `impl AsRef<Path>` for all path parameters.

**Why:**
- Aligned with `std::fs` API conventions.
- Works across all platforms (not limited to UTF-8).
- Ergonomic: accepts `&str`, `String`, `&Path`, `PathBuf`.

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
    .layer(RestrictionsLayer::builder()
        .deny_symlinks()
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

## ADR-007: Restrictions for opt-in restrictions

**Decision:** By default, all operations work. `Restrictions<B>` middleware provides opt-in restrictions.

**Default behavior (no Restrictions):**
- All operations work: `symlink()`, `hard_link()`, `set_permissions()`

**Opt-in restrictions:**
- `.deny_symlinks()` - block `symlink()` calls
- `.deny_hard_links()` - block `hard_link()` calls
- `.deny_permissions()` - block `set_permissions()` calls

When blocked, operations return `FsError::FeatureNotEnabled`.

**Symlink following:** Controlled separately via `set_follow_symlinks(bool)` on virtual backends. VRootFsBackend delegates to OS (strict-path prevents escapes).

**Why:**
- Simple default: everything works out of the box.
- Security is opt-in via middleware composition.
- Clear separation: Restrictions blocks operations, backend settings control behavior.

---

## ADR-008: FileStorage as thin ergonomic wrapper

**Decision:** `FileStorage<B, M>` is a thin wrapper that provides std::fs-aligned ergonomics only. It contains NO policy logic.

**What it does:**
- Provides familiar method names
- Accepts `impl AsRef<Path>` for convenience
- Delegates all operations to the wrapped backend

**What it does NOT do:**
- Quota enforcement (use Quota)
- Feature gating (use Restrictions)
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

**Decision:** `Fs` traits are synchronous for v1. The API is designed to allow adding `AsyncFs` later without breaking changes.

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
    async fn read(&self, path: impl AsRef<Path> + Send) -> Result<Vec<u8>, FsError>;
    async fn write(&mut self, path: impl AsRef<Path> + Send, data: &[u8]) -> Result<(), FsError>;
    // ... mirrors Fs with async

    // Streaming uses AsyncRead/AsyncWrite
    async fn open_read(&self, path: impl AsRef<Path> + Send)
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
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), FsError>;
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
    fn read(&self, path: impl AsRef<Path>) -> Result<FileContent, FsError>;
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
    operation: &'static str,  // "read", "metadata", etc.
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
- Operation field helps distinguish "file not found during read" vs "during metadata".
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
    .deny_symlinks()
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

| Middleware | Rationale |
|------------|-----------|
| `TracingLayer` | Default (global tracing subscriber) is meaningful |
| `ReadOnlyLayer` | No configuration needed |
| `DryRunLayer` | No configuration needed |
| `OverlayLayer` | Takes two backends as required params: `Overlay::new(lower, upper)` |

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
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError>;
}

pub trait FsWrite: Send {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
}
```

**New design:**
```rust
pub trait FsRead: Send + Sync {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError>;
}

pub trait FsWrite: Send + Sync {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
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
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
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

| Aspect | &mut self | &self (interior mutability) |
|--------|-----------|----------------------------|
| Compile-time safety | Single writer enforced | Runtime synchronization |
| Concurrent access | Not possible | Backend decides |
| API simplicity | Simple | Slightly more complex backends |
| Real-world match | Poor | Good |

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
       fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError>;
   }

   // Async (anyfs-async)
   #[async_trait]
   pub trait AsyncFsRead: Send + Sync {
       async fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError>;
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
       async fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
           let path = path.as_ref().to_path_buf();
           let backend = self.0.clone(); // requires Clone
           tokio::task::spawn_blocking(move || backend.read(&path)).await?
       }
   }
   ```

5. **No async-to-sync bridge:** We intentionally don't provide async-to-sync adapters (would require blocking on async runtime, which is problematic).

**Implementation phases:**

| Phase | Scope | Dependency |
|-------|-------|------------|
| 1 | Sync traits stable | Now |
| 2 | Design async traits | When needed |
| 3 | `anyfs-async` crate | When needed |
| 4 | Async middleware | When needed |

**Why parallel traits (not feature flags):**

- **No conditional compilation complexity** - sync and async are separate, clean codebases
- **No trait object issues** - async traits have different object safety requirements
- **Clear dependency boundaries** - sync code doesn't pull in tokio/async-std
- **Ecosystem alignment** - mirrors how `std::io` vs `tokio::io` work

**Trade-offs:**

| Approach | Pros | Cons |
|----------|------|------|
| Parallel traits | Clean separation, no async deps in core | Code duplication in middleware |
| Feature flags | Single codebase | Complex conditional compilation |
| Async-only | Modern, no duplication | Forces async runtime on sync users |
| Sync-only | Simple | Can't support network backends efficiently |

**Conclusion:** Parallel async traits provide the best balance of simplicity now (sync-only core) with a clear migration path for async support later. The `Layer` trait design already accommodates this pattern.
