# AnyFS Ecosystem

**An open standard for pluggable virtual filesystem backends in Rust.**

---

## Overview

AnyFS is an **open standard** for virtual filesystem backends using a **Tower-style middleware pattern** for composable functionality.

You get:
- A familiar `std::fs`-aligned API
- Composable middleware (limits, logging, security)
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
│  (Memory, SQLite, VRootFs, custom...)   │
└─────────────────────────────────────────┘
```

**Each layer has one job.** Compose only what you need.

---

## Three-Crate Structure

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Minimal contract: `VfsBackend` trait + types |
| `anyfs` | Backends + middleware |
| `anyfs-container` | Ergonomic `FilesContainer<B>` wrapper |

---

## Quick Example

```rust
use anyfs::{SqliteBackend, LimitedBackend, FeatureGatedBackend};
use anyfs_container::FilesContainer;

// Compose: storage -> limits -> security
let backend = FeatureGatedBackend::new(
    LimitedBackend::new(SqliteBackend::open("data.db")?)
        .with_max_total_size(100 * 1024 * 1024)
)
.with_symlinks();

let mut fs = FilesContainer::new(backend);

fs.create_dir_all("/data")?;
fs.write("/data/file.txt", b"hello")?;
```

---

## How to Use This Manual

| Section | Audience | Purpose |
|---------|----------|---------|
| Overview | Stakeholders | One-page understanding |
| Getting Started | Developers | Practical examples |
| Design & Architecture | Contributors | Detailed design |
| Traits & APIs | Backend authors | Contract and types |
| Implementation | Implementers | Plan + backend guide |

---

## Status

| Component | Status |
|-----------|--------|
| Design | Complete |
| Implementation | Not started |

---

## Authoritative Documents

1. `AGENTS.md` (for AI assistants)
2. `book/src/architecture/design-overview.md`
3. `book/src/architecture/adrs.md`
