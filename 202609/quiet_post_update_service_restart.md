---
tier: tale
title: Quiet the post-update service-host restart in ACE
goal:
  An update-driven ACE restart (e.g. ,E) restarts the service host without a false
  "Services unhealthy" toast, while real post-restart failures, host shutdown status,
  and lifecycle command results are reported truthfully.
size: medium
proposed_by: bbugyi200.athena.0s1
create_time: 2026-09-25 09:17:19
status: wip
---

# Plan: Stop the false "Services unhealthy" toast after an update-driven restart

## Problem

After `,E` (Update Everything, auto-approved) changes code, ACE restarts itself **and**
the service host. The user sees a warning toast, `Services unhealthy: gateway stopped`,
every time, and the service comes back a moment later. Restarting the service host after
a code update is correct and must stay: the host process and every service proc
(`gateway`, `scheduler`, `telegram_receiver`) run Python from the editable install, so
they only load the new code after a restart. The problem is that ACE reports its own
planned, in-progress restart as an outage.

## Root cause (verified)

Here is the full chain, confirmed against the code, the systemd journal
(`Stopping sase.service` 08:53:14, `Stopped/Started` 08:53:27; screenshot 08:53:32), and
a direct `build_service_status` + `derive_service_health` simulation:

1. `action_update_everything_shortcut` → comprehensive update → `code_changed` →
   `restart_after_update` (`src/sase/ace/tui/update_restart.py`) →
   `_restart_tui(restart_axe=True)` → re-exec `sase tui --restart-service`.
2. The new TUI's `_run_axe_startup_init_body`
   (`src/sase/ace/tui/actions/axe_display/_refresh_full.py`) sees `_restart_axe` and
   awaits `restart_service_host()` → native `systemctl --user restart sase.service`. It
   throws away the result and does not mark the restart as an expected transition.
3. **Misleading shutdown snapshot.** On SIGTERM the old host runs `_stop_all_children()`
   (which clears `_children`/`_pending`), then `write_current_host_status(host)`
   (`src/sase/service/host_lifecycle.py`). `_write_host_status`
   (`src/sase/service/host_reporting.py`) always builds a _live_ host observation
   (`lock_held=True, pid_alive=True`, fresh heartbeat) and no child observations, so the
   final `status.json` says **host `running`, every desired proc `stopped`**. Simulated:
   `ServiceHealth(healthy=False, summary='gateway stopped')`. `gateway` comes first
   alphabetically, which is why the toast always names it.
4. `persisted_or_current_status()` (`src/sase/service/control.py`) trusts any snapshot
   up to 15s old, and the TUI re-reads `status.json` when it changes and also right
   after the restart call returns. So the TUI loads that final snapshot before the new
   host's first reconcile writes an all-running one (~1-3s later).
5. `_push_service_health` (`src/sase/ace/tui/actions/axe_display/_render_footer.py`)
   sees healthy → unhealthy and fires the toast. The footer pill also turns to a red
   `!`, because the health pill outranks the existing `RESTARTING` pill.
6. **Contributing defect:** the native lifecycle command runs through
   `platform_runner.default_runner`, which has a hard `timeout=10`. The host's graceful
   stop is sequential per child (up to `stop_timeout_seconds` 10s + 5s kill wait each).
   The journal shows 2-14s stops (13s for the incident, 14s on 2026-09-24 18:16). So
   `restart_service_host()` often returns `ok=False` "timed out" while systemd finishes
   the restart on its own. `sase service restart`/`stop` misreport failure the same way.
   The non-native path has the same problem: `stop_service_host` waits only 15s
   (`_STOP_WAIT_SECONDS`), and when that expires `restart_service_host` returns early
   and never starts the host again.

The fix has three contained parts. Part A alone removes the toast. Part B makes the data
honest for every consumer. Part C makes the restart call report success or failure
truthfully, which Part A relies on for its error toast.

## Part A — ACE treats its own service-host restart as an expected transition (primary fix)

Presentation and TUI glue only. No Rust/core change: other frontends do not share this
TUI-initiated flow.

1. Add a small pure module for the transition, e.g.
   `src/sase/ace/tui/actions/axe_display/_service_restart_transition.py`:
   - a frozen dataclass
     `ServiceRestartTransition(requested_at: float, deadline: float)`. `requested_at` is
     wall clock (`time.time()`) taken just before the restart request; `deadline` is
     `time.monotonic()`-based.
   - `restart_transition_settled(transition, snapshot, health) -> bool`: True only when
     `health.healthy` **and** the snapshot comes from a host generation started after
     the request
     (`snapshot.host.started_at is not None and snapshot.host.started_at >= transition.requested_at`).
     The generation check is required. Without it, the old host's pre-stop all-running
     snapshot, which the TUI can still read during the stop window, would settle the
     transition too early, and the later shutdown snapshot would toast.
   - constants: a post-start grace (about 20s) and a hard cap equal to the Part C
     lifecycle timeout plus that grace.
2. Add `_service_restart_transition: ServiceRestartTransition | None`. Initialize it to
   `None` next to `self._service_status = None` in
   `src/sase/ace/tui/actions/_state_init_late.py`, and annotate it next to
   `_service_status` in `src/sase/ace/tui/actions/axe_display/_loader_state.py`.
3. In `_run_axe_startup_init_body`'s restart branch:
   - Before the await, begin a transition with `requested_at=time.time()` and
     `deadline=monotonic()+hard_cap`.
   - Wrap `await asyncio.to_thread(restart_service_host)` in `try/except Exception`.
   - On `result.ok`, if the transition is still active and has not settled, narrow its
     deadline to `monotonic()+post_start_grace` and `set_timer(post_start_grace, ...)` a
     thin synchronous expiry callback.
   - On a not-ok result or an exception, clear the transition and `notify` an error
     toast carrying the result message, e.g. `Service host restart failed: <message>`.
     This is currently discarded silently.
   - Keep the existing `_schedule_axe_async_refresh()` call.
   - Follow the file's `getattr`-guarded call style, because this base mixin sits below
     the footer mixin in the MRO.
   - Leave the auto-start (`start_service_host`) branch unchanged.
4. Expiry callback, e.g. `_expire_service_restart_transition(transition)`: it must be
   thin and synchronous per tui_perf rule 2, with no disk I/O. If
   `self._service_restart_transition is transition`, clear it and re-push the cached
   health via `_update_axe_keybinding()`. That call only touches memory, so an unhealthy
   state toasts even when no new snapshot arrives. Otherwise do nothing, because the
   transition already settled or was replaced.
5. `_push_service_health`:
   - While a transition is active:
     - If it has settled, clear it and continue with the normal logic. Health is healthy
       at that point, so no toast.
     - Else, if `monotonic() < deadline`, push the pill in a restarting presentation
       (see 6) and **return without toasting and without updating
       `_service_health_notified`**. Keeping the pre-restart healthy signature means a
       real failure still toasts after the window ends.
     - Else (expired), clear it and fall through to the normal logic. An unhealthy
       snapshot then toasts exactly once, as it does today.
   - Behavior without a transition must stay byte-for-byte the same.
6. Footer pill (`src/sase/ace/tui/widgets/_keybinding_status.py`):
   - Let `set_service_health` accept `restarting: bool = False`. Store it and include it
     in `_status_signature()`.
   - In `_get_status_text`'s service-health branch, render the existing `RESTARTING`
     pill style (the one used for `_axe_restarting`) when it is set, instead of `!` or
     `n/m`.
   - Do not change the precedence of the existing
     `_axe_restarting/_axe_starting/_axe_stopping` flags, and do not touch
     `_refresh_collect`'s flag clearing.

## Part B — the host's final shutdown snapshot reports the host as stopped

1. Give `write_current_host_status` / `_write_host_status`
   (`src/sase/service/host_reporting.py`) a keyword such as `host_exited: bool = False`.
   When it is True, build
   `ServiceHostObservation(record=None, lock_held=False, pid_alive=None, platform_unit=unit)`
   instead of the live record. Verified with the binding: this derives host `stopped`,
   keeps `platform_unit`, and leaves every proc's `desired`/`stop` unchanged. So
   `scheduler_desired_state()` (which reads only proc `desired`/`stop`) is unaffected.
   Do **not** use a record with `pid_alive=False`; that derives `stale`.
2. In `run_host` (`src/sase/service/host_lifecycle.py`), the normal-exit path after
   `_stop_all_children()` calls `write_current_host_status(host, host_exited=True)`.
   Leave the per-tick reconcile writes and the `KeyboardInterrupt` path unchanged.
3. Why it is safe: under both systemd and the detached path, the new host starts only
   after the old one has exited and released the lifetime lock, so its first reconcile
   write always replaces this snapshot. `chat_install`'s "host not running → start host"
   check becomes correct: it used to skip a start because a just-exited host was
   reported `running`, and native `systemctl start` is idempotent anyway.
   `sase service status` right after `sase service stop` stops claiming
   `Service host: running`.

## Part C — lifecycle commands wait long enough for a graceful host stop

1. Add one shared budget constant, e.g.
   `SERVICE_HOST_LIFECYCLE_TIMEOUT_SECONDS = 120.0`, in
   `src/sase/service/platform_runner.py` or `platform_models.py`. Document why 120s: it
   covers systemd's default 90s `TimeoutStopSec` (after which systemd SIGKILLs the host)
   plus start, and the host's worst-case sequential child stop.
2. Give `default_runner` an optional `timeout` keyword that defaults to today's 10s, so
   every other caller and the `CommandRunner` signature stay the same. In
   `control_installed_service` (`src/sase/service/platform_definition.py`), when no
   runner is injected, use `functools.partial(default_runner, timeout=<budget>)` for the
   start/stop/restart commands on both linux (`systemctl`) and darwin
   (`darwin_bootout`/`darwin_bootstrap`). An injected runner is still used as-is.
3. Raise the non-native `_STOP_WAIT_SECONDS` in `src/sase/service/control.py` to the
   same budget, so a slow graceful stop no longer makes `restart_service_host` return
   before starting the new host. Leave `_START_WAIT_SECONDS` unchanged.

## Tests

- New `tests/ace/tui/test_service_restart_transition.py`, using a fake footer that
  records `set_service_health` calls and a recording `notify`, driven through the real
  footer mixin method:
  - Pre-restart healthy state, then a transition begins, then the old host's shutdown
    snapshot (Part B shape and the old "running + all stopped" shape) is pushed: no
    toast, and the pill is in the restarting presentation.
  - An old-generation healthy snapshot during the stop window does **not** settle the
    transition. A later shutdown snapshot still does not toast.
  - A new-generation (`started_at >= requested_at`) healthy snapshot settles and clears
    the transition with no toast.
  - After the deadline passes (monkeypatch `time.monotonic`), the expiry callback clears
    the transition. An unhealthy cached snapshot then toasts `Services unhealthy: ...`
    exactly once. A stale expiry callback for an already-settled or replaced transition
    does nothing.
  - With no transition, the existing toast behavior is unchanged.
- `tests/ace/tui/test_axe_startup_host_lifecycle.py`:
  - Make the `restart_service_host` fakes return result objects with `ok`/`message`. Add
    the harness stubs the new code needs (`notify`, `set_timer`, transition hooks).
  - Assert that a restart begins a transition, that an ok result schedules the expiry
    timer, and that a not-ok result or an exception clears the transition and emits one
    error toast. The existing parametrized start/restart expectations must still pass.
- `tests/ace/tui/test_services_phase_closure.py`: a restarting health pill renders
  `RESTARTING` (not `!`), and the flag changes `_status_signature()`.
- `tests/service/`:
  - Extend `test_service_host_lifetime.py`'s SIGTERM scenario, or add a focused test.
    After `run_host` returns, `read_service_status()` shows `host.state == "stopped"`.
  - Add a unit test that `write_current_host_status(..., host_exited=True)` produces a
    stopped host and keeps the proc desired states.
- `tests/service/test_service_platform_linux.py` (and darwin if it is cheap):
  `control_installed_service` with no injected runner passes the lifecycle budget
  timeout. Monkeypatch `platform_definition.default_runner` to capture the kwargs.
  Injected runners are still called unchanged.
- Run `just fix`, then `just check` (through `sase tool run check`). No PNG golden
  change is expected, because the pill only changes during a restart. If a visual test
  does move, follow the memory's `just fix-tui-screenshots` inspection rules.

## Out of scope / notes

- Restarts started outside this TUI (a CLI `sase service restart`, a crash restart,
  another TUI) still toast honestly (`host stopped` or a proc name). Only ACE's own
  `--restart-service` restart is treated as expected.
- The 2-14s graceful host stop suggests at least one service proc ignores SIGTERM until
  its 10s `stop_timeout_seconds` SIGKILL (the Telegram 30s long-poll is a likely
  suspect). Capture this as a follow-up task bead with `/sase_new_task` instead of
  fixing it here.
- `derive_service_health` stays unchanged, apart from the footer receiving the
  `restarting` presentation flag.
