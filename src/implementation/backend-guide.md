# Backend Implementer's Guide

This guide walks you through implementing a custom AnyFS backend.

---

## Overview

AnyFS uses **layered traits** - you implement only what you need:

```
VfsPosix (full POSIX)
   │
VfsFuse (FUSE-mountable)
   │
VfsFull (std::fs features)
   │
  Vfs (basic - 90% of use cases)
   │
VfsRead + VfsWrite + VfsDir (core)
```

Key properties:
- Backends accept `impl AsRef<Path>` for all path parameters
- Backends handle **storage + filesystem semantics only**
- Policy (limits, feature gates) is handled by middleware, not backends
- Implement only the traits your backend supports

---

## Dependency

Depend only on `anyfs-backend`:

```toml
[dependencies]
anyfs-backend = "0.1"
```

---

## Choosing Which Traits to Implement

| Your Backend Supports | Implement |
|----------------------|-----------|
| Basic file operations | `Vfs` (= `VfsRead` + `VfsWrite` + `VfsDir`) |
| Links, permissions, sync | Add `VfsLink`, `VfsPermissions`, `VfsSync`, `VfsStats` |
| Hardlinks, FUSE mounting | Add `VfsInode` → becomes `VfsFuse` |
| Full POSIX (handles, locks, xattr) | Add `VfsHandles`, `VfsLock`, `VfsXattr` → becomes `VfsPosix` |

---

## Minimal Backend: Just `Vfs`

```rust
use anyfs_backend::{VfsRead, VfsWrite, VfsDir, VfsError, Metadata, DirEntry};
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

pub struct MyBackend {
    // Your storage fields
}

// Implement VfsRead
impl VfsRead for MyBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        let path = path.as_ref();
        todo!()
    }

    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError> {
        let data = self.read(path)?;
        String::from_utf8(data).map_err(|e| VfsError::Backend(e.to_string()))
    }

    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError> {
        todo!()
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        todo!()
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError> {
        todo!()
    }

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError> {
        let data = self.read(path)?;
        Ok(Box::new(std::io::Cursor::new(data)))
    }
}

// Implement VfsWrite
impl VfsWrite for MyBackend {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        todo!()
    }

    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        todo!()
    }

    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!()
    }

    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!()
    }

    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!()
    }

    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError> {
        todo!()
    }

    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError> {
        todo!()
    }
}

// Implement VfsDir
impl VfsDir for MyBackend {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError> {
        todo!()
    }

    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!()
    }

    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!()
    }

    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!()
    }

    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        todo!()
    }
}

// MyBackend now implements Vfs automatically (blanket impl)!
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

### Step 2: Implement `VfsRead` (Layer 1)

Start with read operations (easiest):

```rust
impl VfsRead for MyBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, VfsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, VfsError>;
}
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

### Step 3: Implement `VfsWrite` (Layer 1)

```rust
impl VfsWrite for MyBackend {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), VfsError>;
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), VfsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, VfsError>;
}
```

**Note on truncate:**
- If `size < current`: discard trailing bytes
- If `size > current`: extend with zero bytes
- Required for FUSE support and editor save operations

### Step 4: Implement `VfsDir` (Layer 1)

```rust
impl VfsDir for MyBackend {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
}
```

**Congratulations!** After implementing `VfsRead`, `VfsWrite`, and `VfsDir`, your backend implements `Vfs` automatically (blanket impl). This covers 90% of use cases.

---

## Optional: Layer 2 Traits

Add these if your backend supports the features:

### `VfsLink` - Symlinks and Hardlinks

```rust
impl VfsLink for MyBackend {
    fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), VfsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, VfsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError>;
}
```

- Symlinks store a target path as a string
- Hard links share content with the original (update link count)

### `VfsPermissions`

```rust
impl VfsPermissions for MyBackend {
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), VfsError>;
}
```

### `VfsSync` - Durability

```rust
impl VfsSync for MyBackend {
    fn sync(&mut self) -> Result<(), VfsError>;
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError>;
}
```

- `sync()`: Flush all pending writes to durable storage
- `fsync(path)`: Flush pending writes for a specific file
- `MemoryBackend` can no-op these (volatile by design)
- `SqliteBackend`: `PRAGMA wal_checkpoint` or connection flush
- `VRootFsBackend`: `std::fs::File::sync_all()`

### `VfsStats` - Filesystem Stats

```rust
impl VfsStats for MyBackend {
    fn statfs(&self) -> Result<StatFs, VfsError>;
}
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

## Optional: Layer 3 - `VfsInode` (For FUSE)

Implement `VfsInode` if you need FUSE mounting or proper hardlink support:

```rust
impl VfsInode for MyBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError>;
    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, VfsError>;
    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, VfsError>;
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, VfsError>;
}
```

**Default implementations exist** (via blanket impl) - you only need to override for:
- **Hardlink support**: Two paths must share the same inode
- **FUSE efficiency**: Direct inode operations without path lookup

**Level 1: Simple backend (use defaults)**

Don't implement `VfsInode`. Hardlinks won't work correctly, but basic operations will.

**Level 2: Hardlink support**

Override `path_to_inode` so hardlinked paths return the same inode:

```rust
struct Node {
    id: u64,          // Unique node ID (the inode)
    nlink: u64,       // Hard link count
    content: Vec<u8>,
}

struct MemoryBackend {
    next_id: u64,
    nodes: HashMap<u64, Node>,           // inode -> Node
    paths: HashMap<PathBuf, u64>,        // path -> inode
}

impl VfsInode for MemoryBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, VfsError> {
        self.paths.get(path.as_ref())
            .copied()
            .ok_or_else(|| VfsError::NotFound { path: path.as_ref().into() })
    }
    // ... implement others
}

impl VfsLink for MemoryBackend {
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
impl VfsInode for SqliteBackend {
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

| Your Backend | Implement | Result |
|--------------|-----------|--------|
| Simple (no hardlinks) | Nothing | Works with defaults |
| With hardlinks | `VfsInode::path_to_inode` | Hardlinks work correctly |
| FUSE-optimized | Full `VfsInode` | Maximum performance |

---

## Optional: Layer 4 - POSIX Traits

For full POSIX semantics (file handles, locking, extended attributes):

### `VfsHandles` - File Handle Operations

```rust
impl VfsHandles for MyBackend {
    fn open(&mut self, path: impl AsRef<Path>, flags: OpenFlags) -> Result<Handle, VfsError>;
    fn read_at(&self, handle: Handle, buf: &mut [u8], offset: u64) -> Result<usize, VfsError>;
    fn write_at(&mut self, handle: Handle, data: &[u8], offset: u64) -> Result<usize, VfsError>;
    fn close(&mut self, handle: Handle) -> Result<(), VfsError>;
}
```

### `VfsLock` - File Locking

```rust
impl VfsLock for MyBackend {
    fn lock(&mut self, handle: Handle, lock: LockType) -> Result<(), VfsError>;
    fn try_lock(&mut self, handle: Handle, lock: LockType) -> Result<bool, VfsError>;
    fn unlock(&mut self, handle: Handle) -> Result<(), VfsError>;
}
```

### `VfsXattr` - Extended Attributes

```rust
impl VfsXattr for MyBackend {
    fn get_xattr(&self, path: impl AsRef<Path>, name: &str) -> Result<Vec<u8>, VfsError>;
    fn set_xattr(&mut self, path: impl AsRef<Path>, name: &str, value: &[u8]) -> Result<(), VfsError>;
    fn remove_xattr(&mut self, path: impl AsRef<Path>, name: &str) -> Result<(), VfsError>;
    fn list_xattr(&self, path: impl AsRef<Path>) -> Result<Vec<String>, VfsError>;
}
```

**Note:** Most backends don't need Layer 4. Only implement if you're wrapping a real filesystem (`VRootFsBackend`) or building a database that needs full POSIX semantics.

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
| Ergonomic API | `FileStorage<M>` wrapper |

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

### The Pattern (5 Minutes to Understand)

Middleware is just a struct that:
1. Wraps another `VfsBackend`
2. Implements `VfsBackend` itself
3. Intercepts some methods, delegates others

```rust
//  ┌─────────────────────────────────────┐
//  │  Your Middleware                    │
//  │  ┌─────────────────────────────────┐│
//  │  │  Inner Backend (any VfsBackend) ││
//  │  └─────────────────────────────────┘│
//  └─────────────────────────────────────┘
//
//  Request → Middleware (intercept/modify) → Inner Backend
//  Response ← Middleware (intercept/modify) ← Inner Backend
```

### Simplest Possible Middleware: Operation Counter

This middleware counts how many operations are performed:

```rust
use anyfs_backend::{VfsBackend, VfsError, Metadata, DirEntry, Permissions, StatFs};
use std::sync::atomic::{AtomicU64, Ordering};
use std::path::{Path, PathBuf};

/// Counts all operations performed on the backend.
pub struct Counter<B> {
    inner: B,
    pub count: AtomicU64,
}

impl<B> Counter<B> {
    pub fn new(inner: B) -> Self {
        Self { inner, count: AtomicU64::new(0) }
    }

    pub fn operations(&self) -> u64 {
        self.count.load(Ordering::Relaxed)
    }
}

// Implement each trait the inner backend supports
impl<B: VfsRead> VfsRead for Counter<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        self.count.fetch_add(1, Ordering::Relaxed);  // Count it
        self.inner.read(path)                         // Delegate
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.exists(path)
    }

    // ... repeat for all VfsRead methods
}

impl<B: VfsWrite> VfsWrite for Counter<B> {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        self.count.fetch_add(1, Ordering::Relaxed);  // Count it
        self.inner.write(path, data)                  // Delegate
    }

    // ... repeat for all VfsWrite methods
}

impl<B: VfsDir> VfsDir for Counter<B> {
    // ... implement VfsDir methods
}

// Counter<B> now implements Vfs when B: Vfs (blanket impl)
```

**Usage:**

```rust
let backend = Counter::new(MemoryBackend::new());
backend.write("/file.txt", b"hello")?;
backend.read("/file.txt")?;
backend.read("/file.txt")?;

println!("Operations: {}", backend.operations());  // 3
```

That's it. That's the entire pattern.

### Adding a Layer (for .layer() syntax)

To enable the fluent `.layer()` syntax, add a Layer struct:

```rust
use anyfs_backend::Layer;

pub struct CounterLayer;

impl<B: VfsBackend> Layer<B> for CounterLayer {
    type Backend = Counter<B>;

    fn layer(self, backend: B) -> Counter<B> {
        Counter::new(backend)
    }
}
```

**Usage with .layer():**

```rust
let backend = MemoryBackend::new()
    .layer(CounterLayer);
```

### Real Example: ReadOnly Middleware

A practical middleware that blocks all write operations:

```rust
pub struct ReadOnly<B> {
    inner: B,
}

impl<B> ReadOnly<B> {
    pub fn new(inner: B) -> Self {
        Self { inner }
    }
}

// VfsRead: just delegate
impl<B: VfsRead> VfsRead for ReadOnly<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        self.inner.read(path)
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        self.inner.exists(path)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError> {
        self.inner.metadata(path)
    }

    // ... delegate all VfsRead methods
}

// VfsDir: delegate reads, block writes
impl<B: VfsDir> VfsDir for ReadOnly<B> {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError> {
        self.inner.read_dir(path)
    }

    fn create_dir(&mut self, _path: impl AsRef<Path>) -> Result<(), VfsError> {
        Err(VfsError::ReadOnly { operation: "create_dir" })
    }

    fn create_dir_all(&mut self, _path: impl AsRef<Path>) -> Result<(), VfsError> {
        Err(VfsError::ReadOnly { operation: "create_dir_all" })
    }

    fn remove_dir(&mut self, _path: impl AsRef<Path>) -> Result<(), VfsError> {
        Err(VfsError::ReadOnly { operation: "remove_dir" })
    }

    fn remove_dir_all(&mut self, _path: impl AsRef<Path>) -> Result<(), VfsError> {
        Err(VfsError::ReadOnly { operation: "remove_dir_all" })
    }
}

// VfsWrite: block all operations
impl<B: VfsWrite> VfsWrite for ReadOnly<B> {
    fn write(&mut self, _path: impl AsRef<Path>, _data: &[u8]) -> Result<(), VfsError> {
        Err(VfsError::ReadOnly { operation: "write" })
    }

    fn remove_file(&mut self, _path: impl AsRef<Path>) -> Result<(), VfsError> {
        Err(VfsError::ReadOnly { operation: "remove_file" })
    }

    // ... block all VfsWrite methods
}
```

**Usage:**

```rust
let mut backend = ReadOnly::new(MemoryBackend::new());

backend.read("/file.txt");       // OK (if file exists)
backend.write("/file.txt", b""); // Error: ReadOnly
```

### Middleware Decision Table

| What You Want | Intercept | Delegate | Example |
|---------------|-----------|----------|---------|
| Count operations | All methods (before) | All methods | `Counter` |
| Block writes | Write methods | Read methods | `ReadOnly` |
| Transform data | `read`/`write` | Everything else | `Encryption` |
| Check permissions | All methods (before) | All methods | `PathFilter` |
| Log operations | All methods (before) | All methods | `Tracing` |
| Enforce limits | Write methods (check size) | Read methods | `Quota` |

### Macro for Boilerplate (Optional)

If you don't want to manually delegate all 29 methods, you can use a macro:

```rust
macro_rules! delegate {
    ($self:ident, $method:ident, $($arg:ident),*) => {
        $self.inner.$method($($arg),*)
    };
}

impl<B: VfsBackend> VfsBackend for MyMiddleware<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        // Your logic here
        delegate!(self, read, path)
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        delegate!(self, exists, path)
    }

    // ... etc
}
```

Or provide a `delegate_all!` macro in `anyfs-backend` that generates all the passthrough implementations.

### Complete Example: Encryption Middleware

```rust
use anyfs_backend::{VfsRead, VfsWrite, VfsDir, Layer, VfsError, Metadata, DirEntry};
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

/// Middleware that encrypts/decrypts file contents transparently.
pub struct Encrypted<B> {
    inner: B,
    key: [u8; 32],
}

impl<B> Encrypted<B> {
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

// VfsRead: decrypt on read
impl<B: VfsRead> VfsRead for Encrypted<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError> {
        let encrypted = self.inner.read(path)?;
        Ok(self.decrypt(&encrypted))
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, VfsError> {
        self.inner.exists(path)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, VfsError> {
        self.inner.metadata(path)
    }

    // ... delegate other VfsRead methods
}

// VfsWrite: encrypt on write
impl<B: VfsWrite> VfsWrite for Encrypted<B> {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), VfsError> {
        let encrypted = self.encrypt(data);
        self.inner.write(path, &encrypted)
    }

    // ... delegate/encrypt other VfsWrite methods
}

// VfsDir: just delegate (directories don't need encryption)
impl<B: VfsDir> VfsDir for Encrypted<B> {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, VfsError> {
        self.inner.read_dir(path)
    }

    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), VfsError> {
        self.inner.create_dir(path)
    }

    // ... delegate other VfsDir methods
}

// Encrypted<B> now implements Vfs when B: Vfs (blanket impl)

/// Layer for creating Encrypted middleware.
pub struct EncryptedLayer {
    key: [u8; 32],
}

impl EncryptedLayer {
    pub fn new(key: [u8; 32]) -> Self {
        Self { key }
    }
}

impl<B: Vfs> Layer<B> for EncryptedLayer {
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
- [ ] Implements the same traits as the inner backend (`VfsRead`, `VfsWrite`, `VfsDir`, etc.)
- [ ] Implements `Layer<B>` for `MyMiddlewareLayer`
- [ ] Delegates unmodified operations to inner backend
- [ ] Handles streaming I/O appropriately (wrap, pass-through, or block)
- [ ] Documents which operations are intercepted vs delegated

---

## Backend Checklist

- [ ] Depends only on `anyfs-backend`
- [ ] Implements core traits: `VfsRead`, `VfsWrite`, `VfsDir` (= `Vfs`)
- [ ] Optional: Implements `VfsLink`, `VfsPermissions`, `VfsSync`, `VfsStats` (= `VfsFull`)
- [ ] Optional: Implements `VfsInode` for FUSE support (= `VfsFuse`)
- [ ] Optional: Implements `VfsHandles`, `VfsLock`, `VfsXattr` for POSIX (= `VfsPosix`)
- [ ] Accepts `impl AsRef<Path>` for all paths
- [ ] Returns correct `VfsError` variants
- [ ] Passes conformance tests for implemented traits
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
