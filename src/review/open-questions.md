# Open Questions & Future Considerations

**Status:** Resolved (future considerations tracked)
**Last Updated:** 2025-12-28

---

This document captures previously open questions and design considerations. Unless explicitly marked as future, the items below are resolved.

> **Note:** Final decisions live in the Architecture Decision Records.

---

## Symlink Security: Following vs Creating

**Status:** Resolved

### Decision
- Symlink support is a backend capability (via `FsLink`).
- `FileStorage` resolves paths via pluggable `PathResolver` for non-`SelfResolving` backends.
- The default `IterativeResolver` follows symlinks when `FsLink` is available. Custom resolvers can implement different behaviors.
- `SelfResolving` backends delegate to the OS. `strict-path` prevents escapes.

### Implications
- If you need symlink-free semantics, use a backend that does not implement `FsLink` or block symlink creation and ensure no preexisting symlinks exist.
- `Restrictions` only controls creation, not resolution.

### Why
- Virtual backends have no host filesystem to escape to; symlink resolution stays inside the virtual structure.
- OS-backed backends cannot reliably disable symlink following without TOCTOU risks or platform-specific hacks.

---

## Virtual vs Real Backends: Path Resolution

**Status:** Resolved (see also ADR-033 for PathResolver)

**Question:** Should path resolution logic be different for virtual backends (memory, SQLite) vs filesystem-based backends (StdFsBackend, VRootFsBackend)?

**Resolution:** `FileStorage` handles path resolution via pluggable `PathResolver` trait for non-`SelfResolving` backends. `SelfResolving` backends delegate to the OS, so FileStorage does not pre-resolve paths for them.

| Backend Type   | Path Resolution                               | Symlink Handling                                   |
| -------------- | --------------------------------------------- | -------------------------------------------------- |
| MemoryBackend  | **PathResolver** (default: IterativeResolver) | Resolver follows symlinks                          |
| SqliteBackend  | **PathResolver** (default: IterativeResolver) | Resolver follows symlinks                          |
| VRootFsBackend | **OS** (implements `SelfResolving`)           | OS follows symlinks (strict-path prevents escapes) |

**Key design decisions:**

1. Backends that wrap a real filesystem implement the `SelfResolving` marker trait to tell `FileStorage` to skip resolution:

```rust
impl SelfResolving for VRootFsBackend {}
```

2. Path resolution is pluggable via `PathResolver` trait (ADR-033). Built-in resolvers include:
   - `IterativeResolver` - default symlink-aware resolution (when backend implements `FsLink`)
   - `NoOpResolver` - for `SelfResolving` backends
   - `CachingResolver` - LRU cache wrapper around another resolver

---

## Compression and Encryption

**Question:** Does the current design allow backends to compress/decompress or encrypt/decrypt files transparently?

**Answer:** Yes. The backend receives the data and stores it however it wants. A backend could:
- Compress data before writing to SQLite
- Encrypt blobs with a user-provided key
- Use a remote object store with encryption at rest

This is an implementation detail of the backend, not visible to the `FileStorage` API.

---

## Hooks and Callbacks

**Question:** Should AnyFS support hooks or callbacks for file operations (e.g., audit logging, validation)?

**Considerations:**
- AgentFS (see comparison below) provides audit logging as a core feature
- Hooks add complexity but enable powerful use cases
- Could be implemented as a middleware pattern around FileStorage

**Resolution:** Implemented via `Tracing` middleware. Users can also wrap `FileStorage` or backends for custom hooks.

---

## AgentFS Comparison

**Note:** There are two projects named "AgentFS":

| Project                                                           | Description                                                           |
| ----------------------------------------------------------------- | --------------------------------------------------------------------- |
| [tursodatabase/agentfs](https://github.com/tursodatabase/agentfs) | Full AI agent runtime (Turso/libSQL)                                  |
| [cryptopatrick/agentfs](https://github.com/cryptopatrick/agentfs) | Related to [AgentDB](https://lib.rs/crates/agentdb) abstraction layer |

This section focuses on **Turso's AgentFS**, which has a [published spec](https://github.com/tursodatabase/agentfs/blob/main/SPEC.md).

### What AgentFS Provides

AgentFS is an **agent runtime**, not just a filesystem. It provides three integrated subsystems:

1. **Virtual Filesystem** - POSIX-like, inode-based, chunked storage in SQLite
2. **Key-Value Store** - Agent state and context storage
3. **Tool Call Audit Trail** - Records all tool invocations for debugging/compliance

### AnyFS vs AgentFS: Different Abstractions

| Concern             | AnyFS                           | AgentFS            |
| ------------------- | ------------------------------- | ------------------ |
| **Scope**           | Filesystem abstraction          | Agent runtime      |
| **Filesystem**      | Full                            | Full               |
| **Key-Value store** | Not our domain                  | Included           |
| **Tool auditing**   | `Tracing` middleware            | Built-in           |
| **Backends**        | Memory, SQLite, VRootFs, custom | SQLite only (spec) |
| **Middleware**      | Composable layers               | Monolithic         |

### Relationship Options

**AnyFS could be used BY AgentFS:**
- AgentFS could implement its filesystem portion using `Fs` trait
- Our middleware (Quota, PathFilter, etc.) would work with their system

**AgentFS-compatible backend for AnyFS:**
- Someone could implement `Fs` using AgentFS's SQLite schema
- Would enable interop with AgentFS tooling

**What we should NOT do:**
- Add KV store to `Fs` (different abstraction, scope creep)
- Add tool call auditing to core trait (that's what `Tracing` middleware is for)

### When to Use Which

| Use Case                                       | Recommendation                      |
| ---------------------------------------------- | ----------------------------------- |
| Need just filesystem operations                | **AnyFS**                           |
| Need composable middleware (quota, sandboxing) | **AnyFS**                           |
| Need full agent runtime (FS + KV + auditing)   | **AgentFS**                         |
| Need multiple backend types (memory, real FS)  | **AnyFS**                           |
| Need AgentFS-compatible SQLite format          | **AgentFS** or custom AnyFS backend |

### Takeaway

AnyFS and AgentFS solve different problems at different layers:
- **AnyFS** = filesystem abstraction with composable middleware
- **AgentFS** = complete agent runtime with integrated storage

They can complement each other rather than compete.

---

## VFS Crate Comparison

The [vfs crate](https://docs.rs/vfs/) provides virtual filesystem abstractions with:
- **PhysicalFS**: Host filesystem access
- **MemoryFS**: In-memory storage
- **AltrootFS**: Rooted filesystem (similar to our VRootFsBackend)
- **OverlayFS**: Layered filesystem
- **EmbeddedFS**: Compile resources into binary

**Similarities with AnyFS:**
- Trait-based abstraction over storage
- Memory and physical filesystem backends

**Differences:**
- VFS doesn't have SQLite backend
- VFS doesn't have policy/quota layer
- AnyFS focuses on isolation and limits

**Why not use VFS?** VFS is a good library, but AnyFS's design goals differ:
1. We want SQLite as a first-class backend
2. We need quota/limit enforcement
3. We want feature whitelisting (least privilege)

---

## FUSE Mount Support

**Status:** Designed - Part of `anyfs` crate (feature flags: `fuse`, `winfsp`)

**What is FUSE?**
[FUSE (Filesystem in Userspace)](https://en.wikipedia.org/wiki/Filesystem_in_Userspace) allows implementing filesystems in userspace rather than kernel code. It enables:
- Mounting any backend as a real filesystem
- Using standard Unix tools (ls, cat, etc.) on AnyFS containers
- Integration with existing workflows

**Resolution:** Part of `anyfs` crate with feature flags:
- Linux: FUSE (native) - `fuse` feature
- macOS: macFUSE - `fuse` feature
- Windows: WinFsp - `winfsp` feature

See [Cross-Platform Mounting](../guides/mounting.md) for full details.

---

## Type-System Protection for Cross-Container Operations

**Status:** Resolved - Implemented via marker types

**Question:** Should we use the type system to prevent accidentally mixing data between containers?

**Resolution:** Implemented via `FileStorage<B, M>` where `M` is a marker type:

```rust
struct Sandbox;
struct UserData;

let sandbox: FileStorage<_, Sandbox> = FileStorage::new(MemoryBackend::new());
let userdata: FileStorage<_, UserData> = FileStorage::new(SqliteBackend::open("data.db")?);

fn process_sandbox(fs: &FileStorage<impl Fs, Sandbox>) { /* only accepts Sandbox */ }

process_sandbox(&sandbox);   // OK
process_sandbox(&userdata);  // Compile error!
```

See [FileStorage<B, M>](../traits/files-container.md) for details.

---

## Naming Considerations

Based on review feedback, the following naming concerns were raised:

| Current Name      | Concern                                   | Alternatives Considered         |
| ----------------- | ----------------------------------------- | ------------------------------- |
| `anyfs-traits`    | "traits" is vague                         | `anyfs-backend` (adopted)       |
| `anyfs-container` | Could imply Docker                        | Merged into `anyfs` (adopted)   |
| `anyfs`           | Sounds like Hebrew "ani efes" (I am zero) | `anyfs` retained for simplicity |

**Decision:** Renamed `anyfs-traits` to `anyfs-backend`. Merged `anyfs-container` into `anyfs`.

---

## POSIX Behavior

**Question:** How POSIX-compatible should AnyFS be?

**Answer:** AnyFS is **not** a POSIX emulator. We use `std::fs`-like naming and semantics for familiarity, but we don't aim for full POSIX compliance. Specific differences:

- Symlink-aware path resolution (FileStorage walks the virtual structure using `metadata()` and `read_link()`)
- No file descriptors or open file handles in the basic API
- Simplified permissions model
- No device files, FIFOs, or sockets

---

## Async Support

**Question:** Should `Fs` traits be async?

**Decision:** Sync-first, async-ready (see ADR-010).

**Rationale:**
- All built-in backends are naturally synchronous (rusqlite, std::fs, memory)
- No runtime dependency (tokio/async-std) required
- Rust 1.75+ has native async traits, so adding later is low-cost

**Async-ready design:**
- Traits require `Send` - compatible with async executors
- Return types are `Result<T, FsError>` - works with async
- No hidden blocking state
- Methods are stateless per-call

**Future path:** When needed (e.g., S3/network backends), add parallel `AsyncFs` trait:
- Separate trait, not replacing `Fs`
- Blanket impl possible via `spawn_blocking`
- No breaking changes to existing sync API

---

## Summary

| Topic                     | Decision                                                                      |
| ------------------------- | ----------------------------------------------------------------------------- |
| Symlink security          | Backend-defined (`FsLink`); VRootFsBackend uses `strict-path` for containment |
| Path resolution           | FileStorage (symlink-aware); VRootFs = OS via `SelfResolving`                 |
| Compression/encryption    | Backend responsibility                                                        |
| Hooks/callbacks           | `Tracing` middleware                                                          |
| FUSE mount                | Part of `anyfs` crate (`fuse`, `winfsp` feature flags)                        |
| Type-system protection    | `FileStorage<B, M>` marker types                                              |
| POSIX compatibility       | Not a goal                                                                    |
| `truncate`                | Added to `FsWrite`                                                            |
| `sync` / `fsync`          | Added to `FsSync`                                                             |
| Async support             | Sync-first, async-ready (ADR-010)                                             |
| Layer trait               | Tower-style composition (ADR-011)                                             |
| Logging                   | Tracing with tracing ecosystem (ADR-012)                                      |
| Extension methods         | `FsExt` (ADR-013)                                                             |
| Zero-copy bytes           | Optional `bytes` feature (ADR-014)                                            |
| Error context             | Contextual `FsError` (ADR-015)                                                |
| BackendStack builder      | Fluent API via `.layer()`                                                     |
| Path-based access control | `PathFilter` middleware (ADR-016)                                             |
| Read-only mode            | `ReadOnly` middleware (ADR-017)                                               |
| Rate limiting             | `RateLimit` middleware (ADR-018)                                              |
| Dry-run testing           | `DryRun` middleware (ADR-019)                                                 |
| Read caching              | `Cache` middleware (ADR-020)                                                  |
| Union filesystem          | `Overlay` middleware (ADR-021)                                                |
