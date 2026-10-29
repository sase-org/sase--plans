---
tier: tale
title: Recover bead waiters safely when they wake
goal:
  Complete sase-1pu.2 by proving and fixing wake-time name collisions while preserving
  one execution and the original launch context.
size: medium
proposed_by: bbugyi200.athena.sase-1pu.2
proposed_with: codex/gpt-6-astra
bead: sase-1pu.2
create_time: 2026-10-29 19:36:09
status: wip
---

- **PARENT:**
  [202610/task_backlog_epic_3.md](https://github.com/sase-org/sase--plans/blob/main/202610/task_backlog_epic_3.md)
- **BEAD:**
  [sase-1pu.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pu/sase-1pu.2.md)

# Recover bead waiters safely when they wake

## Assignment and scope

Implement phase `sase-1pu.2` of `plan:202610/task_backlog_epic_3.md`, addressing
`sase-12v`. Read the phase again before implementation, including any new notes. Its
parent decision `optional_scope = keep_five_seeds` is final. This plan authorizes the
waiter fix and necessary regression coverage; it authorizes no memory edits, new feature
work, task-bead creation, or unrelated monitor changes.

This is a medium tale because the remaining work is one bounded launch-lifecycle bug
with a shared reproduction harness. The phase had no previous implementation or
remaining-work notes when inspected. Do not create a child epic simply to split
investigation from its fix. If an existing fix satisfies the entire acceptance matrix,
demonstrate that with real-path regressions and fixing-commit evidence instead of adding
redundant retry machinery.

## Evidence and limits of the investigation

`sase-12v` reports that `sase-11l.6` waited 41 hours and failed with
`NameCollisionError` immediately after its dependency closed on September 18. Its
evidence reference `file:explicit:1eb45f0014c7519cc6731e74` currently resolves as
missing. The precise historical traceback has therefore not been recovered; do not
report a hypothesized cause as established fact.

Current code provides these concrete investigation points:

- `src/sase/axe/run_agent_runner.py` bootstraps and claims the name before waiting, then
  invokes `refresh_runner_code_after_wait` before launching the workflow.
- `run_agent_wait.py` and `run_agent_wait_markers.py` persist completion of the
  dependency barrier. A refreshed pass should not wait again.
- `run_agent_runner_refresh.py` re-executes the same runner with the same argv after a
  blocking wait if the source revision changed. It restores the submitted prompt and
  forwards the current name only after an ownership check. This is normally a
  same-process replay, not a new successor launch.
- `run_agent_runner_bootstrap.py` preserves metadata on a refreshed pass.
  `run_agent_directive_identity.py` then resolves and claims identity again. Its
  generated-name flag is consumed on the first pass; exact planned claims and ordinary
  claims take different registry paths. Compare both passes explicitly.
- `agent/names/_registry_batch.py` reads raw ownership for the planned-name handoff;
  `claim_exact_planned_registered_name` accepts planned reservations. The fallback in
  `_registry_agent_mutations.py` loads the registry and rejects other owners, including
  `collision_owners`. A rebuild can change the evidence seen between the first and
  second claim. Neither a stale reservation nor a rebuild defect has yet been proved to
  cause this report.

Related fixes have already landed: `a723858f27` preserves wait-agent names across
refresh and `1567269ca7` preserves clan launch handoffs. Preserve those fixes. The
following baseline command passed all 37 tests during planning:

```bash
.venv/bin/python -m pytest -q tests/test_run_agent_runner_refresh_handoff.py tests/test_refresh_preserves_launch_handoffs.py tests/test_agent_name_registry_claims.py
```

That is baseline evidence, not reproduction of the reported bug. The two-pass clan
helper in `test_refresh_preserves_launch_handoffs.py` mocks name claiming, and the
refresh handoff tests stop at the exec boundary.

`sase-1el` was reviewed separately. Its gate-create dispatch half was fixed; its
monitor-start collision remains undiagnosed. Monitor start currently passes through
`monitor/start_lane.py` and session promotion, unlike ordinary waiter re-exec. Change
that path only if the waiter reproduction proves the same faulty ownership operation;
otherwise leave it for the parent lander to triage.

## Implementation

### 1. Establish the failing transition with real name ownership

Add a focused test module and small shared fixture if needed. Use an isolated SASE home,
project, registry, bead/dependency fixtures, and workspace. Never use the live agent
registry or reopen a real dependency to reproduce this bug.

Drive the sequence through the actual bootstrap/directive claim functions: reserve a
bead-assigned name, perform the first extraction, park, satisfy the dependency, persist
the wait-completion stamp, simulate changed source identity, then perform the refreshed
extraction. Mock the provider/workflow execution and the exec boundary, but retain the
production registry reservation, claim, and rebuild code. Simulate 41 elapsed hours with
clocks/metadata; do not actually wait.

Exercise normal claimed ownership, registry rebuild during the wait, an obsolete
registry-only owner, and a genuinely different current owner. Include the generated-name
marker and consumed clan metadata used by bead launches. Compare the name, artifact
directory, reservation kind, collision owners, and explicit claim mode before and after
refresh. Use the first failing case to identify the actual incorrect transition, not a
test that merely mocks a random exception. Check plain bead waiters and session-attached
waiters separately; the attach payload loader is not itself a consuming loader, so do
not assume it was lost.

Record the reproduction and whether the historical report can be tied to it in a note on
`sase-1pu.2`. If the historical scenario is already fixed, identify the fix and still
close the acceptance gaps with real-registry replay coverage.

### 2. Make the proven wake transition idempotent and recoverable

Fix the narrowest responsible operation. A refresh of the same durable run must retain
its established identity rather than allocate a new name or treat it as a fresh explicit
launch. Preserve the existing no-new-imports constraint between the source-identity
comparison and `os.execv`; recovery that needs fresh imports belongs in the refreshed
process.

For a stale registry snapshot, re-read/rebuild and revalidate ownership under the
existing allocation lock before retrying the claim. Bound any retry added here to one
reconciliation attempt per wake replay. Never use `force_reuse`, delete another owner's
artifacts, suppress every `NameCollisionError`, or weaken fresh explicit-name collision
checks. A current foreign owner, unrelated live owner, session/clan container, or
mismatched artifact identity remains a refusal. Keep the original launch evidence intact
when refusing, with a diagnostic that names the conflicting owner and the exhausted
recovery attempt.

Do not introduce a second scheduler retry system or launch a successor merely to repeat
a failed in-process claim. Claim/reconciliation must complete before workflow execution
or workspace transfer. Repeated ready signals and repeated refresh processing must not
run the workflow twice. If the proven defect instead occurs in an existing successor
transaction, use that transaction's established idempotency/claim boundary and
demonstrate exactly one successful spawn.

Maintain the original name, artifact identity, dependency targets, bead and epic
metadata, project identity, workspace assignment, prompt/local macros, provider and
model, clan/session membership, and `wait_completed_at`. Do not release a workspace or
discard a wait marker as a side effect of a retryable claim failure.

### 3. Honor the Rust backend boundary

Python runner code may collect filesystem/process evidence, preserve replay metadata,
call existing APIs, and apply effects under its lock. New shared rules for whether
ownership can be reused, reconciled, or retried belong in `sase_core`, not a second
Python policy implementation.

If such a rule is necessary, open `sase-core` with `/sase_repo` and extend the existing
`agent_ownership` domain and its binding in `sase_core_py/agent_custody`. Use the
existing ownership wire types and planner where possible. Return a typed decision
derived from explicit ownership evidence, with tests for same owner, stale owner, and
genuine conflict; Python remains the effect adapter. Add binding round-trip coverage and
a thin host facade. Do not change the Rust API solely for a Python replay bug that can
be fixed by correctly calling the existing API.

A new binding requires the host-owned core commit to land before moving
`sase-core-revision.txt` to a revision containing it. Follow `docs/rust_backend.md` and
host finalization; do not fabricate a revision or manually commit. Arrange a mechanical
continuation if that commit boundary is needed, and keep the phase open until the host
caller and its pin are verified.

## Acceptance and verification

The focused regression suite must demonstrate:

1. A bead waiter crossing a simulated long wait and source update preserves its name and
   launch context and starts its workflow once.
2. The reproduced stale/self-ownership collision recovers automatically, with bounded
   reconciliation and no manual epic relaunch. If already fixed, the real-path test and
   fixing commit establish this behavior without new code.
3. Repeated ready/refresh processing or competing recovery attempts produce at most one
   workflow/successor execution; a different live owner is never stolen.
4. The ordinary no-collision wake works both with and without a source update.
5. A completed dependency barrier is not re-run; original dependency and workspace
   context survive recovery and are retained on a genuine ownership refusal.
6. Fresh explicit-name collisions, consumed clan handoffs, failed `execv` cleanup, and
   the import-free refresh boundary retain their existing behavior.

Read `lint_and_test.md` through `/sase_memory_read` before finishing. Run the new
focused tests and the relevant existing refresh, bootstrap, wait-completion, registry,
and bead-launch modules. Run `sase tool run check` in sase after changes; if Rust
changes, run its targeted tests and `sase tool run check` there too, then verify Python
against the matching binding and CI pin. Do not run `check-full` for this phase. Use
`/sase_monitor` for long commands as instructed; do not leave unmonitored commands
running at turn end.

## Completion

Append
`TASK RESOLVED: sase-12v — <root cause or already-landed fix, tests, and verification evidence>`
to `sase-1pu.2` only when acceptance is satisfied. For discovered unrelated work, use
only `sase bead note sase-1pu.2 'PROPOSED FOLLOW-UP: <summary — evidence>'`. If a check
fails identically on the clean base, record that reproduction and any existing task ID
as a proposed follow-up; it does not prevent phase completion.

Before closing, run `sase bead epic-symbols sase-1pu.2`. Resolve every remaining entry
or re-key it to an appropriate still-open parent/later phase, then run
`sase bead close sase-1pu.2 --note '<what was verified>'`. Planning found no current
epic-symbol entries, but recheck at completion. Do not close `sase-12v`, `sase-1el`,
`sase-1pu`, or an ancestor plan bead; the parent lander owns those decisions. Finish
through `/sase_final`, covering every repository changed.
