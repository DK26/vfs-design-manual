# AnyFS — Getting Started Guide

**A practical introduction with examples**

---

## Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
anyfs = "0.1"
```

For additional backends:

```toml
[dependencies]
anyfs = { version = "0.1", features = ["sqlite", "stdfs", "vrootfs"] }
```

Available features:
- `memory` — In-memory storage (default)
- `sqlite` — SQLite-backed persistent storage
- `stdfs` — Direct `std::fs` delegation (no containment)
- `vrootfs` — Host filesystem backend with path containment

---

## Quick Start

### Hello World

```rust
use anyfs::{MemoryBackend, FileStorage};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let fs = FileStorage::new(MemoryBackend::new());

    fs.write("/hello.txt", b"Hello, AnyFS!")?;
    let content = fs.read("/hello.txt")?;
    println!("{}", String::from_utf8_lossy(&content));

    Ok(())
}
```

### With Quotas

```rust
use anyfs::{SqliteBackend, Quota, FileStorage};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let backend = QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)  // 100 MB
        .max_file_size(10 * 1024 * 1024)    // 10 MB per file
        .build()
        .layer(SqliteBackend::open_or_create("data.db")?);

    let fs = FileStorage::new(backend);

    fs.create_dir_all("/documents")?;
    fs.write("/documents/notes.txt", b"Meeting notes")?;

    Ok(())
}
```

### With Restrictions

```rust
use anyfs::{MemoryBackend, Restrictions, FileStorage};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Block specific operations for untrusted code
    let backend = RestrictionsLayer::builder()
        .deny_hard_links()    // Block hard_link() calls
        .deny_permissions()   // Block set_permissions() calls
        .build()
        .layer(MemoryBackend::new());

    let fs = FileStorage::new(backend);

    // Symlinks work (not blocked)
    fs.write("/original.txt", b"content")?;
    fs.symlink("/original.txt", "/shortcut")?;

    Ok(())
}
```

### Full Stack (Layer-based)

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, TracingLayer, FileStorage};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let backend = SqliteBackend::open_or_create("data.db")?
        .layer(QuotaLayer::builder()
            .max_total_size(100 * 1024 * 1024)
            .max_file_size(10 * 1024 * 1024)
            .build())
        .layer(RestrictionsLayer::builder()
            .deny_hard_links()
            .deny_permissions()
            .build())
        .layer(TracingLayer::new());

    let fs = FileStorage::new(backend);

    fs.create_dir_all("/data")?;
    fs.write("/data/file.txt", b"hello")?;

    Ok(())
}
```

---

## Common Operations

### Creating Directories

```rust
fs.create_dir("/documents")?;              // Single level
fs.create_dir_all("/documents/2024/q1")?;  // Recursive
```

### Reading and Writing Files

```rust
fs.write("/data.txt", b"line 1\n")?;       // Create or overwrite
fs.append("/data.txt", b"line 2\n")?;      // Append

let content = fs.read("/data.txt")?;                    // Read all
let partial = fs.read_range("/data.txt", 0, 6)?;        // Read range
let text = fs.read_to_string("/data.txt")?;             // Read as String
```

### Listing Directories

```rust
for entry in fs.read_dir("/documents")? {
    println!("{}: {:?}", entry.name, entry.file_type);
}
```

### Checking Existence and Metadata

```rust
if fs.exists("/file.txt")? {
    let meta = fs.metadata("/file.txt")?;
    println!("Size: {} bytes", meta.size);
}
```

### Copying and Moving

```rust
fs.copy("/original.txt", "/copy.txt")?;
fs.rename("/original.txt", "/renamed.txt")?;
```

### Deleting

```rust
fs.remove_file("/old-file.txt")?;
fs.remove_dir("/empty-folder")?;
fs.remove_dir_all("/old-folder")?;
```

---

## Middleware

### Quota — Resource Limits

```rust
use anyfs::{MemoryBackend, Quota};

let backend = QuotaLayer::builder()
    .max_total_size(500 * 1024 * 1024)   // 500 MB total
    .max_file_size(50 * 1024 * 1024)     // 50 MB per file
    .max_node_count(100_000)             // 100K files/dirs
    .max_dir_entries(5_000)              // 5K per directory
    .max_path_depth(32)                  // Max nesting
    .build()
    .layer(MemoryBackend::new());

// Check usage
let usage = backend.usage();
println!("Using {} bytes", usage.total_size);

// Check remaining
let remaining = backend.remaining();
if !remaining.can_write {
    println!("Storage full!");
}
```

### Restrictions — Block Operations

```rust
use anyfs::{MemoryBackend, Restrictions};

// By default, all operations work. Use deny_*() to block specific ones.
let backend = RestrictionsLayer::builder()
    .deny_symlinks()      // Block symlink() calls
    .deny_hard_links()    // Block hard_link() calls
    .deny_permissions()   // Block set_permissions() calls
    .build()
    .layer(MemoryBackend::new());

// Blocked operations return FsError::FeatureNotEnabled
```

### Tracing — Instrumentation

```rust
use anyfs::{MemoryBackend, TracingLayer};

// TracingLayer uses the global tracing subscriber by default
let backend = MemoryBackend::new().layer(TracingLayer::new());

// Or configure with custom settings
let backend = MemoryBackend::new()
    .layer(TracingLayer::new()
        .with_target("myapp::fs")
        .with_level(tracing::Level::DEBUG));
```

---

## Error Handling

```rust
use anyfs_backend::FsError;

match fs.write("/file.txt", &large_data) {
    Ok(()) => println!("Written"),

    Err(FsError::NotFound { path, .. }) => println!("Not found: {}", path.display()),
    Err(FsError::AlreadyExists { path, .. }) => println!("Exists: {}", path.display()),
    Err(FsError::QuotaExceeded { .. }) => println!("Quota exceeded"),
    Err(FsError::FeatureNotEnabled { feature }) => println!("Feature disabled: {}", feature),

    Err(e) => println!("Error: {}", e),
}
```

---

## Testing

```rust
use anyfs::{MemoryBackend, FileStorage};

#[test]
fn test_write_and_read() {
    let fs = FileStorage::new(MemoryBackend::new());

    fs.write("/test.txt", b"test data").unwrap();
    let content = fs.read("/test.txt").unwrap();

    assert_eq!(content, b"test data");
}
```

With limits:

```rust
use anyfs::{MemoryBackend, Quota, FileStorage};

#[test]
fn test_quota_exceeded() {
    let backend = QuotaLayer::builder()
        .max_total_size(1024)  // 1 KB
        .build()
        .layer(MemoryBackend::new());
    let fs = FileStorage::new(backend);

    let big_data = vec![0u8; 2048];  // 2 KB
    let result = fs.write("/big.bin", &big_data);

    assert!(result.is_err());
}
```

---

## Best Practices

### 1. Use Appropriate Backend

| Use Case | Backend |
|----------|---------|
| Testing | `MemoryBackend` |
| Production (portable) | `SqliteBackend` |
| Host filesystem (with containment) | `VRootFsBackend` |
| Host filesystem (direct access) | `StdFsBackend` |

### 2. Compose Middleware for Your Needs

```rust
// Minimal: just storage
let fs = FileStorage::new(MemoryBackend::new());

// With limits (layer-based)
let fs = FileStorage::new(
    MemoryBackend::new()
        .layer(QuotaLayer::builder()
            .max_total_size(100 * 1024 * 1024)
            .build())
);

// Sandboxed (layer-based)
let fs = FileStorage::new(
    SqliteBackend::open("data.db")?
        .layer(QuotaLayer::builder()
            .max_total_size(100 * 1024 * 1024)
            .build())
        .layer(RestrictionsLayer::builder()
            .deny_hard_links()
            .deny_permissions()
            .build())
);
```

### 3. Handle Errors Gracefully

Always check for quota exceeded, feature not enabled, and other errors.

---

## Advanced Use Cases (Future)

These use cases require `anyfs-mount` (future crate).

### Database-Backed Drive with Live Monitoring

Mount a database-backed filesystem and query it directly for real-time analytics:

```
┌─────────────────────────────────────────────────────────────┐
│  Database (SQLite, PostgreSQL, etc.)                        │
├─────────────────────────────────────────────────────────────┤
│                         │                                   │
│    anyfs-mount          │         Stats Dashboard           │
│    (write + read)       │         (direct DB queries)       │
│         │               │               │                   │
│         ▼               │               ▼                   │
│  /mnt/workspace         │    SELECT SUM(size) FROM nodes    │
│  $ cp file.txt ./       │    SELECT COUNT(*) FROM nodes     │
│  $ mkdir projects/      │    SELECT * FROM operations_log   │
│                         │               │                   │
│                         │               ▼                   │
│                         │        ┌──────────────┐           │
│                         │        │ Live Graphs  │           │
│                         │        │ - Disk usage │           │
│                         │        │ - File count │           │
│                         │        │ - Recent ops │           │
│                         │        └──────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

**SQLite Example:**

```rust
use anyfs::{SqliteBackend, QuotaLayer, TracingLayer};
use anyfs_fuse::FuseMount;

// Mount the drive
let backend = SqliteBackend::open("tenant.db")?
    .layer(TracingLayer::new())  // Logs operations to tracing subscriber
    .layer(QuotaLayer::builder()
        .max_total_size(1_000_000_000)
        .build());

let mount = FuseMount::mount(backend, "/mnt/workspace")?;
```

```rust
// Meanwhile, in a monitoring dashboard...
use rusqlite::{Connection, OpenFlags};

let conn = Connection::open_with_flags(
    "tenant.db",
    OpenFlags::SQLITE_OPEN_READ_ONLY,  // Safe concurrent reads
)?;

loop {
    let (file_count, total_bytes): (i64, i64) = conn.query_row(
        "SELECT COUNT(*), COALESCE(SUM(size), 0) FROM nodes WHERE type = 'file'",
        [],
        |row| Ok((row.get(0)?, row.get(1)?)),
    )?;

    let recent_ops: Vec<String> = conn
        .prepare("SELECT operation, path, timestamp FROM audit_log ORDER BY timestamp DESC LIMIT 10")?
        .query_map([], |row| Ok(format!("{}: {}", row.get::<_, String>(0)?, row.get::<_, String>(1)?)))?
        .collect::<Result<_, _>>()?;

    render_dashboard(file_count, total_bytes, &recent_ops);
    std::thread::sleep(Duration::from_secs(1));
}
```

**Works with any database backend:**

| Backend | Direct Query Method |
|---------|---------------------|
| `SqliteBackend` | `rusqlite` with `SQLITE_OPEN_READ_ONLY` |
| `PostgresBackend` (future) | Standard `postgres` crate connection |
| `MySqlBackend` (future) | Standard `mysql` crate connection |

**What you can visualize:**
- Real-time storage usage (gauges, bar charts)
- File count over time (line graphs)
- Operations log (live feed)
- Most accessed files (heatmaps)
- Directory tree maps (size visualization)
- Per-tenant usage (multi-tenant dashboards)

This pattern is powerful because the database is the source of truth — you get filesystem semantics via FUSE and SQL analytics via direct queries, from the same data.

### RAM Drive

```rust
use anyfs::{MemoryBackend, QuotaLayer};
use anyfs_fuse::FuseMount;

// 4GB RAM drive
let mount = FuseMount::mount(
    MemoryBackend::new()
        .layer(QuotaLayer::builder()
            .max_total_size(4 * 1024 * 1024 * 1024)
            .build()),
    "/mnt/ramdisk"
)?;

// Use for fast compilation, temp files, etc.
// $ TMPDIR=/mnt/ramdisk cargo build
```

### Sandboxed AI Agent Workspace

```rust
use anyfs::{MemoryBackend, QuotaLayer, PathFilterLayer, RestrictionsLayer, TracingLayer};
use anyfs_fuse::FuseMount;

let mount = FuseMount::mount(
    MemoryBackend::new()
        .layer(PathFilterLayer::builder()
            .allow("/workspace/**")
            .deny("**/..*")           // No hidden files
            .deny("**/.*")            // No dotfiles
            .build())
        .layer(RestrictionsLayer::builder()
            .deny_symlinks()          // Prevent symlink attacks
            .deny_hard_links()
            .build())
        .layer(QuotaLayer::builder()
            .max_total_size(100 * 1024 * 1024)
            .max_file_size(10 * 1024 * 1024)
            .build())
        .layer(TracingLayer::new()),  // Full audit trail
    "/mnt/agent"
)?;

// Agent uses standard filesystem APIs
// All operations are sandboxed, quota-limited, and logged
```

---

## Next Steps

- [API Quick Reference](./api-reference.md)
- [Design Overview](../architecture/design-overview.md)
- [Backend Implementer's Guide](../implementation/backend-guide.md)
