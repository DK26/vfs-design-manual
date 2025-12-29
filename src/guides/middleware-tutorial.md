# Tutorial: Building Your First Middleware

**From zero to intercepting filesystem operations in 15 minutes**

---

## What is Middleware?

Middleware wraps a backend and intercepts operations. That's it.

```
User Request → [Your Middleware] → [Backend] → Storage
              ↑                  ↓
              └── intercept ─────┘
```

You can:
- **Block** operations (ReadOnly, PathFilter)
- **Transform** data (Encryption, Compression)
- **Count/Log** operations (Counter, Tracing)
- **Enforce limits** (Quota, RateLimit)

Let's build one.

---

## The Simplest Middleware: Operation Counter

We'll count every operation. That's our entire goal.

### Step 1: The Struct

```rust
use std::sync::atomic::{AtomicU64, Ordering};

/// Counts every operation performed on the wrapped backend.
pub struct Counter<B> {
    inner: B,                    // The backend we're wrapping
    pub count: AtomicU64,        // Our counter
}

impl<B> Counter<B> {
    pub fn new(inner: B) -> Self {
        Self {
            inner,
            count: AtomicU64::new(0),
        }
    }

    pub fn operations(&self) -> u64 {
        self.count.load(Ordering::Relaxed)
    }
}
```

That's the entire struct. We wrap something (`inner`) and add our state (`count`).

### Step 2: Implement FsRead

Now we implement the same traits as the inner backend, intercepting each method:

```rust
use anyfs_backend::{FsRead, FsError, Metadata};
use std::path::Path;

impl<B: FsRead> FsRead for Counter<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);  // COUNT IT
        self.inner.read(path)                         // DELEGATE
    }

    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.read_to_string(path)
    }

    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.read_range(path, offset, len)
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.exists(path)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.metadata(path)
    }

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn std::io::Read + Send>, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.open_read(path)
    }
}
```

The pattern is always the same:
1. Do your thing (count)
2. Call `self.inner.method(args)` (delegate)

### Step 3: Implement FsWrite

Same pattern:

```rust
use anyfs_backend::FsWrite;

impl<B: FsWrite> FsWrite for Counter<B> {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.write(path, data)
    }

    fn append(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.append(path, data)
    }

    fn remove_file(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.remove_file(path)
    }

    fn rename(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.rename(from, to)
    }

    fn copy(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.copy(from, to)
    }

    fn truncate(&self, path: impl AsRef<Path>, size: u64) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.truncate(path, size)
    }

    fn open_write(&self, path: impl AsRef<Path>) -> Result<Box<dyn std::io::Write + Send>, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.open_write(path)
    }
}
```

### Step 4: Implement FsDir

```rust
use anyfs_backend::{FsDir, ReadDirIter};

impl<B: FsDir> FsDir for Counter<B> {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<ReadDirIter, FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.read_dir(path)
    }

    fn create_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.create_dir(path)
    }

    fn create_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.create_dir_all(path)
    }

    fn remove_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.remove_dir(path)
    }

    fn remove_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        self.count.fetch_add(1, Ordering::Relaxed);
        self.inner.remove_dir_all(path)
    }
}

// Counter<B> now implements Fs when B: Fs (blanket impl)!
```

### Step 5: Use It

```rust
use anyfs::MemoryBackend;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let fs = Counter::new(MemoryBackend::new());

    fs.write("/hello.txt", b"Hello, World!")?;
    fs.read("/hello.txt")?;
    fs.read("/hello.txt")?;
    fs.exists("/hello.txt")?;

    println!("Total operations: {}", fs.operations());  // 4

    Ok(())
}
```

**That's it.** You built middleware.

---

## Adding .layer() Support

Want the fluent `.layer()` syntax? Add a Layer struct:

```rust
use anyfs_backend::{Layer, Fs};

/// Layer for creating Counter middleware.
pub struct CounterLayer;

impl<B: Fs> Layer<B> for CounterLayer {
    type Backend = Counter<B>;

    fn layer(self, backend: B) -> Counter<B> {
        Counter::new(backend)
    }
}
```

Now you can do:

```rust
let fs = MemoryBackend::new()
    .layer(CounterLayer);

fs.write("/test.txt", b"data")?;
println!("Operations: {}", fs.operations());
```

---

## A More Useful Middleware: SecretBlocker

Let's build something practical - block access to files matching a pattern:

```rust
use anyfs_backend::{FsRead, FsWrite, FsDir, FsError, Metadata, ReadDirIter};
use std::path::Path;

/// Blocks access to files containing "secret" in the path.
pub struct SecretBlocker<B> {
    inner: B,
}

impl<B> SecretBlocker<B> {
    pub fn new(inner: B) -> Self {
        Self { inner }
    }

    /// Check if path is forbidden.
    fn is_secret(&self, path: &Path) -> bool {
        path.to_string_lossy().to_lowercase().contains("secret")
    }

    /// Return error if path is secret.
    fn check(&self, path: &Path) -> Result<(), FsError> {
        if self.is_secret(path) {
            Err(FsError::AccessDenied {
                path: path.to_path_buf(),
                reason: "secret files are blocked".to_string(),
            })
        } else {
            Ok(())
        }
    }
}
```

### Implement the Traits

```rust
impl<B: FsRead> FsRead for SecretBlocker<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();
        self.check(path)?;           // BLOCK if secret
        self.inner.read(path)        // DELEGATE otherwise
    }

    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.read_to_string(path)
    }

    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.read_range(path, offset, len)
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.exists(path)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.metadata(path)
    }

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn std::io::Read + Send>, FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.open_read(path)
    }
}

impl<B: FsWrite> FsWrite for SecretBlocker<B> {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.write(path, data)
    }

    fn append(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.append(path, data)
    }

    fn remove_file(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.remove_file(path)
    }

    fn rename(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError> {
        let from = from.as_ref();
        let to = to.as_ref();
        self.check(from)?;
        self.check(to)?;  // Block both source and destination
        self.inner.rename(from, to)
    }

    fn copy(&self, from: impl AsRef<Path>, to: impl AsRef<Path>) -> Result<(), FsError> {
        let from = from.as_ref();
        let to = to.as_ref();
        self.check(from)?;
        self.check(to)?;
        self.inner.copy(from, to)
    }

    fn truncate(&self, path: impl AsRef<Path>, size: u64) -> Result<(), FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.truncate(path, size)
    }

    fn open_write(&self, path: impl AsRef<Path>) -> Result<Box<dyn std::io::Write + Send>, FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.open_write(path)
    }
}

impl<B: FsDir> FsDir for SecretBlocker<B> {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<ReadDirIter, FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.read_dir(path)
    }

    fn create_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.create_dir(path)
    }

    fn create_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.create_dir_all(path)
    }

    fn remove_dir(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.remove_dir(path)
    }

    fn remove_dir_all(&self, path: impl AsRef<Path>) -> Result<(), FsError> {
        let path = path.as_ref();
        self.check(path)?;
        self.inner.remove_dir_all(path)
    }
}
```

### Use It

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let fs = SecretBlocker::new(MemoryBackend::new());

    // These work fine
    fs.write("/public/data.txt", b"Hello!")?;
    fs.read("/public/data.txt")?;

    // These are blocked
    assert!(fs.write("/secret/passwords.txt", b"hunter2").is_err());
    assert!(fs.read("/my-secret-diary.txt").is_err());
    assert!(fs.create_dir("/SECRET").is_err());

    println!("Secret files successfully blocked!");
    Ok(())
}
```

---

## The Middleware Pattern Cheat Sheet

| What You Want | Intercept | Delegate | Return |
|---------------|-----------|----------|--------|
| Count operations | Before call | Always | Inner result |
| Block some paths | Before call | If allowed | Error or inner result |
| Block writes | Write methods | Read methods | Error or inner result |
| Transform data | read/write | Everything else | Modified data |
| Log operations | Before/after | Always | Inner result |

### Three Types of Middleware

**1. Pass-through with side effects** (Counter, Logger)
```rust
fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
    log::info!("Reading: {:?}", path.as_ref());  // Side effect
    self.inner.read(path)                         // Always delegate
}
```

**2. Conditional blocking** (PathFilter, ReadOnly)
```rust
fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
    if self.is_blocked(path.as_ref()) {
        return Err(FsError::AccessDenied { ... });  // Block
    }
    self.inner.write(path, data)                    // Allow
}
```

**3. Data transformation** (Encryption, Compression)
```rust
fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
    let encrypted = self.inner.read(path)?;  // Get data
    Ok(self.decrypt(&encrypted))              // Transform
}

fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
    let encrypted = self.encrypt(data);       // Transform
    self.inner.write(path, &encrypted)        // Store
}
```

---

## Complete Example: ReadOnly Middleware

The classic - block all writes:

```rust
use anyfs_backend::{FsRead, FsWrite, FsDir, FsError, Metadata, ReadDirIter, Layer, Fs};
use std::path::Path;

/// Makes any backend read-only.
pub struct ReadOnly<B> {
    inner: B,
}

impl<B> ReadOnly<B> {
    pub fn new(inner: B) -> Self {
        Self { inner }
    }
}

// FsRead: delegate everything
impl<B: FsRead> FsRead for ReadOnly<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        self.inner.read(path)
    }

    fn read_to_string(&self, path: impl AsRef<Path>) -> Result<String, FsError> {
        self.inner.read_to_string(path)
    }

    fn read_range(&self, path: impl AsRef<Path>, offset: u64, len: usize) -> Result<Vec<u8>, FsError> {
        self.inner.read_range(path, offset, len)
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        self.inner.exists(path)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        self.inner.metadata(path)
    }

    fn open_read(&self, path: impl AsRef<Path>) -> Result<Box<dyn std::io::Read + Send>, FsError> {
        self.inner.open_read(path)
    }
}

// FsWrite: block everything
impl<B: FsWrite> FsWrite for ReadOnly<B> {
    fn write(&self, _: impl AsRef<Path>, _: &[u8]) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "write" })
    }

    fn append(&self, _: impl AsRef<Path>, _: &[u8]) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "append" })
    }

    fn remove_file(&self, _: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_file" })
    }

    fn rename(&self, _: impl AsRef<Path>, _: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "rename" })
    }

    fn copy(&self, _: impl AsRef<Path>, _: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "copy" })
    }

    fn truncate(&self, _: impl AsRef<Path>, _: u64) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "truncate" })
    }

    fn open_write(&self, _: impl AsRef<Path>) -> Result<Box<dyn std::io::Write + Send>, FsError> {
        Err(FsError::ReadOnly { operation: "open_write" })
    }
}

// FsDir: delegate reads, block writes
impl<B: FsDir> FsDir for ReadOnly<B> {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<ReadDirIter, FsError> {
        self.inner.read_dir(path)  // Reading is OK
    }

    fn create_dir(&self, _: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "create_dir" })
    }

    fn create_dir_all(&self, _: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "create_dir_all" })
    }

    fn remove_dir(&self, _: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_dir" })
    }

    fn remove_dir_all(&self, _: impl AsRef<Path>) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_dir_all" })
    }
}

// Layer for .layer() syntax
pub struct ReadOnlyLayer;

impl<B: Fs> Layer<B> for ReadOnlyLayer {
    type Backend = ReadOnly<B>;

    fn layer(self, backend: B) -> Self::Backend {
        ReadOnly::new(backend)
    }
}
```

### Usage

```rust
let fs = MemoryBackend::new()
    .layer(ReadOnlyLayer);

// Reads work
fs.exists("/anything")?;

// Writes fail
assert!(fs.write("/file.txt", b"data").is_err());
assert!(fs.create_dir("/new").is_err());
```

---

## Stacking Middleware

Middleware composes naturally:

```rust
let fs = MemoryBackend::new()
    .layer(SecretBlockerLayer)      // Block secret files
    .layer(ReadOnlyLayer)           // Make read-only
    .layer(CounterLayer);           // Count operations

// Order matters! Outermost runs first.
// Request: Counter → ReadOnly → SecretBlocker → MemoryBackend
```

---

## Middleware Checklist

Before publishing your middleware:

- [ ] Depends only on `anyfs-backend`
- [ ] Implements same traits as inner backend (`FsRead`, `FsWrite`, `FsDir`)
- [ ] Has a `Layer` implementation for `.layer()` syntax
- [ ] Documents which operations are intercepted vs delegated
- [ ] Handles errors properly (doesn't panic)
- [ ] Is thread-safe (`&self` methods, use atomics/locks for state)

---

## Summary

**Middleware is just:**
1. A struct wrapping `inner: B`
2. Implementing the same traits as `B`
3. Intercepting some methods, delegating others

**The three patterns:**
1. **Side effects:** Do something, then delegate
2. **Blocking:** Check condition, return error or delegate
3. **Transform:** Modify data on the way in/out

**That's it.** Go build something useful.

---

*"Middleware: because sometimes you need to do something between nothing and everything."*
