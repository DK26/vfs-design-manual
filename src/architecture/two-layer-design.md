# Layered Design: Backends + Middleware + Ergonomics

AnyFS uses a layered architecture that separates concerns:

1. **Backends**: Pure storage + filesystem semantics
2. **Middleware**: Composable policy layers
3. **FileStorage**: Ergonomic wrapper

---

## Architecture

```
┌─────────────────────────────────────────┐
│  FileStorage                         │  ← Ergonomics only
├─────────────────────────────────────────┤
│  Middleware Stack (composable):         │  ← Policy enforcement
│    Tracing → PathFilter → Restrictions  │
│    → Quota → Backend                    │
├─────────────────────────────────────────┤
│  Fs                             │  ← Pure storage
│  (Memory, SQLite, VRootFs, custom)      │
└─────────────────────────────────────────┘
```

---

## Layer Responsibilities

| Layer         | Responsibility                  | Path Handling                                                           |
| ------------- | ------------------------------- | ----------------------------------------------------------------------- |
| `FileStorage` | Ergonomic API + path resolution | Accepts `impl AsRef<Path>`; resolves paths via pluggable `PathResolver` |
| Middleware    | Policy enforcement              | `&Path` (object-safe core traits)                                       |
| Backend       | Storage + FS semantics          | `&Path` (object-safe core traits)                                       |

Core traits use `&Path` for object safety; `FileStorage`/`FsExt` provide `impl AsRef<Path>` ergonomics. Path resolution is pluggable via `PathResolver` trait (see ADR-033). Backends that wrap a real filesystem implement `SelfResolving` so FileStorage can skip resolution.

---

## Policy via Middleware

**Old design (rejected):** FileStorage contained quota/feature logic.

**Current design:** Policy is handled by composable middleware:

```rust
// Middleware enforces policy
let backend = MemoryBackend::new()
    .layer(QuotaLayer::builder()
        .max_total_size(100 * 1024 * 1024)
        .build())
    .layer(PathFilterLayer::builder()
        .allow("/workspace/**")
        .build())
    .layer(TracingLayer::new());

// FileStorage is ergonomics + path resolution (no policy)
let fs = FileStorage::new(backend);
```

---

## Path Containment

For `VRootFsBackend` (real filesystem), path containment uses `strict-path::VirtualRoot` internally:

```rust
impl Fs for VRootFsBackend {
    fn read(&self, path: &Path) -> Result<Vec<u8>, FsError> {
        // VirtualRoot ensures paths can't escape
        let safe_path = self.root.join(path)?;
        std::fs::read(safe_path).map_err(Into::into)
    }
}
```

For virtual backends (Memory, SQLite), paths are just keys - no OS path traversal possible. FileStorage performs symlink-aware resolution for these backends so normalization is consistent across virtual implementations.

For sandboxing across all backends, use `PathFilter` middleware:

```rust
PathFilterLayer::builder()
    .allow("/workspace/**")
    .deny("**/.env")
    .build()
    .layer(backend)
```

---

## Why This Matters

- **Separation of concerns**: Backends focus on storage, middleware handles policy
- **Composability**: Add/remove policies without touching storage code
- **Flexibility**: Same middleware works with any backend
- **Simplicity**: Each layer has one job
