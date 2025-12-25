# AnyFS — Getting Started Guide

**A practical introduction with examples**

---

## Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
anyfs = "0.1"
anyfs-container = "0.1"  # Optional: for ergonomic wrapper
```

For additional backends:

```toml
[dependencies]
anyfs = { version = "0.1", features = ["sqlite", "vrootfs"] }
```

Available features:
- `memory` — In-memory storage (default)
- `sqlite` — SQLite-backed persistent storage
- `vrootfs` — Host filesystem backend

---

## Quick Start

### Hello World

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut fs = FilesContainer::new(MemoryBackend::new());

    fs.write("/hello.txt", b"Hello, AnyFS!")?;
    let content = fs.read("/hello.txt")?;
    println!("{}", String::from_utf8_lossy(&content));

    Ok(())
}
```

### With Quotas

```rust
use anyfs::{SqliteBackend, Quota};
use anyfs_container::FilesContainer;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let backend = Quota::new(SqliteBackend::open_or_create("data.db")?)
        .with_max_total_size(100 * 1024 * 1024)  // 100 MB
        .with_max_file_size(10 * 1024 * 1024);   // 10 MB per file

    let mut fs = FilesContainer::new(backend);

    fs.create_dir_all("/documents")?;
    fs.write("/documents/notes.txt", b"Meeting notes")?;

    Ok(())
}
```

### With Restrictions

```rust
use anyfs::{MemoryBackend, Restrictions};
use anyfs_container::FilesContainer;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Block specific operations for untrusted code
    let backend = Restrictions::new(MemoryBackend::new())
        .deny_hard_links()    // Block hard_link() calls
        .deny_permissions();  // Block set_permissions() calls

    let mut fs = FilesContainer::new(backend);

    // Symlinks work (not blocked)
    fs.write("/original.txt", b"content")?;
    fs.symlink("/original.txt", "/shortcut")?;

    Ok(())
}
```

### Full Stack (Layer-based)

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, TracingLayer};
use anyfs_container::FilesContainer;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let backend = SqliteBackend::open_or_create("data.db")?
        .layer(QuotaLayer::new()
            .max_total_size(100 * 1024 * 1024)
            .max_file_size(10 * 1024 * 1024))
        .layer(RestrictionsLayer::new()
            .deny_hard_links()
            .deny_permissions())
        .layer(TracingLayer::new());

    let mut fs = FilesContainer::new(backend);

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

let backend = Quota::new(MemoryBackend::new())
    .with_max_total_size(500 * 1024 * 1024)  // 500 MB total
    .with_max_file_size(50 * 1024 * 1024)    // 50 MB per file
    .with_max_node_count(100_000)             // 100K files/dirs
    .with_max_dir_entries(5_000)              // 5K per directory
    .with_max_path_depth(32);                 // Max nesting

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
let backend = Restrictions::new(MemoryBackend::new())
    .deny_symlinks()      // Block symlink() calls
    .deny_hard_links()    // Block hard_link() calls
    .deny_permissions();  // Block set_permissions() calls

// Blocked operations return VfsError::FeatureNotEnabled
```

### Tracing — Instrumentation

```rust
use anyfs::{MemoryBackend, Tracing};

let backend = Tracing::new(MemoryBackend::new())
    .with_logger(MyLogger::new());
```

---

## Error Handling

```rust
use anyfs_backend::VfsError;

match fs.write("/file.txt", &large_data) {
    Ok(()) => println!("Written"),

    Err(VfsError::NotFound(p)) => println!("Not found: {}", p),
    Err(VfsError::AlreadyExists(p)) => println!("Exists: {}", p),
    Err(VfsError::QuotaExceeded) => println!("Quota exceeded"),
    Err(VfsError::FeatureNotEnabled(f)) => println!("Feature disabled: {}", f),

    Err(e) => println!("Error: {}", e),
}
```

---

## Testing

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

#[test]
fn test_write_and_read() {
    let mut fs = FilesContainer::new(MemoryBackend::new());

    fs.write("/test.txt", b"test data").unwrap();
    let content = fs.read("/test.txt").unwrap();

    assert_eq!(content, b"test data");
}
```

With limits:

```rust
use anyfs::{MemoryBackend, Quota};
use anyfs_container::FilesContainer;

#[test]
fn test_quota_exceeded() {
    let backend = Quota::new(MemoryBackend::new())
        .with_max_total_size(1024);  // 1 KB
    let mut fs = FilesContainer::new(backend);

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
| Production | `SqliteBackend` |
| Host filesystem | `VRootFsBackend` |

### 2. Compose Middleware for Your Needs

```rust
// Minimal: just storage
let fs = FilesContainer::new(MemoryBackend::new());

// With limits (layer-based)
let fs = FilesContainer::new(
    MemoryBackend::new()
        .layer(QuotaLayer::new()
            .max_total_size(100 * 1024 * 1024))
);

// Sandboxed (layer-based)
let fs = FilesContainer::new(
    SqliteBackend::open("data.db")?
        .layer(QuotaLayer::new()
            .max_total_size(100 * 1024 * 1024))
        .layer(RestrictionsLayer::new()
            .deny_hard_links()
            .deny_permissions())
);
```

### 3. Handle Errors Gracefully

Always check for quota exceeded, feature not enabled, and other errors.

---

## Next Steps

- [API Quick Reference](./api-reference.md)
- [Design Overview](../architecture/design-overview.md)
- [Backend Implementer's Guide](../implementation/backend-guide.md)
