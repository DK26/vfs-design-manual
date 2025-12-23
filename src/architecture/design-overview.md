# AnyFS - Design Overview

**Status:** Current
**Last updated:** 2025-12-23

---

## What This Project Is

AnyFS is an **open standard** for pluggable virtual filesystem backends in Rust. Anyone can implement a custom backend for their own storage needs—whether that's a database, object store, network filesystem, or any other medium.

The ecosystem provides:
- **A minimal backend trait** that any storage implementer can satisfy
- **Built-in backends** for common cases (memory, SQLite, real filesystem)
- **An optional policy layer** (`FilesContainer`) for quotas, limits, and feature whitelisting

It is designed for:
- portable application storage (SQLite backend = single `.db` file)
- tenant isolation (one container per tenant)
- quotas/limits (enforced in the container layer)
- high-performance use cases (in-memory backend for speed-critical applications)

It is not a POSIX emulator and does not try to expose OS-specific behavior.

---

## Crates

| Crate | Purpose | Who uses it |
|------|---------|-------------|
| `anyfs-backend` | Minimal contract: `VfsBackend` trait + core types | Backend implementers |
| `anyfs` | Low-level execution layer for calling any `VfsBackend`. Also provides built-in backends (feature-gated). | Direct backend manipulation |
| `anyfs-container` | `FilesContainer<B: VfsBackend>` + limits + feature whitelist (least privilege) | Application code |

### Dependency Graph

```
anyfs-backend (trait + types)
    <- anyfs (calls any VfsBackend, provides built-in backends)
    <- anyfs-container (wraps backends with policy)

strict-path (VirtualPath, VirtualRoot)
    <- anyfs [vrootfs feature only]
```

**Key point:** `anyfs-backend` defines the contract. `anyfs` is the execution layer that can call any backend—built-in or custom.

---

## Path Handling

AnyFS uses `impl AsRef<Path>` throughout, aligned with `std::fs`:

1. **User-facing API (`FilesContainer`)** accepts `impl AsRef<Path>`.
2. **Backend API (`VfsBackend`)** accepts `impl AsRef<Path>`.

This keeps the API familiar and works correctly across all platforms.

---

## Core Trait: `VfsBackend` (in `anyfs-backend`)

`VfsBackend` is a path-based trait aligned with `std::fs` naming and signatures.

```rust
use std::path::Path;

pub trait VfsBackend: Send {
    // Read
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;

    // Write
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;

    // Links
    fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;

    // Permissions
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;

    // Streaming I/O (for large files)
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;
}
```

**Semantics (high-level):**
- `read`/`write`/`metadata`/`exists`/`copy` follow symlinks.
- `symlink_metadata` and `read_link` do not follow.
- `remove_file` removes the symlink itself, not the target.
- Streaming methods (`open_read`, `open_write`) enable efficient handling of large files.

---

## Built-in Backends (in `anyfs`)

These are provided by the `anyfs` crate and selected via Cargo features:

- `memory` (default): `MemoryBackend` (test/reference backend)
- `vrootfs`: `VRootFsBackend` (contained real filesystem via `strict_path::VirtualRoot`)
- `sqlite`: `SqliteBackend` (single-file portable database backend)

---

## `FilesContainer` (in `anyfs-container`)

`FilesContainer<B>` wraps any backend and provides:
- ergonomic paths (`impl AsRef<Path>`)
- quota enforcement (limits)
- a **feature whitelist** for advanced behavior (least privilege)

### Feature Whitelist (Least Privilege)

Advanced features are **disabled by default**. Enable only what you need:

- `symlinks` gates symlink creation and symlink-following behavior
- `hard_links` gates hard-link creation
- `permissions` gates permission mutation via `set_permissions`

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

let mut container = FilesContainer::new(MemoryBackend::new())
    .with_symlinks()
    .with_max_symlink_resolution(40)
    .with_hard_links()
    .with_permissions()
    .with_max_total_size(100 * 1024 * 1024);
```

When a feature is disabled, operations that require it return `ContainerError::FeatureNotEnabled("...")`.

### Limits

Limits are enforced by the container layer (not by backends):
- `max_total_size`
- `max_file_size`
- `max_node_count`
- `max_dir_entries`
- `max_path_depth`

---

## Security Model (Summary)

Security and isolation are achieved differently depending on the backend:

| Backend | Isolation Mechanism |
|---------|---------------------|
| `MemoryBackend` | Each instance is isolated by OS process memory |
| `SqliteBackend` | Each container is a separate `.db` file |
| `VRootFsBackend` | Uses `strict-path::VirtualRoot` to clamp all paths to a root directory |

**Common guarantees:**
- **Least privilege:** the container disables advanced features by default and requires explicit opt-in.
- **Path normalization:** all user paths are normalized before reaching the backend.
- **No host paths in API:** application code interacts only with virtual paths; backends decide actual storage.

For more detail, see `book/src/comparisons/security.md`.