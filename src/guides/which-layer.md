# Which Crate Should I Use?

---

## Decision Guide

| You want to...                                    | Use                  |
| ------------------------------------------------- | -------------------- |
| Build an application                              | `anyfs`              |
| Use built-in backends (Memory, SQLite, VRootFs)   | `anyfs`              |
| Use built-in middleware (Quota, PathFilter, etc.) | `anyfs`              |
| Implement a custom backend                        | `anyfs-backend` only |
| Implement custom middleware                       | `anyfs-backend` only |

---

## Quick Examples

### Simple usage

```rust
use anyfs::MemoryBackend;
use anyfs::FileStorage;

let fs = FileStorage::new(MemoryBackend::new());
fs.create_dir_all("/data")?;
fs.write("/data/file.txt", b"hello")?;
```

### With middleware (quotas, sandboxing, tracing)

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, PathFilterLayer, TracingLayer};
use anyfs::FileStorage;

let stack = SqliteBackend::open("tenant.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .deny("**/.env")
        .build())
    .layer(TracingLayer::new());

let fs = FileStorage::new(stack);
```

### Custom backend implementation

```rust
use anyfs_backend::{Fs, FsError, Metadata, DirEntry};
use std::path::Path;

pub struct MyBackend;

impl Fs for MyBackend {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        todo!()
    }

    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        todo!()
    }

    // ... implement all 25 methods
}
```

### Custom middleware implementation

```rust
use anyfs_backend::{Fs, Layer, FsError};
use std::path::Path;

pub struct MyMiddleware<B: Fs> {
    inner: B,
}

impl<B: Fs> Fs for MyMiddleware<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        // Intercept, transform, or delegate
        self.inner.read(path)
    }
    // ... implement all methods
}

pub struct MyMiddlewareLayer;

impl<B: Fs> Layer<B> for MyMiddlewareLayer {
    type Backend = MyMiddleware<B>;
    fn layer(self, backend: B) -> Self::Backend {
        MyMiddleware { inner: backend }
    }
}
```

---

## Common Mistakes

- **Don't depend on `anyfs`** if you're only implementing a backend or middleware. Use `anyfs-backend`.
- **Don't put policy in backends.** Use middleware (Quota, PathFilter, etc.).
- **Don't put policy in FileStorage.** It is an ergonomic wrapper with centralized path resolution, not a policy layer.

