# AnyFS — API Quick Reference

**Condensed reference for developers**

---

## Installation

```toml
[dependencies]
anyfs = "0.1"
anyfs-container = "0.1"  # Optional
```

With backends and optional features:

```toml
anyfs = { version = "0.1", features = ["sqlite", "vrootfs", "bytes"] }
```

---

## Creating a Backend Stack

```rust
use anyfs::{MemoryBackend, SqliteBackend, Quota, FeatureGuard, Tracing};
use anyfs_container::FilesContainer;

// Simple
let fs = FilesContainer::new(MemoryBackend::new());

// With limits
let fs = FilesContainer::new(
    Quota::new(SqliteBackend::open("data.db")?)
        .with_max_total_size(100 * 1024 * 1024)
        .with_max_file_size(10 * 1024 * 1024)
);

// Full stack (manual composition)
let fs = FilesContainer::new(
    Tracing::new(
        FeatureGuard::new(
            Quota::new(SqliteBackend::open("data.db")?)
                .with_max_total_size(100 * 1024 * 1024)
        )
        .with_symlinks()
        .with_hard_links()
    )
);

// Layer-based composition
use anyfs::{QuotaLayer, FeatureGuardLayer, TracingLayer};

let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(FeatureGuardLayer::new().allow_symlinks())
    .layer(TracingLayer::new());

// BackendStack builder (fluent API)
use anyfs_container::BackendStack;

let fs = BackendStack::new(SqliteBackend::open("data.db")?)
    .limited(|l| l.max_total_size(100 * 1024 * 1024))
    .feature_gated(|g| g.allow_symlinks())
    .traced()
    .into_container();
```

---

## Quota Methods

```rust
Quota::new(backend)
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

## FeatureGuard Methods

```rust
FeatureGuard::new(backend)    // All disabled by default
    .with_symlinks()                  // Enable symlink ops
    .with_max_symlink_resolution(40)  // Max hops
    .with_hard_links()                // Enable hard links
    .with_permissions()               // Enable set_permissions
```

---

## Tracing Methods

```rust
Tracing::new(backend)
    .with_target("anyfs")             // tracing target
    .with_level(tracing::Level::DEBUG)
```

---

## PathFilter Methods

```rust
PathFilter::new(backend)
    .allow("/workspace/**")    // Allow glob pattern
    .deny("**/.env")           // Deny glob pattern
    .deny("**/secrets/**")

// Rules evaluated in order; first match wins
// No match = denied (deny by default)
```

---

## ReadOnly Methods

```rust
ReadOnly::new(backend)

// All read operations pass through
// All write operations return VfsError::ReadOnly
```

---

## RateLimit Methods

```rust
RateLimit::new(backend)
    .max_ops(1000)           // Operation limit
    .per_second()            // Window: 1 second
    // or
    .per_minute()            // Window: 60 seconds
```

---

## DryRun Methods

```rust
let mut dry_run = DryRun::new(backend);

// Read operations execute normally
// Write operations are logged but not executed
dry_run.write("/file.txt", b"data")?;  // Logged, returns Ok

// Inspect logged operations
let ops = dry_run.operations();        // -> &[Operation]
dry_run.clear();                       // Clear the log
```

---

## Cache Methods

```rust
Cache::new(backend)
    .max_entries(1000)                     // LRU cache size
    .max_entry_size(1024 * 1024)          // 1MB max per entry
    .ttl(std::time::Duration::from_secs(60))  // Expiration
```

---

## Overlay Methods

```rust
use anyfs::{SqliteBackend, MemoryBackend, Overlay};

let base = SqliteBackend::open("base.db")?;  // Read-only base
let upper = MemoryBackend::new();             // Writable upper

let overlay = Overlay::new(base, upper);

// Read: check upper first, fall back to base
// Write: always to upper layer
// Delete: whiteout marker in upper
```

---

## VfsBackendExt Methods

Extension methods available on all backends:

```rust
use anyfs_backend::VfsBackendExt;

// JSON support
let config: Config = fs.read_json("/config.json")?;
fs.write_json("/config.json", &config)?;

// Type checks
if fs.is_file("/path")? { ... }
if fs.is_dir("/path")? { ... }
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

// Links (requires FeatureGuard)
fs.symlink("/target", "/link")?;
fs.hard_link("/original", "/link")?;
fs.read_link("/link")?;                  // -> PathBuf

// Permissions (requires FeatureGuard)
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
    // Path errors
    Err(VfsError::NotFound { path, operation }) => {
        // e.g., path="/file.txt", operation="read"
    }
    Err(VfsError::AlreadyExists { path, operation }) => ...
    Err(VfsError::NotADirectory { path }) => ...
    Err(VfsError::NotAFile { path }) => ...
    Err(VfsError::DirectoryNotEmpty { path }) => ...

    // Quota middleware errors
    Err(VfsError::QuotaExceeded { limit, requested, usage }) => ...
    Err(VfsError::FileSizeExceeded { path, size, limit }) => ...

    // FeatureGuard middleware errors
    Err(VfsError::FeatureNotEnabled { feature, operation }) => ...

    // PathFilter middleware errors
    Err(VfsError::AccessDenied { path, reason }) => ...

    // ReadOnly middleware errors
    Err(VfsError::ReadOnly { operation }) => ...

    // RateLimit middleware errors
    Err(VfsError::RateLimitExceeded { limit, window_secs }) => ...

    // VfsBackendExt errors
    Err(VfsError::Serialization(msg)) => ...
    Err(VfsError::Deserialization(msg)) => ...

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
| `Quota<B>` | Quota enforcement |
| `FeatureGuard<B>` | Least privilege |
| `PathFilter<B>` | Path-based access control |
| `ReadOnly<B>` | Prevent write operations |
| `RateLimit<B>` | Operation throttling |
| `Tracing<B>` | Instrumentation (tracing ecosystem) |
| `DryRun<B>` | Log without executing |
| `Cache<B>` | LRU read caching |
| `Overlay<B1,B2>` | Union filesystem |

---

## Layers

| Layer | Creates |
|-------|---------|
| `QuotaLayer` | `Quota<B>` |
| `FeatureGuardLayer` | `FeatureGuard<B>` |
| `PathFilterLayer` | `PathFilter<B>` |
| `ReadOnlyLayer` | `ReadOnly<B>` |
| `RateLimitLayer` | `RateLimit<B>` |
| `TracingLayer` | `Tracing<B>` |
| `DryRunLayer` | `DryRun<B>` |
| `CacheLayer` | `Cache<B>` |
| `OverlayLayer` | `Overlay<B1,B2>` |

---

## Type Reference

### From `anyfs-backend`

| Type | Description |
|------|-------------|
| `VfsBackend` | Core trait |
| `Layer` | Middleware composition trait |
| `VfsBackendExt` | Extension methods trait |
| `VfsError` | Error type (with context) |
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
| `Quota<B>` | Quota middleware |
| `FeatureGuard<B>` | Feature gate middleware |
| `PathFilter<B>` | Path access control middleware |
| `ReadOnly<B>` | Read-only middleware |
| `RateLimit<B>` | Rate limiting middleware |
| `Tracing<B>` | Tracing middleware |
| `DryRun<B>` | Dry-run middleware |
| `Cache<B>` | Caching middleware |
| `Overlay<B1,B2>` | Union filesystem middleware |
| `QuotaLayer` | Layer for Quota |
| `FeatureGuardLayer` | Layer for FeatureGuard |
| `PathFilterLayer` | Layer for PathFilter |
| `ReadOnlyLayer` | Layer for ReadOnly |
| `RateLimitLayer` | Layer for RateLimit |
| `TracingLayer` | Layer for Tracing |
| `DryRunLayer` | Layer for DryRun |
| `CacheLayer` | Layer for Cache |
| `OverlayLayer` | Layer for Overlay |

### From `anyfs-container`

| Type | Description |
|------|-------------|
| `FilesContainer<B>` | Ergonomic wrapper |
| `BackendStack` | Fluent builder for middleware stacks |
