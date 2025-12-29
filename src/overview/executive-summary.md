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
│  FileStorage<B>                      │  ← Ergonomics (std::fs API)
├─────────────────────────────────────────┤
│  Middleware (composable):               │
│    Quota<B>                    │  ← Quotas
│    Restrictions<B>               │  ← Security
│    Tracing<B>                    │  ← Audit
├─────────────────────────────────────────┤
│  Fs                             │  ← Storage
└─────────────────────────────────────────┘
```

---

## Why does it matter?

| Problem | How AnyFS helps |
|---------|-----------------|
| Multi-tenant isolation | Separate backend instances per tenant |
| Portability | SQLite backend: tenant data = single `.db` file |
| Security | Restrictions disables dangerous features by default |
| Resource control | Quota enforces quotas |
| Audit compliance | Tracing records all operations |
| Custom storage | Implement Fs for any medium |

---

## Key design points

- **Two-crate structure**
  - `anyfs-backend`: trait contract
  - `anyfs`: backends + middleware + ergonomic wrapper

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
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, FileStorage};

// Layer-based composition
let backend = SqliteBackend::open("tenant.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(RestrictionsLayer::builder()
        .deny_hard_links()
        .deny_permissions()
        .build());

let mut fs = FileStorage::new(backend);

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
