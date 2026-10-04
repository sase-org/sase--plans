---
tier: tale
title: Wait for TUI-owned completion callbacks before update restarts
goal:
  Keep update restarts from dropping the in-memory completion handlers of durable
  operations this TUI started, while independent commands still never delay a restart.
size: small
proposed_by: bbugyi200.athena.0vz.f0
create_time: 2026-10-04 08:39:40
status: wip
---

# Wait for TUI-owned completion callbacks before update restarts

## Outcome

Commit `458dfe59dc` ("feat(ace): restart TUI without waiting on independent commands")
is mostly safe. Agent tool runs, detached commands, `!` oneshots, monitors, and service
daemons survive the TUI re-exec and the service-host restart, and it adds isolated
regressions for that (`tests/ace/tui/test_update_restart_lifecycle.py`). The PNG golden
conflict resolution on top of it is also correct.

It introduced one regression. A durable operation started **from this TUI** (patch
sync/reword/add-tag/accept, agent directive/cleanup/revert, bgcmd launch, monitor stop,
and others) keeps a `ProcCallbackConfig` in the TUI's memory until it finishes. When it
finishes, the TUI runs its callback. An update restart no longer waits for that
operation, so if one is still running, the restart throws away its callback. The durable
work still finishes, but the TUI's follow-up never runs.

Fix: if this TUI still owes a completion callback for an active durable proc, count it
as TUI-local work. Wait for it within the existing bounded restart chain. Change nothing
else.

## Verified diagnosis

- `ProcSubmissionActionsMixin._submit_durable_proc`
  (`actions/_proc_action_submission.py`) stores a `ProcCallbackConfig` in
  `app._proc_completion_callbacks` under the placeholder id.
  `_on_durable_submit_worker_completed` (`actions/_proc_action_completion.py`) pops
  `_durable_submit_workers` and moves the config to the durable `handle.proc_id`. It
  then calls `observer.register_submitted`, which sets the placeholder's
  `durable_proc_id` and adds an observer watch. `_deliver_observed_completion` later
  pops the config and runs `notify` / `on_complete` / reload / `on_settled`. This map
  exists only in memory. After `os.execv` it is empty, so `config is None` and nothing
  runs.
- The new `collect_restart_blockers` (`src/sase/ace/tui/update_restart.py`) only reads
  session overlay rows, `_session_workers`, `_durable_submit_workers`, install-mutation
  rows, and local legacy `kind=tui` rows. It never reads `_proc_completion_callbacks`.
  As a result, once the submit worker map pops, a TUI-submitted proc stops blocking. An
  in-memory probe confirmed this: an active `sync` row with this session's id plus a
  pending callback config returns `()`.
- The old `running_background_procs` used `proc_projection_for(app).active_rows()`. That
  included those rows, so the restart waited up to 60 seconds for them. These operations
  are usually short, so in practice their callbacks almost always ran before.
- Callbacks that now get dropped have real side effects:
  - Sync, reword, and add-tag (`actions/sync.py`, `actions/_base_patch.py`) call
    `reset_dollar_hooks` only from the TUI callback. If it is dropped, `$` hooks keep
    stale verdicts for content that has changed.
  - Accept with "mark ready to mail" (`actions/proposal_rebase.py`) chains
    `action_mail()`.
  - A bgcmd launch (`actions/axe_bgcmd.py`) records command history and releases its
    slot reservation.
  - Cleanup and directive callbacks show failure toasts and resurface failed members.
  - Default configs show the result toast.
- Independent work has no entry in this TUI's callback map:
  - Agent `sase tool run` procs are submitted by the agent process.
  - A long-running `!` oneshot row is created by the short `bgcmd-launch` operation.
    Only that launch operation carries the callback.
  - Monitors and service daemons are not submitted through `_submit_durable_proc`.
  - Waiting on callback entries therefore does not bring back the original complaint.

## Policy addition

| Work                                                                              | Automatic update restart                                    |
| --------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| Active durable proc with a pending `_proc_completion_callbacks` entry in this TUI | Wait, using the existing 60-second chain (new blocker kind) |
| The same proc type without a callback entry (other TUI, CLI, agent)               | Unchanged: does not block                                   |
| Leftover callback entry whose proc is terminal, or absent and unwatched           | Stale; never blocks                                         |

All other rows of the restart dependency table from the original plan stay as they are.

## Implementation

1. **Add a completion-callback blocker in `update_restart.py`.**
   - Add the `"completion"` member to `RestartBlockerKind`. Map it to the existing
     `"tui"` group in `_KIND_GROUP`, so queued, expired, and tooltip copy says "TUI
     task" and the current user-facing strings and PNG goldens stay valid.
   - In `collect_restart_blockers`, after the submission pass and before the
     install/legacy pass, iterate a snapshot (`tuple(...)`) of the keys of
     `getattr(app, "_proc_completion_callbacks", {})`. Use an in-memory read only: no
     I/O, no store scan, and no mutation of the map.
   - For each key, find a cached projection row whose `proc_id` or `durable_proc_id`
     equals the key. Extend `_row_by_id` if it does not already match both fields
     symmetrically.
   - If that row's status is active, add a `"completion"` blocker. Use the row identity
     and label, and alias both the key and the row's `proc_id`, so the existing `seen`
     set dedupes it against a submission placeholder or an install row. The install pass
     must still report an install row as kind `"install"`. Keep the pass order, or give
     install precedence explicitly, and test it.
   - If no cached row matches, block only while the TUI's `ProcObserver` still watches
     that proc id. This covers the window between `register_submitted` and the next
     observer snapshot. Add a small lock-guarded, read-only accessor on `ProcObserver`
     (for example `is_watching(proc_id) -> bool`) instead of reading `_watches` from
     outside. Use the fallback label "TUI follow-up".
   - A matched row that is terminal, or a key that is neither observed nor watched, is
     stale. Ignore it so it cannot hold the restart until the deadline.
2. **Keep the restart chain semantics unchanged.**
   - `_deliver_observed_completion` already pops the config before it invokes callbacks.
     So an update proc whose completion handler requests the restart does not wait on
     itself. Preserve that ordering and cover it with a test.
   - Keep the original deadline, one-second polling, coalescing, tokens, untracked
     chains, and the forced-restart warning.
3. **Docs.** In `docs/ace.md`, wherever the earlier commit says "Independent commands
   keep running" or lists independent work, add one clause: durable operations started
   from this TUI still finish their result handling first, within the same 60-second
   wait. Do not change the feature-flag confirmation string, the update-panel row copy,
   or the yellow-gear tooltip wording. They already say "TUI tasks", which keeps
   `config_center_flags_confirm_120x40.png` unchanged. If you change rendered copy
   anyway, regenerate the goldens with `just fix-tui-screenshots -- <selectors>` and
   inspect the result.

This is TUI lifecycle policy over TUI-process state. Keep it in `src/sase/ace/tui`. No
`sase-core`, wire, configuration, or feature-flag change is needed.

## Acceptance and verification

Add the following to `tests/ace/tui/test_update_restart.py`, reusing its `_App`,
`_projection`, and row helpers:

- An ordinary durable row with a callback entry blocks for each of `pending`, `running`,
  and `settling`, with kind `"completion"`. Keep the existing
  `test_ordinary_durable_work_does_not_block_restart` passing: the same row without a
  callback must not block.
- Handoff dedupe: a placeholder row carrying `durable_proc_id` with the callback rekeyed
  to the durable id gives exactly one blocker. So does a store row whose `proc_id` is
  the durable id. A submit worker still present under the placeholder id plus the
  callback under that same id also gives one blocker.
- An unobserved but watched proc blocks. An absent and unwatched key does not. A
  terminal row with a leftover config does not.
- An install row that also has a callback is reported once, as `"install"`.
- Tool-run, oneshot, monitor, and daemon rows never block, even when they carry the
  current session id.
- Mixed work: an active tool run plus one callback-bearing sync row waits only for the
  sync. After that completion is delivered through the real
  `_apply_proc_observer_snapshot` / `_deliver_observed_completion` path, the callback
  runs (for example a sentinel standing in for `reset_dollar_hooks`) before
  `_restart_tui(restart_axe=True)` is called, while the tool run is still active. Also
  cover an update completion handler that requests the restart from inside its own
  callback: it must not wait on itself.
- Timeout: if the callback proc never finishes, restart at the original deadline with
  the warning naming it.
- Add a unit test for the new `ProcObserver` accessor.

Run, in order:

1. Read `tui.md` and `lint_and_test.md` with `sase memory read` before editing.
2. Run the focused regressions through `sase tool run test` (use `SASE_PYTEST_WORKERS`,
   not a pytest `-n` argument): `test_update_restart.py`,
   `test_update_restart_lifecycle.py`, `test_update_run_actions.py`,
   `test_plugins_browser_pane_sase_update.py`,
   `test_proc_actions_session_workers_overlay.py`, and the proc observer tests.
3. Format, then run `sase tool run check`. Do not run `check-full`.

Report the actual results. Known master-red items, such as the current Symvision
`runner_kill_provenance.py` finding, are not caused by this change.

## Out of scope (follow-up)

The `$` hook reset after sync, reword, and add-tag, and the accept-then-mail chain, live
only in TUI completion callbacks. They are still lost when a user quits or the TUI
crashes mid-operation, or when the operation outlives the 60-second restart deadline.
The CLI `sase patch sync` path also appears never to reset `$` hooks. Moving that reset
into the durable patch operation's settlement is a separate domain fix that may belong
in `sase-core`. Verify this, then file it as a task bead through `/sase_new_task`. Do
not implement it here.
