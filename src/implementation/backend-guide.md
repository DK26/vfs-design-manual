# Backend Implementer's Guide

This guide walks you through implementing a custom AnyFS backend.

---

## Overview

AnyFS uses **layered traits** - you implement only what you need:

```
FsPosix (full POSIX)
   │
FsFuse (FUSE-mountable)
   │
FsFull (std::fs features)
   │
   Fs (basic - 90% of use cases)
   │
FsRead + FsWrite + FsDir (core)
```

Key properties:
- Backends accept `impl AsRef<Path>` for all path parameters
- **Backends receive already-resolved paths** - FileStorage handles path resolution (symlinks, `..`, normalization)
- Backends handle **storage only** - just store/retrieve bytes at given paths
- Policy (limits, feature gates) is handled by middleware, not backends
- Implement only the traits your backend supports
- **Backends must be thread-safe** - all trait methods use `&self`, so backends must use interior mutability (e.g., `RwLock`, `Mutex`) for synchronization

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
| Basic file operations | `Fs` (= `FsRead` + `FsWrite` + `FsDir`) |
| Links, permissions, sync | Add `FsLink`, `FsPermissions`, `FsSync`, `FsStats` |
| Hardlinks, FUSE mounting | Add `FsInode` → becomes `FsFuse` |
| Full POSIX (handles, locks, xattr) | Add `FsHandles`, `FsLock`, `FsXattr` → becomes `FsPosix` |

---

## Minimal Backend: Just `Fs`

```rust
use anyfs_backend::{FsRead, FsWrite, FsDir, FsError, Metadata, DirEntry};
use std::io::{Read, Write};
use std::path::{Path, PathBuf};

pub struct MyBackend {
    // Your storage fields
}

// Implement FsRead
impl FsRead for MyBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();
        todo!()
    }

    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, FsError> {
        let data = self.read(path)?;
        String::from_utf8(data).map_err(|e| FsError::Backend(e.to_string()))
    }

    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, FsError> {
        todo!()
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        todo!()
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        todo!()
    }

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError> {
        let data = self.read(path)?;
        Ok(Box::new(std::io::Cursor::new(data)))
    }
}

// Implement FsWrite
impl FsWrite for MyBackend {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        todo!()
    }

    fn append(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        todo!()
    }

    fn remove_file(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!()
    }

    fn rename(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError> {
        todo!()
    }

    fn copy(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError> {
        todo!()
    }

    fn truncate(&self, path: impl AsRef<Path>, size: u64) -> Result<(), FsError> {
        todo!()
    }

    fn open_write(&self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, FsError> {
        todo!()
    }
}

// Implement FsDir
impl FsDir for MyBackend {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, FsError> {
        todo!()
    }

    fn create_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!()
    }

    fn create_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!()
    }

    fn remove_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!()
    }

    fn remove_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!()
    }
}

// MyBackend now implements Fs automatically (blanket impl)!
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

### Step 2: Implement `FsRead` (Layer 1)

Start with read operations (easiest):

```rust
impl FsRead for MyBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, FsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, FsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError>;
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError>;
}
```

**Streaming implementation options:**

For `MemoryBackend` or similar, you can use `std::io::Cursor`:

```rust
fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError> {
    let data = self.read(path)?;
    Ok(Box::new(std::io::Cursor::new(data)))
}
```

For `VRootFsBackend`, return the actual file handle:

```rust
fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError> {
    let file = std::fs::File::open(self.resolve(path)?)?;
    Ok(Box::new(file))
}
```

### Step 3: Implement `FsWrite` (Layer 1)

```rust
impl FsWrite for MyBackend {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn append(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn remove_file(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn rename(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn copy(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn truncate(&self, path: impl AsRef<Path>, size: u64) -> Result<(), FsError>;
    fn open_write(&self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, FsError>;
}
```

**Note on truncate:**
- If `size < current`: discard trailing bytes
- If `size > current`: extend with zero bytes
- Required for FUSE support and editor save operations

### Step 4: Implement `FsDir` (Layer 1)

```rust
impl FsDir for MyBackend {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, FsError>;
    fn create_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn create_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
}
```

**Congratulations!** After implementing `FsRead`, `FsWrite`, and `FsDir`, your backend implements `Fs` automatically (blanket impl). This covers 90% of use cases.

---

## Optional: Layer 2 Traits

Add these if your backend supports the features:

### `FsLink` - Symlinks and Hardlinks

```rust
impl FsLink for MyBackend {
    fn symlink(&self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError>;
    fn hard_link(&self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, FsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError>;
}
```

- Symlinks store a target path as a string
- Hard links share content with the original (update link count)

### `FsPermissions`

```rust
impl FsPermissions for MyBackend {
    fn set_permissions(&self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), FsError>;
}
```

### `FsSync` - Durability

```rust
impl FsSync for MyBackend {
    fn sync(&self) -> Result<(), FsError>;
    fn fsync(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
}
```

- `sync()`: Flush all pending writes to durable storage
- `fsync(path)`: Flush pending writes for a specific file
- `MemoryBackend` can no-op these (volatile by design)
- `SqliteBackend`: `PRAGMA wal_checkpoint` or connection flush
- `VRootFsBackend`: `std::fs::File::sync_all()`

### `FsStats` - Filesystem Stats

```rust
impl FsStats for MyBackend {
    fn statfs(&self) -> Result<StatFs, FsError>;
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

## Optional: Layer 3 - `FsInode` (For FUSE)

Implement `FsInode` if you need FUSE mounting or proper hardlink support:

```rust
impl FsInode for MyBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, FsError>;
    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, FsError>;
    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, FsError>;
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, FsError>;
}
```

**Default implementations exist** (via blanket impl) - you only need to override for:
- **Hardlink support**: Two paths must share the same inode
- **FUSE efficiency**: Direct inode operations without path lookup

**Level 1: Simple backend (use defaults)**

Don't implement `FsInode`. Hardlinks won't work correctly, but basic operations will.

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

impl FsInode for MemoryBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, FsError> {
        self.paths.get(path.as_ref())
            .copied()
            .ok_or_else(|| FsError::NotFound { path: path.as_ref().into() })
    }
    // ... implement others
}

impl FsLink for MemoryBackend {
    fn hard_link(&self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError> {
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
impl FsInode for SqliteBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, FsError> {
        self.conn.query_row(
            "SELECT id FROM nodes WHERE path = ?",
            [path.as_ref().to_string_lossy()],
            |row| Ok(row.get::<_, i64>(0)? as u64),
        ).map_err(|_| FsError::NotFound { path: path.as_ref().into() })
    }

    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, FsError> {
        self.conn.query_row(
            "SELECT path FROM nodes WHERE id = ?",
            [inode as i64],
            |row| Ok(PathBuf::from(row.get::<_, String>(0)?)),
        ).map_err(|_| FsError::NotFound { path: format!("inode:{}", inode).into() })
    }

    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, FsError> {
        self.conn.query_row(
            "SELECT id FROM nodes WHERE parent_id = ? AND name = ?",
            params![parent_inode as i64, name.to_string_lossy()],
            |row| Ok(row.get::<_, i64>(0)? as u64),
        ).map_err(|_| FsError::NotFound { path: name.into() })
    }

    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, FsError> {
        self.conn.query_row(
            "SELECT type, size, nlink, created, modified FROM nodes WHERE id = ?",
            [inode as i64],
            |row| Ok(Metadata {
                inode,
                nlink: row.get(2)?,
                // ...
            }),
        ).map_err(|_| FsError::NotFound { path: format!("inode:{}", inode).into() })
    }
}
```

**Summary:**

| Your Backend | Implement | Result |
|--------------|-----------|--------|
| Simple (no hardlinks) | Nothing | Works with defaults |
| With hardlinks | `FsInode::path_to_inode` | Hardlinks work correctly |
| FUSE-optimized | Full `FsInode` | Maximum performance |

---

## Optional: Layer 4 - POSIX Traits

For full POSIX semantics (file handles, locking, extended attributes):

### `FsHandles` - File Handle Operations

```rust
impl FsHandles for MyBackend {
    fn open(&self, path: impl AsRef<Path>, flags: OpenFlags) -> Result<Handle, FsError>;
    fn read_at(&self, handle: Handle, buf: &mut [u8], offset: u64) -> Result<usize, FsError>;
    fn write_at(&self, handle: Handle, data: &[u8], offset: u64) -> Result<usize, FsError>;
    fn close(&self, handle: Handle) -> Result<(), FsError>;
}
```

### `FsLock` - File Locking

```rust
impl FsLock for MyBackend {
    fn lock(&self, handle: Handle, lock: LockType) -> Result<(), FsError>;
    fn try_lock(&self, handle: Handle, lock: LockType) -> Result<bool, FsError>;
    fn unlock(&self, handle: Handle) -> Result<(), FsError>;
}
```

### `FsXattr` - Extended Attributes

```rust
impl FsXattr for MyBackend {
    fn get_xattr(&self, path: impl AsRef<Path>, name: &str) -> Result<Vec<u8>, FsError>;
    fn set_xattr(&self, path: impl AsRef<Path>, name: &str, value: &[u8]) -> Result<(), FsError>;
    fn remove_xattr(&self, path: impl AsRef<Path>, name: &str) -> Result<(), FsError>;
    fn list_xattr(&self, path: impl AsRef<Path>) -> Result<Vec<String>, FsError>;
}
```

**Note:** Most backends don't need Layer 4. Only implement if you're wrapping a real filesystem (`VRootFsBackend`) or building a database that needs full POSIX semantics.

---

## Error Handling

Return appropriate `FsError` variants:

| Situation | Error |
|-----------|-------|
| Path doesn't exist | `FsError::NotFound { path, operation }` |
| Path already exists | `FsError::AlreadyExists { path, operation }` |
| Expected file, got dir | `FsError::NotAFile { path }` |
| Expected dir, got file | `FsError::NotADirectory { path }` |
| Remove non-empty dir | `FsError::DirectoryNotEmpty { path }` |
| Internal error | `FsError::Backend { message }` |

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
    use anyfs_backend::Fs;

    fn create_backend() -> MyBackend {
        MyBackend::new()
    }

    #[test]
    fn test_write_read() {
        let backend = create_backend();
        backend.write("/test.txt", b"hello").unwrap();
        let content = backend.read("/test.txt").unwrap();
        assert_eq!(content, b"hello");
    }

    #[test]
    fn test_create_dir() {
        let backend = create_backend();
        backend.create_dir("/foo").unwrap();
        assert!(backend.exists("/foo").unwrap());
    }

    // ... more tests
}
```

---

## Note on VRootFsBackend

If you are implementing a backend that wraps a **real host filesystem directory**, consider using `strict-path::VirtualPath` and `strict-path::VirtualRoot` internally for path containment. This ensures paths cannot escape the designated root directory.

This is an implementation choice for filesystem-based backends, not a requirement of the `Fs` trait.

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
impl<B: Fs> Fs for Quota<B> {
    fn open_write(&self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, FsError> {
        // Check if we're at quota before opening
        if self.usage.total_bytes >= self.limits.max_total_size {
            return Err(FsError::QuotaExceeded { ... });
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
1. Wraps another `Fs`
2. Implements `Fs` itself
3. Intercepts some methods, delegates others

```rust
//  ┌─────────────────────────────────────┐
//  │  Your Middleware                    │
//  │  ┌─────────────────────────────────┐│
//  │  │  Inner Backend (any Fs) ││
//  │  └─────────────────────────────────┘│
//  └─────────────────────────────────────┘
//
//  Request → Middleware (intercept/modify) → Inner Backend
//  Response ← Middleware (intercept/modify) ← Inner Backend
```

### Simplest Possible Middleware: Operation Counter

This middleware counts how many operations are performed:

```rust
use anyfs_backend::{Fs, FsError, Metadata, DirEntry, Permissions, StatFs};
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
impl<B: FsRead> FsRead for Counter<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);  // Count it
        self.inner.read(path)                         // Delegate
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.exists(path)
    }

    // ... repeat for all FsRead methods
}

impl<B: FsWrite> FsWrite for Counter<B> {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);  // Count it
        self.inner.write(path, data)                  // Delegate
    }

    // ... repeat for all FsWrite methods
}

impl<B: FsDir> FsDir for Counter<B> {
    // ... implement FsDir methods
}

// Counter<B> now implements Fs when B: Fs (blanket impl)
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

impl<B: Fs> Layer<B> for CounterLayer {
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

// FsRead: just delegate
impl<B: FsRead> FsRead for ReadOnly<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        self.inner.read(path)
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.inner.exists(path)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        self.inner.metadata(path)
    }

    // ... delegate all FsRead methods
}

// FsDir: delegate reads, block writes
impl<B: FsDir> FsDir for ReadOnly<B> {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, FsError> {
        self.inner.read_dir(path)
    }

    fn create_dir(&self, _path: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "create_dir" })
    }

    fn create_dir_all(&self, _path: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "create_dir_all" })
    }

    fn remove_dir(&self, _path: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_dir" })
    }

    fn remove_dir_all(&self, _path: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_dir_all" })
    }
}

// FsWrite: block all operations
impl<B: FsWrite> FsWrite for ReadOnly<B> {
    fn write(&self, _path: impl AsRef<Path>, _data: &[u8]) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "write" })
    }

    fn remove_file(&self, _path: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_file" })
    }

    // ... block all FsWrite methods
}
```

**Usage:**

```rust
let backend = ReadOnly::new(MemoryBackend::new());

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

impl<B: Fs> Fs for MyMiddleware<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        // Your logic here
        delegate!(self, read, path)
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        delegate!(self, exists, path)
    }

    // ... etc
}
```

Or provide a `delegate_all!` macro in `anyfs-backend` that generates all the passthrough implementations.

### Complete Example: Encryption Middleware

```rust
use anyfs_backend::{FsRead, FsWrite, FsDir, Layer, FsError, Metadata, DirEntry};
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

// FsRead: decrypt on read
impl<B: FsRead> FsRead for Encrypted<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let encrypted = self.inner.read(path)?;
        Ok(self.decrypt(&encrypted))
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.inner.exists(path)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        self.inner.metadata(path)
    }

    // ... delegate other FsRead methods
}

// FsWrite: encrypt on write
impl<B: FsWrite> FsWrite for Encrypted<B> {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let encrypted = self.encrypt(data);
        self.inner.write(path, &encrypted)
    }

    // ... delegate/encrypt other FsWrite methods
}

// FsDir: just delegate (directories don't need encryption)
impl<B: FsDir> FsDir for Encrypted<B> {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, FsError> {
        self.inner.read_dir(path)
    }

    fn create_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        self.inner.create_dir(path)
    }

    // ... delegate other FsDir methods
}

// Encrypted<B> now implements Fs when B: Fs (blanket impl)

/// Layer for creating Encrypted middleware.
pub struct EncryptedLayer {
    key: [u8; 32],
}

impl EncryptedLayer {
    pub fn new(key: [u8; 32]) -> Self {
        Self { key }
    }
}

impl<B: Fs> Layer<B> for EncryptedLayer {
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
- [ ] Implements the same traits as the inner backend (`FsRead`, `FsWrite`, `FsDir`, etc.)
- [ ] Implements `Layer<B>` for `MyMiddlewareLayer`
- [ ] Delegates unmodified operations to inner backend
- [ ] Handles streaming I/O appropriately (wrap, pass-through, or block)
- [ ] Documents which operations are intercepted vs delegated

---

## Backend Checklist

- [ ] Depends only on `anyfs-backend`
- [ ] Implements core traits: `FsRead`, `FsWrite`, `FsDir` (= `Fs`)
- [ ] Optional: Implements `FsLink`, `FsPermissions`, `FsSync`, `FsStats` (= `FsFull`)
- [ ] Optional: Implements `FsInode` for FUSE support (= `FsFuse`)
- [ ] Optional: Implements `FsHandles`, `FsLock`, `FsXattr` for POSIX (= `FsPosix`)
- [ ] Accepts `impl AsRef<Path>` for all paths
- [ ] Returns correct `FsError` variants
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
fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
    let entry = self.entries.get(path.as_ref()).unwrap();  // PANIC!
    Ok(entry.content.clone())
}

// GOOD - returns error
fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
    let path = path.as_ref();
    let entry = self.entries.get(path)
        .ok_or_else(|| FsError::NotFound { path: path.to_path_buf() })?;
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

### 2. Thread Safety (Required)

**All trait methods use `&self`, not `&mut self`.** This means backends MUST use interior mutability for thread-safe concurrent access.

**Why `&self`?**
- Enables concurrent access patterns (multiple readers, concurrent operations)
- Matches real filesystem semantics (concurrent access is normal)
- More flexible API (can share references without exclusive ownership)

**Backend implementer responsibility:**
- Use `RwLock`, `Mutex`, or similar for internal state
- Ensure operations are atomic (a single `write()` call shouldn't produce partial results)
- Handle lock poisoning gracefully

**What the synchronization guarantees:**
- Memory safety (no data corruption)
- Atomic operations (writes don't interleave)

**What it does NOT guarantee:**
- Order of concurrent writes to the same path (last write wins - standard FS behavior)

```rust
use std::sync::{Arc, RwLock};
use std::collections::HashMap;
use std::path::PathBuf;

pub struct MemoryBackend {
    entries: Arc<RwLock<HashMap<PathBuf, Entry>>>,
}

impl FsRead for MemoryBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let entries = self.entries.read()
            .map_err(|_| FsError::Backend("lock poisoned".into()))?;
        // ...
    }
}

impl FsWrite for MemoryBackend {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let mut entries = self.entries.write()
            .map_err(|_| FsError::Backend("lock poisoned".into()))?;
        // ...
    }
}
```

**Common race conditions to avoid:**
- `create_dir_all` called concurrently for same path
- `read` during `write` to same file
- `read_dir` while directory is being modified
- `rename` with concurrent access to source or destination

### 3. Path Resolution - NOT Your Job

**Backends do NOT handle path resolution.** FileStorage handles:
- Resolving `..` and `.` components
- Following symlinks (when `set_follow_symlinks(true)`)
- Normalizing paths (`//` → `/`, trailing slashes, etc.)
- Walking the virtual directory structure

Your backend receives **already-resolved, clean paths**. Just store and retrieve bytes at those paths.

```rust
impl FsRead for MyBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        // Path is already resolved - just use it directly
        let path = path.as_ref();
        self.storage.get(path).ok_or_else(|| FsError::NotFound { path: path.to_path_buf() })
    }
}
```

**Exception:** If your backend wraps a real filesystem (like `VRootFsBackend`), implement `SelfResolving` to tell FileStorage to skip resolution - the OS handles it.

```rust
impl SelfResolving for VRootFsBackend {}
```

### 4. Error Messages

Include context in errors for debugging:

```rust
// BAD - no context
Err(FsError::NotFound)

// GOOD - includes path
Err(FsError::NotFound { path: path.to_path_buf() })

// BETTER - includes operation context
Err(FsError::Io {
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
        Err(FsError::NotFound { .. })
    ));
}

#[test]
fn test_read_dir_on_file_returns_error() {
    let backend = create_backend();
    backend.write("/file.txt", b"data").unwrap();
    assert!(matches!(
        backend.read_dir("/file.txt"),
        Err(FsError::NotADirectory { .. })
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
    let backend = create_backend();
    backend.create_dir_all("/foo/bar").unwrap();
    backend.write("/foo/bar/test.txt", b"data").unwrap();

    // These should all access the same file
    assert_eq!(backend.read("/foo/bar/test.txt").unwrap(), b"data");
    assert_eq!(backend.read("/foo/bar/../bar/test.txt").unwrap(), b"data");
    assert_eq!(backend.read("/foo/./bar/test.txt").unwrap(), b"data");
}
```

---

## MemoryBackend Snapshot & Restore

`MemoryBackend` supports cloning its entire state (snapshot) and serializing to bytes for persistence.

### Core Concept

**Snapshot = Clone the storage.** That's it.

```rust
// MemoryBackend implements Clone
#[derive(Clone)]
pub struct MemoryBackend { ... }

// Snapshot is just .clone()
let snapshot = fs.clone();

// Restore is just assignment
fs = snapshot;
```

### API

```rust
impl MemoryBackend {
    /// Clone the entire filesystem state.
    /// This is a deep copy - modifications to the clone don't affect the original.
    pub fn clone(&self) -> Self { ... }  // via #[derive(Clone)]

    /// Serialize to bytes for persistence/transfer.
    pub fn to_bytes(&self) -> Result<Vec<u8>, FsError>;

    /// Deserialize from bytes.
    pub fn from_bytes(data: &[u8]) -> Result<Self, FsError>;

    /// Save to file.
    pub fn save_to(&self, path: impl AsRef<Path>) -> Result<(), FsError>;

    /// Load from file.
    pub fn load_from(path: impl AsRef<Path>) -> Result<Self, FsError>;
}
```

### Usage

```rust
let fs = MemoryBackend::new();
fs.write("/data.txt", b"important")?;

// Snapshot = clone
let checkpoint = fs.clone();

// Do risky work...
fs.write("/data.txt", b"corrupted")?;

// Rollback = replace with clone
fs = checkpoint;
assert_eq!(fs.read("/data.txt")?, b"important");
```

### Persistence

```rust
// Save to disk
fs.save_to("state.bin")?;

// Load from disk
let fs = MemoryBackend::load_from("state.bin")?;
```

### SqliteBackend

SQLite already has persistence - the database file IS the snapshot. For explicit snapshots:

```rust
impl SqliteBackend {
    /// Create an in-memory copy of the database.
    pub fn clone_to_memory(&self) -> Result<Self, FsError>;

    /// Backup to another file.
    pub fn backup_to(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
}
```
