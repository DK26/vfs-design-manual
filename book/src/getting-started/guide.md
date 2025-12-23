# AnyFS — Getting Started Guide

**A practical introduction with examples**

---

## Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
anyfs = "0.1"
anyfs-container = "0.1"
```

By default, `anyfs` includes only the in-memory backend. For additional backends:

```toml
[dependencies]
anyfs = { version = "0.1", features = ["sqlite", "vrootfs"] }
anyfs-container = "0.1"
```

Available features for `anyfs`:
- `memory` — In-memory storage (default, for testing)
- `sqlite` — SQLite-backed persistent storage
- `vrootfs` — Host filesystem backend (via strict-path)

---

## Quick Start

### Hello World

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create an in-memory container
    let mut container = FilesContainer::new(MemoryBackend::new());

    // Write a file (accepts any path-like type)
    container.write("/hello.txt", b"Hello, AnyFS!")?;

    // Read it back
    let content = container.read("/hello.txt")?;
    println!("{}", String::from_utf8_lossy(&content));
    // Output: Hello, AnyFS!

    Ok(())
}
```

### Persistent Storage with SQLite

```rust
use anyfs::SqliteBackend;
use anyfs_container::FilesContainer;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create or open a SQLite-backed container
    let mut container = FilesContainer::new(
        SqliteBackend::open_or_create("my_data.db")?
    );

    // Create a directory structure
    container.create_dir_all("/documents/work")?;

    // Write some files
    container.write("/documents/work/notes.txt", b"Meeting notes for Monday")?;

    // Data persists across program runs
    Ok(())
}
```

---

## Common Tasks

### Creating Directories

```rust
// Single directory (parent must exist)
container.create_dir("/documents")?;

// Recursive (creates parents as needed)
container.create_dir_all("/documents/projects/2024/q1")?;
```

### Reading and Writing Files

```rust
// Write (creates or overwrites)
container.write("/data.txt", b"line 1\n")?;

// Append
container.append("/data.txt", b"line 2\n")?;

// Read entire file
let content = container.read("/data.txt")?;

// Read partial (offset 0, length 6)
let partial = container.read_range("/data.txt", 0, 6)?;
```

### Listing Directories

```rust
for entry in container.read_dir("/documents")? {
    match entry.file_type {
        FileType::File => {
            println!("  {} (file)", entry.name);
        }
        FileType::Directory => {
            println!("  {}/", entry.name);
        }
        FileType::Symlink => {
            println!("  {} -> ...", entry.name);
        }
    }
}
```

### Checking Existence and Metadata

```rust
if container.exists("/maybe-exists.txt")? {
    let meta = container.metadata("/maybe-exists.txt")?;
    println!("Size: {} bytes", meta.size);
    println!("Created: {:?}", meta.created);
    println!("Modified: {:?}", meta.modified);
}
```

### Copying and Moving

```rust
// Copy a file
container.copy("/original.txt", "/copy.txt")?;

// Move (rename)
container.rename("/original.txt", "/renamed.txt")?;
```

### Deleting

```rust
// Delete a file
container.remove_file("/old-file.txt")?;

// Delete an empty directory
container.remove_dir("/empty-folder")?;

// Delete directory and all contents
container.remove_dir_all("/old-folder")?;
```

---

## Configuration

### Builder Pattern

```rust
use anyfs::SqliteBackend;
use anyfs_container::ContainerBuilder;

let container = ContainerBuilder::new(SqliteBackend::open_or_create("data.db")?)
    // Set capacity limits
    .max_total_size(500 * 1024 * 1024)  // 500 MB total
    .max_file_size(50 * 1024 * 1024)    // 50 MB per file
    .max_node_count(100_000)             // 100K files/dirs
    .max_dir_entries(5_000)              // 5K entries per directory

    // Set depth limits
    .max_path_depth(32)

    .build()?;
```

### Checking Capacity

```rust
// Current usage
let usage = container.usage()?;
println!("Using {} bytes", usage.total_size);
println!("Files: {}", usage.file_count);
println!("Directories: {}", usage.directory_count);

// Remaining capacity
let remaining = container.remaining()?;
if !remaining.can_write {
    println!("Storage is full!");
}
```

---

## Links

Symlinks, hard links, and permission mutation are **disabled by default**. Enable only what you need via `ContainerBuilder` (feature whitelist).

### Symbolic Links

```rust
use anyfs::MemoryBackend;
use anyfs_container::ContainerBuilder;

let mut container = ContainerBuilder::new(MemoryBackend::new())
    .symlinks()                     // disabled by default (least privilege)
    .max_symlink_resolution(40)     // optional; default: 40
    .build()?;

// Create a symlink
container.create_dir_all("/deep/nested/dir")?;
container.symlink("/deep/nested/dir", "/shortcut")?;

// Read the target (without following)
let target = container.read_link("/shortcut")?;

// Regular operations follow symlinks when enabled
container.read_dir("/shortcut")?;  // lists /deep/nested/dir
```

### Hard Links

```rust
use anyfs::MemoryBackend;
use anyfs_container::ContainerBuilder;

let mut container = ContainerBuilder::new(MemoryBackend::new())
    .hard_links()                   // disabled by default (least privilege)
    .build()?;

// Create a file
container.write("/original.txt", b"content")?;

// Create a hard link (same file, different path)
container.hard_link("/original.txt", "/link.txt")?;

// Both paths refer to the same content
// Deleting one doesn't affect the other
container.remove_file("/original.txt")?;
let content = container.read("/link.txt")?;  // Still works
```

---

## Import and Export (Future)

Import/export helpers (copying between a host filesystem path and a container path) are planned, but are not part of the current core API.

---

## Error Handling

### Pattern Matching on Errors

```rust
use anyfs_container::ContainerError;

match container.write("/file.txt", &large_data) {
    Ok(()) => println!("Written successfully"),

    Err(ContainerError::NotFound(p)) => {
        println!("Parent directory doesn't exist: {}", p);
    }

    Err(ContainerError::FileSizeExceeded { size, limit }) => {
        println!("File too large: {} bytes (limit: {})", size, limit);
    }

    Err(ContainerError::TotalSizeExceeded { used, limit }) => {
        println!("Storage full: {} / {} bytes", used, limit);
    }

    Err(ContainerError::FeatureNotEnabled(name)) => {
        println!("Feature is disabled by default: {}", name);
    }

    Err(e) => println!("Other error: {}", e),
}
```

### Common Error Scenarios

| Error | Cause | Solution |
|-------|-------|----------|
| `NotFound` | Path doesn't exist | Check with `exists()` first, or use `create_dir_all` |
| `AlreadyExists` | Creating over existing | Remove first, or check existence |
| `NotADirectory` | Operating on file as dir | Check `metadata().file_type` |
| `DirectoryNotEmpty` | Removing non-empty dir | Use `remove_dir_all()` instead |
| `FileSizeExceeded` | File too large | Increase limit or split file |
| `TotalSizeExceeded` | Storage full | Delete files or increase quota |
| `FeatureNotEnabled` | Feature is disabled by policy | Enable via `ContainerBuilder` (whitelist) |

---

## Testing

### Use In-Memory Backend for Tests

```rust
#[cfg(test)]
mod tests {
    use anyfs::MemoryBackend;
    use anyfs_container::FilesContainer;

    fn test_container() -> FilesContainer<MemoryBackend> {
        FilesContainer::new(MemoryBackend::new())
    }

    #[test]
    fn test_write_and_read() {
        let mut container = test_container();

        container.write("/test.txt", b"test data").unwrap();
        let content = container.read("/test.txt").unwrap();

        assert_eq!(content, b"test data");
    }

    #[test]
    fn test_directory_operations() {
        let mut container = test_container();

        container.create_dir_all("/a/b/c").unwrap();

        assert!(container.exists("/a").unwrap());
        assert!(container.exists("/a/b").unwrap());
        assert!(container.exists("/a/b/c").unwrap());
    }
}
```

### Test with Capacity Limits

```rust
use anyfs::MemoryBackend;
use anyfs_container::ContainerBuilder;

#[test]
fn test_storage_limit() {
    let mut container = ContainerBuilder::new(MemoryBackend::new())
        .max_total_size(1024)  // 1 KB limit
        .build()
        .unwrap();

    let big_data = vec![0u8; 2048];  // 2 KB

    let result = container.write("/big.bin", &big_data);

    assert!(result.is_err());
}
```

---

## Best Practices

### 1. Handle Errors Gracefully

```rust
use anyfs::VfsBackend;
use anyfs_container::FilesContainer;

fn ensure_parent_exists<B: VfsBackend>(
    container: &mut FilesContainer<B>,
    path: &str,
) -> Result<(), ContainerError> {
    // FilesContainer handles path parsing internally
    if let Some(parent) = std::path::Path::new(path).parent() {
        if !container.exists(parent)? {
            container.create_dir_all(parent)?;
        }
    }
    Ok(())
}
```

### 2. Use Appropriate Backend for Use Case

| Use Case | Recommended Backend |
|----------|---------------------|
| Unit tests | `MemoryBackend` |
| Integration tests | `MemoryBackend` or temp `SqliteBackend` |
| Production | `SqliteBackend` |
| Host filesystem access | `VRootFsBackend` |

### 3. Set Reasonable Limits

```rust
use anyfs::SqliteBackend;
use anyfs_container::ContainerBuilder;

// Production defaults suggestion
let container = ContainerBuilder::new(SqliteBackend::create("data.db")?)
    .max_total_size(1024 * 1024 * 1024)  // 1 GB
    .max_file_size(100 * 1024 * 1024)    // 100 MB
    .max_node_count(1_000_000)            // 1M nodes
    .max_dir_entries(10_000)              // 10K per dir
    .max_path_depth(64)
    .build()?;
```

---

## Next Steps

- [API Quick Reference](./api-reference.md) — Condensed API overview
- [Backend Implementer's Guide](../implementation/backend-guide.md) — Create custom backends
- [Design Overview](../architecture/design-overview.md) — Technical architecture

---

*Happy coding!*
