# AnyFS — API Quick Reference

**Condensed reference for developers**

---

## Installation

```toml
[dependencies]
anyfs = "0.1"
anyfs-container = "0.1"  # Optional
```

With backends:

```toml
anyfs = { version = "0.1", features = ["sqlite", "vrootfs"] }
```

---

## Creating a Backend Stack

```rust
use anyfs::{MemoryBackend, SqliteBackend, LimitedBackend, FeatureGatedBackend};
use anyfs_container::FilesContainer;

// Simple
let fs = FilesContainer::new(MemoryBackend::new());

// With limits
let fs = FilesContainer::new(
    LimitedBackend::new(SqliteBackend::open("data.db")?)
        .with_max_total_size(100 * 1024 * 1024)
        .with_max_file_size(10 * 1024 * 1024)
);

// Full stack
let fs = FilesContainer::new(
    FeatureGatedBackend::new(
        LimitedBackend::new(SqliteBackend::open("data.db")?)
            .with_max_total_size(100 * 1024 * 1024)
    )
    .with_symlinks()
    .with_hard_links()
);
```

---

## LimitedBackend Methods

```rust
LimitedBackend::new(backend)
    .with_max_total_size(bytes)      // Total storage limit
    .with_max_file_size(bytes)       // Per-file limit
    .with_max_node_count(count)      // Max files/dirs
    .with_max_dir_entries(count)     // Max entries per dir
    .with_max_path_depth(depth)      // Max nesting

// Query
backend.usage()        // -> Usage { total_size, file_count, ... }
backend.limits()       // -> Limits { max_total_size, ... }
backend.remaining()    // -> Remaining { bytes, can_write, ... }
```

---

## FeatureGatedBackend Methods

```rust
FeatureGatedBackend::new(backend)    // All disabled by default
    .with_symlinks()                  // Enable symlink ops
    .with_max_symlink_resolution(40)  // Max hops
    .with_hard_links()                // Enable hard links
    .with_permissions()               // Enable set_permissions
```

---

## File Operations

```rust
// Check existence
fs.exists("/path")?                     // -> bool

// Metadata
let meta = fs.metadata("/path")?;
meta.size                                // file size
meta.file_type                           // File | Directory | Symlink
meta.permissions
meta.created                             // Option<SystemTime>
meta.modified

// Read
let bytes = fs.read("/path")?;           // -> Vec<u8>
let text = fs.read_to_string("/path")?;  // -> String
let chunk = fs.read_range("/path", 0, 1024)?;

// List directory
let entries = fs.read_dir("/path")?;     // -> Vec<DirEntry>

// Write
fs.write("/path", b"content")?;          // Create or overwrite
fs.append("/path", b"more")?;            // Append

// Directories
fs.create_dir("/path")?;
fs.create_dir_all("/path")?;

// Delete
fs.remove_file("/path")?;
fs.remove_dir("/path")?;                 // Empty only
fs.remove_dir_all("/path")?;             // Recursive

// Move/Copy
fs.rename("/from", "/to")?;
fs.copy("/from", "/to")?;

// Links (requires FeatureGatedBackend)
fs.symlink("/target", "/link")?;
fs.hard_link("/original", "/link")?;
fs.read_link("/link")?;                  // -> PathBuf

// Permissions (requires FeatureGatedBackend)
fs.set_permissions("/path", perms)?;

// File size
fs.truncate("/path", 1024)?;             // Resize to 1024 bytes

// Durability
fs.sync()?;                              // Flush all writes
fs.fsync("/path")?;                      // Flush writes for one file
```

---

## Error Handling

```rust
use anyfs_backend::VfsError;

match result {
    Err(VfsError::NotFound(p)) => ...
    Err(VfsError::AlreadyExists(p)) => ...
    Err(VfsError::NotADirectory(p)) => ...
    Err(VfsError::NotAFile(p)) => ...
    Err(VfsError::DirectoryNotEmpty(p)) => ...
    Err(VfsError::QuotaExceeded) => ...
    Err(VfsError::FileSizeExceeded { size, limit }) => ...
    Err(VfsError::FeatureNotEnabled(name)) => ...
    Err(e) => ...
}
```

---

## Built-in Backends

| Type | Feature | Description |
|------|---------|-------------|
| `MemoryBackend` | `memory` (default) | In-memory |
| `SqliteBackend` | `sqlite` | Persistent |
| `VRootFsBackend` | `vrootfs` | Host filesystem |

---

## Middleware

| Type | Purpose |
|------|---------|
| `LimitedBackend<B>` | Quota enforcement |
| `FeatureGatedBackend<B>` | Least privilege |
| `LoggingBackend<B>` | Audit trail |

---

## Type Reference

### From `anyfs-backend`

| Type | Description |
|------|-------------|
| `VfsBackend` | Core trait |
| `VfsError` | Error type |
| `FileType` | `File`, `Directory`, `Symlink` |
| `Metadata` | File/dir metadata |
| `DirEntry` | Directory entry |
| `Permissions` | File permissions |
| `StatFs` | Filesystem stats |

### From `anyfs`

| Type | Description |
|------|-------------|
| `MemoryBackend` | In-memory backend |
| `SqliteBackend` | SQLite backend |
| `VRootFsBackend` | Host FS backend |
| `LimitedBackend<B>` | Quota middleware |
| `FeatureGatedBackend<B>` | Feature gate middleware |
| `LoggingBackend<B>` | Logging middleware |

### From `anyfs-container`

| Type | Description |
|------|-------------|
| `FilesContainer<B>` | Ergonomic wrapper |
