# Which Layer Should I Use?

**A guide to choosing between anyfs and anyfs-container**

---

## Decision Tree

```
Are you building an application?
│
├── YES → Use `FilesContainer` from `anyfs-container`
│         (std::fs-like API, handles paths for you)
│
└── NO → Are you creating a new storage backend?
         │
         ├── YES → Implement `Vfs` trait from `anyfs`
         │         (low-level inode operations)
         │
         └── NO → Are you creating custom path semantics?
                  │
                  ├── YES → Implement `FsSemantics` trait
                  │
                  └── NO → You probably want `FilesContainer`
```

---

## Quick Summary

| I want to... | Use |
|--------------|-----|
| Store files in my app | `FilesContainer` |
| Create a Redis-backed VFS | `Vfs` trait |
| Support Windows paths on Linux | `WindowsSemantics` |
| Test with in-memory filesystem | `FilesContainer` + `MemoryVfs` |
| Build a FUSE adapter | `Vfs` trait |
| Enforce storage quotas | `FilesContainer` with limits |

---

## Examples by Use Case

### Building a Web App with File Storage

**Layer:** `anyfs-container`

```rust
use anyfs_container::{FilesContainer, LinuxSemantics};
use anyfs::SqliteVfs;

let mut container = FilesContainer::new(
    SqliteVfs::open("app-data.db")?,
    LinuxSemantics::new(),
);

// Just like std::fs!
container.write("/uploads/photo.jpg", &image_data)?;
```

### Creating an S3-Backed Storage

**Layer:** `anyfs` (implement `Vfs`)

```rust
use anyfs::{Vfs, InodeId, InodeKind, VfsError};

pub struct S3Vfs {
    bucket: String,
    client: S3Client,
    // ... inode metadata cache
}

impl Vfs for S3Vfs {
    fn create_inode(&mut self, kind: InodeKind, mode: u32) -> Result<InodeId, VfsError> {
        // Store inode metadata in S3 or a sidecar DB
    }

    fn write(&mut self, id: InodeId, offset: u64, data: &[u8]) -> Result<usize, VfsError> {
        // Upload to S3
    }

    // ... other methods
}
```

### Testing File Operations

**Layer:** `anyfs-container` with `MemoryVfs`

```rust
#[test]
fn test_file_operations() {
    let mut container = FilesContainer::new(
        MemoryVfs::new(),
        SimpleSemantics::new(),  // Fast, no symlink overhead
    );

    container.write("/test.txt", b"hello")?;
    assert_eq!(container.read("/test.txt")?, b"hello");
}
```

### Building a FUSE Filesystem

**Layer:** `anyfs` (use `Vfs` directly)

```rust
use fuser::Filesystem;
use anyfs::{Vfs, InodeId};

pub struct AnyFsFuse<V: Vfs> {
    vfs: V,
}

impl<V: Vfs> Filesystem for AnyFsFuse<V> {
    fn lookup(&mut self, _req: &Request, parent: u64, name: &OsStr, reply: ReplyEntry) {
        let parent_id = InodeId(parent);
        match self.vfs.lookup(parent_id, name.to_str().unwrap()) {
            Ok(child_id) => {
                let inode = self.vfs.get_inode(child_id).unwrap();
                reply.entry(&TTL, &to_fuse_attr(&inode), 0);
            }
            Err(_) => reply.error(ENOENT),
        }
    }
}
```

### Multi-Tenant SaaS with Quotas

**Layer:** `anyfs-container` with limits

```rust
use anyfs_container::{ContainerBuilder, CapacityLimits};

fn create_tenant_storage(tenant_id: &str) -> FilesContainer<SqliteVfs, LinuxSemantics> {
    let db_path = format!("tenants/{}.db", tenant_id);

    ContainerBuilder::new()
        .vfs(SqliteVfs::create(&db_path).unwrap())
        .semantics(LinuxSemantics::new())
        .limits(CapacityLimits {
            max_total_size: Some(100 * 1024 * 1024),  // 100 MB per tenant
            max_file_size: Some(10 * 1024 * 1024),    // 10 MB max file
            max_node_count: Some(10_000),
            ..Default::default()
        })
        .build()
        .unwrap()
}
```

---

## Layer Comparison

| Aspect | anyfs (Vfs) | anyfs-container |
|--------|-------------|-----------------|
| **API** | `InodeId`-based | `impl AsRef<Path>` |
| **Complexity** | More work | Easy |
| **Control** | Full | Limited |
| **Path handling** | You do it | Automatic |
| **Limits** | You enforce | Built-in |
| **Use case** | Custom backends | Applications |

---

## Still Unsure?

**Default to `anyfs-container`** unless you:

1. Need to create a new storage backend
2. Need direct inode access for FUSE
3. Have very specific performance requirements

For most applications, `FilesContainer` is the right choice.
