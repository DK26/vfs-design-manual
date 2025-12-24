# VfsBackend Trait (anyfs-backend)

**The core backend contract for AnyFS**

---

## Overview

`VfsBackend` is the minimal interface a storage backend implements.

- It is **path-based** and aligned with `std::fs` naming.
- It uses `impl AsRef<Path>` for paths (same as `std::fs`).
- It handles **storage + filesystem semantics only** - no policy, no limits.

Policy (limits, feature gates, logging) is handled by middleware.

---

## Trait Definition

```rust
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

pub trait VfsBackend: Send {
    // ═══════════════════════════════════════════════════════════════════════
    // READ OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // WRITE OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // LINKS
    // ═══════════════════════════════════════════════════════════════════════
    fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // PERMISSIONS
    // ═══════════════════════════════════════════════════════════════════════
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // STREAMING I/O
    // ═══════════════════════════════════════════════════════════════════════
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // FILE SIZE
    // ═══════════════════════════════════════════════════════════════════════
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // DURABILITY
    // ═══════════════════════════════════════════════════════════════════════
    fn sync(&mut self) -> Result<(), VfsError>;
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════
    // FILESYSTEM INFO
    // ═══════════════════════════════════════════════════════════════════════
    fn statfs(&self) -> Result<StatFs, VfsError>;
}
```

---

## Semantics

- `read`/`write`/`metadata`/`exists`/`copy` follow symlinks.
- `symlink_metadata` and `read_link` do not follow.
- `remove_file` removes the symlink itself, not the target.
- Streaming methods (`open_read`, `open_write`) enable large file handling.
- `truncate` resizes a file: shrinks by discarding bytes, extends with zeros.
- `sync` flushes all pending writes to durable storage.
- `fsync` flushes pending writes for a specific file.
- `statfs` returns filesystem capacity information.

---

## What VfsBackend Does NOT Do

| Concern | Where It Lives |
|---------|----------------|
| Quota enforcement | `LimitedBackend<B>` middleware |
| Feature gating | `FeatureGatedBackend<B>` middleware |
| Audit logging | `LoggingBackend<B>` middleware |
| Ergonomic API | `FilesContainer<B>` wrapper |

Backends focus on storage. Policy is middleware.

---

## Implementing a Backend

- Depend on `anyfs-backend` only.
- Accept `impl AsRef<Path>` for all path parameters.
- Return `std::fs`-like errors.

See `book/src/implementation/backend-guide.md` for a step-by-step guide.
