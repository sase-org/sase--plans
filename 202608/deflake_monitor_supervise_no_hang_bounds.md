---
tier: tale
title: Deflake the monitor-supervise no-hang bounds
goal:
  The six tests/monitor/test_monitor_supervise.py nodes that bound supervisor runtime
  measure a child process's liveness instead of the pytest worker's own scheduling
  delay, every supervisor behavior assertion they carry survives unchanged, and
  sase-lk's committed reproducible-flake baseline entry is gone.
size: medium
proposed_by: bbugyi200.athena.sase-ns.6.6.6.3
bead: sase-ns.6.6.6.3
create_time: 2026-08-17 06:15:43
status: done
---

- **PROMPT:**
  [prompts/202608/deflake_monitor_supervise_no_hang_bounds.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/deflake_monitor_supervise_no_hang_bounds.md)
- **PARENT:**
  [202608/backlog_top_five_gates_and_flakes.md](backlog_top_five_gates_and_flakes.md)
- **BEAD:**
  [sase-ns.6.6.6.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.6.6.6.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ns.6.6.6.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ns.6.6.6.3.md)
- **COMMITS:**
  - [44df0bf](https://github.com/sase-org/sase/commit/44df0bfb420c3fd2b291e7ed2aace67046fd0b0b)
    — test(monitor): deflake the supervise no-hang bounds

# Deflake the monitor-supervise no-hang bounds

This is phase `supervise` of the epic plan
`202608/backlog_top_five_gates_and_flakes.md`, and it works task bead `sase-lk`.

## Goal

Complete phase bead `sase-ns.6.6.6.3`. The three `sase-lk` nodes
(`test_run_supervisor_escalates_term_ignoring_chatty_child`,
`test_run_supervisor_times_out_after_partial_line`,
`test_run_supervisor_completes_when_grandchild_holds_stdout`) must stop failing other
agents' full parallel lanes, and `sase-lk`'s node-ID entry must come out of
`tests/reproducible_flake_baseline.txt`. No supervisor behavior assertion may be
weakened, and no baseline entry may be added.

## Evidence

All record counts below come from the durable store
`~/.sase/test-selection/gh_sase-org__sase` (schema-2 `full-run` records, 30-day
retention), read on 2026-08-17 from workspace `sase_16` at clean master `cf7eeee03`.

### 1. Exactly the wall-clock-bounded nodes flake

Nine node IDs from this file appear in failure records. Ranked by record count, with
"eligible" meaning inside the gate's current window
(`effective-after: 2026-08-15T17:22:27Z`, at most 5 failures per run):

| node                                                | records | newest               | eligible now |
| --------------------------------------------------- | ------- | -------------------- | ------------ |
| `escalates_term_ignoring_chatty_child`              | 15      | 2026-08-16T18:54:21Z | 0            |
| `completes_when_grandchild_holds_stdout`            | 11      | 2026-08-16T16:59:34Z | 1            |
| `times_out_after_partial_line`                      | 6       | 2026-08-17T01:20:25Z | 1            |
| `times_out_after_child_closes_stdio`                | 4       | 2026-08-16T16:59:34Z | 1            |
| `kills_the_whole_process_group_on_timeout`          | 3       | 2026-08-15T18:22:05Z | 0            |
| `idle_timeout_fires_after_output_stalls`            | 3       | 2026-08-17T01:20:25Z | 2 (retired)  |
| `times_out_continuous_output`                       | 2       | 2026-08-13T18:37:00Z | 0            |
| `holds_the_claim_when_the_followup_launch_succeeds` | 1       | 2026-08-13T16:12:45Z | 0            |
| `releases_the_claim_when_the_followup_launch_fails` | 1       | 2026-08-13T16:12:45Z | 0            |

The first seven are **exactly** the seven nodes in the file that bracket an in-process
`run_supervisor()` call with `time.monotonic()` and assert `elapsed < _NO_HANG_TIMEOUT`.
The seventh (`idle_timeout_fires_after_output_stalls`) was converted to a child process
by `f9ab15d9c` earlier in this epic and has not failed since. The last two are a single
2026-08-13T16:12:45Z record that predates the gate window by two days, in a run whose
other eight failures are `tests/main/test_monitor_handler_list.py`,
`test_monitor_handler_show.py`, and `test_project_handler_list_show.py` nodes — a
CLI-handler-wide failure at that head, not a timing flake. Neither carries a timing
bound and neither is in scope here.

Every other node in the file — including `chatty_command_does_not_hit_idle_timeout`,
`survives_invalid_utf8_output`, and `records_a_clean_completion_and_releases_the_claim`,
which stream output through the same `BoundedLogPipe` — has **zero** records.

### 2. The pipe-EOF theory is falsified

`sase-lk`'s description names `test_run_supervisor_times_out_after_child_closes_stdio`
as the control that never flakes, precisely because its child runs
`exec >/dev/null 2>&1` so the output pipe EOFs before `close()` is ever called. That
node has **4** failure records, including one in the gate's current window
(2026-08-16T16:59:34Z), in the same run as `completes_when_grandchild_holds_stdout`. A
defect in `BoundedLogPipe.close()`'s EOF-versus-drain handling cannot explain a node
whose pipe has already EOF'd, and it cannot explain why nodes that write far more output
through that pipe never fail. The drain-deadline theory this bead was reopened against
is not the residual mechanism.

### 3. The residual mechanism was already measured on a sibling node in this file

Phase `sase-ns.6.6.4` (bead `sase-nd`, plan `202608/deflake_monitor_idle_bound.md`)
diagnosed `idle_timeout_fires_after_output_stalls` in this same file. Its representative
real-lane failure was solely `assert 5.825556540999969 < 5.0`, with the preceding
`exit_status == 1` assertion passing — the supervisor had already fired the idle timeout
and returned the right status, and the test rejected it anyway. That node's serial call
time is ~0.25s, so the measured quantity was inflated more than twentyfold. The accepted
fix (`f9ab15d9c`, retired in `cf7eeee03`) was to run the supervisor in a child process
under `subprocess.run(..., timeout=_NO_HANG_TIMEOUT)`, because `subprocess.run` inspects
the child's state before declaring its deadline expired: a child that finished while the
parent was starved still passes, while a child still alive at the deadline fails.

This phase applies the same, already-validated seam to the nodes that still carry the
in-process bracket.

### 4. The recorded failures look like host pressure, not supervisor logic

Every failure in the gate's current window sits in a run whose other failures are
unrelated to monitors or pipes:

- `2026-08-17T01:20:25Z` (`sase_15`, 3 failures): `times_out_after_partial_line`
  together with `idle_timeout_fires_after_output_stalls` — the node whose in-process
  bracket has already been proven to be the defect — plus a config-center node.
- `2026-08-16T16:59:34Z` (`sase_21`, 3 failures):
  `completes_when_grandchild_holds_stdout` and `times_out_after_child_closes_stdio` in
  one run, alongside a subprocess-heavy `var` CLI end-to-end node.
- `2026-08-16T00:43:14Z` and `2026-08-16T00:57:33Z`:
  `escalates_term_ignoring_chatty_child` in runs whose other failures are entirely
  keybinding, keymap, config, and models-panel nodes.

Two different nodes of this file failing in the same three-failure run, and one of them
being the already-diagnosed bracket node, both point at the shared measurement rather
than at seven independent logic defects.

### 5. Local starvation reproduces the inflation but not a failure — and that is expected

Measured in this workspace at `cf7eeee03`:

- Serial baseline (`-n 0`), whole file: 21 passed in 8.79s. Call times for the six nodes
  in scope: 1.07s, 0.78s, 0.36s, 0.27s, 0.27s, 0.11s.
- `SASE_CONTENTION_CPUS=2,3 SASE_CONTENTION_REPEAT=3 just test-contention tests/monitor/test_monitor_supervise.py`
  (26 workers pinned to 2 CPUs, 97.0s): **3/3 repeats green, 0 nodes failed.**
  Per-repeat _setup_ durations of 5.6-8.1s prove the harness really is starving the
  workers, but it starves collection and setup, not the call window.
- 12, then 20 external CPU burners pinned to the same two cores, with `-n 12` over just
  the six nodes, three repeats: green (18.9s / 18.9s / 19.4s against ~3s serial). With
  20 burners the _call_ durations were 1.11s, 0.85s, 0.43s, 0.34s, 0.31s, 0.15s — barely
  above serial.

The reason is mechanical and worth writing down, because it explains `sase-lk`'s "NOT
REPRODUCED LOCALLY" history (~70 prior attempts, including 3.3x oversubscription):
during the call these tests are almost entirely _sleeping_ (`_wait_for_child` polls with
`time.sleep(0.05)`), and a sleeping process is not starved by CPU contention. A pinned
CPU soak therefore cannot reproduce what a real 13-worker lane does on a 64-core host
that is simultaneously running other agents' full lanes — where memory pressure (the
suite's own peak-worker-RSS budget is 1.1 GiB per worker), fork/exec storms, and disk
contention delay every wakeup, not just the runnable ones.

So this phase does not claim a local reproduction. It relies on: the node-set
correlation in item 1 (7/7 bracketed nodes have records, 0/14 unbracketed nodes do), the
falsifier in item 2, and the directly measured `assert 5.83 < 5.0` failure of the same
bracket in the same file in item 3.

### 6. No production defect is implicated

`run_supervisor`'s waits are all bounded and were re-read for this plan:
`_wait_for_launch_barrier` polls a 30s barrier deadline; `_wait_for_child` returns as
soon as `child.poll()` is not None and polls at `_POLL_SECONDS = 0.05`;
`BoundedLogPipe.close()` joins for `close_drain_seconds` (0.5s here) plus a 0.1s
allowance and ingests already-readable bytes before giving up (`b569cbdc2`). Nothing
waits on pipe EOF. No production change is proposed, and the no-hang guard is kept real
so that a regression which reintroduced an EOF wait would still fail.

## Implementation

All changes are in `tests/monitor/test_monitor_supervise.py` and
`tests/reproducible_flake_baseline.txt`. Do not touch `src/`.

### Step 1 — teach the existing child-process helper to carry constant overrides

`_run_supervisor_subprocess` (added by `f9ab15d9c`) already runs
`python -m sase.monitor.supervise --artifacts-dir <dir>` under a hard
`subprocess.run(timeout=...)`. One node in scope monkeypatches a module constant
(`_KILL_GRACE_SECONDS = 0.2`), and `monkeypatch.setattr` cannot cross a process
boundary, so the helper needs an override path. `monkeypatch.setenv` values do cross,
because the helper already passes `os.environ.copy()`.

Add an optional keyword-only `overrides: Mapping[str, float] | None = None`. When it is
None keep today's `-m sase.monitor.supervise` argv, so the real module entry point stays
covered. When overrides are given, run a small driver program with `-c` that imports
`sase.monitor.supervise`, rebinds each named constant, and exits with
`run_supervisor()`'s status.

The driver MUST fail loudly rather than silently skipping an override — otherwise a
renamed constant would leave the chatty-TERM node running with a 5s kill grace, and an
`AttributeError` exit status of 1 would be indistinguishable from the timeout exit
status that node asserts. Use a `getattr` existence check plus a sentinel exit code that
the helper turns into `pytest.fail`, for example:

```python
_SUPERVISOR_DRIVER_SETUP_FAILURE = 91

_SUPERVISOR_OVERRIDE_DRIVER = f"""\
import json
import sys

try:
    import sase.monitor.supervise as supervise

    for name, value in json.loads(sys.argv[2]).items():
        getattr(supervise, name)  # a renamed constant must not be skipped
        setattr(supervise, name, value)
except BaseException as exc:  # noqa: BLE001 - reported through the exit code
    print(f"supervisor driver setup failed: {{exc!r}}", file=sys.stderr)
    raise SystemExit({_SUPERVISOR_DRIVER_SETUP_FAILURE}) from None

raise SystemExit(supervise.run_supervisor(sys.argv[1]))
"""
```

and, in the helper, after `subprocess.run` returns, `pytest.fail` with
`completed.stderr` when `completed.returncode == _SUPERVISOR_DRIVER_SETUP_FAILURE`.

Keep the helper's single `except subprocess.TimeoutExpired` -> `pytest.fail` path; it is
the no-hang guard for every converted node. Its message must stay explicit that the
child was still alive at the deadline.

An equivalent shape (two thin wrappers over one core) is acceptable, as long as the file
ends up with exactly one no-hang idiom and one place where the deadline is enforced.

### Step 2 — derive the no-hang deadline from the regression it must catch

`_NO_HANG_TIMEOUT = 5.0` was chosen as "comfortably above" a ~1s expected runtime, and
its comment still explains itself in terms of `BoundedLogPipe`'s join timeout. It is not
a latency contract: none of these tests assert anything about how fast the supervisor
is, only about what it does. What it must actually catch is a supervisor that waits on a
descendant or on pipe EOF, and every node in scope arranges a **30-second** hold
(`sleep 30`, or an infinite `echo` loop) for exactly that purpose.

Raise it to `15.0` and rewrite the comment to say what it is: a hard liveness deadline
for a child process, at less than half the 30s hold every node uses, so a reintroduced
EOF/descendant wait still fails deterministically while host pressure has four times
more headroom than 5.0s gave it. Say in the comment that it must not be raised further
to quiet a failure: a child still alive at 15s is a real hang.

This is not the "widen the bound" non-fix the bead warns about. The flake mechanism is
removed by Step 1; this step only stops the deadline from being a number nobody derived.

### Step 3 — convert the six bracketed nodes

For each node below, delete the `started = time.monotonic()` /
`elapsed = time.monotonic() - started` / `assert elapsed < _NO_HANG_TIMEOUT` triple,
replace `exit_status = run_supervisor(artifacts_dir)` with
`completed = _run_supervisor_subprocess(artifacts_dir)`, and replace
`assert exit_status == N` with `assert completed.returncode == N`. Keep **every** other
assertion exactly as it is — `monitor_state`, `monitor_timeout_kind`, log contents, log
size caps, and the grandchild-liveness check:

1. `test_run_supervisor_kills_the_whole_process_group_on_timeout` (keep the
   `is_process_running(grandchild_pid)` assertion)
2. `test_run_supervisor_times_out_continuous_output` (keeps
   `monkeypatch.setenv("SASE_MONITOR_LOG_MAX_BYTES", "4096")`, which the child inherits)
3. `test_run_supervisor_times_out_after_partial_line` (see Step 4)
4. `test_run_supervisor_completes_when_grandchild_holds_stdout` (keep the `finally`
   grandchild cleanup)
5. `test_run_supervisor_times_out_after_child_closes_stdio`
6. `test_run_supervisor_escalates_term_ignoring_chatty_child` — replace
   `monkeypatch.setattr(supervise_module, "_KILL_GRACE_SECONDS", 0.2)` and its
   `import sase.monitor.supervise as supervise_module` with
   `_run_supervisor_subprocess(artifacts_dir, overrides={"_KILL_GRACE_SECONDS": 0.2})`;
   keep the `monkeypatch` parameter for the `SASE_MONITOR_LOG_MAX_BYTES` env patch

None of the six asserts `get_claimed_workspaces` or `load_notifications`, so moving
settlement into a child process changes nothing they observe; both still land under the
`SASE_HOME` the child inherits. Keep the `_restore_signal_handlers` autouse fixture: the
remaining in-process nodes still install process-wide handlers.

Watch two lint gates: the file must stay under `toobig`'s 700-line first threshold (651
lines today, and the conversion is close to net-neutral), and `just fmt-py-check` /
`ruff` must be clean.

### Step 4 — remove the second, independent hazard on the partial-line node

`test_run_supervisor_times_out_after_partial_line` is the only node in scope whose
assertions can fail for a non-timing reason: it asserts `live_reply.md` reads exactly
`"partial"`, which requires `/bin/sh` to spawn and `printf` to run inside the member's
own `timeout_seconds` budget. `b569cbdc2` already raised that budget from 0.2s to 1.0s
for this reason and said so in its message. It is also the node with the most
independent corroborations on `sase-lk` (4 of the 5 `+1` reports name it), one of which
describes something other than an assertion failure.

Raise that node's `timeout_seconds` from `1.0` to `3.0` and update its inline comment.
The child still sleeps 30s, so the node still proves a total timeout, and 3.0s is still
far below the 15s liveness deadline. Cost: that node goes from ~1.1s to ~3.6s.

### Step 5 — retire `sase-lk`'s baseline debt

Delete these two lines from `tests/reproducible_flake_baseline.txt`:

```
# sase-lk
tests/monitor/test_monitor_supervise.py::test_run_supervisor_escalates_term_ignoring_chatty_child
```

Measured before proposing this plan: that node has **zero** eligible evidence in the
gate's current window, so the entry is dead debt today. Running the gate against a
candidate file with those two lines removed
(`just selection-health --fail-on-new-flake --flake-baseline /tmp/candidate_baseline.txt`)
reports the same **2** exceeding nodes as the committed file —
`tests/fakey/test_usage_limit_e2e.py::test_usage_limit_failure_disables_only_fakey_and_preserves_error`
(epic `sase-n4`) and
`tests/test_config_cache.py::test_selector_change_eventually_invalidates_merged_config`
(bead `sase-mv`, phase `configcache`). Re-run that check on the real file after editing
it.

Do **not** add any `# fixed-at:` directive in this phase. The file's convention requires
the preceding comment to name the commit that fixed the node, and a commit hash cannot
be named from the tree that creates it — this is why `f9ab15d9c` deferred its own
directive to `cf7eeee03`. Report the entries for the land agent instead (see "Notes for
the land agent").

Do **not** touch the `# sase-j7` entry for
`test_run_supervisor_kills_the_whole_process_group_on_timeout`, even though Step 3 fixes
that node too. The parent epic plan says to leave it alone and to say explicitly if the
same fix covers it; the bead note is where that gets said.

## Verification

Run in this order, from the workspace checkout.

1. `just install` (ephemeral workspaces may hold stale dependencies).
2. Serial: `.venv/bin/pytest -q -n 0 tests/monitor/test_monitor_supervise.py` — 21
   passed. Record the new per-node call durations for the note.
3. Prove the guard still catches a hang, which is the one thing this change could
   silently destroy. Temporarily, in the working tree only, give one converted node an
   override that stalls the supervisor's own poll loop — for example
   `overrides={"_POLL_SECONDS": 60.0}` on
   `test_run_supervisor_completes_when_grandchild_holds_stdout` — and confirm the node
   fails with the helper's "did not exit within 15s" message rather than passing or
   erroring elsewhere. Revert that edit immediately; it must not be committed.
4. Contention:
   `SASE_CONTENTION_REPEAT=3 just test-contention tests/monitor/test_monitor_supervise.py`,
   green with 0 failed nodes. First check `pgrep -fa "run_pytest contention"` and, if a
   sibling phase's soak is already pinned to the default cores, pass
   `SASE_CONTENTION_CPUS` for a free pair (this plan's measurements used `2,3` because
   phase `forksafe` was soaking `0,1`).
5. Starvation repeat, to confirm the conversion did not make the nodes slower than the
   new deadline under load: with ~12 CPU burners pinned to two cores, run
   `taskset -c <those cores> .venv/bin/pytest -q -n 12 --durations=0 -k "escalates_term or partial_line or grandchild_holds_stdout or child_closes_stdio or continuous_output or process_group_on_timeout" tests/monitor/test_monitor_supervise.py`.
   Green, and no call duration close to 15s. Kill the burners afterwards.
6. `just selection-health --fail-on-new-flake`: exactly the two exceeding nodes named in
   Step 5, with no `tests/monitor/` node among them, and no new "retired nothing in the
   current window" line.
7. `just check`. Inspect the scoped selection summary; the changed test file must be
   selected, and any escalation must be explained in the note.
8. `just check-full` through `/sase_monitor` only, never inline
   (`sase monitor start --command 'just check-full' --next '<follow-up action>'`). Its
   `just test-cost` stage is what confirms the six extra child interpreters (~0.62s
   each, measured, so ~+6s serial including Step 4) stay inside the suite-wide cost
   budgets, and its full-lane record is the durable evidence that these nodes pass a
   real 13/14 worker lane. The pre-existing suite-cost failure tracked by `sase-j0` is
   expected and is not this phase's to fix.
9. Re-read the diff for any weakened assertion before closing.

## Risks and what to do if a node fails again

- A converted node failing with `supervisor subprocess did not exit within 15s` is a
  **real** supervisor hang. Fix production code; do not raise the deadline.
- A `times_out_after_partial_line` failure reading `assert '' == 'partial'` means the
  member's spawn budget lost to host pressure even at 3.0s. The remedy is that node's
  own `timeout_seconds`, not `_NO_HANG_TIMEOUT`.
- A failure of any of these nodes on an assertion about `monitor_state`,
  `monitor_timeout_kind`, or log contents is a genuine behavior regression and belongs
  in `src/sase/monitor/`.
- If host pressure ever defeats the child-process guard too, the next escalation is to
  stop asserting elapsed time at all and rely on the child's own recorded
  `elapsed_seconds` in `agent_meta.json` as a diagnostic only. Record that as a proposed
  follow-up rather than doing it here.

## Out of scope

- `src/sase/logs/pipe.py` and `src/sase/monitor/supervise.py`. Item 6 of the evidence
  explains why nothing there is implicated.
- The `# sase-j7` baseline entry, and epic `sase-j7`'s process-global leak class.
- `tests/monitor/test_monitor_start.py::test_start_monitor_promotes_a_bare_lane_and_runs_to_completion`
  (bead `sase-lf`).
- The two nodes that hold the flake gate red: the `sase-n4` fakey usage-limit node and
  the `sase-mv` config-cache node (phase `configcache` owns the second).
- Creating beads. Discovered work goes on `sase-ns.6.6.6.3` as
  `PROPOSED FOLLOW-UP: <one-line summary — detail>`.

## Notes for the land agent

Carry these into the epic's landing:

1. **`# fixed-at:` directives to declare**, naming this phase's fix commit, following
   the `cf7eeee03` precedent. Each retires only pre-fix evidence for its node; each has
   at least one eligible record to retire, so none will be reported as dead:
   - `2026-08-16T16:59:34Z tests/monitor/test_monitor_supervise.py::test_run_supervisor_completes_when_grandchild_holds_stdout`
   - `2026-08-17T01:20:25Z tests/monitor/test_monitor_supervise.py::test_run_supervisor_times_out_after_partial_line`
   - `2026-08-16T16:59:34Z tests/monitor/test_monitor_supervise.py::test_run_supervisor_times_out_after_child_closes_stdio`

   Use the fix commit's own timestamp as each directive's instant (it postdates all
   three records). Do **not** declare one for `escalates_term_ignoring_chatty_child`,
   `times_out_continuous_output`, or `kills_the_whole_process_group_on_timeout`: they
   have no eligible evidence left, so the gate would report those directives as retiring
   nothing.

2. **`sase-j7`'s node is now fixed by the same change.** The `# sase-j7` entry for
   `kills_the_whole_process_group_on_timeout` is dead debt (zero eligible evidence
   today) and was deliberately left in place for `sase-j7` to remove.
3. This phase does not close `sase-lk`; the phase bead note records what was verified so
   the epic's landing can close the task bead on evidence.
