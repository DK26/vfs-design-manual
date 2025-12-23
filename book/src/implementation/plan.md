# Implementation Plan

This plan describes a phased rollout of the **Two-Layer AnyFS Architecture**:

- **Layer 1 (`anyfs`)**: Inode-based `Vfs` trait + built-in backends
- **Layer 2 (`anyfs-container`)**: Path-based `FilesContainer` + `FsSemantics` + capacity limits

---

## Phase 1: `anyfs` Core (the foundation)

**Goal:** Define the inode-based storage trait and core types.

- Define `Vfs` trait with **13 inode-based methods**:
  - Inode lifecycle: `create_inode`, `get_inode`, `update_inode`, `delete_inode`
  - Directory ops: `link`, `unlink`, `lookup`, `readdir`
  - Content I/O: `read`, `write`, `truncate`
  - Sync & root: `sync`, `root`
- Define core types: `InodeId`, `InodeKind`, `InodeData`
- Define `VfsError` (errors reference `InodeId`, not paths)
- Implement `MemoryVfs` as the reference implementation

**Key insight:** No paths in this layer. Only `InodeId` and entry names.

**Exit criteria:** `MemoryVfs` passes all inode-based conformance tests.

---

## Phase 2: Additional Backends

**Goal:** Provide production-ready backends behind Cargo features.

| Feature | Backend | Storage |
|---------|---------|---------|
| `memory` (default) | `MemoryVfs` | HashMap |
| `sqlite` | `SqliteVfs` | Single `.db` file |
| `vrootfs` | `VRootVfs` | Real filesystem via `strict-path` |

Each backend implements the `Vfs` trait (13 methods).

**Exit criteria:** All backends pass the same conformance test suite.

---

## Phase 3: `anyfs-container` (the user-facing layer)

**Goal:** Provide the ergonomic `std::fs`-like API for application developers.

### 3.1 FsSemantics Trait

Define pluggable path resolution:

```rust
pub trait FsSemantics: Send + Sync {
    fn separator(&self) -> char;
    fn components<'a>(&self, path: &'a str) -> Vec<&'a str>;
    fn normalize(&self, path: &str) -> String;
    fn case_sensitive(&self) -> bool;
    fn max_symlink_depth(&self) -> u32;
}
```

Implement: `LinuxSemantics`, `WindowsSemantics`, `SimpleSemantics`

### 3.2 FilesContainer

```rust
pub struct FilesContainer<V: Vfs, S: FsSemantics> {
    vfs: V,
    semantics: S,
    limits: CapacityLimits,
    usage: CapacityUsage,
}
```

- Methods take `impl AsRef<Path>` (ergonomic)
- Path resolution via `FsSemantics`
- Delegates to `Vfs` after resolving paths to inodes
- **20 methods aligned with `std::fs`**: `read`, `write`, `create_dir`, `create_dir_all`, `remove_file`, `remove_dir`, `remove_dir_all`, `rename`, `copy`, `read_dir`, `metadata`, `symlink_metadata`, `read_link`, `symlink`, `hard_link`, `set_permissions`, `read_to_string`, `exists`

### 3.3 Capacity Limits

- Enforce quotas: `max_total_size`, `max_file_size`, `max_node_count`, `max_dir_entries`, `max_path_depth`
- Track usage: `total_size`, `node_count`, `file_count`, `directory_count`
- Define `ContainerError` separating backend failures from limit violations

**Exit criteria:** Application code uses familiar `std::fs`-like methods; all path handling is centralized in the container.

---

## Phase 4: Conformance Test Suite

**Goal:** Prevent backend divergence and document expected semantics.

### Layer 1 Tests (Vfs trait)

- Inode creation and deletion
- Directory entries (`link`, `unlink`, `lookup`, `readdir`)
- Content I/O at various offsets
- Hard link behavior (`nlink` tracking)
- Symlink storage (`InodeKind::Symlink`)

### Layer 2 Tests (FilesContainer)

- Path resolution with different `FsSemantics`
- Symlink following and loop detection
- Capacity limit enforcement
- Error mapping (`NotFound`, `AlreadyExists`, etc.)
- `std::fs` method compatibility

**Exit criteria:** `MemoryVfs`, `SqliteVfs`, and `VRootVfs` all pass the same suite.

---

## Phase 5: Documentation and Examples

**Goal:** Make the two-layer design easy to understand.

- `backend-guide.md`: How to implement a custom `Vfs` backend
- `semantics-guide.md`: How to implement custom `FsSemantics` (future)
- Keep `design-overview.md` and ADRs authoritative
- Examples showing both layers:
  - Application developer: `FilesContainer` with `LinuxSemantics`
  - Backend implementer: Custom `Vfs` implementation

---

## Future Work (out of scope for MVP)

- Streaming I/O (file handles)
- Async API (`AsyncVfs` trait)
- Import/export helpers (SQLAR, tar, etc.)
- Extended attributes
- FUSE adapter using `Vfs` directly
