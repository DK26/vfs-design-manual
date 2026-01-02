# AnyFS - The Filesystem Abstraction Layer for Rust

**AnyFS is to filesystems what Axum/Tower is to HTTP.**

It lets you control **how a drive acts, looks, and protects itself.**

Whether it's a real filesystem, a SQLite database, or pure RAM, it can be mounted to look like a standard drive to your OS (planned companion crate; roadmap defined), but with *your* rules: quotas, audit logging, and sandbox controls. (Prometheus metrics planned post-v1.)

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

**Switch backends without changing code.** Add middleware without touching storage logic. The goal is to make storage composition easy, safe, and enjoyable for developers.

#### Philosophy: Focused App, Smart Storage

It shifts the architecture from *"Smart App, Dumb Storage"* to **"Focused App, Smart Storage"**.

Instead of burdening your application code with policy logic (*"encrypt this, then log it, then check quota, then write it"*), you define the policy in the storage layer itself:

```
┌─────────────────────────────────────────┐
│  Your Application                       │
│  "Write invoice to /mnt/invoices/..."   │  ← App is simple
├─────────────────────────────────────────┤
│  Infrastructure Enforces Policy:        │
│    1. Encryption (SqliteCipher)         │  ← Safety is intrinsic
│    2. Quota Check (Middleware)          │  ← Limits are enforced
│    3. Audit Log (Tracing)               │  ← Compliance is automatic
│    4. Search Indexing (Sidecar)         │  ← Data is discoverable
├─────────────────────────────────────────┤
│  Backend Storage                        │
└─────────────────────────────────────────┘
```

**Switch backends without changing code.** Add middleware without touching business logic. Create a **Data Mesh** at the filesystem level.

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
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .max_file_size(10 * 1024 * 1024)
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .deny("**/.env")
        .build())
    .layer(TracingLayer::new());

let fs = FileStorage::new(backend);
```

### Layered Trait System

Implement only what you need:

```
FsPosix  ← Full POSIX (handles, locks, xattr)
    ↑
FsFuse   ← FUSE-mountable (+ inodes)
    ↑
FsFull   ← std::fs features (+ links, permissions, sync, stats)
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

| Middleware     | What It Does                                         |
| -------------- | ---------------------------------------------------- |
| `Quota`        | Enforce storage limits, reject writes over quota     |
| `PathFilter`   | Sandbox to allowed paths, block sensitive files      |
| `Restrictions` | Block permission changes (symlinks via trait bounds) |
| `RateLimit`    | Throttle operations per second                       |
| `ReadOnly`     | Block all writes                                     |
| `Cache`        | LRU cache for repeated reads                         |
| `Overlay`      | Union filesystem (Docker-like layers)                |
| `DryRun`       | Log what would happen without executing              |
| Custom         | Implement encryption, compression, deduplication...  |

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

### Cross-Platform Virtual Drive Mounting (Planned, Roadmap Defined)

Mount any compatible backend (requires `FsFuse`) as a real filesystem drive via the planned companion crate (`anyfs-mount`). API sketch:

```rust
use anyfs_mount::MountHandle;

let mount = MountHandle::mount(backend, "/mnt/virtual")?;
// Now any application can access /mnt/virtual
```

| Platform | Technology    |
| -------- | ------------- |
| Linux    | FUSE (native) |
| macOS    | macFUSE       |
| Windows  | WinFsp        |

**Roadmap:** Milestones and MVP scope are tracked in `src/guides/mounting.md`.

### Security by Design

**Path Containment**: `VRootFsBackend` uses [`strict-path`](https://github.com/DK26/strict-path-rs) for real filesystem containment with full canonicalization and symlink resolution.

**Virtual Backends Are Inherently Safe**: `MemoryBackend` and `SqliteBackend` treat paths as keys. Path traversal attacks are structurally impossible.

**Opt-in Restrictions**: Use `Restrictions` middleware to block dangerous operations when sandboxing untrusted code.

---

### Killer Feature: Live Mount Observability 🚀

Future concept: mount your AnyFS stack as a real drive (FUSE/WinFsp) and get **real-time visibility** into OS operations.

Because AnyFS sits *between* the OS and the storage:

- **Live Dashboard**: Watch file IOPS, throughput, and latency in real-time (Prometheus/Grafana).
- **Active Defense**: Scan files for viruses *as they are written* by any external app.
- **Audit Logs**: See exactly which files legacy applications are touching.
- **Access Control**: Revoke write permissions dynamically while the drive is mounted.

This is a future idea that depends on the mounting crate and is not implemented in v1.

---

### Fully Cross-Platform

| Backend               | Windows | Linux | macOS | WASM  |
| --------------------- | :-----: | :---: | :---: | :---: |
| `MemoryBackend`       |    ✅    |   ✅   |   ✅   |   ✅   |
| `SqliteBackend`       |    ✅    |   ✅   |   ✅   |   ✅   |
| `SqliteCipherBackend` |    ✅    |   ✅   |   ✅   |   ❌   |
| `IndexedBackend`      |    ✅    |   ✅   |   ✅   |   ❌   |
| `StdFsBackend`        |    ✅    |   ✅   |   ✅   |   ❌   |
| `VRootFsBackend`      |    ✅    |   ✅   |   ✅   |   ❌   |

Virtual backends work identically everywhere - paths are just keys, symlinks are stored data.

---

## Capabilities

### What AnyFS Provides

| Capability                 | Description                                                                                                                     |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Backend Abstraction**    | Single API for memory, SQLite, encrypted SQLite, host filesystem, or custom storage                                             |
| **Middleware Composition** | Stack policies like quotas, rate limits, access control without changing app code                                               |
| **Path Sandboxing**        | Contain operations to allowed directories (virtual backends are inherently safe; `VRootFsBackend` uses strict-path for real FS) |
| **Storage Quotas**         | Enforce per-file and total size limits with streaming byte counting                                                             |
| **Access Control**         | Glob-based path filtering, operation restrictions, read-only modes                                                              |
| **Tenant Isolation**       | Type-safe markers prevent cross-tenant data access at compile time                                                              |
| **Encryption at Rest**     | SQLCipher backend for AES-256 encrypted storage                                                                                 |
| **Audit & Tracing**        | Log all operations with structured tracing integration                                                                          |
| **Rate Limiting**          | Throttle operations to prevent abuse                                                                                            |
| **Caching**                | LRU cache middleware for repeated reads                                                                                         |
| **Union Filesystems**      | Overlay multiple backends (Docker-like layering)                                                                                |
| **Snapshots**              | Clone in-memory backends for instant checkpoints                                                                                |
| **Virtual Drive Mounting** | Planned companion crate for FUSE/WinFsp mounting (requires `FsFuse`)                                                            |
| **Path Canonicalization**  | Optimizable symlink and `..` resolution per backend                                                                             |
| **FFI-Ready**              | Design supports Python (PyO3), C, and other language bindings                                                                   |
| **Dynamic Middleware**     | Runtime-configured middleware stacks via `Box<dyn Fs>`                                                                          |

---

## Real-World Examples

### Example 1: Multi-Tenant Cloud Storage

Isolate customer data with per-tenant encryption, quotas, and compile-time safety.

```rust
use anyfs::{FileStorage, SqliteCipherBackend, QuotaLayer, TracingLayer, Fs};

struct TenantMarker;

pub fn create_tenant_storage(
    tenant_id: &str, 
    encryption_key: &[u8; 32]
) -> Result<FileStorage<Box<dyn Fs>, TenantMarker>, FsError> {
    let db_path = format!("/data/tenants/{}.db", tenant_id);
    
    let backend = SqliteCipherBackend::open(&db_path, encryption_key)?
        .layer(QuotaLayer::builder()
            .max_total_size(10 * 1024 * 1024 * 1024)  // 10GB per tenant
            .max_file_size(500 * 1024 * 1024)         // 500MB max file
            .build())
        .layer(TracingLayer::new());

    Ok(FileStorage::new(backend).boxed())
}

// Type system prevents accidentally mixing FileStorages meant for different purposes
fn process_tenant<M>(fs: &FileStorage<impl Fs, M>) { /* ... */ }
```

**Benefits:** Per-tenant encryption, automatic quota enforcement, audit trail, compile-time isolation.

---

### Example 2: Organizational File Indexing

Build a searchable file catalog over any storage backend.

```rust
use anyfs::{FileStorage, VRootFsBackend, Fs};

pub struct IndexedStorage<B: Fs> {
    fs: FileStorage<B>,
    index_db: rusqlite::Connection,
}

impl<B: Fs> IndexedStorage<B> {
    /// Scan and index all files
    pub fn reindex(&self) -> Result<u64, FsError> {
        let mut count = 0;
        self.index_recursive("/", &mut count)?;
        Ok(count)
    }

    fn index_recursive(&self, dir: &str, count: &mut u64) -> Result<(), FsError> {
        for entry in self.fs.read_dir(dir)? {
            let entry = entry?;
            let meta = self.fs.metadata(&entry.path)?;
            
            if meta.file_type.is_dir() {
                self.index_recursive(&entry.path.to_string_lossy(), count)?;
            } else {
                self.index_db.execute(
                    "INSERT OR REPLACE INTO files (path, size, modified) VALUES (?, ?, ?)",
                    params![entry.path.to_string_lossy(), meta.size, meta.modified],
                )?;
                *count += 1;
            }
        }
        Ok(())
    }

    /// Search indexed files
    pub fn search(&self, pattern: &str) -> Vec<String> {
        // Query index_db with LIKE pattern
    }
}
```

**Benefits:** Works with any backend, searchable metadata, incremental updates.

---

### Example 3: USB Data Encryption, Migration & Backup

Secure USB storage with encrypted backup and cross-device migration.

```rust
use anyfs::{FileStorage, MemoryBackend, SqliteCipherBackend, VRootFsBackend, Fs};

pub struct SecureUSB {
    memory: FileStorage<MemoryBackend>,  // Fast working copy
}

impl SecureUSB {
    /// Load encrypted USB into memory for fast access
    pub fn open(usb_path: &str, password: &str) -> Result<Self, FsError> {
        let key = derive_key(password);
        let usb = SqliteCipherBackend::open(&format!("{}/vault.db", usb_path), &key)?;
        
        let memory = MemoryBackend::new();
        copy_all(&FileStorage::new(usb), &FileStorage::new(memory.clone()))?;
        
        Ok(Self { memory: FileStorage::new(memory) })
    }

    /// Save encrypted back to USB
    pub fn save_to_usb(&self, usb_path: &str, password: &str) -> Result<(), FsError> {
        let key = derive_key(password);
        let usb = SqliteCipherBackend::open(&format!("{}/vault.db", usb_path), &key)?;
        copy_all(&self.memory, &FileStorage::new(usb))
    }

    /// Migrate to new USB with new password
    pub fn migrate(&self, new_usb: &str, new_password: &str) -> Result<(), FsError> {
        self.save_to_usb(new_usb, new_password)
    }

    /// Export decrypted to folder
    pub fn export(&self, dest: &str) -> Result<(), FsError> {
        let dest_fs = FileStorage::new(VRootFsBackend::new(dest)?);
        copy_all(&self.memory, &dest_fs)
    }
}
```

**Benefits:** Encrypted at-rest, fast in-memory working copy, cross-device migration, backup with different credentials.

---

## Use Cases Summary

| Use Case                | How AnyFS Helps                                  |
| ----------------------- | ------------------------------------------------ |
| **AI Agent Sandboxing** | PathFilter + Quota + RateLimit = safe execution  |
| **Multi-tenant SaaS**   | Per-tenant encrypted DBs, compile-time isolation |
| **Document Management** | Indexing + search + any backend                  |
| **USB Encryption**      | SQLCipher + memory cache + migration             |
| **Game Save Systems**   | SQLite backend = single portable save file       |
| **Testing**             | MemoryBackend for fast, isolated tests           |
| **Plugin Systems**      | Sandbox plugin filesystem access                 |

---

## Quick Start

```rust
use anyfs::{MemoryBackend, QuotaLayer, TracingLayer, FileStorage};

// Build a middleware stack
let backend = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(50 * 1024 * 1024)
        .build())
    .layer(TracingLayer::new());

// Use familiar std::fs-like API
let mut fs = FileStorage::new(backend);
fs.create_dir_all("/docs")?;
fs.write("/docs/hello.txt", b"Hello, AnyFS!")?;
let content = fs.read_to_string("/docs/hello.txt")?;
```

---

## Crate Structure

| Crate           | Purpose                                                                         |
| --------------- | ------------------------------------------------------------------------------- |
| `anyfs-backend` | Core traits (`Fs`, `FsFull`, `FsFuse`, `FsPosix`), `Layer` trait, types, errors |
| `anyfs`         | Built-in backends, middleware, `FileStorage<B, M>` wrapper                      |
| `anyfs-mount`   | Companion crate for cross-platform mounting (planned)                           |

Core AnyFS is the two-crate split (`anyfs-backend`, `anyfs`). `anyfs-mount` is a planned companion crate (design complete; implementation pending), not part of the core crates.

```toml
[dependencies]
anyfs = { version = "0.1", features = ["sqlite"] }

# Optional: for mounting as virtual drive
# Planned companion crate (not yet published):
# anyfs-mount = "0.1"
```

---

## Comparison with Alternatives

| Feature                  | AnyFS | `vfs` crate | AgentFS | `std::fs` |
| ------------------------ | :---: | :---------: | :-----: | :-------: |
| Composable middleware    |   ✅   |      ❌      |    ❌    |     ❌     |
| Multiple backends        |   ✅   |      ✅      |    ❌    |     ❌     |
| SQLite backend           |   ✅   |      ❌      |    ✅    |     ❌     |
| Quota enforcement        |   ✅   |      ❌      |    ❌    |     ❌     |
| Path sandboxing          |   ✅   |   Partial   |    ❌    |     ❌     |
| Symlink-safe containment |   ✅   |      ❌      |   N/A   |    N/A    |
| Rate limiting            |   ✅   |      ❌      |    ❌    |     ❌     |
| FUSE mounting            |   ✅   |      ❌      |    ✅    |    N/A    |
| Cross-platform           |   ✅   |      ✅      |  Linux  |     ✅     |
| Type-safe markers        |   ✅   |      ❌      |    ❌    |     ❌     |

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

🚧 **Design Complete — Implementation Not Started** 🚧

This repository contains the **design manual** for AnyFS. The design is solidifying, and we're collecting feedback before implementation begins.

**We value your feedback!**
- Have a different use case?
- Think a design decision is wrong?
- Want a specific middleware?

Please **open an issue** and push back on our ideas. We want to build something that solves *real* problems, not just theoretical ones.

**Current Phase:** Design complete; implementation will live in separate `anyfs-backend` and `anyfs` crates.

---

## License

This repository contains the design manual and code examples for AnyFS.

### Documentation
The text, diagrams, and media in this manual are licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

### Code
Code examples and snippets are licensed under the **MIT License** and **Apache License 2.0** to allow free use in your own projects.

See [LICENSE](LICENSE) for full details.

