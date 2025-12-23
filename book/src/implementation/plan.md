# Implementation Plan

This plan describes a phased rollout of the **current** AnyFS design:
- `anyfs-traits`: minimal core trait + types
- `anyfs`: built-in backends (feature-gated)
- `anyfs-container`: quotas + isolation wrapper

It intentionally does **not** implement the historical graph-store design.

---

## Phase 1: `anyfs-traits` (the contract)

**Goal:** Define the stable API surface that all backends implement.

- Re-export `strict_path::VirtualPath`
- Define `VfsBackend` with 20 `std::fs`-aligned methods (including symlinks, hard links, permissions)
- Define core types (`Metadata`, `DirEntry`, `FileType`, `Permissions`)
- Define `VfsError` (errors carry `VirtualPath`, not strings)

**Exit criteria:** Downstream crates can implement a backend without depending on `anyfs` or `anyfs-container`.

---

## Phase 2: `anyfs` (built-in backends)

**Goal:** Provide reference implementations behind Cargo features.

- `memory` (default): `MemoryBackend` for tests and as the behavioral reference
- `vrootfs`: `VRootFsBackend` backed by a real directory, contained via `strict_path::VirtualRoot`
- `sqlite`: `SqliteBackend` storing the filesystem inside a single `.db` file

**Exit criteria:** Each backend implements the full `VfsBackend` contract and matches shared conformance tests.

---

## Phase 3: `anyfs-container` (policy layer)

**Goal:** Provide the ergonomic, safe API surface for application code.

- `FilesContainer<B: VfsBackend>` methods take `impl AsRef<Path>` and validate once into `VirtualPath`
- Enforce quotas/limits (total size, max file size, max entries, etc.) consistently across operations
- Default-deny feature whitelist (least privilege): enable symlinks/hard links/permissions explicitly
- Provide `ContainerBuilder` for configuring limits with sane defaults
- Define `ContainerError` that cleanly separates backend failures vs. limit violations

**Exit criteria:** Application code never touches `VirtualPath` for common usage, and all validation is centralized.

---

## Phase 4: Conformance test suite (recommended)

**Goal:** Prevent backend divergence and document expected semantics.

Add a shared test module/crate that can run the same tests against multiple backends:
- Basic file/dir operations (`read`, `write`, `create_dir_all`, `remove_*`, `rename`, `copy`)
- Symlink semantics (`read_link`, loop detection, fixed hop limit, metadata vs. symlink_metadata)
- Hard link semantics (`nlink`, shared content behavior, rename/remove interactions)
- Error mapping consistency (`NotFound`, `AlreadyExists`, `NotADirectory`, etc.)
- Container limits enforcement (quota exceeded errors, partial-write avoidance)

**Exit criteria:** Memory/VRootFs/SQLite backends all pass the same suite.

---

## Phase 5: Documentation and examples

**Goal:** Make the design hard to misunderstand.

- Keep `book/src/architecture/design-overview.md` and ADRs authoritative
- Ensure all examples use current names (`anyfs`, `anyfs-traits`, `anyfs-container`) and current method names
- Document portability expectations: "single file" is a property of the SQLite backend, not the trait

---

## Future Work (explicitly out of scope for MVP)

- Streaming I/O (file handles) and/or chunked writes
- Async API (separate trait or wrapper)
- Import/export helpers (e.g., SQLAR import/export) as tooling, not core semantics
- Extended attributes / richer permissions model (platform-dependent)
