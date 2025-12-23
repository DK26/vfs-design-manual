# Historical: VFS Documentation Review Findings

> This review predates the current AnyFS design and references an earlier architecture (`vfs-switchable` naming, etc.).
>
> The current AnyFS design uses three crates: `anyfs-traits`, `anyfs`, `anyfs-container` with a path-based `VfsBackend` trait.
>
> For current design, see:
> - `book/src/architecture/adrs.md`
> - `book/src/architecture/design-overview.md`

## Executive Summary

The documentation is currently in a **fractured state** with significant contradictions between the "Design" documents and the "User Guide" documents. The ecosystem appears to be transitioning from a monolithic `vfs` crate concept to a split `vfs-switchable`/`vfs-container` architecture, but the documentation has not been fully synchronized.

## Critical Inconsistencies

### 1. Crate Architecture & Naming
- **Conflict**: 
  - `02-api-quick-reference.md` and `05-getting-started-guide.md` instruct users to depend on a single `vfs` crate (v0.1).
  - `11-restructured-design.md` and `vfs-design.md` explicitly define a split architecture: `vfs-switchable` (core trait) and `vfs-container` (isolation).
- **Issue**: The name `vfs` is already an existing, popular crate on crates.io. `vfs-design.md` even lists the `vfs` crate as an alternative/competitor in the Appendix, creating a confusing circular reference where the docs tell you to use the competitor's name.

### 2. Core API Signature (`VfsBackend`)
- **Conflict**:
  - `vfs-design.md` (v0.2.0) defines the core trait methods as taking `impl AsRef<Path>`, arguing this is "idiomatic Rust".
  - `11-restructured-design.md` defines the core trait methods as taking `&VirtualPath`.
- **Impact**: This is a fundamental design divergence.
  - `impl AsRef<Path>` implies the backend is responsible for path validation and safety.
  - `&VirtualPath` implies the caller (Container) guarantees safety/normalization *before* the backend sees it.
  - **Verdict**: `11-restructured-design.md` (using `VirtualPath`) aligns better with the stated goal of "Containment" and safety, whereas `vfs-design.md` seems to cling to standard library interoperability at the cost of safety guarantees.

### 3. Path Resolution Strategy
- **Conflict**:
  - `04-ADR-004` and `11-restructured-design.md` advocate for "Lexical Path Resolution" using `VirtualPath` to prevent escapes.
  - `vfs-design.md` suggests `VRootFsBackend` passes paths *directly* to `strict-path`, but strict-path usually requires a specific path type or setup. The `impl AsRef<Path>` signature in `vfs-design.md` obscures *how* the backend ensures safety if it accepts raw strings.

## Missing or Outdated Content

- **Document Gap**: File `07` is missing from the numbered sequence.
- **Phantom Versioning**: "v0.1" is referenced in guides, but "v0.2.0" is the version of the design doc. The project seems to be pre-alpha but docs imply an existing v0.1 release.

## Recommendations

1.  **Standardize on the Split Architecture**: Adopt the `vfs-switchable` and `vfs-container` structure from `11-restructured-design.md`.
2.  **Rename the "User-Facing" Crate**: Avoid `vfs`. If `vfs-container` is the main entry point, use that or a new unique name (e.g., `tektite-vfs`, `vfs-isolated`).
3.  **Resolve the Trait Signature**: Go with `&VirtualPath`. It provides stronger type safety boundaries and prevents backends from mishandling paths (e.g., resolving `..` incorrectly). 
    - *Correction needed*: Update `vfs-design.md` to reflect `&VirtualPath` instead of `impl AsRef<Path>`.
4.  **Update Guides**: Rewrite `02` and `05` to reflect the multi-crate setup (or a workspace) and correct dependency names.
