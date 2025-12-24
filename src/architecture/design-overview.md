# AnyFS - Design Overview

**Status:** Current
**Last updated:** 2025-12-24

---

## What This Project Is

AnyFS is an **open standard** for pluggable virtual filesystem backends in Rust. It uses a **middleware/decorator pattern** (like Axum/Tower) for composable functionality with complete separation of concerns.

Anyone can:
- Implement a custom backend for their storage needs
- Compose middleware to add limits, logging, feature gates
- Use the ergonomic `FilesContainer` wrapper

---

## Architecture (Tower-style Middleware)

```
┌─────────────────────────────────────────┐
│  FilesContainer<B>                      │  ← Ergonomics only (std::fs API)
├─────────────────────────────────────────┤
│  Middleware (optional, composable):     │
│    LimitedBackend<B>                    │  ← Quota enforcement
│    LoggingBackend<B>                    │  ← Audit logging
│    FeatureGatedBackend<B>               │  ← Symlink/hardlink whitelist
├─────────────────────────────────────────┤
│  VfsBackend                             │  ← Pure storage + fs semantics
│  (Memory, SQLite, VRootFs, custom...)   │
└─────────────────────────────────────────┘
```

**Each layer has exactly one responsibility:**

| Layer | Responsibility |
|-------|----------------|
| `VfsBackend` | Storage + filesystem semantics |
| `LimitedBackend<B>` | Quota enforcement |
| `LoggingBackend<B>` | Audit trail |
| `FeatureGatedBackend<B>` | Feature whitelist |
| `FilesContainer<B>` | Ergonomic std::fs-aligned API |

---

## Crates

| Crate | Purpose | Contains |
|-------|---------|----------|
| `anyfs-backend` | Minimal contract | `VfsBackend` trait + types |
| `anyfs` | Backends + middleware | Built-in backends, `LimitedBackend`, `LoggingBackend`, `FeatureGatedBackend` |
| `anyfs-container` | Ergonomic wrapper | `FilesContainer<B>` (thin std::fs wrapper) |

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
}
```

---

## Middleware (in `anyfs`)

Each middleware is itself a `VfsBackend` that wraps another backend. This enables composition.

### LimitedBackend<B>

Enforces quota limits. Tracks usage and rejects operations that would exceed limits.

```rust
use anyfs::{SqliteBackend, LimitedBackend};

let backend = LimitedBackend::new(SqliteBackend::open("data.db")?)
    .with_max_total_size(100 * 1024 * 1024)   // 100 MB
    .with_max_file_size(10 * 1024 * 1024)     // 10 MB per file
    .with_max_node_count(10_000)               // 10K files/dirs
    .with_max_dir_entries(1_000)               // 1K entries per dir
    .with_max_path_depth(64);

// Check usage
let usage = backend.usage();
let remaining = backend.remaining();
```

### FeatureGatedBackend<B>

Enforces least-privilege by disabling features by default.

```rust
use anyfs::{MemoryBackend, FeatureGatedBackend};

let backend = FeatureGatedBackend::new(MemoryBackend::new())
    .with_symlinks()                          // Enable symlink operations
    .with_max_symlink_resolution(40)          // Max symlink hops
    .with_hard_links()                        // Enable hard links
    .with_permissions();                      // Enable set_permissions
```

When a feature is disabled, operations return `VfsError::FeatureNotEnabled`.

### LoggingBackend<B>

Records all operations for audit trails.

```rust
use anyfs::{SqliteBackend, LoggingBackend};

let backend = LoggingBackend::new(SqliteBackend::open("data.db")?)
    .with_logger(MyLogger::new());
```

---

## FilesContainer (in `anyfs-container`)

`FilesContainer<B>` is a **thin ergonomic wrapper**. It provides a familiar std::fs-aligned API but does nothing else. All policy (limits, feature gates) is handled by middleware.

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

let mut fs = FilesContainer::new(MemoryBackend::new());

fs.create_dir_all("/documents")?;
fs.write("/documents/hello.txt", b"Hello!")?;
let content = fs.read("/documents/hello.txt")?;
```

---

## Composing Middleware

Middleware composes by wrapping. Order matters - innermost applies first.

```rust
use anyfs::{SqliteBackend, LimitedBackend, FeatureGatedBackend, LoggingBackend};
use anyfs_container::FilesContainer;

// Build from inside out:
let backend = SqliteBackend::open("data.db")?;

let limited = LimitedBackend::new(backend)
    .with_max_total_size(100 * 1024 * 1024);

let gated = FeatureGatedBackend::new(limited)
    .with_symlinks();

let logged = LoggingBackend::new(gated);

let mut fs = FilesContainer::new(logged);
```

Or use the layer helper for Axum-style composition:

```rust
let fs = FilesContainer::new(SqliteBackend::open("data.db")?)
    .layer(LimitedLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(FeatureGateLayer::new().allow_symlinks())
    .layer(LoggingLayer::new());
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

## Security Model

Security is achieved through composition:

| Concern | Solution |
|---------|----------|
| Path containment | Backend-specific (VRootFsBackend uses strict-path) |
| Resource exhaustion | `LimitedBackend` enforces quotas |
| Feature restriction | `FeatureGatedBackend` disables dangerous features |
| Audit trail | `LoggingBackend` records operations |
| Tenant isolation | Separate backend instances |

**Defense in depth:** Compose multiple middleware layers for comprehensive security.
