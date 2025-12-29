# Hybrid Backend Design

**SQLite Metadata + Content-Addressed Blob Storage**

This document validates that AnyFS can support hybrid backends and provides implementation patterns for this advanced use case.

---

## Overview

A **hybrid backend** separates:
- **Metadata** (directory structure, inodes, permissions) → SQLite
- **Content** (file bytes) → Content-Addressed Storage (CAS)

```
┌─────────────────────────────────────────────────────────┐
│                    HybridBackend                        │
│  ┌─────────────────────┐    ┌────────────────────────┐  │
│  │   SQLite Metadata   │    │   Blob Store (CAS)     │  │
│  │                     │    │                        │  │
│  │  - inodes           │    │  - content-addressed   │  │
│  │  - dir_entries      │    │  - deduplicated        │  │
│  │  - blob references  │    │  - S3, local, etc.     │  │
│  │  - audit log        │    │                        │  │
│  └─────────────────────┘    └────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Why this pattern?**
- SQLite is great for metadata queries (directory listings, stats, audit)
- Blob stores scale better for large file content
- Content-addressing enables deduplication
- Separating concerns enables independent scaling

---

## Framework Validation

### Do Current Traits Support This?

**Yes.** The `Fs` traits define *operations*, not *storage implementation*.

| Trait Method | Hybrid Implementation |
|--------------|----------------------|
| `read(path)` | SQLite lookup → blob fetch |
| `write(path, data)` | Blob upload → SQLite update |
| `metadata(path)` | SQLite query only |
| `read_dir(path)` | SQLite query only |
| `remove_file(path)` | SQLite update (refcount--) |
| `rename(from, to)` | SQLite update only |
| `copy(from, to)` | SQLite update (refcount++) |

The traits don't care where bytes come from - that's the backend's business.

### Thread Safety

Current design requires `&self` methods with interior mutability. For hybrid:

```rust
pub struct HybridBackend {
    // SQLite needs single-writer (see "Write Queue" below)
    metadata: Arc<Mutex<Connection>>,

    // Blob store is typically already thread-safe
    blobs: Arc<dyn BlobStore>,

    // Write queue for serializing SQLite writes
    write_tx: mpsc::Sender<WriteCmd>,
}
```

This aligns with ADR-023 (interior mutability).

---

## Data Model

### SQLite Schema

```sql
-- Inode table (one row per file/directory/symlink)
CREATE TABLE nodes (
    inode       INTEGER PRIMARY KEY,
    parent      INTEGER NOT NULL,
    name        TEXT NOT NULL,
    node_type   TEXT NOT NULL,  -- 'file', 'dir', 'symlink'
    size        INTEGER NOT NULL DEFAULT 0,
    mode        INTEGER NOT NULL DEFAULT 420,  -- 0o644
    nlink       INTEGER NOT NULL DEFAULT 1,
    blob_id     TEXT,           -- NULL for directories
    symlink_target TEXT,        -- NULL unless symlink
    created_at  INTEGER NOT NULL,
    modified_at INTEGER NOT NULL,
    accessed_at INTEGER NOT NULL,

    UNIQUE(parent, name)
);

-- Root directory (inode 1)
INSERT INTO nodes (inode, parent, name, node_type, size, mode, created_at, modified_at, accessed_at)
VALUES (1, 1, '', 'dir', 0, 493, strftime('%s', 'now'), strftime('%s', 'now'), strftime('%s', 'now'));

-- Blob reference tracking (for dedup + GC)
CREATE TABLE blobs (
    blob_id     TEXT PRIMARY KEY,  -- sha256 hex
    size        INTEGER NOT NULL,
    refcount    INTEGER NOT NULL DEFAULT 0,
    created_at  INTEGER NOT NULL
);

-- Audit log (optional but recommended)
CREATE TABLE audit (
    seq         INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   INTEGER NOT NULL,
    operation   TEXT NOT NULL,
    path        TEXT,
    actor       TEXT,
    details     TEXT  -- JSON
);

-- Indexes
CREATE INDEX idx_nodes_parent ON nodes(parent);
CREATE INDEX idx_nodes_blob ON nodes(blob_id) WHERE blob_id IS NOT NULL;
CREATE INDEX idx_blobs_refcount ON blobs(refcount) WHERE refcount = 0;
```

### Blob Store Interface

```rust
/// Content-addressed blob storage.
pub trait BlobStore: Send + Sync {
    /// Store bytes, returns content hash (blob_id).
    fn put(&self, data: &[u8]) -> Result<String, BlobError>;

    /// Retrieve bytes by content hash.
    fn get(&self, blob_id: &str) -> Result<Vec<u8>, BlobError>;

    /// Check if blob exists.
    fn exists(&self, blob_id: &str) -> Result<bool, BlobError>;

    /// Delete blob (only call after refcount reaches 0).
    fn delete(&self, blob_id: &str) -> Result<(), BlobError>;

    /// Streaming read for large files.
    fn open_read(&self, blob_id: &str) -> Result<Box<dyn Read + Send>, BlobError>;

    /// Streaming write, returns blob_id on completion.
    fn open_write(&self) -> Result<Box<dyn BlobWriter>, BlobError>;
}

pub trait BlobWriter: Write + Send {
    /// Finalize the blob and return its content hash.
    fn finalize(self: Box<Self>) -> Result<String, BlobError>;
}
```

Implementations could be:
- `LocalCasBackend` - local directory with content-addressed files
- `S3BlobStore` - S3-compatible object storage
- `MemoryBlobStore` - in-memory for testing

---

## Implementation Sketch

### Core Structure

```rust
use anyfs_backend::{FsRead, FsWrite, FsDir, FsError, Metadata, ReadDirIter, DirEntry, FileType};
use rusqlite::Connection;
use std::sync::{Arc, Mutex};
use std::path::{Path, PathBuf};
use tokio::sync::mpsc;

pub struct HybridBackend {
    /// SQLite connection (metadata)
    db: Arc<Mutex<Connection>>,

    /// Content-addressed blob storage
    blobs: Arc<dyn BlobStore>,

    /// Write command queue (single-writer pattern)
    write_tx: mpsc::UnboundedSender<WriteCmd>,

    /// Background writer handle
    _writer_handle: Arc<WriterHandle>,
}

enum WriteCmd {
    Write {
        path: PathBuf,
        blob_id: String,
        size: u64,
        reply: oneshot::Sender<Result<(), FsError>>,
    },
    Remove {
        path: PathBuf,
        reply: oneshot::Sender<Result<(), FsError>>,
    },
    CreateDir {
        path: PathBuf,
        reply: oneshot::Sender<Result<(), FsError>>,
    },
    // ... other write operations
}
```

### Read Operations (Direct)

Read operations can query SQLite and blob store directly (no queue needed):

```rust
impl FsRead for HybridBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();

        // 1. Query SQLite for blob_id
        let db = self.db.lock().map_err(|_| FsError::Backend("lock poisoned".into()))?;

        let (blob_id, node_type): (Option<String>, String) = db.query_row(
            "SELECT blob_id, node_type FROM nodes WHERE inode = (
                SELECT inode FROM nodes WHERE parent = ? AND name = ?
            )",
            // ... path resolution params
            |row| Ok((row.get(0)?, row.get(1)?)),
        ).map_err(|_| FsError::NotFound { path: path.to_path_buf() })?;

        if node_type != "file" {
            return Err(FsError::NotAFile { path: path.to_path_buf() });
        }

        let blob_id = blob_id.ok_or_else(|| FsError::NotFound { path: path.to_path_buf() })?;

        drop(db);  // Release lock before blob fetch

        // 2. Fetch from blob store
        self.blobs.get(&blob_id)
            .map_err(|e| FsError::Backend(e.to_string()))
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        let path = path.as_ref();
        let db = self.db.lock().map_err(|_| FsError::Backend("lock poisoned".into()))?;

        // Pure SQLite query
        let exists: bool = db.query_row(
            "SELECT EXISTS(SELECT 1 FROM nodes WHERE parent = ? AND name = ?)",
            // ... params
            |row| row.get(0),
        ).unwrap_or(false);

        Ok(exists)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        let path = path.as_ref();
        let db = self.db.lock().map_err(|_| FsError::Backend("lock poisoned".into()))?;

        // Pure SQLite query - no blob store needed
        db.query_row(
            "SELECT node_type, size, mode, nlink, created_at, modified_at, accessed_at, inode
             FROM nodes WHERE parent = ? AND name = ?",
            // ... params
            |row| {
                let node_type: String = row.get(0)?;
                Ok(Metadata {
                    file_type: match node_type.as_str() {
                        "file" => FileType::File,
                        "dir" => FileType::Directory,
                        "symlink" => FileType::Symlink,
                        _ => FileType::File,
                    },
                    size: row.get(1)?,
                    permissions: Some(row.get(2)?),
                    // ... other fields
                })
            },
        ).map_err(|_| FsError::NotFound { path: path.to_path_buf() })
    }

    // ... other FsRead methods
}
```

### Write Operations (Two-Phase Commit)

Writes use a two-phase pattern: upload blob first, then commit SQLite:

```rust
impl FsWrite for HybridBackend {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();

        // Phase 1: Upload blob (can fail independently)
        let blob_id = self.blobs.put(data)
            .map_err(|e| FsError::Backend(format!("blob upload failed: {}", e)))?;

        // Phase 2: Commit metadata (via write queue)
        let (tx, rx) = oneshot::channel();

        self.write_tx.send(WriteCmd::Write {
            path,
            blob_id,
            size: data.len() as u64,
            reply: tx,
        }).map_err(|_| FsError::Backend("write queue closed".into()))?;

        // Wait for SQLite commit
        rx.blocking_recv()
            .map_err(|_| FsError::Backend("write cancelled".into()))?
    }

    fn remove_file(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();

        // Queue the removal (blob cleanup happens in background via GC)
        let (tx, rx) = oneshot::channel();

        self.write_tx.send(WriteCmd::Remove { path, reply: tx })
            .map_err(|_| FsError::Backend("write queue closed".into()))?;

        rx.blocking_recv()
            .map_err(|_| FsError::Backend("remove cancelled".into()))?
    }

    fn copy(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError> {
        // Copy is just a metadata operation - increment refcount, no blob copy!
        let (tx, rx) = oneshot::channel();

        self.write_tx.send(WriteCmd::Copy {
            from: from.as_ref().to_path_buf(),
            to: to.as_ref().to_path_buf(),
            reply: tx,
        }).map_err(|_| FsError::Backend("write queue closed".into()))?;

        rx.blocking_recv()
            .map_err(|_| FsError::Backend("copy cancelled".into()))?
    }

    // ... other FsWrite methods
}
```

### Write Queue Worker

The single-writer pattern for SQLite:

```rust
async fn write_worker(
    db: Arc<Mutex<Connection>>,
    blobs: Arc<dyn BlobStore>,
    mut rx: mpsc::UnboundedReceiver<WriteCmd>,
) {
    while let Some(cmd) = rx.recv().await {
        let result = {
            let mut db = db.lock().unwrap();

            match cmd {
                WriteCmd::Write { path, blob_id, size, reply } => {
                    let result = db.execute_batch(&format!(r#"
                        BEGIN;

                        -- Upsert blob record
                        INSERT INTO blobs (blob_id, size, refcount, created_at)
                        VALUES ('{blob_id}', {size}, 1, strftime('%s', 'now'))
                        ON CONFLICT(blob_id) DO UPDATE SET refcount = refcount + 1;

                        -- Update or insert node
                        -- (simplified - real impl needs path resolution)

                        -- Audit log
                        INSERT INTO audit (timestamp, operation, path)
                        VALUES (strftime('%s', 'now'), 'write', '{path}');

                        COMMIT;
                    "#));

                    let _ = reply.send(result.map_err(|e| FsError::Backend(e.to_string())));
                }

                WriteCmd::Remove { path, reply } => {
                    // Decrement refcount (GC cleans up when refcount = 0)
                    let result = db.execute_batch(&format!(r#"
                        BEGIN;

                        -- Get blob_id before delete
                        -- Decrement refcount
                        -- Remove node
                        -- Audit log

                        COMMIT;
                    "#));

                    let _ = reply.send(result.map_err(|e| FsError::Backend(e.to_string())));
                }

                // ... other commands
            }
        };
    }
}
```

---

## Deduplication

Content-addressing gives you dedup for free:

```rust
impl BlobStore for LocalCasBackend {
    fn put(&self, data: &[u8]) -> Result<String, BlobError> {
        // Hash the content
        let hash = sha256(data);
        let blob_id = hex::encode(hash);

        // Check if already exists
        let blob_path = self.root.join(&blob_id[0..2]).join(&blob_id);

        if blob_path.exists() {
            // Already have this content - dedup!
            return Ok(blob_id);
        }

        // Store new blob
        std::fs::create_dir_all(blob_path.parent().unwrap())?;
        std::fs::write(&blob_path, data)?;

        Ok(blob_id)
    }
}
```

**Dedup in action:**
- User A writes `report.pdf` (10 MB) → blob `abc123`, refcount = 1
- User B writes identical `report.pdf` → same blob `abc123`, refcount = 2
- Physical storage: 10 MB (not 20 MB)

### Refcount Management

```sql
-- On file write (new reference to blob)
UPDATE blobs SET refcount = refcount + 1 WHERE blob_id = ?;

-- On file delete
UPDATE blobs SET refcount = refcount - 1 WHERE blob_id = ?;

-- On copy (no blob copy needed!)
UPDATE blobs SET refcount = refcount + 1 WHERE blob_id = ?;
```

---

## Garbage Collection

Blobs with `refcount = 0` are orphans and can be deleted:

```rust
impl HybridBackend {
    /// Run garbage collection (call periodically or on-demand).
    pub fn gc(&self) -> Result<GcStats, FsError> {
        let db = self.db.lock().map_err(|_| FsError::Backend("lock".into()))?;

        // Find orphaned blobs
        let orphans: Vec<String> = db.prepare(
            "SELECT blob_id FROM blobs WHERE refcount = 0"
        )?.query_map([], |row| row.get(0))?
          .filter_map(|r| r.ok())
          .collect();

        drop(db);

        // Delete from blob store
        let mut deleted = 0;
        for blob_id in &orphans {
            if self.blobs.delete(blob_id).is_ok() {
                deleted += 1;
            }
        }

        // Remove from SQLite
        let db = self.db.lock().unwrap();
        db.execute(
            "DELETE FROM blobs WHERE refcount = 0",
            [],
        )?;

        Ok(GcStats { orphans_found: orphans.len(), blobs_deleted: deleted })
    }
}
```

**GC Safety:**
- Never delete blobs referenced by snapshots
- Add `snapshot_refs` table or use `refcount` that includes snapshot references
- Run GC in background, not during writes

---

## Snapshots and Backup

### Creating a Snapshot

```rust
impl HybridBackend {
    /// Create a point-in-time snapshot.
    pub fn snapshot(&self, name: &str) -> Result<SnapshotId, FsError> {
        let db = self.db.lock().unwrap();

        db.execute_batch(&format!(r#"
            BEGIN;

            -- Record snapshot
            INSERT INTO snapshots (name, created_at, root_manifest)
            VALUES ('{name}', strftime('%s', 'now'),
                    (SELECT json_group_array(blob_id) FROM blobs WHERE refcount > 0));

            -- Pin all current blobs (prevent GC)
            UPDATE blobs SET refcount = refcount + 1
            WHERE blob_id IN (SELECT blob_id FROM nodes WHERE blob_id IS NOT NULL);

            COMMIT;
        "#))?;

        Ok(SnapshotId(name.to_string()))
    }

    /// Export as single portable artifact.
    pub fn export(&self, dest: impl AsRef<Path>) -> Result<(), FsError> {
        // 1. SQLite backup API for metadata
        let db = self.db.lock().unwrap();
        let backup_db = Connection::open(dest.as_ref().join("metadata.db"))?;
        db.backup(rusqlite::DatabaseName::Main, &backup_db, None)?;

        // 2. Copy referenced blobs
        let blob_ids: Vec<String> = db.prepare(
            "SELECT DISTINCT blob_id FROM nodes WHERE blob_id IS NOT NULL"
        )?.query_map([], |row| row.get(0))?
          .filter_map(|r| r.ok())
          .collect();

        drop(db);

        let blobs_dir = dest.as_ref().join("blobs");
        std::fs::create_dir_all(&blobs_dir)?;

        for blob_id in blob_ids {
            let data = self.blobs.get(&blob_id)?;
            std::fs::write(blobs_dir.join(&blob_id), data)?;
        }

        Ok(())
    }
}
```

---

## Middleware Integration

Middleware works unchanged - it wraps the hybrid backend like any other:

```rust
use anyfs::{FileStorage, QuotaLayer, TracingLayer, RestrictionsLayer};

let backend = HybridBackend::open("drive.db", LocalCasBackend::new("./blobs"))?;

// Standard middleware stack
let backend = backend
    .layer(QuotaLayer::builder()
        .max_total_size(50 * 1024 * 1024 * 1024)  // 50 GB
        .build())
    .layer(RestrictionsLayer::builder()
        .deny_symlinks()
        .build())
    .layer(TracingLayer::new());

let fs = FileStorage::new(backend);

// Use like any other filesystem
fs.write("/documents/report.pdf", &pdf_bytes)?;
```

**Quota tracking note:** `QuotaLayer` tracks logical size (what users see), not physical size (with dedup). For physical tracking, the backend could expose `physical_usage()` separately.

---

## Async Considerations

The hybrid pattern benefits significantly from async (ADR-024):

| Operation | Sync Pain | Async Benefit |
|-----------|-----------|---------------|
| Blob upload to S3 | Blocks thread | Concurrent uploads |
| Multiple reads | Sequential | Parallel fetches |
| Write queue | `blocking_recv()` | Native async channel |
| GC | Blocks all ops | Background task |

When `AsyncFs` traits exist (ADR-024), the hybrid backend can use them naturally:

```rust
#[async_trait]
impl AsyncFsRead for HybridBackend {
    async fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let blob_id = self.lookup_blob_id(path).await?;
        self.blobs.get_async(&blob_id).await  // Non-blocking!
    }
}
```

---

## Identified Gaps

Areas where the current framework could be enhanced:

| Gap | Current State | Recommendation |
|-----|---------------|----------------|
| Two-phase commit pattern | Not documented | Add to backend guide |
| Refcount/GC patterns | Not documented | Add section |
| Streaming large files | `open_read`/`open_write` exist | Document chunked patterns |
| Physical vs logical size | Quota tracks logical only | Consider `PhysicalStats` trait |
| Background tasks (GC) | No pattern | Document spawn pattern |

---

## Summary

**Framework validation: PASSED**

The current AnyFS trait design supports hybrid backends:
- Traits define operations, not storage
- Interior mutability allows single-writer patterns
- Middleware composes unchanged
- Async strategy (ADR-024) enhances this pattern

**Key patterns for hybrid backends:**
1. Single-writer queue for SQLite
2. Two-phase commit (blob upload → SQLite commit)
3. Content-addressing for dedup
4. Refcounting for GC safety
5. Snapshot pinning for backup safety

This validates that AnyFS is flexible enough for advanced storage architectures while maintaining its simple middleware composition model.
