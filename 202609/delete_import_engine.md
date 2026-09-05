---
tier: tale
title: Delete the agents-sync import engine and legacy v1 leg
goal: Remove every runtime that can capture or materialize remote agent imports while
  preserving agents-sidecar publication and the explicit purge of historical local
  import state.
size: medium
bead: sase-ws.4
proposed_by: bbugyi200.apollo.sase-ws.4
status: done
---

- **PARENT:** [202609/remove_agents_sync_import.md](remove_agents_sync_import.md)
- **BEAD:**
  [sase-ws.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ws/sase-ws.4.md)

# Delete the agents-sync import engine and legacy v1 leg

## Context

Phase `sase-ws.4` follows the completed publication-only sync and explicit
`purge-local-state` phases. The parent design reference is currently unresolved in the
plans store, so the authoritative phase and parent bead descriptions, the completed
phase boundaries, and the current repository were used to recover the intended seam.

The agents sidecar remains a publication channel. Prompt archives, agent pages,
publication/referenced-by outboxes, ownership/inventory rendering, and the shared v2
schema/read helpers they consume must continue to work. This phase removes only the
opposite direction: detecting foreign hoods, caching them locally, integrating them into
artifacts and dismissed state, and mutating the name registry on their behalf.

## Implementation

1. Delete the import-only agents-sync subsystem and its tests.
   - Remove the incoming-cache path, metadata, receipt, storage, reconciliation,
     detection, and integration modules.
   - Remove the v2 import package discovery, planning, history, staging/journaling,
     transaction, importer, and v1-adoption modules, fixtures, and focused tests.
   - Remove the legacy v1 per-machine forget implementation and the v1 bundle
     materialization path. Move the small repository/commit inspection helpers still
     used by publication inventory and commit publication into a publication-neutral
     module so deleting the v1 bundle module does not regress outbound publication.
   - Retain common v2 publication models and serialization/read helpers where current
     publication, prompt archives, artifact providers, history, or doctor checks still
     consume them; do not fold the later Rust-API cleanup from `sase-ws.5` into this
     phase.

2. Preserve the explicit all-state purge without depending on deleted import code.
   - Make `purge_local_state.py` own the legacy on-disk import, incoming-cache, and
     receipt path conventions and tolerant JSON reads needed to find already-existing
     state.
   - Keep dry-run/apply behavior, artifact-index and dismissed-index repair, registry
     rebuild, and reporting intact. Adapt its tests to seed historical directories and
     receipt files directly, proving cleanup remains available after the import engine
     is gone.
   - Remove import-journal recovery/index-repair hooks elsewhere; the purge is the only
     remaining code allowed to traverse those historical directories.

3. Remove import-facing CLI and registry mutation surfaces.
   - Delete `sase agent names forget-import` parser registration, dispatch, rendering,
     and tests. Keep `migrate-auto` and `purge-local-state` behavior and update their
     usage/parser assertions.
   - Delete the imported-name claim mutation module and its public facade wrappers and
     exports, then remove tests that exercise those now-unreachable writes. Preserve
     read/rebuild compatibility for existing `import_v1` and `import_v2` registry rows
     so `purge-local-state` can report and clear historical materialization.
   - Remove the `v1_import_retired` enum member, definition, schema entry, and
     both-state tests: deleting the disabled import branch makes retirement
     unconditional. Do not close `sase-wc` or any bead other than the assigned phase;
     bead lifecycle cleanup belongs to the epic land workflow under the user's explicit
     closure constraint.

4. Remove incomplete-import visibility gates from ACE and audit inventories.
   - Treat any historical artifact or dismissed bundle already on disk as ordinary
     readable history; remove transaction-journal completion lookups, visibility
     predicates, cache clearing, and tests that hide partially committed imports.
   - Delete the importer-specific entries from artifact marker/directory/dismissed-save
     audit inventories and adjust affected expected counts. Also fix the known phase-1
     proc-producer inventory literal from 43 to the current 42 producers if it is still
     present, because the parent bead records that deterministic epic-caused failure.
   - Sweep Python production and tests for imports of deleted modules and for the three
     phase epic symbols: `IncomingCaptureProgress`, `capture_fetched_agent_updates`, and
     `integrate_agent_imports_with_receipts`.

5. Refresh test-shard timing metadata and phase symbol ownership.
   - Run the focused agents-sync, registry, CLI, ACE loader/dismissed-history, index
     repair, feature-flag, doctor/publication, and audit suites while iterating.
   - Update `tests/shard_timings.json` using the repository's supported timing refresh
     workflow after the deleted tests settle, without hand-preserving entries for
     removed test nodes.
   - Remove the three `sase-ws.4` `--epic-symbol` registrations once their definitions
     are gone. Before closure, run `sase bead epic-symbols sase-ws.4` and resolve any
     remaining symbol ownership; re-key only genuinely surviving work to the parent or a
     still-open later phase.

## Verification

1. Because this is an ephemeral workspace, run `just install` before repository
   verification.
2. Run focused pytest coverage for the surviving purge, agents-sync publication/status
   path, registry and CLI behavior, ACE artifact/dismissed loaders, index repair,
   feature flags, and filesystem/mutation audit inventories.
3. Search source and tests for references to every deleted import module, the removed
   forget command and flag, imported-name claim APIs, transaction visibility gates, and
   the three epic symbols. Classify remaining words such as “incoming commits” as
   unrelated update/VCS behavior.
4. Run `just fmt`, `git diff --check`, and the mandatory `just check` agent lane.
5. Run `sase bead epic-symbols sase-ws.4`; after it is empty, close only `sase-ws.4`
   with a note naming the focused checks and `just check` result. Do not close the
   parent epic, an ancestor, or `sase-wc`.

## Out of scope

- Removing orphaned shared Rust/core import APIs; `sase-ws.5` owns that boundary.
- The final documentation, memory-decision, generated-reference, and cross-repository
  sweep; `sase-ws.6` owns that work.
- Changing the outbound agents-sidecar publication, prompt archive, agent-page,
  provenance, or Referenced By behavior.
- Closing the parent epic, any ancestor plan bead, or any bead other than `sase-ws.4`.
