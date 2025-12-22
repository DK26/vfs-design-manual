# AnyFS — Backend Implementer's Guide

**Step-by-step guide for implementing a custom storage backend**

---

## Overview

This guide walks you through implementing a custom `VfsBackend`. By the end, you'll have a working backend that can be used with `FilesContainer`.

---

## What You're Building

A backend implements the `VfsBackend` trait — a path-based interface aligned with `std::fs`:

```rust
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

**Key points:**
- All paths are `&VirtualPath` — already validated, safe, normalized
- 20 methods total — aligned with `std::fs` naming conventions
- Symlinks and hard links are simulated (not real OS links unless using `VRootFsBackend`)
- Backends handle storage; `FilesContainer` handles quotas

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
        // Follow symlinks first
        let resolved = self.resolve_symlinks(path)?;

        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let key = self.key(&resolved);
        let data: Option<Vec<u8>> = redis::cmd("GET")
            .arg(&key)
            .query(&mut conn)
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        data.ok_or_else(|| VfsError::NotFound(path.clone()))
    }

    fn read_to_string(&self, path: &VirtualPath) -> Result<String, VfsError> {
        let bytes = self.read(path)?;
        String::from_utf8(bytes)
            .map_err(|e| VfsError::InvalidUtf8(e.utf8_error()))
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
        // Follows symlinks - broken symlink returns false
        match self.resolve_symlinks(path) {
            Ok(resolved) => {
                let mut conn = self.client.get_connection()
                    .map_err(|e| VfsError::Backend(e.to_string()))?;
                let key = self.key(&resolved);
                redis::cmd("EXISTS").arg(&key).query(&mut conn)
                    .map_err(|e| VfsError::Backend(e.to_string()))
            }
            Err(_) => Ok(false),  // Broken symlink
        }
    }

    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError> {
        // Follows symlinks
        let resolved = self.resolve_symlinks(path)?;
        self.get_metadata_internal(&resolved)
    }

    fn symlink_metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError> {
        // Does NOT follow symlinks - returns metadata of symlink itself
        self.get_metadata_internal(path)
    }

    fn read_dir(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError> {
        // List children of directory
        // Return Vec<DirEntry> with name and file_type for each child
        todo!()
    }

    fn read_link(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError> {
        // Return symlink target without following
        // Error if path is not a symlink
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
        // Follow symlinks
        let resolved = self.resolve_symlinks(path)?;

        // Ensure parent directory exists
        if let Some(parent) = resolved.parent() {
            if !self.exists(&parent)? {
                return Err(VfsError::NotFound(parent));
            }
        }

        let mut conn = self.client.get_connection()
            .map_err(|e| VfsError::Backend(e.to_string()))?;

        let key = self.key(&resolved);
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

    fn create_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError> {
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

    fn create_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError> {
        // Create all ancestors, then the target
        let mut current = VirtualPath::root();
        for component in path.components() {
            current = current.join(component)?;
            if !self.exists(&current)? {
                self.create_dir(&current)?;
            }
        }
        Ok(())
    }

    fn remove_file(&mut self, path: &VirtualPath) -> Result<(), VfsError> {
        let meta = self.symlink_metadata(path)?;

        // remove_file works on files and symlinks, not directories
        if meta.file_type == FileType::Directory {
            return Err(VfsError::IsADirectory(path.clone()));
        }

        self.delete_entry(path)
    }

    fn remove_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError> {
        let meta = self.symlink_metadata(path)?;

        if meta.file_type != FileType::Directory {
            return Err(VfsError::NotADirectory(path.clone()));
        }

        // Check if directory is empty
        let children = self.read_dir(path)?;
        if !children.is_empty() {
            return Err(VfsError::DirectoryNotEmpty(path.clone()));
        }

        self.delete_entry(path)
    }

    fn remove_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError> {
        if !self.exists(path)? {
            return Err(VfsError::NotFound(path.clone()));
        }

        // Recursively delete children first
        let meta = self.symlink_metadata(path)?;
        if meta.file_type == FileType::Directory {
            for entry in self.read_dir(path)? {
                let child = path.join(&entry.name)?;
                self.remove_dir_all(&child)?;
            }
        }

        self.delete_entry(path)
    }

    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError> {
        // Renames symlink itself, not target
        if !self.exists(from)? {
            return Err(VfsError::NotFound(from.clone()));
        }

        // Copy data, then delete original
        let data = self.read(from)?;
        self.write(to, &data)?;
        self.remove_file(from)?;

        Ok(())
    }

    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError> {
        // Follows symlinks (copies content, not link)
        let data = self.read(from)?;
        self.write(to, &data)
    }
}
```

---

## Step 4: Implement Link Operations

```rust
impl VfsBackend for RedisBackend {
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        // Store symlink entry (original does not need to exist)
        // This is a simulated symlink, not a real OS symlink
        self.store_entry(link, Entry::Symlink { target: original.clone() })
    }

    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        // Original must exist and be a file
        let meta = self.symlink_metadata(original)?;
        if meta.file_type != FileType::File {
            return Err(VfsError::NotAFile(original.clone()));
        }

        // Create new entry pointing to same content
        // Your implementation: share the content between entries
        todo!()
    }

    fn set_permissions(&mut self, path: &VirtualPath, perm: Permissions) -> Result<(), VfsError> {
        // Update stored permissions for path
        todo!()
    }

    // Helper: resolve symlinks with loop detection
    fn resolve_symlinks(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError> {
        const MAX_DEPTH: u32 = 40;
        let mut current = path.clone();
        let mut depth = 0;

        loop {
            match self.symlink_metadata(&current)?.file_type {
                FileType::Symlink => {
                    depth += 1;
                    if depth > MAX_DEPTH {
                        return Err(VfsError::SymlinkLoop(path.clone()));
                    }
                    current = self.read_link(&current)?;
                }
                _ => return Ok(current),
            }
        }
    }
}
```

---

## Step 5: Handle Types Correctly

### Metadata

```rust
use std::time::SystemTime;

pub struct Metadata {
    pub file_type: FileType,
    pub size: u64,
    pub permissions: Permissions,
    pub nlink: u64,  // Number of hard links
    pub created: Option<SystemTime>,
    pub modified: Option<SystemTime>,
    pub accessed: Option<SystemTime>,
}
```

### Permissions

```rust
pub struct Permissions {
    readonly: bool,
}

impl Permissions {
    pub fn new() -> Self { Self { readonly: false } }
    pub fn readonly(&self) -> bool { self.readonly }
    pub fn set_readonly(&mut self, readonly: bool) { self.readonly = readonly; }
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

## Step 6: Error Handling

Use the appropriate `VfsError` variants:

```rust
pub enum VfsError {
    NotFound(VirtualPath),
    AlreadyExists(VirtualPath),
    NotAFile(VirtualPath),
    NotADirectory(VirtualPath),
    NotASymlink(VirtualPath),
    DirectoryNotEmpty(VirtualPath),
    IsADirectory(VirtualPath),
    InvalidPath(String),
    InvalidUtf8(std::str::Utf8Error),
    PermissionDenied(VirtualPath),
    SymlinkLoop(VirtualPath),
    Io(std::io::Error),
    Backend(String),  // For backend-specific errors
}
```

**Best practices:**
- Use `NotFound` when a path doesn't exist
- Use `AlreadyExists` when creating over existing path
- Use `NotADirectory` when `read_dir` is called on a file
- Use `NotAFile` when reading a directory
- Use `IsADirectory` when `remove_file` is called on a directory
- Use `NotASymlink` when `read_link` is called on a non-symlink
- Use `SymlinkLoop` when symlink resolution exceeds 40 levels
- Use `Backend(msg)` for storage-specific errors

---

## Step 7: Using Your Backend

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

- [ ] All 20 `VfsBackend` methods implemented
- [ ] `read` follows symlinks, returns `NotFound` for missing files
- [ ] `write` follows symlinks, checks parent exists
- [ ] `create_dir` returns `AlreadyExists` if path exists
- [ ] `remove_file` returns `IsADirectory` for directories
- [ ] `remove_dir` returns `DirectoryNotEmpty` for non-empty dirs
- [ ] `remove_dir_all` handles recursive deletion
- [ ] `symlink` stores target (original doesn't need to exist)
- [ ] `hard_link` shares content, increments `nlink`
- [ ] `read_link` returns `NotASymlink` for non-symlinks
- [ ] `symlink_metadata` does NOT follow symlinks
- [ ] `metadata` follows symlinks
- [ ] Symlink loop detection (max 40 levels)
- [ ] Metadata returns correct `FileType` and `nlink`
- [ ] `read_dir` returns only direct children (not recursive)

---

## Example Backends

For reference implementations, see the built-in backends in `anyfs`:

- **MemoryBackend** — HashMap-based, simplest implementation
- **SqliteBackend** — SQLite database, production-ready
- **VRootFsBackend** — Host filesystem via `strict-path`

---

*For the full trait definition, see [Project Structure](../overview/project-structure.md).*
