# AnyFS — Virtual Filesystem for Rust

**A two-layer virtual filesystem abstraction with pluggable backends and semantics**

---

## Overview

A two-layer architecture separating storage from semantics:

| Layer | Crate | API Style | For |
|-------|-------|-----------|-----|
| **Low-level** | `anyfs` | Inode-based (`Vfs` trait) | Backend implementers |
| **High-level** | `anyfs-container` | `std::fs`-like (paths) | Application developers |

```text
┌─────────────────────────────────────────────────────────────┐
│  Application: container.write("/data/file.txt", b"hello")  │
├─────────────────────────────────────────────────────────────┤
│  anyfs-container: std::fs-like API + capacity limits       │
│  (FilesContainer<V, S> with FsSemantics)                   │
├─────────────────────────────────────────────────────────────┤
│  anyfs: Vfs trait (inode operations)                       │
├──────────────────┬──────────────────┬───────────────────────┤
│  MemoryVfs       │  SqliteVfs       │  VRootVfs            │
└──────────────────┴──────────────────┴───────────────────────┘
```

## Quick Example

### For Application Developers (std::fs-like)

```rust
use anyfs_container::{FilesContainer, LinuxSemantics};
use anyfs::MemoryVfs;

// Create container with memory backend
let mut container = FilesContainer::new(
    MemoryVfs::new(),
    LinuxSemantics::new(),
);

// Use exactly like std::fs!
container.create_dir_all("/data/nested")?;
container.write("/data/nested/file.txt", b"hello")?;

let content = container.read("/data/nested/file.txt")?;
assert_eq!(content, b"hello");

for entry in container.read_dir("/data")? {
    println!("{}", entry.name());
}
```

### For Backend Implementers (Low-Level)

```rust
use anyfs::{Vfs, InodeId, InodeKind, InodeData, VfsError};

struct MyCustomVfs { /* ... */ }

impl Vfs for MyCustomVfs {
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        // Allocate and store inode in your storage
    }

    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError> {
        // Find child by name in parent directory
    }

    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError> {
        // Read bytes from file content
    }

    // ... implement other Vfs methods
}
```

## Documentation

**Authoritative design document:** [`book/src/architecture/design-overview.md`](./book/src/architecture/design-overview.md)

Browse the full documentation with `mdbook serve book/`.

## Status

| Component | Status |
|-----------|--------|
| Design | Complete |
| Implementation | Not started |

## Key Design Decisions

See the [Architecture Decision Records](./book/src/architecture/adrs.md) for details:

1. **Two-layer architecture**: `anyfs` (inodes) + `anyfs-container` (paths)
2. **Pluggable semantics**: `LinuxSemantics`, `WindowsSemantics`, `SimpleSemantics`
3. **Inode-based storage**: Enables hard links, efficient renames, FUSE support
4. **Capacity limits**: Enforced at container layer, not in backends
