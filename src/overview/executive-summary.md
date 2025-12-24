# AnyFS - Executive Summary

One-page overview for stakeholders and decision-makers.

---

## What is it?

AnyFS is an **open standard** for pluggable virtual filesystem backends in Rust, using a **Tower-style middleware pattern** for composable functionality.

You get:
- A familiar `std::fs`-aligned API
- Composable middleware for limits, logging, and security
- Choice of storage: memory, SQLite, host filesystem, or custom

---

## Architecture

```
┌─────────────────────────────────────────┐
│  FilesContainer<B>                      │  ← Ergonomics (std::fs API)
├─────────────────────────────────────────┤
│  Middleware (composable):               │
│    LimitedBackend<B>                    │  ← Quotas
│    FeatureGatedBackend<B>               │  ← Security
│    LoggingBackend<B>                    │  ← Audit
├─────────────────────────────────────────┤
│  VfsBackend                             │  ← Storage
└─────────────────────────────────────────┘
```

---

## Why does it matter?

| Problem | How AnyFS helps |
|---------|-----------------|
| Multi-tenant isolation | Separate backend instances per tenant |
| Portability | SQLite backend: tenant data = single `.db` file |
| Security | FeatureGatedBackend disables dangerous features by default |
| Resource control | LimitedBackend enforces quotas |
| Audit compliance | LoggingBackend records all operations |
| Custom storage | Implement VfsBackend for any medium |

---

## Key design points

- **Three-crate structure**
  - `anyfs-backend`: trait contract
  - `anyfs`: backends + middleware
  - `anyfs-container`: ergonomic wrapper

- **Middleware pattern** (like Axum/Tower)
  - Each middleware has one job
  - Compose only what you need
  - Complete separation of concerns

- **std::fs alignment**
  - Familiar method names
  - `impl AsRef<Path>` everywhere

---

## Quick example

```rust
use anyfs::{SqliteBackend, LimitedBackend, FeatureGatedBackend};
use anyfs_container::FilesContainer;

// Compose: storage + limits + security
let backend = FeatureGatedBackend::new(
    LimitedBackend::new(SqliteBackend::open("tenant.db")?)
        .with_max_total_size(100 * 1024 * 1024)
)
.with_symlinks();

let mut fs = FilesContainer::new(backend);

fs.create_dir_all("/documents")?;
fs.write("/documents/hello.txt", b"Hello!")?;
```

---

## Status

| Phase | Status |
|-------|--------|
| Design | Complete |
| Implementation | Not started |

---

For details, see `book/src/architecture/design-overview.md`.
