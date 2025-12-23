# AnyFS — Review Response & Decisions (Historical: Graph-Store Design)

> This document is **historical** and reflects an earlier (rejected) design iteration: a graph-store `StorageBackend` with `NodeId`/`Edge`/`ChunkId` and mandatory transactions.
>
> The **current** AnyFS design is **path-based** (20 `std::fs`-aligned methods) with `VfsBackend` defined in `anyfs-traits`, as captured in:
> - `book/src/architecture/adrs.md` (especially ADR-001)
> - `book/src/architecture/design-overview.md`
> - `book/src/overview/project-structure.md`

**Date:** 2025-12-22  
**In Response To:** Full Architecture Review + SQLite Addendum  
**Status:** Archived (superseded by ADR-001)

---

## Executive Summary

The reviews identified three critical issues and several high/medium priority items. This document records our decisions and the rationale for each.

**Key Decisions Made:**

| Issue | Decision |
|-------|----------|
| CF-1: Competing trait designs | **Graph-store (NodeId/Edge) is authoritative** |
| CF-2: FsBackend ambiguity | **Remove from MVP scope** |
| CF-3: Rename cycle prevention | **Ancestry check required, spec added** |
| HP-1: Large file I/O | **Downgrade AI/ML positioning until streaming exists** |
| HP-2: Concurrency | **Single-threaded by design, external sync required** |
| HP-3: Quota with hard links | **Logical bytes per node (not physical/shared)** |
| SQLite vs Binary | **SQLite-first, schema is versioned contract** |

---

## Critical Findings — Responses

### CF-1: Competing Backend Trait Designs

**Finding:** Documents show two different trait designs (graph-store vs direct path ops).

**Decision:** The **graph-store design (NodeId/Edge/Chunk)** is authoritative.

**Rationale:**
- Graph model naturally supports hard links, rename atomicity, and reference counting
- SQLite backend maps directly to tables
- Direct path ops would require each backend to re-implement path resolution
- The ADRs already established this; the MVP doc was exploratory

**Actions:**
- [x] Document 08 (Technical Comparison) MVP trait section will be marked as "exploratory/superseded"
- [x] README will state: "ADRs and anyfs-container-design.md are authoritative"
- [x] Single trait definition in design doc is the source of truth

**Authoritative Trait (for the record):**

```rust
pub trait StorageBackend: Send {
    fn transact<F, T>(&mut self, f: F) -> Result<T, BackendError>
    where F: FnOnce(&mut dyn Transaction) -> Result<T, BackendError>;
    
    fn snapshot(&self) -> Box<dyn Snapshot + '_>;
}

pub trait Snapshot: Send {
    fn get_node(&self, id: NodeId) -> Result<Option<NodeRecord>, BackendError>;
    fn get_edge(&self, parent: NodeId, name: &Name) -> Result<Option<NodeId>, BackendError>;
    fn list_edges(&self, parent: NodeId) -> Result<Vec<Edge>, BackendError>;
    fn read_chunk(&self, id: ChunkId) -> Result<Option<Vec<u8>>, BackendError>;
}

pub trait Transaction: Snapshot {
    fn insert_node(&mut self, node: &NodeRecord) -> Result<(), BackendError>;
    fn update_node(&mut self, id: NodeId, node: &NodeRecord) -> Result<(), BackendError>;
    fn delete_node(&mut self, id: NodeId) -> Result<(), BackendError>;
    fn insert_edge(&mut self, edge: &Edge) -> Result<(), BackendError>;
    fn delete_edge(&mut self, parent: NodeId, name: &Name) -> Result<(), BackendError>;
    fn write_chunk(&mut self, id: ChunkId, data: &[u8]) -> Result<(), BackendError>;
    fn delete_content(&mut self, id: ContentId) -> Result<(), BackendError>;
    fn next_node_id(&mut self) -> Result<NodeId, BackendError>;
    fn next_content_id(&mut self) -> Result<ContentId, BackendError>;
}
```

---

### CF-2: FsBackend Ambiguity

**Finding:** How does a path-based host filesystem satisfy a NodeId/Edge graph API?

**Decision:** **Remove FsBackend from MVP scope.**

**Rationale:**
- The graph-store trait requires NodeId persistence across restarts
- A true "adapter" over host FS would need a sidecar index (defeating simplicity)
- A "graph-store-on-FS" is just a worse SQLite (files as records)
- MVP focus should be: prove the abstraction with SQLite + Memory
- FsBackend can be added post-MVP if there's demand, with clear semantics

**Actions:**
- [x] Remove `vfs-fs` from MVP crate structure
- [x] Update implementation plan to: SQLite + Memory only
- [x] Add "Future: FsBackend" section explaining the complexity

**MVP Crate Structure (revised):**

```
anyfs-container/
├── anyfs/       # Traits, types (StorageBackend, NodeId, VirtualPath)
├── vfs-sqlite/     # SqliteBackend implementation
├── vfs-memory/     # MemoryBackend implementation
└── vfs/            # FilesContainer + builder (batteries included)
```

**Note on `strict-path`:** We can still use `strict-path` for import/export operations (copying files from/to host filesystem), just not as a backend.

---

### CF-3: Directory Rename Cycle Prevention

**Finding:** Renaming `/a` → `/a/b/a` would create a cycle. This must be prevented.

**Decision:** Add **ancestry check** to rename operation.

**Specification:**

```
INVARIANT: The directory graph is a tree rooted at NodeId(1).
           (No node may be its own ancestor.)

RENAME(source, dest):
  1. Resolve source to source_node_id
  2. Resolve dest parent to dest_parent_id
  3. IF source is a directory:
       Walk ancestors of dest_parent_id up to root
       IF source_node_id is found in ancestor chain:
         RETURN Err(VfsError::WouldCreateCycle)
  4. Proceed with rename (delete old edge, insert new edge)
```

**Error type:**

```rust
pub enum VfsError {
    // ... existing variants ...
    
    /// Rename would create a directory cycle
    WouldCreateCycle { source: VirtualPath, dest: VirtualPath },
}
```

**Conformance tests required:**
- `rename_dir_into_own_subtree_fails`
- `rename_dir_into_deeply_nested_subtree_fails`
- `rename_dir_to_sibling_succeeds`
- `rename_file_over_existing_file` (decide: overwrite or error?)

**Decision on rename-over-existing:**
- File over file: **Overwrite** (like POSIX `rename`)
- Dir over empty dir: **Overwrite** (like POSIX)
- Dir over non-empty dir: **Error** (DirectoryNotEmpty)
- File over dir or dir over file: **Error** (type mismatch)

---

## High Priority Findings — Responses

### HP-1: Large File I/O vs AI/ML Positioning

**Finding:** Executive summary cites AI/ML use cases, but API is bulk bytes only.

**Decision:** **Downgrade AI/ML positioning** in marketing docs until streaming exists.

**Actions:**
- [x] Remove "AI/ML pipelines" from executive summary primary use cases
- [x] Add to "Future Considerations": streaming API for large files
- [x] Keep `read_range()` in MVP (partial reads are useful)
- [x] Note that `rusqlite` blob feature enables future streaming without schema change

**Revised primary use cases:**
- Multi-tenant SaaS with per-tenant file storage
- Desktop applications with portable user data
- Testing with deterministic filesystem state
- Embedded systems with portable storage

---

### HP-2: Concurrency Model

**Finding:** Need explicit contract for concurrent access.

**Decision:** **Single-threaded by design; external synchronization required.**

**Specification:**

```
CONCURRENCY CONTRACT:

1. FilesContainer is NOT thread-safe by itself.
   - All methods take &self or &mut self
   - &mut self methods require exclusive access

2. For multi-threaded use:
   - Wrap in Arc<Mutex<FilesContainer<B>>> or similar
   - Or use a connection pool pattern (one container per thread)

3. Backend-level concurrency:
   - SQLite has internal locking (safe for multi-process read)
   - Memory backend has no internal locking
   - Container API still requires coherent access patterns

4. NOT SUPPORTED in v1:
   - Concurrent writers to the same container
   - Multi-process write access (SQLite limitation anyway)
```

**Actions:**
- [x] Add "Concurrency" section to design doc
- [x] Add example of `Arc<Mutex<_>>` pattern to getting started guide
- [x] Document SQLite WAL mode as recommended default

---

### HP-3: Quota Accounting with Hard Links

**Finding:** How is `total_size` computed when nodes share content via hard links?

**Decision:** **Logical bytes per node** (simpler, more predictable for quotas).

**Specification:**

```
QUOTA ACCOUNTING RULES:

1. total_size = SUM of (size field) for all file nodes
   - Each file node has its own size, even if content is shared
   - This is "logical size", not "physical storage"

2. Hard link behavior:
   - Creating hard link: new node created with same content_id
   - New node has its own size field (equal to original)
   - total_size INCREASES (counts both nodes)
   - This is intentional: quotas should reflect "apparent" usage

3. Delete behavior:
   - Deleting a file node: total_size DECREASES by node's size
   - Content deletion: deferred until no nodes reference content_id
   - Reference counting via link_count in NodeMetadata

4. Copy behavior:
   - Copy creates new node with NEW content_id
   - Content is duplicated (no implicit sharing)
   - total_size INCREASES by copied size

5. Rationale:
   - Predictable for users ("I have 10 files of 1MB each = 10MB used")
   - Avoids "I deleted a file but quota didn't change" confusion
   - Hard links are opt-in and advanced; users opting in understand sharing
```

**Alternative considered:** Physical bytes (deduplicated). Rejected because:
- Complicates quota enforcement (when does space free up?)
- Confuses users about apparent vs actual usage
- Can be added as a separate "physical_size" metric later if needed

---

## Medium Priority Findings — Responses

### MP-1: Import/Export Semantics

**Decision:** Define explicit policy.

**Import Policy:**

```
IMPORT FROM HOST:

1. Filename encoding:
   - Attempt UTF-8 interpretation
   - Non-UTF-8 filenames: SKIP with warning (logged, not fatal)
   - Null bytes in names: REJECT

2. Symlinks on host:
   - Default: DO NOT FOLLOW (skip symlinks)
   - Option: follow_symlinks: bool (if true, import target content)

3. Special files (devices, FIFOs, sockets):
   - SKIP (not supported in virtual filesystem)

4. Metadata preservation:
   - Timestamps: PRESERVE if container has permissions enabled
   - Permissions: PRESERVE if container has permissions enabled
   - Extended attrs: SKIP (too platform-specific)

5. Limits during import:
   - Respect container's capacity limits
   - Fail fast if total_size would exceed max_total_size
   - max_path_depth applies to imported structure
```

**Export Policy:**

```
EXPORT TO HOST:

1. Filename encoding:
   - VirtualPath is always valid UTF-8
   - Host filesystem may reject certain characters (platform-specific)
   - On rejection: FAIL with clear error

2. Symlinks in container:
   - If symlinks enabled: CREATE host symlinks
   - If target doesn't exist in export: create dangling symlink

3. Metadata:
   - Timestamps: SET if source has them
   - Permissions: SET if source has them and host supports
   - Extended attrs: SKIP

4. Overwrites:
   - Default: ERROR if host path exists
   - Option: overwrite: bool
```

---

### MP-2: Extended Attributes Limits

**Decision:** Add limits and separate storage.

**Specification:**

```rust
pub struct CapacityLimits {
    // ... existing fields ...
    
    /// Maximum size of a single xattr value (default: 64 KB)
    pub max_xattr_value_size: Option<usize>,
    
    /// Maximum total xattr size per node (default: 1 MB)
    pub max_xattr_total_size: Option<usize>,
    
    /// Maximum number of xattrs per node (default: 100)
    pub max_xattr_count: Option<u32>,
}
```

**Storage decision:** Xattrs stored in separate table, NOT in NodeRecord.

```sql
CREATE TABLE xattrs (
    node_id INTEGER NOT NULL,
    name TEXT NOT NULL,
    value BLOB NOT NULL,
    PRIMARY KEY (node_id, name),
    FOREIGN KEY (node_id) REFERENCES nodes(id) ON DELETE CASCADE
);
```

**API:**
- `get_node()` does NOT return xattrs (lightweight)
- Separate methods: `get_xattr()`, `set_xattr()`, `list_xattrs()`, `remove_xattr()`

---

### MP-3: Document Numbering

**Decision:** Fix it.

**Actions:**
- [x] Renumber sections consistently
- [x] Update TOC anchors
- [x] Single pass through all docs before implementation

---

## SQLite Addendum — Response

**Decision:** Accept all recommendations.

### Go/No-Go Checklist:

| Requirement | Status |
|-------------|--------|
| Accept SQLite-first for persistence | ✅ Yes |
| Commit to schema versioning | ✅ Yes |
| Safe dashboard surface (read-only views) | ✅ Yes |
| Specify quota/hardlink/GC rules | ✅ Yes (see HP-3) |
| Keep filesystem invariants in FilesContainer | ✅ Yes |

### Schema Contract:

```sql
-- Schema version tracking
CREATE TABLE schema_meta (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
INSERT INTO schema_meta (key, value) VALUES ('version', '1');

-- Read-only views for external consumers (stable contract)
CREATE VIEW v_files AS
SELECT 
    n.id,
    -- reconstruct path via recursive CTE (complex but doable)
    n.size,
    n.created_at,
    n.modified_at
FROM nodes n
WHERE n.kind = 'file';

CREATE VIEW v_usage AS
SELECT 
    COUNT(*) as node_count,
    SUM(CASE WHEN kind = 'file' THEN size ELSE 0 END) as total_size,
    SUM(CASE WHEN kind = 'file' THEN 1 ELSE 0 END) as file_count,
    SUM(CASE WHEN kind = 'directory' THEN 1 ELSE 0 END) as dir_count
FROM nodes;
```

**Migration policy:**
- Forward-only migrations
- Breaking changes require major version bump
- Views are stable interface; internal tables may change

---

## Updated Implementation Plan

### Phase 1: Core Foundation (Week 1)
- `anyfs`: VirtualPath, Name, NodeId, ContentId, ChunkId
- `anyfs`: Error types, NodeRecord, Edge, NodeMetadata
- `anyfs`: StorageBackend, Snapshot, Transaction traits

### Phase 2: Memory Backend (Week 2)
- `vfs-memory`: MemoryBackend with clone-on-transact
- Conformance test suite (50+ tests)
- All tests passing for memory backend

### Phase 3: SQLite Backend (Weeks 3-4)
- `vfs-sqlite`: Schema v1 with WAL mode
- `vfs-sqlite`: SqliteBackend implementation
- Conformance tests passing
- Read-only views for dashboard access

### Phase 4: FilesContainer (Week 5)
- `vfs`: FilesContainer with all operations
- Path resolution, symlink following (if enabled)
- Rename cycle prevention
- Capacity limit enforcement

### Phase 5: Polish (Week 6)
- Import/export with host filesystem
- Documentation updates
- Example applications
- Publish to crates.io

---

## Acceptance Checklist (Updated)

- [ ] Single authoritative backend trait (CF-1) ✅ Decided
- [ ] FsBackend removed from MVP (CF-2) ✅ Decided
- [ ] Rename cycle prevention spec + tests (CF-3) ✅ Decided
- [ ] Quota accounting rules specified (HP-3) ✅ Decided
- [ ] Import/export policies documented (MP-1) ✅ Decided
- [ ] Xattr limits defined (MP-2) ✅ Decided
- [ ] Doc numbering fixed (MP-3) ⬜ Todo
- [ ] Conformance tests pass for SQLite + Memory ⬜ Implementation

---

## Summary

All critical and high-priority issues have been resolved with explicit decisions. The project is **approved to proceed** with implementation, following the revised plan:

1. **Graph-store trait** is authoritative
2. **SQLite + Memory** backends for MVP (no FsBackend)
3. **Rename cycle prevention** via ancestry check
4. **Logical bytes** for quota accounting
5. **Schema versioning** with stable read-only views
6. **Single-threaded** with external sync for multi-thread

The design is now internally consistent and ready for implementation.
