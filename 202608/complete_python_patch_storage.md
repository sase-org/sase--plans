---
tier: tale
title: Complete the Python Patch and ProjectSpec storage migration
goal:
  Make Patch and Stitch canonical in the Python domain and storage layer while
  preserving legacy imports, attributes, ProjectSpec text, and wire compatibility.
size: medium
proposed_by: bbugyi200.athena.sase-hn.2
bead: sase-hn.2
create_time: 2026-08-08 15:50:53
status: wip
---

- **PARENT:**
  [202608/patch_and_stitch_terminology.md](https://github.com/sase-org/sase--plans/blob/main/202608/patch_and_stitch_terminology.md)
- **BEAD:**
  [sase-hn.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hn/sase-hn.2.md)

# Complete the Python Patch and ProjectSpec storage migration

## Goal

Finish phase `sase-hn.2` so `sase.ace.patch` is the canonical Python domain and
ProjectSpec storage implementation for Patch/Stitch terminology, while
`sase.ace.changespec` remains a thin, tested compatibility facade. Preserve all workflow
behavior, legacy imports and attributes, legacy `## ChangeSpec` / `COMMITS:` data,
proposal stitch IDs such as `2a`, and deterministic ProjectSpec/wire output.

## Current state

Commit `3e6da8d5f` already introduced `sase.ace.patch`, `PatchWire` / `StitchWire`, dual
ProjectSpec parsing, canonical `STITCHES:` creation, and compatibility tests. The phase
bead nevertheless remains open, and static inspection shows the canonical package is
still largely the former implementation copied under a new path:

- `Patch` stores `commits` and exposes `stitches` as the alias rather than the other way
  around.
- hook and mentor records store `commit_entry_num` / `entry_id` and expose canonical
  properties as aliases.
- parser state, typed dictionaries, functions, caches, discovery records, validation,
  archive/persistence arguments, docstrings, and internal locals in `sase.ace.patch`
  still use ChangeSpec/CommitEntry vocabulary.
- the compatibility package mostly wildcard-imports the canonical package, so it does
  not yet establish a clear one-way compatibility boundary.
- the new tests cover basic aliases and headers but not the full mixed-file, round-trip,
  archive, section-order, multiline, refs, cache, and atomic-write invariants required
  by the approved epic design.

The work is therefore a completion and hardening pass over the landed migration, not a
new behavioral redesign.

## Scope and constraints

- Keep this phase limited to Python domain models, ProjectSpec parsing/formatting and
  persistence, and the Python adapters for the Rust wire contract delivered by phase
  `sase-hn.1`.
- Do not rename CLI commands, lifecycle/workflow modules, TUI surfaces, configuration,
  notifications, or linked integrations assigned to later epic phases. Adjust such a
  caller only when needed to consume the canonical domain/storage API without changing
  its external contract.
- Keep real VCS commits, SHAs, git-log records, and the `sase commit` command named
  “commit.” Only ProjectSpec history entries become stitches.
- Preserve legacy imports and construction forms through the `sase.ace.changespec`
  compatibility package and explicit compatibility properties or wrappers. Do not
  maintain two implementations.
- Never rewrite an existing legacy section merely because it was read. New Patch records
  and newly added history sections use `## Patch` / `STITCHES:`; an existing `COMMITS:`
  section retains that header during updates.

## Implementation

### 1. Make the canonical domain truly Patch/Stitch-first

- Refactor `src/sase/ace/patch/models.py` so `Patch` stores `stitches`, `Stitch` is the
  concrete history-entry type, hook/mentor cross-references store `stitch_id`, and
  canonical helpers and methods use stitch terminology.
- Retain `commits`, `CommitEntry`, `commit_entry_num`, `entry_id`, and old helper names
  only as explicit compatibility properties, constructor keywords, or aliases exposed
  through the legacy facade. Reject conflicting canonical/legacy constructor values and
  keep mutation through either spelling synchronized.
- Rename canonical cache/discovery record types, parser state, typed dictionaries,
  formatter/parser helpers, validation arguments, archive/persistence parameters,
  internal locals, errors, and docstrings throughout `src/sase/ace/patch/`. Ensure the
  canonical package exports Patch/Stitch names rather than making callers depend on
  legacy vocabulary.

### 2. Establish a one-way compatibility facade

- Keep implementation in `src/sase/ace/patch/` and reduce modules under
  `src/sase/ace/changespec/` to deliberate adapters that re-export canonical behavior
  and define the old class, function, cache, and helper names.
- Preserve identity where historically expected (`ChangeSpec is Patch`,
  `CommitEntry is Stitch`) and preserve legacy constructor keywords, attributes, module
  import paths, return types, and exception behavior.
- Migrate in-scope canonical callers and wire adapters to import/use Patch/Stitch APIs.
  Leave later-phase public command/workflow names intact, routing them through the
  compatibility facade where necessary.

### 3. Harden ProjectSpec parsing and persistence

- Centralize recognition of `## Patch` and `## ChangeSpec` headings plus `STITCHES:` and
  `COMMITS:` sections in the canonical storage layer, and ensure parser state records
  which history header was present when edits must preserve it.
- Audit every in-scope raw-text reader, formatter, ref updater, delta updater, archive
  move, migration, cache, discovery, validation, and atomic writer so canonical and
  mixed ProjectSpec files behave identically.
- Ensure newly created records/sections use canonical spelling, while legacy records
  retain their heading/header during unrelated or append-only edits. Preserve two blank
  lines between records, section ordering, drawers, multiline descriptions and stitch
  bodies, proposal suffixes, refs, archive placement, and path-derived locking.
- Avoid whole-file churn: tests must compare exact text around untouched records and
  prove failed/conflicting updates do not partially write data.

### 4. Complete Rust-wire adapters without duplicating core behavior

- Use the existing `PatchWire` / `StitchWire` records and Rust parser facade as the
  source contract. Make canonical Python conversion functions consume and emit
  `stitches` / `stitch_id` deterministically.
- Retain explicit legacy `ChangeSpecWire` / commit-field adapters for every supported
  schema and field spelling, including mixed canonical/legacy input. Reject conflicting
  aliases instead of silently choosing one, and preserve JSON key order/shape where it
  is contractual.
- Keep shared parser/domain semantics in the Rust core; this phase changes only Python
  bindings, conversion, compatibility, and storage glue.

### 5. Add focused compatibility and round-trip coverage

- Expand canonical tests under `tests/ace/patch/` for construction and mutation through
  both spellings; parser equivalence; canonical new output; legacy-header preservation;
  mixed active/archive records; section ordering; multiline stitch bodies; numeric and
  proposal stitch IDs; refs; caches; and atomic locking/writes.
- Expand wire tests for old-producer/new-consumer and new-producer/legacy-adapter paths,
  deterministic JSON, field alias conflicts, empty/absent collections, hook and mentor
  stitch references, and Patch-without-PR records.
- Keep representative legacy-import tests under the compatibility path, and run the
  existing ProjectSpec/archive/refs/lifecycle suites to prove there is no semantic or
  text-format regression.

## Verification

1. Run `just install` first because this is an ephemeral workspace.
2. Run focused Patch, wire, ProjectSpec parser/persistence, archive, refs, cache, and
   legacy compatibility tests while iterating.
3. Run `just fmt`, then inspect the diff to confirm formatting did not cause unrelated
   ProjectSpec or fixture churn.
4. Run `just check`. If its selection broadens or reports an unusual selection, run
   `just check-full` as required by repository instructions.
5. Re-run a terminology audit over `src/sase/ace/patch` and the canonical wire adapter.
   Every remaining ChangeSpec/CommitEntry/commit-entry occurrence must be either a
   documented compatibility alias or a genuine VCS-commit concept; canonical locals,
   types, errors, and docstrings must use Patch/Stitch.
6. Confirm `git status` contains only intended phase changes and no generated-memory,
   CLI/TUI, linked-repository, or later-phase edits.
7. Close only `sase-hn.2` with a note listing the exact focused and repository-wide
   verification that passed. Record any out-of-scope discovery as a
   `PROPOSED FOLLOW-UP:` note on this phase bead rather than creating a new bead.

## Acceptance criteria

- `sase.ace.patch` has one canonical Patch/Stitch implementation and no ordinary legacy
  vocabulary in its models, parsing, formatting, persistence, or validation internals.
- `sase.ace.changespec`, legacy imports, constructor keywords, properties, helper names,
  and supported wire payloads continue to work as thin adapters with tests.
- Old and new ProjectSpec text parse to equivalent Patch objects; exact round trips
  preserve existing headings/history headers and untouched text; new records/sections
  use canonical spelling.
- Stitch drawers, hook/mentor references, multiline bodies, and letter-suffixed proposal
  IDs survive parsing, mutation, wire conversion, archive moves, and formatting.
- Focused tests and `just check` pass, and the phase bead is closed without closing the
  parent epic.
