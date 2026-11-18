---
tier: tale
title: Fix Agents-tab j/k skipping rows when the painted panel tree drifts
goal:
  Agents-tab j/k always walk the rows the panel shows, because repaint gates compare
  against painted facts.
size: medium
decisions:
  self_heal:
    ask:
      Should j/k also self-heal a focused panel whose painted row order drifted from the
      nav stops?
    choices:
      heal:
        Compare painted order to stops on each stop-cache rebuild; force-rebuild on
        mismatch
      gates_only: Rely only on the painted-snapshot gates and the hydration fix
    default: heal
    why: One check per refresh cycle catches in-place writers nobody has found yet
    answer: heal
proposed_by: bbugyi200.athena.1e6
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 18:58:55
status: wip
---

# Fix Agents-tab j/k skipping rows: painted panel tree drifts from the navigation tree

## Symptom (confirmed from the screencast)

`~/tmp/screencasts/20261118_184241.mp4` (keystrokes `j`, `k`, `j`; Agents tab,
`group: by status`). The `@default` panel was painted as:

```
Running   tailnet (STRANDED ⚐) … tailnet-1d      <- selected
Waiting   ⚡ bob-cli (WAITING ◐ bob-cli-ms) ev
Done      bob-cli (TALE DONE) … f1
          …
```

`j` from `tailnet` jumped straight to `f1`, `k` from `f1` jumped straight back to
`tailnet`, and `j` jumped to `f1` again. The detail panel followed, so `current_idx`
really moved to `f1`. The `ev` row was never reachable. The keymaps themselves are fine:
`j`/`k` route through `action_next_patch` / `action_prev_patch` to
`_navigate_agents_panel` (`src/sase/ace/tui/actions/navigation/_basic.py`).

## Root cause

Two trees that are supposed to match had drifted apart:

- **What j/k walk:** `_panel_navigation_stops()`
  (`src/sase/ace/tui/actions/agents/_navigation_order.py`) rebuilds
  `build_agent_tree(slot.agents, …)` from the _current_ roster.
- **What the user sees:** the `AgentList` widget's rows, built by the last `update_list`
  / `try_insert_rows` / row patch.

Evidence that the painted tree was stale and the navigation tree was correct:

1. A fresh `sase tui` on the same host, against the same data, paints `@default` as
   `Waiting: ev`, then `Done: tailnet (DONE), f1, f0, …`. With `tailnet` in Done and
   sorted after `ev`, the fresh order `[ev, tailnet, f1, …]` explains the recording
   exactly: `j` from tailnet goes to f1 and `k` from f1 goes to tailnet.
2. The live TUI's panel title (recomputed on every paint) read
   `@default · 10 [W1 U1 D8]`, counting tailnet as an unread done row. The roster
   already had tailnet in Done; only the painted rows still showed it under `Running`.
3. The apollo load log shows the session's last turn finishing at 18:39:13. At least
   seven _full_ loads followed before the 18:42 recording, and none of them rebuilt the
   panel.

Point 3 identifies the gate. A full load hands the panel brand-new row objects, yet
`_panel_content_is_unchanged`
(`src/sase/ace/tui/actions/agents/_display_panel_widgets_paint.py`) kept answering
"unchanged". The reason is that it recomputes
`panel_row_signature(widget._agents, mode)` from the _current state_ of the `Agent`
objects the widget holds. Those objects are shared, mutable references, so once one of
them is mutated in place the widget's record of "what I painted" silently becomes "what
is true now". The same blind spot affects:

- `_status_row_patch_is_safe` / `_machine_row_patch_is_safe` /
  `_clan_row_patch_preserves_order` in `_try_patch_agent_row`
  (`src/sase/ace/tui/actions/agents/_display_panel_patches.py`). They compare
  `widget._agents[local_idx]` with the incoming row, so a mutated held object makes a
  Running→Done bucket move look like a "badge-only" change. The row's text is then
  patched in place (which is why it read `STRANDED ⚐` with its final runtime) while its
  tree position stayed under `Running`.
- The in-place insert path (`try_insert_rows`,
  `src/sase/ace/tui/widgets/_agent_list_build_insert.py`) accepts retained rows via
  `old is agent or old == agent`.

Once corrupted, the panel never self-heals until something else forces a rebuild (fold
change, grouping change, or a membership change in that panel).

**Verified in-place writer of a grouping input (likely trigger here):** selecting a
compact/projected row runs detail hydration (`_defer_projected_record_detail` →
`_hydrate_projected_record_detail` → `apply_projected_agent_record`,
`src/sase/ace/tui/models/_projected_record.py`). `_restore_deferred_meta_detail` →
`enrich_agent_from_meta_wire` restores the deferred `status_bucket` override
(`agent.status_bucket = meta.status_bucket`), and `agent_status_bucket()` prefers that
override. On a session root whose status/bucket were mirrored from a live child
(`_mirror_root_from_child`), the restore writes the root turn's own bucket (`Done`) over
the mirrored one. The caller then only calls `_invalidate_agent_panel_cache()` and
reschedules the detail; nothing repaints the panel. `tailnet` was the selected row in
the recording. `hydrate_agent_attempt_history` in `_apply_agent_detail_update`
(`_display_detail_render.py`) follows the same mutate-then-invalidate pattern.

**Suspects ruled out:** the exact-delta copy-on-write fast path (commit `10922458c1`)
was probed directly. Running consecutive exact deltas over a session graph and roster
through `prepare_loaded_agents_worker_boundary` + `share_back_boundary_fast` mutated
**no** previously published object and left no out-of-roster references. Share-back does
keep widget-held objects identical across loads, which widens the aliasing window, but
the defect is the gates reading mutable held objects. That defect predates the fast
path. The fix must not depend on which writer fired.

## Changes

### 1. Snapshot painted grouping facts on the widget (primary fix)

- Add an immutable per-row painted snapshot to `AgentList`, e.g.
  `_painted_row_facts: tuple[PaintedRowFacts, ...]` aligned with `widget._agents`.
  `PaintedRowFacts` is a small frozen dataclass holding: identity, the mode-specific
  grouping signature (`status_grouping_signature` / `machine_grouping_signature` /
  `()`), tribe, `status_bucket_for(agent)`, and the clan wave `(wave_index, rank)` or
  `None`. Keep the computation in one helper next to `panel_row_signature`
  (`_display_panel_widgets_keys.py`) so both sides use identical keys.
- Record it in every place that assigns `widget._agents`: `build_list`
  (`_agent_list_build_rebuild.py`), `try_insert_rows` (`_agent_list_build_insert.py`),
  `try_remove_rows` (`_agent_list_build_remove.py`), `render_collapsed`
  (`_agent_list_widget.py`; empty tuple), and the single-slot replacement in
  `_try_patch_agent_row` (update only that slot, and only after the patch succeeds).
- `_panel_content_is_unchanged`: compare `widget._painted_row_facts` against the facts
  computed for `panel_agents` instead of recomputing `panel_row_signature(current, …)`
  from held objects. Treat a missing snapshot as "changed".
- `_try_patch_agent_row`: run the BY_STATUS / BY_MACHINE / clan-order guards against the
  painted facts for `local_idx` rather than `widget._agents[local_idx]`. A bucket,
  subgroup, anchor, or wave-rank difference keeps returning `False` with the existing
  `status_membership_change` / `clan_member_order_change` reasons.
- `try_insert_rows`: decline with `panel_membership_change` when any retained row's
  painted facts differ from its new facts. Today the `old is agent` short-circuit lets a
  mutated row through.
- Leave `panel_agents_match` / `_panel_paint_key` semantics otherwise unchanged.

### 2. Make detail hydration re-group the panel when it changes grouping inputs

- In `_hydrate_projected_record_detail` and in the `hydrate_agent_attempt_history`
  branch of `_apply_agent_detail_update`, capture the row's grouping facts before
  mutating. If they changed afterwards, schedule the normal display refresh
  (`_refresh_agents_display_debounced()`) in addition to the cache invalidation, so the
  panel re-groups promptly instead of waiting for the next load. With change 1 in place,
  that refresh rebuilds the panel.
- Add a model-level test hydrating a session root whose status/bucket are mirrored from
  a running child. If hydration changes `status_bucket_for(root)`, fix
  `_restore_deferred_meta_detail` so it does not overwrite the list-owned
  `status_bucket` of a mirrored session root. Its docstring promises "without changing
  list behavior"; preserve the pre-hydration list value for mirrored rows. If the test
  shows no change, keep only the refresh trigger.

> [!decision] self_heal = heal In `_panel_navigation_stops`, only on a cache **miss**
> (never on cache hits or per keystroke), compare the agent stops' global-index order
> with the focused `AgentList`'s painted visible agent order (`_row_entries` /
> `_row_by_agent_idx` mapped through `slot.global_indices`). On mismatch, clear that
> widget's `_painted_row_facts` and `_panel_paint_key`, record a new
> `AgentRefreshFallbackReason` (e.g. `painted_order_drift`, registered in
> `ALL_AGENT_REFRESH_FALLBACK_REASONS` in `_refresh_trace.py`), and call
> `_refresh_agents_display_debounced()`. Keep the check out of pump callbacks' slow
> paths and allocation-light (`tui_perf.md` rules 6 and 7).

> [!decision] self_heal = gates_only Skip the navigation-time comparison and the new
> trace reason; changes 1 and 2 are the whole fix.

## Tests

Add regression tests next to the existing Agents-tab suites
(`tests/ace/tui/test_agent_jk_navigation.py`,
`tests/ace/tui/test_agent_display_group_fold_repaint.py`, and the row-patch tests in
`tests/ace/tui/test_agents_live_hint_refresh.py` show the pilot/harness patterns):

1. **Video reproduction (pilot):** BY_STATUS grouping, one panel with `A` RUNNING
   (newest), `W` WAITING (older), and `D1`, `D2` DONE. Paint, then mutate the _held_ `A`
   object in place to a Done bucket (simulating any aliasing writer) and drive the
   ordinary refresh path with a roster whose `A` is Done. Assert the painted visible
   order equals the `_panel_navigation_stops()` agent order (`[W, A, D1, D2]`) and that
   `j` from the top row selects `W`. Confirm this test fails on the pre-fix code before
   landing.
2. `_panel_content_is_unchanged` returns `False` after a held row object is mutated
   across a status-bucket boundary.
3. `_try_patch_agent_row` declines (`status_membership_change`) when the held object was
   mutated Running→Done before the patch, and the subsequent panel rebuild places the
   row in Done.
4. `try_insert_rows` declines when a retained row's painted facts changed.
5. The hydration test from change 2 (session root with a mirrored running child).
6. If `self_heal = heal`: a drifted painted order (forced by corrupting
   `_painted_row_facts` / the rows) is repaired on the next stop-cache rebuild and
   records `painted_order_drift`.

## Verification

- `sase tool run check` (the agent default; do not run `check-full`).
- Run the Agents-tab perf bench (`pytest -s -m slow tests/ace/tui/bench_tui_jk.py`)
  before and after. j/k p95 must stay under 16 ms, and the gate must not cost more than
  the `panel_row_signature` recomputation it replaces.
- The change is not visual, so no PNG goldens should move. If `just check` reports
  visual-snapshot drift, inspect it rather than regenerating blindly.
