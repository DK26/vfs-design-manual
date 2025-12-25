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

### Step 9: Inode Operations (optional overrides)

Inode methods have **sensible defaults**. You only need to override them for:
- **Hardlink support**: Two paths must share the same inode
- **FUSE efficiency**: Direct inode operations without path lookup

```rust
// Default implementations (you get these for free):
fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError> {
    Ok(hash(path.as_ref()))  // Hashes the path
}

fn inode_to_path(&self, _inode: u64) -> Result<PathBuf, VfsError> {
    Err(VfsError::NotSupported { operation: "inode_to_path" })
}

fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, VfsError> {
    let parent = self.inode_to_path(parent_inode)?;
    self.path_to_inode(parent.join(name))
}

fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, VfsError> {
    self.metadata(self.inode_to_path(inode)?)
}
```

**Level 1: Simple backend (use defaults)**

Don't override anything. Hardlinks won't work correctly, but basic operations will.

**Level 2: Hardlink support**

Override `path_to_inode` so hardlinked paths return the same inode:

```rust
struct Node {
    id: u64,          // Unique node ID (the inode)
    nlink: u64,       // Hard link count
    content: Vec<u8>,
    // ...
}

struct MemoryBackend {
    next_id: u64,
    nodes: HashMap<u64, Node>,           // inode -> Node
    paths: HashMap<PathBuf, u64>,        // path -> inode
}

impl VfsBackend for MemoryBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError> {
        self.paths.get(path.as_ref())
            .copied()
            .ok_or_else(|| VfsError::NotFound { path: path.as_ref().into() })
    }

    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError> {
        let inode = self.path_to_inode(&original)?;
        self.paths.insert(link.as_ref().to_path_buf(), inode);
        self.nodes.get_mut(&inode).unwrap().nlink += 1;
        Ok(())
    }
}
```

**Level 3: Full FUSE efficiency**

Override all 4 methods for O(1) inode operations:

```rust
impl VfsBackend for SqliteBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError> {
        self.conn.query_row(
            "SELECT id FROM nodes WHERE path = ?",
            [path.as_ref().to_string_lossy()],
            |row| Ok(row.get::<_, i64>(0)? as u64),
        ).map_err(|_| VfsError::NotFound { path: path.as_ref().into() })
    }

    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, VfsError> {
        self.conn.query_row(
            "SELECT path FROM nodes WHERE id = ?",
            [inode as i64],
            |row| Ok(PathBuf::from(row.get::<_, String>(0)?)),
        ).map_err(|_| VfsError::NotFound { path: format!("inode:{}", inode).into() })
    }

    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, VfsError> {
        self.conn.query_row(
            "SELECT id FROM nodes WHERE parent_id = ? AND name = ?",
            params![parent_inode as i64, name.to_string_lossy()],
            |row| Ok(row.get::<_, i64>(0)? as u64),
        ).map_err(|_| VfsError::NotFound { path: name.into() })
    }

    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, VfsError> {
        self.conn.query_row(
            "SELECT type, size, nlink, created, modified FROM nodes WHERE id = ?",
            [inode as i64],
            |row| Ok(Metadata {
                inode,
                nlink: row.get(2)?,
                // ...
            }),
        ).map_err(|_| VfsError::NotFound { path: format!("inode:{}", inode).into() })
    }
}
```

**Summary:**

| Your Backend | Override | Result |
|--------------|----------|--------|
| Simple (no hardlinks) | Nothing | Works with defaults |
| With hardlinks | `path_to_inode` | Hardlinks work correctly |
| FUSE-optimized | All 4 methods | Maximum performance |

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
| Feature gating | `Restrictions<B>` middleware |
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

## Creating Custom Middleware

Custom middleware only requires `anyfs-backend` as a dependency - same as backends.

### Dependency

```toml
[dependencies]
anyfs-backend = "0.1"
```

### Complete Example: Encryption Middleware

```rust
use anyfs_backend::{VfsBackend, Layer, VfsError, Metadata, DirEntry, Permissions, StatFs};
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

/// Middleware that encrypts/decrypts file contents transparently.
pub struct Encrypted<B: VfsBackend> {
    inner: B,
    key: [u8; 32],
}

impl<B: VfsBackend> Encrypted<B> {
    pub fn new(inner: B, key: [u8; 32]) -> Self {
        Self { inner, key }
    }

    fn encrypt(&self, data: &[u8]) -> Vec<u8> {
        // Your encryption logic here
        data.iter().map(|b| b ^ self.key[0]).collect()
    }

    fn decrypt(&self, data: &[u8]) -> Vec<u8> {
        // Your decryption logic here (symmetric for XOR)
        self.encrypt(data)
    }
}

impl<B: VfsBackend> VfsBackend for Encrypted<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        let encrypted = self.inner.read(path)?;
        Ok(self.decrypt(&encrypted))
    }

    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        let encrypted = self.encrypt(data);
        self.inner.write(path, &encrypted)
    }

    // Most methods just delegate:
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        self.inner.exists(path)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError> {
        self.inner.metadata(path)
    }

    // ... implement all 25 methods
}

/// Layer for creating Encrypted middleware.
pub struct EncryptedLayer {
    key: [u8; 32],
}

impl EncryptedLayer {
    pub fn new(key: [u8; 32]) -> Self {
        Self { key }
    }
}

impl<B: VfsBackend> Layer<B> for EncryptedLayer {
    type Backend = Encrypted<B>;

    fn layer(self, backend: B) -> Self::Backend {
        Encrypted::new(backend, self.key)
    }
}
```

### Usage

```rust
use anyfs::MemoryBackend;
use my_middleware::{EncryptedLayer, Encrypted};

// Direct construction
let fs = Encrypted::new(MemoryBackend::new(), key);

// Or via Layer trait
let fs = MemoryBackend::new()
    .layer(EncryptedLayer::new(key));
```

### Middleware Checklist

- [ ] Depends only on `anyfs-backend`
- [ ] Implements `VfsBackend` for `MyMiddleware<B: VfsBackend>`
- [ ] Implements `Layer<B>` for `MyMiddlewareLayer`
- [ ] Delegates unmodified operations to inner backend
- [ ] Handles streaming I/O appropriately (wrap, pass-through, or block)
- [ ] Documents which operations are intercepted vs delegated

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
- [ ] No panics (see below)
- [ ] Thread-safe (see below)
- [ ] Documents performance characteristics

---

## Critical Implementation Guidelines

These guidelines are derived from issues found in similar projects (`vfs`, `agentfs`). **All implementations MUST follow these.**

### 1. No Panic Policy

**NEVER use `.unwrap()` or `.expect()` in library code.**

```rust
// BAD - will panic on missing file
fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
    let entry = self.entries.get(path.as_ref()).unwrap();  // PANIC!
    Ok(entry.content.clone())
}

// GOOD - returns error
fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
    let path = path.as_ref();
    let entry = self.entries.get(path)
        .ok_or_else(|| VfsError::NotFound { path: path.to_path_buf() })?;
    Ok(entry.content.clone())
}
```

**Edge cases that must NOT panic:**
- File doesn't exist
- Directory doesn't exist
- Path is empty string
- Path is invalid UTF-8 (if using OsStr)
- Parent directory missing
- Trying to read a directory as a file
- Trying to list a file as a directory
- Concurrent access conflicts

### 2. Thread Safety

Backends must be safe for concurrent access. Use appropriate synchronization:

```rust
use std::sync::{Arc, RwLock};
use std::collections::HashMap;
use std::path::PathBuf;

pub struct MemoryBackend {
    entries: Arc<RwLock<HashMap<PathBuf, Entry>>>,
}

impl VfsBackend for MemoryBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        let entries = self.entries.read()
            .map_err(|_| VfsError::Backend("lock poisoned".into()))?;
        // ...
    }

    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        let mut entries = self.entries.write()
            .map_err(|_| VfsError::Backend("lock poisoned".into()))?;
        // ...
    }
}
```

**Common race conditions to avoid:**
- `create_dir_all` called concurrently for same path
- `read` during `write` to same file
- `read_dir` while directory is being modified
- `rename` with concurrent access to source or destination

### 3. Path Normalization

Normalize paths consistently. Recommended approach:

```rust
fn normalize_path(path: &Path) -> PathBuf {
    let mut components = Vec::new();

    for component in path.components() {
        match component {
            Component::RootDir => {
                components.clear();
                components.push(Component::RootDir);
            }
            Component::CurDir => {
                // Skip "."
            }
            Component::ParentDir => {
                // Go up, but don't go past root
                if components.len() > 1 {
                    components.pop();
                }
            }
            Component::Normal(name) => {
                components.push(Component::Normal(name));
            }
            Component::Prefix(_) => {
                // Windows prefix - handle appropriately
            }
        }
    }

    components.iter().collect()
}
```

**Path edge cases to test:**
| Input | Expected Output |
|-------|-----------------|
| `/foo/../bar` | `/bar` |
| `/foo/./bar` | `/foo/bar` |
| `//double//slash` | `/double/slash` |
| `/` | `/` |
| `` (empty) | Error |
| `/foo/bar/` | `/foo/bar` (or keep trailing, be consistent) |

### 4. Error Messages

Include context in errors for debugging:

```rust
// BAD - no context
Err(VfsError::NotFound)

// GOOD - includes path
Err(VfsError::NotFound { path: path.to_path_buf() })

// BETTER - includes operation context
Err(VfsError::Io {
    path: path.to_path_buf(),
    operation: "read",
    source: io_error,
})
```

### 5. Drop Implementation

Ensure cleanup happens correctly:

```rust
impl Drop for SqliteBackend {
    fn drop(&mut self) {
        // Flush any pending writes
        if let Err(e) = self.sync() {
            eprintln!("Warning: failed to sync on drop: {}", e);
        }
    }
}
```

### 6. Performance Documentation

Document the complexity of operations:

```rust
/// Memory-based virtual filesystem backend.
///
/// # Performance Characteristics
///
/// | Operation | Complexity | Notes |
/// |-----------|------------|-------|
/// | `read` | O(1) | HashMap lookup |
/// | `write` | O(n) | n = data size |
/// | `read_dir` | O(k) | k = entries in directory |
/// | `create_dir_all` | O(d) | d = path depth |
/// | `remove_dir_all` | O(n) | n = total descendants |
///
/// # Thread Safety
///
/// All operations are thread-safe. Uses `RwLock` internally.
/// Multiple concurrent reads are allowed.
/// Writes are exclusive.
pub struct MemoryBackend { ... }
```

---

## Testing Requirements

Your backend MUST pass these test categories:

### Basic Functionality
```rust
#[test]
fn test_read_write_roundtrip() { ... }

#[test]
fn test_create_dir_and_list() { ... }

#[test]
fn test_remove_file() { ... }
```

### Edge Cases (No Panics)
```rust
#[test]
fn test_read_nonexistent_returns_error() {
    let backend = create_backend();
    assert!(matches!(
        backend.read("/nonexistent"),
        Err(VfsError::NotFound { .. })
    ));
}

#[test]
fn test_read_dir_on_file_returns_error() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"data").unwrap();
    assert!(matches!(
        backend.read_dir("/file.txt"),
        Err(VfsError::NotADirectory { .. })
    ));
}

#[test]
fn test_empty_path_returns_error() {
    let backend = create_backend();
    assert!(backend.read("").is_err());
}
```

### Thread Safety
```rust
#[test]
fn test_concurrent_reads() {
    let backend = Arc::new(create_backend_with_data());
    let handles: Vec<_> = (0..10).map(|_| {
        let backend = backend.clone();
        std::thread::spawn(move || {
            for _ in 0..100 {
                backend.read("/test.txt").unwrap();
            }
        })
    }).collect();

    for handle in handles {
        handle.join().unwrap();
    }
}

#[test]
fn test_concurrent_create_dir_all() {
    let backend = Arc::new(RwLock::new(create_backend()));
    let handles: Vec<_> = (0..10).map(|_| {
        let backend = backend.clone();
        std::thread::spawn(move || {
            let mut backend = backend.write().unwrap();
            // Should not panic or corrupt state
            let _ = backend.create_dir_all("/a/b/c/d");
        })
    }).collect();

    for handle in handles {
        handle.join().unwrap();
    }
}
```

### Path Normalization
```rust
#[test]
fn test_path_with_dotdot() {
    let mut backend = create_backend();
    backend.create_dir_all("/foo/bar").unwrap();
    backend.write("/foo/bar/test.txt", b"data").unwrap();

    // These should all access the same file
    assert_eq!(backend.read("/foo/bar/test.txt").unwrap(), b"data");
    assert_eq!(backend.read("/foo/bar/../bar/test.txt").unwrap(), b"data");
    assert_eq!(backend.read("/foo/./bar/test.txt").unwrap(), b"data");
}
```
