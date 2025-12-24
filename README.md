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
│  FilesContainer (std::fs-like API)      │
├─────────────────────────────────────────┤
│  Middleware Stack (composable):         │
│    Quota, PathFilter, RateLimit,        │
│    Tracing, Cache, Encryption...        │
├─────────────────────────────────────────┤
│  VfsBackend (pluggable):                │
│    Memory, SQLite, Real FS, S3, ...     │
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
let fs = MemoryBackend::new()
    .layer(QuotaLayer::new()
        .max_total_size(100 * 1024 * 1024)
        .max_file_size(10 * 1024 * 1024))
    .layer(PathFilterLayer::new()
        .allow("/workspace/**")
        .deny("**/.env"))
    .layer(FeatureGuardLayer::new())  // No symlinks by default
    .layer(TracingLayer::new());      // Audit trail
```

### Not Just Monitoring - Intervention

Unlike logging-only solutions, AnyFS middleware can **transform and control**:

| Middleware | What It Does |
|------------|--------------|
| `Quota` | Enforce storage limits, reject writes over quota |
| `PathFilter` | Sandbox to allowed paths, block sensitive files |
| `FeatureGuard` | Disable dangerous features (symlinks, hard links) |
| `RateLimit` | Throttle operations per second |
| `ReadOnly` | Block all writes |
| `Cache` | LRU cache for repeated reads |
| `Overlay` | Union filesystem (Docker-like layers) |
| `DryRun` | Log what would happen without executing |
| Custom | Implement encryption, compression, deduplication... |

### Security by Design

AnyFS treats security as a first-class concern, not an afterthought:

**Path Containment**: The `VRootFsBackend` uses [`strict-path`](https://github.com/DK26/strict-path-rs) for real filesystem containment. Unlike simple path-prefix approaches, `strict-path` performs full canonicalization with symlink resolution, preventing escape via:
- Symlink traversal attacks (`/allowed/link` → `../../../etc/passwd`)
- TOCTOU (time-of-check-to-use) race conditions
- Windows edge cases (8.3 short names, NTFS alternate data streams, junctions)

**Virtual Backends Are Inherently Safe**: `MemoryBackend` and `SqliteBackend` treat paths as keys, not OS paths. There's no underlying filesystem to exploit - path traversal attacks are structurally impossible.

**Least Privilege by Default**: Dangerous features (symlinks, hard links, permission changes) are disabled unless explicitly enabled via `FeatureGuard`.

### Data Portability

Same data, different backends:

```rust
// Read from SQLite
let source = SqliteBackend::open("old.db")?;
let data = source.read("/config.json")?;

// Write to new backend
let dest = MemoryBackend::new();
dest.write("/config.json", &data)?;
```

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
use anyfs::{MemoryBackend, Quota, FeatureGuard, Tracing};
use anyfs_container::FilesContainer;

// Build a middleware stack
let backend = Tracing::new(
    FeatureGuard::new(
        Quota::new(MemoryBackend::new())
            .with_max_total_size(50 * 1024 * 1024)
    )
);

// Use familiar std::fs-like API
let mut fs = FilesContainer::new(backend);
fs.create_dir_all("/docs")?;
fs.write("/docs/hello.txt", b"Hello, AnyFS!")?;
let content = fs.read_to_string("/docs/hello.txt")?;
```

---

## Crate Structure

| Crate | Purpose |
|-------|---------|
| `anyfs-backend` | Core trait (`VfsBackend`), `Layer` trait, types, errors |
| `anyfs` | Built-in backends + middleware (feature-gated) |
| `anyfs-container` | `FilesContainer` ergonomic wrapper, `BackendStack` builder |

```toml
[dependencies]
anyfs = { version = "0.1", features = ["sqlite"] }
anyfs-container = "0.1"
```

---

## Comparison with Alternatives

| Feature | AnyFS | `vfs` crate | AgentFS | `std::fs` |
|---------|:-----:|:-----------:|:-------:|:---------:|
| Composable middleware | Yes | No | No | No |
| Multiple backends | Yes | Yes | No (SQLite) | No |
| SQLite backend | Yes | No | Yes | No |
| Quota enforcement | Yes | No | No | No |
| Path sandboxing | Yes | Partial | No | No |
| Symlink-safe containment | Yes | No | N/A | N/A |
| Rate limiting | Yes | No | No | No |

### vs `vfs` Crate

The [vfs](https://crates.io/crates/vfs) crate is a solid VFS abstraction. AnyFS builds on similar concepts with additional capabilities:

| Aspect | `vfs` | AnyFS |
|--------|-------|-------|
| **Design pattern** | Path wrapper (`VfsPath`) | Backend trait + middleware |
| **Middleware** | Not supported | Tower-style composable layers |
| **Policy enforcement** | Application responsibility | Built-in (Quota, PathFilter, etc.) |
| **Path containment** | Prefix-based (AltrootFS) | Canonicalization-based (`strict-path`) |
| **Path type** | Custom `VfsPath` | `impl AsRef<Path>` (std-compatible) |

**When to use `vfs`:** Simple VFS abstraction without policy requirements.

**When to use AnyFS:** You need middleware (quotas, sandboxing, rate limiting) or security-hardened path containment.

### vs AgentFS

[AgentFS](https://github.com/tursodatabase/agentfs) is an **agent runtime** (filesystem + KV store + tool auditing). AnyFS is a **filesystem abstraction**. They solve different problems:

| Aspect | AgentFS | AnyFS |
|--------|---------|-------|
| **Scope** | Complete agent runtime | Filesystem abstraction only |
| **Backend choice** | SQLite only | Memory, SQLite, RealFS, custom |
| **Middleware** | Fixed capabilities | Composable, extensible |
| **KV store** | Included | Not included (different concern) |

**When to use AgentFS:** You need a complete AI agent runtime with built-in KV and auditing.

**When to use AnyFS:** You need just filesystem with backend flexibility and composable policies.

### vs Direct `std::fs`

`std::fs` is great for simple cases. AnyFS adds:
- Backend swapping without code changes
- Policy enforcement via middleware
- Consistent API across storage types
- Isolated/sandboxed storage

---

## Documentation

Full documentation in `book/`:

```bash
mdbook serve book
```

Key documents:
- `src/architecture/design-overview.md` - Architecture and middleware
- `src/architecture/adrs.md` - Design decisions (21 ADRs)
- `src/implementation/backend-guide.md` - Implement custom backends
- `AGENTS.md` - Quick reference for AI assistants

---

## Status

**Design complete. Implementation in progress.**

The design manual documents the full API, middleware patterns, and implementation guidance. Core implementation is underway.

---

## License

[TBD]
