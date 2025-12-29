# AnyFS - The Filesystem Abstraction Layer for Rust

**AnyFS is to filesystems what Axum/Tower is to HTTP.**

A composable middleware stack for filesystem operations with pluggable backends.

---

## Why AnyFS?

### The Problem

You need filesystem operations in your application. But:

- Today you use the local filesystem. Tomorrow you might need SQLite for portability.
- You want to enforce quotas, but not pollute your business logic.
- You need to sandbox untrusted code to specific directories.
- You want to log all operations for debugging, but only in development.
- You're building for AI agents, game save systems, document storage, or tenant isolation.

Every backend has different APIs. Every policy concern gets tangled with storage code.

### The Solution

AnyFS separates **what** you store from **how** you store it and **what rules** apply:

```
┌─────────────────────────────────────────┐
│  Your Application                       │
├─────────────────────────────────────────┤
│  FileStorage<B, M> (std::fs-like API)   │
├─────────────────────────────────────────┤
│  Middleware Stack (composable):         │
│    Quota, PathFilter, RateLimit,        │
│    Tracing, Cache, Overlay...           │
├─────────────────────────────────────────┤
│  Backend (implements Fs trait):         │
│    Memory, SQLite, VRootFs, custom...   │
└─────────────────────────────────────────┘
```

**Switch backends without changing code.** Add middleware without touching storage logic.

---

## Key Features

### Backend Agnostic

```rust
// Today: SQLite for portability
let backend = SqliteBackend::open("data.db")?;

// Tomorrow: Real filesystem for performance
let backend = VRootFsBackend::new("/var/data")?;

// Testing: In-memory for speed
let backend = MemoryBackend::new();

// Your code doesn't change.
```

### Composable Middleware

Stack capabilities like building blocks:

```rust
let backend = MemoryBackend::new()
    .layer(QuotaLayer::new()
        .max_total_size(100 * 1024 * 1024)
        .max_file_size(10 * 1024 * 1024))
    .layer(PathFilterLayer::new()
        .allow("/workspace/**")
        .deny("**/.env"))
    .layer(RestrictionsLayer::new()
        .deny_symlinks()
        .deny_hard_links())
    .layer(TracingLayer::new());

let mut fs = FileStorage::new(backend);
```

### Layered Trait System

Implement only what you need:

```
FsPosix  ← Full POSIX (handles, locks, xattr)
    ↑
FsFuse   ← FUSE-mountable (+ inodes)
    ↑
FsFull   ← std::fs features (+ links, permissions, sync)
    ↑
   Fs    ← Basic filesystem (90% of use cases)
    ↑
FsRead + FsWrite + FsDir  ← Core traits
```

### Type-Safe Containers

Marker types prevent mixing containers at compile time:

```rust
struct Sandbox;
struct UserData;

let sandbox: FileStorage<_, Sandbox> = FileStorage::new(MemoryBackend::new());
let userdata: FileStorage<_, UserData> = FileStorage::new(SqliteBackend::open("data.db")?);

fn process_sandbox(fs: &FileStorage<impl Fs, Sandbox>) { /* only accepts Sandbox */ }

process_sandbox(&sandbox);   // OK
process_sandbox(&userdata);  // Compile error!
```

### Not Just Monitoring - Intervention

Unlike logging-only solutions, AnyFS middleware can **transform and control**:

| Middleware | What It Does |
|------------|--------------|
| `Quota` | Enforce storage limits, reject writes over quota |
| `PathFilter` | Sandbox to allowed paths, block sensitive files |
| `Restrictions` | Block specific operations (symlinks, hard links, permissions) |
| `RateLimit` | Throttle operations per second |
| `ReadOnly` | Block all writes |
| `Cache` | LRU cache for repeated reads |
| `Overlay` | Union filesystem (Docker-like layers) |
| `DryRun` | Log what would happen without executing |
| Custom | Implement encryption, compression, deduplication... |

### Snapshots & Persistence

```rust
// MemoryBackend implements Clone - that's the snapshot
let checkpoint = fs.clone();

// Rollback
fs = checkpoint;

// Persist to disk
fs.save_to("state.bin")?;
let fs = MemoryBackend::load_from("state.bin")?;
```

### Cross-Platform Virtual Drive Mounting

Mount any backend as a real filesystem drive:

```rust
use anyfs_mount::MountHandle;

let mount = MountHandle::mount(backend, "/mnt/virtual")?;
// Now any application can access /mnt/virtual
```

| Platform | Technology |
|----------|------------|
| Linux | FUSE (native) |
| macOS | macFUSE |
| Windows | WinFsp |

### Security by Design

**Path Containment**: `VRootFsBackend` uses [`strict-path`](https://github.com/nickelpack/strict-path) for real filesystem containment with full canonicalization and symlink resolution.

**Virtual Backends Are Inherently Safe**: `MemoryBackend` and `SqliteBackend` treat paths as keys. Path traversal attacks are structurally impossible.

**Opt-in Restrictions**: Use `Restrictions` middleware to block dangerous operations when sandboxing untrusted code.

### Fully Cross-Platform

| Backend | Windows | Linux | macOS | WASM |
|---------|:-------:|:-----:|:-----:|:----:|
| `MemoryBackend` | ✅ | ✅ | ✅ | ✅ |
| `SqliteBackend` | ✅ | ✅ | ✅ | ✅ |
| `SqliteCipherBackend` | ✅ | ✅ | ✅ | ❌ |
| `VRootFsBackend` | ✅ | ✅ | ✅ | ❌ |

Virtual backends work identically everywhere - paths are just keys, symlinks are stored data.

---

## Use Cases

| Use Case | How AnyFS Helps |
|----------|-----------------|
| **AI Agent Sandboxing** | PathFilter + Quota + RateLimit = safe execution |
| **Multi-tenant SaaS** | One backend per tenant, enforced isolation |
| **Game Save Systems** | SQLite backend = single portable save file |
| **Document Storage** | Quota limits + metadata + any backend |
| **Testing** | MemoryBackend for fast, isolated tests |
| **Embedded Systems** | Swap backends based on available storage |
| **Plugin Systems** | Sandbox plugin filesystem access |

---

## Quick Start

```rust
use anyfs::{MemoryBackend, Quota, Tracing, FileStorage};

// Build a middleware stack
let backend = Tracing::new(
    Quota::new(MemoryBackend::new())
        .with_max_total_size(50 * 1024 * 1024)
);

// Use familiar std::fs-like API
let mut fs = FileStorage::new(backend);
fs.create_dir_all("/docs")?;
fs.write("/docs/hello.txt", b"Hello, AnyFS!")?;
let content = fs.read_to_string("/docs/hello.txt")?;
```

---

## Crate Structure

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Core traits (`Fs`, `FsFull`, `FsFuse`, `FsPosix`), `Layer` trait, types, errors |
| `anyfs` | Built-in backends, middleware, `FileStorage<B, M>` wrapper |
| `anyfs-mount` | Cross-platform virtual drive mounting (optional) |

```toml
[dependencies]
anyfs = { version = "0.1", features = ["sqlite"] }

# Optional: for mounting as virtual drive
anyfs-mount = "0.1"
```

---

## Comparison with Alternatives

| Feature | AnyFS | `vfs` crate | AgentFS | `std::fs` |
|---------|:-----:|:-----------:|:-------:|:---------:|
| Composable middleware | ✅ | ❌ | ❌ | ❌ |
| Multiple backends | ✅ | ✅ | ❌ | ❌ |
| SQLite backend | ✅ | ❌ | ✅ | ❌ |
| Quota enforcement | ✅ | ❌ | ❌ | ❌ |
| Path sandboxing | ✅ | Partial | ❌ | ❌ |
| Symlink-safe containment | ✅ | ❌ | N/A | N/A |
| Rate limiting | ✅ | ❌ | ❌ | ❌ |
| FUSE mounting | ✅ | ❌ | ✅ | N/A |
| Cross-platform | ✅ | ✅ | Linux | ✅ |
| Type-safe markers | ✅ | ❌ | ❌ | ❌ |

---

## Documentation

Full documentation available as mdbook in `src/`:

```bash
mdbook serve
```

Key documents:
- `src/architecture/design-overview.md` - Architecture and traits
- `src/guides/llm-context.md` - Quick implementation patterns
- `src/guides/mounting.md` - Virtual drive mounting
- `src/implementation/backend-guide.md` - Implement custom backends
- `src/implementation/testing-guide.md` - Test suite design
- `AGENTS.md` - Quick reference for AI assistants

---

## Status

**Design complete. Implementation in progress.**

The design manual documents the full API, middleware patterns, and implementation guidance.

---

## License

[TBD]
