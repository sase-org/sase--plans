---
tier: tale
title: Stop pytest from leaking detached sase service hosts
goal:
  Leaked pytest service hosts are terminated, start_service_host and sase service run
  refuse to spawn real hosts under pytest, and athena's load, swap, and routine process
  counts return to normal.
size: medium
proposed_by: bbugyi200.athena.0oo
create_time: 2026-09-21 13:56:04
status: wip
---

# Stop pytest from leaking detached `sase service run` hosts (athena resource spike)

## Diagnosis (evidence gathered 2026-09-21 ~13:20 on athena)

The user suspected the root disk (`/`, 94% used) was causing the resource spike. That is
**mostly not the cause**:

- `/` has 875G total, 810G used, and ~56G free. Plenty of headroom; no ENOSPC symptoms.
  The biggest consumer is `~/.cache/sase/tmp/cargo-targets` (148.7G, 570 per-launch
  entries), which already has an owner-delegated reaper (`sase disk reap`).
- The machine is thrashing: load average ~75-92 on 64 cores, 40G of 64G swap in use, IO
  pressure `full avg300≈10%`, and ~1670 tasks.

**True root cause: 64 leaked `sase service run` hosts spawned by pytest.** There are 65
live `sase service run` processes. Only one (the real host) has `HOME=/home/bryan`. The
other 64 carry pytest sandbox homes
(`/var/tmp/sase-*/pytest-of-bryan/pytest-N/.../homeNNN` or
`~/.cache/sase/tmp/agent-tmp/<scratch>/pytest-of-bryan/...`), plus `PYTEST_CURRENT_TEST`
and `SASE_PYTEST_SANDBOX_DIR` in their environments. Each leaked host runs its own
`sase scheduler run` orchestrator. That orchestrator holds an `orchestrator.lock` inside
a now-deleted sandbox home, so it never contends with the real one, and it spawns the
full routine set (`hooks`, `waits`, `housekeeping`, `checks`, `comments`,
`external_mirror`). The totals are 574 processes, ~16G RSS plus ~24.5G swap, and ~20
cores of continuous routine polling. The oldest leak is ~20h old.

All 64 leaks come from four TUI tests, identified via `PYTEST_CURRENT_TEST`:

- `tests/ace/tui/test_startup_stopwatch_live_update.py::test_slow_mount_state_read_does_not_block_app_key_dispatch`
  (19)
- `tests/ace/tui/test_residual_freeze_soak.py::test_lowered_threshold_soak_keeps_fixed_paths_responsive`
  (15)
- `tests/ace/tui/test_artifacts_agents_loading.py::test_agent_first_page_paints_before_full_extension`
  (15)
- `tests/ace/tui/test_agents_pane_mount.py::test_agents_pane_mounts_activates_and_loads`
  (15)

Mechanism: commit `ef9900990` ("remove the service_host beta flag and its Off branches",
2026-09-20 16:07) made `_run_axe_startup_init_body` in
`src/sase/ace/tui/actions/axe_display/_loader_refresh.py` call `start_service_host()`
unconditionally whenever `_auto_start_axe` is true and no host is running. `AceApp()`
defaults to `auto_start_axe=True`. Before the removal, tests ran with the flag Off, so
this path never ran. In a pytest sandbox no host is ever running, so each such test
reaches the detached fallback in `start_service_host()` (`src/sase/service/control.py`).
That fallback `subprocess.Popen`s `sase service run` with `start_new_session=True`. The
suite sets `SASE_DETACH_SCOPE_DISABLE=1`, so there is no systemd scope, and the host is
reparented to the user systemd manager or init. pytest teardown never reaps it. It lives
forever and polls against its deleted sandbox home.

The native lifecycle path already refuses under pytest:
`src/sase/service/platform_runner.py::_service_lifecycle_blocked_in_tests`, using
`pytest_context_detected` plus the `SASE_SERVICE_ALLOW_LIFECYCLE_IN_TESTS` override from
`src/sase/service/platform_models.py`. The detached Popen fallback bypasses that guard.

Note: `as-symlink`, the tool the user mentioned for freeing space, does not exist on
athena. It is not on PATH, not in the chezmoi source, and not under `~`. Because low
disk space is not the root cause, this plan does **not** move data behind symlinks.

## Step 1 — Immediate remediation: terminate the leaked host trees (ops, no code)

1. Enumerate candidates with `pgrep -f 'sase service run'`. For each PID, read
   `/proc/<pid>/environ`. Classify a process as **leaked** only if its environment
   contains `PYTEST_CURRENT_TEST` (or `SASE_PYTEST_SANDBOX_DIR`) **and** its `HOME` is
   not `/home/bryan`. Never touch the host whose `HOME=/home/bryan`; that is the real
   service host, and its PID matches `~/.sase/service/host.lock`.
2. Each leaked host was started with `start_new_session=True`, so it leads its own
   session/process group together with its scheduler and routine children. Before
   signalling, confirm that `ps -o sid= -p <pid>` equals the PID. Then send `SIGTERM` to
   the whole group (`kill -TERM -- -<pid>`), wait ~15s, and `SIGKILL` the group for any
   survivors.
3. Also terminate any orphaned `sase scheduler run` or `sase axe routine run <name> ...`
   process whose environment matches the same pytest criteria. Afterwards, exactly one
   `sase service run`, one `sase scheduler run`, and one of each routine should remain.
4. Verify and record before/after numbers in the final response: `uptime` load, swap
   used (`free -h`), `pgrep -fc 'sase axe routine run'` (expect ~10, down from ~425),
   and `pgrep -fc 'sase service run'` (expect 1). Also check that `sase service status`
   still reports the real host healthy.

Do not kill the ~65 `run_agent_runner.py` processes. They are real agents parked in
`wait_for_dependencies` and use little CPU on average.

## Step 2 — Code fix: the detached host fallback must refuse to spawn under pytest

In `src/sase/service/control.py::start_service_host`, add a guard before building the
`sase service run` argv and calling `subprocess.Popen`. Reuse the existing pytest
lifecycle policy rather than inventing a new one:

- Share `_service_lifecycle_blocked_in_tests` from
  `src/sase/service/platform_runner.py`. Promote it to a public helper, for example
  `service_lifecycle_blocked_in_tests`, and keep the old name only if other callers need
  it. Check Symvision rules for public symbols.
- When the helper reports blocked, return
  `_ServiceHostActionResult(ok=False, changed=False, message=SERVICE_LIFECYCLE_TEST_BLOCK_MESSAGE)`
  without spawning anything. Honor the `SASE_SERVICE_ALLOW_LIFECYCLE_IN_TESTS=1`
  override exactly as the native path does, so isolated lifecycle tests can still opt
  in.
- Add defense in depth at the host entrypoint as well. `run_service_host`
  (`src/sase/service/host.py`), and therefore `sase service run`, should exit non-zero
  with the same block message when the same policy says it is blocked. This catches any
  other spawn path, such as `mobile_gateway` or `chat_install`, that would launch a real
  host from a test process. First confirm that tests which exercise the host in-process
  (`tests/service/test_service_host_scenarios.py` and similar) either set the override
  or stub the entrypoint. Adjust those tests if needed.

Keep the Rust-core boundary in mind. This is process-spawn policy local to the Python
service control plane, so it stays in Python and needs no `sase-core` change.

## Step 3 — Regression tests

- Add a test (for example under `tests/service/`) proving that `start_service_host()`
  under pytest, without the override, returns `ok=False` with the block message and
  never calls `subprocess.Popen`. Assert this by monkeypatching `subprocess.Popen` in
  `sase.service.control` to fail the test if called. Also cover the override-enabled
  case, where Popen is reached (stubbed).
- Add a test proving that `sase service run` / `run_service_host` refuses under pytest
  without the override.
- Review the existing `start_service_host` tests (`tests/test_mobile_gateway.py`,
  `tests/test_chat_install.py`, `tests/ace/tui/test_axe_startup_host_lifecycle.py`,
  `tests/ace/tui/actions/test_service_host_keys.py`,
  `tests/service/test_service_host_scenarios.py`). Keep them passing: they mostly
  monkeypatch `start_service_host`. Set the override only where a test genuinely drives
  the real fallback.
- Optional hardening, if cheap: pass `auto_start_axe=False` in the four leaking TUI
  tests listed above, since none of them is about host startup. The guard is the real
  fix; this only removes the wasted startup call.

## Step 4 — Safe, owner-delegated disk cleanup (secondary)

Disk is not the root cause, but `cargo-targets` scratch is large. Use the existing owner
cleanup and nothing hand-rolled:

1. Run `sase disk reap --help` to learn the flags. Then preview the reap (dry-run
   default) and report the reclaimable bytes per owner.
2. Apply the default owner reap only for managed tmp build scratch past its configured
   horizon. The reaper already checks launch liveness, so never delete
   `cargo-targets/<key>` for live agents by hand. Report `df -h /` before and after.
3. Do not move data to other disks or create symlinks. `as-symlink` does not exist on
   this host, and the user asked for it only if low disk space were the real cause.

## Verification

- Follow the lint/test memory (`sase memory read lint_and_test.md`) and run
  `just check`.
- After `just check` completes, confirm that the run leaked no hosts: no
  `sase service run` process whose environment contains `PYTEST_CURRENT_TEST` should
  exist. Before the fix, every `just check` added about four.
- Report the before/after load, swap, and process counts from Step 1, plus the disk
  numbers from Step 4.
