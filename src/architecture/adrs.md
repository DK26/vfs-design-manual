# AnyFS - Architecture Decision Records

This file captures the decisions for the current AnyFS design.

---

## ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Path-based `VfsBackend` trait | Accepted |
| ADR-002 | Three-crate structure | Accepted |
| ADR-003 | `impl AsRef<Path>` for all path parameters | Accepted |
| ADR-004 | `strict-path` only for VRootFsBackend | Accepted |
| ADR-005 | `std::fs`-aligned method names | Accepted |
| ADR-006 | Least-privilege feature whitelist | Accepted |
| ADR-007 | Limits enforced in container (not backend) | Accepted |
| ADR-008 | Built-in backends are feature-gated | Accepted |

---

## ADR-001: Path-based `VfsBackend` trait

**Decision:** Backends implement a path-based trait aligned with `std::fs` method naming.

**Why:** Filesystem operations are naturally path-oriented; a single, familiar trait surface is easier to implement and adopt than graph-store or inode models.

---

## ADR-002: Three-crate structure

**Decision:**

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Minimal contract: `VfsBackend` trait + core types. Backend implementers depend on this. |
| `anyfs` | Execution layer that calls any `VfsBackend`. Provides built-in backends (feature-gated). |
| `anyfs-container` | Policy layer: `FilesContainer<B>` with limits + feature whitelist. |

**Why:**
- Backend authors only need `anyfs-backend` (no heavy dependencies).
- `anyfs` can execute operations on any backend (built-in or custom).
- Policy concerns (quotas, feature whitelist) are separated from storage concerns.

---

## ADR-003: `impl AsRef<Path>` for all path parameters

**Decision:** Both `VfsBackend` and `FilesContainer` accept `impl AsRef<Path>` for all path parameters.

**Why:**
- Aligned with `std::fs` API conventions.
- Works across all platforms (not limited to UTF-8).
- Ergonomic: accepts `&str`, `String`, `&Path`, `PathBuf`.

---

## ADR-004: `strict-path` only for VRootFsBackend

**Decision:** The `strict-path` crate is only used internally by `VRootFsBackend` for path containment. It is not part of the core AnyFS API.

**Why:**
- Virtual backends (memory, SQLite) handle containment differently (they're inherently isolated).
- Only the filesystem backend needs to clamp paths to a root directory.
- Keeps the core API simple and dependency-light.

---

## ADR-005: `std::fs`-aligned method names

**Decision:** Prefer `read_dir`, `create_dir_all`, `remove_file`, etc.

**Why:** Familiarity and reduced cognitive overhead.

---

## ADR-006: Least-privilege feature whitelist in `FilesContainer`

**Decision:** Advanced behavior is disabled by default and explicitly enabled per container instance.

- `.with_symlinks()` enables symlink creation and following (bounded by `max_symlink_resolution`, default 40)
- `.with_hard_links()` enables hard link creation
- `.with_permissions()` enables permission mutation via `set_permissions`

When disabled, the relevant operations return `ContainerError::FeatureNotEnabled("...")`.

**Why:** Reduces attack surface by default. Applications opt into only what they need.

---

## ADR-007: Limits enforced in container (not backend)

**Decision:** Capacity limits are enforced at the `anyfs-container` layer.

**Why:** Limits are a policy decision and should be consistent across all backends.

---

## ADR-008: Built-in backends are feature-gated

**Decision:** `anyfs` uses Cargo features so users only pull the dependencies they need.

- `memory` (default)
- `sqlite` (optional)
- `vrootfs` (optional)

**Why:** Minimizes binary size and compile time for users who don't need all backends.
