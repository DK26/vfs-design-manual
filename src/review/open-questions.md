# Open Questions & Future Considerations

**Status:** Under Discussion
**Last Updated:** 2025-12-24

---

This document captures open questions and design considerations that may influence future development of AnyFS.

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
fn resolve_no_follow(&self, path: &Path) -> Result<PathBuf, VfsError> {
    let mut current = self.root.clone();
    for component in path.components() {
        current = current.join(component);
        let meta = std::fs::symlink_metadata(&current)?;
        if meta.file_type().is_symlink() {
            return Err(VfsError::SymlinkNotAllowed { path: current });
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
pub trait VirtualVfsBackend: VfsBackend {
    fn set_symlink_resolution(&mut self, enabled: bool);
}

/// Real filesystem backends where OS controls symlink resolution
pub trait RealFsBackend: VfsBackend {
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
pub trait VfsBackend {
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
- Document that `FeatureGuard` symlink control only applies to virtual backends

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

### v1 Decision: Option 4 (Accept the Limitation)

After analysis, **Option 4** is the pragmatic choice for v1:

1. **Remove misleading "allow symlinks" from FeatureGuard** - `FeatureGuard` controls *operations* (can you call `symlink()`?), not path resolution behavior.

2. **Add `set_follow_symlinks(bool)` to virtual backends only:**
   ```rust
   impl MemoryBackend {
       pub fn set_follow_symlinks(&mut self, follow: bool);
   }
   impl SqliteBackend {
       pub fn set_follow_symlinks(&mut self, follow: bool);
   }
   ```

3. **Document the difference clearly** in security docs (done - see `security.md` section 8).

4. **Accept that VRootFsBackend works differently** - `strict-path` provides escape protection, which is sufficient for most security use cases.

### Rationale

- **Both approaches are secure against jail escapes** - just via different mechanisms
- **Two traits adds complexity** most users won't benefit from
- **Custom path resolution has TOCTOU vulnerabilities** - worse than accepting the limitation
- **The narrow use case** ("no symlink following at all" on real filesystem) is rare enough to defer

### Future Considerations

If demand emerges for "no symlink following" on real filesystems:

1. **Option 1 (custom path resolution)** could be added as opt-in, with documented TOCTOU tradeoffs
2. **Option 3 (capabilities)** could be added to let middleware query backend behavior
3. **A separate `SafeArchiveExtractor`** utility could implement the strict semantics needed for archive extraction

---

## Virtual vs Real Backends: Path Resolution

**Question:** Should path resolution logic be different for virtual backends (memory, SQLite) vs filesystem-based backends (VRootFsBackend)?

**Analysis:**

| Backend Type | Path Resolution | Symlink Handling |
|--------------|-----------------|------------------|
| MemoryBackend | Pure lexical (our code) | We control everything |
| SqliteBackend | Pure lexical (our code) | We control everything |
| VRootFsBackend | OS handles it | OS follows symlinks |

**Current design:** For virtual backends, we perform lexical path resolution ourselves. For VRootFsBackend, we delegate to the OS via `std::fs`, with `strict-path::VirtualRoot` ensuring containment.

**Open question:** Should we abstract this difference, accept it, or use two traits?

---

## Compression and Encryption

**Question:** Does the current design allow backends to compress/decompress or encrypt/decrypt files transparently?

**Answer:** Yes. The backend receives the data and stores it however it wants. A backend could:
- Compress data before writing to SQLite
- Encrypt blobs with a user-provided key
- Use a remote object store with encryption at rest

This is an implementation detail of the backend, not visible to the `FilesContainer` API.

---

## Hooks and Callbacks

**Question:** Should AnyFS support hooks or callbacks for file operations (e.g., audit logging, validation)?

**Considerations:**
- AgentFS (see comparison below) provides audit logging as a core feature
- Hooks add complexity but enable powerful use cases
- Could be implemented as a middleware pattern around FilesContainer

**Options:**
1. **No hooks in v1**: Keep it simple. Users can wrap FilesContainer in their own type.
2. **Event emitter**: FilesContainer emits events that users can subscribe to
3. **Middleware trait**: Allow wrapping backends with cross-cutting concerns

**Recommendation:** Defer to v2. Users can wrap `FilesContainer` or backends for now.

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
- AgentFS could implement its filesystem portion using `VfsBackend`
- Our middleware (Quota, PathFilter, etc.) would work with their system

**AgentFS-compatible backend for AnyFS:**
- Someone could implement `VfsBackend` using AgentFS's SQLite schema
- Would enable interop with AgentFS tooling

**What we should NOT do:**
- Add KV store to `VfsBackend` (different abstraction, scope creep)
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

**Question:** Should AnyFS support mounting as a FUSE filesystem?

**What is FUSE?**
[FUSE (Filesystem in Userspace)](https://en.wikipedia.org/wiki/Filesystem_in_Userspace) allows implementing filesystems in userspace rather than kernel code. It enables:
- Mounting any backend as a real filesystem
- Using standard Unix tools (ls, cat, etc.) on AnyFS containers
- Integration with existing workflows

**Considerations:**
- FUSE requires platform-specific code (Linux, macOS via macFUSE)
- Adds significant complexity
- Performance overhead vs direct API access
- Security implications of exposing to OS

**Recommendation:** Not in v1. If there's demand, we could add a `anyfs-fuse` crate that mounts a FilesContainer as a FUSE filesystem. This would be a separate, optional layer.

---

## Type-System Protection for Cross-Container Operations

**Question:** Should we use the type system to prevent accidentally mixing data between containers?

**Example concern:** Reading from Container A and writing to Container B without explicit acknowledgment.

**Options:**
1. **Marker generics**: `FilesContainer<B, Marker>` where Marker distinguishes instances
2. **Opaque handles**: File content wrapped in typed handles tied to origin container
3. **No protection**: Trust the user to wire things correctly

**Recommendation:** Defer. The complexity may not be worth it for most use cases. Users who need this can implement their own type wrappers.

---

## Naming Considerations

Based on review feedback, the following naming concerns were raised:

| Current Name | Concern | Alternatives Considered |
|--------------|---------|------------------------|
| `anyfs-traits` | "traits" is vague | `anyfs-backend` (adopted) |
| `anyfs-container` | Could imply Docker | `anyfs-storage`, `anyfs-fs`, `fsContainer` |
| `anyfs` | Sounds like Hebrew "ani efes" (I am zero) | `anyfs` retained for simplicity |

**Decision:** Renamed `anyfs-traits` to `anyfs-backend`. Other names retained.

---

## POSIX Behavior

**Question:** How POSIX-compatible should AnyFS be?

**Answer:** AnyFS is **not** a POSIX emulator. We use `std::fs`-like naming and semantics for familiarity, but we don't aim for full POSIX compliance. Specific differences:

- Lexical path resolution (not runtime symlink following during normalization)
- No file descriptors or open file handles in the basic API
- Simplified permissions model
- No device files, FIFOs, or sockets

---

## Async Support

**Question:** Should VfsBackend be async?

**Decision:** Sync-first, async-ready (see ADR-010).

**Rationale:**
- All built-in backends are naturally synchronous (rusqlite, std::fs, memory)
- No runtime dependency (tokio/async-std) required
- Rust 1.75+ has native async traits, so adding later is low-cost

**Async-ready design:**
- Trait requires `Send` - compatible with async executors
- Return types are `Result<T, VfsError>` - works with async
- No hidden blocking state
- Methods are stateless per-call

**Future path:** When needed (e.g., S3/network backends), add parallel `AsyncVfsBackend` trait:
- Separate trait, not replacing `VfsBackend`
- Blanket impl possible via `spawn_blocking`
- No breaking changes to existing sync API

---

## Summary

| Topic | v1 Decision |
|-------|-------------|
| Symlink security | Virtual backends: `set_follow_symlinks()`. VRootFsBackend: `strict-path` escapes only. |
| Path resolution | Virtual = lexical; VRootFs = OS |
| Compression/encryption | Backend responsibility |
| Hooks/callbacks | Defer to v2 |
| FUSE mount | Possible with current trait (has truncate, fsync, statfs) |
| Type-system protection | Defer |
| POSIX compatibility | Not a goal |
| `truncate` | Added to VfsBackend |
| `sync` / `fsync` | Added to VfsBackend |
| Async support | Sync-first, async-ready (ADR-010) |
| Layer trait | Tower-style composition (ADR-011) |
| Logging | Tracing with tracing ecosystem (ADR-012) |
| Extension methods | VfsBackendExt (ADR-013) |
| Zero-copy bytes | Optional `bytes` feature (ADR-014) |
| Error context | Contextual VfsError (ADR-015) |
| BackendStack builder | Fluent API in anyfs-container |
| Path-based access control | PathFilter middleware (ADR-016) |
| Read-only mode | ReadOnly middleware (ADR-017) |
| Rate limiting | RateLimit middleware (ADR-018) |
| Dry-run testing | DryRun middleware (ADR-019) |
| Read caching | Cache middleware (ADR-020) |
| Union filesystem | Overlay middleware (ADR-021) |
