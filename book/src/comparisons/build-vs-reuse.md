# AnyFS Container - Build vs. Reuse Analysis

**Can your goals be achieved with existing crates, or does this project need to exist?**

---

## Core Requirements

1. **Tenant isolation** - each tenant gets an isolated namespace
2. **Capacity limits** - per-tenant quotas and safe failure modes
3. **Portable storage option** - a single-file backend (SQLite) for easy move/copy/backup
4. **Filesystem semantics** - `std::fs`-aligned operations including opt-in symlinks and hard links
5. **Containment by construction** - prevent path traversal and symlink escapes

---

## What Already Exists

### `vfs` crate (Rust)

**What it is:** A general virtual filesystem abstraction with multiple backends.

**What it does well:**
- Provides an abstraction layer similar to `std::fs`
- Has multiple existing backends

**What it does not provide (out of the box):**
- SQLite-backed portable storage
- Capacity limits / quota enforcement
- Built-in tenant isolation patterns
- Two-layer path handling (ergonomic user paths + validated internal paths)

You *can* implement a SQLite backend for `vfs`, but you still need to design quotas + isolation yourself.

### `rusqlite`

**What it is:** SQLite bindings.

**What it provides:** DB access, transactions, blobs (with features), migrations.

**What it does not provide:** Filesystem semantics or path safety.

### `strict-path`

**What it is:** Path validation + containment primitives (`VirtualRoot`, `VirtualPath`).

**What it provides:** The "can't escape the root" guarantee that the whole design depends on.

**What it does not provide:** Storage backends (SQLite, memory, etc.) or a filesystem API.

### SQLAR (SQLite archive format)

**What it is:** A standardized "files in a SQLite table" archive format.

**Why it's not enough:** It's an archive format, not a filesystem API with quotas, isolation, and link semantics.

---

## Gap Analysis

| Requirement | `vfs` crate | SQLAR | `rusqlite` | `strict-path` |
|-------------|------------:|------:|-----------:|--------------:|
| Filesystem API | Yes | No | No | No |
| SQLite-backed portable storage | No | Yes (format) | Yes (raw) | No |
| Tenant isolation model | No | No | Manual | No |
| Capacity limits / quotas | No | No | Manual | No |
| Path traversal safety | Backend-dependent | Manual | Manual | Yes |
| Link semantics (symlink + hard link) | Backend-dependent | Not the focus | Manual | N/A |

**Conclusion:** You can reuse key building blocks, but no existing crate composes into:
> "A tenant-isolated filesystem API with quotas, backed by SQLite, with containment guarantees."

---

## Options

### Option A: Implement a SQLite backend for the `vfs` crate

**Pros**
- Immediate ecosystem compatibility
- Familiar trait surface area

**Cons**
- Still need to build tenant isolation + quotas outside the trait
- Path containment becomes backend/adapter responsibility
- Streaming handle APIs can increase SQLite complexity

### Option B: Current AnyFS architecture (recommended)

**Approach:** Keep responsibilities separated into three crates:
- `anyfs-traits`: `VfsBackend` + types, uses `&VirtualPath`
- `anyfs`: feature-gated built-in backends (`MemoryBackend`, `VRootFsBackend`, `SqliteBackend`)
- `anyfs-container`: `FilesContainer` wrapper (ergonomic `impl AsRef<Path>` + quotas)

**Why this exists:** It makes containment a property of the type system and concentrates validation in one place:
`FilesContainer` validates user paths once, then calls the backend with `&VirtualPath`.

### Option C: Stay on raw `rusqlite` (+ your own helpers)

Fastest to prototype, but it becomes an ad-hoc filesystem without a stable contract, and quotas/isolation are easy to get wrong.

### Option D: Add a compatibility adapter later (optional)

Implement AnyFS as designed, then (optionally) provide an adapter that implements `vfs` traits on top of `FilesContainer`.

---

## Minimal Example (per-tenant SQLite file)

```rust
use anyfs::SqliteBackend;
use anyfs_container::ContainerBuilder;

let tenant_db = "tenant_123.db";

let mut container = ContainerBuilder::new(SqliteBackend::open_or_create(tenant_db)?)
    .max_total_size(100 * 1024 * 1024) // 100 MiB per tenant
    .build()?;

container.create_dir_all("/documents")?;
container.write("/documents/report.pdf", b"...")?;

// Portability: moving/copying the tenant is just moving/copying the DB file.
std::fs::copy(tenant_db, "tenant_123.backup.db")?;
```

---

## Recommendation

Build on existing primitives (`strict-path`, `rusqlite`, `thiserror`), but keep the AnyFS split (traits / backends / container). That combination is what makes the design both ergonomic and hard to misuse.

For a concrete phased rollout, see `book/src/implementation/plan.md`.
