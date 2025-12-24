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
| **Symlink attacks** | Use symlinks to bypass controls | `FeatureGuard` (disabled by default) |
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
use anyfs::{MemoryBackend, Quota, PathFilter, FeatureGuard, RateLimit, Tracing};

let secure_backend = Tracing::new(           // Audit trail
    RateLimit::new(                          // Throttle operations
        PathFilter::new(                     // Sandbox paths
            FeatureGuard::new(               // Block dangerous features
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

### 3. Feature Gating (FeatureGuard)

Dangerous features are disabled by default:

```rust
FeatureGuard::new(backend)
    // Symlinks disabled by default
    // Hard links disabled by default
    // Permissions disabled by default
    .with_symlinks()           // Opt-in if needed
    .with_max_symlink_resolution(40)
```

**Guarantees:**
- Symlinks, hard links, permission changes blocked unless enabled
- Symlink resolution depth is bounded

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

---

## Secure Usage Patterns

### AI Agent Sandbox

```rust
use anyfs::{MemoryBackend, Quota, PathFilter, FeatureGuard, RateLimit, Tracing};
use anyfs_container::FilesContainer;

let sandbox = Tracing::new(
    RateLimit::new(
        PathFilter::new(
            FeatureGuard::new(
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

let mut fs = FilesContainer::new(sandbox);
// Agent code can only access /workspace, limited resources, audited
```

### Multi-Tenant Isolation

```rust
use anyfs::{SqliteBackend, Quota};
use anyfs_container::FilesContainer;

fn create_tenant_storage(tenant_id: &str, quota_bytes: u64) -> FilesContainer<...> {
    let db_path = format!("tenants/{}.db", tenant_id);
    let backend = Quota::new(SqliteBackend::open(&db_path).unwrap())
        .with_max_total_size(quota_bytes);

    FilesContainer::new(backend)
}

// Complete isolation: separate database files
```

### Read-Only Browsing

```rust
use anyfs::{SqliteBackend, ReadOnly};
use anyfs_container::FilesContainer;

let readonly_fs = FilesContainer::new(
    ReadOnly::new(SqliteBackend::open("archive.db")?)
);

// All write operations return VfsError::ReadOnly
```

---

## Security Checklist

### For Application Developers

- [ ] Use `PathFilter` to sandbox untrusted code
- [ ] Use `Quota` to prevent resource exhaustion
- [ ] Use `FeatureGuard` (default) to disable dangerous features
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
