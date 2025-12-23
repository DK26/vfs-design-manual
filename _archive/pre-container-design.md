# Historical: VFS Container — Pre-Container Design Document

> This document is historical and reflects an early design iteration before the current architecture was finalized.
>
> The current AnyFS design uses three crates: `anyfs-traits`, `anyfs`, `anyfs-container` with a path-based `VfsBackend` trait (20 methods).
>
> For current design, see:
> - `book/src/architecture/adrs.md`
> - `book/src/architecture/design-overview.md`

**Version:** 0.1.0-draft
**Status:** Historical (superseded)
**Authors:** [TBD]
**Last Updated:** 2025-01-XX

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Goals and Non-Goals](#2-goals-and-non-goals)
3. [Architecture Overview](#3-architecture-overview)
4. [Core Abstractions](#4-core-abstractions)
5. [Backend Specification](#5-backend-specification)
6. [FilesContainer API](#6-filescontainer-api)
7. [Path Handling](#7-path-handling)
8. [Filesystem Features](#8-filesystem-features)
9. [Capacity Management](#9-capacity-management)
10. [Error Handling](#10-error-handling)
11. [Crate Structure](#11-crate-structure)
12. [Security Considerations](#12-security-considerations)
13. [Implementation Plan](#13-implementation-plan)
14. [Open Questions](#14-open-questions)
15. [Future Considerations](#15-future-considerations)
16. [Appendices](#appendices)

---

## 1. Executive Summary

### 1.1 What Is This?

VFS Container is a **storage-agnostic virtual filesystem abstraction** for Rust. It provides:

- A minimal trait (`StorageBackend`) that any storage system can implement
- A high-level API (`FilesContainer`) with familiar filesystem operations
- Reference implementations for SQLite, in-memory, and host filesystem backends
- Complete isolation from host OS filesystem semantics

### 1.2 Key Insight

Existing virtual filesystem abstractions conflate **filesystem semantics** with **storage operations**. They require backend implementers to understand paths, symlinks, permissions, and resolution logic.

This design separates concerns:

```
┌─────────────────────────────────────────────┐
│  FilesContainer (Filesystem Semantics)      │  ← paths, symlinks, resolution
├─────────────────────────────────────────────┤
│  StorageBackend (Storage Operations)        │  ← nodes, edges, blobs
└─────────────────────────────────────────────┘
```

The backend is a **typed graph store**. It knows nothing about filesystems. It stores nodes (metadata) and edges (parent-child relationships). All filesystem logic lives above it.

### 1.3 Primary Use Cases

- **Multi-tenant isolated storage**: Each tenant gets a completely isolated namespace
- **Portable data containers**: Single-file SQLite databases that can be moved, backed up, versioned
- **Safe sandbox environments**: No code execution, no host filesystem escape
- **Testing**: Deterministic, isolated filesystem for unit/integration tests
- **Embedded applications**: Portable data storage without OS dependencies

---

## 2. Goals and Non-Goals

### 2.1 Goals

| ID | Goal | Rationale |
|----|------|-----------|
| G1 | **Backend simplicity** | Implementing a new backend should take hours, not days. The trait surface must be minimal. |
| G2 | **Impossible to misuse** | Type system prevents invalid states. Transactions are mandatory. IDs are distinct types. |
| G3 | **Complete isolation** | Virtual paths never resolve to host filesystem. Escape is structurally impossible. |
| G4 | **Portable storage** | The SQLite backend produces a single file that works on any platform. |
| G5 | **Configurable semantics** | Features like symlinks, hard links, and permissions are opt-in. |
| G6 | **Resource limits** | First-class support for storage quotas, file size limits, and node counts. |
| G7 | **No execution** | This is a data container. No binary execution, no plugins, no shell semantics. |

### 2.2 Non-Goals

| ID | Non-Goal | Rationale |
|----|----------|-----------|
| NG1 | POSIX compliance | We deliberately avoid full POSIX semantics. Simpler is safer. |
| NG2 | Host filesystem mounting | No FUSE, no kernel integration. Purely userspace. |
| NG3 | Maximum performance | Correctness and safety over raw speed. Backends can optimize internally. |
| NG4 | Distributed consensus | Multi-node coordination is out of scope. Single-writer model. |
| NG5 | Streaming/async in core trait | Start synchronous. Async can be added as a separate trait later. |

---

## 3. Architecture Overview

### 3.1 Layer Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                         User Code                                │
└──────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                      FilesContainer<B>                           │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  • Path parsing & validation                               │  │
│  │  • Symlink resolution (with loop detection)                │  │
│  │  • Hard link reference counting                            │  │
│  │  • Permission checking (if enabled)                        │  │
│  │  • Capacity limit enforcement                              │  │
│  │  • Metadata management                                     │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                    StorageBackend (trait)                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  • Insert/get/update/delete nodes                          │  │
│  │  • Insert/get/delete edges                                 │  │
│  │  • Read/write content chunks                               │  │
│  │  • Transaction boundary                                    │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌───────────────────┬───────────────────┬──────────────────────────┐
│  SqliteBackend    │  MemoryBackend    │  FsBackend               │
│  (vfs-sqlite)     │  (vfs-memory)     │  (vfs-fs)                │
└───────────────────┴───────────────────┴──────────────────────────┘
```

### 3.2 Data Model

The virtual filesystem is modeled as a **directed graph**:

- **Nodes**: Files, directories, symlinks (each with metadata)
- **Edges**: Named parent→child relationships
- **Content**: Binary data, stored separately and referenced by ID

```
         ┌─────────────────┐
         │  Node (Dir)     │
         │  id: 1 (root)   │
         └────────┬────────┘
                  │
        ┌─────────┴─────────┐
        │ "etc"             │ "home"
        ▼                   ▼
  ┌───────────┐       ┌───────────┐
  │ Node (Dir)│       │ Node (Dir)│
  │ id: 2     │       │ id: 3     │
  └─────┬─────┘       └─────┬─────┘
        │ "config"          │ "user"
        ▼                   ▼
  ┌───────────┐       ┌───────────┐
  │ Node(File)│       │ Node (Dir)│
  │ id: 4     │       │ id: 5     │
  │ content:C1│       └───────────┘
  └───────────┘

  Content Store:
  ┌─────────────────────┐
  │ C1 → [chunk0, ...]  │
  └─────────────────────┘
```

### 3.3 Separation of Concerns

| Concern | Handled By |
|---------|------------|
| Path string parsing | `VirtualPath` (in `anyfs`) |
| Path traversal & resolution | `FilesContainer` |
| Symlink following | `FilesContainer` |
| Link count management | `FilesContainer` |
| Permission enforcement | `FilesContainer` |
| Capacity enforcement | `FilesContainer` |
| Node/edge storage | `StorageBackend` |
| Content chunking | `StorageBackend` (fixed chunk size) |
| Transaction atomicity | `StorageBackend` |
| Compression, encryption | Backend-specific (internal) |

---

## 4. Core Abstractions

### 4.1 Identity Types

All IDs are newtypes to prevent accidental misuse:

```rust
/// Unique identifier for a node (file, directory, symlink)
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct NodeId(pub(crate) u64);

/// Unique identifier for content (separate from node for deduplication)
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct ContentId(pub(crate) u64);

/// Reference to a specific chunk within content
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct ChunkId {
    pub content: ContentId,
    pub index: u32,
}

/// Validated filename component (never empty, no `/`, no `\0`, not `.` or `..`)
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
pub struct Name(String);

impl Name {
    pub fn new(s: impl AsRef<str>) -> Result<Self, InvalidNameError> {
        let s = s.as_ref();
        if s.is_empty() 
            || s.contains('/') 
            || s.contains('\0') 
            || s == "." 
            || s == ".." 
        {
            return Err(InvalidNameError(s.to_string()));
        }
        Ok(Name(s.to_string()))
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
}
```

### 4.2 Node Types

```rust
/// The kind of filesystem node
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum NodeKind {
    /// Regular file with content
    File {
        content_id: ContentId,
        size: u64,
    },
    
    /// Directory (children are edges, not stored here)
    Directory,
    
    /// Symbolic link to another path
    Symlink {
        target: String,  // Raw target path (may be relative)
    },
}

/// Complete node record
#[derive(Clone, Debug)]
pub struct NodeRecord {
    pub id: NodeId,
    pub kind: NodeKind,
    pub metadata: NodeMetadata,
}

/// Node metadata
#[derive(Clone, Debug, Default)]
pub struct NodeMetadata {
    pub created_at: Option<Timestamp>,
    pub modified_at: Option<Timestamp>,
    pub accessed_at: Option<Timestamp>,
    pub permissions: Option<Permissions>,
    pub link_count: u32,  // Number of hard links (edges pointing to this node)
    pub extended_attrs: HashMap<String, Vec<u8>>,
}

/// Unix-style permission bits (optional feature)
#[derive(Clone, Copy, Debug, Default)]
pub struct Permissions {
    pub mode: u16,  // e.g., 0o755
}

/// Timestamp in milliseconds since Unix epoch
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord)]
pub struct Timestamp(pub i64);
```

### 4.3 Edges

```rust
/// A named link from parent to child
#[derive(Clone, Debug)]
pub struct Edge {
    pub parent: NodeId,
    pub name: Name,
    pub child: NodeId,
}
```

### 4.4 Content Chunking

Content is stored in fixed-size chunks:

```rust
/// Chunk size: 64 KB
/// 
/// Rationale:
/// - Large enough to minimize chunk count for typical files
/// - Small enough for efficient partial reads
/// - Matches common filesystem block sizes
pub const CHUNK_SIZE: usize = 64 * 1024;
```

---

## 5. Backend Specification

### 5.1 Core Trait

The backend trait is deliberately minimal — **13 methods** total:

```rust
pub trait StorageBackend: Send {
    /// Execute operations within a transaction.
    /// All mutations MUST happen inside a transaction.
    /// Guarantees atomicity: either all operations commit or none do.
    fn transact<F, T>(&mut self, f: F) -> Result<T, BackendError>
    where
        F: FnOnce(&mut dyn Transaction) -> Result<T, BackendError>;

    /// Get a read-only snapshot for consistent reads.
    fn snapshot(&self) -> Box<dyn Snapshot + '_>;
}

/// Read-only operations
pub trait Snapshot: Send {
    fn get_node(&self, id: NodeId) -> Result<Option<NodeRecord>, BackendError>;
    fn get_edge(&self, parent: NodeId, name: &Name) -> Result<Option<NodeId>, BackendError>;
    fn list_edges(&self, parent: NodeId) -> Result<Vec<Edge>, BackendError>;
    fn read_chunk(&self, id: ChunkId) -> Result<Option<Vec<u8>>, BackendError>;
    
    /// Optional: batch read for efficiency
    fn get_nodes(&self, ids: &[NodeId]) -> Result<Vec<Option<NodeRecord>>, BackendError> {
        ids.iter().map(|id| self.get_node(*id)).collect()
    }
}

/// Read-write operations (only available inside transaction)
pub trait Transaction: Snapshot {
    fn insert_node(&mut self, node: &NodeRecord) -> Result<(), BackendError>;
    fn update_node(&mut self, id: NodeId, node: &NodeRecord) -> Result<(), BackendError>;
    fn delete_node(&mut self, id: NodeId) -> Result<(), BackendError>;

    fn insert_edge(&mut self, edge: &Edge) -> Result<(), BackendError>;
    fn delete_edge(&mut self, parent: NodeId, name: &Name) -> Result<(), BackendError>;

    fn write_chunk(&mut self, id: ChunkId, data: &[u8]) -> Result<(), BackendError>;
    fn delete_content(&mut self, id: ContentId) -> Result<(), BackendError>;
    
    /// Generate a new unique node ID
    fn next_node_id(&mut self) -> Result<NodeId, BackendError>;
    
    /// Generate a new unique content ID
    fn next_content_id(&mut self) -> Result<ContentId, BackendError>;
    
    /// Optional: batch operations for efficiency
    fn insert_nodes(&mut self, nodes: &[NodeRecord]) -> Result<(), BackendError> {
        for node in nodes {
            self.insert_node(node)?;
        }
        Ok(())
    }
    
    fn insert_edges(&mut self, edges: &[Edge]) -> Result<(), BackendError> {
        for edge in edges {
            self.insert_edge(edge)?;
        }
        Ok(())
    }
}
```

### 5.2 Lifecycle Trait

Separate from operations because not all backends support all lifecycle operations:

```rust
pub trait BackendLifecycle: Sized {
    type Config;
    
    /// Create new storage (fails if already exists)
    fn create(config: Self::Config) -> Result<Self, BackendError>;
    
    /// Open existing storage (fails if doesn't exist)
    fn open(config: Self::Config) -> Result<Self, BackendError>;
    
    /// Open existing or create new
    fn open_or_create(config: Self::Config) -> Result<Self, BackendError>;
    
    /// Destroy all data (optional — not all backends support this)
    fn destroy(self) -> Result<(), BackendError> {
        Err(BackendError::NotSupported)
    }
}
```

### 5.3 Backend Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum BackendError {
    #[error("node not found: {0:?}")]
    NodeNotFound(NodeId),
    
    #[error("edge not found: {parent:?}/{name}")]
    EdgeNotFound { parent: NodeId, name: String },
    
    #[error("edge already exists: {parent:?}/{name}")]
    EdgeAlreadyExists { parent: NodeId, name: String },
    
    #[error("node already exists: {0:?}")]
    NodeAlreadyExists(NodeId),
    
    #[error("content not found: {0:?}")]
    ContentNotFound(ContentId),
    
    #[error("storage full")]
    StorageFull,
    
    #[error("operation not supported by this backend")]
    NotSupported,
    
    #[error("transaction conflict")]
    TransactionConflict,
    
    #[error("backend I/O error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("backend internal error: {0}")]
    Internal(String),
}
```

### 5.4 What Backends Do NOT Handle

| Concern | Why Not Backend |
|---------|-----------------|
| Path resolution | Backend operates on NodeIds, not paths |
| Symlink following | Filesystem semantic, not storage |
| Hard link counting | Core increments/decrements, backend just stores |
| Permission checking | Policy, not storage |
| Capacity limits | Policy, not storage |
| Content chunking logic | Core splits/reassembles, backend stores fixed chunks |
| Name validation | `Name` type validates on construction |

---

## 6. FilesContainer API

### 6.1 Construction

```rust
impl<B: StorageBackend> FilesContainer<B> {
    /// Create with default configuration
    pub fn new(backend: B) -> Result<Self, VfsError>;
    
    /// Start building with configuration
    pub fn builder() -> ContainerBuilder<NoBackend>;
}

pub struct ContainerBuilder<B> {
    backend: B,
    config: ContainerConfig,
}

impl ContainerBuilder<NoBackend> {
    pub fn new() -> Self;
}

impl<B> ContainerBuilder<B> {
    /// Set the storage backend
    pub fn backend<NewB: StorageBackend>(self, backend: NewB) -> ContainerBuilder<NewB>;
    
    // Feature toggles
    pub fn symlinks(self, enabled: bool) -> Self;
    pub fn hard_links(self, enabled: bool) -> Self;
    pub fn permissions(self, enabled: bool) -> Self;
    pub fn extended_attrs(self, enabled: bool) -> Self;
    
    // Resolution limits
    pub fn max_path_depth(self, depth: u16) -> Self;
    pub fn max_symlink_resolution(self, max: u16) -> Self;
    
    // Capacity limits (fluent)
    pub fn max_total_size(self, bytes: u64) -> Self;
    pub fn max_file_size(self, bytes: u64) -> Self;
    pub fn max_node_count(self, count: u64) -> Self;
    pub fn max_dir_entries(self, count: u32) -> Self;
    pub fn max_name_length(self, length: u16) -> Self;
    
    // Capacity limits (struct)
    pub fn limits(self, limits: CapacityLimits) -> Self;
}

impl<B: StorageBackend> ContainerBuilder<B> {
    /// Build the container (initializes root if needed)
    pub fn build(self) -> Result<FilesContainer<B>, VfsError>;
}
```

### 6.2 File Operations

```rust
impl<B: StorageBackend> FilesContainer<B> {
    // ─────────────────────────────────────────────────────────────
    // Read Operations
    // ─────────────────────────────────────────────────────────────
    
    /// Check if a path exists
    pub fn exists(&self, path: &VirtualPath) -> bool;
    
    /// Get metadata for a path
    pub fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    
    /// Read entire file contents
    pub fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    
    /// Read a range of bytes from a file
    pub fn read_range(&self, path: &VirtualPath, offset: u64, len: u64) -> Result<Vec<u8>, VfsError>;
    
    /// List directory contents
    pub fn list(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;
    
    /// Read symlink target (does not follow the link)
    pub fn read_link(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError>;
    
    // ─────────────────────────────────────────────────────────────
    // Write Operations
    // ─────────────────────────────────────────────────────────────
    
    /// Create a directory
    pub fn mkdir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    
    /// Create a directory and all parent directories
    pub fn mkdir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    
    /// Write file contents (creates or overwrites)
    pub fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    
    /// Append to a file
    pub fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    
    /// Create a symbolic link
    pub fn symlink(&mut self, path: &VirtualPath, target: &VirtualPath) -> Result<(), VfsError>;
    
    /// Create a hard link
    pub fn hard_link(&mut self, path: &VirtualPath, target: &VirtualPath) -> Result<(), VfsError>;
    
    /// Delete a file or empty directory
    pub fn remove(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    
    /// Delete a directory and all contents
    pub fn remove_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    
    /// Rename/move a file or directory
    pub fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    
    /// Copy a file
    pub fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    
    /// Copy a directory recursively
    pub fn copy_all(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    
    // ─────────────────────────────────────────────────────────────
    // Metadata Operations
    // ─────────────────────────────────────────────────────────────
    
    /// Set permissions (if enabled)
    pub fn set_permissions(&mut self, path: &VirtualPath, perms: Permissions) -> Result<(), VfsError>;
    
    /// Set timestamps
    pub fn set_times(&mut self, path: &VirtualPath, times: Times) -> Result<(), VfsError>;
    
    /// Set extended attribute
    pub fn set_xattr(&mut self, path: &VirtualPath, name: &str, value: &[u8]) -> Result<(), VfsError>;
    
    /// Get extended attribute
    pub fn get_xattr(&self, path: &VirtualPath, name: &str) -> Result<Option<Vec<u8>>, VfsError>;
    
    /// Remove extended attribute
    pub fn remove_xattr(&mut self, path: &VirtualPath, name: &str) -> Result<(), VfsError>;
    
    /// List extended attributes
    pub fn list_xattrs(&self, path: &VirtualPath) -> Result<Vec<String>, VfsError>;
}
```

### 6.3 Import/Export

```rust
impl<B: StorageBackend> FilesContainer<B> {
    /// Import a file or directory from the host filesystem
    pub fn import_from_host(
        &mut self,
        host_path: &std::path::Path,
        virtual_dest: &VirtualPath,
    ) -> Result<ImportStats, VfsError>;
    
    /// Export a file or directory to the host filesystem
    pub fn export_to_host(
        &self,
        virtual_src: &VirtualPath,
        host_path: &std::path::Path,
    ) -> Result<ExportStats, VfsError>;
}

pub struct ImportStats {
    pub files_imported: u64,
    pub directories_imported: u64,
    pub bytes_imported: u64,
    pub symlinks_imported: u64,
}

pub struct ExportStats {
    pub files_exported: u64,
    pub directories_exported: u64,
    pub bytes_exported: u64,
    pub symlinks_exported: u64,
}
```

### 6.4 Capacity & Lifecycle

```rust
impl<B: StorageBackend> FilesContainer<B> {
    /// Get current capacity usage
    pub fn usage(&self) -> Result<CapacityUsage, VfsError>;
    
    /// Get configured capacity limits
    pub fn limits(&self) -> &CapacityLimits;
    
    /// Get remaining capacity
    pub fn remaining(&self) -> Result<CapacityRemaining, VfsError>;
    
    /// Remove all files and directories (reset to empty)
    pub fn clear(&mut self) -> Result<(), VfsError>;
}

impl<B: StorageBackend + BackendLifecycle> FilesContainer<B> {
    /// Destroy the container and its backing storage
    pub fn destroy(self) -> Result<(), VfsError>;
}
```

---

## 7. Path Handling

### 7.1 VirtualPath Type

Paths are **lexical only** — no host filesystem interaction:

```rust
/// A validated virtual filesystem path
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct VirtualPath {
    /// Normalized path string (always starts with `/`)
    inner: String,
}

impl VirtualPath {
    /// Parse and normalize a path
    pub fn new(s: impl AsRef<str>) -> Result<Self, PathError>;
    
    /// The root path
    pub fn root() -> Self;
    
    /// Get path components
    pub fn components(&self) -> impl Iterator<Item = &str>;
    
    /// Get parent path
    pub fn parent(&self) -> Option<VirtualPath>;
    
    /// Get filename (last component)
    pub fn file_name(&self) -> Option<&str>;
    
    /// Join with another path
    pub fn join(&self, other: impl AsRef<str>) -> Result<VirtualPath, PathError>;
    
    /// Check if path is absolute
    pub fn is_absolute(&self) -> bool;  // Always true after parsing
    
    /// Get path depth (e.g., `/a/b/c` = 3)
    pub fn depth(&self) -> usize;
    
    /// As string slice
    pub fn as_str(&self) -> &str;
}
```

### 7.2 Normalization Rules

| Input | Output | Rule |
|-------|--------|------|
| `/a/b/c` | `/a/b/c` | Already normalized |
| `a/b/c` | `/a/b/c` | Relative paths made absolute |
| `/a//b/c` | `/a/b/c` | Collapse multiple slashes |
| `/a/./b/c` | `/a/b/c` | Remove `.` components |
| `/a/b/../c` | `/a/c` | Resolve `..` lexically |
| `/a/b/c/..` | `/a/b` | Resolve trailing `..` |
| `/..` | `/` | Cannot escape root |
| `/../../../a` | `/a` | All `..` at root collapse |
| `/a/b/` | `/a/b` | Remove trailing slash |
| `/` | `/` | Root is valid |
| `` | Error | Empty path invalid |
| `/a/\0/b` | Error | Null bytes invalid |

### 7.3 Path Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum PathError {
    #[error("path is empty")]
    Empty,
    
    #[error("path contains null byte")]
    NullByte,
    
    #[error("path component is empty")]
    EmptyComponent,
    
    #[error("path depth {depth} exceeds limit of {limit}")]
    TooDeep { depth: usize, limit: usize },
    
    #[error("path component too long: {length} > {limit}")]
    ComponentTooLong { length: usize, limit: usize },
}
```

---

## 8. Filesystem Features

### 8.1 Feature Flags

All advanced features are **opt-in**:

```rust
pub struct ContainerConfig {
    /// Enable symbolic links (default: false)
    pub symlinks: bool,
    
    /// Enable hard links (default: false)
    pub hard_links: bool,
    
    /// Enable permission checking (default: false)
    pub permissions: bool,
    
    /// Enable extended attributes (default: false)
    pub extended_attrs: bool,
    
    // ... limits ...
}
```

### 8.2 Symbolic Links

When `symlinks: true`:

- `symlink(path, target)` creates a symlink
- `read_link(path)` returns the raw target
- Path resolution follows symlinks (with loop detection)
- `metadata(path)` follows symlinks; `symlink_metadata(path)` does not

**Resolution algorithm:**

```
resolve(path, follow_count = 0):
    if follow_count > max_symlink_resolution:
        return Error::SymlinkLoop
    
    node = lookup(path)
    
    if node is Symlink:
        target = node.target
        if target.is_relative():
            target = path.parent().join(target)
        return resolve(target, follow_count + 1)
    
    return node
```

### 8.3 Hard Links

When `hard_links: true`:

- `hard_link(path, target)` creates a new edge to an existing file node
- `link_count` in metadata tracks the number of edges
- File content is deleted only when `link_count` reaches 0
- Hard links to directories are **not supported** (preventing cycles)

### 8.4 Permissions

When `permissions: true`:

- Nodes store `mode: u16` (Unix-style)
- `set_permissions(path, perms)` modifies the mode
- All operations check permissions before proceeding
- Default mode for new files: `0o644`
- Default mode for new directories: `0o755`

**Note:** There is no concept of user/group — permissions are enforced uniformly. This is a simplification suitable for single-tenant or application-controlled scenarios.

### 8.5 Extended Attributes

When `extended_attrs: true`:

- Arbitrary key-value pairs attached to nodes
- Keys are strings (max 255 bytes, validated)
- Values are byte arrays (max configurable size)
- No namespacing (e.g., `user.`, `system.`) — all attrs are equal

---

## 9. Capacity Management

### 9.1 Limit Configuration

```rust
#[derive(Clone, Debug)]
pub struct CapacityLimits {
    /// Maximum total storage in bytes (sum of all file sizes)
    pub max_total_size: Option<u64>,
    
    /// Maximum single file size in bytes
    pub max_file_size: Option<u64>,
    
    /// Maximum number of nodes (files + directories + symlinks)
    pub max_node_count: Option<u64>,
    
    /// Maximum entries in a single directory
    pub max_dir_entries: Option<u32>,
    
    /// Maximum path depth (e.g., /a/b/c = depth 3)
    pub max_path_depth: Option<u16>,
    
    /// Maximum filename length in bytes
    pub max_name_length: Option<u16>,
}

impl Default for CapacityLimits {
    fn default() -> Self {
        Self {
            max_total_size: None,            // Unlimited
            max_file_size: Some(1 << 30),    // 1 GB
            max_node_count: None,            // Unlimited
            max_dir_entries: Some(10_000),   // 10K entries per dir
            max_path_depth: Some(64),        // 64 levels deep
            max_name_length: Some(255),      // 255 bytes
        }
    }
}
```

### 9.2 Usage Tracking

```rust
#[derive(Clone, Debug, Default)]
pub struct CapacityUsage {
    /// Total bytes used by file contents
    pub total_size: u64,
    
    /// Total number of nodes
    pub node_count: u64,
    
    /// Number of files
    pub file_count: u64,
    
    /// Number of directories
    pub directory_count: u64,
    
    /// Number of symlinks
    pub symlink_count: u64,
}

#[derive(Clone, Debug)]
pub struct CapacityRemaining {
    /// Bytes remaining (None = unlimited)
    pub bytes: Option<u64>,
    
    /// Nodes remaining (None = unlimited)
    pub nodes: Option<u64>,
    
    /// Quick check: can we write anything?
    pub can_write: bool,
}
```

### 9.3 Enforcement Points

| Operation | Limits Checked |
|-----------|----------------|
| `write()` | `max_file_size`, `max_total_size` |
| `append()` | `max_file_size`, `max_total_size` |
| `mkdir()` | `max_node_count`, `max_path_depth`, `max_dir_entries` (parent) |
| `symlink()` | `max_node_count`, `max_dir_entries` (parent) |
| `hard_link()` | `max_dir_entries` (parent) |
| `copy()` | Same as `write()` |
| `rename()` | `max_dir_entries` (destination parent), `max_name_length` |
| Path parsing | `max_path_depth`, `max_name_length` |

### 9.4 Capacity Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum CapacityError {
    #[error("total storage limit exceeded: {used} / {limit} bytes")]
    TotalSizeExceeded { used: u64, limit: u64 },
    
    #[error("file size {size} exceeds limit of {limit} bytes")]
    FileSizeExceeded { size: u64, limit: u64 },
    
    #[error("node count limit exceeded: {count} / {limit}")]
    NodeCountExceeded { count: u64, limit: u64 },
    
    #[error("directory has {count} entries, limit is {limit}")]
    DirEntriesExceeded { count: u32, limit: u32 },
    
    #[error("path depth {depth} exceeds limit of {limit}")]
    PathDepthExceeded { depth: u16, limit: u16 },
    
    #[error("name length {length} bytes exceeds limit of {limit}")]
    NameLengthExceeded { length: u16, limit: u16 },
}
```

---

## 10. Error Handling

### 10.1 Error Hierarchy

```rust
/// Top-level error type for FilesContainer operations
#[derive(Debug, thiserror::Error)]
pub enum VfsError {
    #[error("path error: {0}")]
    Path(#[from] PathError),
    
    #[error("capacity error: {0}")]
    Capacity(#[from] CapacityError),
    
    #[error("backend error: {0}")]
    Backend(#[from] BackendError),
    
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
    
    #[error("symlink loop detected at: {0}")]
    SymlinkLoop(VirtualPath),
    
    #[error("cannot create hard link to directory")]
    HardLinkToDirectory,
    
    #[error("feature not enabled: {0}")]
    FeatureNotEnabled(&'static str),
    
    #[error("permission denied: {0}")]
    PermissionDenied(VirtualPath),
    
    #[error("invalid operation: {0}")]
    InvalidOperation(String),
}
```

### 10.2 Error Context

Consider adding context via extension trait:

```rust
pub trait VfsResultExt<T> {
    fn with_path(self, path: &VirtualPath) -> Result<T, VfsError>;
}

impl<T> VfsResultExt<T> for Result<T, BackendError> {
    fn with_path(self, path: &VirtualPath) -> Result<T, VfsError> {
        self.map_err(|e| match e {
            BackendError::NodeNotFound(_) => VfsError::NotFound(path.clone()),
            other => VfsError::Backend(other),
        })
    }
}
```

---

## 11. Crate Structure

### 11.1 Workspace Layout

```
anyfs-container/
├── Cargo.toml              # Workspace manifest
├── README.md
├── LICENSE
│
├── anyfs/               # Core traits and types
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── backend.rs      # StorageBackend, Transaction, Snapshot traits
│       ├── types.rs        # NodeId, ContentId, ChunkId, Name
│       ├── node.rs         # NodeKind, NodeRecord, NodeMetadata
│       ├── edge.rs         # Edge
│       ├── path.rs         # VirtualPath
│       ├── error.rs        # BackendError, PathError
│       └── limits.rs       # CapacityLimits
│
├── vfs-sqlite/             # SQLite backend
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── backend.rs      # SqliteBackend implementation
│       ├── schema.rs       # SQL schema definitions
│       └── migrations.rs   # Schema migrations
│
├── vfs-memory/             # In-memory backend
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs          # MemoryBackend implementation
│
├── vfs-fs/                 # Host filesystem backend
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs          # FsBackend implementation (via strict-path)
│
└── vfs/                    # High-level API (batteries included)
    ├── Cargo.toml
    └── src/
        ├── lib.rs
        ├── container.rs    # FilesContainer<B>
        ├── builder.rs      # ContainerBuilder
        ├── operations.rs   # File operation implementations
        ├── resolution.rs   # Path resolution, symlink following
        ├── import_export.rs # Host filesystem import/export
        └── error.rs        # VfsError, CapacityError
```

### 11.2 Dependency Graph

```
                    ┌─────────┐
                    │   vfs   │  (high-level API)
                    └────┬────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │vfs-sqlite│  │vfs-memory│  │  vfs-fs  │
    └────┬─────┘  └────┬─────┘  └────┬─────┘
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
                   ┌──────────┐
                   │ anyfs │  (traits, types)
                   └──────────┘
```

### 11.3 Feature Flags

**`vfs` crate:**

```toml
[features]
default = ["sqlite", "memory"]

# Include backends
sqlite = ["dep:vfs-sqlite"]
memory = ["dep:vfs-memory"]
fs = ["dep:vfs-fs"]

# Enable all filesystem features by default
full-features = []
```

### 11.4 Re-exports

The `vfs` crate re-exports everything for convenience:

```rust
// vfs/src/lib.rs

pub use anyfs::{
    // Traits
    StorageBackend, BackendLifecycle, Snapshot, Transaction,
    
    // Types
    NodeId, ContentId, ChunkId, Name, Edge,
    NodeKind, NodeRecord, NodeMetadata,
    VirtualPath, PathError,
    BackendError,
    CapacityLimits,
    CHUNK_SIZE,
};

// Container API
pub use crate::container::FilesContainer;
pub use crate::builder::ContainerBuilder;
pub use crate::error::{VfsError, CapacityError};

// Backends (conditional)
#[cfg(feature = "sqlite")]
pub use vfs_sqlite::SqliteBackend;

#[cfg(feature = "memory")]
pub use vfs_memory::MemoryBackend;

#[cfg(feature = "fs")]
pub use vfs_fs::FsBackend;
```

---

## 12. Security Considerations

### 12.1 Threat Model

| Threat | Mitigation |
|--------|------------|
| **Host filesystem escape** | Paths are lexical strings, never resolved against host FS. No `std::fs` in path handling. |
| **Path traversal via `..`** | Lexical normalization prevents escaping root. `/../../etc/passwd` → `/etc/passwd` (inside VFS). |
| **Symlink attacks** | Resolution has configurable max depth. Loops are detected. |
| **Denial of service** | Capacity limits prevent unbounded growth. |
| **Billion laughs (zip bomb)** | Content is stored as-is. No automatic decompression. |
| **Code execution** | No execution capability. Content is data only. |

### 12.2 Trust Boundaries

```
┌─────────────────────────────────────────────────────────┐
│                    Trusted Zone                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  FilesContainer (enforces policy)               │   │
│  │  ┌─────────────────────────────────────────┐    │   │
│  │  │  StorageBackend (dumb storage)          │    │   │
│  │  └─────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Backend-Specific Storage                   │
│  SQLite file / Memory / Controlled host directory       │
└─────────────────────────────────────────────────────────┘
```

### 12.3 Backend Security Notes

**SqliteBackend:**
- Database file should be in a controlled directory
- Consider SQLite encryption extension for sensitive data
- WAL mode recommended for crash safety

**MemoryBackend:**
- Data lost on drop — suitable for testing/ephemeral use
- No persistence security concerns

**FsBackend:**
- Uses `strict-path` to prevent escape from configured root
- Root directory must be carefully chosen
- Inherits host filesystem permissions

---

## 13. Implementation Plan

### 13.1 Phases

```
Phase 1: Core Foundation
├── anyfs crate
│   ├── [ ] Types: NodeId, ContentId, ChunkId, Name
│   ├── [ ] Node types: NodeKind, NodeRecord, NodeMetadata
│   ├── [ ] Edge type
│   ├── [ ] VirtualPath with normalization
│   ├── [ ] StorageBackend trait
│   ├── [ ] Transaction, Snapshot traits
│   ├── [ ] BackendLifecycle trait
│   ├── [ ] BackendError enum
│   └── [ ] CapacityLimits struct

Phase 2: Memory Backend
├── vfs-memory crate
│   ├── [ ] MemoryBackend struct
│   ├── [ ] In-memory node/edge storage
│   ├── [ ] In-memory content storage
│   ├── [ ] Transaction support (copy-on-write or mutex)
│   └── [ ] Unit tests

Phase 3: Basic Container
├── vfs crate (partial)
│   ├── [ ] FilesContainer struct
│   ├── [ ] ContainerBuilder
│   ├── [ ] Basic operations: read, write, mkdir, remove, list
│   ├── [ ] Path resolution (no symlinks yet)
│   ├── [ ] VfsError enum
│   └── [ ] Integration tests with MemoryBackend

Phase 4: SQLite Backend
├── vfs-sqlite crate
│   ├── [ ] Schema design
│   ├── [ ] SqliteBackend struct
│   ├── [ ] Transaction via SQLite transactions
│   ├── [ ] Lifecycle: create, open, open_or_create, destroy
│   ├── [ ] Migrations strategy
│   └── [ ] Integration tests

Phase 5: Advanced Features
├── vfs crate (complete)
│   ├── [ ] Symlink support (feature-gated)
│   ├── [ ] Symlink resolution with loop detection
│   ├── [ ] Hard link support (feature-gated)
│   ├── [ ] Link count management
│   ├── [ ] Permissions (feature-gated)
│   ├── [ ] Extended attributes (feature-gated)
│   └── [ ] Metadata operations

Phase 6: Capacity Management
├── vfs crate (capacity)
│   ├── [ ] Limit enforcement on all operations
│   ├── [ ] Usage tracking (aggregates in metadata)
│   ├── [ ] usage() and remaining() methods
│   └── [ ] CapacityError handling

Phase 7: Import/Export
├── vfs crate (import/export)
│   ├── [ ] import_from_host
│   ├── [ ] export_to_host
│   ├── [ ] Preserve metadata on import
│   └── [ ] Handle symlinks appropriately

Phase 8: Filesystem Backend
├── vfs-fs crate
│   ├── [ ] FsBackend using strict-path VirtualRoot
│   ├── [ ] Map nodes/edges to directories/files
│   ├── [ ] Content storage as files
│   └── [ ] Integration tests

Phase 9: Polish
├── Documentation
│   ├── [ ] API documentation (rustdoc)
│   ├── [ ] README with examples
│   ├── [ ] Architecture documentation
│   └── [ ] Security considerations
├── Testing
│   ├── [ ] Conformance test suite for backends
│   ├── [ ] Property-based tests (proptest)
│   ├── [ ] Fuzz testing for path parsing
│   └── [ ] Benchmarks
└── Release
    ├── [ ] Publish to crates.io
    └── [ ] Announce
```

### 13.2 Milestone Summary

| Milestone | Deliverable | Est. Effort |
|-----------|-------------|-------------|
| M1: Core Types | `anyfs` crate publishable | 1 week |
| M2: Memory Backend | `vfs-memory` crate with tests | 1 week |
| M3: Basic Container | Read/write/mkdir/remove working | 2 weeks |
| M4: SQLite Backend | `vfs-sqlite` crate with tests | 2 weeks |
| M5: Symlinks + Hard Links | Feature-gated, fully tested | 1 week |
| M6: Capacity + Permissions | Limits enforced, permissions checked | 1 week |
| M7: Import/Export | Host FS interop | 1 week |
| M8: FS Backend | `vfs-fs` crate with tests | 1 week |
| M9: Release | Docs, polish, publish | 1 week |

**Total estimated effort: ~11 weeks**

### 13.3 Testing Strategy

**Unit tests:**
- Every module has tests
- VirtualPath normalization: exhaustive edge cases
- Name validation: all invalid inputs rejected

**Backend conformance tests:**
- Macro generates test suite
- Every backend must pass identical tests

```rust
// anyfs/src/testing.rs

#[macro_export]
macro_rules! backend_conformance_tests {
    ($backend_factory:expr) => {
        #[test]
        fn test_insert_and_get_node() {
            let mut backend = $backend_factory();
            // ...
        }
        
        #[test]
        fn test_transaction_rollback() {
            // ...
        }
        
        // ... 50+ tests
    };
}

// vfs-memory/src/lib.rs
#[cfg(test)]
mod tests {
    anyfs::backend_conformance_tests!(|| MemoryBackend::new());
}
```

**Integration tests:**
- Full workflows through FilesContainer
- Symlink resolution edge cases
- Capacity limit enforcement

**Property-based tests (proptest):**
- Arbitrary path strings normalize correctly
- Arbitrary operations maintain invariants

**Fuzz tests:**
- Path parsing doesn't panic
- Backend operations don't panic

---

## 14. Open Questions

### 14.1 Decisions Needed

| ID | Question | Options | Recommendation |
|----|----------|---------|----------------|
| Q1 | Should symlink targets be validated on creation? | (a) Yes — must be valid path, (b) No — store as-is | **(b)** — matches POSIX, allows dangling symlinks |
| Q2 | Async support? | (a) Sync only, (b) Async trait now, (c) Separate async trait later | **(c)** — start simple, add when needed |
| Q3 | Should `copy()` preserve hard links within a tree? | (a) Yes — complex, (b) No — each copy is independent | **(b)** — simpler, predictable |
| Q4 | Content-addressable storage? | (a) Yes — deduplicate, (b) No — each file has own content | **(b) initially** — can add later as optimization |
| Q5 | How to handle concurrent access? | (a) Single-threaded only, (b) Mutex-based, (c) Backend decides | **(c)** — SQLite has its own locking |
| Q6 | Extended attr size limit? | Specific number | **64 KB per attr** — generous but bounded |
| Q7 | Should backends expose raw handles? | (a) Yes — escape hatch, (b) No — pure abstraction | **(a)** — power users need it |

### 14.2 Future Considerations Deferred

- Streaming read/write API
- Async backend trait
- Watch/notify on changes
- Compression (backend-internal)
- Encryption (backend-internal)
- Distributed backends
- FUSE integration (separate crate, not part of core)

---

## 15. Future Considerations

### 15.1 Async Support

When needed, add a parallel trait hierarchy:

```rust
#[async_trait]
pub trait AsyncStorageBackend: Send + Sync {
    async fn transact<F, T>(&mut self, f: F) -> Result<T, BackendError>
    where
        F: for<'a> FnOnce(&'a mut dyn AsyncTransaction) -> BoxFuture<'a, Result<T, BackendError>> + Send;
    
    async fn snapshot(&self) -> Box<dyn AsyncSnapshot + '_>;
}
```

FilesContainer becomes generic over sync/async.

### 15.2 Content-Addressable Storage

For deduplication:

```rust
pub struct ContentId(pub(crate) [u8; 32]);  // SHA-256

impl ContentId {
    pub fn from_data(data: &[u8]) -> Self {
        let hash = sha256(data);
        ContentId(hash)
    }
}
```

Backend stores content by hash. Multiple nodes can reference same ContentId.

### 15.3 Streaming API

For large files:

```rust
impl<B: StorageBackend> FilesContainer<B> {
    pub fn open_read(&self, path: &VirtualPath) -> Result<impl Read, VfsError>;
    pub fn open_write(&mut self, path: &VirtualPath) -> Result<impl Write, VfsError>;
}
```

Requires backend support for streaming chunks.

### 15.4 Change Notifications

```rust
impl<B: StorageBackend> FilesContainer<B> {
    pub fn watch(&self, path: &VirtualPath) -> Result<Receiver<ChangeEvent>, VfsError>;
}

pub enum ChangeEvent {
    Created(VirtualPath),
    Modified(VirtualPath),
    Deleted(VirtualPath),
    Renamed { from: VirtualPath, to: VirtualPath },
}
```

---

## Appendices

### A. SQLite Schema (Reference)

```sql
-- Schema version for migrations
CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY
);

-- Nodes (files, directories, symlinks)
CREATE TABLE nodes (
    id INTEGER PRIMARY KEY,
    kind INTEGER NOT NULL,          -- 0=file, 1=directory, 2=symlink
    content_id INTEGER,             -- FK to contents (files only)
    size INTEGER DEFAULT 0,         -- file size in bytes
    symlink_target TEXT,            -- symlink target path
    link_count INTEGER DEFAULT 1,   -- hard link count
    permissions INTEGER,            -- mode bits (optional)
    created_at INTEGER,             -- unix timestamp ms
    modified_at INTEGER,
    accessed_at INTEGER
);

-- Directory entries (edges)
CREATE TABLE edges (
    parent_id INTEGER NOT NULL REFERENCES nodes(id),
    name TEXT NOT NULL,
    child_id INTEGER NOT NULL REFERENCES nodes(id),
    PRIMARY KEY (parent_id, name)
);

CREATE INDEX idx_edges_child ON edges(child_id);

-- Content chunks
CREATE TABLE chunks (
    content_id INTEGER NOT NULL,
    chunk_index INTEGER NOT NULL,
    data BLOB NOT NULL,
    PRIMARY KEY (content_id, chunk_index)
);

-- Extended attributes
CREATE TABLE xattrs (
    node_id INTEGER NOT NULL REFERENCES nodes(id),
    name TEXT NOT NULL,
    value BLOB NOT NULL,
    PRIMARY KEY (node_id, name)
);

-- Container metadata (for tracking aggregates)
CREATE TABLE metadata (
    key TEXT PRIMARY KEY,
    value INTEGER
);
-- Keys: 'total_size', 'node_count', 'next_node_id', 'next_content_id'

-- Root node is always ID 1
INSERT INTO nodes (id, kind, link_count) VALUES (1, 1, 1);
INSERT INTO metadata (key, value) VALUES ('next_node_id', 2);
INSERT INTO metadata (key, value) VALUES ('next_content_id', 1);
INSERT INTO metadata (key, value) VALUES ('total_size', 0);
INSERT INTO metadata (key, value) VALUES ('node_count', 1);
```

### B. Memory Backend Data Structures (Reference)

```rust
pub struct MemoryBackend {
    nodes: HashMap<NodeId, NodeRecord>,
    edges: HashMap<(NodeId, String), NodeId>,
    children: HashMap<NodeId, HashSet<String>>,  // For list_edges
    chunks: HashMap<ChunkId, Vec<u8>>,
    next_node_id: u64,
    next_content_id: u64,
    metadata: ContainerMetadata,
}

struct ContainerMetadata {
    total_size: u64,
    node_count: u64,
}
```

### C. Glossary

| Term | Definition |
|------|------------|
| **Backend** | Implementation of `StorageBackend` trait; handles persistence |
| **Node** | A filesystem entity (file, directory, or symlink) |
| **Edge** | A named parent→child relationship |
| **Content** | Binary data stored separately from node metadata |
| **Chunk** | Fixed-size piece of content (64 KB) |
| **Container** | A `FilesContainer` instance; the high-level API |
| **Virtual Path** | A path within the virtual filesystem (not on host) |

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1.0-draft | 2025-XX-XX | — | Initial draft |

---

## Reviewers

- [ ] Reviewer 1: _________________
- [ ] Reviewer 2: _________________
- [ ] Reviewer 3: _________________

**Review deadline:** ____-__-__

---

*End of document.*
