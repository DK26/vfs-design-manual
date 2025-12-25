# Security Considerations

**Security model, threat analysis, and containment guarantees**

---

## Overview

AnyFS is designed with security as a primary concern. Security policies are enforced via **composable middleware**, not hardcoded in backends or the container wrapper.

---

## Threat Model

### In Scope (Mitigated by Middleware)

| Threat | Description | Middleware |
|--------|-------------|------------|
| **Path traversal** | Access files outside allowed paths | `PathFilter` |
| **Symlink attacks** | Use symlinks to bypass controls | Backend-dependent (see below) |
| **Resource exhaustion** | Fill storage or create excessive files | `Quota` |
| **Runaway processes** | Excessive operations consuming resources | `RateLimit` |
| **Unauthorized writes** | Modifications to read-only data | `ReadOnly` |
| **Sensitive file access** | Access to `.env`, secrets, etc. | `PathFilter` |

### Out of Scope

| Threat | Reason |
|--------|--------|
| **Side-channel attacks** | Requires OS-level mitigations |
| **Physical access** | Disk encryption is application's responsibility |
| **SQLite vulnerabilities** | Upstream dependency; update regularly |
| **Network attacks** | AnyFS is local storage, not network-facing |

---

## Security Architecture

### 1. Middleware-Based Policy

Security policies are composable middleware layers:

```rust
use anyfs::{MemoryBackend, Quota, PathFilter, Restrictions, RateLimit, Tracing};

let secure_backend = Tracing::new(           // Audit trail
    RateLimit::new(                          // Throttle operations
        PathFilter::new(                     // Sandbox paths
            Restrictions::new(               // Block dangerous features
                Quota::new(MemoryBackend::new())  // Limit resources
                    .with_max_total_size(100 * 1024 * 1024)
            )
        )
        .allow("/workspace/**")
        .deny("**/.env")
        .deny("**/secrets/**")
    )
    .max_ops(1000)
    .per_second()
);
```

### 2. Path Sandboxing (PathFilter)

`PathFilter` middleware restricts path access using glob patterns:

```rust
PathFilter::new(backend)
    .allow("/workspace/**")    // Allow workspace access
    .deny("**/.env")           // Block .env files
    .deny("**/secrets/**")     // Block secrets directories
    .deny("**/*.key")          // Block key files
```

**Guarantees:**
- First matching rule wins
- No rule = denied (deny by default)
- `read_dir` filters denied entries from results

### 3. Feature Gating (Restrictions)

By default, all operations work. Use `Restrictions` middleware to **opt-in to restrictions**:

```rust
Restrictions::new(backend)
    .deny_hard_links()         // Block hard_link() calls
    .deny_permissions()        // Block set_permissions() calls
    .deny_symlinks()           // Block symlink() calls
```

**Use cases:**
- Sandboxing untrusted code (block permission changes)
- Archive extraction (block symlink/hardlink creation)
- Read-only-ish environments (block permission mutations)

**Note:** This controls operation **availability**. For symlink **following** behavior, see "Symlink Security" below.

### 4. Resource Limits (Quota)

`Quota` middleware enforces capacity limits:

```rust
Quota::new(backend)
    .with_max_total_size(100 * 1024 * 1024)  // 100 MB total
    .with_max_file_size(10 * 1024 * 1024)    // 10 MB per file
    .with_max_node_count(10_000)             // Max files/dirs
    .with_max_dir_entries(1_000)             // Max per directory
    .with_max_path_depth(64)                 // Max nesting
```

**Guarantees:**
- Writes rejected when limits exceeded
- Streaming writes tracked via `CountingWriter`

### 5. Rate Limiting (RateLimit)

`RateLimit` middleware throttles operations:

```rust
RateLimit::new(backend)
    .max_ops(1000)
    .per_second()
```

**Guarantees:**
- Operations rejected when limit exceeded
- Protects against runaway processes

### 6. Backend-Level Containment

Different backends achieve containment differently:

| Backend | Containment Mechanism |
|---------|----------------------|
| `MemoryBackend` | Isolated in process memory |
| `SqliteBackend` | Each container is a separate `.db` file |
| `StdFsBackend` | **None** - full filesystem access (use `PathFilter` for sandboxing) |
| `VRootFsBackend` | Uses `strict-path::VirtualRoot` to contain paths |

### 7. Why Virtual Backends Are Inherently Safe

For `MemoryBackend` and `SqliteBackend`, paths are **just keys** (strings or normalized path components). There is no underlying OS filesystem to exploit.

**Lexical path resolution**: Unlike POSIX filesystems, virtual backends resolve paths **lexically** without consulting any filesystem:

```
POSIX behavior (dangerous):
  /foo/bar/..  where bar → /etc  resolves to /etc/../ → /

AnyFS virtual backend behavior (safe):
  /foo/bar/..  always resolves to /foo (pure string manipulation)
```

This means:
- **No symlink-based escapes** - symlinks are data, not OS-resolved references
- **No TOCTOU via filesystem state** - resolution is deterministic
- **No host path leakage** - paths never touch the OS path resolver

For `VRootFsBackend` (real filesystem), `strict-path::VirtualRoot` provides equivalent guarantees by validating and containing all paths before they reach the OS.

### 8. Symlink Security: Virtual vs Real Backends

**The security concern with symlinks is *following* them, not *creating* them.**

Symlinks are just data. Creating `/sandbox/link -> /etc/passwd` is harmless. The danger is when reading `/sandbox/link` follows the symlink and accesses `/etc/passwd`.

| Backend Type | Symlink Creation | Symlink Following |
|--------------|------------------|-------------------|
| `MemoryBackend` | Always supported | **We control** via `set_follow_symlinks()` |
| `SqliteBackend` | Always supported | **We control** via `set_follow_symlinks()` |
| `VRootFsBackend` | Always supported | **OS controls** - `strict-path` prevents escapes |

#### Virtual Backends (Memory, SQLite)

Virtual backends always support symlinks. They also provide control over symlink following:

```rust
let mut backend = MemoryBackend::new();
backend.set_follow_symlinks(false);  // Don't follow symlinks during path resolution
```

When following is disabled:
- `read("/link")` on a symlink returns `FsError::IsSymlink` (or reads link as opaque data)
- Path resolution treats symlinks as terminal nodes

**This is the actual security feature** - controlling whether symlinks are resolved.

#### Real Filesystem Backend (VRootFsBackend)

VRootFsBackend calls OS functions (`std::fs::read()`, etc.) which follow symlinks automatically. **We cannot control this** - the OS does the symlink resolution, not us.

`strict-path::VirtualRoot` prevents **escapes**:

```
User requests: /sandbox/link
link -> ../../../etc/passwd
strict-path: canonicalize(/sandbox/link) = /etc/passwd
strict-path: /etc/passwd is NOT within /sandbox → DENIED
```

This is "follow and verify containment" - symlinks are followed by the OS, but escapes are blocked by strict-path.

**Limitation:** Symlinks within the jail are followed. We cannot disable this without implementing custom path resolution (TOCTOU risk) or platform-specific hacks.

#### Summary

| Concern | Virtual Backend | VRootFsBackend |
|---------|-----------------|----------------|
| Symlink creation | Always allowed (just data) | Always allowed (just data) |
| Symlink following | `set_follow_symlinks(bool)` | OS controls (strict-path prevents escapes) |
| Jail escape via symlink | `set_follow_symlinks(false)` | Prevented by strict-path |

---

## Secure Usage Patterns

### AI Agent Sandbox

```rust
use anyfs::{MemoryBackend, Quota, PathFilter, Restrictions, RateLimit, Tracing};
use anyfs::FileStorage;

let sandbox = Tracing::new(
    RateLimit::new(
        PathFilter::new(
            Restrictions::new(
                Quota::new(MemoryBackend::new())
                    .with_max_total_size(50 * 1024 * 1024)
                    .with_max_file_size(5 * 1024 * 1024)
            )
        )
        .allow("/workspace/**")
        .deny("**/.env")
        .deny("**/secrets/**")
    )
    .max_ops(1000)
    .per_second()
);

let mut fs = FileStorage::new(sandbox);
// Agent code can only access /workspace, limited resources, audited
```

### Multi-Tenant Isolation

```rust
use anyfs::{SqliteBackend, Quota};
use anyfs::FileStorage;

fn create_tenant_storage(tenant_id: &str, quota_bytes: u64) -> FileStorage<...> {
    let db_path = format!("tenants/{}.db", tenant_id);
    let backend = Quota::new(SqliteBackend::open(&db_path).unwrap())
        .with_max_total_size(quota_bytes);

    FileStorage::new(backend)
}

// Complete isolation: separate database files
```

### Read-Only Browsing

```rust
use anyfs::{SqliteBackend, ReadOnly};
use anyfs::FileStorage;

let readonly_fs = FileStorage::new(
    ReadOnly::new(SqliteBackend::open("archive.db")?)
);

// All write operations return FsError::ReadOnly
```

---

## Security Checklist

### For Application Developers

- [ ] Use `PathFilter` to sandbox untrusted code
- [ ] Use `Quota` to prevent resource exhaustion
- [ ] Use `Restrictions` (default) to disable dangerous features
- [ ] Use `RateLimit` for untrusted/shared environments
- [ ] Use `Tracing` for audit trails
- [ ] Use separate backends for separate tenants
- [ ] Keep dependencies updated

### For Backend Implementers

- [ ] Ensure paths cannot escape intended scope
- [ ] For filesystem backends: use `strict-path` for containment
- [ ] Handle concurrent access safely
- [ ] Don't leak internal paths in errors

### For Middleware Implementers

- [ ] Handle streaming I/O appropriately (wrap or block)
- [ ] Document which operations are intercepted
- [ ] Fail closed (deny on error)

---

## Known Limitations

1. **No encryption at rest**: Use OS-level encryption or implement `Encrypted` middleware
2. **No ACLs**: Simple permissions only (Unix mode bits)
3. **TOCTOU**: Check-then-act patterns may race (like all filesystems)

---

*For implementation details, see [Architecture Decision Records](../architecture/adrs.md).*
