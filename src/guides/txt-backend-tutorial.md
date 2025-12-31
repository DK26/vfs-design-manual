# Tutorial: Building a TXT Backend (Yes, Really)

**How to turn a humble text file into a functioning virtual filesystem**

---

## The Absurd Premise

What if your entire filesystem was just... a text file you can edit in Notepad?

```
path,type,mode,data
/,dir,755,
/hello.txt,file,644,SGVsbG8sIFdvcmxkIQ==
/docs,dir,755,
/docs/readme.md,file,644,IyBXZWxjb21lIQoKWWVzLCB0aGlzIGlzIGluIGEgLnR4dCBmaWxl
```

One line per file. Comma-separated. Base64 content. Open it in Notepad, edit a file, save, done.

Sounds ridiculous? It is. But it *works*. And building it teaches you everything about implementing AnyFS backends.

Let's do this.

---

## Why This Is Actually Useful

Beyond the memes, a TXT backend demonstrates:

1. **Backend flexibility** - AnyFS doesn't care how you store bytes
2. **Trait implementation** - You'll implement `FsRead`, `FsWrite`, `FsDir`
3. **Middleware composition** - We'll add `Quota` to prevent the file from exploding
4. **Real-world patterns** - The same patterns apply to serious backends
5. **Separation of concerns** - Backends just store bytes; FileStorage handles path resolution

Plus, you can literally edit your "filesystem" in Notepad. Try doing that with ext4.

> **Important:** Backends receive **already-resolved paths** from FileStorage. You don't need to handle `..`, symlinks, or normalization - that's FileStorage's job. Your backend just stores and retrieves bytes at the given paths.

---

## The Format

One line per entry. Four comma-separated fields. Dead simple:

```
path,type,mode,data
```

| Field | Description | Example |
|-------|-------------|---------|
| `path` | Absolute path | `/docs/file.txt` |
| `type` | `file` or `dir` | `file` |
| `mode` | Unix permissions (octal) | `644` |
| `data` | Base64-encoded content | `SGVsbG8=` |

Directories have empty data field. That's the entire format. Open in Notepad, add a line, you created a file.

---

## Step 1: Data Structures

```rust
use std::collections::HashMap;
use std::path::{Path, PathBuf};
use base64::{Engine as _, engine::general_purpose::STANDARD as BASE64};

/// A single entry in our TXT filesystem
#[derive(Clone, Debug)]
struct TxtEntry {
    path: PathBuf,
    is_dir: bool,
    mode: u32,
    content: Vec<u8>,
}

impl TxtEntry {
    fn new_dir(path: impl Into<PathBuf>) -> Self {
        Self {
            path: path.into(),
            is_dir: true,
            mode: 0o755,
            content: Vec::new(),
        }
    }

    fn new_file(path: impl Into<PathBuf>, content: Vec<u8>) -> Self {
        Self {
            path: path.into(),
            is_dir: false,
            mode: 0o644,
            content,
        }
    }

    /// Serialize to a line: path,type,mode,data
    fn to_line(&self) -> String {
        let file_type = if self.is_dir { "dir" } else { "file" };
        let data_b64 = if self.content.is_empty() {
            String::new()
        } else {
            BASE64.encode(&self.content)
        };

        format!("{},{},{:o},{}", self.path.display(), file_type, self.mode, data_b64)
    }

    /// Parse from line: path,type,mode,data
    fn from_line(line: &str) -> Result<Self, TxtParseError> {
        let parts: Vec<&str> = line.splitn(4, ',').collect();
        if parts.len() < 3 {
            return Err(TxtParseError::InvalidFormat);
        }

        let content = if parts.len() == 4 && !parts[3].is_empty() {
            BASE64.decode(parts[3]).map_err(|_| TxtParseError::InvalidBase64)?
        } else {
            Vec::new()
        };

        Ok(Self {
            path: PathBuf::from(parts[0]),
            is_dir: parts[1] == "dir",
            mode: u32::from_str_radix(parts[2], 8)
                .map_err(|_| TxtParseError::InvalidNumber)?,
            content,
        })
    }
}

#[derive(Debug)]
enum TxtParseError {
    InvalidFormat,
    InvalidBase64,
    InvalidNumber,
}
```

---

## Step 2: The Backend Structure

```rust
use std::sync::{Arc, RwLock};
use std::fs::File;
use std::io::{BufRead, BufReader, Write};

/// A filesystem backend that stores everything in a .txt file.
///
/// Yes, this is cursed. Yes, it works. Yes, you can edit it in Notepad.
pub struct TxtBackend {
    /// Path to the .txt file on the host filesystem
    txt_path: PathBuf,
    /// In-memory cache of entries (path -> entry)
    entries: Arc<RwLock<HashMap<PathBuf, TxtEntry>>>,
}

impl TxtBackend {
    /// Create a new TXT backend, loading from file if it exists
    pub fn open(txt_path: impl Into<PathBuf>) -> Result<Self, FsError> {
        let txt_path = txt_path.into();
        let mut entries = HashMap::new();

        // Always ensure root directory exists
        entries.insert(PathBuf::from("/"), TxtEntry::new_dir("/"));

        // Load existing entries if file exists
        if txt_path.exists() {
            let file = File::open(&txt_path)
                .map_err(|e| FsError::Io {
                    operation: "open txt",
                    path: txt_path.clone(),
                    source: e,
                })?;

            for (line_num, line) in BufReader::new(file).lines().enumerate() {
                let line = line.map_err(|e| FsError::Io {
                    operation: "read line",
                    path: txt_path.clone(),
                    source: e,
                })?;

                // Skip header line
                if line_num == 0 && line.starts_with("path,") {
                    continue;
                }

                // Skip empty lines
                if line.trim().is_empty() {
                    continue;
                }

                let entry = TxtEntry::from_line(&line)
                    .map_err(|_| FsError::CorruptedData {
                        path: txt_path.clone(),
                        details: format!("line {}", line_num + 1),
                    })?;

                entries.insert(entry.path.clone(), entry);
            }
        }

        Ok(Self {
            txt_path,
            entries: Arc::new(RwLock::new(entries)),
        })
    }

    /// Create a new in-memory backend (won't persist to disk)
    pub fn in_memory() -> Self {
        let mut entries = HashMap::new();
        entries.insert(PathBuf::from("/"), TxtEntry::new_dir("/"));

        Self {
            txt_path: PathBuf::from(":memory:"),
            entries: Arc::new(RwLock::new(entries)),
        }
    }

    /// Flush all entries to the .txt file
    fn flush(&self) -> Result<(), FsError> {
        // Skip if in-memory mode
        if self.txt_path.as_os_str() == ":memory:" {
            return Ok(());
        }

        let entries = self.entries.read().unwrap();

        let mut file = File::create(&self.txt_path)
            .map_err(|e| FsError::Io {
                operation: "create txt",
                path: self.txt_path.clone(),
                source: e,
            })?;

        // Write header
        writeln!(file, "path,type,mode,data")
            .map_err(|e| FsError::Io {
                operation: "write header",
                path: self.txt_path.clone(),
                source: e,
            })?;

        // Write entries (sorted for consistency)
        let mut paths: Vec<_> = entries.keys().collect();
        paths.sort();

        for path in paths {
            let entry = &entries[path];
            writeln!(file, "{}", entry.to_line())
                .map_err(|e| FsError::Io {
                    operation: "write entry",
                    path: path.clone(),
                    source: e,
                })?;
        }

        Ok(())
    }

}
```

---

## Step 3: Implement FsRead

Now the fun part - making it quack like a filesystem:

```rust
use anyfs_backend::{FsRead, FsError, Metadata, FileType};

impl FsRead for TxtBackend {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();
        let entries = self.entries.read().unwrap();

        let entry = entries.get(&path)
            .ok_or_else(|| FsError::NotFound { path: path.clone() })?;

        if entry.is_dir {
            return Err(FsError::NotAFile { path });
        }

        Ok(entry.content.clone())
    }

    fn read_to_string(&self, path: &Path) -> Result<String, FsError> {
        let bytes = self.read(path.as_ref())?;
        String::from_utf8(bytes)
            .map_err(|_| FsError::InvalidData {
                path: path.as_ref().to_path_buf(),
                details: "not valid UTF-8".to_string(),
            })
    }

    fn read_range(&self, path: &Path, offset: u64, len: usize) -> Result<Vec<u8>, FsError> {
        let content = self.read(path)?;
        let start = offset as usize;

        if start >= content.len() {
            return Ok(Vec::new());
        }

        let end = (start + len).min(content.len());
        Ok(content[start..end].to_vec())
    }

    fn exists(&self, path: &Path) -> Result<bool, FsError> {
        let path = path.as_ref();
        let entries = self.entries.read().unwrap();
        Ok(entries.contains_key(path))
    }

    fn metadata(&self, path: &Path) -> Result<Metadata, FsError> {
        let path = path.as_ref();
        let entries = self.entries.read().unwrap();

        let entry = entries.get(path)
            .ok_or_else(|| FsError::NotFound { path: path.to_path_buf() })?;

        Ok(Metadata {
            file_type: if entry.is_dir { FileType::Directory } else { FileType::File },
            size: entry.content.len() as u64,
            created: None,  // We keep it simple - no timestamps
            modified: None,
            accessed: None,
            permissions: Some(entry.mode),
            inode: None,
        })
    }

    fn open_read(&self, path: &Path) -> Result<Box<dyn std::io::Read + Send>, FsError> {
        let content = self.read(path)?;
        Ok(Box::new(std::io::Cursor::new(content)))
    }
}
```

---

## Step 4: Implement FsWrite

Where the magic happens - writing files to a text file:

```rust
use anyfs_backend::FsWrite;

impl FsWrite for TxtBackend {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();

        // Ensure parent directory exists
        if let Some(parent) = path.parent() {
            let parent_str = parent.to_string_lossy();
            if parent_str != "/" && !parent_str.is_empty() {
                let entries = self.entries.read().unwrap();
                if !entries.contains_key(parent) {
                    drop(entries);
                    return Err(FsError::NotFound {
                        path: parent.to_path_buf()
                    });
                }
            }
        }

        let mut entries = self.entries.write().unwrap();

        // Check if it's a directory
        if let Some(existing) = entries.get(&path) {
            if existing.is_dir {
                return Err(FsError::NotAFile { path });
            }
        }

        // Create or update the file
        let entry = TxtEntry::new_file(path.clone(), data.to_vec());
        entries.insert(path, entry);

        drop(entries);
        self.flush()?;

        Ok(())
    }

    fn append(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();
        let mut entries = self.entries.write().unwrap();

        let entry = entries.get_mut(&path)
            .ok_or_else(|| FsError::NotFound { path: path.clone() })?;

        if entry.is_dir {
            return Err(FsError::NotAFile { path });
        }

        entry.content.extend_from_slice(data);

        drop(entries);
        self.flush()?;

        Ok(())
    }

    fn remove_file(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();
        let mut entries = self.entries.write().unwrap();

        let entry = entries.get(&path)
            .ok_or_else(|| FsError::NotFound { path: path.clone() })?;

        if entry.is_dir {
            return Err(FsError::NotAFile { path });
        }

        entries.remove(&path);

        drop(entries);
        self.flush()?;

        Ok(())
    }

    fn rename(&self, from: &Path, to: &Path) -> Result<(), FsError> {
        let from = from.as_ref().to_path_buf();
        let to = to.as_ref().to_path_buf();

        let mut entries = self.entries.write().unwrap();

        let mut entry = entries.remove(&from)
            .ok_or_else(|| FsError::NotFound { path: from.clone() })?;

        entry.path = to.clone();
        entries.insert(to, entry);

        drop(entries);
        self.flush()?;

        Ok(())
    }

    fn copy(&self, from: &Path, to: &Path) -> Result<(), FsError> {
        let from = from.as_ref().to_path_buf();
        let to = to.as_ref().to_path_buf();

        let entries = self.entries.read().unwrap();

        let source = entries.get(&from)
            .ok_or_else(|| FsError::NotFound { path: from.clone() })?;

        if source.is_dir {
            return Err(FsError::NotAFile { path: from });
        }

        let mut new_entry = source.clone();
        new_entry.path = to.clone();

        drop(entries);

        let mut entries = self.entries.write().unwrap();
        entries.insert(to, new_entry);

        drop(entries);
        self.flush()?;

        Ok(())
    }

    fn truncate(&self, path: &Path, size: u64) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();
        let mut entries = self.entries.write().unwrap();

        let entry = entries.get_mut(&path)
            .ok_or_else(|| FsError::NotFound { path: path.clone() })?;

        if entry.is_dir {
            return Err(FsError::NotAFile { path });
        }

        entry.content.truncate(size as usize);

        drop(entries);
        self.flush()?;

        Ok(())
    }

    fn open_write(&self, path: &Path) -> Result<Box<dyn std::io::Write + Send>, FsError> {
        // For simplicity, we buffer writes and apply on drop
        // A real implementation would be more sophisticated
        let path = path.as_ref().to_path_buf();

        // Ensure file exists (create empty if not)
        if !self.exists(&path)? {
            self.write(&path, b"")?;
        }

        Ok(Box::new(TxtFileWriter {
            backend: self.entries.clone(),
            txt_path: self.txt_path.clone(),
            path,
            buffer: Vec::new(),
        }))
    }
}

/// Writer that buffers content and writes to TXT on drop
struct TxtFileWriter {
    backend: Arc<RwLock<HashMap<PathBuf, TxtEntry>>>,
    txt_path: PathBuf,
    path: PathBuf,
    buffer: Vec<u8>,
}

impl std::io::Write for TxtFileWriter {
    fn write(&mut self, buf: &[u8]) -> std::io::Result<usize> {
        self.buffer.extend_from_slice(buf);
        Ok(buf.len())
    }

    fn flush(&mut self) -> std::io::Result<()> {
        Ok(())
    }
}

impl Drop for TxtFileWriter {
    fn drop(&mut self) {
        let mut entries = self.backend.write().unwrap();
        if let Some(entry) = entries.get_mut(&self.path) {
            entry.content = std::mem::take(&mut self.buffer);
        }
        // Note: flush to disk happens on next explicit flush() call
    }
}
```

---

## Step 5: Implement FsDir

Directory operations to complete the `Fs` trait:

```rust
use anyfs_backend::{FsDir, DirEntry};

impl FsDir for TxtBackend {
    fn read_dir(&self, path: &Path) -> Result<ReadDirIter, FsError> {
        let path = path.as_ref().to_path_buf();
        let entries = self.entries.read().unwrap();

        // Verify the path is a directory
        let entry = entries.get(&path)
            .ok_or_else(|| FsError::NotFound { path: path.clone() })?;

        if !entry.is_dir {
            return Err(FsError::NotADirectory { path });
        }

        // Find all direct children
        let mut children = Vec::new();

        for (child_path, child_entry) in entries.iter() {
            if let Some(parent) = child_path.parent() {
                if parent == path && child_path != &path {
                    children.push(DirEntry {
                        name: child_path.file_name()
                            .unwrap_or_default()
                            .to_string_lossy()
                            .into_owned(),
                        path: child_path.clone(),
                        file_type: if child_entry.is_dir {
                            FileType::Directory
                        } else {
                            FileType::File
                        },
                        size: child_entry.size,
                        inode: None,
                    });
                }
            }
        }

        // Sort for consistent ordering
        children.sort_by(|a, b| a.name.cmp(&b.name));

        // Wrap in ReadDirIter (items are Ok since we've already validated them)
        Ok(ReadDirIter::new(children.into_iter().map(Ok)))
    }

    fn create_dir(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();

        // Check parent exists
        if let Some(parent) = path.parent() {
            let parent_str = parent.to_string_lossy();
            if parent_str != "/" && !parent_str.is_empty() {
                let entries = self.entries.read().unwrap();
                let parent_entry = entries.get(parent)
                    .ok_or_else(|| FsError::NotFound {
                        path: parent.to_path_buf()
                    })?;

                if !parent_entry.is_dir {
                    return Err(FsError::NotADirectory {
                        path: parent.to_path_buf()
                    });
                }
            }
        }

        let mut entries = self.entries.write().unwrap();

        // Check if already exists
        if entries.contains_key(&path) {
            return Err(FsError::AlreadyExists { path, operation: "create_dir" });
        }

        entries.insert(path.clone(), TxtEntry::new_dir(path));

        drop(entries);
        self.flush()?;

        Ok(())
    }

    fn create_dir_all(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();

        // Build list of directories to create
        let mut to_create = Vec::new();
        let mut current = path.clone();

        loop {
            {
                let entries = self.entries.read().unwrap();
                if entries.contains_key(&current) {
                    break;
                }
            }

            to_create.push(current.clone());

            match current.parent() {
                Some(parent) if !parent.as_os_str().is_empty() => {
                    current = parent.to_path_buf();
                }
                _ => break,
            }
        }

        // Create directories from root to leaf
        to_create.reverse();
        for dir_path in to_create {
            let mut entries = self.entries.write().unwrap();
            if !entries.contains_key(&dir_path) {
                entries.insert(dir_path.clone(), TxtEntry::new_dir(dir_path));
            }
        }

        self.flush()?;
        Ok(())
    }

    fn remove_dir(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();

        // Can't remove root
        if path.to_string_lossy() == "/" {
            return Err(FsError::PermissionDenied {
                path,
                operation: "remove root directory"
            });
        }

        let entries = self.entries.read().unwrap();

        let entry = entries.get(&path)
            .ok_or_else(|| FsError::NotFound { path: path.clone() })?;

        if !entry.is_dir {
            return Err(FsError::NotADirectory { path });
        }

        // Check if empty
        let has_children = entries.keys().any(|p| {
            p != &path && p.starts_with(&path)
        });

        if has_children {
            return Err(FsError::DirectoryNotEmpty { path });
        }

        drop(entries);

        let mut entries = self.entries.write().unwrap();
        entries.remove(&path);

        drop(entries);
        self.flush()?;

        Ok(())
    }

    fn remove_dir_all(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref().to_path_buf();

        // Can't remove root
        if path.to_string_lossy() == "/" {
            return Err(FsError::PermissionDenied {
                path,
                operation: "remove root directory"
            });
        }

        let mut entries = self.entries.write().unwrap();

        // Verify it exists and is a directory
        let entry = entries.get(&path)
            .ok_or_else(|| FsError::NotFound { path: path.clone() })?;

        if !entry.is_dir {
            return Err(FsError::NotADirectory { path: path.clone() });
        }

        // Remove all entries under this path
        let to_remove: Vec<_> = entries.keys()
            .filter(|p| p.starts_with(&path))
            .cloned()
            .collect();

        for p in to_remove {
            entries.remove(&p);
        }

        drop(entries);
        self.flush()?;

        Ok(())
    }
}
```

---

## Step 6: Putting It All Together

Now you have a complete `Fs` implementation! Let's use it:

```rust
use anyfs::{FileStorage, QuotaLayer, TracingLayer};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create our glorious TXT filesystem
    let backend = TxtBackend::open("my_filesystem.txt")?
        // Wrap it with middleware to prevent the file from exploding
        .layer(QuotaLayer::builder()
            .max_total_size(10 * 1024 * 1024)  // 10 MB max
            .max_file_size(1 * 1024 * 1024)    // 1 MB per file
            .build())
        // Add tracing because why not
        .layer(TracingLayer::new());

    // Create the filesystem wrapper
    let fs = FileStorage::new(backend);

    // Use it like any other filesystem!
    fs.create_dir_all("/projects/secret")?;
    fs.write("/projects/secret/plans.txt", b"World domination via TXT")?;
    fs.write("/projects/readme.md", b"# My TXT-backed project\n\nYes, really.")?;

    // Read it back
    let content = fs.read_to_string("/projects/secret/plans.txt")?;
    println!("Plans: {}", content);

    // List directory
    for entry in fs.read_dir("/projects")? {
        println!("  {} ({})", entry.name,
            if entry.file_type == FileType::Directory { "dir" } else { "file" });
    }

    // Copy a file
    fs.copy("/projects/readme.md", "/projects/readme_backup.md")?;

    // Delete a file
    fs.remove_file("/projects/readme_backup.md")?;

    println!("\nNow open my_filesystem.txt in Notepad!");

    Ok(())
}
```

---

## The Result

After running the code, your `my_filesystem.txt` looks like:

```
path,type,mode,data
/,dir,755,
/projects,dir,755,
/projects/secret,dir,755,
/projects/secret/plans.txt,file,644,V29ybGQgZG9taW5hdGlvbiB2aWEgVFhU
/projects/readme.md,file,644,IyBNeSBUWFQtYmFja2VkIHByb2plY3QKCllzLCByZWFsbHku
```

Open it in Notepad. Marvel at your filesystem. Edit a line. Save. You just modified a file.

---

## Why This Actually Matters

This ridiculous example demonstrates the power of AnyFS's design:

1. **True backend abstraction** - The `FileStorage` API doesn't know or care that it's backed by a text file

2. **Middleware just works** - `Quota` and `Tracing` wrap your custom backend with zero extra code

3. **Type safety preserved** - Compile-time guarantees work with any backend

4. **Easy to implement** - ~250 lines for a complete working backend

5. **Testable** - Use `TxtBackend::in_memory()` for fast tests

6. **Human-editable** - Open in Notepad, add a line, you created a file

---

## Next Steps

If you're feeling brave:

1. **Add symlink support** - Implement `FsLink` trait
2. **Make it async** - Wrap with `tokio::fs` for the host CSV file
3. **Add compression** - Gzip the base64 content
4. **Excel integration** - Add formulas that compute file sizes (why not?)

---

## Bonus: Mount It as a Drive (Planned)

With the planned `anyfs-mount` companion crate, you can mount your text file as a real filesystem:

```rust
use anyfs_mount::MountHandle;

let backend = TxtBackend::open("filesystem.txt")?;
let mount = MountHandle::mount(backend, "/mnt/txt")?;

// Now /mnt/txt is a real mount point backed by a .txt file
// Any application can read/write files there
// The data goes into a text file you can edit in Notepad
// This is fine
```

---

## The Moral

AnyFS doesn't care where bytes come from or where they go. Memory, SQLite, a text file, a REST API, carrier pigeons with USB drives - if you can implement the traits, it's a valid backend.

The middleware layer (quotas, sandboxing, rate limiting, logging) works transparently on any backend. That's the power of good abstractions.

Now go build something less cursed. Or don't. I'm not your supervisor.

---

*"I store my production data in text files" - Nobody, ever (until now)*

*"Can I edit my filesystem in Notepad?" - Yes. Yes you can.*

