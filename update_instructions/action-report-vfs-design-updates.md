# Action Report: VFS Design Manual Updates

**Date:** 2024-12-22  
**Purpose:** Comprehensive changes required for `https://github.com/DK26/vfs-design-manual`  
**Status:** Ready for implementation

---

## Executive Summary

This document outlines all changes needed to update the VFS design manual based on our design session. The key architectural shift is from a **monolithic design** to a **modular architecture** separating:

1. **Storage** (how data is stored — inodes, content)
2. **Semantics** (how paths are interpreted — Linux, Windows, custom)
3. **Container** (capacity limits, quotas)

This allows mixing any semantics with any storage backend.

---

## Table of Contents

1. [Architectural Changes](#1-architectural-changes)
2. [New Trait Definitions](#2-new-trait-definitions)
3. [Method Renaming (std::fs Alignment)](#3-method-renaming-stdfs-alignment)
4. [New Types](#4-new-types)
5. [Inode-Based Internal Model](#5-inode-based-internal-model)
6. [Simulated Filesystem Clarification](#6-simulated-filesystem-clarification)
7. [Lessons from sqlite-vfs](#7-lessons-from-sqlite-vfs)
8. [Shell Emulator Test App](#8-shell-emulator-test-app)
9. [FUSE Mounting Support](#9-fuse-mounting-support)
10. [Updated Crate Structure](#10-updated-crate-structure)
11. [Files to Create/Update](#11-files-to-createupdate)
12. [AGENTS.md Updates](#12-agentsmd-updates)

---

## 1. Architectural Changes

### 1.1 Old Architecture (Superseded)

```
┌─────────────────────────────────────────┐
│  User Application                       │
├─────────────────────────────────────────┤
│  anyfs-container                        │
├─────────────────────────────────────────┤
│  anyfs (VfsBackend trait)               │
├──────────┬──────────┬───────────────────┤
│ VRootFs  │  Memory  │  SQLite           │
└──────────┴──────────┴───────────────────┘
```

- Single `VfsBackend` trait combining storage + semantics
- No pluggable path resolution
- Linux semantics hardcoded

### 1.2 New Architecture (Current)

```
┌─────────────────────────────────────────┐
│  User Application                       │
├─────────────────────────────────────────┤
│  anyfs-container (quotas)               │
├─────────────────────────────────────────┤
│  Vfs<S, B>                              │  ← Combines semantics + storage
├───────────────────┬─────────────────────┤
│  FsSemantics      │  StorageBackend     │
│  (path rules)     │  (raw inode I/O)    │
├───────────────────┼─────────────────────┤
│ • LinuxSemantics  │ • MemoryStorage     │
│ • WindowsSemantics│ • SqliteStorage     │
│ • SimpleSemantics │ • RealFsStorage     │
│ • (Custom)        │ • (Custom)          │
└───────────────────┴─────────────────────┘
```

### 1.3 Rationale

| Benefit | Explanation |
|---------|-------------|
| **Modularity** | Mix any semantics with any storage |
| **Extensibility** | Add new semantics without touching storage |
| **Testability** | Use `SimpleSemantics` for fast tests |
| **Portability** | Windows semantics on Linux, vice versa |
| **Future-proof** | Custom semantics for special use cases |

---

## 2. New Trait Definitions

### 2.1 StorageBackend Trait

Raw inode operations. No path logic, no semantics.

```rust
/// Low-level storage operations on inodes.
/// Does NOT handle path resolution, permissions, or symlink following.
pub trait StorageBackend: Send {
    // ═══════════════════════════════════════════════════════════════
    // INODE OPERATIONS
    // ═══════════════════════════════════════════════════════════════
    
    /// Allocate a new inode
    fn create_inode(&mut self, data: InodeData) -> Result<InodeId, StorageError>;
    
    /// Get inode by ID
    fn get_inode(&self, id: InodeId) -> Result<InodeData, StorageError>;
    
    /// Update inode metadata
    fn update_inode(&mut self, id: InodeId, data: InodeData) -> Result<(), StorageError>;
    
    /// Delete inode (must have nlink=0)
    fn delete_inode(&mut self, id: InodeId) -> Result<(), StorageError>;
    
    /// Get root inode ID
    fn root(&self) -> InodeId;
    
    // ═══════════════════════════════════════════════════════════════
    // DIRECTORY OPERATIONS
    // ═══════════════════════════════════════════════════════════════
    
    /// Add entry to directory inode
    fn link(&mut self, parent: InodeId, name: &str, child: InodeId) -> Result<(), StorageError>;
    
    /// Remove entry from directory inode, returns removed child ID
    fn unlink(&mut self, parent: InodeId, name: &str) -> Result<InodeId, StorageError>;
    
    /// Lookup child by name in directory
    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, StorageError>;
    
    /// List all entries in directory
    fn read_dir(&self, dir: InodeId) -> Result<Vec<(String, InodeId)>, StorageError>;
    
    // ═══════════════════════════════════════════════════════════════
    // CONTENT OPERATIONS
    // ═══════════════════════════════════════════════════════════════
    
    /// Read entire file content
    fn read_content(&self, id: InodeId) -> Result<Vec<u8>, StorageError>;
    
    /// Read content byte range
    fn read_content_range(&self, id: InodeId, offset: u64, len: usize) -> Result<Vec<u8>, StorageError>;
    
    /// Write/replace file content
    fn write_content(&mut self, id: InodeId, data: &[u8]) -> Result<(), StorageError>;
    
    /// Append to file content
    fn append_content(&mut self, id: InodeId, data: &[u8]) -> Result<(), StorageError>;
    
    /// Truncate file content
    fn truncate_content(&mut self, id: InodeId, size: u64) -> Result<(), StorageError>;
    
    // ═══════════════════════════════════════════════════════════════
    // SYNC OPERATIONS (for durability)
    // ═══════════════════════════════════════════════════════════════
    
    /// Sync all pending writes to durable storage
    fn sync(&mut self) -> Result<(), StorageError>;
    
    /// Sync specific inode to durable storage
    fn sync_inode(&mut self, id: InodeId) -> Result<(), StorageError>;
}
```

### 2.2 FsSemantics Trait

Path resolution rules, permissions, naming conventions.

```rust
/// Filesystem semantics — defines behavior rules, not storage.
pub trait FsSemantics: Send + Sync {
    // ═══════════════════════════════════════════════════════════════
    // PATH HANDLING
    // ═══════════════════════════════════════════════════════════════
    
    /// Path separator character (e.g., '/' for Linux, '\\' for Windows)
    fn separator(&self) -> char;
    
    /// Alternative separator (e.g., '/' on Windows also works)
    fn alt_separator(&self) -> Option<char> { None }
    
    /// Is this path absolute?
    fn is_absolute(&self, path: &str) -> bool;
    
    /// Split path into components
    fn components<'a>(&self, path: &'a str) -> Vec<&'a str>;
    
    /// Normalize path (resolve `.`, `..`, redundant separators)
    fn normalize(&self, path: &str) -> String;
    
    /// Join two paths
    fn join(&self, base: &str, child: &str) -> String;
    
    /// Maximum path length in bytes
    fn max_path_len(&self) -> usize;
    
    /// Maximum single component (filename) length
    fn max_name_len(&self) -> usize;
    
    // ═══════════════════════════════════════════════════════════════
    // NAMING RULES
    // ═══════════════════════════════════════════════════════════════
    
    /// Validate filename (single component, not full path)
    fn validate_name(&self, name: &str) -> Result<(), SemanticError>;
    
    /// Is filesystem case-sensitive?
    fn case_sensitive(&self) -> bool;
    
    /// Normalize name for comparison (e.g., lowercase if case-insensitive)
    fn normalize_name(&self, name: &str) -> String {
        if self.case_sensitive() {
            name.to_string()
        } else {
            name.to_lowercase()
        }
    }
    
    // ═══════════════════════════════════════════════════════════════
    // SYMLINK HANDLING
    // ═══════════════════════════════════════════════════════════════
    
    /// Are symlinks supported?
    fn supports_symlinks(&self) -> bool;
    
    /// Maximum symlink resolution depth (prevents loops)
    fn max_symlink_depth(&self) -> u32;
    
    /// How to resolve `.` and `..`
    fn dot_resolution(&self) -> DotResolution;
    
    // ═══════════════════════════════════════════════════════════════
    // HARD LINK HANDLING
    // ═══════════════════════════════════════════════════════════════
    
    /// Are hard links supported?
    fn supports_hard_links(&self) -> bool;
    
    /// Can hard link directories? (usually false)
    fn can_hard_link_directories(&self) -> bool { false }
    
    // ═══════════════════════════════════════════════════════════════
    // PERMISSIONS
    // ═══════════════════════════════════════════════════════════════
    
    /// Check if operation is permitted
    fn check_permission(
        &self,
        inode: &InodeData,
        op: Operation,
        ctx: &OperationContext,
    ) -> Result<(), PermissionError>;
    
    /// Default mode for new files
    fn default_file_mode(&self) -> u32 { 0o644 }
    
    /// Default mode for new directories
    fn default_dir_mode(&self) -> u32 { 0o755 }
    
    // ═══════════════════════════════════════════════════════════════
    // SPECIAL BEHAVIOR
    // ═══════════════════════════════════════════════════════════════
    
    /// Are empty filenames allowed? (usually false)
    fn allow_empty_name(&self) -> bool { false }
    
    /// Can overwrite directory with file in rename?
    fn rename_overwrites(&self) -> RenameOverwrite;
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum DotResolution {
    /// Resolve `.` and `..` lexically (string manipulation)
    Lexical,
    /// Resolve `.` and `..` by following parent inode
    Inode,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum RenameOverwrite {
    /// Overwrite allowed if types match (file→file, dir→empty dir)
    SameType,
    /// Never overwrite, error if target exists
    Never,
    /// Always overwrite (delete target first)
    Always,
}

#[derive(Clone, Copy, Debug)]
pub enum Operation {
    Read,
    Write,
    Execute,
    Delete,
    List,
    CreateChild,
    Rename,
}

#[derive(Clone, Debug, Default)]
pub struct OperationContext {
    pub uid: u32,
    pub gid: u32,
    pub groups: Vec<u32>,
}
```

### 2.3 Vfs<S, B> Combinator

Combines semantics + storage into a usable VFS.

```rust
/// Virtual filesystem combining semantics and storage.
pub struct Vfs<S: FsSemantics, B: StorageBackend> {
    semantics: S,
    storage: B,
    context: OperationContext,
}

impl<S: FsSemantics, B: StorageBackend> Vfs<S, B> {
    pub fn new(semantics: S, storage: B) -> Self { ... }
    
    pub fn with_context(semantics: S, storage: B, ctx: OperationContext) -> Self { ... }
    
    pub fn set_context(&mut self, ctx: OperationContext) { ... }
    
    // Path resolution (internal)
    fn resolve(&self, path: &str) -> Result<InodeId, VfsError> { ... }
    fn resolve_parent(&self, path: &str) -> Result<(InodeId, String), VfsError> { ... }
    
    // Public API — matches std::fs naming
    pub fn read(&self, path: &str) -> Result<Vec<u8>, VfsError>;
    pub fn read_to_string(&self, path: &str) -> Result<String, VfsError>;
    pub fn read_range(&self, path: &str, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    pub fn write(&mut self, path: &str, data: &[u8]) -> Result<(), VfsError>;
    pub fn append(&mut self, path: &str, data: &[u8]) -> Result<(), VfsError>;
    pub fn exists(&self, path: &str) -> Result<bool, VfsError>;
    pub fn metadata(&self, path: &str) -> Result<Metadata, VfsError>;
    pub fn symlink_metadata(&self, path: &str) -> Result<Metadata, VfsError>;
    pub fn read_dir(&self, path: &str) -> Result<Vec<DirEntry>, VfsError>;
    pub fn read_link(&self, path: &str) -> Result<String, VfsError>;
    pub fn create_dir(&mut self, path: &str) -> Result<(), VfsError>;
    pub fn create_dir_all(&mut self, path: &str) -> Result<(), VfsError>;
    pub fn remove_file(&mut self, path: &str) -> Result<(), VfsError>;
    pub fn remove_dir(&mut self, path: &str) -> Result<(), VfsError>;
    pub fn remove_dir_all(&mut self, path: &str) -> Result<(), VfsError>;
    pub fn rename(&mut self, from: &str, to: &str) -> Result<(), VfsError>;
    pub fn copy(&mut self, from: &str, to: &str) -> Result<(), VfsError>;
    pub fn symlink(&mut self, original: &str, link: &str) -> Result<(), VfsError>;
    pub fn hard_link(&mut self, original: &str, link: &str) -> Result<(), VfsError>;
    pub fn set_permissions(&mut self, path: &str, perm: Permissions) -> Result<(), VfsError>;
    pub fn sync(&mut self) -> Result<(), VfsError>;
}
```

---

## 3. Method Renaming (std::fs Alignment)

All method names now match `std::fs`:

| Old Name | New Name | std::fs Equivalent |
|----------|----------|-------------------|
| `list()` | `read_dir()` | `std::fs::read_dir` |
| `mkdir()` | `create_dir()` | `std::fs::create_dir` |
| `mkdir_all()` | `create_dir_all()` | `std::fs::create_dir_all` |
| `remove()` | `remove_file()` | `std::fs::remove_file` |
| `remove()` | `remove_dir()` | `std::fs::remove_dir` |
| `remove_all()` | `remove_dir_all()` | `std::fs::remove_dir_all` |
| *(new)* | `read_to_string()` | `std::fs::read_to_string` |
| *(new)* | `symlink_metadata()` | `std::fs::symlink_metadata` |

### Total Method Count

| Category | Methods | Count |
|----------|---------|-------|
| Read | `read`, `read_to_string`, `read_range`, `exists`, `metadata`, `symlink_metadata`, `read_dir`, `read_link` | 8 |
| Write | `write`, `append`, `create_dir`, `create_dir_all`, `remove_file`, `remove_dir`, `remove_dir_all`, `rename`, `copy` | 9 |
| Links | `symlink`, `hard_link` | 2 |
| Permissions | `set_permissions` | 1 |
| Sync | `sync` | 1 |
| **Total** | | **21** |

---

## 4. New Types

### 4.1 InodeId and InodeData

```rust
/// Unique inode identifier within a storage backend.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct InodeId(pub u64);

/// Data stored for each inode.
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

#[derive(Clone, Debug)]
pub enum InodeKind {
    File,
    Directory,
    Symlink { target: String },
}
```

### 4.2 Metadata (User-Facing)

```rust
/// Metadata returned from `metadata()` and `symlink_metadata()`.
#[derive(Clone, Debug)]
pub struct Metadata {
    pub ino: u64,
    pub file_type: FileType,
    pub mode: u32,
    pub nlink: u64,
    pub size: u64,
    pub uid: u32,
    pub gid: u32,
    pub atime: Option<SystemTime>,
    pub mtime: Option<SystemTime>,
    pub ctime: Option<SystemTime>,
}

impl Metadata {
    pub fn is_file(&self) -> bool;
    pub fn is_dir(&self) -> bool;
    pub fn is_symlink(&self) -> bool;
    pub fn len(&self) -> u64 { self.size }
    pub fn permissions(&self) -> Permissions;
}
```

### 4.3 FileType

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum FileType {
    File,
    Directory,
    Symlink,
}

impl FileType {
    pub fn is_file(&self) -> bool;
    pub fn is_dir(&self) -> bool;
    pub fn is_symlink(&self) -> bool;
}
```

### 4.4 Permissions

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Default)]
pub struct Permissions {
    mode: u32,
}

impl Permissions {
    pub fn from_mode(mode: u32) -> Self;
    pub fn mode(&self) -> u32;
    pub fn readonly(&self) -> bool;
    pub fn set_readonly(&mut self, readonly: bool);
    
    // Unix helpers
    pub fn owner_read(&self) -> bool;
    pub fn owner_write(&self) -> bool;
    pub fn owner_execute(&self) -> bool;
    // ... group, other
}
```

### 4.5 DirEntry

```rust
#[derive(Clone, Debug)]
pub struct DirEntry {
    pub name: String,
    pub ino: u64,
    pub file_type: FileType,
}
```

### 4.6 Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum StorageError {
    #[error("inode not found: {0:?}")]
    InodeNotFound(InodeId),
    
    #[error("entry not found in directory: {0}")]
    EntryNotFound(String),
    
    #[error("entry already exists: {0}")]
    EntryExists(String),
    
    #[error("not a directory: {0:?}")]
    NotADirectory(InodeId),
    
    #[error("not a file: {0:?}")]
    NotAFile(InodeId),
    
    #[error("directory not empty: {0:?}")]
    DirectoryNotEmpty(InodeId),
    
    #[error("inode still has links: nlink={0}")]
    InodeHasLinks(u64),
    
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("backend error: {0}")]
    Backend(String),
}

#[derive(Debug, thiserror::Error)]
pub enum SemanticError {
    #[error("invalid path: {0}")]
    InvalidPath(String),
    
    #[error("invalid name: {0}")]
    InvalidName(String),
    
    #[error("name too long: {len} > {max}")]
    NameTooLong { len: usize, max: usize },
    
    #[error("path too long: {len} > {max}")]
    PathTooLong { len: usize, max: usize },
    
    #[error("reserved name: {0}")]
    ReservedName(String),
    
    #[error("symlinks not supported")]
    SymlinksNotSupported,
    
    #[error("hard links not supported")]
    HardLinksNotSupported,
}

#[derive(Debug, thiserror::Error)]
pub enum PermissionError {
    #[error("permission denied")]
    Denied,
    
    #[error("operation not permitted")]
    NotPermitted,
}

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
    
    #[error("not a symlink: {0}")]
    NotASymlink(String),
    
    #[error("is a directory: {0}")]
    IsADirectory(String),
    
    #[error("directory not empty: {0}")]
    DirectoryNotEmpty(String),
    
    #[error("symlink loop detected: {0}")]
    SymlinkLoop(String),
    
    #[error("invalid UTF-8")]
    InvalidUtf8(#[from] std::string::FromUtf8Error),
    
    #[error(transparent)]
    Storage(#[from] StorageError),
    
    #[error(transparent)]
    Semantic(#[from] SemanticError),
    
    #[error(transparent)]
    Permission(#[from] PermissionError),
    
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}
```

---

## 5. Inode-Based Internal Model

### 5.1 Why Inodes?

The old design used paths directly:
```rust
// Old: path as key
HashMap<VirtualPath, Entry>
```

New design uses inodes internally:
```rust
// New: inode ID as key
HashMap<InodeId, InodeData>
HashMap<(InodeId, String), InodeId>  // parent + name → child
```

### 5.2 Benefits

| Benefit | Explanation |
|---------|-------------|
| **Hard links** | Multiple paths → same inode → same content |
| **Symlink resolution** | Resolve target, get inode, done |
| **Rename efficiency** | Change parent entry, inode unchanged |
| **FUSE support** | FUSE uses inode numbers |
| **Identity checking** | Same inode = same file |
| **nlink tracking** | Count of hard links to inode |

### 5.3 Path Resolution Algorithm

```rust
fn resolve(&self, path: &str) -> Result<InodeId, VfsError> {
    let normalized = self.semantics.normalize(path);
    let mut current = self.storage.root();
    let mut symlink_depth = 0;
    
    for component in self.semantics.components(&normalized) {
        if component == "." {
            continue;
        }
        
        if component == ".." {
            match self.semantics.dot_resolution() {
                DotResolution::Lexical => continue,  // Already handled in normalize
                DotResolution::Inode => {
                    let inode = self.storage.get_inode(current)?;
                    if let Some(parent) = inode.parent {
                        current = parent;
                    }
                    continue;
                }
            }
        }
        
        // Check current is directory
        let inode = self.storage.get_inode(current)?;
        if inode.kind != InodeKind::Directory {
            return Err(VfsError::NotADirectory(path.to_string()));
        }
        
        // Check permission to traverse
        self.semantics.check_permission(&inode, Operation::Execute, &self.context)?;
        
        // Lookup child
        let name = self.semantics.normalize_name(component);
        current = self.storage.lookup(current, &name)?;
        
        // Follow symlink if needed
        loop {
            let child_inode = self.storage.get_inode(current)?;
            match &child_inode.kind {
                InodeKind::Symlink { target } => {
                    if !self.semantics.supports_symlinks() {
                        return Err(SemanticError::SymlinksNotSupported.into());
                    }
                    symlink_depth += 1;
                    if symlink_depth > self.semantics.max_symlink_depth() {
                        return Err(VfsError::SymlinkLoop(path.to_string()));
                    }
                    // Resolve target (relative to current directory or absolute)
                    current = self.resolve(target)?;
                }
                _ => break,
            }
        }
    }
    
    Ok(current)
}
```

---

## 6. Simulated Filesystem Clarification

### 6.1 Key Principle

> **The VFS API is an abstraction. It does NOT imply real OS calls.**

### 6.2 Storage Backend Behavior

| Backend | `write()` | `symlink()` | `hard_link()` |
|---------|-----------|-------------|---------------|
| **MemoryStorage** | Stores bytes in HashMap | Stores target in `InodeKind::Symlink` | Two inodes share `ContentId` |
| **SqliteStorage** | Inserts row in DB | Inserts row with target | Two rows reference same content |
| **RealFsStorage** | Calls `std::fs::write` | Calls `std::os::unix::fs::symlink` | Calls `std::fs::hard_link` |

### 6.3 Documentation Language

**Avoid:**
- "Creates a symlink"
- "Writes to filesystem"

**Prefer:**
- "Records a symlink in the storage backend"
- "Stores data in the backend"

**Or be explicit:**
- "Creates a symlink (real OS symlink for RealFsStorage, simulated for others)"

---

## 7. Lessons from sqlite-vfs

Reviewed `https://github.com/rkusa/sqlite-vfs` for potential problems.

### 7.1 sqlite-vfs Limitations

| Limitation | Their Status |
|------------|--------------|
| WAL not supported | In progress |
| Memory mapping not supported | Not planned |
| Directory sync not supported | Not planned |
| Sector size fixed (1024) | Hardcoded |
| Uses unsafe, not peer-reviewed | Known risk |

### 7.2 Lessons for anyfs

| Lesson | Action for anyfs |
|--------|------------------|
| Locking is complex | Consider `lock()` / `unlock()` methods |
| Sync/durability matters | Added `sync()` to `StorageBackend` |
| WAL is useful | Consider journal/WAL for SqliteStorage |
| VFS is hard, start MVP | Focus on core features first |
| Document unsafe clearly | Keep unsafe minimal, well-documented |

### 7.3 Features to Consider Later

| Feature | Priority | Notes |
|---------|----------|-------|
| File locking (`lock`, `unlock`) | Medium | For concurrent access |
| Memory mapping (`mmap`) | Low | Performance optimization |
| Extended attributes | Low | `getxattr`, `setxattr` |
| ACLs | Low | Beyond Unix permissions |

---

## 8. Shell Emulator Test App

### 8.1 Purpose

A virtual shell operating entirely through anyfs. Great for:
- Testing the VFS implementation
- Demonstrating API usage
- Interactive exploration

### 8.2 Design

```rust
pub struct VirtualShell<S: FsSemantics, B: StorageBackend> {
    vfs: Vfs<S, B>,
    cwd: String,
    env: HashMap<String, String>,
    commands: HashMap<String, Box<dyn ShellCommand<S, B>>>,
    history: Vec<String>,
}

pub trait ShellCommand<S: FsSemantics, B: StorageBackend>: Send + Sync {
    fn name(&self) -> &str;
    fn execute(
        &self,
        shell: &mut VirtualShell<S, B>,
        args: &[&str],
        stdin: &mut dyn Read,
        stdout: &mut dyn Write,
        stderr: &mut dyn Write,
    ) -> i32;
}
```

### 8.3 Built-in Commands

| Command | VFS Method |
|---------|------------|
| `ls` | `read_dir()` |
| `cat` | `read()` |
| `echo > file` | `write()` |
| `mkdir` | `create_dir()` |
| `mkdir -p` | `create_dir_all()` |
| `rm` | `remove_file()` |
| `rmdir` | `remove_dir()` |
| `rm -rf` | `remove_dir_all()` |
| `ln -s` | `symlink()` |
| `ln` | `hard_link()` |
| `mv` | `rename()` |
| `cp` | `copy()` |
| `cd` | Update `cwd` |
| `pwd` | Print `cwd` |
| `stat` | `metadata()` |
| `readlink` | `read_link()` |
| `chmod` | `set_permissions()` |
| `touch` | `write()` or update mtime |

### 8.4 Custom Commands

```rust
// Register custom command
shell.register_command(MyCommand::new());

// Or via closure
shell.register_fn("greet", |shell, args, stdin, stdout, stderr| {
    writeln!(stdout, "Hello from anyfs!").unwrap();
    0  // exit code
});
```

---

## 9. FUSE Mounting Support

### 9.1 Feasibility

With inode-based model, FUSE mounting is possible.

### 9.2 Adapter Design

```rust
use fuser::Filesystem;

pub struct AnyFsFuse<S: FsSemantics, B: StorageBackend> {
    vfs: Vfs<S, B>,
    // FUSE uses u64 inode numbers — our InodeId works directly
}

impl<S: FsSemantics, B: StorageBackend> Filesystem for AnyFsFuse<S, B> {
    fn lookup(&mut self, _req: &Request, parent: u64, name: &OsStr, reply: ReplyEntry) {
        let parent_id = InodeId(parent);
        match self.vfs.storage.lookup(parent_id, name.to_str().unwrap()) {
            Ok(child_id) => {
                let inode = self.vfs.storage.get_inode(child_id).unwrap();
                reply.entry(&TTL, &to_fuse_attr(&inode), 0);
            }
            Err(_) => reply.error(ENOENT),
        }
    }
    
    fn read(&mut self, _req: &Request, ino: u64, ...) {
        let id = InodeId(ino);
        match self.vfs.storage.read_content(id) {
            Ok(data) => reply.data(&data[offset..]),
            Err(_) => reply.error(EIO),
        }
    }
    
    // ... other FUSE methods
}
```

### 9.3 Optional Low-Level Trait

For more efficient FUSE:

```rust
/// Low-level inode operations for FUSE efficiency.
pub trait InodeOps: StorageBackend {
    fn getattr(&self, ino: u64) -> Result<InodeData, StorageError>;
    fn read_by_ino(&self, ino: u64, offset: u64, size: u32) -> Result<Vec<u8>, StorageError>;
    fn write_by_ino(&mut self, ino: u64, offset: u64, data: &[u8]) -> Result<u32, StorageError>;
}
```

---

## 10. Updated Crate Structure

```
anyfs-core/                    # Core traits + types
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── storage.rs             # StorageBackend trait
    ├── semantics.rs           # FsSemantics trait
    ├── vfs.rs                 # Vfs<S, B> combinator
    ├── inode.rs               # InodeId, InodeData, InodeKind
    ├── types.rs               # Metadata, FileType, DirEntry, Permissions
    └── error.rs               # StorageError, SemanticError, VfsError

anyfs-semantics/               # Pre-built semantics
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── linux.rs               # LinuxSemantics
    ├── windows.rs             # WindowsSemantics
    └── simple.rs              # SimpleSemantics (no permissions, no symlinks)

anyfs-storage/                 # Pre-built storage backends
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── memory.rs              # MemoryStorage
    ├── sqlite.rs              # SqliteStorage
    └── realfs.rs              # RealFsStorage (via strict-path)

anyfs/                         # Convenience crate
├── Cargo.toml
└── src/
    └── lib.rs                 # Re-exports + type aliases + defaults

anyfs-container/               # Quota wrapper
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── container.rs           # FilesContainer<V: VfsBackend>
    ├── builder.rs             # ContainerBuilder
    ├── limits.rs              # CapacityLimits
    └── error.rs               # ContainerError

anyfs-shell/                   # Shell emulator (optional)
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── shell.rs               # VirtualShell
    ├── command.rs             # ShellCommand trait
    └── builtins/              # Built-in commands
        ├── ls.rs
        ├── cat.rs
        └── ...

anyfs-fuse/                    # FUSE adapter (optional)
├── Cargo.toml
└── src/
    ├── lib.rs
    └── adapter.rs             # AnyFsFuse
```

---

## 11. Files to Create/Update

### 11.1 Create New Files

| File | Description |
|------|-------------|
| `book/src/architecture/modular-design.md` | New modular architecture |
| `book/src/traits/storage-backend.md` | StorageBackend trait docs |
| `book/src/traits/fs-semantics.md` | FsSemantics trait docs |
| `book/src/semantics/linux.md` | LinuxSemantics |
| `book/src/semantics/windows.md` | WindowsSemantics |
| `book/src/semantics/simple.md` | SimpleSemantics |
| `book/src/internal/inode-model.md` | Inode-based internal model |
| `book/src/clarifications/simulated-fs.md` | Simulated vs real I/O |
| `book/src/future/shell-emulator.md` | Shell app design |
| `book/src/future/fuse-mounting.md` | FUSE adapter design |

### 11.2 Update Existing Files

| File | Changes |
|------|---------|
| `README.md` | Update architecture diagram, crate list |
| `AGENTS.md` | Major updates (see section 12) |
| `book/src/vfs-design.md` | New trait definitions |
| `book/src/api-reference.md` | std-aligned method names |
| All backend docs | Add inode internal model |

### 11.3 Archive/Deprecate

| File | Action |
|------|--------|
| Old trait definitions | Mark as superseded |
| Old method names | Add "renamed" notes |

---

## 12. AGENTS.md Updates

### 12.1 Replace Trait Definition

**Old (remove):**
```rust
pub trait VfsBackend: Send {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn list(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;
    fn mkdir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    // ... 13 methods
}
```

**New (add):**
```rust
// Two-layer architecture
pub trait StorageBackend: Send { /* inode ops */ }
pub trait FsSemantics: Send + Sync { /* path rules */ }
pub struct Vfs<S: FsSemantics, B: StorageBackend> { /* combines both */ }
```

### 12.2 Update Method Names

Add to "Common Mistakes to Avoid":

```markdown
### ❌ WRONG: Old method names

```rust
vfs.list(path)?;       // WRONG
vfs.mkdir(path)?;      // WRONG
vfs.remove(path)?;     // WRONG
```

```rust
vfs.read_dir(path)?;   // CORRECT
vfs.create_dir(path)?; // CORRECT
vfs.remove_file(path)?; // CORRECT (or remove_dir())
```
```

### 12.3 Add Modular Architecture Section

```markdown
## Modular Architecture

The VFS separates:

1. **StorageBackend** — Raw inode storage (no path logic)
2. **FsSemantics** — Path resolution rules (no storage)
3. **Vfs<S, B>** — Combines both

This allows any semantics + any storage combination.
```

### 12.4 Update Quick Reference

| Question | Answer |
|----------|--------|
| How many traits? | Two: `StorageBackend` + `FsSemantics` |
| How are they combined? | `Vfs<S, B>` struct |
| What semantics are available? | `LinuxSemantics`, `WindowsSemantics`, `SimpleSemantics` |
| What storage backends? | `MemoryStorage`, `SqliteStorage`, `RealFsStorage` |
| Total public API methods? | 21 (std::fs aligned) |
| Internal model? | Inode-based |

---

## Summary of All Changes

| Category | Old | New |
|----------|-----|-----|
| **Architecture** | Single `VfsBackend` trait | Two traits: `StorageBackend` + `FsSemantics` |
| **Path handling** | Hardcoded Linux-like | Pluggable via `FsSemantics` |
| **Internal model** | Path-keyed HashMap | Inode-keyed with directory entries |
| **Method names** | Custom (`list`, `mkdir`) | std::fs aligned (`read_dir`, `create_dir`) |
| **Method count** | 13 | 21 |
| **Symlinks** | Deferred | Included |
| **Hard links** | Deferred | Included |
| **Permissions** | Deferred | Included |
| **FUSE support** | Not possible | Possible via inode model |

---

## Implementation Priority

### Phase 1: Core Refactoring
1. Define new traits (`StorageBackend`, `FsSemantics`)
2. Implement `Vfs<S, B>` combinator
3. Implement `LinuxSemantics`
4. Implement `MemoryStorage`

### Phase 2: Additional Backends
1. Implement `SqliteStorage`
2. Implement `RealFsStorage`

### Phase 3: Additional Semantics
1. Implement `WindowsSemantics`
2. Implement `SimpleSemantics`

### Phase 4: Extended Features
1. `anyfs-container` with new architecture
2. `anyfs-shell` test app
3. `anyfs-fuse` adapter

---

## Document History

| Date | Author | Changes |
|------|--------|---------|
| 2024-12-22 | Claude | Initial action report |
