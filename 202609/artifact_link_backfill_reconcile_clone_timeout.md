---
tier: tale
title: Stop artifact_link_backfill from timing out on cross-clone bead reconciliation
goal:
  Make housekeeping artifact_link_backfill finish under its 240s budget by stopping
  machine-lane reconcile from walking every ephemeral workspace clone's bead store.
size: medium
proposed_by: bbugyi200.athena.0nr
create_time: 2026-09-19 10:17:45
status: wip
---

# Plan: Stop `artifact_link_backfill` from timing out on cross-clone bead reconciliation

## 1. The symptom

The `housekeeping` lumberjack's `artifact_link_backfill` chop is again dying at Axe's
hard timeout:

```
Job:        artifact_link_backfill
Error:      timed out after 300s
Exit Code:  -9
```

On 2026-09-19 this host's run history is mixed success-at-the-cliff and SIGKILL:

| run                    | duration | outcome                              |
| ---------------------- | -------- | ------------------------------------ |
| 20260919T005144_136732 | 286s     | success                              |
| 20260919T015631_092761 | 300s     | timeout                              |
| 20260919T030132_488899 | 228s     | success                              |
| 20260919T040521_950504 | 277s     | success                              |
| 20260919T050959_647798 | 241s     | success                              |
| 20260919T061042_356055 | 244s     | success                              |
| 20260919T071043_566708 | 3722s    | stale running process                |
| 20260919T081246_437622 | 300s     | timeout                              |
| 20260919T090630_543072 | 300s     | timeout (digest_20260919_094259.txt) |
| 20260919T094258_849096 | 301s     | timeout                              |

Successful logs are self-diagnosing. `gh_bbugyi200__actstat` finishes in ~5s.
`gh_bobs-org__bob-cli` finishes in ~17–55s. Then `gh_sase-org__sase` spends almost the
entire remaining budget in one job:

```
gh_sase-org__sase: done in 219.22s (..., reconcile_repair=213.81s)
gh_sase-org__sase: done in 262.56s (..., reconcile_repair=255.38s)
```

Timeout logs stop immediately after `gh_sase-org__sase: reconcile` — Axe SIGKILLs the
in-flight call. The chop's 240s `_CHOP_WORK_BUDGET_SECONDS` cannot help: that deadline
is checked before starting reconcile, not inside it, and reconcile starts around
T+20–60s with ~180–220s of remaining budget that the call then overruns.

This is **not** the bead-projection timeout tracked by `sase-12y`
(`plan:202609/artifact_link_projection_timeout.md`). That epic's phases 1–3 already
landed: today's logs get through `publication_retry`, `store_resolution`, `sweep`, and
`drain` and die in `reconcile`. Remaining `sase-12y.4` work (ratchet the published
`sase-core-rs` floor) does not touch this path. A `DISCOVERED ISSUE` note was appended
to `sase-12y` with this evidence.

Do **not** raise the 5-minute Axe timeout or the 240s chop budget. That constraint was
set by the earlier timeout work and still holds.

## 2. Root cause

`reconcile_and_repair_artifact_links` (`src/sase/sdd/artifact_link_backfill.py`) calls
`store.reconcile_aggregate()` with **no deadline**. That writes
`preview_reconciled_aggregate()` (`src/sase/sdd/_artifact_link_store_reconcile.py`).

`preview_reconciled_aggregate` still treats every registered primary-checkout clone as
an independent source of sidecar and bead-link truth:

1. `_iter_reconciliation_stores` yields the machine store, then every
   `collect_repo_inventory` clone of the project's `kind == "primary"` record. On this
   host that is **33 stores**: the hidden machine lane plus the human checkout plus
   every ephemeral `sase_<N>` workspace clone.
2. `_reconciliation_event_snapshot` loads each clone's event store (~18s total; not the
   cliff).
3. For **each** of those 33 stores, `_iter_reconciliation_compatibility_rows` yields
   legacy sidecar rows (if that clone is not imported) plus every bead-endpoint row
   `event_snapshot.covers_row` does not already cover.

The bead half is the cost. `_iter_bead_rows` → `_list_bead_issues` opens that clone's
beads sidecar with `open_bead_project_for_beads_dir` and calls `project.list_issues()`
(`src/sase/sdd/_artifact_link_store_bead_rows.py`). Measured on this host against the
live `gh_sase-org__sase` machine store (read-only):

```
n_stores=33
reconciliation_event_snapshot=18.14s  rows=2565
compatibility_total=230.88s           extra_rows=21861  collected=24426
  per clone: 3.3–19.3s, typically ~764 extra bead rows
```

Those extra rows are almost entirely duplicates of the same uncovered bead-endpoint
facts. `unique_rows` / `project_aggregate_rows` would collapse them — after paying 231s
to collect them. 231s + earlier projects (~20–60s) is exactly the 300s SIGKILL.

This walk is leftover from the pre-hidden-clone world, where each workspace clone's
`sase/repos/{plans,research,beads}` tree could hold unpublished local indexes. Machine
artifact-link writes now live in the host-owned hidden clones under
`~/.sase/projects/<key>/repos/<role>` (the store `resolve_machine_artifact_link_store`
already returns). Ephemeral workspace clones are pull-only copies. Scanning them:

- does not add durable truth the hidden beads clone and the primary checkout do not
  already hold;
- multiplies `list_issues()` by clone count, which grows with normal agent use;
- can surface stale or partial cutover markers. Successful ticks that finish the 231s
  walk then raise from `artifact_link_indexes_imported` during
  `_sidecar_truth_was_consulted` / `_apply_artifact_renames`:
  `artifact-link cutover is incomplete; resume with sase artifact link import-indexes --apply fleet-capable-e2ce6f584527`.
  The hidden machine clones and the primary checkout on this host are already
  `imported`; the raise comes from a sibling clone after the expensive work is done.

The 240s chop budget and the rename-repair deadline (sase-u9) are both downstream of
this unbounded `reconcile_aggregate()` call.

## 3. Why this is safe

Machine-lane reconcile must keep:

- the current store (`self`) — on the housekeeping chop this is the hidden
  plans/research/beads clones;
- the project's primary checkout clone (`RepoCloneRecord.workspace_num` 0, or legacy 1).

It must **not** keep ephemeral workspace clones (`workspace_num > 1`). Those checkouts
are not machine-writable, are not the durable event lane, and their bead sidecars are
pull-only copies of the same remotes.

Merge semantics across _distinct_ stores stay: the existing test
`test_reconcile_aggregate_collects_sidecar_rows_from_known_workspace_stores`
monkeypatches `_iter_reconciliation_stores` and still proves two stores' sidecar rows
union. This plan changes which stores the inventory iterator yields, not the merge
function.

Do not apply `sase artifact link import-indexes --apply` as part of this tale.
Completing cutover on straggler clones is operational fleet work; the code fix is to
stop visiting those clones and to fail-soft if one is consulted anyway. Do not weaken
the write-path fence in `artifact_link_indexes_imported` (it must still raise for
incomplete cutover on a mutation).

## 4. Changes

Stay in this repo's Python store / chop code. Shared reconcile behavior already lives in
`ArtifactLinkStoreReconcileMixin`; do not move it into `sase-core` for this timeout fix.

### 4.1 `src/sase/sdd/_artifact_link_store_reconcile.py` — stop walking ephemeral clones

In `_iter_reconciliation_stores`:

1. Keep yielding `self` first (current identity / hidden machine lane).
2. When iterating `collect_repo_inventory` clones of the matching `kind == "primary"`
   record, skip any `RepoCloneRecord` whose `workspace_num` is greater than
   `LEGACY_PRIMARY_WORKSPACE_NUM` (1). Import that constant from
   `sase.workspace_provider.store` (or the inventory models, if that is the existing
   seam). Primary is 0; some hosts still register it as 1. Ephemeral `sase_<N>` clones
   are `workspace_num >= 2`.
3. Keep the existing `remember()` identity filter, best-effort `except` around missing
   clones, and `from_sdd_store` construction.

Add a short comment that machine-lane aggregate reconcile reads the hidden clones plus
the human primary checkout, not every ephemeral workspace copy of the same remotes.

### 4.2 Same file — deadline and fail-soft

Thread an optional `deadline: float | None = None` (`time.monotonic()` timestamp)
through `preview_reconciled_aggregate` and `reconcile_aggregate`. Interactive / doctor
callers leave it unset and finish. The chop passes its existing `chop_deadline`.

- Check the deadline between stores in `_reconciliation_event_snapshot` and in the
  compatibility-row loop. On expiry, stop starting more stores, record one skip
  diagnostic naming how many stores were left, and reconcile from what was already
  collected (the machine store is first, so the write is still grounded). Do not raise.
- In `_authoritative_source_was_consulted_for_pass` / `_sidecar_truth_was_consulted`, do
  **not** let a sibling clone's incomplete cutover abort the pass. Use
  `inspect_artifact_link_cutover_markers` (or catch the `RuntimeError` from
  `artifact_link_indexes_imported`) so a sibling with `state in {"incomplete", "none"}`
  is treated as "this store cannot prove sidecar truth" rather than failing the whole
  preview after minutes of work. Leave `artifact_link_indexes_imported`'s raise in place
  for write paths.

`durable_sidecar_rows` uses the same store iterator; it automatically stops amplifying
by clone count. Do not change its publishability filter.

### 4.3 `src/sase/sdd/artifact_link_backfill.py` — forward the deadline

`reconcile_and_repair_artifact_links` already accepts `deadline` and forwards it only to
`repair_historical_artifact_renames`. Pass it into
`store.reconcile_aggregate(deadline=...)` as well. The chop already supplies
`chop_deadline` here (`src/sase/scripts/sase_chop_artifact_link_backfill.py`); do not
change the 240s / 5m numbers.

If `reconcile_aggregate` / preview reports skip diagnostics (deadline or incomplete
sibling cutover), include them on `_ArtifactLinkReconcileReport.skip_diagnostics` so the
chop's existing warning logger prints them. Do not turn those skips into a failed chop
result.

## 5. Tests

Add to `tests/sdd/test_artifact_link_store_reconcile.py` (use the existing `_store` /
inventory fakes; do not touch a real workspace tree):

- **Inventory iterator keeps self + primary only.** Stub `collect_repo_inventory` with a
  primary `RepoRecord` whose `clones` include `workspace_num` 0 (or 1) and several
  `workspace_num >= 2` paths. Stub `resolve_sdd_store` /
  `ArtifactLinkStore.from_sdd_store` to return distinct stores keyed by path. Assert
  `_iter_reconciliation_stores` yields `self` and the primary clone store, and never
  constructs a store for the ephemeral clones.
- **Compatibility rows are not multiplied by ephemeral clones.** With the same stub,
  `preview_reconciled_aggregate` (or a monkeypatched `_iter_bead_rows` counter) must
  visit bead/legacy compatibility on `self` and primary only. A regression is "N
  ephemeral clones ⇒ N `list_issues` / `_iter_bead_rows` calls".
- **Deadline defers remaining stores.** Fake `time.monotonic` so the deadline expires
  after the first store's snapshot/compatibility work; assert later stores are not
  visited, a skip diagnostic is present, and the call returns rather than raising.
- **Sibling incomplete cutover does not raise.** One stub store's
  `artifact_link_indexes_imported` / cutover inspection reports `incomplete`; preview
  still returns rows from the healthy stores.

Add / extend in `tests/sdd/test_artifact_link_reconcile.py`:

- `reconcile_and_repair_artifact_links(..., deadline=123.0)` calls `reconcile_aggregate`
  with that deadline (today it calls it with no kwargs). Keep the existing rename-repair
  deadline assertion.

Chop tests in `tests/test_axe_chop_artifact_link_backfill.py` already prove the 240s
budget stops _starting_ later projects. Do not change those numbers. No new chop-level
timeout constant.

Do not reach the network or a live beads sidecar from any test.

## 6. Verification

1. `just install` if this workspace venv is stale, then `just fix` / `just fmt`, then
   `just check`. Consult `sase/memory/lint_and_test.md` after tracked edits.
2. Re-run the read-only measurement against the live `gh_sase-org__sase` machine store:
   `_iter_reconciliation_stores` must drop from 33 to 2 (hidden machine lane + primary
   checkout), and `_iter_reconciliation_compatibility_rows` over those stores must land
   around the previously measured ~10s (6.8s + 3.6s) rather than 231s. Do not write the
   aggregate during this measurement; call preview / the iterators only.
3. Force one real chop run after the change is installed:
   `sase axe job run artifact_link_backfill` (or the equivalent `sase axe chop run`
   housekeeping entry). The newest run under
   `~/.sase/axe/lumberjacks/housekeeping/chops/artifact_link_backfill/runs/` must have
   `status == "success"`, `duration_ms` well under 240000, a non-empty log, and a
   `gh_sase-org__sase: done in ... (reconcile_repair=...)` line whose reconcile_repair
   is tens of seconds, not 200+.
4. `just check-full` through `/sase_monitor` (`TESTING` / `TESTED`) before landing,
   because this mixin is also on `sase artifact doctor`'s path.

## 7. Out of scope

File as follow-up notes on this plan's bead if they still matter after the fix; do not
widen the tale.

- Applying `sase artifact link import-indexes --apply fleet-capable-e2ce6f584527`. The
  hidden machine clones are already imported. Straggler ephemeral clones should simply
  stop being consulted.
- `gh_bobs-org__bob-cli`'s
  `operation_id ... was reused for different artifact link events` (event-store
  validation). That fails in <1s and is a distinct data defect.
- The derivation sweep's unbounded `rglob("*.md")` of already-swept sidecar trees
  (`gh_bobs-org__bob-cli` sweep 11–51s for 2 remaining documents). Annoying, but it fits
  in budget once sase reconcile is ~10–20s.
- `sase-12y.4` core-wheel-floor ratchet.
- Raising `_CHOP_WORK_BUDGET_SECONDS` or the axe `timeout: "5m"`.
