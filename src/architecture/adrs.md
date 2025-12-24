# AnyFS - Architecture Decision Records

This file captures the decisions for the current AnyFS design.

---

## ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Path-based `VfsBackend` trait | Accepted |
| ADR-002 | Three-crate structure | Accepted |
| ADR-003 | `impl AsRef<Path>` for all path parameters | Accepted |
| ADR-004 | Tower-style middleware pattern | Accepted |
| ADR-005 | `std::fs`-aligned method names | Accepted |
| ADR-006 | LimitedBackend for quota enforcement | Accepted |
| ADR-007 | FeatureGatedBackend for least-privilege | Accepted |
| ADR-008 | FilesContainer as thin ergonomic wrapper | Accepted |
| ADR-009 | Built-in backends are feature-gated | Accepted |

---

## ADR-001: Path-based `VfsBackend` trait

**Decision:** Backends implement a path-based trait aligned with `std::fs` method naming.

**Why:** Filesystem operations are naturally path-oriented; a single, familiar trait surface is easier to implement and adopt than graph-store or inode models.

---

## ADR-002: Three-crate structure

**Decision:**

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Minimal contract: `VfsBackend` trait + types |
| `anyfs` | Backends + middleware (LimitedBackend, FeatureGatedBackend, LoggingBackend) |
| `anyfs-container` | Thin ergonomic wrapper: `FilesContainer<B>` |

**Why:**
- Backend authors only need `anyfs-backend` (no heavy dependencies).
- Middleware is composable and lives with backends in `anyfs`.
- `FilesContainer` is purely ergonomic - no policy logic.

---

## ADR-003: `impl AsRef<Path>` for all path parameters

**Decision:** Both `VfsBackend` and `FilesContainer` accept `impl AsRef<Path>` for all path parameters.

**Why:**
- Aligned with `std::fs` API conventions.
- Works across all platforms (not limited to UTF-8).
- Ergonomic: accepts `&str`, `String`, `&Path`, `PathBuf`.

---

## ADR-004: Tower-style middleware pattern

**Decision:** Use composable middleware (decorator pattern) for cross-cutting concerns like limits, logging, and feature gates. Each middleware implements `VfsBackend` by wrapping another `VfsBackend`.

**Why:**
- Complete separation of concerns - each layer has one job.
- Composable - use only what you need.
- Familiar pattern (Axum/Tower use the same approach).
- No code duplication - middleware written once, works with any backend.
- Testable - each layer can be tested in isolation.

**Example:**
```rust
let backend = LoggingBackend::new(
    FeatureGatedBackend::new(
        LimitedBackend::new(SqliteBackend::open("data.db")?)
    )
);
```

---

## ADR-005: `std::fs`-aligned method names

**Decision:** Prefer `read_dir`, `create_dir_all`, `remove_file`, etc.

**Why:** Familiarity and reduced cognitive overhead.

---

## ADR-006: LimitedBackend for quota enforcement

**Decision:** Quota/limit enforcement is handled by `LimitedBackend<B>` middleware, not by backends or FilesContainer.

**Configuration:**
- `with_max_total_size(bytes)` - total storage limit
- `with_max_file_size(bytes)` - per-file limit
- `with_max_node_count(count)` - max files/directories
- `with_max_dir_entries(count)` - max entries per directory
- `with_max_path_depth(depth)` - max directory nesting

**Why:**
- Limits are policy, not storage semantics.
- Written once, works with any backend.
- Optional - users who don't need limits skip this middleware.

---

## ADR-007: FeatureGatedBackend for least-privilege

**Decision:** Dangerous features (symlinks, hard links, permission mutation) are disabled by default via `FeatureGatedBackend<B>` middleware.

**Configuration:**
- `.with_symlinks()` - enable symlink creation/following
- `.with_hard_links()` - enable hard link creation
- `.with_permissions()` - enable `set_permissions`
- `.with_max_symlink_resolution(n)` - limit symlink hops (default: 40)

When disabled, operations return `VfsError::FeatureNotEnabled`.

**Why:**
- Reduces attack surface by default.
- Explicit opt-in for dangerous features.
- Separate from backend - works with any backend.

---

## ADR-008: FilesContainer as thin ergonomic wrapper

**Decision:** `FilesContainer<B>` is a thin wrapper that provides std::fs-aligned ergonomics only. It contains NO policy logic.

**What it does:**
- Provides familiar method names
- Accepts `impl AsRef<Path>` for convenience
- Delegates all operations to the wrapped backend

**What it does NOT do:**
- Quota enforcement (use LimitedBackend)
- Feature gating (use FeatureGatedBackend)
- Logging (use LoggingBackend)
- Any other policy

**Why:**
- Single responsibility - ergonomics only.
- Users who don't need ergonomics can use backends directly.
- Policy is composable via middleware, not hardcoded.

---

## ADR-009: Built-in backends are feature-gated

**Decision:** `anyfs` uses Cargo features so users only pull the dependencies they need.

- `memory` (default)
- `sqlite` (optional)
- `vrootfs` (optional)

**Why:** Minimizes binary size and compile time for users who don't need all backends.
