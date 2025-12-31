# Prior Art Analysis: Filesystem Abstraction Libraries

This document analyzes filesystem abstraction libraries in other languages to learn from their successes, identify features we should adopt, and avoid known vulnerabilities.

---

## Executive Summary

| Library                                                                        | Language | Key Strength                                 | Key Weakness                              | What We Can Learn                |
| ------------------------------------------------------------------------------ | -------- | -------------------------------------------- | ----------------------------------------- | -------------------------------- |
| [fsspec](https://filesystem-spec.readthedocs.io/)                              | Python   | Async + caching + data science integration   | No middleware composition                 | Caching strategies, async design |
| [PyFilesystem2](https://github.com/PyFilesystem/pyfilesystem2)                 | Python   | Clean URL-based API                          | Symlink handling issues                   | Path normalization               |
| [Afero](https://github.com/spf13/afero)                                        | Go       | Composition (CopyOnWrite, Cache, BasePathFs) | Symlink escape in BasePathFs              | Composition patterns             |
| [Apache Commons VFS](https://commons.apache.org/vfs/)                          | Java     | Enterprise-grade, many backends              | **CVE: Path traversal with encoded `..`** | URL encoding attacks             |
| [System.IO.Abstractions](https://github.com/TestableIO/System.IO.Abstractions) | .NET     | Perfect for testing, mirrors System.IO       | No middleware/composition                 | MockFileSystem patterns          |
| [memfs](https://github.com/streamich/memfs)                                    | Node.js  | Browser + Node unified API                   | Fork exists due to "longstanding bugs"    | In-memory implementation         |
| [soft-canonicalize](https://github.com/DK26/soft-canonicalize-rs)              | Rust     | Non-existing path resolution, TOCTOU-safe    | Real FS only (not virtual)                | Attack patterns to defend        |
| [strict-path](https://github.com/DK26/strict-path-rs)                          | Rust     | 19+ attack types blocked, type-safe markers  | Real FS only (not virtual)                | Attack catalog for testing       |

---

## Detailed Analysis

### 1. Python: fsspec

**Repository:** [fsspec/filesystem_spec](https://github.com/fsspec/filesystem_spec)

**What they do well:**

1. **Unified Interface Across 20+ Backends**
   - Local, S3, GCS, Azure, HDFS, HTTP, FTP, SFTP, ZIP, TAR, Git, etc.
   - Same API regardless of backend

2. **Sophisticated Caching**
   ```python
   # Block-wise caching - only download accessed parts
   fs = fsspec.filesystem('blockcache', target_protocol='s3',
                          cache_storage='/tmp/cache')

   # Whole-file caching
   fs = fsspec.filesystem('filecache', target_protocol='s3',
                          cache_storage='/tmp/cache')
   ```

3. **Async Support**
   - `AsyncFileSystem` base class for async implementations
   - Concurrent bulk operations (`cat` fetches many files at once)
   - Used by Dask for parallel data processing

4. **Data Science Integration**
   - Native integration with Pandas, Dask, Intake
   - Parquet optimization with parallel chunk fetching

**What we should adopt:**
- [ ] Block-wise caching strategy (not just whole-file LRU)
- [ ] Async design from the start (our ADR-024 async plan)
- [ ] Consider "parts caching" for large file access patterns

**What they lack that we have:**
- No middleware composition pattern
- No quota/rate limiting built-in
- No path filtering/sandboxing

---

### 2. Python: PyFilesystem2

**Repository:** [PyFilesystem/pyfilesystem2](https://github.com/PyFilesystem/pyfilesystem2)

**What they do well:**

1. **URL-based Filesystem Specification**
   ```python
   from fs import open_fs

   home_fs = open_fs('osfs://~/')
   zip_fs = open_fs('zip://foo.zip')
   ftp_fs = open_fs('ftp://ftp.example.com')
   mem_fs = open_fs('mem://')
   ```

2. **Consistent Path Handling**
   - Forward slashes everywhere (even on Windows)
   - Paths normalized automatically

3. **Glob Support Built-in**
   ```python
   for match in fs.glob('**/*.py'):
       print(match.path)
   ```

**Known Issues (from GitHub):**

| Issue                                                            | Description                                              | Impact               |
| ---------------------------------------------------------------- | -------------------------------------------------------- | -------------------- |
| [#171](https://github.com/PyFilesystem/pyfilesystem2/issues/171) | Symlink loops cause infinite recursion                   | DoS potential        |
| [#417](https://github.com/PyFilesystem/pyfilesystem2/issues/417) | No symlink creation support                              | Missing feature      |
| [#411](https://github.com/PyFilesystem/pyfilesystem2/issues/411) | Incorrect handling of symlinks with non-existing targets | Broken functionality |
| [#61](https://github.com/PyFilesystem/pyfilesystem2/issues/61)   | Symlinks not detected properly                           | Security concern     |

**Lessons for AnyFS:**
- ⚠️ **Symlink handling is complex** - we must handle loops, non-existent targets, and escaping
- ✅ **URL-based opening is convenient** - consider for future
- ✅ **Consistent path format** - we already do this (forward slashes always)

---

### 3. Go: Afero

**Repository:** [spf13/afero](https://github.com/spf13/afero)

**What they do well:**

1. **Composition Pattern (Similar to Ours!)**
   ```go
   // Sandboxing
   baseFs := afero.NewOsFs()
   restrictedFs := afero.NewBasePathFs(baseFs, "/var/data")

   // Caching layer
   cachedFs := afero.NewCacheOnReadFs(baseFs, afero.NewMemMapFs(), time.Hour)

   // Copy-on-write
   cowFs := afero.NewCopyOnWriteFs(baseFs, afero.NewMemMapFs())
   ```

2. **io/fs Compatibility**
   - Works with Go 1.16+ standard library interfaces
   - `ReadDirFS`, `ReadFileFS`, etc.

3. **Extensive Backend Support**
   - OS, Memory, SFTP, GCS
   - Community: S3, MinIO, Dropbox, Google Drive, Git

**Known Issues:**

| Issue                                             | Description                            | Our Mitigation                             |
| ------------------------------------------------- | -------------------------------------- | ------------------------------------------ |
| [#282](https://github.com/spf13/afero/issues/282) | Symlinks in BasePathFs can escape jail | Use `strict-path` crate for VRootFsBackend |
| [#88](https://github.com/spf13/afero/issues/88)   | Symlink handling inconsistent          | Document behavior clearly                  |
| [#344](https://github.com/spf13/afero/issues/344) | BasePathFs fails when basepath is `.`  | Test edge cases                            |

**BasePathFs Symlink Escape Issue:**

> "SymlinkIfPossible will resolve the RealPath of underlayer filesystem before make a symlink. For example, creating a link like '/foo/bar' -> '/foo/file' will be transform into a link point to '/{basepath}/foo/file.'"

This means symlinks can potentially point outside the base path!

**Our Solution:**
- `VRootFsBackend` uses [`strict-path`](https://github.com/DK26/strict-path-rs) for real filesystem containment
- Virtual backends (Memory, SQLite) are inherently safe - paths are just keys
- `PathFilter` middleware provides additional sandboxing layer

**What we should verify:**
- [ ] Test symlink creation pointing outside VRootFsBackend
- [ ] Test `..` in symlink targets
- [ ] Test symlink loops with max depth

---

### 4. Java: Apache Commons VFS

**Repository:** [Apache Commons VFS](https://commons.apache.org/vfs/)

**🔴 CRITICAL VULNERABILITY: CVE in versions < 2.10.0**

**The Bug:**

```java
// FileObject API has resolveFile with scope parameter
FileObject file = baseFile.resolveFile("../secret.txt", NameScope.DESCENDENT);
// SHOULD throw exception - "../secret.txt" is not a descendent

// BUT with URL encoding:
FileObject file = baseFile.resolveFile("%2e%2e/secret.txt", NameScope.DESCENDENT);
// DOES NOT throw exception! Returns file outside base directory.
```

**Root Cause:** Path validation happened BEFORE URL decoding.

**Lesson for AnyFS:**

```rust
// WRONG - validate then decode
fn resolve(path: &str) -> Result<PathBuf, FsError> {
    validate_no_traversal(path)?;  // Checks for ".."
    let decoded = url_decode(path);  // "../" appears after decode!
    Ok(PathBuf::from(decoded))
}

// CORRECT - decode then validate
fn resolve(path: &str) -> Result<PathBuf, FsError> {
    let decoded = url_decode(path);
    let normalized = normalize_path(&decoded);  // Resolve all ".."
    validate_containment(&normalized)?;
    Ok(normalized)
}
```

**Action Items:**
- [ ] Add test: URL-encoded `%2e%2e` path traversal attempt
- [ ] Add test: Double-encoding `%252e%252e`
- [ ] Ensure path normalization happens BEFORE validation
- [ ] Document in security model

---

### 5. .NET: System.IO.Abstractions

**Repository:** [TestableIO/System.IO.Abstractions](https://github.com/TestableIO/System.IO.Abstractions)

**What they do well:**

1. **Perfect API Compatibility**
   - Mirrors `System.IO` exactly
   - Drop-in replacement for testing

2. **MockFileSystem for Testing**
   ```csharp
   var fileSystem = new MockFileSystem(new Dictionary<string, MockFileData>
   {
       { @"c:\myfile.txt", new MockFileData("Testing") },
       { @"c:\demo\jQuery.js", new MockFileData("jQuery content") },
   });

   // Use in tests
   var sut = new MyComponent(fileSystem);
   ```

3. **Analyzers Package**
   - Roslyn analyzers warn when using `System.IO` directly
   - Guides developers to use abstractions

**What they lack:**
- No middleware/composition
- No caching layer
- No sandboxing/path filtering
- Testing-focused, not production backends

**What we should adopt:**
- [ ] Consider Rust analyzer/clippy lint for `std::fs` usage
- [ ] MockFileSystem pattern is similar to our `MemoryBackend`

---

### 6. Node.js: memfs + unionfs

**Repository:** [streamich/memfs](https://github.com/streamich/memfs)

**What they do well:**

1. **Browser + Node Unified**
   - Works in browser via File System API
   - Same API as Node's `fs`

2. **Union Filesystem Composition**
   ```javascript
   import { Union } from 'unionfs';
   import { fs as memfs } from 'memfs';
   import * as fs from 'fs';

   const ufs = new Union();
   ufs.use(fs);        // Real filesystem as base
   ufs.use(memfs);     // Memory overlay
   ```

**Known Issues:**

> "There is a fork of memfs maintained by SageMath (sagemathinc/memfs-js) which was created to fix 13 security vulnerabilities revealed by npm audit. This fork exists because, as their GitHub description notes, 'there are longstanding bugs' in the upstream memfs."

**Lesson:** Even popular libraries can have security issues. Our conformance test suite should be comprehensive.

---

## Vulnerabilities Summary

| Library                | Vulnerability         | Type                                   | Our Mitigation                     |
| ---------------------- | --------------------- | -------------------------------------- | ---------------------------------- |
| **Apache Commons VFS** | CVE (pre-2.10.0)      | URL-encoded path traversal             | Decode before validate             |
| **Afero (Go)**         | Issue #282, #88       | Symlink escape from BasePathFs         | Use `strict-path`, test thoroughly |
| **PyFilesystem2**      | Issue #171            | Symlink loop causes infinite recursion | Loop detection with max depth      |
| **memfs (Node)**       | 13 vulns in npm audit | Various (unspecified)                  | Comprehensive test suite           |

---

## Features Comparison Matrix

| Feature                | fsspec | PyFS2 | Afero | Commons VFS | System.IO.Abs | AnyFS |
| ---------------------- | :----: | :---: | :---: | :---------: | :-----------: | :---: |
| Middleware composition |   ❌    |   ❌   |   ✅   |      ❌      |       ❌       |   ✅   |
| Quota enforcement      |   ❌    |   ❌   |   ❌   |      ❌      |       ❌       |   ✅   |
| Path sandboxing        |   ❌    |   ❌   |   ✅   |      ✅      |       ❌       |   ✅   |
| Rate limiting          |   ❌    |   ❌   |   ❌   |      ❌      |       ❌       |   ✅   |
| Caching layer          |   ✅    |   ❌   |   ✅   |      ❌      |       ❌       |   ✅   |
| Async support          |   ✅    |   ❌   |   ❌   |      ❌      |       ❌       |   🔜   |
| Block-wise caching     |   ✅    |   ❌   |   ❌   |      ❌      |       ❌       |   ❌   |
| URL-based opening      |   ✅    |   ✅   |   ❌   |      ✅      |       ❌       |   ❌   |
| Union/overlay FS       |   ❌    |   ❌   |   ✅   |      ❌      |       ❌       |   ✅   |
| Memory backend         |   ✅    |   ✅   |   ✅   |      ✅      |       ✅       |   ✅   |
| SQLite backend         |   ❌    |   ❌   |   ❌   |      ❌      |       ❌       |   ✅   |
| FUSE mounting          |   ✅    |   ❌   |   ✅   |      ❌      |       ❌       |   ✅   |
| Type-safe markers      |   ❌    |   ❌   |   ❌   |      ❌      |       ❌       |   ✅   |

---

## Future Ideas to Consider

These are optional extensions inspired by other ecosystems. They are intentionally not part of the core v1 scope.

**Keep (post-v1 add-ons that fit the current design):**
- URL-based backend registry (`sqlite://`, `mem://`, `stdfs://`) as a helper crate, not in core APIs.
- Bulk operation helpers (`read_many`, `write_many`, `copy_many`, `glob`, `walk`) as `FsExt` or a utilities crate.
- Early async adapter crate (`anyfs-async`) to support remote backends without changing sync traits.
- Bash-style shell (example app or `anyfs-shell` crate) that routes `ls/cd/cat/cp/mv/rm/mkdir/stat` through `FileStorage` to demonstrate middleware and backend neutrality (navigation and file management only, not full bash scripting).
- Copy-on-write overlay middleware (Afero-style `CopyOnWriteFs`) as a specialized `Overlay` variant.
- Archive backends (zip/tar) as separate crates implementing `Fs` (PyFilesystem/fsspec-style).

**Defer (valuable, but needs data or wider review):**
- Range/block caching middleware for `read_range` heavy workloads (fsspec-style block cache).
- Runtime capability discovery (`Capabilities` struct) for feature detection (symlink control, case sensitivity, max path length).
- Lint/analyzer to discourage direct `std::fs` usage in app code (System.IO.Abstractions-style).
- Retry/timeout middleware for remote backends (once remote backends exist).

**Drop for now (adds noise or cross-platform complexity):**
- Change notification support (optional `FsWatch` trait or polling middleware).

---

## Security Tests to Add

Based on vulnerabilities found in other libraries, add these to our conformance test suite:

### Path Traversal Tests

```rust
#[test]
fn test_url_encoded_path_traversal() {
    let fs = create_sandboxed_fs("/sandbox");

    // These should all fail or be contained
    assert!(fs.read("%2e%2e/etc/passwd").is_err());      // URL-encoded ../
    assert!(fs.read("%252e%252e/secret").is_err());      // Double-encoded
    assert!(fs.read("..%2f..%2fetc/passwd").is_err());   // Mixed encoding
    assert!(fs.read("....//....//etc/passwd").is_err()); // Extra dots
}

#[test]
fn test_symlink_escape() {
    let fs = create_sandboxed_fs("/sandbox");

    // Symlink pointing outside should fail or be contained
    assert!(fs.symlink("/etc/passwd", "/sandbox/link").is_err());
    assert!(fs.symlink("../../../etc/passwd", "/sandbox/link").is_err());

    // Even if symlink created, reading should fail
    fs.symlink("../secret", "/sandbox/link").ok();
    assert!(fs.read("/sandbox/link").is_err());
}

#[test]
fn test_symlink_loop_detection() {
    let fs = MemoryBackend::new();

    // Create loop: a -> b -> a
    fs.symlink("/b", "/a").unwrap();
    fs.symlink("/a", "/b").unwrap();

    // Should detect loop, not hang
    let result = fs.read("/a");
    assert!(matches!(result, Err(FsError::TooManySymlinks { .. })));
}
```

### Resource Exhaustion Tests

```rust
#[test]
fn test_deep_directory_traversal() {
    let fs = create_fs_with_depth_limit(64);

    // Creating very deep paths should fail
    let deep_path = "/".to_string() + &"a/".repeat(100);
    assert!(fs.create_dir_all(&deep_path).is_err());
}

#[test]
fn test_many_open_handles() {
    let fs = create_fs();
    let mut handles = vec![];

    // Opening many files shouldn't crash
    for i in 0..10000 {
        fs.write(format!("/file{}", i), b"x").unwrap();
        if let Ok(h) = fs.open_read(format!("/file{}", i)) {
            handles.push(h);
        }
    }
    // Should either succeed or return resource error, not crash
}
```

---

## Action Items

### High Priority (Before v1.0)

| Task                                        | Source                  | Priority   |
| ------------------------------------------- | ----------------------- | ---------- |
| Add URL-encoded path traversal tests        | Apache Commons VFS CVE  | 🔴 Critical |
| Add symlink escape tests for VRootFsBackend | Afero issues            | 🔴 Critical |
| Add symlink loop detection                  | PyFilesystem2 #171      | 🔴 Critical |
| Verify `strict-path` handles all edge cases | Afero BasePathFs issues | 🔴 Critical |

### Medium Priority (v1.1+)

| Task                                        | Source                     | Priority       |
| ------------------------------------------- | -------------------------- | -------------- |
| Consider block-wise caching for large files | fsspec                     | 🟡 Enhancement  |
| Add async support                           | fsspec async design        | 🟡 Enhancement  |
| URL-based filesystem specification          | PyFilesystem2, Commons VFS | 🟢 Nice-to-have |

### Documentation

| Task                                          | Source                    |
| --------------------------------------------- | ------------------------- |
| Document symlink behavior for each backend    | All libraries have issues |
| Add security considerations for path handling | Apache Commons VFS CVE    |
| Compare AnyFS to alternatives                 | This analysis             |

---

## Sibling Rust Projects: Path Security Libraries

AnyFS builds on foundational security work from two related Rust crates that specifically address path resolution vulnerabilities. These crates are planned to be used in AnyFS's path handling implementation.

### soft-canonicalize-rs

**Repository:** [DK26/soft-canonicalize-rs](https://github.com/DK26/soft-canonicalize-rs)

**Purpose:** Path canonicalization that works with non-existing paths—a critical gap in `std::fs::canonicalize`.

**Security Features:**

| Feature                      | Description                         | Attack Prevented                      |
| ---------------------------- | ----------------------------------- | ------------------------------------- |
| NTFS ADS validation          | Blocks alternate data stream syntax | Hidden data, path escape              |
| Symlink cycle detection      | Bounded depth tracking              | DoS via infinite loops                |
| Path traversal clamping      | Can't ascend past root              | Directory escape                      |
| Null byte rejection          | Early validation                    | Null injection                        |
| TOCTOU resistance            | Atomic-like resolution              | Race conditions                       |
| Windows UNC handling         | Normalizes extended paths           | Path confusion                        |
| Linux namespace preservation | Uses `proc-canonicalize`            | Container escape via `/proc/PID/root` |

**Key Innovation: Anchored Canonicalization**

```rust
// All paths (including symlink targets) are clamped to anchor
let result = anchored_canonicalize("/workspace", user_input)?;
// If symlink points to /etc/passwd, result becomes /workspace/etc/passwd
```

This is exactly what `VRootFsBackend` needs for safe path containment.

### strict-path-rs

**Repository:** [DK26/strict-path-rs](https://github.com/DK26/strict-path-rs)

**Purpose:** Type-safe path handling that prevents traversal attacks at compile time.

**Two Modes:**

| Mode          | Behavior                                     | Use Case                         |
| ------------- | -------------------------------------------- | -------------------------------- |
| `StrictPath`  | Returns `Err(PathEscapesBoundary)` on escape | Archive extraction, file uploads |
| `VirtualPath` | Clamps escape attempts within sandbox        | Multi-tenant, per-user storage   |

**Documented Attack Coverage (19+ vulnerabilities):**

| Attack Type                 | Description                                 |
| --------------------------- | ------------------------------------------- |
| Symlink/junction escapes    | Follows and validates canonical paths       |
| Windows 8.3 short names     | Detects `PROGRA~1` obfuscation              |
| NTFS Alternate Data Streams | Blocks `file.txt:hidden:$DATA`              |
| Zip Slip (CVE-2018-1000178) | Validates archive entries before extraction |
| TOCTOU (CVE-2022-21658)     | Handles time-of-check-time-of-use races     |
| Unicode/encoding bypasses   | Normalizes path representations             |
| Mixed separators            | Handles `/` and `\` on Windows              |
| UNC path tricks             | Prevents `\\?\C:\..\..\` attacks            |

**Type-Safe Marker Pattern (mirrors AnyFS's design!):**

```rust
struct UserFiles;
struct SystemFiles;

fn process_user(f: &StrictPath<UserFiles>) { /* ... */ }
// Wrong marker type = compile error
```

### Applicability to AnyFS

**Important distinction:**

| Backend Type     | Storage Mechanism | Path Resolution Provider        |
| ---------------- | ----------------- | ------------------------------- |
| `VRootFsBackend` | Real filesystem   | OS (backend is `SelfResolving`) |
| `MemoryBackend`  | HashMap keys      | FileStorage (symlink-aware)     |
| `SqliteBackend`  | DB strings        | FileStorage (symlink-aware)     |

**For virtual backends (Memory, SQLite, etc.):**
- These third-party crates perform **real filesystem resolution** (follow actual symlinks on disk)
- Virtual backends treat paths as keys, so these crates can't help
- AnyFS implements its own path resolution in `FileStorage` that:
  1. Walks path components via `metadata()` and `read_link()`
  2. Resolves symlinks by reading targets from virtual storage
  3. Handles `..` correctly after symlink resolution
  4. Detects loops by tracking visited virtual paths

**For `VRootFsBackend` only:**
- Since it wraps the real filesystem, `strict-path` provides safe containment
- The backend implements `SelfResolving`, so FileStorage skips its own resolution

### Security Tests Added to Conformance Suite

Based on these libraries, we've added tests for:

**Windows-Specific:**
- NTFS Alternate Data Streams (`file.txt:hidden`)
- Windows 8.3 short names (`PROGRA~1`)
- UNC path traversal (`\\?\C:\..\..\`)
- Reserved device names (CON, PRN, NUL)
- Junction point escapes

**Linux-Specific:**
- `/proc/PID/root` magic symlinks
- `/dev/fd/N` file descriptor symlinks

**Unicode:**
- NFC vs NFD normalization
- Right-to-Left Override (U+202E)
- Homoglyph confusion (Cyrillic vs Latin)

**TOCTOU:**
- Check-then-use race conditions
- Symlink target changes during resolution

---

## Conclusion

**What makes AnyFS unique:**
1. **Middleware composition** - Only Afero has this, and we do it better (Tower-style)
2. **Quota + rate limiting** - No other library has built-in resource control
3. **Type-safe markers** - Compile-time container isolation is unique to us
4. **SQLite backend** - No other abstraction library offers this

**What we should learn from others:**
1. **Path traversal via encoding** - Apache Commons VFS vulnerability
2. **Symlink handling complexity** - All libraries struggle with this
3. **Caching strategies** - fsspec's block-wise caching is sophisticated
4. **Async support** - fsspec shows how to do this well

**Critical security tests to add:**
1. URL-encoded path traversal (`%2e%2e`)
2. Symlink escape from sandboxed directories
3. Symlink loop detection
4. Deep path exhaustion

---

## Sources

### External Libraries
- [fsspec Documentation](https://filesystem-spec.readthedocs.io/)
- [PyFilesystem2 GitHub](https://github.com/PyFilesystem/pyfilesystem2)
- [Afero GitHub](https://github.com/spf13/afero)
- [Apache Commons VFS](https://commons.apache.org/vfs/)
- [System.IO.Abstractions GitHub](https://github.com/TestableIO/System.IO.Abstractions)
- [memfs GitHub](https://github.com/streamich/memfs)

### Sibling Rust Projects
- [soft-canonicalize-rs GitHub](https://github.com/DK26/soft-canonicalize-rs)
- [strict-path-rs GitHub](https://github.com/DK26/strict-path-rs)

### Vulnerability References
- [Apache Commons VFS CVEs (NVD search)](https://nvd.nist.gov/vuln/search/results?query=Apache%20Commons%20VFS)
- [Afero BasePathFs Issue #282](https://github.com/spf13/afero/issues/282)
- [PyFilesystem2 Symlink Loop Issue #171](https://github.com/PyFilesystem/pyfilesystem2/issues/171)
- [CVE-2018-1000178 (Zip Slip)](https://nvd.nist.gov/vuln/detail/CVE-2018-1000178)
- [CVE-2022-21658 (TOCTOU in Rust std)](https://nvd.nist.gov/vuln/detail/CVE-2022-21658)
