# Addendum — SQLite as Container Format vs Custom Binary Format (Historical)

> This addendum was written during the earlier graph-store design iteration. The SQLite-vs-binary tradeoffs are still relevant, but references to `StorageBackend`/transactions reflect that older architecture.
>
> For the current AnyFS design (path-based `VfsBackend`), see:
> - `book/src/architecture/adrs.md` (especially ADR-001)
> - `book/src/architecture/design-overview.md`

**Date:** 2025-12-22  
**Project:** VFS Container  
**Purpose:** Put the SQLite-vs-binary tradeoffs “on record”, clarify the *backend separation* user story, and propose a practical path forward (including a “stop / proceed” decision gate).

---

## 1. Problem Statement

We need to decide (and document) whether the VFS Container’s primary persistent format should be:

1) **SQLite-backed container** (metadata + content stored as tables / blobs / chunks), or  
2) **A custom binary container format** (a bespoke file layout for metadata + content).

The decision impacts:
- durability & crash consistency,
- performance characteristics,
- ability to query/search/visualize containers,
- ecosystem/tooling,
- maintainability and long-term evolution.

---

## 2. Why Backend Separation Matters (User Story “On Record”)

### 2.1. Core principle
The container should express **filesystem semantics** in one place (`FilesContainer`), and allow **multiple storage strategies** via a backend abstraction (`StorageBackend`).

### 2.2. User story: SQLite for dashboard/analytics vs other backends for other goals
> **As an integrator**, I want to store my virtual filesystem in **SQLite** so I can build dashboards, inspections, and analytics directly using existing SQLite drivers and tools (GUI viewers, SQL queries, reporting pipelines) without writing a custom parser.

> **As another integrator**, I want a different backend that optimizes for my use case (e.g., memory-only testing, cloud object-store persistence, encrypted-at-rest backend, compressed backend, embedded key-value backend), while still using the same `FilesContainer` semantics and API.

### 2.3. Implication
The backend interface must be:
- stable enough to support multiple implementations,
- strict enough to preserve filesystem invariants,
- and clear about what is “backend responsibility” vs “container responsibility”.

---

## 3. SQLite-backed Container — Advantages (Put on Record)

### 3.1. ACID transactions and crash safety
- Mature transactional semantics.
- WAL/journaling patterns are known, tested, and widely deployed.
- You can get strong “don’t corrupt on crash” behavior early.

### 3.2. Queryability and observability (major differentiator)
- Search and filtering over metadata (names, sizes, timestamps, tags, hashes) via indexes.
- Supports “dashboard-style” use cases naturally:
  - auditing (“show me recently changed files”),
  - inventory (“largest directories”, “most common extensions”),
  - compliance (“files missing required metadata”),
  - operations (“who created what/when”, if you store event logs).

### 3.3. Ecosystem tooling and interoperability
- File is inspectable with mature tools (CLI, GUIs).
- Easy to build “support tools” (inspect/repair/export) quickly.
- Backups are a known problem with known solutions (online backup APIs, file copying with appropriate locking, etc).

### 3.4. Schema constraints and basic integrity enforcement
- Foreign keys, unique constraints, and checks can prevent some corruption classes.
- Even if not sufficient for all filesystem invariants, they reduce the “custom format bug surface”.

### 3.5. Incremental I/O is possible
- With the right layout, reads/writes can be incremental (important for large files).
- This lets you evolve toward streaming APIs later without changing the persistence substrate.

---

## 4. SQLite-backed Container — Disadvantages / Risks (Put on Record)

### 4.1. Write amplification and overhead
- Small writes can touch multiple pages + indexes + WAL.
- Chunk-per-64KB design can create many rows and add overhead.

### 4.2. Filesystem invariants are not “pure SQL”
SQLite cannot naturally enforce all the rules you care about:
- directory graph acyclicity,
- rename rules (prevent rename into own subtree),
- hard-link and GC correctness,
- transactional quota accounting.

You’ll still need application-level checks (or complex triggers—usually not worth it).

### 4.3. “Dashboard access” can become a footgun
If you encourage external tools to open the container database:
- schema becomes a public contract (breaking it breaks dashboards),
- direct writes can corrupt invariants,
- migrations need careful versioning.

### 4.4. Concurrency model limitations
- Many readers + one writer is fine; many writers is not.
- For multi-process or network-share access, locking semantics become a support risk.

### 4.5. Encryption/compression are not native
- Requires external solutions (e.g., SQLCipher) or application-layer encryption.
- Compression is usually implemented at the chunk layer (which is doable, but extra complexity).

---

## 5. Custom Binary Container — Advantages (Put on Record)

### 5.1. Performance and layout control
- You can optimize for sequential I/O and specific access patterns.
- Lower overhead for tight loops, heavy write workloads, and large streaming reads.

### 5.2. Tailored features are easier to “bake in”
- Content-addressing, checksums, compression, encryption, snapshots, refcounts, and GC can be first-class from day 1 (if you implement them).

### 5.3. Simple external API surface (sometimes)
- If you make a stable binary format with a clear spec, external consumers can implement readers.
- But this is only true if you maintain the spec rigorously.

---

## 6. Custom Binary Container — Disadvantages / Risks (Put on Record)

### 6.1. You must build durability and recovery yourself
- Crash safety, partial writes, journaling/WAL/CoW, and integrity checks become your job.
- The “format bug surface” is large and long-lived.

### 6.2. Tooling and interoperability cost
- Every “dashboard” or query tool needs a custom parser or library.
- Debugging/support is harder (you can’t just open it in a DB browser).

### 6.3. Migration/versioning complexity
- Any format evolution becomes a careful compatibility exercise.
- You’ll need robust upgrade paths (and possibly dual readers/writers).

---

## 7. Recommended Path Forward (If We Should Move Forward)

### 7.1. Recommendation: proceed with SQLite-first, but formalize the separation
Proceed with SQLite as the **primary** persistent backend for v1, because it:
- minimizes “storage correctness” risk early,
- enables dashboard/query value quickly,
- supports your stated multi-tenant and portability goals.

But do it in a way that keeps the door open for a custom format later.

### 7.2. Decision gate: “should we move forward?”
Move forward **only if** the team agrees to the following constraints:

1) **SQLite is the MVP persistence substrate**  
   - No parallel custom format in MVP.
2) **The backend interface remains a strict semantic boundary**  
   - Filesystem semantics (rename rules, path normalization, feature flags) stay in `FilesContainer`.
3) **Schema becomes an explicit contract**  
   - Versioned schema, migration policy, “read-only external access recommended”.
4) **We decide how dashboards access data safely**  
   - Prefer read-only connections and/or a stable “views” layer.

If these aren’t acceptable, don’t proceed until the format decision is resolved.

---

## 8. Concrete Suggestions to Make SQLite Shine (and Stay Safe)

### 8.1. Treat the database as “internal”, but publish a safe surface
- Publish stable **read-only views** for dashboards (e.g., `v_files`, `v_dirs`, `v_usage_by_dir`).
- Document that direct writes are unsupported and may corrupt invariants.

### 8.2. Version the schema explicitly
- Maintain `schema_version` table.
- Adopt a migration policy:
  - forward-only migrations,
  - compatibility notes for dashboard consumers.

### 8.3. Plan for large-file ergonomics without committing to streaming in MVP
Even if MVP doesn’t expose `Read/Write`:
- keep chunking and `read_range` semantics robust,
- design schema so later a streaming façade can be added without migration pain.

### 8.4. Quota accounting and hardlink behavior must be specified early
- Decide “logical bytes” vs “physical bytes”.
- If hard links are enabled, define reference counting and GC rules.

### 8.5. Provide an “official inspector” CLI
Even with SQLite tooling, an official CLI helps:
- enforce invariants,
- display data in stable ways,
- and prevent users from relying on unstable internal tables.

---

## 9. How Backend Separation Enables “Different Purposes”

Here are realistic backends enabled by keeping the abstraction clean:

- **MemoryBackend:** fast tests, fuzzing, deterministic state.
- **SQLiteBackend:** portable, queryable, dashboard-friendly container file.
- **EncryptedSQLiteBackend / SQLCipherBackend:** security-focused deployments.
- **CompressedChunkBackend:** storage-optimized deployments.
- **ObjectStoreBackend (S3/GCS):** cloud-native persistence, content-addressed chunks.
- **HostFSBackend (Graph-store-on-FS):** use the OS filesystem as a persistence substrate without trying to “adapt” native paths.

The separation lets each backend trade off performance, storage cost, and operational properties without changing the `FilesContainer` behavior.

---

## 10. Summary Recommendation

**Yes — move forward**, but do it with a clear SQLite-first stance and strong boundary enforcement:

- SQLite gives you durable storage + queryability + tooling immediately.
- A custom binary format is a bigger long-term project and should be considered a “v2” only if SQLite becomes a proven bottleneck.
- Backend separation remains valuable because it keeps the architecture future-proof and enables other storage strategies without rewriting filesystem semantics.

---

## Appendix: Quick “Go / No-Go” Checklist

**Go** if:
- you accept SQLite-first for persistence,
- you can commit to schema versioning and a safe dashboard surface,
- you can specify quota/hardlink/GC rules,
- and you can keep filesystem invariants in `FilesContainer`.

**No-Go** if:
- you require many-writer high-throughput workloads immediately,
- you require strong encryption/compression at-rest as a hard requirement in MVP (unless adopting SQLCipher etc),
- or you cannot accept schema stability requirements for external dashboard consumers.
