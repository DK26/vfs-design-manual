# VFS Ecosystem — Design Document

**Version:** 0.3.0
**Status:** Draft
**Last Updated:** 2025-12-22

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Project 1: anyfs](#3-project-1-anyfs)
4. [Project 2: anyfs-container](#4-project-2-anyfs-container)
5. [Backend Implementations](#5-backend-implementations)
6. [Open Design Questions](#6-open-design-questions)
7. [Implementation Plan](#7-implementation-plan)
8. [Appendix: Comparison with Alternatives](#appendix-comparison-with-alternatives)

---

## 1. Overview

### 1.1 What Is This?

A layered virtual filesystem ecosystem for Rust:

- **Crate 1 (`anyfs-traits`)**: Minimal trait + types — for custom backend implementers
- **Crate 2 (`anyfs`)**: Built-in backends (feature-gated), re-exports traits
- **Crate 3 (`anyfs-container`)**: Higher-level wrapper — capacity limits, tenant isolation

### 1.2 Goals

| Goal | Description |
|------|-------------|
| **Switchable I/O** | Write code once, swap storage backends without changes |
| **Tenant Containment** | Isolate tenants via filesystem boundaries and quotas |
| **Backend Extensibility** | Users can implement custom backends |
| **Multiple Storage Options** | Filesystem, SQLite, Memory out of the box |

### 1.3 Non-Goals

- POSIX compliance
- Async (initially)
- Streaming I/O (initially)
- Distributed storage

---

## 2. Architecture

### 2.1 Layer Diagram

```
┌─────────────────────────────────────────┐
│  Your Application                       │  ← Uses clean API
├─────────────────────────────────────────┤
│  anyfs-container                        │  ← Quotas, tenant isolation
│  FilesContainer<B: VfsBackend>          │
├─────────────────────────────────────────┤
│  anyfs                                  │  ← Built-in backends (feature-gated)
├──────────┬──────────┬───────────────────┤
│ VRootFs  │  Memory  │  SQLite           │  ← Optional backend implementations
│ Backend  │  Backend │  Backend          │
├──────────┴──────────┴───────────────────┤
│  anyfs-traits                           │  ← Minimal: trait + types
│  VfsBackend trait, VfsError, Metadata   │
├─────────────────────────────────────────┤
│  strict-path (external)                 │  ← VirtualPath, VirtualRoot
└─────────────────────────────────────────┘
```

### 2.2 Separation of Concerns

| Layer | Responsibility |
|-------|---------------|
| `anyfs-traits` | Defines `VfsBackend` trait, error types, metadata types |
| `anyfs` | Re-exports traits, provides built-in backends (feature-gated) |
| `anyfs-container` | Higher-level wrapper, capacity limits, tenant isolation |
| Application | Uses `FilesContainer`, doesn't know about backend details |

### 2.3 Containment Strategies

All three backends achieve isolation differently:

| Backend | Containment Mechanism |
|---------|----------------------|
| `VRootVRootFsBackend` | `strict-path::VirtualRoot` — real directory acts as root, paths clamped |
| `SqliteBackend` | Each container is a `.db` file — complete isolation by file |
| `MemoryBackend` | Each container is a separate instance — isolation by process memory |

---

## 3. Project 1: anyfs

### 3.1 Purpose

Provide a trait for virtual filesystem operations that any storage backend can implement.

### 3.2 The VfsBackend Trait

The trait is defined in `anyfs-traits` and uses `&VirtualPath` for type safety. Method names align with `std::fs`.

```rust
use anyfs_traits::VirtualPath;

/// A virtual filesystem backend.
/// All backends implement full filesystem semantics including symlinks and hard links.
pub trait VfsBackend: Send {
    // READ OPERATIONS
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: &VirtualPath) -> Result<String, VfsError>;
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError>;

    // WRITE OPERATIONS
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;

    // LINKS
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;

    // PERMISSIONS
    fn set_permissions(&mut self, path: &VirtualPath, perm: Permissions) -> Result<(), VfsError>;
}
```

**Two-layer path handling:**
- **VfsBackend** uses `&VirtualPath` — type-safe, pre-validated paths from `strict-path`
- **FilesContainer** uses `impl AsRef<Path>` — ergonomic user-facing API that converts to `VirtualPath` internally

### 3.3 Types

```rust
/// Type of filesystem entry
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum FileType {
    File,
    Directory,
    Symlink,
}

impl FileType {
    pub fn is_file(&self) -> bool { matches!(self, FileType::File) }
    pub fn is_dir(&self) -> bool { matches!(self, FileType::Directory) }
    pub fn is_symlink(&self) -> bool { matches!(self, FileType::Symlink) }
}

/// File metadata
#[derive(Clone, Debug)]
pub struct Metadata {
    pub file_type: FileType,
    pub size: u64,
    pub permissions: Permissions,
    pub nlink: u64,  // Number of hard links
    pub created: Option<SystemTime>,
    pub modified: Option<SystemTime>,
    pub accessed: Option<SystemTime>,
}

/// File permissions (simplified model)
#[derive(Clone, Copy, Debug, PartialEq, Eq, Default)]
pub struct Permissions {
    readonly: bool,
}

impl Permissions {
    pub fn new() -> Self { Self { readonly: false } }
    pub fn readonly(&self) -> bool { self.readonly }
    pub fn set_readonly(&mut self, readonly: bool) { self.readonly = readonly; }
}

/// Directory entry
#[derive(Clone, Debug)]
pub struct DirEntry {
    pub name: String,
    pub file_type: FileType,
}
```

### 3.4 Errors

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

    #[error("not a symlink: {0}")]
    NotASymlink(VirtualPath),

    #[error("directory not empty: {0}")]
    DirectoryNotEmpty(VirtualPath),

    #[error("is a directory: {0}")]
    IsADirectory(VirtualPath),

    #[error("invalid path: {0}")]
    InvalidPath(String),

    #[error("invalid UTF-8: {0}")]
    InvalidUtf8(#[from] std::str::Utf8Error),

    #[error("permission denied: {0}")]
    PermissionDenied(VirtualPath),

    #[error("read-only filesystem")]
    ReadOnlyFilesystem,

    #[error("too many symlinks (loop detected): {0}")]
    SymlinkLoop(VirtualPath),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("backend error: {0}")]
    Backend(String),
}
```

### 3.5 Crate Structure (Three Crates)

```
anyfs-traits/              # Crate 1: Minimal trait + types
├── Cargo.toml             # Depends on: strict-path, thiserror
├── src/
│   ├── lib.rs             # Re-exports VirtualPath, defines trait
│   ├── backend.rs         # VfsBackend trait
│   ├── types.rs           # Metadata, DirEntry, FileType, Permissions
│   └── error.rs           # VfsError

anyfs/                     # Crate 2: Built-in backends
├── Cargo.toml             # Depends on: anyfs-traits + optional deps
├── src/
│   ├── lib.rs             # Re-exports anyfs-traits::*
│   ├── vrootfs/           # [feature: vrootfs] VRootFsBackend
│   ├── memory/            # [feature: memory] MemoryBackend (default)
│   └── sqlite/            # [feature: sqlite] SqliteBackend
│
└── tests/
    └── conformance.rs     # Tests all backends identically

anyfs-container/           # Crate 3: Isolation layer
├── Cargo.toml             # Depends on: anyfs-traits
├── src/
│   ├── lib.rs
│   ├── container.rs       # FilesContainer<B: VfsBackend>
│   ├── builder.rs         # ContainerBuilder
│   ├── limits.rs          # CapacityLimits
│   └── error.rs           # ContainerError
```

### 3.6 Dependencies

**anyfs-traits/Cargo.toml:**
```toml
[dependencies]
strict-path = "..."
thiserror = "1"
```

**anyfs/Cargo.toml:**
```toml
[dependencies]
anyfs-traits = { version = "0.1", path = "../anyfs-traits" }
thiserror = "1"

[dependencies.strict-path]
version = "..."
optional = true

[dependencies.rusqlite]
version = "0.31"
features = ["bundled"]
optional = true

[features]
default = ["memory"]
memory = []
vrootfs = ["strict-path"]
sqlite = ["rusqlite"]
full = ["memory", "vrootfs", "sqlite"]
```

---

## 4. Project 2: anyfs-container

### 4.1 Purpose

Wrap any `VfsBackend` with:
- Capacity limits (total size, file size, node count, etc.)
- Usage tracking
- (Future: permissions, quotas per directory, etc.)

### 4.2 FilesContainer

```rust
use std::path::Path;
use anyfs::VfsBackend;

/// A contained filesystem with capacity limits.
pub struct FilesContainer<B: VfsBackend> {
    backend: B,
    limits: CapacityLimits,
    usage: CapacityUsage,
}

impl<B: VfsBackend> FilesContainer<B> {
    pub fn new(backend: B) -> Self;
    pub fn with_limits(backend: B, limits: CapacityLimits) -> Self;
    
    // Delegated operations (with limit enforcement)
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    // ... all VfsBackend methods, wrapped
    
    // Capacity management
    pub fn usage(&self) -> &CapacityUsage;
    pub fn limits(&self) -> &CapacityLimits;
    pub fn remaining(&self) -> CapacityRemaining;
}
```

### 4.3 Capacity Types

```rust
#[derive(Clone, Debug, Default)]
pub struct CapacityLimits {
    /// Maximum total bytes (None = unlimited)
    pub max_total_size: Option<u64>,
    
    /// Maximum single file size (None = unlimited)
    pub max_file_size: Option<u64>,
    
    /// Maximum number of files + directories (None = unlimited)
    pub max_node_count: Option<u64>,
    
    /// Maximum entries per directory (None = unlimited)
    pub max_dir_entries: Option<u32>,
    
    /// Maximum path depth (None = unlimited)
    pub max_path_depth: Option<u16>,
}

#[derive(Clone, Debug, Default)]
pub struct CapacityUsage {
    pub total_size: u64,
    pub node_count: u64,
    pub file_count: u64,
    pub directory_count: u64,
}

#[derive(Clone, Debug)]
pub struct CapacityRemaining {
    pub bytes: Option<u64>,
    pub nodes: Option<u64>,
    pub can_write: bool,
}
```

### 4.4 Builder

```rust
pub struct ContainerBuilder<B> {
    backend: B,
    limits: CapacityLimits,
}

impl<B: VfsBackend> ContainerBuilder<B> {
    pub fn new(backend: B) -> Self;
    
    pub fn max_total_size(self, bytes: u64) -> Self;
    pub fn max_file_size(self, bytes: u64) -> Self;
    pub fn max_node_count(self, count: u64) -> Self;
    pub fn max_dir_entries(self, count: u32) -> Self;
    pub fn max_path_depth(self, depth: u16) -> Self;
    pub fn unlimited(self) -> Self;
    
    pub fn build(self) -> FilesContainer<B>;
}
```

### 4.5 Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum ContainerError {
    #[error(transparent)]
    Vfs(#[from] VfsError),
    
    #[error("total size limit exceeded: {used} / {limit} bytes")]
    TotalSizeExceeded { used: u64, limit: u64 },
    
    #[error("file size {size} exceeds limit of {limit} bytes")]
    FileSizeExceeded { size: u64, limit: u64 },
    
    #[error("node count limit exceeded: {count} / {limit}")]
    NodeCountExceeded { count: u64, limit: u64 },
    
    #[error("directory entry limit exceeded: {count} / {limit}")]
    DirEntriesExceeded { count: u32, limit: u32 },
    
    #[error("path depth {depth} exceeds limit of {limit}")]
    PathDepthExceeded { depth: u16, limit: u16 },
}
```

### 4.6 Crate Structure

```
anyfs-container/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── container.rs    # FilesContainer
│   ├── builder.rs      # ContainerBuilder
│   ├── limits.rs       # CapacityLimits
│   ├── usage.rs        # CapacityUsage, CapacityRemaining
│   └── error.rs        # ContainerError
│
└── tests/
    └── limits.rs       # Capacity enforcement tests
```

### 4.7 Dependencies

```toml
[dependencies]
anyfs = { version = "0.1", path = "../anyfs" }
thiserror = "1"
```

---

## 5. Backend Implementations

### 5.1 VRootFsBackend

Uses `strict-path::VirtualRoot` for containment.

```rust
use std::path::Path;
use strict_path::VirtualRoot;
use anyfs_traits::VirtualPath;

pub struct VRootFsBackend {
    vroot: VirtualRoot,
}

impl VRootFsBackend {
    /// Create backend with given directory as virtual root.
    pub fn new(root_dir: impl AsRef<Path>) -> Result<Self, VfsError> {
        let vroot = VirtualRoot::try_new_create(root_dir.as_ref())
            .map_err(|e| VfsError::Backend(e.to_string()))?;
        Ok(Self { vroot })
    }

    /// Open existing directory as virtual root.
    pub fn open(root_dir: impl AsRef<Path>) -> Result<Self, VfsError> {
        let vroot = VirtualRoot::try_new(root_dir.as_ref())
            .map_err(|e| VfsError::Backend(e.to_string()))?;
        Ok(Self { vroot })
    }
}

impl VfsBackend for VRootFsBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        let real_path = self.vroot.virtual_join(path.as_str())
            .map_err(|e| VfsError::InvalidPath(e.to_string()))?;
        std::fs::read(real_path.as_std_path()).map_err(VfsError::Io)
    }

    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError> {
        let real_path = self.vroot.virtual_join(path.as_str())
            .map_err(|e| VfsError::InvalidPath(e.to_string()))?;
        std::fs::write(real_path.as_std_path(), data).map_err(VfsError::Io)
    }

    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        let link_path = self.vroot.virtual_join(link.as_str())?;
        // Store symlink target as virtual path (relative to root)
        #[cfg(unix)]
        std::os::unix::fs::symlink(original.as_str(), link_path.as_std_path())
            .map_err(VfsError::Io)?;
        Ok(())
    }

    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        let original_path = self.vroot.virtual_join(original.as_str())?;
        let link_path = self.vroot.virtual_join(link.as_str())?;
        std::fs::hard_link(original_path.as_std_path(), link_path.as_std_path())
            .map_err(VfsError::Io)
    }

    // ... other methods follow same pattern
}
```

**Key behavior:**
- Paths like `/etc/passwd` are **clamped** to `root_dir/etc/passwd`
- Symlink escapes are prevented by `strict-path`
- Uses real filesystem for storage
- Real OS symlinks and hard links via `std::fs`

### 5.2 MemoryBackend

In-memory storage using HashMap with support for symlinks and hard links.

```rust
use std::collections::HashMap;
use std::sync::Arc;
use std::time::SystemTime;
use anyfs_traits::VirtualPath;

pub struct MemoryBackend {
    entries: HashMap<VirtualPath, Entry>,
    contents: HashMap<ContentId, Arc<ContentData>>,
    next_content_id: ContentId,
}

#[derive(Clone, Copy, PartialEq, Eq, Hash)]
struct ContentId(u64);

struct ContentData {
    data: Vec<u8>,
    ref_count: std::sync::atomic::AtomicU64,
}

enum Entry {
    File {
        content_id: ContentId,
        permissions: Permissions,
        created: SystemTime,
        modified: SystemTime,
    },
    Directory {
        permissions: Permissions,
        created: SystemTime,
        modified: SystemTime,
    },
    Symlink {
        target: VirtualPath,
        created: SystemTime,
    },
}

impl MemoryBackend {
    pub fn new() -> Self {
        let mut entries = HashMap::new();
        entries.insert(VirtualPath::root(), Entry::Directory {
            permissions: Permissions::new(),
            created: SystemTime::now(),
            modified: SystemTime::now(),
        });
        Self {
            entries,
            contents: HashMap::new(),
            next_content_id: ContentId(1),
        }
    }
}

impl VfsBackend for MemoryBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        let resolved = self.resolve_symlinks(path)?;
        match self.entries.get(&resolved) {
            Some(Entry::File { content_id, .. }) => {
                Ok(self.contents[content_id].data.clone())
            }
            Some(Entry::Directory { .. }) => Err(VfsError::NotAFile(resolved)),
            Some(Entry::Symlink { .. }) => unreachable!(), // resolved above
            None => Err(VfsError::NotFound(resolved)),
        }
    }

    // Hard links: multiple Entry::File can share the same content_id
    // nlink = count of entries with same content_id
    // ... other methods
}
```

**Key behavior:**
- Uses `VirtualPath` as HashMap keys
- Hard links share content via `ContentId`
- Symlinks stored as target path
- No persistence — data lost when dropped
- Useful for testing

### 5.3 SqliteBackend

Single-file storage using SQLite with symlink and hard link support.

```rust
use rusqlite::Connection;
use std::path::Path;

pub struct SqliteBackend {
    conn: Connection,
}

impl SqliteBackend {
    pub fn create(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open_or_create(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open_in_memory() -> Result<Self, VfsError>;
}
```

**Schema:**

```sql
-- Entries table: files, directories, symlinks
CREATE TABLE entries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    path TEXT NOT NULL UNIQUE,
    entry_type TEXT NOT NULL CHECK (entry_type IN ('file', 'directory', 'symlink')),
    content_id INTEGER REFERENCES contents(id) ON DELETE SET NULL,
    symlink_target TEXT,
    permissions INTEGER NOT NULL DEFAULT 0,
    created_at INTEGER,
    modified_at INTEGER,
    accessed_at INTEGER
);

-- Content table: actual file data (shared by hard links)
CREATE TABLE contents (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    data BLOB NOT NULL,
    ref_count INTEGER NOT NULL DEFAULT 1
);

-- Indexes
CREATE UNIQUE INDEX idx_entries_path ON entries(path);
CREATE INDEX idx_entries_content ON entries(content_id) WHERE content_id IS NOT NULL;

-- Constraints (enforced in application):
-- - entry_type = 'file' => content_id NOT NULL, symlink_target IS NULL
-- - entry_type = 'directory' => content_id IS NULL, symlink_target IS NULL
-- - entry_type = 'symlink' => content_id IS NULL, symlink_target NOT NULL
```

**Hard link implementation:**
- Multiple rows with different `path` share same `content_id`
- `ref_count` tracks hard link count
- When last hard link removed, content row is deleted

**nlink query:**
```sql
SELECT COUNT(*) FROM entries WHERE content_id = ?
```

**Key behavior:**
- Entire container is a single `.db` file
- Portable — copy file to move container
- Hard links share content via `content_id`
- Symlinks stored as `symlink_target` path

---

## 6. Resolved Design Questions

### 6.1 Path Type in Trait — ✅ RESOLVED

**Decision:** Two-layer approach:
- `VfsBackend` trait uses `&VirtualPath` (type-safe, pre-validated)
- `FilesContainer` uses `impl AsRef<Path>` (ergonomic user API)

**Rationale:**
- Type safety: backends receive pre-validated paths
- User ergonomics: applications can pass `&str`, `String`, `&Path`, `PathBuf`
- Single validation point: `FilesContainer` validates once, backend trusts the path

### 6.2 Error Path Representation — ✅ RESOLVED

**Decision:** `VfsError` variants use `VirtualPath`

```rust
NotFound(VirtualPath),
AlreadyExists(VirtualPath),
// etc.
```

**Rationale:** Errors originate from operations on validated paths. Using `VirtualPath` maintains type consistency.

### 6.3 Symlink & Hard Link Support — ✅ RESOLVED

**Decision:** Full symlink and hard link support in all backends.

New trait methods:
- `symlink()` — create symbolic link
- `hard_link()` — create hard link
- `read_link()` — read symlink target
- `symlink_metadata()` — get metadata without following symlinks

Symlink loop detection: 40 levels max (matches Linux default).

### 6.4 Method Naming — ✅ RESOLVED

**Decision:** Align with `std::fs` naming conventions.

| Old Name | New Name (std::fs aligned) |
|----------|---------------------------|
| `list()` | `read_dir()` |
| `mkdir()` | `create_dir()` |
| `mkdir_all()` | `create_dir_all()` |
| `remove()` | `remove_file()` + `remove_dir()` (split) |
| `remove_all()` | `remove_dir_all()` |

### 6.5 Backend Discovery of Usage

**Question:** How does `FilesContainer` know current usage when wrapping an existing backend?

**Current approach:** Backend provides optional `fn usage(&self) -> Option<CapacityUsage>`. If `None`, container scans on construction.

---

## 7. Implementation Plan

### Phase 1: Core Types (Week 1)

- [x] ~~Decide path type~~ → `impl AsRef<Path>`
- [ ] `anyfs`: `VfsBackend` trait
- [ ] `anyfs`: `Metadata`, `DirEntry`, `FileType`
- [ ] `anyfs`: `VfsError`
- [ ] `anyfs`: `MemoryBackend` (simplest, for testing)

### Phase 2: VRootFsBackend (Week 2)

- [ ] Integrate `strict-path`
- [ ] Implement `VRootFsBackend`
- [ ] Conformance tests (same tests as MemoryBackend)

### Phase 3: SqliteBackend (Week 3)

- [ ] Schema design
- [ ] Implement `SqliteBackend`
- [ ] Conformance tests

### Phase 4: anyfs-container (Week 4)

- [ ] `FilesContainer`
- [ ] `CapacityLimits`, `CapacityUsage`
- [ ] `ContainerBuilder`
- [ ] Limit enforcement
- [ ] Tests

### Phase 5: Polish (Week 5)

- [ ] Documentation
- [ ] Examples
- [ ] Publish

---

## Appendix: Comparison with Alternatives

### vs. `vfs` crate

| Feature | vfs | anyfs |
|---------|-----|----------------|
| Path type | `&str` | TBD |
| Backends | Physical, Memory, Overlay, Embedded | Fs, Memory, SQLite |
| Capacity limits | ❌ | ✅ via anyfs-container |
| Containment | ❌ | ✅ via strict-path |
| Overlay/layering | ✅ | ❌ |

### vs. `OpenDAL`

| Feature | OpenDAL | anyfs |
|---------|---------|----------------|
| Focus | Cloud/object storage | Local/embedded storage |
| API style | Object storage (get/put) | Filesystem (read/write/mkdir) |
| Backends | 50+ (S3, GCS, Azure, etc.) | 3 (Fs, Memory, SQLite) |
| Async | ✅ | ❌ (sync only) |

### When to Use What

- **anyfs**: Local storage, tenant isolation, portable containers
- **vfs crate**: Overlay filesystems, embedded resources
- **OpenDAL**: Cloud storage, async workloads

---

## Appendix: Usage Examples

### Basic Usage (anyfs only)

```rust
use anyfs::{VfsBackend, VRootFsBackend, MemoryBackend};

fn save_data(vfs: &mut impl VfsBackend, name: &str, data: &[u8]) -> Result<(), VfsError> {
    vfs.create_dir_all("/data")?;
    vfs.write(&format!("/data/{}", name), data)?;
    Ok(())
}

// With filesystem
let mut fs = VRootFsBackend::new("/var/myapp")?;
save_data(&mut fs, "report.txt", b"...")?;

// With memory (for tests)
let mut mem = MemoryBackend::new();
save_data(&mut mem, "report.txt", b"...")?;
```

### With Container (anyfs-container)

```rust
use anyfs_container::{FilesContainer, ContainerBuilder};
use anyfs::SqliteBackend;

// Create tenant with 100 MB quota
let backend = SqliteBackend::create("tenant_123.db")?;
let mut container = ContainerBuilder::new(backend)
    .max_total_size(100 * 1024 * 1024)
    .build();

// Use it
container.write("/uploads/doc.pdf", &pdf_data)?;

// Check usage
println!("Used: {} bytes", container.usage().total_size);

// Move tenant = move file
std::fs::rename("tenant_123.db", "archive/tenant_123.db")?;
```

### Multi-Tenant Setup

```rust
use anyfs_container::{FilesContainer, ContainerBuilder};
use anyfs::VRootFsBackend;

fn create_tenant_container(tenant_id: &str) -> Result<FilesContainer<VRootFsBackend>, Error> {
    let root = format!("/data/tenants/{}", tenant_id);
    let backend = VRootFsBackend::new(&root)?;
    
    Ok(ContainerBuilder::new(backend)
        .max_total_size(100 * 1024 * 1024)
        .build())
}
```

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2025-12-22 | Initial design (graph-store trait) |
| 0.2.0 | 2025-12-22 | Restructured: two projects, path-based trait, three backends |
| 0.3.0 | 2025-12-22 | Three-crate structure, std::fs aligned API (20 methods), symlinks, hard links, permissions |
