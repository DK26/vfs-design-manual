# AnyFS Implementation Patterns

**Purpose:** Quick reference for implementing backends, middleware, and adapters.

---

## Trait Hierarchy (Pick Your Level)

```
FsPosix  ← Full POSIX (handles, locks, xattr)
    ↑
FsFuse   ← FUSE-mountable (+ inodes)
    ↑
FsFull   ← std::fs features (+ links, permissions, sync, stats)
    ↑
   Fs    ← Basic filesystem (90% of use cases)
    ↑
FsRead + FsWrite + FsDir  ← Core traits
```

**Rule:** Implement the lowest level you need. Higher levels include all below.

---

## Pattern 1: Implement a Backend

> **Thread Safety:** All methods use `&self`. Backends MUST use interior mutability (`RwLock`, `Mutex`) for thread-safe concurrent access.

### Minimum: Implement `Fs` (3 traits)

```rust
use anyfs_backend::{FsRead, FsWrite, FsDir, FsError, Metadata, DirEntry, FileType, Permissions};
use std::path::Path;

pub struct MyBackend { /* your storage */ }

impl FsRead for MyBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        todo!("return file contents")
    }
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, FsError> {
        String::from_utf8(self.read(path)?).map_err(|e| FsError::Backend(e.to_string()))
    }
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, FsError> {
        todo!("return slice of file")
    }
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        todo!("check if path exists")
    }
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        todo!("return Metadata { inode, nlink, file_type, size, permissions, created, modified, accessed }")
    }
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn std::io::Read + Send>, FsError> {
        Ok(Box::new(std::io::Cursor::new(self.read(path)?)))
    }
}

impl FsWrite for MyBackend {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        todo!("create or overwrite file")
    }
    fn append(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        todo!("append to file")
    }
    fn remove_file(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("delete file")
    }
    fn rename(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("move/rename")
    }
    fn copy(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("copy file")
    }
    fn truncate(&self, path: impl AsRef<Path>, size: u64) -> Result<(), FsError> {
        todo!("resize file")
    }
    fn open_write(&self, path: impl AsRef<Path>) -> Result<Box<dyn std::io::Write + Send>, FsError> {
        todo!("return writer")
    }
}

impl FsDir for MyBackend {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<ReadDirIter, FsError> {
        todo!("return ReadDirIter::new(entries.into_iter().map(Ok))")
    }
    fn create_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("create single directory")
    }
    fn create_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("create directory and parents")
    }
    fn remove_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("remove empty directory")
    }
    fn remove_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("remove directory recursively")
    }
}
// MyBackend now implements Fs automatically!
```

### Add Links/Permissions: Implement `FsFull` (add 4 traits)

```rust
impl FsLink for MyBackend {
    fn symlink(&self, target: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("create symlink pointing to target")
    }
    fn hard_link(&self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("create hard link (same inode)")
    }
    fn read_link(&self, path: impl AsRef<Path>) -> Result<std::path::PathBuf, FsError> {
        todo!("return symlink target")
    }
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        todo!("metadata without following symlink")
    }
}

impl FsPermissions for MyBackend {
    fn set_permissions(&self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), FsError> {
        todo!("set file permissions")
    }
}

impl FsSync for MyBackend {
    fn sync(&self) -> Result<(), FsError> {
        todo!("flush all writes to storage")
    }
    fn fsync(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        todo!("flush writes for one file")
    }
}

impl FsStats for MyBackend {
    fn statfs(&self) -> Result<anyfs_backend::StatFs, FsError> {
        todo!("return StatFs { total_bytes, available_bytes, ... }")
    }
}
// MyBackend now implements FsFull!
```

### Add FUSE Support: Implement `FsFuse` (add 1 trait)

```rust
impl FsInode for MyBackend {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, FsError> {
        todo!("return unique inode for path")
    }
    fn inode_to_path(&self, inode: u64) -> Result<std::path::PathBuf, FsError> {
        todo!("return path for inode")
    }
    fn lookup(&self, parent_inode: u64, name: &std::ffi::OsStr) -> Result<u64, FsError> {
        todo!("find child inode by name")
    }
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, FsError> {
        todo!("get metadata by inode (fast path)")
    }
}
// MyBackend now implements FsFuse!
```

---

## Pattern 2: Implement Middleware

**Rule:** Middleware implements the same traits as its inner backend, intercepting some methods.

### Template

```rust
use anyfs_backend::{FsRead, FsWrite, FsDir, FsError, Layer};

pub struct MyMiddleware<B> {
    inner: B,
    // your state
}

impl<B> MyMiddleware<B> {
    pub fn new(inner: B) -> Self {
        Self { inner }
    }
}

// Implement each trait, delegating or intercepting as needed
impl<B: FsRead> FsRead for MyMiddleware<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        // INTERCEPT: add your logic
        let data = self.inner.read(path)?;  // DELEGATE
        // INTERCEPT: modify result
        Ok(data)
    }

    // For passthrough methods, just delegate:
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.inner.exists(path)
    }
    // ... implement all FsRead methods
}

impl<B: FsWrite> FsWrite for MyMiddleware<B> {
    // ... implement all FsWrite methods
}

impl<B: FsDir> FsDir for MyMiddleware<B> {
    // ... implement all FsDir methods
}

// Optional: Layer for .layer() syntax
pub struct MyMiddlewareLayer { /* config */ }

impl<B: Fs> Layer<B> for MyMiddlewareLayer {
    type Backend = MyMiddleware<B>;
    fn layer(self, backend: B) -> Self::Backend {
        MyMiddleware::new(backend)
    }
}
```

### Common Middleware Patterns

| Pattern | Intercept | Delegate | Example |
|---------|-----------|----------|---------|
| **Logging** | All (before/after) | All | `Tracing` |
| **Block writes** | Write methods → return error | Read methods | `ReadOnly` |
| **Transform data** | `read`/`write` | Everything else | `Encryption` |
| **Check permissions** | All (before) | All | `PathFilter` |
| **Enforce limits** | Write methods (check size) | Read methods | `Quota` |
| **Count operations** | All (increment counter) | All | `Counter` |

### Example: ReadOnly Middleware

```rust
impl<B: FsWrite> FsWrite for ReadOnly<B> {
    fn write(&self, _: impl AsRef<Path>, _: &[u8]) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "write" })
    }
    fn remove_file(&self, _: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_file" })
    }
    // ... all write methods return ReadOnly error
}
```

### Example: Encryption Middleware

```rust
impl<B: FsRead> FsRead for Encrypted<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let encrypted = self.inner.read(path)?;
        Ok(self.decrypt(&encrypted))
    }
}

impl<B: FsWrite> FsWrite for Encrypted<B> {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let encrypted = self.encrypt(data);
        self.inner.write(path, &encrypted)
    }
}
```

---

## Pattern 3: Implement an Adapter

### Adapter FROM another crate's trait

```rust
// Wrap external crate's filesystem to use as AnyFS backend
pub struct ExternalCompat<F>(F);

impl<F: external::FileSystem> FsRead for ExternalCompat<F> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        self.0.read_file(path).map_err(|e| FsError::Backend(e.to_string()))
    }
    // Map each method, converting errors
}

impl<F: external::FileSystem> FsWrite for ExternalCompat<F> { /* ... */ }
impl<F: external::FileSystem> FsDir for ExternalCompat<F> { /* ... */ }
// Only implement traits the external crate supports
```

### Adapter TO another crate's trait

```rust
// Wrap AnyFS backend to use with external crate
pub struct AnyFsCompat<B: Fs>(B);

impl<B: Fs> external::FileSystem for AnyFsCompat<B> {
    fn read_file(&self, path: &Path) -> external::Result<Vec<u8>> {
        self.0.read(path).map_err(|e| external::Error::from(e.to_string()))
    }
    // Only expose methods the external trait requires
}
```

---

## Error Handling

**Rule:** Always return `FsError`, never panic.

```rust
// Common error returns
FsError::NotFound { path, operation }      // Path doesn't exist
FsError::AlreadyExists { path, operation } // Path already exists
FsError::NotAFile { path }                 // Expected file, got directory
FsError::NotADirectory { path }            // Expected directory, got file
FsError::DirectoryNotEmpty { path }        // Can't remove non-empty dir
FsError::ReadOnly { operation }            // Write blocked by ReadOnly middleware
FsError::AccessDenied { path, reason }     // Blocked by PathFilter
FsError::QuotaExceeded { limit, requested, usage }
FsError::NotSupported { operation }        // Backend doesn't support this
FsError::Backend(String)                   // Backend-specific error
```

---

## Quick Reference: What to Implement

| You want to... | Implement |
|----------------|-----------|
| Basic file operations | `FsRead` + `FsWrite` + `FsDir` (= `Fs`) |
| + Symlinks/hardlinks | + `FsLink` |
| + Permissions | + `FsPermissions` |
| + Durability (sync) | + `FsSync` |
| + Filesystem stats | + `FsStats` |
| All of above | `Fs` + `FsLink` + `FsPermissions` + `FsSync` + `FsStats` (= `FsFull`) |
| + FUSE mounting | + `FsInode` (= `FsFuse`) |
| + File handles/locks | + `FsHandles` + `FsLock` + `FsXattr` (= `FsPosix`) |

---

## Usage Examples

### Create a backend stack

```rust
use anyfs::{MemoryBackend, QuotaLayer, PathFilterLayer, TracingLayer};

let backend = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .deny("**/.env")
        .build())
    .layer(TracingLayer::new());
```

### With FileStorage wrapper

```rust
use anyfs::FileStorage;

let fs = FileStorage::new(backend);
fs.write("/file.txt", b"hello")?;
let data = fs.read("/file.txt")?;
```

### Type-safe containers

```rust
struct Sandbox;
struct UserData;

// Specify marker in type annotation, infer backend with _
let sandbox: FileStorage<_, Sandbox> = FileStorage::new(MemoryBackend::new());
let userdata: FileStorage<_, UserData> = FileStorage::new(SqliteBackend::open("data.db")?);

fn process(fs: &FileStorage<impl Fs, Sandbox>) { /* only accepts Sandbox */ }
```
