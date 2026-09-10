---
tier: tale
title: Exact AXE config composition and mutation planning
goal:
  Centralize layered AXE composition and exact-key sparse mutation planning in Rust,
  with conflict-safe atomic Python application and runtime-preview parity.
bead: plans-7.1
create_time: 2026-09-09 19:53:00
status: wip
---

- **PARENT:** [202607/axe_config_editor.md](https://github.com/sase-org/sase--plans/blob/main/202607/axe_config_editor.md)
- **BEAD:** plans-7.1
- **AGENTS:**
  - [bbugyi200.athena.plans-7.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.plans-7.1.md)

# Exact AXE config composition and mutation planning

## Goal

Complete bead `plans-7.1` by making `sase-core` the frontend-neutral authority for
layered AXE composition and exact-key entry mutation planning, routing the Python
runtime and editor facades through that contract, and hardening config application
against stale previews and partial writes. Preserve existing Python patch surfaces where
tests or callers depend on them, and do not implement the later Textual add/edit
workflow from sibling phases.

## Context and constraints

- The current runtime only invokes keyed AXE composition when a merged result contains
  map-form chops; the actual normalization, replace/concatenate handling, duplicate
  detection, provenance tracking, and keyed merge live in
  `src/sase/axe/_config_layers.py`.
- The generic Rust config editor currently accepts only a dotted `path` and derives
  `key_path` by splitting on `.`, which cannot represent dynamic lumberjack or chop
  names containing dots exactly.
- Python config application currently writes `EditPlanResult.new_text` directly, without
  checking that the previewed target bytes are still current and without atomic
  replacement/mode preservation.
- Layer discovery, file/YAML I/O, and comment-preserving rewrites stay in Python.
  Composition, inventory, mutation decisions, previews, and validation belong in
  `sase-core`.
- Existing dotted scalar-path callers and patchable Python wrappers must remain
  compatible. The later shared modal and AXE TUI integration phases need typed,
  frontend-neutral facades from this work, but no Textual UI should be added here.
- Do not edit crate versions. Keep the parent epic `plans-7` open and do not create
  beads.

## Implementation plan

1. **Add the Rust AXE composition contract.**
   - Introduce focused AXE config wire/domain modules in `sase-core` for ordered layer
     input, composed effective config, exact-segment provenance, normalization/migration
     metadata, and deterministic diagnostics.
   - Port the behavior now in `sase.axe._config_layers`: normalize legacy string/object
     chop lists to keyed identities for composition, merge map-form chop entries
     sparsely by identity, honor each contributing layer's concatenate/replace policy
     (including replacement of a lumberjack's chop collection), preserve field-level
     source attribution, and retain fail-closed duplicate/migration/strict-shape
     diagnostics.
   - Export the APIs through `sase_core` and `sase_core_rs`, using JSON-compatible wire
     shapes and explicit schema versions consistent with existing config and AXE
     bindings.

2. **Provide exact entry inventory and mutation planning.**
   - Model a lumberjack or base chop identity with exact key-path segments rather than a
     display string. Inventory results should expose the effective entry,
     base-versus-expanded identity metadata, enabled state, field provenance, and every
     writable layer's raw contribution and original chop representation.
   - Accept a target layer plus a batch of per-field set/unset operations. Apply only
     touched fields to that layer's sparse raw contribution; an unset removes the
     target-layer override so lower layers can become effective again.
   - When editing a legacy list-form chop in the selected layer, promote only that
     layer's chop collection to an equivalent keyed map when necessary, preserving all
     entries and semantics. Adding an overlay map entry above built-in list defaults
     must retain all lower-layer chops in the effective result.
   - Return the generic write plan, target raw before/after values for source-preserving
     diff generation, effective before/after AXE previews, generic schema diagnostics,
     strict AXE diagnostics, and deterministic composition diagnostics.

3. **Extend generic config paths without breaking dotted callers.**
   - Add optional exact `key_path` segments to the generic Rust config edit request/wire
     and Python `plan_config_edit` facade, with validation that rejects empty or
     conflicting path inputs.
   - Continue accepting the existing dotted `path` for scalar callers and preserve their
     preview/display path behavior. Always carry the resolved exact segments in
     `ConfigWritePlan` and use those segments for lookup, set, unset, and YAML
     rewriting.
   - Add coverage for mapping keys containing dots, dashes, and other already-valid
     characters; do not introduce a new global AXE identity grammar.

4. **Route Python runtime and facades through Rust.**
   - Add thin typed Python serializers/dataclasses/facades for AXE composition, entry
     inventory, and mutation plans, following the existing `sase.core.axe_chop_facade`
     and `sase.config.inventory/edit` patterns.
   - Replace Python-owned keyed composition in the runtime load path with the Rust
     result so runtime parsing and editor previews share exactly one effective AXE
     config. Retain narrow compatibility wrappers/names in `_config_layers.py` and
     `sase.axe.config` only where current imports and monkeypatch-based tests require
     them.
   - Ensure cache keys and invalidation continue to reflect the discovered layer stack
     while diagnostics are converted to the established
     `AxeConfigDiagnostic`/`AxeConfigError` types.

5. **Make Python apply conflict-safe and atomic.**
   - Treat the previewed target bytes (including the missing-file state) as the
     optimistic concurrency token. Immediately before writing, re-read the target and
     return/raise a typed stale-plan conflict without changing the file when it differs.
   - Write the previewed bytes to a temporary file in the destination directory,
     flush/fsync as appropriate, preserve existing applicable mode bits, and atomically
     replace the destination. Clean up temporary files on every failure path.
   - Clear config caches only after successful replacement. Expose enough typed conflict
     information for a later modal to retain its draft and offer reload/re-plan, while
     keeping existing successful `AppliedResult` behavior compatible.

6. **Test domain parity and failure behavior.**
   - In `sase-core`, add unit/integration and Python-wire parity cases for exact dynamic
     keys, list-to-map normalization, sparse inherited edits, field reset, built-in list
     plus overlay map, replace layers, in-layer promotion,
     duplicate/migration/provenance diagnostics, generic schema validation, and strict
     AXE validation.
   - In `sase`, migrate/extend AXE layer tests to prove the compatibility wrappers use
     Rust, and add facade/edit tests for exact key segments, source-preserving comments,
     runtime-versus-preview parity, stale-file and missing-file conflicts, mode
     preservation, atomic replace failures, temp cleanup, and cache invalidation only
     after success.
   - Keep tests deterministic and avoid adding TUI work or filesystem reads to
     navigation/render paths.

## Validation

1. In the linked `sase-core` checkout, run `cargo fmt --all -- --check`,
   `cargo clippy --workspace --all-targets --all-features -- -D warnings`, and
   `cargo test --workspace --all-features` (or the repository's equivalent full checks
   if a checked-in command documents them).
2. In the `sase` checkout, run `just install` first so the editable Rust binding
   reflects the linked core changes.
3. Run focused Python tests for config editing/inventory and AXE lumberjack
   configuration while iterating.
4. Run the required full `just check` before completion.
5. Reinspect both worktrees, confirm only intended source/test changes remain, close
   `plans-7.1`, and verify `plans-7` is still open.

## Completion criteria

- Runtime-loaded and mutation-previewed AXE config are produced by the same Rust
  composition logic for the same layer stack.
- Exact segment paths safely address dynamic lumberjack/chop names containing dots or
  dashes while existing dotted scalar edits remain compatible.
- Entry edits are sparse, resettable, layer-aware, and preserve lower-layer list
  defaults plus untouched target-layer legacy entries.
- Composition, generic schema, and strict AXE diagnostics include stable provenance and
  duplicate/migration information.
- Config apply rejects stale previews, performs an atomic mode-preserving replacement,
  leaves files/caches untouched on failure, and preserves comment-aware YAML output.
- Both repositories' focused and full required checks pass; only `plans-7.1` is closed.
