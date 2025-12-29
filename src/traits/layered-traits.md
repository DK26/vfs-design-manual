# Layered Traits (anyfs-backend)

AnyFS uses a **layered trait architecture** for maximum flexibility with minimal complexity.

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
```

**Simple rule:** Import `Fs` for basic use. Add traits as needed for advanced features.

---

## Layer 1: Core Traits (Required)

### FsRead

```rust
pub trait FsRead: Send + Sync {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError>;
    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, FsError>;
    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, FsError>;
    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;
    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError>;
    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn Read + Send>, FsError>;
}
```

### FsWrite

```rust
pub trait FsWrite: Send + Sync {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn append(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn remove_file(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn rename(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn copy(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn truncate(&self, path: impl AsRef<Path>, size: u64) -> Result<(), FsError>;
    fn open_write(&self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, FsError>;
}
```

### FsDir

```rust
pub trait FsDir: Send + Sync {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, FsError>;
    fn create_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn create_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
}
```

---

## Layer 2: Extended Traits (Optional)

### FsLink

```rust
pub trait FsLink: Send + Sync {
    fn symlink(&self, target: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError>;
    fn hard_link(&self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, FsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError>;
}
```

### FsPermissions

```rust
pub trait FsPermissions: Send + Sync {
    fn set_permissions(&self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), FsError>;
}
```

### FsSync

```rust
pub trait FsSync: Send + Sync {
    fn sync(&self) -> Result<(), FsError>;
    fn fsync(&self, path: impl AsRef<Path>) -> Result<(), FsError>;
}
```

### FsStats

```rust
pub trait FsStats: Send + Sync {
    fn statfs(&self) -> Result<StatFs, FsError>;
}
```

---

## Layer 3: Inode Trait (For FUSE)

### FsInode

```rust
pub trait FsInode: Send + Sync {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, FsError>;
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
    fn open(&self, path: impl AsRef<Path>, flags: OpenFlags) -> Result<Handle, FsError>;
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
    fn get_xattr(&self, path: impl AsRef<Path>, name: &str) -> Result<Vec<u8>, FsError>;
    fn set_xattr(&self, path: impl AsRef<Path>, name: &str, value: &[u8]) -> Result<(), FsError>;
    fn remove_xattr(&self, path: impl AsRef<Path>, name: &str) -> Result<(), FsError>;
    fn list_xattr(&self, path: impl AsRef<Path>) -> Result<Vec<String>, FsError>;
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

| Level | Trait | Use When |
|-------|-------|----------|
| 1 | `Fs` | Basic file operations (read, write, dirs) |
| 2 | `FsFull` | Need links, permissions, sync, or stats |
| 3 | `FsFuse` | FUSE mounting or hardlink support |
| 4 | `FsPosix` | Full POSIX (file handles, locks, xattr) |

---

## Implementing Functions

Use trait bounds to specify requirements:

```rust
// Works with any backend
fn process_files(fs: &impl Fs) -> Result<(), FsError> {
    let data = fs.read("/input.txt")?;
    fs.write("/output.txt", &data)?;
    Ok(())
}

// Requires link support
fn create_backup(fs: &(impl Fs + FsLink)) -> Result<(), FsError> {
    fs.hard_link("/data.txt", "/data.txt.bak")?;
    Ok(())
}

// Requires FUSE-level support
fn mount_filesystem(fs: impl FsFuse) -> Result<(), FsError> {
    anyfs_mount::FuseMount::mount(fs, "/mnt/myfs")?;
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

