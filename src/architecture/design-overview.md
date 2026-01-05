# AnyFS - Design Overview

**Status:** Current
**Last updated:** 2025-12-24

---

## What This Project Is

AnyFS is an **open standard** for pluggable virtual filesystem backends in Rust. It uses a **middleware/decorator pattern** (like Axum/Tower) for composable functionality with complete separation of concerns.

### Philosophy: Focused App, Smart Storage

It decouples application logic from storage policy, enabling a **Data Mesh** at the filesystem level.

- **The App** focuses on business value ("save the document").
- **The Storage Layer** enforces non-functional requirements ("encrypt, audit, limit, index").

Anyone can:
- **Control how a drive acts, looks, and protects itself.**
- Implement a custom backend for their specific storage needs (Cloud, DB, RAM).
- Compose middleware to add limits, logging, and security.
- Use the ergonomic `FileStorage<B, M>` wrapper for a standard `std::fs`-like API.

---

## Architecture (Tower-style Middleware)

```
┌─────────────────────────────────────────┐
│  FileStorage<B, M>                      │  ← Ergonomics + type-safe marker
├─────────────────────────────────────────┤
│  Middleware (optional, composable):     │
│                                         │
│  Policy:                                │
│    Quota<B>         - Resource limits   │
│    Restrictions<B>  - Least privilege   │
│    PathFilter<B>    - Sandbox paths     │
│    ReadOnly<B>      - Prevent writes    │
│    RateLimit<B>     - Ops/sec limit     │
│                                         │
│  Observability:                         │
│    Tracing<B>       - Instrumentation   │
│    DryRun<B>        - Test mode         │
│                                         │
│  Performance:                           │
│    Cache<B>         - LRU caching       │
│                                         │
│  Composition:                           │
│    Overlay<B1,B2>   - Layered FS        │
│                                         │
├─────────────────────────────────────────┤
│  Backend (implements Fs, FsFull,        │  ← Pure storage + fs semantics
│           FsFuse, or FsPosix)           │
│  (Memory, SQLite, VRootFs, custom...)   │
└─────────────────────────────────────────┘
```

**Each layer has exactly one responsibility:**

| Layer             | Responsibility                       |
| ----------------- | ------------------------------------ |
| Backend (`Fs`+)   | Storage + filesystem semantics       |
| `Quota<B>`        | Resource limits (size, count, depth) |
| `Restrictions<B>` | Opt-in operation restrictions        |
| `PathFilter<B>`   | Path-based access control            |
| `ReadOnly<B>`     | Prevent all write operations         |
| `RateLimit<B>`    | Limit operations per second          |
| `Tracing<B>`      | Instrumentation / audit trail        |

---

## Design Principle: Predictable Defaults, Opt-in Security

**The `Fs` traits mimic `std::fs` with predictable, permissive defaults.**

See ADR-027 for the decision rationale.

The traits are low-level interfaces that any backend can implement - memory, SQLite, real filesystem, network storage, etc. To maintain consistent behavior across all backends:

- All operations work by default (`symlink()`, `hard_link()`, `set_permissions()`)
- No security restrictions at the trait level
- Behavior matches what you'd expect from a real filesystem

**Why not secure-by-default at this layer?**

1. **Predictability**: A backend should behave like a filesystem. Surprising restrictions break expectations.
2. **Backend-agnostic**: The traits don't know if they're wrapping a sandboxed memory store or a real filesystem. Restrictions that make sense for one may not for another.
3. **Composition**: Security is achieved by layering middleware, not by baking it into the storage layer.

**Security is the responsibility of higher-level APIs:**

| Layer                                           | Security Responsibility          |
| ----------------------------------------------- | -------------------------------- |
| Backend (`Fs`+)                                 | None - pure filesystem semantics |
| Middleware (`Restrictions`, `PathFilter`, etc.) | Opt-in restrictions              |
| `FileStorage` or application code               | Configure appropriate middleware |

**Example: Secure AI Agent Sandbox**

```rust
use anyfs::{MemoryBackend, QuotaLayer, PathFilterLayer, FileStorage};

struct AiSandbox;  // Marker type

// Application composes secure defaults (marker in type annotation)
let sandbox: FileStorage<_, AiSandbox> = FileStorage::new(
    MemoryBackend::new()
        .layer(QuotaLayer::builder()
            .max_total_size(50 * 1024 * 1024)
            .build())
        .layer(PathFilterLayer::builder()
            .allow("/workspace/**")
            .deny("**/.env")
            .build())
);
```

The backend is permissive. The application adds restrictions appropriate for its use case.

---

## Crates

| Crate           | Purpose                            | Contains                                                                              |
| --------------- | ---------------------------------- | ------------------------------------------------------------------------------------- |
| `anyfs-backend` | Minimal contract                   | Layered traits (`Fs`, `FsFull`, `FsFuse`, `FsPosix`), `Layer` trait, types, `FsExt`   |
| `anyfs`         | Backends + middleware + ergonomics | Built-in backends, all middleware layers, `FileStorage<B, M>`, `BackendStack` builder |

### Dependency Graph

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware + ergonomics)
           ^-- vrootfs feature may use strict-path
```

---

## Future Considerations

These are optional extensions to explore after the core is stable.

**Keep (add-ons that fit the current design):**
- URL-based backend registry (`sqlite://`, `mem://`, `stdfs://`) as a helper crate, not in core APIs.
- Bulk operation helpers (`read_many`, `write_many`, `copy_many`, `glob`, `walk`) as `FsExt` or a utilities crate.
- Early async adapter crate (`anyfs-async`) to support remote backends without changing sync traits.
- Bash-style shell (example app or `anyfs-shell` crate) that routes `ls/cd/cat/cp/mv/rm/mkdir/stat` through `FileStorage` to demonstrate middleware and backend neutrality (navigation and file management only, not full bash scripting).
- Copy-on-write overlay middleware (Afero-style `CopyOnWriteFs`) as a specialized `Overlay` variant.
- Archive backends (zip/tar) as separate crates implementing `Fs` (inspired by PyFilesystem/fsspec).
- Indexing middleware (`Indexing<B>` + `IndexLayer`) with pluggable index engines (SQLite default). See [Indexing Middleware](./indexed-realfs-backend.md).

**Defer (valuable, but needs data or wider review):**
- Range/block caching middleware for `read_range` heavy workloads (fsspec-style block cache).
- Runtime capability discovery (`Capabilities` struct) for feature detection (symlink control, case sensitivity, max path length).
- Lint/analyzer to discourage direct `std::fs` usage in app code (System.IO.Abstractions-style).
- Retry/timeout middleware for remote backends (when network backends are real).

**Drop for now (adds noise or cross-platform complexity):**
- Change notification support (optional `FsWatch` trait or polling middleware).

Detailed rationale lives in `src/comparisons/prior-art-analysis.md`.

---

### Language Bindings (Python, C, etc.)

The AnyFS design is **FFI-friendly** and can be exposed to other languages with minimal friction.

**Why the design works well for FFI:**

| Design Choice              | FFI Benefit                                                                    |
| -------------------------- | ------------------------------------------------------------------------------ |
| `&self` methods (ADR-023)  | Interior mutability allows holding a single `Arc<FileStorage<...>>` across FFI |
| `Box<dyn Fs>` type erasure | `FileStorage::boxed()` provides a concrete type suitable for FFI               |
| Owned return types         | `Vec<u8>`, `String`, `bool` - no lifetime issues across FFI boundary           |
| Simple structs             | `Metadata`, `DirEntry`, `Permissions` map directly to Python/C structs         |

**Recommended approach for Python (PyO3):**

```rust
// anyfs-python/src/lib.rs
use pyo3::prelude::*;
use anyfs::{FileStorage, MemoryBackend, SqliteBackend, Fs};

#[pyclass]
struct PyFileStorage {
    inner: FileStorage<Box<dyn Fs>>,  // Type-erased for FFI
}

#[pymethods]
impl PyFileStorage {
    #[staticmethod]
    fn memory() -> Self {
        Self { inner: FileStorage::new(MemoryBackend::new()).boxed() }
    }

    #[staticmethod]
    fn sqlite(path: &str) -> PyResult<Self> {
        let backend = SqliteBackend::open(path)
            .map_err(|e| PyErr::new::<pyo3::exceptions::PyIOError, _>(e.to_string()))?;
        Ok(Self { inner: FileStorage::new(backend).boxed() })
    }

    fn read(&self, path: &str) -> PyResult<Vec<u8>> {
        self.inner.read(path)
            .map_err(|e| PyErr::new::<pyo3::exceptions::PyIOError, _>(e.to_string()))
    }

    fn write(&self, path: &str, data: &[u8]) -> PyResult<()> {
        self.inner.write(path, data)
            .map_err(|e| PyErr::new::<pyo3::exceptions::PyIOError, _>(e.to_string()))
    }
}

#[pymodule]
fn anyfs_python(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<PyFileStorage>()?;
    Ok(())
}
```

**Python usage:**

```python
from anyfs_python import PyFileStorage

fs = PyFileStorage.memory()
fs.write("/hello.txt", b"Hello from Python!")
data = fs.read("/hello.txt")
print(data)  # b"Hello from Python!"
```

**Key considerations for FFI:**

| Concern                        | Solution                                                  |
| ------------------------------ | --------------------------------------------------------- |
| Generics (`FileStorage<B, M>`) | Use `FileStorage<Box<dyn Fs>>` (boxed form) for FFI layer |
| Streaming (`Box<dyn Read>`)    | Wrap in language-native class with `read(n)` method       |
| Middleware composition         | Pre-build common stacks, expose as factory functions      |
| Error handling                 | Convert `FsError` to language-native exceptions           |

**Future crate:** `anyfs-python`

### Dynamic Middleware

The current design uses **compile-time generics** for zero-cost middleware composition:

```rust
// Static: type known at compile time
let fs: Tracing<Quota<MemoryBackend>> = MemoryBackend::new()
    .layer(QuotaLayer::builder().max_total_size(100).build())
    .layer(TracingLayer::new());
```

For **runtime-configured** middleware (e.g., based on config files), use `Box<dyn Fs>`:

```rust
fn build_from_config(config: &Config) -> FileStorage<Box<dyn Fs>> {
    let mut backend: Box<dyn Fs> = Box::new(MemoryBackend::new());

    if config.enable_quota {
        backend = Box::new(Quota::new(backend, config.quota_limit));
    }

    if config.enable_antivirus {
        backend = Box::new(AntivirusMiddleware::new(backend, config.av_scanner_path));
    }

    if config.enable_tracing {
        backend = Box::new(Tracing::new(backend));
    }

    FileStorage::new(backend)
}
```

**Trade-off:** One `Box` allocation per layer + vtable dispatch. For I/O-bound workloads, this overhead is negligible (<1% of operation time).

**Example: Antivirus Middleware**

```rust
pub struct Antivirus<B> {
    inner: B,
    scanner: Arc<dyn VirusScanner + Send + Sync>,
}

pub trait VirusScanner: Send + Sync {
    fn scan(&self, data: &[u8]) -> Option<String>;  // Returns threat name if detected
}

impl<B: FsWrite> FsWrite for Antivirus<B> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        if let Some(threat) = self.scanner.scan(data) {
            return Err(FsError::ThreatDetected { 
                path: path.to_path_buf(), 
                threat_name: threat,
            });
        }
        self.inner.write(path, data)
    }

    fn open_write(&self, path: &Path) -> Result<Box<dyn Write + Send>, FsError> {
        let inner = self.inner.open_write(path)?;
        Ok(Box::new(ScanningWriter::new(inner, self.scanner.clone())))
    }
}
```

**Future: Plugin System**

For true runtime-loaded plugins (`.so`/`.dll`), a future `MiddlewarePlugin` trait could enable:

```rust
pub trait MiddlewarePlugin: Send + Sync {
    fn name(&self) -> &str;
    fn wrap(&self, backend: Box<dyn Fs>) -> Box<dyn Fs>;
}

// Load at runtime
let plugin = libloading::Library::new("antivirus_plugin.so")?;
let create_plugin: fn() -> Box<dyn MiddlewarePlugin> = plugin.get(b"create_plugin")?;
let av_plugin = create_plugin();

let backend = av_plugin.wrap(backend);
```

**When to use each approach:**

| Scenario                 | Approach                 | Overhead            |
| ------------------------ | ------------------------ | ------------------- |
| Fixed middleware stack   | Generics (compile-time)  | Zero-cost           |
| Config-driven middleware | `Box<dyn Fs>` chaining   | ~50ns per layer     |
| Runtime-loaded plugins   | `MiddlewarePlugin` trait | ~50ns + plugin load |

**Verdict:** The current design supports dynamic middleware via `Box<dyn Fs>`. A formal `MiddlewarePlugin` trait for hot-loading is a future enhancement.

### Middleware with Configurable Backends

Some middleware benefit from pluggable backends for their own storage or output. The pattern is to inject a trait object or configuration at construction time.

**Metrics Middleware with Prometheus Exporter:**
*(Requires `features = ["metrics"]`)*

```rust
use prometheus::{Counter, Histogram, Registry};

pub struct Metrics<B> {
    inner: B,
    reads: Counter,
    writes: Counter,
    read_bytes: Counter,
    write_bytes: Counter,
    latency: Histogram,
}

impl<B> Metrics<B> {
    pub fn new(inner: B, registry: &Registry) -> Self {
        let reads = Counter::new("anyfs_reads_total", "Total read operations").unwrap();
        let writes = Counter::new("anyfs_writes_total", "Total write operations").unwrap();
        registry.register(Box::new(reads.clone())).unwrap();
        registry.register(Box::new(writes.clone())).unwrap();
        // ... register all metrics
        Self { inner, reads, writes, /* ... */ }
    }
}

impl<B: FsRead> FsRead for Metrics<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        self.reads.inc();
        let start = Instant::now();
        let result = self.inner.read(path);
        self.latency.observe(start.elapsed().as_secs_f64());
        if let Ok(ref data) = result {
            self.read_bytes.inc_by(data.len() as u64);
        }
        result
    }
}

// Expose via HTTP endpoint
async fn metrics_handler(registry: web::Data<Registry>) -> impl Responder {
    let encoder = TextEncoder::new();
    let metrics = registry.gather();
    encoder.encode_to_string(&metrics).unwrap()
}
```

**Indexing Middleware with Remote Database:**

```rust
pub trait IndexBackend: Send + Sync {
    fn record_write(&self, path: &Path, size: u64, hash: &str) -> Result<(), IndexError>;
    fn record_delete(&self, path: &Path) -> Result<(), IndexError>;
    fn query(&self, pattern: &str) -> Result<Vec<IndexEntry>, IndexError>;
}

// SQLite implementation
pub struct SqliteIndex { conn: Connection }

// PostgreSQL implementation  
pub struct PostgresIndex { pool: PgPool }

// MariaDB implementation
pub struct MariaDbIndex { pool: MySqlPool }

pub struct Indexing<B, I: IndexBackend> {
    inner: B,
    index: I,
}

impl<B: FsWrite, I: IndexBackend> FsWrite for Indexing<B, I> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        self.inner.write(path, data)?;
        let hash = sha256(data);
        self.index.record_write(path, data.len() as u64, &hash)
            .map_err(|e| FsError::Backend(e.to_string()))?;
        Ok(())
    }
}

// Usage with PostgreSQL
let index = PostgresIndex::connect("postgres://user:pass@db.example.com/files").await?;
let backend = MemoryBackend::new()
    .layer(IndexingLayer::new(index));
```

**Configurable Tracing with Multiple Sinks:**

```rust
pub trait TraceSink: Send + Sync {
    fn log_operation(&self, op: &Operation);
}

// Structured JSON logs
pub struct JsonSink { writer: Box<dyn Write + Send> }

// CEF (Common Event Format) for SIEM integration
pub struct CefSink { 
    host: String,
    port: u16,
    device_vendor: String 
}

impl TraceSink for CefSink {
    fn log_operation(&self, op: &Operation) {
        let cef = format!(
            "CEF:0|AnyFS|FileStorage|1.0|{}|{}|{}|src={} dst={}",
            op.event_id, op.name, op.severity, op.source_path, op.dest_path
        );
        self.send_syslog(&cef);
    }
}

// Remote sink (e.g., Loki, Elasticsearch)
pub struct RemoteSink { endpoint: String, client: reqwest::Client }

pub struct Tracing<B, S: TraceSink> {
    inner: B,
    sink: S,
}
```

---

## Performance: Strategic Boxing (ADR-025)

AnyFS follows Tower/Axum's approach to dynamic dispatch: **zero-cost on the hot path, box at boundaries where flexibility is needed**. We avoid heap allocations and dynamic dispatch unless they add flexibility without meaningful performance impact.

| Path                     | Operations                                    | Cost                          |
| ------------------------ | --------------------------------------------- | ----------------------------- |
| **Hot path** (zero-cost) | `read()`, `write()`, `metadata()`, `exists()` | Concrete types, no boxing     |
| **Hot path** (zero-cost) | Middleware composition: `Quota<Tracing<B>>`   | Generics, monomorphized       |
| **Cold path** (boxed)    | `open_read()`, `open_write()`, `read_dir()`   | One `Box` allocation per call |
| **Opt-in**               | `FileStorage::boxed()`                        | Explicit type erasure         |

**Hot-loop guidance:** If you open many small files and care about micro-overhead (especially on virtual backends), prefer `read()`/`write()` or the typed streaming extension (`FsReadTyped`/`FsWriteTyped`) when the backend type is known. These are the zero-allocation fast paths.

**Why box streams and iterators?**
1. Middleware needs to wrap them (`QuotaWriter` counts bytes, `PathFilter` filters entries)
2. Box allocation (~50ns) is <1% of actual I/O time
3. Avoids type explosion: `QuotaReader<PathFilterReader<TracingReader<Cursor<...>>>>`

**Why NOT box bulk operations?**
1. `read()` and `write()` are the most common operations
2. They return concrete types (`Vec<u8>`, `()`)
3. Zero overhead for the typical use case

See [ADR-025](./adrs.md#adr-025-strategic-boxing-tower-style) and [Zero-Cost Alternatives](./zero-cost-alternatives.md) for full analysis.

---

## Trait Architecture (in `anyfs-backend`)

AnyFS uses **layered traits** for maximum flexibility with minimal complexity.

See ADR-030 for the rationale behind the layered hierarchy.

```
                        FsPosix
                           │
            ┌──────────────┼──────────────┐
            │              │              │
       FsHandles      FsLock       FsXattr
            │              │              │
            └──────────────┼──────────────┘
                           │
                        FsFuse
                           │
                       FsInode
                           │
                        FsFull
                           │
            ┌──────┬───────┼───────┬──────┐
            │      │       │       │      │
       FsLink  FsPerm  FsSync FsStats │
            │      │       │       │      │
            └──────┴───────┼───────┴──────┘
                           │
                           Fs  ← Most users only need this
                           │
               ┌───────────┼───────────┐
               │           │           │
            FsRead    FsWrite     FsDir
```

**Simple rule:** Import `Fs` for basic use. Add traits as needed for advanced features.

---

## Core Traits (Layer 1)

### FsRead - Read Operations

```rust
pub trait FsRead: Send + Sync {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError>;
    fn read_to_string(&self, path: &Path) -> Result<String, FsError>;
    fn read_range(&self, path: &Path, offset: u64, len: usize) -> Result<Vec<u8>, FsError>;
    fn exists(&self, path: &Path) -> Result<bool, FsError>;
    fn metadata(&self, path: &Path) -> Result<Metadata, FsError>;
    fn open_read(&self, path: &Path) -> Result<Box<dyn Read + Send>, FsError>;
}
```

### FsWrite - Write Operations

```rust
pub trait FsWrite: Send + Sync {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError>;
    fn append(&self, path: &Path, data: &[u8]) -> Result<(), FsError>;
    fn remove_file(&self, path: &Path) -> Result<(), FsError>;
    fn rename(&self, from: &Path, to: &Path) -> Result<(), FsError>;
    fn copy(&self, from: &Path, to: &Path) -> Result<(), FsError>;
    fn truncate(&self, path: &Path, size: u64) -> Result<(), FsError>;
    fn open_write(&self, path: &Path) -> Result<Box<dyn Write + Send>, FsError>;
}
```

> **Note:** All methods use `&self` (interior mutability). Backends manage their own synchronization. See ADR-023.

### FsDir - Directory Operations

```rust
pub trait FsDir: Send + Sync {
    fn read_dir(&self, path: &Path) -> Result<ReadDirIter, FsError>;
    fn create_dir(&self, path: &Path) -> Result<(), FsError>;
    fn create_dir_all(&self, path: &Path) -> Result<(), FsError>;
    fn remove_dir(&self, path: &Path) -> Result<(), FsError>;
    fn remove_dir_all(&self, path: &Path) -> Result<(), FsError>;
}
```

---

## Extended Traits (Layer 2 - Optional)

```rust
pub trait FsLink: Send + Sync {
    fn symlink(&self, original: &Path, link: &Path) -> Result<(), FsError>;
    fn hard_link(&self, original: &Path, link: &Path) -> Result<(), FsError>;
    fn read_link(&self, path: &Path) -> Result<PathBuf, FsError>;
    fn symlink_metadata(&self, path: &Path) -> Result<Metadata, FsError>;
}

pub trait FsPermissions: Send + Sync {
    fn set_permissions(&self, path: &Path, perm: Permissions) -> Result<(), FsError>;
}

pub trait FsSync: Send + Sync {
    fn sync(&self) -> Result<(), FsError>;
    fn fsync(&self, path: &Path) -> Result<(), FsError>;
}

pub trait FsStats: Send + Sync {
    fn statfs(&self) -> Result<StatFs, FsError>;
}
```

---

## Inode Traits (Layer 3 - For FUSE)

```rust
pub trait FsInode: Send + Sync {
    fn path_to_inode(&self, path: &Path) -> Result<u64, FsError>;
    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, FsError>;
    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, FsError>;
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, FsError>;
}
```

---

## POSIX Traits (Layer 4 - Full POSIX)

```rust
pub trait FsHandles: Send + Sync {
    fn open(&self, path: &Path, flags: OpenFlags) -> Result<Handle, FsError>;
    fn read_at(&self, handle: Handle, buf: &mut [u8], offset: u64) -> Result<usize, FsError>;
    fn write_at(&self, handle: Handle, data: &[u8], offset: u64) -> Result<usize, FsError>;
    fn close(&self, handle: Handle) -> Result<(), FsError>;
}

pub trait FsLock: Send + Sync {
    fn lock(&self, handle: Handle, lock: LockType) -> Result<(), FsError>;
    fn try_lock(&self, handle: Handle, lock: LockType) -> Result<bool, FsError>;
    fn unlock(&self, handle: Handle) -> Result<(), FsError>;
}

pub trait FsXattr: Send + Sync {
    fn get_xattr(&self, path: &Path, name: &str) -> Result<Vec<u8>, FsError>;
    fn set_xattr(&self, path: &Path, name: &str, value: &[u8]) -> Result<(), FsError>;
    fn remove_xattr(&self, path: &Path, name: &str) -> Result<(), FsError>;
    fn list_xattr(&self, path: &Path) -> Result<Vec<String>, FsError>;
}
```

---

## Convenience Supertraits (Simple API)

```rust
/// Basic filesystem - covers 90% of use cases
pub trait Fs: FsRead + FsWrite + FsDir {}
impl<T: FsRead + FsWrite + FsDir> Fs for T {}

/// Full filesystem with all std::fs features
pub trait FsFull: Fs + FsLink + FsPermissions + FsSync + FsStats {}
impl<T: Fs + FsLink + FsPermissions + FsSync + FsStats> FsFull for T {}

/// FUSE-mountable filesystem
pub trait FsFuse: FsFull + FsInode {}
impl<T: FsFull + FsInode> FsFuse for T {}

/// Full POSIX filesystem
pub trait FsPosix: FsFuse + FsHandles + FsLock + FsXattr {}
impl<T: FsFuse + FsHandles + FsLock + FsXattr> FsPosix for T {}
```

---

## Usage Examples

Application code should use `FileStorage` for the std::fs-style DX (string paths). Core trait examples are shown separately for implementers and generic code.

### Most Users: FileStorage

```rust
use anyfs::{FileStorage, MemoryBackend};

fn process_files() -> Result<(), Box<dyn std::error::Error>> {
    let fs = FileStorage::new(MemoryBackend::new());
    let data = fs.read("/input.txt")?;
    fs.write("/output.txt", &processed(data))?;
    Ok(())
}
```

### Generic Code over Core Traits

```rust
use anyfs::{FileStorage, Fs};

fn process_files<B: Fs>(fs: &FileStorage<B>) {
    let data = fs.read("/input.txt")?;
    fs.write("/output.txt", &processed(data))?;
}
```

### Need Links? Add the Trait

```rust
use anyfs::{FileStorage, Fs, FsLink};

fn with_symlinks<B: Fs + FsLink>(fs: &FileStorage<B>) {
    fs.write("/target.txt", b"content")?;
    fs.symlink("/target.txt", "/link.txt")?;
}
```

### FUSE Mount

Mounting is part of `anyfs` crate with `fuse` and `winfsp` feature flags; see `src/guides/mounting.md`.

```rust
use anyfs::{FsFuse, MountHandle};

fn mount_filesystem(fs: impl FsFuse) {
    MountHandle::mount(fs, "/mnt/myfs")?;
}
```

### Full POSIX Application

```rust
use anyfs::{FileStorage, FsPosix};

fn database_app<B: FsPosix>(fs: &FileStorage<B>) {
    let handle = fs.open("/data.db", OpenFlags::READ_WRITE)?;
    fs.lock(handle, LockType::Exclusive)?;
    fs.write_at(handle, data, offset)?;
    fs.unlock(handle)?;
    fs.close(handle)?;
}

---

## Core Types (in `anyfs-backend`)

### Constants

```rust
/// Root directory inode. FUSE convention.
pub const ROOT_INODE: u64 = 1;
```

### Metadata

```rust
/// File or directory metadata.
#[derive(Debug, Clone)]
pub struct Metadata {
    /// Type: File, Directory, or Symlink.
    pub file_type: FileType,

    /// Size in bytes (0 for directories).
    pub size: u64,

    /// Permission mode bits. Default to 0o755/0o644 if unsupported.
    pub permissions: Permissions,

    /// Creation time (UNIX_EPOCH if unsupported).
    pub created: SystemTime,

    /// Last modification time.
    pub modified: SystemTime,

    /// Last access time.
    pub accessed: SystemTime,

    /// Inode number (0 if unsupported).
    pub inode: u64,

    /// Number of hard links (1 if unsupported).
    pub nlink: u64,
}

impl Metadata {
    /// Check if this is a file.
    pub fn is_file(&self) -> bool { self.file_type == FileType::File }

    /// Check if this is a directory.
    pub fn is_dir(&self) -> bool { self.file_type == FileType::Directory }

    /// Check if this is a symlink.
    pub fn is_symlink(&self) -> bool { self.file_type == FileType::Symlink }
}
```

### FileType

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum FileType {
    File,
    Directory,
    Symlink,
}
```

### DirEntry

```rust
/// Entry in a directory listing.
#[derive(Debug, Clone)]
pub struct DirEntry {
    /// File or directory name (not full path).
    pub name: String,

    /// Full path to the entry.
    pub path: PathBuf,

    /// Type: File, Directory, or Symlink.
    pub file_type: FileType,

    /// Size in bytes (0 for directories, can be lazy).
    pub size: u64,

    /// Inode number (0 if unsupported).
    pub inode: u64,
}
```

### Permissions

```rust
/// Unix-style permission bits.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Permissions(u32);

impl Permissions {
    /// Create permissions from a mode (e.g., 0o755).
    pub fn from_mode(mode: u32) -> Self { Permissions(mode) }

    /// Get the mode bits.
    pub fn mode(&self) -> u32 { self.0 }

    /// Read-only permissions (0o444).
    pub fn readonly() -> Self { Permissions(0o444) }

    /// Default file permissions (0o644).
    pub fn default_file() -> Self { Permissions(0o644) }

    /// Default directory permissions (0o755).
    pub fn default_dir() -> Self { Permissions(0o755) }
}
```

### StatFs

```rust
/// Filesystem statistics.
#[derive(Debug, Clone)]
pub struct StatFs {
    /// Total size in bytes (0 = unlimited).
    pub total_bytes: u64,

    /// Used bytes.
    pub used_bytes: u64,

    /// Available bytes.
    pub available_bytes: u64,

    /// Total number of inodes (0 = unlimited).
    pub total_inodes: u64,

    /// Used inodes.
    pub used_inodes: u64,

    /// Available inodes.
    pub available_inodes: u64,

    /// Filesystem block size.
    pub block_size: u64,

    /// Maximum filename length.
    pub max_name_len: u64,
}
```

---

## Middleware (in `anyfs`)

Each middleware implements the same traits as its inner backend. This enables composition while preserving capabilities.

### Quota<B>

Enforces quota limits. Tracks usage and rejects operations that would exceed limits.

```rust
use anyfs::{SqliteBackend, Quota};

let backend = QuotaLayer::builder()
    .max_total_size(100 * 1024 * 1024)   // 100 MB
    .max_file_size(10 * 1024 * 1024)     // 10 MB per file
    .max_node_count(10_000)              // 10K files/dirs
    .max_dir_entries(1_000)              // 1K entries per dir
    .max_path_depth(64)
    .build()
    .layer(SqliteBackend::open("data.db")?);

// Check usage
let usage = backend.usage();
let remaining = backend.remaining();
```

### Restrictions<B>

Blocks permission-related operations when needed.

```rust
use anyfs::{MemoryBackend, Restrictions};

// Symlink/hard-link capability is determined by trait bounds (FsLink).
// Restrictions only controls permission changes.
let backend = RestrictionsLayer::builder()
    .deny_permissions()    // Block set_permissions() calls
    .build()
    .layer(MemoryBackend::new());
```

When blocked, operations return `FsError::FeatureNotEnabled`.

### Tracing<B>

Integrates with the [tracing](https://docs.rs/tracing) ecosystem for structured logging and instrumentation.

```rust
use anyfs::{SqliteBackend, TracingLayer};

let backend = SqliteBackend::open("data.db")?
    .layer(TracingLayer::new()
        .with_target("anyfs")
        .with_level(tracing::Level::DEBUG));

// Users configure tracing subscribers as they prefer
tracing_subscriber::fmt::init();
```

**Why tracing instead of custom logging?**
- Works with existing tracing infrastructure
- Structured logging with spans
- Compatible with OpenTelemetry, Jaeger, etc.
- Users choose their subscriber (console, file, distributed tracing)

### PathFilter<B>

Restricts access to specific paths. Essential for sandboxing.

```rust
use anyfs::{MemoryBackend, PathFilter};

let backend = PathFilterLayer::builder()
    .allow("/workspace/**")           // Allow all under /workspace
    .allow("/tmp/**")                  // Allow temp files
    .deny("/workspace/.env")           // But deny .env files
    .deny("**/.git/**")               // Deny all .git directories
    .build()
    .layer(MemoryBackend::new());
```

When a path is denied, operations return `FsError::AccessDenied`.

### ReadOnly<B>

Prevents all write operations. Useful for publishing immutable data.

```rust
use anyfs::{SqliteBackend, ReadOnly, FileStorage};

// Wrap any backend to make it read-only
let backend = ReadOnly::new(SqliteBackend::open("published.db")?);
let fs = FileStorage::new(backend);

fs.read("/doc.txt")?;     // OK
fs.write("/doc.txt", b"x"); // Error: FsError::ReadOnly
```

### RateLimit<B>

Limits operations per second. Prevents runaway agents.

```rust
use anyfs::{MemoryBackend, RateLimit};
use std::time::Duration;

let backend = RateLimitLayer::builder()
    .max_ops(100)                        // 100 ops per window
    .window(Duration::from_secs(1))      // 1 second window
    .max_burst(10)                       // Allow bursts up to 10
    .build()
    .layer(MemoryBackend::new());

// When rate exceeded: FsError::RateLimitExceeded
```

### DryRun<B>

Logs operations without executing writes. Great for testing and debugging.

```rust
use anyfs::{MemoryBackend, DryRun, FileStorage};

let backend = DryRun::new(MemoryBackend::new());
let fs = FileStorage::new(backend);

fs.write("/test.txt", b"hello")?;  // Logged but not written
let _ = fs.read("/test.txt");       // Error: file doesn't exist

// To inspect recorded operations, keep the DryRun handle before wrapping it.
```

### Cache<B>

LRU cache for read operations. Essential for slow backends (S3, network).

```rust
use anyfs::{SqliteBackend, Cache, FileStorage};

let backend = CacheLayer::builder()
    .max_size(100 * 1024 * 1024)      // 100 MB cache
    .max_entries(10_000)              // Max 10K entries
    .ttl(Duration::from_secs(300))    // 5 min TTL
    .build()
    .layer(SqliteBackend::open("data.db")?);
let fs = FileStorage::new(backend);

// First read: hits backend, caches result
let data = fs.read("/file.txt")?;

// Second read: served from cache (fast!)
let data = fs.read("/file.txt")?;
```

### Overlay<Base, Upper>

Union filesystem with a read-only base and writable upper layer. Like Docker.

```rust
use anyfs::{SqliteBackend, MemoryBackend, Overlay};

// Base: read-only template
let base = SqliteBackend::open("template.db")?;

// Upper: writable layer for changes
let upper = MemoryBackend::new();

let backend = Overlay::new(base, upper);

// Reads check upper first, then base
// Writes always go to upper
// Deletes in upper "shadow" base files
```

**Use cases:**
- Container images (base image + writable layer)
- Template filesystems with per-user modifications
- Testing with rollback capability

---

## FileStorage<B, M> (in `anyfs`)

`FileStorage<B, M>` is a **zero-cost ergonomic wrapper** with:
- **`B`** - Backend type (generic, no boxing)
- **`M`** - Optional marker type for compile-time safety

**Axum-style design:** Zero-cost by default, type erasure opt-in via `.boxed()`.

### Basic Usage

```rust
use anyfs::{MemoryBackend, FileStorage};

// Type is inferred - no need to write it out
let fs = FileStorage::new(MemoryBackend::new());

fs.create_dir_all("/documents")?;
fs.write("/documents/hello.txt", b"Hello!")?;
let content = fs.read("/documents/hello.txt")?;
```

### Marker Types (Type-Safe Containers)

Use `_` to infer backend while specifying marker:

```rust
use anyfs::{MemoryBackend, SqliteBackend, FileStorage};

// Define marker types for your domains
struct Sandbox;
struct UserData;

// Specify marker, infer backend with _
let sandbox: FileStorage<_, Sandbox> = FileStorage::new(MemoryBackend::new());
let userdata: FileStorage<_, UserData> = FileStorage::new(SqliteBackend::open("data.db")?);

// Type-safe function signatures prevent mixing containers
fn process_sandbox(fs: &FileStorage<impl Fs, Sandbox>) {
    // Can only accept Sandbox-marked containers
}

fn save_user_file(fs: &FileStorage<impl Fs, UserData>, name: &str, data: &[u8]) {
    // Can only accept UserData-marked containers
}

// Compile-time safety:
process_sandbox(&sandbox);   // OK
process_sandbox(&userdata);  // Compile error! Type mismatch
```

### Self-Documenting Types

Both type parameters are meaningful:

```rust
FileStorage<SqliteBackend, TenantA>   // SQLite storage for TenantA
FileStorage<MemoryBackend, Sandbox>   // In-memory sandbox
FileStorage<StdFsBackend, Production> // Real filesystem, production
```

### Type Aliases for Clean Code

```rust
// Define your standard secure stack
type SecureBackend = Tracing<Restrictions<Quota<SqliteBackend>>>;

// Type aliases for common combinations
type SandboxFs = FileStorage<MemoryBackend, Sandbox>;
type UserDataFs = FileStorage<SecureBackend, UserData>;

// Clean function signatures
fn run_agent(fs: &SandboxFs) { ... }
```

### FileStorage Implementation

```rust
use std::marker::PhantomData;
use anyfs_backend::PathResolver;
use anyfs::resolvers::IterativeResolver;

/// Zero-cost ergonomic wrapper.
/// Generic over backend (B), resolver (R), and marker (M).
pub struct FileStorage<B, R = IterativeResolver, M = ()> {
    backend: B,
    resolver: R,
    _marker: PhantomData<M>,
}

impl<B: Fs, M> FileStorage<B, IterativeResolver, M> {
    /// Create with default resolver (IterativeResolver).
    /// Marker type is specified via type annotation:
    /// `let fs: FileStorage<_, _, MyMarker> = FileStorage::new(backend);`
    pub fn new(backend: B) -> Self { ... }
}

impl<B: Fs, R: PathResolver, M> FileStorage<B, R, M> {
    /// Create with custom path resolver (see ADR-033).
    pub fn with_resolver(backend: B, resolver: R) -> Self { ... }

    /// Type-erase the backend (opt-in boxing).
    /// Note: resolver type is preserved.
    pub fn boxed(self) -> FileStorage<Box<dyn Fs>, R, M> { ... }
}
```

### Type Erasure (Opt-in)

When you need uniform types (e.g., collections), use `.boxed()`:

```rust
// Type-erased for uniform storage
let filesystems: Vec<FileStorage<Box<dyn Fs>>> = vec![
    FileStorage::new(MemoryBackend::new()).boxed(),
    FileStorage::new(SqliteBackend::open("a.db")?).boxed(),
];
```

---

## Layer Trait (in `anyfs-backend`)

The `Layer` trait (inspired by Tower) standardizes middleware composition:

```rust
/// A layer that wraps a backend to add functionality.
pub trait Layer<B: Fs> {
    type Backend: Fs;
    fn layer(self, backend: B) -> Self::Backend;
}
```

Each middleware provides a corresponding `Layer` implementation:

```rust
// QuotaLayer, TracingLayer, RestrictionsLayer, etc.
pub struct QuotaLayer { limits: QuotaLimits }

impl<B: Fs> Layer<B> for QuotaLayer {
    type Backend = Quota<B>;
    fn layer(self, backend: B) -> Self::Backend {
        Quota::from_limits(self.limits).layer(backend)
    }
}
```

**Note:** Middleware that implements additional traits (like `FsInode`) can use more specific bounds to preserve capabilities through the layer.

---

## Composing Middleware

Middleware composes by wrapping. Order matters - innermost applies first.

### Fluent Composition

Use the `.layer()` extension method for Axum-style composition:

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, TracingLayer};

let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(RestrictionsLayer::builder()
        .deny_permissions()  // Block set_permissions()
        .build())
    .layer(TracingLayer::new());
```

### BackendStack Builder

For complex stacks, use `BackendStack` for a fluent API:

```rust
use anyfs::BackendStack;

let fs = BackendStack::new(SqliteBackend::open("data.db")?)
    .limited(|l| l
        .max_total_size(100 * 1024 * 1024)
        .max_file_size(10 * 1024 * 1024))
    .restricted(|g| g
        .deny_permissions())    // Block set_permissions() calls
    .traced()
    .into_container();
```

---

## Built-in Backends

| Backend               | Feature            | Description                                                    |
| --------------------- | ------------------ | -------------------------------------------------------------- |
| `MemoryBackend`       | `memory` (default) | In-memory storage                                              |
| `SqliteBackend`       | `sqlite`           | Single-file portable database                                  |
| `SqliteCipherBackend` | `sqlite-cipher`    | Encrypted SQLite via SQLCipher (AES-256)                       |
| `IndexedBackend`      | `indexed`          | Virtual paths + disk blobs (large file support with isolation) |
| `StdFsBackend`        | `stdfs`            | Direct `std::fs` delegation (no containment)                   |
| `VRootFsBackend`      | `vrootfs`          | Host filesystem with path containment (via strict-path)        |

**Note:** `sqlite` and `sqlite-cipher` features are mutually exclusive (both use rusqlite with different SQLite builds).

---

## Path Handling

Core traits take `&Path` so they are object-safe (`dyn Fs` works). The ergonomic layer (`FileStorage` and `FsExt`) accepts `impl AsRef<Path>`:

```rust
// These work via FileStorage/FsExt
fs.write("/file.txt", data)?;
fs.write(String::from("/file.txt"), data)?;
fs.write(PathBuf::from("/file.txt"), data)?;
```

---

## Path Resolution

Path resolution (walking directory structure, following symlinks) operates on the **`Fs` abstraction**, not reimplemented per-backend.

See ADR-029 for the path-resolution decision.

### Why Abstract Path Resolution?

We simulate inodes - that's the whole point of virtualizing a filesystem. Path resolution must work on that abstraction:

- `/foo/../bar` cannot be resolved lexically - `foo` might be a symlink to `/other/place`, making `..` resolve to `/other`
- Resolution requires following the actual directory structure (inodes)
- The `Fs` traits have the needed methods: `metadata()`, `read_link()`, `read_dir()`

### Path Resolution via PathResolver Trait

`FileStorage` delegates path resolution to a pluggable `PathResolver` (see ADR-033). The default `IterativeResolver` walks paths component by component:

```rust
/// Default resolver algorithm (simplified):
/// - Walk path component by component
/// - Use backend.metadata() to check node types
/// - If backend implements FsLink, use read_link() to follow symlinks
/// - Detect circular symlinks (max depth: 40)
/// - Return fully resolved canonical path
pub struct IterativeResolver {
    max_symlink_depth: usize,  // Default: 40
}
```

**Resolution behavior depends on the resolver used.** The default `IterativeResolver` follows symlinks when the backend implements `FsLink`. For backends without `FsLink`, it traverses directories but treats symlinks as regular files. Users can provide custom resolvers for case-insensitive matching, caching, or other behaviors.

**Note:** All built-in virtual backends (MemoryBackend, SqliteBackend) implement `FsLink`, so symlink-aware resolution works out of the box.

### When Resolution Is Needed

| Backend          | Needs Our Resolution? | Why                                                   |
| ---------------- | --------------------- | ----------------------------------------------------- |
| `MemoryBackend`  | Yes                   | Storage (HashMap) has no FS semantics                 |
| `SqliteBackend`  | Yes                   | Storage (SQL tables) has no FS semantics              |
| `VRootFsBackend` | No                    | OS handles resolution; `strict-path` prevents escapes |

### Opt-out Mechanism

Virtual backends need resolution by default. Real filesystem backends opt out via a marker trait or associated constant:

```rust
/// Marker trait for backends that handle their own path resolution.
/// VRootFsBackend implements this because the OS handles resolution.
pub trait SelfResolving {}

impl SelfResolving for VRootFsBackend {}
```

`FileStorage` applies resolution via its `PathResolver` for backends that don't implement `SelfResolving`. The default `IterativeResolver` follows symlinks when `FsLink` is available. Custom resolvers can implement different behaviors (e.g., no symlink following, caching, case-insensitivity).

```rust
impl<B: Fs, M> FileStorage<B, IterativeResolver, M> {
    pub fn new(backend: B) -> Self { /* uses IterativeResolver */ }
}

impl<B: Fs, R: PathResolver, M> FileStorage<B, R, M> {
    pub fn with_resolver(backend: B, resolver: R) -> Self { /* custom resolver */ }
}
```

---

## Path Canonicalization Utilities

`FileStorage` provides path canonicalization methods modeled after the [soft-canonicalize](https://crates.io/crates/soft-canonicalize) crate, adapted to work on the virtual filesystem abstraction.

### Why We Need Our Own Canonicalization

`std::fs::canonicalize` operates on the **real** filesystem. For virtual backends (`MemoryBackend`, `SqliteBackend`), there is no real filesystem - we need canonicalization that queries the virtual structure via `metadata()` and `read_link()`.

### Core Methods

```rust
impl<B: Fs + FsLink, M> FileStorage<B, M> {
    /// Strict canonicalization - entire path must exist.
    ///
    /// Resolves all symlinks and normalizes the path.
    /// Returns error if any component doesn't exist.
    pub fn canonicalize(&self, path: impl AsRef<Path>) -> Result<PathBuf, FsError>;

    /// Soft canonicalization - resolves existing components,
    /// appends non-existent remainder lexically.
    ///
    /// Walks path component-by-component:
    /// 1. For existing components → resolve symlinks, follow them
    /// 2. When hitting non-existent component → append remainder lexically
    ///
    /// Inspired by Python's `pathlib.Path.resolve(strict=False)`.
    pub fn soft_canonicalize(&self, path: impl AsRef<Path>) -> Result<PathBuf, FsError>;

    /// Anchored soft canonicalization - like soft_canonicalize but
    /// clamps result within a boundary directory.
    ///
    /// Useful for sandboxing: ensures the resolved path never escapes
    /// the anchor directory, even via symlinks or `..` traversal.
    pub fn anchored_canonicalize(
        &self,
        path: impl AsRef<Path>,
        anchor: impl AsRef<Path>
    ) -> Result<PathBuf, FsError>;
}

/// Standalone lexical normalization (no backend needed).
///
/// Pure string manipulation:
/// - Collapses `//` to `/`
/// - Removes trailing slashes
/// - Does NOT resolve `.` or `..` (those require filesystem context)
/// - Does NOT follow symlinks
pub fn normalize(path: impl AsRef<Path>) -> PathBuf;
```

### Algorithm: Component-by-Component Resolution

The canonicalization algorithm walks the path one component at a time:

```
Input: /a/b/c/d/e

1. Start at root (/)
2. Check /a exists?
   - Yes, and it's a symlink → follow to target
   - Yes, and it's a directory → continue
3. Check /a/b exists?
   - Yes → continue
4. Check /a/b/c exists?
   - No → stop resolution, append "c/d/e" lexically
5. Result: /resolved/path/to/b/c/d/e
```

**Key behaviors:**
- **Symlink following**: Existing symlinks are resolved to their targets
- **Non-existent handling**: When a component doesn't exist, the remainder is appended as-is
- **Cycle detection**: Bounded depth tracking prevents infinite loops from circular symlinks
- **Root boundary**: Never ascends past the filesystem root

### Comparison with std::fs

| Function       | `std::fs`                     | `FileStorage`                                    |
| -------------- | ----------------------------- | ------------------------------------------------ |
| `canonicalize` | Requires all components exist | Same - returns error if path doesn't exist       |
| N/A            | N/A                           | `soft_canonicalize` - handles non-existent paths |
| N/A            | N/A                           | `anchored_canonicalize` - sandboxed resolution   |

### Security Considerations

**For virtual backends:** Canonicalization happens entirely within the virtual structure. There is no host filesystem to escape to.

**For `VRootFsBackend`:** Delegates to OS canonicalization + `strict-path` containment. The `anchored_canonicalize` provides additional safety by clamping paths within a boundary.

### Platform Notes (VRootFsBackend only)

When delegating to OS canonicalization:
- **Windows**: Returns extended-length UNC paths (`\\?\C:\path`) by default
- **Linux/macOS**: Standard canonical paths

#### Windows UNC Path Simplification

The `dunce` crate provides `simplified()` - a **lexical** function that converts UNC paths to regular paths without filesystem access:

```rust
use dunce::simplified;

// \\?\C:\Users\foo\bar.txt → C:\Users\foo\bar.txt
let path = simplified(r"\\?\C:\Users\foo\bar.txt");
```

**Why this matters for `soft_canonicalize`:**
- `soft_canonicalize` works with non-existent paths
- We can't use `dunce::canonicalize` (requires path to exist)
- `dunce::simplified` is pure string manipulation - works on any path

**When UNC can be simplified:**
- Path is on a local drive (C:, D:, etc.)
- Path doesn't exceed MAX_PATH (260 chars)
- No reserved names (CON, PRN, etc.)

**When UNC must be kept:**
- Network paths (`\\?\UNC\server\share`)
- Paths exceeding MAX_PATH
- Paths with reserved device names

Virtual backends have no platform differences - paths are just strings.

---

## Filesystem Semantics: Linux-like by Default

**Design principle:** Simple, secure defaults. Don't close doors for alternative semantics.

See ADR-028 for the decision rationale.

### Default Behavior (Built-in Backends)

| Aspect           | Behavior           | Rationale                           |
| ---------------- | ------------------ | ----------------------------------- |
| Case sensitivity | **Case-sensitive** | Simpler, more secure, Unix standard |
| Path separator   | **`/` internally** | Cross-platform consistency          |
| Reserved names   | **None**           | No artificial restrictions          |
| Max path length  | **No limit**       | Virtual, no OS constraints          |
| ADS (`:stream`)  | **Not supported**  | Security risk, complexity           |

### Trait is Agnostic

The `Fs` trait doesn't enforce filesystem semantics - backends decide their behavior:

```rust
use anyfs::{FileStorage, MemoryBackend};
use std::path::Path;

// Built-in backends: Linux-like (case-sensitive)
let linux_fs = FileStorage::new(MemoryBackend::new());
assert!(linux_fs.exists("/Foo.txt")? != linux_fs.exists("/foo.txt")?);

// For case-insensitive behavior, implement a custom PathResolver:
// (Not built-in because real-world demand is minimal - VRootFsBackend on 
// Windows/macOS already gets case-insensitivity from the OS)
struct CaseFoldingResolver;
impl PathResolver for CaseFoldingResolver {
    fn canonicalize(&self, path: &Path, fs: &dyn Fs) -> Result<PathBuf, FsError> {
        // Normalize path components to lowercase during lookup
        todo!()
    }
    
    fn soft_canonicalize(&self, path: &Path, fs: &dyn Fs) -> Result<PathBuf, FsError> {
        // Same but allows non-existent final component
        todo!()
    }
}

let ntfs_like = FileStorage::with_resolver(
    MemoryBackend::new(),
    CaseFoldingResolver  // User-implemented
);
```

### FUSE Mount: Report What You Support

When mounting, the FUSE layer reports backend capabilities to the OS:

```rust
impl FuseOps for AnyFsFuse<B> {
    fn get_volume_params(&self) -> VolumeParams {
        VolumeParams {
            case_sensitive: self.backend.is_case_sensitive(),
            supports_hard_links: /* check if B: FsLink */,
            supports_symlinks: /* check if B: FsLink */,
            // ...
        }
    }
}
```

Windows respects these flags - a case-sensitive mounted filesystem works correctly (modern Windows/WSL handle this).

### Illustrative: Custom Middleware for Windows Compatibility

For users who need Windows-safe paths in virtual backends, here are example middleware patterns (not built-in - implement as needed):

```rust
/// Example: Middleware that validates paths are Windows-compatible.
/// Rejects: CON, PRN, NUL, COM1-9, LPT1-9, trailing dots/spaces, ADS.
pub struct NtfsValidation<B> { /* user-implemented */ }

/// Example: Middleware that makes a backend case-insensitive.
/// Stores canonical (lowercase) keys, preserves original case in metadata.
pub struct CaseInsensitive<B> { /* user-implemented */ }
```

**Not built-in** - these are illustrative patterns for users who need NTFS-like behavior.

---

## Security Model

Security is achieved through composition:

| Concern             | Solution                                   |
| ------------------- | ------------------------------------------ |
| Path containment    | `PathFilter` + VRootFsBackend              |
| Resource exhaustion | `Quota` enforces quotas                    |
| Rate limiting       | `RateLimit` prevents abuse                 |
| Feature restriction | `Restrictions` disables dangerous features |
| Read-only access    | `ReadOnly` prevents writes                 |
| Audit trail         | `Tracing` instruments operations           |
| Tenant isolation    | Separate backend instances                 |
| Testing             | `DryRun` logs without executing            |

**Defense in depth:** Compose multiple middleware layers for comprehensive security.

### AI Agent Sandbox Example

```rust
use anyfs::{MemoryBackend, Quota, PathFilter, RateLimit, Tracing};

// Build a secure sandbox for an AI agent
let sandbox = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(50 * 1024 * 1024)  // 50 MB
        .max_file_size(5 * 1024 * 1024)    // 5 MB per file
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .deny("**/.env")
        .deny("**/secrets/**")
        .build())
    .layer(RateLimitLayer::builder()
        .max_ops(1000)
        .per_second()
        .build())
    .layer(TracingLayer::new());
```

---

## Extension Traits (in `anyfs-backend`)

The `FsExt` trait provides convenience methods for any `Fs` backend:

```rust
/// Extension methods for Fs (auto-implemented for all backends).
pub trait FsExt: Fs {
    /// Check if path is a file.
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.metadata(path).map(|m| m.file_type == FileType::File)
    }

    /// Check if path is a directory.
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.metadata(path).map(|m| m.file_type == FileType::Directory)
    }

    // JSON methods require `serde` feature (see below)
    #[cfg(feature = "serde")]
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, FsError>;
    #[cfg(feature = "serde")]
    fn write_json<T: Serialize>(&self, path: impl AsRef<Path>, value: &T) -> Result<(), FsError>;
}

// Blanket implementation for all Fs backends
impl<B: Fs> FsExt for B {}
```

### JSON Methods (feature: `serde`)

The `read_json` and `write_json` methods require the `serde` feature:

```toml
anyfs-backend = { version = "0.1", features = ["serde"] }
```

```rust
use serde::{Serialize, de::DeserializeOwned};

#[cfg(feature = "serde")]
impl<B: Fs> FsExt for B {
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, FsError> {
        let bytes = self.read(path)?;
        serde_json::from_slice(&bytes).map_err(|e| FsError::Deserialization(e.to_string()))
    }

    fn write_json<T: Serialize>(&self, path: impl AsRef<Path>, value: &T) -> Result<(), FsError> {
        let bytes = serde_json::to_vec(value).map_err(|e| FsError::Serialization(e.to_string()))?;
        self.write(path, &bytes)
    }
}
```

Users can define their own extension traits for domain-specific operations.

---

## Optional Features

### Bytes Support (feature: `bytes`)

For zero-copy efficiency, enable the `bytes` feature to get `Bytes`-returning convenience methods on `FileStorage`:

```toml
anyfs = { version = "0.1", features = ["bytes"] }
```

```rust
use anyfs::{FileStorage, MemoryBackend};
use bytes::Bytes;

let fs = FileStorage::new(MemoryBackend::new());

// With bytes feature, FileStorage provides read_bytes() convenience method
let data: Bytes = fs.read_bytes("/large-file.bin")?;
let slice = data.slice(1000..2000);  // Zero-copy!

// Core trait still uses Vec<u8> for object safety
// read_bytes() wraps the Vec<u8> in Bytes::from()
```

**Note:** Core traits (`FsRead`, etc.) always use `Vec<u8>` for object safety (`dyn Fs`). The `bytes` feature adds convenience methods to `FileStorage` that wrap results in `Bytes`.

**When to use:**
- Large file handling with frequent slicing
- Network-backed storage
- Streaming scenarios

**Default:** `Vec<u8>` (no extra dependency)

---

## Error Types

`FsError` includes context for better debugging. It implements `std::error::Error` via `thiserror` and uses `#[non_exhaustive]` for forward compatibility.

```rust
/// Filesystem error with context.
///
/// All variants include enough information for meaningful error messages.
/// Use `#[non_exhaustive]` to allow adding variants in minor versions.
#[non_exhaustive]
#[derive(Debug, thiserror::Error)]
pub enum FsError {
    // ========================================================================
    // Path/File Errors
    // ========================================================================

    /// Path not found.
    #[error("not found: {path}")]
    NotFound {
        path: PathBuf,
    },

    /// Security threat detected (e.g., virus).
    #[error("threat detected: {reason} in {path}")]
    ThreatDetected {
        path: PathBuf,
        reason: String,
    },

    /// Path already exists.
    #[error("{operation}: already exists: {path}")]
    AlreadyExists {
        path: PathBuf,
        operation: &'static str,
    },

    /// Expected a file, found directory.
    NotAFile { path: PathBuf },

    /// Expected a directory, found file.
    NotADirectory { path: PathBuf },

    /// Directory not empty (for remove_dir).
    DirectoryNotEmpty { path: PathBuf },

    // ========================================================================
    // Permission/Access Errors
    // ========================================================================

    /// Permission denied (general filesystem permission error).
    PermissionDenied {
        path: PathBuf,
        operation: &'static str,
    },

    /// Access denied (from PathFilter or RBAC).
    AccessDenied {
        path: PathBuf,
        reason: String,  // Dynamic reason string
    },

    /// Read-only filesystem (from ReadOnly middleware).
    ReadOnly {
        operation: &'static str,
    },

    /// Feature not enabled (from Restrictions middleware).
    /// Note: Symlink/hard-link capability is determined by trait bounds (FsLink),
    /// not middleware. Restrictions only controls "permissions".
    FeatureNotEnabled {
        feature: &'static str,  // "permissions"
        operation: &'static str,
    },

    // ========================================================================
    // Resource Limit Errors
    // ========================================================================

    /// Quota exceeded (total storage).
    QuotaExceeded {
        limit: u64,
        requested: u64,
        usage: u64,
    },

    /// File size limit exceeded.
    FileSizeExceeded {
        path: PathBuf,
        size: u64,
        limit: u64,
    },

    /// Rate limit exceeded (from RateLimit middleware).
    RateLimitExceeded {
        limit: u32,
        window_secs: u64,
    },

    // ========================================================================
    // Data Errors
    // ========================================================================

    /// Invalid data (e.g., not valid UTF-8 when string expected).
    InvalidData {
        path: PathBuf,
        details: String,
    },

    /// Corrupted data (e.g., failed checksum, parse error).
    CorruptedData {
        path: PathBuf,
        details: String,
    },

    /// Data integrity verification failed (AEAD tag mismatch, HMAC failure).
    IntegrityError {
        path: PathBuf,
    },

    /// Serialization error (from FsExt JSON methods).
    Serialization(String),

    /// Deserialization error (from FsExt JSON methods).
    Deserialization(String),

    // ========================================================================
    // Backend/Operation Errors
    // ========================================================================

    /// Operation not supported by this backend.
    NotSupported {
        operation: &'static str,
    },

    /// Invalid password or encryption key (from SqliteCipherBackend).
    InvalidPassword,

    /// Conflict during sync (from offline mode).
    Conflict {
        path: PathBuf,
    },

    /// Backend-specific error (catch-all for custom backends).
    Backend(String),

    /// I/O error wrapper.
    Io {
        operation: &'static str,
        path: PathBuf,
        source: std::io::Error,
    },
}

// Required implementations
impl From<std::io::Error> for FsError {
    fn from(err: std::io::Error) -> Self {
        FsError::Io {
            operation: "io",
            path: PathBuf::new(),
            source: err,
        }
    }
}
```

**Implementation notes:**
- All variants have `#[error("...")]` attributes (shown for first two, omitted for brevity)
- `#[non_exhaustive]` allows adding variants in minor versions without breaking changes
- `From<std::io::Error>` enables `?` operator with std::io functions
- Consider `#[must_use]` on functions returning `Result<_, FsError>`

---

## Cross-Platform Compatibility

AnyFS is designed for cross-platform use. Virtual backends work everywhere; real filesystem backends have platform considerations.

### Backend Compatibility

| Backend               | Windows | Linux | macOS | WASM  |
| --------------------- | :-----: | :---: | :---: | :---: |
| `MemoryBackend`       |    ✅    |   ✅   |   ✅   |   ✅   |
| `SqliteBackend`       |    ✅    |   ✅   |   ✅   |  ✅*   |
| `SqliteCipherBackend` |    ✅    |   ✅   |   ✅   |   ❌   |
| `IndexedBackend`      |    ✅    |   ✅   |   ✅   |   ❌   |
| `StdFsBackend`        |    ✅    |   ✅   |   ✅   |   ❌   |
| `VRootFsBackend`      |    ✅    |   ✅   |   ✅   |   ❌   |

*SQLite on WASM requires `wasm32` build of rusqlite with bundled SQLite.

### Feature Compatibility

| Feature             |   Virtual Backends   |         VRootFsBackend         |
| ------------------- | :------------------: | :----------------------------: |
| Basic I/O (`Fs`)    |   ✅ All platforms    |        ✅ All platforms         |
| Symlinks            |   ✅ All platforms    | Platform-dependent (see below) |
| Hard links          |   ✅ All platforms    |       Platform-dependent       |
| Permissions         | ✅ Stored as metadata |       Platform-dependent       |
| Extended attributes | ✅ Stored as metadata |       Platform-dependent       |
| FUSE mounting       |         N/A          |       Platform-dependent       |

### Platform-Specific Notes

#### Virtual Backends (MemoryBackend, SqliteBackend)

**Fully cross-platform.** All features work identically everywhere because:
- Paths are just strings/keys - no OS path resolution
- Symlinks are stored data, not OS constructs
- Permissions are metadata, not enforced by OS
- No filesystem syscalls involved

```rust
// This works identically on Windows, Linux, macOS, and WASM
let fs = FileStorage::new(MemoryBackend::new());
fs.symlink("/target", "/link")?;           // Just stores the link
fs.set_permissions("/file", 0o755.into())?; // Just stores metadata
```

#### VRootFsBackend (Real Filesystem)

Wraps the host filesystem. Platform differences apply:

| Feature                 | Linux   | macOS                 | Windows                |
| ----------------------- | ------- | --------------------- | ---------------------- |
| Symlinks                | ✅       | ✅                     | ⚠️ Requires privileges* |
| Hard links              | ✅       | ✅                     | ✅ (NTFS only)          |
| Permissions (mode bits) | ✅       | ✅                     | ⚠️ Limited mapping      |
| Extended attributes     | ✅ xattr | ✅ xattr               | ⚠️ ADS (different API)  |
| Case sensitivity        | ✅       | ⚠️ Default insensitive | ⚠️ Insensitive          |

*Windows requires `SeCreateSymbolicLinkPrivilege` or Developer Mode for symlinks.

#### FUSE Mounting

| Platform | Support       | Library         |
| -------- | ------------- | --------------- |
| Linux    | ✅ Native      | libfuse         |
| macOS    | ⚠️ Third-party | macFUSE         |
| Windows  | ⚠️ Third-party | WinFsp or Dokan |
| WASM     | ❌             | N/A             |

### Path Handling

Virtual backends use `/` as separator internally, regardless of platform:

```rust
// Always use forward slashes with virtual backends
fs.write("/project/src/main.rs", code)?;  // Works everywhere
```

`VRootFsBackend` translates to native paths internally:
- Linux/macOS: `/` stays `/`
- Windows: `/project/file.txt` → `C:\root\project\file.txt`

### Recommendations

| Use Case               | Recommended Backend                | Why                               |
| ---------------------- | ---------------------------------- | --------------------------------- |
| Cross-platform app     | `MemoryBackend` or `SqliteBackend` | No platform differences           |
| Portable storage       | `SqliteBackend`                    | Single file, works everywhere     |
| WASM/browser           | `MemoryBackend` or `SqliteBackend` | No filesystem access needed       |
| Host filesystem access | `VRootFsBackend`                   | With awareness of platform limits |
| Testing                | `MemoryBackend`                    | Fast, no cleanup, deterministic   |

### Feature Detection

Check platform capabilities at runtime if needed:

```rust
/// Check if symlinks are supported on the current platform.
pub fn symlinks_available() -> bool {
    #[cfg(unix)]
    return true;

    #[cfg(windows)]
    {
        // Check for Developer Mode or symlink privilege
        // ...
    }
}
```

On platforms without symlink support, use a backend that doesn't implement `FsLink`, or check `symlinks_available()` before calling symlink operations.
