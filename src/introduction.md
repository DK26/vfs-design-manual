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
│  FileStorage<M>                         │  ← Ergonomics + type-safe marker
├─────────────────────────────────────────┤
│  Middleware (composable):               │
│    Quota<B>                             │  ← Quotas
│    Restrictions<B>                      │  ← Security
│    Tracing<B>                           │  ← Audit
├─────────────────────────────────────────┤
│  Fs                             │  ← Storage
│  (Memory, SQLite, VRootFs, custom...)   │
└─────────────────────────────────────────┘
```

**Each layer has one job.** Compose only what you need.

---

## Two-Crate Structure

| Crate             | Purpose                                      |
| ----------------- | -------------------------------------------- |
| `anyfs-backend`   | Minimal contract: `Fs` trait + types         |
| `anyfs`           | Backends + middleware + ergonomic `FileStorage<M>` wrapper |

---

## Quick Example

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, FileStorage};

// Layer-based composition
let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(RestrictionsLayer::builder()
        .deny_hard_links()
        .deny_permissions()
        .build());

let mut fs = FileStorage::new(backend);

fs.create_dir_all("/data")?;
fs.write("/data/file.txt", b"hello")?;
```

---

## How to Use This Manual

| Section               | Audience        | Purpose                |
| --------------------- | --------------- | ---------------------- |
| Overview              | Stakeholders    | One-page understanding |
| Getting Started       | Developers      | Practical examples     |
| Design & Architecture | Contributors    | Detailed design        |
| Traits & APIs         | Backend authors | Contract and types     |
| Implementation        | Implementers    | Plan + backend guide   |

---

## Status

| Component      | Status      |
| -------------- | ----------- |
| Design         | Complete    |
| Implementation | Not started |

---

## Authoritative Documents

1. `AGENTS.md` (for AI assistants)
2. `book/src/architecture/design-overview.md`
3. `book/src/architecture/adrs.md`
