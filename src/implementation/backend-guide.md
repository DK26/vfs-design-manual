# Backend Implementer's Guide

This guide walks you through implementing a custom AnyFS backend.

---

## Overview

A backend implements `VfsBackend`: a path-based trait aligned with `std::fs`.

Key properties:
- Backends accept `impl AsRef<Path>` for all path parameters
- Backends handle **storage + filesystem semantics only**
- Policy (limits, feature gates) is handled by middleware, not backends

---

## Dependency

Depend only on `anyfs-backend`:

```toml
[dependencies]
anyfs-backend = "0.1"
```

---

## Trait to Implement

```rust
use anyfs_backend::{VfsBackend, VfsError, Metadata, DirEntry, Permissions, StatFs};
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

pub struct MyBackend {
    // Your storage fields
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

## Implementation Steps

### Step 1: Pick a Data Model

Your backend needs internal storage. Options:

- **HashMap-based**: `HashMap<PathBuf, Entry>` for simple cases
- **Tree-based**: Explicit directory tree structure
- **Database-backed**: SQLite, key-value store, etc.

Minimum metadata per entry:
- File type (file/directory/symlink)
- Size (for files)
- Content (for files)
- Timestamps (optional)
- Permissions (optional)

### Step 2: Implement Read Operations

Start with these (easiest):

```rust
fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;
```

### Step 3: Implement Write Operations

```rust
fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
```

### Step 4: Implement Links

```rust
fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
```

- Symlinks store a target path as a string
- Hard links share content with the original (update link count)

### Step 5: Implement Permissions and Streaming

```rust
fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;
fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;
```

**Streaming implementation options:**

For `MemoryBackend` or similar, you can use `std::io::Cursor`:

```rust
fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError> {
    let data = self.read(path)?;
    Ok(Box::new(std::io::Cursor::new(data)))
}
```

For `VRootFsBackend`, return the actual file handle:

```rust
fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError> {
    let file = std::fs::File::open(self.resolve(path)?)?;
    Ok(Box::new(file))
}
```

### Step 6: Implement truncate

```rust
fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError>;
```

- If `size < current`: discard trailing bytes
- If `size > current`: extend with zero bytes
- Required for FUSE support and editor save operations

### Step 7: Implement sync/fsync

```rust
fn sync(&mut self) -> Result<(), VfsError>;
fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
```

- `sync()`: Flush all pending writes to durable storage
- `fsync(path)`: Flush pending writes for a specific file
- `MemoryBackend` can no-op these (volatile by design)
- `SqliteBackend`: `PRAGMA wal_checkpoint` or connection flush
- `VRootFsBackend`: `std::fs::File::sync_all()`

### Step 8: Implement statfs

```rust
fn statfs(&self) -> Result<StatFs, VfsError>;
```

Return filesystem capacity information:

```rust
StatFs {
    total_bytes: 0,      // 0 = unlimited
    used_bytes: ...,
    available_bytes: ...,
    total_inodes: 0,
    used_inodes: ...,
    available_inodes: ...,
    block_size: 4096,
    max_name_len: 255,
}
```

---

## Error Handling

Return appropriate `VfsError` variants:

| Situation | Error |
|-----------|-------|
| Path doesn't exist | `VfsError::NotFound(path)` |
| Path already exists | `VfsError::AlreadyExists(path)` |
| Expected file, got dir | `VfsError::NotAFile(path)` |
| Expected dir, got file | `VfsError::NotADirectory(path)` |
| Remove non-empty dir | `VfsError::DirectoryNotEmpty(path)` |
| Internal error | `VfsError::Backend(message)` |

---

## What Backends Do NOT Do

| Concern | Where It Lives |
|---------|----------------|
| Quota enforcement | `Quota<B>` middleware |
| Feature gating | `FeatureGuard<B>` middleware |
| Logging | `Tracing<B>` middleware |
| Ergonomic API | `FilesContainer<B>` wrapper |

**Backends focus on storage.** Keep them simple.

---

## Testing Your Backend

Use the conformance test suite:

```rust
#[cfg(test)]
mod tests {
    use super::MyBackend;
    use anyfs_backend::VfsBackend;

    fn create_backend() -> MyBackend {
        MyBackend::new()
    }

    #[test]
    fn test_write_read() {
        let mut backend = create_backend();
        backend.write("/test.txt", b"hello").unwrap();
        let content = backend.read("/test.txt").unwrap();
        assert_eq!(content, b"hello");
    }

    #[test]
    fn test_create_dir() {
        let mut backend = create_backend();
        backend.create_dir("/foo").unwrap();
        assert!(backend.exists("/foo").unwrap());
    }

    // ... more tests
}
```

---

## Note on VRootFsBackend

If you are implementing a backend that wraps a **real host filesystem directory**, consider using `strict-path::VirtualPath` and `strict-path::VirtualRoot` internally for path containment. This ensures paths cannot escape the designated root directory.

This is an implementation choice for filesystem-based backends, not a requirement of the `VfsBackend` trait.

---

## For Middleware Authors: Wrapping Streams

Middleware that needs to intercept streaming I/O must wrap the returned `Box<dyn Read/Write>`.

### CountingWriter Example

```rust
use std::io::{self, Write};
use std::sync::{Arc, atomic::{AtomicU64, Ordering}};

pub struct CountingWriter<W: Write> {
    inner: W,
    bytes_written: Arc<AtomicU64>,
}

impl<W: Write> CountingWriter<W> {
    pub fn new(inner: W, counter: Arc<AtomicU64>) -> Self {
        Self { inner, bytes_written: counter }
    }
}

impl<W: Write + Send> Write for CountingWriter<W> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        let n = self.inner.write(buf)?;
        self.bytes_written.fetch_add(n as u64, Ordering::Relaxed);
        Ok(n)
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}
```

### Using in Quota Middleware

```rust
impl<B: VfsBackend> VfsBackend for Quota<B> {
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError> {
        // Check if we're at quota before opening
        if self.usage.total_bytes >= self.limits.max_total_size {
            return Err(VfsError::QuotaExceeded { ... });
        }

        let inner = self.inner.open_write(path)?;
        Ok(Box::new(CountingWriter::new(inner, self.usage.bytes_counter.clone())))
    }
}
```

### Alternatives to Wrapping

| Middleware | Alternative to wrapping |
|------------|------------------------|
| PathFilter | Check path at open time, pass stream through |
| ReadOnly | Block `open_write` entirely |
| RateLimit | Count the open call, not stream bytes |
| Tracing | Log the open call, pass stream through |
| DryRun | Return `std::io::sink()` instead of real writer |

---

## Checklist

- [ ] Depends only on `anyfs-backend`
- [ ] Implements all `VfsBackend` methods (25 methods)
- [ ] Accepts `impl AsRef<Path>` for all paths
- [ ] Returns correct `VfsError` variants
- [ ] Handles symlinks correctly (stores target path)
- [ ] Handles hard links correctly (shares content)
- [ ] Implements `truncate` (shrink/extend files)
- [ ] Implements `sync`/`fsync` (durability)
- [ ] Passes conformance tests
