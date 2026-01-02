# Indexing Middleware (Design Plan)

**Status:** Accepted (Post-v1) — See ADR-031
**Scope:** Design plan only (no API break)

---

## Summary

Provide a consistent, queryable index of file activity and metadata for real filesystems (SQLite default). The index tracks operations (create, write, rename, delete) and maintains a catalog of files for fast queries and statistics. This enables workflows like "manage a flash drive and query every change" and "mount a drive and get an implicit audit trail."

**Direction:** Middleware-only. Indexing is a composable layer users opt into when they want a queryable catalog of file activity.

---

## Goals

- Preserve std::fs-style DX via `FileStorage` (no change to core traits).
- Track file operations with timestamps in a durable index (SQLite default).
- Provide fast queries (by path, prefix, mtime, size, hash).
- Keep index consistent for operations executed through AnyFS.
- Keep the design open to future index engines via a small trait (SQLite default).

---

## Non-Goals

- Full OS-level auditing outside AnyFS (requires kernel hooks).
- Mandatory hashing of all files (optional and expensive).
- Replacing `Tracing` (indexing is storage + query, tracing is instrumentation).

---

## Architecture (Middleware-Only)

### Indexing Middleware (Primary)

**Shape:** `Indexing<B>` where `B: Fs`

**Layer:** `IndexLayer` (builder-based, like `QuotaLayer`, `TracingLayer`)

**Behavior:** Intercepts operations and writes entries into an index (SQLite by default). Works on all backends (Memory, SQLite, VRootFs, custom). Guarantees apply only to operations that flow through AnyFS.

**Pros**
- Backend-agnostic.
- Useful for virtual backends too.
- No special-case OS behavior.

**Cons**
- External changes on real FS are not captured (unless a watcher/scan helper is added later).

---

## Where It Fits

| Use Case                                          | Recommended                                      |
| ------------------------------------------------- | ------------------------------------------------ |
| AnyFS app wants an audit trail                    | `Indexing<B>` middleware                         |
| Virtual backend needs queryable catalog           | `Indexing<B>` middleware                         |
| Real FS with external edits to track              | Indexing middleware + future watcher/scan helper |
| Mounted drive where all access goes through AnyFS | Indexing middleware (enough)                     |

---

## Consistency Model

- **Through AnyFS:** Strong consistency for index updates in strict mode.
- **External OS changes:** Not captured by default. A future watcher/scan helper can reconcile.

**Modes:**

```rust
enum IndexConsistency {
    Strict,      // If index update fails, return error from FS op
    BestEffort,  // FS op succeeds even if index update fails
}
```

---

## Index Schema (SQLite)

Minimal schema focused on query speed and durability:

```sql
CREATE TABLE IF NOT EXISTS nodes (
  path TEXT PRIMARY KEY,
  parent_path TEXT NOT NULL,
  file_type INTEGER NOT NULL,      -- 0=file, 1=dir, 2=symlink
  size INTEGER NOT NULL DEFAULT 0,
  inode INTEGER,
  nlink INTEGER,
  permissions INTEGER,
  created_at INTEGER,
  modified_at INTEGER,
  accessed_at INTEGER,
  symlink_target TEXT,
  hash BLOB,                        -- optional
  exists INTEGER NOT NULL DEFAULT 1,
  last_seen_at INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_nodes_parent ON nodes(parent_path);
CREATE INDEX IF NOT EXISTS idx_nodes_mtime ON nodes(modified_at);
CREATE INDEX IF NOT EXISTS idx_nodes_hash ON nodes(hash);

CREATE TABLE IF NOT EXISTS ops (
  id INTEGER PRIMARY KEY,
  ts INTEGER NOT NULL,
  op TEXT NOT NULL,                 -- "write", "rename", "remove", ...
  path TEXT,
  path_to TEXT,
  bytes INTEGER,
  status TEXT NOT NULL,             -- "ok" | "err"
  error TEXT
);

CREATE TABLE IF NOT EXISTS config (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

**Path normalization:** Store virtual paths (what the user sees), not host paths. For host FS, optionally store host paths in a separate table if needed.

---

## Operation Mapping

| Operation                      | Index Update                                 |
| ------------------------------ | -------------------------------------------- |
| `write`, `append`, `truncate`  | Upsert node, update size/mtime, log op       |
| `create_dir`, `create_dir_all` | Insert dir nodes, log op                     |
| `remove_file`                  | Mark `exists=0`, log op                      |
| `remove_dir`, `remove_dir_all` | Mark subtree removed (prefix query), log op  |
| `rename`                       | Update path + parent for subtree, log op     |
| `copy`                         | Insert new node from source metadata, log op |
| `symlink`, `hard_link`         | Insert node, set link metadata, log op       |
| `read`/`read_range`            | Optional op log only (configurable)          |

**Streaming writes:** Wrap `open_write()` with a counting writer that records final size and timestamps on close.

---

## Configuration

```rust
pub struct IndexConfig {
    pub index_file: PathBuf,            // sidecar index file (SQLite default)
    pub consistency: IndexConsistency,
    pub track_reads: bool,
    pub track_errors: bool,
    pub track_metadata: bool,
    pub content_hashing: ContentHashing, // None | OnWrite | OnDemand
    pub initial_scan: InitialScan,       // None | OnDemand | FullScan
}
```

### Naming

- Middleware type: `Indexing<B>`
- Layer: `IndexLayer`
- Builder methods emphasize intent: `index_file`, `consistency`, `track_*`, `content_hashing`, `initial_scan`

### Example Configuration

```rust
let backend = MemoryBackend::new()
    .layer(IndexLayer::builder()
        .index_file("index.db")
        .consistency(IndexConsistency::Strict)
        .track_reads(false)
        .build());
```

### Index Engine Abstraction (Future)

To keep the middleware ergonomic while enabling alternate engines, define a small storage trait and keep SQLite as the default implementation:

```rust
pub trait IndexStore: Send + Sync {
    fn upsert_node(&self, node: IndexNode) -> Result<(), IndexError>;
    fn mark_removed(&self, path: &Path) -> Result<(), IndexError>;
    fn rename_prefix(&self, from: &Path, to: &Path) -> Result<(), IndexError>;
    fn record_op(&self, entry: OpEntry) -> Result<(), IndexError>;
}

```

The default implementation uses SQLite at `index_file`. If/when alternate engines are needed, the `IndexLayer` builder can accept a boxed `IndexStore` for advanced use without introducing an enum.

---

## Performance Notes

- Use WAL mode for concurrency.
- Batch updates for recursive operations (rename/remove_dir_all).
- Hashing is optional and should be off by default.
- Keep op logs bounded (optional retention policy).

---

## Security and Containment

- Index file should live **outside** the root path by default.
- For mounted drives, use a dedicated index path per mount.
- Respect `PathFilter` and `Restrictions` when operating through middleware.

---

## Mounting Scenario

When mounted via the planned `anyfs-mount` companion crate, all access goes through AnyFS. The index becomes an implicit audit trail:

- Every file operation is logged.
- Queries reflect all operations routed through AnyFS.

---

## Open Questions

- Should op logs be bounded by size/time by default?
- Do we need a query API in `anyfs` or a separate `anyfs-index` crate?
- Should middleware expose a read-only `IndexStore` handle for queries?
- Should we add a companion watcher/scan tool for external changes on real FS?
