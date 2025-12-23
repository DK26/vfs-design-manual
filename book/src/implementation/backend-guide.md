# AnyFS — Backend Implementer's Guide

**Step-by-step guide for implementing a custom storage backend**

---

## Overview

This guide walks you through implementing a custom `Vfs` backend. By the end, you'll have a working backend that can be used with `FilesContainer`.

---

## What You're Building

A backend implements the `Vfs` trait — a low-level inode-based interface:

```rust
pub trait Vfs: Send {
    // INODE LIFECYCLE
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError>;
    fn get_inode(&self, id: InodeId) -> Result<InodeData, VfsError>;
    fn update_inode(&mut self, id: InodeId, data: InodeData) -> Result<(), VfsError>;
    fn delete_inode(&mut self, id: InodeId) -> Result<(), VfsError>;

    // DIRECTORY OPERATIONS
    fn link(&mut self, parent: InodeId, name: &str, child: InodeId) -> Result<(), VfsError>;
    fn unlink(&mut self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn readdir(&self, dir: InodeId) -> Result<Vec<(String, InodeId)>, VfsError>;

    // CONTENT I/O
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError>;
    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError>;
    fn truncate(&mut self, id: InodeId, size: u64) -> Result<(), VfsError>;

    // SYNC & ROOT
    fn sync(&mut self) -> Result<(), VfsError>;
    fn root(&self) -> InodeId;
}
```

**Key points:**
- All operations use `InodeId` — no path handling required
- 13 methods total — simple, focused operations
- Symlinks stored as `InodeKind::Symlink { target }` — container handles resolution
- Hard links: multiple directory entries can point to the same inode
- Backends handle storage; `FilesContainer` handles paths, semantics, and limits

---

## Prerequisites

Depend on `anyfs`:

```toml
[dependencies]
anyfs = "0.1"
```

---

## Step 1: Define Your Backend Struct

```rust
use anyfs::{Vfs, InodeId, InodeKind, InodeData, VfsError};
use std::collections::HashMap;
use std::time::SystemTime;

/// Example: A backend that stores files in Redis
pub struct RedisVfs {
    client: redis::Client,
    prefix: String,
    root_id: InodeId,
}

impl RedisVfs {
    pub fn new(url: &str, prefix: &str) -> Result<Self, VfsError> {
        let client = redis::Client::open(url)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let mut vfs = Self {
            client,
            prefix: prefix.to_string(),
            root_id: InodeId(0),
        };

        // Ensure root directory exists
        vfs.ensure_root()?;

        Ok(vfs)
    }

    fn ensure_root(&mut self) -> Result<(), VfsError> {
        // Check if root exists, create if not
        if self.get_inode(self.root_id).is_err() {
            self.store_inode(self.root_id, InodeData {
                ino: 0,
                kind: InodeKind::Directory,
                mode: 0o755,
                uid: 0,
                gid: 0,
                nlink: 2,  // . and ..
                size: 0,
                atime: None,
                mtime: None,
                ctime: None,
            })?;
        }
        Ok(())
    }
}
```

---

## Step 2: Implement Inode Lifecycle

```rust
impl Vfs for RedisVfs {
    fn root(&self) -> InodeId {
        self.root_id
    }

    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        // Allocate new inode ID
        let id = self.allocate_inode_id()?;

        let data = InodeData {
            ino: id.0,
            kind,
            mode,
            uid: 0,
            gid: 0,
            nlink: 0,  // Not linked yet - will be incremented by link()
            size: 0,
            atime: Some(SystemTime::now()),
            mtime: Some(SystemTime::now()),
            ctime: Some(SystemTime::now()),
        };

        self.store_inode(id, data)?;
        Ok(id)
    }

    fn get_inode(&self, id: InodeId) -> Result<InodeData, VfsError> {
        // Retrieve inode data from Redis
        let key = format!("{}:inode:{}", self.prefix, id.0);
        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let data: Option<Vec<u8>> = redis::cmd("GET")
            .arg(&key)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        match data {
            Some(bytes) => self.deserialize_inode(&bytes),
            None => Err(VfsError::NotFound(id)),
        }
    }

    fn update_inode(&mut self, id: InodeId, data: InodeData) -> Result<(), VfsError> {
        // Verify inode exists
        let _ = self.get_inode(id)?;

        // Update the inode
        self.store_inode(id, data)
    }

    fn delete_inode(&mut self, id: InodeId) -> Result<(), VfsError> {
        let inode = self.get_inode(id)?;

        // Can only delete if nlink == 0
        if inode.nlink > 0 {
            return Err(VfsError::InodeInUse);
        }

        // Delete inode data
        let key = format!("{}:inode:{}", self.prefix, id.0);
        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        redis::cmd("DEL")
            .arg(&key)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        // Also delete content if file
        if matches!(inode.kind, InodeKind::File) {
            let content_key = format!("{}:content:{}", self.prefix, id.0);
            redis::cmd("DEL")
                .arg(&content_key)
                .query(&mut conn)
                .map_err(|e| VfsError::Backend(e.to_string()))?;
        }

        Ok(())
    }
}
```

---

## Step 3: Implement Directory Operations

```rust
impl Vfs for RedisVfs {
    // ... inode lifecycle from above ...

    fn link(&mut self, parent: InodeId, name: &str, child: InodeId) -> Result<(), VfsError> {
        // Verify parent is a directory
        let parent_inode = self.get_inode(parent)?;
        if !matches!(parent_inode.kind, InodeKind::Directory) {
            return Err(VfsError::NotADirectory(parent));
        }

        // Check entry doesn't already exist
        if self.lookup(parent, name).is_ok() {
            return Err(VfsError::EntryExists(name.to_string()));
        }

        // Add directory entry
        self.store_entry(parent, name, child)?;

        // Increment child's nlink
        let mut child_inode = self.get_inode(child)?;
        child_inode.nlink += 1;
        self.update_inode(child, child_inode)?;

        Ok(())
    }

    fn unlink(&mut self, parent: InodeId, name: &str) -> Result<InodeId, VfsError> {
        // Verify parent is a directory
        let parent_inode = self.get_inode(parent)?;
        if !matches!(parent_inode.kind, InodeKind::Directory) {
            return Err(VfsError::NotADirectory(parent));
        }

        // Get the child inode
        let child = self.lookup(parent, name)?;

        // If child is a directory, it must be empty
        let child_inode = self.get_inode(child)?;
        if matches!(child_inode.kind, InodeKind::Directory) {
            let entries = self.readdir(child)?;
            if !entries.is_empty() {
                return Err(VfsError::DirectoryNotEmpty(child));
            }
        }

        // Remove directory entry
        self.delete_entry(parent, name)?;

        // Decrement child's nlink
        let mut child_inode = self.get_inode(child)?;
        child_inode.nlink -= 1;
        self.update_inode(child, child_inode)?;

        Ok(child)
    }

    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError> {
        // Find child by name in parent directory
        let key = format!("{}:entry:{}:{}", self.prefix, parent.0, name);
        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let child_id: Option<u64> = redis::cmd("GET")
            .arg(&key)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        child_id
            .map(InodeId)
            .ok_or_else(|| VfsError::EntryNotFound(name.to_string()))
    }

    fn readdir(&self, dir: InodeId) -> Result<Vec<(String, InodeId)>, VfsError> {
        // Verify dir is a directory
        let dir_inode = self.get_inode(dir)?;
        if !matches!(dir_inode.kind, InodeKind::Directory) {
            return Err(VfsError::NotADirectory(dir));
        }

        // List all entries in directory
        let pattern = format!("{}:entry:{}:*", self.prefix, dir.0);
        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let keys: Vec<String> = redis::cmd("KEYS")
            .arg(&pattern)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let mut entries = Vec::new();
        for key in keys {
            // Extract name from key
            let name = key.rsplit(':').next().unwrap().to_string();
            let child_id = self.lookup(dir, &name)?;
            entries.push((name, child_id));
        }

        Ok(entries)
    }
}
```

---

## Step 4: Implement Content I/O

```rust
impl Vfs for RedisVfs {
    // ... previous methods ...

    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError> {
        let inode = self.get_inode(id)?;
        if !matches!(inode.kind, InodeKind::File) {
            return Err(VfsError::NotAFile(id));
        }

        // Read content from Redis
        let content_key = format!("{}:content:{}", self.prefix, id.0);
        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let content: Vec<u8> = redis::cmd("GET")
            .arg(&content_key)
            .query(&mut conn)
            .unwrap_or_default();

        // Calculate read range
        let start = offset as usize;
        if start >= content.len() {
            return Ok(0);
        }

        let end = (start + buf.len()).min(content.len());
        let bytes_read = end - start;
        buf[..bytes_read].copy_from_slice(&content[start..end]);

        Ok(bytes_read)
    }

    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError> {
        let mut inode = self.get_inode(id)?;
        if !matches!(inode.kind, InodeKind::File) {
            return Err(VfsError::NotAFile(id));
        }

        // Read existing content
        let content_key = format!("{}:content:{}", self.prefix, id.0);
        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let mut content: Vec<u8> = redis::cmd("GET")
            .arg(&content_key)
            .query(&mut conn)
            .unwrap_or_default();

        // Extend content if needed
        let start = offset as usize;
        let end = start + data.len();
        if end > content.len() {
            content.resize(end, 0);
        }

        // Write data
        content[start..end].copy_from_slice(data);

        // Store content
        redis::cmd("SET")
            .arg(&content_key)
            .arg(&content)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        // Update inode size and mtime
        inode.size = content.len() as u64;
        inode.mtime = Some(SystemTime::now());
        self.update_inode(id, inode)?;

        Ok(data.len())
    }

    fn truncate(&mut self, id: InodeId, size: u64) -> Result<(), VfsError> {
        let mut inode = self.get_inode(id)?;
        if !matches!(inode.kind, InodeKind::File) {
            return Err(VfsError::NotAFile(id));
        }

        let content_key = format!("{}:content:{}", self.prefix, id.0);
        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let mut content: Vec<u8> = redis::cmd("GET")
            .arg(&content_key)
            .query(&mut conn)
            .unwrap_or_default();

        // Resize content
        content.resize(size as usize, 0);

        // Store truncated content
        redis::cmd("SET")
            .arg(&content_key)
            .arg(&content)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        // Update inode
        inode.size = size;
        inode.mtime = Some(SystemTime::now());
        self.update_inode(id, inode)?;

        Ok(())
    }

    fn sync(&mut self) -> Result<(), VfsError> {
        // Redis is typically synchronous, but you could flush here
        Ok(())
    }
}
```

---

## Step 5: Handle Types Correctly

### InodeId

```rust
/// Unique inode identifier
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

## Step 6: Error Handling

Use the appropriate `VfsError` variants:

```rust
pub enum VfsError {
    NotFound(InodeId),          // Inode doesn't exist
    NotAFile(InodeId),          // Expected file, got directory/symlink
    NotADirectory(InodeId),     // Expected directory
    DirectoryNotEmpty(InodeId), // Cannot delete non-empty directory
    EntryExists(String),        // Directory entry already exists
    EntryNotFound(String),      // Directory entry doesn't exist
    InodeInUse,                 // Cannot delete inode with nlink > 0
    Io(std::io::Error),         // I/O error
    Backend(String),            // Backend-specific error
}
```

**Best practices:**
- Use `NotFound` when an inode doesn't exist
- Use `EntryNotFound` when a directory entry doesn't exist
- Use `EntryExists` when creating a duplicate entry
- Use `NotADirectory` when directory operations are called on non-directories
- Use `NotAFile` when file operations are called on non-files
- Use `DirectoryNotEmpty` when unlinking a non-empty directory
- Use `InodeInUse` when deleting an inode that still has links
- Use `Backend(msg)` for storage-specific errors

---

## Step 7: Using Your Backend

```rust
use anyfs_container::{FilesContainer, LinuxSemantics, ContainerBuilder, CapacityLimits};

// Simple usage
let vfs = RedisVfs::new("redis://localhost", "myapp")?;
let mut container = FilesContainer::new(vfs, LinuxSemantics::new());

container.write("/data/file.txt", b"hello")?;

// With capacity limits
let vfs = RedisVfs::new("redis://localhost", "tenant_123")?;
let mut container = ContainerBuilder::new()
    .vfs(vfs)
    .semantics(LinuxSemantics::new())
    .limits(CapacityLimits {
        max_total_size: Some(100 * 1024 * 1024),  // 100 MB
        max_file_size: Some(10 * 1024 * 1024),    // 10 MB
        ..Default::default()
    })
    .build()?;

container.write("/uploads/doc.pdf", &pdf_data)?;
```

---

## Checklist

Before shipping your backend:

- [ ] All 13 `Vfs` methods implemented
- [ ] `root()` returns root directory inode
- [ ] `create_inode()` allocates unique inode IDs
- [ ] `get_inode()` returns `NotFound` for missing inodes
- [ ] `delete_inode()` returns `InodeInUse` if `nlink > 0`
- [ ] `link()` increments child's `nlink`
- [ ] `unlink()` decrements child's `nlink`
- [ ] `unlink()` returns `DirectoryNotEmpty` for non-empty directories
- [ ] `lookup()` returns `EntryNotFound` for missing entries
- [ ] `readdir()` returns all direct children
- [ ] `read()` respects offset and buffer size
- [ ] `write()` extends file size if needed
- [ ] `truncate()` handles both shrinking and extending
- [ ] Symlinks stored as `InodeKind::Symlink { target }`
- [ ] Hard links share same inode (multiple entries, same `InodeId`)

---

## Example Backends

For reference implementations, see the built-in backends in `anyfs`:

- **MemoryVfs** — HashMap-based, simplest implementation
- **SqliteVfs** — SQLite database, production-ready
- **VRootVfs** — Host filesystem via `strict-path`

---

*For the full trait definition, see [Vfs Trait](../traits/vfs-trait.md).*
