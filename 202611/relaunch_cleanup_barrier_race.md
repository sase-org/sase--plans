---
tier: tale
title: Hold ,x relaunches until kill/dismiss cleanup settles
goal:
  A ,x kill-and-edit relaunch no longer fails with "left agent name reserved after
  rebuild" when its kill/dismiss cleanup proc is slow.
size: medium
proposed_by: bbugyi200.athena.1hf
proposed_with: claude/opus
create_time: 2026-11-27 15:00:11
status: wip
---

# Hold `,x` relaunches until their kill/dismiss cleanup actually settles

## Problem

`,x` (kill-and-edit) on `bob-cli-q5.8.1.6.5.1.3` relaunched with a forced name reuse
(`%id(!3, ...)`), and the durable `sase run` failed with:

```
Agent name reuse failed: forced reuse cleanup batch failed: forced reuse cleanup left
agent name 'bob-cli-q5.8.1.6.5.1.3' reserved after rebuild; resolve the conflicting
owner and retry; remaining cleanup targets: bob-cli-q5.8.1.6.5.1.3
```

## Root cause (verified from proc records, toasts, `tui.log`, and file timestamps)

Times are UTC on athena, 2026-11-27.

| Time         | Event                                                                                                                                                                                                                                                                               |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 19:32:41.7   | `,x` kills `.3` (PID 3804192). It submits the `sase agent persist-cleanup` proc `e2ajaa1p8pn0` and opens a relaunch cleanup barrier with a fixed 30s timeout (`RELAUNCH_CLEANUP_BARRIER_TIMEOUT_SECONDS`).                                                                          |
| 19:32:45     | The user submits the edited prompt. The launch is parked ("Waiting for kill/dismiss cleanup…").                                                                                                                                                                                     |
| 19:33:11     | The barrier times out ("Kill/dismiss cleanup did not settle in time; launching without waiting for it."), and launch proc `1anhvgww6h5q` starts while the cleanup proc is still running.                                                                                            |
| 19:33:12–~30 | Inside `sase run`, `wipe_force_reuse_owners` snapshots the registry and scans the wipe catalog (~25k artifact dirs and ~70k dismissed bundles). At ≥14s even on an idle machine, the scan finishes before any bundle exists, so the plan covers only artifact dir `20261127070031`. |
| 19:33:29.79  | The still-running cleanup proc writes dismissed bundle `dismissed_bundles/202611/20261127070031.json`, which names `.3`. The file's birth, mtime, and ctime are all 19:33:29, so nothing touched it later.                                                                          |
| 19:33:30.9   | The cleanup proc finishes, 48s after it started.                                                                                                                                                                                                                                    |
| 19:35:30     | The wipe's final `rebuild_name_registry()` re-derives `.3` from that late bundle (`source=dismissed_bundle`). `_forced_reuse.py` then raises "left … reserved after rebuild".                                                                                                       |

The 30s timeout came from `plan:202608/kill_and_edit_prompt_latency.md`, which assumed
that "30s is far beyond the ~1.5–3s a healthy cleanup proc takes, so a real cleanup
never trips it." That assumption no longer holds:

- On athena, the 13 most recent `persist-cleanup` procs ran 14–55s each (median ~31s).
  Nine of them ran 29s or longer.
- All four `,x` cleanups in this incident tripped the barrier.
- `tui.log` shows the barrier timing out several times a day since at least 2026-11-15.
- Earlier "left … reserved after rebuild" launch failures line up with barrier timeouts
  a few minutes before them, often with `source=dismissed_bundle` residuals. Examples:
  `sase-1t3.5` and `sase-1t3.1` (11-17), `sase-1tc.4` (11-19), and `tailnet-1e.8.1.land`
  (11-22).

So this is a recurring race, not a one-off. There are two defects:

1. **The TUI barrier gives up while the cleanup is still running.** Its fixed 30s
   deadline treats a slow cleanup as a hung one. The barrier exists so that a late
   bundle write cannot resurrect a reused name, and the timeout defeats exactly that.
2. **The forced-reuse wipe cannot recover from a late writer.** It plans from one
   snapshot. When an owner appears between its catalog scan and its registry rebuild, it
   fails instead of re-planning. Its batch-path error also omits the residual owner's
   source and paths. The session path already includes them, and their absence is what
   made this incident hard to read.

Immediate workaround: nothing needs repair. The stray bundle is the killed `.3` agent's
legitimate dismissal record. A dry-run `build_wipe_plan` for `.3` now includes it, so
re-submitting the stashed `.3` relaunch prompt should succeed.

## Changes

### 1. Barrier waits for a slow cleanup instead of abandoning it

File: `src/sase/ace/tui/actions/agent_workflow/_relaunch_barrier.py`.

- Replace `RELAUNCH_CLEANUP_BARRIER_TIMEOUT_SECONDS = 30.0` with two constants:
  - `RELAUNCH_CLEANUP_BARRIER_RECHECK_SECONDS = 30.0`
  - `RELAUNCH_CLEANUP_BARRIER_MAX_WAIT_SECONDS = 300.0`

  The ceiling is about 5x the worst observed cleanup (~55s). It exists only for a truly
  wedged supervisor or proc observer, which is the degrade-not-wedge concern from the
  original plan.

- Give `_RelaunchCleanupBarrier` an `opened_at: float` field (`time.monotonic()` at
  open) and a `still_running_notified: bool = False` field.
- `open_relaunch_cleanup_barrier` arms a recheck timer instead of a deadline timer. Keep
  storing the handle in `barrier._timer`, so `settle_relaunch_cleanup_barrier` still
  stops whichever timer is current.
- Add `_recheck_barrier(app, barrier)`. It returns if the barrier is already settled.
  Otherwise:
  - If `elapsed >= MAX_WAIT`, use the existing deadline path (`_settle_on_timeout`):
    warning log, settle, warning toast. Reword the toast to say the cleanup is still
    unconfirmed after the ceiling and that the relaunch may fail with an agent-name
    reuse error that is safe to retry.
  - Otherwise, keep the launch parked:
    - Do not settle and do not drain waiters.
    - When `still_running_notified` is unset and a launch is actually parked for this
      barrier's operation, emit one informational toast, then set the flag. Suggested
      text: "Kill/dismiss cleanup is still running; the relaunch stays held until it
      finishes (,X cancels it)". The pending-launch row already shows
      `PendingLaunchStage.WAITING_CLEANUP`.
    - Re-arm the timer for `min(RECHECK, MAX_WAIT - elapsed)`.
- Settlement semantics stay as they are: `on_settled`, idempotent settle, per-operation
  scoping, parked-launch replay exactly once, and `,X` cancellation dropping waiters.
- Update the module docstring to describe the recheck and ceiling model.

### 2. Forced-reuse wipe re-plans once when a late owner appears

The wipe is Python filesystem and process side-effect orchestration in
`sase.agent.names`, so it stays in Python (`decisions:rust-core-required` keeps side
effects out of the deterministic core).

- `src/sase/agent/names/_forced_reuse.py`, in `wipe_force_reuse_owners`:
  - After the first `_wipe_names_for_reuse_batch(materialized)`, find the residual
    targets: results with `found`, no `errors`, no `skipped_container_kind`, and
    `target_name not in registry_names_removed`.
  - If any exist, call `_wipe_names_for_reuse_batch(residual_names)` exactly once more.
    Pass no `catalog`, so the retry takes a fresh registry snapshot and a fresh catalog
    scan that now sees the late bundle or artifact. Wrap exceptions the same way as the
    first pass.
  - Merge the retry results by `target_name` over the first results. Run the existing
    classification loop (errors, agent-session names, stale clans) over the merged
    results so a retry outcome of any kind is handled uniformly.
  - For a name that is still residual after the retry, enrich the error with the rebuilt
    registry owner's details. Read the registry best-effort through
    `load_name_registry()` and reuse `_describe_residual_owner` from
    `_session_owners.py` (source, artifacts_dir, bundle_path, collision_owners). Keep
    the existing phrase "forced reuse cleanup left agent name '<name>' reserved after
    rebuild" and the "resolve the conflicting owner and retry" guidance, so log searches
    and tests keep matching.
- `src/sase/agent/names/_session_owners.py`, in `wipe_agent_sessions_for_forced_reuse`:
  give residual sessions the same one-shot retry.
  - When `residual_errors` is non-empty, re-run the session wipe once for just the
    residual session names, for example through a keyword-only
    `retry_residual: bool = True` parameter that the recursive call sets to `False`.
  - Raise the existing residual error only if the retry still leaves reservations.
  - Leave `release_stale_containers` unchanged.
- The retry runs only on the failure path. The common case pays nothing extra.

Note: the retry closes the scan-to-rebuild window seen here. A cleanup that writes after
the whole wipe has finished can only be prevented by the barrier fix in change 1. Keep
both; neither one alone covers both windows.

### 3. Tests

- `tests/ace/tui/test_kill_and_edit_launch_barrier_hold.py`:
  - Replace `test_barrier_timeout_releases_held_launch_with_warning` with recheck
    coverage. A recheck while unsettled and under the ceiling keeps the launch parked:
    no launch proc, the barrier is still pending, one info toast, and a second recheck
    adds no duplicate toast.
  - Add a test that a recheck past the ceiling releases the launch with the warning
    toast. Drive time by back-dating `barrier.opened_at` or monkeypatching
    `time.monotonic`; never sleep.
  - Add a test that a real settle after one or more rechecks replays the parked launch
    exactly once and stops the timer.
  - Update the comment that references `RELAUNCH_CLEANUP_BARRIER_TIMEOUT_SECONDS`.
- `tests/test_agent_names_forced_reuse.py`, using the helpers in
  `tests/_agent_name_wipe_helpers.py` (`make_wipe_artifact`, `make_wipe_bundle`):
  - **Late-bundle regression.** Seed an artifact-backed concrete owner and rebuild the
    registry. Patch `_wipe_execute.execute_wipe_plan` (or the bundle-removal step) so
    the first execution writes a dismissed bundle for the same name and suffix, after
    the catalog scan and before the rebuild. Assert that
    `wipe_force_reuse_owners([name], allow_container_skip=False)` succeeds, the late
    bundle is gone, and the name is no longer reserved.
  - **Persistent residual.** The same shape, where the residual cannot be removed
    (pattern: `test_wipe_force_reuse_residual_error_names_blocking_file`), still raises
    after exactly one retry. Count the batch calls. The message includes `source=` and
    `bundle_path=` or `artifacts_dir=`.
  - **Session-path counterpart.** A late `--plan` bundle for an agent-session root is
    absorbed by the one retry. Confirm the existing
    `test_wipe_force_reuse_residual_error_names_blocking_file` still passes.

### 4. Verification

- Read the `lint_and_test` reference memory and run what it prescribes (`just check`
  through `sase tool run`). Run the targeted tests first:
  `tests/ace/tui/test_kill_and_edit_launch_barrier*.py`,
  `tests/test_agent_names_forced_reuse.py`, `tests/test_agent_name_wipe_sessions.py`,
  and `tests/test_force_reuse_launch_seam*.py`.
- Keep every touched module under the repo's line cap. If `_forced_reuse.py` grows past
  it, move the retry and merge helper into a small private helper module.

### 5. Follow-up bead (do not fix here)

Use `/sase_new_task` to file one task bead for the `sase agent persist-cleanup` latency
regression. Recent procs ran 14–55s each on athena (median ~31s), against the ~1.5–3s
the barrier design assumed. Include the proc IDs `kwycaj6tx9da`, `e2ajaa1p8pn0`,
`4tpkamx0c58d`, and `bphq3bybz5cf` as evidence, and leave the root cause open (startup,
index sync, termination grace, or load) for that task to profile.
