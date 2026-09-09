---
tier: tale
title: Fix AXE daemon dying with its launching tmux pane's systemd scope
goal:
  The axe orchestrator survives closure of the tmux pane/session that launched it,
  signal-driven shutdowns are journaled, and doctor/status warn when the daemon sits in
  a doomed cgroup.
size: medium
proposed_by: bbugyi200.athena.0h6
create_time: 2026-09-09 19:52:56
status: wip
---

# Fix AXE daemon dying when its launching pane's systemd scope is torn down

## Root cause (diagnosed 2026-09-06)

The AXE orchestrator keeps dying "on its own" because it never escapes the systemd
cgroup of whatever process launched it.

Verified evidence chain on athena:

- `start_axe_daemon_result()` in `src/sase/axe/_process_start.py` spawns the daemon with
  `subprocess.Popen(..., start_new_session=True)`. `setsid` detaches the controlling TTY
  and session, but the daemon **stays in the launcher's systemd cgroup**.
- On athena every tmux pane runs inside a transient `tmux-spawn-<uuid>.scope` user unit
  with `KillMode=control-group`. AXE is habitually (re)started from tmux panes
  (`ace --restart-axe`, `sase axe start`), so the orchestrator and all ~10 lumberjack
  processes land in the launching pane's scope (confirmed live:
  `/proc/<orchestrator-pid>/cgroup` → `user@1000.service/tmux-spawn-….scope`).
- When that pane closes, systemd stops the scope and kills every process left in its
  cgroup. The orchestrator's SIGTERM handler (`_handle_shutdown` in
  `src/sase/axe/orchestrator.py`) only sets `_running = False` and terminates children —
  it appends **no lifecycle journal event** — so the death is silent. The lifecycle
  journal (`~/.sase/axe/lifecycle.jsonl`) confirms this: repeated `start` events with no
  intervening `stop` (e.g. 2026-09-06 18:20:42 then 18:21:19), and the systemd user
  journal shows matching `tmux-spawn-*.scope … Consumed …` teardown lines (including one
  week-old scope with 1w CPU time torn down 2026-09-06 18:29).
- The optional heal timer (`sase axe ensure install`) is not installed on athena, so
  nothing restarts a downed AXE until the next `ace` launch or a waiting agent runner's
  ensure call.

A related latent bug: the ensure watchdog's oneshot `sase-axe-ensure.service`
(`src/sase/axe/_ensure_timer.py`) heals by starting the daemon _inside the oneshot
service's cgroup_; when that unit deactivates, systemd (default
`KillMode=control-group`) kills leftover processes, so a watchdog-healed daemon would be
killed moments after being started. The fix below repairs this path too, because the
daemon always escapes into its own scope regardless of the launch context.

## Fix

### 1. Spawn the daemon into its own transient systemd user scope

In `src/sase/axe/_process_start.py`:

- Add a helper that decides whether systemd scope wrapping is available: Linux,
  `shutil.which("systemd-run")` resolves, and not explicitly disabled. Do not add a
  config/flag surface; availability probing plus graceful fallback is the whole policy.
- When available, prefix the daemon command built by `_build_axe_start_command()` with:

  ```
  systemd-run --user --scope --quiet --collect
      --unit=sase-axe-<uniquifier>
      --description="SASE axe orchestrator"
      --
  ```

  Scope mode (`--scope`) is required, not service mode: in scope mode `systemd-run`
  fork/execs the command as its own direct child, so the inherited lifecycle-lock FD
  handoff (`pass_fds=(lifecycle_lock.fd,)` + `AXE_LOCK_FD_ENV`) keeps working. Service
  mode spawns via the user manager and would break FD inheritance. `--collect` prevents
  failed unit residue; the `<uniquifier>` (e.g. epoch milliseconds) prevents name
  collisions with a lingering previous scope. Verified on athena that
  `systemd-run --user --scope` places the child under
  `user@1000.service/app.slice/run-*.scope`, fully outside the pane scope.

- Fallback: if `systemd-run` is unavailable, or the wrapped spawn exits before
  publishing a PID (the existing `_wait_for_daemon_start` + `process.poll()` failure
  branch), retry the spawn once unwrapped so non-systemd hosts (macOS, containers) and
  broken user managers behave exactly as today. Guard the retry so it happens at most
  once and only for the wrapped attempt.
- `_wait_for_daemon_start` currently returns the PID read from PID files, not
  `process.pid` — with the wrapper, `process.pid` is `systemd-run`'s PID, so keep all
  success/liveness decisions based on the published PID file (already the case; just do
  not introduce any new reliance on `process.pid`).

### 2. Journal signal-driven shutdowns

In `src/sase/axe/orchestrator.py`:

- Record which signal (if any) triggered shutdown in `_handle_shutdown` (store the
  signum on the instance; keep the handler async-signal minimal).
- After the main loop exits in `run()`, when shutdown was signal-initiated, append a
  lifecycle event (`append_lifecycle_event`) — event `stop`, outcome `terminated`,
  source `signal`, reason naming the signal and the orchestrator PID. This makes any
  future external kill (scope teardown, OOM-adjacent TERM, manual kill) visible in
  `lifecycle.jsonl` instead of silent. A normal `sase axe stop` will now produce both
  its own CLI `stop` event and this `terminated` event; that duplication is acceptable
  and informative (different sources).

### 3. Detect a doomed cgroup and warn

- In `src/sase/doctor/checks_axe.py`, add a check: when an orchestrator PID is live and
  `/proc/<pid>/cgroup` parses as a systemd v2 hierarchy, warn if the daemon's unit is a
  session-tied scope (unit name matching `tmux-*` / `session-*.scope`, or more generally
  any scope whose name does not start with `sase-axe`) while `systemd-run` is available.
  Message should say the daemon will be killed when the launching pane/session closes
  and to restart axe. Skip silently (pass) when the cgroup file is missing/unparseable
  or the host lacks systemd.
- Surface the same condition as an Attention entry in `sase axe status`
  (`src/sase/axe/status_render.py` feeds from the status snapshot builder; follow the
  existing pattern used for the stale-heartbeat warning).

### 4. Tests

- `_process_start` unit tests: command wrapping when `systemd-run` is present
  (monkeypatched), no wrapping when absent, unwrapped retry after a wrapped early-exit
  failure, and lock-FD passing unchanged. Follow the existing daemon-start test style
  (the suite already covers `test_run_writes_pid_and_releas*`, wedged-lock recovery,
  etc.).
- Orchestrator test: SIGTERM-driven exit appends the `terminated` lifecycle event.
- Doctor test: cgroup parsing + warn/pass matrix (tmux scope → warn, `sase-axe-*` scope
  → pass, no systemd → pass).

## Non-goals / alternatives considered

- Converting AXE into a permanent `sase-axe.service` user service with `Restart=always`
  was rejected for now: AXE lifecycle is deliberately desired-state
  - CLI/ACE controlled (see the ensure watchdog design); the minimal scope-escape keeps
    that architecture intact. A future decision record could revisit this.
- No new CLI options, config keys, or feature flags: fallback is automatic, so no flag
  lifecycle is needed.

## Verification

- `just check` in the workspace (per the repo's two-speed verification rule) plus the
  targeted new tests.
- Manual smoke on athena after landing: `sase axe stop && sase axe start` from a tmux
  pane, then confirm `cat /proc/$(cat ~/.sase/axe/orchestrator.pid)/cgroup` shows a
  `sase-axe-*` scope (not `tmux-spawn-*`), and `sase doctor` no longer warns.

## Operator follow-up (not part of the code change)

After this lands, running `sase axe ensure install` on athena is recommended so a future
silent death (any cause) heals within 5 minutes; with the scope escape in place the
healed daemon survives the oneshot service teardown.
