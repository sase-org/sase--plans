---
tier: tale
title: Stop the precomputed finalize plan from republishing a local-only roster
goal:
  A disk apply that uses its off-thread finalize plan publishes the same rows the UI
  thread just projected, so the Agents-tab @epic tribe panel survives every apply on a
  host with fleet rows and sase-13i.4's athena soak passes with evidence that the plan
  path was actually exercised.
size: medium
proposed_by: bbugyi200.athena.sase-13i.4.f0
create_time: 2026-09-20 07:55:05
status: wip
---

# Stop The Precomputed Finalize Plan From Republishing A Local-Only Roster

## Goal

A disk apply that uses its off-thread finalize plan publishes the same rows the UI
thread just projected. The Agents-tab `@epic` tribe panel survives every apply on a host
with fleet rows, and `sase-13i.4`'s athena soak passes with evidence that the plan path
was actually exercised.

## Problem

`sase-13i.1`, `.2` and `.3` landed. The `sase-13i.4` soak on athena (30 min,
`v0.17.1+945.g9231c9352`, `by_status` + `NOT machine:apollo`, 82k trace records at
`~/.sase/perf/sase-13i.4-soak-tui_trace.jsonl`) still caught three
`2 panels → 1 panel → 2 panels` dips: one at startup (`ts 1789902029.67`) and two
mid-run (`ts 1789902356.01`, `ts 1789903497.27`).

The note on `sase-13i.4` attributes them to "an incomplete apply publishing 3-6 rows
though `finalize_query_filter` yielded ~30". **That attribution is wrong** and should
not be carried into this work. `agents.finalize_query_filter` traces
`agents=len(visible_agents)` — the filter's _input_, not its output
(`_loading_compute_finalize.py:340`). The incomplete-load merge is not involved; `.3`'s
guard is working.

### Root cause: the finalize plan is computed pre-projection and applied post-projection

The plan is computed off-thread over the **local-only** roster, and installed on the UI
thread **after** the fleet projection has already widened that roster. Installing it
silently reverts the projection.

1. In the worker (`_loading_disk_full.py:328`, `_loading_disk_delta.py:141`),
   `attach_finalize_plan_to_boundary` computes the plan from
   `boundary.fold.visible_agents` (`_loading_compute_finalize.py:102`). At that point
   the boundary holds disk/cache rows plus proc shells only — `sase-13i.1` merged the
   proc projection in, but nothing merges the fleet projection.
2. On the UI thread, `_apply_loaded_agents_prepared_inner` (`_loading_apply.py:594-611`)
   calls `_project_agents_for_current_mode_after_load`, which runs
   `project_mixed_agent_tree(local_rows, _agents_fleet_rows)`
   (`_fleet_projection.py:135-162`) and assigns the **mixed** result to
   `self._agents_with_children` and `self._agents`.
3. `_finalize_agent_list(..., precomputed_plan=finalize_plan)` reaches
   `_apply_finalize_plan`, whose first roster statement is
   `app._agents = list(plan.query.filtered_agents)` (`_loading_finalize.py:278`). The
   plan's rows came from step 1, so **every fleet row is dropped**, along with any tribe
   panel those rows were the only occupants of.

Two further consequences of the same line, both already visible in the epic's symptom
list:

- `plan.panel_group_keys` was enumerated from local-only rows, so
  `reconcile_panel_fold_registries(app, plan.panel_group_keys)`
  (`_loading_finalize.py:317`) garbage-collects the fold registry entries of fleet-only
  tribes. That is the `initially_expanded` reset on remount.
- `plan.selection` was computed over local-only rows, so the cursor is restored against
  a roster the user is not looking at.

### Why it is intermittent, and why it is 100% reproducible when it fires

`_select_finalize_plan` (`_loading_apply.py:265-290`) discards the plan whenever
`PreparedFinalizeStaleToken` drift is detected. That token captures selection, fold
levels, query, status overrides, grouping mode, hide flag, unread ids and
`proc_generation` (`_loading_compute_finalize.py:71-91`) — and **nothing about the fleet
projection**. So the token cannot detect this defect: the worker and the UI thread see
the _same_ `_agents_fleet_rows`. This is a scope bug, not a staleness bug.

What the token does do is discard the plan on unrelated churn. In the soak, 128 plans
were computed and, judging by the published row counts, 3 survived every token field —
and all 3 produced a dip. When the plan is applied on a host with fleet rows, the roster
is wrong every time.

The arithmetic in the trace confirms it end to end, at `ts 1789903497`:

| span                              | value               | meaning                                        |
| --------------------------------- | ------------------- | ---------------------------------------------- |
| `incomplete_load_merge`           | cached=421          | mixed roster before the apply                  |
| `fold_filtering` (compute.py:295) | 392                 | local-only unfiltered (421 − 29 fleet)         |
| `finalize_query_filter`           | 30                  | filter **input**: local-only visible           |
| `finalize_agent_list`             | 49                  | `len(self._agents)` after projection           |
| `final_display_refresh`           | 6                   | `len(app._agents)` after the plan overwrote it |
| `refresh_panel_widgets`           | agents=6, panels=1  | the dip                                        |
| next `fleet_refresh` 0.4 s later  | agents=16, panels=2 | the recovery                                   |

The projection contributes a constant 29 rows on every apply (436→407, 421→392); 19 of
them survive fold, which is exactly the 49 − 30 gap. The recovery is
`_reproject_agents_from_current_mode` (`_fleet_projection.py:178-214`), which rebuilds
from `_agents_with_children` — never clobbered — and finalizes with **no** precomputed
plan, so the inline path publishes the correct 16 rows.

The inline path is correct because `apply_agents_live_query_filter` refuses a cached
facade that does not `covers(materialized)` (`agent_live_query_engine.py:257-262`). The
plan path has no equivalent guard. That asymmetry is the fix's shape.

## Approach

Two independent changes. The first makes the defect impossible; the second keeps the
off-thread optimization useful.

1. **Scope guard (correctness).** Fingerprint the rows the plan was computed over, and
   discard the plan at commit time if the roster the UI thread is about to publish is
   not that row set. On a fleet host today this makes the plan always discard, degrading
   to the inline path that is already correct.
2. **Project before planning (performance).** Carry the fleet rows onto the prepared
   snapshot and apply the projection inside the boundary, so the plan is computed over
   the roster that will actually be published and the fingerprint matches.

Doing (1) first means the tree is correct even if (2) is backed out.

## What not to do

- Do not "fix" the incomplete-load merge or `has_more` handling. `sase-13i.3` landed and
  the traces show the merge behaving: `cached` never shrinks and `_agents_with_children`
  is intact across every dip.
- Do not add a fleet field to `PreparedFinalizeStaleToken` and stop there. The token
  compares worker state to UI state; both see identical fleet rows, so no token field
  detects this. The fingerprint must describe the plan's **input rows**, not a mutable
  input it captured.
- Do not delete `_project_agents_for_current_mode_after_load` or move
  `_fleet_rows_with_dispatch_provisionals` off the UI thread. It mutates
  `_agents_dispatch_provisional_rows` (pops reconciled rows,
  `_fleet_dispatch_launches.py:185-199`) and is not worker-safe. Capture its _result_ on
  the snapshot instead, at snapshot time, on the UI thread.
- Do not change the order of fold filtering relative to the projection. Today fold
  filtering and `refresh_runner_slot_context` run on local-only rows and the projection
  runs after. Fleet rows are not fold-filtered. Preserve that exactly; widening fold to
  fleet rows is a separate behavior change and is out of scope.
- Do not change `sase-core`. This is Textual presentation state and TUI loader glue
  only, consistent with the `sase-13i` epic's boundary.
- Do not restart or kill the user's interactive TUI (PID was 1880150, started Sep 15) to
  run the soak. Launch a separate traced instance, as the previous `sase-13i.4` attempt
  did.
- Do not edit `sase/memory/tui_perf.md`. The `sase-13i.4` bead already carries a
  `PROPOSED FOLLOW-UP:` note for that; leave it for the epic's land agent.
- Do not close `sase-13i` or any ancestor bead. Close only `sase-13i.4`.

## Steps

### 1. Reproduce first

Add a failing test to `tests/test_agents_tab_finalize_plan.py` that drives a full apply
with fleet rows present:

- seed `app._agents_fleet_rows` with a row whose `fleet_origin_alias` is set and whose
  `tribe` is `epic`, and a local roster whose rows are all in a different tribe;
- build the boundary and attach the finalize plan exactly as the worker does
  (`prepare_loaded_agents_apply_boundary` → `attach_finalize_plan_to_boundary`);
- apply it through `_apply_loaded_agents_prepared` with a snapshot that produces **no**
  stale-token drift, so `_select_finalize_plan` returns the plan;
- assert the published `app._agents` still contains the fleet row, and that
  `panel_keys_for(app._agents)` includes `"epic"`.

It must fail on the current tree with the fleet row absent and one panel key. Model the
fake on the existing `FakeAgentApp` in `tests/_agents_tab_query_helpers.py`; note that
file's autouse fixture pins the legacy query dialect, so add the new case where the
unified engine is active too (`override_flags(agents_unified_query=True)`) — the host
runs the unified engine.

Add a second failing case asserting the fold registry keeps the fleet-only tribe's entry
(today `reconcile_panel_fold_registries` drops it).

### 2. Scope guard: a plan only applies to the rows it was planned over

- Add `input_row_identities: tuple[AgentIdentity, ...]` to `PreparedFinalizePlan`
  (`_loading_compute_types.py`), populated in `_compute_finalize_plan` from the
  `visible_agents` argument before filtering.
- In `_select_finalize_plan` (`_loading_apply.py:265`), after the existing token
  comparison, also require
  `precomputed.input_row_identities == tuple(a.identity for a in self._agents)`. Call it
  _after_ the roster assignment so `self._agents` is the projected list — this means
  moving the `_select_finalize_plan` call below `self._agents = visible_agents` at
  `_loading_apply.py:611`, which it already is. Verify that ordering rather than
  assuming it.
- On mismatch return `None` and record the discard reason. Do not raise, do not re-run
  the query inline here: returning `None` is exactly the existing "recompute
  synchronously" path.

After this step alone, step 1's tests pass and the dip is gone.

### 3. Project the fleet rows before the plan is computed

- Add `fleet_rows: tuple[Agent, ...]` to `PreparedApplySnapshot`
  (`_loading_compute_types.py`), populated in `_make_prepared_apply_snapshot`
  (`_loading_apply.py:~240`) from
  `self._fleet_rows_with_dispatch_provisionals(list(self._agents_fleet_rows))`. That
  call already runs on the UI thread there.
- In `prepare_loaded_agents_apply_boundary` (`_loading_compute.py:223`), after fold
  filtering and after `refresh_runner_slot_context`, project both lists with
  `project_mixed_agent_tree` (pure; `models/_agent_tree.py:595`) and keep the
  pre-projection lists on `PreparedFoldFiltering` as `local_unfiltered_agents` /
  `local_visible_agents` so the UI thread can set the `_agents_local_*` mirrors without
  recomputing.
- In `_loading_apply.py`, feed `_project_agents_for_current_mode_after_load` the
  boundary's local lists (so the `_agents_local_*` mirrors and the dispatch provisional
  reconciliation keep their current behavior) and publish the boundary's
  already-projected lists. The UI-thread re-projection is idempotent when the fleet rows
  have not moved; if it is not, step 2's fingerprint check fires and the plan is
  discarded. Both outcomes are correct.
- `rebase_prepared_apply_boundary_on_proc_projection` (`_loading_compute.py:200`) sets
  `finalize=None`; it must keep doing so, and must re-run the projection on the rebased
  lists so the boundary never carries an unprojected roster.

Confirm with a test that the plan is now _used_ (not discarded) when the fleet rows are
unchanged between snapshot and commit, and that it publishes the mixed roster. Without
this assertion step 3 is unverifiable — step 2 alone would make the tests in step 1 pass
either way.

### 4. Make the next soak able to prove this

The previous soak could not distinguish these paths, which is how the misdiagnosis
happened. Fix the three traces that misled it:

- `agents.finalize_query_filter` (`_loading_compute_finalize.py:340`): record both
  `agents_in` and `agents_out`. Keep `agents` as the input for continuity with the
  existing soak, or drop it and say so in the bead note — but do not leave a single
  ambiguous `agents` field.
- `agents.apply_loaded_agents_prepared` (`_loading_apply.py`): record `finalize_plan` as
  `applied` / `discarded` / `absent`, and on a discard record
  `finalize_plan_discard_reason` (`stale_token` or `roster_fingerprint`). This is what
  proves step 3 exercised the fixed path instead of silently always discarding.
- `agents.refresh_panel_widgets` (`_display_panel_widgets.py:57-70`): add
  `panel_widget_ids` from `panel_widget_id_for_key` over `self._panel_group.panel_keys`.
  The `sase-13i` acceptance criterion "`@epic`'s tribe-stable widget id is continuously
  present in `agents.refresh_panel_widgets` spans" is not assertable without it; this is
  also the bead's existing `PROPOSED FOLLOW-UP:` note #3, which this step closes out.

### 5. Guard the class of bug, not just the instance

Add a repro invariant in `src/sase/ace/tui/repro/invariants.py` alongside
`_check_post_complete_incomplete_shrink`: a published visible roster must not omit an
identity that is present in the same apply's `_agents_with_children` unless a fold level
or the committed query explains the omission. Wire it into `check_bundle_invariants`.
This is the invariant that would have caught the defect at `sase-13i.1` time.

Add the cross-phase guard to `tests/perf/test_agents_display_rebuild_guard.py` that
`sase-13i.4` step 3 still owes: a full apply on a non-drifting stale token with fleet
rows contributing the `epic` tribe leaves the `epic` `AgentList` object identity
unchanged and never drops `panels` to 1.

### 6. Verify

- `just fix` inline, then `sase tool run check`. **Baseline note:** `mypy` is already
  red on master with 20 `no-untyped-def` errors in `src/sase/main/ace_tmux.py`,
  `ace_tmux_session.py` and `ace_tmux_window.py` (landed in `e89aa2566a`, unrelated to
  this work). Confirm the count is still exactly 20 and in those three files only; do
  not fix them here and do not treat them as this plan's failure. Record them for the
  epic's land agent with `sase bead note sase-13i.4 'PROPOSED FOLLOW-UP: ...'`.
- `just fix-tui-screenshots` only if Agents-tab goldens actually move. Inspect the
  report; generation is not approval.

### 7. Soak on athena and close the bead

- Launch a separate TUI with `SASE_TUI_TRACE=1` on the landed tree, with the real
  persisted state (`by_status`, committed `NOT machine:apollo`). Confirm the imported
  SHA matches HEAD before measuring (tui_perf rule 15). Leave the user's interactive TUI
  alone. Capture recipes are in `docs/perf_runbook.md`.
- Run it at least 30 minutes through `/sase_monitor`; do not block the turn on `sleep`.
- Assert from the trace, not from watching:
  - zero `panels` transitions from 2 to 1, and zero `refresh_panel_widgets` spans whose
    `panel_widget_ids` lack `agent-list-panel-epic` while a prior span contained it;
  - `finalize_plan=applied` occurs at least once — otherwise step 3 is unproven and the
    soak is not evidence;
  - zero `fallback_reason=active_search`;
  - no `display_full_rebuild` on unchanged occupancy keys;
  - one `agents.apply_loaded_agents_prepared` span never pairs with a second finalize
    for the same load.
- Record before/after evidence in a bead note, including the counts above and a
  comparison against the three dips in the prior soak.
- Run `sase bead epic-symbols sase-13i.4` and resolve or re-key any `--epic-symbol`
  leftovers before closing.
- Close **only** `sase-13i.4`:
  `sase bead close sase-13i.4 --note "<what you verified>"`. Leave `sase-13i` open for
  its land agent.

## Acceptance

- An apply whose stale token does not drift, on an app with fleet rows in a tribe no
  local row occupies, publishes those fleet rows and keeps that tribe's panel key. Fails
  before step 2, passes after.
- A finalize plan whose input rows differ from the roster the UI thread is about to
  publish is discarded with a recorded reason, and the inline pipeline publishes the
  correct roster.
- With fleet rows unchanged across the worker boundary, the plan is applied (not
  discarded) and publishes the mixed roster — proving step 3 restored the off-thread
  path rather than disabling it.
- The fold registry retains entries for tribes occupied only by fleet rows across an
  apply.
- `agents.finalize_query_filter` reports input and output counts distinctly;
  `agents.apply_loaded_agents_prepared` reports whether the plan was applied;
  `agents.refresh_panel_widgets` reports panel widget ids.
- A 30-minute athena soak on the landed tree shows zero `2 → 1` panel transitions, at
  least one `finalize_plan=applied`, and zero `active_search` fallbacks.
- `sase tool run check` is green apart from the 20 pre-existing `ace_tmux*` mypy errors,
  whose count and location are unchanged.
- `sase-13i.4` is closed with the soak evidence in its note. `sase-13i` is not closed.

## Code map

| Path                                                           | Role                                                                     |
| -------------------------------------------------------------- | ------------------------------------------------------------------------ |
| `src/sase/ace/tui/actions/agents/_loading_finalize.py`         | `_apply_finalize_plan:278` overwrites the projected roster               |
| `src/sase/ace/tui/actions/agents/_loading_apply.py`            | Projection at 594-611; `_select_finalize_plan` at 265; snapshot build    |
| `src/sase/ace/tui/actions/agents/_fleet_projection.py`         | `_project_agents_for_current_mode_after_load`; the recovering reproject  |
| `src/sase/ace/tui/actions/agents/_loading_compute.py`          | `prepare_loaded_agents_apply_boundary`; proc rebase                      |
| `src/sase/ace/tui/actions/agents/_loading_compute_finalize.py` | Plan computation; stale token; misleading trace                          |
| `src/sase/ace/tui/actions/agents/_loading_compute_types.py`    | `PreparedApplySnapshot`, `PreparedFinalizePlan`, `PreparedFoldFiltering` |
| `src/sase/ace/tui/actions/agents/_display_panel_widgets.py`    | `agents.refresh_panel_widgets` span                                      |
| `src/sase/ace/tui/models/_agent_tree.py`                       | `project_mixed_agent_tree` (pure, worker-safe)                           |
| `src/sase/ace/tui/repro/invariants.py`                         | Where the new published-roster invariant goes                            |
| `tests/test_agents_tab_finalize_plan.py`                       | Plan-path tests; no fleet coverage today                                 |
| `tests/perf/test_agents_display_rebuild_guard.py`              | Cross-phase display guards                                               |
