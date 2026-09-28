---
tier: tale
title: Fix recurring artifact_link_backfill 300s timeout
goal:
  The housekeeping artifact_link_backfill chop finishes the sase project's
  reconcile/repair well inside its 300s axe timeout, because artifact-link event
  coverage checks are no longer quadratic and the chop's dangling-ref scan only resolves
  refs the rename repair can use.
size: medium
proposed_by: bbugyi200.athena.0tn
create_time: 2026-09-28 12:46:49
status: wip
---

# Plan: Fix the recurring `artifact_link_backfill` 300s SIGKILL (quadratic `covers_row`)

## Symptom

The housekeeping chop `artifact_link_backfill` is repeatedly killed by axe
(`timed out after 300s`, exit -9), about once or twice a day since at least 2026-09-07
(35 error digests mention it). Every killed run has the same shape:

```
gh_sase-org__sase: store_resolution
gh_sase-org__sase: drain
gh_sase-org__sase: reconcile          <- last line; SIGKILL ~250s later
```

In those runs the sase project skips `sweep`, because `gh_bobs-org__bob-cli`'s sweep
(30–50s) already used up the 45s sweep budget. So sase goes straight to drain +
reconcile/repair, and reconcile never returns. In every _successful_ run, sase does get
to sweep. That sweep overruns its 45s budget to 100–170s, and the chop then logs
`deferred outbox/reconcile after sweep budget`. **The sase project's reconcile/repair
has never completed in any retained run.**

## Root cause (measured)

Measured on the live `gh_sase-org__sase` data with read-only profiling. The hidden
machine store was resolved without fresh integration, and only the non-writing
`preview_*` methods were called.

1. **Primary: `ArtifactLinkEventSnapshot.covers_row` is O(edges) per call.**
   `src/sase/sdd/_artifact_link_event_store.py` `covers_row()` calls
   `_event_edge_identities(self.edges)` on _every_ call. That rebuilds a frozenset of
   ~6.2k edge identities each time. The callers invoke it once per row:
   - `predicate` in `_artifact_link_store_core.py` (per prior aggregate row, 7,260
     rows),
   - `_iter_reconciliation_compatibility_rows` /
     `_iter_reconciliation_legacy_sidecar_rows` in `_artifact_link_store_reconcile.py`,
   - `_rows_not_covered_by_events` / `_rows_not_removed_by_events` in
     `_artifact_link_store_rows.py`.

   The result is quadratic work: 14,746 calls → 91.7M `_edge_identity` calls. Real
   wall-clock on the live data:

   | method                                 | today  | with identities cached once per snapshot |
   | -------------------------------------- | ------ | ---------------------------------------- |
   | `store.preview_aggregate()`            | 97.2s  | 18.9s                                    |
   | `store.preview_reconciled_aggregate()` | 133.9s | 20.2s                                    |

   Under cProfile, ~90% of both is `covers_row` → `_event_edge_identities`. With a
   cached identity set, the output rows were identical, apart from rows written live
   between the two runs.

   `reconcile_aggregate()` wraps the preview in `_write_merged_aggregate`. That
   recomputes the whole preview on every generation race, up to
   `_MAX_AGGREGATE_WRITE_ATTEMPTS = 5` times, and sase agents write the aggregate
   constantly. This one step alone can run past the remaining ~250s. Every
   `rebuild_aggregate()` pays the same cost, and that includes the sweep's persist →
   outbox drain path. That is why the sase sweep overruns its budget to 100–170s.
   Interactive `sase artifact` paths that rebuild the aggregate are slow for the same
   reason.

   The chop's `deadline` cannot bound this. `preview_reconciled_aggregate` checks the
   deadline only between stores, and all the time is spent in `project_aggregate_rows`
   over the prior rows, which cannot safely stop partway (a partial projection would
   drop rows).

2. **Secondary: the reconcile's dangling-ref scan resolves every ref, with no
   deadline.** `reconcile_and_repair_artifact_links()`
   (`src/sase/sdd/artifact_link_backfill.py`) calls
   `dangling_and_orphaned_artifact_link_refs(store)`
   (`src/sase/artifact_cli/link_health.py`). That resolves every unique non-bead ref in
   the store-backed rows: ~2k agent, ~1.3k plan, ~0.5k research and 180 file refs, plus
   it loads all ~6.5k bead ids. Each id-based `file:<source>:<digest>` ref costs ~1.2s,
   because `_find_file_record` (`src/sase/artifact_cli/references.py`) reloads and
   converts the entire ~24k-row artifact-file index per call. Measured estimate: ~150s
   for this scan. The only consumer, `repair_historical_artifact_renames`
   (`src/sase/sdd/_artifact_link_renames.py`), ignores every ref whose kind is not
   `plan`/`research`. So the chop resolves ~21k refs it then throws away. The
   plan/research-only portion costs ~2s, plus ~4s for `orphaned_link_indexes`.

Together, items 1 and 2 account for the observed kill: ≥134s reconcile (more with write
retries) plus ~150s dangling scan, against ~250s of budget left.

## Changes

### 1. Cache edge identities once per event snapshot (the root-cause fix)

In `src/sase/sdd/_artifact_link_event_store.py`:

- `ArtifactLinkEventSnapshot` is `@dataclass(frozen=True, slots=True)`. Add a private
  lazily-filled field, for example
  `_edge_identity_cache: frozenset[tuple[str, ...]] | None = field(default=None, init=False, repr=False, compare=False)`.
  Add a property or method that computes `_event_edge_identities(self.edges)` on first
  use and stores it with `object.__setattr__(self, "_edge_identity_cache", ...)`. Use
  `field(..., compare=False)` so equality and `repr` are unchanged.
  `dataclasses.replace()` gives a fresh instance with `None`, so the cache can never go
  stale for replaced `edges`.
- `covers_row()` uses the cached set instead of calling `_event_edge_identities` per
  call. Keep its semantics exactly as they are: the `has_inputs` /
  `blocking_problem_messages` early return, the direct identity check, then the
  alias-resolved identity check.
- Do **not** change `_edge_identity` or `_event_edge_identities` behavior.

This is a pure performance fix to existing Python adapter logic. It adds no new domain
behavior, so it stays in this repo. Moving event-coverage logic into `sase-core` is out
of scope.

### 2. Scope the chop's dangling-ref scan to kinds the rename repair can fix

- In `src/sase/sdd/_artifact_link_renames.py`, add a module constant
  `REPAIRABLE_RENAME_KINDS = frozenset({"plan", "research"})`. Use it in
  `repair_historical_artifact_renames` in place of the inline `{"plan", "research"}`
  literal. Add it to that module's `__all__` if the module declares one.
- In `src/sase/artifact_cli/_link_health_refs.py`, give `dangling_refs()` an optional
  keyword `kinds: Collection[str] | None = None`. When it is set:
  - skip, before resolution, any ref whose `kind_of_ref(ref)` is not in `kinds`;
  - do not call `known_bead_ids(store)` unless `"bead"` is in `kinds` (it costs ~1.1s
    for sase).

  `kinds=None` keeps today's behavior exactly: the interactive
  `inspect_artifact_link_health` / `sase artifact doctor` report every dangling ref.

- In `src/sase/artifact_cli/link_health.py`, give
  `dangling_and_orphaned_artifact_link_refs(store, *, kinds=None)` the same keyword.
  Forward it to `dangling_refs`, and when set also filter `orphaned_link_indexes(store)`
  results by kind. Update its docstring: with `kinds` it returns only candidates of
  those kinds.
- In `reconcile_and_repair_artifact_links()` (`src/sase/sdd/artifact_link_backfill.py`),
  call `dangling_and_orphaned_artifact_link_refs(store, kinds=REPAIRABLE_RENAME_KINDS)`.

### 3. Deadline guard and sub-stage progress in `reconcile_and_repair_artifact_links`

These are defense in depth, so that a future slowdown degrades into a logged deferral
instead of a silent SIGKILL (the chop's stated design intent):

- After `store.reconcile_aggregate(deadline=deadline)` returns, if `deadline` is set and
  `time.monotonic() >= deadline`, skip the dangling scan, rename repair and commit.
  Return the report with the reconcile's skip diagnostics plus one new diagnostic, for
  example `"reconcile deadline expired; deferred dangling-ref repair"`. The chop already
  turns `skip_diagnostics` into warnings. Interactive callers pass no deadline and are
  unaffected.
- Add an optional keyword `progress: Callable[[str], None] | None = None` to
  `reconcile_and_repair_artifact_links`. Call it with `"aggregate"`, `"dangling_scan"`,
  `"repair"` and, only when there are changed paths, `"commit"`, before each sub-step.
- In `src/sase/scripts/sase_chop_artifact_link_backfill.py` `_run_project`, pass
  `progress=lambda stage: _log_project_stage(runtime, project_key, f"reconcile/{stage}")`.
  The next time something stalls, the axe error digest (which captures only log lines)
  will name the sub-step instead of only `reconcile`.

Non-goals, to keep this change focused:

- no changes to `_write_merged_aggregate`'s retry or force semantics;
- no partial or deadline-interrupted `project_aggregate_rows`;
- no change to sweep budgets or the 300s axe timeout in `src/sase/default_config.yml`.

With change 1, the existing budgets hold: expected sase ≈ 20s rebuild per sweep chunk
and ≈ 20s reconcile plus ≈ 6s scoped dangling scan.

### 4. Tests

Add or extend tests next to the existing ones (follow their fixture style):

- `tests/sdd/test_artifact_link_event_store.py`: build an `ArtifactLinkEventSnapshot`
  with inputs (for example `durable_event_count=1`) and a few directed and undirected
  `edges`. Monkeypatch `sase.sdd._artifact_link_event_store._event_edge_identities` with
  a counting wrapper. Call `covers_row` on several covered and uncovered rows, and
  assert the wrapper ran exactly once and the true/false results are unchanged. Also
  assert that snapshots with a filled cache still compare equal to fresh ones
  (`compare=False`).
- `dangling_refs` kinds filter (a new test module under `tests/` next to the link-health
  tests, or an existing link-health test module): rows mixing `plan:`, `agent:`, `file:`
  and `bead:` refs, with a recording fake `resolve_reference`. With
  `kinds={"plan", "research"}`, only plan/research refs are resolved and
  `known_bead_ids` is not called (monkeypatch it to fail if called). With `kinds=None`,
  all non-bead refs are resolved, as today.
- `tests/sdd/test_artifact_link_reconcile.py`:
  - assert `reconcile_and_repair_artifact_links` passes `kinds=REPAIRABLE_RENAME_KINDS`
    to `dangling_and_orphaned_artifact_link_refs`;
  - when the deadline has already expired after `reconcile_aggregate`, the dangling scan
    and repair are not called and the new skip diagnostic is present;
  - `progress` receives the stages in order.

  The existing fakes use `lambda _store: ...` for
  `dangling_and_orphaned_artifact_link_refs`. Change them to accept `**_kwargs`, and do
  the same in `tests/sdd/test_artifact_link_machine_authorize.py`.

- `tests/test_axe_chop_artifact_link_backfill_budgets.py`: extend
  `test_per_project_progress_is_logged` (or add a sibling test) so the fake
  `reconcile_and_repair_artifact_links` invokes `progress("aggregate")`. Assert that
  `proj: reconcile/aggregate` is logged.

### 5. Follow-ups to file (do not fix here)

These are real but separate from the timeout. File each through `/sase_new_task` (dedupe
against existing beads first):

- `gh_bobs-org__bob-cli` artifact-link event store is invalid:
  `operation_id de29d2e25c1cfb4381f223c44d576f8c was reused for different artifact link events`.
  This fails bob-cli's publication, drain and reconcile every tick.
- In `gh_sase-org__sase`, sweep persistence fails with
  `artifact-link bead event publication failed: 'Issue not found: sase-164'`. Because
  `_swept_refs_after_chunk` marks nothing swept when a chunk has errors, the same ~50
  documents are retried every tick, and `sweep_remaining` only grows (256 → 266 over
  2026-09-28).
- `resolve_cli_reference` → `_find_file_record` does a full artifact-file index load and
  conversion (~1.2s) per `file:<source>:<digest>` ref. That makes the interactive
  `sase artifact doctor` spend ~150s on the sase project. It needs an indexed or
  memoized by-id lookup.

## Verification

- Run `sase tool run check` (the repo's standard gate: lint + mypy + symvision + scoped
  tests) and make it pass.
- Optional live sanity check (read-only): in a Python session against the installed
  sase, resolve the `gh_sase-org__sase` store with
  `sase.sdd._artifact_link_machine_store.resolve_machine_artifact_link_store`, and time
  `store.preview_aggregate()` and `store.preview_reconciled_aggregate(deadline=...)`.
  Both should take roughly 20s, down from ~97s / ~134s.
- After landing, the next scheduled `artifact_link_backfill` runs should log
  `gh_sase-org__sase: reconcile/...` stages and `gh_sase-org__sase: done in ...` with a
  `reconcile_repair=` timing, and should produce no further 300s timeouts.
