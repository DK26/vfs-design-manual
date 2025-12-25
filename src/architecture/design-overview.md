# AnyFS - Design Overview

**Status:** Current
**Last updated:** 2025-12-24

---

## What This Project Is

AnyFS is an **open standard** for pluggable virtual filesystem backends in Rust. It uses a **middleware/decorator pattern** (like Axum/Tower) for composable functionality with complete separation of concerns.

Anyone can:
- Implement a custom backend for their storage needs
- Compose middleware to add limits, logging, feature gates
- Use the ergonomic `FileStorage<M>` wrapper

---

## Architecture (Tower-style Middleware)

```
┌─────────────────────────────────────────┐
│  FileStorage<M>                         │  ← Ergonomics + type-safe marker
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

| Layer | Responsibility |
|-------|----------------|
| Backend (`Fs`+) | Storage + filesystem semantics |
| `Quota<B>` | Resource limits (size, count, depth) |
| `Restrictions<B>` | Opt-in operation restrictions |
| `PathFilter<B>` | Path-based access control |
| `ReadOnly<B>` | Prevent all write operations |
| `RateLimit<B>` | Limit operations per second |
| `Tracing<B>` | Instrumentation / audit trail |

---

## Design Principle: Predictable Defaults, Opt-in Security

**The `Fs` traits mimic `std::fs` with predictable, permissive defaults.**

The traits are low-level interfaces that any backend can implement - memory, SQLite, real filesystem, network storage, etc. To maintain consistent behavior across all backends:

- All operations work by default (`symlink()`, `hard_link()`, `set_permissions()`)
- No security restrictions at the trait level
- Behavior matches what you'd expect from a real filesystem

**Why not secure-by-default at this layer?**

1. **Predictability**: A backend should behave like a filesystem. Surprising restrictions break expectations.
2. **Backend-agnostic**: The traits don't know if they're wrapping a sandboxed memory store or a real filesystem. Restrictions that make sense for one may not for another.
3. **Composition**: Security is achieved by layering middleware, not by baking it into the storage layer.

**Security is the responsibility of higher-level APIs:**

| Layer | Security Responsibility |
|-------|------------------------|
| Backend (`Fs`+) | None - pure filesystem semantics |
| Middleware (`Restrictions`, `PathFilter`, etc.) | Opt-in restrictions |
| `FileStorage` or application code | Configure appropriate middleware |

**Example: Secure AI Agent Sandbox**

```rust
use anyfs::FileStorage;

struct AiSandbox;  // Marker type

// Application composes secure defaults
let sandbox: FileStorage<AiSandbox> = FileStorage::with_marker(
    PathFilter::new(
        Restrictions::new(
            Quota::new(MemoryBackend::new())
                .with_max_total_size(50 * 1024 * 1024)
        )
        .deny_hard_links()
        .deny_permissions()
    )
    .allow("/workspace/**")
    .deny("**/.env")
);
```

The backend is permissive. The application adds restrictions appropriate for its use case.

---

## Crates

| Crate | Purpose | Contains |
|-------|---------|----------|
| `anyfs-backend` | Minimal contract | Layered traits (`Fs`, `FsFull`, `FsFuse`, `FsPosix`), `Layer` trait, types, `FsExt` |
| `anyfs` | Backends + middleware + ergonomics | Built-in backends, all middleware layers, `FileStorage<M>`, `BackendStack` builder |

### Dependency Graph

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware + ergonomics)
           ^-- vrootfs feature may use strict-path
```

---

## Trait Architecture (in `anyfs-backend`)

AnyFS uses **layered traits** for maximum flexibility with minimal complexity.

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
pub trait FsRead: Send {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, FsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, FsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError>;
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError>;
}
```

### FsWrite - Write Operations

```rust
pub trait FsWrite: Send {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), FsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, FsError>;
}
```

### FsDir - Directory Operations

```rust
pub trait FsDir: Send {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, FsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
}
```

---

## Extended Traits (Layer 2 - Optional)

```rust
pub trait FsLink: Send {
    fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, FsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError>;
}

pub trait FsPermissions: Send {
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), FsError>;
}

pub trait FsSync: Send {
    fn sync(&mut self) -> Result<(), FsError>;
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
}

pub trait FsStats: Send {
    fn statfs(&self) -> Result<StatFs, FsError>;
}
```

---

## Inode Traits (Layer 3 - For FUSE)

```rust
pub trait FsInode: Send {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, FsError>;
    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, FsError>;
    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, FsError>;
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, FsError>;
}
```

---

## POSIX Traits (Layer 4 - Full POSIX)

```rust
pub trait FsHandles: Send {
    fn open(&mut self, path: impl AsRef<Path>, flags: OpenFlags) -> Result<Handle, FsError>;
    fn read_at(&self, handle: Handle, buf: &mut [u8], offset: u64) -> Result<usize, FsError>;
    fn write_at(&mut self, handle: Handle, data: &[u8], offset: u64) -> Result<usize, FsError>;
    fn close(&mut self, handle: Handle) -> Result<(), FsError>;
}

pub trait FsLock: Send {
    fn lock(&mut self, handle: Handle, lock: LockType) -> Result<(), FsError>;
    fn try_lock(&mut self, handle: Handle, lock: LockType) -> Result<bool, FsError>;
    fn unlock(&mut self, handle: Handle) -> Result<(), FsError>;
}

pub trait FsXattr: Send {
    fn get_xattr(&self, path: impl AsRef<Path>, name: &str) -> Result<Vec<u8>, FsError>;
    fn set_xattr(&mut self, path: impl AsRef<Path>, name: &str, value: &[u8]) -> Result<(), FsError>;
    fn remove_xattr(&mut self, path: impl AsRef<Path>, name: &str) -> Result<(), FsError>;
    fn list_xattr(&self, path: impl AsRef<Path>) -> Result<Vec<String>, FsError>;
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

### Most Users: Just `Fs`

```rust
use anyfs::Fs;

fn process_files(fs: &impl Fs) {
    let data = fs.read("/input.txt")?;
    fs.write("/output.txt", &processed(data))?;
}
```

### Need Links? Add the Trait

```rust
use anyfs::{Fs, FsLink};

fn with_symlinks(fs: &mut (impl Fs + FsLink)) {
    fs.write("/target.txt", b"content")?;
    fs.symlink("/target.txt", "/link.txt")?;
}
```

### FUSE Mount

```rust
use anyfs::FsFuse;
use anyfs_fuse::FuseMount;

fn mount_filesystem(fs: impl FsFuse) {
    FuseMount::mount(fs, "/mnt/myfs")?;
}
```

### Full POSIX Application

```rust
use anyfs::FsPosix;

fn database_app(fs: &mut impl FsPosix) {
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
pub struct Metadata {
    /// Inode number (unique identifier for this file/directory).
    pub inode: u64,

    /// Number of hard links pointing to this inode.
    pub nlink: u64,

    /// Type: File, Directory, or Symlink.
    pub file_type: FileType,

    /// Size in bytes (0 for directories).
    pub size: u64,

    /// Permission bits.
    pub permissions: Permissions,

    /// Creation time (if supported by backend).
    pub created: Option<SystemTime>,

    /// Last modification time.
    pub modified: Option<SystemTime>,

    /// Last access time (if supported by backend).
    pub accessed: Option<SystemTime>,
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
pub struct DirEntry {
    /// File or directory name (not full path).
    pub name: OsString,

    /// Inode number (avoids extra stat calls).
    pub inode: u64,

    /// Type: File, Directory, or Symlink.
    pub file_type: FileType,
}
```

### Permissions

```rust
/// Unix-style permission bits.
pub struct Permissions {
    /// Permission mode (e.g., 0o755).
    pub mode: u32,
}

impl Permissions {
    pub fn readonly() -> Self { Permissions { mode: 0o444 } }
    pub fn default_file() -> Self { Permissions { mode: 0o644 } }
    pub fn default_dir() -> Self { Permissions { mode: 0o755 } }
}
```

### StatFs

```rust
/// Filesystem statistics.
pub struct StatFs {
    /// Total size in bytes.
    pub total_bytes: u64,

    /// Available bytes.
    pub available_bytes: u64,

    /// Total number of inodes.
    pub total_inodes: u64,

    /// Available inodes.
    pub available_inodes: u64,

    /// Filesystem block size.
    pub block_size: u64,
}
```

---

## Middleware (in `anyfs`)

Each middleware implements the same traits as its inner backend. This enables composition while preserving capabilities.

### Quota<B>

Enforces quota limits. Tracks usage and rejects operations that would exceed limits.

```rust
use anyfs::{SqliteBackend, Quota};

let backend = Quota::new(SqliteBackend::open("data.db")?)
    .with_max_total_size(100 * 1024 * 1024)   // 100 MB
    .with_max_file_size(10 * 1024 * 1024)     // 10 MB per file
    .with_max_node_count(10_000)               // 10K files/dirs
    .with_max_dir_entries(1_000)               // 1K entries per dir
    .with_max_path_depth(64);

// Check usage
let usage = backend.usage();
let remaining = backend.remaining();
```

### Restrictions<B>

Blocks specific operations when needed.

```rust
use anyfs::{MemoryBackend, Restrictions};

// By default, all operations work. Use deny_*() to block specific ones.
let backend = Restrictions::new(MemoryBackend::new())
    .deny_symlinks()       // Block symlink() calls
    .deny_hard_links()     // Block hard_link() calls
    .deny_permissions();   // Block set_permissions() calls
```

When blocked, operations return `FsError::FeatureNotEnabled`.

### Tracing<B>

Integrates with the [tracing](https://docs.rs/tracing) ecosystem for structured logging and instrumentation.

```rust
use anyfs::{SqliteBackend, Tracing};

let backend = Tracing::new(SqliteBackend::open("data.db")?)
    .with_target("anyfs")           // tracing target
    .with_level(tracing::Level::DEBUG);

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

let backend = PathFilter::new(MemoryBackend::new())
    .allow("/workspace/**")           // Allow all under /workspace
    .allow("/tmp/**")                  // Allow temp files
    .deny("/workspace/.env")           // But deny .env files
    .deny("**/.git/**");               // Deny all .git directories
```

When a path is denied, operations return `FsError::AccessDenied`.

### ReadOnly<B>

Prevents all write operations. Useful for publishing immutable data.

```rust
use anyfs::{SqliteBackend, ReadOnly};

// Wrap any backend to make it read-only
let backend = ReadOnly::new(SqliteBackend::open("published.db")?);

backend.read("/doc.txt")?;     // OK
backend.write("/doc.txt", b"x"); // Error: FsError::ReadOnly
```

### RateLimit<B>

Limits operations per second. Prevents runaway agents.

```rust
use anyfs::{MemoryBackend, RateLimit};
use std::time::Duration;

let backend = RateLimit::new(MemoryBackend::new())
    .with_max_ops(100)                        // 100 ops per window
    .with_window(Duration::from_secs(1))      // 1 second window
    .with_max_burst(10);                      // Allow bursts up to 10

// When rate exceeded: FsError::RateLimitExceeded
```

### DryRun<B>

Logs operations without executing writes. Great for testing and debugging.

```rust
use anyfs::{MemoryBackend, DryRun};

let mut backend = DryRun::new(MemoryBackend::new());

backend.write("/test.txt", b"hello")?;  // Logged but not written
backend.read("/test.txt");               // Error: file doesn't exist

// Get recorded operations
let ops = backend.operations();
// [Operation::Write { path: "/test.txt", size: 5 }]
```

### Cache<B>

LRU cache for read operations. Essential for slow backends (S3, network).

```rust
use anyfs::{SqliteBackend, Cache};

let backend = Cache::new(SqliteBackend::open("data.db")?)
    .with_max_size(100 * 1024 * 1024)  // 100 MB cache
    .with_max_entries(10_000)           // Max 10K entries
    .with_ttl(Duration::from_secs(300)); // 5 min TTL

// First read: hits backend, caches result
let data = backend.read("/file.txt")?;

// Second read: served from cache (fast!)
let data = backend.read("/file.txt")?;
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

## FileStorage<M> (in `anyfs`)

`FileStorage<M>` is a **thin ergonomic wrapper** with an optional **marker type** for compile-time safety. It type-erases the backend for a clean API. All policy is handled by middleware.

### Basic Usage

```rust
use anyfs::{MemoryBackend, FileStorage};

let mut fs = FileStorage::new(MemoryBackend::new());

fs.create_dir_all("/documents")?;
fs.write("/documents/hello.txt", b"Hello!")?;
let content = fs.read("/documents/hello.txt")?;
```

### Marker Types (Type-Safe Containers)

The second type parameter `M` (default: `()`) enables compile-time container differentiation:

```rust
use anyfs::{MemoryBackend, SqliteBackend, FileStorage};

// Define marker types for your domains
struct Sandbox;
struct UserData;
struct TempFiles;

// Create typed containers - backend type is erased!
let sandbox: FileStorage<Sandbox> = FileStorage::with_marker(MemoryBackend::new());
let userdata: FileStorage<UserData> = FileStorage::with_marker(SqliteBackend::open("data.db")?);

// Type-safe function signatures prevent mixing containers
fn process_sandbox(fs: &FileStorage<Sandbox>) {
    // Can only accept Sandbox-marked containers
}

fn save_user_file(fs: &mut FileStorage<UserData>, name: &str, data: &[u8]) {
    // Can only accept UserData-marked containers
}

// Compile-time safety:
process_sandbox(&sandbox);   // OK
process_sandbox(&userdata);  // Compile error! Type mismatch
```

### When to Use Markers

| Scenario | Use Markers? | Why |
|----------|--------------|-----|
| Single container | No | `FileStorage` is sufficient |
| Multiple containers, same type | **Yes** | Prevent accidental mixing |
| Multi-tenant systems | **Yes** | Compile-time tenant isolation |
| Sandbox + user data | **Yes** | Never write user data to sandbox |
| Testing | Maybe | Tag test vs production containers |

### FileStorage Implementation

```rust
use std::marker::PhantomData;

/// Ergonomic filesystem wrapper with optional type marker.
/// Backend is type-erased to `Box<dyn Fs>` for a clean API.
pub struct FileStorage<M = ()> {
    backend: Box<dyn Fs>,
    _marker: PhantomData<M>,
}

impl<M> FileStorage<M> {
    /// Create a new FileStorage (no marker).
    pub fn new(backend: impl Fs + 'static) -> FileStorage {
        FileStorage { backend: Box::new(backend), _marker: PhantomData }
    }

    /// Create a new FileStorage with a specific marker type.
    pub fn with_marker<N>(backend: impl Fs + 'static) -> FileStorage<N> {
        FileStorage { backend: Box::new(backend), _marker: PhantomData }
    }

    /// Change the marker type (zero-cost, compile-time only).
    pub fn retype<N>(self) -> FileStorage<N> {
        FileStorage { backend: self.backend, _marker: PhantomData }
    }
}
```

**Note:** `FileStorage` uses `Box<dyn Fs>` which provides Layer 1 operations. If you need `FsFull`, `FsFuse`, or `FsPosix` operations, use the backend directly without type erasure.

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
pub struct QuotaLayer { /* config */ }

impl<B: Fs> Layer<B> for QuotaLayer {
    type Backend = Quota<B>;
    fn layer(self, backend: B) -> Self::Backend {
        Quota::new(backend).with_limits(self.limits)
    }
}
```

**Note:** Middleware that implements additional traits (like `FsInode`) can use more specific bounds to preserve capabilities through the layer.

---

## Composing Middleware

Middleware composes by wrapping. Order matters - innermost applies first.

### Manual Composition

```rust
use anyfs::{SqliteBackend, Quota, Restrictions, Tracing, FileStorage};

// Build from inside out:
let backend = SqliteBackend::open("data.db")?;

let limited = Quota::new(backend)
    .with_max_total_size(100 * 1024 * 1024);

let restricted = Restrictions::new(limited)
    .deny_hard_links()
    .deny_permissions();

let traced = Tracing::new(restricted);

let mut fs = FileStorage::new(traced);
```

### Layer-based Composition

Use the `Layer` trait for Axum-style composition:

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, TracingLayer};

let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(RestrictionsLayer::new().deny_permissions())  // Block set_permissions()
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
        .deny_hard_links()      // Block hard_link() calls
        .deny_permissions())    // Block set_permissions() calls
    .traced()
    .into_container();
```

---

## Built-in Backends

| Backend | Feature | Description |
|---------|---------|-------------|
| `MemoryBackend` | `memory` (default) | In-memory storage |
| `SqliteBackend` | `sqlite` | Single-file portable database |
| `StdFsBackend` | `stdfs` | Direct `std::fs` delegation (no containment) |
| `VRootFsBackend` | `vrootfs` | Host filesystem with path containment (via strict-path) |

---

## Path Handling

All layers use `impl AsRef<Path>`, aligned with `std::fs`:

```rust
// All of these work
fs.write("/file.txt", data)?;
fs.write(String::from("/file.txt"), data)?;
fs.write(Path::new("/file.txt"), data)?;
fs.write(PathBuf::from("/file.txt"), data)?;
```

---

## Path Resolution

Path resolution (walking directory structure, following symlinks) operates on the **`Vfs` abstraction**, not reimplemented per-backend.

### Why Abstract Path Resolution?

We simulate inodes - that's the whole point of virtualizing a filesystem. Path resolution must work on that abstraction:

- `/foo/../bar` cannot be resolved lexically - `foo` might be a symlink to `/other/place`, making `..` resolve to `/other`
- Resolution requires following the actual directory structure (inodes)
- The `Fs` traits have the needed methods: `metadata()`, `read_link()`, `read_dir()`

### Resolution Utility (in `anyfs`)

```rust
/// Resolve a path, optionally following symlinks.
/// Works on any Fs + FsLink backend.
pub fn resolve_path(
    backend: &(impl Fs + FsLink),
    path: impl AsRef<Path>,
    follow_symlinks: bool,
) -> Result<PathBuf, FsError> {
    // Walk path component by component
    // Use backend.metadata() to check node types
    // Use backend.read_link() to follow symlinks (if enabled)
    // Detect circular symlinks
    // Return fully resolved canonical path
}
```

### When Resolution Is Needed

| Backend | Needs Our Resolution? | Why |
|---------|----------------------|-----|
| `MemoryBackend` | Yes | Storage (HashMap) has no FS semantics |
| `SqliteBackend` | Yes | Storage (SQL tables) has no FS semantics |
| `VRootFsBackend` | No | OS handles resolution; `strict-path` prevents escapes |

### Opt-out Mechanism

Virtual backends need resolution by default. Real filesystem backends opt out via a marker trait or associated constant:

```rust
/// Marker trait for backends that handle their own path resolution.
/// VRootFsBackend implements this because the OS handles resolution.
pub trait SelfResolving {}

impl SelfResolving for VRootFsBackend {}
```

`FileStorage` (or a dedicated wrapper) applies resolution automatically for backends that don't implement `SelfResolving`:

```rust
impl<M> FileStorage<M> {
    pub fn new(backend: impl Vfs + 'static) -> FileStorage {
        // Resolution applied automatically if backend doesn't implement SelfResolving
    }
}
```

---

## Security Model

Security is achieved through composition:

| Concern | Solution |
|---------|----------|
| Path containment | `PathFilter` + VRootFsBackend |
| Resource exhaustion | `Quota` enforces quotas |
| Rate limiting | `RateLimit` prevents abuse |
| Feature restriction | `Restrictions` disables dangerous features |
| Read-only access | `ReadOnly` prevents writes |
| Audit trail | `Tracing` instruments operations |
| Tenant isolation | Separate backend instances |
| Testing | `DryRun` logs without executing |

**Defense in depth:** Compose multiple middleware layers for comprehensive security.

### AI Agent Sandbox Example

```rust
use anyfs::{MemoryBackend, Quota, PathFilter, RateLimit, Restrictions, Tracing};

// Build a secure sandbox for an AI agent
let sandbox = MemoryBackend::new()
    .layer(QuotaLayer::new()
        .max_total_size(50 * 1024 * 1024)  // 50 MB
        .max_file_size(5 * 1024 * 1024))   // 5 MB per file
    .layer(PathFilterLayer::new()
        .allow("/workspace/**")
        .deny("**/.env")
        .deny("**/secrets/**"))
    .layer(RestrictionsLayer::new())        // No symlinks, no hardlinks
    .layer(RateLimitLayer::new()
        .max_ops(1000)
        .per_second())
    .layer(TracingLayer::new());
```

---

## Extension Traits (in `anyfs-backend`)

The `FsExt` trait provides convenience methods for any `Vfs` backend:

```rust
use serde::{Serialize, de::DeserializeOwned};

/// Extension methods for Fs (auto-implemented for all backends).
pub trait FsExt: Fs {
    /// Read and deserialize JSON.
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, FsError> {
        let bytes = self.read(path)?;
        serde_json::from_slice(&bytes).map_err(|e| FsError::Deserialization(e.to_string()))
    }

    /// Serialize and write JSON.
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), FsError> {
        let bytes = serde_json::to_vec(value).map_err(|e| FsError::Serialization(e.to_string()))?;
        self.write(path, &bytes)
    }

    /// Check if path is a file.
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.metadata(path).map(|m| m.file_type == FileType::File)
    }

    /// Check if path is a directory.
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.metadata(path).map(|m| m.file_type == FileType::Directory)
    }
}

// Blanket implementation for all Fs backends
impl<B: Fs> FsExt for B {}
```

Users can define their own extension traits for domain-specific operations.

---

## Optional Features

### Bytes Support (feature: `bytes`)

For zero-copy efficiency, enable the `bytes` feature to use `Bytes` instead of `Vec<u8>`:

```toml
anyfs = { version = "0.1", features = ["bytes"] }
```

```rust
use bytes::Bytes;

// With bytes feature, read returns Bytes (O(1) slicing)
let data: Bytes = backend.read("/large-file.bin")?;
let slice = data.slice(1000..2000);  // Zero-copy!
```

**When to use:**
- Large file handling with frequent slicing
- Network-backed storage
- Streaming scenarios

**Default:** `Vec<u8>` (no extra dependency)

---

## Error Types

`FsError` includes context for better debugging:

```rust
pub enum FsError {
    /// Path not found.
    NotFound {
        path: PathBuf,
        operation: &'static str,  // "read", "metadata", etc.
    },

    /// Path already exists.
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

    /// Quota exceeded.
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

    /// Feature not enabled (from Restrictions).
    FeatureNotEnabled {
        feature: &'static str,  // "symlinks", "hard_links", "permissions"
        operation: &'static str,
    },

    /// Access denied (from PathFilter).
    AccessDenied {
        path: PathBuf,
        reason: &'static str,  // "path_denied", "pattern_blocked"
    },

    /// Read-only filesystem (from ReadOnly).
    ReadOnly {
        operation: &'static str,
    },

    /// Rate limit exceeded (from RateLimit).
    RateLimitExceeded {
        limit: u32,
        window_secs: u64,
    },

    /// Serialization error (from FsExt).
    Serialization(String),

    /// Deserialization error (from FsExt).
    Deserialization(String),

    /// Backend-specific error.
    Backend(String),

    /// I/O error.
    Io(std::io::Error),
}
```

All error variants include enough context for meaningful error messages.
