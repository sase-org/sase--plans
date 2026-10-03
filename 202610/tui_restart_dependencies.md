---
tier: tale
title: Let independent commands continue through TUI update restarts
goal:
  Restart the TUI after updates without waiting for independent agent commands, while
  preserving waits for actual TUI dependencies and installation mutations.
size: medium
proposed_by: bbugyi200.athena.0vz
create_time: 2026-10-03 19:15:23
status: wip
---

# Let independent commands continue through TUI update restarts

## Outcome

After a successful `,E` or `,U` update, restart the TUI promptly while an agent's
`sase tool run check` continues. Keep the command visible in the blue proc gear and
Procs pane. Wait only for work that actually depends on this TUI, or known operations
still changing the installation that the restarted TUI will load.

This is one bounded TUI lifecycle correction, suitable for one implementation agent
(`tale`, `medium`). Do not implement a new process supervisor, ToolRun lifecycle,
configuration option, or feature flag.

## Verified diagnosis

- `src/sase/default_config.yml` binds `,E` to `update_everything`. The app-level
  completion path in `actions/update_run.py`, plugin update paths, stale-code restart,
  and feature-flag restart all reach `src/sase/ace/tui/update_restart.py`.
- Its `running_background_procs()` returns every active observed row except monitors and
  service daemons. This conflates visibility with restart dependency. The helper polls
  every second, queues a yellow restart indicator, and restarts anyway after 60 seconds.
  Waiting is already a bounded courtesy, not a process-survival guarantee.
- `src/sase/tool/inline_escalation.py` explains the intermittent appearance: when an
  agent has a provider sync budget, an eligible plain tool run starts detached and
  follows its output inline. `-d` and non-agent `-H` also use `tool/handoff_launch.py`,
  which submits a durable proc with origin `tool-run`. True inline runs do not create
  this additional proc; monitor-owned runs use their existing owner. Never infer safety
  from the command text or the label `tool:check`.
- `procs/spawn.py` and `procs/supervisor_bootstrap.py` launch a reparented supervisor
  with independent stdio/session, using `detach_scope.py` for systemd scope escape where
  available. Agent-detached ToolRuns remain tied to their starting agent's lifetime
  unless joined by a monitor; restarting the TUI does not end that agent. Inline tool
  children are owned by their wrapper, also outside the TUI.
- `ProcObserver.stop()` explicitly does not touch proc lifetime. TUI teardown in
  `actions/lifecycle.py` stops observers and unregisters its session, then
  `main/ace_handler.py` re-execs the TUI. The update path requests the service-host
  restart too, through `actions/axe_display/_refresh_full.py`.
- `service/host_spawn.py::_stop_all_children()` stops the host's own children. Ordinary
  detached proc supervisors are separate; transient `!` oneshots use that same detached
  submission path (`procs/oneshot.py`). Service daemons deliberately restart. The
  investigation did not restart the user's live TUI or service host; service-host
  independence is supported by the ownership and scope code, with an isolated
  integration regression required below. Scope escape is best-effort on
  unsupported/disabled configurations; a blanket 60-second wait cannot fix those.
- `src/sase/ace/tui/quit_impact.py` already distinguishes work lost at TUI exit: session
  workers and unfinished durable submission workers. It explicitly excludes durable
  procs, monitors, and background commands. Update restart should use the same ownership
  facts, while retaining an installation-mutation barrier.
- Concurrent installers are a real exception: plugin update/install/uninstall can mutate
  the SASE environment from a durable proc, and their completion handlers arrange
  restart. Existing typed scopes include `plugin-update:NAME`, `plugin-install:NAME`,
  and `plugin-uninstall:NAME`; session-local core/dev/CLI update work is registered
  through `_submit_session_worker()`.

The investigation ran 36 existing focused tests successfully, including update restart,
normal quit, detached supervisor survival after its starter exits, real oneshot
execution, and plain-tool automatic detachment. A separate in-memory probe using a
store-backed `tool-run` row reproduced blue gear count 1, no normal-quit impact, and an
unnecessary update-restart timer/toast. No implementation files were changed. The first
test invocation used an unsupported `-n` argument; rerunning with the supported
`SASE_PYTEST_WORKERS=1` completed successfully.

## Restart dependency policy

| Work                                                                               | Automatic update restart                                                                   |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Agent tool run, detached ordinary command, named proc, monitor, transient oneshot  | Continue immediately when these are the only active work                                   |
| Active session-local worker in this TUI                                            | Wait using the existing bounded restart chain                                              |
| Durable submission still being accepted/handled by this TUI                        | Wait until submission handling finishes; do not wait for the resulting independent command |
| Known core/plugin/CLI installation mutation                                        | Retain the bounded wait, including durable plugin install/update/uninstall operations      |
| Service daemon                                                                     | Do not wait for termination; the existing restart path replaces it                         |
| Terminal row, safely stashed prompt draft, queued launch that exit already stashes | Not a restart blocker                                                                      |

TUI session attribution is not process ownership: a durable command bearing the current
`session_id` still survives. Legacy `kind=tui` records must not accidentally be treated
as detached; preserve conservative handling of explicitly TUI-owned current-session
legacy work. Rows from another TUI must not masquerade as local workers. Known
installation mutations visible in the cached projection remain barriers regardless of
which TUI submitted them.

For installation barriers, inspect active statuses in the cached rows without the Procs
pane's session-liveness filter: an exited submitting TUI does not terminate a durable
installer. This remains an in-memory read, not a new synchronous store scan.

## Implementation

1. **Collect actual restart dependencies.** Replace the broad projection filter in
   `update_restart.py` with one TUI-local collector providing stable identity and
   display label for each blocker. Reuse or extract the in-memory ownership checks from
   `quit_impact.py`; avoid two drifting definitions of session-owned work. Read session
   workers from the app's session-worker/overlay registries and pending submissions from
   `_durable_submit_workers`, not from a guessed origin or label. Use existing proc
   types and concurrency scopes to recognize installation mutations. Reuse update-lane
   classification where appropriate, but include plugin install/uninstall:
   `is_update_row()` alone does not cover them. Deduplicate a submission placeholder,
   its durable row, and an update scope into one blocker. Supply a stable fallback label
   when a submission has no observer row yet.

   Cover the completion-message boundary: a worker can have returned while its
   handle/completion callback is still queued. Keep a submit protected until its result
   has been processed on the UI thread, and keep a completed session worker protected
   until its callback is delivered. Existing completion handlers remove session records
   before invoking their restart callback, which prevents an update from waiting on
   itself. Preserve that ordering. A stale observer placeholder must not extend the wait
   after the handoff to an ordinary durable command.

2. **Use the collector throughout the existing restart chain.** Initial requests,
   coalesced requests, timer polls, pending indicator identities/labels, and timeout
   warnings must all use the same blockers. If only independent commands remain, restart
   immediately without a queued toast or yellow pending state. If a later request
   arrives after blockers drain, do not emit a misleading zero-proc wait. Preserve the
   original 60-second deadline, one-second nonblocking polling, message/purpose
   coalescing, `track_pending=False`, restart-with-service-host behavior, prompt
   stashing, and existing forced-restart warning after expiry. Ensure stale timer
   callbacks cannot trigger a second restart. Update internal helper names/re-exports
   and affected consumers consistently; no proc-store writes or process signals belong
   in this collector.

3. **Make the user-facing wait accurate.** Update queued/expired notices and the yellow
   tooltip to describe the actual work being waited on (TUI tasks, submissions, or
   installation changes). Adjust the stale-code row description in
   `update_panel_state.py`, feature-flag confirmation text in
   `modals/feature_flags_pane_rendering.py`, and relevant `?` help text to remove the
   claim that all active procs must finish. Keep identities usable by the pending
   restart detail UI. Do not change blue gear eligibility/counts, monitor/service
   visibility, Procs-pane filtering, keys, or defaults. No `default_config.yml` change
   is needed unless implementation actually changes a configuration value.

4. **Add behavior and lifetime regressions.** Update the old tests that encode the broad
   policy, using real session-worker/submission fixtures for legitimate blockers rather
   than labeling every observed row ordinary work. Cover the matrix and races below. Add
   an isolated real-process test connecting the TUI observer lifecycle and a detached
   tool command; reuse existing proc/service harnesses. Do not run a production update,
   restart the live host, or run the full `just check` suite merely as the child payload
   in a lifecycle test.

This behavior is TUI presentation/lifecycle policy over existing ownership data. Keep it
in `src/sase/ace/tui`; no new shared domain or wire behavior is required. If an actual
backend ownership defect is discovered, stop and reassess that separate scope rather
than implementing a Python fallback; shared backend changes belong in the linked
`sase-core` repository and require its binding/pin workflow.

## Acceptance and verification

- Parameterize tool-run rows over `pending`, `running`, and `settling`, including
  current-session and unattributed rows. With only independent work, `,E` completion
  calls `_restart_tui(restart_axe=True)` immediately, schedules no wait timer, and
  leaves proc rows, blue gear counts, logs, and ownership untouched. Renaming the tool
  or using an ad-hoc argv does not change the result.
- Cover ordinary durable work, transient oneshots with and without additive service
  metadata, monitors, service daemons, explicit legacy TUI ownership, and other-TUI
  attribution. Assert that visibility changes cannot control restart safety.
- Mixed independent work plus one local worker/submission waits for exactly that
  dependency. On its completion, restart even while the tool command remains active.
  Verify absent observer placeholders, completed-but-undelivered callbacks,
  placeholder-to-durable replacement, failure/cancellation cleanup, and identity
  deduplication. Protect submission handling without extending the wait to command
  completion.
- Cover concurrent durable plugin update/install/uninstall and session core/dev/CLI
  updates. They still wait, their initiating completion does not block itself, and
  receipts/completion handling occurs before restart. Keep existing no-change and
  failed-update behavior.
- Preserve timeout, coalescing, latest message, original deadline, untracked restart,
  missing restart callback, and pending-state clearing tests. Exercise app-level `,E`,
  plugin completion, stale-code restart, and feature-flag restart consumers.
- The isolated process regression must observe the same command PID/run identity across
  observer/TUI shutdown and reconnection, then verify exact output, exit code, and one
  terminal outcome. Also stop/recreate an isolated service host with a known child and
  prove the unrelated detached command survives; use temporary SASE state, bounded
  handshakes, and unconditional cleanup. Reuse existing scope tests for the systemd
  escape contract; platform-specific probes should skip explicitly when unavailable. A
  generic child payload is sufficient, but use the real ToolRun/proc submission path so
  the integration covers the reported case.

Primary regression locations are `tests/ace/tui/test_update_restart.py`,
`test_update_run_actions.py`, `test_plugins_browser_pane_sase_update.py`,
`test_proc_actions_session_workers_overlay.py`, `test_feature_flags_pane.py`,
`test_feature_flags_pane_rendering.py`, update-panel/tooltip tests, and
`tests/ace/tui/actions/test_lifecycle_quit_confirm.py`. Existing process harnesses are
in `tests/test_procs_supervisor.py`, `tests/test_procs_oneshot.py`,
`tests/tool/test_inline_escalation.py`, `tests/test_detach_scope.py`, and
`tests/service/`.

Before implementation verification, read the required `tui.md`, `tui_perf.md`, and
`lint_and_test.md` through `sase memory read`. Run the focused regressions through
`sase tool run test` (use `SASE_PYTEST_WORKERS` rather than a pytest `-n` argument),
then format and run the required `sase tool run check`. Do not run `check-full`. For
changed rendered copy, read `tui_screenshot.md` and run targeted
`just fix-tui-screenshots -- <selectors>` with the established tooling; inspect the
report and any golden diffs. Use the monitor/finalizer skills if final verification
needs a handoff. Finish with the actual results and any platform-specific limits.
