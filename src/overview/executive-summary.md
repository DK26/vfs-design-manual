# AnyFS - Executive Summary

One-page overview for stakeholders and decision-makers.

---

## What is it?

AnyFS is an **open standard** for pluggable virtual filesystem backends in Rust.

You get a familiar, `std::fs`-aligned API (read/write/create_dir/read_dir/etc.) while choosing where the data lives:
- in-memory (fast caching, testing, speed-critical applications)
- a single SQLite database file (portable storage)
- a contained host filesystem directory (sandboxed by VRootFsBackend)
- or any custom backend you implement

---

## Why does it matter?

| Problem | How AnyFS helps |
|---------|------------------|
| Multi-tenant isolation | One container per tenant, separate namespaces |
| Portability | SQLite backend: a tenant filesystem is a single `.db` file |
| Security | Path normalization prevents traversal attacks |
| Resource control | Built-in limits: max bytes, max file size, max nodes, etc. |
| Speed-critical storage | In-memory backend for caching or temporary storage |
| Custom storage | Implement your own backend for any storage medium |

---

## Key design points

- **Three-crate structure**
  - `anyfs-backend`: minimal backend contract (`VfsBackend` trait) and types
  - `anyfs`: low-level execution layer for calling any `VfsBackend` (also provides built-in backends)
  - `anyfs-container`: `FilesContainer<B>` policy layer (limits + least privilege)

- **Simple path handling**
  - User APIs accept `impl AsRef<Path>` for ergonomics.
  - Backends receive normalized `&str` paths.
  - `strict-path` is only used internally by VRootFsBackend.

- **Least privilege by default**
  - Advanced behavior is denied unless explicitly enabled per container:
    - symlinks
    - hard links
    - permission mutation

---

## Quick example

```rust
use anyfs::SqliteBackend;
use anyfs_container::FilesContainer;

let mut fs = FilesContainer::new(SqliteBackend::open_or_create("tenant_123.db")?)
    .with_max_total_size(100 * 1024 * 1024);

fs.create_dir_all("/documents")?;
fs.write("/documents/hello.txt", b"Hello!")?;
let content = fs.read("/documents/hello.txt")?;
```

---

## Status

| Phase | Status |
|-------|--------|
| Design | Complete |
| Implementation | Not started |

---

For details, see `book/src/architecture/design-overview.md`.
