---
tier: tale
title: Fix the phantom running-proc count and the restart it blocks
goal:
  The header proc indicator, the Procs pane, the post-update restart gate, and the quit
  modal agree on one session-scoped active-proc count, orphaned proc rows heal
  themselves, and a finished update always restarts ACE.
size: medium
proposed_by: bbugyi200.athena.03i
create_time: 2026-08-16 10:10:24
status: wip
---

- **PROMPT:**
  [prompts/202608/phantom_running_proc.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/phantom_running_proc.md)

# Plan: Fix the phantom running-proc count and the restart it blocks

## Problem

The ACE top-bar proc indicator showed `⚙ 1` while the Admin Center Procs tab ("this
session" scope) showed `[0 running · 80 done]`, and a completed `sase update` refused to
restart the TUI because it believed a proc was still running.

All three symptoms come from one stuck row plus three surfaces that each compute "how
many procs are running" differently.

### The stuck row

`~/.sase/procs/procs.jsonl` held exactly one active row:

```json
{
  "proc_id": "hbw7szt6mnbh",
  "label": "kill my_feature",
  "kind": "tui",
  "lifecycle": "legacy",
  "status": "running",
  "origin": "ace",
  "session_id": "20260816T013806Z-2654527",
  "pid": 2654527,
  "started_at": "2026-08-16T01:39:49.409131Z",
  "finished_at": null
}
```

Pid 2654527 was gone and its session was not in `~/.sase/sessions/`, so the row had been
orphaned for roughly twelve hours. Every other row in the store was terminal.

`reconcile_running_procs()` (`src/sase/procs/runner.py:188`) already terminalizes
exactly this shape: `_legacy_is_orphaned()` (`src/sase/procs/runner.py:257`) returns
`True` for a non-supervisor kind whose recorded pid is dead, and `is_proc_shell_row()`
is `False` for a `legacy` lifecycle row. The row survived because **nothing in ACE ever
calls it**. The only call sites are `_reconcile_quietly()` in the `sase proc` CLI
handler (`src/sase/main/proc_handler.py:116`, `:236`, `:288`, `:508`). A long-lived ACE
session therefore never heals its own store.

### Why the two surfaces disagreed

`ProcObserver._build_snapshot()` computes `active_count` over **every** row it projects
(`src/sase/ace/tui/proc_observer.py:396`), and `_is_relevant()`
(`src/sase/ace/tui/proc_observer.py:480`) admits this row because
`proc.origin == "ace"`, even though it belongs to a dead foreign session.
`_update_proc_indicator()` feeds that number straight to the header widget
(`src/sase/ace/tui/actions/proc_actions.py:106-113`).

The Procs pane counts something else entirely: `_merged_tasks()` returns
`scoped_rows(all_sessions=self._all_sessions)`
(`src/sase/ace/tui/modals/procs_pane_selection.py:116`), and `scoped_rows()` keeps only
rows whose `session_id` is `None` or this session's
(`src/sase/ace/tui/proc_observer.py:173-181`). `ProcsSessionState.all_sessions` defaults
to `False` (`src/sase/ace/tui/modals/config_center_session.py:67`), so the pane dropped
the dead session's row and rendered `0 running`
(`src/sase/ace/tui/modals/procs_pane_selection.py:353-364`).

### Why the update never restarted

`running_background_procs()`
(`src/sase/ace/tui/modals/plugins_browser_sase_update_procs.py:298-302`) reads the same
unscoped `projection.rows`. The orphan matched, so `_restart_after_update_when_ready()`
(`:273-290`) re-armed its one-second timer forever: there is no deadline, no liveness
check, and no way for the user to override it. The quit/restart options modal reads the
same unscoped number via `_count_running_tasks()`
(`src/sase/ace/tui/actions/lifecycle.py:194-197`, consumed at
`src/sase/ace/tui/actions/axe.py:224`), so it was wrong too.

That gate carries a second, independent inconsistency: it matches only
`status == "running"`, while every other surface uses `ACTIVE_PROC_STATUSES` (`pending`,
`running`, `settling`). A `pending` proc this session genuinely owns is counted by the
indicator but does **not** block a restart.

### Why the user could not clear it by hand

The pane hid the row in its default scope. Even after pressing `a` to widen the scope,
`K` routes to `kill_store_task()` → `kill_proc()` (`src/sase/procs/runner.py:179-182`),
which raises "TUI-owned procs can only be killed from their owning ACE session" — and
that session was gone.

### Contributing defect: ACE procs are unattributed

`_submit_durable_proc()` never passes `session_id`
(`src/sase/ace/tui/actions/proc_actions.py:219-232`), so `submit_durable_proc_request()`
(`src/sase/ace/tui/durable_submit.py:75`) defaults it to `None` and every ACE-submitted
durable row lands unattributed. In the observed store, 81 of 102 rows had
`session_id: null` and **zero** belonged to the live ACE session. The pane's "this
session" label is therefore not true today: it shows every ACE session's procs. Scoping
the indicator without fixing attribution would make the surfaces agree while both stayed
wrong whenever two ACE sessions run at once.

## Goals

1. One shared definition of "active procs" that the header indicator, the Procs pane
   title, the post-update restart gate, and the quit/restart modal all use.
2. Orphaned rows heal themselves, with and without ACE running.
3. A finished update always restarts ACE, even if some proc never terminalizes.
4. ACE-submitted durable procs are attributed to the ACE session that submitted them, so
   "this session" scope means what it says.

## Non-goals

- Making foreign/orphaned `tui`-kind rows killable from the Procs pane. Reconciliation
  terminalizes them, which is the better fix; the `kill_proc` ownership rule stays
  as-is.
- Adding a fail-closed pytest sandbox guard to proc-store writes. The store's write
  helpers in `src/sase/procs/store.py` have no equivalent of
  `assert_test_state_write_isolated`, unlike the bead and telemetry stores, and the
  stuck row's provenance (cwd inside an ephemeral agent workspace, sibling rows carrying
  test-double payloads such as the `my_feature` fixture agent and the message
  `Kill cleanup failed for my_feature: boom`) points at a pytest process writing into
  the user's real store after fixture teardown. That is a separate problem class with
  suite-wide blast radius, and it is already tracked: it was corroborated on ready task
  **sase-ml** ("Full just test run fails ~60 gate/ops/launch tests via live host
  runtime-path contamination"), which reports the read direction of the same missing
  boundary. Do not take that guard on inside this tale; the reconciliation in Step 5
  heals leaked rows regardless of where they came from.
- Any change under `../sase-core`. Proc-store I/O already crosses into Rust through
  `_call_binding`, but reconciliation policy and every surface touched here are
  Python-side.

## Implementation

### Step 1 — Give `ProcProjection` one active-set definition

In `src/sase/ace/tui/proc_observer.py`:

- Add `ProcProjection.active_rows(*, all_sessions: bool = False) -> list[ObservedProc]`.
  It filters `scoped_rows(all_sessions=all_sessions)` down to rows whose `status` is in
  `ACTIVE_PROC_STATUSES` **and** whose owner is still alive — that is, drop any row with
  a non-`None` `session_id` whose `session_live` is `False`. Locally registered
  placeholders keep `session_id is None` and `session_live` defaulted to `True`, so
  in-flight submissions this session has not yet heard back about still count.
- Widen `scoped_rows()` so the session scope also keeps rows whose owning session is
  dead (`session_id is not None and not session_live`). An orphan belongs to nobody;
  hiding it is what made this bug invisible in the pane. Combined with `active_rows()`,
  such a row is **visible but never blocking**. This also stops a proc submitted before
  an ACE restart from vanishing from the default scope once the new session gets a new
  id (Step 4).
- Redefine the stored `active_count` field as `len(active_rows())` — session scoped,
  live-owner only — and compute it that way in `_build_snapshot()` (`:396`). Update the
  other construction site, `src/sase/ace/tui/repro/replay.py:270`, to match.

Add `session_live` to `_snapshot_signature()` (`:563`) so a session dying is a change
the observer publishes rather than swallows.

### Step 2 — Point every consumer at that definition

- `src/sase/ace/tui/modals/plugins_browser_sase_update_procs.py:298`: rewrite
  `running_background_procs()` to return `projection.active_rows()`, keeping the
  existing fail-open behavior when `_proc_projection` is missing or its rows are `None`
  (both covered by `tests/ace/tui/test_plugins_browser_pane_sase_update.py:79-107`).
  This fixes the `"running"`-only status check as a side effect.
- `src/sase/ace/tui/actions/lifecycle.py:194`: `_count_running_tasks()` returns
  `len(projection.active_rows())`.
- `src/sase/ace/tui/modals/procs_pane_selection.py:353`: `_title_text()` keeps counting
  `self._tasks` (it must reflect the pane's current scope), but must use the same
  active-and-live predicate, so that in the default scope the title count is identical
  to `active_count`. Extend `is_active()` in
  `src/sase/ace/tui/modals/procs_pane_render.py:39` or add a shared helper rather than
  duplicating the rule a fourth time.
- `_update_proc_indicator()` (`src/sase/ace/tui/actions/proc_actions.py:106`) needs no
  change once `active_count` is redefined; lock the behavior in with a test.

### Step 3 — Never let a finished update hang on the restart gate

In `src/sase/ace/tui/modals/plugins_browser_sase_update_procs.py:269-295`:

- Give `_restart_after_update_when_ready()` a bounded total deferral (60s is ample; the
  current caller re-arms at 1s). Track the deadline across the timer chain — an extra
  keyword-only parameter with a default is enough; do not stash it on `self`, since the
  mixin is shared.
- On expiry, restart anyway and toast a warning naming what it stopped waiting for.
  Durable procs are supervisor-owned and survive an ACE restart, so waiting is a
  courtesy, not a correctness requirement.
- Re-evaluate blockers from `active_rows()` on every tick, so a proc whose owner dies
  mid-wait stops blocking immediately.

### Step 4 — Attribute ACE durable submissions to the ACE session

- In `_submit_durable_proc()` (`src/sase/ace/tui/actions/proc_actions.py:218-232`), pass
  `session_id=` down to `submit_durable_proc_request()`. Resolve it from
  `sase.sessions.current_session_id()`, guarded with `try`/`except` so a registry
  failure degrades to `None` (today's behavior) instead of failing the submit.
  `ProcObserver._load_context()` (`src/sase/ace/tui/proc_observer.py:501`) already
  resolves the same id off the UI thread; reuse that resolution or cache it once per app
  rather than calling the registry inside the submit worker on every submission.
- `submit_durable_proc_request()` (`src/sase/ace/tui/durable_submit.py:62`) already
  accepts and forwards `session_id`; no signature change needed there. Populate
  `session_label` only if it is free to obtain — `submit_proc()` resolves it via
  `_session_label()` (`src/sase/procs/runner.py:311`), which walks `live_sessions()`, so
  do not add that cost to the submit path if it is not already paid.

Note for verification: existing rows in a developer's store stay unattributed. The Step
1 `scoped_rows()` widening keeps them visible, since they are neither session-owned nor
orphan-marked — confirm this by reading the pane with a store that mixes attributed,
unattributed, and dead-session rows.

### Step 5 — Reconcile orphaned rows

Two independent sweeps, because either one alone leaves a hole.

**ACE-side.** Add `src/sase/ace/tui/proc_reconciler.py` exposing a single function that
calls `sase.procs.reconcile_running_procs()` and returns the reconciled rows, wrapped so
no exception escapes. It **must** be guarded by
`pytest_path_is_sandboxed(proc_store_path())` exactly as `ProcObserver.start()` is
(`src/sase/ace/tui/proc_observer.py:250`), so a test process cannot write into the real
store. Drive it from the app: once shortly after startup, then on a slow interval (30s —
this reads the whole store and stats pids, so it must not ride the observer's 0.5s
cadence), always in a thread worker. Do **not** put the write inside `ProcObserver`:
that class is documented as a read-only projection and the whole design depends on it
staying that way. The existing observer poll picks the healed rows up on its own.

**Axe-side.** `HookJobs.run_stale_running_cleanup()`
(`src/sase/axe/hook_jobs.py:374-397`) already reconciles dead monitor supervisors and
stale workspace claims. Add a `reconcile_running_procs()` call there, following the
`_reconcile_dead_monitor_supervisors()` pattern (`:399-407`): failures log and degrade
rather than aborting the sweep. Include the count in the summary line and in the
`reason=` suffix condition. Update both `stale_running_cleanup` chop descriptions in
`src/sase/default_config.yml` (around lines 670 and 756) to say that the sweep also
terminalizes orphaned proc rows.

## Tests

Add to `tests/ace/tui/test_proc_observer.py`:

- A dead-session `running` row is projected (so the pane can show it) but is excluded
  from `active_count` and `active_rows()`.
- A row owned by a **live** foreign session is excluded from the default scope's
  `active_rows()` and included when `all_sessions=True`.
- An unattributed row and a locally registered pending placeholder both count.
- `pending` and `settling` count as active; `success`/`error`/`killed` do not.

Add a cross-surface regression test (new file or `tests/ace/tui/_procs_pane_helpers.py`
consumers) asserting that for one projection the header indicator count,
`ProcsPane._title_text()`'s running count in default scope,
`running_background_procs()`, and `_count_running_tasks()` all agree. This is the
invariant whose absence caused the bug; assert it directly.

Extend `tests/ace/tui/test_plugins_browser_pane_sase_update.py`:

- A dead-session `running` row does not block the restart.
- A live-session `pending` row does block it.
- The bounded wait expires and restarts, emitting the warning toast.
- The existing fail-open cases at `:79-107` still pass.

Extend `tests/ace/tui/actions/test_lifecycle_quit_confirm.py` for the scoped count, and
`tests/ace/tui/test_proc_actions_session_workers.py` (or the durable submit tests) for
`session_id` reaching `submit_durable_proc_request()`.

Add a reconciler test that a `legacy`-lifecycle row with a dead pid is terminalized by
the new ACE-side entry point against a sandboxed store, and that the entry point is a
no-op when `pytest_path_is_sandboxed()` is `False`.

For the axe leg, extend the `run_stale_running_cleanup` tests so the summary reports
reconciled procs and a reconciliation failure does not abort the claim sweep.

## Verification

```bash
just install
just check
```

Then hand `just check-full` to `/sase_monitor` — this change touches the ACE proc
surface, the axe hooks lane, and `default_config.yml`, so the scoped test lane is not
sufficient. Do not run `just check-full` inline.

Manual confirmation, since the failure was only visible in the running TUI:

1. Append a synthetic orphan to a scratch proc store (`SASE_HOME` pointed at a throwaway
   directory): status `running`, `lifecycle: legacy`, `kind: tui`, `origin: ace`, a
   `session_id` that is not in `~/.sase/sessions/`, and a `pid` that is not running.
2. Start ACE against that `SASE_HOME`. The header indicator must not show a count, and
   the Procs tab must list the row while reporting `0 running`.
3. Within one reconcile interval the row must flip to `error` with "supervisor exited
   without reporting", and stay flipped.
4. With a genuinely running proc of the current session, the indicator count and the
   pane's default-scope running count must match, and a restart must wait for it and
   then proceed.

## Immediate workaround for the reported instance

Any `sase proc list` run terminalizes the stuck row today, because the CLI handler calls
`_reconcile_quietly()` before it renders. That clears the phantom count and unblocks the
restart gate without waiting for this fix to land.
