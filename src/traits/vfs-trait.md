# VfsBackend Trait (anyfs-backend)

**The core backend contract for AnyFS**

---

## Overview

`VfsBackend` is the minimal interface a storage backend implements.

- It is **path-based** and aligned with `std::fs` naming and signatures.
- It uses `impl AsRef<Path>` for paths (same as `std::fs`).
- It does not include quotas or application policy; that lives in `anyfs-container`.

If you are implementing a custom backend, depend only on `anyfs-backend`.

---

## Trait Surface

```rust
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

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

---

## Notes on Semantics

- `read`/`write`/`metadata`/`exists`/`copy` follow symlinks.
- `symlink_metadata` and `read_link` do not follow.
- `remove_file` removes the symlink itself, not the target.
- Streaming methods (`open_read`, `open_write`) enable efficient handling of large files.

The container layer may still deny certain operations via feature whitelisting.

---

## Implementing a Backend

- Depend on `anyfs-backend` only.
- Accept `impl AsRef<Path>` for all path parameters (aligned with std::fs).
- Implement the semantics consistently across backends (a shared conformance suite is recommended).

See `book/src/implementation/backend-guide.md` for a step-by-step guide.

