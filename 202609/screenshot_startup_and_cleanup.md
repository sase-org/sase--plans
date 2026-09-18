---
tier: tale
title: Fix live screenshot startup settling and owned tmux cleanup
goal:
  Capture a settled live TUI within the overall deadline and leave no newly created
  placeholder session after success or failure.
size: medium
proposed_by: bbugyi200.athena.0mr
create_time: 2026-09-18 08:00:59
status: wip
---

# Fix live screenshot startup settling and owned tmux cleanup

## Scope and context

Implement this as one medium-sized tale. The defects are bounded to the Python live
Textual export and tmux automation paths, with shared integration tests. No Rust domain
behavior, new CLI flags, renderer changes, memory edits, or unrelated TUI performance
overhaul is needed. Preserve the existing SIGUSR2 request-directory protocol, canonical
PNG renderer, unique tmux targets, remote transport, `--keep`, and `--window` contracts.

The user reported plain `sase screenshot` failing with
`wait for settled frame: RuntimeError: timed out waiting for screenshot refresh; deadline=3.000s`,
while the pane still displayed `Loading...` and the startup stopwatch. The
`sase_ace_agents` session survived. Context is the closed `sase-123` epic and its
audited original plan, `plan:202609/tui_agent_screenshots.md`. The relevant descendant,
`sase-123.7.6.4`, added bounded refresh waits and finite visual worker detection in
commit `80ba6ce0ef`; preserve those protections. This is a repair of the shipped
behavior, not a request to reopen or close the original epic.

## Diagnosis and investigation evidence

1. **The live refresh barrier silently disappears.**
   `src/sase/main/ace_handler.py::_run_ace_app` runs `app.run_async()` directly. In the
   installed Textual implementation, `run_test()` assigns `App._task`, but the live path
   does not. Instrumented real tmux captures confirmed `app._task is None`.
   `DOMNode.wait_for_refresh()` asserts that `_task` exists.
   `ScreenshotExportMixin._wait_for_screenshot_refresh_if_available()` catches that
   assertion and returns, so successful headless tests do not establish a working live
   refresh barrier. Five to twenty-six nominal refresh waits in observed live attempts
   completed without a real barrier; their successful SVGs still contained `Loading...`
   and `starting`.
2. **Startup, settling, and the CLI deadline are disconnected.**
   `src/sase/screenshot/local.py::_wait_for_startup_frame` accepts two identical
   nonblank pane captures 50 ms apart. A loading screen qualifies. The exporter checks
   selected finite workers but not visible startup readiness, gives each attempt three
   seconds, and the CLI retries only the old Textual assertion error. A recoverable
   settle timeout therefore fails the entire nominally 30-second capture. Do not claim
   the supplied output identifies one particular slow loader: the exact host scheduling
   at that failure is unavailable.
3. **The error can misidentify nonconvergence as a stalled refresh.** The convergence
   loop sleeps to its deadline and enters the next refresh before checking expiry. An
   inline deterministic probe with instantly completing refreshes and continuously
   changing SVGs reproduced `timed out waiting for screenshot refresh` after five
   exports. Thus this message alone is not evidence of a blocked refresh callback.
4. **Cleanup owns only the TUI window.**
   `src/sase/main/ace_tmux.py::_resolve_or_create_agent_session` creates an independent
   `placeholder` shell window. The local capture `finally` removes its TUI window and
   claim, leaving the shell and session alive. Multiple real private-socket captures
   reproduced `sase_ace_agents @0 placeholder zsh` after successful capture as well.
   Session default-size setup also currently precedes the window cleanup guard.

All real probes used isolated tmux sockets and removed their own servers afterward; no
user's existing session was killed. No implementation files were changed. An in-memory
experiment replaced the ineffective wait with a `call_after_refresh` event, counted
`_startup_visible_surface_ready()` as pending finite work, and retried known settle
timeouts under the original overall deadline. It captured a loaded SVG without
`Loading...` or `starting` in about 20 seconds, after four three-second attempt
timeouts. This validates the repair direction, not a finished patch or a guarantee about
every host's startup time.

## Implementation

### 1. Establish a real bounded live refresh barrier

Update `src/sase/ace/tui/actions/screenshot_export.py` so the export task schedules a
thin synchronous post-refresh callback and awaits its acknowledgement outside Textual's
serial message pump. `call_after_refresh(event.set)` is the proven direction and does
not require changing Textual's private `App._task`. Register the acknowledgement before
requesting the refresh, handle rejected scheduling explicitly, and bound the await by
the existing attempt deadline. A canceled or expired acknowledgement must be safe if its
callback arrives later.

Remove the assumption that swallowing a live refresh assertion proves a refresh
occurred. Keep SVG creation and widget mutations on the UI thread, request/response file
I/O off-thread, cursor restoration in `finally`, and export tasks registered for
shutdown cancellation. Preserve sequence-numbered atomic `.svg`, `.done`, and `.error`
responses; unsuccessful or canceled exports must not appear successful or strand a live
request indefinitely.

### 2. Let finite startup and settling use the overall capture budget

Treat first-load readiness of the captured visible surface as finite visual work. Reuse
the startup readiness state in `_startup_telemetry.py` and the existing Agents/Services
first-load flags; account for a tab changed by requested keys. Do not wait for hidden
tabs, every worker, periodic refreshes, update checks, or all-host quiescence. Keep
waiting for the already supported detail debouncers, hydration tasks, explicitly visual
workers, and bounded transient animations. Export only after a real refresh and the
existing stable-frame requirement.

In `src/sase/screenshot/local.py`, retry only recognized readiness/refresh/ convergence
timeout outcomes through `_request_export_with_retries`, using its existing sequence
tracking, backoff, and single monotonic `_Deadline`. Do not reset the overall deadline,
relaunch for every attempt, repeat user keypresses, or retry permanent export/rendering
failures. Preserve the old assertion retry if needed for existing retained-window
compatibility. A three-second attempt remains bounded; increasing that constant alone is
not the fix.

Check convergence expiry before starting another refresh. Distinguish an actual
unacknowledged refresh from exhausted convergence/startup time, and include useful
pending-work labels and frame diagnostics. When the overall budget expires during
retries, report that overall timeout together with the last meaningful attempt failure
and the last known pane capture. Keep diagnostic and cleanup overhead separately
bounded. Preserve the remote command's bounded use of this same local capture path; do
not add a second settling implementation.

### 3. Own and release bootstrap tmux windows

Refactor the agent-session creation path in `src/sase/main/ace_tmux.py` to retain
explicit ownership of a newly created bootstrap window, including its unique window ID
and a unique temporary name for recovery if tmux creates it but the client times out
before reading the ID. Remove that owned bootstrap once the TUI window exists. Removing
the TUI's final window will then let tmux remove an empty session naturally, including
after export failure. `--keep` retains the TUI, not an unnecessary bootstrap shell.

Place session creation, size setup, window creation, metadata/resize setup, and
ownership transfer under cleanup guards. On failures before a usable TUI window is
returned, remove only this invocation's bootstrap and TUI resources and release its
claim. Keep cleanup bounded and retain the primary error. Session creation races must
re-inspect/retry within the deadline: another creator may win, or a session may
disappear between inspection and creating a window.

Never solve this with unconditional `kill-session`, a list-then-kill shared-session
race, or deleting a window merely because it is named `placeholder`. Preserve
preexisting sessions/windows, concurrent captures, renamed windows, retained windows,
and `--window` targets. Historical unowned leftovers are not safe to delete by name.
Preserve ordinary `sase tui --tmux` behavior through the shared launcher and its
existing tests.

## Regression and acceptance checks

Extend existing tests rather than building a second harness:

- `tests/ace/tui/test_screenshot_export.py`: exercise a real Textual app with the live
  `_task` condition, prove a delayed refresh is actually acknowledged before export, and
  keep key events responsive while waiting. Cover rejected/stalled callbacks, late
  callbacks after cancellation, marker completion, cursor restoration, and changing
  frames producing convergence diagnostics. Existing finite-worker and
  ignored-background-worker tests must continue to pass.
- Cover delayed visible startup that outlasts one export attempt but completes within
  the overall command budget. The resulting SVG must contain the loaded sentinel rather
  than the loading sentinel. Use events and short test budgets for deterministic units,
  plus one real default-budget smoke capture. Hidden work must not block a ready visible
  surface.
- `tests/main/test_screenshot_command.py` and
  `tests/main/test_screenshot_launch_failures.py`: known transient errors followed by
  success, exhaustion without deadline reset, permanent errors without retry, preserved
  pane/attempt diagnostics, and cleanup on error and interruption.
- `tests/main/test_ace_tmux.py`: private-socket tests must inspect actual sessions and
  bootstrap windows, not just a fake runner's TUI-window dictionary. A fresh server has
  no `sase_ace_agents` session after success or forced failure without `--keep`.
  Exercise size-setup failure, post-create timeout with lost output, concurrent
  creation/capture, preexisting same-named placeholder preservation, unrelated windows,
  retained capture/recapture, and ordinary `tui --tmux`.
- Run the focused export, screenshot command, launch failure, tmux launcher, and
  `tests/main/test_screenshot_remote_command.py` suites. Remote fake-SSH coverage must
  retain unique targets, argument boundaries, failure cleanup, and hints.
- Run a real PNG capture against the working tree on a private socket; inspect the PNG
  itself and verify that the loaded visible view is rendered. Also force an
  overall-timeout failure and inspect session state. Verify `--keep` followed by
  `--window` still works and cannot delete unrelated windows. Remove only test-owned
  resources. Do not claim live image bytes are deterministic goldens.

Read current `lint_and_test.md` and the TUI notes through `sase memory read` before
implementation verification. Run `just fix`, then `just check`. Use `sase_monitor` if
verification is long-running; use it for any required `just check-full`. Broaden to the
visual snapshot lane if the implementation changes shared rendering or golden fixtures,
and inspect differences before accepting intentional changes. Do not blanket-rebaseline
snapshots or conflate unrelated host/test failures with this repair. Report the focused
and governed results, live capture evidence, and any remaining limitation.
