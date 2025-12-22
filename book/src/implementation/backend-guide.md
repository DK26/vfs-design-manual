# AnyFS — Backend Implementer's Guide

**Step-by-step guide for implementing a custom storage backend**

---

## Overview

This guide walks you through implementing a custom `VfsBackend`. By the end, you'll have a working backend that can be used with `FilesContainer`.

---

## What You're Building

A backend implements the `VfsBackend` trait — a simple, path-based interface for filesystem operations:

```rust
pub trait VfsBackend: Send {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    fn list(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;
    fn mkdir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn mkdir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
}
```

**Key points:**
- All paths are `&VirtualPath` — already validated, safe, normalized
- 13 methods total — matches familiar `std::fs` operations
- Backends handle storage; `FilesContainer` handles quotas and features

---

## Prerequisites

Only depend on `anyfs-traits` — minimal dependencies:

```toml
[dependencies]
anyfs-traits = "0.1"
```

You do NOT need `anyfs` (which has the built-in backends) or `anyfs-container`.

---

## Step 1: Define Your Backend Struct

```rust
use anyfs_traits::{VfsBackend, VirtualPath, VfsError, Metadata, DirEntry, FileType};
use std::time::SystemTime;

/// Example: A backend that stores files in Redis
pub struct RedisBackend {
    client: redis::Client,
    prefix: String,
}

impl RedisBackend {
    pub fn new(url: &str, prefix: &str) -> Result<Self, VfsError> {
        let client = redis::Client::open(url)
            .map_err(|e| VfsError::Backend(e.to_string()))?;
        Ok(Self {
            client,
            prefix: prefix.to_string(),
        })
    }

    fn key(&self, path: &VirtualPath) -> String {
        format!("{}:{}", self.prefix, path.as_str())
    }
}
```

---

## Step 2: Implement Read Operations

```rust
impl VfsBackend for RedisBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let key = self.key(path);
        let data: Option<Vec<u8>> = redis::cmd("GET")
            .arg(&key)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        data.ok_or_else(|| VfsError::NotFound(path.clone()))
    }

    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError> {
        let data = self.read(path)?;
        let start = offset as usize;
        let end = (start + len).min(data.len());

        if start >= data.len() {
            return Ok(Vec::new());
        }

        Ok(data[start..end].to_vec())
    }

    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError> {
        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let key = self.key(path);
        let exists: bool = redis::cmd("EXISTS")
            .arg(&key)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        Ok(exists)
    }

    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError> {
        // Your implementation - fetch stored metadata
        // Return Metadata { file_type, size, created, modified }
        todo!()
    }

    fn list(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError> {
        // Your implementation - list children of directory
        // Return Vec<DirEntry> with name and file_type for each child
        todo!()
    }
}
```

---

## Step 3: Implement Write Operations

```rust
impl VfsBackend for RedisBackend {
    // ... read operations from above ...

    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError> {
        // Ensure parent directory exists
        if let Some(parent) = path.parent() {
            if !self.exists(&parent)? {
                return Err(VfsError::NotFound(parent));
            }
        }

        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let key = self.key(path);
        redis::cmd("SET")
            .arg(&key)
            .arg(data)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        Ok(())
    }

    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError> {
        let mut existing = self.read(path).unwrap_or_default();
        existing.extend_from_slice(data);
        self.write(path, &existing)
    }

    fn mkdir(&mut self, path: &VirtualPath) -> Result<(), VfsError> {
        if self.exists(path)? {
            return Err(VfsError::AlreadyExists(path.clone()));
        }

        // Ensure parent exists
        if let Some(parent) = path.parent() {
            if !self.exists(&parent)? {
                return Err(VfsError::NotFound(parent));
            }
        }

        // Store directory marker
        // Your implementation here
        todo!()
    }

    fn mkdir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError> {
        // Create all ancestors, then the target
        let mut current = VirtualPath::root();
        for component in path.components() {
            current = current.join(component)?;
            if !self.exists(&current)? {
                self.mkdir(&current)?;
            }
        }
        Ok(())
    }

    fn remove(&mut self, path: &VirtualPath) -> Result<(), VfsError> {
        if !self.exists(path)? {
            return Err(VfsError::NotFound(path.clone()));
        }

        // Check if directory is empty
        let meta = self.metadata(path)?;
        if meta.file_type == FileType::Directory {
            let children = self.list(path)?;
            if !children.is_empty() {
                return Err(VfsError::DirectoryNotEmpty(path.clone()));
            }
        }

        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let key = self.key(path);
        redis::cmd("DEL")
            .arg(&key)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        Ok(())
    }

    fn remove_all(&mut self, path: &VirtualPath) -> Result<(), VfsError> {
        if !self.exists(path)? {
            return Err(VfsError::NotFound(path.clone()));
        }

        // Recursively delete children first
        let meta = self.metadata(path)?;
        if meta.file_type == FileType::Directory {
            for entry in self.list(path)? {
                let child = path.join(&entry.name)?;
                self.remove_all(&child)?;
            }
        }

        self.remove(path)
    }

    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError> {
        if !self.exists(from)? {
            return Err(VfsError::NotFound(from.clone()));
        }

        // Copy data, then delete original
        let data = self.read(from)?;
        self.write(to, &data)?;
        self.remove(from)?;

        Ok(())
    }

    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError> {
        let data = self.read(from)?;
        self.write(to, &data)
    }
}
```

---

## Step 4: Handle Types Correctly

### Metadata

```rust
use std::time::SystemTime;

pub struct Metadata {
    pub file_type: FileType,
    pub size: u64,
    pub created: Option<SystemTime>,
    pub modified: Option<SystemTime>,
}
```

### DirEntry

```rust
pub struct DirEntry {
    pub name: String,
    pub file_type: FileType,
}
```

### FileType

```rust
pub enum FileType {
    File,
    Directory,
    Symlink,
}
```

---

## Step 5: Error Handling

Use the appropriate `VfsError` variants:

```rust
pub enum VfsError {
    NotFound(VirtualPath),
    AlreadyExists(VirtualPath),
    NotAFile(VirtualPath),
    NotADirectory(VirtualPath),
    DirectoryNotEmpty(VirtualPath),
    InvalidPath(String),
    Io(std::io::Error),
    Backend(String),  // For backend-specific errors
}
```

**Best practices:**
- Use `NotFound` when a path doesn't exist
- Use `AlreadyExists` when creating over existing path
- Use `NotADirectory` when listing a file
- Use `NotAFile` when reading a directory
- Use `Backend(msg)` for storage-specific errors

---

## Step 6: Using Your Backend

```rust
use anyfs_container::{FilesContainer, ContainerBuilder};

// Direct usage
let backend = RedisBackend::new("redis://localhost", "myapp")?;
let mut container = FilesContainer::new(backend);

container.write("/data/file.txt", b"hello")?;

// With capacity limits
let backend = RedisBackend::new("redis://localhost", "tenant_123")?;
let mut container = ContainerBuilder::new(backend)
    .max_total_size(100 * 1024 * 1024)  // 100 MB
    .max_file_size(10 * 1024 * 1024)    // 10 MB
    .build()?;
```

---

## Checklist

Before shipping your backend:

- [ ] All 13 `VfsBackend` methods implemented
- [ ] `read` returns `NotFound` for missing files
- [ ] `write` checks parent directory exists
- [ ] `mkdir` returns `AlreadyExists` if path exists
- [ ] `remove` returns `DirectoryNotEmpty` for non-empty dirs
- [ ] `remove_all` handles recursive deletion
- [ ] Metadata returns correct `FileType`
- [ ] `list` returns only direct children (not recursive)

---

## Example Backends

For reference implementations, see the built-in backends in `anyfs`:

- **MemoryBackend** — HashMap-based, simplest implementation
- **SqliteBackend** — SQLite database, production-ready
- **VRootFsBackend** — Host filesystem via `strict-path`

---

*For the full trait definition, see [Project Structure](../overview/project-structure.md).*
