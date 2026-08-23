---
tier: tale
title: Keep a focused stand-alone proc shell selected across Agents-tab refreshes
goal:
  A stand-alone `%proc` shell row focused in the Agents tab stays focused across every
  auto-refresh, and each refresh runs one finalize pass instead of two.
size: small
proposed_by: bbugyi200.athena.0bx
---

- **AGENTS:**
  - [bbugyi200.athena.0bx](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0bx.md)
- **COMMITS:**
  - [6eb51ac](https://github.com/sase-org/sase/commit/6eb51ac49cc08f23ec240f9b73ed059788395a5c)
    — fix(ace): preserve proc shell selection on refresh

# Plan: Keep a focused stand-alone proc shell selected across Agents-tab refreshes

## Symptom

Focus a stand-alone `%proc` shell row (`AgentType.PROC_SHELL`) in the Agents tab and
wait for the TUI auto-refresh. The cursor silently jumps off the proc row onto a
neighboring sase agent node. The proc row itself is still there afterwards — only the
selection moved.

## Root cause (verified)

The disk loader never produces proc-shell rows. They exist only as a presentation
projection of the proc observer, built by `proc_shell_agents_from_observed` and merged
into the roster by `_sync_proc_shell_agents_from_projection`
(`src/sase/ace/tui/actions/_proc_action_completion.py:132`).

`feat(ace): surface proc shells in agents tab` (`a6a184fad`) wired that merge in at the
**end** of the loader apply —
`src/sase/ace/tui/actions/agents/_loading_apply.py:440-444`. So one ordinary refresh
does this, in order:

1. `_apply_loaded_agents_prepared_inner` installs the loader payload:
   `self._agents_with_children = boundary.fold.unfiltered_agents` and
   `self._agents = boundary.fold.visible_agents` (`_loading_apply.py:393-394`). Neither
   list contains any proc-shell row — they were dropped.
2. `self._finalize_agent_list(...)` (`_loading_apply.py:410`) runs selection restore
   against that proc-less roster. `restore_selection_by_identity`
   (`src/sase/ace/tui/util/selection.py`) cannot find the focused proc shell's identity,
   so it falls through to the nearest-neighbor rule and clamps `prior_visual_row` into
   the shorter list. `current_idx` now points at a disk-loaded sase agent.
3. Only then does `sync_proc_shells()` run (`_loading_apply.py:444`). It re-merges the
   proc rows and calls `_finalize_agent_list` a second time — but it captures
   `selected_identity` from `previous_agents[current_idx]`, and `current_idx` was
   already clobbered in step 2. The wrong row is therefore **pinned**, not recovered.

Two things follow from this:

- The jump happens on essentially every refresh, not just occasionally.
  `merge_incomplete_load_after_complete_history`
  (`src/sase/ace/tui/actions/agents/_loading_compute_merge.py:191`) does carry cached
  rows — including proc shells — forward, but it returns early unless the load is
  incomplete-history **and** (`agents_seen_complete_history` or an artifact-delta
  source). An ordinary Tier 1 indexed refresh (`complete_history=False`,
  `artifact_source="artifact_index"`) on a session that has never done a full-history
  scan takes that early return, so the proc rows are dropped.
- Every refresh pays for two full finalize + display passes whenever any stand-alone
  proc exists.

The row the cursor lands on is `min(prior_idx, len(agents_without_procs) - 1)`. Because
`merge_proc_shell_agents` appends proc rows after the disk-loaded rows, a focused proc
shell usually clamps onto the **last** disk-loaded agent row — which matches the
reported "the `0bh` / `0bd` sase agent node gets selected".

### Reproduced

Driving `_apply_loaded_agents_prepared` against a `FakeAgentApp` seeded with two disk
agents plus one observed proc shell, with the proc row focused:

```
roster after observer merge: [('RUNNING','alpha',...), ('RUNNING','beta',...), ('PROC_SHELL','sase','0123456789ab')]
selected before refresh: (PROC_SHELL, 'sase', '0123456789ab')
roster after refresh:   [('RUNNING','alpha',...), ('RUNNING','beta',...), ('PROC_SHELL','sase','0123456789ab')]
selected after refresh: (RUNNING, 'beta', ...)        <-- jumped
finalize calls per apply: 2
```

Pre-merging the observer's proc rows into `prep.filtered_agents` before the boundary
runs makes the same scenario print
`selected after refresh: (PROC_SHELL, 'sase', '0123456789ab')` and
`finalize calls per apply: 1`.

## Fix

Carry the observer-owned proc-shell rows into the loader payload **before** fold
filtering and selection restore, instead of stapling them on afterwards.

`prepare_loaded_agents_apply_boundary`
(`src/sase/ace/tui/actions/agents/_loading_compute.py:166`) is the single choke point:
the synchronous apply path calls it with `merge_incomplete=True`, and the async worker
path reaches it through `prepare_loaded_agents_worker_boundary` with
`merge_incomplete=False`. It already receives a `PreparedApplySnapshot` whose
`cached_agents_with_children` field is a copy of the live `_agents_with_children` —
which is exactly where the currently displayed proc rows live. No new snapshot field, no
new I/O, and no proc-shell `Agent` construction on the loader path are required.

### Step 1 — carry proc-shell rows through the apply boundary

In `src/sase/ace/tui/actions/agents/_loading_compute.py`, inside
`prepare_loaded_agents_apply_boundary`, after the `merge_incomplete` block and
**before** `refresh_runner_slot_context` /
`unfiltered_agents = list(prep.filtered_agents)`:

```python
proc_shells = [a for a in snapshot.cached_agents_with_children if a.is_proc_shell]
if proc_shells:
    prep.filtered_agents = merge_proc_shell_agents(prep.filtered_agents, proc_shells)
    prep.has_always_visible = True
```

Use `merge_proc_shell_agents` from `sase.ace.tui.models.agent_proc_shells` (import it
the same lazy way the surrounding code imports model helpers). It strips every existing
`is_proc_shell` row before appending, so the step is idempotent and also collapses any
duplicate the Tier 1 cached merge already carried forward. The resulting order (disk
rows first, proc rows appended) is byte-for-byte the order the post-apply merge produces
today, so nothing about rendered row order changes.

Do **not** recompute `prep.hideable_agents` or `prep.hidden_count` from
`prep.filtered_agents`: when `hide_non_run_agents` is on,
`merge_incomplete_load_after_complete_history` deliberately leaves `hideable_agents`
holding rows that are absent from `filtered_agents`. Proc-shell rows are never
`is_workflow_child` and never carry `hidden=True`, so `is_always_visible` is always true
for them — `hideable_agents` and `hidden_count` are correct untouched, and only
`has_always_visible` needs the update above. Write a short comment stating that
invariant so the next reader does not "fix" it.

Add a focused docstring/comment on the new block explaining that the disk loader has no
proc-shell source, that these rows are owned by the proc observer, and that they must be
present before selection restore runs.

### Step 2 — leave the post-apply reconcile in place, and confirm it now short-circuits

Keep the `sync_proc_shells()` call at `_loading_apply.py:440-444`. It is still the path
that picks up genuinely new proc state (status transitions, newly launched procs,
settled procs pruned from the projection). After step 1 it early-returns via the
`proc_shell_agent_signature` equality check in `_sync_proc_shell_agents_from_projection`
whenever the projection has not changed, which is what removes the duplicate finalize
pass. Assert that with a test rather than assuming it.

### Step 3 — check proc-shell identity stability across the observer handover

`Agent.identity` is `(agent_type, cl_name, raw_suffix)`. For a proc shell `raw_suffix`
is the proc id, but `cl_name` comes from `row.project or row.cl_name or "proc"`
(`src/sase/ace/tui/models/agent_proc_shells.py`, `_observed_proc_to_agent`).
Session-local overlay rows (`_session_overlay_rows` in
`src/sase/ace/tui/actions/_proc_action_observer.py`) are replaced by durable observer
rows once the store sees the same `proc_id` — `compose_proc_projection` lets the durable
row win. If those two rows disagree on `project`, the proc-shell row's identity flips
mid-life and selection is lost again for a different reason.

Verify whether that flip actually occurs for a TUI-submitted `%proc`. If it does, make
the proc-shell identity depend only on the proc id (the natural place is the
`Agent.identity` property's existing special-case branch in
`src/sase/ace/tui/models/agent.py`, alongside the clan-container case) and cover it with
a test. If it does not, record that in a test that pins `_observed_proc_to_agent`'s
identity across an overlay→durable transition, so a future change cannot silently
introduce the flip. Do not change `Agent.identity` on speculation — identity keys
dismissals, marks, and status overrides.

## Tests

Keep new modules inside the `toobig` budget; the recent house convention is to split
test modules before they reach ~500 lines.

### New module: `tests/ace/tui/test_proc_shell_selection_survives_refresh.py`

Harness (verified to work headlessly, no Textual mount required): subclass
`ProcCompletionActionsMixin` (`sase.ace.tui.actions._proc_action_completion`) together
with `FakeAgentApp` from `tests/_agents_tab_query_helpers.py`, set
`current_tab = "agents"`, and provide `_proc_projection` (a `ProcProjection` of
`ObservedProc` rows with `lifecycle=PROC_LIFECYCLE_PROC_SHELL` and
`origin=XPROMPT_PROC_ORIGIN`), plus no-op `_update_proc_indicator` and
`_invalidate_agent_panel_cache`, and empty `_session_completion_callbacks` /
`_proc_completion_callbacks` dicts. Seed the roster with
`_sync_proc_shell_agents_from_projection()`, then drive one refresh with
`compute_apply_loaded_agents(...)` +
`_apply_loaded_agents_prepared(..., load_state=AgentLoadState(tier="tier1", complete_history=False, complete_visible_inbox=True, artifact_source="artifact_index", used_artifact_index=True, ...))`.
Pin `project_display_name_for` the way
`tests/ace/tui/visual/_ace_agents_proc_shell_png_fixtures.py` does if any config I/O
shows up.

1. `test_focused_proc_shell_keeps_selection_across_loader_apply` — the repro above:
   after the apply, `app._agents[app.current_idx].identity` still equals the proc
   shell's identity. This test must fail on `master`.
2. `test_loader_apply_keeps_proc_shell_rows_in_roster` — the proc row is present in both
   `_agents` and `_agents_with_children` at the moment `_finalize_agent_list` runs, not
   just after the trailing reconcile. Capture roster state from inside a patched
   `_finalize_agent_list` (or a patched `_sync_proc_shell_agents_from_projection` that
   records and then delegates) so the assertion cannot be satisfied by the late merge.
3. `test_unchanged_proc_projection_runs_one_finalize_pass` — counting wrapper around
   `_finalize_agent_list` sees exactly one call per apply. On `master` this is 2.
4. `test_off_tab_refresh_preserves_saved_proc_shell_selection` — with
   `current_tab != "agents"` and `_agents_last_identity` set to the proc shell, the
   apply leaves `_agents_last_identity` on the proc shell and `_agents_last_idx`
   pointing at it.
5. `test_settled_proc_removed_from_projection_leaves_roster` — drop the proc row from
   the projection and confirm the apply plus trailing reconcile removes it (step 1 must
   not resurrect stale procs).

### Extend `tests/test_agents_tab_apply_boundary.py`

6. A pure-boundary test: `prepare_loaded_agents_apply_boundary` with a
   `PreparedApplySnapshot` whose `cached_agents_with_children` holds one proc-shell row
   returns that row in `boundary.fold.unfiltered_agents` and
   `boundary.fold.visible_agents`, for **both** `merge_incomplete=True` and
   `merge_incomplete=False`, and never duplicates it when the incoming payload already
   carries one.
7. A guard that a proc-shell row does not change `runner_capacity`
   (`refresh_runner_slot_context` must not treat it as an occupied runner lane — it is
   excluded today because `_is_ace_run_root` is false for it, and that must stay true
   now that proc rows reach the boundary).

### Step 3 test

8. Whatever step 3 concludes: either an identity-stability test for the overlay→durable
   handover, or a regression test pinning the current identity derivation.

## Verification

- `just install` first (workspace virtualenvs are ephemeral).
- `just check` for the lint gates plus the diff-scoped test lane. `_lint-toobig`,
  `_lint-symvision`, ruff, and mypy all run there.
- `just check-full` before landing, through `/sase_monitor` — never inline:

  ```
  sase monitor start --command 'just check-full' \
    --start-status TESTING --stop-status TESTED --next '<follow-up action>'
  ```

- Manual smoke: launch a stand-alone `%proc`, focus its row in the Agents tab, and idle
  through several auto-refresh ticks (including one past the 60s full sanity refresh).
  The cursor must not move.

## Out of scope

- Making the disk loader itself aware of procs, or persisting proc-shell rows as agent
  artifacts. The observer stays the single source of truth for proc-shell content; this
  plan only fixes _when_ its rows enter the roster.
- Adding proc-shell state to `make_finalize_stale_token`. A projection change between
  the worker snapshot and the apply is already reconciled by the trailing
  `sync_proc_shells()` pass; widening the stale token would discard otherwise-valid
  precomputed finalize plans. Note it as a known, accepted gap.
- Any change to grouping, rendering, or the Procs pane.

## Risks

- **Proc rows now flow through the pure worker boundary.** They reach
  `refresh_runner_slot_context`, `filter_agents_by_fold_state`, the content-search
  index, and the query filter one step earlier than before. Each is already exercised on
  these rows today (the post-apply reconcile's `_finalize_agent_list` call applies fold
  filtering and the query filter to them), and `_is_ace_run_root` excludes them from
  runner lanes — test 7 pins that. Watch for any panel-count or capacity-chip drift in
  `just check-full`.
- **Stale proc rows.** Step 1 replays the last-known proc rows from cache. If a proc
  settled and was pruned between the observer's last snapshot and this apply, the row
  reappears for the duration of the apply and is then removed by the trailing reconcile
  in the same UI callback — no user-visible flash. Test 5 pins this.
