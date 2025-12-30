# Design Review: Rust Community Alignment

This document critically reviews AnyFS design decisions against Rust community expectations and best practices. The goal is to identify potential friction points before implementation.

---

## Summary

| Category | Issues Found | Status |
|----------|--------------|--------|
| Critical (must fix) | 2 | ✅ Fixed |
| Should fix | 4 | 🟡 In progress |
| Document clearly | 3 | 🟢 Ongoing |
| Non-issues | 5 | ✅ Verified |

---

## ✅ Critical Issues (Fixed)

### 1. FsError Missing `#[non_exhaustive]` — FIXED

**Problem:** Our `FsError` enum doesn't have `#[non_exhaustive]`. This is a **semver hazard**.

**Status:** ✅ Fixed in `design-overview.md`. FsError now has `#[non_exhaustive]`, `thiserror::Error` derive, and `From<std::io::Error>` impl.

```rust
// Current (problematic)
pub enum FsError {
    NotFound { path: PathBuf },
    AlreadyExists { path: PathBuf, operation: &'static str },
    // ...
}

// If we add a variant in 1.1:
pub enum FsError {
    NotFound { path: PathBuf },
    AlreadyExists { path: PathBuf, operation: &'static str },
    TooManySymlinks { path: PathBuf },  // NEW - breaks exhaustive matches!
}
```

**Impact:** Users with exhaustive matches will get compile errors on minor version bumps.

**Fix:**

```rust
#[non_exhaustive]
#[derive(Debug, thiserror::Error)]
pub enum FsError {
    #[error("not found: {path}")]
    NotFound { path: PathBuf },
    // ...
}
```

**Also needed:**
- `impl std::error::Error for FsError`
- `impl From<std::io::Error> for FsError`
- Consider `#[non_exhaustive]` on struct variants too

---

### 2. Documentation Shows `&mut self` Despite ADR-023 — FIXED

**Problem:** Several code examples still show `&mut self` or `&mut impl Fs`.

**Status:** ✅ Fixed. All examples in `design-overview.md` and `files-container.md` now use `&self`.

```rust
// In design-overview.md line 346:
fn with_symlinks(fs: &mut (impl Fs + FsLink)) {  // WRONG
    fs.write("/target.txt", b"content")?;
    fs.symlink("/target.txt", "/link.txt")?;
}

// Should be:
fn with_symlinks(fs: &(impl Fs + FsLink)) {  // Correct
    fs.write("/target.txt", b"content")?;
    fs.symlink("/target.txt", "/link.txt")?;
}
```

**Impact:** Contradicts ADR-023 (interior mutability). Confuses implementers.

**Fix:** Audit all examples and ensure `&self` everywhere.

---

## 🟡 Should Fix

### 3. Sync-Only Design May Limit Adoption

**Problem:** No async support. Many modern Rust projects are async-first.

**Current stance (ADR-024):** Sync now, async later via parallel traits.

**Community expectation:** Projects like `tokio`, `async-std` are dominant. Users may expect:
```rust
async fn read(&self, path: impl AsRef<Path>) -> Result<Vec<u8>, FsError>;
```

**Mitigation:**
1. Document clearly: "Sync-first by design, async planned"
2. Ensure `Send + Sync` bounds enable `spawn_blocking` wrapper
3. Consider shipping `anyfs-async` adapter crate early

**Recommendation:** Acceptable for v1.0, but should be high priority for v1.1.

---

### 4. Interior Mutability May Surprise Users

**Problem:** `&self` for write operations is unusual in Rust.

```rust
// Our design
pub trait FsWrite: Send + Sync {
    fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
}

// What users might expect (std::io::Write pattern)
pub trait FsWrite {
    fn write(&mut self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
}
```

**Why we chose this (ADR-023):**
- Filesystems are shared resources
- Enables concurrent access
- Matches how `std::fs::write()` works (takes path, not mutable handle)

**Potential friction:**
- Users may try to use `&mut self` patterns
- May conflict with borrowck mental models

**Mitigation:**
1. Document prominently with rationale
2. Show examples of concurrent usage
3. Explain: "Like std::fs, not like std::io::Write"

**Recommendation:** Keep the design, but add prominent documentation.

---

### 5. Layer Trait Doesn't Match Tower Exactly

**Problem:** Tower's `Layer` trait has a different signature:

```rust
// Tower's Layer
pub trait Layer<S> {
    type Service;
    fn layer(&self, inner: S) -> Self::Service;
}

// Our Layer (appears to be)
pub trait Layer<B: Fs> {
    type Backend: Fs;
    fn layer(self, backend: B) -> Self::Backend;
}
```

**Differences:**
1. Tower uses `&self`, we use `self` (consumes the layer)
2. Tower calls it `Service`, we call it `Backend`
3. Tower doesn't require bounds on `S`

**Impact:** Users familiar with Tower may be confused.

**Options:**
1. **Match Tower exactly** - maximum familiarity
2. **Keep our design** - `self` consumption is arguably cleaner for our use case
3. **Document differences** - explain why we diverge

**Recommendation:** Document the differences. Our `self` consumption prevents accidental reuse of configured layers, which is appropriate for our use case.

---

### 6. No `#[must_use]` on Results

**Problem:** Functions returning `Result` should have `#[must_use]` to catch ignored errors.

```rust
// Current
fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;

// Better
#[must_use]
fn write(&self, path: impl AsRef<Path>, data: &[u8]) -> Result<(), FsError>;
```

**Impact:** Users might accidentally ignore errors.

**Fix:** Add `#[must_use]` to all `Result`-returning methods, or use `#[must_use]` on the trait itself.

---

## 🟢 Document Clearly

### 7. Path Semantics Are Virtual, Not OS

**Consideration:** Our paths are virtual filesystem paths, not OS paths.

```rust
// On Windows, this works the same as on Unix:
fs.write("/documents/file.txt", data)?;  // Forward slashes always
```

**Potential confusion:**
- Windows users might expect backslashes
- Path normalization rules may differ from OS

**Mitigation:** Document:
- "Paths are virtual, always use forward slashes"
- "Path resolution is platform-independent"
- Show examples on Windows

---

### 8. `Fs` as Marker Trait Pattern

**Pattern:**
```rust
pub trait Fs: FsRead + FsWrite + FsDir {}
impl<T: FsRead + FsWrite + FsDir> Fs for T {}
```

**This is valid Rust** but may surprise some users. They might expect:
```rust
pub trait Fs {
    fn read(...);
    fn write(...);
    // etc
}
```

**Why we do it:**
- Granular traits for partial implementations
- Middleware only needs to implement what it wraps

**Mitigation:** Document the pattern clearly with examples.

---

### 9. Builder Pattern Requires Configuration

**Pattern:**
```rust
// This won't compile - no build() on unconfigured builder
let quota = QuotaLayer::builder().build();  // Error!

// Must configure at least one limit
let quota = QuotaLayer::builder()
    .max_total_size(1_000_000)
    .build();  // OK
```

**This is intentional** (ADR-022) but may surprise users expecting defaults.

**Mitigation:** Clear error messages and documentation.

---

## ✅ Non-Issues (We're Doing It Right)

### 10. `impl AsRef<Path>` Everywhere
✅ Matches `std::fs` conventions. Idiomatic.

### 11. `Send + Sync` Requirements
✅ Standard for thread-safe abstractions. Enables use across async boundaries.

### 12. Feature-Gated Backends
✅ Standard Cargo pattern. Reduces compile time for unused backends.

### 13. Strategic Boxing (ADR-025)
✅ Matches Tower/Axum approach. Well-documented rationale.

### 14. Generic Middleware Composition
✅ Zero-cost abstractions. Idiomatic Rust.

---

## Action Items

### Before v0.1 (MVP)

| Priority | Issue | Action |
|----------|-------|--------|
| 🔴 Critical | FsError non_exhaustive | Add `#[non_exhaustive]` and `thiserror` derive |
| 🔴 Critical | &mut in examples | Audit all examples for &self consistency |
| 🟡 Should | #[must_use] | Add to all Result-returning methods |
| 🟢 Document | Interior mutability | Add prominent section explaining why |
| 🟢 Document | Path semantics | Add section on virtual paths |

### Before v1.0

| Priority | Issue | Action |
|----------|-------|--------|
| 🟡 Should | Async support | Ship `anyfs-async` or document workaround |
| 🟡 Should | Layer trait docs | Document differences from Tower |
| 🟢 Document | Marker trait pattern | Explain Fs = FsRead + FsWrite + FsDir |

---

## Comparison to Axum's Success Factors

| Factor | Axum | AnyFS | Assessment |
|--------|------|-------|------------|
| Tower integration | Native | Inspired by | 🟡 Different but similar |
| Async support | Yes | No (planned) | 🟡 Gap, but documented |
| Error handling | thiserror | Planned | 🔴 Must add |
| Documentation | Excellent | In progress | 🟡 Continue |
| Examples | Comprehensive | In progress | 🟡 Continue |
| Ecosystem fit | tokio native | std::fs native | ✅ Different target |

---

## Conclusion

**Overall assessment:** The design is sound and follows Rust best practices. The main gaps are:

1. **Critical:** `#[non_exhaustive]` on FsError (semver hazard)
2. **Critical:** Inconsistent `&mut` in examples (contradicts ADR-023)
3. **Important:** No async yet (but documented path forward)
4. **Minor:** Documentation gaps (being addressed)

With these fixes, the design should be well-received by the Rust community.
