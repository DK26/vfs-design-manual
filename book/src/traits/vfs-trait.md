# Vfs Trait (anyfs)

**The low-level filesystem interface for backend implementers**

---

## Overview

The `Vfs` trait is the foundation of anyfs. It defines how backends store inodes and content.

## Target Audience

**Backend implementers** — people creating new storage backends.

**Not for:** Application developers (use `FilesContainer` instead).

---

## Key Characteristics

| Aspect | Description |
|--------|-------------|
| Path handling | **None** — uses `InodeId` |
| Semantics | **None** — raw operations |
| Capacity limits | **None** — just storage |

---

## Trait Definition

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

---

## Types

### InodeId

```rust
/// Unique inode identifier within a storage backend.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct InodeId(pub u64);
```

### InodeKind

```rust
#[derive(Clone, Debug)]
pub enum InodeKind {
    File,
    Directory,
    Symlink { target: String },
}
```

### InodeData

```rust
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

---

## Implementing a Backend

See [Backend Implementer's Guide](../implementation/backend-guide.md) for a complete walkthrough.

### Minimal Example

```rust
use anyfs::{Vfs, InodeId, InodeKind, InodeData, VfsError};
use std::collections::HashMap;

pub struct SimpleVfs {
    inodes: HashMap<InodeId, InodeData>,
    content: HashMap<InodeId, Vec<u8>>,
    entries: HashMap<(InodeId, String), InodeId>,
    next_id: u64,
}

impl Vfs for SimpleVfs {
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        let id = InodeId(self.next_id);
        self.next_id += 1;

        self.inodes.insert(id, InodeData {
            ino: id.0,
            kind,
            mode,
            uid: 0,
            gid: 0,
            nlink: 1,
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
            .ok_or(VfsError::NotFound)
    }

    // ... implement remaining methods
}
```

---

## Method Reference

| Method | Purpose |
|--------|---------|
| `create_inode` | Allocate new inode |
| `get_inode` | Read inode metadata |
| `update_inode` | Modify inode metadata |
| `delete_inode` | Remove inode (must have nlink=0) |
| `link` | Add directory entry |
| `unlink` | Remove directory entry |
| `lookup` | Find child by name |
| `readdir` | List directory entries |
| `read` | Read file content |
| `write` | Write file content |
| `truncate` | Resize file |
| `sync` | Flush to durable storage |
| `root` | Get root inode |

---

## Important Notes

1. **No paths:** The Vfs trait never receives paths. Path resolution is done by `FilesContainer`.

2. **nlink tracking:** Backends must track hard link counts. An inode can only be deleted when `nlink == 0`.

3. **Symlinks:** Stored as `InodeKind::Symlink { target }`. The target is just a string — resolution is done at the container level.

4. **Root inode:** Must exist before any operations. Typically created in the backend constructor.
