# AnyFS — Full Architecture & Design Review (Historical: Graph-Store Design)

> This review covers an earlier (rejected) design iteration that modeled the backend as a transactional graph store (`NodeId`/`Edge`/chunks).
>
> The current AnyFS design is path-based (`VfsBackend` with 20 `std::fs`-aligned methods). For current decisions see:
> - `book/src/architecture/adrs.md` (especially ADR-001)
> - `book/src/architecture/design-overview.md`

**Review Date:** 2025-12-22  
**Project:** VFS Container  
**Scope:** Architecture, data model, API surface, backend design, security model, documentation consistency  
**Overall Status:** ✅ *Conditionally approved* (implementation can begin **after** resolving the critical items below)

---

## 0. Documents Reviewed

- `README.md`
- `01-executive-summary.md`
- `02-api-quick-reference.md`
- `03-backend-implementers-guide.md`
- `04-architecture-decision-records.md`
- `05-getting-started-guide.md`
- `06-comparison-and-positioning.md`
- `08-technical-comparison-with-alternatives.md`
- `09-build-vs-reuse-analysis.md`
- `anyfs-container-design.md`

---

## 1. Executive Summary

The core architectural direction is strong: **filesystem semantics live in `FilesContainer`**, while the storage layer is a **typed graph store** (`NodeId` + `Edge` + chunk store), guarded by **mandatory transactions** and **lexical-only virtual path normalization**.

This gives you the right primitives for:
- safe directory renames/moves (without path-as-key issues),
- hard links (multiple edges to the same node),
- strict escape prevention (virtual paths are not host-resolved),
- and predictable multi-tenant enforcement (capacity + optional permission checks in the core).

However, several critical gaps must be resolved **before** implementation, because they affect feasibility and correctness across all backends:
1) the docs currently contain **two competing backend trait designs**,  
2) the FS backend is **underspecified** (what it is, and how it can satisfy the graph trait),  
3) the story for large-file I/O, quotas, and import/export semantics needs explicit decisions.

---

## 2. Strategic Strengths (Keep These)

### 2.1 Clean separation of concerns
- The ADRs clearly define a backend that knows nothing about paths/symlinks/permissions; it’s just a typed graph store with chunk IO.
- This is a great boundary for safety and testability.

### 2.2 Lexical path normalization is a strong security primitive
- Your `VirtualPath` rules (collapse `//`, eliminate `.`, resolve `..` lexically, never escape root) eliminate a large class of path traversal bugs at the type level.

### 2.3 Mandatory transactions are the right default
- The “all mutations must happen inside `transact()`” decision makes multi-step ops crash-safe by construction.
- This also prevents “forgot to wrap in a transaction” corruption.

### 2.4 Opt-in feature model reduces MVP risk
- Symlinks / hard links / permissions / xattrs are explicitly feature-gated.
- This avoids implementing complex semantics accidentally for every user.

---

## 3. Critical Findings (Must Fix)

### CF-1 — **Competing backend trait designs across docs**
**Severity:** Critical (design/implementation ambiguity)

**What’s happening**
- `anyfs-container-design.md` + ADRs define the backend as a **graph store** with `transact()` and `snapshot()` and node/edge/chunk primitives.
- `08-technical-comparison-with-alternatives.md` shows a “StorageBackend (MVP Design)” that starts as **direct path-based operations** (`exists(path)`, `read(path)`, `write(path, ...)`, etc.), and then later in the same section also shows Snapshot/Transaction methods.

**Why this matters**
- Whether a backend is “path-based” vs “NodeId graph store” completely changes feasibility for FS, SQLite, conformance tests, and invariants.
- This also confuses reviewers and implementers (and will lead to the wrong backend being built).

**Required action**
- Declare **one** design as authoritative (recommended: the ADR + `anyfs-container-design.md` graph-store trait).
- Update or remove the “direct operations” trait definition from `08-technical-comparison-with-alternatives.md`.
- Add a prominent note in `README.md` that ADRs are the source of truth.

**Acceptance criteria**
- There is exactly **one** “StorageBackend trait” definition across the entire doc set.

---

### CF-2 — **FsBackend is underspecified / ambiguous**
**Severity:** High (feasibility risk)

**What’s happening**
- The design mentions a `vfs-fs` crate implemented “via strict-path” and that it “inherits host filesystem permissions”.
- But there is no concrete spec for how a path-based host filesystem can satisfy a **NodeId/Edge graph** API.

**There are two different possible meanings**
1) **“Adapter backend”** over a normal directory tree  
   - Requires a persistent mapping: `NodeId ↔ HostPath` (and stable identity across restarts).
   - Must define how rename/hardlink/symlink map to host semantics.
2) **“Graph store implemented on host FS”** (host FS used only as bytes+directory persistence)  
   - Node/edge/content records are stored in an internal layout keyed by NodeId/ContentId.
   - This *can* satisfy the trait without an external DB — but it’s not a “native filesystem adapter”.

**Required action**
Pick one and document it explicitly:

- **Option A (Recommended for MVP):** Remove `vfs-fs` from MVP scope; use only SQLite + memory.
- **Option B:** Make `vfs-fs` a **graph-store-on-FS** (internal layout), explicitly NOT a transparent adapter.
- **Option C:** Make `vfs-fs` an **adapter backend** and require a sidecar index (SQLite or metadata files) to preserve identity.

**Acceptance criteria**
- The docs explain which option you chose and include:
  - how NodeId persistence works,
  - what happens on restart,
  - what semantics are supported / not supported,
  - how import/export differs from FsBackend (if FsBackend remains).

---

### CF-3 — **Directory rename safety (avoid cycles) is not specified**
**Severity:** High (correctness + potential corruption)

A graph-based filesystem must forbid renaming a directory into its own subtree (e.g., `/a` → `/a/b/a`), which would create a cycle.

**Required action**
- Add a formal invariant: the directory graph must remain acyclic (a tree rooted at NodeId(1), unless hard links are enabled — and even then, directory hard links are disallowed).
- Define the algorithm/check for `rename()`:
  - when moving a directory, ensure destination parent is not a descendant of the source directory.
- Add conformance tests for this behavior.

**Acceptance criteria**
- Rename into subtree returns a deterministic error.
- Conformance tests include at least:
  - rename dir into its subtree,
  - rename across dirs with existing target,
  - rename file over file (if allowed/forbidden—must be decided).

---

## 4. High Priority Findings (Should Fix Before MVP)

### HP-1 — Large-file story is inconsistent with “AI/ML use case”
**Severity:** Medium/High (product positioning vs API design)

- Executive summary cites AI/ML use cases, but the core API is “bulk bytes” (`Vec<u8>`) with streaming deferred.
- You *do* have `read_range()`, and the internal model is chunked (64KB), so large files aren’t impossible — but ergonomics and memory behavior are not good for multi-GB workloads.

**Recommended action**
Choose one:
- **A:** Explicitly downgrade AI/ML positioning in `01-executive-summary.md` until streaming exists.
- **B:** Add a small streaming façade early (e.g., `open_read/open_write` returning `Read/Write`) as a supported “phase 2”.

Also note: if SQLite is the primary backend, `rusqlite` supports incremental blob I/O (with the `blob` feature), which can make streaming practical without changing the on-disk model.

---

### HP-2 — Concurrency model needs an explicit contract
**Severity:** Medium

The docs say “backend decides” regarding concurrency, but:
- `StorageBackend::transact(&mut self, ...)` implies exclusive mutable access,
- and `FilesContainer` methods are largely `&mut self` for writes.

This can still be fine, but it needs to be stated clearly.

**Recommended action**
Document the concurrency contract explicitly:
- MVP: `FilesContainer` is *not* intended to be shared concurrently without external synchronization.
- Recommended pattern: `Arc<Mutex<FilesContainer<B>>>` for multi-threaded servers.
- Backends may have internal locking (SQLite does), but the container API still requires coherent external access.

---

### HP-3 — Quota/usage accounting + hard links needs a correctness spec
**Severity:** Medium/High (multi-tenant safety)

Capacity limits are a core differentiator (`usage()`, `remaining()`, enforcement points), but correctness becomes tricky with:
- hard links (multiple directory entries pointing at the same node/content),
- copy semantics (open question: preserve hard links or not),
- and delete/content GC.

**Recommended action**
Add explicit accounting rules:
- Is `total_size` based on logical file sizes per node, or physical bytes per content_id?
- When two nodes share the same content via hard links, how is usage computed?
- What exactly triggers `delete_content(content_id)`?

Add tests for:
- create hard link → usage unchanged,
- delete one link → usage unchanged,
- delete last link → usage decreases and content removed.

---

## 5. Medium Priority Findings (Good to Fix Early)

### MP-1 — Import/Export semantics and threat model are underspecified
Import/export is a big attack surface:
- symlinks on host (follow vs preserve),
- special files (devices, FIFOs),
- resource exhaustion (deep trees, huge file counts),
- and **non-UTF-8 filenames** on Linux / Windows path semantics (your VirtualPath is a `String`).

**Recommended action**
Define an import/export policy section:
- filename encoding policy (reject? lossy mapping? base64 escape?),
- whether to follow host symlinks,
- whether to preserve metadata (timestamps/perms/xattrs),
- limits applied during import (max depth, max nodes, total bytes).

---

### MP-2 — Extended attributes: enforce limits and avoid heavy reads
You state xattrs are opt-in, and open questions recommend “64KB per attr”, but:
- `CapacityLimits` doesn’t include xattr size/count limits,
- `NodeMetadata` contains a full `HashMap<String, Vec<u8>>`, which can become heavy if `get_node()` always returns it.

**Recommended action**
- Add `max_xattr_value_size`, `max_xattr_total_size`, `max_xattr_count` to limits/config.
- Clarify whether backends must return xattrs as part of NodeRecord always, or if xattr access is separate.

---

### MP-3 — Doc numbering / TOC drift
`anyfs-container-design.md` has numbering inconsistencies:
- TOC points to “14. Open Questions” but the header is “15. Open Questions”.
- “15. Open Questions” and “15. Future Considerations” both exist.
- The “Open Questions” subsections are numbered “14.x”.

**Recommended action**
- Fix TOC and headings, and ensure anchors match.
- Add a “Docs are generated / do not edit anchors manually” note if applicable.

---

## 6. Backend-Specific Notes

### 6.1 SQLite backend
- Schema versioning exists — define migration approach (forward-only? breaking changes?).
- WAL mode is recommended; explicitly document it as default for crash safety.
- Add an index strategy for edges/chunks if performance matters (directory listing + partial reads).

### 6.2 Memory backend
The implementer guide provides a minimal snapshot-by-clone approach. That’s fine as a reference, but call it out explicitly as “for tests only”.

**Recommended “production memory backend” approach**
- diff/journal transaction (record mutations; apply on commit; discard on rollback),
- or copy-on-write with structural sharing (if you want immutability).

### 6.3 Fs backend (if kept)
After resolving CF-2, add conformance test scope:
- which operations are guaranteed identical to SQLite,
- what is intentionally different (permissions inheritance, xattrs, timestamps, etc.).

---

## 7. Required Decisions (Put in One Place)

1) **Backend trait:** graph-store only (recommended) vs direct path ops  
2) **FsBackend meaning:** remove vs adapter vs graph-store-on-FS  
3) **Large file I/O:** keep `read_range` only vs add `open_read/open_write`  
4) **Concurrency:** single-threaded by design vs documented external locking pattern  
5) **Quota accounting:** logical vs physical bytes with hard links  
6) **Import/export policy:** encoding, symlink behavior, special files, metadata preservation  
7) **Xattr limits:** sizes/counts and how they’re stored/retrieved

---

## 8. Acceptance Checklist for “Full Approval”

- [ ] Only one authoritative backend trait definition across all docs
- [ ] FsBackend scope/semantics are explicit (or removed from MVP)
- [ ] Rename directory cycle prevention spec + tests exist
- [ ] Quota accounting rules are specified and tested (incl. hard links)
- [ ] Import/export threat model + policies are documented
- [ ] Xattr size/count limits are defined and enforced
- [ ] Design doc numbering/TOC is corrected
- [ ] Conformance tests pass for SQLite + memory backends

---

## 9. Recommended Next Steps (Practical Order)

1) **Docs first:** choose authoritative trait + fix doc drift (CF-1)  
2) **Define FsBackend decision** (CF-2)  
3) **Specify rename invariants** + write tests (CF-3)  
4) Implement SQLite backend (MVP core) + conformance suite  
5) Implement memory backend (reference) + add a “production strategy” section  
6) Decide large-file plan and update positioning docs  
7) Lock down import/export and xattr policies

---

*End of review.*
