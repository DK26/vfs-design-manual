# Open Questions & Future Considerations

**Status:** Under Discussion
**Last Updated:** 2025-12-28

---

This document captures open questions and design considerations that may influence future development of AnyFS.

> **Note:** Many items originally in this document have been resolved and implemented. See the Architecture Decision Records for final decisions.

---

## Symlink Security: Following vs Creating

**IMPORTANT:** The current discussion about symlinks has been fundamentally reconsidered.

### The Correct Understanding

| Action | Security Concern? | Why |
|--------|-------------------|-----|
| **Creating symlinks** | No | Symlinks are just data. Creating them is harmless. |
| **Following symlinks** | **YES** | Following a symlink can escape containment. |

**Example attack:**
```
/sandbox/escape -> /etc/passwd

If we follow the symlink:
  read("/sandbox/escape") → reads /etc/passwd → JAIL ESCAPE
```

The security feature is **controlling symlink resolution**, not symlink creation.

### Virtual vs Real Backends: A Fundamental Difference

| Backend | Who Controls Symlink Resolution? |
|---------|----------------------------------|
| `MemoryBackend` | **We do** (symlinks are stored data, we choose whether to follow) |
| `SqliteBackend` | **We do** (symlinks are stored data, we choose whether to follow) |
| `VRootFsBackend` | **The OS does** (we call `std::fs::read`, OS follows symlinks) |

**This is the core problem:** For virtual backends, we have full control. For real filesystem backends, the OS controls symlink resolution and we cannot easily change that.

### Options Analysis

#### Option 1: Custom Path Resolution for VRootFsBackend

Walk each path component manually using `lstat` (symlink_metadata):

```rust
fn resolve_no_follow(&self, path: &Path) -> Result<PathBuf, FsError> {
    let mut current = self.root.clone();
    for component in path.components() {
        current = current.join(component);
        let meta = std::fs::symlink_metadata(&current)?;
        if meta.file_type().is_symlink() {
            return Err(FsError::SymlinkNotAllowed { path: current });
        }
    }
    Ok(current)
}
```

| Pros | Cons |
|------|------|
| Would work | Complex implementation |
| Consistent with virtual backends | TOCTOU vulnerability (symlink created between check and use) |
| | Performance overhead (lstat per component) |
| | Platform differences (Windows junctions, etc.) |

#### Option 2: Two Different Traits

```rust
/// Virtual backends where we control everything
pub trait VirtualFs: Fs {
    fn set_symlink_resolution(&self, enabled: bool);
}

/// Real filesystem backends where OS controls symlink resolution
pub trait RealFs: Fs {
    // OS controls symlinks, we just ensure containment
}
```

| Pros | Cons |
|------|------|
| Honest about capabilities | Complicates API |
| Clear separation of concerns | Middleware must handle both |
| No false promises | Harder to swap backends |
| | Users must understand the difference |

#### Option 3: Capability Query

```rust
pub trait Fs {
    fn capabilities(&self) -> Capabilities;
    // ...
}

pub struct Capabilities {
    pub can_control_symlink_following: bool,
    pub supports_hard_links: bool,
    // ...
}
```

| Pros | Cons |
|------|------|
| Single trait surface | Runtime capability discovery |
| Backends self-describe | Middleware becomes conditional |
| Extensible | Can't enforce at compile time |

#### Option 4: Accept the Limitation

- For virtual backends: We control symlink following
- For VRootFsBackend: Rely on `strict-path` to prevent escapes
- Document that `Restrictions` symlink control only applies to virtual backends

| Pros | Cons |
|------|------|
| Simple | Inconsistent behavior |
| Honest | Feature doesn't work everywhere |
| No complexity | |

#### Option 5: Drop the Symlink Control Feature

User's suggestion: Low ROI, just drop it.

- Virtual backends: Symlinks are data, inherently safe
- VRootFsBackend: `strict-path` prevents escapes anyway

| Pros | Cons |
|------|------|
| Simplest | Loses archive extraction use case |
| No false expectations | |
| Focus on what we can actually control | |

### The Archive Extraction Use Case

One valid use case for symlink control:

> "Someone using our files container to contain their own user-based-tenants or as an archive extraction, where they wish to prevent symlinks as an easy resolution to make sure there is no jail-escape."

For this use case:
- **Virtual backends**: Works perfectly - we can refuse to follow symlinks
- **VRootFsBackend**: Would need custom path resolution or OS-specific flags

### What `strict-path` Actually Does

`strict-path::VirtualRoot`:
1. Takes a user path
2. Canonicalizes it (follows symlinks, resolves `.` and `..`)
3. Checks the canonical path is within the root
4. Returns the safe path

**Key insight:** `strict-path` FOLLOWS symlinks but ensures the final path is contained. It does NOT prevent symlink following.

```
/root/link -> ../escape
User requests: /link
strict-path: canonicalize(/root/link) = /escape
strict-path: /escape is NOT within /root → DENIED
```

This is secure against escape, but it's "follow and check" not "don't follow".

### v1 Decision: Accept the Limitation

The design is straightforward:

1. **Virtual backends always support symlinks** - creating symlinks is just storing data.

2. **Virtual backends provide `set_follow_symlinks(bool)`** - the actual security control:
   ```rust
   let backend = MemoryBackend::new();
   backend.set_follow_symlinks(false);  // Symlinks not resolved during path operations
   ```

3. **VRootFsBackend cannot control symlink following** - the OS handles resolution. `strict-path` prevents escapes but cannot disable following entirely.

4. **Everything works by default** - all operations (`symlink()`, `hard_link()`, `set_permissions()`) work out of the box. Use `Restrictions` middleware to opt-in to restrictions when needed.

### Summary

| Concern | Virtual Backends | VRootFsBackend |
|---------|------------------|----------------|
| Symlink creation | Always allowed (just data) | Always allowed (just data) |
| Symlink following | `set_follow_symlinks(bool)` | OS controls (strict-path prevents escapes) |
| Jail escape protection | `set_follow_symlinks(false)` | strict-path containment |

### Why VRootFsBackend Cannot Control Symlink Following

VRootFsBackend calls OS functions (`std::fs::read()`, etc.) which follow symlinks automatically. We don't implement path resolution - the OS does.

**Theoretical alternatives (not recommended):**
- Walk path manually before each operation → TOCTOU race condition
- Use `O_NOFOLLOW` flags → Only affects final component, platform-specific
- FUSE mount → Way too complex for a library

**The honest answer:** VRootFsBackend cannot disable symlink following. `strict-path` prevents escapes, but symlinks within the jail are followed by the OS. This is a fundamental limitation of wrapping a real filesystem.

---

## Virtual vs Real Backends: Path Resolution

**Status:** Resolved

**Question:** Should path resolution logic be different for virtual backends (memory, SQLite) vs filesystem-based backends (StdFsBackend, VRootFsBackend)?

**Resolution:** `FileStorage` handles symlink-aware path resolution for ALL backends by default. Backends do NOT perform path resolution - they receive already-resolved paths.

| Backend Type | Path Resolution | Symlink Handling |
|--------------|-----------------|------------------|
| MemoryBackend | **FileStorage** (symlink-aware) | `set_follow_symlinks(bool)` |
| SqliteBackend | **FileStorage** (symlink-aware) | `set_follow_symlinks(bool)` |
| VRootFsBackend | **OS** (implements `SelfResolving`) | OS follows symlinks (strict-path prevents escapes) |

**Key design decision:** Backends that wrap a real filesystem implement the `SelfResolving` marker trait to tell `FileStorage` to skip resolution:

```rust
impl SelfResolving for VRootFsBackend {}
```

This ensures consistent behavior: all virtual backends get symlink-aware resolution from `FileStorage`, while real filesystem backends delegate to the OS.

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

| Project | Description |
|---------|-------------|
| [tursodatabase/agentfs](https://github.com/tursodatabase/agentfs) | Full AI agent runtime (Turso/libSQL) |
| [cryptopatrick/agentfs](https://github.com/cryptopatrick/agentfs) | Related to [AgentDB](https://lib.rs/crates/agentdb) abstraction layer |

This section focuses on **Turso's AgentFS**, which has a [published spec](https://github.com/tursodatabase/agentfs/blob/main/SPEC.md).

### What AgentFS Provides

AgentFS is an **agent runtime**, not just a filesystem. It provides three integrated subsystems:

1. **Virtual Filesystem** - POSIX-like, inode-based, chunked storage in SQLite
2. **Key-Value Store** - Agent state and context storage
3. **Tool Call Audit Trail** - Records all tool invocations for debugging/compliance

### AnyFS vs AgentFS: Different Abstractions

| Concern | AnyFS | AgentFS |
|---------|-------|---------|
| **Scope** | Filesystem abstraction | Agent runtime |
| **Filesystem** | Full | Full |
| **Key-Value store** | Not our domain | Included |
| **Tool auditing** | `Tracing` middleware | Built-in |
| **Backends** | Memory, SQLite, VRootFs, custom | SQLite only (spec) |
| **Middleware** | Composable layers | Monolithic |

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

| Use Case | Recommendation |
|----------|----------------|
| Need just filesystem operations | **AnyFS** |
| Need composable middleware (quota, sandboxing) | **AnyFS** |
| Need full agent runtime (FS + KV + auditing) | **AgentFS** |
| Need multiple backend types (memory, real FS) | **AnyFS** |
| Need AgentFS-compatible SQLite format | **AgentFS** or custom AnyFS backend |

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

**Status:** Resolved - Implemented in `anyfs-mount`

**What is FUSE?**
[FUSE (Filesystem in Userspace)](https://en.wikipedia.org/wiki/Filesystem_in_Userspace) allows implementing filesystems in userspace rather than kernel code. It enables:
- Mounting any backend as a real filesystem
- Using standard Unix tools (ls, cat, etc.) on AnyFS containers
- Integration with existing workflows

**Resolution:** Implemented via `anyfs-mount` crate with cross-platform support:
- Linux: FUSE (native)
- macOS: macFUSE
- Windows: WinFsp

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

| Current Name | Concern | Alternatives Considered |
|--------------|---------|------------------------|
| `anyfs-traits` | "traits" is vague | `anyfs-backend` (adopted) |
| `anyfs-container` | Could imply Docker | Merged into `anyfs` (adopted) |
| `anyfs` | Sounds like Hebrew "ani efes" (I am zero) | `anyfs` retained for simplicity |

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

| Topic | v1 Decision |
|-------|-------------|
| Symlink security | Virtual backends: `set_follow_symlinks()`. VRootFsBackend: `strict-path` escapes only. |
| Path resolution | FileStorage (symlink-aware); VRootFs = OS via `SelfResolving` |
| Compression/encryption | Backend responsibility |
| Hooks/callbacks | `Tracing` middleware |
| FUSE mount | `anyfs-mount` crate (cross-platform) |
| Type-system protection | `FileStorage<B, M>` marker types |
| POSIX compatibility | Not a goal |
| `truncate` | Added to `FsWrite` |
| `sync` / `fsync` | Added to `FsSync` |
| Async support | Sync-first, async-ready (ADR-010) |
| Layer trait | Tower-style composition (ADR-011) |
| Logging | Tracing with tracing ecosystem (ADR-012) |
| Extension methods | `FsExt` (ADR-013) |
| Zero-copy bytes | Optional `bytes` feature (ADR-014) |
| Error context | Contextual `FsError` (ADR-015) |
| BackendStack builder | Fluent API via `.layer()` |
| Path-based access control | `PathFilter` middleware (ADR-016) |
| Read-only mode | `ReadOnly` middleware (ADR-017) |
| Rate limiting | `RateLimit` middleware (ADR-018) |
| Dry-run testing | `DryRun` middleware (ADR-019) |
| Read caching | `Cache` middleware (ADR-020) |
| Union filesystem | `Overlay` middleware (ADR-021) |
