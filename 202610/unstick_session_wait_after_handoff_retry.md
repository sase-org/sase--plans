---
tier: tale
title: Unstick session waits after a retried handoff turn
goal:
  Waiters on an agent session release once the session's handoff lane completes, even if
  an earlier turn was stamped failed. A retried attempt never inherits the previous
  attempt's failed finalizer verdict. sase-1oj.6 and sase-1oj.land then resume.
size: medium
proposed_by: bbugyi200.apollo.ak
proposed_with: claude/opus
create_time: 2026-10-27 07:52:46
status: wip
---

# Unstick session waits whose failed turn already handed off to a completed lane

## Problem

Epic `sase-1oj` ("Cut athena's SSD writes from SASE's own churn") is stuck. Its
measurement phase `sase-1oj.6` and its landing agent `sase-1oj.land` have been `WAITING`
since 2026-10-26 16:08 EDT. All their bead dependencies (`sase-1oj.3`, `.4`, `.5`) were
closed by 2026-10-27 03:43Z. Both waiters are parked on the **agent-session name**
`sase-1oj.4`, which can never resolve and never raises a terminal alert. A read-only
replay of `sase.axe.run_agent_wait_deps._resolve_marker_release` against `sase-1oj.6`'s
`waiting.json` returns `state='waiting', blocked_on=('sase-1oj.4',)`. A fleet-wide scan
found no other waiter in this state today.

### Evidence (session `sase-1oj.4`, artifacts under `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202610/26/`)

| member              | dir              | parent           | kind    | done outcome                                           |
| ------------------- | ---------------- | ---------------- | ------- | ------------------------------------------------------ |
| `sase-1oj.4--plan`  | `20261026160816` | —                | turn    | **failed** (`failure_facts.lifecycle_phase = handoff`) |
| `sase-1oj.4--mon`   | `20261026221539` | `20261026160816` | monitor | failed, `followup_outcome=launched` → `--1`            |
| `sase-1oj.4--1`     | `20261026231556` | `20261026160816` | turn    | completed                                              |
| `sase-1oj.4--mon-0` | `20261026232303` | `20261026231556` | monitor | completed, launched → `--2`                            |
| `sase-1oj.4--2`     | `20261026233958` | `20261026160816` | turn    | completed (closed bead `sase-1oj.4`)                   |

### Root cause, two layers

**1. A retried attempt inherits the previous attempt's failed finalizer verdict.**
Timeline of `sase-1oj.4--plan`:

- 01:00:46Z: attempt 1 submits its final declaration.
- 01:01:45Z: the commit finalizer's `stitch main` exits 2, so it starts a Codex
  `conflict-repair-main` model turn.
- 02:07:48Z: that turn dies with "Codex turn integrity failure: task completed with no
  final agent message...".
- 02:07:49Z: `sase.finalizers.controller_run` publishes a `failed` aggregate to
  `finalizer_result.json` and re-raises.
- 02:07:52Z: `handle_workflow_error` in `src/sase/axe/run_agent_exec_retry.py` treats
  the error as retryable. `snapshot_attempt` in
  `src/sase/axe/run_agent_exec_attempts.py` moves only `live_reply*` and `continuation/`
  into `attempts/01/`. The failed `finalizer_result.json` stays in the root artifacts
  dir.
- Attempt 2 reopens the bead, fixes the leftover tests, and hands off cleanly to a
  monitor (`--mon`, 02:15:39Z). The runner log ends
  `Agent completed with status: SUCCESS` / `shell-handoff:monitored`.
- 02:16:32Z: `src/sase/axe/run_agent_exec_finalize.py` (success branch, around line 302)
  checks `_finalizer_reports_failure()` for shell-handoff outcomes. That check reads
  **attempt 1's** stale `finalizer_result.json`, so it writes `done.json` with
  `outcome: failed`.

`deferred_commit_details()` (`src/sase/axe/runner_reporting.py`) and
`_load_aggregate_result()` (`src/sase/monitor/host_completion_state.py`) read the same
file, so a retried attempt can also inherit a stale deferral or verdict there.

**2. Session-wait resolution has a settled state that is neither resolved nor
terminal.** `WaitDependencyEntityQueries._agent_session_entity` (in
`src/sase/core/wait_dependency_resolution/_index_entities.py`) builds the effective
generation `{--plan, --1, --2}`. Monitors whose follow-up is present are already
filtered out by `_agent_session_members_after_shell_handoffs`. The entity is resolved
only when every effective member is resolved. `--plan` is failed, so the session **never
resolves**.

`_terminal_blocker_members` (in
`src/sase/core/wait_dependency_resolution/_index_queries.py`) counts only the newest
failed member. Here that is `--mon`, and it is skipped because its launched follow-up is
present. So `--plan` is **not** a terminal blocker either. The waiter stays `WAITING`
forever, with no stranded or terminal alert.

The failed turn did not end the session. Its own shell handoff (a monitor parented to
it) launched a follow-up that is present in the generation, and the lane continued and
completed. Today's supersession rules handle failed _monitors_ with launched follow-ups
and same-kind shell retries (`_is_superseded_terminal_turn_member`). They do not handle
an _agent turn_ whose failure was recorded at the moment it handed off.

## Changes

### A. Treat a failed agent turn as superseded by its own launched shell handoff

In `src/sase/core/wait_dependency_resolution/_index_entities.py`, extend the effective
generation filter used by `_agent_session_members_after_shell_handoffs`. Add a small
helper next to `_is_superseded_terminal_turn_member` that drops a member when **all** of
these hold:

- it is an agent turn (`turn_member_kind is None`);
- it has a done marker with a non-success outcome
  (`outcome not in WAIT_SUCCESS_OUTCOMES`);
- some shell member in the same candidate set (`turn_member_kind` in `monitor` / `gate`)
  meets all of the following:
  - its `parent_timestamp == member.timestamp`;
  - its `turn_followup_outcome in SUCCESSFUL_TURN_FOLLOWUP_OUTCOMES`;
  - its `turn_followup_agent` is in `names_in_generation`, including
    `extra_present_names`.

Requiring the shell member to be parented to the failed turn is what keeps the rule
narrow.

The follow-up lane then carries the session's verdict. If the follow-up and its
successors complete, the session resolves. If the newest follow-up fails,
`_terminal_blocker_members` still reports it, as today. The failed turn stays `failed`
in its own `done.json` and in ACE. Only dependency release stops treating it as live
evidence, which preserves the "a handoff must not mask a failed finalizer" visibility
goal of `_finalizer_reports_failure`.

`_agent_session_entity`, `agent_session_candidate`, `agent_session_candidate_for_root`,
and `terminal_blocking_artifacts_for_name` all consume `_agent_session_handoff_state`.
Confirm they pick up the new filter without other edits, and keep
`WAIT_SUCCESS_OUTCOMES` / `SUCCESSFUL_TURN_FOLLOWUP_OUTCOMES` imported from `._types`.
Update the docstrings of `_agent_session_members_after_shell_handoffs` and
`_terminal_blocker_members` so they describe the new supersession case.

Must keep passing unchanged:
`tests/test_axe_chop_wait_checks_plan_agent_sessions_handoffs.py::test_failed_or_killed_plan_chain_handoff_done_blocks_agent_session_dependency`.
In that test the failed `--2` has no monitor or gate child, so it must still block.

### B. Start every retry attempt with a clean finalizer verdict

In `snapshot_attempt` in `src/sase/axe/run_agent_exec_attempts.py`, also move these
attempt-scoped finalizer aggregate files into `attempts/<N>/`, next to `live_reply.md`
and `continuation/`:

- `finalizer_result.json`, the aggregate verdict, stored as
  `attempts/<N>/finalizer_result.json`;
- `finalizers/progress.jsonl`, the controller journal, stored as
  `attempts/<N>/finalizers/progress.jsonl`. This keeps the root run view
  (`sase.finalizers.run_view_inputs`) from pairing attempt N's finished-failed journal
  with attempt N+1's verdict.

Use the same tolerate-missing, move-into-tmp-then-rename pattern already in that
function. Leave the `finalizer_result.lock` file and the per-instance
`finalizers/<instance_id>/` artifacts in place, out of scope. Update the module and
function docstrings to list the newly archived files.

After this change, `_finalizer_reports_failure`, `deferred_commit_details`, and
`host_completion_state._load_aggregate_result` only ever see a verdict that the current
attempt published. A retried attempt that ends in a monitor or gate handoff without
running its own finalizer phase writes `outcome: completed`, as it would have without
the retry. Check `src/sase/finalizers/cli_status.py` and
`src/sase/finalizers/run_view_inputs.py`. If either one should surface archived attempt
verdicts, it may read them from `attempts/<N>/`, but no reader may fall back to an
archived verdict for the current outcome.

## Tests

Add focused tests, reusing `tests/_agent_names_fixtures.make_agent` and the helpers in
`tests/_monitor_wait_dependency_helpers.py` (`_update_meta`, `_write_monitor_done`), as
`tests/test_monitor_wait_dependency_supersession.py` does.

1. **Wait resolution (new file, e.g.
   `tests/test_wait_dependency_failed_handoff_turn_supersession.py`):**
   - Reproduce the `sase-1oj.4` shape exactly: failed `--plan` root, failed `--mon`
     (parent = root, follow-up launched → `--1`), completed `--1` (parent = root),
     completed `--mon-0` (parent = `--1`, launched → `--2`), completed `--2` (parent =
     root).
     - `dependency_resolution_status(index, ["lane"])` is `resolved`.
     - `terminal_blocking_artifacts_for_name("lane") == ()`.
     - A `run_wait_checks` round writes `ready.json` for a waiter, using the harness in
       `tests/test_axe_chop_wait_checks_plan_agent_sessions_handoffs.py`.
   - Same shape, but the newest follow-up `--2` failed: still `waiting`, and
     `terminal_blocking_artifacts_for_name` returns `--2`.
   - Failed turn with a launched monitor whose recorded follow-up agent is absent: stays
     `waiting` (handoff unproven).
   - Failed turn whose monitor child has a non-successful `turn_followup_outcome`: stays
     `waiting`.
   - Parametrize the supersession case over a `gate` shell member, with
     `tests/test_gate_wait_dependency_handoff.py` as the fixture reference.
   - A failed turn with a launched shell member parented to a _different_ turn: not
     superseded.
2. **Retry snapshot (`tests/axe/test_run_agent_exec_attempts.py`):**
   - `snapshot_attempt` moves `finalizer_result.json` and `finalizers/progress.jsonl`
     into `attempts/<N>/`, leaves `finalizers/<instance_id>/` alone, and tolerates
     either file being missing.
3. **Handoff done marker (near the existing
   `tests/test_axe_run_agent_exec_finalize_*.py` files):**
   - Simulate a failed-then-retried attempt: a failed `finalizer_result.json` that was
     snapshotted on retry, then a monitor-handoff loop outcome. The written `done.json`
     has `outcome: completed`.
   - Keep the existing guarantee: a failed `finalizer_result.json` published by the
     _current_ attempt before a shell handoff still yields `outcome: failed`.

## Verification

- Run the new and touched test files directly, then the repo's standard pre-finish check
  per the lint-and-test memory (`just check` through `sase tool run`).
- **Live, read-only acceptance check** (do not write any marker files): with the
  workspace's code, replay
  `sase.axe.run_agent_wait_deps._resolve_marker_release(json.load(open(<waiting.json>)), project_name="gh_sase-org__sase", artifacts_dir=<dir>)`
  for `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202610/26/20261026160817`
  (`sase-1oj.6`). The decision must be `releasable=True`. For `.../20261026160818`
  (`sase-1oj.land`), `blocked_on` must contain only `sase-1oj.6`. Record both results in
  the final response. If either waiter has already been released by the time you run
  this, say so instead.
- After the change lands, the `wait_checks` chop (a fresh `sase_job_wait_checks`
  subprocess every tick) picks it up and writes `ready.json` for `sase-1oj.6`. The
  parked runners need no restart. `sase-1oj.land` then releases after `sase-1oj.6`
  completes.

## Bookkeeping

- Add a note to epic bead `sase-1oj` saying that its two waiters were parked on the
  `sase-1oj.4` session. Give the stale-finalizer-verdict cause and name the fix.

## Out of scope

- The bead read-model `crate_version` ping-pong reported in `sase-1oj` note #7 is a
  separate sase-core cache-keying problem. `sase-1oj.6`'s own instructions already
  require it to re-measure and file follow-ups for missed targets. Do not change the
  read model here.
- Hand-editing `sase-1oj.4--plan/done.json`, or writing `ready.json` for the waiters.
  Change A releases them through the normal path. If the user wants them released
  sooner, that is their TUI "run now" action, not this tale's.
- Why the Codex conflict-repair turn itself lost its final message: that is a provider
  integrity failure, and the retry handled it correctly.
