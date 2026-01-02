# Critical Repository Consistency Report (v5)

## Status: Mixed State / Version Mismatch

The repository is currently in a fragmented state between two major architectural versions.

### The Conflict

*   **Version 0.4.0 (Target):** Two-Layer Architecture (`anyfs` = Inode-based, `anyfs-container` = Path-based).
    *   **Source:** `book/src/architecture/design-overview.md` (Updated)
*   **Version 0.3.0 (Legacy):** Single-Layer Architecture (`anyfs` = Path-based `VfsBackend` trait).
    *   **Source:** `book/src/review/decisions.md` (Outdated)
    *   **Source:** `book/src/comparisons/technical-comparison.md` (Outdated)

### Details

1.  **Main Design Document (`design-overview.md`)**
    *   Correctly describes the new **Two-Layer Architecture**.
    *   Defines `Vfs` trait as **Inode-based** (`create_inode`, `lookup`, etc.).
    *   This is the "Source of Truth" for v0.4.0.

2.  **Peripheral Documents (`decisions.md`, `technical-comparison.md`)**
    *   Still reference `VfsBackend` as a **20-method Path-based trait** (matching `std::fs`).
    *   This directly contradicts `design-overview.md`.
    *   `decisions.md` explicitly calls the "Path-Based" design authoritative, which is now false relative to `design-overview.md`.

### Recommendation

**Proceed with `update_instructions` immediately.**

The `update_instructions/update-instructions-anyfs-design-manual.md` file contains the specific steps required to update the peripheral documents to match the v0.4.0 architecture in `design-overview.md`.

**Action Items:**
1.  Execute `update_instructions` to align `decisions.md` and `technical-comparison.md`.
2.  Update `AGENTS.md` and `README.md` as per instructions.
3.  Create new Architecture diagrams.
