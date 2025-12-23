# AGENTS.md - Instructions for AI Assistants

READ THIS FIRST before making any changes to this repository.

Priority: reviews/comments.md is the highest-priority source of truth. If anything conflicts, follow reviews/comments.md and update this file.

---

## Project Overview

AnyFS is an open standard for pluggable virtual filesystem backends in Rust. It is not tied to any specific storage medium.

This is a three-crate ecosystem:

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Minimal contract: `VfsBackend` trait + core types (backend implementers depend on this) |
| `anyfs` | Execution layer that calls any `VfsBackend`; provides built-in backends (feature-gated) |
| `anyfs-container` | Policy layer: limits + least-privilege feature whitelist |

---

## Single Current Design

There is only one design. Do not mention older iterations or deprecated APIs in docs.

---

## Core Design (authoritative)

- Trait crate: `anyfs-backend` defines `VfsBackend` with std::fs-aligned, path-based methods.
- Backend path type: `impl AsRef<Path>` (aligned with std::fs). Do not use `VirtualPath` in the core trait.
- FilesContainer API: `impl AsRef<Path>` for ergonomics.
- `strict-path` is optional and only relevant to `VRootFsBackend` (real filesystem containment). It is not part of the core AnyFS API.
- Streaming I/O should be treated as in-scope (do not label as "future").
- Prefer a single `FilesContainer` type as the main API. Avoid a separate `ContainerBuilder`.
- Backends may expose their own builders (for example, `VRootFsBackend::new().with_root(...).build()`).

Path resolution guidance:
- Virtual backends (memory, SQLite) can apply lexical rules internally.
- VRootFsBackend delegates path resolution and symlink behavior to the OS; containment is enforced at the backend boundary.

---

## Architecture

```
+------------------------------------+
| User Application                   |
+------------------------------------+
| anyfs-container                    |  policy (limits + feature whitelist)
| FilesContainer<B: VfsBackend>      |
+------------------------------------+
| anyfs                              |  built-in backends (feature-gated)
+------------+-----------+-----------+
| VRootFs    | Memory    | SQLite    |
| Backend    | Backend   | Backend   |
+------------+-----------+-----------+
| anyfs-backend                     |  trait + core types
+------------------------------------+
```

`VRootFsBackend` (anyfs feature `vrootfs`) may use `strict-path` internally for containment.

### Dependency Graph

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (execution layer; calls any VfsBackend; built-in backends)
     |     ^-- vrootfs feature may use strict-path (optional)
     |
     +-- anyfs-container (policy layer; wraps any VfsBackend)
```

Both layers use `impl AsRef<Path>` for std::fs alignment.

---

## VfsBackend Trait (in anyfs-backend)

```rust
use std::io::{Read, Write};
use std::path::Path;

/// A virtual filesystem backend.
/// Method names and signatures align with std::fs.
pub trait VfsBackend: Send {
    // READ OPERATIONS
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;

    // WRITE OPERATIONS
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;

    // LINKS
    fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;

    // PERMISSIONS
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;

    // STREAMING I/O
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;
}
```

---

## FilesContainer (User-Facing API)

```rust
use std::path::{Path, PathBuf};
use anyfs_backend::VfsBackend;

impl<B: VfsBackend> FilesContainer<B> {
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    pub fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, ContainerError>;
    pub fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, ContainerError>;
    pub fn exists(&self, path: impl AsRef<Path>) -> Result<bool, ContainerError>;
    pub fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, ContainerError>;
    pub fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, ContainerError>;

    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;

    pub fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), ContainerError>;
}
```

### Least-privilege feature whitelist (container policy)

AnyFS applies a default-deny posture at the container layer:

- Advanced behavior is disabled by default.
- Enable only what you need on `FilesContainer` (no `enable_*` prefix): `.symlinks()`, `.hard_links()`, `.permissions()`.
- Symlink resolution is bounded by `max_symlink_resolution` (default 40).

Runtime policy is not compile-time gated. Cargo features are only for selecting optional built-in backends.

---

## The three backends

### 1. VRootFsBackend

- Uses `strict-path::VirtualRoot` for containment (optional tool used only here).
- A real directory on disk acts as the virtual root.
- Paths are clamped (for example, `/etc/passwd` -> `root_dir/etc/passwd`).
- Name conveys "Virtual Root Filesystem" containment.

### 2. MemoryBackend

- In-memory storage.
- Uses normalized string paths as keys.
- Useful for testing, caching, and speed-critical applications.

### 3. SqliteBackend

- Single `.db` file contains the entire filesystem.
- Portable: copy the file to move the container.
- Internal schema is an implementation detail.

---

## Key dependencies

| Crate | Used by | Purpose |
|-------|---------|---------|
| `thiserror` | `anyfs-backend` | Error types |
| `strict-path` | `anyfs` [vrootfs] | `VirtualRoot` containment for VRootFsBackend |
| `rusqlite` | `anyfs` [sqlite] | SQLite database access |

---

## Common mistakes to avoid

- Do not use `VirtualPath` or `strict-path` in the core trait or FilesContainer API.
- Do not imply strict-path provides security guarantees for all backends.
- Do not present a separate `ContainerBuilder` as the primary API.
- Do not describe the feature whitelist as a compile-time setting.
- Do not reference older design iterations in docs.

---

## File structure

```
anyfs-backend/
  Cargo.toml
  src/
    lib.rs
    backend.rs
    types.rs
    error.rs

anyfs/
  Cargo.toml
  src/
    lib.rs
    vrootfs/
    memory/
    sqlite/

anyfs-container/
  Cargo.toml
  src/
    lib.rs
    container.rs
    limits.rs
    error.rs
```

---

## When in doubt

1. Crate names: `anyfs-backend`, `anyfs`, `anyfs-container`
2. Trait location: `anyfs-backend`
3. `VfsBackend` path type: `impl AsRef<Path>` (std::fs aligned)
4. `FilesContainer` path type: `impl AsRef<Path>` (std::fs aligned)
5. `strict-path` is optional and only used by VRootFsBackend

If documentation conflicts with reviews/comments.md, reviews/comments.md wins.
