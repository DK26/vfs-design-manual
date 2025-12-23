# AGENTS.md — Instructions for AI Assistants

**READ THIS FIRST before making any changes to this repository.**

---

## Project Overview

This is the **AnyFS Ecosystem** — a **two-layer architecture** for virtual filesystem abstraction:

### Layer 1: anyfs (Low-Level)

| Crate | Purpose | Target User |
|-------|---------|-------------|
| `anyfs` | Filesystem-like `Vfs` trait (inode operations) | Backend implementers |

### Layer 2: anyfs-container (High-Level)

| Crate | Purpose | Target User |
|-------|---------|-------------|
| `anyfs-container` | `std::fs`-like API with capacity limits | Application developers |

### Supporting Concepts

| Concept | Purpose |
|---------|---------|
| `FsSemantics` | Pluggable path resolution rules (Linux, Windows, Simple) |

### Key Insight

- **anyfs** = "How do I store inodes and content?" (low-level, no paths)
- **anyfs-container** = "How do I use this like std::fs?" (high-level, paths)

---

## The Two-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  APPLICATION CODE                                           │
│                                                             │
│  // This looks just like std::fs!                           │
│  container.write("/data/file.txt", b"hello")?;              │
│  container.read_dir("/data")?;                              │
│  container.create_dir_all("/var/log")?;                     │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  anyfs-container: FilesContainer<V, S>                      │
│  ═══════════════════════════════════════                    │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │  Path Resolution │  │ Capacity Limits │                   │
│  │  (FsSemantics)   │  │ (CapacityLimits)│                   │
│  └────────┬────────┘  └────────┬────────┘                   │
│           │                    │                            │
│           └────────┬───────────┘                            │
│                    ▼                                        │
│           ┌───────────────┐                                 │
│           │ std::fs-like  │                                 │
│           │     API       │                                 │
│           └───────┬───────┘                                 │
│                   │                                         │
├───────────────────┼─────────────────────────────────────────┤
│                   ▼                                         │
│  anyfs: Vfs trait                                           │
│  ════════════════════                                       │
│                                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │create_inode │ │   lookup    │ │    read     │            │
│  │delete_inode │ │   link      │ │    write    │            │
│  │ get_inode   │ │   unlink    │ │  truncate   │            │
│  │update_inode │ │   readdir   │ │    sync     │            │
│  └─────────────┘ └─────────────┘ └─────────────┘            │
│                                                             │
│  NO PATHS — ONLY InodeId                                    │
│                                                             │
├──────────────────┬──────────────────┬───────────────────────┤
│                  │                  │                       │
│   MemoryVfs      │   SqliteVfs      │   RealFsVfs           │
│   ══════════     │   ═════════      │   ═════════           │
│   HashMap        │   .db file       │   strict-path         │
│                  │                  │                       │
└──────────────────┴──────────────────┴───────────────────────┘
```

---

## ⚠️ CRITICAL: Old vs New Design

This repository contains documentation from multiple design iterations. **IGNORE OLD DESIGN DOCUMENTS.**

### ✅ CURRENT DESIGN (use this)

- **Two layers:** `anyfs` (low-level) and `anyfs-container` (high-level)
- **Low-level trait:** `Vfs` — inode-based operations (no paths!)
- **High-level API:** `FilesContainer<V, S>` — `std::fs`-like (paths)
- **Semantics:** Pluggable via `FsSemantics` trait
- **Backends:** `MemoryVfs`, `SqliteVfs`, `RealFsVfs`

### ❌ OLD DESIGN (ignore this)

If you see any of these, **it's from the old design — do not use:**

- `VfsBackend` trait with path-based methods — **WRONG** (old single-layer design)
- `&VirtualPath` in trait methods — **WRONG** (now uses `InodeId`)
- Single trait handling both paths and storage — **WRONG** (now separated)
- `anyfs-traits` as separate crate — **OUTDATED** (merged into `anyfs`)
- Any mention of "20 path-based methods" — **WRONG** (old design)
- `vfs-switchable` crate name — **WRONG** (renamed)
- `NodeId`, `ContentId`, `ChunkId` — **WRONG** (old graph-store model)

---

## The Two-Layer API

### Layer 1: anyfs — Vfs Trait (Filesystem-Like)

```rust
/// Low-level filesystem trait. NO path handling.
/// Backend implementers work directly with inodes.
pub trait Vfs: Send {
    // ═══════════════════════════════════════════════════════════
    // INODE LIFECYCLE
    // ═══════════════════════════════════════════════════════════

    /// Create a new inode (file, directory, or symlink)
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError>;

    /// Get inode metadata
    fn get_inode(&self, id: InodeId) -> Result<InodeData, VfsError>;

    /// Update inode metadata (mode, timestamps, etc.)
    fn update_inode(&mut self, id: InodeId, data: InodeData) -> Result<(), VfsError>;

    /// Delete inode (must have nlink=0)
    fn delete_inode(&mut self, id: InodeId) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════
    // DIRECTORY OPERATIONS
    // ═══════════════════════════════════════════════════════════

    /// Add entry to directory: parent[name] = child
    fn link(&mut self, parent: InodeId, name: &str, child: InodeId) -> Result<(), VfsError>;

    /// Remove entry from directory, returns removed child inode
    fn unlink(&mut self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;

    /// Lookup child by name in directory
    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;

    /// List all entries in directory
    fn readdir(&self, dir: InodeId) -> Result<Vec<(String, InodeId)>, VfsError>;

    // ═══════════════════════════════════════════════════════════
    // CONTENT I/O
    // ═══════════════════════════════════════════════════════════

    /// Read bytes from file at offset
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError>;

    /// Write bytes to file at offset
    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError>;

    /// Truncate or extend file to size
    fn truncate(&mut self, id: InodeId, size: u64) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════
    // SYNC & ROOT
    // ═══════════════════════════════════════════════════════════

    /// Flush all pending writes to durable storage
    fn sync(&mut self) -> Result<(), VfsError>;

    /// Get root directory inode
    fn root(&self) -> InodeId;
}
```

### Layer 2: anyfs-container — FilesContainer (std::fs-Like)

```rust
/// High-level container with std::fs-like API.
/// Application developers use this — never touch Vfs directly.
impl<V: Vfs, S: FsSemantics> FilesContainer<V, S> {
    // ═══════════════════════════════════════════════════════════
    // READ OPERATIONS (match std::fs)
    // ═══════════════════════════════════════════════════════════

    /// Like std::fs::read
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;

    /// Like std::fs::read_to_string
    pub fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, ContainerError>;

    /// Like std::fs::metadata
    pub fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;

    /// Like std::fs::symlink_metadata
    pub fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;

    /// Like std::fs::read_dir
    pub fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, ContainerError>;

    /// Like std::fs::read_link
    pub fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, ContainerError>;

    /// Like Path::exists
    pub fn exists(&self, path: impl AsRef<Path>) -> bool;

    // ═══════════════════════════════════════════════════════════
    // WRITE OPERATIONS (match std::fs)
    // ═══════════════════════════════════════════════════════════

    /// Like std::fs::write
    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;

    /// Like std::fs::create_dir
    pub fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;

    /// Like std::fs::create_dir_all
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;

    /// Like std::fs::remove_file
    pub fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;

    /// Like std::fs::remove_dir
    pub fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;

    /// Like std::fs::remove_dir_all
    pub fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;

    /// Like std::fs::rename
    pub fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;

    /// Like std::fs::copy
    pub fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;

    // ═══════════════════════════════════════════════════════════
    // LINK OPERATIONS (match std::fs / std::os::unix::fs)
    // ═══════════════════════════════════════════════════════════

    /// Like std::os::unix::fs::symlink
    pub fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;

    /// Like std::fs::hard_link
    pub fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;

    // ═══════════════════════════════════════════════════════════
    // PERMISSIONS (match std::fs)
    // ═══════════════════════════════════════════════════════════

    /// Like std::fs::set_permissions
    pub fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), ContainerError>;

    // ═══════════════════════════════════════════════════════════
    // CONTAINER-SPECIFIC (not in std::fs)
    // ═══════════════════════════════════════════════════════════

    /// Get current capacity usage
    pub fn usage(&self) -> &CapacityUsage;

    /// Get capacity limits
    pub fn limits(&self) -> &CapacityLimits;
}
```

---

## FsSemantics Trait

Path resolution is pluggable via `FsSemantics`:

```rust
/// Filesystem semantics — defines behavior rules, not storage.
pub trait FsSemantics: Send + Sync {
    /// Path separator character (e.g., '/' for Linux, '\\' for Windows)
    fn separator(&self) -> char;

    /// Split path into components
    fn components<'a>(&self, path: &'a str) -> Vec<&'a str>;

    /// Normalize path (resolve `.`, `..`, redundant separators)
    fn normalize(&self, path: &str) -> String;

    /// Is filesystem case-sensitive?
    fn case_sensitive(&self) -> bool;

    /// Maximum symlink resolution depth
    fn max_symlink_depth(&self) -> u32;

    // ... more methods for permissions, naming rules, etc.
}
```

**Built-in semantics:**
- `LinuxSemantics` — POSIX-like paths, case-sensitive
- `WindowsSemantics` — Windows paths, case-insensitive
- `SimpleSemantics` — Minimal, no symlinks, for testing

---

## The Three Backends

### 1. MemoryVfs

- In-memory HashMap storage
- Stores inodes and content in Rust data structures
- For testing and ephemeral use

### 2. SqliteVfs

- Single `.db` file contains entire filesystem
- Portable — copy file to move container
- Production-ready, crash-safe

### 3. RealFsVfs

- Uses `strict-path::VirtualRoot` for containment
- A real directory on disk acts as the virtual root
- Paths are clamped (cannot escape root)

---

## Common Mistakes to Avoid

### ❌ WRONG: Confusing the two layers

```rust
// WRONG — Vfs trait doesn't have path-based methods
impl Vfs for MyBackend {
    fn read(&self, path: &str) -> Result<Vec<u8>, VfsError> { ... }
}
```

```rust
// CORRECT — Vfs trait uses InodeId
impl Vfs for MyBackend {
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError> { ... }
}
```

### ❌ WRONG: Application code using Vfs directly

```rust
// WRONG — Applications should use FilesContainer
let vfs = MemoryVfs::new();
let ino = vfs.lookup(vfs.root(), "file.txt")?;
vfs.read(ino, 0, &mut buf)?;
```

```rust
// CORRECT — Applications use std::fs-like API
let container = FilesContainer::new(MemoryVfs::new(), LinuxSemantics::new());
let data = container.read("/file.txt")?;
```

### ❌ WRONG: Backend handling path resolution

```rust
// WRONG — Backends shouldn't parse paths
impl Vfs for MyBackend {
    fn create_file(&mut self, path: &str) -> Result<(), VfsError> {
        let components = path.split('/');  // NO! Not your job!
        ...
    }
}
```

```rust
// CORRECT — Backends just store inodes
impl Vfs for MyBackend {
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        let id = self.next_id();
        self.inodes.insert(id, InodeData::new(kind, mode));
        Ok(id)
    }
}
```

### ❌ WRONG: Using old path-based VfsBackend trait

```rust
// WRONG - old single-layer design
trait VfsBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
}
```

```rust
// CORRECT - new two-layer design
trait Vfs {
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError>;
    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError>;
}
```

---

## How Path Resolution Works

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

## Quick Reference

| Question | Answer |
|----------|--------|
| How many layers? | Two: `anyfs` (low-level) and `anyfs-container` (high-level) |
| What trait do backends implement? | `Vfs` (in `anyfs`) |
| What do applications use? | `FilesContainer` (in `anyfs-container`) |
| Does Vfs handle paths? | **No** — it uses `InodeId` only |
| Does FilesContainer handle paths? | **Yes** — it uses `impl AsRef<Path>` |
| Which API matches std::fs? | `FilesContainer` methods |
| Where is path resolution? | `FsSemantics` trait, used by `FilesContainer` |
| Where are capacity limits? | `FilesContainer` only |
| What implements Vfs? | `MemoryVfs`, `SqliteVfs`, `RealFsVfs` |
| What semantics are available? | `LinuxSemantics`, `WindowsSemantics`, `SimpleSemantics` |

---

## Crate Structure

```
anyfs/                         # Low-level: Vfs trait + backends
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── vfs.rs                 # Vfs trait
    ├── inode.rs               # InodeId, InodeData, InodeKind
    ├── types.rs               # FileType, Metadata, DirEntry
    ├── error.rs               # VfsError
    ├── memory/                # MemoryVfs backend
    ├── sqlite/                # SqliteVfs backend
    └── realfs/                # RealFsVfs backend

anyfs-container/               # High-level: std::fs-like API
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── container.rs           # FilesContainer<V, S>
    ├── builder.rs             # ContainerBuilder
    ├── semantics/             # FsSemantics trait + implementations
    │   ├── mod.rs
    │   ├── linux.rs           # LinuxSemantics
    │   ├── windows.rs         # WindowsSemantics
    │   └── simple.rs          # SimpleSemantics
    ├── limits.rs              # CapacityLimits
    ├── usage.rs               # CapacityUsage
    └── error.rs               # ContainerError
```

---

## Design Decisions

### Why Two Layers?

| Concern | Handled By |
|---------|------------|
| How to store inodes/content | `anyfs` (Vfs trait) |
| How to interpret paths | `anyfs-container` (FsSemantics) |
| Capacity limits | `anyfs-container` (FilesContainer) |
| User-facing API | `anyfs-container` (std::fs-like) |

### Why Inodes Instead of Paths?

| Benefit | Explanation |
|---------|-------------|
| **Hard links** | Multiple paths → same inode → same content |
| **Symlink resolution** | Resolve target, get inode, done |
| **Rename efficiency** | Change parent entry, inode unchanged |
| **FUSE support** | FUSE uses inode numbers |
| **Identity checking** | Same inode = same file |

### Why Pluggable Semantics?

- Mix any semantics with any storage
- Test with `SimpleSemantics` (no symlinks, fast)
- Use `WindowsSemantics` on Linux for compatibility
- Custom semantics for special use cases

---

## When in Doubt

1. **Two layers:** `anyfs` = low-level inodes, `anyfs-container` = high-level paths
2. **Backend implementers:** Implement `Vfs` trait (inode operations)
3. **Application developers:** Use `FilesContainer` (std::fs-like API)
4. **Path handling:** Done by `FsSemantics` in `anyfs-container`
5. **Capacity limits:** Enforced by `FilesContainer`

If documentation conflicts with this file, **this file is correct**.
