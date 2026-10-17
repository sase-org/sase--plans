---
tier: tale
title: Make a deliberate agent kill never surface as a runner crash
goal:
  A user or agent kill of a SASE agent runner always settles as a clean kill, never as a
  FAILED run with an error report, even when several SIGTERMs arrive at once.
size: medium
decisions:
  dedupe_sigterm:
    ask:
      Should the verified terminator skip its direct SIGTERM to processes its group
      SIGTERM already reached?
    choices:
      dedupe:
        In-group processes get one graceful SIGTERM per pass; out-of-group ones are
        still signalled
      keep: Every pinned pid still gets a direct SIGTERM on top of the group signal
    default: dedupe
    why: Duplicate SIGTERMs can cut short a provider CLI's graceful shutdown.
    answer: dedupe
proposed_by: bbugyi200.athena.114
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-17 15:16:21
status: wip
---

# Plan: Make a deliberate agent kill never surface as a runner crash

## Incident

`sase-1lh.7.5.4.3.land` (agent artifacts:
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202610/17/20261017002443/`) was
parked in `wait_for_dependencies` for 14h, waiting on `sase-1lh.7.5.4.3.2`, whose
release watch had failed. Phase worker `sase-1lh.7.5.4.3.3.1` killed it on purpose,
following epic plan `green_gates_ship_0_18_0.md` ("stop the hung previous watch turn and
its parked lander"). It ran `sase agent kill -n sase-1lh.7.5.4.3.land`, which wrote the
`.sase_user_kill_pending` intent (`source: agents_kill`, `status: killed`).

The kill itself was correct. The bug is in how the runner reported it: instead of the
clean `Agent killed while waiting` → `SystemExit(143)` → `exec_outcome="killed"` path,
the runner wrote `error_report.md`, a failed done marker, counted
`AGENT_KILLS{reason="error"}`, and logged `Agent completed with status: FAILED` with:

```
File ".../run_agent_wait.py", line 288, in wait_for_dependencies
    time.sleep(_WAIT_READY_POLL_INTERVAL)
File ".../runner_signals.py", line 48, in _handler
    print(f"\nReceived SIGTERM - {description} was killed", file=sys.stderr)
File ".../runner_signals.py", line 48, in _handler
    print(f"\nReceived SIGTERM - {description} was killed", file=sys.stderr)
RuntimeError: reentrant call inside <_io.BufferedWriter name='<stderr>'>
```

No operational recovery is needed. The nested epic `sase-1lh.7.5.4.3.3`'s lander is
designed to resume the landing chain.

## Root cause

1. **Duplicate SIGTERMs are routine.** `_run_termination` in
   `src/sase/agent/_user_kill_tree.py` first sends `killpg(runner_pgid, SIGTERM)` and
   then a direct `os.kill(pid, SIGTERM)` to every pinned plain target, including the
   runner itself (`REASON_RUNNER`). So `sase agent kill` delivers two SIGTERMs to the
   runner a few milliseconds apart. A TUI kill can deliver three: the immediate `killpg`
   from `request_user_kill(wait=False)`, then the durable terminator's two.
2. **The SIGTERM handler is not reentrancy-safe.** `install_sigterm_handler._handler` in
   `src/sase/axe/runner_signals.py` calls `print(..., file=sys.stderr)` from signal
   context. CPython's `BufferedWriter` flush runs pending signal handlers
   (`PyErr_CheckSignals`) while it holds its internal lock. If a second SIGTERM arrives
   during the first handler's write, the handler runs again inside that write, and its
   `print` raises `RuntimeError: reentrant call`. A single SIGTERM that lands while the
   main thread is mid-write to stderr has the same hazard.
3. **The exception escapes into the main thread.** The `RuntimeError` propagates out of
   the handler into whatever the main thread was doing (`time.sleep` in the wait loop).
   That skips the clean kill exit, and `main()`'s `except Exception` branch in
   `src/sase/axe/run_agent_runner.py` records it as a crash.
   `record_runner_error(..., killed=True)` only suppresses the auto-restart doorbell; it
   still writes the failed marker and the error report.
4. **Sibling bug in hard (non-soft) handlers.** The mentor runner uses
   `install_sigterm_handler("mentor")`. There, a repeated SIGTERM raises a second
   `SystemExit` while the first one's `finally` cleanup is still running, which aborts
   that cleanup.

The runner's signal handling is runner-process plumbing, not shared domain behavior, so
it stays in this repo (no `sase-core` change).

## Changes

### 1. Idempotent, reentrancy-safe SIGTERM handler — `src/sase/axe/runner_signals.py`

Rewrite `_handler` inside `install_sigterm_handler`:

- **Idempotent.** If `_killed_state["killed"]` is already true, return immediately: no
  message, no `on_signal`, no second `sys.exit`. A nested or repeated delivery of the
  same kill becomes a no-op. The guard keys off the existing flag, so `reset_killed()`
  (used between gate and monitor follow-up iterations) still re-arms the handler for the
  next real kill.
- **Flag first.** Set `killed` and `killed_at` before any I/O, so the main loop sees the
  kill even if everything after it fails.
- **No buffered stderr in signal context.** Replace the `print` with a best-effort
  unbuffered write of the encoded message, ending in a newline, straight to the stderr
  file descriptor via `os.write`. Get the fd from `sys.stderr.fileno()` and fall back to
  `2`. Swallow `OSError`, `ValueError` and `AttributeError`. Never touch the buffered
  `sys.stderr` object from the handler. The trailing newline also fixes the log
  run-together seen in the incident (`...was killedError running agent: ...`).
- **`on_signal` failures stay contained.** Keep the `try/except Exception` around
  `on_signal()`. Also guard the `logger.exception(...)` call in its own
  `try/except Exception: pass`, because a logging `StreamHandler` writing to stderr from
  signal context can raise the same reentrancy `RuntimeError` through `handleError`. Add
  a comment: the signal handler must never raise anything except the deliberate
  `SystemExit`.
- Update the docstring to cover the idempotency contract and the unbuffered-write
  rationale.

Tests, next to the existing handler tests in `tests/test_axe_runner_utils.py`:

- **Nested delivery.** An `on_signal` callback that calls the captured handler again:
  `on_signal` runs once, nothing raises, and `killed_at()` keeps the first timestamp.
- **Repeated delivery.** A second call after the first completes is a no-op. Patch
  `os.write` and count calls: one message, one `on_signal`.
- **Stderr independence.** Replace `sys.stderr` with a mock whose `write` raises
  `RuntimeError("reentrant call ...")`. The handler still succeeds, writes through
  `os.write`, and never calls `sys.stderr.write`.
- **Hard mode.** The first call raises `SystemExit(143)` and the second call returns
  without raising.
- **Re-arm.** After `reset_killed()`, the handler runs `on_signal` again.
- **Logging failure.** `on_signal` raises and `logger.exception` is patched to raise
  `RuntimeError`. The handler still returns normally in soft mode.
- **Subprocess regression.** Spawn a child with `sys.executable` and
  `start_new_session=True`, stderr redirected to a temp file. The child installs
  `install_sigterm_handler("agent", soft=True)`, prints a ready line on stdout, then
  loops writing sizeable chunks to `sys.stderr` with short sleeps until `was_killed()`.
  It then exits 0 after printing `clean-exit`. The parent waits for the ready line,
  sends a burst of SIGTERMs (for example `killpg` plus about 20 direct `os.kill` calls),
  and waits with a timeout. Assert exit code 0, `clean-exit` on stdout, and no
  `reentrant` or `Traceback` in the stderr file. Before the fix this reproduces the
  incident often; after the fix it passes deterministically. Clean up the child on
  timeout.

### 2. Exceptions that coincide with a user kill count as kills — `src/sase/axe/run_agent_runner.py`

- Extract the user-kill bookkeeping from `main()`'s `except SystemExit` branch into a
  small helper, for example `_record_user_kill(state)`. It sets
  `exec_outcome = "killed"` and `suppress_completion_notification = True`, and calls
  `record_kill_provenance(state.current_artifacts_dir or state.artifacts_dir, None, kill_time=killed_at())`.
  Use the helper there unchanged.
- In the `except Exception as e` branch, before `record_runner_error`: if `was_killed()`
  is true and `has_user_kill_intent(state.current_artifacts_dir or state.artifacts_dir)`
  is true (from `sase.agent._user_kill_intent`, matching
  `run_agent_exec._handle_killed_iteration`):
  - print a one-line note plus the traceback to stderr, so the output log keeps the
    evidence;
  - call the helper;
  - `raise SystemExit(128 + signal.SIGTERM) from e`.

  This takes the same route as a clean kill: no failed done marker, no
  `error_report.md`, no completion notification, and no `AGENT_KILLS{reason="error"}`.
  The outer `finally` still sweeps the agent scope, finalizes shutdown and cleans up
  scratch.

- Requiring the intent marker keeps handoff SIGTERMs out of this path, since plan,
  questions, monitor, gate and pipe handoffs write no user-kill intent. External and OOM
  kills are also excluded. Both keep today's failure recording.

Tests, in `tests/test_run_agent_runner_bootstrap_errors.py`, alongside
`test_user_kill_during_bootstrap_preserves_exit_and_skips_error_recording`:

- (a) `_run_agent` raises `RuntimeError`, `was_killed()` is true and an intent marker is
  present: exit code 143, `exec_outcome == "killed"`, and the error recorder and failed
  done marker are not called.
- (b) Same exception and `was_killed()` true, but no intent marker: still recorded as a
  failure, with `killed=True` passed through.
- (c) `was_killed()` false: unchanged failure recording.

### 3. One graceful SIGTERM per terminate pass — `src/sase/agent/_user_kill_tree.py`

> [!decision] dedupe_sigterm = dedupe In `_run_termination`, capture
> `group_signalled = termination.signal_group(signal.SIGTERM)`. When it is true, skip
> the direct SIGTERM for each plain target whose current `read_process_row(target.pid)`
> still reports `pgid == termination.pid`; the group signal already reached it. Re-read
> the row at signal time instead of trusting discovery-time data. If the row is
> unreadable, fall back to signalling directly. Targets outside the group still get
> their direct SIGTERM: setsid orphans, scratch-key members, and processes in other
> sessions or below the runner in the ppid tree. The supervisor canonical stop and the
> SIGKILL escalation are unchanged. Update the `terminate_agent_processes` docstring's
> "Sequence:" sentence to match. In `tests/test_agent_terminate_processes.py`, use the
> injectable `kill`/`killpg` recorders: a group-leader runner and an in-group child get
> the group SIGTERM and no direct SIGTERM; a setsid child outside the group still gets a
> direct SIGTERM; the non-leader runner path (`_signal_non_leader`, and `signal_group`
> returning false) still signals the runner directly. Existing real-process tests must
> keep passing.

> [!decision] dedupe_sigterm = keep Skip this section. Step 1 alone makes duplicate
> SIGTERMs harmless to the runner.

## Out of scope

- The kill decision and its source agent: the plan told the phase worker to stop the
  lander. Do not relaunch `sase-1lh.7.5.4.3.land`.
- Moving `on_signal`'s workspace-claim release out of signal context. It takes file
  locks from a signal handler, which is a separate latent hazard. Mention it as a
  `PROPOSED FOLLOW-UP` in the final summary if the implementer agrees it is real.
- The TUI's immediate plus durable double kill. Step 1 makes it harmless.

## Verification

- Run the focused modules: `tests/test_axe_runner_utils.py`,
  `tests/test_run_agent_runner_bootstrap_errors.py`,
  `tests/test_agent_terminate_processes.py`, `tests/test_agent_user_kill.py`,
  `tests/test_kill_durable_termination.py`, and the `tests/test_run_agent_runner_*`
  family.
- Then run the repo's standard lint and test gate, per the `lint_and_test` reference
  memory (`just check` through `sase tool run`).
