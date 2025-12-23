# AnyFS - Project Structure

**Status:** Current
**Last updated:** 2025-12-23

---

## Repository Layout

```
anyfs-backend/             # Crate 1: core trait + types
  Cargo.toml
  src/
    lib.rs
    backend.rs
    types.rs
    error.rs

anyfs/                     # Crate 2: built-in backends (feature-gated)
  Cargo.toml
  src/
    lib.rs                 # backend implementations
    memory/                # [feature: memory] (default)
    vrootfs/               # [feature: vrootfs]
    sqlite/                # [feature: sqlite]

anyfs-container/           # Crate 3: policy layer (quotas + isolation)
  Cargo.toml
  src/
    lib.rs
    container.rs           # FilesContainer<B: VfsBackend>
    limits.rs              # CapacityLimits
    usage.rs               # CapacityUsage, CapacityRemaining
    error.rs               # ContainerError
```

---

## Dependency Model

```
anyfs-backend (trait + types)
    <- anyfs (execution layer, calls any VfsBackend)
    <- anyfs-container (wraps backends with policy)

strict-path (VirtualPath, VirtualRoot)
    <- anyfs [vrootfs feature only]
```

**Key point:** Custom backends depend only on `anyfs-backend`. The `anyfs` crate is the execution layer that can call any backend—built-in or custom.

---

## Path Flow

```
User code:  container.read("/a/b.txt")
   -> FilesContainer (policy checks, normalization)
   -> backend.read(path)  # receives impl AsRef<Path>
```

Both layers use `impl AsRef<Path>` for std::fs alignment.

---

## Cargo Features

Cargo features in `anyfs` select which built-in backends to include:

- `memory` — In-memory storage (default)
- `sqlite` — SQLite-backed persistent storage
- `vrootfs` — Host filesystem backend (uses `strict-path` internally)

The runtime feature whitelist (symlinks, hard_links, permissions) is configured per `FilesContainer` instance, not at compile time.

---

## Where To Start

- Application usage: `book/src/getting-started/guide.md`
- Trait details: `book/src/traits/vfs-trait.md`
- Decisions and rationale: `book/src/architecture/adrs.md`