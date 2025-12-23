# FilesContainer (anyfs-container)

**The high-level std::fs-like API for application developers**

---

## Overview

`FilesContainer` provides a familiar `std::fs`-like API on top of any `Vfs` backend.

## Target Audience

**Application developers** — people building apps on top of anyfs.

---

## Key Characteristics

| Aspect | Description |
|--------|-------------|
| Path handling | `impl AsRef<Path>` — accepts `&str`, `String`, `PathBuf`, etc. |
| Semantics | Pluggable via `FsSemantics` |
| Capacity limits | Enforced before operations |
| API style | Matches `std::fs` method names |

---

## Method Mapping to std::fs

| FilesContainer | std::fs |
|----------------|---------|
| `read()` | `std::fs::read` |
| `read_to_string()` | `std::fs::read_to_string` |
| `write()` | `std::fs::write` |
| `read_dir()` | `std::fs::read_dir` |
| `create_dir()` | `std::fs::create_dir` |
| `create_dir_all()` | `std::fs::create_dir_all` |
| `remove_file()` | `std::fs::remove_file` |
| `remove_dir()` | `std::fs::remove_dir` |
| `remove_dir_all()` | `std::fs::remove_dir_all` |
| `rename()` | `std::fs::rename` |
| `copy()` | `std::fs::copy` |
| `metadata()` | `std::fs::metadata` |
| `symlink_metadata()` | `std::fs::symlink_metadata` |
| `symlink()` | `std::os::unix::fs::symlink` |
| `hard_link()` | `std::fs::hard_link` |
| `read_link()` | `std::fs::read_link` |
| `set_permissions()` | `std::fs::set_permissions` |
| `exists()` | `Path::exists` |

---

## API Definition

```rust
impl<V: Vfs, S: FsSemantics> FilesContainer<V, S> {
    // ═══════════════════════════════════════════════════════════
    // READ OPERATIONS
    // ═══════════════════════════════════════════════════════════

    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    pub fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, ContainerError>;
    pub fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, ContainerError>;
    pub fn read_link(&self, path: impl AsRef<Path>) -> Result<PathBuf, ContainerError>;
    pub fn exists(&self, path: impl AsRef<Path>) -> bool;

    // ═══════════════════════════════════════════════════════════
    // WRITE OPERATIONS
    // ═══════════════════════════════════════════════════════════

    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;

    // ═══════════════════════════════════════════════════════════
    // LINK OPERATIONS
    // ═══════════════════════════════════════════════════════════

    pub fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;

    // ═══════════════════════════════════════════════════════════
    // PERMISSIONS
    // ═══════════════════════════════════════════════════════════

    pub fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), ContainerError>;

    // ═══════════════════════════════════════════════════════════
    // CONTAINER-SPECIFIC
    // ═══════════════════════════════════════════════════════════

    pub fn usage(&self) -> &CapacityUsage;
    pub fn limits(&self) -> &CapacityLimits;
}
```

---

## Usage Example

```rust
use anyfs_container::{FilesContainer, ContainerBuilder, LinuxSemantics, CapacityLimits};
use anyfs::MemoryVfs;

// Simple creation
let mut container = FilesContainer::new(
    MemoryVfs::new(),
    LinuxSemantics::new(),
);

// Or with builder for limits
let mut container = ContainerBuilder::new()
    .vfs(MemoryVfs::new())
    .semantics(LinuxSemantics::new())
    .limits(CapacityLimits {
        max_total_size: Some(10 * 1024 * 1024),  // 10 MB
        max_file_size: Some(1024 * 1024),         // 1 MB per file
        ..Default::default()
    })
    .build()?;

// Now use like std::fs
container.create_dir_all("/var/log")?;
container.write("/var/log/app.log", b"Started\n")?;

let data = container.read("/var/log/app.log")?;
assert_eq!(data, b"Started\n");

for entry in container.read_dir("/var")? {
    println!("{}: {:?}", entry.name(), entry.file_type());
}
```

---

## Capacity Limits

```rust
pub struct CapacityLimits {
    pub max_total_size: Option<u64>,
    pub max_file_size: Option<u64>,
    pub max_node_count: Option<u64>,
    pub max_dir_entries: Option<u32>,
    pub max_path_depth: Option<u16>,
}
```

Limits are enforced **before** operations:

```rust
container.write("/big-file", &large_data)?;
// Returns ContainerError::TotalSizeExceeded if over limit
```

---

## Error Handling

```rust
use anyfs_container::ContainerError;

match container.write("/path", &data) {
    Ok(()) => println!("Written"),
    Err(ContainerError::NotFound(p)) => println!("Path not found: {:?}", p),
    Err(ContainerError::AlreadyExists(p)) => println!("Already exists: {:?}", p),
    Err(ContainerError::TotalSizeExceeded { used, limit }) => {
        println!("Storage full: {} / {} bytes", used, limit);
    }
    Err(e) => println!("Other error: {}", e),
}
```

---

## Symlink Behavior

| Operation | Follows Symlinks? |
|-----------|-------------------|
| `read()` | Yes |
| `write()` | Yes |
| `copy()` | Yes (copies content) |
| `exists()` | Yes (broken = false) |
| `metadata()` | Yes |
| `symlink_metadata()` | **No** |
| `read_link()` | **No** |
| `remove_file()` | **No** (removes link) |
| `rename()` | **No** (renames link) |
