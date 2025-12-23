# FilesContainer (anyfs-container)

**The user-facing API: ergonomic paths + policy (limits + feature whitelist)**

---

## Overview

`FilesContainer<B: VfsBackend>` wraps a backend and provides:
- ergonomic path inputs (`impl AsRef<Path>`)
- centralized path normalization
- quota enforcement (limits)
- deny-by-default feature whitelisting for advanced behavior

---

## Creating a Container

```rust
use anyfs::SqliteBackend;
use anyfs_container::FilesContainer;

let mut container = FilesContainer::new(SqliteBackend::open_or_create("data.db")?)
    .with_max_total_size(100 * 1024 * 1024)
    .with_max_file_size(10 * 1024 * 1024)
    // advanced features are opt-in
    .with_symlinks()
    .with_max_symlink_resolution(40)
    .with_hard_links()
    .with_permissions();
```

---

## std::fs-aligned Methods

Most methods map directly to `std::fs` naming:

| FilesContainer | std::fs |
|---------------|---------|
| `read()` | `std::fs::read` |
| `read_to_string()` | `std::fs::read_to_string` |
| `write()` | `std::fs::write` |
| `append()` | `OpenOptions::append` |
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

## Feature Whitelist (Least Privilege)

Advanced capabilities are disabled by default and must be explicitly enabled:

- `symlinks()` enables symlink operations and symlink following
- `hard_links()` enables hard-link creation
- `permissions()` enables `set_permissions`

If disabled, the operation returns `ContainerError::FeatureNotEnabled("...")`.

---

## Limits (Quotas)

Limits are enforced by the container, consistently across backends:
- total bytes
- max file size
- max nodes
- max directory entries
- max path depth

See `book/src/getting-started/guide.md` for usage examples.