---
tier: tale
title: Resolve %wait dependencies across monitor family members
goal:
  A lane that has run a monitor resolves %wait normally again, and a monitor whose
  supervisor dies is reaped instead of blocking its lane forever.
size: medium
proposed_by: bbugyi200.athena.z8
create_time: 2026-09-09 20:00:13
status: wip
---

# Monitor members permanently block `%wait` on their lane

## Symptom

Agent `sase-kp.land.w1` was launched with `%w:sase-kp.land` and stayed `WAITING`
indefinitely even though the `sase-kp.land` lane renders as `TALE DONE` in `sase ace`. A
second waiter on the same lane, `sase-kp.land.w0`, is stuck the same way.

## Root cause (confirmed)

`sase monitor`'s terminal marker writes `{"outcome": "monitored", ...}`
(`src/sase/monitor/supervise.py:232`, `src/sase/monitor/store.py:156`,
`src/sase/monitor/start.py:305`). The string `"monitored"` was never added to the
wait-dependency outcome vocabulary in `src/sase/core/dismissed_agent_completion.py`:

```python
SUCCESS_OUTCOME = "completed"
WAIT_SUCCESS_OUTCOMES = frozenset({"completed", "noop", "epic_approved", "plan_committed"})
FAILURE_OUTCOMES = frozenset({"failed", "killed", "stopped", "epic_launch_failed"})
IDENTITY_SUCCESS_OUTCOMES = WAIT_SUCCESS_OUTCOMES | frozenset({"plan_rejected"})
KNOWN_DONE_OUTCOMES = WAIT_SUCCESS_OUTCOMES | FAILURE_OUTCOMES | frozenset({"plan_rejected"})
```

So a monitor member that has finished lands in permanent limbo — it has a done marker,
but it is classified as neither resolved, done, failed, nor identity-success:

1. `artifact_is_resolved()`
   (`src/sase/core/wait_dependency_resolution/_artifact_state.py:66`) returns
   `outcome in WAIT_SUCCESS_OUTCOMES` → `False`.
2. `WaitDependencyIndex._add_prepared()`
   (`src/sase/core/wait_dependency_resolution/_index.py:153`) indexes the monitor member
   into `families[<lane>]` and `workflows[<lane>]` exactly like any other family member,
   because `create_monitor_member` writes `agent_family`, `workflow_name`, and
   `parent_timestamp` into its `agent_meta.json`.
3. `family_candidate()`
   (`src/sase/core/wait_dependency_resolution/_index_queries.py:282`) computes
   `is_resolved = all(member.is_resolved)` → `False`, forever.
4. `is_resolved()` (`_index_queries.py:402`) picks the **newest-timestamp** candidate
   among clan/family/workflow/named. The monitor member is newer than the plan-chain
   root, so the unresolved family entity beats the resolved workflow entity → `False`.
5. `sase_chop_wait_checks` therefore never writes `ready.json`, and every waiter on that
   lane blocks forever.

### Evidence

Against the live artifact store, **every** lane that has ever started a monitor is
permanently un-waitable, including lanes whose work is entirely terminal:

```
z2              is_resolved=False   (z2--plan completed, z2--code completed,
                                     z2--mon outcome='monitored', z2--1 completed)
smoke-sleep     is_resolved=False
smoke-fail      is_resolved=False
smoke-timeout   is_resolved=False
smoke-stop      is_resolved=False
sase-kp.land    is_resolved=False
```

Reproduce with the venv python from a directory outside the workspace root (the
workspace root has a `sase/` memory directory that shadows the package for a bare
`python3`):

```python
import sase.memory, sase.project_aliases  # avoid a circular-import ordering trap
from sase.core.wait_dependency_resolution import WaitDependencyIndex

idx = WaitDependencyIndex.build("gh_sase-org__sase")
print(idx.is_resolved("z2"))            # False, but every z2 member is terminal
print(idx.family_candidate("z2"))       # is_resolved=False, is_done=True
for c in sorted(idx.families["z2"], key=lambda c: c.timestamp):
    print(c.timestamp, c.name, c.is_resolved, repr(c.outcome))
```

This is **not** specific to `sase-kp.land`: starting a single monitor permanently
poisons `%wait` for that lane.

## Secondary defect: a dead monitor supervisor is never reaped

`sase-kp.land--mon` (artifact `.../ace-run/202608/13/20260813083112`) still reports
`monitor_state: "running"` with supervisor pid 3608804 long dead. It never created
`live_reply.md` — `OutputCapture.__init__` opens that file before the monitored command
is spawned (`src/sase/monitor/output.py:36`, `src/sase/monitor/supervise.py:98-99`), so
the supervisor died before reaching that line and left no `done.json`.

`store._reconcile_dead_supervisor()` (`src/sase/monitor/store.py:134`) already knows how
to settle exactly this state, but it only runs from `stop_monitor()`, i.e. when a human
runs `sase monitor stop`. Nothing periodic reaps it. Meanwhile
`cleanup_stale_running_entries()` (`src/sase/ace/scheduler/stale_running_cleanup.py:29`)
_did_ release the monitor's workspace claim once the pid died, so the workspace was
handed to another agent while a nominally-running monitor still owned it.

Consequences of an unreaped monitor: the lane can never start another monitor
(`MonitorAlreadyRunningError` from `active_monitor_for_lane`), its `--next` follow-up
never launches, `sase monitor list` shows a phantom active monitor forever, and — even
after the outcome-vocabulary fix above — the lane's family stays unresolved.

The supervisor is spawned with `stdout=stderr=subprocess.DEVNULL`
(`src/sase/monitor/start.py:151-167`), so there is no record of why it died.

## Intended behavior after this change

- A monitor member that finished successfully (`monitor_state` `completed` or `stopped`)
  counts as a resolved family member, so `%wait <lane>` releases.
- A monitor member that failed (`monitor_state` `failed` or `timeout`) counts as a
  _failure_, which keeps blocking the waiter — the same contract as a failed agent
  member — but `sase_chop_wait_checks` now reports it through the existing "Terminal
  dependency still blocks waiter" path instead of the "Unknown done outcome blocks
  waiter" alarm.
- A monitor member that is genuinely still running keeps blocking the waiter. That is
  correct: the lane is busy, holds a workspace claim, and may still launch a `--next`
  follow-up agent. Do **not** "fix" this by excluding monitor members from the family
  aggregate.
- A monitor whose supervisor died without writing a terminal marker is reaped
  automatically into `monitor_state: "failed"` with a `done.json`, so it stops blocking
  indefinitely and stops occupying its lane's monitor slot.

`stopped` maps to success rather than failure on purpose: `monitor_state_bucket`
(`src/sase/monitor_state.py:5`) already buckets it as `Done`,
`_release_claim_and_notify` reports it as success, and a stopped monitor deliberately
skips its `--next` follow-up (`src/sase/monitor/supervise.py:251`), so nothing further
will happen in that lane and the waiter should proceed.

## Implementation

### 1. Classify the `monitored` outcome in the wait vocabulary

In `src/sase/core/dismissed_agent_completion.py`:

- Add a named constant for the monitor outcome (`"monitored"`) and for the set of
  monitor states that count as success (`{"completed", "stopped"}`).
- Add `"monitored"` to `KNOWN_DONE_OUTCOMES` so the wait chop's unknown-outcome alarm no
  longer fires for monitors.
- Export a helper that maps a whole done-marker mapping to its effective wait outcome,
  e.g. `effective_done_outcome(done_data)`: when `outcome == "monitored"`, return
  `SUCCESS_OUTCOME` for a success `monitor_state` and `"failed"` otherwise (including a
  missing or unrecognized `monitor_state` — fail closed); otherwise return the raw
  outcome unchanged.

The mapping must key off `monitor_state`, not `status_label`: the stop label is
user-configurable through `sase monitor start --stop-status`.

### 2. Route wait-dependency classification through the helper

In `src/sase/core/wait_dependency_resolution/_artifact_state.py`, make
`done_outcome_from_data()` (and therefore `artifact_is_resolved`,
`artifact_succeeded_for_identity`, and `artifact_failed_for_identity`) use the new
helper. Keep the _raw_ outcome on `ArtifactCandidate.outcome` if that is what the
blocker logging in `sase_chop_wait_checks` should print — decide one way and make the
`_terminal_blockers` log line read sensibly for a failed monitor either way.

Verify the change flows into all four consumers: `WaitDependencyIndex._add_prepared`,
`family_candidate`, `clan_candidate`, and `terminal_blocking_artifacts_for_name`.

### 3. Handle the dismissed-archive fallback path

`_archived_outcome_from_status()` in `src/sase/core/dismissed_agent_completion.py`
returns `None` for a dismissed monitor member's `MONITORED` status, which drops the row
and leaves the family unresolved. Map the monitor stop status to the same effective
outcome as step 1 where the archived summary carries enough information; where it does
not (custom `--stop-status`), keep the current fail-closed behavior rather than
guessing.

### 4. Reap dead monitor supervisors periodically

Add automatic reconciliation so a monitor whose supervisor pid is gone and whose
`done.json` is missing gets settled into `monitor_state: "failed"` with a terminal
marker and an explicit error string naming the dead supervisor.

Reuse `store._reconcile_dead_supervisor()` rather than duplicating its marker-writing
logic; promote it to a public helper if that reads better. Prefer extending the existing
`sase_chop_stale_running_cleanup` chop (it already walks claims and calls
`is_process_running`) over adding a new chop, unless the monitor scan does not fit its
per-project-file iteration — in that case add a small `monitor_checks` builtin chop
following the shape of `src/sase/scripts/sase_chop_wait_checks.py`. Follow
`sase/memory/cli_rules.md` if a new subcommand or option is introduced.

Guard against reaping a monitor whose supervisor is merely slow to start: only reap when
the member's `pid` field is present, that pid is not running, and no `done.json` exists.

### 5. Stop discarding supervisor diagnostics

Change the supervisor spawn in `src/sase/monitor/start.py` so the child's `stdout`/
`stderr` land in a file inside the monitor member's artifacts directory (for example
`supervisor.log`) instead of `subprocess.DEVNULL`. A supervisor that dies before its
first captured output must leave a traceback behind. Keep `start_new_session=True`,
`stdin=DEVNULL`, and `close_fds=True` unchanged.

## Tests

Add or extend, at minimum:

- `tests/monitor/` (or a new focused module): a monitor member with `done.json`
  `{"outcome": "monitored", "monitor_state": "completed"}` makes
  `WaitDependencyIndex.is_resolved("<lane>")` return `True`; `"stopped"` also resolves;
  `"failed"` and `"timeout"` do not resolve and _are_ reported by
  `terminal_blocking_artifacts_for_name`; a missing `monitor_state` does not resolve.
- A regression test reproducing the reported shape end to end: a lane with
  `<lane>--plan` (completed) → `<lane>--code` (completed) → `<lane>--mon`
  (monitored/completed), plus a waiting agent whose `waiting.json` has
  `waiting_for: ["<lane>"]`; running the `wait_checks` chop writes `ready.json`.
  `tests/_axe_chop_wait_checks_helpers.py` and the existing
  `tests/test_axe_chop_wait_checks_plan_families.py` show the fixture pattern.
- A running monitor (no `done.json`) still blocks the waiter.
- The reaper: a monitor member with `monitor_state: "running"`, a dead pid, and no
  `done.json` is settled to `failed` with a `done.json`; a monitor with a live pid is
  left alone; a monitor that already has a `done.json` is left alone.
- `sase monitor start` writes supervisor stdout/stderr to the member's log file.

Check `tests/monitor/_fixtures.py` for reusable monitor-member builders before writing
new ones.

## Verification

- `just check` after each step; `just check-full` through `/sase_monitor` before landing
  (it routinely outruns a single agent turn — never run it inline).
- Run `just install` first: these workspace directories are ephemeral and dependencies
  may be stale.
- After the reaper lands, confirm the live phantom clears: `sase monitor list` should no
  longer show `h8bkkdxmxvzm` (`sase-kp.land--mon`) as active. If the reaper has not run
  yet, `sase monitor stop h8bkkd` settles it through the existing
  `_reconcile_dead_supervisor` path.
- Sanity-check the real store after the fix (read-only):
  `WaitDependencyIndex.build("gh_sase-org__sase").is_resolved("z2")` should become
  `True`, and the two stuck waiters (`sase-kp.land.w0`, artifact
  `.../202608/13/20260813082737`, and `sase-kp.land.w1`) should be releasable.

## Out of scope

Leave these alone; record them as follow-ups via `/sase_new_task` if they still matter
after this change:

- **Why the supervisor died.** Step 5 makes the next occurrence diagnosable; this plan
  does not chase the original cause, because the discarded stderr left no evidence.
- **Lane status display.** `sase ace` renders the lane as `TALE DONE` while a monitor
  member is running, because the family status policy in
  `src/sase/ace/tui/models/_agent_status_family_policy.py` ignores monitor members. That
  mismatch is what made the reported symptom look like a contradiction, but it is a
  presentation change with its own blast radius.
- **Surfacing the wait blocker in the TUI.** `sase_chop_wait_checks` already logs why a
  waiter is blocked; the agent detail panel's `Wait:` line does not show it.
- **The Rust core.** Wait-dependency resolution has no `sase_core` counterpart —
  `crates/sase_core` only carries `monitor_state` on `DoneMarkerWire` for the agent-scan
  wire. This fix stays in Python and does not cross the core backend boundary.
