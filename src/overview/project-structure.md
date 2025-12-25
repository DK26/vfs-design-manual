# AnyFS - Project Structure

**Status:** Current
**Last updated:** 2025-12-24

---

## Repository Layout

```
anyfs-backend/              # Crate 1: trait + types
  Cargo.toml
  src/
    lib.rs
    backend.rs              # Fs traits (FsRead, FsWrite, FsDir, etc.)
    types.rs                # Metadata, DirEntry, Permissions, StatFs
    error.rs                # FsError

anyfs/                      # Crate 2: backends + middleware
  Cargo.toml
  src/
    lib.rs
    backends/
      memory.rs             # MemoryBackend [feature: memory, default]
      sqlite.rs             # SqliteBackend [feature: sqlite]
      vrootfs.rs            # VRootFsBackend [feature: vrootfs]
    middleware/
      quota.rs              # Quota<B>
      tracing.rs            # Tracing<B>
      feature_guard.rs      # Restrictions<B>

anyfs-container/            # Crate 3: ergonomic wrapper
  Cargo.toml
  src/
    lib.rs
    container.rs            # FileStorage<B>
```

---

## Dependency Model

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware)
     |     ^-- vrootfs feature uses strict-path
     |
     +-- anyfs-container (ergonomic wrapper)
```

**Key points:**
- Custom backends depend only on `anyfs-backend`
- `anyfs` provides built-in backends and middleware
- `anyfs-container` provides the ergonomic `FileStorage` wrapper

---

## Middleware Pattern

```
FileStorage<B>
    wraps -> Tracing<B>
        wraps -> Restrictions<B>
            wraps -> Quota<B>
                wraps -> SqliteBackend (or any Fs)
```

Each layer implements `Fs`, enabling composition.

---

## Cargo Features

Features in `anyfs` select which backends to include:

- `memory` — In-memory storage (default)
- `sqlite` — SQLite-backed persistent storage
- `vrootfs` — Host filesystem backend (uses `strict-path`)

Middleware is always available (no feature flags).

---

## Where To Start

- Application usage: `book/src/getting-started/guide.md`
- Trait details: `book/src/traits/layered-traits.md`
- Middleware: `book/src/architecture/design-overview.md`
- Decisions: `book/src/architecture/adrs.md`
