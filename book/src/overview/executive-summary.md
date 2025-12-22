# AnyFS — Executive Summary

**One-page overview for stakeholders and decision-makers**

---

## What Is It?

AnyFS is a **virtual filesystem library** for Rust with swappable storage backends. Store files in SQLite, memory, or the host filesystem — your code stays the same.

```
Traditional Filesystem          AnyFS Container
─────────────────────          ─────────────────
/home/user/docs/               ┌─────────────────┐
├── report.pdf        →        │  tenant_1.db    │  ← Single portable file
├── notes.txt                  │  (SQLite)       │
└── images/                    └─────────────────┘
    └── photo.jpg
```

---

## Why Does It Matter?

| Problem | How AnyFS Solves It |
|---------|---------------------|
| **Multi-tenant isolation** | Each tenant gets their own container. Complete namespace isolation. |
| **Portability** | SQLite backend: single file works on Windows, Mac, Linux. |
| **Security** | Path traversal attacks are structurally impossible via `VirtualPath`. |
| **Resource control** | Built-in quotas: max storage, max file size, max files. |
| **Testing** | In-memory backend for fast, deterministic tests. No temp file cleanup. |

---

## Who Is It For?

- **SaaS platforms** needing per-tenant file storage
- **AI/ML systems** requiring sandboxed file operations
- **Desktop applications** wanting portable user data
- **Testing frameworks** needing reproducible filesystem state
- **Embedded systems** needing portable storage without OS dependencies

---

## Key Properties

| Property | Guarantee |
|----------|-----------|
| **Isolated** | Virtual paths never touch the host filesystem |
| **Portable** | SQLite backend: single file, cross-platform |
| **Bounded** | Configurable limits on size, file count, depth |
| **Extensible** | Pluggable backends (SQLite, memory, host FS, custom) |

---

## What It's NOT

- Not a replacement for OS filesystems
- Not a container runtime (Docker, etc.)
- Not a distributed filesystem
- Not optimized for maximum throughput

---

## Technical Approach

Three crates with clear separation of concerns:

```
┌─────────────────────────────────────┐
│  anyfs-container                    │  ← Your code talks to this
│  FilesContainer<B>                  │
│  - read, write, copy, delete        │
│  - quota enforcement                │
├─────────────────────────────────────┤
│  anyfs                              │  ← Built-in backends
├──────────┬──────────┬───────────────┤
│ VRootFs  │  Memory  │  SQLite       │
├──────────┴──────────┴───────────────┤
│  anyfs-traits                       │  ← VfsBackend trait
│  (for custom backend implementers)  │
└─────────────────────────────────────┘
```

This means:
- Swapping storage backends doesn't change application code
- Custom backends only need to depend on `anyfs-traits`
- All filesystem complexity is handled once, in the core

---

## Project Status

| Phase | Status |
|-------|--------|
| Design | Complete |
| Implementation | Not started |

---

## Quick Example

```rust
use anyfs::SqliteBackend;
use anyfs_container::{FilesContainer, ContainerBuilder};

// Create a container backed by SQLite
let mut container = ContainerBuilder::new(
        SqliteBackend::open_or_create("tenant_123.db")?
    )
    .max_total_size(100 * 1024 * 1024)  // 100 MB quota
    .build()?;

// Use familiar filesystem operations
container.mkdir("/documents")?;
container.write("/documents/hello.txt", b"Hello!")?;

let content = container.read("/documents/hello.txt")?;
// content == b"Hello!"
```

---

*For technical details, see the [Design Overview](../architecture/design-overview.md).*
