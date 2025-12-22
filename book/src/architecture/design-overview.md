# VFS Ecosystem — Design Document

**Version:** 0.2.0  
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

```rust
use std::path::Path;

pub trait VfsBackend: Send {
    /// Read entire file contents
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    
    /// Read a byte range from a file
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    
    /// Write file contents (create or overwrite)
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    
    /// Append to file
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    
    /// Check if path exists
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    
    /// Get metadata
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    
    /// List directory contents
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    
    /// Create directory (parent must exist)
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    
    /// Create directory and all parents
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    
    /// Remove file or empty directory
    fn remove(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    
    /// Remove directory and all contents
    fn remove_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    
    /// Rename/move
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    
    /// Copy file
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
}
```

**Path type rationale:** `impl AsRef<Path>` is idiomatic Rust for filesystem APIs. It accepts `&str`, `String`, `&Path`, `PathBuf`, etc. Each backend handles paths according to its storage:
- `VRootFsBackend`: passes to `strict-path` directly
- `MemoryBackend`: uses `PathBuf` as HashMap key
- `SqliteBackend`: converts to string internally (its concern, not the API's)

### 3.3 Types

```rust
/// File metadata
#[derive(Clone, Debug)]
pub struct Metadata {
    pub file_type: FileType,
    pub size: u64,
    pub created: Option<SystemTime>,
    pub modified: Option<SystemTime>,
}

/// Type of filesystem entry
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum FileType {
    File,
    Directory,
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
    NotFound(String),
    
    #[error("already exists: {0}")]
    AlreadyExists(String),
    
    #[error("not a file: {0}")]
    NotAFile(String),
    
    #[error("not a directory: {0}")]
    NotADirectory(String),
    
    #[error("directory not empty: {0}")]
    DirectoryNotEmpty(String),
    
    #[error("invalid path: {0}")]
    InvalidPath(String),
    
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
│   ├── lib.rs          # Re-exports
│   ├── backend.rs      # VfsBackend trait
│   ├── types.rs        # Metadata, DirEntry, FileType
│   ├── error.rs        # VfsError
│   │
│   ├── vrootfs/        # Filesystem backend via strict-path
│   │   └── mod.rs
│   │
│   ├── memory/         # In-memory backend
│   │   └── mod.rs
│   │
│   └── sqlite/         # SQLite backend
│       ├── mod.rs
│       └── schema.rs
│
└── tests/
    └── conformance.rs  # Tests all backends identically
```

### 3.6 Dependencies

```toml
[dependencies]
thiserror = "1"

[dependencies.strict-path]
version = "..."
optional = true

[dependencies.rusqlite]
version = "0.31"
features = ["bundled"]
optional = true

[features]
default = ["vrootfs", "memory"]
fs = ["strict-path"]
memory = []
sqlite = ["rusqlite"]
full = ["vrootfs", "memory", "sqlite"]
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

pub struct VRootFsBackend {
    vroot: VirtualRoot,
}

impl VRootFsBackend {
    /// Create backend with given directory as virtual root.
    /// The directory is created if it doesn't exist.
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
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        let vpath = self.vroot.virtual_join(path.as_ref())
            .map_err(|e| VfsError::InvalidPath(e.to_string()))?;
        vpath.read().map_err(VfsError::Io)
    }
    
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        let vpath = self.vroot.virtual_join(path.as_ref())
            .map_err(|e| VfsError::InvalidPath(e.to_string()))?;
        vpath.write(data).map_err(VfsError::Io)
    }
    
    // ... delegate other methods via VirtualRoot
}
```

**Key behavior:**
- Paths like `/etc/passwd` are **clamped** to `root_dir/etc/passwd`
- Symlink escapes are prevented by `strict-path`
- Uses real filesystem for storage

### 5.2 MemoryBackend

In-memory storage using HashMap.

```rust
use std::collections::HashMap;
use std::path::{Path, PathBuf};

pub struct MemoryBackend {
    entries: HashMap<PathBuf, Entry>,
}

enum Entry {
    File { data: Vec<u8>, modified: SystemTime },
    Directory { modified: SystemTime },
}

impl MemoryBackend {
    pub fn new() -> Self {
        let mut entries = HashMap::new();
        // Root directory always exists
        entries.insert(PathBuf::from("/"), Entry::Directory {
            modified: SystemTime::now(),
        });
        Self { entries }
    }
}

impl VfsBackend for MemoryBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        let key = normalize_path(path.as_ref());
        match self.entries.get(&key) {
            Some(Entry::File { data, .. }) => Ok(data.clone()),
            Some(Entry::Directory { .. }) => Err(VfsError::NotAFile(key.display().to_string())),
            None => Err(VfsError::NotFound(key.display().to_string())),
        }
    }
    
    // ... implement other methods
}

fn normalize_path(path: &Path) -> PathBuf {
    // Normalize the path (handle . and .., ensure absolute)
    // Implementation detail
    todo!()
}
```

**Key behavior:**
- Paths are normalized and used as HashMap keys
- No persistence — data lost when dropped
- Useful for testing

### 5.3 SqliteBackend

Single-file storage using SQLite.

```rust
use rusqlite::Connection;

pub struct SqliteBackend {
    conn: Connection,
}

impl SqliteBackend {
    pub fn create(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open(path: impl AsRef<Path>) -> Result<Self, VfsError>;
    pub fn open_in_memory() -> Result<Self, VfsError>;
}
```

**Schema:**

```sql
CREATE TABLE entries (
    id INTEGER PRIMARY KEY,
    path TEXT NOT NULL UNIQUE,
    is_dir INTEGER NOT NULL,
    size INTEGER NOT NULL DEFAULT 0,
    created_at INTEGER,
    modified_at INTEGER
);

CREATE TABLE content (
    entry_id INTEGER PRIMARY KEY,
    data BLOB NOT NULL,
    FOREIGN KEY (entry_id) REFERENCES entries(id) ON DELETE CASCADE
);

-- For large files, could use chunking:
-- CREATE TABLE chunks (
--     entry_id INTEGER NOT NULL,
--     chunk_index INTEGER NOT NULL,
--     data BLOB NOT NULL,
--     PRIMARY KEY (entry_id, chunk_index)
-- );

CREATE INDEX idx_entries_path ON entries(path);
```

**Key behavior:**
- Entire container is a single `.db` file
- Portable — copy file to move container
- Supports large files via chunking (optional)

---

## 6. Open Design Questions

### 6.1 Path Type in Trait — ✅ RESOLVED

**Decision:** `impl AsRef<Path>`

**Rationale:**
- Idiomatic Rust for filesystem APIs
- Accepts `&str`, `String`, `&Path`, `PathBuf`, `&OsStr`, etc.
- `Path` wraps `OsStr` — handles OS-specific semantics naturally
- `VRootFsBackend` uses real filesystem — OS paths are natural
- SQLite is secondary use case — shouldn't dictate API design

**How backends handle it:**
- `VRootFsBackend`: Pass directly to `strict-path::VirtualRoot`
- `MemoryBackend`: Convert to `PathBuf` as HashMap key  
- `SqliteBackend`: Convert to string internally (its internal concern)

### 6.2 Error Path Representation

**Question:** Should `VfsError` variants store `String` or `PathBuf`?

```rust
// Option A: String
NotFound(String),

// Option B: PathBuf
NotFound(PathBuf),

// Option C: Box<Path> (owned, unsized)
NotFound(Box<Path>),
```

**Current leaning:** `String` — simpler, works for both real paths and SQLite keys.

### 6.3 Symlink Support

**Question:** Should the trait support symlinks?

- `VRootFsBackend`: The underlying filesystem has symlinks; `strict-path` handles them
- `MemoryBackend`: Could support symlinks as a stored target path
- `SqliteBackend`: Could store symlink targets

**Current leaning:** Defer symlinks to v2. Keep v1 simple.

### 6.4 Backend Discovery of Usage

**Question:** How does `FilesContainer` know current usage when wrapping an existing backend?

Options:
1. Scan on construction (slow for large containers)
2. Backend provides `usage()` method
3. Store usage metadata separately

**Current leaning:** Backend provides optional `fn usage(&self) -> Option<CapacityUsage>`. If `None`, container scans.

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
