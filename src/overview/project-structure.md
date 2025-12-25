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

anyfs/                      # Crate 2: backends + middleware + ergonomics
  Cargo.toml
  src/
    lib.rs
    backends/
      memory.rs             # MemoryBackend [feature: memory, default]
      sqlite.rs             # SqliteBackend [feature: sqlite]
      stdfs.rs              # StdFsBackend [feature: stdfs]
      vrootfs.rs            # VRootFsBackend [feature: vrootfs]
    middleware/
      quota.rs              # Quota<B>
      tracing.rs            # Tracing<B>
      feature_guard.rs      # Restrictions<B>
    container.rs            # FileStorage<M>
    stack.rs                # BackendStack builder
```

---

## Dependency Model

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware + ergonomics)
           ^-- vrootfs feature uses strict-path
```

**Key points:**
- Custom backends depend only on `anyfs-backend`
- `anyfs` provides built-in backends, middleware, and the ergonomic `FileStorage<M>` wrapper

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
- `stdfs` — Direct `std::fs` delegation (no containment)
- `vrootfs` — Host filesystem backend with path containment (uses `strict-path`)

Middleware is always available (no feature flags).

---

## Where To Start

- Application usage: `book/src/getting-started/guide.md`
- Trait details: `book/src/traits/layered-traits.md`
- Middleware: `book/src/architecture/design-overview.md`
- Decisions: `book/src/architecture/adrs.md`
