# AnyFS - Comparison and Positioning

**How AnyFS compares to existing solutions**

AnyFS is a **filesystem abstraction with composable middleware**. It separates storage (backends) from policy (middleware) using a Tower-style pattern. With the SQLite backend, a tenant's filesystem becomes a single portable `.db` file.

---

## Executive Comparison

| Solution | Type | Middleware | Backend Choice | Quotas | Portable Storage |
|----------|------|:----------:|:--------------:|:------:|:----------------:|
| **AnyFS** | Library | Yes | Yes | Yes | Yes (SQLite) |
| `vfs` crate | Library | No | Yes | No | No |
| AgentFS | Runtime | No | No (SQLite) | No | Yes |
| SQLAR | Format | N/A | No (SQLite) | No | Yes |
| libsqlfs | FUSE | N/A | No (SQLite) | No | Yes |
| OpenDAL | Library | Yes | Yes (cloud) | No | No |

---

## Detailed Comparisons

### vs. `vfs` crate (Rust)

| Aspect | `vfs` crate | AnyFS |
|--------|-------------|-------|
| Middleware pattern | No | Yes (Tower-style) |
| Quotas/limits | No | Yes (Quota middleware) |
| Path sandboxing | Backend-dependent | Yes (PathFilter middleware) |
| Feature gating | No | Yes (Restrictions middleware) |
| SQLite backend | No | Yes (built-in) |
| Path containment | Prefix-based (AltrootFS) | Canonicalization-based (`strict-path`) |
| Third-party extensibility | Implement trait | Implement trait + Layer |
| Path type | Custom `VfsPath` | `impl AsRef<Path>` (std-compatible) |

**Path containment difference:**

The `vfs` crate's `AltrootFS` uses path prefix translation - it prepends a root path before delegating to the underlying filesystem. This is vulnerable to symlink-based escapes:

```
AltrootFS root: /data/tenant1/
Symlink: /data/tenant1/link → ../tenant2/secrets.txt
Access: /link → escapes to /data/tenant2/secrets.txt
```

AnyFS's `VRootFsBackend` uses `strict-path` for full canonicalization. Symlinks are resolved and validated *before* any filesystem operation, preventing escapes.

**Use `vfs` when:** You need simple VFS abstraction without policies or security-sensitive containment.

**Use AnyFS when:** You need composable middleware (quotas, sandboxing, logging) or hardened path containment.

---

### vs. AgentFS (Turso)

| Aspect | AgentFS | AnyFS |
|--------|---------|-------|
| Scope | Agent runtime (FS + KV + auditing) | Filesystem abstraction |
| Backend choice | SQLite only | Memory, SQLite, RealFS, custom |
| Middleware | No | Yes (composable) |
| KV store | Included | Not included (different abstraction) |
| Tool auditing | Built-in | Use Tracing middleware |

**Use AgentFS when:** You need a complete AI agent runtime with KV and auditing.

**Use AnyFS when:** You need just filesystem with backend flexibility and composable policies.

---

### vs. SQLAR (SQLite archive)

| Aspect | SQLAR | AnyFS |
|--------|-------|-------|
| API | SQL statements | Filesystem methods |
| Quotas | No | Yes |
| Middleware | No | Yes |
| Streaming I/O | No | Yes |

**Use SQLAR when:** You want a simple archive format.

**Use AnyFS when:** You need filesystem operations with policies.

---

### vs. libsqlfs (Guardian Project)

| Aspect | libsqlfs | AnyFS |
|--------|----------|-------|
| Interface | FUSE mount | Library API |
| Deployment | Requires OS integration | No mount needed |
| Backend choice | SQLite only | Multiple |

**Use libsqlfs when:** You need a mounted filesystem.

**Use AnyFS when:** You want embedded storage without mounting.

---

### vs. OpenDAL

| Aspect | OpenDAL | AnyFS |
|--------|---------|-------|
| Focus | Cloud object storage | Local/embedded filesystem |
| Async | Yes (async-first) | Sync-first (async planned) |
| Backends | Cloud services (S3, GCS, etc.) | Local (Memory, SQLite, RealFS) |
| Middleware | Yes | Yes |

**Use OpenDAL when:** Your storage is cloud object storage.

**Use AnyFS when:** Your storage is local/embedded.

---

## Feature Matrix

| Feature | AnyFS | `vfs` | AgentFS | OpenDAL |
|---------|:-----:|:-----:|:-------:|:-------:|
| Composable middleware | Yes | No | No | Yes |
| Multiple backends | Yes | Yes | No | Yes |
| SQLite backend | Yes | No | Yes | No |
| Memory backend | Yes | Yes | No | Yes |
| Real FS backend | Yes | Yes | No | No |
| Quota enforcement | Yes | No | No | No |
| Path sandboxing | Yes | Partial | No | No |
| Symlink-safe containment | Yes | No | N/A | N/A |
| Rate limiting | Yes | No | No | No |
| Streaming I/O | Yes | Yes | Yes | Yes |
| Async API | Planned | Partial | No | Yes |
| FUSE mount | Possible | No | Yes | No |

---

## When to Use AnyFS

**Good fit:**
- Multi-tenant SaaS with per-tenant quotas
- AI agent sandboxing with path restrictions
- Desktop apps with portable user data
- Plugin systems with isolated storage
- Testing with deterministic isolated storage
- Any case where you might swap backends later

**Not a good fit:**
- You must mount a filesystem (use FUSE solution)
- You need full POSIX behavior (xattrs, ACLs)
- Your storage is cloud object storage (use OpenDAL)
- You need async-first design today (wait for AnyFS async)

---

## Migration Examples

### From `std::fs`

```rust
// Before: host filesystem
std::fs::write("/data/file.txt", content)?;

// After: AnyFS (can swap backend later)
fs.write("/data/file.txt", content)?;
```

### From raw SQLite

```rust
// Before: raw SQL
conn.execute("INSERT INTO files(path, data) VALUES (?, ?)", [path, data])?;

// After: AnyFS with SQLite backend
fs.write(path, data)?;
```

---

## Summary

AnyFS is positioned as the **Tower/Axum of filesystems**: composable middleware over pluggable backends. It fills the gap between simple VFS abstractions (no policies) and complete runtimes (too much bundled together).

*For technical details, see [Design Overview](../architecture/design-overview.md).*
