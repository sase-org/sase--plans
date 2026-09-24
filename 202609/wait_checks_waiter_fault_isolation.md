---
tier: tale
title: Isolate wait_checks waiter failures and deploy the wait-index rename fix
goal:
  Stuck waiters such as 0r2--1 are released. A future exception while resolving one
  waiter parks only that waiter, is logged, and still marks the tick check_error; it no
  longer aborts every wait_checks release or fails silently in the runner fallback.
size: small
proposed_by: bbugyi200.athena.0r7
create_time: 2026-09-24 15:16:33
status: wip
---

# Plan: Stop one bad waiter from stalling every `%wait` release

## Diagnosis: the suspicion is confirmed, with a twist

The `0r2` family's newest shell, `0r2--1`, is the monitor follow-up launched when
`0r2--mon` failed. It waits on its fork source, `0r2--code`, which finished successfully
at about 14:15. Since then, every `wait_checks` tick has crashed before it reached
`0r2--1`. `~/.sase/axe/logs/lumberjack-waits.log` shows this from 14:22:53 to now:

```
File ".../wait_dependency_resolution/_index_queries.py", line 491, in add_candidate_members
    family_name = candidate.family_name or candidate.name
AttributeError: 'ArtifactCandidate' object has no attribute 'family_name'
Error in wait_checks: exit code 1
```

Causal chain:

1. `7e1b05964` (the agent-session wire cutover, 13:23) renamed the wait-index
   vocabulary: `WaitDependencyIndex.families` became `agent_sessions`, and
   `ArtifactCandidate.family_name` became `agent_session_name`.
2. `9bd351b67` (the recent fix, "confirm release membership before family handoff",
   13:36) was written against the pre-cutover tree. It added `dependency_member_dirs()`,
   which still used `self.families` and `candidate.family_name`. It landed on top of the
   cutover without a textual conflict, so nobody re-checked it. Master Gate did fail on
   it: mypy reported `attr-defined` and about a dozen wait tests failed with
   `AttributeError`. But Master Gate was already red on the commits before it, so the
   new failure was masked.
3. The dev update at 13:47 deployed `fbee04cd1`, which contains `9bd351b67`, to the host
   install. The first crash was at 14:03, on the `families` name path. `0r2--1` started
   waiting at 14:20:39, and every tick since 14:22 has crashed on the `family_name`
   fork-source path.
4. Two design gaps made one typo stop every release:
   - In `sase_chop_wait_checks._run`, the new `dependency_member_dirs(...)` call and the
     `confirmation_projects` scan sit outside the per-waiter `try`. Any exception there
     aborts the whole chop tick, for every waiter in every project, not just this one.
   - The runner's own fallback poll (`initial_dependencies_resolved`, called every 60s
     by `run_agent_wait.py`) exists "so chop outages cannot strand the runner forever".
     It runs the same confirmation code and turns any exception into a silent
     `return False`. That fallback was disabled by the same bug, with nothing logged.
5. `f2790e0e5` (14:52) already fixed the attribute names on master. But the host install
   was last updated at 14:54 to `742c1df38`, which predates `f2790e0e5`, so the deployed
   code is still broken.

Evidence that master is correct: `wait_checks`' resolve-then-confirm logic was replayed
read-only with the current master code against the live `0r2--1` marker. It resolves,
confirms with no new members, and would write `ready.json`. The same replay over all
live waiters shows `0r2--1` and `research.2i.final` would be released now. Any
bead-gated epic phase whose beads close before the deploy will also hit the crash. All
71 tests in the wait-dependency suites pass on master.

So the recent fix did cause this. It was not a logic error in the confirmation design.
It was a stale-base rebase over a rename, deployed while CI was red. The fault
amplification in items 4a and 4b is what turned it into an outage for every wait.

## Step 1: immediate remediation (deploy master)

Check whether the host install still predates `f2790e0e5`. The TUI header version or the
latest `~/.sase/logs/dev_update.jsonl` entry shows the deployed version. If it does, run
`sase update -q`; approving this plan authorizes that. The user may have run it already,
in which case skip it. Then confirm the stall has cleared:

- `lumberjack-waits.log` shows new `wait_checks` summaries and no new `AttributeError`.
- `0r2--1` and `research.2i.final` leave WAITING. Their runners pick up `ready.json`
  within a few seconds of it being written.

Do not hand-write `ready.json` files.

## Step 2: isolate per-waiter failures in `wait_checks`

In `src/sase/scripts/sase_chop_wait_checks.py` `_run`, make one waiter's exception cost
only that waiter:

- Wrap each waiter's resolution work in a guard. That covers
  `dependency_resolution_status`, `dependency_member_dirs`, the `confirmation_projects`
  scan, the `fresh_index` setup, and the terminal-blocker bookkeeping. The simplest
  shape is a helper that processes one `_WaitingMarker`, called inside
  `try/except Exception` in the loop. Keep the existing inner handling of confirmation
  failures ("a failed confirmation must park").
- On exception, park the waiter: never write `ready.json`. Increment a new
  `waiter_errors` summary counter and log
  `[wait_checks] Waiter <artifact dir> (<cl_name>) failed: <exc>`. Include the traceback
  for the first few failures per tick, capped like `_MAX_TERMINAL_BLOCKER_LOGS`, then
  `continue` to the next waiter.
- After the loop, emit the summary as today and add `waiter_errors` to the fields. Then,
  if `waiter_errors > 0`, set `result.status = "check_error"`, as
  `sase_chop_gate_shell_reclaim.py` does. The lumberjack still records the tick as
  failed and the axe error digest still fires, but healthy waiters in the same tick are
  released first.
- Update the `wait_checks` paragraph in `docs/axe.md`, next to the
  `deferred_unconfirmed` text that `9bd351b67` added. State that a waiter whose
  resolution raises stays parked, is counted in `waiter_errors`, and marks the tick
  `check_error` without blocking other waiters.

## Step 3: make the runner fallback's failures visible

In `src/sase/axe/run_agent_wait_deps.py` `initial_dependencies_resolved`, keep returning
`False` on exceptions, because parking is the conservative choice. But stop swallowing
them silently. For both the index build and the `confirm_dependency_resolution` call,
print a one-line warning with the exception type and message to the runner's stdout,
which is the agent's output log. For example:
`Wait dependency check failed (confirmation): AttributeError: ...; staying parked`. The
fallback runs at most once per 60s, so no extra rate limiting is needed.

## Step 4: tests

Add them next to the existing wait-check tests, using the helpers in
`tests/_axe_chop_wait_checks_helpers.py` and `tests/_agent_names_fixtures.py`:

1. **Fault isolation** (new test in
   `tests/test_axe_chop_wait_checks_artifact_identities.py`, or a small new
   `tests/test_axe_chop_wait_checks_fault_isolation.py`):
   - Create two waiters on completed dependencies.
   - Monkeypatch `WaitDependencyIndex.dependency_member_dirs` (or
     `dependency_resolution_status` as imported by the chop module) to raise only for
     the first waiter's `self_artifact_dir`.
   - Assert the healthy waiter gets `ready.json`, the failing one does not, the log
     contains `waiter_errors=1`, and the chop's structured result status is
     `check_error`. Write a result file through the chop context if the helper supports
     it; otherwise assert on the returned builder by calling `_run` with a fake runtime.
2. **Production-shape regression**:
   - Build a session with `--plan` done, `--code` done, `--mon` failed with
     `monitor_followup_agent="<name>--1"` and `monitor_followup_outcome="launched"`, and
     a live `--1` waiter. Reuse `_update_meta` and `_write_monitor_done` from
     `tests/_monitor_wait_dependency_helpers.py`.
   - Give the waiter's marker `waiting_for=["<name>--code"]` plus a `kind: "agent"`
     `wait_for_fork_sources` entry pointing at the `--code` artifact dir. This is
     exactly the `0r2--1` marker.
   - Assert `run_wait_checks` writes `ready.json`. This pins the `add_candidate_members`
     fork-source path that crashed in production.
3. **Runner fallback visibility** (in `tests/test_run_agent_wait_deps.py`):
   - Monkeypatch `confirm_dependency_resolution` in `sase.axe.run_agent_wait_deps` to
     raise.
   - Assert `initial_dependencies_resolved(...)` returns `False` and `capsys` captures
     the warning line.

## Verification

- Run the targeted suites directly first: `tests/test_axe_chop_wait_checks*.py`,
  `tests/test_wait_dependency_release_confirmation.py`, and
  `tests/test_run_agent_wait_deps.py`.
- Then run `just check` through `sase tool run check`, following the project's lint/test
  memory.
- Do not run `just check-full`.

## Out of scope: follow-up task beads to propose via `/sase_new_task`

- Landing re-verification: the host-owned finalizer rebased `9bd351b67` onto a master
  that had moved under it (the rename in `7e1b05964`) without re-running at least mypy.
  A semantic conflict with no textual conflict should not land unchecked.
- Master Gate has been red on every master push since at least `2bdd70c2d`. That hid
  this regression, and dev update deploys master HEAD regardless of gate status.
  Consider having `sase update` warn when the target SHA's Master Gate newly fails
  lint/mypy.
