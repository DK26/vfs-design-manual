# Open Questions & Future Considerations

**Status:** Under Discussion
**Last Updated:** 2025-12-23

---

This document captures open questions and design considerations that may influence future development of AnyFS.

---

## Symlink Handling with VRootFsBackend

**Question:** If VRootFsBackend wraps a real filesystem directory, and the user has symlinks disabled at the FilesContainer level, what happens to symlinks that exist in the underlying host filesystem?

**Considerations:**

1. **VRootFsBackend inherits real FS behavior**: The underlying filesystem still has symlinks. Even if we don't expose symlink *creation* APIs, operations like `read` or `metadata` might still follow symlinks on the host.

2. **Options:**
   - **Transparent following**: VRootFsBackend always follows symlinks on the host (simplest, matches host behavior)
   - **Refuse symlinks**: VRootFsBackend returns an error if it encounters a symlink
   - **Configurable per-backend**: Let VRootFsBackend accept a `follow_symlinks: bool` option

3. **Recommendation:** VRootFsBackend should transparently follow symlinks on the host filesystem. The "symlinks disabled" policy at the container level means *creating* symlinks is disabled, not that the backend must refuse to traverse existing ones.

---

## Virtual vs Real Backends: Path Resolution

**Question:** Should path resolution logic be different for virtual backends (memory, SQLite) vs filesystem-based backends (VRootFsBackend)?

**Analysis:**

| Backend Type | Path Resolution | Symlink Handling |
|--------------|-----------------|------------------|
| MemoryBackend | Pure lexical (in our code) | We control everything |
| SqliteBackend | Pure lexical (in our code) | We control everything |
| VRootFsBackend | OS handles it | OS follows symlinks on host |

**Recommendation:** For virtual backends, we perform lexical path resolution ourselves. For VRootFsBackend, we delegate to the OS via `std::fs`. The `strict-path::VirtualRoot` ensures we can't escape the root directory, but symlink resolution within that root is handled by the OS.

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

[AgentFS](https://github.com/tursodatabase/agentfs) is a filesystem designed for AI agents with:
- **Auditability**: Every operation recorded in SQLite
- **Reproducibility**: Snapshot and restore agent state
- **Portability**: Everything in a single SQLite file

**Similarities with AnyFS:**
- SQLite as portable storage
- Single-file deployment
- Agent/tenant isolation

**Differences:**
- AgentFS is specialized for AI agent workflows (includes key-value store, tool call logging)
- AnyFS is a general-purpose virtual filesystem abstraction
- AgentFS has more features; AnyFS has more flexibility

**Could AnyFS use AgentFS?** Possibly as a backend, but AgentFS's API is different from `VfsBackend`. Integration would require an adapter.

**Takeaway:** AgentFS validates the SQLite-based portable filesystem approach. We could learn from their audit logging if we add hooks in the future.

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

These questions inform future development but don't block v1:

| Topic | v1 Decision |
|-------|-------------|
| Symlinks with VRootFsBackend | Transparent following on host |
| Path resolution | Virtual = lexical; VRootFs = OS |
| Compression/encryption | Backend responsibility |
| Hooks/callbacks | Defer to v2 |
| FUSE mount | Possible with current trait (has truncate, fsync, statfs) |
| Type-system protection | Defer |
| POSIX compatibility | Not a goal |
| `truncate` | ✅ Added to VfsBackend |
| `sync` / `fsync` | ✅ Added to VfsBackend |
| Async support | ✅ Sync-first, async-ready (ADR-010) |
