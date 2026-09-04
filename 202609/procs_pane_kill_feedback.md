---
tier: tale
title: Fix silent no-op K (kill) keymap on the Admin Center Procs tab
goal:
  Pressing K on the Procs tab either opens the kill confirmation for every store-backed
  active proc row (including dead-session rows) or explains via a toast why the selected
  row cannot be killed, and the shift+k/shift+d aliases match the pane's existing
  shift+g convention.
size: small
proposed_by: bbugyi200.kellys_mbp.p
create_time: 2026-09-04 12:28:54
status: wip
---

# Fix silent no-op `K` (kill) keymap on the Admin Center Procs tab

## Problem

Pressing `K` on the "Procs" tab of the SASE Admin Center frequently does nothing: no
confirm modal, no toast, no log line. The binding, confirm modal, and kill plumbing are
all correct (covered by the passing
`test_tasks_tab_kill_confirms_and_signals_store_row`); the defect is the guard at the
top of `action_kill_task` in `src/sase/ace/tui/modals/procs_pane_actions.py`:

```python
task = self._get_selected_task()
if task is None or not is_active(task) or not task.store_backed:
    return
```

It silently declines the keypress for whole classes of rows that render exactly like
killable running procs, because the row spinner is driven purely by
`status == "running"` (`_task_status_token` in `procs_pane_render.py`), while the guard
demands more:

1. **Rows whose owning session is no longer live.** `is_active()`
   (`procs_pane_render.py`) requires `task.session_live` for session-attributed rows. A
   durable proc still `running` after its ACE session died keeps spinning in the list
   (dead-session rows are intentionally included by `ProcProjection.scoped_rows`), but
   `K` refuses silently. The refusal is also functionally wrong: `sase.procs.kill_proc`
   handles dead-owner rows correctly (`stop_proc_shell` records stop intent and
   reconciles when the supervisor is gone; `_legacy_kill_proc` terminalizes orphaned
   rows), and `sase proc kill` from the CLI already allows killing them. The TUI log
   regularly shows "ACE exiting with live worker threads", which strands exactly these
   rows.
2. **Pending placeholders and session-overlay worker rows** (`store_backed=False`).
   Freshly submitted procs (before `ProcObserver.register_submitted` flips the
   placeholder) and in-process session workers composed by `_effective_proc_projection`
   silently refuse `K` with no explanation.

Separately, the pane binds only the bare uppercase key: `("K", "kill_task", "Kill")`.
House convention pairs every uppercase mnemonic with its `shift+<letter>` twin
(`G`/`shift+g` in this same file and in `TaskList`, `Y`/`shift+y`, `Z`/`shift+z`,
`N`/`shift+n` in the preview modals) because some terminal stacks re-encode shifted
letters as CSI-u sequences, which Textual's parser lowercases to `shift+k` — a key the
bare `"K"` binding never matches. The Procs pane has the alias for `G` but not for `K`
or `D`.

## Changes

### 1. Rework the kill guard to act or explain (`src/sase/ace/tui/modals/procs_pane_actions.py`)

Replace the body of `action_kill_task` so every keypress either opens the confirm modal
or tells the user why not:

- `task is None` → return (empty list; nothing to explain).
- If `task.store_backed` and `proc_status_is_active(task.status)` → push the existing
  `ConfirmActionModal` and kill on confirmation, exactly as today. This drops the
  `is_active()` session-liveness requirement: a store-backed row with an active status
  is killable even when its owning session is gone. Import `proc_status_is_active` from
  `..proc_observer` (it is re-exported there) and stop importing/using `is_active` in
  this method (keep `is_active` imports used by other methods in the module).
- Else if `not proc_status_is_active(task.status)` (finished row) →
  `self.notify("Proc already finished", severity="warning")` and return.
- Else (active status but `store_backed` is `False`):
  - `task.status == "pending"` →
    `self.notify("Proc is still submitting — try again in a moment", severity="warning")`.
  - otherwise →
    `self.notify("Session-local task; it cannot be killed from the Procs tab", severity="warning")`.

Rationale for the widened kill: `kill_proc` already gives the right outcome for every
store-backed active row. Live supervisor → SIGTERM and settlement; dead or rebooted
supervisor → the row is reconciled/terminalized, which clears the stale "running"
spinner the user was trying to get rid of. A kill failure already surfaces through the
existing `_on_store_kill_finished` error toast.

### 2. Add the missing shift aliases (`src/sase/ace/tui/modals/procs_pane.py`)

In `ProcsPane.BINDINGS`, mirror the existing `G`/`shift+g` pair:

- after `("K", "kill_task", "Kill")` add `("shift+k", "kill_task", "Kill")`;
- after `("D", "dismiss_all_done", "Dismiss All Done")` add
  `("shift+d", "dismiss_all_done", "Dismiss All Done")`.

No hint-footer text changes: the pane already advertises `K: kill` and `d/D: dismiss`,
and the alias does not change the displayed key. The Procs pane keys are hard-coded (the
pane takes no `keymaps` argument from the Admin Center tab catalog), so there is no
`src/sase/default_config.yml` keymap entry to update.

### 3. Regression tests (`tests/ace/tui/test_procs_pane.py`)

Using the existing `ProcsTestApp` / `open_procs_pane` harness and the `task(...)` row
factory:

1. **Dead-owner rows are killable**: a `status="running"`, `store_backed=True` row with
   `session_id` set and `session_live=False`; press `K` → `ConfirmActionModal` appears;
   press `y` → the patched `kill_store_task` receives the row's durable proc id and the
   "Killed:" toast fires.
2. **Finished row explains itself**: a `status="success"` store-backed row; press `K` →
   no modal is pushed and the last notification is
   `("Proc already finished", "warning")`.
3. **Non-store-backed running row explains itself**: a `status="running"` row with
   `store_backed=False`; press `K` → no modal and the session-local warning toast fires.
   Also cover `status="pending"` → the "still submitting" toast.
4. **Alias works**: with a killable row selected, `pilot.press("shift+k")` opens the
   confirm modal just like `"K"`.

Keep `test_tasks_tab_kill_confirms_and_signals_store_row` passing unchanged.

## Out of scope

- Cancelling session-overlay UI workers from the Procs tab (would need worker handles,
  not the durable store).
- The confusing `kill_proc` refusal message for `TUI_PROC_KIND` rows.
- Auditing the ~27 other TUI modals whose uppercase bindings lack `shift+` twins
  (tracked separately).

## Verification

- `just check` must pass (lint, types, and the focused test lanes).
- Manually sanity-check the touched tests: `pytest tests/ace/tui/test_procs_pane.py -q`.
