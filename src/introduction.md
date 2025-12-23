# AnyFS Ecosystem

**An open standard for pluggable virtual filesystem backends in Rust.**

---

## Overview

AnyFS is an **open standard** that allows anyone to create custom storage backends. Whether you need to store files in a database, object store, network filesystem, or any other medium—AnyFS provides the contract and tools to make it happen.

The ecosystem is a three-crate design:

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Minimal contract: `VfsBackend` trait + core types. Backend implementers depend on this. |
| `anyfs` | Low-level execution layer for calling any `VfsBackend`. Also provides built-in backends (MemoryBackend, SqliteBackend, VRootFsBackend). |
| `anyfs-container` | `FilesContainer<B: VfsBackend>` policy layer (limits + least-privilege feature whitelist) |

High-level data flow:

```text
Your application
  -> anyfs-container (FilesContainer: ergonomic paths + policy)
      -> anyfs (executes operations on any VfsBackend)
          -> any backend (built-in or custom)
```

> **Note:** The `strict-path` crate is **only** used by `VRootFsBackend`—a backend that wraps a real host filesystem directory. It is not part of the core AnyFS API.

---

## Quick example

```rust
use anyfs::MemoryBackend;
use anyfs_container::FilesContainer;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut container = FilesContainer::new(MemoryBackend::new());

    container.create_dir_all("/data")?;
    container.write("/data/file.txt", b"hello")?;

    Ok(())
}
```

---

## How to use this manual

| Section | Audience | Purpose |
|---------|----------|---------|
| Overview | Stakeholders | One-page understanding |
| Getting Started | Developers | Practical examples |
| Design & Architecture | Contributors | Detailed design |
| Traits & APIs | Backend authors | Contract and types |
| Implementation | Implementers | Plan + backend guide |
| Review | Contributors | Historical review record |

---

## Status

| Component | Status |
|-----------|--------|
| Design | Complete |
| Implementation | Not started |

---

## Authoritative documents

1. `book/src/architecture/design-overview.md`
2. `book/src/architecture/adrs.md`

If something conflicts with `AGENTS.md`, treat `AGENTS.md` as authoritative.
