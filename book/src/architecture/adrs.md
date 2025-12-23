# AnyFS — Architecture Decision Records

**Key design decisions with context and rationale**

---

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](#adr-001-inode-based-vfs-trait) | Inode-Based Vfs Trait | **Accepted** |
| [ADR-002](#adr-002-two-crate-structure) | Two-Crate Structure | **Accepted** |
| [ADR-003](#adr-003-two-layer-architecture) | Two-Layer Architecture | **Accepted** |
| [ADR-004](#adr-004-pluggable-fssemantics) | Pluggable FsSemantics | **Accepted** |
| [ADR-005](#adr-005-lexical-path-resolution) | Lexical Path Resolution | **Accepted** |
| [ADR-006](#adr-006-full-symlink-and-hardlink-support) | Full Symlink and Hard Link Support | **Accepted** |
| [ADR-007](#adr-007-capacity-enforcement-in-container) | Capacity Enforcement in Container | **Accepted** |
| [ADR-008](#adr-008-synchronous-first-api) | Synchronous-First API | **Accepted** |
| [ADR-009](#adr-009-no-posix-compliance) | No POSIX Compliance | **Accepted** |
| [ADR-010](#adr-010-feature-gated-backends) | Feature-Gated Backends | **Accepted** |
| [ADR-011](#adr-011-stdfs-aligned-method-names) | std::fs Aligned Method Names | **Accepted** |

### Superseded ADRs (Historical)

These decisions were from earlier designs that have been superseded:

| Old Design | Topic | Superseded By |
|------------|-------|---------------|
| Graph-Store Model | `StorageBackend` with nodes/edges | ADR-001 (inode-based) |
| Path-Based VfsBackend | 20 methods with `&VirtualPath` | ADR-001 (inode-based) |
| Three-Crate Structure | `anyfs-traits` + `anyfs` + `anyfs-container` | ADR-002 (two crates) |
| Single Path Semantics | Fixed POSIX-like paths | ADR-004 (pluggable semantics) |

---

## ADR-001: Inode-Based Vfs Trait

### Status
**Accepted** (Updated: replaces path-based VfsBackend)

### Context
We need to support multiple storage backends (SQLite, memory, filesystem). Earlier designs used either:
1. A graph-store model with `NodeId`, edges, and transactions (over-engineered)
2. A path-based `VfsBackend` with 20 methods using `&VirtualPath` (paths in backend)

Both approaches mixed storage concerns with path semantics.

### Decision
Create an **inode-based** `Vfs` trait with 13 methods:

```rust
pub trait Vfs: Send {
    // Inode lifecycle (4)
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError>;
    fn get_inode(&self, id: InodeId) -> Result<InodeData, VfsError>;
    fn update_inode(&mut self, id: InodeId, data: InodeData) -> Result<(), VfsError>;
    fn delete_inode(&mut self, id: InodeId) -> Result<(), VfsError>;

    // Directory operations (4)
    fn link(&mut self, parent: InodeId, name: &str, child: InodeId) -> Result<(), VfsError>;
    fn unlink(&mut self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn lookup(&self, parent: InodeId, name: &str) -> Result<InodeId, VfsError>;
    fn readdir(&self, dir: InodeId) -> Result<Vec<(String, InodeId)>, VfsError>;

    // Content I/O (3)
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError>;
    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError>;
    fn truncate(&mut self, id: InodeId, size: u64) -> Result<(), VfsError>;

    // Sync & root (2)
    fn sync(&mut self) -> Result<(), VfsError>;
    fn root(&self) -> InodeId;
}
```

Key insight: **No paths in Vfs**. Only `InodeId` and entry names.

### Consequences
**Positive:**
- Backend simplicity: no path parsing or normalization
- FUSE compatibility: inode model maps directly to FUSE
- Semantic flexibility: any path semantics can be layered on top
- Clear separation: backends handle storage, containers handle paths
- Natural hard link support: multiple entries can point to same inode

**Negative:**
- Unfamiliar to developers expecting path-based APIs
- Requires container layer for usable path API

### Alternatives Considered
1. **Graph-store model**: Rejected — over-engineered for filesystem operations
2. **Path-based VfsBackend**: Rejected — mixed storage with path semantics
3. **FUSE-style trait**: Rejected — too POSIX-focused

---

## ADR-002: Two-Crate Structure

### Status
**Accepted** (Updated: was three crates)

### Context
Earlier design used three crates (`anyfs-traits`, `anyfs`, `anyfs-container`). With the inode-based design, `anyfs-traits` is no longer needed as a separate crate since the trait is small enough to include in `anyfs`.

### Decision
Split into **two crates**:

| Crate | Purpose | Dependencies |
|-------|---------|--------------|
| `anyfs` | Vfs trait + built-in backends | `thiserror` + optional deps |
| `anyfs-container` | FilesContainer + FsSemantics | `anyfs` |

```
anyfs (Vfs trait + backends)
     ↑
anyfs-container (FilesContainer + FsSemantics)
     ↑
Your Application
```

### Consequences
**Positive:**
- Simpler structure than three crates
- Backend implementers depend only on `anyfs`
- Built-in backends are opt-in via feature flags
- Follows Rust ecosystem patterns

**Negative:**
- Backend implementers pull in the trait crate with backend code (though feature-gated)

### Alternatives Considered
1. **Single crate**: Rejected — forces `rusqlite` on everyone
2. **Three crates**: Rejected — separate traits crate unnecessary with small Vfs trait

---

## ADR-003: Two-Layer Architecture

### Status
**Accepted** (New)

### Context
Path handling and storage are separate concerns. Mixing them in one API leads to complexity and inflexibility.

### Decision
Separate concerns into **two layers**:

```
┌─────────────────────────────────────────────────────────────┐
│  anyfs-container: FilesContainer<V, S>                      │
│  • std::fs-like API (impl AsRef<Path>)                      │
│  • Path resolution via FsSemantics                          │
│  • Capacity enforcement                                     │
├─────────────────────────────────────────────────────────────┤
│  anyfs: Vfs trait                                           │
│  • Inode-based operations (InodeId)                         │
│  • No path logic                                            │
└─────────────────────────────────────────────────────────────┘
```

**Layer 1 (anyfs)**: Inode-based storage for backend implementers
**Layer 2 (anyfs-container)**: Path-based API for application developers

### Consequences
**Positive:**
- Backend simplicity: no path parsing in backends
- Semantic flexibility: mix any semantics with any storage
- Clear responsibilities: each layer does one thing well
- Testability: use `SimpleSemantics` for fast tests
- FUSE compatibility: inode model maps directly to FUSE

**Negative:**
- Two concepts to understand (inodes vs paths)
- More code in container layer for path resolution

### Alternatives Considered
1. **Single-layer path API**: Rejected — backends need to handle path semantics
2. **Single-layer inode API**: Rejected — poor developer ergonomics

---

## ADR-004: Pluggable FsSemantics

### Status
**Accepted** (New)

### Context
Different platforms have different path conventions (Linux uses `/`, Windows uses `\`, case sensitivity varies). A fixed path model limits flexibility.

### Decision
Make path resolution **pluggable** via `FsSemantics` trait:

```rust
pub trait FsSemantics: Send + Sync {
    fn separator(&self) -> char;
    fn components<'a>(&self, path: &'a str) -> Vec<&'a str>;
    fn normalize(&self, path: &str) -> String;
    fn case_sensitive(&self) -> bool;
    fn max_symlink_depth(&self) -> u32;
}
```

**Built-in implementations:**

| Semantics | Separator | Case Sensitive | Symlinks |
|-----------|-----------|----------------|----------|
| `LinuxSemantics` | `/` | Yes | Yes |
| `WindowsSemantics` | `\` | No | Yes |
| `SimpleSemantics` | `/` | Yes | No |

### Consequences
**Positive:**
- Cross-platform flexibility: same backend with different path rules
- Testing: `SimpleSemantics` for fast, simple tests
- Customization: applications can implement custom semantics

**Negative:**
- More configuration for simple use cases
- Potential confusion about which semantics to use

### Alternatives Considered
1. **Fixed POSIX semantics**: Rejected — limits Windows support
2. **Runtime-configurable**: Rejected — complexity for rare use case

---

## ADR-005: Lexical Path Resolution

### Status
**Accepted**

### Context
Path handling is a major source of security vulnerabilities (path traversal, symlink attacks).

### Decision
Paths are **lexically normalized** without consulting the filesystem:

```rust
// Pure string manipulation, no filesystem access
normalize("/foo/../bar")           // → "/bar"
normalize("/../../../etc/passwd")  // → "/etc/passwd" (clamped)
```

Key properties:
- No host filesystem interaction during resolution
- `..` cannot escape the virtual root
- Symlinks are followed after normalization (with depth limit)

### Consequences
**Positive:**
- Path traversal attacks are structurally impossible
- Deterministic behavior regardless of filesystem state
- No race conditions in path resolution

**Negative:**
- Some POSIX behaviors aren't replicated
- Users expecting POSIX semantics may be surprised

---

## ADR-006: Full Symlink and Hard Link Support

### Status
**Accepted**

### Context
Filesystems support symlinks and hard links. Should backends support them?

### Decision
Full support in all backends:

- **Symlinks**: Stored as `InodeKind::Symlink { target }`. Container resolves them.
- **Hard links**: Multiple directory entries point to the same `InodeId`.
- **nlink tracking**: Backends track hard link count in `InodeData::nlink`.

```rust
pub enum InodeKind {
    File,
    Directory,
    Symlink { target: String },
}
```

Symlink resolution happens at the container layer via `FsSemantics::max_symlink_depth()`.

### Consequences
**Positive:**
- Consistent contract across all backends
- Natural fit for inode model (hard links are just multiple entries)
- Container controls symlink resolution policy

**Negative:**
- Backends must track nlink correctly
- Symlink resolution adds complexity to container

---

## ADR-007: Capacity Enforcement in Container

### Status
**Accepted**

### Context
Multi-tenant systems need resource limits. Where should limits be enforced?

### Decision
Capacity limits are enforced in `FilesContainer`, not in backends:

```rust
pub struct CapacityLimits {
    pub max_total_size: Option<u64>,
    pub max_file_size: Option<u64>,
    pub max_node_count: Option<u64>,
    pub max_dir_entries: Option<u32>,
    pub max_path_depth: Option<u16>,
}
```

### Consequences
**Positive:**
- Consistent enforcement across all backends
- Backends stay simple (pure storage)
- Limits configurable per-container

**Negative:**
- Usage tracking adds overhead
- Backend-specific limits need separate handling

---

## ADR-008: Synchronous-First API

### Status
**Accepted**

### Context
Should the API be sync, async, or both?

### Decision
Start with a **synchronous API**:

```rust
pub trait Vfs: Send {
    fn read(&self, id: InodeId, offset: u64, buf: &mut [u8]) -> Result<usize, VfsError>;
    // ... all sync methods
}
```

Async support can be added later as a separate trait if needed.

### Consequences
**Positive:**
- Simpler implementation
- SQLite (primary backend) is sync anyway
- No async runtime dependency
- Easier to reason about

**Negative:**
- Network-backed backends will block
- May need refactoring to add async

---

## ADR-009: No POSIX Compliance

### Status
**Accepted**

### Context
Most filesystems aim for POSIX compliance. Should we?

### Decision
Explicitly **do not target POSIX compliance**. We implement a simpler, safer subset.

Differences from POSIX:
- No device files, sockets, FIFOs
- No hard links to directories
- Lexical (not physical) path resolution
- Simplified permissions model
- No file locking primitives

### Consequences
**Positive:**
- Simpler implementation
- Smaller attack surface
- Predictable cross-platform behavior

**Negative:**
- Cannot be a drop-in OS filesystem replacement
- POSIX-expecting tools may not work

---

## ADR-010: Feature-Gated Backends

### Status
**Accepted**

### Context
The `anyfs` crate provides built-in backends. Should they all be included by default?

### Decision
Backends are **feature-gated**:

```toml
[features]
default = ["memory"]
memory = []
sqlite = ["rusqlite"]
vrootfs = ["strict-path"]
full = ["memory", "sqlite", "vrootfs"]
```

### Consequences
**Positive:**
- Minimal default dependencies
- Users opt-in to heavy deps (rusqlite)
- Faster compilation for simple cases

**Negative:**
- Users must enable features explicitly
- Documentation must explain features

---

## ADR-011: std::fs Aligned Method Names

### Status
**Accepted**

### Context
The `FilesContainer` API should be familiar to Rust developers.

### Decision
Method names align with `std::fs`:

| FilesContainer | std::fs |
|----------------|---------|
| `read()` | `std::fs::read` |
| `write()` | `std::fs::write` |
| `read_to_string()` | `std::fs::read_to_string` |
| `create_dir()` | `std::fs::create_dir` |
| `create_dir_all()` | `std::fs::create_dir_all` |
| `remove_file()` | `std::fs::remove_file` |
| `remove_dir()` | `std::fs::remove_dir` |
| `remove_dir_all()` | `std::fs::remove_dir_all` |
| `read_dir()` | `std::fs::read_dir` |
| `rename()` | `std::fs::rename` |
| `copy()` | `std::fs::copy` |
| `metadata()` | `std::fs::metadata` |
| `symlink_metadata()` | `std::fs::symlink_metadata` |

### Consequences
**Positive:**
- Familiar naming for Rust developers
- Less cognitive load when switching between `std::fs` and `anyfs`

**Negative:**
- Some `std::fs` methods don't have direct equivalents

---

## Template for Future ADRs

```markdown
## ADR-XXX: [Title]

### Status
[Proposed | Accepted | Deprecated | Superseded by ADR-YYY]

### Context
[What is the issue? Why do we need to make a decision?]

### Decision
[What is the change being proposed? Be specific.]

### Consequences
**Positive:**
- [Good outcome 1]
- [Good outcome 2]

**Negative:**
- [Tradeoff 1]
- [Tradeoff 2]

### Alternatives Considered
1. **[Alternative 1]**: [Why rejected]
2. **[Alternative 2]**: [Why rejected]
```

---

*For full technical details, see the [Design Overview](./design-overview.md).*
