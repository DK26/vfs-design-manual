# Benchmarking Plan

> **This document specifies the benchmarking strategy for AnyFS when the implementation exists.**
> Functionality and security are the primary goals; performance validation is secondary but important.

---

## Goals

1. **Validate design decisions** - Confirm that the Tower-style middleware approach doesn't introduce unacceptable overhead
2. **Identify optimization opportunities** - Find hot paths that need attention
3. **Establish baselines** - Know where we stand relative to alternatives
4. **Prevent regressions** - Track performance across versions

---

## Benchmark Categories

### 1. Backend Benchmarks

Compare AnyFS backends against equivalent solutions for their specific use cases.

#### MemoryBackend vs Alternatives

| Competitor                  | Use Case               | Why Compare                  |
| --------------------------- | ---------------------- | ---------------------------- |
| `std::collections::HashMap` | Raw key-value baseline | Theoretical minimum overhead |
| `tempfile` + `std::fs`      | In-memory temp files   | Common testing approach      |
| `vfs::MemoryFS`             | Virtual filesystem     | Direct competitor            |
| `virtual-fs`                | In-memory FS           | Another VFS crate            |

**Metrics:**
- Sequential read/write throughput (1KB, 64KB, 1MB, 16MB files)
- Random access latency (small reads at random offsets)
- Directory listing performance (10, 100, 1000, 10000 entries)
- Memory overhead per file/directory

#### SqliteBackend vs Alternatives

| Competitor      | Use Case                    | Why Compare                  |
| --------------- | --------------------------- | ---------------------------- |
| `rusqlite` raw  | Baseline SQLite performance | Measure our abstraction cost |
| `sled`          | Embedded database           | Alternative storage engine   |
| `redb`          | Embedded database           | Modern alternative           |
| File-per-record | Direct filesystem           | Traditional approach         |

**Metrics:**
- Insert throughput (batch vs individual)
- Read throughput (sequential vs random)
- Transaction overhead
- Database size vs raw file size
- Startup time (opening existing database)

#### VRootFsBackend vs Alternatives

| Competitor          | Use Case               | Why Compare                  |
| ------------------- | ---------------------- | ---------------------------- |
| `std::fs` direct    | Baseline filesystem    | Measure containment overhead |
| `cap-std`           | Capability-based FS    | Security-focused alternative |
| `chroot` simulation | Traditional sandboxing | System-level approach        |

**Metrics:**
- Path resolution overhead
- Symlink traversal cost
- Escape attempt detection cost

### 2. Middleware Overhead Benchmarks

Measure the cost of each middleware layer.

| Middleware      | What to Measure                      |
| --------------- | ------------------------------------ |
| `Quota<B>`      | Size tracking overhead per operation |
| `PathFilter<B>` | Glob matching cost per path          |
| `ReadOnly<B>`   | Should be zero (just error return)   |
| `RateLimit<B>`  | Fixed-window counter check overhead  |
| `Tracing<B>`    | Span creation/logging cost           |
| `Cache<B>`      | Cache hit/miss latency difference    |

**Key question:** What's the cost of a 5-layer middleware stack vs direct backend access?

**Target:** Middleware overhead should be <5% of I/O time for typical operations.

### 3. Composition Benchmarks

Measure real-world stacks, not isolated components.

#### AI Agent Sandbox Stack

```
Quota → PathFilter → RateLimit → Tracing → MemoryBackend
```

Compare against:
- Raw MemoryBackend (baseline)
- Manual checks in application code (alternative approach)

#### Persistent Database Stack

```
Cache → Tracing → SqliteBackend
```

Compare against:
- Raw SqliteBackend (baseline)
- Application-level caching (alternative approach)

### 4. Trait Implementation Benchmarks

Validate that strategic boxing doesn't hurt performance.

| Operation                       | Expected Cost                       |
| ------------------------------- | ----------------------------------- |
| `read()` / `write()`            | Zero-cost (monomorphized)           |
| `open_read()` → `Box<dyn Read>` | ~50ns allocation, negligible vs I/O |
| `read_dir()` → `ReadDirIter`    | One allocation per call             |
| `FileStorage::boxed()`          | One-time cost at setup              |

---

## Competitor Matrix

### By Use Case

| Use Case              | AnyFS Component  | Primary Competitors                    |
| --------------------- | ---------------- | -------------------------------------- |
| Testing/mocking       | MemoryBackend    | `tempfile`, `vfs::MemoryFS`            |
| Embedded database     | SqliteBackend    | `sled`, `redb`, raw SQLite             |
| Sandboxed host access | VRootFsBackend   | `cap-std`, `chroot`                    |
| Policy enforcement    | Middleware stack | Manual application code                |
| Union filesystem      | Overlay          | `overlayfs` (kernel), `fuse-overlayfs` |

### Crate Comparison

| Crate         | Strengths              | Weaknesses                      | Compare For          |
| ------------- | ---------------------- | ------------------------------- | -------------------- |
| `vfs`         | Simple API             | No middleware, limited features | API ergonomics       |
| `virtual-fs`  | WASM support           | Less composable                 | Cross-platform       |
| `cap-std`     | Security-focused       | Different abstraction level     | Sandboxing           |
| `tempfile`    | Battle-tested          | Not a VFS                       | Temp file operations |
| `include_dir` | Compile-time embedding | Read-only                       | Embedded assets      |

---

## Benchmark Infrastructure

### Framework

Use `criterion` for statistical rigor:
- Warm-up iterations
- Outlier detection
- Comparison between runs

### Test Data Sets

| Dataset         | Contents               | Purpose                 |
| --------------- | ---------------------- | ----------------------- |
| Small files     | 1000 files × 1KB       | Metadata-heavy workload |
| Large files     | 10 files × 100MB       | Throughput workload     |
| Deep hierarchy  | 10 levels × 10 dirs    | Path resolution stress  |
| Wide directory  | 1 dir × 10000 files    | Listing performance     |
| Mixed realistic | Project-like structure | Real-world simulation   |

### Reporting

Generate:
- Throughput charts (ops/sec, MB/sec)
- Latency histograms (p50, p95, p99)
- Memory usage graphs
- Comparison tables vs competitors

---

## Performance Targets

These are aspirational targets to validate during implementation:

| Metric                      | Target          | Rationale                         |
| --------------------------- | --------------- | --------------------------------- |
| Middleware overhead         | <5% of I/O time | Composability shouldn't cost much |
| MemoryBackend vs HashMap    | <2x slower      | Abstraction cost                  |
| SqliteBackend vs raw SQLite | <1.5x slower    | Thin wrapper                      |
| VRootFsBackend vs std::fs   | <1.2x slower    | Path checking cost                |
| 5-layer stack               | <10% overhead   | Real-world composition            |

---

## Benchmark Workflow

### Development Phase

```
cargo bench --bench <component>
```

Run focused benchmarks during development to catch regressions.

### Release Phase

```
cargo bench --all
```

Full benchmark suite before releases, with comparison to previous version.

### CI Integration

- Run subset of benchmarks on PR (smoke test)
- Full benchmark suite on main branch
- Store results for trend analysis

---

## Non-Goals

- **Beating std::fs at raw I/O** - We add abstraction; some overhead is acceptable
- **Micro-optimizing cold paths** - Focus on hot paths (read, write, metadata)
- **Benchmark gaming** - Optimize for real use cases, not synthetic benchmarks

---

## Tracking

**GitHub Issue:** Implement benchmark suite
- **Blocked by:** Core AnyFS implementation
- **Dependencies:** `criterion`, test data generation
- **Milestone:** Post-1.0 (after functionality and security are solid)
