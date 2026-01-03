# AnyFS - Project Structure

**Status:** Target layout (design spec)
**Last updated:** 2025-12-24

---

This manual describes the intended code repository layout; this repository contains documentation only.

## Repository Layout

```
anyfs-backend/              # Crate 1: traits + types (minimal dependencies)
  Cargo.toml
  src/
    lib.rs
    traits/
      fs_read.rs            # FsRead trait
      fs_write.rs           # FsWrite trait
      fs_dir.rs             # FsDir trait
      fs_link.rs            # FsLink trait
      fs_permissions.rs     # FsPermissions trait
      fs_sync.rs            # FsSync trait
      fs_stats.rs           # FsStats trait
      fs_path.rs            # FsPath trait (canonicalization, blanket impl)
      fs_inode.rs           # FsInode trait
      fs_handles.rs         # FsHandles trait
      fs_lock.rs            # FsLock trait
      fs_xattr.rs           # FsXattr trait
    layer.rs                # Layer trait (Tower-style)
    ext.rs                  # FsExt (extension methods)
    markers.rs              # SelfResolving marker trait
    path_resolver.rs        # PathResolver trait (pluggable resolution)
    types.rs                # Metadata, DirEntry, Permissions, StatFs
    error.rs                # FsError

anyfs/                      # Crate 2: backends + middleware + ergonomics
  Cargo.toml
  src/
    lib.rs
    backends/
      memory.rs             # MemoryBackend [feature: memory, default]
      sqlite.rs             # SqliteBackend [feature: sqlite]
      sqlite_cipher.rs      # SqliteCipherBackend [feature: sqlite-cipher]
      indexed.rs            # IndexedBackend [feature: indexed]
      stdfs.rs              # StdFsBackend [feature: stdfs]
      vrootfs.rs            # VRootFsBackend [feature: vrootfs]
    middleware/
      quota.rs              # Quota<B>
      restrictions.rs       # Restrictions<B>
      path_filter.rs        # PathFilter<B>
      read_only.rs          # ReadOnly<B>
      rate_limit.rs         # RateLimit<B>
      tracing.rs            # Tracing<B>
      dry_run.rs            # DryRun<B>
      cache.rs              # Cache<B>
      overlay.rs            # Overlay<B1, B2>
    resolvers/
      iterative.rs          # IterativeResolver (default)
      noop.rs               # NoOpResolver (for SelfResolving backends)
      caching.rs            # CachingResolver (LRU cache wrapper)
    container.rs            # FileStorage<B, M>
    stack.rs                # BackendStack builder
```

---

## Dependency Model

```
anyfs-backend (trait + types)
     ^
     |-- anyfs (backends + middleware + ergonomics)
     |     ^-- vrootfs feature uses strict-path
```

**Key points:**
- Custom backends depend only on `anyfs-backend`
- `anyfs` provides built-in backends, middleware, and the ergonomic `FileStorage<B, M>` wrapper

**Companion crate (planned):**
- `anyfs-mount` (FUSE/WinFsp mounting) is planned; design and roadmap are documented in `src/guides/mounting.md`.

---

## Middleware Pattern

```
FileStorage<B, M>
    wraps -> Tracing<B>
        wraps -> Restrictions<B>
            wraps -> Quota<B>
                wraps -> SqliteBackend (or any Fs)
```

Each layer implements `Fs`, enabling composition.

---

## Cargo Features

### Backends
- `memory` — In-memory storage (default)
- `sqlite` — SQLite-backed persistent storage
- `sqlite-cipher` — Encrypted SQLite via SQLCipher (mutually exclusive with `sqlite`)
- `indexed` — SQLite index + disk blobs for large file performance
- `stdfs` — Direct `std::fs` delegation (no containment)
- `vrootfs` — Host filesystem backend with path containment (uses `strict-path`)

### Middleware (MVP Scope)
Following the **Tower/Axum** pattern, feature flags keep the core lightweight:

- `quota` — Storage limits (default)
- `path-filter` — Glob-based access control (default)
- `restrictions` — Permission control (default)
- `read-only` — Write blocking (default)
- `rate-limit` — Fixed-window rate limiting (default)
- `tracing` — Detailed audit logging (requires `tracing` crate)

Use `default-features = false` to cherry-pick exactly what you need.

### Middleware (Future Scope)

- `metrics` — Prometheus integration (requires `prometheus` crate)

---

## Where To Start

- Application usage: [Getting Started Guide](../getting-started/guide.md)
- Trait details: [Layered Traits](../traits/layered-traits.md)
- Middleware: [Design Overview](../architecture/design-overview.md)
- Decisions: [Architecture Decision Records](../architecture/adrs.md)
