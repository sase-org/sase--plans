---
tier: tale
title: Make terminal stand-alone proc-shell rows dismissable on the Agents tab
goal:
  A finished stand-alone proc shell can be dismissed from the Agents tab with `x` and
  with the cleanup panel's dismiss-done actions, the row stays gone across observer
  ticks, disk reloads, and ACE restarts, and every dismiss affordance that counts a
  proc-shell row also acts on it.
size: medium
proposed_by: bbugyi200.athena.0c7
create_time: 2026-08-24 07:02:38
status: wip
---

# Plan: Make terminal stand-alone proc-shell rows dismissable

## Problem

A stand-alone `%proc` proc shell that has finished cannot be removed from the Agents
tab. Selecting the terminal row (for example `unit-1 (DONE) [bash]`) and pressing `x`
raises the warning toast `Proc shell has already finished` and nothing happens. The row
stays in the Agents list until the durable proc store's retention
(`procs.history_limit`, default `100`) eventually ages the underlying proc row out,
which can take days of normal use. Meanwhile the footer and the cleanup panel both
_count_ that row as done, so the user is told there is cleanup available that no action
can actually perform.

Reproduce:

1. Launch a stand-alone proc shell (an xprompt `%proc` unit, i.e. a proc-store row with
   `lifecycle == "proc-shell"` and `origin == "xprompt-proc"`), e.g. one running
   `echo hello && sleep 30 && echo world`.
2. Wait for it to reach `DONE`.
3. On the Agents tab, select the `▣` proc-shell row and press `x`.
4. Observe the `Proc shell has already finished` warning; the row remains.
5. Press `X` and choose a dismiss-done action. The count includes the proc shell and the
   toast claims it was dismissed, but the row reappears within ~0.5s.

## Root cause

There are three distinct defects. All three must be fixed for dismissal to work; fixing
only the first produces a row that visibly reappears.

### Defect 1 — the kill dispatcher has no terminal branch for proc shells

`src/sase/ace/tui/actions/agents/_kill_action_flow.py` `action_kill_agent()` routes
_every_ proc-shell row into the kill flow:

```python
if agent.is_monitor and agent.monitor_state == "running":   # guarded
    self._handle_monitor_stop_action(agent)
    return

if agent.is_proc_shell:                                     # NOT guarded
    self._handle_proc_shell_kill_action(agent)
    return
```

The monitor branch immediately above it is guarded on `monitor_state == "running"`, so a
finished monitor falls through to the ordinary cleanup-plan/dismiss path. The proc-shell
branch has no such guard, so a terminal proc shell reaches
`MonitorStopActionFlowMixin._handle_proc_shell_kill_action`
(`src/sase/ace/tui/actions/agents/_monitor_stop_flow.py`), whose first statement is:

```python
if agent.proc_status not in ACTIVE_PROC_STATUSES:
    self.notify("Proc shell has already finished", severity="warning")
    return
```

That `return` is the dead end the user hits. There is no other code path in the app that
can remove a proc-shell row.

### Defect 2 — the proc-shell projection is re-merged unconditionally

Stand-alone proc-shell rows are not loaded from disk like agents; they are a projection
of proc-store rows held by the proc observer.
`ProcCompletionActionsMixin._sync_proc_shell_agents_from_projection`
(`src/sase/ace/tui/actions/_proc_action_completion.py`) rebuilds them on every observer
snapshot (poll interval `POLL_SECONDS = 0.5`) via
`proc_shell_agents_from_observed(projection.rows)` and then calls
`merge_proc_shell_agents`, which replaces _all_ proc-shell rows in the roster:

```python
return [
    *(agent for agent in agents if not agent.is_proc_shell),
    *proc_shells,
]
```

Nothing in that path consults the dismissed set. The same is true of the loader boundary
in `prepare_loaded_agents_apply_boundary`
(`src/sase/ace/tui/actions/agents/_loading_compute.py`), which carries roster proc-shell
rows across a disk refresh _after_ `compute_apply_loaded_agents` applied the
dismissed-identity filter — the disk loader never sees proc shells, so the existing
filter can never apply to them. Any in-memory removal is therefore undone on the next
observer tick.

### Defect 3 — bulk dismiss paths count proc shells but cannot dismiss them

The bulk paths disagree with each other:

- `agent_panel_index` (`src/sase/ace/tui/models/agent_panel_index.py`) counts any row
  whose status is in `DISMISSABLE_STATUSES` toward `completed_count`. A terminal proc
  shell's status is `DONE` / `FAILED` / `STOPPED`, so it is counted. That count drives
  the footer's `X cleanup (N done)` hint and the cleanup panel's counts
  (`_build_agent_cleanup_panel_state` in
  `src/sase/ace/tui/actions/agents/_kill_cleanup_panel.py`).
- `_dismiss_done_agents_from` (`src/sase/ace/tui/actions/agents/_dismissing.py`), which
  backs the panel's `dismiss_panel_done` / `dismiss_all_done` actions, filters only on
  `status in DISMISSABLE_STATUSES and raw_suffix is not None` — a terminal proc shell
  passes. It is then run through the full _agent_ dismissal machinery
  (`plan_dismissal_side_effects` → the Rust cleanup planner, dismissed-bundle save,
  artifact deletion, `dismissed_agents.json` persistence, artifact-index sync,
  same-session revive pool via `_apply_dismissal_in_memory`). None of that is meaningful
  for a proc shell, and because of Defect 2 the row returns anyway.
- `_present_bulk_kill_modal` (`src/sase/ace/tui/actions/agents/_marking_kill.py`), used
  by marked-set `x`, group `x`, clan `x`, and panel `x`, also does not filter proc
  shells; a marked terminal proc shell lands in its `dismissable` bucket (`pid is None`)
  and takes the same ineffective agent path.
- By contrast `_agent_cleanup_targets_from_candidates`
  (`src/sase/ace/tui/actions/agents/_kill_cleanup_panel.py`) and
  `_present_custom_cleanup`
  (`src/sase/ace/tui/actions/agents/_kill_cleanup_selection.py`) explicitly
  `continue`/`return` on `agent.is_proc_shell`, dropping them from the planner-backed
  scopes.

So the same row is simultaneously counted as cleanable, silently skipped by some bulk
scopes, and mis-dismissed by others.

### Secondary — no affordance is offered

`_compute_agent_bindings` (`src/sase/ace/tui/widgets/_keybinding_bindings.py`) only
emits an `x` hint for a proc shell when `agent.proc_status in ACTIVE_PROC_STATUSES`, so
a terminal proc-shell row shows no `x` action at all — while `app.kill_agent` still
reports available for it in `src/sase/ace/tui/commands/availability.py`. The footer is
consistent with "nothing to do here", which is exactly the behaviour to change.

## Design decision

**Dismissal of a proc-shell row is host-side ACE inbox state, keyed by native proc id,
persisted in its own small store — it does not mutate the durable proc store.**

Rationale:

- A dismissed proc-shell row must stay visible in the Procs pane and in `sase proc list`
  / `sase proc show`. The Procs pane deliberately has no per-row dismissal
  (`action_dismiss_task` in `src/sase/ace/tui/modals/procs_pane_actions.py` says
  "Durable procs age out with procs.history_limit"). Dismissal here means "clear it from
  my Agents inbox", exactly as it does for agents, not "forget this proc".
- This mirrors the existing precedent: dismissed _agents_ are tracked host-side in
  `~/.sase/dismissed_agents.json` by `src/sase/ace/dismissed_agents.py`, not in the Rust
  core.
- Rejected alternative: add a `dismissed_at` field to the durable proc row in the Rust
  core (`sase-core/crates/sase_core/src/procs/wire.rs`). That requires a
  `PROC_WIRE_SCHEMA_VERSION` bump (currently `3`), a new store mutation, new bindings, a
  new `sase proc dismiss` CLI verb, and it changes what every proc consumer sees — a
  much larger, cross-repo change that also gives the Procs pane a dismissal concept it
  intentionally does not have. Do not do this.
- Rejected alternative: reuse `_dismissed_agents` / `dismissed_agents.json` with
  `AgentType.PROC_SHELL` identities. It drags in dismissed-bundle saving, artifact
  deletion, artifact-index projection sync, and the rewind/revive pool — all of which
  are wrong for a row that has no artifacts dir and no bundle — and each would need its
  own proc-shell guard.

Bounding the new store: proc ids are never reused, so the set must be pruned. The
durable proc store is already bounded by `procs.history_limit`, so pruning the dismissed
set down to ids still present in the proc store is a natural and sufficient bound. Prune
once per ACE start, off the paint path.

## Implementation

### Step 1 — new persistence module

Add `src/sase/ace/dismissed_proc_shells.py`, a small module in the same package as
`dismissed_agents.py`:

- `dismissed_proc_shells_file() -> Path` → `sase_home() / "dismissed_proc_shells.json"`
  (`sase_home` from `sase.core.paths`), overridable by a module-level
  `_DISMISSED_PROC_SHELLS_FILE` hook so tests can redirect it, matching the
  `_DISMISSED_AGENTS_FILE` pattern in `dismissed_agents.py`.
- `load_dismissed_proc_shells() -> set[str]` — tolerant reader. On missing file,
  `OSError`, or malformed JSON return an empty set; ignore non-string entries. Accept
  both the canonical object form `{"schema_version": 1, "proc_ids": [...]}` and a bare
  `[...]` list.
- `record_dismissed_proc_shells(proc_ids: Collection[str], *, live_proc_ids: Collection[str] | None = None) -> bool`
  — read the current file, union in `proc_ids`, drop ids absent from `live_proc_ids`
  when it is supplied, and write the result atomically with
  `sase.ace.dismissed_agents_bundles.write_json_file_atomic`. Read-modify-write (rather
  than writing the caller's whole in-memory set) so two concurrently running ACE
  instances cannot clobber each other's dismissals. Return `False` on write failure
  rather than raising.
- `prune_dismissed_proc_shells(live_proc_ids: Collection[str]) -> set[str]` — intersect
  the persisted set with `live_proc_ids`, write back only when it shrank, and return the
  pruned set.

Sort ids on write so the file is stable and diffable. Keep the whole module free of
Textual and `Agent` imports so it is trivially unit-testable.

### Step 2 — filter the projection at its one chokepoint

`proc_shell_agents_from_observed` in `src/sase/ace/tui/models/agent_proc_shells.py` is
the only place an `ObservedProc` becomes a proc-shell `Agent`. Give it a keyword-only
`dismissed_proc_ids: Collection[str] = ()` and skip any row whose `proc_id` is in that
set, alongside the existing `_is_standalone_xprompt_row` check.

Update the one production caller, `_sync_proc_shell_agents_from_projection` in
`src/sase/ace/tui/actions/_proc_action_completion.py`, to pass
`dismissed_proc_ids=self._dismissed_proc_shells` (use
`getattr(self, "_dismissed_proc_shells", ())` so the existing test doubles that do not
define it keep working).

This is sufficient for the loader boundary too: `prepare_loaded_agents_apply_boundary`
carries forward the _already-filtered_ roster rows, so no change is needed there. Add a
regression test rather than a second filter.

Note the early-return in `_sync_proc_shell_agents_from_projection` compares
`proc_shell_agent_signature(current_proc_shells)` against the freshly projected list.
Once the projection is filtered, the post-dismiss roster and the post-dismiss projection
agree, so the sync short-circuits and no re-merge occurs. Verify this in the test.

### Step 3 — app state and startup prune

- In `src/sase/ace/tui/actions/_state_init_agents.py`, next to the existing
  `self._dismissed_agents = load_dismissed_agents()` block, add
  `self._dismissed_proc_shells = load_dismissed_proc_shells()`. This is a single small
  JSON read; do **not** prune here — `_init_app_state` runs before first paint.
- In `src/sase/ace/tui/actions/_startup_loads.py`, add a startup prune modelled on the
  existing `_schedule_dismissed_index_startup_sync` /
  `_run_dismissed_index_startup_sync` pair: schedule it with `run_worker(...)` in the
  `"startup-loads"` group, and inside it `await asyncio.to_thread(...)` a worker that
  calls `read_procs()` (from `sase.procs`) for the live id set and then
  `prune_dismissed_proc_shells(live_ids)`. Fold the pruned set back into
  `self._dismissed_proc_shells` on the UI thread. Swallow and log exceptions — a prune
  failure must never break startup.

### Step 4 — dismiss one proc-shell row

Add the dismissal to `MonitorStopActionFlowMixin`
(`src/sase/ace/tui/actions/agents/_monitor_stop_flow.py`), next to the existing
proc-shell kill flow, or to a new focused sibling module if that file grows past the
repo's size norms:

```python
def _dismiss_proc_shell_rows(self, agents: list[Agent]) -> None:
```

Behaviour:

- Filter to rows with `is_proc_shell`, a resolvable `proc_id`, and
  `proc_status not in ACTIVE_PROC_STATUSES`. If nothing remains, notify
  `No finished proc shells to dismiss` (warning) and return.
- No confirmation modal. Dismissing a done agent (`_dismiss_done_agent`) does not
  confirm either, and nothing is destroyed — the proc row and its log stay put.
- Update `self._dismissed_proc_shells` immediately (optimistic), then remove the rows
  from the roster without touching any agent-dismissal machinery:

  ```python
  prior_pos = self._capture_focused_visible_pos()
  removed = {a.identity for a in targets}
  fast_path = hasattr(self, "_try_remove_agent_rows") and self._try_remove_agent_rows(removed)
  self._agents_with_children = [a for a in self._agents_with_children if a.identity not in removed]
  if fast_path:
      self._apply_dismissal_in_memory_fast_finish(removed, prior_pos=prior_pos)
  else:
      self._refilter_agents(prior_pos=prior_pos)
  ```

  Deliberately **not** used: `_apply_dismissal_in_memory` (it appends to
  `_dismissed_agent_objects`, the same-session rewind pool, and runs the clan/workflow
  cascade), `plan_dismissal_side_effects`, `save_dismissed_bundle`,
  `delete_agent_artifacts`, `sync_dismissed_agent_artifact_index`, and
  `dismiss_notifications_for_agents`. A proc shell has no artifacts dir, no bundle, no
  notifications, and is not revivable.

- Toast via the existing `_notify_after_refresh` helper: `Dismissed proc shell <label>`
  for one row, `Dismissed N proc shells` for several, where `<label>` is
  `proc_label or agent_name or short_proc_id(proc_id)` (the same label the kill flow
  uses).
- Persist off the paint path: `run_worker` / `asyncio.to_thread` calling
  `record_dismissed_proc_shells(ids)`. Notify at `severity="warning"` if it returns
  `False`, since the row will come back at next restart.

### Step 5 — wire the dispatch

In `src/sase/ace/tui/actions/agents/_kill_action_flow.py` `action_kill_agent()`, replace
the unguarded proc-shell branch with:

```python
if agent.is_proc_shell:
    if agent.proc_status in ACTIVE_PROC_STATUSES:
        self._handle_proc_shell_kill_action(agent)
    else:
        self._dismiss_proc_shell_rows([agent])
    return
```

Leave `_handle_proc_shell_kill_action`'s own `ACTIVE_PROC_STATUSES` guard and its
warning in place — it is still correct as a defensive check for any other caller.

### Step 6 — bulk paths

- `_dismiss_done_agents_from` (`src/sase/ace/tui/actions/agents/_dismissing.py`):
  partition `dismissable` into proc shells and agents. Build the confirmation
  description from the agent subset via `confirmation_sase_agent_summary` as today, and
  append a `Dismiss N proc shell(s)` line when proc shells are present. On confirm, call
  `_do_dismiss_all(agents)` only when `agents` is non-empty, and
  `_dismiss_proc_shell_rows(proc_shells)` when proc shells are present. When only proc
  shells are selected, skip the agent path entirely rather than calling it with an empty
  list.
- `_present_bulk_kill_modal` (`src/sase/ace/tui/actions/agents/_marking_kill.py`):
  partition proc shells out of `agents` before computing `killable` / `dismissable`.
  Terminal proc shells become a third `proc_dismissable` bucket described in the modal;
  _active_ proc shells are excluded from bulk scopes entirely (bulk kill has no proc
  kill path) — say so in the description, e.g. `Skipping N running proc shell(s)`. On
  confirm, run the existing `on_confirm` / `_do_bulk_kill_agents` for the agent buckets
  and `_dismiss_proc_shell_rows` for `proc_dismissable`. This single change covers
  marked-set `x`, group `x`, and clan-container `x`.
- `_bulk_kill_panel_agents` (`src/sase/ace/tui/actions/agents/_kill_action_flow.py`)
  builds its targets with `_agent_cleanup_targets_from_candidates`, which drops proc
  shells. Collect the panel's terminal proc shells separately from `panel_agents` and
  pass them through to `_present_bulk_kill_modal` so a panel that contains only proc
  shells no longer reports `No agents remain in panel`.
- Leave `_agent_cleanup_targets_from_candidates` and `_present_custom_cleanup` skipping
  proc shells for the _planner-backed_ scopes (tribe cleanup and custom selection go
  through `plan_agent_cleanup` in the Rust core, which has no proc-shell concept). Keep
  a short comment saying so. Extending the Rust cleanup planner is out of scope.
- No change is needed to `agent_panel_index.completed_count` or the cleanup panel's
  count helpers: once dismissal works, counting terminal proc shells as "done" becomes
  correct.

### Step 7 — footer and command availability

- `src/sase/ace/tui/widgets/_keybinding_bindings.py`: in the `is_proc_shell` branch,
  emit `(x, "kill proc")` for active rows as today and `(x, "dismiss proc")` for
  terminal rows (same `marked_count == 0 and not panel_focused and not group_focused`
  guard).
- `src/sase/ace/tui/commands/availability.py`: `app.kill_agent` already resolves to
  available for a focused proc-shell row; confirm no change is needed and add a test
  pinning it, so a terminal proc shell is reachable from the command palette.

### Step 8 — docs

Update the stand-alone `%proc` paragraph in `docs/ace.md` (the one beginning "A
stand-alone `%proc` unit (beta, behind `typed_launch_units`)", near the `▣` glyph table
entries) to state that `x` kills a running proc shell and dismisses a finished one, that
dismissal only clears the Agents-tab row and leaves the proc visible in the Procs pane
and `sase proc show`, and that a dismissed row does not come back. If the Procs-pane
section claims the Agents tab has no dismissal for procs, correct it too.

## Testing

Add tests alongside the existing proc-shell suites.
`tests/ace/tui/test_proc_shell_selection_survives_refresh.py` already provides the
`ProcShellFakeApp` harness (a `ProcCompletionActionsMixin` + `FakeAgentApp` composition
with `_seed_roster_with_proc_shell` and `_observed_proc` helpers) — reuse it.

1. **Dispatch (Defect 1).** `x` on a terminal proc-shell row calls the dismissal and
   never emits `Proc shell has already finished`; `x` on a `running` proc-shell row
   still pushes `ConfirmKillProcShellModal`. Model the assertion on
   `tests/test_agent_monitor_stop_action.py`, which already asserts
   `("Monitor has already finished", "warning") not in app._notifications`.
2. **Stickiness (Defect 2) — the key regression.** Seed a roster with a terminal proc
   shell, dismiss it, then re-run `_sync_proc_shell_agents_from_projection()` with the
   _unchanged_ projection and assert the row does not return to `_agents` or
   `_agents_with_children`. Then run a full disk-refresh cycle
   (`compute_apply_loaded_agents` + `prepare_loaded_agents_apply_boundary`, as
   `_apply_disk_refresh` in that test module already does) and assert it is still gone.
3. **Persistence round trip.** `record_dismissed_proc_shells` then
   `load_dismissed_proc_shells` returns the ids; a second writer's ids are preserved
   (read-modify-write, not clobber); a malformed/absent file yields an empty set and no
   exception; a write failure returns `False`.
4. **Prune.** `prune_dismissed_proc_shells` drops ids absent from the live proc-id set,
   keeps the rest, and does not rewrite the file when nothing changed.
5. **Bulk (Defect 3).** With a roster of one done agent plus one terminal proc shell,
   the cleanup panel's `dismiss_all_done` dismisses both, and the proc shell path saves
   no dismissed bundle, deletes no artifacts, and adds nothing to
   `_dismissed_agent_objects`. Same assertion for marked-set `x` through
   `_present_bulk_kill_modal`. Also assert an _active_ proc shell in a marked set is
   skipped rather than dismissed.
6. **Footer.** Extend `tests/test_keybinding_footer_agent.py` (it already has proc-shell
   footer cases) with a terminal proc-shell row showing `x dismiss proc`.
7. **Command availability.** Pin `app.kill_agent` as available for a focused terminal
   proc-shell row.

Run the PNG snapshot suite as well —
`tests/ace/tui/visual/test_ace_png_snapshots_agents_proc_shells.py` renders proc-shell
rows and may capture the footer:

```bash
just test-visual
```

Accept intentional changes with `--sase-update-visual-snapshots` only after inspecting
the diff artifacts in `.pytest_cache/sase-visual/`.

## Verification

This is an ephemeral numbered workspace, so install first:

```bash
just install
just check
```

`just check` runs the whole-repo lint gates plus the diff-scoped test lane. Because this
change touches the Agents-tab roster pipeline and the keybinding footer, also run the
exhaustive lane through a monitor (never inline — it routinely outruns a single turn):

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED \
  --next 'Report just check-full results for the proc-shell dismissal tale and fix any failures'
```

If Symvision reports unused symbols for the new module, read `sase/memory/symvision.md`
with `/sase_memory_read` before adding any pragma or whitelist entry.

Manual check in a live ACE: start a stand-alone proc shell, let it finish, press `x` on
its row, confirm the row disappears immediately and stays gone for at least a minute
(several observer polls), then restart ACE and confirm it is still gone while
`sase proc show <id>` still works.

## Out of scope

- Any change to the durable proc store, its wire schema, or the Rust core
  (`sase-core/crates/sase_core`). No `sase proc dismiss` CLI verb.
- Per-row dismissal in the Procs pane — it keeps its
  `Durable procs age out with procs.history_limit` behaviour.
- Reviving / rewinding a dismissed proc shell. The proc row remains reachable via the
  Procs pane and `sase proc show <id>`; there is no un-dismiss UI.
- Teaching the Rust cleanup planner about proc shells, and therefore the tribe-cleanup
  and custom-selection scopes, which continue to skip proc-shell rows.
- Any edit to `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction
  shims. The glossary's "a proc shell belongs to an agent" wording is known to be stale
  for stand-alone proc shells, but changing it requires explicit user permission in the
  implementing conversation. If the implementer believes a memory update is warranted,
  file it with `/sase_new_task` as a `memory` task bead instead.
