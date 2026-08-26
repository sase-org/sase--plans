---
tier: tale
title:
  Stop the artifact_link_backfill chop from timing out by hoisting the artifact-ref
  context out of the reconcile row loop
goal:
  The hourly artifact_link_backfill housekeeping chop finishes well inside its 5m
  budget, its cost stops scaling with the number of workspace clones, and a future
  slowdown degrades into a logged deferral instead of a silent SIGKILL.
size: medium
proposed_by: bbugyi200.athena.0ec
---

- **AGENTS:**
  - [bbugyi200.athena.0ec](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ec.md)
- **COMMITS:**
  - [cc2d907](https://github.com/sase-org/sase/commit/cc2d90721baf4188eb6e19b116fd594f89ea9bc4)
    — fix(sdd): bound artifact link backfill reconciliation

# Plan: Stop the `artifact_link_backfill` chop from timing out

## 1. The symptom

The `housekeeping` lumberjack's `artifact_link_backfill` chop times out every hour:

```
Lumberjack: housekeeping
Job:        artifact_link_backfill
Error:      timed out after 300s
Traceback:  <no python traceback: subprocess error>
```

Four such errors are in `~/.sase/axe/recent_errors.json` for 2026-08-26 alone (06:42,
07:15, 08:15, 10:01). It is the only recurring `housekeeping` failure. Every timed-out
run's output log is **0 bytes**, so the failure carries no diagnostic beyond the timeout
string.

Run history at `~/.sase/axe/lumberjacks/housekeeping/chops/artifact_link_backfill/runs/`
shows the chop sitting right on the edge and crossing it:

| run                 | source    | duration        | outcome                                                           |
| ------------------- | --------- | --------------- | ----------------------------------------------------------------- |
| 2026-08-26T01:01:22 | oneshot   | 105s            | success (`sweep_scanned=0`)                                       |
| 2026-08-26T08:10:50 | scheduled | 300s            | **timeout**                                                       |
| 2026-08-26T08:46:18 | scheduled | ~640s to result | orphaned by a daemon restart, finished anyway (`sweep_scanned=6`) |
| 2026-08-26T09:56:17 | scheduled | 300s            | **timeout**                                                       |

The successful runs prove the sweep is not the cost: `sweep_scanned` was 0 and 6. The
cost is entirely in the chop's other three jobs.

## 2. Root cause

Measured on this host with the real store (all numbers reproduced below in §3).

`sase_chop_artifact_link_backfill._run_project` calls
`reconcile_and_repair_artifact_links`, whose first step is `store.reconcile_aggregate()`
→ `ArtifactLinkStoreReconcileMixin.preview_reconciled_aggregate`
(`src/sase/sdd/_artifact_link_store_reconcile.py:50`):

```python
context = self._artifact_ref_context_for_store()          # line 53
collected: list[dict[str, Any]] = []
for store in self._iter_reconciliation_stores():          # every workspace clone
    collected.extend(self._iter_reconciliation_sidecar_rows(store))
    collected.extend(self._iter_reconciliation_bead_rows(store))
collected.extend(self.load_aggregate().get("rows", []))
return {
    "schema_version": ARTIFACT_LINK_ROW_SCHEMA_VERSION,
    "rows": unique_rows(
        row for row in collected if self._row_is_publishable(row, context=context)
    ),
}
```

Three defects compose multiplicatively:

**(a) `context` is `None` in this chop, and the `None` fallback is per-call.**
`_artifact_ref_context_for_store` (`:127`) walks `sdd_store.repo_root` and the sidecar
roots looking for a workspace marker via `find_marker_from_cwd`, and returns `None` when
none is found. The chop resolves stores against the project's **primary checkout**
(`/home/bryan/projects/github/sase-org/sase/`), not a `sase_<N>` workspace, because the
chop's cwd is the axe state dir and `_prefer_current_workspace_record` therefore finds
no marker to substitute. A primary checkout carries no workspace marker, so this returns
`None` on every tick — verified directly:
`store._artifact_ref_context_for_store() is None` → `True`.

`_row_is_publishable` (`:106`) then passes that `None` straight into
`resolve_cli_reference`, and `src/sase/artifact_cli/references.py:92` reads:

```python
resolved_context = context or launch_artifact_ref_context(is_home_mode=False)
```

So **every agent-ref endpoint in every collected row rebuilds the entire launch
artifact-ref context from scratch**, and `launch_artifact_ref_context` →
`build_launch_artifact_ref_context` → `artifact_ref_context` →
`collect_repo_inventory()` walks all 83 workspace clones on this host. A cProfile over
20 resolutions attributes 2.947s of 3.119s total (**95%**) to
`launch_artifact_ref_context`, ~70–150ms per call. There is no memoization on
`launch_artifact_ref_context` or `collect_repo_inventory` anywhere.

**(b) The publishability filter runs before deduplication.** `unique_rows(...)` wraps a
generator that has _already_ applied `_row_is_publishable`, so the filter sees the raw
cross-workspace union including duplicates. For `gh_sase-org__sase` that is **22,239
collected rows that dedupe down to 1,293** — a 17× amplification driven entirely by the
24 workspace clones holding copies of the same sidecar link indexes.

**(c) No memoization of agent-ref publishability within a pass.** Those 22,239 rows
carry **3,096 agent-ref endpoint occurrences across only 193 distinct agent refs** — a
further 16× amplification.

`3096 × ~70ms ≈ 216s`, which matches the measured 213.58s exactly. That is 98% of the
chop's measured 223.67s read-only cost against a 300s budget; the writes, outbox drain,
rename repair, commits, and interpreter startup push it over, and machine load decides
whether any given tick lands at 105s or 640s.

`durable_sidecar_rows` (`:37`) has the identical shape and therefore the identical bug;
it is on `sase artifact doctor`'s path via `inspect_artifact_link_health`.

**Why now.** Cost grows as `clones × rows × per-call inventory walk`, and all three
factors grow with normal use. The chop was introduced in `e391b1a28` when the numbers
were small; it crossed 300s on 2026-08-26.

**Why the chop cannot report it.** Nothing is written to stdout until
`runtime.emit_summary` at the very end, so a SIGKILL at 300s leaves a 0-byte log. And
only job 1 (the sweep) respects a budget — `_SWEEP_WORK_BUDGET_SECONDS = 45.0` at
`src/sase/scripts/sase_chop_artifact_link_backfill.py:51`. Jobs 2, 3, and 4 run
unconditionally for every project with no clock, so the chop has no way to degrade
gracefully.

## 3. Measurements to reproduce

Run from a workspace with `just install` completed. These are read-only except where
noted.

```python
from pathlib import Path
from sase.sdd.artifact_link_store import resolve_artifact_link_store
store = resolve_artifact_link_store(cwd=Path("/home/bryan/projects/github/sase-org/sase/"))
store._artifact_ref_context_for_store() is None      # -> True   (defect (a))
```

Baseline, per enabled project (read-only simulation of the chop's work):

```
gh_bbugyi200__actstat: preview_reconcile=0.06s  dangling=0.10s  rename_hist=0.01s
gh_bobs-org__bob-cli:  preview_reconcile=1.24s  dangling=0.07s  rename_hist=0.11s
gh_sase-org__sase:     preview_reconcile=218.74s dangling=2.55s rename_hist=0.59s
TOTAL: 223.67s  (excludes writes, commits, outbox drain, interpreter startup)
```

Isolating the three amplifiers on `gh_sase-org__sase` (22,239 collected rows, 1,293
unique, 3,096 agent-ref occurrences over 193 distinct refs):

```
row collection (24 clones):                                 10.20s
publishability filter, as shipped:                         213.58s
  ... with the context hoisted once per pass:                1.24s
  ... hoisted + dedupe-before-filter:                        0.09s
  ... hoisted + dedupe-before-filter + memoized:             0.08s
building the hoisted context once:                           0.10s
```

Equivalence check: a prototype applying dedupe-before-filter plus memoization produced
`1279` rows against the shipped path's `1279`, **byte-identical and in identical order**
(`json.dumps(..., sort_keys=True)` compared elementwise).

## 4. Why dedupe-before-filter is safe

`_row_is_publishable` reads only `source_ref` and `target_ref`. `_row_identity`
(`src/sase/sdd/_artifact_link_store_support.py:238`) always includes both endpoint refs
— `("directed", source, relation, target)` or
`("undirected", relation, *sorted((source, target)))`. So publishability is constant
across every row in an identity group, which makes filter-then-dedupe and
dedupe-then-filter select the same representative row in the same order. This is an
argued invariant, not just an empirical one; §3's byte-identical comparison corroborates
it.

## 5. Changes

### 5.1 `src/sase/sdd/_artifact_link_store_reconcile.py` — the fix

1. Add a private helper that resolves the pass context **once**, falling back to
   `launch_artifact_ref_context(is_home_mode=False)` when
   `_artifact_ref_context_for_store()` returns `None`, so no `None` context ever reaches
   `resolve_cli_reference`. Import `launch_artifact_ref_context` lazily inside the
   helper, matching the module's existing lazy-import style for `resolve_cli_reference`
   and `find_marker_from_cwd` (this module is imported from the store impl; a
   module-level import of `sase.artifact_refs` risks a cycle). If the fallback itself
   raises, keep today's behaviour and proceed with `None` — a slow pass is better than a
   failed reconcile.

2. Add a per-pass memo of agent-ref publishability keyed by the agent ref string. Scope
   it to one call, not module state: a process-lifetime cache would make a long-lived
   TUI miss newly published agents. Give `_row_is_publishable` an optional cache
   parameter (defaulting to no cache) so existing callers and tests keep working
   unchanged.

3. In both `durable_sidecar_rows` and `preview_reconciled_aggregate`, dedupe with
   `unique_rows` **before** applying the publishability filter, and build one memo per
   call. Preserve the existing return types exactly: `durable_sidecar_rows` returns a
   `tuple`, `preview_reconciled_aggregate` returns the same `{"schema_version", "rows"}`
   mapping with `rows` a `list`.

Order matters: (1) alone takes the sase project from 213.58s to 1.24s; (2) and (3) exist
so the cost stops scaling with clone count and row count, which is what put the chop on
this cliff in the first place.

### 5.2 `src/sase/scripts/sase_chop_artifact_link_backfill.py` — guardrails

The chop must never again be SIGKILLed with an empty log. Two additions, both small:

1. **A whole-chop wall-clock budget**, distinct from the existing
   `_SWEEP_WORK_BUDGET_SECONDS = 45.0` sweep budget. Add `_CHOP_WORK_BUDGET_SECONDS` set
   safely under the configured 5m timeout (240s is a reasonable choice; leave the config
   timeout alone). Before starting each project in `_run`'s loop, and between jobs 2/3/4
   inside `_run_project`, check the chop deadline; when it has passed, stop starting new
   work, record a warning naming the projects and jobs deferred, and fall through to
   `emit_summary` so the chop exits `ok` with an honest report. Do not interrupt work
   already in flight. Add a `deferred_projects` counter to the emitted summary so the
   deferral is visible in run history.

2. **Progress logging.** Emit one `runtime.log.info` line per project as it starts and
   one per project on completion carrying that project's per-job elapsed seconds (sweep
   / drain / reconcile / repair). These stream to the run log through
   `stream_chop_script`'s pump thread, so the _next_ timeout — if one ever happens —
   names the project and job that consumed the budget instead of leaving a 0-byte file.

Keep the existing per-project `try/except` structure and the existing `_write_state`
checkpoint behaviour untouched.

## 6. Tests

Add to `tests/sdd/test_artifact_link_store_reconcile.py`:

- **Context is built once per pass.** Build a store whose
  `_artifact_ref_context_for_store` returns `None`, monkeypatch the module's
  `launch_artifact_ref_context` seam with a counting fake, seed rows with agent
  endpoints across two stubbed reconciliation stores, call
  `preview_reconciled_aggregate`, and assert the counter is exactly 1 — and that a
  `None` context is never handed to `resolve_cli_reference`. This is the regression test
  for the actual root cause; it must fail against today's code.
- **One resolution per distinct agent ref.** With a counting fake
  `resolve_cli_reference`, feed duplicate rows referencing the same agent ref from
  several stubbed stores and assert the call count equals the number of _distinct_ agent
  refs, not the number of row endpoints.
- **Unpublished agents are still excluded.** A row whose agent ref resolves to a
  non-publishable status must stay out of the reconciled rows, and a row whose agent ref
  resolves must stay in — dedupe-before-filter must not weaken the filter.
- **Same coverage for `durable_sidecar_rows`**, which shares the fixed code path.

Add to `tests/test_axe_chop_artifact_link_backfill.py`:

- **The chop stops starting projects past its budget.** With a monkeypatched clock and
  several fake project records, assert that projects after the deadline are not started,
  that a warning naming them is logged, that the emitted summary reports them as
  deferred, and that the chop still returns a successful result rather than raising.
- **Per-project progress is logged.** Assert the runtime log receives a line naming each
  project processed, so a future timeout is self-diagnosing.

Follow the existing fixtures in both files (`_store`/`_row` from
`tests/sdd/_artifact_link_store_helpers.py`; the `BuiltinChopRuntime` fake and
`monkeypatch.setattr(backfill_chop, "run_artifact_link_backfill_batch", ...)` pattern
already used in the chop tests). Do not reach the network or a real workspace inventory
from any test.

## 7. Verification

1. `just check` (this workspace may need `just install` first — the ephemeral
   workspace's venv can be stale, and `sase_core_rs` will fail to import if so).
2. Re-run the §3 measurement script against the real store and confirm the
   `gh_sase-org__sase` `preview_reconciled_aggregate` drops from ~219s to under ~15s,
   with row collection (~10s) now the dominant remaining term.
3. Confirm output equivalence before/after on the real store: the reconciled row list
   must be identical in content and order (compare `json.dumps(row, sort_keys=True)`
   elementwise), not merely the same length.
4. Force one real chop run and confirm it completes well inside budget with a non-empty
   log:
   ```
   sase axe chop run housekeeping artifact_link_backfill
   ```
   then read the newest
   `~/.sase/axe/lumberjacks/housekeeping/chops/artifact_link_backfill/runs/*.json` and
   assert `status == "success"`, `duration_ms` far below 300000, and a non-zero
   `output_bytes`.
5. Because the change touches shared store code on `sase artifact doctor`'s path, run
   `just check-full` through `/sase_monitor` (`TESTING` / `TESTED`) before landing.

## 8. Out of scope — file as task beads, do not fix here

Each of these was found while diagnosing and is real, but none is required to stop the
timeout. The implementing agent should use `/sase_new_task` for them rather than
widening this plan.

- **`collect_repo_inventory`'s cache key costs more than the cache saves.** In the same
  profile, `_linked_repo_config.repo_config_cache_key` → `_freeze_config_value` made
  **435,600 calls to compute 300 cache keys**, 1.375s of a 3.098s profile. This makes
  every `collect_repo_inventory` caller in sase slower, not just this chop. Worth its
  own investigation.
- **The sweep's 45s budget starves later projects.** `_run`'s `deadline` is a single
  global clock and projects are iterated in a fixed order, so if project 1 exhausts the
  budget, projects 2..N never sweep — permanently, tick after tick. Latent today
  (`sweep_remaining=0` everywhere), but a real fairness bug. Rotating the start index
  across ticks, or giving each project a share, would fix it.
- **`repaired_renames` is reported when nothing changed.**
  `reconcile_and_repair_artifact_links` returns `len(repair.renames)` unconditionally,
  while `inspect_artifact_link_health` uses
  `len(repair.renames) if repair.changed else 0`. That is why every successful run
  reports `repaired_renames=8`/`9` and appears never to converge. The chop should match
  doctor's semantics.
- **An orphaned chop keeps running with no timeout after a daemon restart.** The 08:46
  run above kept running past 300s and wrote its result at ~640s because the daemon that
  owned its timeout had restarted; the replacement daemon only reconciled the record
  afterwards as `stale running chop process exited`. `stream_chop_script`'s `killpg`
  path is correct; the gap is that nothing re-enforces a timeout on an adopted run.
