# AnyFS — Design Document

**Version:** 0.4.0
**Status:** Draft
**Last Updated:** 2025-12-23

---

## Table of Contents

1. [Overview](#1-overview)
2. [Two-Layer Architecture](#2-two-layer-architecture)
3. [Project 1: anyfs](#3-project-1-anyfs)
4. [Project 2: anyfs-container](#4-project-2-anyfs-container)
5. [Backend Implementations](#5-backend-implementations)
6. [FsSemantics](#6-fssemantics)
7. [Resolved Design Questions](#7-resolved-design-questions)
8. [Implementation Plan](#8-implementation-plan)
9. [Appendix](#appendix)

---

## 1. Overview

### 1.1 What Is This?

A two-layer virtual filesystem ecosystem for Rust:

| Layer | Crate | API Style | Target User |
|-------|-------|-----------|-------------|
| **Low-level** | `anyfs` | Inode-based (`Vfs` trait) | Backend implementers |
| **High-level** | `anyfs-container` | `std::fs`-like (paths) | Application developers |

### 1.2 Goals

| Goal | Description |
|------|-------------|
| **Switchable I/O** | Write code once, swap storage backends without changes |
| **Tenant Containment** | Isolate tenants via filesystem boundaries and quotas |
| **Backend Extensibility** | Users can implement custom backends via `Vfs` trait |
| **Semantic Flexibility** | Pluggable path semantics (Linux, Windows, custom) |
| **Multiple Storage Options** | Memory, SQLite, Real filesystem out of the box |

### 1.3 Non-Goals

- POSIX compliance
- Async (initially)
- Streaming I/O (initially)
- Distributed storage

---

## 2. Two-Layer Architecture

### 2.1 Layer Diagram

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

### 2.2 Why Two Layers?

**Separation of Concerns:**

| Concern | Layer |
|---------|-------|
| How to store data | anyfs (`Vfs` trait) |
| How to interpret paths | anyfs-container (`FsSemantics`) |
| Capacity limits | anyfs-container (`FilesContainer`) |
| User-facing API | anyfs-container (`std::fs`-like) |

**Benefits:**

1. **Backend simplicity**: Backend implementers don't deal with paths
2. **Semantic flexibility**: Mix any semantics with any storage
3. **Clear responsibilities**: Each layer does one thing well
4. **Testability**: Use `SimpleSemantics` for fast tests
5. **FUSE compatibility**: Inode-based model maps directly to FUSE

### 2.3 How They Connect

When a user calls `container.write("/data/file.txt", b"hello")`:

```
1. Parse path via FsSemantics
   "/data/file.txt" → ["data", "file.txt"]

2. Walk to parent via Vfs
   vfs.root() → InodeId(1)
   vfs.lookup(1, "data") → InodeId(5)

3. Check capacity limits
   usage + 5 bytes ≤ limit?

4. Create/get file inode
   vfs.lookup(5, "file.txt") or vfs.create_inode(...)
   → InodeId(12)

5. Write content
   vfs.truncate(12, 0)
   vfs.write(12, 0, b"hello")

6. Update usage tracking
   self.usage.total_size += 5
```

---

## 3. Project 1: anyfs

### 3.1 Purpose

Provide a low-level trait for inode-based filesystem operations that any storage backend can implement.

**Key insight:** No paths. Only `InodeId`.

### 3.2 The Vfs Trait

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

### 3.3 Types

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

### 3.4 Errors

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

### 3.5 Crate Structure

```
anyfs/
├── Cargo.toml
├── src/
│   ├── lib.rs             # Re-exports, feature gates
│   ├── vfs.rs             # Vfs trait
│   ├── types.rs           # InodeId, InodeData, InodeKind
│   ├── error.rs           # VfsError
│   ├── memory/            # [feature: memory] MemoryVfs (default)
│   ├── sqlite/            # [feature: sqlite] SqliteVfs
│   └── vrootfs/            # [feature: vrootfs] VRootVfs
│
└── tests/
    └── conformance.rs     # Tests all backends identically
```

### 3.6 Dependencies

```toml
[dependencies]
thiserror = "1"

[dependencies.rusqlite]
version = "0.31"
features = ["bundled"]
optional = true

[dependencies.strict-path]
version = "..."
optional = true

[features]
default = ["memory"]
memory = []
sqlite = ["rusqlite"]
vrootfs = ["strict-path"]
full = ["memory", "sqlite", "vrootfs"]
```

---

## 4. Project 2: anyfs-container

### 4.1 Purpose

Provide a high-level `std::fs`-like API on top of any `Vfs` backend with:

- Path resolution via pluggable `FsSemantics`
- Capacity limits (total size, file size, node count, etc.)
- Usage tracking

### 4.2 FilesContainer

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

### 4.3 Capacity Types

```rust
#[derive(Clone, Debug, Default)]
pub struct CapacityLimits {
    pub max_total_size: Option<u64>,
    pub max_file_size: Option<u64>,
    pub max_node_count: Option<u64>,
    pub max_dir_entries: Option<u32>,
    pub max_path_depth: Option<u16>,
}

#[derive(Clone, Debug, Default)]
pub struct CapacityUsage {
    pub total_size: u64,
    pub node_count: u64,
    pub file_count: u64,
    pub directory_count: u64,
}
```

### 4.4 Builder

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

### 4.5 Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum ContainerError {
    #[error("not found: {0}")]
    NotFound(PathBuf),

    #[error("already exists: {0}")]
    AlreadyExists(PathBuf),

    #[error("not a file: {0}")]
    NotAFile(PathBuf),

    #[error("not a directory: {0}")]
    NotADirectory(PathBuf),

    #[error("directory not empty: {0}")]
    DirectoryNotEmpty(PathBuf),

    #[error("invalid path: {0}")]
    InvalidPath(String),

    #[error("symlink loop detected: {0}")]
    SymlinkLoop(PathBuf),

    #[error("total size limit exceeded: {used} / {limit} bytes")]
    TotalSizeExceeded { used: u64, limit: u64 },

    #[error("file size {size} exceeds limit of {limit} bytes")]
    FileSizeExceeded { size: u64, limit: u64 },

    #[error("node count limit exceeded: {count} / {limit}")]
    NodeCountExceeded { count: u64, limit: u64 },

    #[error("Vfs error: {0}")]
    Vfs(#[from] VfsError),
}
```

### 4.6 Crate Structure

```
anyfs-container/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── container.rs       # FilesContainer<V, S>
│   ├── builder.rs         # ContainerBuilder
│   ├── semantics/         # FsSemantics implementations
│   │   ├── mod.rs
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

### 4.7 Dependencies

```toml
[dependencies]
anyfs = { version = "0.1", path = "../anyfs" }
thiserror = "1"
```

---

## 5. Backend Implementations

### 5.1 MemoryVfs

In-memory storage using HashMap. Best for testing.

```rust
use std::collections::HashMap;

pub struct MemoryVfs {
    inodes: HashMap<InodeId, InodeData>,
    content: HashMap<InodeId, Vec<u8>>,
    entries: HashMap<(InodeId, String), InodeId>,
    next_id: u64,
}

impl MemoryVfs {
    pub fn new() -> Self {
        let mut vfs = Self {
            inodes: HashMap::new(),
            content: HashMap::new(),
            entries: HashMap::new(),
            next_id: 1,
        };
        // Create root directory
        let root_id = InodeId(0);
        vfs.inodes.insert(root_id, InodeData {
            ino: 0,
            kind: InodeKind::Directory,
            mode: 0o755,
            uid: 0,
            gid: 0,
            nlink: 2,
            size: 0,
            atime: None,
            mtime: None,
            ctime: None,
        });
        vfs
    }
}

impl Vfs for MemoryVfs {
    fn root(&self) -> InodeId { InodeId(0) }

    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        let id = InodeId(self.next_id);
        self.next_id += 1;

        self.inodes.insert(id, InodeData {
            ino: id.0,
            kind,
            mode,
            uid: 0,
            gid: 0,
            nlink: 0,  // Not linked yet
            size: 0,
            atime: None,
            mtime: None,
            ctime: None,
        });

        Ok(id)
    }

    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError> {
        self.entries
            .get(&(parent, name.to_string()))
            .copied()
            .ok_or(VfsError::EntryNotFound(name.to_string()))
    }

    // ... other methods
}
```

### 5.2 SqliteVfs

Single-file storage using SQLite.

```rust
use rusqlite::Connection;

pub struct SqliteVfs {
    conn: Connection,
}

impl SqliteVfs {
    pub fn create(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open_or_create(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open_in_memory() -> Result<Self, VfsError>;
}
```

**Schema:**

```sql
-- Inode table
CREATE TABLE inodes (
    id INTEGER PRIMARY KEY,
    kind TEXT NOT NULL CHECK (kind IN ('file', 'directory', 'symlink')),
    mode INTEGER NOT NULL DEFAULT 493,  -- 0o755
    uid INTEGER NOT NULL DEFAULT 0,
    gid INTEGER NOT NULL DEFAULT 0,
    nlink INTEGER NOT NULL DEFAULT 0,
    size INTEGER NOT NULL DEFAULT 0,
    symlink_target TEXT,
    atime INTEGER,
    mtime INTEGER,
    ctime INTEGER
);

-- Directory entries table
CREATE TABLE entries (
    parent_id INTEGER NOT NULL REFERENCES inodes(id),
    name TEXT NOT NULL,
    child_id INTEGER NOT NULL REFERENCES inodes(id),
    PRIMARY KEY (parent_id, name)
);

-- File content table
CREATE TABLE content (
    inode_id INTEGER PRIMARY KEY REFERENCES inodes(id),
    data BLOB NOT NULL DEFAULT X''
);

-- Root inode (id=0)
INSERT INTO inodes (id, kind, mode, nlink) VALUES (0, 'directory', 493, 2);
```

### 5.3 VRootVfs

Maps inodes to real filesystem using `strict-path` for containment.

```rust
use strict_path::VirtualRoot;

pub struct VRootVfs {
    vroot: VirtualRoot,
    // Maps InodeId to path within vroot
    inode_paths: HashMap<InodeId, PathBuf>,
    path_inodes: HashMap<PathBuf, InodeId>,
    next_id: u64,
}

impl VRootVfs {
    pub fn new(root_dir: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open(root_dir: impl AsRef<Path>) -> Result<Self, VfsError>;
}
```

**Key behavior:**

- Uses `strict-path::VirtualRoot` for containment
- Paths like `/etc/passwd` are clamped to `root_dir/etc/passwd`
- Symlink escapes are prevented by `strict-path`

---

## 6. FsSemantics

Path resolution is pluggable via the `FsSemantics` trait.

### 6.1 Trait Definition

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

### 6.2 Built-in Implementations

| Semantics | Separator | Case Sensitive | Symlinks | Notes |
|-----------|-----------|----------------|----------|-------|
| `LinuxSemantics` | `/` | Yes | Yes | POSIX-like |
| `WindowsSemantics` | `\` (also `/`) | No | Yes | Windows-like |
| `SimpleSemantics` | `/` | Yes | No | Fast, minimal |

### 6.3 Example

```rust
use anyfs_container::{FilesContainer, LinuxSemantics, WindowsSemantics};
use anyfs::MemoryVfs;

// Linux-style paths
let mut linux = FilesContainer::new(
    MemoryVfs::new(),
    LinuxSemantics::new(),
);
linux.write("/var/log/app.log", b"data")?;

// Windows-style paths (same backend!)
let mut windows = FilesContainer::new(
    MemoryVfs::new(),
    WindowsSemantics::new(),
);
windows.write("\\Users\\Admin\\file.txt", b"data")?;
```

---

## 7. Resolved Design Questions

### 7.1 Path vs Inode API — RESOLVED

**Decision:** Two-layer architecture:
- `Vfs` trait uses `InodeId` (low-level, for backend implementers)
- `FilesContainer` uses `impl AsRef<Path>` (high-level, for app developers)

**Rationale:**
- Backend simplicity: no path parsing in backends
- FUSE compatibility: inode model maps directly
- Semantic flexibility: any path rules with any storage

### 7.2 Symlink & Hard Link Support — RESOLVED

**Decision:** Full support in all backends.

- Symlinks stored as `InodeKind::Symlink { target }`
- Hard links: multiple directory entries point to same inode
- `nlink` tracks hard link count
- Symlink loop detection: configurable via `FsSemantics::max_symlink_depth()`

### 7.3 Method Naming — RESOLVED

**Decision:** `FilesContainer` aligns with `std::fs` naming.

| FilesContainer | std::fs |
|----------------|---------|
| `read()` | `std::fs::read` |
| `write()` | `std::fs::write` |
| `create_dir_all()` | `std::fs::create_dir_all` |
| `remove_file()` | `std::fs::remove_file` |
| `read_dir()` | `std::fs::read_dir` |

---

## 8. Implementation Plan

### Phase 1: Core Types

- [ ] `anyfs`: `Vfs` trait
- [ ] `anyfs`: `InodeId`, `InodeData`, `InodeKind`
- [ ] `anyfs`: `VfsError`
- [ ] `anyfs`: `MemoryVfs` (simplest, for testing)

### Phase 2: SqliteVfs

- [ ] Schema design
- [ ] Implement `SqliteVfs`
- [ ] Conformance tests

### Phase 3: anyfs-container

- [ ] `FsSemantics` trait
- [ ] `LinuxSemantics`, `WindowsSemantics`, `SimpleSemantics`
- [ ] `FilesContainer<V, S>`
- [ ] `CapacityLimits`, `CapacityUsage`
- [ ] `ContainerBuilder`
- [ ] Limit enforcement

### Phase 4: VRootVfs

- [ ] Integrate `strict-path`
- [ ] Implement `VRootVfs`
- [ ] Conformance tests

### Phase 5: Polish

- [ ] Documentation
- [ ] Examples
- [ ] Publish

---

## Appendix

### Comparison with Alternatives

| Feature | anyfs | vfs crate | OpenDAL |
|---------|-------|-----------|---------|
| API level | Inode + Path | Path only | Object storage |
| Backends | Memory, SQLite, RealFs | Memory, Physical, Overlay | 50+ cloud services |
| Capacity limits | Yes (container) | No | No |
| Path semantics | Pluggable | Fixed | N/A |
| Async | No | No | Yes |
| Use case | Embedded VFS | Resource overlays | Cloud storage |

### Usage Examples

**For Application Developers:**

```rust
use anyfs_container::{FilesContainer, LinuxSemantics, ContainerBuilder, CapacityLimits};
use anyfs::MemoryVfs;

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

**For Backend Implementers:**

```rust
use anyfs::{Vfs, InodeId, InodeKind, InodeData, VfsError};

pub struct MyCustomVfs { /* ... */ }

impl Vfs for MyCustomVfs {
    fn root(&self) -> InodeId { InodeId(0) }

    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        // Allocate and store inode
    }

    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError> {
        // Find child by name
    }

    // ... other methods
}

// Now use with any semantics:
let container = FilesContainer::new(MyCustomVfs::new(), LinuxSemantics::new());
```

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2025-12-22 | Initial design (graph-store trait) |
| 0.2.0 | 2025-12-22 | Restructured: two projects, path-based trait |
| 0.3.0 | 2025-12-22 | Three-crate structure, std::fs aligned API |
| 0.4.0 | 2025-12-23 | Two-layer architecture: inode-based `Vfs` + path-based `FilesContainer` |
