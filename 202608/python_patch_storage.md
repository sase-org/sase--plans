---
tier: tale
title: Canonicalize the Python Patch domain and ProjectSpec storage
goal:
  Python exposes one canonical Patch/Stitch domain and storage implementation while
  legacy ChangeSpec APIs and persisted ProjectSpec data remain fully compatible.
size: medium
proposed_by: bbugyi200.athena.sase-hn.2
bead: sase-hn.2
create_time: 2026-08-08 13:46:35
status: wip
---

- **PARENT:**
  [202608/patch_and_stitch_terminology.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_and_stitch_terminology.md)
- **BEAD:**
  [sase-hn.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.2.md)

# Complete `sase-hn.2`: canonical Python Patch domain and ProjectSpec storage

## Goal

Make `sase.ace.patch` and the Python `Patch`/`Stitch` wire surface canonical while
preserving the behavior and data contracts of the existing ChangeSpec implementation.
Both canonical (`## Patch`, `STITCHES:`, `stitches`, `stitch_id`) and legacy
(`## ChangeSpec`, `COMMITS:`, `commits`, `commit_entry_num`/`entry_id`) inputs must
remain readable. Existing legacy Python imports and attributes must remain thin aliases
to the same implementations and objects, not a second implementation.

This tale is limited to bead `sase-hn.2`. Workflow/CLI/TUI terminology migration is
owned by later epic phases, so those call sites may continue importing legacy names, but
their low-level ProjectSpec mutations must work with either section spelling and must
use `STITCHES:` when creating a section that did not already exist.

## Existing contracts to preserve

- The completed Rust-core phase exposes `parse_patch_project_bytes` and canonical
  `PatchWire`-shaped dictionaries with `stitches`, hook `stitch_id`, and mentor
  `stitch_id`, while `parse_project_bytes` and the legacy wire shape remain available.
- ProjectSpec schema version remains 5; canonical naming is additive and does not
  require a schema bump or a dependency-version edit.
- A Patch may have no PR URL. Proposal stitch IDs such as `2a`, multiline bodies,
  CHAT/DIFF/PLAN drawers, suffixes, hooks, mentors, timestamps, deltas, refs, archive
  placement, locking, and atomic writes retain their current semantics.
- An unrelated edit to a legacy `COMMITS:` section preserves that header and avoids
  whole-file churn. New Patch blocks and newly inserted stitch sections use `STITCHES:`.
  Both `## Patch` and `## ChangeSpec` delimit records, including files that omit
  headings and use `NAME:` boundaries.
- Real VCS commit types, hashes, history, statistics, and `sase commit` vocabulary are
  outside this rename.

## Implementation

1. Establish `src/sase/ace/patch/` as the one implementation package by moving the
   existing models, parser, discovery, cache, locks, archive, migration, raw-text,
   section-order, refs, validation, formatting, and suffix helpers there. Rename the
   canonical public symbols and internal stitch-domain state: `Patch`, `Stitch`,
   `StitchDict`, stitch-ID parsing/accessors, Patch discovery/cache/raw-text/locking/
   persistence functions, and `PATCH_SECTION_ORDER`. Keep established neutral names such
   as hook/comment/mentor/timestamp/delta records. Teach heading and section boundary
   helpers to accept both canonical and legacy spellings.

2. Implement constructor and attribute compatibility at the canonical model boundary.
   `Patch(..., stitches=...)` is canonical; `commits=` remains accepted, conflicting
   simultaneous values fail clearly, and `.commits` is a read/write alias for
   `.stitches`. `ChangeSpec is Patch` and `CommitEntry is Stitch` through the legacy
   facade so `isinstance`, dataclass equality, mutable lists, caches, and callers all
   observe one source of truth. Provide analogous aliases for renamed public helpers and
   stitch-reference fields without duplicating state.

3. Replace `sase.ace.changespec` with deliberate compatibility modules that re-export
   canonical objects and legacy spellings from `sase.ace.patch`, including supported
   submodule imports such as `.models`, `.parser`, `.archive`, `.locking`, and
   `.section_order`. Keep these shims small and explicit. Update canonical package
   internals and Python core facades to import Patch/Stitch names only; do not migrate
   unrelated workflow, CLI, scheduler, integration, or TUI call sites assigned to later
   phases.

4. Add canonical Python wire records mirroring the Rust phase:
   `PATCH_WIRE_SCHEMA_VERSION`, `SUPPORTED_PATCH_WIRE_SCHEMA_VERSIONS`, `StitchWire`,
   canonical hook/mentor stitch reference records, and `PatchWire(stitches=...)`.
   Preserve `ChangeSpecWire`, `CommitWire`, legacy constants, field order, JSON shape,
   and conversions as aliases or explicit compatibility adapters. Make tolerant
   canonical rehydration accept every supported schema version and either canonical or
   legacy field spelling, including `stitches`/`commits`,
   `stitch_id`/`commit_entry_num`, `stitch_id`/`entry_id`, and `pr_url`/`cl_or_pr`.

5. Make `sase.core.parser_facade` canonical: Patch file parsing returns `Patch` objects
   and byte parsing calls the Rust `parse_patch_project_bytes` binding and returns
   `PatchWire`. Retain legacy parse entry points with their historical wire shape and
   imports. Update the query-corpus adapter to use canonical Patch wire conversion while
   preserving its deterministic payload behavior.

6. Centralize ProjectSpec stitch-section recognition and creation rules in the canonical
   storage package, then apply them to every current low-level writer and mutator that
   scans, inserts, formats, renumbers, or checks stitch entries. Existing `STITCHES:`
   and `COMMITS:` headers are both recognized; operations preserve the header already
   present, while a missing section is created as `STITCHES:`. Ensure
   archive/raw-text/ref/delta/status-field operations treat both `## Patch` and
   `## ChangeSpec` boundaries correctly and preserve section ordering, two-blank-line
   boundaries, drawers, multiline bodies, and atomic-write locking.

7. Add focused tests under canonical Patch/core-wire coverage and retain the legacy
   suites as compatibility tests. Cover:
   - canonical and legacy imports resolving to identical classes/functions;
   - `stitches=`/`.stitches` plus `commits=`/`.commits`, conflict behavior, proposal
     IDs, optional PRs, and stitch-reference aliases;
   - old/new/mixed headings and section headers parsing to equivalent Patch objects;
   - canonical and legacy Rust binding shapes across all supported schema versions,
     deterministic JSON key order/shape, and both parse facade entry points;
   - new-section `STITCHES:` output, preservation of an existing `COMMITS:` header,
     append/modify/renumber behavior, archives, refs, mixed records, two blank lines,
     section order, drawers, multiline bodies, and no-op/locked atomic writes.

## Validation

1. Run `just install` so the workspace builds and installs the completed local Rust core
   contract.
2. Iterate with focused Patch, ProjectSpec persistence, parser, wire, query-corpus,
   archive, refs, locking, commit-entry mutation, and renumber tests.
3. Run `just fmt`, inspect the diff for accidental VCS-commit renames and duplicated
   implementations, then run `just check` as required for SASE source changes.
4. Because this phase crosses the Rust binding boundary and ProjectSpec storage is a
   broad compatibility surface, run `just check-full` and `just rust-check`. Re-run the
   focused canonical/legacy compatibility matrix after any fixes.
5. Record any out-of-scope issue as a `PROPOSED FOLLOW-UP:` note on `sase-hn.2`, never
   as a new bead. Close only `sase-hn.2` with a note naming the exact verification that
   passed; leave parent epic `sase-hn` open for its land agent.

## Non-goals

- Renaming lifecycle, CLI, metadata, scheduler, integration, or TUI public surfaces
  beyond the minimum storage-header compatibility needed here.
- Changing Patch state transitions, proposal/renumber semantics, hook/mentor behavior,
  archive placement, PR policy, or external-PR discovery.
- Renaming actual VCS commit concepts, hand-editing Rust release versions, deleting
  legacy aliases, or rewriting legacy files solely to canonicalize their spelling.
