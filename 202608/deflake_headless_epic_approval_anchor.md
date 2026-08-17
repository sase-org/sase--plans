---
tier: tale
title: Deflake headless epic approval against an in-flight launch (sase-nz)
goal:
  tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor
  stops failing intermittently under the full parallel test lane, with its
  inflight-launch-holds-anchor assertion intact and no change to production approval or
  launch semantics.
size: medium
proposed_by: bbugyi200.athena.sase-ns.6.6.5
bead: sase-ns.6.6.5
create_time: 2026-08-17 04:39:24
status: done
---

- **PARENT:** [202608/backlog_top5_gates_green.md](backlog_top5_gates_green.md)
- **BEAD:**
  [sase-ns.6.6.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.6.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ns.6.6.5](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ns.6.6.5.md)
- **COMMITS:**
  - [b6246f1](https://github.com/sase-org/sase/commit/b6246f1cfb8b1d4d9c2d524efab7c4082ba2ee93)
    — test: deflake headless epic approval against an in-flight launch

# Deflake Headless Epic Approval Against An In-Flight Launch (sase-nz)

## Goal

`tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor`
stops failing intermittently under the full parallel test lane, with its
inflight-launch-holds-anchor assertion intact and no change to production approval or
launch semantics.

This is epic phase `approval_anchor` of `plan:202608/backlog_top5_gates_green.md` (bead
**sase-ns.6.6.5**, task bead **sase-nz**).

## Root Cause — Already Established

Do not re-derive this. It was measured before this plan was written; the numbers below
are reproducible with the commands named in each bullet.

### What the test does today

```python
process_context = multiprocessing.get_context("fork")
holder_acquired = process_context.Event()
release_holder = process_context.Event()
process = process_context.Process(target=_hold_epic_approval_lock, args=(...))
process.start()
assert holder_acquired.wait(2.0) is True        # bound 1
try:
    with (...four patches...):
        result = execute_plan_approval_response(context, "epic")
finally:
    release_holder.set()
    process.join(timeout=2.0)                   # bound 3
assert result.epic_launch_monitor_id == "mon-contended"
start_launch.assert_called_once()
assert process.exitcode == 0
```

and the forked holder (`_hold_epic_approval_lock`, top of the same file) holds the
anchor's `flock` only until `release.wait(5.0)` returns — **bound 2**.

### Every failure mode of this node is a harness artifact

`require_epic_launch_store_health()` (`src/sase/bead/cli_work_from_plan_store.py`) calls
`resolve_beads_location(...)` **only when it acquires the anchor lock**. The test
patches `resolve_beads_location` to raise
`AssertionError("contended preflight must not materialize the sidecar")`. So that
assertion can only fire if the holder let the lock go early — i.e. only if bound 2
expired. The remaining assertions are bound 1 and bound 3. There is no path by which
production code reaches the sidecar while a foreign holder actually holds the anchor, so
nothing here indicts production behavior.

### The mechanism

The node forks from a **multi-threaded** pytest-xdist worker, which Python 3.14 itself
flags:

```
multiprocessing/popen_fork.py:70: DeprecationWarning: This process (pid=...) is
multi-threaded, use of fork() may lead to deadlocks in the child.
```

Measured (instrumented `Process.start`, `Event.wait`, `Process.join`):

| Run                            | `len(sys._current_frames())` at fork | `threading.enumerate()` |
| ------------------------------ | ------------------------------------ | ----------------------- |
| serial pytest                  | 1                                    | `["MainThread"]`        |
| `-n 2 --dist=loadfile` (xdist) | **2**                                | `["MainThread"]`        |

The extra thread is execnet's receiver thread. It is created through
`_thread.start_new_thread`, so `threading.enumerate()` cannot see it — but it holds a
real interpreter thread state, and every fork in the parallel lane is therefore a fork
from a multi-threaded process. Any lock a co-resident thread holds at fork time (the
inherited `sys.stdout`/`sys.stderr` buffer locks that `BaseProcess._bootstrap` flushes
on the way out are the classic one, and are not reinitialised by `PyOS_AfterFork_Child`)
stays locked forever in the child, which shows up as bound 1 or bound 3 expiring. The
live thread set depends on which other test files that worker ran first, which is
exactly why:

- the node never reproduces in isolation or in a file-scoped contention soak
  (`SASE_CONTENTION_REPEAT=4 just test-contention -- tests/test_plan_approval_actions.py`
  measured 0 failures across 4 repeats — the sase-mv shape the epic plan warns about),
- but it has 12 full-run failure records from six workspaces at twelve HEADs with
  pairwise-disjoint change sets.

Wall-clock starvation alone does not explain it: the holder window is **0.039 s** of
real work in-process (instrumented stage timings: `can_claim` → `run_plan_side_effects`
→ preflight), fork takes 0.005 s and join 0.006 s. The reported full-lane failures ran
6–13 xdist workers on a 64-core host, where a 125×–300× stall of one process is not
credible; a fork-inherited lock that never unlocks is.

### Why the fix below is not a weakening

`flock(2)` locks belong to the **open file description**, not the process: a second
`open()` of the same path in the same process is denied exactly as a separate process
is. This repo already relies on that in
`tests/test_sdd_git_contention.py::test_plan_file_launch_holds_anchor_lock_before_store_resolution`,
which probes the same lock path from the test process and asserts `BlockingIOError`.

It was verified end-to-end against the real production helper before this plan was
written: holding `_epic_plan_launch_lock_path(anchor)` on one file descriptor, a second
descriptor in the same process is refused with `BlockingIOError`, and
`require_epic_launch_store_health(workspace)` logs

```
Timed out after 0.020s waiting for epic plan launch lock ... during epic approval
preflight; deferring to holder pid ... is running test in-flight epic launch ...
```

and returns **without calling `resolve_beads_location`**. That is the identical
production path the forked holder exercises, minus the fork.

Genuine cross-process coverage of the lock itself is not lost — it lives in
`tests/test_sdd_git_contention.py`
(`test_epic_plan_launch_lock_blocks_other_process_for_same_canonical_anchor`,
`test_epic_plan_launch_lock_does_not_serialize_distinct_anchors`,
`test_epic_launch_preflight_defers_without_materializing_under_contention`), which stay
untouched by this plan.

## Scope

Exactly one file changes: `tests/test_plan_approval_actions.py`. No production source
file changes. No other test file changes.

### 1. Add an in-process foreign-holder seam

Add a module-level context manager to `tests/test_plan_approval_actions.py` that holds
the anchor's launch lock the way another process would:

- Resolve the lock path with `_epic_plan_launch_lock_path(anchor)` from
  `sase.bead.cli_work_from_plan_store` (importing that private helper into a test module
  is the existing convention — `tests/test_sdd_git_contention.py` already imports it and
  four sibling private symbols).
- `mkdir(parents=True, exist_ok=True)` the parent, open the lock path `"a+"`, take
  `fcntl.flock(fd, fcntl.LOCK_EX | fcntl.LOCK_NB)`.
- Write the same holder-identity JSON shape `_write_lock_holder` writes (`pid`, `op`,
  `plan_file`, `started_at`) and flush, so the production deferral path reads a real
  holder record instead of falling back to "the current holder did not record its
  identity". Use `op="test in-flight epic launch"` and the plan file the test already
  passes, keeping the existing holder's identity.
- Self-check inside the helper, before yielding: open the same path a second time and
  assert `pytest.raises(BlockingIOError)` on a non-blocking `LOCK_EX`. This makes the
  seam prove it is indistinguishable from a foreign holder rather than assuming it, and
  fails loudly if the anchor or lock-path derivation ever stops matching.
- `LOCK_UN` and close on exit.

Give it a docstring that states why it is not a fork: `flock` ownership is per open file
description, the contended window is bounded by the `with` block instead of a wall-clock
timer, and forking a multi-threaded xdist worker is the defect this replaces.

### 2. Rewrite the node to use it

`test_headless_epic_approval_submits_while_inflight_launch_holds_anchor` becomes:

- keep `_epic_context(tmp_path)`, `epic_launch_lock_anchor(workspace)`, and
  `monkeypatch.setenv("SASE_EPIC_APPROVAL_PREFLIGHT_LOCK_TIMEOUT", "0.02")` unchanged;
- enter the new holder context manager, and inside it the **same four `patch(...)`
  calls, unchanged** — `resolve_epic_launch_project`, `get_workspace_directory`,
  `resolve_beads_location` (still
  `side_effect=AssertionError("contended preflight must not materialize the sidecar")`),
  and `start_epic_launch_monitor`;
- call `execute_plan_approval_response(context, "epic")` inside that block;
- keep `assert result.epic_launch_monitor_id == "mon-contended"` and
  `start_launch.assert_called_once()`.

Drop `assert process.exitcode == 0` — it asserted the _harness holder subprocess_ exited
cleanly, not any property of the code under test, and there is no subprocess left.
Nothing else may be removed or relaxed.

### 3. Delete what the fork left behind

- Delete `_hold_epic_approval_lock` (now unused).
- Drop the `import multiprocessing` and any other import the deletion orphans (e.g.
  `typing.Any` if nothing else in the file uses it). Let `just lint` decide; do not
  guess.

## Verification

Run in this order. `just install` first — workspaces are ephemeral and may hold stale
dependencies.

1. `just install`
2. `.venv/bin/python -m pytest tests/test_plan_approval_actions.py -q` — the whole file
   green.
3. Confirm the fork is gone. Under xdist the node used to emit the multi-threaded-fork
   `DeprecationWarning`; it must not any more:

   ```bash
   .venv/bin/python -m pytest -n 2 --dist=loadfile -q \
     tests/test_plan_approval_actions.py tests/test_sdd_git_contention.py 2>&1 \
     | grep -c "use of fork() may lead to deadlocks"
   ```

   Expect `0` from `tests/test_plan_approval_actions.py`.
   `tests/test_sdd_git_contention.py` still forks by design and may still warn — check
   the warnings summary names only that file. This is the deterministic proof the defect
   class is removed; it does not depend on winning a race.

4. `SASE_CONTENTION_REPEAT=4 just test-contention -- tests/test_plan_approval_actions.py`
   — 0 failures across 4 repeats. Report the tally verbatim. Note this lane was already
   green before the fix, so it is a no-regression check, not the evidence that the fix
   works; step 3 is that evidence.
5. `just check` — green. Report the scoped selection's outcome.
6. Full-lane evidence. `just check-full` routinely outruns one agent turn, so start it
   **only** through `/sase_monitor`:

   ```bash
   sase monitor start --command 'just check-full' --next '<action>'
   ```

   The `--next` action must instruct the follow-up agent to (a) append the result as a
   note on bead **sase-ns.6.6.5**, naming the exact `flake baseline gate:` node list,
   and (b) treat a failure of
   `tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor`
   as a regression of this phase.

## The Flake Gate Will Stay Red On This Node — Say So

`just selection-health --fail-on-new-flake` currently names three exceeding nodes:

```
tests/fakey/test_usage_limit_e2e.py::test_usage_limit_failure_disables_only_fakey_and_preserves_error
tests/monitor/test_monitor_supervise.py::test_run_supervisor_idle_timeout_fires_after_output_stalls
tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor
```

The first belongs to epic sase-n4 and the second to sibling phase sase-ns.6.6.4 — leave
both alone. The third is this node, held red by **12 pre-fix full-run failure records**
(2026-08-06T13:35:09Z … 2026-08-17T01:37:59Z). Those retire only through a `# fixed-at:`
entry in `tests/reproducible_flake_baseline.txt`, and the file's convention — reaffirmed
by the epic plan's `flake_retire` guardrail — is that such an entry must name the commit
that fixed the node.

**Do not add that entry, and do not commit.** Per `sase/memory/xprompts.md`, agents
never create git commits directly; the provider-neutral finalizer commits the working
tree after the phase ends, so the fix commit's hash does not exist while this plan is
being implemented. Instead, leave the land agent an exact handoff on bead sase-ns.6.6.5:

```
PROPOSED FOLLOW-UP: retire sase-nz's pre-fix flake evidence — once this phase's fix
commit lands, append to tests/reproducible_flake_baseline.txt a comment naming bead
sase-nz and that commit, then
`# fixed-at: <commit UTC timestamp> tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor`
(12 pre-fix records, last 2026-08-17T01:37:59Z), and confirm
`just selection-health --fail-on-new-flake` no longer names the node.
```

Sibling phase sase-ns.6.6.4 also writes to that file, so leaving the entry to the land
agent additionally avoids two workspaces editing the same block concurrently.

## Bead Protocol

Bead **sase-ns.6.6.5** is already `in_progress` — do not set its status by hand.

- Record discovered follow-up work as
  `sase bead note sase-ns.6.6.5 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`. Do
  not create beads.
- Close only sase-ns.6.6.5, with
  `sase bead close sase-ns.6.6.5 --note "<what you verified>"`. Never close the parent
  epic or any ancestor.
- The note must state explicitly that the inflight-launch-holds-anchor assertion is
  unchanged: the four patches are byte-identical, `resolve_beads_location` still raises
  `AssertionError` if the preflight ever materializes the sidecar, and
  `start_launch.assert_called_once()` plus the `mon-contended` monitor-id assertion both
  survive. Name the one dropped assertion (`process.exitcode == 0`) and why it was about
  the deleted harness subprocess rather than the code under test.

### Approval trigger

This phase's approval trigger (per the epic plan) fires only if the fix requires
changing **production** approval or launch semantics — the anchor protocol, the approval
lock's scope, or when a launch claims a workspace. This plan changes none of them; it
changes one test file. So no `TASK NEEDS APPROVAL` note is owed. If implementation
discovers that a production change is unavoidable after all, stop at that boundary and
append a `TASK NEEDS APPROVAL` note to sase-ns.6.6.5 describing the semantic change and
why test-level isolation is insufficient.

## Known Follow-Ups To Record (Not To Do)

Record these as `PROPOSED FOLLOW-UP:` notes on sase-ns.6.6.5; they are out of this
phase's scope:

1. The baseline retirement handoff quoted above.
2. `tests/test_sdd_git_contention.py` still forks a multi-threaded xdist worker in three
   places (`_acquire_epic_plan_launch_lock`, `_hold_epic_plan_launch_lock` and their
   three call sites) with the same 2.0 s / 5.0 s / 2.0 s bounds. None of those nodes is
   a reported flake today — their holder windows are a single fast call rather than a
   whole approval — but they carry the same hazard and the seam this phase adds is
   directly reusable.
3. In-progress epic **sase-j7** owns process-global state leaking between tests. This
   node's failure mode is a fork-inherited lock rather than a leaked module global, so
   sase-j7's phase sase-j7.2 leak detector (`tests/_global_state_leak_detector.py`) does
   not instrument it — it fingerprints module globals, env, and caches, not live
   interpreter threads. Worth recording that "how many interpreter threads does this
   worker have, and who left them" is an adjacent gap that detector does not cover.
