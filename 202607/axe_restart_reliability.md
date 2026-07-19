---
tier: epic
title: Axe restart reliability and lumberjack outage self-healing
goal: 'Axe outages after `sase update` restarts no longer happen silently: restarts
  are retried and verified, a watchdog heals a downed axe, log rotation stops thrashing
  the disk, and every failure is surfaced through the notification inbox instead of
  vanishing.

  '
phases:
- id: log-rotation
  title: Bounded-log rotation hysteresis and temp-file cleanup
  depends_on: []
  description: '''Bounded-log rotation hysteresis and temp-file cleanup'' section:
    stop the at-cap full-file rewrite on every append, and clean up orphaned rotation
    temp files.'
- id: orchestrator
  title: Orchestrator output streaming and crash-loop backoff
  depends_on: []
  description: '''Orchestrator output streaming and crash-loop backoff'' section:
    make child stdout reach aggregate logs promptly and add restart backoff plus loud
    surfacing for crash-looping lumberjacks.'
- id: restart
  title: Verified, journaled axe restart with desired-state marker
  depends_on: []
  description: '''Verified, journaled axe restart with desired-state marker'' section:
    make restart retry and verify startup, record intent in a desired-state marker,
    notify on failure, and journal the restart outcome in dev-update records.'
- id: watchdog
  title: Axe self-healing via sase axe ensure
  depends_on:
  - restart
  description: '''Axe self-healing via sase axe ensure'' section: add an idempotent
    heal command driven by the desired-state marker, wire opportunistic healing into
    waiting agent runners, and extend doctor checks.'
- id: smoke
  title: End-to-end outage and recovery smoke exercises
  depends_on:
  - log-rotation
  - orchestrator
  - restart
  - watchdog
  description: '''End-to-end outage and recovery smoke exercises'' section: exercise
    the failure scenarios found in production (killed restart, crash loop, at-cap
    logging) and confirm recovery plus notifications.'
  model: haiku
create_time: 2026-07-19 17:23:09
status: wip
bead_id: sase-7p
---

# Plan: Axe restart reliability and lumberjack outage self-healing

## Problem

Bryan periodically stops receiving Telegram messages from sase (and sase stops reacting to his messages), and agents
that are waiting on dependencies do not get started even after those dependencies complete. Both symptoms share one
cause: the axe daemon (orchestrator + lumberjack processes) is down or degraded during those windows. Telegram delivery
is performed by the `tg_inbound`/`tg_outbound` chops of the `telegram` lumberjack, and waiting agent runners poll for a
`ready.json` marker that only the `wait_checks` chop (in the `waits` lumberjack) writes
(`src/sase/axe/run_agent_wait.py`, `src/sase/scripts/sase_chop_wait_checks.py`). When axe is down, both go dark at once.

## Root-cause diagnosis (evidence from 2026-07-19 on athena)

1. **`sase update` restarts axe frequently, and the restart's start half sometimes never happens — silently.** Every
   dev-update with `changed: true` calls `restart_after_update()` → `restart_axe_daemon_result()` which is a bare
   `stop_axe_daemon_result()` followed by `start_axe_daemon_result()` (`src/sase/axe/process.py`). On 2026-07-19 there
   were ~14 updates; every lumberjack stop matches an update timestamp exactly. Two of those restarts stopped axe but
   never brought it back:
   - update 11:28:49 → all lumberjacks SIGTERMed 11:28:54 → nothing ran until the _next_ update at 12:55:57 restarted it
     (87-minute total outage);
   - update 16:33:39 → stop 16:33:44 → nothing until a manual `sase axe start` at 17:03:45 (30-minute outage). Failure
     paths that produce exactly this: the `sase update` process dying between stop and start (Ctrl-C, closed terminal),
     or the start half failing (lock race with the dying orchestrator, executable being swapped mid-update).
     `restart_after_update` swallows all exceptions into a `RestartInfo`, `sase update` exits 0 regardless, and the only
     trace is one yellow terminal line. The dev-update journal (`~/.sase/logs/dev_update.jsonl`) is written _before_ the
     restart runs (`src/sase/main/update_handler.py`), so every journal record shows `restart: null` — there is no
     durable record of restart outcomes at all.

2. **Nothing self-heals and nothing can even report the outage.** No watchdog notices a downed orchestrator or stale
   lumberjack. Recovery waits for the next `sase update` or a manual start. Worse, the primary alerting channel
   (Telegram) is itself delivered by an axe chop, so "axe is down" can never reach Bryan through it — a structural blind
   spot.

3. **Bounded-log rotation has no hysteresis and causes an I/O storm.** `append_bounded_log()`
   (`src/sase/axe/_state_lumberjack.py`) keeps a capped file pinned at exactly `max_bytes` (50 MiB): once at the cap,
   _every_ append reads the ~50 MiB tail and rewrites the whole file through a temp file + rename. The `telegram` and
   `hooks` per-jack logs and two aggregate logs currently sit at exactly 52,428,800 bytes, so each lumberjack tick costs
   ~100 MB of disk I/O; the telegram lumberjack logged a tick overrun ("took 6.8s but interval is 5s") right before
   today's second outage. This degrades the whole host and widens every restart race window.

4. **Killed-mid-rotation temp files leak: 3.2 GB of orphaned `.lumberjack-*.log.*.tmp` files** in `~/.sase/axe/logs/`.
   Because every at-cap append is a rotation, any SIGKILL during logging leaves a ~50 MiB orphan. The disk actually
   filled up this morning: `tg_inbound`/`tg_outbound` failed with `[Errno 28] No space left on device` at 08:54 — a
   second, independent way Telegram goes silent while axe is nominally up.

5. **The orchestrator restarts crashing children in a tight loop with no backoff and no surfacing.**
   `~/.sase/axe/logs/axe.log` shows 400+ consecutive `Lumberjack 'waits'/'hooks' exited (code 1), restarting...` lines
   (version-skew crashes after updates, e.g. Jul 13 `ValueError: invalid project lifecycle state "enabled"`). Errors
   never reach the notification inbox; the loop also spams Prometheus port-conflict tracebacks.

6. **Aggregate logs lag by up to tens of minutes, hiding evidence.** The orchestrator's pump thread
   (`Orchestrator._stream_child_output`) uses `stream.read(64 * 1024)` on a buffered pipe, which blocks until a full 64
   KiB accumulates. Quiet lumberjacks emit a few hundred bytes per tick, so startup lines and crash tracebacks appear in
   `~/.sase/axe/logs/lumberjack-*.log` only much later (or never, before a kill) — which is why these outages were so
   hard to diagnose.

All of this is host-side daemon/process infrastructure, so it belongs in this repo (no `sase-core` boundary crossing).

## Bounded-log rotation hysteresis and temp-file cleanup

Fix `append_bounded_log()` so a capped log is cheap to append to:

- On a cap crossing, truncate the retained payload to roughly half the cap (truncation marker + tail + new data, capped
  at `max_bytes // 2`), so the file then grows with plain O(len(data)) appends until the next crossing. Keep the atomic
  temp-file + `os.replace` install for the truncation path only.
- Clean up orphaned rotation temp files: when `_atomic_replace_bytes` rotates `<dir>/<name>`, best-effort unlink stale
  `.<name>.*.tmp` siblings older than a small age threshold. Also sweep them once at orchestrator startup so the
  existing 3.2 GB of litter disappears without manual action.
- Preserve current semantics for readers (`read_tail_seek`, TUI log tails) and the truncation marker line.

Testing: unit tests that appends beyond the cap produce a file no larger than the cap, that the post-truncation size
leaves headroom (hysteresis), that appends below the cap never rewrite the file, and that orphan temps get reaped.

## Orchestrator output streaming and crash-loop backoff

Two independent fixes in `src/sase/axe/orchestrator.py`:

- **Timely streaming:** replace the blocking full-buffer `read(65536)` with `read1(65536)` (or reads on the raw stream)
  so each chunk of child output is appended to the aggregate `lumberjack-<name>.log` as soon as the child produces it.
  Startup banners and crash tracebacks then land in the log within milliseconds.
- **Crash-loop backoff and surfacing:** track recent restart times per lumberjack. When a child exits unexpectedly,
  restart it with exponential backoff (e.g. 1s doubling to a 60s ceiling, reset after a healthy run of a few minutes).
  After N rapid failures (e.g. 3 within 60s), write an error via `append_error` _and_ create a notification-inbox entry
  (reusing the existing notification senders used by chops) that names the lumberjack, the exit code, and a short tail
  of its recent output. Keep retrying at the ceiling — never give up permanently — but stop the tight loop and make the
  condition visible in the ace TUI axe tab and, once Telegram is back, on the phone.

Testing: unit tests with fake `Popen` objects covering backoff schedule, reset-on-healthy-run, and the notification
threshold; a streaming test asserting partial chunks flush promptly.

## Verified, journaled axe restart with desired-state marker

Make the restart pipeline honest about outcomes:

- **Desired-state marker:** record axe's intended state in `~/.sase/axe/desired_state.json`. `sase axe start` and the
  restart flow record `running` (with source and timestamp); an explicit `sase axe stop` records `stopped`.
  `restart_axe_daemon_result()` writes `running` _before_ its stop phase, so if the restarting process dies mid-flight
  the intent survives and the watchdog (next phase) can heal.
- **Retry + verify:** in `restart_axe_daemon_result()` retry the start phase (e.g. 3 attempts with short backoff,
  tolerant of the old orchestrator's lock lingering briefly), then verify success: orchestrator PID alive _and_
  lumberjack heartbeats fresh (per-jack `status.json` `last_cycle` advancing within a bounded wait). Return a structured
  result including per-attempt failures.
- **Fail loudly:** when the restart ultimately fails, write a notification-inbox entry and an `append_error` record in
  addition to the console rendering, so the failure shows up in the ace TUI immediately and in Telegram after recovery.
- **Journal the outcome:** in `sase update` (`src/sase/main/update_handler.py`), include the restart result in the
  dev-update journal record (append after the restart completes instead of before, or amend with a follow-up record) so
  `dev_update.jsonl` stops reporting `restart: null` and outages can be correlated after the fact.

Testing: unit tests through the already-injectable seams (`RestartAxeFn`, `AxeRunningFn`, fake clocks) for
retry/verify/notify behavior and for journal records containing restart outcomes; marker-file round-trip tests covering
start, stop, restart, and crash-mid-restart sequences.

## Axe self-healing via sase axe ensure

Close the loop so a downed axe cannot stay down silently:

- **New CLI:** `sase axe ensure` — idempotent heal. If the desired state is `running` (or the marker is absent) and no
  live orchestrator exists, start axe and write a notification-inbox entry noting the heal and how long axe appears to
  have been down; if axe is healthy or was explicitly stopped, do nothing. Follow the CLI rules: alphabetically sorted
  listings, a short alias for every public long option, colored, scannable `-h` output.
- **Opportunistic healing from waiting runners:** the `run_agent_wait` poll loop is exactly the population harmed by a
  dead `waits` lumberjack, and those runner processes are already alive during outages (20+ were waiting during today's
  outage). While polling for `ready.json`, have runners invoke the ensure path at a rate-limited cadence (e.g. at most
  once per 5 minutes host-wide, guarded by a marker/lock under `~/.sase/axe/`) so a downed axe heals even with no TUI
  open. Respect the explicit-`stopped` desired state.
- **Scheduled belt-and-braces:** ship a user-level systemd service+timer template that runs `sase axe ensure` every few
  minutes, installable via an ensure subcommand (e.g. `sase axe ensure install` / `uninstall`, defaulting bare `ensure`
  to the heal action). This covers fully idle hosts where neither TUI nor waiting runners exist. Keep it opt-in and
  document it in the command help.
- **Doctor visibility:** extend the axe doctor checks (`src/sase/doctor/checks_axe.py` / `checks_deep_axe.py`) to flag:
  desired state `running` but orchestrator down; stale lumberjack heartbeats; aggregate/per-jack logs pinned at the cap;
  and orphan rotation temp litter above a threshold.

Testing: unit tests for ensure's decision table (desired state × orchestrator liveness × rate-limit marker) and for the
doctor checks; an integration-style test that a fake downed state plus `ensure` produces a start attempt and an inbox
notification.

## End-to-end outage and recovery smoke exercises

Reproduce today's production failures against the fixed code, in a sandboxed `SASE_HOME`/state dir:

- Simulated update-restart interruption: stop axe, skip the start (as if the updater died), run `sase axe ensure`, and
  assert axe comes back, the heal notification lands in the inbox, and the desired-state marker reads `running`
  throughout.
- Crash-loop: configure a lumberjack whose child exits nonzero immediately; assert restarts back off, the inbox entry
  appears after the threshold, and recovery resumes promptly once the child starts succeeding.
- At-cap logging: drive a capped log past `max_bytes` repeatedly; assert appends stay cheap (no full-file rewrite
  between truncations), the file never exceeds the cap, and no `.tmp` orphans accumulate after simulated kills.
- Restart verification: exercise `restart_axe_daemon_result()` end-to-end against a real orchestrator process in the
  sandbox; assert the structured result reports verified startup and that `sase update`'s journal record contains the
  restart outcome.

## Risks and notes

- The orchestrator/lumberjack lifecycle is production-critical on athena; every phase must keep `just check` green and
  avoid changing the default lifecycle semantics (an explicit `sase axe stop` must stay stopped — only the marker
  distinguishes intent).
- Backoff must never strand a lumberjack permanently; ceilings, not give-ups.
- The systemd timer is optional/opt-in; the ensure command and runner-side healing must not assume systemd exists.
- New tunables (backoff ceiling, ensure cadence, orphan-age threshold) should ride the existing axe config layering with
  sensible defaults rather than new required config.
- Concurrent appends to one bounded log can still race across processes; hysteresis makes the rewrite path rare, which
  is sufficient here — full cross-process log locking is out of scope.
