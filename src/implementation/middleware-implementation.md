# Middleware Implementation Guide

This document provides implementation sketches for all AnyFS middleware, verifying that each is implementable within our framework.

**Verdict: All 9 middleware are implementable.** Some have interesting challenges documented below.

---

## Implementation Pattern

All middleware follow the same pattern:

```rust
pub struct MiddlewareName<B> {
    inner: B,
    state: MiddlewareState,  // Interior mutability if needed
}

impl<B: Fs> FsRead for MiddlewareName<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        // 1. Pre-check (validate, log, check limits)
        // 2. Delegate to inner.read(path)
        // 3. Post-process (update state, transform result)
    }
}

// Implement FsWrite, FsDir similarly...
// Blanket impl for Fs is automatic
```

---

## 1. ReadOnly<B>

**Complexity:** Trivial
**State:** None
**Dependencies:** None

### Implementation

```rust
pub struct ReadOnly<B> {
    inner: B,
}

impl<B> ReadOnly<B> {
    pub fn new(inner: B) -> Self {
        Self { inner }
    }
}

impl<B: FsRead> FsRead for ReadOnly<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        self.inner.read(path)  // Pass through
    }

    fn read_to_string(&self, path: &Path) -> Result<String, FsError> {
        self.inner.read_to_string(path)  // Pass through
    }

    fn read_range(&self, path: &Path, offset: u64, len: usize) -> Result<Vec<u8>, FsError> {
        self.inner.read_range(path, offset, len)  // Pass through
    }

    fn exists(&self, path: &Path) -> Result<bool, FsError> {
        self.inner.exists(path)  // Pass through
    }

    fn metadata(&self, path: &Path) -> Result<Metadata, FsError> {
        self.inner.metadata(path)  // Pass through
    }

    fn open_read(&self, path: &Path) -> Result<Box<dyn Read + Send>, FsError> {
        self.inner.open_read(path)  // Pass through
    }
}

impl<B: FsWrite> FsWrite for ReadOnly<B> {
    fn write(&self, _path: &Path, _data: &[u8]) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "write" })
    }

    fn append(&self, _path: &Path, _data: &[u8]) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "append" })
    }

    fn remove_file(&self, _path: &Path) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_file" })
    }

    fn rename(&self, _from: &Path, _to: &Path) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "rename" })
    }

    fn copy(&self, _from: &Path, _to: &Path) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "copy" })
    }

    fn truncate(&self, _path: &Path, _size: u64) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "truncate" })
    }

    fn open_write(&self, _path: &Path) -> Result<Box<dyn Write + Send>, FsError> {
        Err(FsError::ReadOnly { operation: "open_write" })
    }
}

impl<B: FsDir> FsDir for ReadOnly<B> {
    fn read_dir(&self, path: &Path) -> Result<ReadDirIter, FsError> {
        self.inner.read_dir(path)  // Pass through (reading)
    }

    fn create_dir(&self, _path: &Path) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "create_dir" })
    }

    fn create_dir_all(&self, _path: &Path) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "create_dir_all" })
    }

    fn remove_dir(&self, _path: &Path) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_dir" })
    }

    fn remove_dir_all(&self, _path: &Path) -> Result<(), FsError> {
        Err(FsError::ReadOnly { operation: "remove_dir_all" })
    }
}
```

### Verdict: ✅ Trivially Implementable

No challenges. Pure delegation for reads, error return for writes.

---

## 2. Restrictions<B>

**Complexity:** Simple
**State:** Configuration flags only
**Dependencies:** None

> **Note:** Symlink/hard-link capability is determined by trait bounds (`B: FsLink`), not middleware.
> Restrictions only controls permission-related operations.

### Implementation

```rust
pub struct Restrictions<B> {
    inner: B,
    deny_permissions: bool,
}

pub struct RestrictionsBuilder {
    deny_permissions: bool,
}

impl RestrictionsBuilder {
    pub fn deny_permissions(mut self) -> Self {
        self.deny_permissions = true;
        self
    }

    pub fn build<B>(self, inner: B) -> Restrictions<B> {
        Restrictions {
            inner,
            deny_permissions: self.deny_permissions,
        }
    }
}

// FsRead, FsDir, FsLink: pure delegation (Restrictions doesn't block these)

impl<B: FsLink> FsLink for Restrictions<B> {
    fn symlink(&self, target: &Path, link: &Path) -> Result<(), FsError> {
        self.inner.symlink(target, link)  // Pure delegation
    }

    fn hard_link(&self, original: &Path, link: &Path) -> Result<(), FsError> {
        self.inner.hard_link(original, link)  // Pure delegation
    }

    fn read_link(&self, path: &Path) -> Result<PathBuf, FsError> {
        self.inner.read_link(path)
    }

    fn symlink_metadata(&self, path: &Path) -> Result<Metadata, FsError> {
        self.inner.symlink_metadata(path)
    }
}

impl<B: FsPermissions> FsPermissions for Restrictions<B> {
    fn set_permissions(&self, path: &Path, perm: Permissions) -> Result<(), FsError> {
        if self.deny_permissions {
            return Err(FsError::FeatureNotEnabled {
                feature: "permissions",
                operation: "set_permissions",
            });
        }
        self.inner.set_permissions(path, perm)
    }
}
```

### Verdict: ✅ Trivially Implementable

Simple flag check on `set_permissions()`. Link operations delegate to inner backend.

---

## 3. Tracing<B>

**Complexity:** Simple
**State:** Configuration only
**Dependencies:** `tracing` crate

### Implementation

```rust
use tracing::{instrument, info, debug, Level};

pub struct Tracing<B> {
    inner: B,
    target: &'static str,
    level: Level,
}

impl<B: FsRead> FsRead for Tracing<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();
        let span = tracing::span!(Level::DEBUG, "fs::read", ?path);
        let _guard = span.enter();

        let result = self.inner.read(path);

        match &result {
            Ok(data) => debug!(bytes = data.len(), "read succeeded"),
            Err(e) => debug!(?e, "read failed"),
        }

        result
    }

    fn exists(&self, path: &Path) -> Result<bool, FsError> {
        let path = path.as_ref();
        let span = tracing::span!(Level::DEBUG, "fs::exists", ?path);
        let _guard = span.enter();

        let result = self.inner.exists(path);
        debug!(?result, "exists check");
        result
    }

    // ... similar for all other methods
}

// FsWrite and FsDir follow the same pattern
```

### Verdict: ✅ Trivially Implementable

Pure instrumentation wrapper. No state mutation, no complex logic.

---

## 4. RateLimit<B>

**Complexity:** Moderate
**State:** Counter + timestamp (requires interior mutability)
**Dependencies:** None (uses `std::time`)
**Algorithm:** Fixed-window counter (simpler than token bucket, sufficient for v1)

### Implementation

```rust
use std::sync::atomic::{AtomicU64, AtomicU32, Ordering};
use std::time::{Duration, Instant};
use std::sync::RwLock;

pub struct RateLimit<B> {
    inner: B,
    max_ops: u32,
    window: Duration,
    state: RwLock<RateLimitState>,
}

struct RateLimitState {
    window_start: Instant,
    count: u32,
}

impl<B> RateLimit<B> {
    fn check_rate_limit(&self) -> Result<(), FsError> {
        let mut state = self.state.write().unwrap();

        let now = Instant::now();
        if now.duration_since(state.window_start) >= self.window {
            // Window expired, reset
            state.window_start = now;
            state.count = 1;
            return Ok(());
        }

        if state.count >= self.max_ops {
            return Err(FsError::RateLimitExceeded {
                limit: self.max_ops,
                window_secs: self.window.as_secs(),
            });
        }

        state.count += 1;
        Ok(())
    }
}

impl<B: FsRead> FsRead for RateLimit<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        self.check_rate_limit()?;
        self.inner.read(path)
    }

    fn exists(&self, path: &Path) -> Result<bool, FsError> {
        self.check_rate_limit()?;
        self.inner.exists(path)
    }

    // ... all methods call check_rate_limit() first
}
```

### Considerations

- **Fixed window vs sliding window:** Fixed window is simpler and sufficient for most use cases.
- **Thread safety:** Uses `RwLock` for state. Could optimize with atomics for lock-free path.
- **What counts as an operation?** Each method call counts as 1 operation.

### Verdict: ✅ Implementable

Straightforward with interior mutability.

---

## 5. DryRun<B>

**Complexity:** Moderate
**State:** Operation log
**Dependencies:** None

### Implementation

```rust
use std::sync::RwLock;

pub struct DryRun<B> {
    inner: B,
    operations: RwLock<Vec<String>>,
}

impl<B> DryRun<B> {
    pub fn operations(&self) -> Vec<String> {
        self.operations.read().unwrap().clone()
    }

    fn log(&self, op: String) {
        self.operations.write().unwrap().push(op);
    }
}

impl<B: FsRead> FsRead for DryRun<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        // Reads execute normally - we need real state to test against
        self.inner.read(path)
    }

    // All read operations pass through unchanged
}

impl<B: FsWrite> FsWrite for DryRun<B> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref();
        self.log(format!("write {} ({} bytes)", path.display(), data.len()));
        Ok(())  // Don't actually write
    }

    fn remove_file(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref();
        self.log(format!("remove_file {}", path.display()));
        Ok(())  // Don't actually remove
    }

    fn open_write(&self, path: &Path) -> Result<Box<dyn Write + Send>, FsError> {
        let path = path.as_ref();
        self.log(format!("open_write {}", path.display()));
        // Return a sink that discards all writes
        Ok(Box::new(std::io::sink()))
    }

    // ... similar for all write operations
}

impl<B: FsDir> FsDir for DryRun<B> {
    fn read_dir(&self, path: &Path) -> Result<ReadDirIter, FsError> {
        self.inner.read_dir(path)  // Pass through
    }

    fn create_dir(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref();
        self.log(format!("create_dir {}", path.display()));
        Ok(())
    }

    // ... similar for all directory mutations
}
```

### Semantics Clarification

**DryRun is NOT an isolation layer.** It's for answering "what would this code do?"

- Reads see the **real** backend state (unchanged from before DryRun was applied)
- Writes are **logged but not executed**
- After a dry write, reads won't see the change (because it wasn't written)

This is intentional. For isolation, use `MemoryBackend::clone()` for snapshots.

### Verdict: ✅ Implementable

The semantics are clear once documented. Uses `std::io::sink()` for discarding streamed writes.

---

## 6. PathFilter<B>

**Complexity:** Moderate
**State:** Compiled glob patterns
**Dependencies:** `globset` crate

### Implementation

```rust
use globset::{Glob, GlobSet, GlobSetBuilder};

pub struct PathFilter<B> {
    inner: B,
    rules: Vec<PathRule>,
    compiled: GlobSet,  // For efficient matching
}

enum PathRule {
    Allow(String),
    Deny(String),
}

impl<B> PathFilter<B> {
    fn check_access(&self, path: &Path) -> Result<(), FsError> {
        let path_str = path.to_string_lossy();

        for rule in &self.rules {
            match rule {
                PathRule::Allow(pattern) => {
                    if glob_matches(pattern, &path_str) {
                        return Ok(());
                    }
                }
                PathRule::Deny(pattern) => {
                    if glob_matches(pattern, &path_str) {
                        return Err(FsError::AccessDenied {
                            path: path.to_path_buf(),
                            reason: format!("path matches deny pattern: {}", pattern),
                        });
                    }
                }
            }
        }

        // Default: deny if no rules matched
        Err(FsError::AccessDenied {
            path: path.to_path_buf(),
            reason: "no matching allow rule".to_string(),
        })
    }
}

impl<B: FsRead> FsRead for PathFilter<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();
        self.check_access(path)?;
        self.inner.read(path)
    }

    // ... all methods check access first
}

impl<B: FsDir> FsDir for PathFilter<B> {
    fn read_dir(&self, path: &Path) -> Result<ReadDirIter, FsError> {
        let path = path.as_ref();
        self.check_access(path)?;

        let inner_iter = self.inner.read_dir(path)?;

        // Filter the iterator to exclude denied entries
        Ok(ReadDirIter::new(FilteredDirIter {
            inner: inner_iter,
            filter: self.clone(),  // Need access to rules
        }))
    }
}

// Custom iterator that filters denied entries
struct FilteredDirIter<B> {
    inner: ReadDirIter,
    rules: Vec<PathRule>,
}

impl Iterator for FilteredDirIter {
    type Item = Result<DirEntry, FsError>;

    fn next(&mut self) -> Option<Self::Item> {
        loop {
            match self.inner.next()? {
                Ok(entry) => {
                    if self.is_allowed(&entry.path) {
                        return Some(Ok(entry));
                    }
                    // Skip denied entries (don't reveal their existence)
                }
                Err(e) => return Some(Err(e)),
            }
        }
    }
}
```

### Considerations

- **Rule evaluation order:** First match wins, consistent with firewall rules.
- **Default policy:** Deny if no rules match (secure by default).
- **Directory listing:** Filters out denied entries so their existence isn't revealed.
- **Parent directory access:** If you allow `/workspace/**`, accessing `/workspace` itself needs to be allowed.

### Implementation Detail: ReadDirIter Filtering

Our `ReadDirIter` type needs to support wrapping. Options:

```rust
// Option 1: ReadDirIter is a trait object
pub struct ReadDirIter(Box<dyn Iterator<Item = Result<DirEntry, FsError>> + Send>);

// Option 2: ReadDirIter has a filter method
impl ReadDirIter {
    pub fn filter<F>(self, predicate: F) -> ReadDirIter
    where
        F: Fn(&DirEntry) -> bool + Send + 'static
    { ... }
}
```

**Recommendation:** Option 1 (trait object) is more flexible and aligns with `open_read`/`open_write` returning `Box<dyn ...>`.

### Verdict: ✅ Implementable

Requires `ReadDirIter` to be a trait object wrapper (already the case) so we can filter entries.

---

## 7. Cache<B>

**Complexity:** Moderate
**State:** LRU cache with entries
**Dependencies:** `lru` crate (or custom implementation)

### Implementation

```rust
use lru::LruCache;
use std::sync::RwLock;
use std::time::{Duration, Instant};

pub struct Cache<B> {
    inner: B,
    cache: RwLock<LruCache<PathBuf, CacheEntry>>,
    max_entry_size: usize,
    ttl: Duration,
}

struct CacheEntry {
    data: Vec<u8>,
    metadata: Metadata,
    inserted_at: Instant,
}

impl<B: FsRead> FsRead for Cache<B> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();

        // Check cache
        {
            let cache = self.cache.read().unwrap();
            if let Some(entry) = cache.peek(path) {
                if entry.inserted_at.elapsed() < self.ttl {
                    return Ok(entry.data.clone());
                }
            }
        }

        // Cache miss - fetch from backend
        let data = self.inner.read(path)?;

        // Store in cache if not too large
        if data.len() <= self.max_entry_size {
            let metadata = self.inner.metadata(path)?;
            let mut cache = self.cache.write().unwrap();
            cache.put(path.to_path_buf(), CacheEntry {
                data: data.clone(),
                metadata,
                inserted_at: Instant::now(),
            });
        }

        Ok(data)
    }

    fn metadata(&self, path: &Path) -> Result<Metadata, FsError> {
        let path = path.as_ref();

        // Check cache for metadata
        {
            let cache = self.cache.read().unwrap();
            if let Some(entry) = cache.peek(path) {
                if entry.inserted_at.elapsed() < self.ttl {
                    return Ok(entry.metadata.clone());
                }
            }
        }

        // Fetch from backend
        self.inner.metadata(path)
    }

    fn open_read(&self, path: &Path) -> Result<Box<dyn Read + Send>, FsError> {
        // DO NOT CACHE - streams are for large files
        self.inner.open_read(path)
    }

    fn exists(&self, path: &Path) -> Result<bool, FsError> {
        // Could cache this too, or derive from metadata cache
        let path = path.as_ref();
        {
            let cache = self.cache.read().unwrap();
            if let Some(entry) = cache.peek(path) {
                if entry.inserted_at.elapsed() < self.ttl {
                    return Ok(true);  // If in cache, it exists
                }
            }
        }
        self.inner.exists(path)
    }
}

impl<B: FsWrite> FsWrite for Cache<B> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref();
        let result = self.inner.write(path, data)?;

        // Invalidate cache entry
        let mut cache = self.cache.write().unwrap();
        cache.pop(path);

        Ok(result)
    }

    fn remove_file(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref();
        let result = self.inner.remove_file(path)?;

        // Invalidate cache entry
        let mut cache = self.cache.write().unwrap();
        cache.pop(path);

        Ok(result)
    }

    // ... all mutations invalidate cache
}
```

### What Gets Cached

| Method             | Cached? | Reason                                               |
| ------------------ | ------- | ---------------------------------------------------- |
| `read()`           | Yes     | Small files benefit from caching                     |
| `read_to_string()` | Yes     | Same as read                                         |
| `read_range()`     | Maybe   | Could cache full file, serve ranges from cache       |
| `metadata()`       | Yes     | Frequently accessed                                  |
| `exists()`         | Derived | Can derive from metadata cache                       |
| `open_read()`      | **No**  | Streams are for large files that shouldn't be cached |
| `read_dir()`       | Maybe   | Directory listings change frequently                 |

### Verdict: ✅ Implementable

Standard LRU cache pattern. Key decision: don't cache `open_read()` streams.

---

## 8. Quota<B>

**Complexity:** High
**State:** Usage counters (requires accurate tracking)
**Dependencies:** None

### The Challenge

Quota must track:
- Total bytes used
- Total file count
- Total directory count
- Per-directory entry count (optional)
- Maximum path depth (optional)

The tricky part: **streaming writes via `open_write()`**. We must track bytes as they're written, not just when the operation completes.

### Implementation

```rust
use std::sync::{Arc, RwLock};
use std::io::Write;

pub struct Quota<B> {
    inner: B,
    config: QuotaConfig,
    usage: Arc<RwLock<QuotaUsage>>,
}

struct QuotaConfig {
    max_total_size: Option<u64>,
    max_file_size: Option<u64>,
    max_node_count: Option<u64>,
    max_dir_entries: Option<u64>,  // Max entries per directory
    max_path_depth: Option<usize>,
}

/// Current usage statistics.
#[derive(Debug, Clone, Default)]
pub struct Usage {
    pub total_size: u64,
    pub file_count: u64,
    pub dir_count: u64,
}

/// Configured limits.
#[derive(Debug, Clone)]
pub struct Limits {
    pub max_total_size: Option<u64>,
    pub max_file_size: Option<u64>,
    pub max_node_count: Option<u64>,
    pub max_dir_entries: Option<u64>,
    pub max_path_depth: Option<usize>,
}

/// Remaining capacity.
#[derive(Debug, Clone)]
pub struct Remaining {
    pub bytes: Option<u64>,
    pub nodes: Option<u64>,
    pub can_write: bool,
}

struct QuotaUsage {
    total_size: u64,
    file_count: u64,
    dir_count: u64,
}

impl<B> Quota<B> {
    /// Get current usage statistics.
    pub fn usage(&self) -> Usage {
        let u = self.usage.read().unwrap();
        Usage {
            total_size: u.total_size,
            file_count: u.file_count,
            dir_count: u.dir_count,
        }
    }

    /// Get configured limits.
    pub fn limits(&self) -> Limits {
        Limits {
            max_total_size: self.config.max_total_size,
            max_file_size: self.config.max_file_size,
            max_node_count: self.config.max_node_count,
            max_dir_entries: self.config.max_dir_entries,
            max_path_depth: self.config.max_path_depth,
        }
    }

    /// Get remaining capacity.
    pub fn remaining(&self) -> Remaining {
        let u = self.usage.read().unwrap();
        let bytes = self.config.max_total_size.map(|max| max.saturating_sub(u.total_size));
        let nodes = self.config.max_node_count.map(|max| max.saturating_sub(u.file_count + u.dir_count));
        Remaining {
            bytes,
            nodes,
            can_write: bytes.map(|b| b > 0).unwrap_or(true),
        }
    }
}
```impl<B: Fs> Quota<B> {
    pub fn new(inner: B, config: QuotaConfig) -> Result<Self, FsError> {
        // IMPORTANT: Scan backend to initialize usage counters
        let usage = Self::scan_usage(&inner)?;

        Ok(Self {
            inner,
            config,
            usage: Arc::new(RwLock::new(usage)),
        })
    }

    fn scan_usage(backend: &B) -> Result<QuotaUsage, FsError> {
        let mut usage = QuotaUsage::default();
        Self::scan_dir(backend, Path::new("/"), &mut usage)?;
        Ok(usage)
    }

    fn scan_dir(backend: &B, path: &Path, usage: &mut QuotaUsage) -> Result<(), FsError> {
        for entry in backend.read_dir(path)? {
            let entry = entry?;
            let meta = backend.metadata(&entry.path)?;

            if meta.is_file {
                usage.file_count += 1;
                usage.total_size += meta.size;
            } else if meta.is_dir {
                usage.dir_count += 1;
                Self::scan_dir(backend, &entry.path, usage)?;
            }
        }
        Ok(())
    }

    fn check_size_limit(&self, additional_bytes: u64) -> Result<(), FsError> {
        let usage = self.usage.read().unwrap();

        if let Some(max) = self.config.max_total_size {
            if usage.total_size + additional_bytes > max {
                return Err(FsError::QuotaExceeded {
                    limit: max,
                    requested: additional_bytes,
                    usage: usage.total_size,
                });
            }
        }

        Ok(())
    }

    fn check_node_limit(&self) -> Result<(), FsError> {
        if let Some(max) = self.config.max_node_count {
            let usage = self.usage.read().unwrap();
            if usage.file_count + usage.dir_count >= max {
                return Err(FsError::QuotaExceeded {
                    limit: max,
                    requested: 1,
                    usage: usage.file_count + usage.dir_count,
                });
            }
        }
        Ok(())
    }

    fn check_dir_entries(&self, parent: &Path) -> Result<(), FsError>
    where B: FsDir {
        if let Some(max) = self.config.max_dir_entries {
            // Count entries in parent directory
            let count = self.inner.read_dir(parent)?
                .filter(|e| e.is_ok())
                .count() as u64;
            if count >= max {
                return Err(FsError::QuotaExceeded {
                    limit: max,
                    requested: 1,
                    usage: count,
                });
            }
        }
        Ok(())
    }

    fn check_path_depth(&self, path: &Path) -> Result<(), FsError> {
        if let Some(max) = self.config.max_path_depth {
            let depth = path.components().count();
            if depth > max {
                return Err(FsError::QuotaExceeded {
                    limit: max as u64,
                    requested: depth as u64,
                    usage: depth as u64,
                });
            }
        }
        Ok(())
    }
}

impl<B: FsWrite + FsRead + FsDir> FsWrite for Quota<B> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref();
        let new_size = data.len() as u64;

        // Check path depth limit
        self.check_path_depth(path)?;

        // Check per-file limit
        if let Some(max) = self.config.max_file_size {
            if new_size > max {
                return Err(FsError::FileSizeExceeded {
                    path: path.to_path_buf(),
                    size: new_size,
                    limit: max,
                });
            }
        }

        // Get old size (if file exists)
        let old_size = self.inner.metadata(path)
            .map(|m| m.size)
            .unwrap_or(0);

        // If creating a new file, check node count and dir entries
        let is_new_file = old_size == 0;
        if is_new_file {
            self.check_node_limit()?;
            if let Some(parent) = path.parent() {
                self.check_dir_entries(parent)?;
            }
        }

        let size_delta = new_size as i64 - old_size as i64;

        if size_delta > 0 {
            self.check_size_limit(size_delta as u64)?;
        }

        // Perform write
        self.inner.write(path, data)?;

        // Update usage
        let mut usage = self.usage.write().unwrap();
        usage.total_size = (usage.total_size as i64 + size_delta) as u64;
        if is_new_file {
            usage.file_count += 1;
        }

        Ok(())
    }

    fn open_write(&self, path: &Path) -> Result<Box<dyn Write + Send>, FsError> {
        let path = path.as_ref().to_path_buf();

        // Get the underlying writer
        let inner_writer = self.inner.open_write(&path)?;

        // Wrap in a counting writer
        Ok(Box::new(QuotaWriter {
            inner: inner_writer,
            path,
            bytes_written: 0,
            usage: Arc::clone(&self.usage),
            max_file_size: self.config.max_file_size,
            max_total_size: self.config.max_total_size,
        }))
    }
}

/// Wrapper that counts bytes and enforces quota on streaming writes
struct QuotaWriter {
    inner: Box<dyn Write + Send>,
    path: PathBuf,
    bytes_written: u64,
    usage: Arc<RwLock<QuotaUsage>>,
    max_file_size: Option<u64>,
    max_total_size: Option<u64>,
}

impl Write for QuotaWriter {
    fn write(&mut self, buf: &[u8]) -> std::io::Result<usize> {
        let additional = buf.len() as u64;

        // Check per-file limit
        if let Some(max) = self.max_file_size {
            if self.bytes_written + additional > max {
                return Err(std::io::Error::new(
                    std::io::ErrorKind::Other,
                    "file size limit exceeded"
                ));
            }
        }

        // Check total size limit
        if let Some(max) = self.max_total_size {
            let usage = self.usage.read().unwrap();
            if usage.total_size + additional > max {
                return Err(std::io::Error::new(
                    std::io::ErrorKind::Other,
                    "quota exceeded"
                ));
            }
        }

        // Write to inner
        let written = self.inner.write(buf)?;

        // Update counters
        self.bytes_written += written as u64;
        let mut usage = self.usage.write().unwrap();
        usage.total_size += written as u64;

        Ok(written)
    }

    fn flush(&mut self) -> std::io::Result<()> {
        self.inner.flush()
    }
}

impl Drop for QuotaWriter {
    fn drop(&mut self) {
        // If we need to track "committed" vs "in-progress" writes,
        // this is where we'd finalize the accounting
    }
}

impl<B: FsDir + FsRead> FsDir for Quota<B> {
    fn create_dir(&self, path: &Path) -> Result<(), FsError> {
        // Check path depth
        self.check_path_depth(path)?;

        // Check node count
        self.check_node_limit()?;

        // Check parent directory entries
        if let Some(parent) = path.parent() {
            self.check_dir_entries(parent)?;
        }

        // Create directory
        self.inner.create_dir(path)?;

        // Update usage
        let mut usage = self.usage.write().unwrap();
        usage.dir_count += 1;

        Ok(())
    }

    // create_dir_all, remove_dir, etc. delegate similarly
    // ...
}
```

### Challenges and Solutions

| Challenge             | Solution                                    |
| --------------------- | ------------------------------------------- |
| Initial usage unknown | Scan backend on construction                |
| Streaming writes      | `QuotaWriter` wrapper counts bytes          |
| Concurrent writes     | `RwLock` on usage counters                  |
| File replacement      | Calculate delta (new_size - old_size)       |
| New file detection    | Check `exists()` before write               |
| Accurate accounting   | Update counters after successful operations |
| Node count limit      | Check before creating files/directories     |
| Dir entries limit     | Count parent entries before creating child  |
| Path depth limit      | Count path components on create             |

### Edge Cases

1. **Partial write failure:** If `inner.write()` fails, don't update counters.
2. **Streaming write failure:** `QuotaWriter` updates optimistically; on error, may need rollback.
3. **Rename:** Doesn't change total size.
4. **Copy:** Adds destination size.
5. **Append:** Adds appended bytes only.

### Verdict: ✅ Implementable

The most complex middleware, but well-understood patterns. The `QuotaWriter` wrapper is the key insight.

---

## 9. Overlay<B1, B2>

**Complexity:** High
**State:** Two backends + whiteout tracking
**Dependencies:** None

### Overlay Semantics (Docker-style)

- **Lower layer (base):** Read-only source
- **Upper layer:** Writable overlay
- **Whiteouts:** Files named `.wh.<filename>` mark deletions
- **Opaque directories:** `.wh..wh..opq` hides entire lower directory

### Implementation

```rust
pub struct Overlay<Lower, Upper> {
    lower: Lower,
    upper: Upper,
}

impl<Lower, Upper> Overlay<Lower, Upper> {
    const WHITEOUT_PREFIX: &'static str = ".wh.";
    const OPAQUE_MARKER: &'static str = ".wh..wh..opq";

    fn whiteout_path(path: &Path) -> PathBuf {
        let parent = path.parent().unwrap_or(Path::new("/"));
        let name = path.file_name().unwrap_or_default();
        parent.join(format!("{}{}", Self::WHITEOUT_PREFIX, name.to_string_lossy()))
    }

    fn is_whiteout(name: &str) -> bool {
        name.starts_with(Self::WHITEOUT_PREFIX)
    }

    fn original_name(whiteout_name: &str) -> &str {
        &whiteout_name[Self::WHITEOUT_PREFIX.len()..]
    }
}

impl<Lower: FsRead, Upper: FsRead> FsRead for Overlay<Lower, Upper> {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();

        // Check if whiteout exists in upper
        let whiteout = Self::whiteout_path(path);
        if self.upper.exists(&whiteout).unwrap_or(false) {
            return Err(FsError::NotFound { path: path.to_path_buf() });
        }

        // Try upper first
        match self.upper.read(path) {
            Ok(data) => return Ok(data),
            Err(FsError::NotFound { .. }) => {}
            Err(e) => return Err(e),
        }

        // Fall back to lower
        self.lower.read(path)
    }

    fn exists(&self, path: &Path) -> Result<bool, FsError> {
        let path = path.as_ref();

        // Check whiteout first
        let whiteout = Self::whiteout_path(path);
        if self.upper.exists(&whiteout).unwrap_or(false) {
            return Ok(false);  // Whited out = doesn't exist
        }

        // Check upper, then lower
        if self.upper.exists(path).unwrap_or(false) {
            return Ok(true);
        }

        self.lower.exists(path)
    }

    fn metadata(&self, path: &Path) -> Result<Metadata, FsError> {
        let path = path.as_ref();

        // Check whiteout
        let whiteout = Self::whiteout_path(path);
        if self.upper.exists(&whiteout).unwrap_or(false) {
            return Err(FsError::NotFound { path: path.to_path_buf() });
        }

        // Upper first, then lower
        match self.upper.metadata(path) {
            Ok(meta) => return Ok(meta),
            Err(FsError::NotFound { .. }) => {}
            Err(e) => return Err(e),
        }

        self.lower.metadata(path)
    }
}

impl<Lower: FsRead, Upper: Fs> FsWrite for Overlay<Lower, Upper> {
    fn write(&self, path: &Path, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref();

        // Remove whiteout if it exists
        let whiteout = Self::whiteout_path(path);
        let _ = self.upper.remove_file(&whiteout);  // Ignore if doesn't exist

        // Write to upper
        self.upper.write(path, data)
    }

    fn remove_file(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref();

        // Try to remove from upper
        let _ = self.upper.remove_file(path);

        // If file exists in lower, create whiteout
        if self.lower.exists(path).unwrap_or(false) {
            let whiteout = Self::whiteout_path(path);
            self.upper.write(&whiteout, b"")?;  // Create whiteout marker
        }

        Ok(())
    }

    fn rename(&self, from: &Path, to: &Path) -> Result<(), FsError> {
        let from = from.as_ref();
        let to = to.as_ref();

        // Copy-on-write: read from overlay, write to upper, whiteout original
        let data = self.read(from)?;
        self.write(to, &data)?;
        self.remove_file(from)?;

        Ok(())
    }
}

impl<Lower: FsRead + FsDir, Upper: Fs> FsDir for Overlay<Lower, Upper> {
    fn read_dir(&self, path: &Path) -> Result<ReadDirIter, FsError> {
        let path = path.as_ref();

        // Check for opaque marker
        let opaque_marker = path.join(Self::OPAQUE_MARKER);
        let is_opaque = self.upper.exists(&opaque_marker).unwrap_or(false);

        // Get entries from upper
        let mut entries: HashMap<String, DirEntry> = HashMap::new();
        let mut whiteouts: HashSet<String> = HashSet::new();

        if let Ok(upper_iter) = self.upper.read_dir(path) {
            for entry in upper_iter {
                let entry = entry?;
                let name = entry.name.clone();

                if Self::is_whiteout(&name) {
                    whiteouts.insert(Self::original_name(&name).to_string());
                } else if name != Self::OPAQUE_MARKER {
                    entries.insert(name, entry);
                }
            }
        }

        // Merge lower entries (unless opaque)
        if !is_opaque {
            if let Ok(lower_iter) = self.lower.read_dir(path) {
                for entry in lower_iter {
                    let entry = entry?;
                    let name = entry.name.clone();

                    // Skip if already in upper or whited out
                    if !entries.contains_key(&name) && !whiteouts.contains(&name) {
                        entries.insert(name, entry);
                    }
                }
            }
        }

        // Convert to iterator
        let entries_vec: Vec<_> = entries.into_values().map(Ok).collect();
        Ok(ReadDirIter::from_vec(entries_vec))
    }

    fn create_dir(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref();

        // Remove whiteout if exists
        let whiteout = Self::whiteout_path(path);
        let _ = self.upper.remove_file(&whiteout);

        self.upper.create_dir(path)
    }

    fn remove_dir(&self, path: &Path) -> Result<(), FsError> {
        let path = path.as_ref();

        // Try to remove from upper
        let _ = self.upper.remove_dir(path);

        // If exists in lower, create whiteout
        if self.lower.exists(path).unwrap_or(false) {
            let whiteout = Self::whiteout_path(path);
            self.upper.write(&whiteout, b"")?;
        }

        Ok(())
    }
}
```

### Key Concepts

| Concept           | Description                                                      |
| ----------------- | ---------------------------------------------------------------- |
| **Whiteout**      | `.wh.<name>` file in upper marks deletion of `<name>` from lower |
| **Opaque**        | `.wh..wh..opq` file in a directory hides all lower entries       |
| **Copy-on-write** | First write copies from lower to upper, then modifies            |
| **Merge**         | `read_dir()` combines both layers, respecting whiteouts          |

### Challenges

1. **Whiteout storage:** Whiteouts are regular files - backend doesn't need special support.
2. **Directory listing merge:** Must be memory-buffered to remove duplicates and whiteouts.
3. **Rename:** Implemented as copy + delete (standard CoW pattern).
4. **Symlinks in lower:** Need to handle carefully - symlink targets might point to lower layer.

### ReadDirIter Consideration

For Overlay, we need to buffer the merged directory listing. This means `ReadDirIter` must support construction from a `Vec`:

```rust
impl ReadDirIter {
    pub fn from_vec(entries: Vec<Result<DirEntry, FsError>>) -> Self {
        Self(Box::new(entries.into_iter()))
    }
}
```

### Verdict: ✅ Implementable

The most complex middleware, but uses well-established patterns from OverlayFS. Key insight: whiteouts are just marker files, no special backend support needed.

---

## Summary

| Middleware   | Complexity | Key Implementation Insight                  |
| ------------ | ---------- | ------------------------------------------- |
| ReadOnly     | Trivial    | Block all writes                            |
| Restrictions | Simple     | Flag checks                                 |
| Tracing      | Simple     | Wrap operations in spans                    |
| RateLimit    | Moderate   | Atomic counter + time window                |
| DryRun       | Moderate   | Log writes, return Ok without executing     |
| PathFilter   | Moderate   | Glob matching + filtered ReadDirIter        |
| Cache        | Moderate   | LRU with TTL, invalidate on writes          |
| Quota        | High       | Usage counters + QuotaWriter wrapper        |
| Overlay      | High       | Whiteout markers + merged directory listing |

### Required Framework Features

These middleware implementations assume:

1. **`ReadDirIter` is a trait object wrapper** - allows filtering and composition
2. **All methods use `&self`** - interior mutability for state
3. **`FsError` has all necessary variants** - ReadOnly, RateLimitExceeded, QuotaExceeded, AccessDenied, FeatureNotEnabled

All of these are already part of our design. **All middleware are implementable.**

---

## Appendix: Layer Trait Implementation

Each middleware provides a corresponding Layer type for composition:

```rust
// Example for Quota
pub struct QuotaLayer {
    config: QuotaConfig,
}

impl QuotaLayer {
    pub fn builder() -> QuotaLayerBuilder<Unconfigured> {
        QuotaLayerBuilder::new()
    }
}

impl<B: Fs> Layer<B> for QuotaLayer {
    type Backend = Quota<B>;

    fn layer(self, backend: B) -> Self::Backend {
        Quota::new(backend, self.config).expect("quota initialization failed")
    }
}

// Usage:
let fs = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(100_000_000)
        .build());
```

