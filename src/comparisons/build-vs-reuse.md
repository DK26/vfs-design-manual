# AnyFS - Build vs. Reuse Analysis

**Can your goals be achieved with existing crates, or does this project need to exist?**

---

## Core Requirements

1. **Backend flexibility** - swap storage without changing application code
2. **Composable middleware** - add/remove capabilities (quotas, sandboxing, logging)
3. **Tenant isolation** - each tenant gets an isolated namespace
4. **Portable storage** - single-file backend (SQLite) for easy move/copy/backup
5. **Filesystem semantics** - `std::fs`-aligned operations including symlinks and hard links
6. **Path containment** - prevent traversal attacks

---

## What Already Exists

### `vfs` crate (Rust)

**What it provides:**
- Filesystem abstraction with multiple backends
- MemoryFS, PhysicalFS, AltrootFS, OverlayFS, EmbeddedFS

**What it lacks:**
- SQLite backend
- Composable middleware pattern
- Quota/limit enforcement
- Policy layers (feature gating, path filtering)

### AgentFS (Turso)

**What it provides:**
- SQLite-based filesystem for AI agents
- Key-value store
- Tool call auditing
- FUSE mounting

**What it lacks:**
- Multiple backend types (SQLite only)
- Composable middleware
- Backend-agnostic abstraction

### `rusqlite`

**What it provides:** SQLite bindings, transactions, blobs.

**What it lacks:** Filesystem semantics, quota enforcement.

### `strict-path`

**What it provides:** Path validation and containment (`VirtualRoot`).

**What it lacks:** Storage backends, filesystem API.

---

## Gap Analysis

| Requirement | `vfs` | AgentFS | `rusqlite` | `strict-path` |
|-------------|:-----:|:-------:|:----------:|:-------------:|
| Filesystem API | Yes | Yes | No | No |
| Multiple backends | Yes | No | N/A | No |
| SQLite backend | No | Yes | Yes (raw) | No |
| Composable middleware | No | No | No | No |
| Quota enforcement | No | No | Manual | No |
| Path sandboxing | Partial | No | Manual | Yes |
| Symlink/hard link control | Backend-dep | Yes | Manual | N/A |

**Conclusion:** No existing crate provides:
> "Backend-agnostic filesystem abstraction with composable middleware for quotas, sandboxing, and policy enforcement."

---

## Why AnyFS Exists

AnyFS fills the gap by separating concerns:

| Crate | Responsibility |
|-------|----------------|
| `anyfs-backend` | Trait (`Fs`, `Layer`) + types |
| `anyfs` | Backends + middleware + ergonomic wrapper (`FileStorage<M>`) |

The middleware pattern (like Tower/Axum) enables composition:

```rust
use anyfs::{SqliteBackend, QuotaLayer, PathFilterLayer, RestrictionsLayer, TracingLayer, FileStorage};

let backend = SqliteBackend::open("tenant.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(RestrictionsLayer::builder()
        .deny_symlinks()
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .build())
    .layer(TracingLayer::new());

let mut fs = FileStorage::new(backend);
fs.write("/workspace/doc.txt", b"hello")?;
```

---

## Alternatives Considered

### Option A: Implement SQLite backend for `vfs` crate

**Pros:** Ecosystem compatibility.

**Cons:**
- No middleware pattern for quotas/policies
- Would still need to build quota/sandboxing outside the trait
- Doesn't solve the composability problem

### Option B: Use AgentFS

**Pros:** Already exists, SQLite-based, FUSE support.

**Cons:**
- Locked to SQLite (can't swap to memory/real FS)
- No composable middleware
- Includes KV store and auditing we may not need

### Option C: AnyFS (recommended)

**Pros:**
- Backend-agnostic (swap storage without code changes)
- Composable middleware (add/remove capabilities)
- Clean separation of concerns
- Third-party extensibility

**Cons:**
- New project, not yet widely adopted

---

## Recommendation

Build AnyFS with reusable primitives (`rusqlite`, `strict-path`, `thiserror`, `tracing`) but maintain the two-crate split. The middleware pattern is what makes the design both flexible and safe.

**Compatibility option:** Later, provide an adapter that implements `vfs` traits on top of `Fs` for projects that need `vfs` compatibility.
