# AnyFS Ecosystem

**An open standard for pluggable virtual filesystem backends in Rust.**

---

## Overview

AnyFS is an **open standard** for virtual filesystem backends using a **Tower-style middleware pattern** for composable functionality.

You get:
- A familiar `std::fs`-aligned API
- Composable middleware (limits, logging, security)
- Choice of storage: memory, SQLite, host filesystem, or custom
- A developer-first goal: make storage composition easy, safe, and enjoyable

---

## Architecture

```
┌─────────────────────────────────────────┐
│  FileStorage<B, M>                      │  ← Ergonomics + type-safe marker
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

| Crate           | Purpose                                                             |
| --------------- | ------------------------------------------------------------------- |
| `anyfs-backend` | Minimal contract: `Fs` trait + types                                |
| `anyfs`         | Backends + middleware + ergonomic `FileStorage<B, M>` wrapper       |
| `anyfs-mount`   | Companion crate for FUSE/WinFsp mounting (planned; roadmap defined) |

**Note:** The core design is two crates (`anyfs-backend`, `anyfs`). `anyfs-mount` is a planned companion crate (design complete; implementation pending), not part of the core crates.

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
        .deny_permissions()
        .build());

let fs = FileStorage::new(backend);

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

## Future Ideas (Post-v1)

These are optional extensions that fit the design but are intentionally out of scope for v1:

- URL-based backend registry and bulk helpers (`FsExt`/utilities)
- Async adapter crate for remote backends (`anyfs-async`)
- Companion shell (`anyfs-shell`) for interactive exploration
- Copy-on-write overlay and archive backends (zip/tar) as separate crates

See [Design Overview](./architecture/design-overview.md#future-ideas-post-v1) for the full list and rationale.

---

## Status

| Component      | Status                                 |
| -------------- | -------------------------------------- |
| Design         | Complete                               |
| Implementation | Not started (mounting roadmap defined) |

---

## Authoritative Documents

1. `AGENTS.md` (for AI assistants)
2. `src/architecture/design-overview.md`
3. `src/architecture/adrs.md`
