---
tier: tale
title: Make legacy cutover resumable and preserve frozen history
goal:
  A legacy artifact-link cutover never loses an unconverted queue row, resumes exactly
  the same import after any interrupted marker/baseline/commit step, requires an
  operator capability attestation before mutating, and leaves `links/**` byte-identical
  once imported.
size: medium
bead: sase-yy.8.4
proposed_by: bbugyi200.athena.sase-yy.8.4
status: done
---

- **PARENT:**
  [202609/artifact_link_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/artifact_link_landing_repairs.md)
- **BEAD:**
  [sase-yy.8.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.8.4.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-yy.8.4](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-yy.8.4.md)
- **COMMITS:**
  - [2dcd6a1](https://github.com/sase-org/sase/commit/2dcd6a136c715427c3916581a4e942822dc47155)
    — feat(artifact-links): make cutover import resumable

# Make legacy cutover resumable and preserve frozen history

Implements phase `sase-yy.8.4` (`cutover_recovery`) of epic `sase-yy.8`, the repair
child of `sase-yy`. Phases `sase-yy.8.1` (frozen producer identity), `sase-yy.8.2`
(durable ownership plus atomic installation) and `sase-yy.8.3` (event-union reduction
and bead projection) are closed; build on them and do not re-litigate their contracts.

## 1. Outcome

When this phase is done:

1. An ordinary outbox drain never removes a physical queue line it did not drain or
   explicitly drop. A mixed `[v1, v2]` queue stays `[v1, v2]` after an ineligible drain,
   and a malformed line survives too, counted and reported instead of deleted.
2. Legacy v1 rows become v2 events only through the importer's explicit deterministic
   conversion, which retains the original run's release eligibility, is exactly-once
   across replays, and reconciles rows the frozen baseline already represents rather
   than double counting them.
3. An import interrupted at any seam — partial marker write, marker commit, marker push,
   baseline object write, baseline commit, baseline push, or the final marker transition
   — resumes into _exactly the same_ import operation with unchanged bytes. Ordinary
   readers fail closed with an actionable diagnostic while the cutover is incomplete;
   the importer does not.
4. Two independent machines importing the same frozen source heads converge on the same
   markers. Conflicting source or import identities are refused with a diagnostic and no
   destructive reset of anything already written.
5. A project with zero legacy `links/**` rows has a defined, tested cutover path.
6. `sase artifact link import-indexes` previews by default and refuses to mutate without
   a matching operator fleet-capability attestation printed by that preview.
7. After import, every maintenance path is event-only: `_apply_artifact_renames`,
   `backfill_bead_endpoint_links`, and cross-workspace reconciliation leave `links/**`
   byte-for-byte unchanged and never mint a row with a direct imperative write. Doctor
   still reports post-import stragglers loudly.
8. Marker validation, migration identities, cutover state transitions, reader policy,
   baseline policy, and legacy-queue conversion identity live in Rust `sase-core`;
   Python keeps filesystem, git, locking, and CLI adapters.

## 2. Load this context before editing

- `sase bead show sase-yy.8.4` (including its notes).
- The epic plan, especially its `cutover_recovery` section:
  `sase artifact read plan:202609/artifact_link_landing_repairs.md "implementing sase-yy.8.4"`.
- The original requirements, especially "Fence, import legacy indexes, and cut over":
  `sase artifact read plan:202609/artifact_link_events_v2.md "sase-yy.8.4 cutover requirements"`.
- The landing audit and its reproduction evidence (blockers 2, 5, and 6 are this
  phase's; 1, 3, 4, and 7 belong to closed sibling phases and must not be re-repaired
  here):
  - `sase artifact read file:explicit:e7416d60ce1cf8d99a71fe3c "sase-yy.8.4 cutover defects"`
  - `sase artifact read file:explicit:7f5e04013d3b9e6e1b5a54db "sase-yy.8.4 reproduction script"`
  - `sase artifact read file:explicit:a6e3025e88ceccabf89d82ae "sase-yy.8.4 reproduction output"`
- `sase memory read cli_rules.md lint_and_test.md symvision.md -r "verify sase-yy.8.4 changes"`.
- Open the Rust core with `sase repo open sase-core -r "sase-yy.8.4 cutover contract"`
  and follow its `AGENTS.md`. Use only the path that command prints.

## 3. Defects to repair

1. **Silent legacy queue loss.** `read_artifact_link_outbox_entries` and
   `_read_entries_unlocked` in `src/sase/sdd/_artifact_link_outbox_io.py` drop any line
   `entry_from_line` cannot parse — which is every schema-v1 row-only entry, because
   `_entry_from_mapping` in `src/sase/sdd/_artifact_link_outbox_types.py` raises
   `artifact-link outbox row-only entries require import-indexes`, and every malformed
   line. `rewrite_artifact_link_outbox_without_ids` then rebuilds the file from the
   surviving parsed entries only, so an acknowledgement rewrite deletes them. The audit
   reproduces `[1, 2]` becoming `[2]` with `drained=0, dropped=0`.
2. **Conversion aborts on one bad line and cannot reconcile the baseline.**
   `convert_legacy_artifact_link_outbox_entries` raises on the first malformed line, so
   one corrupt row blocks the whole import, and it converts every legacy row into a new
   observation event even when the frozen baseline already carries that edge.
3. **Interrupted multi-root import cannot resume.** `_publish_marker` writes markers
   root by root and commits once; `inspect_artifact_link_cutover_markers` in
   `src/sase/sdd/_artifact_link_cutover_state.py` raises
   `artifact-link cutover markers are partial; missing roles: ...` for any mixed set, so
   the retry dies before recovery. `_frozen_roles` additionally calls
   `_assert_clean_git_root`, so an interrupted run's uncommitted marker makes the resume
   refuse the sidecar as dirty. A marker written but not committed is never
   re-committed, because `_publish_marker` skips byte-identical files and only commits
   `changed` paths.
4. **Post-import rename destroys frozen history.** `_apply_artifact_renames` in
   `src/sase/sdd/_artifact_link_renames.py` always calls `_rewrite_sidecar_indexes` and
   `_rewrite_aggregate` after queueing alias events. The audit reproduces one changed
   and one removed legacy index after a post-import document rename.
5. **`backfill_bead_endpoint_links` reintroduces frozen rows imperatively.**
   `src/sase/sdd/_artifact_link_store_reconcile.py` feeds `self._iter_sidecar_rows()`
   straight into `self._upsert_bead(...)`, bypassing both the import fence and the
   event-reduced bead projection landed by `sase-yy.8.3`.
6. **No operator capability attestation.** `src/sase/artifact_cli/link_import.py` prints
   a warning and mutates on `--apply` alone; the original plan requires the command to
   refuse until the operator confirms every machine writing this project's links runs a
   capable release.
7. **Shared policy lives in Python.** Marker parsing, import identity derivation,
   baseline event construction, and every state transition live in
   `_artifact_link_cutover_state.py` (504 lines) and `artifact_link_import_indexes.py`
   (740 lines, already past the `toobig` 700 warning tier), contrary to the core
   boundary.
8. **Empty legacy projects have no path.** `canonicalize_baseline_rows` in
   `crates/sase_core/src/artifact_link/events.rs` rejects a `baseline-import` with zero
   rows, so a project with no legacy `links/**` content cannot cut over at all.

## 4. Design

### 4.1 Cutover, migration, and baseline policy lives in Rust

Add `crates/sase_core/src/artifact_link/cutover.rs`, register it in
`crates/sase_core/src/artifact_link/mod.rs`, and re-export from
`crates/sase_core/src/lib.rs` following `ownership.rs`'s shape (that module is the
`sase-yy.8.2` precedent for wire structs, `deny_unknown_fields`, and error mapping).

```rust
pub const ARTIFACT_LINK_CUTOVER_WIRE_SCHEMA_VERSION: u64 = 1;

pub enum ArtifactLinkCutoverStateWire { Fenced, Imported }

pub struct ArtifactLinkCutoverRoleWire {
    pub role: String, pub kind: String, pub head: String,
    pub links_tree: String, pub remote_url: String,
}
pub struct ArtifactLinkCutoverImportIdentityWire {
    pub import_id: String, pub operation_id: String,
    pub source_head: String, pub created_at: String,
}
pub struct ArtifactLinkCutoverBaselineEventWire { pub digest: String, pub path: String }
pub struct ArtifactLinkCutoverEventStoreWire {
    pub schema_version: u64, pub minimum_event_schema_version: u64,
}
pub struct ArtifactLinkCutoverMarkerWire {
    pub schema_version: u64,
    pub state: ArtifactLinkCutoverStateWire,
    pub project_key: String,
    pub event_store: ArtifactLinkCutoverEventStoreWire,
    pub import: ArtifactLinkCutoverImportIdentityWire,
    pub roles: Vec<ArtifactLinkCutoverRoleWire>,
    pub baseline_event: ArtifactLinkCutoverBaselineEventWire,
}
```

Functions, all pure:

- `parse_artifact_link_cutover_marker(json: &str) -> Result<Marker, ArtifactLinkError>`
  — strict parse plus every validation Python's `_parse_*` helpers do today: schema pin,
  known state, 32-hex `operation_id`, sha256 `baseline_event.digest`, relative
  `baseline_event.path`, non-empty unique sorted roles, `role`/`kind` agreement.
- `artifact_link_cutover_marker_canonical_json(&Marker) -> String` — the canonical bytes
  Python writes and byte-compares.
- `artifact_link_cutover_import_identity(request) -> ImportIdentity` — moves
  `_baseline_import_identity`'s hashing wholesale: `source_head` from project key plus
  frozen roles, `import_id` from project key plus roles plus rows plus `source_head`,
  `created_at` as the maximum role commit time, and `operation_id` through the existing
  event canonicalization. Byte-identical output for byte-identical input, on any
  machine.
- `artifact_link_cutover_baseline_event(request) -> ArtifactLinkEventWire` — the
  canonical `baseline-import` event for that identity and row set.
- `artifact_link_cutover_marker(state, project_key, event_store, identity, roles, baseline) -> Marker`
  — replaces `build_artifact_link_cutover_marker_payload`.
- `artifact_link_cutover_attestation(&Marker) -> String` — the operator token, derived
  as `fleet-capable-<first 12 hex of sha256(canonical fenced-marker bytes)>`. It is a
  pure function of project key, frozen source heads, import identity, baseline digest,
  and the event-store version floor, so it changes the moment any of those move.
- `artifact_link_cutover_read_state(roots) -> ReadState` — reader policy. `none` when
  every root has no marker, `imported` when every root is `imported`, `fenced` when
  every root is `fenced`, otherwise `incomplete` with the offending roles listed. Any
  state other than `none` fences legacy writes; only `imported` retires legacy indexes
  as reducer input.
- `artifact_link_cutover_progress(observation) -> Progress` — the resumable state
  machine.

```rust
pub struct ArtifactLinkCutoverRootObservationWire {
    pub role: String,
    pub marker: Option<ArtifactLinkCutoverMarkerWire>,
    pub marker_committed: bool,
    pub baseline_durable: bool,
}
pub struct ArtifactLinkCutoverProgressWire {
    pub schema_version: u64,
    pub phase: String,               // fence | publish_baseline | mark_imported | complete
    pub roles_needing_fence_marker: Vec<String>,
    pub roles_needing_baseline_event: Vec<String>,
    pub roles_needing_imported_marker: Vec<String>,
    pub conflicts: Vec<String>,
}
```

Transition rules, in order:

1. Any present marker whose `import`, `baseline_event`, `roles`, `project_key`, or
   `event_store` differs from the expected import is a **conflict**. Conflicts are
   reported, never repaired, and no phase is proposed.
2. A root needs a fence marker when it has no marker, or has a marker whose bytes match
   but which is not committed.
3. Baseline publication is required until `baseline_durable` holds for **every** root.
4. No root may receive an `imported` marker until rule 3 is satisfied; a root then needs
   one when its marker is `fenced`, or is `imported` but uncommitted.
5. `complete` only when every root has a committed `imported` marker and the baseline is
   durable everywhere.

That ordering is what makes every seam resumable: a mixed marker set is a legal
intermediate state with a defined next step rather than an error.

- `artifact_link_outbox_classify_line(line: &str, project_key: &str) -> Classification`
  —
  `{ kind: "event" | "legacy_row" | "invalid", operation_id: Option<String>, diagnostic: Option<String> }`.
  A `legacy_row` is a `schema_version: 1` object with a valid row, matching
  `project_key`, numeric `created_at`, and non-empty `agent_name`; anything else that is
  not a valid v2 event entry is `invalid` with a reason.
- `artifact_link_outbox_legacy_conversion(entry, project_key, baseline_rows) -> Conversion`
  — `{ outcome: "convert" | "covered", event: Option<ArtifactLinkEventWire> }`. The
  converted event reuses today's deterministic
  `artifact_link_stable_operation_id("legacy-outbox-import", project_key, id, created_at, agent_name, run_id, row)`
  identity, so replay is exactly-once. `covered` is returned when the row's dedup
  identity already appears in `baseline_rows`, which is how the importer avoids double
  counting a use the frozen baseline already carries.

Also relax `canonicalize_baseline_rows` in `events.rs` to accept an empty row vector
(keeping duplicate-edge rejection) so an empty legacy project has a real cutover path.
This only widens acceptance inside event schema v1; it does not bump a wire version.

Expose all of the above through `crates/sase_core_py/src/lib.rs` as
`artifact_link_cutover_wire_schema_version`, `artifact_link_cutover_marker_parse`,
`artifact_link_cutover_marker_canonical_json`, `artifact_link_cutover_marker_build`,
`artifact_link_cutover_import_identity`, `artifact_link_cutover_baseline_event`,
`artifact_link_cutover_attestation`, `artifact_link_cutover_read_state`,
`artifact_link_cutover_progress`, `artifact_link_outbox_classify_line`, and
`artifact_link_outbox_legacy_conversion`. Add every name plus the new schema-version
constant to `tools/validate_sase_core_rs`. Do not hand-edit release-managed crate
versions.

### 4.2 Outbox records preserve every physical line

Rework `src/sase/sdd/_artifact_link_outbox_io.py` around a record, not an entry:

```python
@dataclass(frozen=True, slots=True)
class _ArtifactLinkOutboxRecord:
    line: str                                  # verbatim, including its original bytes
    kind: str                                  # event | legacy_row | invalid
    entry: _ArtifactLinkOutboxEntry | None
    diagnostic: str | None
```

`_read_outbox_records(path, project_key)` classifies every non-blank line through the
Rust classifier and keeps file order. `read_artifact_link_outbox_entries` keeps its
current public signature and still returns only v2 entries, so no existing caller
changes. `rewrite_artifact_link_outbox_without_ids` rewrites _from records_: every
`legacy_row` and `invalid` line is written back verbatim in its original position, and
only `event` records whose id is drained or dropped are removed. `_write_jsonl` keeps
its fsync-plus-`os.replace`-plus-directory-fsync ordering from `sase-yy.8.2`.

`inspect_artifact_link_outbox` gains `legacy_queued` and `invalid_queued` counts, and
`_ArtifactLinkOutboxDrainReport` in `_artifact_link_outbox_drain.py` gains
`retained_legacy` and `retained_invalid`. Leave `queued`, `drained`, `retained`, and
`dropped` counting v2 entries exactly as they do now, so existing drain assertions and
the emitted queue p95 age and link-only commit counts are untouched.

Surface the two new counts in `src/sase/artifact_cli/link_health.py` and
`src/sase/doctor/checks_artifact_links.py` so an operator can see unconverted legacy and
quarantined lines instead of discovering them by absence.

### 4.3 Deterministic, exactly-once legacy conversion

`convert_legacy_artifact_link_outbox_entries(project_key, *, baseline_rows=())` returns
a small report instead of an `int`:

```python
@dataclass(frozen=True, slots=True)
class _ArtifactLinkOutboxConversionReport:
    converted: int = 0
    covered: int = 0
    invalid: tuple[str, ...] = ()
```

It never raises on a malformed line: an `invalid` record stays in the file verbatim and
its diagnostic is collected. A `legacy_row` record goes through
`artifact_link_outbox_legacy_conversion`; `convert` replaces the line with the canonical
v2 entry, preserving `created_at`, `agent_name`, and `run_id` so the original run's
release eligibility (`_entry_is_eligible` in `_artifact_link_outbox_drain.py`) is
exactly what it was. `covered` retires the line to the existing dropped-entry audit
JSONL with `drop_reason="legacy_row_covered_by_baseline_import"` — recorded, never
silently discarded. Update the two callers (`artifact_link_outbox.py`'s facade re-export
and the importer) and `ArtifactLinkIndexImportReport.converted_outbox_entries`, adding
`covered_outbox_entries` and `outbox_conversion_diagnostics` alongside it.

### 4.4 Resumable import protocol

Split `src/sase/sdd/artifact_link_import_indexes.py` (740 lines) so no module lands near
the `toobig` 700 warning tier:

- `artifact_link_import_indexes.py` keeps only the public surface:
  `import_artifact_link_indexes`, `ArtifactLinkIndexImportReport`,
  `artifact_link_legacy_links_tree_identity`, and `__all__`.
- New `src/sase/sdd/_artifact_link_import_plan.py`: frozen role discovery, legacy row
  collection and conflict detection, `_links_tree_identity`, and the plan built from the
  Rust identity/baseline/marker contracts.
- New `src/sase/sdd/_artifact_link_import_apply.py`: the resumable apply loop, marker
  publication, and durability assertions.

The apply loop becomes observation-driven, with no new machine-local journal file — the
markers, the committed baseline objects, and git are the state:

1. Take the existing project publisher lock
   (`sase_projects_dir()/<key>/ARTIFACT_LINK_EVENT_LOCK_FILENAME`).
2. Build the plan. `_assert_clean_git_root` is relaxed to `_assert_resumable_git_root`:
   a sidecar may be dirty **only** at the expected cutover marker path or the expected
   baseline event object path, and only when the marker found there belongs to this same
   import identity. Any other dirt still refuses.
3. Observe every root: its marker (parsed through Rust, `None` when absent), whether
   that marker path is committed in `HEAD`, and
   `event_object_is_durable(root, baseline)` from
   `src/sase/sdd/_artifact_link_event_install.py`.
4. Call `artifact_link_cutover_progress`. Conflicts raise immediately with the reported
   reasons and write nothing.
5. Execute exactly the reported phase, then re-observe and loop until `complete` or no
   progress is made (a no-progress loop raises with the outstanding roles). Each phase:
   - `fence`: write and commit the fenced marker for the named roles only.
   - `publish_baseline`: convert the legacy queue (§4.3) once, then publish the baseline
     event through
     `publish_artifact_link_events(..., already_locked=True, extra_roots=plan.sidecar_roots)`,
     then `_assert_baseline_durable`.
   - `mark_imported`: write and commit the imported marker for the named roles only.
6. Rebuild the aggregate and return the report.

`_publish_marker` must commit a root whose marker bytes already match but whose path is
not in `HEAD` — the current `changed`-only commit is precisely the seam that strands an
uncommitted marker. Push failures stay the existing per-root publication ledger's job;
do not add a second retry lane.

Two machines importing the same frozen heads derive the same identity and therefore
write byte-identical markers, so their imports converge. Two machines whose sidecar
HEADs differ derive different `source_head` values, which rule 1 reports as a conflict
without touching what the other machine already wrote.

### 4.5 Reader policy for an incomplete cutover

`src/sase/sdd/_artifact_link_cutover_state.py` shrinks to a thin adapter: marker path
resolution, file read/write, git observation, and the Rust calls.
`_ArtifactLinkCutoverInspection.state` gains `"incomplete"`, and:

- `legacy_artifact_link_writes_fenced` returns `True` for `fenced`, `imported`, and
  `incomplete` — an incomplete cutover fences legacy writes.
- `artifact_link_indexes_imported` returns `True` only for `imported` and raises for
  `incomplete`, naming the roles and the exact resume command. That is how ordinary
  reads fail closed while the importer, which uses the non-raising inspection, does not.
- Keep `read_artifact_link_cutover_marker`, `artifact_link_cutover_marker_path`, and
  `role_for_artifact_link_kind` as they are; delete every `_parse_*` / `_require_*`
  helper the Rust parser replaces.

`src/sase/doctor/checks_artifact_links.py` and `src/sase/artifact_cli/link_health.py`
report `incomplete` as its own actionable state instead of the current
`invalid`/"markers are partial" error.

### 4.6 Post-import maintenance is event-only

In `_artifact_link_renames.py`, `_apply_artifact_renames` reads the cutover state once.
When imported it queues alias events, calls `store.rebuild_aggregate()` so the pending
alias is visible through the normal overlay, and skips `_rewrite_sidecar_indexes` and
`_rewrite_aggregate` entirely. `_ArtifactLinkRenameReport` gains
`legacy_indexes_frozen: bool`; `changed_indexes`, `removed_indexes`, and
`rewritten_rows` are then necessarily empty, so `consume_recent_artifact_renames`
produces no `links/**` commit. Pre-import behaviour is unchanged.

In `_artifact_link_store_reconcile.py`, `backfill_bead_endpoint_links` post-import stops
reading `_iter_sidecar_rows()` and stops calling `_upsert_bead`. It instead reprojects
bead endpoints from the reduced durable event union through the `sase-yy.8.3` path: give
`apply_events_to_beads` in `src/sase/sdd/_artifact_link_event_project.py` a
`force: bool = False` keyword so it can run with an empty `objects` batch, and call it
from the backfill. Keep the `{"candidates": ..., "written": ...}` return shape.
Pre-import behaviour is unchanged.

`_iter_reconciliation_legacy_sidecar_rows` already honours the fence; leave it alone.
`src/sase/bead/plan_archive_doctor.py` only _reads_ legacy indexes, which stays correct
for frozen history. Keep `_artifact_link_conflict_resolver.py` and
`_artifact_link_markdown_conflict_resolver.py` wired for old history and stragglers, and
keep the per-root ledger for old unpublished heads.

### 4.7 Operator capability attestation

Follow the CLI memory: preview stays the default, options stay optional, and the value
the mutation requires is a positional argument.

```
sase artifact link import-indexes [ATTESTATION]
  -a, --apply   Publish the cutover: fence markers, baseline event, imported markers
  -j, --json    Emit a machine-readable import report
```

- Bare and `--json` runs preview and write nothing. Preview prints the roles and their
  frozen heads, unique and duplicate row counts, the queued legacy and invalid outbox
  line counts, the machines whose capability the operator is attesting
  (`MachineService().list_machines()` aliases plus this machine; a best-effort note when
  the registry is unavailable), the honest warning that a marker cannot stop an old
  binary, the attestation token, and the exact copy-paste command to apply.
- `--apply` without `ATTESTATION`, or with one that does not match
  `artifact_link_cutover_attestation`, prints the preview plus a refusal naming the
  expected token and exits `1`. Nothing is written.
- `--apply ATTESTATION` with a match runs the resumable import.
- Because the token is a pure function of the fenced marker's canonical bytes, an
  attestation captured before a sidecar head moved no longer matches, so source-head
  fencing is enforced at the CLI boundary too.
- Keep help text sorted, give every long option its short alias, and update the
  `import-indexes` examples in `src/sase/main/parser_artifact_link.py`'s epilog and
  description.

## 5. Tests

Split by scenario; `tests` is under the same `toobig` limits as `src`. Reuse the
existing fixture styles: `_cluster` / `_init_local_repo` / `_write_legacy_index` in
`tests/sdd/test_artifact_link_event_acceptance.py`, and `allow_machine_sidecar_writes` /
`_row` in `tests/sdd/_artifact_link_store_helpers.py`.

`tests/sdd/test_artifact_link_outbox_legacy.py`:

1. The audit's blocker 2 verbatim: a valid v1 row plus an ineligible v2 entry, then
   `drain_artifact_link_outbox(drop_stale_terminal=False)`. Assert the physical file
   still has both lines with schemas `[1, 2]`, the v1 line is byte-identical, and the
   report shows `drained=0, dropped=0, retained_legacy=1`.
2. A malformed line survives an eligible drain that drains a real v2 neighbour, is
   reported through `inspect_artifact_link_outbox().invalid_queued`, and reaches
   `sase artifact doctor`'s link health output.
3. Conversion is deterministic and exactly-once: convert, assert the operation id, run
   the importer's conversion again and assert `converted == 0` with no new event.
4. Conversion preserves eligibility: a converted entry recorded by a run with release
   evidence drains; one recorded by a run without it stays queued.
5. A legacy row whose edge the frozen baseline already carries is reported as `covered`,
   lands in the dropped audit JSONL with the new reason, and does not raise `uses`.
6. Conversion does not raise on a malformed line; the import still completes and reports
   the diagnostic.

`tests/sdd/test_artifact_link_cutover_resume.py` (two roles: plans and research):

7. The audit's blocker 5 verbatim: fail the second `atomic_write_bytes`, then retry.
   Assert the retry completes, the final markers are byte-identical to a clean run's,
   and the import identity never changed.
8. Resume after a marker write succeeded but its commit failed: the second run commits
   the existing marker instead of skipping it.
9. Resume after the baseline event object is installed but not committed, after it is
   committed but not pushed (recovery is solely the existing per-root ledger), and after
   both roots are durable but only one has an `imported` marker.
10. A root carrying a marker with a different `source_head` is a conflict: the run
    raises, writes nothing, and leaves both roots' existing files untouched.
11. Simultaneous independent-machine import: two clones of the same frozen heads both
    apply and converge to byte-identical markers with one baseline operation id; a clone
    whose head has moved is refused as a conflict.
12. An empty valid project (sidecar roots present, zero `links/**` rows) imports: an
    empty `baseline-import` event is published and durable, both markers reach
    `imported`, and reads return no rows without error.
13. While a cutover is incomplete, `store.load_durable_rows()` raises a diagnostic
    naming the resume command, `store.upsert_row(...)` is fenced, and
    `sase artifact doctor` reports state `incomplete`.

`tests/sdd/test_artifact_link_frozen_maintenance.py`:

14. The audit's blocker 6 verbatim: import, `git mv` the document, then
    `consume_recent_artifact_renames`. Assert `changed_indexes` and `removed_indexes`
    are empty, `legacy_indexes_frozen` is `True`, every `links/**` file is
    byte-identical to its pre-rename bytes, the alias event is queued, and the alias is
    visible through the pending overlay.
15. `repair_historical_artifact_renames` post-import likewise leaves `links/**`
    byte-identical.
16. `backfill_bead_endpoint_links` post-import writes no `links/**` bytes, calls no
    `_upsert_bead`, and produces bead endpoints equal to the Rust-reduced event truth; a
    second run is a no-op.
17. `reconcile_and_repair_artifact_links` post-import commits nothing into `links/**`.
18. A post-import straggler (a hand-written `links/**` change) is still reported loudly
    by `sase artifact doctor` and by `link_health`, and the conservative legacy conflict
    resolver still merges it.

`tests/main/test_artifact_cli_link_import.py`:

19. Bare preview writes nothing, exits `0`, and prints the roles, counts, fleet machine
    list, old-binary warning, attestation token, and the apply command.
20. `--apply` with no token and `--apply` with a stale token both exit `1`, write
    nothing, and name the expected token.
21. `--apply <token>` applies, and re-running it is the `already_imported` no-op.
22. `-j/--json` preview and applied reports carry the new fields; the parser exposes the
    positional and both short aliases.

Update `tests/sdd/test_artifact_link_import_indexes.py` to the new plan/apply split and
report fields rather than working around them, and update any
`convert_legacy_artifact_link_outbox_entries` caller assertions in
`tests/main/test_artifact_link_outbox.py`.

Rust: unit tests in `cutover.rs` for marker parse/canonicalization round-trips, identity
determinism across role orderings, attestation stability and sensitivity to a moved
head, every `artifact_link_cutover_progress` transition including each conflict class,
and `artifact_link_cutover_read_state`'s four states; plus an `events.rs` test that an
empty `baseline-import` canonicalizes and reduces to no rows. Add `sase_core_py` binding
tests for each new function.

## 6. Verification

1. `just install` — this workspace's virtualenv may have drifted, and the Rust change
   must be rebuilt into it from the linked `sase-core` checkout.
2. In the `sase-core` checkout opened with `/sase_repo`: `just check` (or
   `./scripts/check.sh`), which runs `fmt-check`, `clippy`, and the full workspace suite
   including the `sase_core_py` binding tests. Never verify with
   `cargo test -p sase_core` alone. Commit and push `sase-core` before ratcheting the
   pin below.
3. `.venv/bin/python tools/validate_sase_core_rs` must exit 0, and
   `.venv/bin/python tools/check_sase_core_rs_bindings` must be clean.
4. Ratchet the pinned revision with `tools/ratchet_core_revision` — do not hand-edit
   `sase-core-revision.txt`, and do not hand-edit release versions or the
   `sase-core-rs>=0.33.0,<0.34.0` floor. Note that the pin is stale on `master` today:
   it still names `da0a738` while `sase-yy.8.3` shipped Python that calls
   `set_link_projection`, which first exists at `717c36e`. This phase's ratchet must
   move past both that commit and this phase's.
5. Focused `pytest`: `tests/sdd/test_artifact_link_*`,
   `tests/main/test_artifact_cli_link*`, `tests/main/test_artifact_link_outbox.py`,
   `tests/test_validate_sase_core_rs_tool.py`,
   `tests/test_check_sase_core_rs_bindings_tool.py`.
6. `just check` for the repo-wide gates. One failure is known and pre-existing on
   `master`: `tools/check_feature_flags` rule 8 reports that live flag bead `sase-z0`
   has no `link_events` registry definition. Confirm it is the only failure and do
   **not** fix it here — closing `sase-z0` belongs to `sase-yy.8.5`. Re-record it with
   `sase bead note sase-yy.8.4 'PROPOSED FOLLOW-UP: ...'` if it is still present.
7. Run `just check-full` through `/sase_monitor` with the `TESTING`/`TESTED` status pair
   only if `just check`'s scoped selection escalates or reports an unusual selection.
   The epic's combined-tree `check-full` is a landing action, not this phase's.
8. Inspect both repository diffs and working trees for unrelated changes.
9. Run `sase bead epic-symbols sase-yy.8.4`, resolve or re-key every remaining entry,
   then close **only** `sase-yy.8.4` with a note naming the verified queue-preservation,
   resume, attestation, and frozen-history guarantees. Leave `sase-yy.8`, `sase-yy`, and
   every other ancestor open.

## 7. Constraints

- `sase-yy.8.5` (acceptance) is in progress in parallel and owns
  `tests/sdd/test_artifact_link_event_acceptance.py`, the `sase-z0` flag closure, and
  the end-to-end process-death suite. Do not edit that acceptance module or close that
  flag bead here; add this phase's coverage in the new sibling test files above.
- Do not re-repair the closed phases' defects: producer identity (`sase-yy.8.1`),
  ownership receipts and atomic installation (`sase-yy.8.2`), or event-union reduction
  and bead projection (`sase-yy.8.3`). Reuse their contracts.
- Preserve the landed module splits and public facades, the sidecar eviction protection,
  the clone fallback, and the per-root publication ledger with its deadline, fairness,
  aging, unpublished-head discovery, and release-evidence behavior. Never add a second
  push-retry ledger.
- Never overwrite or delete a published event object, and never rewrite a `links/**`
  file once its root is fenced or imported. Legacy indexes stay on disk as read-only
  history; this phase does not delete them.
- Shared identity, ownership, reduction, migration, transition, and baseline policy
  belongs in Rust `sase-core`; Python keeps filesystem, git, locking, process glue, and
  thin binding adapters.
- Follow this package's symbol convention so `symvision` stays green: new
  `sase.sdd._artifact_link_*` modules are private _modules_ defining _public_ symbols
  with `__all__`, and importers alias them locally (`... as _name`) the way
  `_artifact_link_event_publish.py` already imports from
  `_artifact_link_event_canonical.py`. Never import a `_`-prefixed symbol across files.
- Keep every touched module comfortably under the `toobig` 700-line warning tier.
- This phase adds and tests the product mechanism only. Do not run a production import,
  do not mutate any real project's `links/**`, and do not upgrade any machine.
- Record discovered out-of-scope work as `PROPOSED FOLLOW-UP:` notes on `sase-yy.8.4`
  with `sase bead note`. Do not create beads, and do not close any ancestor bead.
