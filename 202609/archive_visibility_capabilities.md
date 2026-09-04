---
tier: tale
title: Immutable agent archive, local visibility, and honest capabilities
goal:
  Archived runs retain canonical provenance and content while local visibility and
  validated capabilities accurately govern viewing, revival, and restart behavior.
size: medium
proposed_by: bbugyi200.apollo.sase-w2.7
bead: sase-w2.7
create_time: 2026-09-04 00:21:11
status: wip
---

- **PARENT:** [202609/athena_agent_sync_repair.md](athena_agent_sync_repair.md)
- **BEAD:**
  [sase-w2.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w2/sase-w2.7.md)

# Immutable agent archive, local visibility, and honest capabilities

## Objective

Complete phase `sase-w2.7` by making the archived run record immutable and canonically
identified by `(source_username, source_machine, source_run_id)`, moving hide/show/pin
state into a machine-local projection, and publishing explicit view/revive/restart
capabilities whose assertions are validated against the persisted inputs.

The implementation must preserve the existing v2 import transaction's atomicity, keep
foreign terminal imports hidden by default, retain compatibility with dismissed bundles
and loader/index markers, and ensure that dismissing or reviving an imported run never
changes its source provenance or archive bytes.

## Existing seams and constraints

- Shared identity, capability policy, visibility states, and transition validation
  belong in `sase-core/crates/sase_core`; expose them through `sase_core_rs`. Python may
  own the SQLite/filesystem projection and TUI scheduling, but must not reimplement the
  policy.
- `src/sase/agents_sync/v2_import_rendering.py` already places `imported_source_owner`
  and `imported_source_run_id` in imported artifacts and bundle payloads.
  `src/sase/agents_sync/v2_import_transactions.py` stages those bundles and name-based
  dismissed identities in one recoverable journal transaction.
- `src/sase/ace/dismissed_agents_state.py` and the dismissed-bundle index still use the
  collision-prone `(AgentType, cl_name, raw_suffix)` projection, and revival currently
  deletes bundles. These become compatibility projections; they cannot remain the
  archive's source of truth.
- `src/sase/agents_sync/publication_snapshot.py` always writes v2 metadata/state/commit
  payloads and conditionally writes prompt/chat/restart payloads. Its strict v2 readers
  are the publication boundary at which capability claims must be checked.
- All archive/index reads used by ACE must remain paged and off the Textual event loop.
  Rendering and keystroke handlers must consume already-loaded capability/provenance
  fields and perform no new disk work.

## Implementation

1. Extend the Rust archive domain and Python binding with explicit canonical contracts.
   Add wire types for the canonical archive key, `hidden | visible | pinned` visibility,
   capability input facts, the three capability booleans (`historically_viewable`,
   `durably_revivable`, `restartable`), and a stable sorted `missing_requirements` list.
   Centralize validation so malformed owner/run IDs, unsupported visibility values, and
   asserted capabilities that exceed the available persisted inputs are rejected. A
   restartable record must have its durable prompt and complete recorded
   model/provider/effort parameters; a durable-revival claim must have the
   loader-reconstructible archive inputs. Export typed PyO3 functions and add Python
   facade/wire conversion without a fallback policy implementation.

2. Evolve dismissed-bundle storage into a canonical immutable archive plus local
   projection. Add canonical source identity, capability fields, archive schema/hash,
   and projection visibility to the bundle/index models. Derive local-run archive
   identity from the configured local owner and the same stable run-id inputs used by
   publication; preserve imported owner/run identity verbatim through Agent loading,
   cleanup DTO serialization, artifact restoration, and re-dismissal. Store/query
   visibility separately in the machine-local SQLite index keyed by the canonical
   triple. Make archive writes idempotent for byte-identical records and reject or
   diagnose conflicting bytes for an existing key instead of overwriting provenance.
   Rebuild/migrate legacy bundles by deriving canonical keys only from available
   evidence and retain a clearly marked compatibility identity when historical local
   inputs cannot be recovered.

3. Change dismissal, revival, and v2 import to operate on canonical archive keys.
   Foreign terminal imports create immutable archive records with `hidden` projection
   rows inside the existing journaled transaction. Dismissal changes a record to
   `hidden`; revival changes it to `visible`; pin support uses `pinned`. None of these
   transitions edits or deletes the canonical bundle. Continue generating
   `dismissed_agents.json`, dismissed-bundle summaries, and artifact-index hidden
   markers as compatibility views of hidden canonical rows so existing scanners remain
   functional during the transition. Key caches, pagination, group references, and
   revive selection by canonical identity, using the legacy tuple only as a display or
   loader bridge. Update recovery/finalization so a crash cannot publish a visible row
   without its archive/projection state or partially flip a family.

4. Publish and import explicit capabilities. Compute capability facts from the actual
   files and portable execution metadata assembled for each `InventoryRun`, validate
   them through Rust before `publication_snapshot` writes a run, and persist the
   resulting capability object in the strict v2 run record. Teach snapshot/run decoding
   and package validation to accept older records by deriving their conservative
   capabilities, while every newly rendered record carries explicit fields. Reject a
   claimed `restartable: true` when prompt or required model parameters are absent.
   Propagate validated capabilities into imported artifacts, canonical bundles, saved
   family references, and archive summaries.

5. Make ACE capability-aware without adding synchronous UI I/O. Add canonical source
   identity and capability fields to the loaded `Agent` model. The revival archive
   should distinguish historical viewing, durable restoration, and restartability, show
   `missing_requirements` for incomplete historical records, and never label an
   incomplete record fully revivable. Gate prompt-based retry/edit/relaunch paths on the
   persisted `restartable` capability while continuing to allow historical viewing and
   ordinary visibility restoration where their respective capabilities permit. Keep
   archive page loading and content hydration in the existing background-worker paths.

6. Cover migrations and lifecycle behavior with focused Rust, binding, Python, and
   end-to-end regression tests. Include owner/run identity collisions with reused local
   names; invalid capability assertions; old v2 records without explicit capabilities;
   foreign-import hidden defaults; pinned/visible/hidden transitions; crash recovery;
   and capability-aware UI/action behavior. Add the required publish -> import ->
   dismiss -> revive test that byte-compares the canonical archive before and after
   every visibility transition, plus a restart-planning matrix proving every record
   marked restartable succeeds and incomplete records are not offered as restartable.

## Verification

1. In the linked `sase-core` repository, run focused archive/capability and PyO3 binding
   tests, then the repository-required `just check`.
2. In the primary `sase` repository, run focused agents-sync publication/import,
   dismissed-archive migration/lifecycle, cleanup DTO, and ACE revival/relaunch tests.
3. Run `just install` before repository-wide verification in the ephemeral workspace,
   then run `just check` as required by project memory.
4. Run `sase bead epic-symbols sase-w2.7`; resolve every remaining entry or re-key it to
   the still-open parent/later phase, then close only `sase-w2.7` with a note naming the
   verified Rust, binding, lifecycle, publication/import, and UI test coverage.
