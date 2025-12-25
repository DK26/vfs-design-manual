# FilesContainer (anyfs-container)

**Thin ergonomic wrapper for std::fs-aligned API**

---

## Overview

`FilesContainer<B: VfsBackend>` is a **thin wrapper** that provides a familiar std::fs-aligned API.

**It does ONE thing:** Ergonomics. Nothing else.

All policy (limits, feature gates, logging) is handled by middleware, not FilesContainer.

---

## Creating a Container

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

// Simple: just ergonomics
let mut fs = FilesContainer::new(MemoryBackend::new());
```

With middleware (layer-based):

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer};
use anyfs_container::FilesContainer;

let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::new()
        .max_total_size(100 * 1024 * 1024))
    .layer(RestrictionsLayer::new()
        .deny_hard_links()
        .deny_permissions());

let mut fs = FilesContainer::new(backend);
```

---

## std::fs-aligned Methods

FilesContainer mirrors std::fs naming:

| FilesContainer | std::fs |
|---------------|---------|
| `read()` | `std::fs::read` |
| `read_to_string()` | `std::fs::read_to_string` |
| `write()` | `std::fs::write` |
| `read_dir()` | `std::fs::read_dir` |
| `create_dir()` | `std::fs::create_dir` |
| `create_dir_all()` | `std::fs::create_dir_all` |
| `remove_file()` | `std::fs::remove_file` |
| `remove_dir()` | `std::fs::remove_dir` |
| `remove_dir_all()` | `std::fs::remove_dir_all` |
| `rename()` | `std::fs::rename` |
| `copy()` | `std::fs::copy` |
| `metadata()` | `std::fs::metadata` |
| `symlink_metadata()` | `std::fs::symlink_metadata` |
| `read_link()` | `std::fs::read_link` |
| `set_permissions()` | `std::fs::set_permissions` |

---

## What FilesContainer Does NOT Do

| Concern | Use Instead |
|---------|-------------|
| Quota enforcement | `Quota<B>` |
| Feature gating | `Restrictions<B>` |
| Audit logging | `Tracing<B>` |
| Path containment | Backend-specific (VRootFsBackend) |

FilesContainer is **purely ergonomic**. If you need policy, compose middleware.

---

## Layer Helper

For Axum-style composition:

```rust
let fs = FilesContainer::new(SqliteBackend::open("data.db")?)
    .layer(QuotaLayer::new().max_total_size(100 * 1024 * 1024))
    .layer(RestrictionsLayer::new().deny_hard_links());  // Block hard_link() calls
```

---

## Direct Backend Access

If you don't need ergonomics, use backends directly:

```rust
use anyfs::{MemoryBackend, Quota};
use anyfs_backend::VfsBackend;

let mut backend = Quota::new(MemoryBackend::new())
    .with_max_total_size(100 * 1024 * 1024);

// Use VfsBackend methods directly
backend.write("/file.txt", b"data")?;
```
