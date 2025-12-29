# Remote Backend Patterns

**Building networked filesystem backends and clients**

This guide covers patterns for exposing AnyFS backends over a network and building clients that mount remote filesystems.

---

## Overview

A remote filesystem has three components:

```
┌─────────────┐      Network       ┌─────────────┐      ┌─────────────┐
│   Client    │ ←───────────────→  │   Server    │  ──→ │   Backend   │
│  (FUSE)     │     RPC/REST       │   (API)     │      │  (Storage)  │
└─────────────┘                    └─────────────┘      └─────────────┘
     User's                           Cloud               SQLite/CAS
     Machine                          Service             Hybrid/etc
```

AnyFS backends are local by design. To go remote, you need:
1. **Server**: Exposes backend operations over network
2. **Protocol**: Wire format for requests/responses
3. **Client**: Implements `Fs` traits by calling server

---

## Protocol Design

### Operations to Expose

Map `Fs` trait methods to RPC operations:

| Trait Method | RPC Operation | Notes |
|--------------|---------------|-------|
| `read(path)` | `Read(path, range?)` | Support partial reads |
| `write(path, data)` | `Write(path, data)` | Chunked for large files |
| `exists(path)` | `Exists(path)` | Or combine with Metadata |
| `metadata(path)` | `Metadata(path)` | Return full stat |
| `read_dir(path)` | `ListDir(path, cursor?)` | Paginated for large dirs |
| `create_dir(path)` | `CreateDir(path)` | |
| `create_dir_all(path)` | `CreateDirAll(path)` | Or client-side loop |
| `remove_file(path)` | `Remove(path)` | |
| `remove_dir(path)` | `RemoveDir(path)` | |
| `remove_dir_all(path)` | `RemoveDirAll(path)` | Recursive |
| `rename(from, to)` | `Rename(from, to)` | |
| `copy(from, to)` | `Copy(from, to)` | Server-side copy |

### Request/Response Format

Use a simple, efficient format. Here's a protobuf-style schema:

```protobuf
// requests.proto

message Request {
  string request_id = 1;  // For idempotency
  string auth_token = 2;  // Authentication

  oneof operation {
    ReadRequest read = 10;
    WriteRequest write = 11;
    MetadataRequest metadata = 12;
    ListDirRequest list_dir = 13;
    CreateDirRequest create_dir = 14;
    RemoveRequest remove = 15;
    RenameRequest rename = 16;
    CopyRequest copy = 17;
  }
}

message ReadRequest {
  string path = 1;
  optional uint64 offset = 2;
  optional uint64 length = 3;
}

message WriteRequest {
  string path = 1;
  bytes data = 2;
  bool append = 3;
}

message MetadataRequest {
  string path = 1;
}

message ListDirRequest {
  string path = 1;
  optional string cursor = 2;  // For pagination
  optional uint32 limit = 3;
}

// ... other requests

message Response {
  string request_id = 1;
  bool success = 2;

  oneof result {
    ErrorResult error = 10;
    ReadResult read = 11;
    WriteResult write = 12;
    MetadataResult metadata = 13;
    ListDirResult list_dir = 14;
    // ... others return empty success
  }
}

message ErrorResult {
  string code = 1;    // "not_found", "permission_denied", etc.
  string message = 2;
  string path = 3;
}

message MetadataResult {
  string file_type = 1;  // "file", "dir", "symlink"
  uint64 size = 2;
  uint32 mode = 3;
  optional uint64 created_at = 4;
  optional uint64 modified_at = 5;
  optional uint64 inode = 6;
}

message ListDirResult {
  repeated DirEntry entries = 1;
  optional string next_cursor = 2;  // Null if no more
}

message DirEntry {
  string name = 1;
  string file_type = 2;
  uint64 size = 3;
}
```

### Protocol Choices

| Protocol | Pros | Cons | Use When |
|----------|------|------|----------|
| **gRPC** | Fast, typed, streaming | Complex setup | High performance |
| **REST/JSON** | Simple, debuggable | Slower, no streaming | Compatibility |
| **WebSocket** | Bidirectional, real-time | More complex | Live updates |
| **Custom TCP** | Maximum control | Build everything | Special needs |

**Recommendation:** Start with gRPC (tonic in Rust). Fall back to REST for web clients.

---

## Server Implementation

### Basic Server Structure

```rust
use tonic::{transport::Server, Request, Response, Status};
use anyfs_backend::Fs;

pub struct FsServer<B: Fs> {
    backend: B,
}

impl<B: Fs + Send + Sync + 'static> FsServer<B> {
    pub fn new(backend: B) -> Self {
        Self { backend }
    }

    pub async fn serve(self, addr: &str) -> Result<(), Box<dyn std::error::Error>> {
        let addr = addr.parse()?;

        Server::builder()
            .add_service(FsServiceServer::new(self))
            .serve(addr)
            .await?;

        Ok(())
    }
}

#[tonic::async_trait]
impl<B: Fs + Send + Sync + 'static> FsService for FsServer<B> {
    async fn read(
        &self,
        request: Request<ReadRequest>,
    ) -> Result<Response<ReadResponse>, Status> {
        let req = request.into_inner();

        let data = match req.length {
            Some(len) => self.backend.read_range(&req.path, req.offset.unwrap_or(0), len as usize),
            None => self.backend.read(&req.path),
        };

        match data {
            Ok(bytes) => Ok(Response::new(ReadResponse {
                data: bytes,
                success: true,
                error: None,
            })),
            Err(e) => Ok(Response::new(ReadResponse {
                data: vec![],
                success: false,
                error: Some(fs_error_to_proto(e)),
            })),
        }
    }

    async fn write(
        &self,
        request: Request<WriteRequest>,
    ) -> Result<Response<WriteResponse>, Status> {
        let req = request.into_inner();

        let result = if req.append {
            self.backend.append(&req.path, &req.data)
        } else {
            self.backend.write(&req.path, &req.data)
        };

        match result {
            Ok(()) => Ok(Response::new(WriteResponse {
                success: true,
                error: None,
            })),
            Err(e) => Ok(Response::new(WriteResponse {
                success: false,
                error: Some(fs_error_to_proto(e)),
            })),
        }
    }

    // ... implement other methods
}

fn fs_error_to_proto(e: FsError) -> ProtoError {
    match e {
        FsError::NotFound { path } => ProtoError {
            code: "not_found".into(),
            message: "File not found".into(),
            path: path.to_string_lossy().into(),
        },
        FsError::AlreadyExists { path, .. } => ProtoError {
            code: "already_exists".into(),
            message: "File already exists".into(),
            path: path.to_string_lossy().into(),
        },
        // ... map other errors
        _ => ProtoError {
            code: "internal".into(),
            message: e.to_string(),
            path: String::new(),
        },
    }
}
```

### Authentication Middleware

Add authentication as a tower layer:

```rust
use tonic::service::Interceptor;

#[derive(Clone)]
pub struct AuthInterceptor {
    valid_tokens: Arc<HashSet<String>>,
}

impl Interceptor for AuthInterceptor {
    fn call(&mut self, mut request: Request<()>) -> Result<Request<()>, Status> {
        let token = request
            .metadata()
            .get("authorization")
            .and_then(|v| v.to_str().ok())
            .map(|s| s.trim_start_matches("Bearer "));

        match token {
            Some(t) if self.valid_tokens.contains(t) => Ok(request),
            _ => Err(Status::unauthenticated("Invalid or missing token")),
        }
    }
}

// Usage
Server::builder()
    .add_service(FsServiceServer::with_interceptor(fs_server, auth_interceptor))
    .serve(addr)
    .await?;
```

### Rate Limiting

Protect against abuse:

```rust
use governor::{Quota, RateLimiter};
use std::num::NonZeroU32;

pub struct RateLimitedServer<B: Fs> {
    inner: FsServer<B>,
    limiter: RateLimiter<String>,  // Per-user rate limiter
}

impl<B: Fs> RateLimitedServer<B> {
    pub fn new(backend: B, requests_per_second: u32) -> Self {
        let quota = Quota::per_second(NonZeroU32::new(requests_per_second).unwrap());
        Self {
            inner: FsServer::new(backend),
            limiter: RateLimiter::keyed(quota),
        }
    }

    async fn check_rate_limit(&self, user_id: &str) -> Result<(), Status> {
        self.limiter
            .check_key(&user_id.to_string())
            .map_err(|_| Status::resource_exhausted("Rate limit exceeded"))?;
        Ok(())
    }
}
```

### Idempotency

Handle retried requests safely:

```rust
use std::collections::HashMap;
use std::time::{Duration, Instant};

pub struct IdempotencyCache {
    cache: RwLock<HashMap<String, (Instant, Response)>>,
    ttl: Duration,
}

impl IdempotencyCache {
    pub fn new(ttl: Duration) -> Self {
        Self {
            cache: RwLock::new(HashMap::new()),
            ttl,
        }
    }

    /// Check if we've seen this request before.
    pub fn get(&self, request_id: &str) -> Option<Response> {
        let cache = self.cache.read().unwrap();
        cache.get(request_id)
            .filter(|(ts, _)| ts.elapsed() < self.ttl)
            .map(|(_, resp)| resp.clone())
    }

    /// Store response for future duplicate requests.
    pub fn put(&self, request_id: String, response: Response) {
        let mut cache = self.cache.write().unwrap();
        cache.insert(request_id, (Instant::now(), response));
    }

    /// Clean up expired entries (call periodically).
    pub fn cleanup(&self) {
        let mut cache = self.cache.write().unwrap();
        cache.retain(|_, (ts, _)| ts.elapsed() < self.ttl);
    }
}
```

---

## Client Implementation

### Remote Backend (Client-Side)

The client implements `Fs` traits by making RPC calls:

```rust
use anyfs_backend::{FsRead, FsWrite, FsDir, FsError, Metadata, ReadDirIter, DirEntry};
use std::path::Path;

pub struct RemoteBackend {
    client: FsServiceClient<tonic::transport::Channel>,
    auth_token: String,
}

impl RemoteBackend {
    pub async fn connect(addr: &str, auth_token: String) -> Result<Self, FsError> {
        let client = FsServiceClient::connect(addr.to_string())
            .await
            .map_err(|e| FsError::Backend(format!("connect failed: {}", e)))?;

        Ok(Self { client, auth_token })
    }

    fn request<T>(&self, req: T) -> tonic::Request<T> {
        let mut request = tonic::Request::new(req);
        request.metadata_mut().insert(
            "authorization",
            format!("Bearer {}", self.auth_token).parse().unwrap(),
        );
        request
    }
}

impl FsRead for RemoteBackend {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        // Note: This is sync, but we're calling async code
        // In practice, use tokio::runtime::Handle or async traits

        let rt = tokio::runtime::Handle::current();
        rt.block_on(async {
            let req = self.request(ReadRequest {
                path: path.as_ref().to_string_lossy().into(),
                offset: None,
                length: None,
            });

            let response = self.client.clone().read(req)
                .await
                .map_err(|e| FsError::Backend(format!("rpc failed: {}", e)))?
                .into_inner();

            if response.success {
                Ok(response.data)
            } else {
                Err(proto_error_to_fs(response.error.unwrap()))
            }
        })
    }

    fn exists(&self, path: impl AsRef<Path>) -> Result<bool, FsError> {
        // Could be a dedicated RPC or use metadata
        match self.metadata(path) {
            Ok(_) => Ok(true),
            Err(FsError::NotFound { .. }) => Ok(false),
            Err(e) => Err(e),
        }
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        let rt = tokio::runtime::Handle::current();
        rt.block_on(async {
            let req = self.request(MetadataRequest {
                path: path.as_ref().to_string_lossy().into(),
            });

            let response = self.client.clone().metadata(req)
                .await
                .map_err(|e| FsError::Backend(format!("rpc failed: {}", e)))?
                .into_inner();

            if response.success {
                Ok(proto_metadata_to_fs(response.metadata.unwrap()))
            } else {
                Err(proto_error_to_fs(response.error.unwrap()))
            }
        })
    }

    // ... other methods
}

impl FsWrite for RemoteBackend {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let rt = tokio::runtime::Handle::current();
        rt.block_on(async {
            let req = self.request(WriteRequest {
                path: path.as_ref().to_string_lossy().into(),
                data: data.to_vec(),
                append: false,
            });

            let response = self.client.clone().write(req)
                .await
                .map_err(|e| FsError::Backend(format!("rpc failed: {}", e)))?
                .into_inner();

            if response.success {
                Ok(())
            } else {
                Err(proto_error_to_fs(response.error.unwrap()))
            }
        })
    }

    // ... other methods
}

impl FsDir for RemoteBackend {
    fn read_dir(&self, path: impl AsRef<Path>) -> Result<ReadDirIter, FsError> {
        let rt = tokio::runtime::Handle::current();
        rt.block_on(async {
            let mut all_entries = Vec::new();
            let mut cursor = None;

            // Paginate through all results
            loop {
                let req = self.request(ListDirRequest {
                    path: path.as_ref().to_string_lossy().into(),
                    cursor: cursor.clone(),
                    limit: Some(1000),
                });

                let response = self.client.clone().list_dir(req)
                    .await
                    .map_err(|e| FsError::Backend(format!("rpc failed: {}", e)))?
                    .into_inner();

                if !response.success {
                    return Err(proto_error_to_fs(response.error.unwrap()));
                }

                all_entries.extend(response.entries.into_iter().map(proto_entry_to_fs));

                match response.next_cursor {
                    Some(c) => cursor = Some(c),
                    None => break,
                }
            }

            Ok(ReadDirIter::new(all_entries.into_iter().map(Ok)))
        })
    }

    // ... other methods
}
```

---

## Caching Layer

Network calls are slow. Add caching:

```rust
use std::collections::HashMap;
use std::sync::RwLock;
use std::time::{Duration, Instant};

/// Client-side cache for remote filesystem.
pub struct CachingBackend<B> {
    inner: B,
    metadata_cache: RwLock<HashMap<PathBuf, (Instant, Metadata)>>,
    content_cache: RwLock<LruCache<PathBuf, Vec<u8>>>,
    metadata_ttl: Duration,
    max_cached_file_size: u64,
}

impl<B> CachingBackend<B> {
    pub fn new(inner: B) -> Self {
        Self {
            inner,
            metadata_cache: RwLock::new(HashMap::new()),
            content_cache: RwLock::new(LruCache::new(100)),  // 100 files
            metadata_ttl: Duration::from_secs(5),
            max_cached_file_size: 1024 * 1024,  // 1 MB
        }
    }

    /// Invalidate cache for a path (call after writes).
    pub fn invalidate(&self, path: &Path) {
        self.metadata_cache.write().unwrap().remove(path);
        self.content_cache.write().unwrap().pop(path);
    }

    /// Invalidate everything under a directory.
    pub fn invalidate_prefix(&self, prefix: &Path) {
        let mut meta = self.metadata_cache.write().unwrap();
        let mut content = self.content_cache.write().unwrap();

        meta.retain(|k, _| !k.starts_with(prefix));
        // LruCache doesn't have retain, so we'd need a different structure
    }
}

impl<B: FsRead> FsRead for CachingBackend<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();

        // Check cache first
        if let Some(data) = self.content_cache.read().unwrap().peek(path) {
            return Ok(data.clone());
        }

        // Cache miss - fetch from remote
        let data = self.inner.read(path)?;

        // Cache if small enough
        if data.len() as u64 <= self.max_cached_file_size {
            self.content_cache.write().unwrap().put(path.to_path_buf(), data.clone());
        }

        Ok(data)
    }

    fn metadata(&self, path: impl AsRef<Path>) -> Result<Metadata, FsError> {
        let path = path.as_ref();

        // Check cache
        {
            let cache = self.metadata_cache.read().unwrap();
            if let Some((ts, meta)) = cache.get(path) {
                if ts.elapsed() < self.metadata_ttl {
                    return Ok(meta.clone());
                }
            }
        }

        // Cache miss
        let meta = self.inner.metadata(path)?;

        // Store in cache
        self.metadata_cache.write().unwrap()
            .insert(path.to_path_buf(), (Instant::now(), meta.clone()));

        Ok(meta)
    }

    // ... other methods
}

impl<B: FsWrite> FsWrite for CachingBackend<B> {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref();

        // Write through to remote
        self.inner.write(path, data)?;

        // Invalidate cache
        self.invalidate(path);

        Ok(())
    }

    // ... other methods - all invalidate cache after modifying
}
```

### Cache Invalidation Strategies

| Strategy | How | When to Use |
|----------|-----|-------------|
| **TTL** | Expire after N seconds | Read-heavy, eventual consistency OK |
| **Write-through** | Invalidate on local write | Single client |
| **Server push** | WebSocket notifications | Real-time consistency |
| **Version/ETag** | Check version on read | Balance of consistency/perf |

---

## Offline Mode

Handle network failures gracefully:

```rust
pub struct OfflineCapableBackend<B> {
    remote: B,
    local_cache: SqliteBackend,  // Local SQLite for offline ops
    mode: RwLock<ConnectionMode>,
    pending_writes: RwLock<Vec<PendingWrite>>,
}

#[derive(Clone, Copy)]
enum ConnectionMode {
    Online,
    Offline,
    Reconnecting,
}

struct PendingWrite {
    path: PathBuf,
    operation: WriteOperation,
    timestamp: Instant,
}

enum WriteOperation {
    Write(Vec<u8>),
    Append(Vec<u8>),
    Remove,
    CreateDir,
    // ...
}

impl<B: Fs> OfflineCapableBackend<B> {
    fn is_online(&self) -> bool {
        matches!(*self.mode.read().unwrap(), ConnectionMode::Online)
    }

    fn go_offline(&self) {
        *self.mode.write().unwrap() = ConnectionMode::Offline;
    }

    fn try_reconnect(&self) -> bool {
        *self.mode.write().unwrap() = ConnectionMode::Reconnecting;

        // Try a simple operation
        if self.remote.exists("/").is_ok() {
            *self.mode.write().unwrap() = ConnectionMode::Online;
            self.sync_pending_writes();
            true
        } else {
            *self.mode.write().unwrap() = ConnectionMode::Offline;
            false
        }
    }

    fn sync_pending_writes(&self) {
        let mut pending = self.pending_writes.write().unwrap();

        for write in pending.drain(..) {
            let result = match write.operation {
                WriteOperation::Write(data) => self.remote.write(&write.path, &data),
                WriteOperation::Append(data) => self.remote.append(&write.path, &data),
                WriteOperation::Remove => self.remote.remove_file(&write.path),
                WriteOperation::CreateDir => self.remote.create_dir(&write.path),
            };

            if result.is_err() {
                // Put back and stop syncing
                // (In practice, need conflict resolution)
                break;
            }
        }
    }
}

impl<B: FsRead> FsRead for OfflineCapableBackend<B> {
    fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError> {
        let path = path.as_ref();

        if self.is_online() {
            match self.remote.read(path) {
                Ok(data) => {
                    // Update local cache
                    let _ = self.local_cache.write(path, &data);
                    Ok(data)
                }
                Err(FsError::Backend(_)) => {
                    // Network error - go offline, try cache
                    self.go_offline();
                    self.local_cache.read(path)
                }
                Err(e) => Err(e),
            }
        } else {
            // Offline - use cache
            self.local_cache.read(path)
        }
    }
}

impl<B: FsWrite> FsWrite for OfflineCapableBackend<B> {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError> {
        let path = path.as_ref();

        // Always write to local cache
        self.local_cache.write(path, data)?;

        if self.is_online() {
            match self.remote.write(path, data) {
                Ok(()) => Ok(()),
                Err(FsError::Backend(_)) => {
                    // Network error - queue for later
                    self.go_offline();
                    self.pending_writes.write().unwrap().push(PendingWrite {
                        path: path.to_path_buf(),
                        operation: WriteOperation::Write(data.to_vec()),
                        timestamp: Instant::now(),
                    });
                    Ok(())  // Return success - we wrote locally
                }
                Err(e) => Err(e),
            }
        } else {
            // Offline - queue for later sync
            self.pending_writes.write().unwrap().push(PendingWrite {
                path: path.to_path_buf(),
                operation: WriteOperation::Write(data.to_vec()),
                timestamp: Instant::now(),
            });
            Ok(())
        }
    }
}
```

### Conflict Resolution

When syncing offline writes, conflicts can occur:

```rust
enum ConflictResolution {
    /// Server version wins (discard local changes)
    ServerWins,
    /// Client version wins (overwrite server)
    ClientWins,
    /// Keep both (rename local to .conflict)
    KeepBoth,
    /// Ask user
    Manual,
}

fn resolve_conflict(
    path: &Path,
    local_data: &[u8],
    server_data: &[u8],
    strategy: ConflictResolution,
) -> Result<(), FsError> {
    match strategy {
        ConflictResolution::ServerWins => {
            // Discard local, use server version
            Ok(())
        }
        ConflictResolution::ClientWins => {
            // Overwrite server with local
            remote.write(path, local_data)
        }
        ConflictResolution::KeepBoth => {
            // Rename local to path.conflict
            let conflict_path = format!("{}.conflict", path.display());
            remote.write(&conflict_path, local_data)?;
            Ok(())
        }
        ConflictResolution::Manual => {
            Err(FsError::Conflict { path: path.to_path_buf() })
        }
    }
}
```

---

## FUSE Client

Mount the remote filesystem locally using FUSE:

```rust
use fuser::{Filesystem, MountOption, Request, ReplyData, ReplyEntry, ReplyAttr, ReplyDirectory};

pub struct RemoteFuse<B: Fs> {
    backend: B,
    // Inode management for FUSE
    inodes: RwLock<BiMap<u64, PathBuf>>,
    next_inode: AtomicU64,
}

impl<B: Fs> Filesystem for RemoteFuse<B> {
    fn lookup(&mut self, _req: &Request, parent: u64, name: &OsStr, reply: ReplyEntry) {
        let parent_path = self.inode_to_path(parent);
        let path = parent_path.join(name);

        match self.backend.metadata(&path) {
            Ok(meta) => {
                let inode = self.path_to_inode(&path);
                let attr = metadata_to_fuse_attr(inode, &meta);
                reply.entry(&Duration::from_secs(1), &attr, 0);
            }
            Err(_) => reply.error(libc::ENOENT),
        }
    }

    fn read(
        &mut self,
        _req: &Request,
        ino: u64,
        _fh: u64,
        offset: i64,
        size: u32,
        _flags: i32,
        _lock_owner: Option<u64>,
        reply: ReplyData,
    ) {
        let path = self.inode_to_path(ino);

        match self.backend.read_range(&path, offset as u64, size as usize) {
            Ok(data) => reply.data(&data),
            Err(_) => reply.error(libc::EIO),
        }
    }

    fn write(
        &mut self,
        _req: &Request,
        ino: u64,
        _fh: u64,
        offset: i64,
        data: &[u8],
        _write_flags: u32,
        _flags: i32,
        _lock_owner: Option<u64>,
        reply: fuser::ReplyWrite,
    ) {
        let path = self.inode_to_path(ino);

        // For simplicity, read-modify-write
        // (Real impl would use open_write with seeking)
        match self.backend.read(&path) {
            Ok(mut content) => {
                let offset = offset as usize;
                if offset > content.len() {
                    content.resize(offset, 0);
                }
                if offset + data.len() > content.len() {
                    content.resize(offset + data.len(), 0);
                }
                content[offset..offset + data.len()].copy_from_slice(data);

                match self.backend.write(&path, &content) {
                    Ok(()) => reply.written(data.len() as u32),
                    Err(_) => reply.error(libc::EIO),
                }
            }
            Err(_) => reply.error(libc::EIO),
        }
    }

    fn readdir(
        &mut self,
        _req: &Request,
        ino: u64,
        _fh: u64,
        offset: i64,
        mut reply: ReplyDirectory,
    ) {
        let path = self.inode_to_path(ino);

        match self.backend.read_dir(&path) {
            Ok(entries) => {
                let entries: Vec<_> = entries.filter_map(|e| e.ok()).collect();

                for (i, entry) in entries.iter().enumerate().skip(offset as usize) {
                    let child_path = path.join(&entry.name);
                    let child_inode = self.path_to_inode(&child_path);
                    let file_type = match entry.file_type {
                        FileType::File => fuser::FileType::RegularFile,
                        FileType::Directory => fuser::FileType::Directory,
                        FileType::Symlink => fuser::FileType::Symlink,
                    };

                    if reply.add(child_inode, (i + 1) as i64, file_type, &entry.name) {
                        break;  // Buffer full
                    }
                }
                reply.ok();
            }
            Err(_) => reply.error(libc::EIO),
        }
    }

    // ... implement other FUSE methods
}

// Mount the remote filesystem
pub fn mount_remote(backend: impl Fs, mountpoint: &Path) -> Result<(), Box<dyn Error>> {
    let fuse = RemoteFuse::new(backend);

    fuser::mount2(
        fuse,
        mountpoint,
        &[
            MountOption::RO,  // Or RW
            MountOption::FSName("anyfs-remote".to_string()),
            MountOption::AutoUnmount,
        ],
    )?;

    Ok(())
}
```

---

## Summary: Building a Cloud Filesystem

To build a complete cloud filesystem service:

### Server Side
1. Wrap your backend (e.g., `HybridBackend`) with middleware
2. Expose via gRPC/REST server
3. Add authentication, rate limiting, idempotency

### Client Side
1. Implement `RemoteBackend` that calls server RPC
2. Wrap with `CachingBackend` for performance
3. Optionally add `OfflineCapableBackend`
4. Mount via FUSE for native OS integration

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Client Machine                                             │
│  ┌─────────┐    ┌────────────┐    ┌───────────────────┐    │
│  │  FUSE   │ →  │  Caching   │ →  │  RemoteBackend    │    │
│  │ Mount   │    │  Backend   │    │  (RPC Client)     │    │
│  └─────────┘    └────────────┘    └─────────┬─────────┘    │
└────────────────────────────────────────────│───────────────┘
                                              │ Network
┌────────────────────────────────────────────│───────────────┐
│  Server                                     ▼               │
│  ┌─────────────────┐    ┌─────────────────────────────┐    │
│  │   RPC Server    │ →  │  Middleware Stack           │    │
│  │  (Auth, Rate)   │    │  Quota → Tracing → Backend  │    │
│  └─────────────────┘    └─────────────┬───────────────┘    │
│                                        ▼                    │
│                         ┌─────────────────────────────┐    │
│                         │     HybridBackend           │    │
│                         │  SQLite + Blob Store        │    │
│                         └─────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

This gives you a complete cloud filesystem with:
- Native OS mounting (FUSE)
- Offline support
- Caching for performance
- Server-side quotas and logging
- Scalable blob storage with dedup
