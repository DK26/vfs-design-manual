# SQLite Operational Guide

**Production-ready SQLite for filesystem backends**

This guide covers everything you need to run SQLite-backed storage at scale.

---

## Overview

SQLite is an excellent choice for filesystem backends:
- Single-file deployment (portable, easy backup)
- ACID transactions (data integrity)
- Rich query capabilities (dashboards, analytics)
- Proven at scale (handles terabytes)

But it has specific requirements for concurrent access that you must understand.

---

## The Golden Rule: Single Writer

**SQLite supports many readers but only ONE writer at a time.**

Even in WAL mode, concurrent writes will block. This isn't a bug - it's a design choice that enables SQLite's reliability.

### The Write Queue Pattern

For filesystem backends, use a single-writer queue:

```rust
use tokio::sync::mpsc;
use rusqlite::Connection;

pub struct SqliteBackend {
    /// Read-only connection pool (many readers OK)
    read_pool: Pool<Connection>,

    /// Write commands go through this channel
    write_tx: mpsc::UnboundedSender<WriteCmd>,
}

enum WriteCmd {
    Write { path: PathBuf, data: Vec<u8>, reply: oneshot::Sender<Result<(), FsError>> },
    Remove { path: PathBuf, reply: oneshot::Sender<Result<(), FsError>> },
    Rename { from: PathBuf, to: PathBuf, reply: oneshot::Sender<Result<(), FsError>> },
    CreateDir { path: PathBuf, reply: oneshot::Sender<Result<(), FsError>> },
    // ...
}

// Single writer task
async fn writer_loop(conn: Connection, mut rx: mpsc::UnboundedReceiver<WriteCmd>) {
    while let Some(cmd) = rx.recv().await {
        let result = match cmd {
            WriteCmd::Write { path, data, reply } => {
                let r = execute_write(&conn, &path, &data);
                let _ = reply.send(r);
            }
            // ... handle other commands
        };
    }
}
```

**Why this works:**
- No `SQLITE_BUSY` errors (single writer = no contention)
- Predictable latency (queue depth = backpressure)
- Natural batching opportunity (combine multiple ops per transaction)
- Clean audit logging (all writes go through one place)

---

## WAL Mode (Required)

**Always enable WAL (Write-Ahead Logging) mode for concurrent access.**

```rust
fn open_connection(path: &Path) -> Result<Connection, rusqlite::Error> {
    let conn = Connection::open(path)?;

    // Enable WAL mode (persistent - only need to set once per database)
    conn.execute_batch("
        PRAGMA journal_mode = WAL;
        PRAGMA synchronous = NORMAL;
        PRAGMA cache_size = -64000;  -- 64MB cache
        PRAGMA temp_store = MEMORY;
    ")?;

    Ok(conn)
}
```

### WAL vs Rollback Journal

| Aspect | WAL Mode | Rollback Journal |
|--------|----------|------------------|
| Concurrent reads during write | ✅ Yes | ❌ No (blocked) |
| Read performance | Faster | Slower |
| Write performance | Similar | Similar |
| File count | 3 files (.db, .wal, .shm) | 1-2 files |
| Crash recovery | Automatic | Automatic |

**Always use WAL for filesystem backends.**

### WAL Checkpointing

WAL files grow until checkpointed. SQLite auto-checkpoints at 1000 pages, but you can control this:

```rust
// Manual checkpoint (call periodically or after bulk operations)
conn.execute_batch("PRAGMA wal_checkpoint(TRUNCATE);")?;

// Or configure auto-checkpoint threshold
conn.execute_batch("PRAGMA wal_autocheckpoint = 1000;")?;  // pages
```

**Checkpoint modes:**
- `PASSIVE` - Checkpoint without blocking writers (may not complete)
- `FULL` - Wait for writers, then checkpoint completely
- `RESTART` - Like FULL, but also resets WAL file
- `TRUNCATE` - Like RESTART, but truncates WAL to zero bytes

For filesystem backends, run `TRUNCATE` checkpoint:
- During quiet periods
- After bulk imports
- Before backups

---

## Busy Handling

Even with a write queue, reads might briefly block during checkpoints. Handle this gracefully:

```rust
fn open_connection(path: &Path) -> Result<Connection, rusqlite::Error> {
    let conn = Connection::open(path)?;

    // Wait up to 30 seconds if database is busy
    conn.busy_timeout(Duration::from_secs(30))?;

    // Or use a custom busy handler
    conn.busy_handler(Some(|attempts| {
        if attempts > 100 {
            false  // Give up after 100 retries
        } else {
            std::thread::sleep(Duration::from_millis(10 * attempts as u64));
            true   // Keep trying
        }
    }))?;

    Ok(conn)
}
```

**Never let `SQLITE_BUSY` propagate to users** - it's a transient condition.

---

## Connection Pooling

For read operations, use a connection pool:

```rust
use r2d2::{Pool, PooledConnection};
use r2d2_sqlite::SqliteConnectionManager;

pub struct SqliteBackend {
    read_pool: Pool<SqliteConnectionManager>,
    write_tx: mpsc::UnboundedSender<WriteCmd>,
}

impl SqliteBackend {
    pub fn open(path: impl AsRef<Path>) -> Result<Self, FsError> {
        let manager = SqliteConnectionManager::file(path.as_ref())
            .with_flags(rusqlite::OpenFlags::SQLITE_OPEN_READ_ONLY);

        let read_pool = Pool::builder()
            .max_size(10)  // 10 concurrent readers
            .build(manager)
            .map_err(|e| FsError::Backend(e.to_string()))?;

        // ... set up write queue

        Ok(Self { read_pool, write_tx })
    }
}

impl FsRead for SqliteBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let conn = self.read_pool.get()
            .map_err(|e| FsError::Backend(format!("pool exhausted: {}", e)))?;

        // Use read-only connection
        query_file_content(&conn, path.as_ref())
    }
}
```

**Pool sizing:**
- Start with `max_size = CPU cores * 2`
- Monitor pool exhaustion
- Increase if reads queue up

---

## Vacuum and Maintenance

SQLite doesn't automatically reclaim space from deleted data. You need `VACUUM`.

### Auto-Vacuum (Recommended)

Enable incremental auto-vacuum for gradual space reclamation:

```sql
-- Set once when creating the database
PRAGMA auto_vacuum = INCREMENTAL;

-- Then periodically run (e.g., daily or after large deletes)
PRAGMA incremental_vacuum(1000);  -- Free up to 1000 pages
```

### Manual Vacuum

Full vacuum rebuilds the entire database (expensive but thorough):

```rust
impl SqliteBackend {
    /// Compact the database. Call during maintenance windows.
    pub fn vacuum(&self) -> Result<(), FsError> {
        // Vacuum needs exclusive access - pause writes
        let conn = self.get_write_connection()?;

        // This can take a long time for large databases
        conn.execute_batch("VACUUM;")?;

        Ok(())
    }
}
```

**When to vacuum:**
- After deleting >25% of data
- After schema migrations
- During scheduled maintenance
- Never during peak usage

### Integrity Check

Periodically verify database integrity:

```rust
impl SqliteBackend {
    pub fn check_integrity(&self) -> Result<bool, FsError> {
        let conn = self.read_pool.get()?;

        let result: String = conn.query_row(
            "PRAGMA integrity_check;",
            [],
            |row| row.get(0),
        )?;

        Ok(result == "ok")
    }
}
```

Run integrity checks:
- After crash recovery
- Before backups
- Periodically (weekly/monthly)

---

## Schema Migrations

Filesystem schemas evolve. Handle migrations properly:

### Version Tracking

```sql
-- Store schema version in the database
CREATE TABLE IF NOT EXISTS meta (
    key TEXT PRIMARY KEY,
    value TEXT
);

INSERT OR REPLACE INTO meta (key, value) VALUES ('schema_version', '1');
```

### Migration Pattern

```rust
const CURRENT_VERSION: i32 = 3;

impl SqliteBackend {
    fn migrate(&self, conn: &Connection) -> Result<(), FsError> {
        let version: i32 = conn.query_row(
            "SELECT COALESCE((SELECT value FROM meta WHERE key = 'schema_version'), '0')",
            [],
            |row| row.get::<_, String>(0)?.parse().map_err(|_| rusqlite::Error::InvalidQuery),
        ).unwrap_or(0);

        if version < 1 {
            self.migrate_v0_to_v1(conn)?;
        }
        if version < 2 {
            self.migrate_v1_to_v2(conn)?;
        }
        if version < 3 {
            self.migrate_v2_to_v3(conn)?;
        }

        conn.execute(
            "INSERT OR REPLACE INTO meta (key, value) VALUES ('schema_version', ?)",
            [CURRENT_VERSION.to_string()],
        )?;

        Ok(())
    }

    fn migrate_v1_to_v2(&self, conn: &Connection) -> Result<(), FsError> {
        conn.execute_batch("
            -- Add new column
            ALTER TABLE nodes ADD COLUMN checksum TEXT;

            -- Backfill (expensive but necessary)
            -- UPDATE nodes SET checksum = compute_checksum(content) WHERE content IS NOT NULL;
        ")?;
        Ok(())
    }
}
```

**Migration rules:**
- Always wrap in transaction
- Test migrations on copy of production data
- Have rollback plan (backup before migration)
- Never delete columns in SQLite (not supported) - add new ones instead

---

## Backup Strategies

### Online Backup API (Recommended)

SQLite's backup API creates consistent snapshots of a live database:

```rust
use rusqlite::backup::{Backup, Progress};

impl SqliteBackend {
    /// Create a consistent backup while database is in use.
    pub fn backup(&self, dest_path: impl AsRef<Path>) -> Result<(), FsError> {
        let src = self.get_read_connection()?;
        let mut dest = Connection::open(dest_path.as_ref())?;

        let backup = Backup::new(&src, &mut dest)?;

        // Copy in chunks (allows progress reporting)
        loop {
            let more = backup.step(100)?;  // 100 pages at a time

            if !more {
                break;
            }

            // Optional: report progress
            let progress = backup.progress();
            println!("Backup: {}/{} pages", progress.pagecount - progress.remaining, progress.pagecount);
        }

        Ok(())
    }
}
```

**Benefits:**
- No downtime (backup while serving requests)
- Consistent snapshot (point-in-time)
- Can copy to any destination (file, memory, network)

### File Copy (Simple but Risky)

Only safe if database is not in use:

```rust
// DANGER: Only do this if no connections are open!
impl SqliteBackend {
    pub fn backup_cold(&self, dest: impl AsRef<Path>) -> Result<(), FsError> {
        // Ensure WAL is checkpointed first
        let conn = self.get_write_connection()?;
        conn.execute_batch("PRAGMA wal_checkpoint(TRUNCATE);")?;
        drop(conn);

        // Now safe to copy
        std::fs::copy(&self.db_path, dest.as_ref())?;
        Ok(())
    }
}
```

### Backup Schedule

| Scenario | Strategy |
|----------|----------|
| Development | Manual or none |
| Small production (<1GB) | Hourly online backup |
| Large production (>1GB) | Daily full + WAL archiving |
| Critical data | Continuous WAL shipping to replica |

---

## Performance Tuning

### Essential PRAGMAs

```sql
-- Performance-critical settings
PRAGMA journal_mode = WAL;           -- Required for concurrent access
PRAGMA synchronous = NORMAL;         -- Good balance of safety/speed
PRAGMA cache_size = -64000;          -- 64MB cache (negative = KB)
PRAGMA temp_store = MEMORY;          -- Temp tables in memory
PRAGMA mmap_size = 268435456;        -- 256MB memory-mapped I/O

-- For read-heavy workloads
PRAGMA read_uncommitted = ON;        -- Allow dirty reads (faster, use carefully)

-- For write-heavy workloads
PRAGMA wal_autocheckpoint = 10000;   -- Checkpoint less frequently
```

### Indexing Strategy

```sql
-- Essential indexes for filesystem operations
CREATE INDEX idx_nodes_parent ON nodes(parent_inode);
CREATE INDEX idx_nodes_name ON nodes(parent_inode, name);

-- For metadata queries
CREATE INDEX idx_nodes_type ON nodes(node_type);
CREATE INDEX idx_nodes_modified ON nodes(modified_at);

-- For GC queries
CREATE INDEX idx_blobs_orphan ON blobs(refcount) WHERE refcount = 0;

-- Composite indexes for common queries
CREATE INDEX idx_nodes_parent_type ON nodes(parent_inode, node_type);
```

### Query Optimization

```rust
// BAD: Multiple queries
fn get_children_with_metadata(parent: i64) -> Vec<Node> {
    let children = query("SELECT * FROM nodes WHERE parent = ?", [parent]);
    for child in children {
        let metadata = query("SELECT * FROM metadata WHERE inode = ?", [child.inode]);
        // ...
    }
}

// GOOD: Single query with JOIN
fn get_children_with_metadata(parent: i64) -> Vec<Node> {
    query("
        SELECT n.*, m.*
        FROM nodes n
        LEFT JOIN metadata m ON n.inode = m.inode
        WHERE n.parent = ?
    ", [parent])
}
```

### Prepared Statements

**Always use prepared statements for repeated queries:**

```rust
impl SqliteBackend {
    fn prepare_statements(conn: &Connection) -> Statements {
        Statements {
            read_file: conn.prepare_cached(
                "SELECT content FROM nodes WHERE parent_inode = ? AND name = ?"
            ).unwrap(),

            list_dir: conn.prepare_cached(
                "SELECT name, node_type, size FROM nodes WHERE parent_inode = ?"
            ).unwrap(),

            // ... other common queries
        }
    }
}
```

---

## Monitoring and Diagnostics

### Key Metrics to Track

```rust
impl SqliteBackend {
    pub fn stats(&self) -> DbStats {
        let conn = self.read_pool.get().unwrap();

        DbStats {
            // Database size
            page_count: pragma_i64(&conn, "page_count"),
            page_size: pragma_i64(&conn, "page_size"),

            // WAL status
            wal_pages: pragma_i64(&conn, "wal_checkpoint"),

            // Cache efficiency
            cache_hit: pragma_i64(&conn, "cache_hit"),
            cache_miss: pragma_i64(&conn, "cache_miss"),

            // Fragmentation
            freelist_count: pragma_i64(&conn, "freelist_count"),
        }
    }
}

fn pragma_i64(conn: &Connection, name: &str) -> i64 {
    conn.query_row(&format!("PRAGMA {}", name), [], |r| r.get(0)).unwrap_or(0)
}
```

### Health Checks

```rust
impl SqliteBackend {
    pub fn health_check(&self) -> HealthStatus {
        // 1. Can we connect?
        let conn = match self.read_pool.get() {
            Ok(c) => c,
            Err(e) => return HealthStatus::Unhealthy(format!("pool: {}", e)),
        };

        // 2. Is database intact?
        let integrity: String = conn.query_row("PRAGMA integrity_check", [], |r| r.get(0))
            .unwrap_or_else(|_| "error".to_string());

        if integrity != "ok" {
            return HealthStatus::Unhealthy(format!("integrity: {}", integrity));
        }

        // 3. Is WAL file reasonable size?
        let wal_size = std::fs::metadata(format!("{}-wal", self.db_path))
            .map(|m| m.len())
            .unwrap_or(0);

        if wal_size > 100 * 1024 * 1024 {  // > 100MB
            return HealthStatus::Degraded("WAL file large - checkpoint needed".into());
        }

        // 4. Is write queue backed up?
        if self.write_queue_depth() > 1000 {
            return HealthStatus::Degraded("Write queue backlog".into());
        }

        HealthStatus::Healthy
    }
}
```

---

## Common Pitfalls

### 1. Opening Too Many Connections

```rust
// BAD: New connection per operation
fn read(&self, path: &Path) -> Vec<u8> {
    let conn = Connection::open(&self.db_path).unwrap();  // DON'T
    // ...
}

// GOOD: Use connection pool
fn read(&self, path: &Path) -> Vec<u8> {
    let conn = self.pool.get().unwrap();  // Reuse connections
    // ...
}
```

### 2. Long-Running Transactions

```rust
// BAD: Transaction open while doing slow work
let tx = conn.transaction()?;
for file in files {
    tx.execute("INSERT ...", [&file])?;
    upload_to_s3(&file)?;  // SLOW - blocks other writers!
}
tx.commit()?;

// GOOD: Minimize transaction scope
for file in files {
    upload_to_s3(&file)?;  // Do slow work outside transaction
}
let tx = conn.transaction()?;
for file in files {
    tx.execute("INSERT ...", [&file])?;  // Fast inserts only
}
tx.commit()?;
```

### 3. Ignoring SQLITE_BUSY

```rust
// BAD: Crash on busy
conn.execute("INSERT ...", [])?;  // May return SQLITE_BUSY

// GOOD: Retry logic (or use busy_timeout)
loop {
    match conn.execute("INSERT ...", []) {
        Ok(_) => break,
        Err(rusqlite::Error::SqliteFailure(e, _)) if e.code == ErrorCode::DatabaseBusy => {
            std::thread::sleep(Duration::from_millis(10));
            continue;
        }
        Err(e) => return Err(e.into()),
    }
}
```

### 4. Forgetting to Checkpoint

```rust
// BAD: WAL grows forever
// (no checkpoint calls)

// GOOD: Periodic checkpoint
impl SqliteBackend {
    pub fn maintenance(&self) -> Result<(), FsError> {
        let conn = self.get_write_connection()?;
        conn.execute_batch("PRAGMA wal_checkpoint(TRUNCATE);")?;
        Ok(())
    }
}
```

### 5. Not Using Transactions for Batch Operations

```rust
// BAD: 1000 separate transactions
for item in items {
    conn.execute("INSERT ...", [item])?;  // Each is auto-committed
}

// GOOD: Single transaction
let tx = conn.transaction()?;
for item in items {
    tx.execute("INSERT ...", [item])?;
}
tx.commit()?;  // 10-100x faster
```

---

## SQLCipher (Encryption)

For encrypted databases, use SQLCipher:

```rust
use rusqlite::Connection;

fn open_encrypted(path: &Path, key: &str) -> Result<Connection, rusqlite::Error> {
    let conn = Connection::open(path)?;

    // Set encryption key (must be first operation)
    conn.execute_batch(&format!("PRAGMA key = '{}';", key))?;

    // Verify encryption is working
    conn.execute_batch("SELECT count(*) FROM sqlite_master;")?;

    // Now configure as normal
    conn.execute_batch("
        PRAGMA journal_mode = WAL;
        PRAGMA synchronous = NORMAL;
    ")?;

    Ok(conn)
}
```

**Key management:**
- Never hardcode keys
- Rotate keys periodically (requires re-encryption)
- Use key derivation (PBKDF2) for password-based keys
- Store key metadata separately from data

See [Security Model](./security-model.md) for key rotation patterns.

---

## Summary Checklist

Before deploying SQLite backend to production:

- [ ] WAL mode enabled (`PRAGMA journal_mode = WAL`)
- [ ] Busy timeout configured (30+ seconds)
- [ ] Single-writer queue implemented
- [ ] Connection pool for readers
- [ ] Auto-vacuum configured
- [ ] Backup strategy in place
- [ ] Monitoring for WAL size, queue depth
- [ ] Integrity checks scheduled
- [ ] Migration path for schema changes
- [ ] Indexes for common queries

---

*"SQLite is not a toy database. It's a serious database that looks like a toy."*
