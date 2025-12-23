# Two-Layer Architecture

**The core design principle of AnyFS**

---

## Overview

AnyFS separates concerns into two distinct layers:

| Layer | Crate | Responsibility | Target User |
|-------|-------|----------------|-------------|
| **Low-level** | `anyfs` | Inode storage (no paths) | Backend implementers |
| **High-level** | `anyfs-container` | Path resolution + limits | Application developers |

```
┌─────────────────────────────────────────────────────────────┐
│  APPLICATION CODE                                           │
│                                                             │
│  container.write("/data/file.txt", b"hello")?;              │
│  container.read_dir("/data")?;                              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  anyfs-container: FilesContainer<V, S>                      │
│  ═══════════════════════════════════════                    │
│  • std::fs-like API (impl AsRef<Path>)                      │
│  • Path resolution via FsSemantics                          │
│  • Capacity enforcement (limits, quotas)                    │
│  • Target: Application developers                           │
├─────────────────────────────────────────────────────────────┤
│  anyfs: Vfs trait                                           │
│  ════════════════════                                       │
│  • Inode-based operations (InodeId, not paths)              │
│  • create_inode, lookup, link, unlink, read, write          │
│  • No path logic — raw inode operations                     │
│  • Target: Backend implementers                             │
├──────────────────┬──────────────────┬───────────────────────┤
│  MemoryVfs       │  SqliteVfs       │  RealFsVfs            │
│  (HashMap)       │  (.db file)      │  (strict-path)        │
└──────────────────┴──────────────────┴───────────────────────┘
```

---

## Why Two Layers?

### Separation of Concerns

| Concern | Layer |
|---------|-------|
| How to store data | anyfs (Vfs trait) |
| How to interpret paths | anyfs-container (FsSemantics) |
| Capacity limits | anyfs-container (FilesContainer) |
| User-facing API | anyfs-container (std::fs-like) |

### Benefits

1. **Backend simplicity**: Backend implementers don't deal with paths
2. **Semantic flexibility**: Mix any semantics with any storage
3. **Clear responsibilities**: Each layer does one thing well
4. **Testability**: Use `SimpleSemantics` for fast tests
5. **FUSE compatibility**: Inode-based model maps directly to FUSE

---

## Layer 1: anyfs

The `Vfs` trait defines low-level inode operations:

```rust
pub trait Vfs: Send {
    // Inode lifecycle
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError>;
    fn get_inode(&self, id: InodeId) -> Result<InodeData, VfsError>;
    fn update_inode(&mut self, id: InodeId, data: InodeData) -> Result<(), VfsError>;
    fn delete_inode(&mut self, id: InodeId) -> Result<(), VfsError>;

    // Directory operations
    fn link(&mut self, parent: InodeId, name: &str, child: InodeId) -> Result<(), VfsError>;
    fn unlink(&mut self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn readdir(&self, dir: InodeId) -> Result<Vec<(String, InodeId)>, VfsError>;

    // Content I/O
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError>;
    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError>;
    fn truncate(&mut self, id: InodeId, size: u64) -> Result<(), VfsError>;

    // Sync & root
    fn sync(&mut self) -> Result<(), VfsError>;
    fn root(&self) -> InodeId;
}
```

**Key insight:** No paths. Only `InodeId`.

---

## Layer 2: anyfs-container

`FilesContainer` provides a `std::fs`-like API:

```rust
impl<V: Vfs, S: FsSemantics> FilesContainer<V, S> {
    // Matches std::fs::read
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;

    // Matches std::fs::write
    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;

    // Matches std::fs::create_dir_all
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;

    // ... all std::fs-like methods
}
```

**Key insight:** Handles path resolution, enforces limits, delegates to Vfs.

---

## How They Connect

When a user calls `container.write("/data/file.txt", b"hello")`:

```
1. Parse path via FsSemantics
   "/data/file.txt" → ["data", "file.txt"]

2. Walk to parent via Vfs
   vfs.root() → InodeId(1)
   vfs.lookup(1, "data") → InodeId(5)

3. Check capacity limits
   usage + 5 bytes ≤ limit?

4. Create/get file inode
   vfs.lookup(5, "file.txt") or vfs.create_inode(...)
   → InodeId(12)

5. Write content
   vfs.truncate(12, 0)
   vfs.write(12, 0, b"hello")

6. Update usage tracking
   self.usage.total_size += 5
```

---

## FsSemantics

Path resolution is pluggable:

```rust
pub trait FsSemantics: Send + Sync {
    fn separator(&self) -> char;
    fn components<'a>(&self, path: &'a str) -> Vec<&'a str>;
    fn normalize(&self, path: &str) -> String;
    fn case_sensitive(&self) -> bool;
    fn max_symlink_depth(&self) -> u32;
    // ...
}
```

**Built-in implementations:**

| Semantics | Path Separator | Case Sensitive | Notes |
|-----------|----------------|----------------|-------|
| `LinuxSemantics` | `/` | Yes | POSIX-like |
| `WindowsSemantics` | `\` (also `/`) | No | Windows-like |
| `SimpleSemantics` | `/` | Yes | Minimal, no symlinks |

---

## Summary

| Aspect | anyfs | anyfs-container |
|--------|-------|-----------------|
| **Trait** | `Vfs` | `FilesContainer` |
| **Works with** | `InodeId` | `impl AsRef<Path>` |
| **Target user** | Backend implementers | App developers |
| **Handles paths** | No | Yes |
| **Capacity limits** | No | Yes |
| **API style** | Low-level | std::fs-like |
