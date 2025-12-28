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
pub trait FsRead: Send {
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
pub trait FsWrite: Send {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
    fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError>;
    fn truncate(&mut self, path: impl AsRef<Path>, size: u64) -> Result<(), FsError>;
    fn open_write(&mut self, path: impl AsRef<Path>) -> Result<Box<dyn Write + Send>, FsError>;
}
```

### FsDir

```rust
pub trait FsDir: Send {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, FsError>;
    fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
    fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
}
```

---

## Layer 2: Extended Traits (Optional)

### FsLink

```rust
pub trait FsLink: Send {
    fn symlink(&mut self, target: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError>;
    fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), FsError>;
    fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, FsError>;
    fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError>;
}
```

### FsPermissions

```rust
pub trait FsPermissions: Send {
    fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), FsError>;
}
```

### FsSync

```rust
pub trait FsSync: Send {
    fn sync(&mut self) -> Result<(), FsError>;
    fn fsync(&mut self, path: impl AsRef<Path>) -> Result<(), FsError>;
}
```

### FsStats

```rust
pub trait FsStats: Send {
    fn statfs(&self) -> Result<StatFs, FsError>;
}
```

---

## Layer 3: Inode Trait (For FUSE)

### FsInode

```rust
pub trait FsInode: Send {
    fn path_to_inode(&self, path: impl AsRef<Path>) -> Result<u64, FsError>;
    fn inode_to_path(&self, inode: u64) -> Result<PathBuf, FsError>;
    fn lookup(&self, parent_inode: u64, name: &OsStr) -> Result<u64, FsError>;
    fn metadata_by_inode(&self, inode: u64) -> Result<Metadata, FsError>;
}
```

---

## Layer 4: POSIX Traits (Full POSIX)

### FsHandles

```rust
pub trait FsHandles: Send {
    fn open(&mut self, path: impl AsRef<Path>, flags: OpenFlags) -> Result<Handle, FsError>;
    fn read_at(&self, handle: Handle, buf: &mut [u8], offset: u64) -> Result<usize, FsError>;
    fn write_at(&mut self, handle: Handle, data: &[u8], offset: u64) -> Result<usize, FsError>;
    fn close(&mut self, handle: Handle) -> Result<(), FsError>;
}
```

### FsLock

```rust
pub trait FsLock: Send {
    fn lock(&mut self, handle: Handle, lock: LockType) -> Result<(), FsError>;
    fn try_lock(&mut self, handle: Handle, lock: LockType) -> Result<bool, FsError>;
    fn unlock(&mut self, handle: Handle) -> Result<(), FsError>;
}
```

### FsXattr

```rust
pub trait FsXattr: Send {
    fn get_xattr(&self, path: impl AsRef<Path>, name: &str) -> Result<Vec<u8>, FsError>;
    fn set_xattr(&mut self, path: impl AsRef<Path>, name: &str, value: &[u8]) -> Result<(), FsError>;
    fn remove_xattr(&mut self, path: impl AsRef<Path>, name: &str) -> Result<(), FsError>;
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
fn create_backup(fs: &mut (impl Fs + FsLink)) -> Result<(), FsError> {
    fs.hard_link("/data.txt", "/data.txt.bak")?;
    Ok(())
}

// Requires FUSE-level support
fn mount_filesystem(fs: impl FsFuse) -> Result<(), FsError> {
    anyfs_fuse::FuseMount::mount(fs, "/mnt/myfs")?;
    Ok(())
}
```

---

## Extension Trait

`FsExt` provides convenience methods for any `Fs` backend:

```rust
pub trait FsExt: Fs {
    fn read_json<T: DeserializeOwned>(&self, path: impl AsRef<Path>) -> Result<T, FsError>;
    fn write_json<T: Serialize>(&mut self, path: impl AsRef<Path>, value: &T) -> Result<(), FsError>;
    fn is_file(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;
    fn is_dir(&self, path: impl AsRef<Path>) -> Result<bool, FsError>;
}

// Blanket implementation
impl<B: Fs> FsExt for B {}
```

