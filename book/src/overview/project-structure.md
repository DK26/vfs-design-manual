# AnyFS — Project Structure

**Date:** 2025-12-23
**Status:** Current

---

## Two-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  APPLICATION CODE                                           │
│                                                             │
│  container.write("/data/file.txt", b"hello")?;              │
│  container.read_dir("/data")?;                              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  anyfs-container: FilesContainer<V, S>                      │
│  ═══════════════════════════════════════                    │
│  • std::fs-like API (impl AsRef<Path>)                      │
│  • Path resolution via FsSemantics                          │
│  • Capacity enforcement (limits, quotas)                    │
│  • Target: Application developers                           │
├─────────────────────────────────────────────────────────────┤
│  anyfs: Vfs trait                                           │
│  ════════════════════                                       │
│  • Inode-based operations (InodeId, not paths)              │
│  • create_inode, lookup, link, unlink, read, write          │
│  • No path logic — raw inode operations                     │
│  • Target: Backend implementers                             │
├──────────────────┬──────────────────┬───────────────────────┤
│  MemoryVfs       │  SqliteVfs       │  VRootVfs            │
│  (HashMap)       │  (.db file)      │  (strict-path)        │
└──────────────────┴──────────────────┴───────────────────────┘
```

### Dependency Graph

```
anyfs (Vfs trait + inode types + backends)
     ↑
anyfs-container (FilesContainer<V, S> + FsSemantics)
     ↑
Your Application
```

---

# Crate 1: anyfs

**Purpose:** Low-level inode-based filesystem trait and built-in backends.

**Target Audience:** Backend implementers

## Crate Structure

```
anyfs/
├── Cargo.toml
├── src/
│   ├── lib.rs             # Re-exports, feature gates
│   ├── vfs.rs             # Vfs trait
│   ├── types.rs           # InodeId, InodeData, InodeKind
│   ├── error.rs           # VfsError
│   ├── memory/            # [feature: memory] MemoryVfs (default)
│   │   └── mod.rs
│   ├── sqlite/            # [feature: sqlite] SqliteVfs
│   │   ├── mod.rs
│   │   └── schema.rs
│   └── vrootfs/            # [feature: vrootfs] VRootVfs
│       └── mod.rs
│
└── tests/
    └── conformance.rs     # Tests all backends identically
```

## Cargo.toml

```toml
[package]
name = "anyfs"
version = "0.1.0"

[features]
default = ["memory"]
memory = []
sqlite = ["rusqlite"]
vrootfs = ["strict-path"]
full = ["memory", "sqlite", "vrootfs"]

[dependencies]
thiserror = "1"

# Optional backend dependencies
rusqlite = { version = "0.31", features = ["bundled"], optional = true }
strict-path = { version = "0.1", optional = true }
```

## Core Trait: Vfs

The `Vfs` trait defines inode-based operations. No paths — only `InodeId`.

```rust
/// Low-level filesystem trait.
/// Backend implementers work directly with inodes, not paths.
pub trait Vfs: Send {
    // ═══════════════════════════════════════════════════════════
    // INODE LIFECYCLE
    // ═══════════════════════════════════════════════════════════

    /// Create a new inode (file, directory, or symlink)
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError>;

    /// Get inode metadata
    fn get_inode(&self, id: InodeId) -> Result<InodeData, VfsError>;

    /// Update inode metadata (mode, timestamps, etc.)
    fn update_inode(&mut self, id: InodeId, data: InodeData) -> Result<(), VfsError>;

    /// Delete inode (must have nlink=0)
    fn delete_inode(&mut self, id: InodeId) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════
    // DIRECTORY OPERATIONS
    // ═══════════════════════════════════════════════════════════

    /// Add entry to directory: parent[name] = child
    fn link(&mut self, parent: InodeId, name: &str, child: InodeId) -> Result<(), VfsError>;

    /// Remove entry from directory, returns removed child inode
    fn unlink(&mut self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;

    /// Lookup child by name in directory
    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;

    /// List all entries in directory
    fn readdir(&self, dir: InodeId) -> Result<Vec<(String, InodeId)>, VfsError>;

    // ═══════════════════════════════════════════════════════════
    // CONTENT I/O
    // ═══════════════════════════════════════════════════════════

    /// Read bytes from file at offset
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError>;

    /// Write bytes to file at offset
    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError>;

    /// Truncate or extend file to size
    fn truncate(&mut self, id: InodeId, size: u64) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════
    // SYNC & ROOT
    // ═══════════════════════════════════════════════════════════

    /// Flush all pending writes to durable storage
    fn sync(&mut self) -> Result<(), VfsError>;

    /// Get root directory inode
    fn root(&self) -> InodeId;
}
```

## Types

```rust
/// Unique inode identifier within a storage backend.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct InodeId(pub u64);

#[derive(Clone, Debug)]
pub enum InodeKind {
    File,
    Directory,
    Symlink { target: String },
}

#[derive(Clone, Debug)]
pub struct InodeData {
    pub ino: u64,
    pub kind: InodeKind,
    pub mode: u32,              // Unix permission bits
    pub uid: u32,
    pub gid: u32,
    pub nlink: u64,             // Hard link count
    pub size: u64,
    pub atime: Option<SystemTime>,
    pub mtime: Option<SystemTime>,
    pub ctime: Option<SystemTime>,
}
```

## Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum VfsError {
    #[error("inode not found: {0:?}")]
    NotFound(InodeId),

    #[error("not a file: {0:?}")]
    NotAFile(InodeId),

    #[error("not a directory: {0:?}")]
    NotADirectory(InodeId),

    #[error("directory not empty: {0:?}")]
    DirectoryNotEmpty(InodeId),

    #[error("entry already exists: {0}")]
    EntryExists(String),

    #[error("entry not found: {0}")]
    EntryNotFound(String),

    #[error("inode still has links (nlink > 0)")]
    InodeInUse,

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("backend error: {0}")]
    Backend(String),
}
```

## Built-in Backends

### MemoryVfs (feature: memory)

```rust
use std::collections::HashMap;

pub struct MemoryVfs {
    inodes: HashMap<InodeId, InodeData>,
    content: HashMap<InodeId, Vec<u8>>,
    entries: HashMap<(InodeId, String), InodeId>,
    next_id: u64,
}

impl MemoryVfs {
    pub fn new() -> Self;
}

impl Vfs for MemoryVfs {
    fn root(&self) -> InodeId { InodeId(0) }
    // ... inode-based operations
}
```

### SqliteVfs (feature: sqlite)

```rust
pub struct SqliteVfs {
    conn: rusqlite::Connection,
}

impl SqliteVfs {
    pub fn create(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open_or_create(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open_in_memory() -> Result<Self, VfsError>;
}

impl Vfs for SqliteVfs {
    // Internal schema: inodes, entries, content tables
}
```

### VRootVfs (feature: vrootfs)

```rust
use strict_path::VirtualRoot;

pub struct VRootVfs {
    vroot: VirtualRoot,
    inode_paths: HashMap<InodeId, PathBuf>,
    path_inodes: HashMap<PathBuf, InodeId>,
    next_id: u64,
}

impl VRootVfs {
    pub fn new(root_dir: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open(root_dir: impl AsRef<Path>) -> Result<Self, VfsError>;
}

impl Vfs for VRootVfs {
    // Maps inode operations to real filesystem via strict-path
}
```

---

# Crate 2: anyfs-container

**Purpose:** High-level std::fs-like API with path semantics and capacity limits.

**Target Audience:** Application developers

## Crate Structure

```
anyfs-container/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── container.rs       # FilesContainer<V, S>
│   ├── builder.rs         # ContainerBuilder
│   ├── semantics/         # FsSemantics implementations
│   │   ├── mod.rs         # FsSemantics trait
│   │   ├── linux.rs       # LinuxSemantics
│   │   ├── windows.rs     # WindowsSemantics
│   │   └── simple.rs      # SimpleSemantics
│   ├── limits.rs          # CapacityLimits
│   ├── usage.rs           # CapacityUsage
│   └── error.rs           # ContainerError
│
└── tests/
    └── container_tests.rs
```

## Cargo.toml

```toml
[package]
name = "anyfs-container"
version = "0.1.0"

[dependencies]
anyfs = { version = "0.1", path = "../anyfs" }
thiserror = "1"
```

## FilesContainer

```rust
use std::path::Path;
use anyfs::Vfs;

/// A contained filesystem with path semantics and capacity limits.
pub struct FilesContainer<V: Vfs, S: FsSemantics> {
    vfs: V,
    semantics: S,
    limits: CapacityLimits,
    usage: CapacityUsage,
}

impl<V: Vfs, S: FsSemantics> FilesContainer<V, S> {
    pub fn new(vfs: V, semantics: S) -> Self;

    // ═══════════════════════════════════════════════════════════
    // READ OPERATIONS (matches std::fs)
    // ═══════════════════════════════════════════════════════════

    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    pub fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, ContainerError>;
    pub fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, ContainerError>;
    pub fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, ContainerError>;
    pub fn exists(&self, path: impl AsRef<Path>) -> bool;

    // ═══════════════════════════════════════════════════════════
    // WRITE OPERATIONS (matches std::fs)
    // ═══════════════════════════════════════════════════════════

    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;

    // ═══════════════════════════════════════════════════════════
    // LINK OPERATIONS
    // ═══════════════════════════════════════════════════════════

    pub fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;

    // ═══════════════════════════════════════════════════════════
    // PERMISSIONS
    // ═══════════════════════════════════════════════════════════

    pub fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), ContainerError>;

    // ═══════════════════════════════════════════════════════════
    // CONTAINER-SPECIFIC
    // ═══════════════════════════════════════════════════════════

    pub fn usage(&self) -> &CapacityUsage;
    pub fn limits(&self) -> &CapacityLimits;
}
```

## FsSemantics

```rust
pub trait FsSemantics: Send + Sync {
    /// Path separator character
    fn separator(&self) -> char;

    /// Split path into components
    fn components<'a>(&self, path: &'a str) -> Vec<&'a str>;

    /// Normalize path (resolve `.` and `..`)
    fn normalize(&self, path: &str) -> String;

    /// Case-sensitive comparisons?
    fn case_sensitive(&self) -> bool;

    /// Maximum symlink resolution depth
    fn max_symlink_depth(&self) -> u32;
}
```

### Built-in Implementations

| Semantics | Separator | Case Sensitive | Symlinks | Notes |
|-----------|-----------|----------------|----------|-------|
| `LinuxSemantics` | `/` | Yes | Yes | POSIX-like |
| `WindowsSemantics` | `\` (also `/`) | No | Yes | Windows-like |
| `SimpleSemantics` | `/` | Yes | No | Fast, minimal |

## Builder

```rust
pub struct ContainerBuilder<V, S> {
    vfs: V,
    semantics: S,
    limits: CapacityLimits,
}

impl<V: Vfs, S: FsSemantics> ContainerBuilder<V, S> {
    pub fn new() -> Self;
    pub fn vfs(self, vfs: V) -> Self;
    pub fn semantics(self, semantics: S) -> Self;
    pub fn limits(self, limits: CapacityLimits) -> Self;
    pub fn build(self) -> Result<FilesContainer<V, S>, ContainerError>;
}
```

---

# Usage Examples

## Application Developer (std::fs-like API)

```rust
use anyfs_container::{FilesContainer, LinuxSemantics, ContainerBuilder, CapacityLimits};
use anyfs::MemoryVfs;

// Simple creation
let mut container = FilesContainer::new(
    MemoryVfs::new(),
    LinuxSemantics::new(),
);

// Or with builder for limits
let mut container = ContainerBuilder::new()
    .vfs(MemoryVfs::new())
    .semantics(LinuxSemantics::new())
    .limits(CapacityLimits {
        max_total_size: Some(100 * 1024 * 1024),  // 100 MB
        ..Default::default()
    })
    .build()?;

// Use like std::fs
container.create_dir_all("/var/log")?;
container.write("/var/log/app.log", b"Started\n")?;
let data = container.read("/var/log/app.log")?;
```

## Backend Implementer (Vfs trait)

```rust
use anyfs::{Vfs, InodeId, InodeKind, InodeData, VfsError};

pub struct S3Vfs {
    bucket: String,
    client: aws_sdk_s3::Client,
    // ... inode metadata cache
}

impl Vfs for S3Vfs {
    fn root(&self) -> InodeId { InodeId(0) }

    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        // Allocate inode, store metadata
    }

    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError> {
        // Find child by name in parent directory
    }

    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError> {
        // Read from S3
    }

    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError> {
        // Write to S3
    }

    // ... other methods
}

// Now use with any semantics:
let container = FilesContainer::new(S3Vfs::new("my-bucket"), LinuxSemantics::new());
```

---

# Summary

| Crate | Purpose | Target User |
|-------|---------|-------------|
| `anyfs` | Inode-based Vfs trait + backends | Backend implementers |
| `anyfs-container` | std::fs-like API + semantics + limits | Application developers |

**anyfs** is the low-level foundation — inode operations only.
**anyfs-container** provides the familiar std::fs-like API with path resolution and limits.
