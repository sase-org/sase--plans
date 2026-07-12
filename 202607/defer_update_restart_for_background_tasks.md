---
create_time: 2026-07-09 12:31:58
status: done
prompt: 202607/prompts/defer_update_restart_for_background_tasks.md
tier: tale
---
# Plan: Defer Update Restart Until Background Tasks Finish

## Problem

The Updates tab binds `u` to `action_update_sase` in `src/sase/ace/tui/modals/plugins_browser_pane.py`. After
confirmation, the SASE update runs through the tracked task queue via `_submit_sase_update_task` or
`_submit_dev_update_task` in `src/sase/ace/tui/modals/plugins_browser_sase_update.py`.

When the update changes installed code, `_handle_code_update_completion` writes the pending update toast and calls
`_restart_after_update`. That path currently waits only for other code-update task types via
`_running_code_update_tasks`. Once those are done, it calls `_restart_tui(restart_axe=True)`. `_restart_tui` then kills
all still-running tracked tasks, so unrelated background work can be stopped even though the normal quit flow would have
warned the user first.

The quit confirmation flow in `src/sase/ace/tui/actions/lifecycle.py` defines the relevant scope: tracked background
tasks in `TaskQueue` whose `status` is `"running"`. `QuitConfirmModal` receives those tasks and warns that quitting now
will kill them. The update restart gate should use that same definition.

## Goal

For a successful code-changing SASE update from the Updates tab, ACE should not restart until all other tracked
background tasks have completed. While waiting, the UI should remain responsive, the task indicator and Task Queue
should remain authoritative, and the user should get a concise queued-restart notification.

## Non-Goals

- Do not change manual quit or restart behavior from `q` / `Q`; those flows can still prompt or intentionally kill tasks
  as they do today.
- Do not wait for untracked Textual workers such as update preview planning or catalog loading. This plan follows the
  quit-confirmation definition of "background task": entries in the shared tracked `TaskQueue`.
- Do not change update execution, receipt generation, or pending post-restart toast semantics.

## Implementation Steps

1. Replace the code-update-only blocker helper in `plugins_browser_sase_update.py`.
   - Add a helper that safely snapshots `self.app._task_queue.get_all()` and returns tasks with `status == "running"`.
   - Keep the current fail-open behavior: if the queue cannot be inspected, return an empty list so a completed update
     does not leave ACE permanently waiting.
   - The just-completed update task should naturally be excluded because `TaskActionsMixin._on_task_worker_completed`
     marks it non-running before invoking the completion callback.

2. Update `_restart_after_update_when_ready`.
   - Use the new running-background-task helper instead of `_running_code_update_tasks`.
   - Change the first queued notification from "update task(s)" to "background task(s)".
   - Continue using `app.set_timer(1.0, ...)` to poll without blocking the Textual event loop.
   - When no running tasks remain, notify that ACE is restarting and call `_restart_tui(restart_axe=True)`.
   - Remove the current final suffix that says background tasks "will be stopped"; at that point the gate should have
     proven there are no running tracked tasks left.

3. Keep the behavior shared across code-changing update operations.
   - `PluginInstallActionsMixin`, `PluginUpdateActionsMixin`, `PluginUninstallActionsMixin`, `ModeSwitchActionsMixin`,
     and `SaseUpdateActionsMixin` all route through the same restart helper.
   - Although the user-visible bug was reported for the SASE-wide `u` update, the shared helper should be fixed once so
     plugin install/update/uninstall and mode switching also avoid killing unrelated tracked tasks after a code change.

4. Preserve responsive TUI behavior.
   - Only inspect in-memory `TaskQueue` state on the UI thread.
   - Do not add subprocess calls, disk reads, sleeps, or synchronous expensive work to action handlers or completion
     callbacks.
   - Continue relying on the existing tracked task queue for task visibility, deduplication, cancellation, and task
     indicator counts.

## Tests

1. Update the existing SASE update restart test in `tests/ace/tui/test_plugins_browser_pane_sase_update.py`.
   - Stop monkeypatching `_count_running_tasks` to force the old "will be stopped" suffix.
   - Create a real running `TaskInfo` in `page.app._task_queue` before the update task completes.
   - Assert the update completion does not call `_restart_tui` while that task is running.
   - Assert the queued notification mentions the number of background tasks.
   - Mark the background task complete and trigger or await the timer callback.
   - Assert `_restart_tui(restart_axe=True)` is called only after the queue has no running tasks.

2. Add a focused helper test for the restart blocker.
   - Running non-update tasks such as `sync`, `mail`, or `launch` should block the restart.
   - Completed `success` / `error` tasks should not block.
   - A completed `sase-update` task should not block.
   - Queue inspection failures should return no blockers.

3. Keep no-task update behavior covered.
   - Existing managed and dev SASE update tests should still prove immediate restart when no other tracked task is
     running.
   - Existing plugin install/update/uninstall restart tests should continue to pass when the queue is empty.

## Verification

Run the focused tests first:

```bash
pytest tests/ace/tui/test_plugins_browser_pane_sase_update.py
pytest tests/ace/tui/actions/test_lifecycle_quit_confirm.py
```

Then run the repository check required after code changes:

```bash
just install
just check
```

## Risks and Edge Cases

- If a long-running task never completes, the automatic restart will remain queued. That matches the requirement and
  keeps the task kill decision in the user's hands through the existing Task Queue or manual quit/restart flows.
- If the user starts a new tracked task while a restart is queued, the next timer tick should see it and continue
  waiting. This follows the live task queue rather than only the tasks that existed at update completion time.
- If ACE exits manually before the queued restart fires, Textual teardown should cancel pending timers as part of normal
  app shutdown.
