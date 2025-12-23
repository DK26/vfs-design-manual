# Backend Implementer's Guide

This guide walks you through implementing a custom AnyFS backend.

If you are writing a backend for other people to consume, depend only on `anyfs-backend`.

---

## What you are implementing

A backend implements `VfsBackend`: a path-based trait aligned with `std::fs` naming and signatures.

Key properties:
- Backends accept `impl AsRef<Path>` for all path parameters (same as std::fs).
- The policy layer (`anyfs-container`) handles quotas/limits and feature whitelisting.

```rust
use anyfs_backend::{DirEntry, Metadata, Permissions, VfsBackend, VfsError};
use std::io::{Read, Write};
use std::path::Path;

pub struct MyBackend {
    // storage fields...
}

impl VfsBackend for MyBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        let path = path.as_ref();
        todo!()
    }

    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        let path = path.as_ref();
        todo!()
    }

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError> {
        let path = path.as_ref();
        todo!()
    }

    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError> {
        let path = path.as_ref();
        todo!()
    }

    // ... implement all methods
}
```

---

## Step 1: Pick a data model

A practical internal model is a small "virtual inode" representation:

- **Directory**: owns a mapping `name -> child` (or can be derived from a global path index).
- **File**: points to bytes (inline, blob table, object store key, etc.).
- **Symlink**: stores a *target* path (as a string).
- **Hard links**: multiple paths refer to the same underlying file content.

You do not need to expose any of this publicly; it is an implementation detail.

### Recommended minimum metadata

Your backend needs enough metadata to populate `Metadata` and return correct errors:
- file type (file/dir/symlink)
- size
- link count (`nlink`) for hard links
- timestamps (optional)
- permissions (even if simplified)

---

## Step 2: Implement the "easy" surface first

Start with these operations:

- `exists`
- `create_dir` / `create_dir_all`
- `read_dir`
- `write` / `read` / `append` / `read_range`
- `open_read` / `open_write` (streaming I/O)
- `remove_file` / `remove_dir` / `remove_dir_all`
- `rename` / `copy`

Guidelines:

- You receive `impl AsRef<Path>` - call `.as_ref()` to get `&Path`.
- Match `std::fs`-like error expectations:
  - creating an existing directory should return `AlreadyExists`
  - `remove_dir` should fail on non-empty directories
  - `remove_file` should fail on directories

---

## Step 3: Links (symlinks + hard links)

AnyFS includes link methods in the core trait.

### Symlinks

- `symlink(original, link)` creates a symlink at `link` whose target is `original`.
- `read_link(path)` returns the stored target.
- `symlink_metadata(path)` returns metadata about the symlink itself (not the target).

Note: `anyfs-container` may deny symlink operations unless explicitly enabled (least privilege).

### Hard links

- `hard_link(original, link)` creates a second directory entry pointing at the same underlying file.
- Update `nlink` and any reference-counting you maintain.
- `remove_file` decrements link count and only deletes the underlying content when it reaches zero.

---

## Step 4: Permissions

AnyFS models permissions as a minimal type (`Permissions`) to keep the cross-platform story simple.

- `set_permissions` should mutate the stored permissions.
- Backends may implement stricter semantics internally, but the API stays small.

As with links, `anyfs-container` may deny permission mutation unless enabled.

---

## Step 5: Error mapping

Backends should return `VfsError` variants that are meaningful to callers.

Recommended principles:
- Prefer structured variants like `NotFound(String)` over generic errors.
- Keep backend-specific errors available (e.g., `VfsError::Backend(String)`) but do not leak host paths when possible.

---

## Step 6: Validate with a conformance suite

Backends drift without tests.

A useful conformance suite includes:
- directory creation/removal edge cases
- `rename` semantics for files vs directories
- symlink metadata vs metadata-following behavior
- hard link link-count accounting
- copy semantics (copy follows symlinks)

See `book/src/implementation/plan.md` for the recommended test breakdown.

---

## Checklist

- [ ] Depends only on `anyfs-backend`
- [ ] Implements all `VfsBackend` methods (including streaming I/O)
- [ ] Returns correct `std::fs`-like errors
- [ ] Symlinks store target paths as strings
- [ ] Hard links update `nlink` and share content
- [ ] Conformance suite passes

---

For the trait definition, see `book/src/traits/vfs-trait.md`.

---

## Note on VRootFsBackend

If you are implementing a backend that wraps a **real host filesystem directory**, consider using `strict-path::VirtualPath` and `strict-path::VirtualRoot` internally for path containment. This is what `VRootFsBackend` does—it ensures paths cannot escape the designated root directory.

This is an implementation choice for filesystem-based backends, not a requirement of the `VfsBackend` trait.
