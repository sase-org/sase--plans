---
tier: tale
title: Green the Master Gate shard-4 red
goal:
  The coordinated bead-writer test fails with per-worker stacks instead of a silent
  _queue.Empty, and real-startup AcePage tests never run the live host/network update
  check.
size: medium
proposed_by: bbugyi200.athena.1gt.w0
proposed_with: claude/opus
create_time: 2026-11-26 16:29:05
status: wip
---

# Green the Master Gate shard-4 red: self-diagnosing bead writers and hermetic real-startup update checks

Governing decisions: `decisions:ci-two-speed-split` (Master Gate is the per-SHA fast
gate this fixes), `decisions:check-full-is-explicit` and `decisions:guarded-recipes`
(verify with `sase tool run check`, never `just check-full`).

## Evidence

`actstat` shows one red sase workflow: Master Gate #3802 (run 45886580131) on master
`ebe2eb17c4`. Only job `test (4)` failed, on
`tests/test_bead/test_beads_db_single_owner.py::test_coordinated_writers_keep_unique_child_ids`.
It raised `_queue.Empty` at line 332 after 120.47s. The same test, with the same
signature, turned Master Gate 45843695662 red on `e251ae713d` (job 139991960239). Epic
`sase-1tc` carries a DISCOVERED ISSUE note for that run (its phase `sase-1tc.4` wrote
the test). No task bead owns it.

The other red Master Gates from the last day fail different tests, so they are not this
plan's work:

- Shard 1, `test_refresh_warmup_gc`. Already owned by flake bead `sase-1v2`.
- Shard 5,
  `tests/axe/test_axe_chop_batch_process_recovery.py::test_typed_clan_kill_after_pause_stays_cancelled`
  (run 45870206302). It raised `ChopBatchError: ... launch request is missing`. No
  owner.

### Finding 1: the bead test hides why its writers never started

- The parent waits on `ready_queue.get(timeout=120)`. The first `get` timed out, so none
  of the four spawn-context children reached `ready_queue.put`.
- A child only has to run `BeadProject(Path(project_dir))` before `put`.
  `BeadProject.__enter__` is a no-op and `load_config` is a JSON read. Locally the child
  finishes its import in about 0.2s. `strace` shows no `execve`, `connect`, or `flock`
  before the ready message. The whole test passes in about 0.6s.
- The test throws away every failure signal:
  - An exception inside `BeadProject(...)` is reported on `result_queue`. The parent
    never reads that queue before it fails.
  - A dead or stuck child is never noticed. The parent never checks liveness or
    `exitcode`, and the child never dumps its stack.
- A spawn child's crash under xdist does show up as "Captured stderr call". I verified
  this with a scratch test: `exitcode=1` and the traceback were shown. The CI failure
  has no captured output at all. So the four children were stuck, or killed by a signal,
  before their first line of test code. They did not raise.
- The stall was not runner-wide. In run 45843695662 the other workers kept finishing
  tests through the whole 120s window: there is no progress gap before the `F`.
- Ruled out:
  - Tests that patch `sys.executable`. `multiprocessing.spawn` caches it as
    `_python_exe` on first import, but none of those tests are in shard 4. I computed
    shard 4's file list from `tests/_test_shards.py`.
  - The shard-4 tests that send signals. They only signal their own worker's descendants
    or process groups.
- The exact stall frame cannot be recovered from CI, because the test records none.
- The spawn mechanism also carries more process-global state than the test needs.
  `multiprocessing.get_context("spawn")` starts each child from the long-lived xdist
  worker's cached multiprocessing globals: `spawn._python_exe`, `process.ORIGINAL_DIR`,
  a copied `sys.path`, the resource-tracker fd, and signal dispositions (spawn uses
  `restore_signals=False`). Each child also re-imports pytest and the test module (under
  `--import-mode=importlib`), only to unpickle `_child_creator`. None of that is what
  the test proves. The test is about the Rust bead mutation lock and child-ID allocation
  across real processes.

### Finding 2: real-startup AcePage tests run the live host and network update check

- Both failing shard-4 runs also had teardowns of about 120s:
  - `tests/ace/tui/test_agents_pane_mount.py::test_agents_pane_mounts_activates_and_loads`
    took 122.92s and 123.18s.
  - Run #3802 also had two `tests/ace/tui/test_artifacts_post_paint_publication.py`
    tests at 121.82s and 121.42s.

  Passing shard-4 runs (45861705514, 45846683375) have none of these. They added about
  170s of wall time to the shard.

- I reproduced the slow teardowns locally (3.5–4.8s each) and dumped thread stacks.
  Teardown is blocked in pytest-asyncio's `Runner.close()` →
  `loop.shutdown_default_executor()`. It is joining the `automatic-update-check` Textual
  thread worker, which is still running real host probes. The call chain is:
  1. `AceApp._run_startup_update_toast_check`
  2. `update_toast._get_automatic_update_status`
  3. `get_cached_update_status`
  4. `compute_update_status`
  5. `_compute_agent_cli_source`
  6. `detect_agent_cli_statuses`
  7. `_probe_npm_environment`
  8. `subprocess.run(["npm", ...])`

  `compute_update_status` also makes many PyPI and GitHub version fetches, each with a
  2–20s timeout.

- `AcePage._drain_textual_workers` returns in 0.00s because it marks the thread worker
  cancelled. The OS thread keeps running until the event-loop runner joins it.
- `startup_policy="fast"` turns `_schedule_startup_update_toast_check` into a no-op.
  `startup_policy="real"` (13 test files) runs the update check against the real host
  and network. The exception is when the test patches
  `sase.ace.tui.actions.update_toast.get_cached_update_status` itself. Only
  `tests/ace/tui/test_update_toast_startup.py` and
  `tests/ace/tui/visual/test_ace_png_snapshots_update_toast.py` do that.
  `update_toast.py` is the only ACE module that imports `get_cached_update_status`.
- Finding 2 is the proven cause of the ~120s teardown stalls in both failing jobs. It is
  not proven to cause Finding 1's child stall. Fix both: the stalls are a non-hermetic
  network dependency in the same red job, and fixing them removes most of the shard's
  extra wall time.

## Changes

### 1. Rewrite the coordinated-writers test with plain subprocess workers and a real barrier

In `tests/test_bead/test_beads_db_single_owner.py`:

- Remove the `multiprocessing` import and `_child_creator`. Add a module-level worker
  script constant. Run it as
  `[sys.executable, "-c", _WORKER_SCRIPT, project_dir, epic_id, str(idx), str(_WORKER_STALL_DUMP_SECONDS)]`
  with:
  - `cwd=REPO_ROOT`, the same as this file's existing subprocess helpers;
  - binary `stdin=PIPE` and `stdout=PIPE`;
  - stderr sent to a per-worker file under `tmp_path`, so a large stack dump can never
    deadlock a pipe.
- The worker script must do the following, in this order:
  1. Call `import faulthandler, sys` and then
     `faulthandler.dump_traceback_later(float(sys.argv[4]), exit=True)`. These must be
     its very first statements, before any `sase` import. A stalled worker then writes
     every thread's stack to its stderr file and exits non-zero.
  2. Import only `IssueType` from `sase.bead.model` and `BeadProject` from
     `sase.bead.project`. Never import pytest or the test module.
  3. Build `BeadProject(Path(sys.argv[1]))`, write `READY <idx>\n` to stdout, and flush.
  4. Block on `sys.stdin.readline()` until it reads `GO`.
  5. Create the phase child under the epic. Update its description to `worker <idx>`.
     Print one JSON line with `worker` and `child_id`.
  6. On any exception, print the traceback to stderr and exit 1.
- Set `_WORKER_STALL_DUMP_SECONDS = 90.0`, below the unchanged
  `_PROCESS_TIMEOUT_SECONDS = 120.0`, so the dump lands before the parent's bound. Do
  not raise either bound. They are hang detectors.
- The parent does the following:
  - Start all four workers.
  - Wait for every `READY` line under one deadline of `_PROCESS_TIMEOUT_SECONDS`. Use
    `selectors` over the stdout pipes with `os.read` into per-worker buffers, so a
    partial line can never overrun the deadline. EOF before `READY` means the worker
    died and fails at once. Do not sleep-poll.
  - On a READY failure, raise one `AssertionError`. For each worker it lists the index,
    the `returncode` (poll briefly after kill), stdout so far, and the full stderr file,
    including any faulthandler dump.
  - Then write `GO\n` to every stdin and close it, so all four writers contend for the
    Rust mutation lock at the same moment. The current test has no go-barrier, so this
    is a strictly stronger version of the same proof.
  - Collect each worker with `communicate(timeout=<remaining deadline>)`. A non-zero
    exit or a missing or invalid JSON line produces the same per-worker diagnostic
    message.
  - In `finally`, kill and wait any live worker.
- Keep every existing assertion unchanged:
  - no failed workers;
  - four unique child IDs equal to `f"{epic_id}.{n}"` for n in 1..4;
  - a reopened project's `get_epic_children` matches;
  - each child's description is `worker <idx>` for the worker that created it.
- Update the module docstring sentence about the "coordinated subprocess writer" to
  describe the ready/go barrier and the stall dump.
- Wait-lint conformance (`tools/check_test_wait_helpers`, which `sase tool run check`
  runs):
  - Use no fixed sleep. If one is unavoidable, it needs an inline
    `# sase-test-wait: <reason>` pragma.
  - Add no private wait helper of the `_wait_until` style.

### 2. Pin the update-status source under AcePage's real startup policy

- In `src/sase/ace/testing/_startup.py`:
  - Capture `_ORIGINAL_GET_CACHED_UPDATE_STATUS` at import, alongside the existing
    `_ORIGINAL_*` sentinels.
  - Add an `_install_real_startup_isolation(stack)` helper (name is flexible). It
    patches `sase.ace.tui.actions.update_toast.get_cached_update_status` with an inert
    stub that returns `None`, but only while the attribute is still the original. This
    is the same patch-if-unchanged idea as `_patch_method_if_unchanged`, so a test that
    installs its own fake before entering `AcePage` keeps it. Both update-toast test
    files rely on that.
- In `src/sase/ace/testing/ace_page.py`, call the helper when
  `self._startup_policy == "real"`. Put it beside the existing fast-policy branch
  (around line 297), before `AceApp` is constructed.
- Under the real policy the following still run for real:
  - `_schedule_startup_update_toast_check`;
  - the `automatic-update-check` thread worker;
  - `_finish_automatic_update_check_from_worker` and the guard release.

  So `test_ace_page_real_startup_policy_invokes_production_boundaries` stays valid
  unchanged. A `None` status returns before `_schedule_rust_prebuild` and before any
  npm, PyPI, or GitHub probe.

- Update the `AceStartupPolicy` / `AcePage` docstring. The real policy runs every
  production startup boundary but never reaches the host or network update-status source
  unless the test supplies one.

### 3. Regression tests

Add these next to `test_ace_page_real_startup_policy_invokes_production_boundaries` in
`tests/_ace_testing_startup.py`, which is collected through `tests/test_ace_testing.py`:

- Hermeticity test. Monkeypatch `sase.updates.status.compute_update_status` and
  `sase.agent_clis.runner.run_command` to record any call and raise. Mount
  `AcePage(startup_policy="real")`. Wait under a bounded deadline, using the existing
  awaitable pattern in that file, until the update check has finished
  (`app._automatic_update_check_in_flight` is False after it was scheduled). Assert that
  neither recorder was called.
- Fake-wins test. Monkeypatch `update_toast.get_cached_update_status` to a recording
  fake before entering `AcePage(startup_policy="real")`. Assert that the fake was
  called, which proves the patch-if-unchanged guard leaves caller fixtures alone.

### 4. Bookkeeping

- Use `/sase_new_task` to file a `flake` task bead, size `large`, for
  `test_typed_clan_kill_after_pause_stays_cancelled`. Include the run, job, error text,
  and the observation that the next Master Gate's shard 5 passed. This plan does not fix
  it.
- Add a note on epic `sase-1tc` that points its DISCOVERED ISSUE (the Master Gate
  45843695662 `_queue.Empty`) at this fix. Do not edit bead status by hand.

## Verification

- Run the bead test serially, then with stress:
  - `.venv/bin/python -m pytest -q -p no:randomly tests/test_bead/test_beads_db_single_owner.py`
  - `just test-contention -- tests/test_bead/test_beads_db_single_owner.py`. This is the
    existing pinned-CPU repeat harness. Run it through `sase tool run`, or through
    `/sase_monitor` if it outruns the turn.

  Record the tally.

- Prove the new diagnostics work, once and locally, without committing anything.
  Temporarily make the worker script sleep past a lowered stall-dump bound. The failure
  message must show the worker's faulthandler stack and its non-zero `returncode`
  instead of a bare `Empty`. Revert the change.
- Run the three formerly slow real-startup tests with `--durations=5`. Their teardowns
  must drop from about 3.5–4.8s to well under 1s locally:
  - `tests/ace/tui/test_agents_pane_mount.py`
  - `test_first_paint_fixed_panes_then_one_coherent_publication`
  - `test_user_navigation_supersedes_pending_provider_jump`
- Run `tests/test_ace_testing.py`, `tests/ace/tui/test_update_toast_startup.py`, and
  every other file that uses `startup_policy="real"` (find them with
  `grep -rl 'startup_policy="real"' tests`). Visual PNG files run only through
  `just test-visual -- <selectors>`. Run them only if their rendered output could have
  changed; it should not.
- Run `sase tool run check` for every lint gate (ruff, mypy, symvision, the test-waits
  lint, and the rest) plus the scoped test lane. Do not run `just check-full`.
- After host completion lands the commit, Master Gate runs on the new tip. Shard 4 is
  the proof. A later recurrence of the bead test must now fail with per-worker stacks,
  not `_queue.Empty`.

## Out of scope

- Fixing `test_typed_clan_kill_after_pause_stays_cancelled` (it is filed in step 4).
- Fixing `sase-1v2` (the warmup-GC Textual `active_worker` pin).
- Changing `AcePage._drain_textual_workers`' thread-worker semantics, or making the
  production update check itself cheaper.
- Any `sase-core` change. This is test infrastructure only.
