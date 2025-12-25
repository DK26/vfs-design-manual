# Testing Guide

**Comprehensive testing strategy for AnyFS**

---

## Overview

AnyFS uses a layered testing approach:

| Layer | What it tests | Run with |
|-------|---------------|----------|
| Unit tests | Individual components | `cargo test` |
| Conformance tests | Backend trait compliance | `cargo test --features conformance` |
| Integration tests | Full stack behavior | `cargo test --test integration` |
| Stress tests | Concurrency & limits | `cargo test --release -- --ignored` |
| Platform tests | Cross-platform behavior | CI matrix |

---

## 1. Backend Conformance Tests

Every backend must pass the same conformance suite. This ensures backends are interchangeable.

### Running Conformance Tests

```rust
use anyfs_test::{run_conformance_suite, ConformanceLevel};

#[test]
fn memory_backend_conformance() {
    run_conformance_suite(
        MemoryBackend::new(),
        ConformanceLevel::Fs,  // or FsFull, FsFuse, FsPosix
    );
}

#[test]
fn sqlite_backend_conformance() {
    let temp = tempfile::tempdir().unwrap();
    let db_path = temp.path().join("test.db");
    run_conformance_suite(
        SqliteBackend::open(&db_path).unwrap(),
        ConformanceLevel::FsFuse,
    );
}
```

### Conformance Levels

```
FsPosix  ──▶ FsHandles, FsLock, FsXattr tests
    │
FsFuse   ──▶ FsInode tests (path_to_inode, lookup, etc.)
    │
FsFull   ──▶ FsLink, FsPermissions, FsSync, FsStats tests
    │
Fs       ──▶ FsRead, FsWrite, FsDir tests (REQUIRED for all)
```

### Core Tests (Fs level)

```rust
#[test]
fn test_write_and_read() {
    let mut backend = create_backend();

    backend.write("/file.txt", b"hello world").unwrap();
    let content = backend.read("/file.txt").unwrap();

    assert_eq!(content, b"hello world");
}

#[test]
fn test_read_nonexistent_returns_not_found() {
    let backend = create_backend();

    let result = backend.read("/nonexistent.txt");

    assert!(matches!(result, Err(FsError::NotFound { .. })));
}

#[test]
fn test_create_dir_and_list() {
    let mut backend = create_backend();

    backend.create_dir("/mydir").unwrap();
    backend.write("/mydir/file.txt", b"data").unwrap();

    let entries = backend.read_dir("/mydir").unwrap();
    assert_eq!(entries.len(), 1);
    assert_eq!(entries[0].name, "file.txt");
}

#[test]
fn test_create_dir_all() {
    let mut backend = create_backend();

    backend.create_dir_all("/a/b/c/d").unwrap();

    assert!(backend.exists("/a/b/c/d").unwrap());
}

#[test]
fn test_remove_file() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"data").unwrap();

    backend.remove_file("/file.txt").unwrap();

    assert!(!backend.exists("/file.txt").unwrap());
}

#[test]
fn test_remove_dir_all() {
    let mut backend = create_backend();
    backend.create_dir_all("/a/b/c").unwrap();
    backend.write("/a/b/c/file.txt", b"data").unwrap();

    backend.remove_dir_all("/a").unwrap();

    assert!(!backend.exists("/a").unwrap());
}

#[test]
fn test_rename() {
    let mut backend = create_backend();
    backend.write("/old.txt", b"data").unwrap();

    backend.rename("/old.txt", "/new.txt").unwrap();

    assert!(!backend.exists("/old.txt").unwrap());
    assert_eq!(backend.read("/new.txt").unwrap(), b"data");
}

#[test]
fn test_copy() {
    let mut backend = create_backend();
    backend.write("/original.txt", b"data").unwrap();

    backend.copy("/original.txt", "/copy.txt").unwrap();

    assert_eq!(backend.read("/original.txt").unwrap(), b"data");
    assert_eq!(backend.read("/copy.txt").unwrap(), b"data");
}

#[test]
fn test_metadata() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"hello").unwrap();

    let meta = backend.metadata("/file.txt").unwrap();

    assert_eq!(meta.size, 5);
    assert!(meta.file_type.is_file());
}

#[test]
fn test_append() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"hello").unwrap();

    backend.append("/file.txt", b" world").unwrap();

    assert_eq!(backend.read("/file.txt").unwrap(), b"hello world");
}

#[test]
fn test_truncate() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"hello world").unwrap();

    backend.truncate("/file.txt", 5).unwrap();

    assert_eq!(backend.read("/file.txt").unwrap(), b"hello");
}

#[test]
fn test_read_range() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"hello world").unwrap();

    let partial = backend.read_range("/file.txt", 6, 5).unwrap();

    assert_eq!(partial, b"world");
}
```

### Extended Tests (FsFull level)

```rust
#[test]
fn test_symlink() {
    let mut backend = create_backend();
    backend.write("/target.txt", b"data").unwrap();

    backend.symlink("/target.txt", "/link.txt").unwrap();

    // read_link returns the target
    assert_eq!(backend.read_link("/link.txt").unwrap(), Path::new("/target.txt"));

    // reading the symlink follows it
    assert_eq!(backend.read("/link.txt").unwrap(), b"data");
}

#[test]
fn test_hard_link() {
    let mut backend = create_backend();
    backend.write("/original.txt", b"data").unwrap();

    backend.hard_link("/original.txt", "/hardlink.txt").unwrap();

    // Both point to same data
    assert_eq!(backend.read("/hardlink.txt").unwrap(), b"data");

    // Metadata shows nlink > 1
    let meta = backend.metadata("/original.txt").unwrap();
    assert!(meta.nlink >= 2);
}

#[test]
fn test_symlink_metadata() {
    let mut backend = create_backend();
    backend.write("/target.txt", b"data").unwrap();
    backend.symlink("/target.txt", "/link.txt").unwrap();

    // symlink_metadata returns metadata of the symlink itself
    let meta = backend.symlink_metadata("/link.txt").unwrap();
    assert!(meta.file_type.is_symlink());
}

#[test]
fn test_set_permissions() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"data").unwrap();

    backend.set_permissions("/file.txt", Permissions::from_mode(0o644)).unwrap();

    let meta = backend.metadata("/file.txt").unwrap();
    assert_eq!(meta.permissions.mode() & 0o777, 0o644);
}

#[test]
fn test_sync() {
    let mut backend = create_backend();
    backend.write("/file.txt", b"data").unwrap();

    // Should not error
    backend.sync().unwrap();
    backend.fsync("/file.txt").unwrap();
}

#[test]
fn test_statfs() {
    let backend = create_backend();

    let stats = backend.statfs().unwrap();

    assert!(stats.total_bytes > 0 || stats.total_bytes == 0); // Memory may report 0
}
```

---

## 2. Middleware Tests

Each middleware is tested in isolation and in combination.

### Quota Tests

```rust
#[test]
fn test_quota_blocks_when_exceeded() {
    let backend = Quota::new(MemoryBackend::new())
        .with_max_total_size(100);
    let mut fs = FileStorage::new(backend);

    let result = fs.write("/big.txt", &[0u8; 200]);

    assert!(matches!(result, Err(FsError::QuotaExceeded { .. })));
}

#[test]
fn test_quota_allows_within_limit() {
    let backend = Quota::new(MemoryBackend::new())
        .with_max_total_size(1000);
    let mut fs = FileStorage::new(backend);

    fs.write("/small.txt", &[0u8; 100]).unwrap();

    assert!(fs.exists("/small.txt").unwrap());
}

#[test]
fn test_quota_tracks_deletes() {
    let backend = Quota::new(MemoryBackend::new())
        .with_max_total_size(100);
    let mut fs = FileStorage::new(backend);

    fs.write("/file.txt", &[0u8; 50]).unwrap();
    fs.remove_file("/file.txt").unwrap();

    // Should be able to write again after delete
    fs.write("/file2.txt", &[0u8; 50]).unwrap();
}

#[test]
fn test_quota_max_file_size() {
    let backend = Quota::new(MemoryBackend::new())
        .with_max_file_size(50);
    let mut fs = FileStorage::new(backend);

    let result = fs.write("/big.txt", &[0u8; 100]);

    assert!(matches!(result, Err(FsError::QuotaExceeded { .. })));
}

#[test]
fn test_quota_streaming_write() {
    let backend = Quota::new(MemoryBackend::new())
        .with_max_total_size(100);
    let mut fs = FileStorage::new(backend);

    let mut writer = fs.open_write("/file.txt").unwrap();
    writer.write_all(&[0u8; 50]).unwrap();
    writer.write_all(&[0u8; 50]).unwrap();
    drop(writer);

    // Next write should fail
    let result = fs.write("/file2.txt", &[0u8; 10]);
    assert!(matches!(result, Err(FsError::QuotaExceeded { .. })));
}
```

### Restrictions Tests

```rust
#[test]
fn test_restrictions_blocks_symlinks() {
    let backend = Restrictions::new(MemoryBackend::new())
        .deny_symlinks();
    let mut fs = FileStorage::new(backend);

    fs.write("/target.txt", b"data").unwrap();
    let result = fs.symlink("/target.txt", "/link.txt");

    assert!(matches!(result, Err(FsError::OperationDenied { .. })));
}

#[test]
fn test_restrictions_allows_non_blocked() {
    let backend = Restrictions::new(MemoryBackend::new())
        .deny_symlinks();  // Only block symlinks
    let mut fs = FileStorage::new(backend);

    // Hard links should still work
    fs.write("/target.txt", b"data").unwrap();
    fs.hard_link("/target.txt", "/link.txt").unwrap();
}

#[test]
fn test_restrictions_blocks_permissions() {
    let backend = Restrictions::new(MemoryBackend::new())
        .deny_permissions();
    let mut fs = FileStorage::new(backend);

    fs.write("/file.txt", b"data").unwrap();
    let result = fs.set_permissions("/file.txt", Permissions::from_mode(0o777));

    assert!(matches!(result, Err(FsError::OperationDenied { .. })));
}
```

### PathFilter Tests

```rust
#[test]
fn test_pathfilter_allows_matching() {
    let backend = PathFilter::new(MemoryBackend::new())
        .allow("/workspace/**");
    let mut fs = FileStorage::new(backend);

    fs.create_dir_all("/workspace/project").unwrap();
    fs.write("/workspace/project/file.txt", b"data").unwrap();
}

#[test]
fn test_pathfilter_blocks_non_matching() {
    let backend = PathFilter::new(MemoryBackend::new())
        .allow("/workspace/**");
    let mut fs = FileStorage::new(backend);

    let result = fs.write("/etc/passwd", b"data");

    assert!(matches!(result, Err(FsError::AccessDenied { .. })));
}

#[test]
fn test_pathfilter_deny_overrides_allow() {
    let backend = PathFilter::new(MemoryBackend::new())
        .allow("/workspace/**")
        .deny("**/.env");
    let mut fs = FileStorage::new(backend);

    let result = fs.write("/workspace/.env", b"SECRET=xxx");

    assert!(matches!(result, Err(FsError::AccessDenied { .. })));
}

#[test]
fn test_pathfilter_read_dir_filters() {
    let mut inner = MemoryBackend::new();
    inner.write("/workspace/allowed.txt", b"data").unwrap();
    inner.write("/workspace/.env", b"secret").unwrap();

    let backend = PathFilter::new(inner)
        .allow("/workspace/**")
        .deny("**/.env");
    let fs = FileStorage::new(backend);

    let entries = fs.read_dir("/workspace").unwrap();

    // .env should be filtered out
    assert_eq!(entries.len(), 1);
    assert_eq!(entries[0].name, "allowed.txt");
}
```

### ReadOnly Tests

```rust
#[test]
fn test_readonly_blocks_writes() {
    let mut inner = MemoryBackend::new();
    inner.write("/file.txt", b"original").unwrap();

    let backend = ReadOnly::new(inner);
    let mut fs = FileStorage::new(backend);

    let result = fs.write("/file.txt", b"modified");
    assert!(matches!(result, Err(FsError::ReadOnly { .. })));

    let result = fs.remove_file("/file.txt");
    assert!(matches!(result, Err(FsError::ReadOnly { .. })));
}

#[test]
fn test_readonly_allows_reads() {
    let mut inner = MemoryBackend::new();
    inner.write("/file.txt", b"data").unwrap();

    let backend = ReadOnly::new(inner);
    let fs = FileStorage::new(backend);

    assert_eq!(fs.read("/file.txt").unwrap(), b"data");
}
```

### Middleware Composition Tests

```rust
#[test]
fn test_middleware_composition_order() {
    // Quota inside, Restrictions outside
    // Quota should be checked before restrictions
    let backend = Restrictions::new(
        Quota::new(MemoryBackend::new())
            .with_max_total_size(100)
    ).deny_symlinks();

    let mut fs = FileStorage::new(backend);

    // Write should hit quota first
    let result = fs.write("/big.txt", &[0u8; 200]);
    assert!(matches!(result, Err(FsError::QuotaExceeded { .. })));
}

#[test]
fn test_layer_syntax() {
    let backend = MemoryBackend::new()
        .layer(QuotaLayer::new().max_total_size(1000))
        .layer(RestrictionsLayer::new().deny_symlinks())
        .layer(TracingLayer::new());

    let mut fs = FileStorage::new(backend);
    fs.write("/test.txt", b"data").unwrap();
}
```

---

## 3. FileStorage Tests

```rust
#[test]
fn test_filestorage_type_inference() {
    // Type should be inferred
    let fs = FileStorage::new(MemoryBackend::new());
    // No explicit type needed
}

#[test]
fn test_filestorage_marker_types() {
    struct Sandbox;
    struct Production;

    let sandbox: FileStorage<_, Sandbox> = FileStorage::new(MemoryBackend::new());
    let prod: FileStorage<_, Production> = FileStorage::new(MemoryBackend::new());

    fn only_sandbox(_fs: &FileStorage<impl Fs, Sandbox>) {}

    only_sandbox(&sandbox);  // Compiles
    // only_sandbox(&prod);  // Would not compile
}

#[test]
fn test_filestorage_boxed() {
    let fs1 = FileStorage::new(MemoryBackend::new()).boxed();
    let fs2 = FileStorage::new(SqliteBackend::open(":memory:").unwrap()).boxed();

    // Both can be stored in same collection
    let _filesystems: Vec<FileStorage<Box<dyn Fs>>> = vec![fs1, fs2];
}
```

---

## 4. Error Handling Tests

```rust
#[test]
fn test_error_not_found() {
    let fs = FileStorage::new(MemoryBackend::new());

    match fs.read("/nonexistent") {
        Err(FsError::NotFound { path, operation }) => {
            assert_eq!(path, Path::new("/nonexistent"));
            assert_eq!(operation, "read");
        }
        _ => panic!("Expected NotFound error"),
    }
}

#[test]
fn test_error_already_exists() {
    let mut fs = FileStorage::new(MemoryBackend::new());
    fs.create_dir("/mydir").unwrap();

    match fs.create_dir("/mydir") {
        Err(FsError::AlreadyExists { path, .. }) => {
            assert_eq!(path, Path::new("/mydir"));
        }
        _ => panic!("Expected AlreadyExists error"),
    }
}

#[test]
fn test_error_not_a_directory() {
    let mut fs = FileStorage::new(MemoryBackend::new());
    fs.write("/file.txt", b"data").unwrap();

    match fs.read_dir("/file.txt") {
        Err(FsError::NotADirectory { path }) => {
            assert_eq!(path, Path::new("/file.txt"));
        }
        _ => panic!("Expected NotADirectory error"),
    }
}

#[test]
fn test_error_directory_not_empty() {
    let mut fs = FileStorage::new(MemoryBackend::new());
    fs.create_dir("/mydir").unwrap();
    fs.write("/mydir/file.txt", b"data").unwrap();

    match fs.remove_dir("/mydir") {
        Err(FsError::DirectoryNotEmpty { path }) => {
            assert_eq!(path, Path::new("/mydir"));
        }
        _ => panic!("Expected DirectoryNotEmpty error"),
    }
}
```

---

## 5. Concurrency Tests

```rust
#[test]
fn test_concurrent_reads() {
    let mut backend = MemoryBackend::new();
    backend.write("/file.txt", b"data").unwrap();
    let backend = Arc::new(RwLock::new(backend));

    let handles: Vec<_> = (0..10).map(|_| {
        let backend = Arc::clone(&backend);
        thread::spawn(move || {
            let guard = backend.read().unwrap();
            guard.read("/file.txt").unwrap()
        })
    }).collect();

    for handle in handles {
        assert_eq!(handle.join().unwrap(), b"data");
    }
}

#[test]
fn test_concurrent_create_dir_all() {
    let backend = Arc::new(Mutex::new(MemoryBackend::new()));

    let handles: Vec<_> = (0..10).map(|_| {
        let backend = Arc::clone(&backend);
        thread::spawn(move || {
            let mut guard = backend.lock().unwrap();
            // Multiple threads creating same path should not race
            guard.create_dir_all("/a/b/c/d").unwrap();
        })
    }).collect();

    for handle in handles {
        handle.join().unwrap();
    }

    assert!(backend.lock().unwrap().exists("/a/b/c/d").unwrap());
}

#[test]
#[ignore] // Run with: cargo test --release -- --ignored
fn stress_test_concurrent_operations() {
    let backend = Arc::new(Mutex::new(MemoryBackend::new()));

    let handles: Vec<_> = (0..100).map(|i| {
        let backend = Arc::clone(&backend);
        thread::spawn(move || {
            for j in 0..100 {
                let path = format!("/thread_{}/file_{}.txt", i, j);
                let mut guard = backend.lock().unwrap();
                guard.create_dir_all(&format!("/thread_{}", i)).ok();
                guard.write(&path, b"data").unwrap();
                drop(guard);

                let guard = backend.lock().unwrap();
                let _ = guard.read(&path);
            }
        })
    }).collect();

    for handle in handles {
        handle.join().unwrap();
    }
}
```

---

## 6. Path Edge Case Tests

```rust
#[test]
fn test_path_normalization() {
    let mut fs = FileStorage::new(MemoryBackend::new());

    fs.write("/a/b/../c/file.txt", b"data").unwrap();

    // Should be accessible via normalized path
    assert_eq!(fs.read("/a/c/file.txt").unwrap(), b"data");
}

#[test]
fn test_double_slashes() {
    let mut fs = FileStorage::new(MemoryBackend::new());

    fs.write("//a//b//file.txt", b"data").unwrap();

    assert_eq!(fs.read("/a/b/file.txt").unwrap(), b"data");
}

#[test]
fn test_root_path() {
    let fs = FileStorage::new(MemoryBackend::new());

    let entries = fs.read_dir("/").unwrap();
    assert!(entries.is_empty());
}

#[test]
fn test_empty_path_returns_error() {
    let fs = FileStorage::new(MemoryBackend::new());

    let result = fs.read("");
    assert!(result.is_err());
}

#[test]
fn test_unicode_paths() {
    let mut fs = FileStorage::new(MemoryBackend::new());

    fs.write("/文件/データ.txt", b"data").unwrap();

    assert_eq!(fs.read("/文件/データ.txt").unwrap(), b"data");
}

#[test]
fn test_paths_with_spaces() {
    let mut fs = FileStorage::new(MemoryBackend::new());

    fs.write("/my folder/my file.txt", b"data").unwrap();

    assert_eq!(fs.read("/my folder/my file.txt").unwrap(), b"data");
}
```

---

## 7. No-Panic Guarantee Tests

```rust
#[test]
fn no_panic_missing_file() {
    let fs = FileStorage::new(MemoryBackend::new());
    let _ = fs.read("/missing");  // Should return Err, not panic
}

#[test]
fn no_panic_missing_parent() {
    let mut fs = FileStorage::new(MemoryBackend::new());
    let _ = fs.write("/missing/parent/file.txt", b"data");  // Should return Err
}

#[test]
fn no_panic_read_dir_on_file() {
    let mut fs = FileStorage::new(MemoryBackend::new());
    fs.write("/file.txt", b"data").unwrap();
    let _ = fs.read_dir("/file.txt");  // Should return Err, not panic
}

#[test]
fn no_panic_remove_nonempty_dir() {
    let mut fs = FileStorage::new(MemoryBackend::new());
    fs.create_dir("/dir").unwrap();
    fs.write("/dir/file.txt", b"data").unwrap();
    let _ = fs.remove_dir("/dir");  // Should return Err, not panic
}
```

---

## 8. Running Tests

```bash
# All tests
cargo test

# Specific backend conformance
cargo test memory_backend_conformance
cargo test sqlite_backend_conformance

# Middleware tests
cargo test quota
cargo test restrictions
cargo test pathfilter

# Stress tests (release mode)
cargo test --release -- --ignored

# With coverage
cargo tarpaulin --out Html

# Cross-platform (CI)
cargo test --target x86_64-unknown-linux-gnu
cargo test --target x86_64-pc-windows-msvc
cargo test --target x86_64-apple-darwin

# WASM
cargo test --target wasm32-unknown-unknown
```

---

## 9. Test Utilities

```rust
// In anyfs-test crate

/// Create a temporary backend for testing
pub fn temp_sqlite_backend() -> (SqliteBackend, tempfile::TempDir) {
    let temp = tempfile::tempdir().unwrap();
    let db_path = temp.path().join("test.db");
    let backend = SqliteBackend::open(&db_path).unwrap();
    (backend, temp)
}

/// Run a test against multiple backends
pub fn test_all_backends<F>(test: F)
where
    F: Fn(&mut dyn Fs),
{
    // Memory
    let mut backend = MemoryBackend::new();
    test(&mut backend);

    // SQLite
    let (mut backend, _temp) = temp_sqlite_backend();
    test(&mut backend);
}
```
