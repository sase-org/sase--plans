---
tier: epic
title: Stop the Agents-tab @epic tribe panel from flickering
goal: "The Agents-tab @epic tribe panel stays mounted as a tribe-keyed widget across
  disk applies, proc-shell sync, bounded loads, and sibling-tribe occupancy churn: one
  logical apply publishes one roster, incomplete bounded loads never delete by omission,
  and a standing filter query no longer forces a full rebuild of untouched panels.

  "
phases:
  - id: atomic-roster-publication
    title: Publish one aggregate roster per disk apply
    depends_on: []
    size: medium
    description: "atomic-roster-publication: merge the generation-stamped proc
      projection into the disk/cached roster before the single finalize, rebase if the
      generation moved during the worker, and cover the empty-disk plus live-projection
      regression.

      "
  - id: tribe-stable-widgets
    title: Key AgentList widgets by tribe and stop blanking untouched panels
    depends_on: []
    size: medium
    description: "tribe-stable-widgets: give each tribe a stable widget id, insert or
      remove one panel without rebuilding siblings, skip clear_options on unchanged row
      sets, and finish the standing-query row-remove path.

      "
  - id: removal-authority
    title: Stop incomplete bounded loads from replacing a larger cache
    depends_on:
      - atomic-roster-publication
    size: medium
    description: "removal-authority: patch same-query bounded loads regardless of
      has_more, keep a nonempty cache across a bounded zero, clear the complete-history
      latch only on a committed-query change, and converge revalidate with auto-refresh.

      "
  - id: verify-on-athena
    title: Prove panel stability on the live host with traces
    depends_on:
      - atomic-roster-publication
      - tribe-stable-widgets
      - removal-authority
    size: medium
    description:
      "verify-on-athena: restart onto the landed tree, soak under by_status and the
      standing NOT machine:apollo query, and assert from traces that @epic never
      unmounts and one apply never publishes N then 0 then N."
proposed_by: bbugyi200.athena.0ns
create_time: 2026-09-19 10:43:09
status: wip
---

- **PROMPT:**
  [prompts/202609/epic_tribe_panel_flicker.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/epic_tribe_panel_flicker.md)

# Stop The Agents-Tab @epic Tribe Panel From Flickering

## Problem

The `@epic` Agents-tab tribe panel still disappears and reappears after two closed epics
that targeted this symptom:

- sase-127 (`7058f16ce` and follow-ups) stopped the 171↔484 auto-refresh oscillation,
  admitted a _stable_ search query to the incremental display path, and skipped no-op
  fleet reprojection repaints.
- sase-12p (`1fa7e5fc3`) admitted `BY_STATUS` grouping to that incremental path and
  surfaced stale imported TUI code.

Those fixes are installed and running. Live traces on athena
(`~/.sase/perf/tui_trace.jsonl`, `~/.sase/logs/tui_agent_loads.jsonl`) still show
`2 panels / 17 agents → 1 panel / 0 agents → 2 panels / 17 agents` in ~25–55 ms — long
enough for multiple terminal frames. The empty frame is a published apply, not a paint
artifact. Prior 30-minute soaks measured `"epic" in panel_keys_for` under the loads they
exercised; they did not assert widget object identity, `clear_options` blanks, or 50 ms
unmounts.

The host's persisted Agents state is grouping `by_status` and committed query
`NOT machine:apollo`, with split tribe panels. The mechanism is not epic-specific;
`@epic` is the panel the user is watching.

Three verified defects compound. Closing any one of them alone is not enough.

### Defect 1: dual publication per disk apply

`_apply_loaded_agents_prepared()` in `src/sase/ace/tui/actions/agents/_loading_apply.py`
assigns the prepared roster and calls `_finalize_agent_list()` — which computes panel
keys and renders — then calls `_sync_proc_shell_agents_from_projection()` in
`src/sase/ace/tui/actions/_proc_action_completion.py`, which merges the live
proc-observer projection and finalizes **again**.

`PreparedApplySnapshot` (`src/sase/ace/tui/actions/agents/_loading_compute_types.py`)
captures `cached_agents_with_children` (proc rows already in the roster, per
`6eb51ac49`) but not the live proc projection or a projection generation.
`prepare_loaded_agents_apply_boundary()`
(`src/sase/ace/tui/actions/agents/_loading_compute.py`) carries proc shells from that
cached roster only. A proc row present only in the newer projection is therefore absent
from the first publication by construction. The traced `0 → 17` sequences are that
ordering.

`test_loader_apply_keeps_proc_shell_rows_in_roster_before_finalize` and
`test_unchanged_proc_projection_runs_one_finalize_pass` in
`tests/ace/tui/test_proc_shell_selection_survives_refresh.py` seed **both** the roster
and the projection. They do not cover a projection-only proc row over an empty disk
load, and they do not assert that a whole apply performs only one _visible_ finalize
when the projection is richer than the cache.

### Defect 2: tribe panels are index-addressed slots

`panel_widget_id(panel_idx)` in `src/sase/ace/tui/actions/agents/_display_helpers.py`
keys widgets as `agent-list-panel`, `agent-list-panel-1`, … . `_refresh_panel_widgets`
in `src/sase/ace/tui/actions/agents/_display_panel_widgets.py` mounts missing indices,
removes extra indices, and calls `update_list` on every remaining `AgentList`.
`build_list` in `src/sase/ace/tui/widgets/_agent_list_build_rebuild.py` starts with
`widget.clear_options()`.

`panel_keys_for` in `src/sase/ace/tui/models/agent_panels.py` builds the panel set from
_rendered_ rows (`status != "STARTING"` excluded). Zero members ⇒ no key ⇒ widget
removed. An empty tab is special-cased to a single `@default` panel, so a 0-row apply
unmounts every named tribe. Unmounting retires fold intents (`retire_panel_fold_intents`
in `src/sase/ace/tui/actions/agents/_panel_fold_intent.py`), so a remount re-applies
`ace.tribes.*.initially_expanded`.

`_try_refresh_agents_display_incremental` in
`src/sase/ace/tui/actions/agents/_display.py` requires
`next_panel_keys == old_panel_keys`; any membership change records
`panel_membership_change` and full-rebuilds. There is no single-panel insert/remove
path.

The row-remove helper still bails on any nonempty `_agent_search_query`
(`src/sase/ace/tui/actions/agents/_display_panel_patches.py`). sase-127.2 removed the
blanket `active_search` fallback from the display-diff path when the query _text_ is
unchanged, but not from this helper. With the standing `NOT machine:apollo` query, every
identity removal forces a full rebuild that `clear_options()`s `@epic`.

A sibling tribe appearing or disappearing _before_ `epic` in sort order also repaints a
different tribe into the widget the user was watching, because the widget is the slot,
not the tribe.

### Defect 3: the merge guard treats `has_more=false` as removal authority

`merge_incomplete_load_after_complete_history` in
`src/sase/ace/tui/actions/agents/_loading_compute_merge.py` patches over the cache only
when the complete-history latch is set, the load is an `artifact_delta`, or the load is
`bounded_prefix AND has_more`. Otherwise it returns the incoming prep unchanged —
replacement.

A zero-row bounded result necessarily reports `has_more=false` (in the index, `has_more`
means matching completed candidates exceeded the budget — a pagination fact, not an
authoritativeness fact). A bounded zero therefore bypasses the patch path.

The latch is fragile: `_loading_apply.py` clears `_agents_seen_complete_history` on
**any** incomplete apply whose `history_query_key` does not match. One mismatched-key
load disarms the guard for everything that follows.

The `7058f16ce` tests in `tests/test_agents_tab_apply_boundary.py` cover
`bounded_prefix=true, has_more=true`. They do not cover a same-query bounded zero, nor
`tier1_index_revalidate` swings (still 41–696 rows on this host) publishing a smaller
identity set.

The repro invariant `post_complete_incomplete_shrink`
(`src/sase/ace/tui/repro/invariants.py`) is disabled until a complete-history snapshot
is observed, so the traced nonempty→empty transitions were never rejected.

## What not to do

- Do not slow refreshes, add debounce delays, or paint a spinner. The empty frame is a
  published apply, not a slow paint.
- Do not force Tier 2 / unwindowed history on every refresh. That reverts sase-127's CPU
  win (~54% → ~0.25 cores). sase-124 owns freshness/cost.
- Do not only debounce `initially_expanded`. That hides remount fold-reset and leaves
  the unmount in place.
- Do not collapse split tribes into merged mode as the fix. Merged mode avoids the
  slot-index bug by having one widget; it does not fix the 0-row apply or the dual
  publication.
- Do not treat `has_more=false` as proof a bounded snapshot may delete by omission —
  anywhere.
- Do not change `sase-core` in this epic. The index rebuild is transactional and is not
  the demonstrated producer of the zeros. Core hardening (one SQLite read transaction,
  logical index generation, explicit `snapshot_authoritative`) is justified follow-up,
  not the flicker fix. All changes here are Textual presentation state and TUI loader
  glue in this repo.
- Do not block this epic on BY_DATE incremental support. This host is on `by_status`;
  `unsupported_grouping` for BY_DATE stays residual.
- Do not edit `sase/memory/tui_perf.md` in these phases. Record a `PROPOSED FOLLOW-UP:`
  note on the verify bead instead.

Relevant `tui_perf.md` rules: route refreshes through the existing fast path (rule 5),
prefer selective updates over full rebuilds (rule 6), guard programmatic `OptionList`
updates (rule 12), and confirm the interactive TUI's imported SHA before measuring (rule
15).

## Phases

### Phase atomic-roster-publication: Publish one aggregate roster per disk apply

Make roster publication transactional: compose disk/cached/fleet rows with the latest
proc-observer projection _before_ fold filtering, query filtering, selection planning,
panel-key calculation, and the one finalize.

1. Reproduce first. Extend `tests/ace/tui/test_proc_shell_selection_survives_refresh.py`
   with a failing case: current roster has no proc shell, the effective proc projection
   contains an `epic` proc-shell row, the disk load is empty (`bounded_prefix=True`,
   `has_more=False`, `returned_count=0`). Record every `_finalize_agent_list` call.
   Assert the first and only finalize's roster contains the epic proc shell,
   `panel_keys_for` of that roster includes `"epic"`, and `len(finalize_calls) == 1`.
   Model the fake on `ProcShellFakeApp`; do not seed the proc row into
   `_agents_with_children` before the apply.
2. Capture the effective proc projection **and a monotonically increasing proc
   generation** on `PreparedApplySnapshot`. Bump the generation in
   `_apply_observed_snapshot` / whenever `_proc_projection` is replaced. Include the
   generation on `PreparedFinalizeStaleToken` so a moved projection discards a stale
   worker finalize plan.
3. In `prepare_loaded_agents_apply_boundary`, merge proc-shell agents from the
   _snapshot's projection_ (not only `cached_agents_with_children`) before slot
   annotation, fold filtering, and selection. Keep using `merge_proc_shell_agents` in
   `src/sase/ace/tui/models/agent_proc_shells.py`.
4. At the UI-thread commit in `_apply_loaded_agents_prepared_inner`, compare the live
   proc generation to the snapshot's. If it moved, rebase the prepared roster on
   `_effective_proc_projection()` before assigning app state. Then assign state and call
   `_finalize_agent_list()` **exactly once**.
5. Remove the post-finalize `_sync_proc_shell_agents_from_projection()` call from the
   disk-apply path. Observer events outside an apply keep the event-facing wrapper.
   While a disk apply is in flight (worker started, UI commit not yet done), observer
   updates must only replace `_proc_projection` and bump the generation — they must not
   finalize. The in-flight apply coalesces with them at commit.
6. Add a second failing-then-fixed test: proc generation moves between worker prep and
   UI commit; the published roster is the latest projection. Keep the existing
   unchanged-projection single-finalize test green.
7. Trace `proc_generation` and `proc_shell_count` on
   `agents.apply_loaded_agents_prepared` (and the finalize span if one exists) so later
   soaks can see one publication.

Do not merge incomplete-load semantics in this phase beyond what is required to compose
the proc projection. That is `removal-authority`.

### Phase tribe-stable-widgets: Key AgentList widgets by tribe and stop blanking untouched panels

This phase can land in parallel with `atomic-roster-publication`. It is the only fix for
sibling-tribe slot remapping, which no merge or apply change can address.

1. Reproduce first. Extend `tests/perf/test_agents_display_rebuild_guard.py` and the
   `_DisplayDiffApp` harness in `tests/ace/tui/_agent_display_diff_helpers.py`: render
   an `epic` row and a `review` row, then add and remove a `chop` row while the epic row
   stays put. Assert the epic widget **object identity** is unchanged, `update_list` is
   not called on it, and its option count never goes to 0. A second test: standing query
   `NOT machine:apollo`, `BY_STATUS`, one identity removed from a sibling tribe, zero
   `active_search` fallbacks, epic widget untouched.
2. Replace index-keyed ids with tribe-keyed ids. Add `panel_widget_id_for_key(key)`:
   - reserved default (`None`) → `agent-list-panel` (keep `_MAIN_PANEL_ID`)
   - named tribe → `agent-list-panel-{public_tribe_name(key)}` (for example
     `agent-list-panel-epic`) Keep `_panel_widget_id` as a test-compat wrapper that maps
     `self._panel_group.panel_keys[idx]` through the key helper, so existing callers
     that pass an index still get the tribe-stable id for that slot. Do **not** keep
     "whoever is first" as `agent-list-panel`; that is the current bug. Update
     production callers that meant "some AgentList" (`_startup_mount.py` loading flag,
     `_loading_apply.py` loading clear, `_panel_artifact_actions.py` focus,
     `src/sase/ace/testing/_startup.py`) to query `#agent-list-container`'s AgentList
     children or the default-panel id explicitly.
3. Insert or remove one panel without rebuilding siblings. When occupancy keys grow or
   shrink, `mount` or `remove()` only the affected `AgentList`, then reorder container
   children to match the canonical sort (expanded before collapsed, tribes
   alphabetical). Drop the `next_panel_keys != old_panel_keys ⇒ display_full_rebuild`
   shortcut in `_try_refresh_agents_display_incremental`. Record new display costs
   `display_panel_insert` and `display_panel_remove` on `AgentRefreshDisplayCost` in
   `src/sase/ace/tui/actions/agents/_refresh_trace.py`. Fall back to
   `panel_membership_change` only if the single-panel path cannot run.
4. Session-sticky occupancy for configured tribes. Once a tribe widget has been mounted
   this session, do not unmount it because rendered occupancy hit zero under the
   **same** committed query. Render the existing collapsed title strip at 0 rows (not an
   expanded empty list) unless the user has an explicit expand intent. Do not retire
   fold intents for those sticky keys. Apply `ace.tribes.*.initially_expanded` only on
   first mount of that widget in the session. Do **not** pre-mount configured tribes
   that have never appeared this session (`job` / `pinned` / `review` empty chrome the
   user never had); identity-stable ids plus session-sticky occupancy are what stop the
   `@epic` flash. A committed query _change_ may unmount tribes that the new query does
   not match.
5. Never `clear_options()` / `update_list` a widget whose visible row identities,
   grouping signatures, and collapsed state are unchanged. Title-only updates go through
   `_set_agent_panel_title`. `clear_options` is the frame of blank the user reads as
   "disappeared." Guard programmatic `OptionList` writes per tui_perf rule 12.
6. Delete the `if self._agent_search_query: return False` bail in
   `_try_remove_agent_rows` (finish sase-127.2). Row-remove already operates on
   post-filter visible lists, exactly like the display diff. Keep the fallback when the
   query _text_ changed (`search_query_changed`).
7. Update tests that hardcode `#agent-list-panel` / `#agent-list-panel-1` as "epic then
   review because alphabetical" to use `panel_widget_id_for_key`. Keep a default-panel
   test on `#agent-list-panel`. Visual snapshots: if sticky empty strips or widget-id
   migration change Agents-tab goldens, run targeted `just fix-tui-screenshots` and
   inspect the report; generation is not approval.

`panel_keys_for` occupancy semantics stay as they are (empty → `[None]`). Stickiness is
a widget-layer union with a session set, not a change to the panel-key model.

### Phase removal-authority: Stop incomplete bounded loads from replacing a larger cache

Depends on `atomic-roster-publication` because both edit the apply commit in
`_loading_apply.py`.

1. Reproduce first. Extend `tests/test_agents_tab_apply_boundary.py`: after a nonempty
   cached roster (two tribes including `epic`), apply a same-query bounded load with
   `returned_count=0`, `bounded_prefix=True`, `has_more=False`. Assert identities and
   `panel_keys_for` are unchanged and `_agents_seen_complete_history` stays true. A
   paired test: a _changed_ query key with a genuine empty result is allowed to empty
   the tab.
2. Widen `merge_incomplete_load_after_complete_history`: an incomplete load may replace
   the cache only when `load_state.complete_history` is true or it is an
   `artifact_delta` with explicit tombstones. `bounded_prefix` loads always patch by
   identity regardless of `has_more`. In particular, a same-query bounded zero over a
   nonempty cache keeps the cache, traces `empty_incomplete_apply_ignored`, and
   schedules one revalidated load. Do not apply this guard across a committed query
   change.
3. Clear `_agents_seen_complete_history` / `_agents_complete_history_query_key` only
   when the committed query key actually changed (the same `search_query_changed` /
   stale-query discard path). Incomplete loads with a missing or mismatched
   `history_query_key` must not reset the latch. Keep setting the latch on
   `complete_history=True` applies.
4. Converge `tier1_index_revalidate` with auto-refresh patching semantics. Add the
   sase-127.1 panel-key stability test for a revalidate-shaped load (not only
   `bounded_prefix=True, has_more=True`): 41 incoming identities over a 696-identity
   cache must not change the applied identity set or panel keys.
5. Arm `post_complete_incomplete_shrink` (and add a sibling invariant) so a
   nonempty→empty bounded transition is rejected even before a complete-history
   watermark, unless the committed query changed. A logical apply must not emit
   `N → 0 → N` visible rows or lose and regain a panel key. Trace `history_query_key`,
   snapshot authority (complete vs bounded vs delta), removal evidence, and published
   panel keys on the apply span.

Do not add a Python-only `snapshot_authoritative` wire flag that pretends the Rust index
grew a new field. If a later core change adds that flag, the merge guard here already
treats bounded loads as non-authoritative.

### Phase verify-on-athena: Prove panel stability on the live host with traces

1. Before measuring, confirm the interactive TUI's PID and imported SHA (sase-12p.2
   stale-code check). Two PIDs were writing load logs on 2026-09-19; a soak against the
   wrong process is not evidence. Restart onto the landed tree if the imported SHA is
   behind HEAD.
2. Soak on athena with the real persisted state (`by_status`, `NOT machine:apollo`, live
   churn) and `SASE_TUI_TRACE=1`. Capture recipes live in `docs/perf_runbook.md`.
   Minimum 30 minutes. Assert from traces, not observation:
   - no `2 → 1 (agents=0) → 2` panel sequences
   - `@epic`'s tribe-stable widget id is continuously present in
     `agents.refresh_panel_widgets` spans
   - `fallback_reason=active_search` is zero in steady state
   - `display_full_rebuild` on unchanged occupancy keys is zero
   - incoming revalidate size may still vary; _applied_ identity set and epic widget
     identity may not
   - one `agents.apply_loaded_agents_prepared` span does not pair with a second
     finalize/publication for the same load
3. Extend `tests/perf/test_agents_display_rebuild_guard.py` with any remaining
   cross-phase guards not already landed:
   - sibling-tribe insert/remove does not rebuild epic
   - nonempty standing query plus row remove stays incremental
   - empty incomplete apply does not unmount a session-sticky configured tribe
   - projection-only proc shell over empty disk still has `@epic` at the first finalize
4. Record before/after evidence in the phase notes (trace excerpts, widget-id longevity,
   rebuild counts and reasons, CPU if sampled). Record a `PROPOSED FOLLOW-UP:` note
   proposing:
   - a `tui_perf.md` addition for dual-publication, `has_more` ≠ removal authority, and
     tribe-stable widget identity (memory edits route through the memory-write
     procedure, not this epic)
   - Rust core hardening: one SQLite read transaction for candidate selection /
     machine-tree expansion / hydration, a logical index generation the Python cache
     validates against, and an explicit snapshot-authority flag
   - optional BY_DATE incremental support (remaining `unsupported_grouping` source; not
     this host's grouping)

Do not absorb sase-124 freshness/latency verification artifacts.

## Acceptance

- A disk apply whose cached roster lacks a proc shell, whose live projection contains an
  `@epic` proc shell, and whose disk load is empty, publishes `@epic` on the first and
  only finalize.
- Adding or removing a sibling tribe row while an `epic` row stays put leaves the epic
  `AgentList` object identity unchanged, does not call `update_list` / `clear_options`
  on it, and never drops its option count to 0.
- A same-query bounded zero over a nonempty cache keeps the cache and panel keys; a
  changed-query zero is allowed to empty the tab.
- Under `by_status` and `NOT machine:apollo`, a 30-minute athena soak against the
  process that imported the landed SHA shows no `N → 0 → N` publications and no unmount
  of `@epic`'s tribe-stable widget id.
- Steady-state traces show zero `active_search` fallbacks and zero full rebuilds on
  unchanged occupancy keys.
- `just check` is green for each implementation phase. The widgets phase inspects any
  Agents-tab visual golden updates it caused. Landing runs `just check-full` through
  `/sase_monitor`.

## Code map

| Path                                                           | Role                                         |
| -------------------------------------------------------------- | -------------------------------------------- |
| `src/sase/ace/tui/actions/agents/_loading_apply.py`            | Finalize-then-sync ordering; latch set/clear |
| `src/sase/ace/tui/actions/_proc_action_completion.py`          | Second finalize from proc-shell sync         |
| `src/sase/ace/tui/actions/agents/_loading_compute.py`          | Boundary merge; cached-roster proc carryover |
| `src/sase/ace/tui/actions/agents/_loading_compute_types.py`    | `PreparedApplySnapshot` fields               |
| `src/sase/ace/tui/actions/agents/_loading_compute_finalize.py` | Finalize stale token                         |
| `src/sase/ace/tui/actions/agents/_loading_compute_merge.py`    | Incomplete-load patch vs replace             |
| `src/sase/ace/tui/models/agent_proc_shells.py`                 | `merge_proc_shell_agents`                    |
| `src/sase/ace/tui/actions/agents/_display_helpers.py`          | Index-based `panel_widget_id`                |
| `src/sase/ace/tui/actions/agents/_display_panel_widgets.py`    | Mount/unmount by index; `update_list`        |
| `src/sase/ace/tui/actions/agents/_display.py`                  | Incremental vs full rebuild                  |
| `src/sase/ace/tui/actions/agents/_display_panel_patches.py`    | `active_search` row-remove bail              |
| `src/sase/ace/tui/actions/agents/_display_panel_layout.py`     | Focus/layout by slot index                   |
| `src/sase/ace/tui/actions/agents/_display_panel_collection.py` | Fold-intent retirement on unmount            |
| `src/sase/ace/tui/actions/agents/_panel_fold_intent.py`        | `retire_panel_fold_intents`                  |
| `src/sase/ace/tui/models/agent_panels.py`                      | `panel_keys_for`; empty → `[None]`           |
| `src/sase/ace/tui/widgets/_agent_list_build_rebuild.py`        | `clear_options()` at every `update_list`     |
| `src/sase/ace/tui/actions/agents/_refresh_trace.py`            | Display costs and fallback reasons           |
| `src/sase/ace/tui/repro/invariants.py`                         | `post_complete_incomplete_shrink`            |
| `tests/ace/tui/test_proc_shell_selection_survives_refresh.py`  | Proc-shell apply coverage                    |
| `tests/test_agents_tab_apply_boundary.py`                      | Bounded-prefix merge coverage                |
| `tests/perf/test_agents_display_rebuild_guard.py`              | Don't-rebuild `@epic` guards                 |
