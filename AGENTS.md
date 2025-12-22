# AGENTS.md — Instructions for AI Assistants

**READ THIS FIRST before making any changes to this repository.**

---

## Project Overview

This is the **VFS Ecosystem** — three Rust crates for virtual filesystem abstraction:

| Crate | Purpose |
|-------|---------|
| `anyfs-traits` | Minimal crate — trait definition, types, re-exports `VirtualPath` |
| `anyfs` | Core VFS — re-exports traits, provides built-in backends (optional features) |
| `anyfs-container` | Higher-level wrapper — capacity limits, tenant isolation |

---

## ⚠️ CRITICAL: Old vs New Design

This repository contains documentation from multiple design iterations. **IGNORE OLD DESIGN DOCUMENTS.**

### ✅ CURRENT DESIGN (use this)

- **Crate names:** `anyfs-traits`, `anyfs`, `anyfs-container`
- **Trait crate:** `anyfs-traits` — minimal, contains `VfsBackend` trait + types
- **Path type in VfsBackend trait:** `&VirtualPath` (from `strict-path` crate)
- **Path type in FilesContainer API:** `impl AsRef<Path>` (for user ergonomics)
- **Trait name:** `VfsBackend` (defined in `anyfs-traits`)
- **Trait style:** Path-based methods aligned with `std::fs` (`read`, `write`, `create_dir`, etc.)
- **Backends:** `VRootFsBackend`, `MemoryBackend`, `SqliteBackend` (in `anyfs`, feature-gated)
- **Three crates:** `anyfs-traits` (trait), `anyfs` (backends), `anyfs-container` (wrapper with limits)

### ❌ OLD DESIGN (ignore this)

If you see any of these, **it's from the old design — do not use:**

- `vfs-switchable` crate name — **WRONG** (renamed to `anyfs`)
- `vfs` as single crate name — **WRONG** (conflicts with existing crates.io package)
- `impl AsRef<Path>` in VfsBackend trait — **WRONG** (VfsBackend uses `&VirtualPath`)
- Custom `VirtualPath` type definition — **WRONG** (use re-export from `strict-path`)
- `NodeId`, `ContentId`, `ChunkId` — **WRONG** (old graph-store model)
- `StorageBackend` trait with `insert_node`, `insert_edge` — **WRONG** (old graph-store model)
- `Transaction`, `Snapshot` traits — **WRONG** (old transactional model)
- `FsBackend` — **WRONG** name (it's `VRootFsBackend` to convey virtual root containment)
- `FilesContainer` as the only project — **WRONG** (there are THREE crates now)
- Two-crate structure (`anyfs` + `anyfs-container`) — **OUTDATED** (now three crates)
- Any mention of "graph store" or "node/edge" model — **WRONG**
- `list()` method — **WRONG** (renamed to `read_dir()` for std::fs alignment)
- `mkdir()` / `mkdir_all()` — **WRONG** (renamed to `create_dir()` / `create_dir_all()`)
- Single `remove()` method — **WRONG** (split into `remove_file()` and `remove_dir()`)
- `remove_all()` — **WRONG** (renamed to `remove_dir_all()`)
- "13 methods" in trait — **OUTDATED** (now 20 methods with symlinks, hard links, permissions)

---

## The Correct Architecture

```
┌─────────────────────────────────────────┐
│  User Application                       │
├─────────────────────────────────────────┤
│  anyfs-container                        │  ← Capacity limits, tenant isolation
│  FilesContainer<B: VfsBackend>          │     Uses impl AsRef<Path> (ergonomic)
├─────────────────────────────────────────┤
│  anyfs                                  │  ← Built-in backends (feature-gated)
├──────────┬──────────┬───────────────────┤
│ VRootFs  │  Memory  │  SQLite           │  ← Optional backend implementations
│ Backend  │  Backend │  Backend          │
├──────────┴──────────┴───────────────────┤
│  anyfs-traits                           │  ← Minimal: trait + types
│  VfsBackend trait, VfsError, Metadata   │     Re-exports VirtualPath
├─────────────────────────────────────────┤
│  strict-path (external)                 │  ← VirtualPath, VirtualRoot
└─────────────────────────────────────────┘
```

### Dependency Graph

```
strict-path (external)
     ↑
anyfs-traits (trait + types)
     ↑
     ├── anyfs (re-exports traits, provides backends)
     │
     └── anyfs-container (wraps any VfsBackend)
```

**Key insight:** Two-layer path handling:
1. **User-facing (FilesContainer):** `impl AsRef<Path>` — ergonomic, accepts any path-like type
2. **Internal (VfsBackend):** `&VirtualPath` — type-safe, pre-validated

---

## The Correct Trait (in anyfs-traits)

```rust
// anyfs-traits/src/lib.rs
pub use strict_path::VirtualPath;

/// A virtual filesystem backend.
/// All backends implement full filesystem semantics including symlinks and hard links.
/// Method names align with std::fs where applicable.
pub trait VfsBackend: Send {
    // ═══════════════════════════════════════════════════════════════════════════
    // READ OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════════

    /// Read entire file contents as bytes. Follows symlinks. (like std::fs::read)
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;

    /// Read entire file contents as UTF-8 string. Follows symlinks. (like std::fs::read_to_string)
    fn read_to_string(&self, path: &VirtualPath) -> Result<String, VfsError>;

    /// Read a byte range from a file. Follows symlinks. (extension)
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;

    /// Check if path exists. Follows symlinks. (like Path::exists)
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;

    /// Get metadata, following symlinks. (like std::fs::metadata)
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;

    /// Get metadata without following symlinks. (like std::fs::symlink_metadata)
    fn symlink_metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;

    /// Read directory contents. (like std::fs::read_dir)
    fn read_dir(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;

    /// Read symbolic link target. (like std::fs::read_link)
    fn read_link(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError>;

    // ═══════════════════════════════════════════════════════════════════════════
    // WRITE OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════════

    /// Write bytes to file, creating or overwriting. Follows symlinks. (like std::fs::write)
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;

    /// Append bytes to file. Follows symlinks. (like OpenOptions::append)
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;

    /// Create directory. (like std::fs::create_dir)
    fn create_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Create directory and all parent directories. (like std::fs::create_dir_all)
    fn create_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Remove file. Removes symlink itself, not target. (like std::fs::remove_file)
    fn remove_file(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Remove empty directory. (like std::fs::remove_dir)
    fn remove_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Remove directory and all contents recursively. (like std::fs::remove_dir_all)
    fn remove_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

    /// Rename or move file/directory. (like std::fs::rename)
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;

    /// Copy file. Follows symlinks. (like std::fs::copy)
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════════
    // LINKS
    // ═══════════════════════════════════════════════════════════════════════════

    /// Create symbolic link. `link` will point to `original`. (like std::os::unix::fs::symlink)
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;

    /// Create hard link. `link` will share content with `original`. (like std::fs::hard_link)
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;

    // ═══════════════════════════════════════════════════════════════════════════
    // PERMISSIONS
    // ═══════════════════════════════════════════════════════════════════════════

    /// Set permissions on file or directory. (like std::fs::set_permissions)
    fn set_permissions(&mut self, path: &VirtualPath, perm: Permissions) -> Result<(), VfsError>;
}
```

**This is 20 path-based methods aligned with `std::fs`. NOT a graph store. NOT transactional.**

**VirtualPath** comes from `strict-path` crate — re-exported by `anyfs-traits` (and `anyfs`):
```rust
// anyfs-traits/src/lib.rs
pub use strict_path::VirtualPath;

// anyfs/src/lib.rs (re-exports everything from traits)
pub use anyfs_traits::*;
```

---

## FilesContainer (User-Facing API)

```rust
use std::path::Path;
use anyfs::{VfsBackend, VirtualPath};

impl<B: VfsBackend> FilesContainer<B> {
    // User-facing: accepts flexible paths for ergonomics
    // Method names match VfsBackend (aligned with std::fs)

    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    pub fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, ContainerError>;
    pub fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, ContainerError>;
    pub fn exists(&self, path: impl AsRef<Path>) -> Result<bool, ContainerError>;
    pub fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, ContainerError>;
    pub fn read_link(&self, path: impl AsRef<Path>) -> Result<VirtualPath, ContainerError>;

    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;

    pub fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), ContainerError>;
}
```

---

## The Three Backends

### 1. VRootFsBackend

- Uses `strict-path::VirtualRoot` for containment
- A real directory on disk acts as the virtual root
- Paths are clamped (e.g., `/etc/passwd` → `root_dir/etc/passwd`)
- Name conveys "Virtual Root Filesystem" — NOT just "Fs"

### 2. MemoryBackend

- In-memory HashMap storage
- Uses `VirtualPath` as keys directly
- For testing

### 3. SqliteBackend

- Single `.db` file contains entire filesystem
- Portable — copy file to move container
- Internal schema is implementation detail

---

## Key Dependencies

| Crate | Used By | Purpose |
|-------|---------|---------|
| `strict-path` | `anyfs-traits` | VirtualPath type (re-exported) |
| `thiserror` | `anyfs-traits` | Error types |
| `strict-path` | `anyfs` [vrootfs] | VirtualRoot for containment |
| `rusqlite` | `anyfs` [sqlite] | SQLite database access |

---

## Common Mistakes to Avoid

### ❌ WRONG: Using old crate name

```rust
// WRONG - old crate name
use vfs_switchable::{VfsBackend, VRootFsBackend};
```

```rust
// CORRECT - current crate name
use anyfs::{VfsBackend, VRootFsBackend};
```

### ❌ WRONG: impl AsRef<Path> in VfsBackend

```rust
// WRONG - VfsBackend uses &VirtualPath, not impl AsRef<Path>
trait VfsBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, VfsError>;
}
```

```rust
// CORRECT - VfsBackend uses &VirtualPath
trait VfsBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
}
```

### ❌ WRONG: Defining custom VirtualPath

```rust
// WRONG - don't define your own VirtualPath
pub struct VirtualPath(String);
```

```rust
// CORRECT - re-export from strict-path
pub use strict_path::VirtualPath;
```

### ❌ WRONG: FsBackend name

```rust
// WRONG - loses semantic meaning of virtual root containment
let backend = FsBackend::new("/data")?;
```

```rust
// CORRECT - name conveys virtual root containment
let backend = VRootFsBackend::new("/data")?;
```

### ❌ WRONG: Graph-store trait

```rust
// WRONG - this is the old design
trait StorageBackend {
    fn insert_node(&mut self, node: &NodeRecord) -> Result<(), Error>;
    fn insert_edge(&mut self, edge: &Edge) -> Result<(), Error>;
}
```

```rust
// CORRECT - simple path-based trait
trait VfsBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
}
```

### ❌ WRONG: Using old method names

```rust
// WRONG - old method names (pre-std::fs alignment)
vfs.list(path)?;
vfs.mkdir(path)?;
vfs.mkdir_all(path)?;
vfs.remove(path)?;
vfs.remove_all(path)?;
```

```rust
// CORRECT - std::fs aligned names
vfs.read_dir(path)?;
vfs.create_dir(path)?;
vfs.create_dir_all(path)?;
vfs.remove_file(path)?;   // for files
vfs.remove_dir(path)?;    // for empty directories
vfs.remove_dir_all(path)?;
```

---

## File Structure

```
anyfs-traits/              # Crate 1: Minimal trait + types
├── Cargo.toml             # Depends on: strict-path, thiserror
├── src/
│   ├── lib.rs             # Re-exports VirtualPath, defines trait
│   ├── backend.rs         # VfsBackend trait
│   ├── types.rs           # Metadata, DirEntry, FileType
│   └── error.rs           # VfsError

anyfs/                     # Crate 2: Built-in backends
├── Cargo.toml             # Depends on: anyfs-traits + optional deps
├── src/
│   ├── lib.rs             # Re-exports anyfs-traits::*
│   ├── vrootfs/           # [feature: vrootfs] VRootFsBackend
│   ├── memory/            # [feature: memory] MemoryBackend (default)
│   └── sqlite/            # [feature: sqlite] SqliteBackend

anyfs-container/           # Crate 3: Isolation layer
├── Cargo.toml             # Depends on: anyfs-traits
├── src/
│   ├── lib.rs
│   ├── container.rs       # FilesContainer<B: VfsBackend>
│   ├── builder.rs         # ContainerBuilder
│   ├── limits.rs          # CapacityLimits
│   └── error.rs           # ContainerError
```

---

## Quick Reference

| Question | Answer |
|----------|--------|
| Crate names? | `anyfs-traits`, `anyfs`, `anyfs-container` |
| Where is the trait defined? | `anyfs-traits` (re-exported by `anyfs`) |
| Path type in VfsBackend? | `&VirtualPath` (from strict-path) |
| Path type in FilesContainer? | `impl AsRef<Path>` (for ergonomics) |
| Where does VirtualPath come from? | `strict-path` crate (re-exported by anyfs-traits) |
| Backend trait name? | `VfsBackend` |
| Filesystem backend name? | `VRootFsBackend` (NOT `FsBackend`) |
| How to implement custom backend? | Depend on `anyfs-traits` only |
| Total trait methods? | **20** (aligned with std::fs) |
| Directory listing method? | `read_dir()` (NOT `list()`) |
| Create directory method? | `create_dir()` (NOT `mkdir()`) |
| Remove file method? | `remove_file()` (NOT `remove()`) |
| Remove directory method? | `remove_dir()` (NOT `remove()`) |
| Symlinks supported? | Yes, all backends |
| Hard links supported? | Yes, all backends |
| Does it use transactions? | No (old design) |
| Does it use NodeId/edges? | No (old design) |
| What provides containment? | `strict-path::VirtualRoot` |

---

## When in Doubt

1. **Crate names:** `anyfs-traits`, `anyfs`, `anyfs-container`
2. **Trait location:** `anyfs-traits` crate (re-exported by `anyfs`)
3. **VfsBackend path type:** `&VirtualPath` — type-safe, from strict-path
4. **FilesContainer path type:** `impl AsRef<Path>` — ergonomic user-facing API
5. **Backend model:** Simple path-based methods, NOT graph store
6. **Crate structure:** THREE crates
7. **Backend names:** `VRootFsBackend` (NOT `FsBackend`), `MemoryBackend`, `SqliteBackend`
8. **Custom backend:** Depend only on `anyfs-traits`

If documentation conflicts with this file, **this file is correct**.

---

## Design Decisions & Rationale

This section documents WHY design choices were made, to help future sessions understand the reasoning.

### Decision 1: Crate Name — `anyfs` (not `vfs-*`)

**Choice:** `anyfs` and `anyfs-container`

**Rejected alternatives:**
- `vfs` — Conflicts with existing popular crate on crates.io (1.5M+ downloads)
- `vfs-core` — Still in `vfs-*` namespace, could cause confusion
- `vfs-switchable` — Implies the VFS itself is "switchable", which is misleading
- `vdrive`, `vstorage` — Considered but `anyfs` better conveys "any filesystem backend"

**Rationale:**
- `anyfs` clearly communicates "any filesystem backend can be plugged in"
- No namespace collision with existing crates
- Short, memorable, and unique
- The project is about storage simplicity and containment, which `anyfs` supports

### Decision 2: Two-Layer Path Handling

**Choice:**
- `VfsBackend` trait methods use `&VirtualPath`
- `FilesContainer` API uses `impl AsRef<Path>`

**Rationale:**
- **User ergonomics:** Application code can pass `&str`, `String`, `&Path`, `PathBuf` — whatever is convenient
- **Type safety internally:** Backends receive pre-validated `VirtualPath` — they don't need to re-validate
- **Single validation point:** Path validation happens once in `FilesContainer`, not in every backend
- **Containment guarantee:** `VirtualPath` (from `strict-path`) cannot escape root — structural safety

**How it works:**
```
User calls: container.read("/data/file.txt")  // Any path-like type
                    ↓
FilesContainer: VirtualPath::new(path)?       // Validate once
                    ↓
Backend: self.backend.read(&vpath)            // Receives safe path
```

### Decision 3: VirtualPath from strict-path (not custom)

**Choice:** Re-export `strict_path::VirtualPath`, don't define our own

**Rationale:**
- `strict-path` already provides a battle-tested `VirtualPath` type
- Avoids code duplication
- `strict-path` also provides `VirtualRoot` which `VRootFsBackend` needs
- Single source of truth for path validation logic

**Wrong approach:**
```rust
// DON'T DO THIS - duplicates strict-path functionality
pub struct VirtualPath(String);
impl VirtualPath { ... }
```

**Correct approach:**
```rust
// DO THIS - re-export from strict-path
pub use strict_path::VirtualPath;
```

### Decision 4: Backend Name — `VRootFsBackend` (not `FsBackend`)

**Choice:** `VRootFsBackend`

**Rationale:**
- `FsBackend` implies direct filesystem access — **misleading and dangerous**
- `VRootFsBackend` communicates:
  - `VRoot` = Virtual Root (contained)
  - `Fs` = Filesystem
  - `Backend` = Implements VfsBackend trait
- Users immediately understand this is sandboxed, not raw filesystem access
- Matches the underlying `strict_path::VirtualRoot` it uses

### Decision 5: Three Crates

**Choice:** `anyfs-traits` + `anyfs` + `anyfs-container`

**Rationale:**
- **Separation of concerns:**
  - `anyfs-traits` = minimal trait + types (for custom backend implementers)
  - `anyfs` = built-in backends (feature-gated)
  - `anyfs-container` = policy layer (quotas, limits, isolation)
- **Minimal dependencies for custom backends:** Implementers depend only on `anyfs-traits`
- **No forced transitive deps:** Custom backend doesn't pull in `rusqlite`, etc.
- **Follows Rust ecosystem patterns:** Like `tower-service` vs `tower`, `futures-core` vs `futures`

**Dependency flow:**
```
strict-path → anyfs-traits → anyfs (backends)
                          → anyfs-container
```

### Decision 6: Path-Based Trait Aligned with std::fs

**Choice:** 20 methods aligned with `std::fs` naming (`read()`, `write()`, `create_dir()`, etc.)

**Rejected alternative:** Graph-store model with `NodeId`, `ContentId`, edges, transactions

**Rationale:**
- **Simplicity:** Filesystem operations are naturally path-based
- **Familiarity:** Method names match `std::fs` (e.g., `create_dir` not `mkdir`)
- **Full filesystem semantics:** Symlinks, hard links, and permissions built-in
- **Backend simplicity:** Easier to implement backends
- **No over-engineering:** Graph model was solving problems we don't have

**Method renames for std::fs alignment:**
- `list()` → `read_dir()`
- `mkdir()` → `create_dir()`
- `mkdir_all()` → `create_dir_all()`
- `remove()` → split into `remove_file()` + `remove_dir()`
- `remove_all()` → `remove_dir_all()`

### Decision 7: Errors Use VirtualPath (not String)

**Choice:** `VfsError::NotFound(VirtualPath)` instead of `VfsError::NotFound(String)`

**Rationale:**
- Errors originate from operations on validated paths
- `VirtualPath` is already owned and cloneable
- Type consistency — if we use `VirtualPath` everywhere else, errors should too
- Can extract path info without parsing strings

---

## Historical Context

This repository went through several design iterations:

1. **v0.1 (Graph Store):** Used `NodeId`, `ContentId`, edges — over-engineered
2. **v0.2 (Path-Based, `impl AsRef<Path>`):** Simplified to path-based, but used `impl AsRef<Path>` in trait
3. **v0.3 (Current):** Two-layer path handling, `anyfs` naming, `&VirtualPath` in trait

The `review_findings.md` file documents inconsistencies found between v0.2 documents. This session resolved those inconsistencies and established v0.3 as the canonical design.

---

## Files to Trust vs Ignore

### ✅ Trust These (Current Design)
- `AGENTS.md` (this file) — **AUTHORITATIVE**
- `book/src/` — All documentation lives here in the mdbook

### ⚠️ Historical Content (Appendix)
- `book/src/appendix/review-findings.md` — Documents OLD inconsistencies (resolved)
- `book/src/appendix/pre-container-design.md` — Historical, pre-container design

### 🔴 Ignore If Conflicts
- Any document using `vfs-switchable`, `vfs-core`, or single `vfs` crate name
- Any document defining custom `VirtualPath` type
- Any document with graph-store model (`NodeId`, `insert_edge`, etc.)
