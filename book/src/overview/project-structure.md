# VFS Ecosystem — Project Structure

**Date:** 2025-12-22
**Status:** Current

---

## Crate Structure

```
┌─────────────────────────────────────────┐
│  Your App                               │  ← Consumer
│  (uses FilesContainer API)              │
├─────────────────────────────────────────┤
│  anyfs-container                        │  ← Crate 3: Isolation layer
│  (quotas, tenant isolation)             │
├─────────────────────────────────────────┤
│  anyfs                                  │  ← Crate 2: Built-in backends
├──────────┬──────────┬───────────────────┤
│ VRootFs  │  Memory  │  SQLite           │  ← Feature-gated
│ Backend  │  Backend │  Backend          │
├──────────┴──────────┴───────────────────┤
│  anyfs-traits                           │  ← Crate 1: Minimal trait + types
│  VfsBackend trait, VfsError, Metadata   │
├─────────────────────────────────────────┤
│  strict-path (external)                 │  ← VirtualPath, VirtualRoot
└─────────────────────────────────────────┘
```

### Dependency Graph

```
strict-path (external)
     ↑
anyfs-traits (trait + types)
     ↑
     ├── anyfs (re-exports traits, provides backends)
     │
     └── anyfs-container (wraps any VfsBackend)
```

---

# Crate 1: anyfs-traits

**Purpose:** Minimal crate containing only the trait and types. For custom backend implementers.

**Depends on:** `strict-path`, `thiserror`

## Crate Structure

```
anyfs-traits/
├── Cargo.toml          # minimal dependencies
└── src/
    ├── lib.rs          # re-exports VirtualPath from strict-path
    ├── backend.rs      # VfsBackend trait
    ├── types.rs        # Metadata, DirEntry, FileType
    └── error.rs        # VfsError
```

## Core Trait

```rust
// anyfs-traits/src/lib.rs
pub use strict_path::VirtualPath;

// anyfs-traits/src/backend.rs
use crate::VirtualPath;

/// A virtual filesystem backend.
///
/// Implementations provide storage; callers get uniform I/O.
pub trait VfsBackend: Send {
    // Read operations
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;
    fn list(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;

    // Write operations
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn mkdir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn mkdir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
}
```

## Types

```rust
#[derive(Clone, Debug)]
pub struct Metadata {
    pub file_type: FileType,
    pub size: u64,
    pub created: Option<SystemTime>,
    pub modified: Option<SystemTime>,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum FileType {
    File,
    Directory,
    Symlink,
}

#[derive(Clone, Debug)]
pub struct DirEntry {
    pub name: String,
    pub file_type: FileType,
}
```

## Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum VfsError {
    #[error("not found: {0}")]
    NotFound(VirtualPath),

    #[error("already exists: {0}")]
    AlreadyExists(VirtualPath),

    #[error("not a file: {0}")]
    NotAFile(VirtualPath),

    #[error("not a directory: {0}")]
    NotADirectory(VirtualPath),

    #[error("directory not empty: {0}")]
    DirectoryNotEmpty(VirtualPath),

    #[error("invalid path: {0}")]
    InvalidPath(String),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("backend error: {0}")]
    Backend(String),
}
```

---

# Crate 2: anyfs

**Purpose:** Built-in backends (feature-gated). Re-exports `anyfs-traits`.

**Depends on:** `anyfs-traits` + optional backend dependencies

## Cargo.toml

```toml
[package]
name = "anyfs"
version = "0.1.0"

[features]
default = ["memory"]
memory = []
vrootfs = ["strict-path"]
sqlite = ["rusqlite"]
all = ["memory", "vrootfs", "sqlite"]

[dependencies]
anyfs-traits = { version = "0.1", path = "../anyfs-traits" }

# Optional backend dependencies
strict-path = { version = "0.1", optional = true }
rusqlite = { version = "0.31", optional = true }
```

## Crate Structure

```
anyfs/
├── Cargo.toml
└── src/
    ├── lib.rs          # re-exports anyfs_traits::*
    ├── vrootfs/        # [feature: vrootfs] VRootFsBackend
    │   └── mod.rs
    ├── memory/         # [feature: memory] MemoryBackend
    │   └── mod.rs
    └── sqlite/         # [feature: sqlite] SqliteBackend
        ├── mod.rs
        └── schema.rs
```

## lib.rs

```rust
// Re-export everything from anyfs-traits
pub use anyfs_traits::*;

// Feature-gated backends
#[cfg(feature = "memory")]
mod memory;
#[cfg(feature = "memory")]
pub use memory::MemoryBackend;

#[cfg(feature = "vrootfs")]
mod vrootfs;
#[cfg(feature = "vrootfs")]
pub use vrootfs::VRootFsBackend;

#[cfg(feature = "sqlite")]
mod sqlite;
#[cfg(feature = "sqlite")]
pub use sqlite::SqliteBackend;
```

## Backend Implementations

### VRootFsBackend (feature: vrootfs)

```rust
use strict_path::VirtualRoot;
use anyfs_traits::{VfsBackend, VirtualPath, VfsError};

pub struct VRootFsBackend {
    root: VirtualRoot,
}

impl VRootFsBackend {
    pub fn new(root_dir: &Path) -> Result<Self, VfsError> {
        Ok(Self {
            root: VirtualRoot::new(root_dir)?,
        })
    }
}

impl VfsBackend for VRootFsBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        Ok(self.root.read(path)?)
    }
    // ... straightforward delegation to VirtualRoot
}
```

### MemoryBackend (feature: memory)

```rust
use anyfs_traits::{VfsBackend, VirtualPath, VfsError};
use std::collections::HashMap;

pub struct MemoryBackend {
    entries: HashMap<VirtualPath, Entry>,
}

enum Entry {
    File(Vec<u8>),
    Directory,
}

impl MemoryBackend {
    pub fn new() -> Self {
        let mut entries = HashMap::new();
        entries.insert(VirtualPath::root(), Entry::Directory);
        Self { entries }
    }
}

impl VfsBackend for MemoryBackend {
    // ... HashMap operations
}
```

### SqliteBackend (feature: sqlite)

```rust
use anyfs_traits::{VfsBackend, VirtualPath, VfsError};

pub struct SqliteBackend {
    conn: rusqlite::Connection,
}

impl SqliteBackend {
    pub fn open(path: &Path) -> Result<Self, VfsError>;
    pub fn open_in_memory() -> Result<Self, VfsError>;
    pub fn create(path: &Path) -> Result<Self, VfsError>;
}

impl VfsBackend for SqliteBackend {
    // Internal schema is implementation detail
}
```

---

# Crate 3: anyfs-container

**Purpose:** Tenant isolation and containment layer.

**Depends on:** `anyfs-traits` (NOT `anyfs` — doesn't need built-in backends)

## Crate Structure

```
anyfs-container/
├── Cargo.toml          # depends on anyfs-traits
└── src/
    ├── lib.rs
    ├── container.rs    # FilesContainer<B>
    ├── builder.rs      # ContainerBuilder
    ├── limits.rs       # CapacityLimits
    ├── usage.rs        # CapacityUsage tracking
    └── error.rs        # ContainerError
```

## FilesContainer

```rust
use anyfs_traits::{VfsBackend, VirtualPath, VfsError};
use std::path::Path;

/// A contained filesystem with quotas and isolation.
pub struct FilesContainer<B: VfsBackend> {
    backend: B,
    limits: CapacityLimits,
    usage: CapacityUsage,
}

impl<B: VfsBackend> FilesContainer<B> {
    /// User-facing API accepts any path-like type
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError> {
        let vpath = VirtualPath::new(path.as_ref())?;
        Ok(self.backend.read(&vpath)?)
    }

    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError> {
        let vpath = VirtualPath::new(path.as_ref())?;

        // Check limits BEFORE writing
        self.check_file_size(data.len())?;
        self.check_total_size(data.len())?;

        self.backend.write(&vpath, data)?;
        self.usage.add_bytes(data.len() as u64);
        Ok(())
    }

    // ... other operations with limit checks
}
```

## Builder

```rust
pub struct ContainerBuilder<B> {
    backend: B,
    limits: CapacityLimits,
}

impl<B: VfsBackend> ContainerBuilder<B> {
    pub fn new(backend: B) -> Self;
    pub fn max_total_size(mut self, bytes: u64) -> Self;
    pub fn max_file_size(mut self, bytes: u64) -> Self;
    pub fn build(self) -> FilesContainer<B>;
}
```

---

# Usage Examples

## App using built-in backend with container

```rust
use anyfs::{SqliteBackend, VRootFsBackend};
use anyfs_container::{FilesContainer, ContainerBuilder};

// Create tenant storage with quota
let backend = SqliteBackend::create("tenant_123.db")?;
let container = ContainerBuilder::new(backend)
    .max_total_size(100 * 1024 * 1024)  // 100 MB
    .build();

container.write("/uploads/file.txt", b"hello")?;
```

## Custom backend implementer

```rust
// Only depend on anyfs-traits — no rusqlite, no strict-path
use anyfs_traits::{VfsBackend, VirtualPath, VfsError, Metadata, DirEntry};

pub struct S3Backend {
    bucket: String,
    client: aws_sdk_s3::Client,
}

impl VfsBackend for S3Backend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        // Implement using S3 API
    }
    // ... implement other methods
}
```

---

# Summary

| Crate | Purpose | Depends On |
|-------|---------|------------|
| `anyfs-traits` | Minimal trait + types | `strict-path`, `thiserror` |
| `anyfs` | Built-in backends (feature-gated) | `anyfs-traits` |
| `anyfs-container` | Tenant isolation + quotas | `anyfs-traits` |
| Your App | Business logic | `anyfs-container` + `anyfs` |

**anyfs-traits** is the minimal foundation — trait and types only.
**anyfs** provides built-in backends for common use cases.
**anyfs-container** adds containment — quotas, limits, isolation.
