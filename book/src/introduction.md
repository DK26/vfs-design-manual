# VFS Ecosystem

**Switchable virtual filesystem backends for Rust**

---

## Overview

Three crates:

| Crate | Purpose |
|-------|---------|
| `anyfs-traits` | Minimal — trait definition + types (for custom backend implementers) |
| `anyfs` | Built-in backends (feature-gated), re-exports traits |
| `anyfs-container` | Higher-level wrapper — capacity limits, tenant isolation |

```
┌─────────────────────────────────────────┐
│  Your Application                       │
├─────────────────────────────────────────┤
│  anyfs-container (quotas, isolation)    │
├─────────────────────────────────────────┤
│  anyfs (built-in backends)              │
├──────────┬──────────┬───────────────────┤
│ VRootFs  │  Memory  │  SQLite           │
├──────────┴──────────┴───────────────────┤
│  anyfs-traits (VfsBackend trait)        │
└─────────────────────────────────────────┘
```

## Quick Example

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

## How to Use This Manual

This manual is organized into logical sections for different audiences:

| Section | Audience | Purpose |
|---------|----------|---------|
| [Overview](./overview/executive-summary.md) | Stakeholders, Decision-makers | High-level understanding |
| [Getting Started](./getting-started/guide.md) | Developers | Practical introduction |
| [Design & Architecture](./architecture/design-overview.md) | Architects, Contributors | Technical deep-dive |
| [Comparisons](./comparisons/positioning.md) | Evaluators | How we compare to alternatives |
| [Implementation](./implementation/backend-guide.md) | Backend Implementers | Custom backend creation |
| [Review & Decisions](./review/decisions.md) | Contributors | Historical review record (graph-store era) |

## Status

| Component | Status |
|-----------|--------|
| Design | Completed |
| Implementation | Not started |

## Authoritative Documents

The following documents are authoritative sources of truth:

1. **[Design Overview](./architecture/design-overview.md)** - Current architecture and API
2. **[Architecture Decision Records](./architecture/adrs.md)** - Key design decisions
3. **[Project Structure](./overview/project-structure.md)** - Crate layout and type contracts

The `book/src/review/` documents are kept for historical context and describe an earlier (rejected) graph-store design.

---

*For questions about specific features, start with the [Getting Started Guide](./getting-started/guide.md).*
