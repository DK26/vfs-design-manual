# AGENTS.md - Instructions for AI Assistants

READ THIS FIRST before making any changes to this repository.

---

## Project Overview

AnyFS is an open standard for pluggable virtual filesystem backends in Rust. It uses a **middleware/decorator pattern** (like Axum/Tower) for composable functionality.

**Key design principle:** Complete separation of concerns via composable layers.

---

## Architecture (Tower-style Middleware)

```
┌─────────────────────────────────────────┐
│  FileStorage<B, M>                      │  ← Ergonomics + type-safe marker
├─────────────────────────────────────────┤
│  Middleware (optional, composable):     │
│                                         │
│  Policy:        Quota, Restrictions,    │
│                 PathFilter, ReadOnly,   │
│                 RateLimit               │
│                                         │
│  Observability: Tracing, DryRun         │
│                                         │
│  Performance:   Cache                   │
│                                         │
│  Composition:   Overlay                 │
├─────────────────────────────────────────┤
│  Backend (implements Fs, FsFull,        │  ← Pure storage + fs semantics
│           FsFuse, or FsPosix)           │
│  (Memory, SQLite, VRootFs, custom...)   │
└─────────────────────────────────────────┘
```

Each layer has **exactly one responsibility**:

| Layer | Responsibility |
|-------|----------------|
| Backend (`Fs`+) | Storage + filesystem semantics |
| `Quota<B>` | Resource limits (size, count, depth) |
| `Restrictions<B>` | Opt-in operation restrictions |
| `PathFilter<B>` | Path-based access control (sandbox) |
| `ReadOnly<B>` | Prevent all write operations |
| `RateLimit<B>` | Limit operations per second |
| `Tracing<B>` | Instrumentation / audit trail |
| `DryRun<B>` | Log operations without executing |
| `Cache<B>` | LRU cache for reads |
| `Overlay<B1,B2>` | Union filesystem (base + upper) |
| `FileStorage<B,M>` | Ergonomic std::fs-aligned API + type-safe marker |

---

## Crate Structure (2 Crates)

```
anyfs-backend/              # Crate 1: traits + types (no dependencies)
  src/
    lib.rs
    traits/
      fs_read.rs            # FsRead trait
      fs_write.rs           # FsWrite trait
      fs_dir.rs             # FsDir trait
      fs_link.rs            # FsLink trait
      fs_permissions.rs     # FsPermissions trait
      fs_sync.rs            # FsSync trait
      fs_stats.rs           # FsStats trait
      fs_inode.rs           # FsInode trait
      fs_handles.rs         # FsHandles trait
      fs_lock.rs            # FsLock trait
      fs_xattr.rs           # FsXattr trait
    layer.rs                # Layer trait (Tower-style)
    ext.rs                  # FsExt (extension methods)
    types.rs                # Metadata, DirEntry, Permissions, StatFs
    error.rs                # FsError

anyfs/                      # Crate 2: backends + middleware + FileStorage
  src/
    lib.rs
    backends/
      memory.rs             # MemoryBackend
      sqlite.rs             # SqliteBackend
      stdfs.rs              # StdFsBackend (no containment)
      vrootfs.rs            # VRootFsBackend (with containment)
    middleware/
      quota.rs              # Quota<B>
      restrictions.rs       # Restrictions<B>
      path_filter.rs        # PathFilter<B>
      read_only.rs          # ReadOnly<B>
      rate_limit.rs         # RateLimit<B>
      tracing.rs            # Tracing<B>
      dry_run.rs            # DryRun<B>
      cache.rs              # Cache<B>
      overlay.rs            # Overlay<B1, B2>
    container.rs            # FileStorage<B, M>
```

---

## Trait Hierarchy

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

**Rule:** Implement the lowest level you need. Higher levels include all below.

### Core Traits (FsRead + FsWrite + FsDir = Fs)

```rust
pub trait FsRead: Send {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, FsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, FsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError>;
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError>;
}

pub trait FsWrite: Send {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), FsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, FsError>;
}

pub trait FsDir: Send {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, FsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
}

/// Basic filesystem - covers 90% of use cases
pub trait Fs: FsRead + FsWrite + FsDir {}
impl<T: FsRead + FsWrite + FsDir> Fs for T {}
```

### Extended Traits

```rust
pub trait FsLink: Send {
    fn symlink(&mut self, target: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError>;
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

/// Full filesystem with all std::fs features
pub trait FsFull: Fs + FsLink + FsPermissions + FsSync + FsStats {}
impl<T: Fs + FsLink + FsPermissions + FsSync + FsStats> FsFull for T {}
```

---

## FileStorage<B, M> (Zero-Cost Wrapper)

```rust
pub struct FileStorage<B, M = ()> {
    backend: B,
    _marker: PhantomData<M>,
}

impl<B: Fs, M> FileStorage<B, M> {
    pub fn new(backend: B) -> Self;
    pub fn boxed(self) -> FileStorage<Box<dyn Fs>, M>;  // Opt-in type erasure
}
```

**Marker types for compile-time safety:**

```rust
struct Sandbox;
struct UserData;

let sandbox: FileStorage<_, Sandbox> = FileStorage::new(MemoryBackend::new());
let userdata: FileStorage<_, UserData> = FileStorage::new(SqliteBackend::open("data.db")?);

fn process_sandbox(fs: &FileStorage<impl Fs, Sandbox>) { /* only accepts Sandbox */ }
```

---

## Built-in Backends

| Backend | Description |
|---------|-------------|
| `MemoryBackend` | In-memory storage, implements `Clone` for snapshots |
| `SqliteBackend` | Single-file portable database |
| `SqliteCipherBackend` | Encrypted SQLite via SQLCipher (AES-256) |
| `StdFsBackend` | Direct `std::fs` delegation (no containment) |
| `VRootFsBackend` | Host filesystem with path containment (via strict-path) |

### MemoryBackend Snapshots

```rust
// Snapshot = Clone
let checkpoint = fs.clone();

// Rollback = replace
fs = checkpoint;

// Persistence
fs.save_to("state.bin")?;
let fs = MemoryBackend::load_from("state.bin")?;
```

---

## Middleware

### Restrictions<B> (replaces FeatureGuard)

Blocks specific operations when needed. **Default: all operations work.**

```rust
let backend = Restrictions::new(backend)
    .deny_symlinks()       // Block symlink() calls
    .deny_hard_links()     // Block hard_link() calls
    .deny_permissions();   // Block set_permissions() calls
```

### Other Middleware

| Middleware | Purpose |
|------------|---------|
| `Quota<B>` | Resource limits (size, count, depth) |
| `PathFilter<B>` | Path-based access control |
| `ReadOnly<B>` | Prevent all writes |
| `RateLimit<B>` | Ops/sec limit |
| `Tracing<B>` | Instrumentation |
| `DryRun<B>` | Test mode |
| `Cache<B>` | LRU caching |
| `Overlay<B1,B2>` | Layered FS |

---

## Cross-Platform Compatibility

| Backend | Windows | Linux | macOS | WASM |
|---------|:-------:|:-----:|:-----:|:----:|
| `MemoryBackend` | ✅ | ✅ | ✅ | ✅ |
| `SqliteBackend` | ✅ | ✅ | ✅ | ✅* |
| `VRootFsBackend` | ✅ | ✅ | ✅ | ❌ |

**Virtual backends (Memory, SQLite) are fully cross-platform** - all features work identically because paths are just keys, symlinks are stored data, permissions are metadata.

**VRootFsBackend has platform limitations:**
- Windows: symlinks need privileges, permissions have limited mapping

---

## Virtual Drive Mounting

Backends implementing `FsFuse` can be mounted as real filesystem drives via `anyfs-mount`:

| Platform | Technology | Required |
|----------|------------|----------|
| Linux | FUSE | Usually pre-installed |
| macOS | macFUSE | User must install |
| Windows | WinFsp | User must install |

```rust
use anyfs_mount::MountHandle;

let mount = MountHandle::mount(backend, "/mnt/drive")?;
// Now /mnt/drive is a real mount point any app can use
```

See `guides/mounting.md` for full details.

---

## Usage Examples

### Simple

```rust
let mut fs = FileStorage::new(MemoryBackend::new());
fs.write("/hello.txt", b"Hello!")?;
```

### With Middleware

```rust
let backend = MemoryBackend::new()
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(RestrictionsLayer::new().deny_symlinks())
    .layer(TracingLayer::new());

let mut fs = FileStorage::new(backend);
```

### AI Agent Sandbox

```rust
let sandbox = MemoryBackend::new()
    .layer(QuotaLayer::new()
        .max_total_size(50 * 1024 * 1024)
        .max_file_size(5 * 1024 * 1024))
    .layer(PathFilterLayer::new()
        .allow("/workspace/**")
        .deny("**/.env"))
    .layer(RestrictionsLayer::new())
    .layer(RateLimitLayer::new().max_ops(1000).per_second())
    .layer(TracingLayer::new());
```

---

## Common Mistakes to Avoid

- Do NOT put policy logic in FileStorage - use middleware
- Do NOT use old names: `VfsBackend` → `Fs`, `FilesContainer` → `FileStorage`, `FeatureGuard` → `Restrictions`
- Middleware order matters: innermost applies first
- Use `FsExt` for convenience methods, don't add them to core traits

---

## When in Doubt

| Question | Answer |
|----------|--------|
| Where do limits go? | `Quota<B>` middleware |
| Where do feature restrictions go? | `Restrictions<B>` middleware |
| Where does logging go? | `Tracing<B>` middleware |
| Where does path filtering go? | `PathFilter<B>` middleware |
| What does FileStorage do? | Thin std::fs-aligned wrapper + type-safe marker |
| How to snapshot MemoryBackend? | `.clone()` or `.save_to()` |
| Sync or async? | Sync for v1, async-ready for future |

---

## Documentation Structure

All detailed documentation is in `src/` (mdbook):

| Section | Key Files |
|---------|-----------|
| Architecture | `architecture/design-overview.md` |
| Traits | `traits/layered-traits.md`, `traits/files-container.md` |
| Implementation | `implementation/backend-guide.md`, `implementation/testing-guide.md` |
| Security | `comparisons/security.md` |

---

## Implementation Requirements

**Non-negotiable:**

1. **No Panic Policy** - Always return `Result`, never `.unwrap()`
2. **Thread Safety** - `MemoryBackend` uses `Arc<RwLock<...>>`, `SqliteBackend` uses WAL mode
3. **Error Context** - Include path and operation in all errors
4. **Path Edge Cases** - Handle `/../`, `//`, empty paths, unicode, long paths
