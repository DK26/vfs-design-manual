# Implementation Plan

This plan describes a phased rollout of the AnyFS ecosystem:

- `anyfs-backend`: Core trait (`VfsBackend`, `Layer`) + types
- `anyfs`: Built-in backends + middleware (feature-gated)
- `anyfs-container`: `FileStorage<M>` ergonomic wrapper

---

## Implementation Guidelines

These guidelines apply to ALL implementation work. Derived from analysis of issues in similar projects (`vfs`, `agentfs`).

### 1. No Panic Policy

**NEVER panic in library code.** Always return `Result<T, VfsError>`.

- Audit all `.unwrap()` and `.expect()` calls - replace with `?` or proper error handling
- Use `ok_or_else(|| VfsError::...)` instead of `.unwrap()`
- Edge cases must return errors, not panic
- Test in constrained environments (WASM) to catch hidden panics

```rust
// BAD
let entry = self.entries.get(&path).unwrap();

// GOOD
let entry = self.entries.get(&path)
    .ok_or_else(|| VfsError::NotFound { path: path.to_path_buf() })?;
```

### 2. Thread Safety Requirements

All backends must be safe for concurrent access:

- `MemoryBackend`: Use `Arc<RwLock<...>>` for internal state
- `SqliteBackend`: Use WAL mode, handle `SQLITE_BUSY`
- `VRootFsBackend`: File operations are inherently concurrent-safe

**Required:** Concurrent stress tests in conformance suite.

### 3. Consistent Path Handling

Normalize paths in ONE place (FilesContainer or dedicated normalizer):

- Always absolute paths internally
- Always `/` separator (even on Windows)
- Normalize `..` and `.` components
- Handle edge cases: `//`, trailing `/`, empty string

### 4. Error Type Design

`VfsError` must be:
- Easy to pattern match
- Include context (path, operation)
- Derive `thiserror` for good messages

```rust
#[derive(Debug, thiserror::Error)]
pub enum VfsError {
    #[error("not found: {path}")]
    NotFound { path: PathBuf },

    #[error("already exists: {path}")]
    AlreadyExists { path: PathBuf },

    #[error("quota exceeded: limit {limit}, attempted {attempted}")]
    QuotaExceeded { limit: u64, attempted: u64 },

    #[error("feature not enabled: {feature}")]
    FeatureNotEnabled { feature: &'static str },

    #[error("permission denied: {path} ({operation})")]
    PermissionDenied { path: PathBuf, operation: &'static str },

    // ... etc
}
```

### 5. Documentation Requirements

Every backend and middleware must document:
- Thread safety guarantees
- Performance characteristics
- Which operations are O(1) vs O(n)
- Any platform-specific behavior

---

## Phase 1: `anyfs-backend` (core contract)

**Goal:** Define the stable backend interface and composition traits.

- Define `VfsBackend` trait (29 methods)
  - 25 `std::fs`-aligned path-based methods (required)
  - 4 inode methods with sensible defaults (override for hardlinks/FUSE):
    - `path_to_inode(path)` - default: hash path
    - `inode_to_path(inode)` - default: NotSupported
    - `lookup(parent_inode, name)` - default: path-based fallback
    - `metadata_by_inode(inode)` - default: path-based fallback
  - Include `const NEEDS_PATH_RESOLUTION: bool = true` (opt-out for real FS backends)
  - Define `ROOT_INODE = 1` constant
- Define `Layer` trait (Tower-style middleware composition)
- Define `VfsBackendExt` trait (extension methods)
- Define core types (`Metadata`, `Permissions`, `FileType`, `DirEntry`, `StatFs`)
- Define `VfsError` with contextual variants (see guidelines above)

**Exit criteria:** `anyfs-backend` stands alone with minimal dependencies (`thiserror`).

---

## Phase 2: `anyfs` (backends + middleware)

**Goal:** Provide reference backends and core middleware.

### Path Resolution Utility

- `resolve_path(backend, path, follow_symlinks)` - works on any `VfsBackend`
  - Walks path component by component using `metadata()` and `read_link()`
  - Handles `..` correctly (requires knowing what each component resolves to)
  - Detects circular symlinks (max depth or visited set)
  - Returns canonical resolved path
- Applied automatically by `FileStorage` when backend's `NEEDS_PATH_RESOLUTION == true`

### Backends (feature-gated)

- `memory` (default): `MemoryBackend`
  - `NEEDS_PATH_RESOLUTION = true` (uses path resolution utility)
  - Inode source: internal node IDs (incrementing counter)
  - `set_follow_symlinks(bool)` - control symlink resolution
- `sqlite` (optional): `SqliteBackend`
  - `NEEDS_PATH_RESOLUTION = true` (uses path resolution utility)
  - Inode source: SQLite row IDs (`INTEGER PRIMARY KEY`)
  - `set_follow_symlinks(bool)` - control symlink resolution
- `vrootfs` (optional): `VRootFsBackend` using `strict-path` for containment
  - `NEEDS_PATH_RESOLUTION = false` (OS handles resolution)
  - Inode source: OS inode numbers (`std::fs::Metadata::ino()`)
  - `strict-path` prevents symlink escapes

### Middleware

- `Quota<B>` + `QuotaLayer` - Resource limits
- `Restrictions<B>` + `RestrictionsLayer` - Opt-in restrictions (`.deny_symlinks()`, `.deny_hard_links()`, etc.)
- `PathFilter<B>` + `PathFilterLayer` - Path-based access control
- `ReadOnly<B>` + `ReadOnlyLayer` - Block writes
- `RateLimit<B>` + `RateLimitLayer` - Operation throttling
- `Tracing<B>` + `TracingLayer` - Instrumentation
- `DryRun<B>` + `DryRunLayer` - Log without executing
- `Cache<B>` + `CacheLayer` - LRU read cache
- `Overlay<B1,B2>` + `OverlayLayer` - Union filesystem

**Exit criteria:** Each backend implements `VfsBackend` and passes conformance suite. Each middleware wraps any `VfsBackend`.

---

## Phase 3: `anyfs-container` (ergonomics)

**Goal:** Provide user-facing ergonomic wrapper.

- `FileStorage<M>` - Thin wrapper with `std::fs`-aligned API
  - Type-erased backend (`Box<dyn VfsBackend>`) for clean API
  - Optional marker type `M` for compile-time container differentiation
- Accepts `impl AsRef<Path>` for convenience
- Delegates all operations to wrapped backend

**Note:** `FileStorage` contains NO policy logic. Policy is handled by middleware.

**Exit criteria:** Applications can use `FileStorage` as drop-in for `std::fs` patterns.

---

## Phase 4: Conformance test suite

**Goal:** Prevent backend divergence and validate middleware behavior.

### Backend conformance tests

#### Basic Operations
- `read`/`write`/`append` semantics
- Directory operations (`create_dir*`, `read_dir`, `remove_dir*`)
- `rename` and `copy` semantics
- Link behavior (`symlink`, `read_link`, `hard_link`)
- Streaming I/O (`open_read`, `open_write`)
- `truncate`, `sync`, `fsync`, `statfs`

#### Path Resolution Tests (virtual backends only)
- `/foo/../bar` resolves correctly when `foo` is a regular directory
- `/foo/../bar` resolves correctly when `foo` is a symlink (follows symlink, then `..`)
- Symlink chains resolve correctly (A → B → C → target)
- Circular symlink detection (A → B → A returns error, not infinite loop)
- Max symlink depth enforced (prevent deep chains)
- `set_follow_symlinks(true)`: reading symlink follows to target
- `set_follow_symlinks(false)`: reading symlink returns symlink metadata, not target

#### Path Edge Cases (learned from `vfs` issues)
- `//double//slashes//` normalizes correctly
- Note: `/foo/../bar` requires resolution (see above), not simple normalization
- Trailing slashes handled consistently
- Empty path returns error (not panic)
- Root path `/` works correctly
- Very long paths (near OS limits)
- Unicode paths
- Paths with spaces and special characters

#### Thread Safety Tests (learned from `vfs` #72, #47)
- Concurrent `read` from multiple threads
- Concurrent `write` to different files
- Concurrent `create_dir_all` to same path (must not race)
- Concurrent `read_dir` while modifying directory
- Stress test: 100 threads, 1000 operations each

#### Error Handling Tests (learned from `vfs` #8, #23)
- Missing file returns `NotFound`, not panic
- Missing parent directory returns error, not panic
- Invalid UTF-8 in path returns error, not panic
- All error variants are matchable

#### Platform Tests
- Windows path separators (`\` vs `/`)
- Case sensitivity differences
- Symlink behavior differences

### Middleware tests

- `Quota`: Limit enforcement, usage tracking, streaming writes
- `Restrictions`: Operation blocking via `.deny_*()` methods, error messages
- `PathFilter`: Glob pattern matching, deny-by-default
- `RateLimit`: Throttling behavior, burst handling
- `ReadOnly`: All write operations blocked
- `Tracing`: Operations logged correctly
- Middleware composition order (inner to outer)
- Middleware with streaming I/O (wrappers work correctly)

### No-Panic Tests

```rust
#[test]
fn no_panic_on_missing_file() {
    let backend = create_backend();
    let result = backend.read("/nonexistent");
    assert!(matches!(result, Err(VfsError::NotFound { .. })));
}

#[test]
fn no_panic_on_invalid_operation() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"data").unwrap();
    // Try to read directory on a file
    let result = backend.read_dir("/file.txt");
    assert!(matches!(result, Err(VfsError::NotADirectory { .. })));
}
```

### WASM Compatibility Tests (learned from `vfs` #68)

```rust
#[cfg(target_arch = "wasm32")]
#[wasm_bindgen_test]
fn memory_backend_works_in_wasm() {
    let mut backend = MemoryBackend::new();
    backend.write("/test.txt", b"hello").unwrap();
    // Should not panic
}
```

**Exit criteria:** All backends pass same suite; middleware tests are backend-agnostic; zero panics in any test.

---

## Phase 5: Documentation + examples

- Keep `AGENTS.md` and `src/architecture/design-overview.md` authoritative
- Provide example per backend
- Provide backend implementer guide
- Provide middleware implementer guide
- Document performance characteristics per backend
- Document thread safety guarantees per backend
- Document platform-specific behavior

---

## Phase 6: CI/CD Pipeline

**Goal:** Ensure quality across platforms and prevent regressions.

### Cross-Platform Testing

```yaml
# .github/workflows/ci.yml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    rust: [stable, beta]
```

Required CI checks:
- `cargo test` on all platforms
- `cargo clippy -- -D warnings`
- `cargo fmt --check`
- `cargo doc --no-deps`
- WASM build test: `cargo build --target wasm32-unknown-unknown`

### Additional CI Jobs

- **Miri** (undefined behavior detection): `cargo +nightly miri test`
- **Address Sanitizer**: Detect memory issues
- **Thread Sanitizer**: Detect data races
- **Coverage**: Minimum 80% line coverage

### Release Checklist

- [ ] All CI checks pass
- [ ] No new `clippy` warnings
- [ ] CHANGELOG updated
- [ ] Version bumped appropriately
- [ ] Documentation builds without warnings

---

## Future work (post-MVP)

- Async API (`AsyncVfsBackend` trait)
- Import/export helpers (host path <-> container)
- Encryption middleware
- Compression middleware
- `no_std` support (learned from `vfs` #38)
- Batch operations for performance (learned from `agentfs` #130)

### `VfsBackendPosixExt` - Optional POSIX Semantics

Optional trait extension for backends that can support POSIX-like semantics.

**Why optional?** The core `VfsBackend` is path-based (25 methods), covering 90% of use cases. Full POSIX (41+ methods) adds complexity not all backends can support.

**What it adds:**

```rust
pub trait VfsBackendPosixExt: VfsBackend {
    // File handle operations (stateful)
    fn open(&mut self, path: impl AsRef<Path>, flags: OpenFlags) -> Result<Handle, VfsError>;
    fn read_handle(&self, handle: Handle, buf: &mut [u8]) -> Result<usize, VfsError>;
    fn write_handle(&mut self, handle: Handle, data: &[u8]) -> Result<usize, VfsError>;
    fn seek(&mut self, handle: Handle, pos: SeekFrom) -> Result<u64, VfsError>;
    fn close(&mut self, handle: Handle) -> Result<(), VfsError>;

    // File locking
    fn lock(&mut self, handle: Handle, lock: LockType) -> Result<(), VfsError>;
    fn try_lock(&mut self, handle: Handle, lock: LockType) -> Result<bool, VfsError>;
    fn unlock(&mut self, handle: Handle) -> Result<(), VfsError>;

    // Extended attributes
    fn get_xattr(&self, path: impl AsRef<Path>, name: &str) -> Result<Vec<u8>, VfsError>;
    fn set_xattr(&mut self, path: impl AsRef<Path>, name: &str, value: &[u8]) -> Result<(), VfsError>;
    fn remove_xattr(&mut self, path: impl AsRef<Path>, name: &str) -> Result<(), VfsError>;
    fn list_xattr(&self, path: impl AsRef<Path>) -> Result<Vec<String>, VfsError>;
}
```

**Use cases:**
- Applications requiring file handles (seek, partial read/write)
- Concurrent access with file locking
- Storing metadata in extended attributes

### `anyfs-fuse` - Mount as Real Filesystem

Adapter to expose any `VfsBackend` as a FUSE mount point.

```rust
use anyfs::{MemoryBackend, QuotaLayer};
use anyfs_fuse::FuseMount;

// RAM drive with 1GB quota
let backend = MemoryBackend::new()
    .layer(QuotaLayer::new().max_total_size(1024 * 1024 * 1024));

let mount = FuseMount::mount(backend, "/mnt/ramdisk")?;

// Now it's a real mount point:
// $ df -h /mnt/ramdisk
// $ cp large_file.bin /mnt/ramdisk/  # fast!
// $ gcc -o /mnt/ramdisk/build ...    # compile in RAM
```

**Cross-Platform Support:**

| Platform | FUSE Provider | Rust Crate | User Must Install |
|----------|---------------|------------|-------------------|
| Linux | Native kernel | `fuser` | `fuse3` package |
| macOS | macFUSE | `fuser` | [macFUSE](https://osxfuse.github.io/) |
| FreeBSD | Native | `fuser` | (built-in) |
| Windows | WinFsp | `winfsp-rs` | [WinFsp](https://winfsp.dev/) |

`anyfs-fuse` provides a unified API across platforms:

```rust
impl<B: VfsBackend> FuseMount<B> {
    #[cfg(unix)]
    pub fn mount(backend: B, path: &Path) -> Result<Self, ...> {
        // Uses fuser crate
    }

    #[cfg(windows)]
    pub fn mount(backend: B, path: &Path) -> Result<Self, ...> {
        // Uses winfsp-rs crate
    }
}
```

**Creative Use Cases:**

| Backend Stack | What You Get |
|---------------|--------------|
| `MemoryBackend` | RAM drive |
| `MemoryBackend` + `Quota` | RAM drive with size limit |
| `SqliteBackend` | Single-file portable drive |
| `Overlay<SqliteBackend, MemoryBackend>` | Persistent base + RAM scratch layer |
| `Cache<SqliteBackend>` | SQLite with RAM read cache |
| `Tracing<MemoryBackend>` | RAM drive with full audit log |
| `ReadOnly<SqliteBackend>` | Immutable snapshot mount |
| `Encryption<SqliteBackend>` | Encrypted portable drive |

**Example: AI Agent Sandbox**

```rust
// Sandboxed workspace mounted as real filesystem
let sandbox = FuseMount::mount(
    MemoryBackend::new()
        .layer(PathFilterLayer::new()
            .allow("/**")
            .deny("**/..*"))           // No hidden files
        .layer(RestrictionsLayer::new()
            .deny_symlinks())           // No symlink escapes
        .layer(QuotaLayer::new()
            .max_total_size(100 * 1024 * 1024)),
    "/mnt/agent-workspace"
)?;

// Agent's tools can now use standard filesystem APIs
// All operations are sandboxed, logged, and quota-limited
```

**Architecture:**

```
┌────────────────────────────────────────────────┐
│  /mnt/myfs (FUSE mount point)                  │
├────────────────────────────────────────────────┤
│  anyfs-fuse                                    │
│    - Linux/macOS/BSD: fuser                    │
│    - Windows: winfsp-rs                        │
├────────────────────────────────────────────────┤
│  VfsBackendPosixExt (optional, for locks/xattr)│
├────────────────────────────────────────────────┤
│  Middleware stack (Quota, PathFilter, etc.)    │
├────────────────────────────────────────────────┤
│  VfsBackend (Memory, SQLite, etc.)             │
└────────────────────────────────────────────────┘
```

**Requirements:**
- Backend should implement `VfsBackendPosixExt` for full functionality
- Falls back gracefully for backends without POSIX extension
- Platform-specific FUSE provider must be installed

### `anyfs-vfs-compat` - Interop with `vfs` crate

Adapter crate for bidirectional compatibility with the [`vfs`](https://github.com/manuel-woelker/rust-vfs) crate ecosystem.

**Why not adopt their trait?** The `vfs::FileSystem` trait is too limited:
- No symlinks, hard links, or permissions
- No `sync`/`fsync` for durability
- No `truncate`, `statfs`, or `read_range`
- No middleware composition pattern

**Our trait is a superset** - we support everything they do, plus more.

**Adapters:**

```rust
// Wrap a vfs::FileSystem to use as AnyFS backend
// Missing features (symlinks, permissions, etc.) return VfsError::NotSupported
pub struct VfsCompat<F: vfs::FileSystem>(F);
impl<F: vfs::FileSystem> VfsBackend for VfsCompat<F> { ... }

// Wrap an AnyFS backend to use as vfs::FileSystem
// Only exposes the subset that vfs supports
pub struct AnyFsCompat<B: VfsBackend>(B);
impl<B: VfsBackend> vfs::FileSystem for AnyFsCompat<B> { ... }
```

**Use cases:**
- Migrate from `vfs` to AnyFS incrementally
- Use existing `vfs` backends (EmbeddedFS) in AnyFS
- Use AnyFS backends in projects that depend on `vfs`

### Cloud Storage & Remote Access

The `VfsBackend` abstraction enables building cloud storage services with multiple access patterns.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                          YOUR SERVER                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Quota<Tracing<SqliteBackend>>  (or any backend stack)        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│         ▲              ▲              ▲              ▲              │
│         │              │              │              │              │
│    ┌────┴────┐   ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐       │
│    │ S3 API  │   │ gRPC/REST │  │    NFS    │  │  WebDAV   │       │
│    │ adapter │   │  adapter  │  │   server  │  │  server   │       │
│    └────┬────┘   └─────┬─────┘  └─────┬─────┘  └─────┬─────┘       │
└─────────┼──────────────┼──────────────┼──────────────┼─────────────┘
          │              │              │              │
          ▼              ▼              ▼              ▼
    AWS SDK/CLI    Your SDK/app    mount /cloud   mount /webdav
```

**Future crates for remote access:**

| Crate | Purpose |
|-------|---------|
| `anyfs-s3-server` | Expose VfsBackend as S3-compatible API |
| `anyfs-sftp-server` | SFTP server - shell-like file access |
| `anyfs-ssh-shell` | SSH server with sandboxed home directories |
| `anyfs-remote` | `RemoteBackend` client for remote VfsBackend access |
| `anyfs-grpc` | gRPC protocol adapter for VfsBackend |
| `anyfs-webdav` | WebDAV server adapter |
| `anyfs-nfs` | NFS server adapter |

#### `anyfs-s3-server` - S3-Compatible Object Storage

Expose any `VfsBackend` as an S3-compatible API. Users access your storage with standard AWS SDKs.

```rust
use anyfs::{SqliteBackend, Quota, Tracing};
use anyfs_s3_server::S3Server;

// Your storage backend with quotas and audit logging
let backend = Quota::new(Tracing::new(SqliteBackend::open("storage.db")?))
    .with_max_total_size(100 * 1024 * 1024 * 1024);  // 100GB

S3Server::new(backend)
    .with_auth(auth_provider)       // Your auth implementation
    .with_bucket("user-files")      // Virtual bucket name
    .bind("0.0.0.0:9000")
    .run()
    .await?;
```

**Client usage (standard AWS CLI/SDK):**

```bash
# Upload a file
aws s3 cp document.pdf s3://user-files/ --endpoint-url http://yourserver:9000

# List files
aws s3 ls s3://user-files/ --endpoint-url http://yourserver:9000

# Download a file
aws s3 cp s3://user-files/document.pdf ./local.pdf --endpoint-url http://yourserver:9000
```

#### `anyfs-remote` - Remote Backend Client

A `VfsBackend` implementation that connects to a remote server. Works with `FileStorage` or `anyfs-fuse`.

```rust
use anyfs_remote::RemoteBackend;
use anyfs_container::FileStorage;

// Connect to your cloud service
let remote = RemoteBackend::connect("https://api.yourservice.com")
    .with_auth(api_key)
    .await?;

// Use like any other backend
let mut fs = FileStorage::new(remote);
fs.write("/documents/report.pdf", data)?;
```

**Combined with FUSE for transparent mount:**

```rust
use anyfs_remote::RemoteBackend;
use anyfs_fuse::FuseMount;

// Mount remote storage as local directory
let remote = RemoteBackend::connect("https://yourserver.com")?;
FuseMount::mount(remote, "/mnt/cloud")?;

// Now use standard filesystem tools:
// $ cp file.txt /mnt/cloud/
// $ ls /mnt/cloud/
// $ cat /mnt/cloud/file.txt
```

#### `anyfs-grpc` - gRPC Protocol

Efficient binary protocol for VfsBackend remote access.

**Server side:**

```rust
use anyfs_grpc::GrpcServer;

let backend = SqliteBackend::open("storage.db")?;
GrpcServer::new(backend)
    .bind("[::1]:50051")
    .serve()
    .await?;
```

**Client side:**

```rust
use anyfs_grpc::GrpcBackend;

let backend = GrpcBackend::connect("http://[::1]:50051").await?;
let mut fs = FileStorage::new(backend);
```

#### Multi-Tenant Cloud Storage Example

```rust
use anyfs::{SqliteBackend, Quota, PathFilter, Tracing};
use anyfs_s3_server::S3Server;

// Per-tenant backend factory
fn create_tenant_storage(tenant_id: &str, quota_bytes: u64) -> impl VfsBackend {
    let db_path = format!("/data/tenants/{}.db", tenant_id);

    Quota::new(
        PathFilter::new(
            Tracing::new(SqliteBackend::open(&db_path).unwrap())
                .with_target(&format!("tenant.{}", tenant_id))
        )
        .allow("/**")
        .deny("../**")  // No path traversal
    )
    .with_max_total_size(quota_bytes)
}

// Tenant-aware S3 server
S3Server::new_multi_tenant(|request| {
    let tenant_id = extract_tenant(request)?;
    let quota = get_tenant_quota(tenant_id)?;
    Ok(create_tenant_storage(tenant_id, quota))
})
.bind("0.0.0.0:9000")
.run()
.await?;
```

#### `anyfs-sftp-server` - SFTP Access with Shell Commands

Expose VfsBackend as an SFTP server. Users connect with standard SSH/SFTP clients and navigate with familiar shell commands.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      YOUR SERVER                                │
│                                                                 │
│  ┌───────────────┐    ┌───────────────────────────────────────┐ │
│  │ SFTP Server   │───▶│ User's isolated FileStorage           │ │
│  │ (anyfs-sftp)  │    │   └─▶ Quota<SqliteBackend>            │ │
│  └───────────────┘    │       └─▶ /data/users/alice.db        │ │
│         ▲             └───────────────────────────────────────┘ │
└─────────┼───────────────────────────────────────────────────────┘
          │
          │ sftp://
          │
    ┌─────┴─────┐
    │  Remote   │  $ cd /documents
    │  User     │  $ ls
    │  (shell)  │  $ put file.txt
    └───────────┘
```

**Server implementation:**

```rust
use anyfs::{SqliteBackend, Quota, Tracing};
use anyfs_sftp_server::SftpServer;

// Per-user isolated backend factory
fn get_user_storage(username: &str) -> impl VfsBackend {
    let db_path = format!("/data/users/{}.db", username);

    Quota::new(
        Tracing::new(SqliteBackend::open(&db_path).unwrap())
            .with_target(&format!("user.{}", username))
    )
    .with_max_total_size(10 * 1024 * 1024 * 1024)  // 10GB per user
}

SftpServer::new(get_user_storage)
    .with_host_key("/etc/ssh/host_key")
    .bind("0.0.0.0:22")
    .run()
    .await?;
```

**User experience (standard SFTP client):**

```bash
$ sftp alice@yourserver.com
Connected to yourserver.com.
sftp> pwd
/
sftp> ls
documents/  photos/  backup/
sftp> cd documents
sftp> ls
report.pdf  notes.txt
sftp> put local_file.txt
Uploading local_file.txt to /documents/local_file.txt
sftp> get notes.txt
Downloading /documents/notes.txt
sftp> mkdir projects
sftp> rm old_file.txt
```

All operations happen on the user's isolated SQLite database on your server.

#### `anyfs-ssh-shell` - Full Shell Access with Sandboxed Home

Give users a real SSH shell where their home directory is backed by VfsBackend.

**Server implementation:**

```rust
use anyfs::{SqliteBackend, Quota};
use anyfs_fuse::FuseMount;
use anyfs_ssh_shell::SshShellServer;

// On user login, mount their isolated storage as $HOME
fn on_user_login(username: &str) -> Result<(), Error> {
    let db_path = format!("/data/users/{}.db", username);
    let backend = Quota::new(SqliteBackend::open(&db_path)?)
        .with_max_total_size(10 * 1024 * 1024 * 1024);

    let mount_point = format!("/home/{}", username);
    FuseMount::mount(backend, &mount_point)?;
    Ok(())
}

SshShellServer::new()
    .on_login(on_user_login)
    .bind("0.0.0.0:22")
    .run()
    .await?;
```

**User experience (full shell):**

```bash
$ ssh alice@yourserver.com
Welcome to YourServer!

alice@server:~$ pwd
/home/alice
alice@server:~$ ls -la
total 3
drwxr-xr-x  4 alice alice 4096 Dec 25 10:00 .
drwxr-xr-x  2 alice alice 4096 Dec 25 10:00 documents
drwxr-xr-x  2 alice alice 4096 Dec 25 10:00 photos

alice@server:~$ cat documents/notes.txt
Hello world!

alice@server:~$ echo "new content" > documents/new_file.txt

alice@server:~$ du -sh .
150M    .

# Everything they do is actually stored in /data/users/alice.db on the server!
# They can use vim, gcc, python - all working on their isolated VfsBackend
```

#### Isolated Shell Hosting Use Cases

| Use Case | Backend Stack | What Users Get |
|----------|---------------|----------------|
| Shared hosting | `Quota<SqliteBackend>` | Shell + isolated home in SQLite |
| Dev containers | `Overlay<BaseImage, MemoryBackend>` | Shared base + ephemeral scratch |
| Coding education | `Quota<MemoryBackend>` | Temporary sandboxed environment |
| CI/CD runners | `Tracing<MemoryBackend>` | Audited ephemeral workspace |
| Secure file drop | `PathFilter<SqliteBackend>` | Write-only inbox directory |

#### Access Pattern Summary

| Access Method | Crate | Client Requirement | Best For |
|---------------|-------|-------------------|----------|
| S3 API | `anyfs-s3-server` | AWS SDK (any language) | Object storage, web apps |
| SFTP | `anyfs-sftp-server` | Any SFTP client | Shell-like file access |
| SSH Shell | `anyfs-ssh-shell` + `anyfs-fuse` | SSH client | Full shell with sandboxed home |
| gRPC | `anyfs-grpc` | Generated client | High-performance apps |
| REST | Custom adapter | HTTP client | Simple integrations |
| FUSE mount | `anyfs-fuse` + `anyfs-remote` | FUSE installed | Transparent local access |
| WebDAV | `anyfs-webdav` | WebDAV client/OS | File manager access |
| NFS | `anyfs-nfs` | NFS client | Unix network shares |

---

## Lessons Learned (Reference)

This plan incorporates lessons from issues in similar projects:

| Source | Issue | Lesson Applied |
|--------|-------|----------------|
| vfs #72 | RwLock panic | Thread safety tests |
| vfs #47 | `create_dir_all` race | Concurrent stress tests |
| vfs #8, #23 | Panics instead of errors | No-panic policy |
| vfs #24, #42 | Path inconsistencies | Path edge case tests |
| vfs #33 | Hard to match errors | Ergonomic `VfsError` design |
| vfs #68 | WASM panics | WASM compatibility tests |
| vfs #66 | `'static` confusion | Minimal trait bounds |
| agentfs #130 | Slow file deletion | Performance documentation |
| agentfs #129 | Signal handling | Proper `Drop` implementations |

See [Lessons from Similar Projects](./lessons-learned.md) for full analysis.
