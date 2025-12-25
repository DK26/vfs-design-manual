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
use anyfs::{SqliteBackend, Quota, PathFilter, FeatureGuard, Tracing};
use anyfs_container::FilesContainer;

// Compose middleware stack
let backend = Tracing::new(
    PathFilter::new(
        FeatureGuard::new(
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
| **KV store** | No | No | Yes | No |

---

## 3. Middleware Stack

AnyFS middleware can **intercept, transform, and control** operations:

| Middleware | Intercepts | Action |
|------------|------------|--------|
| `Quota` | Writes | Reject if over limit |
| `PathFilter` | All ops | Block denied paths |
| `FeatureGuard` | `symlink()`, `hard_link()`, etc. | Block disabled operations |
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

## 6. Tradeoffs

### AnyFS Advantages
- Composable middleware pattern
- Backend-agnostic
- Third-party extensibility
- Clean separation of concerns

### AnyFS Limitations
- Sync-first (async planned)
- Smaller ecosystem (new project)
- Not full POSIX emulation

---

If this document conflicts with `AGENTS.md` or `src/architecture/design-overview.md`, treat those as authoritative.
