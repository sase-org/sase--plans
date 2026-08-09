---
tier: tale
title: Resolve %wait on every successful terminal agent outcome
goal:
  A %wait naming an agent that finished successfully resolves for every terminal
  done.json outcome, not just "completed", so an epic-approved, noop, or plan-committed
  dependency can no longer park a waiting agent forever.
proposed_by: bbugyi200.athena.wi
create_time: 2026-08-09 09:21:01
status: wip
---

# Plan: Fix `%wait` stalls on terminal-but-not-`completed` agent outcomes

## Problem

A waiting agent (`wb.f0`, renamed `wb.f1` by the manual unwait) was parked on `%wait`
for two agents that had both already finished. The `wait_checks` chop never wrote its
`ready.json`, so the agent had to be released by hand with ACE's run-now/unwait action.

### Evidence

The waiter's `agent_meta.json`
(`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/09/20260809074339`)
recorded `"wait_for": ["sase-i1.land", "wb"]`.

Rebuilding the wait index against the real on-disk artifacts reproduces the stall
exactly:

| dependency     | index verdict | why                                                           |
| -------------- | ------------- | ------------------------------------------------------------- |
| `wb`           | resolved      | family aggregate of `wb--plan` + `wb--code`, both `completed` |
| `sase-i1.land` | **waiting**   | `done.json` exists with `"outcome": "epic_approved"`          |

`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/09/20260809074248/done.json`
holds `"outcome": "epic_approved"`: that epic-clan `land` member proposed an epic, the
user approved it, and the host launched the follow-on `sase-i1.4` clan. The agent was
finished and successful — everywhere in SASE except the wait index.

The chop itself was healthy and running on schedule; it just kept reporting the waiter
as unresolved (`~/.sase/axe/lumberjacks/waits/chops/wait_checks/runs/*.result.json`):

```
wait_checks: projects=23 artifacts=6784 waiting=505 ready_written=0
             already_ready=65 invalid=38 unresolved=402
             reason=dependencies_not_ready
```

### Root cause

`src/sase/core/wait_dependency_resolution` recognizes exactly one successful terminal
outcome, `SUCCESS_OUTCOME = "completed"`:

- `_artifact_state.artifact_is_resolved()` returns `outcome == SUCCESS_OUTCOME`.
- `_index.WaitDependencyIndex._add_prepared()` sets
  `is_done = outcome == SUCCESS_OUTCOME`.
- `dismissed_agent_completion.ArchivedAgentCompletion.is_resolved` / `.is_done` compare
  against the same single value.
- `IDENTITY_SUCCESS_OUTCOMES = {"completed", "plan_rejected"}` and
  `FAILURE_OUTCOMES = {"failed", "killed", "stopped"}`.

`done.json` markers, however, carry a wider outcome vocabulary. Every writer path
funnels through `build_done_marker` in `src/sase/axe/run_agent_exec_finalize.py`, which
writes `state.loop_outcome` verbatim (mapping only `failed_retried` to `failed`, and
`completed` to `noop` when the workflow launched zero agents):

| outcome              | meaning                                           | in wait index today                               |
| -------------------- | ------------------------------------------------- | ------------------------------------------------- |
| `completed`          | ordinary success                                  | resolves                                          |
| `noop`               | workflow succeeded, launched zero agents          | **neither success nor failure**                   |
| `epic_approved`      | planner's epic was approved; host owns the launch | **neither success nor failure**                   |
| `plan_committed`     | plan approved and committed, no coder follow-up   | **neither success nor failure**                   |
| `plan_rejected`      | plan rejected by the user                         | identity-terminal, not wait success (intentional) |
| `failed`             | failure                                           | failure                                           |
| `killed`             | failure                                           | failure                                           |
| `stopped`            | failure / queue cancellation                      | failure                                           |
| `epic_launch_failed` | epic launch failed                                | **neither success nor failure**                   |

An outcome in neither bucket is a permanent stall: `is_done` stays `False` so the wait
never resolves, and `is_failed` stays `False` so the identity-wait failure fallback
never fires either. Both resolution paths share this code — the `wait_checks` chop
(`src/sase/scripts/sase_chop_wait_checks.py`) and the runner's own periodic fallback
(`src/sase/axe/run_agent_wait_deps.py`) — so the runner's chop-outage backstop cannot
rescue it. Manual unwait is the only escape.

The classification is also self-inconsistent. `_archived_outcome_from_status()` in
`src/sase/core/dismissed_agent_completion.py` maps the dismissed-archive display
statuses `PLAN COMMITTED` and `EPIC CREATED` onto `SUCCESS_OUTCOME`, so the _same_
epic-approved planner satisfies a `%wait` once it has been dismissed and archived, while
its live `done.json` does not. `EPIC APPROVED` has no mapping at all, so an archived row
in that state degrades to "no terminal record" instead of success.

Nothing else in the codebase agrees with the wait index. `epic_approved` is a success in
`src/sase/axe/chop_lifecycle.py`
(`_SUCCESS_DONE_OUTCOMES = {"completed", "epic_approved", "noop", "plan_committed", "plan_rejected"}`),
in `run_agent_runner_finalize.classify_exec_success()`, in
`run_agent_exec_finalize.finalize_loop()`'s `AgentExecResult.success`, and in
`run_agent_runner_lifecycle._NON_HOLD_FAILURE_OUTCOMES`. The TUI renders it as the
terminal `EPIC APPROVED` / `EPIC CREATED` status. Only wait resolution treats it as
"still pending, forever".

There are no tests covering `epic_approved`, `noop`, `plan_committed`, or
`epic_launch_failed` in any wait-resolution suite — the gap is untested, not a pinned
decision. The one deliberate exclusion, `plan_rejected`, _is_ pinned, by
`tests/test_dismissed_agent_completion.py::test_plan_rejected_archive_is_identity_terminal_but_not_wait_success`,
and must keep its current behavior.

## Goals

1. A `%wait` naming an agent that terminated successfully resolves, for every successful
   terminal outcome — not just `completed`.
2. Live `done.json` and dismissed-archive fallback classify an outcome identically.
3. A future outcome value cannot silently reintroduce a permanent stall.
4. When a wait does stall behind a finished dependency, the `wait_checks` chop log says
   so instead of only reporting an aggregate `unresolved` count.

## Non-goals

- Changing `plan_rejected` semantics. It stays identity-terminal but not a name-wait
  success.
- Touching `_completed_handoff_workflow_state()`'s
  `state_data.get("status") != SUCCESS_OUTCOME` check in `_artifact_state.py`. That
  reads `workflow_state.json`'s own status field, a different vocabulary that happens to
  share the string `"completed"`.
- The separate `SUCCESS_OUTCOME` in `src/sase/agent/names/_lookup_artifacts.py` used by
  agent-name lookup (`sase agent names`, chat lookup). It has the same shape of gap but
  a different blast radius; file a follow-up task bead rather than widening this change.
- The `wb.f0` -> `wb.f1` rename. That is the expected effect of releasing and
  relaunching a waiting agent, not part of this defect.

## Implementation

### 1. One source of truth for done-outcome classification

`src/sase/core/dismissed_agent_completion.py` already owns the outcome constants. Extend
it:

- Add
  `WAIT_SUCCESS_OUTCOMES = frozenset({"completed", "noop", "epic_approved", "plan_committed"})`
  — terminal outcomes meaning "this agent finished the work it was launched to do",
  which is what `%wait` asks about.
- Add `epic_launch_failed` to `FAILURE_OUTCOMES`.
- Redefine `IDENTITY_SUCCESS_OUTCOMES` as
  `WAIT_SUCCESS_OUTCOMES | frozenset({"plan_rejected"})`.
- Add
  `KNOWN_DONE_OUTCOMES = WAIT_SUCCESS_OUTCOMES | FAILURE_OUTCOMES | frozenset({"plan_rejected"})`
  for the fail-loud check in step 4.
- Leave `SUCCESS_OUTCOME = "completed"` alone and keep exporting it. It is the value
  _written_ for ordinary success and is read for other purposes; do not repurpose it.
- Export the new names from `__all__`.

Update `ArchivedAgentCompletion.is_resolved` and `.is_done` to test
`self.outcome in WAIT_SUCCESS_OUTCOMES`.

### 2. Consume the classification in wait resolution

- `src/sase/core/wait_dependency_resolution/_artifact_state.py`:
  `artifact_is_resolved()` returns `outcome in WAIT_SUCCESS_OUTCOMES` when
  `outcome is not None`.
- `src/sase/core/wait_dependency_resolution/_index.py`: `_add_prepared()` sets
  `is_done = outcome in WAIT_SUCCESS_OUTCOMES`.
- `src/sase/core/wait_dependency_resolution/_types.py`: re-export
  `WAIT_SUCCESS_OUTCOMES` and `KNOWN_DONE_OUTCOMES` alongside the existing constants so
  the package keeps a single import surface.

The chop and the runner fallback both go through `dependency_resolution_status()`, so no
change is needed in `src/sase/scripts/sase_chop_wait_checks.py` or
`src/sase/axe/run_agent_wait_deps.py` for the resolution fix itself.

### 3. Live/archive parity

In `src/sase/core/dismissed_agent_completion.py`, teach
`_archived_outcome_from_status()` about `EPIC APPROVED` (-> `"epic_approved"`). Audit
the rest of the mapping against `src/sase/agent/status_buckets.py` and
`src/sase/ace/tui/models/_loaders/_done_loaders.py` so every display status a terminal
row can carry round-trips to the outcome that produced it. Keep `EPIC CREATED` and
`PLAN COMMITTED` resolving as successes; after step 1 they may stay in
`_SUCCESS_STATUSES` or map to their specific outcomes, as long as the resulting
classification is unchanged.

### 4. Fail loud on an unrecognized outcome

An unknown outcome keeps today's conservative behavior — it does not resolve a wait —
but it must stop being invisible.

- In `src/sase/scripts/sase_chop_wait_checks.py`, when a waiter is unresolved, check
  whether any blocking dependency names an artifact that already has a terminal
  `done.json` whose outcome is not in `KNOWN_DONE_OUTCOMES`. Count those in a new
  `unknown_outcome` summary counter and `runtime.log()` the artifact dir plus the
  offending outcome.
- Add a unit test asserting that every outcome literal reachable by
  `build_done_marker`'s callers is a member of `KNOWN_DONE_OUTCOMES`, so adding a new
  outcome without classifying it fails CI rather than stranding an agent.

### 5. Make a genuine stall diagnosable

Today `dependency_resolution_status()` returns a bare `"waiting"`, and the chop reports
only `unresolved=402` — which is why this incident needed hand diagnosis.

- Extend `WaitDependencyStatus` in `src/sase/core/wait_dependency_resolution/_types.py`
  with `blocked_on: tuple[str, ...] = ()`, populated by `_resolution.py` with the
  dependency names, bead ids, or artifact identities that failed to resolve. `.resolved`
  keeps its current contract; existing callers are unaffected.
- In the chop, log a bounded, low-noise sample: only waiters whose blocking dependency
  has a **terminal `done.json`** (the dependency is finished yet the wait is still
  parked). Normal churn — dependencies that simply have not finished — stays unlogged,
  so this does not spam a run that legitimately sees hundreds of live waiters. Cap the
  sample (e.g. 10 lines per cycle) and note the count of any suppressed remainder.

### 6. Tests

New and updated coverage, following the existing
`tests/_axe_chop_wait_checks_helpers.py` fixtures:

- `tests/test_axe_chop_wait_checks.py` (or a focused new module): parametrize the
  dependency's `done.json` outcome. `completed`, `noop`, `epic_approved`, and
  `plan_committed` write `ready.json`; `failed`, `killed`, `stopped`,
  `epic_launch_failed`, and `plan_rejected` do not.
- A regression test named for this incident: a waiter parked on an epic-clan `land`
  member whose `done.json` is `epic_approved` resolves on the next chop cycle.
- Identity-dependency coverage: a `wait_for_artifacts` entry pointing at an
  `epic_approved` artifact resolves, and one pointing at `epic_launch_failed` is treated
  as failed (so the name fallback runs) rather than hanging.
- Runner-fallback coverage in the `run_agent_wait_deps` suite:
  `initial_dependencies_resolved()` and `waiting_marker_dependencies_resolved()` agree
  with the chop for the same outcomes.
- `tests/test_dismissed_agent_completion.py`: live-`done.json`-versus-archived parity
  for each outcome, including a new `EPIC APPROVED` archived row. Keep
  `test_plan_rejected_archive_is_identity_terminal_but_not_wait_success` passing
  unchanged.
- The `KNOWN_DONE_OUTCOMES` exhaustiveness test from step 4.

### 7. Docs

`docs/axe.md` currently states that `wait_checks` unblocks only on a `done.json` outcome
of `"completed"`. Rewrite that paragraph to describe the successful-terminal-outcome
set, call out that `plan_rejected` deliberately does not satisfy a name wait, and
mention the new `unknown_outcome` counter.

## Verification

```bash
just install
just check-full
```

Targeted while iterating:

```bash
.venv/bin/pytest tests/test_axe_chop_wait_checks.py \
                 tests/test_axe_chop_wait_checks_artifact_identities.py \
                 tests/test_axe_chop_wait_checks_plan_families.py \
                 tests/test_axe_chop_wait_checks_beads.py \
                 tests/test_dismissed_agent_completion.py -q
```

Confirm the original stall is fixed against the real artifacts. Before the fix this
prints `sase-i1.land -> False`; after it, `True`:

```bash
.venv/bin/python -c "
from sase.core.wait_dependency_resolution import WaitDependencyIndex
idx = WaitDependencyIndex.build('gh_sase-org__sase')
waiter = ('/home/bryan/.sase/projects/gh_sase-org__sase/artifacts/ace-run'
          '/202608/09/20260809074339')
for name in ('wb', 'sase-i1.land'):
    print(name, '->', idx.is_resolved(name, exclude_artifact_dir=waiter))
"
```

## Risks

- Widening what satisfies a `%wait` could release a dependent agent earlier than a user
  expected for `epic_approved`: the planner is done, but the epic it spawned is not.
  This is the correct reading — `%wait <name>` waits on that agent, and
  `%wait(bead=...)` or a wait on the epic's own clan is the tool for waiting on the
  spawned work. It also matches what every other subsystem, the TUI status, and the
  dismissed-archive path already believe. Call it out in the `docs/axe.md` update.
- `noop` rows are hidden in the TUI, so a resolved wait on one is invisible in the agent
  list. That is strictly better than parking forever, but worth a sentence in the docs.
- Adding `epic_launch_failed` to `FAILURE_OUTCOMES` changes identity-wait behavior from
  "hang" to "treated as failed, fall back to the name lookup". That is the intended
  repair; the new identity test pins it.
