# AnyFS: Comparison, Positioning & Honest Assessment

**A comprehensive look at why AnyFS exists, how it compares, and where it falls short**

---

## Origin Story

AnyFS didn't start as a filesystem abstraction. It started as a security problem.

### The Path Security Problem

While exploring filesystem security, I created the [`strict-path`](https://github.com/DK26/strict-path-rs) crate to ensure that externally-sourced paths could never escape their boundaries. The approach: resolve a boundary path, resolve the provided path, and validate containment.

This proved far more challenging than expected. Attack vectors kept appearing:

- Symlinks pointing outside the boundary
- Windows junction points
- NTFS Alternate Data Streams (`file.txt:hidden:$DATA`)
- Windows 8.3 short names (`PROGRA~1`)
- Linux `/proc` magic symlinks that escape namespaces
- Unicode normalization tricks (NFC vs NFD)
- URL-encoded traversal (`%2e%2e`)
- TOCTOU race conditions

Eventually, `strict-path` addressed 19+ attack vectors, making it (apparently) comprehensive. But it came with costs:

- **I/O overhead** - Real filesystem resolution is expensive
- **Existing paths only** - `std::fs::canonicalize` requires paths to exist
- **Residual TOCTOU risk** - A symlink created between verification and operation (extremely rare, but possible)

### The SQLite Revelation

Then a new idea emerged: *What if the filesystem didn't exist on disk at all?*

A SQLite-backed virtual filesystem would:
- **Eliminate path security issues** - Paths are just database keys, not real files
- **Be fully portable** - A tenant's entire filesystem in one `.db` file
- **Have no TOCTOU** - Database transactions are atomic
- **Work on non-existing paths** - No canonicalization needed

### The Abstraction Need

But then: *What if I wanted to switch from SQLite to something else later?*

I didn't want to rewrite code just to explore different backends. I needed an abstraction.

### The Framework Vision

Research revealed that existing VFS solutions were either:
- **Too simple** - Just swappable backends, no policies
- **Too fixed** - Specific to one use case (AI agents, archives, etc.)
- **Insecure** - Basic `..` traversal prevention, missing 17+ attack vectors

My niche is security: **isolating filesystems, limiting actions, controlling resources**.

The Tower/Axum pattern for HTTP showed how to compose middleware elegantly. Why not apply the same pattern to filesystems?

Thus AnyFS: **A composable middleware framework for filesystem operations.**

---

## The Landscape: What Already Exists

### Rust Ecosystem

| Library                                                          | Stars | Downloads   | Purpose                        |
| ---------------------------------------------------------------- | ----- | ----------- | ------------------------------ |
| [`vfs`](https://github.com/manuel-woelker/rust-vfs)              | 464   | 1,700+ deps | Swappable filesystem backends  |
| [`virtual-filesystem`](https://lib.rs/crates/virtual-filesystem) | ~30   | ~260/mo     | Backends with basic sandboxing |
| [`AgentFS`](https://github.com/tursodatabase/agentfs)            | New   | Alpha       | AI agent state management      |

### Other Languages

| Library                                                                        | Language | Strength                     |
| ------------------------------------------------------------------------------ | -------- | ---------------------------- |
| [fsspec](https://filesystem-spec.readthedocs.io/)                              | Python   | Async, caching, 20+ backends |
| [PyFilesystem2](https://github.com/PyFilesystem/pyfilesystem2)                 | Python   | Clean URL-based API          |
| [Afero](https://github.com/spf13/afero)                                        | Go       | Composition patterns         |
| [Apache Commons VFS](https://commons.apache.org/vfs/)                          | Java     | Enterprise, many backends    |
| [System.IO.Abstractions](https://github.com/TestableIO/System.IO.Abstractions) | .NET     | Testing, mirrors System.IO   |

---

## Honest Comparison

### What Others Do Well

**`vfs` crate:**
- Mature (464 stars, 1,700+ dependent projects)
- Multiple backends (Memory, Physical, Overlay, Embedded)
- Async support (though being sunset)
- Simple, focused API

**`virtual-filesystem`:**
- ZIP/TAR archive support
- Mountable filesystem
- Basic sandboxing attempt

**AgentFS:**
- Purpose-built for AI agents
- SQLite backend with FUSE mounting
- Key-value store included
- Audit trail built-in
- Backed by Turso (funded company)
- TypeScript/Python SDKs

**fsspec (Python):**
- Block-wise caching (not just whole-file)
- Async-first design
- Excellent data science integration

### What Others Do Poorly

**Security in existing solutions is inadequate.**

I examined `virtual-filesystem`'s `SandboxedPhysicalFS`. Here's their **entire** security implementation:

```rust
impl PathResolver for SandboxedPathResolver {
    fn resolve_path(root: &Path, path: &str) -> Result<PathBuf> {
        let root = root.canonicalize()?;
        let host_path = root.join(make_relative(path)).canonicalize()?;

        if !host_path.starts_with(root) {
            return Err(io::Error::new(ErrorKind::PermissionDenied, "Traversal prevented"));
        }
        Ok(host_path)
    }
}
```

That's it. ~10 lines covering **2 out of 19+** attack vectors.

| Attack Vector               | virtual-filesystem | strict-path |
| --------------------------- | :----------------: | :---------: |
| Basic `..` traversal        |         ✅          |      ✅      |
| Symlink following           |         ✅          |      ✅      |
| NTFS Alternate Data Streams |         ❌          |      ✅      |
| Windows 8.3 short names     |         ❌          |      ✅      |
| Unicode normalization       |         ❌          |      ✅      |
| TOCTOU race conditions      |         ❌          |      ✅      |
| Non-existing paths          |      ❌ FAILS       |      ✅      |
| URL-encoded traversal       |         ❌          |      ✅      |
| Windows UNC paths           |         ❌          |      ✅      |
| Linux /proc magic symlinks  |         ❌          |      ✅      |
| Null byte injection         |         ❌          |      ✅      |
| Unicode direction override  |         ❌          |      ✅      |
| Windows reserved names      |         ❌          |      ✅      |
| Junction point escapes      |         ❌          |      ✅      |
| **Coverage**                |      **2/19**      |  **19/19**  |

The `vfs` crate's `AltrootFS` is similarly basic - just path prefix translation.

**No middleware composition exists anywhere.**

None of the filesystem libraries offer Tower-style middleware. You can't do something like:

```rust
// Hypothetical - doesn't exist in other libraries
backend
    .layer(QuotaLayer)
    .layer(RateLimitLayer)
    .layer(TracingLayer)
```

If you want quotas in `vfs`, you'd have to build it INTO each backend. Then build it again for the next backend.

---

## What Makes AnyFS Unique

### 1. Middleware Composition (Nobody Else Has This)

```rust
let fs = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(100_MB)
        .max_file_count(1000)
        .build())
    .layer(RateLimitLayer::builder()
        .max_ops_per_second(100)
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .deny("/workspace/.git/**")
        .build())
    .layer(TracingLayer::new());
```

Add, remove, or reorder middleware without touching backends. Write middleware once, use with any backend.

### 2. Type-Safe Domain Separation (Nobody Else Has This)

```rust
struct Sandbox;
struct UserData;

let sandbox: FileStorage<_, Sandbox> = FileStorage::new(memory_backend);
let userdata: FileStorage<_, UserData> = FileStorage::new(sqlite_backend);

fn process_sandbox(fs: &FileStorage<impl Fs, Sandbox>) { ... }

process_sandbox(&sandbox);   // OK
process_sandbox(&userdata);  // COMPILE ERROR
```

Compile-time prevention of mixing storage domains.

### 3. Backend-Agnostic Policies (Nobody Else Has This)

| Middleware        | Function                  | Works on ANY backend |
| ----------------- | ------------------------- | :------------------: |
| `Quota<B>`        | Size/count limits         |          ✅           |
| `RateLimit<B>`    | Ops per second            |          ✅           |
| `PathFilter<B>`   | Path-based access control |          ✅           |
| `Restrictions<B>` | Disable operations        |          ✅           |
| `Tracing<B>`      | Audit logging             |          ✅           |
| `ReadOnly<B>`     | Block all writes          |          ✅           |
| `Cache<B>`        | LRU caching               |          ✅           |
| `Overlay<B1,B2>`  | Union filesystem          |          ✅           |

### 4. Comprehensive Security Testing

The planned conformance test suite targets 50+ security tests covering:

- Path traversal (URL-encoded, backslash, mixed)
- Symlink attacks (escape, loops, TOCTOU)
- Platform-specific (NTFS ADS, 8.3 names, /proc)
- Unicode (normalization, RTL override, homoglyphs)
- Resource exhaustion

Derived from vulnerabilities in Apache Commons VFS, Afero, PyFilesystem2, and our own `strict-path` research.

---

## Honest Downsides of AnyFS

### 1. We're New, They're Established

| Metric             | `vfs`  | AnyFS   |
| ------------------ | ------ | ------- |
| Stars              | 464    | 0 (new) |
| Dependent projects | 1,700+ | 0 (new) |
| Years maintained   | 5+     | New     |
| Contributors       | 17     | 1       |

**Reality:** The `vfs` crate works fine for 90% of use cases. If you just need swappable backends for testing, `vfs` is battle-tested.

### 2. Complexity vs Simplicity

```rust
// vfs: Simple
let fs = MemoryFS::new();
fs.create_file("test.txt")?.write_all(b"hello")?;

// AnyFS: More setup if you use middleware
let fs = MemoryBackend::new()
    .layer(QuotaLayer::builder().max_total_size(1_MB).build());
fs.write("/test.txt", b"hello")?;
```

If you don't need middleware, AnyFS adds conceptual overhead.

### 3. Sync-Only (For Now)

AnyFS is sync-first. In an async-dominated ecosystem (Tokio, etc.), this may limit adoption.

`fsspec` (Python) and `OpenDAL` (Rust) are async-first. We're not.

**Mitigation:** ADR-024 plans async support. Our `Send + Sync` bounds enable `spawn_blocking` wrappers today.

### 4. AgentFS Has Momentum for AI Agents

If you're building AI agents specifically:

| Feature            | AgentFS    | AnyFS                     |
| ------------------ | ---------- | ------------------------- |
| SQLite backend     | ✅          | ✅                         |
| FUSE mounting      | ✅          | Planned                   |
| Key-value store    | ✅          | ❌ (different abstraction) |
| Tool call auditing | ✅ Built-in | Via Tracing middleware    |
| TypeScript SDK     | ✅          | ❌                         |
| Python SDK         | Coming     | ❌                         |
| Corporate backing  | Turso      | None                      |

AgentFS is purpose-built for AI agents with corporate resources. We're a general-purpose framework.

### 5. Performance Overhead

Middleware composition has costs:
- Each layer adds a function call
- Quota tracking requires size accounting
- Rate limiting needs timestamp checks

For hot paths with millions of ops/second, this matters. For normal usage, it doesn't.

### 6. Real Filesystem Security Has Limits

For `VRootFsBackend` (wrapping real filesystem):
- Still has I/O costs for path resolution
- Residual TOCTOU risk (extremely rare)
- `strict-path` covers 19 vectors, but unknown unknowns exist

**Virtual backends (Memory, SQLite) don't have these issues** - paths are just keys.

---

## Feature Matrix

| Feature               |     AnyFS      |  `vfs`  |   `virtual-fs`    | AgentFS | OpenDAL |
| --------------------- | :------------: | :-----: | :---------------: | :-----: | :-----: |
| Composable middleware |       ✅        |    ❌    |         ❌         |    ❌    |    ✅    |
| Multiple backends     |       ✅        |    ✅    |         ✅         |    ❌    |    ✅    |
| SQLite backend        |       ✅        |    ❌    |         ❌         |    ✅    |    ❌    |
| Memory backend        |       ✅        |    ✅    |         ✅         |    ❌    |    ✅    |
| Quota enforcement     |       ✅        |    ❌    |         ❌         |    ❌    |    ❌    |
| Rate limiting         |       ✅        |    ❌    |         ❌         |    ❌    |    ❌    |
| Type-safe markers     |       ✅        |    ❌    |         ❌         |    ❌    |    ❌    |
| Path sandboxing       |       ✅        |  Basic  | Basic (2 vectors) |    ❌    |    ❌    |
| Async API             |       🔜        | Partial |         ❌         |    ❌    |    ✅    |
| std::fs-aligned API   |       ✅        | Custom  |         ✅         |    ✅    | Custom  |
| FUSE mounting         |   MVP scope    |    ❌    |         ❌         |    ✅    |    ❌    |
| Conformance tests     | Planned (80+)  | Unknown |      Unknown      | Unknown | Unknown |

---

## When to Use AnyFS

### Good Fit

- **Multi-tenant SaaS** - Per-tenant quotas, path isolation, rate limiting
- **Untrusted input sandboxing** - Comprehensive path security
- **Policy-heavy environments** - When you need composable rules
- **Backend flexibility** - When you might swap storage later
- **Type-safe domain separation** - When mixing containers is dangerous

### Not a Good Fit

- **Simple testing** - `vfs` is simpler if you just need mock FS
- **AI agent runtime** - AgentFS has more features for that specific use case
- **Cloud storage** - OpenDAL is async-first with cloud backends
- **Async-first codebases** - Wait for AnyFS async support
- **Must mount filesystem** - Use a FUSE solution directly today (anyfs-mount is planned)

---

## Summary

**AnyFS exists because:**

1. Existing VFS libraries have basic, inadequate security (2/19 attack vectors)
2. No filesystem library offers middleware composition
3. No filesystem library offers type-safe domain separation
4. Policy enforcement (quotas, rate limits, path filtering) doesn't exist elsewhere

**AnyFS is honest about:**

1. We're new, `vfs` is established
2. We add complexity if you don't need middleware
3. We're sync-only for now
4. AgentFS has more resources for AI-specific use cases

**AnyFS is positioned as:**

> **"Tower for filesystems"** - Composable middleware over pluggable backends, with comprehensive security testing.

---

## Sources

- [vfs crate](https://github.com/manuel-woelker/rust-vfs)
- [virtual-filesystem crate](https://lib.rs/crates/virtual-filesystem)
- [AgentFS](https://github.com/tursodatabase/agentfs)
- [strict-path](https://github.com/DK26/strict-path-rs)
- [soft-canonicalize](https://github.com/DK26/soft-canonicalize-rs)
- [In-Memory Filesystems in Rust](https://andre.arko.net/2025/08/18/in-memory-filesystems-in-rust/) - Performance analysis
- [Rust Forum: Virtual Filesystems](https://users.rust-lang.org/t/virtual-filesystems-for-rust/117173)
- [Prior Art Analysis](./prior-art-analysis.md) - Detailed vulnerability research

