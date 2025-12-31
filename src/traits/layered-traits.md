# Layered Traits (anyfs-backend)

AnyFS uses a **layered trait architecture** for maximum flexibility with minimal complexity.

See ADR-030 for the design rationale.

---

## Trait Hierarchy

```
                    FsPosix
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   FsHandles       FsLock        FsXattr
        │              │              │
        └──────────────┼──────────────┘
                       │
                    FsFuse
                       │
                   FsInode
                       │
                    FsFull
                       │
        ┌──────┬───────┼───────┬──────┐
        │      │       │       │      │
   FsLink   FsPerm  FsSync  FsStats   │
        │      │       │       │      │
        └──────┴───────┼───────┴──────┘
                       │
                      Fs   ← Most users only need this
                       │
           ┌───────────┼───────────┐
           │           │           │
        FsRead      FsWrite     FsDir

                                              Derived Traits (auto-impl)
                                              ───────────────────────────
                                              FsPath: FsRead + FsLink
                                                (path canonicalization)
```

**Simple rule:** Import `Fs` for basic use. Add traits as needed for advanced features.

**Note:** `FsPath` is a derived trait with a blanket impl. Any type implementing `FsRead + FsLink` automatically gets `FsPath`. `SelfResolving` is a marker trait that opts out of `FileStorage` path resolution.

---

## Layer 1: Core Traits (Required)

> **Thread Safety:** All traits require `Send + Sync` and use `&self` for all methods. Backend implementers MUST use interior mutability (`RwLock`, `Mutex`, etc.) to ensure thread-safe concurrent access. See ADR-023 for rationale.
>
> **Path Parameters:** Core traits use `&Path` so they are object-safe (`dyn Fs` works). For ergonomics, `FileStorage` and `FsExt` accept `impl AsRef<Path>` and forward to the core traits.

### FsRead

```rust
pub trait FsRead: Send + Sync {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError>;
    fn read_to_string(&self, path: &Path) -> Result<String, FsError>;
    fn read_range(&self, path: &Path, offset: u64, len: usize) -> Result<Vec<u8>, FsError>;
    fn exists(&self, path: &Path) -> Result<bool, FsError>;
    fn metadata(&self, path: &Path) -> Result<Metadata, FsError>;
    fn open_read(&self, path: &Path) -> Result<Box<dyn Read + Send>, FsError>;
}
```

### FsWrite

```rust
pub trait FsWrite: Send + Sync {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError>;
    fn append(&self, path: &Path, data: &[u8]) -> Result<(), FsError>;
    fn remove_file(&self, path: &Path) -> Result<(), FsError>;
    fn rename(&self, from: &Path, to: &Path) -> Result<(), FsError>;
    fn copy(&self, from: &Path, to: &Path) -> Result<(), FsError>;
    fn truncate(&self, path: &Path, size: u64) -> Result<(), FsError>;
    fn open_write(&self, path: &Path) -> Result<Box<dyn Write + Send>, FsError>;
}
```

### FsDir

```rust
pub trait FsDir: Send + Sync {
    fn read_dir(&self, path: &Path) -> Result<ReadDirIter, FsError>;
    fn create_dir(&self, path: &Path) -> Result<(), FsError>;
    fn create_dir_all(&self, path: &Path) -> Result<(), FsError>;
    fn remove_dir(&self, path: &Path) -> Result<(), FsError>;
    fn remove_dir_all(&self, path: &Path) -> Result<(), FsError>;
}

/// Iterator over directory entries. Wraps a boxed iterator for flexibility.
///
/// - Outer `Result` (from `read_dir()`) = "can I open this directory?"
/// - Inner `Result` (per item) = "can I read this entry?"
pub struct ReadDirIter(Box<dyn Iterator<Item = Result<DirEntry, FsError>> + Send + 'static>);

impl Iterator for ReadDirIter {
    type Item = Result<DirEntry, FsError>;
    fn next(&mut self) -> Option<Self::Item> { self.0.next() }
}

impl ReadDirIter {
    pub fn new(iter: impl Iterator<Item = Result<DirEntry, FsError>> + Send + 'static) -> Self {
        Self(Box::new(iter))
    }

    /// Collect all entries, short-circuiting on first error.
    pub fn collect_all(self) -> Result<Vec<DirEntry>, FsError> {
        self.collect()
    }
}
```

---

## Layer 2: Extended Traits (Optional)

### FsLink

```rust
pub trait FsLink: Send + Sync {
    fn symlink(&self, target: &Path, link: &Path) -> Result<(), FsError>;
    fn hard_link(&self, original: &Path, link: &Path) -> Result<(), FsError>;
    fn read_link(&self, path: &Path) -> Result<PathBuf, FsError>;
    fn symlink_metadata(&self, path: &Path) -> Result<Metadata, FsError>;
}
```

### FsPermissions

```rust
pub trait FsPermissions: Send + Sync {
    fn set_permissions(&self, path: &Path, perm: Permissions) -> Result<(), FsError>;
}
```

### FsSync

```rust
pub trait FsSync: Send + Sync {
    fn sync(&self) -> Result<(), FsError>;
    fn fsync(&self, path: &Path) -> Result<(), FsError>;
}
```

### FsStats

```rust
pub trait FsStats: Send + Sync {
    fn statfs(&self) -> Result<StatFs, FsError>;
}
```

### FsPath (Optimizable)

Path canonicalization with a default implementation. Backends can override for optimized resolution.

```rust
pub trait FsPath: FsRead + FsLink {
    /// Resolve all symlinks and normalize path (.., .).
    /// Default: iterative resolution via read_link() and symlink_metadata().
    fn canonicalize(&self, path: &Path) -> Result<PathBuf, FsError> {
        // ... default impl ...
    }

    /// Like canonicalize, but allows non-existent final component.
    fn soft_canonicalize(&self, path: &Path) -> Result<PathBuf, FsError> {
        // ... default impl ...
    }
}
impl<T: FsRead + FsLink> FsPath for T {}
```

### SelfResolving (Marker)

Marker trait for backends that handle their own path resolution (e.g., `VRootFsBackend`, `StdFsBackend`). `FileStorage` will NOT perform virtual path resolution for these backends.

```rust
pub trait SelfResolving {}
```

---

## Layer 3: Inode Trait (For FUSE)

### FsInode

```rust
pub trait FsInode: Send + Sync {
    fn path_to_inode(&self, path: &Path) -> Result<u64, FsError>;
    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, FsError>;
    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, FsError>;
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, FsError>;
}
```

---

## Layer 4: POSIX Traits (Full POSIX)

### POSIX Types

```rust
/// Opaque file handle (inode-based for efficiency)
pub struct Handle(pub u64);

/// File open flags (mirrors POSIX)
#[derive(Clone, Copy, Debug)]
pub struct OpenFlags {
    pub read: bool,
    pub write: bool,
    pub create: bool,
    pub truncate: bool,
    pub append: bool,
}

impl OpenFlags {
    pub const READ: Self = Self { read: true, write: false, create: false, truncate: false, append: false };
    pub const WRITE: Self = Self { read: false, write: true, create: true, truncate: true, append: false };
    pub const READ_WRITE: Self = Self { read: true, write: true, create: false, truncate: false, append: false };
    pub const APPEND: Self = Self { read: false, write: true, create: true, truncate: false, append: true };
}

/// File lock type (mirrors POSIX flock)
#[derive(Clone, Copy, Debug)]
pub enum LockType {
    Shared,     // Multiple readers
    Exclusive,  // Single writer
}
```

### FsHandles

```rust
pub trait FsHandles: Send + Sync {
    fn open(&self, path: &Path, flags: OpenFlags) -> Result<Handle, FsError>;
    fn read_at(&self, handle: Handle, buf: &mut [u8], offset: u64) -> Result<usize, FsError>;
    fn write_at(&self, handle: Handle, data: &[u8], offset: u64) -> Result<usize, FsError>;
    fn close(&self, handle: Handle) -> Result<(), FsError>;
}
```

### FsLock

```rust
pub trait FsLock: Send + Sync {
    fn lock(&self, handle: Handle, lock: LockType) -> Result<(), FsError>;
    fn try_lock(&self, handle: Handle, lock: LockType) -> Result<bool, FsError>;
    fn unlock(&self, handle: Handle) -> Result<(), FsError>;
}
```

### FsXattr

```rust
pub trait FsXattr: Send + Sync {
    fn get_xattr(&self, path: &Path, name: &str) -> Result<Vec<u8>, FsError>;
    fn set_xattr(&self, path: &Path, name: &str, value: &[u8]) -> Result<(), FsError>;
    fn remove_xattr(&self, path: &Path, name: &str) -> Result<(), FsError>;
    fn list_xattr(&self, path: &Path) -> Result<Vec<String>, FsError>;
}
```

---

## Convenience Supertraits

These are automatically implemented via blanket impls:

```rust
/// Basic filesystem - covers 90% of use cases
pub trait Fs: FsRead + FsWrite + FsDir {}
impl<T: FsRead + FsWrite + FsDir> Fs for T {}

/// Full filesystem with all std::fs features
pub trait FsFull: Fs + FsLink + FsPermissions + FsSync + FsStats {}
impl<T: Fs + FsLink + FsPermissions + FsSync + FsStats> FsFull for T {}

/// FUSE-mountable filesystem
pub trait FsFuse: FsFull + FsInode {}
impl<T: FsFull + FsInode> FsFuse for T {}

/// Full POSIX filesystem
pub trait FsPosix: FsFuse + FsHandles + FsLock + FsXattr {}
impl<T: FsFuse + FsHandles + FsLock + FsXattr> FsPosix for T {}
```

---

## When to Use Each Level

| Level | Trait     | Use When                                  |
| ----- | --------- | ----------------------------------------- |
| 1     | `Fs`      | Basic file operations (read, write, dirs) |
| 2     | `FsFull`  | Need links, permissions, sync, or stats   |
| 3     | `FsFuse`  | FUSE mounting or hardlink support         |
| 4     | `FsPosix` | Full POSIX (file handles, locks, xattr)   |

---

## Implementing Functions

Use trait bounds to specify requirements:

```rust
use anyfs::FileStorage;

// Works with any backend, keeps std::fs-style paths
fn process_files<B: Fs>(fs: &FileStorage<B>) -> Result<(), FsError> {
    let data = fs.read("/input.txt")?;
    fs.write("/output.txt", &data)?;
    Ok(())
}

// Requires link support
fn create_backup<B: Fs + FsLink>(fs: &FileStorage<B>) -> Result<(), FsError> {
    fs.hard_link("/data.txt", "/data.txt.bak")?;
    Ok(())
}

// Requires FUSE-level support (planned companion crate)
fn mount_filesystem(fs: impl FsFuse) -> Result<(), FsError> {
    anyfs_mount::MountHandle::mount(fs, "/mnt/myfs")?;
    Ok(())
}
```

---

## Extension Trait

`FsExt` provides convenience methods for any `Fs` backend:

```rust
pub trait FsExt: Fs {
    /// Check if path is a file.
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;

    /// Check if path is a directory.
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;

    /// JSON methods (require `serde` feature in anyfs-backend)
    #[cfg(feature = "serde")]
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, FsError>;
    #[cfg(feature = "serde")]
    fn write_json<T: Serialize>(&self, path: impl AsRef<Path>, value: &T) -> Result<(), FsError>;
}

// Blanket implementation
impl<B: Fs> FsExt for B {}
```
