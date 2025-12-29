# Cross-Platform Virtual Drive Mounting

**Mounting AnyFS backends as real filesystem mount points**

---

## Overview

AnyFS backends can be mounted as real filesystem drives that any application can access. This requires platform-specific adapters because each OS uses different userspace filesystem technologies.

---

## Platform Technologies

| Platform | Technology | Rust Crate | User Installation |
|----------|------------|------------|-------------------|
| Linux | FUSE | `fuser` | Usually pre-installed |
| macOS | macFUSE | `fuser` | [macFUSE](https://osxfuse.github.io/) |
| Windows | WinFsp | `winfsp` | [WinFsp](https://winfsp.dev/) |
| Windows | Dokan | `dokan` | [Dokan](https://dokan-dev.github.io/) |

**Key insight:** Linux and macOS both use FUSE (via `fuser` crate), but Windows requires a completely different API (WinFsp or Dokan).

---

## Architecture

### Unified Mount Trait

```rust
/// Platform-agnostic mount handle.
/// Drop to unmount.
pub struct MountHandle {
    inner: Box<dyn MountHandleInner>,
}

impl MountHandle {
    /// Mount a backend at the specified path.
    ///
    /// Platform requirements:
    /// - Linux: FUSE (usually available)
    /// - macOS: macFUSE must be installed
    /// - Windows: WinFsp or Dokan must be installed
    pub fn mount<B: FsFuse>(backend: B, path: impl AsRef<Path>) -> Result<Self, MountError> {
        #[cfg(unix)]
        return fuse_mount(backend, path);

        #[cfg(windows)]
        return winfsp_mount(backend, path);

        #[cfg(not(any(unix, windows)))]
        return Err(MountError::PlatformNotSupported);
    }

    /// Check if mounting is available on this platform.
    pub fn is_available() -> bool {
        #[cfg(target_os = "linux")]
        return check_fuse_available();

        #[cfg(target_os = "macos")]
        return check_macfuse_available();

        #[cfg(windows)]
        return check_winfsp_available() || check_dokan_available();

        #[cfg(not(any(unix, windows)))]
        return false;
    }

    /// Unmount the filesystem.
    pub fn unmount(self) -> Result<(), MountError> {
        self.inner.unmount()
    }
}

impl Drop for MountHandle {
    fn drop(&mut self) {
        let _ = self.inner.unmount();
    }
}
```

### Platform Adapters

```
┌─────────────────────────────────────────────────────────────┐
│                     MountHandle (unified API)               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ FuseAdapter │  │ FuseAdapter │  │ WinFspAdapter       │ │
│  │   (Linux)   │  │   (macOS)   │  │    (Windows)        │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
│         │                │                     │            │
│         ▼                ▼                     ▼            │
│    ┌─────────┐      ┌─────────┐         ┌──────────┐       │
│    │  fuser  │      │  fuser  │         │  winfsp  │       │
│    │  crate  │      │  crate  │         │  crate   │       │
│    └────┬────┘      └────┬────┘         └────┬─────┘       │
│         │                │                   │              │
│         ▼                ▼                   ▼              │
│    ┌─────────┐      ┌─────────┐         ┌──────────┐       │
│    │  FUSE   │      │ macFUSE │         │  WinFsp  │       │
│    │ (kernel)│      │ (kext)  │         │ (driver) │       │
│    └─────────┘      └─────────┘         └──────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Crate Structure

```
anyfs-mount/                    # Unified mounting crate
  Cargo.toml
  src/
    lib.rs                      # MountHandle, MountError

    unix/
      mod.rs                    # cfg(unix)
      fuse_adapter.rs           # FUSE implementation via fuser

    windows/
      mod.rs                    # cfg(windows)
      winfsp_adapter.rs         # WinFsp implementation
      dokan_adapter.rs          # Dokan implementation (alternative)
```

### Cargo.toml

```toml
[package]
name = "anyfs-mount"
version = "0.1.0"

[dependencies]
anyfs-backend = { version = "0.1" }

[target.'cfg(unix)'.dependencies]
fuser = "0.14"

[target.'cfg(windows)'.dependencies]
winfsp = "0.4"
# or: dokan = "0.3"

[features]
default = []
dokan = ["dep:dokan"]  # Use Dokan instead of WinFsp on Windows
```

---

## FUSE Adapter (Linux/macOS)

The FUSE adapter translates between `fuser::Filesystem` trait and our `FsFuse` trait:

```rust
use fuser::{Filesystem, Request, ReplyEntry, ReplyAttr, ReplyData, ReplyDirectory};
use anyfs_backend::{FsFuse, FsError, Metadata, FileType};

pub struct FuseAdapter<B: FsFuse> {
    backend: B,
}

impl<B: FsFuse> Filesystem for FuseAdapter<B> {
    fn lookup(&mut self, _req: &Request, parent: u64, name: &OsStr, reply: ReplyEntry) {
        match self.backend.lookup(parent, name) {
            Ok(inode) => {
                match self.backend.metadata_by_inode(inode) {
                    Ok(meta) => reply.entry(&TTL, &to_fuse_attr(&meta), 0),
                    Err(e) => reply.error(to_errno(&e)),
                }
            }
            Err(e) => reply.error(to_errno(&e)),
        }
    }

    fn getattr(&mut self, _req: &Request, ino: u64, reply: ReplyAttr) {
        match self.backend.metadata_by_inode(ino) {
            Ok(meta) => reply.attr(&TTL, &to_fuse_attr(&meta)),
            Err(e) => reply.error(to_errno(&e)),
        }
    }

    fn read(&mut self, _req: &Request, ino: u64, _fh: u64, offset: i64, size: u32, _flags: i32, _lock: Option<u64>, reply: ReplyData) {
        let path = match self.backend.inode_to_path(ino) {
            Ok(p) => p,
            Err(e) => return reply.error(to_errno(&e)),
        };

        match self.backend.read_range(&path, offset as u64, size as usize) {
            Ok(data) => reply.data(&data),
            Err(e) => reply.error(to_errno(&e)),
        }
    }

    fn readdir(&mut self, _req: &Request, ino: u64, _fh: u64, offset: i64, mut reply: ReplyDirectory) {
        let path = match self.backend.inode_to_path(ino) {
            Ok(p) => p,
            Err(e) => return reply.error(to_errno(&e)),
        };

        match self.backend.read_dir(&path) {
            Ok(entries) => {
                for (i, entry) in entries.iter().enumerate().skip(offset as usize) {
                    let file_type = match entry.file_type {
                        FileType::File => fuser::FileType::RegularFile,
                        FileType::Directory => fuser::FileType::Directory,
                        FileType::Symlink => fuser::FileType::Symlink,
                    };

                    if reply.add(entry.inode, (i + 1) as i64, file_type, &entry.name) {
                        break;
                    }
                }
                reply.ok();
            }
            Err(e) => reply.error(to_errno(&e)),
        }
    }

    // ... write, create, mkdir, unlink, rmdir, rename, symlink, etc.
}

fn to_errno(e: &FsError) -> i32 {
    match e {
        FsError::NotFound { .. } => libc::ENOENT,
        FsError::AlreadyExists { .. } => libc::EEXIST,
        FsError::NotADirectory { .. } => libc::ENOTDIR,
        FsError::NotAFile { .. } => libc::EISDIR,
        FsError::DirectoryNotEmpty { .. } => libc::ENOTEMPTY,
        FsError::AccessDenied { .. } => libc::EACCES,
        FsError::ReadOnly { .. } => libc::EROFS,
        FsError::QuotaExceeded { .. } => libc::ENOSPC,
        _ => libc::EIO,
    }
}
```

---

## WinFsp Adapter (Windows)

WinFsp has a different API but similar concepts:

```rust
use winfsp::filesystem::{FileSystem, FileSystemContext, FileInfo, DirInfo};
use anyfs_backend::{FsFuse, FsError};

pub struct WinFspAdapter<B: FsFuse> {
    backend: B,
}

impl<B: FsFuse> FileSystem for WinFspAdapter<B> {
    fn get_file_info(&self, file_context: &FileContext) -> Result<FileInfo, NTSTATUS> {
        let meta = self.backend.metadata(&file_context.path)
            .map_err(to_ntstatus)?;
        Ok(to_file_info(&meta))
    }

    fn read(&self, file_context: &FileContext, buffer: &mut [u8], offset: u64) -> Result<usize, NTSTATUS> {
        let data = self.backend.read_range(&file_context.path, offset, buffer.len())
            .map_err(to_ntstatus)?;
        buffer[..data.len()].copy_from_slice(&data);
        Ok(data.len())
    }

    fn read_directory(&self, file_context: &FileContext, marker: Option<&str>, callback: impl FnMut(DirInfo)) -> Result<(), NTSTATUS> {
        let entries = self.backend.read_dir(&file_context.path)
            .map_err(to_ntstatus)?;

        for entry in entries {
            callback(to_dir_info(&entry));
        }
        Ok(())
    }

    // ... write, create, delete, rename, etc.
}

fn to_ntstatus(e: FsError) -> NTSTATUS {
    match e {
        FsError::NotFound { .. } => STATUS_OBJECT_NAME_NOT_FOUND,
        FsError::AlreadyExists { .. } => STATUS_OBJECT_NAME_COLLISION,
        FsError::AccessDenied { .. } => STATUS_ACCESS_DENIED,
        FsError::ReadOnly { .. } => STATUS_MEDIA_WRITE_PROTECTED,
        _ => STATUS_INTERNAL_ERROR,
    }
}
```

---

## Usage

### Basic Mount

```rust
use anyfs::{MemoryBackend, QuotaLayer};
use anyfs_mount::MountHandle;

// Create backend with middleware
let backend = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build());

// Mount as drive
let mount = MountHandle::mount(backend, "/mnt/ramdisk")?;

// Now /mnt/ramdisk is a real mount point
// Any application can read/write files there

// Unmount when done (or on drop)
mount.unmount()?;
```

### Windows Drive Letter

```rust
#[cfg(windows)]
let mount = MountHandle::mount(backend, "X:")?;

// Now X: is a virtual drive
```

### Check Availability

```rust
if MountHandle::is_available() {
    let mount = MountHandle::mount(backend, path)?;
} else {
    eprintln!("Mounting not available. Install:");
    #[cfg(target_os = "macos")]
    eprintln!("  - macFUSE: https://osxfuse.github.io/");
    #[cfg(windows)]
    eprintln!("  - WinFsp: https://winfsp.dev/");
}
```

### Mount Options

```rust
let mount = MountHandle::builder(backend)
    .mount_point("/mnt/data")
    .read_only(true)                    // Force read-only mount
    .allow_other(true)                  // Allow other users (Linux/macOS)
    .auto_unmount(true)                 // Unmount on process exit
    .uid(1000)                          // Override UID (Linux/macOS)
    .gid(1000)                          // Override GID (Linux/macOS)
    .mount()?;
```

---

## Error Handling

```rust
pub enum MountError {
    /// Platform doesn't support mounting (e.g., WASM)
    PlatformNotSupported,

    /// Required driver not installed (macFUSE, WinFsp)
    DriverNotInstalled {
        driver: &'static str,
        install_url: &'static str,
    },

    /// Mount point doesn't exist or isn't accessible
    InvalidMountPoint { path: PathBuf },

    /// Mount point already in use
    MountPointBusy { path: PathBuf },

    /// Permission denied (need root/admin)
    PermissionDenied,

    /// Backend error during mount
    Backend(FsError),

    /// Platform-specific error
    Platform(String),
}
```

---

## Integration with Middleware

All middleware works transparently when mounted:

```rust
use anyfs::{SqliteBackend, Quota, PathFilter, Tracing, RateLimit};
use anyfs_mount::MountHandle;

// Build secure, audited, rate-limited mount
let backend = SqliteBackend::open("data.db")?
    .layer(QuotaLayer::builder()
        .max_total_size(1024 * 1024 * 1024)  // 1 GB
        .build())
    .layer(PathFilterLayer::builder()
        .deny("**/.git/**")
        .deny("**/.env")
        .build())
    .layer(RateLimitLayer::builder()
        .max_ops(10000)
        .per_second()
        .build())
    .layer(TracingLayer::new());

let mount = MountHandle::mount(backend, "/mnt/secure")?;

// External apps see a normal filesystem
// But all operations are:
// - Quota-limited
// - Path-filtered
// - Rate-limited
// - Traced/audited
```

---

## Use Cases

### Temporary Workspace

```rust
let workspace = MemoryBackend::new();
let mount = MountHandle::mount(workspace, "/tmp/workspace")?;

// Run build tools that expect real filesystem
std::process::Command::new("cargo")
    .current_dir("/tmp/workspace")
    .arg("build")
    .status()?;
```

### Portable Database as Drive

```rust
// User's files stored in SQLite
let db = SqliteBackend::open("user_files.db")?;
let mount = MountHandle::mount(db, "U:")?;

// User can browse U: in Explorer
// Files are actually in SQLite database
```

### Network Storage

```rust
// Remote backend (future anyfs-s3, anyfs-sftp, etc.)
let remote = S3Backend::new("my-bucket")?;
let cached = remote.layer(CacheLayer::builder()
    .max_size(100 * 1024 * 1024)
    .build());
let mount = MountHandle::mount(cached, "/mnt/cloud")?;

// Local apps see /mnt/cloud as regular filesystem
// Actually reads/writes to S3 with local caching
```

---

## Platform Requirements Summary

| Platform | Driver | Install Command / URL |
|----------|--------|----------------------|
| Linux | FUSE | Usually pre-installed. If not: `apt install fuse3` |
| macOS | macFUSE | https://osxfuse.github.io/ |
| Windows | WinFsp | https://winfsp.dev/ (recommended) |
| Windows | Dokan | https://dokan-dev.github.io/ (alternative) |

---

## Limitations

1. **Requires external driver** - Users must install macFUSE (macOS) or WinFsp (Windows)
2. **Root/admin may be required** - Some mount operations need elevated privileges
3. **Not available on WASM** - Browser environment has no filesystem mounting
4. **Performance overhead** - Userspace filesystem has kernel boundary crossing overhead
5. **Backend must implement FsFuse** - Requires `FsInode` trait for inode operations

---

## Alternative: No Mount Needed

For many use cases, mounting isn't necessary. AnyFS backends can be used directly:

| Need | With Mounting | Without Mounting |
|------|---------------|------------------|
| Build tools | Mount, run tools | Use tool's VFS plugin if available |
| File browser | Mount as drive | Build custom UI with AnyFS API |
| Backup | Mount, use rsync | Use AnyFS API directly |
| Database | Mount for SQL tools | Query SQLite directly |

**Rule of thumb:** Only mount when you need compatibility with external applications that expect real filesystem paths.
