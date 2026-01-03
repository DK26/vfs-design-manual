# AGENTS.md - Instructions for AI Assistants

READ THIS FIRST before making any changes to this repository.

---

## Project Overview

> **This repository contains the design manual only.**
> The AnyFS crates (`anyfs-backend`, `anyfs`) are not yet implemented.
> All documentation describes the intended design and API.

AnyFS is an open standard for pluggable virtual filesystem backends in Rust. It uses a **middleware/decorator pattern** (like Axum/Tower) for composable functionality.

**Key design principle:** Complete separation of concerns via composable layers.

**Development methodology:** LLM-Oriented Architecture (LOA) - every component is independently understandable, testable, and fixable with only local context. See [ADR-034](src/architecture/adrs.md#adr-034-llm-oriented-architecture-loa) and the [LLM Development Methodology Guide](src/guides/llm-development-methodology.md).

---

## LLM-Oriented Architecture (Quick Reference)

AnyFS is structured for **AI-assisted development**. Each component should be:

| Property       | Meaning                | How to Verify                      |
| -------------- | ---------------------- | ---------------------------------- |
| **Isolated**   | One file = one concept | File has single responsibility     |
| **Contracted** | Trait defines the spec | Trait doc has invariants           |
| **Testable**   | Tests use mocks only   | No real backends in unit tests     |
| **Debuggable** | Errors explain the fix | Error has path, operation, context |
| **Documented** | Examples at every API  | Doc comment has usage example      |

### Before Making Changes

1. **Read only the component file** + the trait it implements
2. **Run only the component's tests** to verify your change
3. **Keep changes within one file** when possible
4. **Make error messages self-explanatory** - include what failed and why

### File Structure Convention

Every implementation file follows this structure:
```rust
//! # Component Name
//! Brief description.
//! ## Responsibility - Single bullet
//! ## Dependencies - Traits/types only  
//! ## Usage - Minimal example

// Types → Trait Impls → Public API → Private Helpers → Tests
```

See [LLM Development Methodology](src/guides/llm-development-methodology.md) for complete details.

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

| Layer              | Responsibility                                   |
| ------------------ | ------------------------------------------------ |
| Backend (`Fs`+)    | Storage + filesystem semantics                   |
| `Quota<B>`         | Resource limits (size, count, depth)             |
| `Restrictions<B>`  | Block permission changes                         |
| `PathFilter<B>`    | Path-based access control (sandbox)              |
| `ReadOnly<B>`      | Prevent all write operations                     |
| `RateLimit<B>`     | Fixed-window rate limiting                       |
| `Tracing<B>`       | Instrumentation / audit trail                    |
| `DryRun<B>`        | Log operations without executing                 |
| `Cache<B>`         | LRU cache for reads                              |
| `Overlay<B1,B2>`   | Union filesystem (base + upper)                  |
| `FileStorage<B,M>` | Ergonomic std::fs-aligned API + type-safe marker |

---

## Crate Structure (2 Crates)

```
anyfs-backend/              # Crate 1: traits + types (minimal deps: thiserror; optional: serde)
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
      fs_path.rs            # FsPath trait (canonicalization, blanket impl)
      fs_inode.rs           # FsInode trait
      fs_handles.rs         # FsHandles trait
      fs_lock.rs            # FsLock trait
      fs_xattr.rs           # FsXattr trait
    layer.rs                # Layer trait (Tower-style)
    ext.rs                  # FsExt (extension methods)
    markers.rs              # SelfResolving marker trait
    path_resolver.rs        # PathResolver trait (pluggable path resolution)
    types.rs                # Metadata, DirEntry, Permissions, StatFs
    error.rs                # FsError

anyfs/                      # Crate 2: backends + middleware + FileStorage
  src/
    lib.rs
    backends/
      memory.rs             # MemoryBackend
      sqlite.rs             # SqliteBackend
      sqlite_cipher.rs      # SqliteCipherBackend (feature: sqlite-cipher)
      indexed.rs            # IndexedBackend (feature: indexed)
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
    resolvers/
      iterative.rs          # IterativeResolver (default)
      noop.rs               # NoOpResolver (for SelfResolving backends)
      case_folding.rs       # CaseFoldingResolver (case-insensitive)
      caching.rs            # CachingResolver (LRU cache wrapper)
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

Derived traits (auto-implemented):
  FsPath: FsRead + FsLink  ← Path canonicalization

Marker traits:
  SelfResolving  ← Opt-out of FileStorage path resolution

Strategy traits:
  PathResolver   ← Pluggable path resolution algorithm
```

**Rule:** Implement the lowest level you need. Higher levels include all below.

### Core Traits (FsRead + FsWrite + FsDir = Fs)

> **Thread Safety:** All traits require `Send + Sync` and use `&self` for all methods.
> Backends MUST use interior mutability (`RwLock`, `Mutex`) for thread-safe concurrent access.
> See ADR-023 for rationale.
>
> **Path Parameters:** Core traits use `&Path` so they are object-safe (`dyn Fs` works). `FileStorage`/`FsExt` provide `impl AsRef<Path>` ergonomics.

```rust
pub trait FsRead: Send + Sync {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError>;
    fn read_to_string(&self, path: &Path) -> Result<String, FsError>;
    fn read_range(&self, path: &Path, offset: u64, len: usize) -> Result<Vec<u8>, FsError>;
    fn exists(&self, path: &Path) -> Result<bool, FsError>;
    fn metadata(&self, path: &Path) -> Result<Metadata, FsError>;
    fn open_read(&self, path: &Path) -> Result<Box<dyn Read + Send>, FsError>;
}

pub trait FsWrite: Send + Sync {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError>;
    fn append(&self, path: &Path, data: &[u8]) -> Result<(), FsError>;
    fn remove_file(&self, path: &Path) -> Result<(), FsError>;
    fn rename(&self, from: &Path, to: &Path) -> Result<(), FsError>;
    fn copy(&self, from: &Path, to: &Path) -> Result<(), FsError>;
    fn truncate(&self, path: &Path, size: u64) -> Result<(), FsError>;
    fn open_write(&self, path: &Path) -> Result<Box<dyn Write + Send>, FsError>;
}

pub trait FsDir: Send + Sync {
    fn read_dir(&self, path: &Path) -> Result<ReadDirIter, FsError>;
    fn create_dir(&self, path: &Path) -> Result<(), FsError>;
    fn create_dir_all(&self, path: &Path) -> Result<(), FsError>;
    fn remove_dir(&self, path: &Path) -> Result<(), FsError>;
    fn remove_dir_all(&self, path: &Path) -> Result<(), FsError>;
}

/// Basic filesystem - covers 90% of use cases
pub trait Fs: FsRead + FsWrite + FsDir {}
impl<T: FsRead + FsWrite + FsDir> Fs for T {}
```

### Extended Traits

```rust
pub trait FsLink: Send + Sync {
    fn symlink(&self, target: &Path, link: &Path) -> Result<(), FsError>;
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

| Backend               | Description                                                    |
| --------------------- | -------------------------------------------------------------- |
| `MemoryBackend`       | In-memory storage, implements `Clone` for snapshots            |
| `SqliteBackend`       | Single-file portable database                                  |
| `SqliteCipherBackend` | Encrypted SQLite via SQLCipher (AES-256)                       |
| `IndexedBackend`      | Virtual paths + disk blobs (large file support with isolation) |
| `StdFsBackend`        | Direct `std::fs` delegation (no containment)                   |
| `VRootFsBackend`      | Host filesystem with path containment (via strict-path)        |

> **Backend vs Middleware:** A backend is WHERE data lives. Middleware WRAPS a backend to add behavior.
> `IndexedBackend` is a backend (stores virtual paths in SQLite index + file content as disk blobs).
> `Indexing<B>` middleware (different thing) adds queryable audit trail to ANY backend.

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

### Restrictions<B> (Runtime Policy)

Blocks specific operations at runtime. **Default: all operations work if backend supports them.**

> **Note:** Symlink/hard-link capability is determined by whether the backend implements `FsLink`.
> If `B: FsLink`, symlinks work. If not, they don't. No middleware needed.

```rust
let backend = Restrictions::new(backend)
    .deny_permissions();   // Block set_permissions() calls
```

**Symlink support:** Determined by trait bounds, not middleware.
- `MemoryBackend: FsLink` → supports symlinks
- `SqliteBackend: FsLink` → supports symlinks  
- Custom backend without `FsLink` → no symlinks (compile-time enforced)

### Other Middleware

| Middleware       | Purpose                              |
| ---------------- | ------------------------------------ |
| `Quota<B>`       | Resource limits (size, count, depth) |
| `PathFilter<B>`  | Path-based access control            |
| `ReadOnly<B>`    | Prevent all writes                   |
| `RateLimit<B>`   | Ops/sec limit                        |
| `Tracing<B>`     | Instrumentation                      |
| `DryRun<B>`      | Test mode                            |
| `Cache<B>`       | LRU caching                          |
| `Overlay<B1,B2>` | Layered FS                           |

---

## Cross-Platform Compatibility

| Backend          | Windows | Linux | macOS | WASM  |
| ---------------- | :-----: | :---: | :---: | :---: |
| `MemoryBackend`  |    ✅    |   ✅   |   ✅   |   ✅   |
| `SqliteBackend`  |    ✅    |   ✅   |   ✅   |  ✅*   |
| `VRootFsBackend` |    ✅    |   ✅   |   ✅   |   ❌   |

**Virtual backends (Memory, SQLite) are fully cross-platform** - all features work identically because paths are just keys, symlinks are stored data, permissions are metadata.

**VRootFsBackend has platform limitations:**
- Windows: symlinks need privileges, permissions have limited mapping

---

## Virtual Drive Mounting

Backends implementing `FsFuse` can be mounted as real filesystem drives via `anyfs-mount`:

| Platform | Technology | Required              |
| -------- | ---------- | --------------------- |
| Linux    | FUSE       | Usually pre-installed |
| macOS    | macFUSE    | User must install     |
| Windows  | WinFsp     | User must install     |

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
let fs = FileStorage::new(MemoryBackend::new());
fs.write("/hello.txt", b"Hello!")?;
```

### With Middleware

```rust
let backend = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(TracingLayer::new());

let fs = FileStorage::new(backend);
```

### AI Agent Sandbox

```rust
let sandbox = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(50 * 1024 * 1024)
        .max_file_size(5 * 1024 * 1024)
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .deny("**/.env")
        .build())
    .layer(RateLimitLayer::builder()
        .max_ops(1000)
        .per_second()
        .build())
    .layer(TracingLayer::new());
```

---

## Common Mistakes to Avoid

- Do NOT put policy logic in FileStorage - use middleware
- Do NOT use old names: `VfsBackend` → `Fs`, `FilesContainer` → `FileStorage`, `FeatureGuard` → `Restrictions`
- Middleware order matters: innermost applies first
- Use `FsExt` for convenience methods, don't add them to core traits
- Do NOT run `mdbook build` - the user or CI will build, you only edit `src/` files

---

## When in Doubt

| Question                          | Answer                                              |
| --------------------------------- | --------------------------------------------------- |
| Where do limits go?               | `Quota<B>` middleware                               |
| Where do feature restrictions go? | `Restrictions<B>` middleware                        |
| Where does logging go?            | `Tracing<B>` middleware                             |
| Where does path filtering go?     | `PathFilter<B>` middleware                          |
| Where does path resolution go?    | `PathResolver` strategy (pluggable via FileStorage) |
| What does FileStorage do?         | Thin std::fs-aligned wrapper + type-safe marker     |
| How to snapshot MemoryBackend?    | `.clone()` or `.save_to()`                          |
| Sync or async?                    | Sync for v1, async-ready for future                 |

---

## Documentation Structure

All detailed documentation is in `src/` (mdbook):

| Section        | Key Files                                                            |
| -------------- | -------------------------------------------------------------------- |
| Architecture   | `architecture/design-overview.md`                                    |
| Traits         | `traits/layered-traits.md`, `traits/files-container.md`              |
| Implementation | `implementation/backend-guide.md`, `implementation/testing-guide.md` |
| Security       | `comparisons/security.md`                                            |

---

## Performance: Strategic Boxing (ADR-025)

**We follow Tower/Axum's approach: zero-cost on hot path, box at boundaries.**

```
HOT PATH (zero-cost):
  read(), write(), metadata(), exists()     ← Concrete types
  Middleware: Quota<Tracing<B>>             ← Generics, monomorphized

COLD PATH (boxed, I/O dominates):
  open_read() → Box<dyn Read>               ← Once per file open
  open_write() → Box<dyn Write>             ← Once per file open
  read_dir() → ReadDirIter                  ← Boxed inner iterator

OPT-IN TYPE ERASURE:
  FileStorage::boxed()                      ← Like Tower's BoxService
```

**Why:** Box allocation (~50ns) is <1% of actual I/O time. Enables middleware flexibility (QuotaWriter, PathFilter filtering, Overlay merging) without type explosion.

**Do NOT:** Add boxing to `read()`, `write()`, `metadata()`, or middleware composition.

---

## Implementation Requirements

**Non-negotiable:**

1. **No Panic Policy** - Always return `Result`, never `.unwrap()`
2. **Thread Safety** - `MemoryBackend` uses `Arc<RwLock<...>>`, `SqliteBackend` uses WAL mode
3. **Error Context** - Include path and operation in all errors
4. **Path Edge Cases** - Handle `/../`, `//`, empty paths, unicode, long paths

---

## Critical: Reviewer & AI Guidelines

**Before suggesting removal or renaming of any backend, middleware, or API:**

1. **Understand the difference between backends and middleware:**
   - Backend = WHERE data is stored (MemoryBackend, SqliteBackend, IndexedBackend, etc.)
   - Middleware = WRAPS a backend to add behavior (Quota, Tracing, Indexing, etc.)
   - These are NOT interchangeable. You cannot replace a backend with middleware.

2. **IndexedBackend vs Indexing<B> middleware - DIFFERENT THINGS:**
   - `IndexedBackend` = A backend that stores virtual paths in SQLite + file content as disk blobs
   - `Indexing<B>` = Middleware that adds queryable audit trail to any backend
   - One stores data. The other observes operations. Not the same.

3. **Do NOT remove backends because "it can be done with middleware":**
   - If the component defines WHERE data lives, it's a backend
   - Middleware cannot create new storage locations

4. **Naming consistency:**
   - Use `VRootFsBackend` (not `VirtualRootBackend`)
   - Use `IndexedBackend` (not `HybridBackend` or `MappedBackend`)
   - Check AGENTS.md Built-in Backends table for canonical names

5. **AnyFS makes promises and delivers:**
   - Every backend supports ALL filesystem operations (read, write, streaming, large files)
   - Performance tradeoffs exist, but functionality is complete
   - Do NOT claim "X isn't designed for Y" - if it's in the API, it works

6. **Do NOT run `mdbook build`** - the user or CI will build, you only edit `src/` files

