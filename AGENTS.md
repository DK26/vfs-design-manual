# AGENTS.md - Instructions for AI Assistants

READ THIS FIRST before making any changes to this repository.

Priority: reviews/comments.md is the highest-priority source of truth. If anything conflicts, follow reviews/comments.md and update this file.

---

## Project Overview

AnyFS is an open standard for pluggable virtual filesystem backends in Rust. It uses a **middleware/decorator pattern** (like Axum/Tower) for composable functionality.

**Key design principle:** Complete separation of concerns via composable layers.

---

## AnyFS vs AgentFS

AnyFS is a **filesystem abstraction**. [AgentFS](https://github.com/tursodatabase/agentfs) is an **agent runtime**.

| | AnyFS | AgentFS |
|-|-------|---------|
| Scope | Filesystem only | FS + KV store + tool auditing |
| Backends | Multiple (Memory, SQLite, VRootFs) | SQLite only |
| Middleware | Composable layers | Monolithic |

**They complement each other:**
- AnyFS could provide the filesystem layer for AgentFS
- AgentFS-compatible `VfsBackend` could be implemented

**Do NOT add to AnyFS:**
- Key-value store (different abstraction)
- Tool call auditing in core trait (use `Tracing` middleware)

---

## Architecture (Tower-style Middleware)

```
┌─────────────────────────────────────────┐
│  FilesContainer<B>                      │  ← Ergonomics only (std::fs API)
├─────────────────────────────────────────┤
│  Middleware (optional, composable):     │
│                                         │
│  Policy:        Quota, FeatureGuard,    │
│                 PathFilter, ReadOnly,   │
│                 RateLimit               │
│                                         │
│  Observability: Tracing, DryRun         │
│                                         │
│  Performance:   Cache                   │
│                                         │
│  Composition:   Overlay                 │
├─────────────────────────────────────────┤
│  VfsBackend                             │  ← Pure storage + fs semantics
│  (Memory, SQLite, VRootFs, custom...)   │
└─────────────────────────────────────────┘
```

Each layer has **exactly one responsibility**:

| Layer | Responsibility |
|-------|----------------|
| `VfsBackend` | Storage + filesystem semantics |
| `Quota<B>` | Resource limits (size, count, depth) |
| `FeatureGuard<B>` | Feature whitelist (symlinks, hard links) |
| `PathFilter<B>` | Path-based access control (sandbox) |
| `ReadOnly<B>` | Prevent all write operations |
| `RateLimit<B>` | Limit operations per second |
| `Tracing<B>` | Instrumentation / audit trail |
| `DryRun<B>` | Log operations without executing |
| `Cache<B>` | LRU cache for reads |
| `Overlay<B1,B2>` | Union filesystem (base + upper) |
| `FilesContainer<B>` | Ergonomic std::fs-aligned API |

---

## Crate Structure

```
anyfs-backend/              # Crate 1: trait + types
  Cargo.toml
  src/
    lib.rs
    backend.rs              # VfsBackend trait
    layer.rs                # Layer trait (Tower-style composition)
    ext.rs                  # VfsBackendExt (extension methods)
    types.rs                # Metadata, DirEntry, Permissions, StatFs
    error.rs                # VfsError (with context)

anyfs/                      # Crate 2: backends + middleware
  Cargo.toml
  src/
    lib.rs
    backends/
      memory.rs             # MemoryBackend [feature: memory, default]
      sqlite.rs             # SqliteBackend [feature: sqlite]
      vrootfs.rs            # VRootFsBackend [feature: vrootfs]
    middleware/
      quota.rs              # Quota<B> + QuotaLayer
      tracing.rs            # Tracing<B> + TracingLayer
      feature_guard.rs      # FeatureGuard<B> + FeatureGuardLayer
      path_filter.rs        # PathFilter<B> + PathFilterLayer
      read_only.rs          # ReadOnly<B> + ReadOnlyLayer
      rate_limit.rs         # RateLimit<B> + RateLimitLayer
      dry_run.rs            # DryRun<B> + DryRunLayer
      cache.rs              # Cache<B> + CacheLayer
      overlay.rs            # Overlay<B1, B2> + OverlayLayer

anyfs-container/            # Crate 3: ergonomic wrapper
  Cargo.toml
  src/
    lib.rs
    container.rs            # FilesContainer<B> (thin std::fs wrapper)
    stack.rs                # BackendStack (fluent builder)
```

---

## Dependency Graph

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware)
     |     ^-- vrootfs feature may use strict-path
     |
     +-- anyfs-container (ergonomic wrapper)
     |
     +-- third-party backends (only need anyfs-backend)
     |
     +-- third-party middleware (only need anyfs-backend)
```

**Third-party crates only need `anyfs-backend`** to implement:
- Custom backends (implement `VfsBackend`)
- Custom middleware (implement `VfsBackend` + `Layer`)

---

## VfsBackend Trait (in anyfs-backend)

Pure storage + filesystem semantics. No policy, no limits.

```rust
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

pub trait VfsBackend: Send {
    // ═══════════════════════════════════════════════════════════════════════
    // READ OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // WRITE OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // LINKS
    // ═══════════════════════════════════════════════════════════════════════
    fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // PERMISSIONS
    // ═══════════════════════════════════════════════════════════════════════
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // STREAMING I/O
    // ═══════════════════════════════════════════════════════════════════════
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // FILE SIZE
    // ═══════════════════════════════════════════════════════════════════════
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // DURABILITY
    // ═══════════════════════════════════════════════════════════════════════
    fn sync(&mut self) -> Result<(), VfsError>;
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // FILESYSTEM INFO
    // ═══════════════════════════════════════════════════════════════════════
    fn statfs(&self) -> Result<StatFs, VfsError>;
}
```

---

## Layer Trait (in anyfs-backend)

Standardizes middleware composition (inspired by Tower):

```rust
pub trait Layer<B: VfsBackend> {
    type Backend: VfsBackend;
    fn layer(self, backend: B) -> Self::Backend;
}
```

Each middleware provides a corresponding Layer:
- `QuotaLayer` → `Quota<B>`
- `TracingLayer` → `Tracing<B>`
- `FeatureGuardLayer` → `FeatureGuard<B>`
- `PathFilterLayer` → `PathFilter<B>`
- `ReadOnlyLayer` → `ReadOnly<B>`
- `RateLimitLayer` → `RateLimit<B>`
- `DryRunLayer` → `DryRun<B>`
- `CacheLayer` → `Cache<B>`
- `OverlayLayer` → `Overlay<B1, B2>`

---

## VfsBackendExt (in anyfs-backend)

Extension methods auto-implemented for all backends:

```rust
pub trait VfsBackendExt: VfsBackend {
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, VfsError>;
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), VfsError>;
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
}

impl<B: VfsBackend> VfsBackendExt for B {}
```

---

## Middleware (in anyfs)

Each middleware implements `VfsBackend` by wrapping another `VfsBackend`.

### Quota<B>

Enforces quota limits. Tracks usage and rejects operations that would exceed limits.

```rust
pub struct Quota<B: VfsBackend> {
    inner: B,
    limits: Limits,
    usage: Usage,
}

impl<B: VfsBackend> Quota<B> {
    pub fn new(inner: B) -> Self;
    pub fn with_max_total_size(self, bytes: u64) -> Self;
    pub fn with_max_file_size(self, bytes: u64) -> Self;
    pub fn with_max_node_count(self, count: u64) -> Self;
    pub fn with_max_dir_entries(self, count: u32) -> Self;
    pub fn with_max_path_depth(self, depth: u32) -> Self;

    pub fn usage(&self) -> &Usage;
    pub fn limits(&self) -> &Limits;
    pub fn remaining(&self) -> Remaining;
}

impl<B: VfsBackend> VfsBackend for Quota<B> {
    // Delegates to inner, checking limits before writes
}
```

### Tracing<B>

Integrates with the [tracing](https://docs.rs/tracing) ecosystem for structured logging.

```rust
pub struct Tracing<B: VfsBackend> {
    inner: B,
    target: &'static str,
    level: tracing::Level,
}

impl<B: VfsBackend> Tracing<B> {
    pub fn new(inner: B) -> Self;
    pub fn with_target(self, target: &'static str) -> Self;
    pub fn with_level(self, level: tracing::Level) -> Self;
}

impl<B: VfsBackend> VfsBackend for Tracing<B> {
    // Delegates to inner, emitting tracing spans/events
}
```

**Why tracing instead of custom logging?**
- Works with existing tracing subscribers
- Structured logging with spans
- Compatible with OpenTelemetry, Jaeger, etc.

### FeatureGuard<B>

Enforces least-privilege by disabling features by default.

```rust
pub struct FeatureGuard<B: VfsBackend> {
    inner: B,
    allow_symlinks: bool,
    allow_hard_links: bool,
    allow_permissions: bool,
    max_symlink_resolution: u32,
}

impl<B: VfsBackend> FeatureGuard<B> {
    pub fn new(inner: B) -> Self;  // All features disabled by default
    pub fn with_symlinks(self) -> Self;
    pub fn with_hard_links(self) -> Self;
    pub fn with_permissions(self) -> Self;
    pub fn with_max_symlink_resolution(self, max: u32) -> Self;
}

impl<B: VfsBackend> VfsBackend for FeatureGuard<B> {
    // Delegates to inner, rejecting disabled operations
}
```

### PathFilter<B>

Path-based access control using glob patterns. Essential for AI agent sandboxing.

```rust
pub struct PathFilter<B: VfsBackend> {
    inner: B,
    rules: Vec<PathRule>,
}

impl<B: VfsBackend> PathFilter<B> {
    pub fn new(inner: B) -> Self;
    pub fn allow(self, pattern: &str) -> Self;  // e.g., "/workspace/**"
    pub fn deny(self, pattern: &str) -> Self;   // e.g., "**/.env"
}

impl<B: VfsBackend> VfsBackend for PathFilter<B> {
    // Delegates to inner, checking path against rules first
    // Returns VfsError::AccessDenied if path is denied
}
```

### ReadOnly<B>

Prevents all write operations. Useful for safe browsing of container contents.

```rust
pub struct ReadOnly<B: VfsBackend> {
    inner: B,
}

impl<B: VfsBackend> ReadOnly<B> {
    pub fn new(inner: B) -> Self;
}

impl<B: VfsBackend> VfsBackend for ReadOnly<B> {
    // Read operations: delegate to inner
    // Write operations: return VfsError::ReadOnly
}
```

### RateLimit<B>

Limits operations per second. Protects against runaway processes.

```rust
pub struct RateLimit<B: VfsBackend> {
    inner: B,
    max_ops: u32,
    window: Duration,
    counter: AtomicU32,
}

impl<B: VfsBackend> RateLimit<B> {
    pub fn new(inner: B) -> Self;
    pub fn max_ops(self, ops: u32) -> Self;
    pub fn per_second(self) -> Self;
    pub fn per_minute(self) -> Self;
}

impl<B: VfsBackend> VfsBackend for RateLimit<B> {
    // Increments counter, returns VfsError::RateLimitExceeded if over limit
}
```

### DryRun<B>

Logs operations without executing writes. Perfect for testing and debugging.

```rust
pub struct DryRun<B: VfsBackend> {
    inner: B,
    log: Vec<Operation>,
}

impl<B: VfsBackend> DryRun<B> {
    pub fn new(inner: B) -> Self;
    pub fn operations(&self) -> &[Operation];
    pub fn clear(&mut self);
}

impl<B: VfsBackend> VfsBackend for DryRun<B> {
    // Read operations: delegate to inner
    // Write operations: log and return Ok(()) without executing
}
```

### Cache<B>

LRU cache for read operations. Improves performance for repeated reads.

```rust
pub struct Cache<B: VfsBackend> {
    inner: B,
    cache: LruCache<PathBuf, CachedEntry>,
}

impl<B: VfsBackend> Cache<B> {
    pub fn new(inner: B) -> Self;
    pub fn max_entries(self, count: usize) -> Self;
    pub fn max_entry_size(self, bytes: usize) -> Self;
    pub fn ttl(self, duration: Duration) -> Self;
}

impl<B: VfsBackend> VfsBackend for Cache<B> {
    // Read operations: check cache first, populate on miss
    // Write operations: invalidate cache entries, delegate to inner
}
```

### Overlay<B1, B2>

Union filesystem with copy-on-write semantics (like Docker layers).

```rust
pub struct Overlay<B1: VfsBackend, B2: VfsBackend> {
    base: B1,    // Read-only lower layer
    upper: B2,   // Writable upper layer
}

impl<B1: VfsBackend, B2: VfsBackend> Overlay<B1, B2> {
    pub fn new(base: B1, upper: B2) -> Self;
}

impl<B1: VfsBackend, B2: VfsBackend> VfsBackend for Overlay<B1, B2> {
    // Read: check upper first, fall back to base
    // Write: always to upper layer
    // Delete: create whiteout marker in upper
}
```

---

## Middleware Implementation Notes

### Streaming I/O

Middleware must handle `open_read`/`open_write` which return `Box<dyn Read/Write + Send>`. Options:

```rust
// Option 1: Wrap the stream (full interception)
fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError> {
    let inner = self.inner.open_write(&path)?;
    Ok(Box::new(CountingWriter::new(inner, self.usage.clone())))
}

// Option 2: Intercept at open, pass stream through (partial)
fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError> {
    self.check_path(&path)?;  // PathFilter checks here
    self.inner.open_write(path)
}

// Option 3: Discard stream content (DryRun)
fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError> {
    self.log.push(Operation::OpenWrite { path: path.as_ref().to_owned() });
    Ok(Box::new(std::io::sink()))
}
```

### Per-Middleware Streaming Behavior

| Middleware | `open_read` | `open_write` |
|------------|-------------|--------------|
| Quota | Wrap with `CountingReader` | Wrap with `CountingWriter` |
| FeatureGuard | Pass through | Pass through |
| PathFilter | Check path, then delegate | Check path, then delegate |
| ReadOnly | Pass through | Return `VfsError::ReadOnly` |
| RateLimit | Count open call | Count open call |
| Tracing | Log and delegate | Log and delegate |
| DryRun | Delegate to inner | Return `sink()` (discard) |
| Cache | Delegate (don't cache streams) | Delegate and invalidate |
| Overlay | Check upper, fall back to base | Delegate to upper |

### Quota Initialization

When wrapping an existing backend with data, `Quota` must scan to determine current usage:

```rust
impl<B: VfsBackend> Quota<B> {
    pub fn new(inner: B) -> Self {
        let usage = Self::scan_usage(&inner, "/");
        Self { inner, limits: Limits::default(), usage }
    }

    fn scan_usage(backend: &B, path: &str) -> Usage {
        // Recursively scan to count files and bytes
    }
}
```

### Cache Scope

`Cache` caches bulk operations only:
- **Cached:** `read()`, `read_to_string()`, `read_range()`, `metadata()`, `exists()`
- **Not cached:** `open_read()` (streams are for large files that shouldn't be cached)
- **Invalidates:** Any write operation to the path

### Overlay Whiteouts

Deleted files are marked with whiteout files in the upper layer:

```rust
// When deleting "/foo/bar.txt" that exists in base:
// Create "/foo/.wh.bar.txt" in upper layer

fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
    let path = path.as_ref();
    if self.upper.exists(path)? {
        self.upper.remove_file(path)
    } else if self.base.exists(path)? {
        let whiteout = whiteout_path(path);
        self.upper.write(&whiteout, b"")
    } else {
        Err(VfsError::NotFound { path: path.to_owned(), operation: "remove_file" })
    }
}
```

---

## FilesContainer (in anyfs-container)

**Thin ergonomic wrapper only.** No policy, no limits - just std::fs-aligned API.

```rust
use std::path::{Path, PathBuf};
use anyfs_backend::VfsBackend;

pub struct FilesContainer<B: VfsBackend> {
    backend: B,
}

impl<B: VfsBackend> FilesContainer<B> {
    pub fn new(backend: B) -> Self;

    // All methods delegate directly to backend
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    // ... etc (mirrors VfsBackend exactly)
}
```

---

## Usage Examples

### Simple (no middleware)

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

let mut fs = FilesContainer::new(MemoryBackend::new());
fs.write("/hello.txt", b"Hello!")?;
```

### With limits

```rust
use anyfs::{SqliteBackend, Quota};
use anyfs_container::FilesContainer;

let backend = Quota::new(SqliteBackend::open("data.db")?)
    .with_max_total_size(100 * 1024 * 1024)
    .with_max_file_size(10 * 1024 * 1024);

let mut fs = FilesContainer::new(backend);
```

### Full stack (limits + feature gates + tracing)

```rust
use anyfs::{SqliteBackend, Quota, FeatureGuard, Tracing};
use anyfs_container::FilesContainer;

let backend = SqliteBackend::open("data.db")?;
let limited = Quota::new(backend)
    .with_max_total_size(100 * 1024 * 1024);
let gated = FeatureGuard::new(limited)
    .with_symlinks();
let traced = Tracing::new(gated);

let mut fs = FilesContainer::new(traced);
```

### Layer-based composition

```rust
use anyfs::{SqliteBackend, QuotaLayer, FeatureGuardLayer, TracingLayer};

let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(FeatureGuardLayer::new().allow_symlinks())
    .layer(TracingLayer::new());
```

### BackendStack builder (fluent API)

```rust
use anyfs_container::BackendStack;

let fs = BackendStack::new(SqliteBackend::open("data.db")?)
    .limited(|l| l.max_total_size(100 * 1024 * 1024).max_file_size(10 * 1024 * 1024))
    .feature_gated(|g| g.allow_symlinks().allow_hard_links())
    .traced()
    .into_container();
```

### AI Agent Sandbox

```rust
use anyfs::{MemoryBackend, QuotaLayer, PathFilterLayer, FeatureGuardLayer, RateLimitLayer, TracingLayer};
use anyfs_backend::Layer;

let sandbox = MemoryBackend::new()
    .layer(QuotaLayer::new()
        .max_total_size(50 * 1024 * 1024)
        .max_file_size(5 * 1024 * 1024))
    .layer(PathFilterLayer::new()
        .allow("/workspace/**")
        .deny("**/.env")
        .deny("**/secrets/**"))
    .layer(FeatureGuardLayer::new())  // No symlinks/hard links
    .layer(RateLimitLayer::new()
        .max_ops(1000)
        .per_second())
    .layer(TracingLayer::new());  // Audit trail
```

### Overlay filesystem (Docker-like layers)

```rust
use anyfs::{SqliteBackend, MemoryBackend, Overlay};

let base = SqliteBackend::open("base-image.db")?;  // Read-only base
let upper = MemoryBackend::new();                   // Writable layer

let fs = Overlay::new(base, upper);
// Reads check upper first, writes go to upper only
```

---

## Built-in Backends

| Backend | Feature | Description |
|---------|---------|-------------|
| `MemoryBackend` | `memory` (default) | In-memory storage |
| `SqliteBackend` | `sqlite` | Single-file portable database |
| `VRootFsBackend` | `vrootfs` | Host filesystem (contained via strict-path) |

---

## Key Dependencies

| Crate | Used by | Purpose |
|-------|---------|---------|
| `thiserror` | `anyfs-backend` | Error types |
| `tracing` | `anyfs` | Instrumentation (Tracing) |
| `strict-path` | `anyfs` [vrootfs] | VirtualRoot containment |
| `rusqlite` | `anyfs` [sqlite] | SQLite database access |
| `bytes` | `anyfs` [bytes] | Zero-copy byte handling (optional) |
| `serde` | `anyfs-backend` | JSON support in VfsBackendExt |

---

## Optional Features

| Feature | Crate | Description |
|---------|-------|-------------|
| `memory` | `anyfs` | MemoryBackend (default) |
| `sqlite` | `anyfs` | SqliteBackend |
| `vrootfs` | `anyfs` | VRootFsBackend |
| `bytes` | `anyfs` | Use `Bytes` instead of `Vec<u8>` for zero-copy |

### FileContent Type Alias

The `bytes` feature uses a type alias for seamless switching:

```rust
#[cfg(feature = "bytes")]
pub type FileContent = bytes::Bytes;

#[cfg(not(feature = "bytes"))]
pub type FileContent = Vec<u8>;
```

Middleware passes `FileContent` through unchanged - no special handling needed.

---

## Common Mistakes to Avoid

- Do NOT put quota/limit logic in FilesContainer - use Quota
- Do NOT put feature gates in FilesContainer - use FeatureGuard
- Do NOT implement custom logging - use Tracing with tracing ecosystem
- Do NOT implement path filtering in backends - use PathFilter
- Do NOT implement read-only mode in backends - use ReadOnly
- FilesContainer is a thin wrapper only - no policy logic
- Middleware order matters: innermost applies first
- Use `VfsBackendExt` for convenience methods, don't add them to VfsBackend
- Use appropriate middleware for the job (see table below)

---

## Async Support

**Decision:** Sync-first, async-ready (ADR-010).

- `VfsBackend` is **synchronous** for v1
- All built-in backends are naturally sync (rusqlite, std::fs, memory)
- API is designed to allow `AsyncVfsBackend` trait later without breaking changes

**Async-ready principles:**
- Trait requires `Send` - works with async executors
- Return types are `Result<T, VfsError>` - compatible with async
- No hidden blocking state
- Users can wrap in `spawn_blocking` if needed

**Future:** When async is needed, add parallel `AsyncVfsBackend` trait with blanket impl for sync backends.

---

## When in Doubt

| Question | Answer |
|----------|--------|
| Where do limits go? | `Quota<B>` middleware |
| Where do feature gates go? | `FeatureGuard<B>` middleware |
| Where does logging go? | `Tracing<B>` middleware (uses tracing crate) |
| Where does path filtering go? | `PathFilter<B>` middleware |
| How to make read-only? | `ReadOnly<B>` middleware |
| How to rate limit? | `RateLimit<B>` middleware |
| How to test without side effects? | `DryRun<B>` middleware |
| How to cache reads? | `Cache<B>` middleware |
| How to layer filesystems? | `Overlay<B1, B2>` middleware |
| What does FilesContainer do? | Thin std::fs-aligned wrapper only |
| Path type everywhere? | `impl AsRef<Path>` (std::fs aligned) |
| Is strict-path used? | Only internally by VRootFsBackend |
| Sync or async? | Sync for v1, async-ready for future |
| How to compose middleware? | Use `Layer` trait or `BackendStack` builder |
| Where do convenience methods go? | `VfsBackendExt` (extension trait) |
| Zero-copy bytes? | Enable `bytes` feature for `Bytes` instead of `Vec<u8>` |
