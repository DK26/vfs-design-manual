# FileStorage<B, M> (anyfs)

**Zero-cost ergonomic wrapper for std::fs-aligned API**

---

## Overview

`FileStorage<B, M>` is a **thin wrapper** that provides a familiar std::fs-aligned API with:
- **`B`** - Backend type (generic, zero-cost)
- **`M`** - Optional marker type for compile-time safety

**Axum-style design:** Zero-cost by default, type erasure opt-in.

**It does TWO things:**
1. Ergonomics (std::fs-aligned API)
2. Optional type-safety via marker types

All policy (limits, feature gates, logging) is handled by middleware, not FileStorage.

---

## Creating a Container

```rust
use anyfs::{MemoryBackend, FileStorage};

// Simple: just ergonomics (type inferred)
let mut fs = FileStorage::new(MemoryBackend::new());
```

With middleware (layer-based):

```rust
use anyfs::{SqliteBackend, QuotaLayer, RestrictionsLayer, FileStorage};

// Type is inferred - no need to write it out
let mut fs = FileStorage::new(
    SqliteBackend::open("data.db")?
        .layer(QuotaLayer::builder()
            .max_total_size(100 * 1024 * 1024)
            .build())
        .layer(RestrictionsLayer::builder()
            .deny_hard_links()
            .deny_permissions()
            .build())
);
```

---

## Marker Types (Compile-Time Safety)

Use `_` to infer the backend type while specifying the marker:

```rust
use anyfs::{MemoryBackend, SqliteBackend, FileStorage};

// Define marker types for your domains
struct Sandbox;
struct UserData;

// Specify marker in type annotation, infer backend with _
let sandbox: FileStorage<_, Sandbox> = FileStorage::new(MemoryBackend::new());
let userdata: FileStorage<_, UserData> = FileStorage::new(SqliteBackend::open("data.db")?);

// Type-safe function signatures prevent mixing containers
fn process_sandbox(fs: &FileStorage<impl Fs, Sandbox>) {
    // Can only accept Sandbox-marked containers
}

fn save_user_file(fs: &mut FileStorage<impl Fs, UserData>, name: &str, data: &[u8]) {
    // Can only accept UserData-marked containers
}

// Compile-time safety:
process_sandbox(&sandbox);   // OK
process_sandbox(&userdata);  // Compile error! Type mismatch
```

### Self-Documenting Types

Both dimensions are meaningful:

```rust
FileStorage<SqliteBackend, TenantA>   // SQLite storage for TenantA
FileStorage<MemoryBackend, Sandbox>   // In-memory sandbox
FileStorage<StdFsBackend, Production> // Real filesystem, production
```

### Type Aliases for Clean Code

```rust
// Define your standard secure stack
type SecureBackend = Tracing<Restrictions<Quota<SqliteBackend>>>;

// Type aliases for common combinations
type SandboxFs = FileStorage<MemoryBackend, Sandbox>;
type UserDataFs = FileStorage<SecureBackend, UserData>;
type TenantFs<T> = FileStorage<SecureBackend, T>;

// Now signatures are clean AND informative
fn run_agent(fs: &mut SandboxFs) { ... }
fn save_document(fs: &mut UserDataFs, doc: &Document) { ... }
```

### When to Use Markers

| Scenario | Use Markers? | Why |
|----------|--------------|-----|
| Single container | No | `FileStorage<B>` is sufficient |
| Multiple containers, same type | **Yes** | Prevent accidental mixing |
| Multi-tenant systems | **Yes** | Compile-time tenant isolation |
| Sandbox + user data | **Yes** | Never write user data to sandbox |
| Testing | Maybe | Tag test vs production containers |

---

## std::fs-aligned Methods

FileStorage mirrors std::fs naming:

| FileStorage | std::fs |
|-------------|---------|
| `read()` | `std::fs::read` |
| `read_to_string()` | `std::fs::read_to_string` |
| `write()` | `std::fs::write` |
| `read_dir()` | `std::fs::read_dir` |
| `create_dir()` | `std::fs::create_dir` |
| `create_dir_all()` | `std::fs::create_dir_all` |
| `remove_file()` | `std::fs::remove_file` |
| `remove_dir()` | `std::fs::remove_dir` |
| `remove_dir_all()` | `std::fs::remove_dir_all` |
| `rename()` | `std::fs::rename` |
| `copy()` | `std::fs::copy` |
| `metadata()` | `std::fs::metadata` |
| `symlink_metadata()` | `std::fs::symlink_metadata` |
| `read_link()` | `std::fs::read_link` |
| `set_permissions()` | `std::fs::set_permissions` |

---

## What FileStorage Does NOT Do

| Concern | Use Instead |
|---------|-------------|
| Quota enforcement | `Quota<B>` |
| Feature gating | `Restrictions<B>` |
| Audit logging | `Tracing<B>` |
| Path containment | Backend-specific (VRootFsBackend) |

FileStorage is **purely ergonomic**. If you need policy, compose middleware.

---

## FileStorage Implementation

```rust
use std::marker::PhantomData;
use anyfs_backend::Fs;

/// Zero-cost ergonomic wrapper.
/// Generic over backend (B) and marker (M).
pub struct FileStorage<B, M = ()> {
    backend: B,
    _marker: PhantomData<M>,
}

impl<B: Fs, M> FileStorage<B, M> {
    /// Create a new FileStorage.
    /// Marker type is specified via type annotation:
    /// `let fs: FileStorage<_, MyMarker> = FileStorage::new(backend);`
    pub fn new(backend: B) -> Self {
        FileStorage { backend, _marker: PhantomData }
    }

    /// Type-erase the backend for simpler types (opt-in boxing).
    pub fn boxed(self) -> FileStorage<Box<dyn Fs>, M> {
        FileStorage { backend: Box::new(self.backend), _marker: PhantomData }
    }
}
```

---

## Type Erasure (Opt-in)

When you need simpler types (e.g., storing in collections), use `.boxed()`:

```rust
use anyfs::{MemoryBackend, SqliteBackend, FileStorage, Fs};

// Type-erased for uniform storage
let filesystems: Vec<FileStorage<Box<dyn Fs>>> = vec![
    FileStorage::new(MemoryBackend::new()).boxed(),
    FileStorage::new(SqliteBackend::open("a.db")?).boxed(),
    FileStorage::new(SqliteBackend::open("b.db")?).boxed(),
];

// Or use type alias
type DynFileStorage<M = ()> = FileStorage<Box<dyn Fs>, M>;
```

**When to use `.boxed()`:**

| Situation | Use Generic | Use `.boxed()` |
|-----------|-------------|----------------|
| Local variables | Yes | No |
| Function params | Yes (`impl Fs`) | No |
| Return types | Yes (`impl Fs`) | No |
| Collections of mixed backends | No | **Yes** |
| Struct fields (want simple type) | Maybe | **Yes** |

---

## Direct Backend Access

If you don't need the wrapper, use backends directly:

```rust
use anyfs::{MemoryBackend, QuotaLayer, Fs};

let mut backend = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build());

// Use Fs trait methods directly
backend.write("/file.txt", b"data")?;
```

`FileStorage<B, M>` is part of the `anyfs` crate, not a separate crate.
