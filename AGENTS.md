# AGENTS.md — Instructions for AI Assistants

**READ THIS FIRST before making any changes to this repository.**

---

## Project Overview

This is the **VFS Ecosystem** — two Rust crates for virtual filesystem abstraction:

| Crate | Purpose |
|-------|---------|
| `anyfs` | Generic VFS trait with swappable backends |
| `anyfs-container` | Wraps anyfs, adds capacity limits |

---

## ⚠️ CRITICAL: Old vs New Design

This repository contains documentation from multiple design iterations. **IGNORE OLD DESIGN DOCUMENTS.**

### ✅ CURRENT DESIGN (use this)

- **Crate names:** `anyfs` (NOT `vfs-switchable` or `vfs`)
- **Path type in VfsBackend trait:** `&VirtualPath` (from `strict-path` crate)
- **Path type in FilesContainer API:** `impl AsRef<Path>` (for user ergonomics)
- **Trait name:** `VfsBackend`
- **Trait style:** Path-based methods (`read`, `write`, `mkdir`, etc.)
- **Backends:** `VRootFsBackend`, `MemoryBackend`, `SqliteBackend`
- **Two crates:** `anyfs` (trait) and `anyfs-container` (wrapper with limits)

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
- `FilesContainer` as the only project — **WRONG** (there are TWO projects now)
- Any mention of "graph store" or "node/edge" model — **WRONG**

---

## The Correct Architecture

```
┌─────────────────────────────────────────┐
│  User Application                       │
├─────────────────────────────────────────┤
│  anyfs-container                          │  ← Capacity limits, isolation
│  FilesContainer<B: VfsBackend>          │     Uses impl AsRef<Path> (ergonomic)
├─────────────────────────────────────────┤
│  anyfs                               │  ← Core trait
│  VfsBackend trait                       │     Uses &VirtualPath (type-safe)
├──────────┬──────────┬───────────────────┤
│ VRootFs  │  Memory  │  SQLite           │  ← Backends
│ Backend  │  Backend │  Backend          │
└──────────┴──────────┴───────────────────┘
```

**Key insight:** Two-layer path handling:
1. **User-facing (FilesContainer):** `impl AsRef<Path>` — ergonomic, accepts any path-like type
2. **Internal (VfsBackend):** `&VirtualPath` — type-safe, pre-validated

---

## The Correct Trait

```rust
use strict_path::VirtualPath;

/// A virtual filesystem backend.
/// Implementations provide storage; callers get uniform I/O.
pub trait VfsBackend: Send {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    fn list(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;
    fn mkdir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn mkdir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
}
```

**This is 13 simple path-based methods. NOT a graph store. NOT transactional.**

**VirtualPath** comes from `strict-path` crate — re-exported by `anyfs`:
```rust
// In anyfs/src/lib.rs
pub use strict_path::VirtualPath;
```

---

## FilesContainer (User-Facing API)

```rust
use std::path::Path;
use anyfs::{VfsBackend, VirtualPath};

impl<B: VfsBackend> FilesContainer<B> {
    // User-facing: accepts flexible paths for ergonomics
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError> {
        let vpath = VirtualPath::new(path.as_ref())?;  // Validate & convert
        Ok(self.backend.read(&vpath)?)                  // Backend receives &VirtualPath
    }

    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError> {
        let vpath = VirtualPath::new(path.as_ref())?;
        self.check_limits(data.len())?;
        self.backend.write(&vpath, data)?;
        Ok(())
    }
    // ... other methods follow same pattern
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
| `strict-path` | `anyfs` (required) | VirtualPath type + VirtualRoot for containment |
| `rusqlite` | `SqliteBackend` (optional) | SQLite database access |
| `thiserror` | Both | Error types |

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

---

## File Structure

```
anyfs/                 # Project 1: Core trait + backends
├── src/
│   ├── lib.rs            # Re-exports (including VirtualPath from strict-path)
│   ├── backend.rs        # VfsBackend trait
│   ├── types.rs          # Metadata, DirEntry, FileType
│   ├── error.rs          # VfsError
│   ├── vrootfs/          # VRootFsBackend (uses strict-path)
│   ├── memory/           # MemoryBackend
│   └── sqlite/           # SqliteBackend

anyfs-container/            # Project 2: Isolation layer
├── src/
│   ├── lib.rs
│   ├── container.rs      # FilesContainer<B>
│   ├── builder.rs        # ContainerBuilder
│   ├── limits.rs         # CapacityLimits
│   └── error.rs          # ContainerError
```

---

## Quick Reference

| Question | Answer |
|----------|--------|
| Crate names? | `anyfs` and `anyfs-container` |
| Path type in VfsBackend? | `&VirtualPath` (from strict-path) |
| Path type in FilesContainer? | `impl AsRef<Path>` (for ergonomics) |
| Where does VirtualPath come from? | `strict-path` crate (re-exported by anyfs) |
| Backend trait name? | `VfsBackend` |
| Filesystem backend name? | `VRootFsBackend` (NOT `FsBackend`) |
| Does it use transactions? | No (old design) |
| Does it use NodeId/edges? | No (old design) |
| What provides containment? | `strict-path::VirtualRoot` |

---

## When in Doubt

1. **Crate name:** `anyfs` (NOT `vfs-switchable` or `vfs`)
2. **VfsBackend path type:** `&VirtualPath` — type-safe, from strict-path
3. **FilesContainer path type:** `impl AsRef<Path>` — ergonomic user-facing API
4. **Backend model:** Simple path-based methods, NOT graph store
5. **Crate structure:** TWO crates, not one
6. **Backend names:** `VRootFsBackend` (NOT `FsBackend`), `MemoryBackend`, `SqliteBackend`

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

### Decision 5: Two Crates (not one)

**Choice:** `anyfs` (core) + `anyfs-container` (isolation layer)

**Rationale:**
- **Separation of concerns:**
  - `anyfs` = pure I/O abstraction (trait + backends)
  - `anyfs-container` = policy layer (quotas, limits, isolation)
- **Flexibility:** Users who don't need quotas can use `anyfs` directly
- **Testability:** Core trait can be tested without container overhead
- **Dependency management:** Container depends on core, not vice versa

### Decision 6: Simple Path-Based Trait (not graph store)

**Choice:** 13 simple methods like `read()`, `write()`, `mkdir()`

**Rejected alternative:** Graph-store model with `NodeId`, `ContentId`, edges, transactions

**Rationale:**
- **Simplicity:** Filesystem operations are naturally path-based
- **Familiarity:** Matches `std::fs` mental model
- **Backend simplicity:** Easier to implement backends
- **No over-engineering:** Graph model was solving problems we don't have

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
