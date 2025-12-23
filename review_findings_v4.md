# Review Findings (v4) - Status Update

This file tracks the documentation consistency work needed to align the manual with the **current** AnyFS design (ADR-001):
- Path-based `VfsBackend` trait (20 `std::fs`-aligned methods)
- Three crates: `anyfs-traits`, `anyfs`, `anyfs-container`
- Backend paths are `&VirtualPath` (from `strict-path`), container API is ergonomic (`impl AsRef<Path>`)

---

## Resolved

1. The "Decisions" paradox
   - `book/src/review/decisions.md` is now explicitly marked **historical/superseded**.
   - Related review docs are also marked historical to avoid contradicting ADR-001.

2. Technical comparison outdated
   - `book/src/comparisons/technical-comparison.md` now compares the current AnyFS design (crate names, 20-method `VfsBackend`, no graph-store model).

3. Missing implementation plan
   - Added `book/src/implementation/plan.md` and linked it from `book/src/SUMMARY.md`.

---

## Additional inconsistencies fixed (not in original v4 list)

- Updated getting-started docs and examples to use `std::fs`-aligned method names (`create_dir*`, `read_dir`, `remove_file/remove_dir/remove_dir_all`).
- Documented least-privilege feature whitelisting: advanced features (symlinks, hard links, permissions) are disabled by default and must be explicitly enabled (policy in `anyfs-container`).
- Rewrote comparison docs to use current terminology (`AnyFS Container`) and avoid broken glyphs in tables.
- Moved the historical graph-store RFC (`book/src/architecture/vfs-container-design.md`) into the Appendix navigation and labeled it as historical.

---

## Remaining gaps / suggestions

- Encoding cleanup: several historical docs still contain mojibake/box-drawing artifacts. Consider normalizing to UTF-8 and using plain `Yes/No/Varies` tables in the book.
- Add an explicit "backend conformance test suite" section (or a small dedicated chapter) to document expected semantics across backends.
