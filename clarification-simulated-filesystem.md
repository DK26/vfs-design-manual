# Clarification: Simulated Filesystem — Not Real I/O

**This document clarifies a critical design point that may cause confusion.**

---

## The Core Principle

> **The `VfsBackend` trait defines a filesystem-like API. It does NOT imply real filesystem operations.**

When you call `backend.symlink()`, `backend.write()`, or `backend.hard_link()`:
- You are calling **our API methods**
- The backend decides **how to store** that information
- **No real OS filesystem calls happen** unless the backend explicitly does so

---

## What Each Backend Actually Does

| Backend | `write("/foo", data)` | `symlink("/target", "/link")` | `hard_link("/a", "/b")` |
|---------|----------------------|------------------------------|------------------------|
| **MemoryBackend** | Stores bytes in a `HashMap` | Stores target path in an `Entry::Symlink` | Two entries share same `ContentId` |
| **SqliteBackend** | Inserts row in `contents` table | Inserts row with `entry_type='symlink'` | Two rows reference same `content_id` |
| **VRootFsBackend** | Calls real `fs::write()` via strict-path | Calls real `std::os::unix::fs::symlink()` | Calls real `std::fs::hard_link()` |

---

## MemoryBackend: Pure Data Structures

```rust
// Nothing here touches the real filesystem

pub struct MemoryBackend {
    entries: HashMap<VirtualPath, Entry>,
    contents: HashMap<ContentId, Vec<u8>>,
}

enum Entry {
    File { content_id: ContentId, ... },
    Directory { ... },
    Symlink { target: VirtualPath, ... },  // Just a stored path, not a real symlink
}

impl VfsBackend for MemoryBackend {
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        // This does NOT call any OS function
        // It just inserts a data structure
        self.entries.insert(link.clone(), Entry::Symlink {
            target: original.clone(),
            created: SystemTime::now(),
            modified: SystemTime::now(),
        });
        Ok(())
    }
    
    fn read(&self, path: &VirtualPath) -> Result<Vec<u8>, VfsError> {
        // Follow simulated symlinks by looking up entries
        let resolved = self.resolve_path(path)?;  // Internal lookup, not OS call
        match self.entries.get(&resolved) {
            Some(Entry::File { content_id, .. }) => {
                Ok(self.contents.get(content_id).cloned().unwrap_or_default())
            }
            // ...
        }
    }
}
```

**Key point:** `MemoryBackend` is entirely in-process. It simulates a filesystem using Rust data structures. The "symlinks" are just entries that store a target path. When you "read" through a symlink, the backend resolves it by looking up entries in its HashMap — no OS involvement.

---

## SqliteBackend: Database Rows

```sql
-- This is NOT the real filesystem
-- This is a SQLite database file

CREATE TABLE entries (
    path TEXT PRIMARY KEY,
    entry_type TEXT,        -- 'file', 'directory', 'symlink'
    content_id INTEGER,     -- FK to contents table (for files)
    symlink_target TEXT,    -- Target path (for symlinks)
    ...
);

CREATE TABLE contents (
    id INTEGER PRIMARY KEY,
    data BLOB,
    ref_count INTEGER       -- For hard links
);
```

```rust
impl VfsBackend for SqliteBackend {
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        // This does NOT create a real symlink on disk
        // It inserts a row into SQLite
        self.conn.execute(
            "INSERT INTO entries (path, entry_type, symlink_target) VALUES (?, 'symlink', ?)",
            params![link.as_str(), original.as_str()],
        )?;
        Ok(())
    }
    
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        // This does NOT create a real hard link
        // It creates a new row pointing to the same content_id
        let content_id: i64 = self.conn.query_row(
            "SELECT content_id FROM entries WHERE path = ?",
            params![original.as_str()],
            |row| row.get(0),
        )?;
        
        self.conn.execute(
            "INSERT INTO entries (path, entry_type, content_id) VALUES (?, 'file', ?)",
            params![link.as_str(), content_id],
        )?;
        
        self.conn.execute(
            "UPDATE contents SET ref_count = ref_count + 1 WHERE id = ?",
            params![content_id],
        )?;
        
        Ok(())
    }
}
```

**Key point:** `SqliteBackend` stores everything in a `.db` file. The "filesystem" is a set of database tables. Symlinks are rows with `entry_type='symlink'`. Hard links are multiple rows sharing the same `content_id`. No OS filesystem structures are created.

---

## VRootFsBackend: The Only One That Touches Real Filesystem

```rust
use strict_path::{VirtualRoot, VirtualPath as StrictVirtualPath};

pub struct VRootFsBackend {
    vroot: VirtualRoot,
}

impl VfsBackend for VRootFsBackend {
    fn write(&mut self, path: &VirtualPath, data: &[u8]) -> Result<(), VfsError> {
        // THIS actually writes to the real filesystem
        let vpath = self.vroot.virtual_join(path.as_str())?;
        vpath.write(data)?;  // strict-path's VirtualPath has built-in I/O
        Ok(())
    }
    
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        // THIS actually creates a real OS symlink
        let link_vpath = self.vroot.virtual_join(link.as_str())?;
        // Use std::os for real symlink creation
        #[cfg(unix)]
        std::os::unix::fs::symlink(original.as_str(), link_vpath.interop_os())?;
        Ok(())
    }
    
    fn hard_link(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError> {
        // THIS actually creates a real OS hard link
        let orig_vpath = self.vroot.virtual_join(original.as_str())?;
        let link_vpath = self.vroot.virtual_join(link.as_str())?;
        std::fs::hard_link(orig_vpath.interop_os(), link_vpath.interop_os())?;
        Ok(())
    }
}
```

**Key point:** `VRootFsBackend` is the **only** backend that interacts with the real OS filesystem. It uses `strict-path`'s `VirtualRoot` to ensure all operations stay within a designated directory. The symlinks and hard links it creates are real OS constructs.

---

## Why This Matters

### For API Users

When you write code like:

```rust
let mut backend = MemoryBackend::new();
backend.symlink("/target", "/link")?;
```

You are **not** creating a real symlink anywhere. You're manipulating an in-memory data structure that *behaves like* a filesystem with symlinks.

### For Backend Implementers

When implementing `VfsBackend`, you choose how to store data:

- **Want in-memory testing?** → Store in `HashMap`, simulate symlinks as stored paths
- **Want portable containers?** → Store in SQLite, simulate symlinks as rows
- **Want real filesystem?** → Delegate to OS calls (like `VRootFsBackend`)
- **Want cloud storage?** → Store in S3/GCS, simulate symlinks as metadata

The trait defines **what** operations exist. The backend defines **how** they're implemented.

---

## The Trait Is An Interface, Not An Implementation

```rust
pub trait VfsBackend: Send {
    /// Create symbolic link.
    /// 
    /// This is an API method. It does NOT imply a real OS symlink.
    /// Each backend decides how to represent symlinks:
    /// - MemoryBackend: Entry::Symlink { target }
    /// - SqliteBackend: Row with entry_type='symlink'
    /// - VRootFsBackend: Real OS symlink
    fn symlink(&mut self, original: &VirtualPath, link: &VirtualPath) -> Result<(), VfsError>;
}
```

---

## Summary Table

| Concept | Our API | MemoryBackend | SqliteBackend | VRootFsBackend |
|---------|---------|---------------|---------------|----------------|
| **File** | `write()` | `HashMap` entry | DB row + blob | Real file |
| **Directory** | `create_dir()` | `HashMap` entry | DB row | Real directory |
| **Symlink** | `symlink()` | Stored target path | DB row with target | Real OS symlink |
| **Hard link** | `hard_link()` | Shared `ContentId` | Shared `content_id` | Real OS hard link |
| **Read** | `read()` | HashMap lookup | SQL query | Real `fs::read()` |
| **Follows symlinks** | Yes (API contract) | HashMap resolution | SQL joins | OS resolution |

---

## Common Misconception

❌ **Wrong understanding:**
> "When I call `backend.symlink()`, it creates a symlink on my computer's filesystem."

✅ **Correct understanding:**
> "When I call `backend.symlink()`, it tells the backend to record a symlink. How that's stored depends on the backend. Only `VRootFsBackend` actually creates real OS symlinks."

---

## Analogy

Think of it like a database:

- `VfsBackend` trait = SQL interface (SELECT, INSERT, UPDATE)
- `MemoryBackend` = In-memory database (like SQLite `:memory:`)
- `SqliteBackend` = SQLite file database
- `VRootFsBackend` = Database that stores data as actual files

Just because SQL has `INSERT INTO files` doesn't mean it creates real files. Similarly, just because `VfsBackend` has `symlink()` doesn't mean it creates real symlinks.

---

## For Documentation Authors

When writing docs, be careful with language:

❌ **Avoid:**
- "Creates a symlink"
- "Writes to the filesystem"
- "Links the file"

✅ **Prefer:**
- "Records a symlink in the backend"
- "Stores data in the backend"
- "Creates a hard link entry"

Or be explicit:
- "Creates a symlink (real OS symlink for VRootFsBackend, simulated for others)"
