# AnyFS — Architecture Decision Records

**Key design decisions with context and rationale**

---

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](#adr-001-path-based-vfsbackend-trait) | Path-Based VfsBackend Trait (20 methods) | **Accepted** |
| [ADR-002](#adr-002-three-crate-structure) | Three-Crate Structure | **Accepted** |
| [ADR-003](#adr-003-two-layer-path-handling) | Two-Layer Path Handling | **Accepted** |
| [ADR-004](#adr-004-virtualpath-from-strict-path) | VirtualPath from strict-path | **Accepted** |
| [ADR-005](#adr-005-lexical-path-resolution) | Lexical Path Resolution | **Accepted** |
| [ADR-006](#adr-006-full-symlink-and-hardlink-support) | Full Symlink and Hard Link Support | **Accepted** |
| [ADR-007](#adr-007-capacity-enforcement-in-container) | Capacity Enforcement in Container | **Accepted** |
| [ADR-008](#adr-008-synchronous-first-api) | Synchronous-First API | **Accepted** |
| [ADR-009](#adr-009-no-posix-compliance) | No POSIX Compliance | **Accepted** |
| [ADR-010](#adr-010-feature-gated-backends) | Feature-Gated Backends | **Accepted** |
| [ADR-011](#adr-011-stdfssfs-aligned-method-names) | std::fs Aligned Method Names | **Accepted** |

### Superseded ADRs (Historical)

These decisions were from an earlier graph-store design that was rejected:

| Old ADR | Topic | Superseded By |
|---------|-------|---------------|
| Storage-Agnostic Backend (graph) | `StorageBackend` with nodes/edges | ADR-001 (path-based) |
| Node/Edge Data Model | `NodeId`, `ContentId`, `Edge` | ADR-001 (path-based) |
| Mandatory Transactions | `transact()` closure API | ADR-001 (simple methods) |
| Newtype IDs | `NodeId`, `ContentId`, `ChunkId` | ADR-001 (paths as identifiers) |

---

## ADR-001: Path-Based VfsBackend Trait

### Status
**Accepted** (Updated: now 20 methods with symlinks/hardlinks)

### Context
We need to support multiple storage backends (SQLite, memory, filesystem). The question is: what abstraction should backends implement?

An earlier design used a graph-store model with `NodeId`, `ContentId`, edges, and mandatory transactions. This was over-engineered for filesystem operations.

### Decision
Create a **path-based** `VfsBackend` trait with 20 methods aligned with `std::fs`:

```rust
pub trait VfsBackend: Send {
    // Read operations (8)
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    fn read_to_string(&self, path: &VirtualPath) -> Result<String, VfsError>;
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    fn symlink_metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    fn read_dir(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;
    fn read_link(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError>;

    // Write operations (9)
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    fn create_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn create_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_file(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn remove_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;

    // Links (2)
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;

    // Permissions (1)
    fn set_permissions(&mut self, path: &VirtualPath, perm: Permissions) -> Result<(), VfsError>;
}
```

### Consequences
**Positive:**
- Familiar API (method names match `std::fs`)
- Full filesystem semantics (symlinks, hard links, permissions)
- Simple to implement backends
- No graph-store complexity
- Paths are natural identifiers for filesystem operations

**Negative:**
- More methods to implement (20 vs original 13)
- Backends must handle symlink resolution

### Alternatives Considered
1. **Graph-store model** (`NodeId`, edges, transactions): Rejected — over-engineered for our needs
2. **FUSE-style trait**: Rejected — too complex, POSIX-focused
3. **13-method trait without symlinks**: Superseded — needed full filesystem semantics

---

## ADR-002: Three-Crate Structure

### Status
**Accepted**

### Context
How should we organize the codebase? A single crate is simple but forces all dependencies on all users.

### Decision
Split into **three crates**:

| Crate | Purpose | Dependencies |
|-------|---------|--------------|
| `anyfs-traits` | Minimal trait + types | `strict-path`, `thiserror` |
| `anyfs` | Built-in backends (feature-gated) | `anyfs-traits` + optional deps |
| `anyfs-container` | Capacity limits, tenant isolation | `anyfs-traits` |

```
strict-path
     ↑
anyfs-traits
     ↑
     ├── anyfs (backends)
     └── anyfs-container (isolation layer)
```

### Consequences
**Positive:**
- Custom backend implementers only depend on `anyfs-traits` (tiny, no heavy deps)
- Built-in backends are opt-in via feature flags
- Follows Rust ecosystem patterns (`tower-service` vs `tower`)

**Negative:**
- More crates to maintain
- Users need to depend on multiple crates

### Alternatives Considered
1. **Single crate**: Rejected — forces `rusqlite` on everyone
2. **Two crates** (`anyfs` + `anyfs-container`): Rejected — custom backends would pull in built-in backend deps

---

## ADR-003: Two-Layer Path Handling

### Status
**Accepted**

### Context
Paths need validation for safety. Where should validation happen?

### Decision
Use **two-layer path handling**:

1. **User-facing (FilesContainer)**: Accepts `impl AsRef<Path>` for ergonomics
2. **Internal (VfsBackend)**: Uses `&VirtualPath` for type safety

```rust
// FilesContainer (user-facing)
impl<B: VfsBackend> FilesContainer<B> {
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError> {
        let vpath = VirtualPath::new(path.as_ref())?;  // Validate once
        Ok(self.backend.read(&vpath)?)                  // Backend gets safe path
    }
}

// VfsBackend (internal)
pub trait VfsBackend {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
}
```

### Consequences
**Positive:**
- User ergonomics: Pass `&str`, `String`, `&Path`, `PathBuf`
- Type safety: Backends receive pre-validated paths
- Single validation point: No redundant checks

**Negative:**
- Two different path types to understand

### Alternatives Considered
1. **`impl AsRef<Path>` everywhere**: Rejected — backends would need to re-validate
2. **`&VirtualPath` everywhere**: Rejected — poor user ergonomics

---

## ADR-004: VirtualPath from strict-path

### Status
**Accepted**

### Context
We need a safe, validated path type. Should we define our own or reuse an existing crate?

### Decision
**Re-export `VirtualPath` from `strict-path`** crate:

```rust
// In anyfs-traits/src/lib.rs
pub use strict_path::VirtualPath;
```

The `strict-path` crate provides:
- `VirtualPath` — validated, normalized paths
- `VirtualRoot` — containment for filesystem backend

### Consequences
**Positive:**
- No code duplication
- Battle-tested implementation
- `VRootFsBackend` can use `VirtualRoot` directly

**Negative:**
- External dependency
- Must track upstream changes

### Alternatives Considered
1. **Custom `VirtualPath`**: Rejected — duplicates `strict-path` functionality
2. **Raw `String` paths**: Rejected — no type safety

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
VirtualPath::new("/../../../etc/passwd")  // → "/etc/passwd" (clamped)
VirtualPath::new("/foo/../bar")           // → "/bar"
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

### Alternatives Considered
1. **POSIX-style resolution**: Rejected — complex, security implications
2. **Reject all `..`**: Rejected — too restrictive

---

## ADR-006: Full Symlink and Hard Link Support

### Status
**Accepted** (Updated: symlinks and hard links are now built-in, not opt-in)

### Context
Earlier design made symlinks and hard links opt-in features. This was reconsidered because:
- All three backends can support symlinks and hard links
- The API should be consistent across backends
- Simulated symlinks/hardlinks are just data structures (no security implications)

### Decision
Symlinks and hard links are **built-in to all backends**:

```rust
// All backends support these methods
backend.symlink("/target", "/link")?;
backend.hard_link("/original", "/link")?;
backend.read_link("/link")?;
backend.symlink_metadata("/link")?;
```

**Important clarification:** For `MemoryBackend` and `SqliteBackend`, these are *simulated* operations:
- Symlinks are stored as entries pointing to a target path
- Hard links share content via a common `ContentId`
- No real OS symlinks are created

Only `VRootFsBackend` creates real OS symlinks and hard links.

### Consequences
**Positive:**
- Consistent API across all backends
- Full filesystem semantics in the trait
- Simpler mental model (no feature flags)

**Negative:**
- Backend implementers must implement all 20 methods
- Slightly more complex backends

### Alternatives Considered
1. **Opt-in via ContainerBuilder**: Superseded — unnecessary complexity
2. **Separate traits for link support**: Rejected — fragmenting the API

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
- Backends stay simple
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
pub trait VfsBackend: Send {
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
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
- No user/group ownership
- No file locking primitives
- Symlinks are optional

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
The original 13-method trait used common but non-standard names:
- `list()` instead of `read_dir()`
- `mkdir()` instead of `create_dir()`
- `remove()` for both files and directories

Users familiar with Rust's `std::fs` found this confusing.

### Decision
Rename methods to align with `std::fs`:

| Old Name | New Name | std::fs Equivalent |
|----------|----------|-------------------|
| `list()` | `read_dir()` | `std::fs::read_dir` |
| `mkdir()` | `create_dir()` | `std::fs::create_dir` |
| `mkdir_all()` | `create_dir_all()` | `std::fs::create_dir_all` |
| `remove()` | `remove_file()` + `remove_dir()` | Split per std::fs |
| `remove_all()` | `remove_dir_all()` | `std::fs::remove_dir_all` |

Additional methods added:
- `read_to_string()` — matches `std::fs::read_to_string`
- `symlink_metadata()` — matches `std::fs::symlink_metadata`

### Consequences
**Positive:**
- Familiar naming for Rust developers
- Clear distinction between `remove_file` and `remove_dir`
- Less cognitive load when switching between `std::fs` and `anyfs`

**Negative:**
- Breaking change from earlier versions
- Documentation must be updated

### Alternatives Considered
1. **Keep original names**: Rejected — inconsistency with `std::fs` caused confusion
2. **Provide both aliases**: Rejected — API bloat, maintenance burden

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
