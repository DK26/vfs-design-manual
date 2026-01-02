# Update Instructions: anyfs-design-manual Repository

**Repository:** https://github.com/DK26/anyfs-design-manual  
**Date:** 2024-12-23  
**Purpose:** Step-by-step instructions to update documentation with new two-layer API design

---

## Executive Summary

The key architectural insight is **layer separation**:

| Layer          | Crate             | API Style                | Target User            |
| -------------- | ----------------- | ------------------------ | ---------------------- |
| **Low-level**  | `anyfs`           | Filesystem-like (inodes) | Backend implementers   |
| **High-level** | `anyfs-container` | `std::fs`-like (paths)   | Application developers |

This document provides exact instructions for updating each file in the repository.

---

## Table of Contents

1. [Understanding the Layer Separation](#1-understanding-the-layer-separation)
2. [AGENTS.md Updates](#2-agentsmd-updates)
3. [README.md Updates](#3-readmemd-updates)
4. [New Files to Create](#4-new-files-to-create)
5. [Existing Files to Update](#5-existing-files-to-update)
6. [mdbook Structure Updates](#6-mdbook-structure-updates)
7. [Diagrams to Add](#7-diagrams-to-add)
8. [Checklist](#8-checklist)

---

## 1. Understanding the Layer Separation

### 1.1 The Two Layers

```
┌─────────────────────────────────────────────────────────────┐
│  Application Code                                           │
│                                                             │
│  // Looks just like std::fs!                                │
│  container.write("/data/file.txt", b"hello")?;              │
│  container.read_dir("/data")?;                              │
├─────────────────────────────────────────────────────────────┤
│  anyfs-container                                            │
│  ─────────────────                                          │
│  • std::fs-like API (paths, ergonomic)                      │
│  • Path resolution via FsSemantics                          │
│  • Capacity enforcement                                     │
│  • For: Application developers                              │
├─────────────────────────────────────────────────────────────┤
│  anyfs                                                      │
│  ─────                                                      │
│  • Filesystem-like trait (inodes, raw ops)                  │
│  • No path handling — just inode IDs                        │
│  • No capacity limits — just storage                        │
│  • For: Backend implementers                                │
├──────────────────┬──────────────────┬───────────────────────┤
│  MemoryVfs       │  SqliteVfs       │  RealFsVfs            │
└──────────────────┴──────────────────┴───────────────────────┘
```

### 1.2 Why This Matters

| Concern                     | Handled By                         |
| --------------------------- | ---------------------------------- |
| How to store inodes/content | `anyfs` (Vfs trait)                |
| How to interpret paths      | `anyfs-container` (FsSemantics)    |
| Capacity limits             | `anyfs-container` (FilesContainer) |
| User-facing API             | `anyfs-container` (std::fs-like)   |

### 1.3 The Traits

**anyfs — Low-Level Vfs Trait:**
```rust
/// Filesystem-like trait for backend implementers.
/// Works with inodes, not paths.
pub trait Vfs: Send {
    // Inode lifecycle
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError>;
    fn get_inode(&self, id: InodeId) -> Result<InodeData, VfsError>;
    fn update_inode(&mut self, id: InodeId, data: InodeData) -> Result<(), VfsError>;
    fn delete_inode(&mut self, id: InodeId) -> Result<(), VfsError>;
    
    // Directory operations  
    fn link(&mut self, parent: InodeId, name: &str, child: InodeId) -> Result<(), VfsError>;
    fn unlink(&mut self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn readdir(&self, dir: InodeId) -> Result<Vec<(String, InodeId)>, VfsError>;
    
    // Content I/O
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError>;
    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError>;
    fn truncate(&mut self, id: InodeId, size: u64) -> Result<(), VfsError>;
    
    // Sync
    fn sync(&mut self) -> Result<(), VfsError>;
    
    // Root
    fn root(&self) -> InodeId;
}
```

**anyfs-container — High-Level FilesContainer:**
```rust
/// std::fs-like API for application developers.
/// Works with paths, enforces capacity limits.
impl<V: Vfs, S: FsSemantics> FilesContainer<V, S> {
    // Matches std::fs::read
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    
    // Matches std::fs::read_to_string  
    pub fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, ContainerError>;
    
    // Matches std::fs::write
    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    
    // Matches std::fs::read_dir
    pub fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, ContainerError>;
    
    // Matches std::fs::create_dir
    pub fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    // Matches std::fs::create_dir_all
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    // ... etc, all matching std::fs
}
```

---

## 2. AGENTS.md Updates

### 2.1 Replace Project Overview Section

**Find and replace this section:**

```markdown
## Project Overview

This is the **VFS Ecosystem** — three Rust crates for virtual filesystem abstraction:

| Crate             | Purpose                                                                      |
| ----------------- | ---------------------------------------------------------------------------- |
| `anyfs-traits`    | Minimal crate — trait definition, types, re-exports `VirtualPath`            |
| `anyfs`           | Core VFS — re-exports traits, provides built-in backends (optional features) |
| `anyfs-container` | Higher-level wrapper — capacity limits, tenant isolation                     |
```

**Replace with:**

```markdown
## Project Overview

This is the **VFS Ecosystem** — a two-layer architecture for virtual filesystem abstraction:

### Layer 1: anyfs (Low-Level)

| Crate   | Purpose                                        | Target User          |
| ------- | ---------------------------------------------- | -------------------- |
| `anyfs` | Filesystem-like `Vfs` trait (inode operations) | Backend implementers |

### Layer 2: anyfs-container (High-Level)

| Crate             | Purpose                                 | Target User            |
| ----------------- | --------------------------------------- | ---------------------- |
| `anyfs-container` | `std::fs`-like API with capacity limits | Application developers |

### Supporting Crates

| Crate             | Purpose                                        |
| ----------------- | ---------------------------------------------- |
| `anyfs-semantics` | Path resolution rules (Linux, Windows, Simple) |

### Key Insight

- **anyfs** = "How do I store inodes and content?" (low-level, no paths)
- **anyfs-container** = "How do I use this like std::fs?" (high-level, paths)
```

### 2.2 Replace Architecture Diagram

**Find:**
```
┌─────────────────────────────────────────┐
│  User Application                       │
├─────────────────────────────────────────┤
│  anyfs-container                        │
...
```

**Replace with:**
```
┌─────────────────────────────────────────────────────────────┐
│  Application Code                                           │
│                                                             │
│  container.write("/data/file.txt", b"hello")?;  // std::fs! │
│  container.read_dir("/data")?;                              │
├─────────────────────────────────────────────────────────────┤
│  anyfs-container: FilesContainer<V, S>                      │
│  ─────────────────────────────────────────                  │
│  • std::fs-like API (impl AsRef<Path>)                      │
│  • Path resolution via FsSemantics                          │
│  • Capacity enforcement (limits, quotas)                    │
│  • Target: Application developers                           │
├─────────────────────────────────────────────────────────────┤
│  anyfs: Vfs trait                                           │
│  ────────────────────                                       │
│  • Filesystem-like trait (InodeId, not paths)               │
│  • create_inode, lookup, link, unlink, read, write          │
│  • No path logic — raw inode operations                     │
│  • Target: Backend implementers                             │
├──────────────────┬──────────────────┬───────────────────────┤
│  MemoryVfs       │  SqliteVfs       │  RealFsVfs            │
│  (HashMap)       │  (.db file)      │  (strict-path)        │
└──────────────────┴──────────────────┴───────────────────────┘
```

### 2.3 Replace Trait Definitions

**Find the current trait definition section and replace with:**

```markdown
## The Two-Layer API

### Layer 1: anyfs — Vfs Trait (Filesystem-Like)

```rust
/// Low-level filesystem trait. NO path handling.
/// Backend implementers work directly with inodes.
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

### Layer 2: anyfs-container — FilesContainer (std::fs-Like)

```rust
/// High-level container with std::fs-like API.
/// Application developers use this — never touch Vfs directly.
impl<V: Vfs, S: FsSemantics> FilesContainer<V, S> {
    // ═══════════════════════════════════════════════════════════
    // READ OPERATIONS (match std::fs)
    // ═══════════════════════════════════════════════════════════
    
    /// Like std::fs::read
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    
    /// Like std::fs::read_to_string
    pub fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, ContainerError>;
    
    /// Like std::fs::metadata
    pub fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    
    /// Like std::fs::symlink_metadata
    pub fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    
    /// Like std::fs::read_dir
    pub fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, ContainerError>;
    
    /// Like std::fs::read_link
    pub fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, ContainerError>;
    
    /// Like Path::exists
    pub fn exists(&self, path: impl AsRef<Path>) -> bool;
    
    // ═══════════════════════════════════════════════════════════
    // WRITE OPERATIONS (match std::fs)
    // ═══════════════════════════════════════════════════════════
    
    /// Like std::fs::write
    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    
    /// Like std::fs::create_dir
    pub fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    /// Like std::fs::create_dir_all
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    /// Like std::fs::remove_file
    pub fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    /// Like std::fs::remove_dir
    pub fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    /// Like std::fs::remove_dir_all
    pub fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    /// Like std::fs::rename
    pub fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    /// Like std::fs::copy
    pub fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    // ═══════════════════════════════════════════════════════════
    // LINK OPERATIONS (match std::fs / std::os::unix::fs)
    // ═══════════════════════════════════════════════════════════
    
    /// Like std::os::unix::fs::symlink
    pub fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    /// Like std::fs::hard_link
    pub fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    // ═══════════════════════════════════════════════════════════
    // PERMISSIONS (match std::fs)
    // ═══════════════════════════════════════════════════════════
    
    /// Like std::fs::set_permissions
    pub fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), ContainerError>;
    
    // ═══════════════════════════════════════════════════════════
    // CONTAINER-SPECIFIC (not in std::fs)
    // ═══════════════════════════════════════════════════════════
    
    /// Get current capacity usage
    pub fn usage(&self) -> &CapacityUsage;
    
    /// Get capacity limits
    pub fn limits(&self) -> &CapacityLimits;
}
```
```

### 2.4 Update Quick Reference Table

**Find the Quick Reference section and replace with:**

```markdown
## Quick Reference

| Question                          | Answer                                                      |
| --------------------------------- | ----------------------------------------------------------- |
| How many layers?                  | Two: `anyfs` (low-level) and `anyfs-container` (high-level) |
| What trait do backends implement? | `Vfs` (in `anyfs`)                                          |
| What do applications use?         | `FilesContainer` (in `anyfs-container`)                     |
| Does Vfs handle paths?            | **No** — it uses `InodeId` only                             |
| Does FilesContainer handle paths? | **Yes** — it uses `impl AsRef<Path>`                        |
| Which API matches std::fs?        | `FilesContainer` methods                                    |
| Where is path resolution?         | `FsSemantics` trait, used by `FilesContainer`               |
| Where are capacity limits?        | `FilesContainer` only                                       |
| What implements Vfs?              | `MemoryVfs`, `SqliteVfs`, `RealFsVfs`                       |
```

### 2.5 Add Common Mistakes Section

**Add to "Common Mistakes to Avoid":**

```markdown
### ❌ WRONG: Confusing the two layers

```rust
// WRONG — Vfs trait doesn't have path-based methods
impl Vfs for MyBackend {
    fn read(&self, path: &str) -> Result<Vec<u8>, VfsError> { ... }
}
```

```rust
// CORRECT — Vfs trait uses InodeId
impl Vfs for MyBackend {
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError> { ... }
}
```

### ❌ WRONG: Application code using Vfs directly

```rust
// WRONG — Applications should use FilesContainer
let vfs = MemoryVfs::new();
let ino = vfs.lookup(vfs.root(), "file.txt")?;
vfs.read(ino, 0, &mut buf)?;
```

```rust
// CORRECT — Applications use std::fs-like API
let container = FilesContainer::new(MemoryVfs::new(), LinuxSemantics);
let data = container.read("/file.txt")?;
```

### ❌ WRONG: Backend handling path resolution

```rust
// WRONG — Backends shouldn't parse paths
impl Vfs for MyBackend {
    fn create_file(&mut self, path: &str) -> Result<(), VfsError> {
        let components = path.split('/');  // NO! Not your job!
        ...
    }
}
```

```rust
// CORRECT — Backends just store inodes
impl Vfs for MyBackend {
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        let id = self.next_id();
        self.inodes.insert(id, InodeData::new(kind, mode));
        Ok(id)
    }
}
```
```

---

## 3. README.md Updates

### 3.1 Update Overview Section

**Replace the overview with:**

```markdown
## Overview

A two-layer virtual filesystem abstraction for Rust:

| Layer      | Crate             | API Style                | For                    |
| ---------- | ----------------- | ------------------------ | ---------------------- |
| Low-level  | `anyfs`           | Filesystem-like (inodes) | Backend implementers   |
| High-level | `anyfs-container` | `std::fs`-like (paths)   | Application developers |

```
┌─────────────────────────────────────────────────────────────┐
│  Application: container.write("/data/file.txt", b"hello")  │
├─────────────────────────────────────────────────────────────┤
│  anyfs-container: std::fs-like API + capacity limits       │
├─────────────────────────────────────────────────────────────┤
│  anyfs: Vfs trait (inode operations)                       │
├──────────────────┬──────────────────┬───────────────────────┤
│  MemoryVfs       │  SqliteVfs       │  RealFsVfs            │
└──────────────────┴──────────────────┴───────────────────────┘
```
```

### 3.2 Update Quick Example

**Replace the example with:**

```markdown
## Quick Example

### For Application Developers (std::fs-like)

```rust
use anyfs_container::{FilesContainer, LinuxSemantics};
use anyfs::MemoryVfs;

// Create container with memory backend
let mut container = FilesContainer::new(
    MemoryVfs::new(),
    LinuxSemantics::new(),
);

// Use exactly like std::fs!
container.create_dir_all("/data/nested")?;
container.write("/data/nested/file.txt", b"hello")?;

let content = container.read("/data/nested/file.txt")?;
assert_eq!(content, b"hello");

for entry in container.read_dir("/data")? {
    println!("{}", entry.name());
}
```

### For Backend Implementers (Low-Level)

```rust
use anyfs::{Vfs, InodeId, InodeKind, InodeData, VfsError};

struct MyCustomVfs { /* ... */ }

impl Vfs for MyCustomVfs {
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        // Allocate and store inode in your storage
    }
    
    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError> {
        // Find child by name in parent directory
    }
    
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError> {
        // Read bytes from file content
    }
    
    // ... implement other Vfs methods
}
```
```

---

## 4. New Files to Create

### 4.1 Create `book/src/architecture/two-layer-design.md`

```markdown
# Two-Layer Architecture

## Overview

The anyfs ecosystem separates concerns into two layers:

1. **anyfs** — Low-level storage (inodes)
2. **anyfs-container** — High-level API (paths)

## Why Two Layers?

| Concern                | Layer                         |
| ---------------------- | ----------------------------- |
| How to store data      | anyfs (Vfs trait)             |
| How to interpret paths | anyfs-container (FsSemantics) |
| Capacity limits        | anyfs-container               |
| User-facing API        | anyfs-container               |

## Layer 1: anyfs

The `Vfs` trait defines filesystem-like operations on **inodes**, not paths:

- `create_inode()` — Allocate new inode
- `lookup()` — Find child in directory
- `link()` / `unlink()` — Manage directory entries
- `read()` / `write()` — Content I/O with offset

**Key insight:** No path parsing. No semantics. Just inode storage.

## Layer 2: anyfs-container

The `FilesContainer` provides a `std::fs`-like API:

- `read()` — Like `std::fs::read`
- `write()` — Like `std::fs::write`
- `create_dir_all()` — Like `std::fs::create_dir_all`
- etc.

**Key insight:** Handles path resolution, enforces limits, delegates to Vfs.

## How They Connect

```rust
// User calls std::fs-like API
container.write("/data/file.txt", b"hello")?;

// FilesContainer internally:
// 1. Parse path via FsSemantics
let components = semantics.components("/data/file.txt");

// 2. Walk to parent via Vfs
let mut ino = vfs.root();
ino = vfs.lookup(ino, "data")?;

// 3. Check capacity
self.check_capacity(data.len())?;

// 4. Create/update file via Vfs
let file_ino = vfs.create_inode(InodeKind::File, 0o644)?;
vfs.link(ino, "file.txt", file_ino)?;
vfs.write(file_ino, 0, data)?;
```
```

### 4.2 Create `book/src/traits/vfs-trait.md`

```markdown
# Vfs Trait (anyfs)

The `Vfs` trait is the **low-level** filesystem interface.

## Target Audience

**Backend implementers** — people creating new storage backends.

**Not for:** Application developers (use `FilesContainer` instead).

## Key Characteristics

| Aspect          | Description               |
| --------------- | ------------------------- |
| Path handling   | **None** — uses `InodeId` |
| Semantics       | **None** — raw operations |
| Capacity limits | **None** — just storage   |

## Trait Definition

```rust
pub trait Vfs: Send {
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError>;
    fn get_inode(&self, id: InodeId) -> Result<InodeData, VfsError>;
    fn update_inode(&mut self, id: InodeId, data: InodeData) -> Result<(), VfsError>;
    fn delete_inode(&mut self, id: InodeId) -> Result<(), VfsError>;
    
    fn link(&mut self, parent: InodeId, name: &str, child: InodeId) -> Result<(), VfsError>;
    fn unlink(&mut self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn readdir(&self, dir: InodeId) -> Result<Vec<(String, InodeId)>, VfsError>;
    
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError>;
    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError>;
    fn truncate(&mut self, id: InodeId, size: u64) -> Result<(), VfsError>;
    
    fn sync(&mut self) -> Result<(), VfsError>;
    fn root(&self) -> InodeId;
}
```

## Implementing a Backend

See [Backend Implementer's Guide](../guides/backend-implementer.md).
```

### 4.3 Create `book/src/traits/files-container.md`

```markdown
# FilesContainer (anyfs-container)

`FilesContainer` is the **high-level** std::fs-like API.

## Target Audience

**Application developers** — people building apps on top of anyfs.

## Key Characteristics

| Aspect          | Description                                                    |
| --------------- | -------------------------------------------------------------- |
| Path handling   | `impl AsRef<Path>` — accepts `&str`, `String`, `PathBuf`, etc. |
| Semantics       | Pluggable via `FsSemantics`                                    |
| Capacity limits | Enforced before operations                                     |
| API style       | Matches `std::fs` method names                                 |

## Method Mapping to std::fs

| FilesContainer       | std::fs                      |
| -------------------- | ---------------------------- |
| `read()`             | `std::fs::read`              |
| `read_to_string()`   | `std::fs::read_to_string`    |
| `write()`            | `std::fs::write`             |
| `read_dir()`         | `std::fs::read_dir`          |
| `create_dir()`       | `std::fs::create_dir`        |
| `create_dir_all()`   | `std::fs::create_dir_all`    |
| `remove_file()`      | `std::fs::remove_file`       |
| `remove_dir()`       | `std::fs::remove_dir`        |
| `remove_dir_all()`   | `std::fs::remove_dir_all`    |
| `rename()`           | `std::fs::rename`            |
| `copy()`             | `std::fs::copy`              |
| `metadata()`         | `std::fs::metadata`          |
| `symlink_metadata()` | `std::fs::symlink_metadata`  |
| `symlink()`          | `std::os::unix::fs::symlink` |
| `hard_link()`        | `std::fs::hard_link`         |
| `read_link()`        | `std::fs::read_link`         |
| `set_permissions()`  | `std::fs::set_permissions`   |
| `exists()`           | `Path::exists`               |

## Usage Example

```rust
use anyfs_container::{FilesContainer, LinuxSemantics, CapacityLimits};
use anyfs::MemoryVfs;

let mut container = FilesContainer::builder()
    .vfs(MemoryVfs::new())
    .semantics(LinuxSemantics::new())
    .limits(CapacityLimits {
        max_total_size: Some(10 * 1024 * 1024),  // 10 MB
        max_file_size: Some(1024 * 1024),         // 1 MB per file
        ..Default::default()
    })
    .build()?;

// Now use like std::fs
container.create_dir_all("/var/log")?;
container.write("/var/log/app.log", b"Started\n")?;
```
```

### 4.4 Create `book/src/guides/which-layer.md`

```markdown
# Which Layer Should I Use?

## Decision Tree

```
Are you building an application?
│
├── YES → Use `FilesContainer` from `anyfs-container`
│         (std::fs-like API, handles paths for you)
│
└── NO → Are you creating a new storage backend?
         │
         ├── YES → Implement `Vfs` trait from `anyfs`
         │         (low-level inode operations)
         │
         └── NO → Are you creating custom path semantics?
                  │
                  ├── YES → Implement `FsSemantics` trait
                  │
                  └── NO → You probably want `FilesContainer`
```

## Examples

| Use Case                             | Layer     | Crate                           |
| ------------------------------------ | --------- | ------------------------------- |
| Building a web app with file storage | High      | `anyfs-container`               |
| Creating an S3-backed storage        | Low       | `anyfs` (implement Vfs)         |
| Adding Windows path support          | Semantics | `anyfs-semantics`               |
| Testing file operations              | High      | `anyfs-container` + `MemoryVfs` |
| Building a FUSE adapter              | Low       | `anyfs` (use Vfs directly)      |
```

---

## 5. Existing Files to Update

### 5.1 Update `book/src/SUMMARY.md`

Add new sections:

```markdown
# Summary

- [Introduction](./introduction.md)

# Architecture

- [Two-Layer Design](./architecture/two-layer-design.md)
- [Why Inodes?](./architecture/inode-model.md)
- [Simulated vs Real I/O](./architecture/simulated-filesystem.md)

# Traits

- [Vfs Trait (Low-Level)](./traits/vfs-trait.md)
- [FilesContainer (High-Level)](./traits/files-container.md)
- [FsSemantics](./traits/fs-semantics.md)

# Guides

- [Which Layer Should I Use?](./guides/which-layer.md)
- [For Application Developers](./guides/application-developer.md)
- [For Backend Implementers](./guides/backend-implementer.md)

# Backends

- [MemoryVfs](./backends/memory.md)
- [SqliteVfs](./backends/sqlite.md)
- [RealFsVfs](./backends/realfs.md)

# Semantics

- [LinuxSemantics](./semantics/linux.md)
- [WindowsSemantics](./semantics/windows.md)
- [SimpleSemantics](./semantics/simple.md)

# Appendix

- [std::fs Method Mapping](./appendix/std-fs-mapping.md)
- [Design Decisions](./appendix/design-decisions.md)
```

### 5.2 Update Any File Using Old Trait Names

Search for and replace:

| Find          | Replace With                                       |
| ------------- | -------------------------------------------------- |
| `VfsBackend`  | `Vfs` (if referring to low-level trait)            |
| `VfsBackend`  | `FilesContainer` (if referring to user-facing API) |
| `list(`       | `read_dir(`                                        |
| `mkdir(`      | `create_dir(`                                      |
| `mkdir_all(`  | `create_dir_all(`                                  |
| `remove(`     | `remove_file(` or `remove_dir(`                    |
| `remove_all(` | `remove_dir_all(`                                  |

---

## 6. mdbook Structure Updates

### 6.1 Recommended Directory Structure

```
book/
├── book.toml
└── src/
    ├── SUMMARY.md
    ├── introduction.md
    ├── architecture/
    │   ├── two-layer-design.md      # NEW
    │   ├── inode-model.md           # NEW  
    │   └── simulated-filesystem.md  # From clarification doc
    ├── traits/
    │   ├── vfs-trait.md             # NEW
    │   ├── files-container.md       # NEW
    │   └── fs-semantics.md
    ├── guides/
    │   ├── which-layer.md           # NEW
    │   ├── application-developer.md
    │   └── backend-implementer.md
    ├── backends/
    │   ├── memory.md
    │   ├── sqlite.md
    │   └── realfs.md
    ├── semantics/
    │   ├── linux.md
    │   ├── windows.md
    │   └── simple.md
    └── appendix/
        ├── std-fs-mapping.md        # NEW
        └── design-decisions.md
```

---

## 7. Diagrams to Add

### 7.1 Layer Separation Diagram

Add this ASCII diagram wherever the architecture is explained:

```
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION CODE                         │
│                                                             │
│   // This looks just like std::fs!                          │
│   container.write("/data/file.txt", b"hello")?;             │
│   container.read_dir("/data")?;                             │
│   container.create_dir_all("/var/log")?;                    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   anyfs-container: FilesContainer<V, S>                     │
│   ═══════════════════════════════════════                   │
│                                                             │
│   ┌─────────────────┐  ┌─────────────────┐                  │
│   │  Path Resolution │  │ Capacity Limits │                  │
│   │  (FsSemantics)   │  │ (CapacityLimits)│                  │
│   └────────┬────────┘  └────────┬────────┘                  │
│            │                    │                           │
│            └────────┬───────────┘                           │
│                     ▼                                       │
│            ┌───────────────┐                                │
│            │ std::fs-like  │                                │
│            │     API       │                                │
│            └───────┬───────┘                                │
│                    │                                        │
├────────────────────┼────────────────────────────────────────┤
│                    ▼                                        │
│   anyfs: Vfs trait                                          │
│   ════════════════════                                      │
│                                                             │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│   │create_inode │ │   lookup    │ │    read     │           │
│   │delete_inode │ │   link      │ │    write    │           │
│   │ get_inode   │ │   unlink    │ │  truncate   │           │
│   │update_inode │ │   readdir   │ │    sync     │           │
│   └─────────────┘ └─────────────┘ └─────────────┘           │
│                                                             │
│   NO PATHS — ONLY InodeId                                   │
│                                                             │
├──────────────────┬──────────────────┬───────────────────────┤
│                  │                  │                       │
│   MemoryVfs      │   SqliteVfs      │   RealFsVfs           │
│   ══════════     │   ═════════      │   ═════════           │
│   HashMap        │   .db file       │   strict-path         │
│                  │                  │                       │
└──────────────────┴──────────────────┴───────────────────────┘
```

### 7.2 Method Flow Diagram

```
container.write("/data/file.txt", b"hello")
                    │
                    ▼
    ┌───────────────────────────────┐
    │  1. Parse path (FsSemantics)  │
    │     "/data/file.txt"          │
    │     → ["data", "file.txt"]    │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │  2. Resolve to parent inode   │
    │     vfs.root() → InodeId(1)   │
    │     vfs.lookup(1, "data")     │
    │     → InodeId(5)              │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │  3. Check capacity limits     │
    │     usage + 5 bytes ≤ limit?  │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │  4. Create/get file inode     │
    │     vfs.lookup(5, "file.txt") │
    │     or vfs.create_inode(...)  │
    │     → InodeId(12)             │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │  5. Write content             │
    │     vfs.truncate(12, 0)       │
    │     vfs.write(12, 0, b"hello")│
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │  6. Update usage tracking     │
    │     self.usage.total_size += 5│
    └───────────────────────────────┘
```

---

## 8. Checklist

### 8.1 AGENTS.md

- [ ] Replace project overview with two-layer description
- [ ] Replace architecture diagram
- [ ] Replace trait definitions (Vfs + FilesContainer)
- [ ] Update quick reference table
- [ ] Add layer confusion to "Common Mistakes"
- [ ] Remove references to `VfsBackend` (old name)
- [ ] Remove references to old method names (`list`, `mkdir`, etc.)

### 8.2 README.md

- [ ] Update overview with two-layer table
- [ ] Update architecture diagram
- [ ] Add two quick examples (application dev + backend implementer)
- [ ] Update status section if needed

### 8.3 New Files

- [ ] Create `book/src/architecture/two-layer-design.md`
- [ ] Create `book/src/traits/vfs-trait.md`
- [ ] Create `book/src/traits/files-container.md`
- [ ] Create `book/src/guides/which-layer.md`
- [ ] Create `book/src/appendix/std-fs-mapping.md`

### 8.4 Existing Files

- [ ] Update `book/src/SUMMARY.md` with new structure
- [ ] Search/replace old trait names
- [ ] Search/replace old method names
- [ ] Add diagrams where appropriate

### 8.5 Consistency Check

- [ ] All references to "VfsBackend" removed or updated
- [ ] All references to `list()`, `mkdir()` etc. updated
- [ ] No confusion between Vfs (inodes) and FilesContainer (paths)
- [ ] Clear indication of target audience for each section

---

## Summary

The key change is recognizing the **two-layer API design**:

| Layer    | Trait/Struct     | API Style       | Works With         | Target               |
| -------- | ---------------- | --------------- | ------------------ | -------------------- |
| **Low**  | `Vfs`            | Filesystem-like | `InodeId`          | Backend implementers |
| **High** | `FilesContainer` | `std::fs`-like  | `impl AsRef<Path>` | App developers       |

This separation means:
- Backend implementers never deal with paths
- App developers never deal with inodes
- Semantics are pluggable at the container level
- Capacity limits are enforced at the container level

Update all documentation to reflect this clean separation.
