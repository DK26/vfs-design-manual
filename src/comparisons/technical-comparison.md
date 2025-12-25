# AnyFS - Technical Comparison with Alternatives

This document compares AnyFS with existing Rust filesystem abstractions.

---

## Executive Summary

AnyFS is **to filesystems what Axum/Tower is to HTTP**: a composable middleware stack with pluggable backends.

**Key differentiators:**
- **Composable middleware** - Stack quota, sandboxing, tracing, caching as independent layers
- **Backend agnostic** - Swap Memory/SQLite/RealFS without code changes
- **Policy separation** - Storage logic separate from policy enforcement
- **Third-party extensibility** - Custom backends and middleware depend only on `anyfs-backend`

---

## Compared Solutions

| Solution | What it is | Middleware | Multiple Backends |
|----------|------------|:----------:|:-----------------:|
| `vfs` | VFS trait + backends | No | Yes |
| AgentFS | SQLite agent runtime | No | No (SQLite only) |
| OpenDAL | Object storage layer | Yes | Yes (cloud-focused) |
| **AnyFS** | VFS + middleware stack | **Yes** | **Yes** |

---

## 1. Architecture Comparison

### `vfs` Crate

Path-based trait, no middleware pattern:

```rust
pub trait FileSystem: Send + Sync {
    fn read_dir(&self, path: &str) -> VfsResult<Box<dyn Iterator<Item = String>>>;
    fn open_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndRead>>;
    fn create_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndWrite>>;
    // ...
}
```

**Limitations:**
- No standard way to add quotas, logging, sandboxing
- Each concern must be built into backends or wrapped externally
- Path validation is backend-specific

### AgentFS

SQLite-based agent runtime:

```rust
// Fixed to SQLite, includes KV store and tool auditing
let fs = AgentFS::open("agent.db")?;
fs.write_file("/path", data)?;
fs.kv_set("key", "value")?;  // KV store bundled
fs.toolcall_start("tool")?;  // Auditing bundled
```

**Limitations:**
- Locked to SQLite (no memory backend for testing, no real FS)
- Monolithic design (can't use FS without KV/auditing)
- No composable middleware

### AnyFS

Tower-style middleware + pluggable backends:

```rust
use anyfs::{SqliteBackend, Quota, PathFilter, Restrictions, Tracing};
use anyfs_container::FilesContainer;

// Compose middleware stack
let backend = Tracing::new(
    PathFilter::new(
        Restrictions::new(
            Quota::new(SqliteBackend::open("data.db")?)
                .with_max_total_size(100 * 1024 * 1024)
        )
    )
    .allow("/workspace/**")
    .deny("**/.env")
);

let mut fs = FilesContainer::new(backend);
```

**Advantages:**
- Add/remove middleware without touching backends
- Swap backends without touching middleware
- Third-party extensions via `anyfs-backend` trait

---

## 2. Feature Comparison

| Feature | AnyFS | `vfs` | AgentFS | OpenDAL |
|---------|:-----:|:-----:|:-------:|:-------:|
| **Middleware pattern** | Yes | No | No | Yes |
| **Multiple backends** | Yes | Yes | No | Yes |
| **SQLite backend** | Yes | No | Yes | No |
| **Memory backend** | Yes | Yes | No | Yes |
| **Real FS backend** | Yes | Yes | No | No |
| **Quota enforcement** | Middleware | Manual | No | No |
| **Path sandboxing** | Middleware | Manual | No | No |
| **Feature gating** | Middleware | No | No | No |
| **Rate limiting** | Middleware | No | No | No |
| **Tracing/logging** | Middleware | Manual | Built-in | Middleware |
| **Streaming I/O** | Yes | Yes | Yes | Yes |
| **Async API** | Future | Partial | No | Yes |
| **POSIX extension** | Future | No | No | No |
| **FUSE mountable** | Future | No | No | No |
| **KV store** | No | No | Yes | No |

---

## 3. Middleware Stack

AnyFS middleware can **intercept, transform, and control** operations:

| Middleware | Intercepts | Action |
|------------|------------|--------|
| `Quota` | Writes | Reject if over limit |
| `PathFilter` | All ops | Block denied paths |
| `Restrictions` | Configurable operations | Block via `.deny_*()` methods |
| `RateLimit` | All ops | Throttle per second |
| `ReadOnly` | Writes | Block all writes |
| `Tracing` | All ops | Log with tracing crate |
| `DryRun` | Writes | Log without executing |
| `Cache` | Reads | LRU caching |
| `Overlay` | All ops | Union filesystem |
| Custom | Any | Encryption, compression, ... |

---

## 4. Backend Trait

```rust
pub trait VfsBackend: Send {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;
    // ... 25 methods total, aligned with std::fs
}
```

**Design principles:**
- `impl AsRef<Path>` for ergonomics (accepts `&str`, `String`, `PathBuf`)
- Aligned with `std::fs` naming
- Streaming I/O via `open_read`/`open_write`
- `Send` bound for async compatibility

---

## 5. When to Use What

| Use Case | Recommendation |
|----------|----------------|
| Need composable middleware | **AnyFS** |
| Need backend flexibility | **AnyFS** |
| Need SQLite + Memory + RealFS | **AnyFS** |
| Need just VFS abstraction (no policies) | `vfs` |
| Need AI agent runtime with KV + auditing | AgentFS |
| Need cloud object storage | OpenDAL |
| Need async-first design | OpenDAL (or wait for AnyFS async) |

---

## 6. Deep Dive: `vfs` Crate Compatibility

The [`vfs`](https://github.com/manuel-woelker/rust-vfs) crate is the most similar project. This section details why we don't adopt their trait and how we'll provide interop.

### `vfs::FileSystem` Trait (Complete)

```rust
pub trait FileSystem: Send + Sync {
    // Required (9 methods)
    fn read_dir(&self, path: &str) -> VfsResult<Box<dyn Iterator<Item = String>>>;
    fn create_dir(&self, path: &str) -> VfsResult<()>;
    fn open_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndRead>>;
    fn create_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndWrite>>;
    fn append_file(&self, path: &str) -> VfsResult<Box<dyn SeekAndWrite>>;
    fn metadata(&self, path: &str) -> VfsResult<VfsMetadata>;
    fn exists(&self, path: &str) -> VfsResult<bool>;
    fn remove_file(&self, path: &str) -> VfsResult<()>;
    fn remove_dir(&self, path: &str) -> VfsResult<()>;

    // Optional - default to NotSupported (6 methods)
    fn set_creation_time(&self, path: &str, time: SystemTime) -> VfsResult<()>;
    fn set_modification_time(&self, path: &str, time: SystemTime) -> VfsResult<()>;
    fn set_access_time(&self, path: &str, time: SystemTime) -> VfsResult<()>;
    fn copy_file(&self, src: &str, dest: &str) -> VfsResult<()>;
    fn move_file(&self, src: &str, dest: &str) -> VfsResult<()>;
    fn move_dir(&self, src: &str, dest: &str) -> VfsResult<()>;
}
```

### Feature Gap Analysis

| Feature | `vfs` | AnyFS | Gap |
|---------|:-----:|:-----:|-----|
| Basic read/write | Yes | Yes | - |
| Directory ops | Yes | Yes | - |
| Streaming I/O | Yes | Yes | - |
| `rename` | `move_file` | Yes | - |
| `copy` | `copy_file` | Yes | - |
| **Symlinks** | No | Yes | Critical |
| **Hard links** | No | Yes | Critical |
| **Permissions** | No | Yes | Critical |
| **truncate** | No | Yes | Missing |
| **sync/fsync** | No | Yes | Missing |
| **statfs** | No | Yes | Missing |
| **read_range** | No | Yes | Missing |
| **symlink_metadata** | No | Yes | Missing |
| Path type | `&str` | `impl AsRef<Path>` | Different |
| Middleware | No | Yes | Architectural |

### Why Not Adopt Their Trait?

1. **No symlinks/hardlinks** - Can't virtualize real filesystem semantics
2. **No permissions** - Our `Restrictions` middleware needs `set_permissions` to gate
3. **No durability primitives** - No `sync`/`fsync` for data integrity
4. **No middleware pattern** - Their `VfsPath` bakes in behaviors we want composable
5. **`&str` paths** - We prefer `impl AsRef<Path>` for ergonomics

**Our trait is a strict superset.** Everything `vfs` can do, we can do. The reverse is not true.

### `vfs` Backends

| vfs Backend | AnyFS Equivalent | Notes |
|-------------|------------------|-------|
| `PhysicalFS` | `VRootFsBackend` | Both use real filesystem |
| `MemoryFS` | `MemoryBackend` | Both in-memory |
| `OverlayFS` | `Overlay<B1,B2>` | Both union filesystems |
| `AltrootFS` | `PathFilter` (partial) | vfs is simpler chroot |
| `EmbeddedFS` | (none) | Read-only embedded assets |
| (none) | `SqliteBackend` | We have SQLite |

### Interoperability Plan

Future `anyfs-vfs-compat` crate provides bidirectional adapters:

```rust
use anyfs_vfs_compat::{VfsCompat, AnyFsCompat};

// Use a vfs backend in AnyFS
// Missing features return VfsError::NotSupported
let backend = VfsCompat::new(vfs::MemoryFS::new());
let fs = FilesContainer::new(backend);

// Use an AnyFS backend in vfs-based code
// Only exposes what vfs supports
let anyfs_backend = MemoryBackend::new();
let vfs_fs: Box<dyn vfs::FileSystem> = Box::new(AnyFsCompat::new(anyfs_backend));
```

**Use cases:**
- Migrate from `vfs` to AnyFS incrementally
- Use `vfs::EmbeddedFS` in AnyFS (read-only embedded assets)
- Use AnyFS backends in projects depending on `vfs`

---

## 7. Tradeoffs

### AnyFS Advantages
- Composable middleware pattern
- Backend-agnostic
- Third-party extensibility
- Clean separation of concerns
- Full filesystem semantics (symlinks, permissions, durability)

### AnyFS Limitations
- Sync-first (async planned)
- Smaller ecosystem (new project)
- Not full POSIX emulation

---

If this document conflicts with `AGENTS.md` or `src/architecture/design-overview.md`, treat those as authoritative.
