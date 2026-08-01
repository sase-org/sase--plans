---
tier: tale
title: Stop `sase ace` from hanging after quit
goal: Pressing `q` in `sase ace` returns the shell prompt within about two seconds,
  even while blocking git work is in flight.
create_time: 2026-07-25 11:17:20
status: done
---

- **PROMPT:** [prompts/202607/ace_quit_hang.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/ace_quit_hang.md)
- **AGENTS:**
  - [bbugyi200.athena.kt](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.kt/README.md)
  - [bbugyi200.athena.kt--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.kt.md#member-code)
- **COMMITS:**
  - [c0f1c6e](https://github.com/sase-org/sase/commit/c0f1c6e5a3c775ee314a6ca14c16ca5913b83d05) — fix(ace): prevent quit hangs on in-flight workers

# Fix `sase ace` hanging after quit

## Problem

Pressing `q` in `sase ace` dismisses the TUI (terminal is restored, alternate screen exited) but the process never
exits, so the shell prompt never comes back. Users see a dead terminal for a minute or more, and sometimes forever.

## Root cause (verified with live stacks)

Textual runs every `run_worker(..., thread=True)` worker through `loop.run_in_executor(None, ...)` — i.e. the **asyncio
default executor**. Textual's `App._shutdown()` closes the driver and restores the terminal, but it never waits for or
cancels in-flight thread workers; cancelling the asyncio task that awaits `run_in_executor` does not stop the
already-running thread function.

`App.run()` then calls `asyncio.run(...)`, whose teardown (`asyncio.runners.Runner.close()`) calls
`loop.run_until_complete(loop.shutdown_default_executor(constants.THREAD_JOIN_TIMEOUT))`. That blocks the main thread
joining every default-executor worker thread. Because the terminal was already restored, the user sees exactly the
reported symptom: TUI gone, shell not back.

A live `py-spy dump` of a hung process captured the three-way stack:

```
Thread MainThread (idle):
    run_until_complete (asyncio/base_events.py)
    close (asyncio/runners.py:74)
    run (textual/app.py)
    handle_ace_command (sase/main/ace_handler.py:167)
Thread "Thread-1 (_do_shutdown)" (idle):
    join (threading.py)
    shutdown (concurrent/futures/thread.py:273)   # ThreadPoolExecutor.shutdown(wait=True)
    _do_shutdown (asyncio/base_events.py:621)
Thread "asyncio_3" (idle):
    communicate (subprocess.py)                   # blocking git subprocess
    run_sdd_git (sase/sdd/_git.py:43)
    run_git (sase/agents_sync/git.py:50)
    read_bytes (sase/agents_sync/git_objects.py:105)
    capture_fetched_agent_updates (sase/agents_sync/incoming_detection.py:199)
    _refresh_project_status (sase/agents_sync/status.py:160)
    get_agents_sync_status (sase/agents_sync/status.py:97)
    _run_agents_sync_status_check (sase/ace/tui/actions/agents_sync.py:129)
    run_callable (textual/worker.py:309)
```

The dominant offender is the periodic agents-sync status worker (`src/sase/ace/tui/actions/agents_sync.py`,
`run_worker(..., thread=True)`), whose `_refresh_project_status` runs, per project target, a network
`git fetch --prune origin` (120s timeout) followed by an unbounded number of `git show` object reads (30s timeout each),
with no cancellation check between calls. It is not the only candidate: ACE has ~65 `thread=True` worker sites.

Two bounds make this "sometimes minutes, sometimes forever":

- `THREAD_JOIN_TIMEOUT` is 300s, so `Runner.close()` gives up after five minutes, and
- interpreter shutdown then runs `concurrent.futures.thread._python_exit`, which joins the same non-daemon worker
  threads **with no timeout at all**.

### Reproduction

```bash
tmux new-session -d -s acehang -x 200 -y 50 "cd <checkout> && exec python -m sase ace --no-axe"
sleep 8
PID=$(tmux list-panes -t acehang -F '#{pane_pid}')
tmux send-keys -t acehang q
# pane goes blank immediately (TUI dismissed); process stays alive
py-spy dump --pid "$PID"
```

Measured hang in one run: **65 seconds** (single project target, one network fetch in flight). Scales with the number of
enabled projects and the latency of each git call, capped at 300s by asyncio, then unbounded at interpreter exit.

### Validated fix mechanism

A standalone probe confirms the mechanism and the fix:

- `asyncio.run(body())` where `body` leaves one stuck `run_in_executor` thread: still hung at 20s (timeout killed it).
- Owning the loop and hard-exiting after the app coroutine completes: returns in **0.35s**.

Note that installing a custom default executor with daemon threads is _not_ a viable alternative:
`loop.set_default_executor` rejects non-`ThreadPoolExecutor` instances, and `ThreadPoolExecutor` creates its worker
threads non-daemon and registers them with `concurrent.futures.thread._threads_queues` before they can be flipped
(`RuntimeError: cannot set daemon status of active thread`). This was tried and rejected during diagnosis.

## Approach

Three layers, in dependency order. Layer 1 alone fixes the user-visible bug; layers 2 and 3 stop ACE from routinely
orphaning work and make future regressions diagnosable.

1. **Guarantee the exit.** `sase ace` owns its event-loop teardown instead of delegating to `asyncio.run`, and hard
   exits once the TUI is down. This bounds quit latency deterministically no matter what any thread worker is doing.
2. **Cooperative shutdown.** A process-wide shutdown signal that long blocking sweeps check between subprocess calls, so
   the common case drains cleanly instead of being abandoned.
3. **Diagnostics.** Log any still-running thread workers at exit so an overrunning worker is visible in `tui.log` rather
   than invisible.

### Rust core backend boundary

This work stays on the Python side and does not cross into `../sase-core`. Process and event-loop teardown is
Python/Textual process lifecycle, and `src/sase/agents_sync/` is pure Python today (no `sase_core_rs` calls). The
cooperative-shutdown signal is a Python-process concern — a web app or CLI frontend would not share it — so it belongs
here.

## Implementation

### 1. Own the event-loop teardown and hard exit (`src/sase/main/ace_handler.py`)

`handle_ace_command` currently calls `app.run()` (lines 163 and 167) and then `_exec_ace_restart_if_requested(...)` +
`sys.exit(0)`.

Replace `app.run()` with a helper that drives `app.run_async()` on an explicitly-owned loop:

- Create a new event loop, `run_until_complete(app.run_async())`, and do **not** call
  `loop.shutdown_default_executor()`. Textual's `run_async` already restores the terminal in its `finally` via
  `App._shutdown()`, so the terminal is clean when this returns.
- Keep the `--profile` behavior: `profiler.stop()` and `_write_profile_output(...)` must still run after the app
  coroutine completes and before process exit.
- Keep `_exec_ace_restart_if_requested(...)` where it is. It calls `os.execv`, which replaces the process image and
  therefore also escapes the blocked join — no extra handling needed for the restart path.
- Replace the final `sys.exit(0)` with an explicit hard exit:
  - flush telemetry that would otherwise be lost, since `os._exit` skips `atexit` handlers (see `register_flush_on_exit`
    in `src/sase/telemetry/_registry.py`);
  - `sys.stdout.flush()` and `sys.stderr.flush()`;
  - `os._exit(0)`.

Put the loop-driving helper and the hard-exit helper in `ace_handler.py` (or a small sibling module if `ace_handler.py`
grows past what the file conventions allow) with docstrings that state _why_ the normal teardown is bypassed, so nobody
"fixes" it back to `app.run()`.

Do not hard-exit from inside `_do_quit`/`action_quit`: at that point Textual has not yet restored the terminal, and
exiting there leaves the shell in raw/alt-screen mode.

### 2. Cooperative shutdown signal

Add a small module (suggested: `src/sase/ace/tui/util/shutdown.py`) exposing a process-wide `threading.Event` wrapper —
`request_shutdown()`, `is_shutdown_requested()`, and a test-only reset.

- Set it at the very top of `LifecycleMixin._do_quit` in `src/sase/ace/tui/actions/lifecycle.py` (before the existing
  `cleanup(...)` sequence), and in `_request_controlled_exit` / `_begin_controlled_exit` so both the confirmed-quit and
  direct-quit paths signal it.
- Thread it into the agents-sync sweep so a shutdown aborts between blocking calls rather than after them:
  - `_refresh_project_status` in `src/sase/agents_sync/status.py` — check before the `git fetch --prune origin` and
    before/inside the `capture_fetched_agent_updates` object-read loop.
  - Prefer plumbing this as an injectable predicate (default: a no-op that always returns `False`) rather than importing
    ACE TUI state into `src/sase/agents_sync/`. `status.py` already takes an injectable `git_runner`; follow that
    existing pattern. `sase.agents_sync` must not import from `sase.ace`.
  - On abort, return the reconciled/previous status rather than raising, so the snapshot write stays consistent.
- Leave the residual bound honest: one already-started git subprocess can still run to its 30s/120s timeout. That is
  fine — layer 1 guarantees the process exits regardless.

### 3. Exit diagnostics

Before the hard exit, enumerate live non-daemon threads whose names indicate default-executor workers (`asyncio_*`) or
the loader pool (`sase-loader*`) and, if any exist, log one WARNING to `tui.log` naming them. This turns a future
regression into a grep instead of another `py-spy` session.

## Testing

- `tests/main/` — a test for the new teardown helper proving that a coroutine which leaves a stuck `run_in_executor`
  thread behind still lets the helper return promptly (assert on a short wall-clock bound, not on process exit). Do not
  call `os._exit` in-process from a test; test the helper that decides to exit, and assert the hard-exit call via a
  patched hook.
- `tests/agents_sync/test_status.py` — with the shutdown predicate returning `True`, assert `_refresh_project_status`
  performs no `git fetch` and no object reads (inject a `git_runner` that records calls), and that the returned status
  matches the reconciled/previous status.
- A regression test that `_do_quit` sets the shutdown signal.
- An end-to-end check is worth doing manually with the tmux reproduction above; assert the process exits in under ~2s
  with an agents-sync sweep deliberately in flight.

## Acceptance criteria

- Quitting `sase ace` with `q` returns the shell prompt within ~2 seconds, including while a network `git fetch` from
  the agents-sync sweep is in flight.
- `sase ace -R` / `--restart-axe` restart-on-exit still works.
- `sase ace --profile` still writes and reports the profile file.
- Telemetry that previously flushed via `atexit` still flushes.
- No `sase.ace` import appears in `src/sase/agents_sync/`.
- `just check` passes (run `just install` first — workspaces are ephemeral and dependencies may be stale).

## Notes for the implementer

- Read `sase/memory/tui_perf.md` via the `/sase_memory_read` skill before touching TUI startup/teardown paths.
- Read `sase/memory/symvision.md` before adding new module-level symbols; new private helpers must satisfy the symvision
  lint.
- `py-spy` is the right tool to confirm the fix: `py-spy dump --pid <pid>` on a post-quit process should find nothing to
  dump because the process is already gone.
- Unrelated observation found during diagnosis, **do not fix in this plan**: `action_quit` swallows `q` entirely when
  `_toggle_tracked_artifact_file_tmux_pane()` returns truthy. That is intended behavior (close the artifact pane first),
  but it makes "q did nothing" a plausible separate user report. Worth a separate bead if it comes up.
