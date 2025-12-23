# Design Alignment Suggestions

**Status:** Proposed changes to align with `std::fs` and add full filesystem capabilities  
**Date:** 2024-12-22  
**Applies to:** `anyfs-traits`, `anyfs`, `anyfs-container`

---

## Summary of Changes

| Category | Changes |
|----------|---------|
| Method renaming | 6 methods renamed to match `std::fs` |
| New methods | 6 methods added |
| Type updates | `FileType`, `Metadata`, `Permissions`, `DirEntry` |
| Error updates | New error variants |
| Backend schemas | Updated for symlinks/hardlinks |

**Total trait methods:** 20 (was 13)

---

## 1. Method Renaming

### 1.1 Changes Required

| Current Name | New Name | Rationale |
|--------------|----------|-----------|
| `list()` | `read_dir()` | Matches `std::fs::read_dir` |
| `mkdir()` | `create_dir()` | Matches `std::fs::create_dir` |
| `mkdir_all()` | `create_dir_all()` | Matches `std::fs::create_dir_all` |
| `remove()` | `remove_file()` | Matches `std::fs::remove_file` |
| `remove()` | `remove_dir()` | Matches `std::fs::remove_dir` (split) |
| `remove_all()` | `remove_dir_all()` | Matches `std::fs::remove_dir_all` |

### 1.2 The `remove()` Split

**Current (wrong):**
```rust
/// Remove file or empty directory
fn remove(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
```

**New (correct):**
```rust
/// Remove file (like std::fs::remove_file)
fn remove_file(&mut self, path: &VirtualPath) -> Result<(), VfsError>;

/// Remove empty directory (like std::fs::remove_dir)
fn remove_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
```

**Rationale:** `std::fs` has separate functions. This makes intent explicit and error handling clearer.

---

## 2. New Methods to Add

### 2.1 `read_to_string()`

```rust
/// Read entire file contents as UTF-8 string (like std::fs::read_to_string)
fn read_to_string(&self, path: &VirtualPath) -> Result<String, VfsError>;
```

**Default implementation possible:**
```rust
fn read_to_string(&self, path: &VirtualPath) -> Result<String, VfsError> {
    let bytes = self.read(path)?;
    String::from_utf8(bytes).map_err(|e| VfsError::InvalidUtf8(e.utf8_error()))
}
```

### 2.2 `symlink_metadata()`

```rust
/// Get metadata without following symlinks (like std::fs::symlink_metadata / lstat)
fn symlink_metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
```

**Difference from `metadata()`:**
- `metadata()` — follows symlinks, returns metadata of target
- `symlink_metadata()` — does NOT follow symlinks, returns metadata of the link itself

### 2.3 `read_link()`

```rust
/// Read symbolic link target (like std::fs::read_link)
fn read_link(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError>;
```

**Errors:**
- `NotFound` — path doesn't exist
- `NotASymlink` — path exists but is not a symlink

### 2.4 `symlink()`

```rust
/// Create symbolic link pointing to original (like std::os::unix::fs::symlink)
fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
```

**Parameters:**
- `original` — the path the symlink points TO (target)
- `link` — the path of the symlink itself (created)

**Note:** `original` does NOT need to exist (dangling symlinks are valid).

### 2.5 `hard_link()`

```rust
/// Create hard link (like std::fs::hard_link)
fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
```

**Parameters:**
- `original` — existing file to link to (must exist, must be a file)
- `link` — new path that will point to same content

**Constraints:**
- `original` must exist and be a file (not directory)
- Cannot hard link across different backends (not applicable — single backend)

### 2.6 `set_permissions()`

```rust
/// Set permissions on file or directory (like std::fs::set_permissions)
fn set_permissions(&mut self, path: &VirtualPath, perm: Permissions) -> Result<(), VfsError>;
```

---

## 3. Complete Trait Definition

```rust
use std::time::SystemTime;

/// A virtual filesystem backend.
/// All backends implement full filesystem semantics including symlinks and hard links.
pub trait VfsBackend: Send {
    // ═══════════════════════════════════════════════════════════════════════════
    // READ OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════════
    
    /// Read entire file contents as bytes.
    /// Follows symlinks.
    /// (like std::fs::read)
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError>;
    
    /// Read entire file contents as UTF-8 string.
    /// Follows symlinks.
    /// (like std::fs::read_to_string)
    fn read_to_string(&self, path: &VirtualPath) -> Result<String, VfsError>;
    
    /// Read a byte range from a file.
    /// Follows symlinks.
    /// (extension — not in std)
    fn read_range(&self, path: &VirtualPath, offset: u64, len: usize) -> Result<Vec<u8>, VfsError>;
    
    /// Check if path exists.
    /// Follows symlinks (returns false for broken symlinks).
    /// (like Path::exists)
    fn exists(&self, path: &VirtualPath) -> Result<bool, VfsError>;
    
    /// Get metadata, following symlinks.
    /// (like std::fs::metadata)
    fn metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    
    /// Get metadata without following symlinks.
    /// (like std::fs::symlink_metadata)
    fn symlink_metadata(&self, path: &VirtualPath) -> Result<Metadata, VfsError>;
    
    /// Read directory contents.
    /// Does not follow symlinks for the directory itself.
    /// (like std::fs::read_dir)
    fn read_dir(&self, path: &VirtualPath) -> Result<Vec<DirEntry>, VfsError>;
    
    /// Read symbolic link target.
    /// (like std::fs::read_link)
    fn read_link(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError>;
    
    // ═══════════════════════════════════════════════════════════════════════════
    // WRITE OPERATIONS
    // ═══════════════════════════════════════════════════════════════════════════
    
    /// Write bytes to file, creating or overwriting.
    /// Follows symlinks.
    /// (like std::fs::write)
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    
    /// Append bytes to file.
    /// Follows symlinks.
    /// (like OpenOptions::append)
    fn append(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError>;
    
    /// Create directory.
    /// (like std::fs::create_dir)
    fn create_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    
    /// Create directory and all parent directories.
    /// (like std::fs::create_dir_all)
    fn create_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    
    /// Remove file.
    /// Removes symlink itself, not target.
    /// (like std::fs::remove_file)
    fn remove_file(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    
    /// Remove empty directory.
    /// (like std::fs::remove_dir)
    fn remove_dir(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    
    /// Remove directory and all contents recursively.
    /// (like std::fs::remove_dir_all)
    fn remove_dir_all(&mut self, path: &VirtualPath) -> Result<(), VfsError>;
    
    /// Rename or move file/directory.
    /// (like std::fs::rename)
    fn rename(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    
    /// Copy file.
    /// Follows symlinks (copies content, not link).
    /// (like std::fs::copy)
    fn copy(&mut self, from: &VirtualPath, to: &VirtualPath) -> Result<(), VfsError>;
    
    // ═══════════════════════════════════════════════════════════════════════════
    // LINKS
    // ═══════════════════════════════════════════════════════════════════════════
    
    /// Create symbolic link.
    /// `link` will point to `original`.
    /// `original` does not need to exist (dangling symlink).
    /// (like std::os::unix::fs::symlink)
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
    
    /// Create hard link.
    /// `link` will share content with `original`.
    /// `original` must exist and be a file.
    /// (like std::fs::hard_link)
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
    
    // ═══════════════════════════════════════════════════════════════════════════
    // PERMISSIONS
    // ═══════════════════════════════════════════════════════════════════════════
    
    /// Set permissions on file or directory.
    /// (like std::fs::set_permissions)
    fn set_permissions(&mut self, path: &VirtualPath, perm: Permissions) -> Result<(), VfsError>;
}
```

---

## 4. Type Definitions

### 4.1 FileType

```rust
/// Type of filesystem entry.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum FileType {
    /// Regular file
    File,
    /// Directory
    Directory,
    /// Symbolic link
    Symlink,
}

impl FileType {
    pub fn is_file(&self) -> bool {
        matches!(self, FileType::File)
    }
    
    pub fn is_dir(&self) -> bool {
        matches!(self, FileType::Directory)
    }
    
    pub fn is_symlink(&self) -> bool {
        matches!(self, FileType::Symlink)
    }
}
```

### 4.2 Metadata

```rust
/// Metadata about a filesystem entry.
#[derive(Clone, Debug)]
pub struct Metadata {
    /// Type of entry (file, directory, symlink)
    pub file_type: FileType,
    
    /// Size in bytes (0 for directories, target path length for symlinks)
    pub size: u64,
    
    /// Permissions
    pub permissions: Permissions,
    
    /// Number of hard links pointing to this content
    pub nlink: u64,
    
    /// Creation time (if available)
    pub created: Option<SystemTime>,
    
    /// Last modification time (if available)
    pub modified: Option<SystemTime>,
    
    /// Last access time (if available)
    pub accessed: Option<SystemTime>,
}

impl Metadata {
    pub fn is_file(&self) -> bool {
        self.file_type.is_file()
    }
    
    pub fn is_dir(&self) -> bool {
        self.file_type.is_dir()
    }
    
    pub fn is_symlink(&self) -> bool {
        self.file_type.is_symlink()
    }
    
    pub fn len(&self) -> u64 {
        self.size
    }
}
```

### 4.3 Permissions

```rust
/// File permissions.
/// Simplified model — can expand to full Unix mode later.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Default)]
pub struct Permissions {
    /// If true, file cannot be written
    readonly: bool,
}

impl Permissions {
    pub fn new() -> Self {
        Self { readonly: false }
    }
    
    pub fn readonly(&self) -> bool {
        self.readonly
    }
    
    pub fn set_readonly(&mut self, readonly: bool) {
        self.readonly = readonly;
    }
}
```

**Future expansion:**
```rust
// Later, could expand to:
pub struct Permissions {
    mode: u32,  // Unix permission bits (rwxrwxrwx)
}
```

### 4.4 DirEntry

```rust
/// Entry in a directory listing.
#[derive(Clone, Debug)]
pub struct DirEntry {
    /// Name of the entry (not full path)
    pub name: String,
    
    /// Type of entry
    pub file_type: FileType,
}

impl DirEntry {
    pub fn name(&self) -> &str {
        &self.name
    }
    
    pub fn file_type(&self) -> FileType {
        self.file_type
    }
}
```

---

## 5. Error Types

### 5.1 VfsError

```rust
#[derive(Debug, thiserror::Error)]
pub enum VfsError {
    #[error("not found: {0}")]
    NotFound(VirtualPath),
    
    #[error("already exists: {0}")]
    AlreadyExists(VirtualPath),
    
    #[error("not a file: {0}")]
    NotAFile(VirtualPath),
    
    #[error("not a directory: {0}")]
    NotADirectory(VirtualPath),
    
    #[error("not a symlink: {0}")]
    NotASymlink(VirtualPath),
    
    #[error("directory not empty: {0}")]
    DirectoryNotEmpty(VirtualPath),
    
    #[error("is a directory: {0}")]
    IsADirectory(VirtualPath),
    
    #[error("invalid path: {0}")]
    InvalidPath(String),
    
    #[error("invalid UTF-8: {0}")]
    InvalidUtf8(#[from] std::str::Utf8Error),
    
    #[error("permission denied: {0}")]
    PermissionDenied(VirtualPath),
    
    #[error("read-only filesystem")]
    ReadOnlyFilesystem,
    
    #[error("too many symlinks (loop detected)")]
    SymlinkLoop(VirtualPath),
    
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("backend error: {0}")]
    Backend(String),
}
```

### 5.2 New Error Variants Explained

| Variant | When Used |
|---------|-----------|
| `NotASymlink` | `read_link()` called on file or directory |
| `IsADirectory` | `remove_file()` called on directory |
| `PermissionDenied` | Write to readonly file, or other permission failure |
| `SymlinkLoop` | Too many symlink levels (prevent infinite loops) |
| `InvalidUtf8` | `read_to_string()` on non-UTF-8 file |

---

## 6. Backend Implementation Details

### 6.1 Symlink Loop Detection

All backends must detect symlink loops. Recommended: limit to 40 symlink resolutions (matches Linux default).

```rust
const MAX_SYMLINK_DEPTH: u32 = 40;

fn resolve_symlinks(&self, path: &VirtualPath) -> Result<VirtualPath, VfsError> {
    let mut current = path.clone();
    let mut depth = 0;
    
    loop {
        match self.symlink_metadata(&current)?.file_type {
            FileType::Symlink => {
                depth += 1;
                if depth > MAX_SYMLINK_DEPTH {
                    return Err(VfsError::SymlinkLoop(path.clone()));
                }
                current = self.read_link(&current)?;
            }
            _ => return Ok(current),
        }
    }
}
```

### 6.2 MemoryBackend Internal Structure

```rust
use std::collections::HashMap;
use std::sync::Arc;
use std::time::SystemTime;

pub struct MemoryBackend {
    entries: HashMap<VirtualPath, Entry>,
    contents: HashMap<ContentId, Arc<ContentData>>,
    next_content_id: ContentId,
}

#[derive(Clone, Copy, PartialEq, Eq, Hash)]
struct ContentId(u64);

struct ContentData {
    data: Vec<u8>,
    ref_count: std::sync::atomic::AtomicU64,
}

enum Entry {
    File {
        content_id: ContentId,
        permissions: Permissions,
        created: SystemTime,
        modified: SystemTime,
        accessed: SystemTime,
    },
    Directory {
        permissions: Permissions,
        created: SystemTime,
        modified: SystemTime,
        accessed: SystemTime,
    },
    Symlink {
        target: VirtualPath,
        created: SystemTime,
        modified: SystemTime,
    },
}
```

**Hard link implementation:**
- Multiple `Entry::File` entries can share the same `content_id`
- `nlink` = count of entries with same `content_id`
- When last hard link is removed, content is deleted

### 6.3 SqliteBackend Schema

```sql
-- Entries table: files, directories, symlinks
CREATE TABLE entries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    path TEXT NOT NULL UNIQUE,
    entry_type TEXT NOT NULL CHECK (entry_type IN ('file', 'directory', 'symlink')),
    content_id INTEGER REFERENCES contents(id) ON DELETE SET NULL,
    symlink_target TEXT,
    permissions INTEGER NOT NULL DEFAULT 0,
    created_at INTEGER,
    modified_at INTEGER,
    accessed_at INTEGER
);

-- Content table: actual file data (shared by hard links)
CREATE TABLE contents (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    data BLOB NOT NULL,
    ref_count INTEGER NOT NULL DEFAULT 1
);

-- Indexes
CREATE UNIQUE INDEX idx_entries_path ON entries(path);
CREATE INDEX idx_entries_content ON entries(content_id) WHERE content_id IS NOT NULL;

-- Constraints enforced in application:
-- - entry_type = 'file' => content_id NOT NULL, symlink_target IS NULL
-- - entry_type = 'directory' => content_id IS NULL, symlink_target IS NULL
-- - entry_type = 'symlink' => content_id IS NULL, symlink_target NOT NULL
```

**Hard link implementation:**
- Multiple rows with different `path` can share same `content_id`
- `ref_count` in contents table tracks hard link count
- When `ref_count` reaches 0, content row is deleted

**nlink query:**
```sql
SELECT COUNT(*) FROM entries WHERE content_id = ?
```

### 6.4 VRootFsBackend

Uses real OS symlinks and hard links via `std::os::unix::fs` (or Windows equivalents).

```rust
impl VfsBackend for VRootFsBackend {
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        let link_path = self.vroot.virtual_join(link)?;
        // original is stored as-is (relative to virtual root)
        std::os::unix::fs::symlink(original.as_os_str(), link_path.as_std_path())
            .map_err(VfsError::Io)
    }
    
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        let original_path = self.vroot.virtual_join(original)?;
        let link_path = self.vroot.virtual_join(link)?;
        std::fs::hard_link(original_path.as_std_path(), link_path.as_std_path())
            .map_err(VfsError::Io)
    }
}
```

---

## 7. FilesContainer Updates

### 7.1 Method Signatures

```rust
impl<B: VfsBackend> FilesContainer<B> {
    // All methods use impl AsRef<Path> for ergonomics
    // Convert to VirtualPath internally
    
    pub fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, ContainerError>;
    pub fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, ContainerError>;
    pub fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, ContainerError>;
    pub fn exists(&self, path: impl AsRef<Path>) -> Result<bool, ContainerError>;
    pub fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn symlink_metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, ContainerError>;
    pub fn read_dir(&self, path: impl AsRef<Path>) -> Result<Vec<DirEntry>, ContainerError>;
    pub fn read_link(&self, path: impl AsRef<Path>) -> Result<VirtualPath, ContainerError>;
    
    pub fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn append(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), ContainerError>;
    pub fn create_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn create_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_file(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn remove_dir_all(&mut self, path: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn rename(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn copy(&mut self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), ContainerError>;
    
    pub fn symlink(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError>;
    pub fn set_permissions(&mut self, path: impl AsRef<Path>, perm: Permissions) -> Result<(), ContainerError>;
}
```

### 7.2 Capacity Limit Considerations

**Symlinks:**
- Count toward `max_node_count`
- Do NOT count toward `max_total_size` (they're just pointers)

**Hard links:**
- Count toward `max_node_count` (each link is a separate entry)
- Content size counted ONCE toward `max_total_size` (not per-link)

```rust
impl<B: VfsBackend> FilesContainer<B> {
    pub fn hard_link(&mut self, original: impl AsRef<Path>, link: impl AsRef<Path>) -> Result<(), ContainerError> {
        // Check node count limit
        self.check_node_limit()?;
        
        // Do NOT check size limit — no new content created
        
        let original_vpath = VirtualPath::new(original.as_ref())?;
        let link_vpath = VirtualPath::new(link.as_ref())?;
        
        self.backend.hard_link(&original_vpath, &link_vpath)?;
        
        // Update usage
        self.usage.node_count += 1;
        self.usage.file_count += 1;
        // total_size unchanged — content is shared
        
        Ok(())
    }
}
```

---

## 8. AGENTS.md Updates Required

### 8.1 Update Trait Definition

Replace the current 13-method trait with the new 20-method trait (Section 3).

### 8.2 Update Method List

**Old (remove):**
```
list, mkdir, mkdir_all, remove, remove_all
```

**New (add):**
```
read_dir, create_dir, create_dir_all, remove_file, remove_dir, remove_dir_all,
read_to_string, symlink_metadata, read_link, symlink, hard_link, set_permissions
```

### 8.3 Add to "Common Mistakes to Avoid"

```markdown
### ❌ WRONG: Using old method names

```rust
// WRONG - old method names
vfs.list(path)?;
vfs.mkdir(path)?;
vfs.remove(path)?;
```

```rust
// CORRECT - std-aligned names
vfs.read_dir(path)?;
vfs.create_dir(path)?;
vfs.remove_file(path)?;  // or remove_dir()
```
```

### 8.4 Update Quick Reference Table

| Question | Answer |
|----------|--------|
| Directory listing method? | `read_dir()` (not `list()`) |
| Create directory method? | `create_dir()` (not `mkdir()`) |
| Remove file method? | `remove_file()` (not `remove()`) |
| Remove directory method? | `remove_dir()` (not `remove()`) |
| Total trait methods? | 20 |
| Symlinks supported? | Yes, all backends |
| Hard links supported? | Yes, all backends |

---

## 9. Documentation Updates

### 9.1 Files to Update

| File | Changes Required |
|------|------------------|
| `AGENTS.md` | Update trait, add method rename warnings |
| `book/src/vfs-design.md` | Update trait, types, errors |
| `book/src/api-reference.md` | Update method list |
| `book/src/backends/*.md` | Add symlink/hardlink implementation notes |

### 9.2 Add New Section: Symlink Semantics

Document which operations follow symlinks and which don't:

| Operation | Follows Symlinks? |
|-----------|-------------------|
| `read()` | Yes |
| `read_to_string()` | Yes |
| `read_range()` | Yes |
| `write()` | Yes |
| `append()` | Yes |
| `copy()` | Yes (copies content, not link) |
| `exists()` | Yes (broken symlink = false) |
| `metadata()` | Yes |
| `symlink_metadata()` | **No** |
| `read_dir()` | No (lists symlink as entry) |
| `read_link()` | **No** (reads the link itself) |
| `remove_file()` | **No** (removes link, not target) |
| `remove_dir()` | No |
| `remove_dir_all()` | No (removes links, not targets) |
| `rename()` | **No** (renames link, not target) |
| `set_permissions()` | Yes |

---

## 10. Migration Checklist

### 10.1 anyfs-traits Crate

- [ ] Rename `list()` → `read_dir()`
- [ ] Rename `mkdir()` → `create_dir()`
- [ ] Rename `mkdir_all()` → `create_dir_all()`
- [ ] Split `remove()` → `remove_file()` + `remove_dir()`
- [ ] Rename `remove_all()` → `remove_dir_all()`
- [ ] Add `read_to_string()`
- [ ] Add `symlink_metadata()`
- [ ] Add `read_link()`
- [ ] Add `symlink()`
- [ ] Add `hard_link()`
- [ ] Add `set_permissions()`
- [ ] Add `FileType::Symlink`
- [ ] Update `Metadata` struct (add `nlink`, `permissions`, `accessed`)
- [ ] Add `Permissions` struct
- [ ] Add new `VfsError` variants

### 10.2 anyfs Crate (Backends)

- [ ] Update `MemoryBackend` for new trait
- [ ] Add symlink/hardlink support to `MemoryBackend`
- [ ] Update `SqliteBackend` schema
- [ ] Add symlink/hardlink support to `SqliteBackend`
- [ ] Update `VRootFsBackend` for new trait
- [ ] Add symlink loop detection

### 10.3 anyfs-container Crate

- [ ] Update `FilesContainer` methods
- [ ] Update capacity tracking for symlinks/hardlinks
- [ ] Update `ContainerError` if needed

### 10.4 Documentation

- [ ] Update `AGENTS.md`
- [ ] Update `book/src/vfs-design.md`
- [ ] Update all mdbook chapters
- [ ] Add symlink semantics documentation

---

## 11. Summary

### Methods: 13 → 20

| Category | Old | New | Delta |
|----------|-----|-----|-------|
| Read | 6 | 8 | +2 |
| Write | 7 | 9 | +2 (split remove) |
| Links | 0 | 2 | +2 |
| Permissions | 0 | 1 | +1 |
| **Total** | **13** | **20** | **+7** |

### Key Changes

1. **Method names aligned with `std::fs`**
2. **Full symlink support** — create, read, metadata, loop detection
3. **Full hard link support** — shared content, reference counting
4. **Permissions support** — readonly flag (expandable to Unix mode)
5. **Split remove()** — separate `remove_file()` and `remove_dir()`
6. **New error variants** — `NotASymlink`, `SymlinkLoop`, `PermissionDenied`, etc.
