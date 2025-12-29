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
    let backend = MemoryBackend::new()
        .layer(QuotaLayer::builder().max_total_size(100).build());
    let mut fs = FileStorage::new(backend);

    let result = fs.write("/big.txt", &[0u8; 200]);

    assert!(matches!(result, Err(FsError::QuotaExceeded { .. })));
}

#[test]
fn test_quota_allows_within_limit() {
    let backend = MemoryBackend::new()
        .layer(QuotaLayer::builder().max_total_size(1000).build());
    let mut fs = FileStorage::new(backend);

    fs.write("/small.txt", &[0u8; 100]).unwrap();

    assert!(fs.exists("/small.txt").unwrap());
}

#[test]
fn test_quota_tracks_deletes() {
    let backend = MemoryBackend::new()
        .layer(QuotaLayer::builder().max_total_size(100).build());
    let mut fs = FileStorage::new(backend);

    fs.write("/file.txt", &[0u8; 50]).unwrap();
    fs.remove_file("/file.txt").unwrap();

    // Should be able to write again after delete
    fs.write("/file2.txt", &[0u8; 50]).unwrap();
}

#[test]
fn test_quota_max_file_size() {
    let backend = MemoryBackend::new()
        .layer(QuotaLayer::builder().max_file_size(50).build());
    let mut fs = FileStorage::new(backend);

    let result = fs.write("/big.txt", &[0u8; 100]);

    assert!(matches!(result, Err(FsError::QuotaExceeded { .. })));
}

#[test]
fn test_quota_streaming_write() {
    let backend = MemoryBackend::new()
        .layer(QuotaLayer::builder().max_total_size(100).build());
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
    let backend = MemoryBackend::new()
        .layer(RestrictionsLayer::builder().deny_symlinks().build());
    let mut fs = FileStorage::new(backend);

    fs.write("/target.txt", b"data").unwrap();
    let result = fs.symlink("/target.txt", "/link.txt");

    assert!(matches!(result, Err(FsError::OperationDenied { .. })));
}

#[test]
fn test_restrictions_allows_non_blocked() {
    let backend = MemoryBackend::new()
        .layer(RestrictionsLayer::builder().deny_symlinks().build());  // Only block symlinks
    let mut fs = FileStorage::new(backend);

    // Hard links should still work
    fs.write("/target.txt", b"data").unwrap();
    fs.hard_link("/target.txt", "/link.txt").unwrap();
}

#[test]
fn test_restrictions_blocks_permissions() {
    let backend = MemoryBackend::new()
        .layer(RestrictionsLayer::builder().deny_permissions().build());
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
    let backend = MemoryBackend::new()
        .layer(PathFilterLayer::builder().allow("/workspace/**").build());
    let mut fs = FileStorage::new(backend);

    fs.create_dir_all("/workspace/project").unwrap();
    fs.write("/workspace/project/file.txt", b"data").unwrap();
}

#[test]
fn test_pathfilter_blocks_non_matching() {
    let backend = MemoryBackend::new()
        .layer(PathFilterLayer::builder().allow("/workspace/**").build());
    let mut fs = FileStorage::new(backend);

    let result = fs.write("/etc/passwd", b"data");

    assert!(matches!(result, Err(FsError::AccessDenied { .. })));
}

#[test]
fn test_pathfilter_deny_overrides_allow() {
    let backend = MemoryBackend::new()
        .layer(PathFilterLayer::builder()
            .allow("/workspace/**")
            .deny("**/.env")
            .build());
    let mut fs = FileStorage::new(backend);

    let result = fs.write("/workspace/.env", b"SECRET=xxx");

    assert!(matches!(result, Err(FsError::AccessDenied { .. })));
}

#[test]
fn test_pathfilter_read_dir_filters() {
    let mut inner = MemoryBackend::new();
    inner.write("/workspace/allowed.txt", b"data").unwrap();
    inner.write("/workspace/.env", b"secret").unwrap();

    let backend = inner
        .layer(PathFilterLayer::builder()
            .allow("/workspace/**")
            .deny("**/.env")
            .build());
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
    let backend = MemoryBackend::new()
        .layer(QuotaLayer::builder().max_total_size(100).build())
        .layer(RestrictionsLayer::builder().deny_symlinks().build());

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

## 8. Symlink Security Tests

```rust
// Virtual backend symlink following control
#[test]
fn test_virtual_backend_follow_symlinks_enabled() {
    let mut backend = MemoryBackend::new();
    backend.write("/target.txt", b"secret").unwrap();
    backend.symlink("/target.txt", "/link.txt").unwrap();

    // Default: following enabled
    assert_eq!(backend.read("/link.txt").unwrap(), b"secret");
}

#[test]
fn test_virtual_backend_follow_symlinks_disabled() {
    let mut backend = MemoryBackend::new();
    backend.set_follow_symlinks(false);
    backend.write("/target.txt", b"secret").unwrap();
    backend.symlink("/target.txt", "/link.txt").unwrap();

    // Reading symlink should fail or return symlink data
    let result = backend.read("/link.txt");
    assert!(matches!(result, Err(FsError::IsSymlink { .. })));
}

#[test]
fn test_symlink_chain_resolution() {
    let mut backend = MemoryBackend::new();
    backend.write("/target.txt", b"data").unwrap();
    backend.symlink("/target.txt", "/link1.txt").unwrap();
    backend.symlink("/link1.txt", "/link2.txt").unwrap();

    // Should follow chain
    assert_eq!(backend.read("/link2.txt").unwrap(), b"data");
}

#[test]
fn test_symlink_loop_detection() {
    let mut backend = MemoryBackend::new();
    backend.symlink("/link2.txt", "/link1.txt").unwrap();
    backend.symlink("/link1.txt", "/link2.txt").unwrap();

    let result = backend.read("/link1.txt");
    assert!(matches!(result, Err(FsError::SymlinkLoop { .. })));
}

#[test]
fn test_virtual_symlink_cannot_escape() {
    let mut backend = MemoryBackend::new();
    // Create a symlink pointing "outside" - but in virtual backend, paths are just keys
    backend.symlink("../../../etc/passwd", "/link.txt").unwrap();

    // Reading should fail (target doesn't exist), not read real /etc/passwd
    let result = backend.read("/link.txt");
    assert!(matches!(result, Err(FsError::NotFound { .. })));
}
```

### VRootFsBackend Containment Tests

```rust
#[test]
fn test_vroot_prevents_path_traversal() {
    let temp = tempfile::tempdir().unwrap();
    let backend = VRootFsBackend::new(temp.path()).unwrap();
    let fs = FileStorage::new(backend);

    // Attempt to escape via ..
    let result = fs.read("/../../../etc/passwd");
    assert!(matches!(result, Err(FsError::AccessDenied { .. })));
}

#[test]
fn test_vroot_prevents_symlink_escape() {
    let temp = tempfile::tempdir().unwrap();
    std::fs::write(temp.path().join("file.txt"), b"data").unwrap();

    // Create symlink pointing outside the jail
    #[cfg(unix)]
    std::os::unix::fs::symlink("/etc/passwd", temp.path().join("escape")).unwrap();

    let backend = VRootFsBackend::new(temp.path()).unwrap();
    let fs = FileStorage::new(backend);

    // Reading should be blocked by strict-path
    let result = fs.read("/escape");
    assert!(matches!(result, Err(FsError::AccessDenied { .. })));
}

#[test]
fn test_vroot_allows_internal_symlinks() {
    let temp = tempfile::tempdir().unwrap();
    std::fs::write(temp.path().join("target.txt"), b"data").unwrap();

    #[cfg(unix)]
    std::os::unix::fs::symlink("target.txt", temp.path().join("link.txt")).unwrap();

    let backend = VRootFsBackend::new(temp.path()).unwrap();
    let fs = FileStorage::new(backend);

    // Internal symlinks should work
    assert_eq!(fs.read("/link.txt").unwrap(), b"data");
}

#[test]
fn test_vroot_canonicalizes_paths() {
    let temp = tempfile::tempdir().unwrap();
    let backend = VRootFsBackend::new(temp.path()).unwrap();
    let mut fs = FileStorage::new(backend);

    fs.create_dir("/a").unwrap();
    fs.write("/a/file.txt", b"data").unwrap();

    // Access via normalized path
    assert_eq!(fs.read("/a/../a/./file.txt").unwrap(), b"data");
}
```

---

## 9. RateLimit Middleware Tests

```rust
#[test]
fn test_ratelimit_allows_within_limit() {
    let backend = MemoryBackend::new()
        .layer(RateLimitLayer::builder().max_ops(10).per_second().build());
    let mut fs = FileStorage::new(backend);

    // Should succeed within limit
    for i in 0..5 {
        fs.write(&format!("/file{}.txt", i), b"data").unwrap();
    }
}

#[test]
fn test_ratelimit_blocks_when_exceeded() {
    let backend = MemoryBackend::new()
        .layer(RateLimitLayer::builder().max_ops(3).per_second().build());
    let mut fs = FileStorage::new(backend);

    fs.write("/file1.txt", b"data").unwrap();
    fs.write("/file2.txt", b"data").unwrap();
    fs.write("/file3.txt", b"data").unwrap();

    let result = fs.write("/file4.txt", b"data");
    assert!(matches!(result, Err(FsError::RateLimitExceeded { .. })));
}

#[test]
fn test_ratelimit_resets_after_window() {
    let backend = MemoryBackend::new()
        .layer(RateLimitLayer::builder().max_ops(2).per(Duration::from_millis(100)).build());
    let mut fs = FileStorage::new(backend);

    fs.write("/file1.txt", b"data").unwrap();
    fs.write("/file2.txt", b"data").unwrap();

    // Wait for window to reset
    std::thread::sleep(Duration::from_millis(150));

    // Should succeed again
    fs.write("/file3.txt", b"data").unwrap();
}

#[test]
fn test_ratelimit_counts_all_operations() {
    let backend = MemoryBackend::new()
        .layer(RateLimitLayer::builder().max_ops(3).per_second().build());
    let mut fs = FileStorage::new(backend);

    fs.write("/file.txt", b"data").unwrap();  // 1
    let _ = fs.read("/file.txt");              // 2
    let _ = fs.exists("/file.txt");            // 3

    let result = fs.metadata("/file.txt");
    assert!(matches!(result, Err(FsError::RateLimitExceeded { .. })));
}
```

---

## 10. Tracing Middleware Tests

```rust
use std::sync::{Arc, Mutex};

#[derive(Default)]
struct TestLogger {
    logs: Arc<Mutex<Vec<String>>>,
}

impl TestLogger {
    fn entries(&self) -> Vec<String> {
        self.logs.lock().unwrap().clone()
    }
}

#[test]
fn test_tracing_logs_operations() {
    let logger = TestLogger::default();
    let logs = Arc::clone(&logger.logs);

    let backend = MemoryBackend::new()
        .layer(TracingLayer::new()
            .with_logger(move |op| {
                logs.lock().unwrap().push(op.to_string());
            }));
    let mut fs = FileStorage::new(backend);

    fs.write("/file.txt", b"data").unwrap();
    fs.read("/file.txt").unwrap();

    let entries = logger.entries();
    assert!(entries.iter().any(|e| e.contains("write")));
    assert!(entries.iter().any(|e| e.contains("read")));
}

#[test]
fn test_tracing_includes_path() {
    let logger = TestLogger::default();
    let logs = Arc::clone(&logger.logs);

    let backend = MemoryBackend::new()
        .layer(TracingLayer::new()
            .with_logger(move |op| {
                logs.lock().unwrap().push(op.to_string());
            }));
    let mut fs = FileStorage::new(backend);

    fs.write("/important/secret.txt", b"data").unwrap();

    let entries = logger.entries();
    assert!(entries.iter().any(|e| e.contains("/important/secret.txt")));
}

#[test]
fn test_tracing_logs_errors() {
    let logger = TestLogger::default();
    let logs = Arc::clone(&logger.logs);

    let backend = MemoryBackend::new()
        .layer(TracingLayer::new()
            .with_logger(move |op| {
                logs.lock().unwrap().push(op.to_string());
            }));
    let fs = FileStorage::new(backend);

    let _ = fs.read("/nonexistent.txt");

    let entries = logger.entries();
    assert!(entries.iter().any(|e| e.contains("NotFound") || e.contains("error")));
}

#[test]
fn test_tracing_with_span_context() {
    use tracing::{info_span, Instrument};

    let backend = MemoryBackend::new().layer(TracingLayer::new());
    let mut fs = FileStorage::new(backend);

    async {
        fs.write("/async.txt", b"data").unwrap();
    }
    .instrument(info_span!("test_operation"))
    .now_or_never();
}
```

---

## 11. Backend Interchangeability Tests

```rust
/// Ensure all backends can be used interchangeably
fn generic_filesystem_test<B: Fs>(mut backend: B) {
    backend.create_dir("/test").unwrap();
    backend.write("/test/file.txt", b"hello").unwrap();
    assert_eq!(backend.read("/test/file.txt").unwrap(), b"hello");
    backend.remove_dir_all("/test").unwrap();
    assert!(!backend.exists("/test").unwrap());
}

#[test]
fn test_memory_backend_interchangeable() {
    generic_filesystem_test(MemoryBackend::new());
}

#[test]
fn test_sqlite_backend_interchangeable() {
    let (backend, _temp) = temp_sqlite_backend();
    generic_filesystem_test(backend);
}

#[test]
fn test_vroot_backend_interchangeable() {
    let temp = tempfile::tempdir().unwrap();
    let backend = VRootFsBackend::new(temp.path()).unwrap();
    generic_filesystem_test(backend);
}

#[test]
fn test_middleware_stack_interchangeable() {
    let backend = MemoryBackend::new()
        .layer(QuotaLayer::builder()
            .max_total_size(1024 * 1024)
            .build())
        .layer(TracingLayer::new());
    generic_filesystem_test(backend);
}
```

---

## 12. Property-Based Tests

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn prop_write_read_roundtrip(data: Vec<u8>) {
        let mut backend = MemoryBackend::new();
        backend.write("/file.bin", &data).unwrap();
        let read_data = backend.read("/file.bin").unwrap();
        prop_assert_eq!(data, read_data);
    }

    #[test]
    fn prop_path_normalization_idempotent(path in "[a-z/]{1,50}") {
        let mut backend = MemoryBackend::new();
        if let Ok(()) = backend.create_dir_all(&path) {
            // Creating again should either succeed or return AlreadyExists
            let result = backend.create_dir_all(&path);
            prop_assert!(result.is_ok() || matches!(result, Err(FsError::AlreadyExists { .. })));
        }
    }

    #[test]
    fn prop_quota_never_exceeds_limit(
        file_count in 1..10usize,
        file_sizes in prop::collection::vec(1..100usize, 1..10)
    ) {
        let limit = 500usize;
        let backend = MemoryBackend::new()
            .layer(QuotaLayer::builder().max_total_size(limit as u64).build());
        let mut fs = FileStorage::new(backend);

        let mut total_written = 0usize;
        for (i, size) in file_sizes.into_iter().take(file_count).enumerate() {
            let data = vec![0u8; size];
            match fs.write(&format!("/file{}.txt", i), &data) {
                Ok(()) => total_written += size,
                Err(FsError::QuotaExceeded { .. }) => break,
                Err(e) => panic!("Unexpected error: {:?}", e),
            }
        }
        prop_assert!(total_written <= limit);
    }
}
```

---

## 13. Snapshot & Restore Tests

```rust
// MemoryBackend implements Clone - that's the snapshot mechanism
#[test]
fn test_clone_creates_independent_copy() {
    let mut original = MemoryBackend::new();
    original.write("/file.txt", b"original").unwrap();

    // Clone = snapshot
    let mut snapshot = original.clone();

    // Modify original
    original.write("/file.txt", b"modified").unwrap();
    original.write("/new.txt", b"new").unwrap();

    // Snapshot is unchanged
    assert_eq!(snapshot.read("/file.txt").unwrap(), b"original");
    assert!(!snapshot.exists("/new.txt").unwrap());
}

#[test]
fn test_checkpoint_and_rollback() {
    let mut fs = MemoryBackend::new();
    fs.write("/important.txt", b"original").unwrap();

    // Checkpoint = clone
    let checkpoint = fs.clone();

    // Do risky work
    fs.write("/important.txt", b"corrupted").unwrap();

    // Rollback = replace with checkpoint
    fs = checkpoint;
    assert_eq!(fs.read("/important.txt").unwrap(), b"original");
}

#[test]
fn test_persistence_roundtrip() {
    let temp = tempfile::tempdir().unwrap();
    let path = temp.path().join("state.bin");

    let mut fs = MemoryBackend::new();
    fs.write("/data.txt", b"persisted").unwrap();

    // Save
    fs.save_to(&path).unwrap();

    // Load
    let restored = MemoryBackend::load_from(&path).unwrap();
    assert_eq!(restored.read("/data.txt").unwrap(), b"persisted");
}

#[test]
fn test_to_bytes_from_bytes() {
    let mut fs = MemoryBackend::new();
    fs.create_dir_all("/a/b/c").unwrap();
    fs.write("/a/b/c/file.txt", b"nested").unwrap();

    let bytes = fs.to_bytes().unwrap();
    let restored = MemoryBackend::from_bytes(&bytes).unwrap();

    assert_eq!(restored.read("/a/b/c/file.txt").unwrap(), b"nested");
}

#[test]
fn test_from_bytes_invalid_data() {
    let result = MemoryBackend::from_bytes(b"garbage");
    assert!(result.is_err());
}
```

---

## 14. Running Tests

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
