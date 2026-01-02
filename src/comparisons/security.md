# Security Considerations

**Security model, threat analysis, and containment guarantees**

---

## Overview

AnyFS is designed with security as a primary concern. Security policies are enforced via **composable middleware**, not hardcoded in backends or the container wrapper.

---

## Threat Model

### In Scope (Mitigated by Middleware)

| Threat                    | Description                              | Middleware                    |
| ------------------------- | ---------------------------------------- | ----------------------------- |
| **Path traversal**        | Access files outside allowed paths       | `PathFilter`                  |
| **Symlink attacks**       | Use symlinks to bypass controls          | Backend-dependent (see below) |
| **Resource exhaustion**   | Fill storage or create excessive files   | `Quota`                       |
| **Runaway processes**     | Excessive operations consuming resources | `RateLimit`                   |
| **Unauthorized writes**   | Modifications to read-only data          | `ReadOnly`                    |
| **Sensitive file access** | Access to `.env`, secrets, etc.          | `PathFilter`                  |

### Out of Scope

| Threat                     | Reason                                          |
| -------------------------- | ----------------------------------------------- |
| **Side-channel attacks**   | Requires OS-level mitigations                   |
| **Physical access**        | Disk encryption is application's responsibility |
| **SQLite vulnerabilities** | Upstream dependency; update regularly           |
| **Network attacks**        | AnyFS is local storage, not network-facing      |

---

## Security Architecture

### 1. Middleware-Based Policy

Security policies are composable middleware layers:

```rust
use anyfs::{MemoryBackend, QuotaLayer, PathFilterLayer, RateLimitLayer, TracingLayer};

let secure_backend = MemoryBackend::new()
    .layer(QuotaLayer::builder()              // Limit resources
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(PathFilterLayer::builder()         // Sandbox paths
        .allow("/workspace/**")
        .deny("**/.env")
        .deny("**/secrets/**")
        .build())
    .layer(RateLimitLayer::builder()          // Throttle operations
        .max_ops(1000)
        .per_second()
        .build())
    .layer(TracingLayer::new());              // Audit trail
```

### 2. Path Sandboxing (PathFilter)

`PathFilter` middleware restricts path access using glob patterns:

```rust
PathFilterLayer::builder()
    .allow("/workspace/**")    // Allow workspace access
    .deny("**/.env")           // Block .env files
    .deny("**/secrets/**")     // Block secrets directories
    .deny("**/*.key")          // Block key files
    .build()
    .layer(backend)
```

**Guarantees:**
- First matching rule wins
- No rule = denied (deny by default)
- `read_dir` filters denied entries from results

### 3. Symlink Capability via Trait Bounds

Symlink/hard-link capability is determined by **trait bounds**, not middleware:

```rust
// MemoryBackend implements FsLink → symlinks work
let fs = FileStorage::new(MemoryBackend::new());
fs.symlink("/target", "/link")?;  // ✅ Works

// Custom backend without FsLink → symlinks won't compile
let fs = FileStorage::new(MySimpleBackend::new());
fs.symlink("/target", "/link")?;  // ❌ Compile error
```

**If you don't want symlinks:** Use a backend that doesn't implement `FsLink`.

The `Restrictions` middleware only controls permission operations:

```rust
RestrictionsLayer::builder()
    .deny_permissions()        // Block set_permissions() calls
    .build()
    .layer(backend)
```

**Use cases:**
- Sandboxing untrusted code (block permission changes)
- Read-only-ish environments (block permission mutations)

### 4. Resource Limits (Quota)

`Quota` middleware enforces capacity limits:

```rust
QuotaLayer::builder()
    .max_total_size(100 * 1024 * 1024)  // 100 MB total
    .max_file_size(10 * 1024 * 1024)    // 10 MB per file
    .max_node_count(10_000)             // Max files/dirs
    .max_dir_entries(1_000)             // Max per directory
    .max_path_depth(64)                 // Max nesting
    .build()
    .layer(backend)
```

**Guarantees:**
- Writes rejected when limits exceeded
- Streaming writes tracked via `CountingWriter`

### 5. Rate Limiting (RateLimit)

`RateLimit` middleware throttles operations:

```rust
RateLimitLayer::builder()
    .max_ops(1000)
    .per_second()
    .build()
    .layer(backend)
```

**Guarantees:**
- Operations rejected when limit exceeded
- Protects against runaway processes

### 6. Backend-Level Containment

Different backends achieve containment differently:

| Backend          | Containment Mechanism                                               |
| ---------------- | ------------------------------------------------------------------- |
| `MemoryBackend`  | Isolated in process memory                                          |
| `SqliteBackend`  | Each container is a separate `.db` file                             |
| `IndexedBackend` | SQLite index + isolated blob directory (content-addressable)        |
| `StdFsBackend`   | **None** - full filesystem access (do NOT use with untrusted input) |
| `VRootFsBackend` | Uses `strict-path::VirtualRoot` to contain paths                    |

> ⚠️ **Warning:** `PathFilter` middleware on `StdFsBackend` does NOT provide sandboxing.
> The OS still resolves paths (including symlinks) before `PathFilter` can check them.
> For path containment with real filesystems, use `VRootFsBackend`.

### 7. Why Virtual Backends Are Inherently Safe

For `MemoryBackend` and `SqliteBackend`, the underlying storage is isolated from the host filesystem. There is no OS filesystem to exploit - paths operate entirely within the virtual structure.

**Path resolution is symlink-aware but contained**: FileStorage resolves paths by walking the *virtual* directory structure (using `metadata()` and `read_link()` on the backend), not the OS filesystem:

```
Virtual backend symlink example:
  /foo/bar  where bar → /other/place
  /foo/bar/..  resolves to /other (following the symlink target's parent)

This is correct filesystem semantics - but it happens entirely within
the virtual structure. There is no host filesystem to escape to.
```

This means:
- **No host filesystem access** - symlinks point to paths within the virtual structure only
- **No TOCTOU via OS state** - resolution uses the backend's own data
- **No runtime toggle** - symlink following is part of backend semantics (virtual backends that implement `FsLink` are resolved by `FileStorage`)

For `VRootFsBackend` (real filesystem), `strict-path::VirtualRoot` provides equivalent guarantees by validating and containing all paths before they reach the OS.

### 8. Symlink Security: Virtual vs Real Backends

**The security concern with symlinks is *following* them, not *creating* them.**

Symlinks are just data. Creating `/sandbox/link -> /etc/passwd` is harmless. The danger is when reading `/sandbox/link` follows the symlink and accesses `/etc/passwd`.

| Backend Type     | Symlink Creation     | Symlink Following                                |
| ---------------- | -------------------- | ------------------------------------------------ |
| `MemoryBackend`  | Supported (`FsLink`) | `FileStorage` resolves (non-`SelfResolving`)     |
| `SqliteBackend`  | Supported (`FsLink`) | `FileStorage` resolves (non-`SelfResolving`)     |
| `VRootFsBackend` | Supported (`FsLink`) | **OS controls** - `strict-path` prevents escapes |

#### Virtual Backends (Memory, SQLite)

Virtual backends that implement `FsLink` follow symlinks during `FileStorage` resolution. Symlink capability is determined by trait bounds:

- `MemoryBackend: FsLink` → supports symlinks
- `SqliteBackend: FsLink` → supports symlinks
- Custom backend without `FsLink` → no symlinks (compile-time enforced)

If you need symlink-free behavior, use a backend that does not implement `FsLink`.

**This is the actual security feature** - controlling whether symlinks are even possible via trait bounds.

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

| Concern                 | Virtual Backend                              | VRootFsBackend                             |
| ----------------------- | -------------------------------------------- | ------------------------------------------ |
| Symlink creation        | Supported (`FsLink`)                         | Supported (`FsLink`)                       |
| Symlink following       | `FileStorage` resolves (non-`SelfResolving`) | OS controls (strict-path prevents escapes) |
| Jail escape via symlink | No host FS to escape                         | Prevented by strict-path                   |

---

## Secure Usage Patterns

### AI Agent Sandbox

```rust
use anyfs::{MemoryBackend, QuotaLayer, PathFilterLayer, RateLimitLayer, TracingLayer, FileStorage};

let sandbox = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(50 * 1024 * 1024)
        .max_file_size(5 * 1024 * 1024)
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .deny("**/.env")
        .deny("**/secrets/**")
        .build())
    .layer(RateLimitLayer::builder()
        .max_ops(1000)
        .per_second()
        .build())
    .layer(TracingLayer::new());

let fs = FileStorage::new(sandbox);
// Agent code can only access /workspace, limited resources, audited
// Note: MemoryBackend implements FsLink, so symlinks work if needed
```

### Multi-Tenant Isolation

```rust
use anyfs::{SqliteBackend, Quota, FileStorage};

fn create_tenant_storage(tenant_id: &str, quota_bytes: u64) -> FileStorage<impl Fs> {
    let db_path = format!("tenants/{}.db", tenant_id);
    let backend = QuotaLayer::builder()
        .max_total_size(quota_bytes)
        .build()
        .layer(SqliteBackend::open(&db_path).unwrap());

    FileStorage::new(backend)
}

// Complete isolation: separate database files
```

### Read-Only Browsing

```rust
use anyfs::{SqliteBackend, ReadOnly, FileStorage};

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
- [ ] Use `Restrictions` when you need to disable risky operations
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

## Encryption and Integrity Protection

AnyFS's design enables encryption at multiple levels. Understanding the difference between **container-level** and **file-level** protection is crucial for choosing the right approach.

### Container-Level vs File-Level Protection

| Level               | What's Protected                                     | Integrity                | Implementation        |
| ------------------- | ---------------------------------------------------- | ------------------------ | --------------------- |
| **Container-level** | Entire storage medium (`.db` file, serialized state) | Full structure protected | Encrypted backend     |
| **File-level**      | Individual file contents                             | File contents only       | Encryption middleware |

**Key insight:** File-level encryption alone is NOT sufficient. If an attacker can modify the container structure (directory tree, metadata, file names), they can sabotage integrity even without decrypting file contents.

### Threat Analysis

| Threat                               | File-Level Encryption | Container-Level Encryption  |
| ------------------------------------ | --------------------- | --------------------------- |
| Read file contents                   | Protected             | Protected                   |
| Modify file contents                 | Detected (with AEAD)  | Detected                    |
| Delete files                         | **NOT protected**     | Protected                   |
| Rename/move files                    | **NOT protected**     | Protected                   |
| Corrupt directory structure          | **NOT protected**     | Protected                   |
| Replay old file versions             | **NOT protected**     | Protected (with versioning) |
| Metadata exposure (filenames, sizes) | **NOT protected**     | Protected                   |

**Recommendation:** For sensitive data, prefer container-level encryption. Use file-level encryption when you need selective access (some files encrypted, others not).

### Container-Level Encryption

#### Option 1: SQLCipher Backend

[SQLCipher](https://www.zetetic.net/sqlcipher/) provides transparent AES-256 encryption for SQLite:

```rust
/// SQLite backend with full database encryption via SQLCipher.
pub struct SqliteCipherBackend {
    conn: rusqlite::Connection,  // Built with sqlcipher feature
}

impl SqliteCipherBackend {
    pub fn open(path: &str, password: &str) -> Result<Self, FsError> {
        let conn = Connection::open(path)?;
        // SQLCipher: derive key from password, encrypt everything
        conn.pragma_update(None, "key", password)?;
        Ok(Self { conn })
    }

    pub fn open_with_key(path: &str, key: &[u8; 32]) -> Result<Self, FsError> {
        let conn = Connection::open(path)?;
        // Use raw key instead of password
        conn.pragma_update(None, "key", &format!("x'{}'", hex::encode(key)))?;
        Ok(Self { conn })
    }
}

impl Fs for SqliteCipherBackend { /* same as SqliteBackend */ }
```

**What's protected:**
- All file contents
- All metadata (names, sizes, timestamps, permissions)
- Directory structure
- Inode mappings
- Everything in the `.db` file

**Usage:**
```rust
let backend = SqliteCipherBackend::open("secure.db", "correct-horse-battery-staple")?;
let fs = FileStorage::new(backend);

// If someone gets secure.db without the password, they see random bytes
```

#### Option 2: Encrypted Serialization (MemoryBackend)

For in-memory backends that need persistence:

```rust
impl MemoryBackend {
    /// Serialize entire state to encrypted blob.
    pub fn serialize_encrypted(&self, key: &[u8; 32]) -> Result<Vec<u8>, FsError> {
        let plaintext = bincode::serialize(&self.state)?;
        let nonce = generate_nonce();
        let ciphertext = aes_gcm_encrypt(key, &nonce, &plaintext)?;
        Ok([nonce.as_slice(), &ciphertext].concat())
    }

    /// Deserialize from encrypted blob.
    pub fn deserialize_encrypted(data: &[u8], key: &[u8; 32]) -> Result<Self, FsError> {
        let (nonce, ciphertext) = data.split_at(12);
        let plaintext = aes_gcm_decrypt(key, nonce, ciphertext)?;
        let state = bincode::deserialize(&plaintext)?;
        Ok(Self { state })
    }
}
```

**Use case:** Periodically save encrypted snapshots, load on startup.

### File-Level Encryption (Middleware)

When you need selective encryption or per-file keys:

```rust
/// Middleware that encrypts file contents on write, decrypts on read.
/// Does NOT protect metadata, filenames, or directory structure.
pub struct FileEncryption<B> {
    inner: B,
    key: Secret<[u8; 32]>,
}

impl<B: Fs> FsWrite for FileEncryption<B> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        // Encrypt content with authenticated encryption (AES-GCM)
        let nonce = generate_nonce();
        let ciphertext = aes_gcm_encrypt(&self.key, &nonce, data)?;
        let encrypted = [nonce.as_slice(), &ciphertext].concat();
        self.inner.write(path, &encrypted)
    }
}

impl<B: Fs> FsRead for FileEncryption<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let encrypted = self.inner.read(path)?;
        let (nonce, ciphertext) = encrypted.split_at(12);
        aes_gcm_decrypt(&self.key, nonce, ciphertext)
            .map_err(|_| FsError::IntegrityError { path: path.as_ref().to_path_buf() })
    }
}
```

**Limitations:**
- Filenames visible
- Directory structure visible
- File sizes visible (roughly - ciphertext slightly larger)
- Metadata unprotected

**When to use:**
- Some files need encryption, others don't
- Different files need different keys
- Interop with systems that expect plaintext structure

### Integrity Without Encryption

Sometimes you need tamper detection without hiding contents:

```rust
/// Middleware that adds HMAC to each file for integrity verification.
pub struct IntegrityVerified<B> {
    inner: B,
    key: Secret<[u8; 32]>,
}

impl<B: Fs> FsWrite for IntegrityVerified<B> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        let mac = hmac_sha256(&self.key, data);
        let protected = [data, mac.as_slice()].concat();
        self.inner.write(path, &protected)
    }
}

impl<B: Fs> FsRead for IntegrityVerified<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let protected = self.inner.read(path)?;
        let (data, mac) = protected.split_at(protected.len() - 32);
        if !hmac_verify(&self.key, data, mac) {
            return Err(FsError::IntegrityError { path: path.as_ref().to_path_buf() });
        }
        Ok(data.to_vec())
    }
}
```

### RAM Encryption and Secure Memory

For high-security scenarios where memory dumps are a threat:

#### Threat Levels

| Threat                               | Mitigation              | Library-Level?         |
| ------------------------------------ | ----------------------- | ---------------------- |
| Memory inspection after process exit | `zeroize` on drop       | Yes                    |
| Core dumps                           | Disable via `setrlimit` | Yes (process config)   |
| Swap file exposure                   | `mlock()` to pin pages  | Yes (OS permitting)    |
| Live memory scanning (same user)     | OS process isolation    | No                     |
| Cold boot attack                     | Hardware RAM encryption | No (Intel TME/AMD SME) |
| Hypervisor/DMA attack                | SGX/SEV enclaves        | No (hardware)          |

#### Encrypted Memory Backend (Illustrative Pattern)

> **Note:** `EncryptedMemoryBackend` is an illustrative pattern for users who need encrypted RAM storage. It is not a built-in backend in v1. Users can implement this pattern using the guidance below.

Keep data encrypted even in RAM - decrypt only during active use:

```rust
use zeroize::{Zeroize, ZeroizeOnDrop};
use secrecy::Secret;

/// Memory backend that stores all data encrypted in RAM.
/// Plaintext exists only briefly during read operations.
pub struct EncryptedMemoryBackend {
    /// All nodes stored as encrypted blobs
    nodes: HashMap<PathBuf, EncryptedNode>,
    /// Encryption key - auto-zeroized on drop
    key: Secret<[u8; 32]>,
}

struct EncryptedNode {
    /// Encrypted file content (nonce || ciphertext)
    encrypted_data: Vec<u8>,
    /// Metadata can be encrypted too, or stored in the encrypted blob
    metadata: EncryptedMetadata,
}

impl FsRead for EncryptedMemoryBackend {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let node = self.nodes.get(path.as_ref())
            .ok_or_else(|| FsError::NotFound { path: path.as_ref().to_path_buf() })?;

        // Decrypt - plaintext briefly in RAM
        let plaintext = self.decrypt(&node.encrypted_data)?;

        // Return owned Vec - caller responsible for zeroizing if sensitive
        Ok(plaintext)
    }
}

impl FsWrite for EncryptedMemoryBackend {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        // Encrypt immediately - plaintext never stored
        let encrypted = self.encrypt(data)?;

        self.nodes.insert(path.as_ref().to_path_buf(), EncryptedNode {
            encrypted_data: encrypted,
            metadata: self.encrypt_metadata(...)?,
        });
        Ok(())
    }
}

impl Drop for EncryptedMemoryBackend {
    fn drop(&mut self) {
        // Zeroize all encrypted data (defense in depth)
        for node in self.nodes.values_mut() {
            node.encrypted_data.zeroize();
        }
        // Key is auto-zeroized via Secret<>
    }
}
```

#### Serialization of Encrypted RAM

When persisting an encrypted memory backend:

```rust
impl EncryptedMemoryBackend {
    /// Serialize to disk - data stays encrypted throughout.
    /// RAM encrypted → Serialized encrypted → Disk encrypted
    pub fn save_to_file(&self, path: &Path) -> Result<(), FsError> {
        // Data is already encrypted in self.nodes
        // Serialize the encrypted blobs directly - no decryption needed
        let serialized = bincode::serialize(&self.nodes)?;

        // Optionally add another encryption layer with different key
        // (defense in depth: compromise of runtime key doesn't expose persisted data)
        std::fs::write(path, &serialized)?;
        Ok(())
    }

    /// Load from disk - data stays encrypted throughout.
    /// Disk encrypted → Deserialized encrypted → RAM encrypted
    pub fn load_from_file(path: &Path, key: Secret<[u8; 32]>) -> Result<Self, FsError> {
        let serialized = std::fs::read(path)?;
        let nodes = bincode::deserialize(&serialized)?;

        Ok(Self { nodes, key })
    }
}
```

**Key property:** Plaintext NEVER exists during save/load. Data flows:
```
Write: plaintext → encrypt → RAM (encrypted) → serialize → disk (encrypted)
Read:  disk (encrypted) → deserialize → RAM (encrypted) → decrypt → plaintext
```

#### Secure Allocator Considerations

```rust
// In Cargo.toml - mimalloc secure mode zeros on free
mimalloc = { version = "0.1", features = ["secure"] }

// Note: This prevents USE-AFTER-FREE info leaks, but does NOT:
// - Encrypt RAM contents
// - Prevent live memory scanning
// - Protect against cold boot attacks
```

For true defense against memory scanning, combine:
1. `EncryptedMemoryBackend` (data encrypted at rest in RAM)
2. `zeroize` (immediate cleanup of temporary plaintext)
3. `mlock()` (prevent swapping sensitive pages)
4. Minimize plaintext lifetime (decrypt → use → zeroize immediately)

### Encryption Summary

| Approach                                | Protects Contents | Protects Structure | RAM Security                   | Persistence          |
| --------------------------------------- | ----------------- | ------------------ | ------------------------------ | -------------------- |
| `SqliteCipherBackend`                   | Yes               | Yes                | No (SQLite uses plaintext RAM) | Encrypted `.db` file |
| `FileEncryption<B>` middleware          | Yes               | No                 | Depends on B                   | Depends on B         |
| `EncryptedMemoryBackend` (illustrative) | Yes               | Yes                | Yes (encrypted in RAM)         | Via `save_to_file()` |
| `IntegrityVerified<B>` middleware       | No                | No (files only)    | No                             | Depends on B         |

### Recommended Configurations

#### Sensitive Data Storage
```rust
// Full protection: encrypted container + secure memory practices
let backend = SqliteCipherBackend::open("secure.db", password)?;
let fs = FileStorage::new(backend);
```

#### High-Security RAM Processing (Illustrative)
```rust
// Data never plaintext at rest (RAM or disk)
// Note: EncryptedMemoryBackend is user-implemented (see pattern above)
let backend = EncryptedMemoryBackend::new(derive_key(password));
// ... use fs ...
backend.save_to_file("snapshot.enc")?;  // Persists encrypted
```

#### Selective File Encryption
```rust
// Some files encrypted, structure visible
let backend = FileEncryption::new(SqliteBackend::open("data.db")?)
    .with_key(key);
```

---

## TOCTOU-Proof Tenant Isolation with Virtual Backends

### Why Virtual Backends Eliminate TOCTOU

Traditional path security libraries like `strict-path` work against a **real filesystem**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    REAL FILESYSTEM SECURITY                      │
│                                                                  │
│   Your Process          OS Filesystem         Other Processes   │
│   ┌──────────┐         ┌───────────┐         ┌──────────────┐   │
│   │ Check    │────────▶│ Canonical │◀────────│ Create       │   │
│   │ path     │         │ path      │         │ symlink      │   │
│   └──────────┘         └───────────┘         └──────────────┘   │
│        │                     │                      │           │
│        │    TOCTOU WINDOW    │                      │           │
│        ▼                     ▼                      ▼           │
│   ┌──────────┐         ┌───────────┐         ┌──────────────┐   │
│   │ Use      │────────▶│ DIFFERENT │◀────────│ Modified!    │   │
│   │ path     │         │ path now! │         │              │   │
│   └──────────┘         └───────────┘         └──────────────┘   │
│                                                                  │
│   Problem: OS state can change between check and use             │
└─────────────────────────────────────────────────────────────────┘
```

Virtual backends eliminate this entirely:

```
┌─────────────────────────────────────────────────────────────────┐
│                   VIRTUAL BACKEND SECURITY                       │
│                                                                  │
│   Your Process                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    FileStorage                           │   │
│   │  ┌──────────┐    ┌───────────┐    ┌──────────────────┐  │   │
│   │  │ Resolve  │───▶│ SQLite    │───▶│ Return data      │  │   │
│   │  │ path     │    │ Transaction│   │                  │  │   │
│   │  └──────────┘    └───────────┘    └──────────────────┘  │   │
│   │                        │                                 │   │
│   │              ATOMIC - No external modification possible  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   No OS filesystem. No other processes. No TOCTOU.               │
└─────────────────────────────────────────────────────────────────┘
```

### Security Comparison: strict-path vs Virtual Backend

| Threat                      | strict-path (Real FS)             | Virtual Backend                            |
| --------------------------- | --------------------------------- | ------------------------------------------ |
| Path traversal              | Prevented (canonicalize + verify) | **Impossible** (no host FS to traverse to) |
| Symlink race (TOCTOU)       | Mitigated (canonicalize first)    | **Impossible** (we control all symlinks)   |
| External symlink creation   | Vulnerable window exists          | **Impossible** (single-process ownership)  |
| Windows 8.3 short names     | Partial (only existing files)     | **N/A** (no Windows FS)                    |
| Namespace escapes (/proc)   | Fixed in soft-canonicalize        | **Impossible** (no /proc exists)           |
| Concurrent modification     | OS handles (may race)             | **Atomic** (SQLite transactions)           |
| Tenant A accessing Tenant B | Requires careful path filtering   | **Impossible** (separate .db files)        |

### Encryption: Separation of Concerns

**Design principle:** Backends handle storage, middleware handles policy. Container-level encryption is the exception.

| Security Level              | Implementation                             | Why                                             |
| --------------------------- | ------------------------------------------ | ----------------------------------------------- |
| **Locked (container)**      | `SqliteCipherBackend`                      | Must encrypt entire `.db` file at storage level |
| **Privacy (file contents)** | `FileEncryption<SqliteBackend>` middleware | Content encryption is policy                    |
| **Normal**                  | `SqliteBackend`                            | User applies encryption as needed               |

**Why Locked mode requires a separate backend:**
- SQLCipher encrypts the entire database file transparently
- Connection must be opened with password before ANY query
- Cannot be added as middleware - it's a property of the connection itself
- Everything is encrypted: file contents, filenames, directory structure, timestamps, inodes

### SqliteCipherBackend (Built-in, feature: `sqlite-cipher`)

Full container encryption using [SQLCipher](https://www.zetetic.net/sqlcipher/):

```rust
/// SQLite backend with full AES-256 encryption via SQLCipher.
/// Requires `sqlite-cipher` feature (uses rusqlite with bundled-sqlcipher).
///
/// Without the password, the .db file is indistinguishable from random bytes.
pub struct SqliteCipherBackend {
    conn: Connection,
}

impl SqliteCipherBackend {
    /// Open with password (derives key via PBKDF2).
    pub fn open(path: impl AsRef<Path>, password: &str) -> Result<Self, FsError> {
        let conn = Connection::open(path.as_ref())?;

        // SQLCipher: Set encryption key derived from password
        conn.pragma_update(None, "key", password)?;

        // Verify we can read (wrong password = SQLITE_NOTADB)
        conn.query_row("SELECT count(*) FROM sqlite_master", [], |_| Ok(()))
            .map_err(|_| FsError::InvalidPassword)?;

        Self::init_schema(&conn)?;
        Ok(Self { conn })
    }

    /// Open with raw 256-bit key (no key derivation).
    pub fn open_with_key(path: impl AsRef<Path>, key: &[u8; 32]) -> Result<Self, FsError> {
        let conn = Connection::open(path.as_ref())?;

        // SQLCipher: Set raw key (hex-encoded with x'' prefix)
        let hex_key = format!("x'{}'", hex::encode(key));
        conn.pragma_update(None, "key", &hex_key)?;

        conn.query_row("SELECT count(*) FROM sqlite_master", [], |_| Ok(()))
            .map_err(|_| FsError::InvalidPassword)?;

        Self::init_schema(&conn)?;
        Ok(Self { conn })
    }

    /// Create new encrypted database with password.
    pub fn create(path: impl AsRef<Path>, password: &str) -> Result<Self, FsError> {
        if path.as_ref().exists() {
            return Err(FsError::AlreadyExists { path: path.as_ref().to_path_buf(), operation: "create" });
        }
        Self::open(path, password)
    }

    /// Change the password on an open database.
    pub fn change_password(&self, new_password: &str) -> Result<(), FsError> {
        self.conn.pragma_update(None, "rekey", new_password)?;
        Ok(())
    }

    fn init_schema(conn: &Connection) -> Result<(), FsError> {
        conn.execute_batch(r#"
            -- Node table: directories, files, symlinks
            CREATE TABLE IF NOT EXISTS nodes (
                inode INTEGER PRIMARY KEY,
                parent_inode INTEGER NOT NULL,
                name TEXT NOT NULL,
                node_type INTEGER NOT NULL,  -- 0=file, 1=dir, 2=symlink
                size INTEGER NOT NULL DEFAULT 0,
                mode INTEGER NOT NULL DEFAULT 0o644,
                nlink INTEGER NOT NULL DEFAULT 1,
                uid INTEGER NOT NULL DEFAULT 0,
                gid INTEGER NOT NULL DEFAULT 0,
                created_at INTEGER,
                modified_at INTEGER,
                accessed_at INTEGER,
                symlink_target TEXT,
                UNIQUE(parent_inode, name),
                FOREIGN KEY (parent_inode) REFERENCES nodes(inode)
            );

            -- Content table: file data (separate for efficient large files)
            CREATE TABLE IF NOT EXISTS content (
                inode INTEGER PRIMARY KEY,
                data BLOB NOT NULL,
                FOREIGN KEY (inode) REFERENCES nodes(inode) ON DELETE CASCADE
            );

            -- Index for directory listing performance
            CREATE INDEX IF NOT EXISTS idx_parent ON nodes(parent_inode);

            -- Root directory (inode 1, parent is self)
            INSERT OR IGNORE INTO nodes (inode, parent_inode, name, node_type, mode)
            VALUES (1, 1, '', 1, 0o755);
        "#)?;
        Ok(())
    }
}

// Implements same traits as SqliteBackend - only difference is encrypted storage
impl Fs for SqliteCipherBackend { /* ... */ }
impl FsFull for SqliteCipherBackend { /* ... */ }
impl FsFuse for SqliteCipherBackend { /* ... */ }
```

#### What SQLCipher Encrypts

| Data                           | Encrypted? |
| ------------------------------ | ---------- |
| File contents                  | Yes        |
| Filenames                      | Yes        |
| Directory structure            | Yes        |
| File sizes                     | Yes        |
| Timestamps                     | Yes        |
| Permissions                    | Yes        |
| Inode mappings                 | Yes        |
| SQLite metadata                | Yes        |
| **Everything in the .db file** | **Yes**    |

#### Cargo Configuration

```toml
[dependencies]
# Regular SQLite (no encryption)
rusqlite = { version = "0.31", features = ["bundled"] }

# SQLCipher (full encryption) - mutually exclusive with above
rusqlite = { version = "0.31", features = ["bundled-sqlcipher"] }
```

**Feature flags in anyfs:**
```toml
[features]
default = ["memory"]
sqlite = ["rusqlite/bundled"]
sqlite-cipher = ["rusqlite/bundled-sqlcipher"]  # Replaces sqlite
```

**Note:** `sqlite` and `sqlite-cipher` are mutually exclusive. SQLCipher is a drop-in replacement with the same schema and API.

### Achieving Security Modes with Composition

Users compose backends and middleware to achieve their desired security level:

#### Locked Mode (Full Container Encryption)

```rust
// Everything encrypted - password required to access anything
let backend = SqliteCipherBackend::open("tenant.db", "correct-horse-battery-staple")?;
let fs = FileStorage::new(backend);

// Without password: .db file is random bytes
// With password: full access to everything
```

#### Privacy Mode (Contents Encrypted, Metadata Visible)

```rust
// File contents encrypted, metadata (names, sizes, structure) visible
let backend = FileEncryption::new(
    SqliteBackend::open("tenant.db")?
)
.with_key(content_key);

let fs = FileStorage::new(backend);

// Host can: list files, see sizes, run statistics
// Host cannot: read file contents
```

#### Normal Mode (No Encryption)

```rust
// No encryption - user encrypts sensitive files themselves
let backend = SqliteBackend::open("tenant.db")?;
let fs = FileStorage::new(backend);

// User applies per-file encryption as needed
```

#### Mode Comparison

| Aspect              | Locked                     | Privacy                         | Normal          |
| ------------------- | -------------------------- | ------------------------------- | --------------- |
| Implementation      | `SqliteCipherBackend`      | `FileEncryption<SqliteBackend>` | `SqliteBackend` |
| File contents       | Encrypted (SQLCipher)      | Encrypted (AES-GCM)             | Plaintext       |
| Filenames           | Encrypted                  | Visible                         | Visible         |
| Directory structure | Encrypted                  | Visible                         | Visible         |
| File sizes          | Encrypted                  | Visible                         | Visible         |
| Timestamps          | Encrypted                  | Visible                         | Visible         |
| Host can analyze    | Nothing                    | Metadata only                   | Everything      |
| Performance         | Slowest (~10-15% overhead) | Medium                          | Fastest         |
| Feature flag        | `sqlite-cipher`            | `sqlite` + middleware           | `sqlite`        |

#### Why This Is TOCTOU-Proof

1. **No external filesystem** - Paths exist only in our SQLite tables
2. **Atomic transactions** - Path resolution + data access in single transaction
3. **Single-process ownership** - No other process can modify the .db during operation
4. **We control symlinks** - Symlinks are just rows in `nodes` table, we decide when to follow
5. **No OS involvement** - OS never resolves our virtual paths

```rust
// This is TOCTOU-proof:
impl SecureSqliteBackend {
    fn resolve_and_read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        // Single transaction wraps everything
        let tx = self.conn.transaction()?;

        // 1. Resolve path (following symlinks in OUR table)
        let inode = self.resolve_path_internal(&tx, path)?;

        // 2. Read content
        // No TOCTOU - same transaction, same snapshot
        let data = tx.query_row(
            "SELECT data FROM content WHERE inode = ?",
            [inode],
            |row| row.get(0)
        )?;

        // Transaction ensures atomicity
        Ok(data)
    }
}
```

#### Multi-Tenant Isolation

```rust
/// Each tenant gets their own .db file - complete physical isolation
fn create_tenant_storage(tenant_id: &str, encrypted: bool) -> impl Fs {
    let path = format!("tenants/{}.db", tenant_id);

    if encrypted {
        let password = get_tenant_password(tenant_id);
        SqliteCipherBackend::open(&path, &password).unwrap()
    } else {
        SqliteBackend::open(&path).unwrap()
    }
}

// Tenant A literally cannot access Tenant B's data:
// - Different .db files
// - Different passwords (if encrypted)
// - No shared state whatsoever
// - No path filtering bugs possible - there's nothing to filter
```

**Comparison with strict-path approach:**

| Approach                         | Tenant Isolation                        |
| -------------------------------- | --------------------------------------- |
| Shared filesystem + strict-path  | Logical isolation (paths filtered)      |
| Shared filesystem + PathFilter   | Logical isolation (middleware enforced) |
| **Separate .db file per tenant** | **Physical isolation (separate files)** |

Physical isolation is strictly stronger - there's no bug in path filtering that could leak data because **there's no shared data to leak**.

#### Host Analysis with Privacy Mode

When using `FileEncryption<SqliteBackend>` (Privacy mode), the host can query metadata directly from SQLite:

```rust
// Host can analyze metadata without the content encryption key
fn get_tenant_statistics(tenant_db: &str) -> TenantStats {
    // Connect directly to SQLite (no content key needed)
    let conn = Connection::open(tenant_db)?;

    let (file_count, dir_count, total_size) = conn.query_row(
        "SELECT
            COUNT(*) FILTER (WHERE node_type = 0),
            COUNT(*) FILTER (WHERE node_type = 1),
            SUM(size)
         FROM nodes",
        [],
        |row| Ok((row.get(0)?, row.get(1)?, row.get(2)?))
    )?;

    TenantStats { file_count, dir_count, total_size }
}

// List all files (names visible, contents encrypted)
fn list_tenant_files(tenant_db: &str) -> Vec<FileInfo> {
    let conn = Connection::open(tenant_db)?;
    conn.prepare("SELECT name, size, modified_at FROM nodes WHERE node_type = 0")?
        .query_map([], |row| Ok(FileInfo { ... }))?
        .collect()
}
```

#### Replacing strict-path Usage

For projects currently using strict-path for tenant isolation:

**Before (strict-path):**
```rust
use strict_path::VirtualRoot;

fn handle_tenant_request(tenant_id: &str, requested_path: &str) -> Result<Vec<u8>> {
    // Shared filesystem, path containment via strict-path
    let root = VirtualRoot::new(format!("/data/tenants/{}", tenant_id))?;
    let safe_path = root.resolve(requested_path)?;  // TOCTOU window here
    std::fs::read(safe_path)  // Another process could have modified
}
```

**After (SqliteCipherBackend):**
```rust
use anyfs::SqliteCipherBackend;

fn handle_tenant_request(tenant_id: &str, requested_path: &str) -> Result<Vec<u8>> {
    // Separate encrypted database per tenant - no path containment needed
    let backend = get_tenant_backend(tenant_id);  // Cached connection
    backend.read(requested_path)  // Atomic, TOCTOU-proof
}
```

| Aspect                | strict-path                    | Virtual Backend                                |
| --------------------- | ------------------------------ | ---------------------------------------------- |
| Isolation model       | Logical (path filtering)       | Physical (separate files)                      |
| TOCTOU                | Mitigated                      | **Eliminated**                                 |
| External interference | Possible                       | **Impossible**                                 |
| Symlink attacks       | Resolved at check time         | **We control all symlinks**                    |
| Cross-tenant leakage  | Bug in filtering could leak    | **No shared data exists**                      |
| Performance           | Real FS I/O + canonicalization | SQLite (often faster for small files)          |
| Encryption            | Separate concern               | Built-in (`SqliteCipherBackend`) or middleware |

---

## Known Limitations

1. **No ACLs**: Simple permissions only (Unix mode bits)
2. **Side channels**: Timing attacks, cache attacks require OS/hardware mitigations
3. **SQLite file access**: Host OS can still access the `.db` file (use Locked mode for encryption)

---

*For implementation details, see [Architecture Decision Records](../architecture/adrs.md).*
