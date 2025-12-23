# AnyFS Container - Comparison and Positioning

**How AnyFS Container compares to existing solutions**

AnyFS Container (`anyfs-container`) is a policy layer (quotas + isolation) over a pluggable virtual filesystem backend (`VfsBackend` from `anyfs-traits`). With the SQLite backend (`anyfs` feature `sqlite`), a tenant's filesystem becomes a single portable `.db` file. It defaults to a least-privilege feature set; advanced features (symlinks, hard links, permissions) are opt-in.

---

## Executive Comparison

| Solution | Type | Isolation | Portability | Multi-tenant | Backend choice |
|----------|------|----------:|------------:|-------------:|---------------:|
| **AnyFS Container** | Library | Yes | Backend-dependent (SQLite = single file) | Yes (per-tenant containers + quotas) | Yes (trait-based) |
| SQLAR | Archive format | No | Yes (single file) | No | No (SQLite only) |
| libsqlfs | FUSE filesystem | Varies (via mount) | Yes (single file) | No | No (SQLite only) |
| AgentFS | Agent sandbox | Yes (namespaces) | Yes (SQLite) | Partial | No (SQLite only) |
| `vfs` crate | Abstraction | Backend-dependent | Backend-dependent | No | Yes (trait-based) |
| Docker/containers | Runtime | Yes | Complex | Yes | N/A |

---

## Detailed Comparisons

### vs. SQLAR (SQLite archive)

**What it is:** SQLite's official archive format. It's great for import/export and backups, but it's not a VFS API.

**Schema (representative):**
```sql
CREATE TABLE sqlar(
  name TEXT PRIMARY KEY,
  mode INT,
  mtime INT,
  sz INT,
  data BLOB
);
```

| Aspect | SQLAR | AnyFS Container |
|--------|-------|----------------|
| API | SQL statements | `FilesContainer` methods (`read`, `write`, `read_dir`, ...) |
| Quotas | No | Yes (container-enforced) |
| Isolation | No | Yes (validated virtual paths, no host escape) |
| Transactions (public API) | Manual | No (SQLite backend uses transactions internally) |
| Link semantics | Not the focus | Supported (opt-in: symlink + hard link + permissions) |

**Use SQLAR when:** you want a simple archive/restore format.

**Use AnyFS Container when:** your application needs filesystem-like operations + isolation + quotas.

---

### vs. libsqlfs (Guardian Project)

**What it is:** A FUSE filesystem backed by SQLite, aimed at secure mobile storage.

| Aspect | libsqlfs | AnyFS Container |
|--------|----------|----------------|
| Interface | Mounted filesystem | Library API |
| Deployment | Requires FUSE + OS integration | No mount, no host path access |
| Scope | POSIX filesystem | `std::fs`-aligned trait (not full POSIX) |

**Use libsqlfs when:** you need a real mounted filesystem.

**Use AnyFS Container when:** you want embedded application storage without mounting.

---

### vs. AgentFS (Turso)

**What it is:** A SQLite-backed virtual filesystem aimed at sandboxing AI agents.

| Aspect | AgentFS | AnyFS Container |
|--------|---------|----------------|
| Focus | Agent sandboxing + audit | General-purpose embedded storage |
| Isolation | OS namespaces | Structural (validated virtual paths) |
| Backend | SQLite only | Pluggable (SQLite is one option) |

**Use AgentFS when:** you want OS-level sandboxing + auditing.

**Use AnyFS Container when:** you want a small, embeddable library with backend choice and quotas.

---

### vs. `vfs` crate (Rust)

**What it is:** A general Rust VFS trait + backends.

| Aspect | `vfs` crate | AnyFS Container |
|--------|-------------|----------------|
| Trait design | Path-based operations | Path-based `VfsBackend` (`std::fs`-aligned) |
| Path safety | Backend-dependent | Centralized validation into `VirtualPath` |
| SQLite backend | Not included | Built-in backend (feature-gated) |
| Quotas | No | Yes (`anyfs-container`) |

**Use `vfs` when:** you want a drop-in abstraction close to `std::fs`.

**Use AnyFS Container when:** you need portable storage + quotas + containment guarantees.

---

### vs. Docker/containers

**What it is:** OS-level virtualization with isolated filesystem namespaces.

| Aspect | Containers | AnyFS Container |
|--------|------------|----------------|
| Isolation level | OS process | Library |
| Overhead | High | Low |
| Primary use | Execute code | Store data |

**Use containers when:** you need to run untrusted code.

**Use AnyFS Container when:** you just need isolated application data storage.

---

## Feature Matrix

| Feature | AnyFS Container | SQLAR | libsqlfs | AgentFS | `vfs` crate |
|---------|------------------|-------|----------|---------|------------|
| Directories | Yes | Yes | Yes | Yes | Yes |
| Files | Yes | Yes | Yes | Yes | Yes |
| Symlinks | Yes | Limited/unstandardized | Yes | Varies | Backend-dependent |
| Hard links | Yes | No | Yes | Varies | Backend-dependent |
| Permissions | Yes (simple) | Limited (mode only) | Yes | Varies | Backend-dependent |
| Extended attrs | No (future) | No | Varies | Varies | No |
| Transactions (public API) | No | Manual | Yes | Varies | No |
| Capacity limits | Yes (built-in) | No | No | Varies | No |
| Streaming I/O | No (future) | No | Yes | Varies | Backend-dependent |
| Async API | No (future) | N/A | No | Varies | Backend-dependent |
| FUSE mount | No (by design) | No | Yes | Yes | Backend-dependent |
| Compression | Backend | Yes (deflate) | Varies | Varies | Backend-dependent |
| Encryption | Backend | No | Yes (SQLCipher) | Varies | Backend-dependent |

---

## When to Use AnyFS Container

**Good fit:**
- Multi-tenant SaaS that needs per-tenant quotas
- Desktop apps that want portable user data
- Testing that needs deterministic isolated storage
- Plugin systems that need per-plugin isolation

**Not a good fit:**
- You must mount a filesystem into the host OS (use a FUSE solution)
- You need full POSIX behavior (xattrs/ACLs/etc.)
- You need maximum throughput over safety/portability

---

## Migration Paths

### From raw SQLite/SQLAR

```rust
// Old: raw SQL
conn.execute("INSERT INTO sqlar(name, data) VALUES (?, ?)", [path, data])?;

// New: AnyFS Container
container.write(path, data)?;
```

### From `std::fs`

```rust
// Old: host filesystem
std::fs::write("/data/file.txt", content)?;

// New: AnyFS Container
container.write("/data/file.txt", content)?;
```

---

## Summary

AnyFS Container occupies a niche between archives (SQLAR) and mounted filesystems (FUSE): it is an **application-level** filesystem API with **quotas** and **containment guarantees**, with **backend choice** (including a portable SQLite option).

*For technical details, see the [Design Overview](../architecture/design-overview.md).* 
