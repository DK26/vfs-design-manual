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
│  VfsBackend                             │  ← Pure storage + fs semantics
│  (Memory, SQLite, VRootFs, custom...)   │
└─────────────────────────────────────────┘
```

**Each layer has exactly one responsibility:**

| Layer | Responsibility |
|-------|----------------|
| `VfsBackend` | Storage + filesystem semantics |
| `Quota<B>` | Resource limits (size, count, depth) |
| `Restrictions<B>` | Opt-in operation restrictions |
| `PathFilter<B>` | Path-based access control |
| `ReadOnly<B>` | Prevent all write operations |
| `RateLimit<B>` | Limit operations per second |
| `Tracing<B>` | Instrumentation / audit trail |

---

## Design Principle: Predictable Defaults, Opt-in Security

**`VfsBackend` mimics `std::fs` with predictable, permissive defaults.**

The trait is a low-level interface that any backend can implement - memory, SQLite, real filesystem, network storage, etc. To maintain consistent behavior across all backends:

- All operations work by default (`symlink()`, `hard_link()`, `set_permissions()`)
- No security restrictions at the trait level
- Behavior matches what you'd expect from a real filesystem

**Why not secure-by-default at this layer?**

1. **Predictability**: A backend should behave like a filesystem. Surprising restrictions break expectations.
2. **Backend-agnostic**: The trait doesn't know if it's wrapping a sandboxed memory store or a real filesystem. Restrictions that make sense for one may not for another.
3. **Composition**: Security is achieved by layering middleware, not by baking it into the storage layer.

**Security is the responsibility of higher-level APIs:**

| Layer | Security Responsibility |
|-------|------------------------|
| `VfsBackend` | None - pure filesystem semantics |
| Middleware (`Restrictions`, `PathFilter`, etc.) | Opt-in restrictions |
| `FileStorage` or application code | Configure appropriate middleware |

**Example: Secure AI Agent Sandbox**

```rust
use anyfs_container::FileStorage;

struct AiSandbox;  // Marker type

// Application composes secure defaults
let sandbox: FileStorage<_, AiSandbox> = FileStorage::with_marker(
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
| `anyfs-backend` | Minimal contract | `VfsBackend` trait, `Layer` trait, types, `VfsBackendExt` |
| `anyfs` | Backends + middleware | Built-in backends, all middleware layers |
| `anyfs-container` | Ergonomic wrapper | `FileStorage<M>`, `BackendStack` builder |

### Dependency Graph

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware)
     |     ^-- vrootfs feature may use strict-path
     |
     +-- anyfs-container (ergonomic wrapper)
```

---

## Core Trait: `VfsBackend` (in `anyfs-backend`)

`VfsBackend` is a path-based trait aligned with `std::fs`. It handles **storage + filesystem semantics only**. No policy, no limits.

```rust
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

pub trait VfsBackend: Send {
    // Read
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;

    // Write
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;

    // Links
    fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;

    // Permissions
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;

    // Streaming I/O
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;

    // File size
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError>;

    // Durability
    fn sync(&mut self) -> Result<(), VfsError>;
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;

    // Filesystem info
    fn statfs(&self) -> Result<StatFs, VfsError>;

    // --- Inode operations (with sensible defaults) ---
    // Backends can override for hardlink support and FUSE efficiency.
    // Default implementations work but are less efficient.

    /// Get the inode number for a path.
    /// Default: hashes the path (works, but hardlinks won't share inodes).
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError> {
        use std::hash::{Hash, Hasher};
        let mut hasher = std::collections::hash_map::DefaultHasher::new();
        path.as_ref().hash(&mut hasher);
        Ok(hasher.finish())
    }

    /// Get the path for an inode number.
    /// Default: not supported (override for FUSE efficiency).
    fn inode_to_path(&self, _inode: u64) -> Result<PathBuf, VfsError> {
        Err(VfsError::NotSupported { operation: "inode_to_path" })
    }

    /// Lookup a child by name within a parent directory (by inode).
    /// Default: uses path-based lookup (override for FUSE efficiency).
    fn lookup(&self, parent_inode: u64, name: &std::ffi::OsStr) -> Result<u64, VfsError> {
        let parent_path = self.inode_to_path(parent_inode)?;
        self.path_to_inode(parent_path.join(name))
    }

    /// Get metadata by inode (avoids path resolution).
    /// Default: uses path-based metadata (override for FUSE efficiency).
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, VfsError> {
        let path = self.inode_to_path(inode)?;
        self.metadata(path)
    }
}
```

### Inode Support

Inode methods have **sensible defaults** that work out of the box. Override them for:
- **Hardlink support**: Two paths must share the same inode
- **FUSE efficiency**: Native inode operations avoid path lookups

**Three levels of implementation:**

| Level | Override | Use Case |
|-------|----------|----------|
| **Basic** | Nothing | Simple backends, no hardlinks, no FUSE |
| **Hardlinks** | `path_to_inode` | Backends that support `hard_link()` |
| **Full FUSE** | All 4 methods | Maximum FUSE performance |

**Level 1: Basic (use defaults)**

```rust
impl VfsBackend for SimpleBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> { ... }
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> { ... }
    // ... implement 25 path-based methods

    // Inode methods: use defaults (no override needed!)
    // - path_to_inode: hashes path
    // - inode_to_path: returns NotSupported
    // - lookup/metadata_by_inode: fall back to path-based
}
```

**Level 2: Hardlink support**

For `hard_link()` to work correctly, two paths must return the same inode:

```rust
impl VfsBackend for MemoryBackend {
    // ... path-based methods ...

    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError> {
        // Look up the node ID in our internal HashMap
        let node = self.nodes.get(path.as_ref())
            .ok_or(VfsError::NotFound { path: path.as_ref().into() })?;
        Ok(node.id)  // Same ID for hardlinked paths!
    }

    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError> {
        // Both paths point to the same node
        let node_id = self.path_to_inode(&original)?;
        self.paths.insert(link.as_ref().to_path_buf(), node_id);
        self.nodes.get_mut(&node_id).unwrap().nlink += 1;
        Ok(())
    }
}
```

**Level 3: Full FUSE efficiency**

Override all 4 methods for O(1) inode operations:

```rust
impl VfsBackend for SqliteBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError> {
        let id: i64 = self.conn.query_row(
            "SELECT id FROM nodes WHERE path = ?",
            [path.as_ref().to_string_lossy()],
            |row| row.get(0),
        )?;
        Ok(id as u64)
    }

    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, VfsError> {
        let path: String = self.conn.query_row(
            "SELECT path FROM nodes WHERE id = ?",
            [inode as i64],
            |row| row.get(0),
        )?;
        Ok(PathBuf::from(path))
    }

    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, VfsError> {
        let id: i64 = self.conn.query_row(
            "SELECT id FROM nodes WHERE parent_id = ? AND name = ?",
            params![parent_inode as i64, name.to_string_lossy()],
            |row| row.get(0),
        )?;
        Ok(id as u64)
    }

    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, VfsError> {
        self.conn.query_row(
            "SELECT type, size, created, modified, nlink FROM nodes WHERE id = ?",
            [inode as i64],
            |row| Ok(Metadata { ... }),
        )
    }
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

Each middleware is itself a `VfsBackend` that wraps another backend. This enables composition.

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

When blocked, operations return `VfsError::FeatureNotEnabled`.

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

When a path is denied, operations return `VfsError::AccessDenied`.

### ReadOnly<B>

Prevents all write operations. Useful for publishing immutable data.

```rust
use anyfs::{SqliteBackend, ReadOnly};

// Wrap any backend to make it read-only
let backend = ReadOnly::new(SqliteBackend::open("published.db")?);

backend.read("/doc.txt")?;     // OK
backend.write("/doc.txt", b"x"); // Error: VfsError::ReadOnly
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

// When rate exceeded: VfsError::RateLimitExceeded
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

## FileStorage<M> (in `anyfs-container`)

`FileStorage<M>` is a **thin ergonomic wrapper** with an optional **marker type** for compile-time safety. It type-erases the backend for a clean API. All policy is handled by middleware.

### Basic Usage

```rust
use anyfs::MemoryBackend;
use anyfs_container::FileStorage;

let mut fs = FileStorage::new(MemoryBackend::new());

fs.create_dir_all("/documents")?;
fs.write("/documents/hello.txt", b"Hello!")?;
let content = fs.read("/documents/hello.txt")?;
```

### Marker Types (Type-Safe Containers)

The second type parameter `M` (default: `()`) enables compile-time container differentiation:

```rust
use anyfs::{MemoryBackend, SqliteBackend};
use anyfs_container::FileStorage;

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
/// Backend is type-erased for a clean API.
pub struct FileStorage<M = ()> {
    backend: Box<dyn VfsBackend>,
    _marker: PhantomData<M>,
}

impl<M> FileStorage<M> {
    /// Create a new FileStorage (no marker).
    pub fn new(backend: impl VfsBackend + 'static) -> FileStorage {
        FileStorage { backend: Box::new(backend), _marker: PhantomData }
    }

    /// Create a new FileStorage with a specific marker type.
    pub fn with_marker<N>(backend: impl VfsBackend + 'static) -> FileStorage<N> {
        FileStorage { backend: Box::new(backend), _marker: PhantomData }
    }

    /// Change the marker type (zero-cost, compile-time only).
    pub fn retype<N>(self) -> FileStorage<N> {
        FileStorage { backend: self.backend, _marker: PhantomData }
    }
}
```

---

## Layer Trait (in `anyfs-backend`)

The `Layer` trait (inspired by Tower) standardizes middleware composition:

```rust
/// A layer that wraps a backend to add functionality.
pub trait Layer<B: VfsBackend> {
    type Backend: VfsBackend;
    fn layer(self, backend: B) -> Self::Backend;
}
```

Each middleware provides a corresponding `Layer` implementation:

```rust
// QuotaLayer, TracingLayer, RestrictionsLayer, etc.
pub struct QuotaLayer { /* config */ }

impl<B: VfsBackend> Layer<B> for QuotaLayer {
    type Backend = Quota<B>;
    fn layer(self, backend: B) -> Self::Backend {
        Quota::new(backend).with_limits(self.limits)
    }
}
```

---

## Composing Middleware

Middleware composes by wrapping. Order matters - innermost applies first.

### Manual Composition

```rust
use anyfs::{SqliteBackend, Quota, Restrictions, Tracing};
use anyfs_container::FileStorage;

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
use anyfs_container::BackendStack;

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
| `VRootFsBackend` | `vrootfs` | Host filesystem via strict-path |

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

Path resolution (walking directory structure, following symlinks) operates on the **`VfsBackend` abstraction**, not reimplemented per-backend.

### Why Abstract Path Resolution?

We simulate inodes - that's the whole point of virtualizing a filesystem. Path resolution must work on that abstraction:

- `/foo/../bar` cannot be resolved lexically - `foo` might be a symlink to `/other/place`, making `..` resolve to `/other`
- Resolution requires following the actual directory structure (inodes)
- The `VfsBackend` trait has the needed methods: `metadata()`, `read_link()`, `read_dir()`

### Resolution Utility (in `anyfs`)

```rust
/// Resolve a path, optionally following symlinks.
/// Works on any VfsBackend.
pub fn resolve_path(
    backend: &impl VfsBackend,
    path: impl AsRef<Path>,
    follow_symlinks: bool,
) -> Result<PathBuf, VfsError> {
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

Virtual backends need resolution by default. Real filesystem backends opt out:

```rust
pub trait VfsBackend: Send {
    /// Whether this backend needs virtual path resolution.
    /// Returns `false` for backends where the OS handles resolution.
    const NEEDS_PATH_RESOLUTION: bool = true;

    // ... existing methods
}

impl VfsBackend for VRootFsBackend {
    const NEEDS_PATH_RESOLUTION: bool = false;
    // ...
}
```

`FileStorage` (or a dedicated wrapper) applies resolution automatically for backends that need it:

```rust
impl<M> FileStorage<M> {
    pub fn new(backend: impl VfsBackend + 'static) -> FileStorage {
        // Resolution applied automatically if backend needs it
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

The `VfsBackendExt` trait provides convenience methods without modifying `VfsBackend`:

```rust
use serde::{Serialize, de::DeserializeOwned};

/// Extension methods for VfsBackend (auto-implemented for all backends).
pub trait VfsBackendExt: VfsBackend {
    /// Read and deserialize JSON.
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, VfsError> {
        let bytes = self.read(path)?;
        serde_json::from_slice(&bytes).map_err(|e| VfsError::Deserialization(e.to_string()))
    }

    /// Serialize and write JSON.
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), VfsError> {
        let bytes = serde_json::to_vec(value).map_err(|e| VfsError::Serialization(e.to_string()))?;
        self.write(path, &bytes)
    }

    /// Check if path is a file.
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        self.metadata(path).map(|m| m.file_type == FileType::File)
    }

    /// Check if path is a directory.
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        self.metadata(path).map(|m| m.file_type == FileType::Directory)
    }
}

// Blanket implementation for all backends
impl<B: VfsBackend> VfsBackendExt for B {}
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

`VfsError` includes context for better debugging:

```rust
pub enum VfsError {
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

    /// Serialization error (from VfsBackendExt).
    Serialization(String),

    /// Deserialization error (from VfsBackendExt).
    Deserialization(String),

    /// Backend-specific error.
    Backend(String),

    /// I/O error.
    Io(std::io::Error),
}
```

All error variants include enough context for meaningful error messages.
