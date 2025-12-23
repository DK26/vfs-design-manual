# AnyFS — API Quick Reference

**Condensed reference for developers**

---

## Installation

```toml
[dependencies]
anyfs = "0.1"
anyfs-container = "0.1"
```

With specific backends:

```toml
[dependencies]
anyfs = { version = "0.1", features = ["sqlite", "vrootfs"] }
anyfs-container = "0.1"
```

---

## Creating a Container

```rust
use anyfs::{MemoryBackend, SqliteBackend};
use anyfs_container::{FilesContainer, ContainerBuilder};

// In-memory (testing)
let container = FilesContainer::new(MemoryBackend::new());

// SQLite (production)
let container = FilesContainer::new(
    SqliteBackend::open_or_create("data.db")?
);

// Full configuration with capacity limits
let container = ContainerBuilder::new(SqliteBackend::open_or_create("data.db")?)
    // Security posture: advanced features are opt-in (whitelist)
    .symlinks()
    .max_symlink_resolution(40)
    .hard_links()
    .permissions()
    .max_total_size(100 * 1024 * 1024)  // 100 MB
    .max_file_size(10 * 1024 * 1024)    // 10 MB
    .max_node_count(10_000)
    .max_dir_entries(1_000)
    .max_path_depth(64)
    .build()?;

// Note: advanced features (symlinks/hard links/permissions) are disabled by default and must be explicitly enabled
// When symlinks are disabled, operations reject paths that traverse symlinks (nofollow).
```

---

## Path Handling

FilesContainer accepts any path-like type (`&str`, `String`, `&Path`, `PathBuf`):

```rust
// All of these work
container.write("/documents/report.pdf", data)?;
container.write(String::from("/file.txt"), data)?;
container.write(Path::new("/data.bin"), data)?;

// Normalization (automatic)
// "//a/../b/./c"  → "/b/c"
// "/../../../a"   → "/a" (can't escape root)
```

For custom backends, use `VirtualPath` directly:

```rust
use anyfs::VirtualPath;

let path = VirtualPath::new("/documents/report.pdf")?;
path.parent()           // Some(VirtualPath("/documents"))
path.file_name()        // Some("report.pdf")
path.as_str()           // "/documents/report.pdf"
```

---

## File Operations

### Reading

```rust
// Check existence
container.exists("/path")?                    // → bool

// Get metadata (follows symlinks)
let meta = container.metadata("/path")?;
meta.size                                     // file size
meta.file_type                                // File | Directory | Symlink
meta.permissions                              // Permissions
meta.nlink                                    // hard link count
meta.created                                  // Option<SystemTime>
meta.modified                                 // Option<SystemTime>
meta.accessed                                 // Option<SystemTime>

// Get metadata without following symlinks
let meta = container.symlink_metadata("/path")?;

// Read file
let bytes = container.read("/path")?;         // → Vec<u8>
let text = container.read_to_string("/path")?; // → String (UTF-8)
let chunk = container.read_range("/path", 0, 1024)?;  // partial read

// List directory (std::fs aligned name)
let entries = container.read_dir("/path")?;   // → Vec<DirEntry>
for entry in entries {
    println!("{}: {:?}", entry.name, entry.file_type);
}

// Read symlink target (without following; requires symlinks enabled)
let target = container.read_link("/path")?;   // → VirtualPath
```

### Writing

```rust
// Create directory (std::fs aligned names)
container.create_dir("/path")?;               // single level
container.create_dir_all("/path")?;           // recursive

// Write file (create or overwrite)
container.write("/path", b"content")?;

// Append to file
container.append("/path", b"more")?;

// Create symlink (requires symlinks enabled)
container.symlink("/target", "/link")?;       // link points to target

// Create hard link (requires hard links enabled)
container.hard_link("/original", "/link")?;   // link shares content with original

// Delete (std::fs aligned: separate file/dir methods)
container.remove_file("/path")?;              // file only
container.remove_dir("/path")?;               // empty directory only
container.remove_dir_all("/path")?;           // recursive

// Set permissions (requires permissions enabled)
container.set_permissions("/path", Permissions::new())?;

// Rename/move
container.rename("/from", "/to")?;

// Copy
container.copy("/from", "/to")?;              // single file
```

---

## Import / Export (Future)

Import/export helpers (copying between a host filesystem path and a container path) are planned, but are not part of the current core API.

---

## Capacity Management

```rust
// Check usage
let usage = container.usage()?;
usage.total_size       // bytes used
usage.node_count       // total nodes
usage.file_count
usage.directory_count

// Check limits
let limits = container.limits();
limits.max_total_size  // Option<u64>
limits.max_file_size

// Check remaining
let remaining = container.remaining()?;
remaining.bytes        // Option<u64>
remaining.nodes
remaining.can_write    // quick check
```

---

## Error Handling

```rust
use anyfs_container::ContainerError;

match container.write("/path", &data) {
    Ok(()) => println!("Written"),

    Err(ContainerError::NotFound(p)) => println!("Path not found: {}", p),
    Err(ContainerError::AlreadyExists(p)) => println!("Already exists: {}", p),
    Err(ContainerError::NotADirectory(p)) => println!("Not a directory: {}", p),
    Err(ContainerError::NotAFile(p)) => println!("Not a file: {}", p),
    Err(ContainerError::DirectoryNotEmpty(p)) => println!("Not empty: {}", p),

    Err(ContainerError::TotalSizeExceeded { used, limit }) => {
        println!("Storage full: {} / {} bytes", used, limit);
    }
    Err(ContainerError::FileSizeExceeded { size, limit }) => {
        println!("File too large: {} > {} bytes", size, limit);
    }

    Err(ContainerError::FeatureNotEnabled(name)) => {
        println!("Feature not enabled: {}", name);
    }

    Err(e) => println!("Other error: {}", e),
}
```

---

## Backend Lifecycle

```rust
use anyfs::SqliteBackend;

// Create new (fails if exists)
let backend = SqliteBackend::create("new.db")?;

// Open existing (fails if doesn't exist)
let backend = SqliteBackend::open("existing.db")?;

// Open or create
let backend = SqliteBackend::open_or_create("data.db")?;
```

---

## Quick Patterns

### Ensure parent directories exist

```rust
use anyfs::VfsBackend;
use anyfs_container::FilesContainer;

fn write_with_parents<B: VfsBackend>(
    container: &mut FilesContainer<B>,
    path: &str,
    data: &[u8],
) -> Result<(), ContainerError> {
    if let Some(parent) = std::path::Path::new(path).parent() {
        container.create_dir_all(parent)?;
    }
    container.write(path, data)
}
```

### Walk directory tree

```rust
use anyfs::VfsBackend;
use anyfs_container::FilesContainer;

fn walk<B: VfsBackend>(
    container: &FilesContainer<B>,
    path: &str,
    f: &mut impl FnMut(&str, &DirEntry),
) -> Result<(), ContainerError> {
    for entry in container.read_dir(path)? {
        let child_path = format!("{}/{}", path, entry.name);
        f(&child_path, &entry);
        if entry.file_type == FileType::Directory {
            walk(container, &child_path, f)?;
        }
    }
    Ok(())
}
```

---

## Type Reference

### From `anyfs-traits` (re-exported by `anyfs`)

| Type | Description |
|------|-------------|
| `VfsBackend` | Core trait for backends (20 methods) |
| `VirtualPath` | Validated, normalized path |
| `FileType` | `File`, `Directory`, `Symlink` |
| `Metadata` | File/directory metadata (includes `nlink`, `permissions`, `accessed`) |
| `Permissions` | File permissions (readonly flag) |
| `DirEntry` | Directory listing entry |
| `VfsError` | Backend-level errors |

### From `anyfs`

| Type | Description |
|------|-------------|
| `MemoryBackend` | In-memory storage |
| `SqliteBackend` | SQLite-backed storage |
| `VRootFsBackend` | Host filesystem (contained) |

### From `anyfs-container`

| Type | Description |
|------|-------------|
| `FilesContainer<B>` | Main API, generic over backend |
| `ContainerBuilder<B>` | Builder for FilesContainer |
| `CapacityLimits` | Quota configuration |
| `CapacityUsage` | Current usage stats |
| `ContainerError` | Container-level errors |

---

*For full details, see the [Design Overview](../architecture/design-overview.md).*
