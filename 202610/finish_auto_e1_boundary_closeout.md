---
tier: tale
title: Finish autonomy target-project inheritance and verified epic closeout
goal:
  Direct-approved session coders inherit live autonomy regardless of shell CWD,
  meaningful boundary tests prove the remaining E1 acceptance, and sase-1ip is closed
  normally.
size: medium
proposed_by: bbugyi200.athena.sase-1ip.land
bead: sase-1ip
create_time: 2026-10-10 07:02:18
status: wip
---

- **BEAD:**
  [sase-1ip](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ip/README.md)

# Remaining work for sase-1ip

This is a bounded repair and the final landing of existing epic `sase-1ip`. Implement it
directly, including the last closeout step in this turn. A tale has no land agent of its
own. Do not wait for this work's own commit SHA, push, or CI result: the host commits
after your turn, and closing the epic with the final code still uncommitted is the
normal workflow.

Read `sase bead read sase-1ip -r "Need the renewed landing audit and decisions"`,
`sase artifact read plan:202610/auto_e1_autonomy_record.md "Need original E1 acceptance"`,
and
`sase artifact read plan:202610/finish_auto_e1_landing.md "Need the previous repair requirements"`.
The original `decision_record=no` remains final. Edit no memory. This plan adds no
configuration, profile semantics, CLI options, or new feature flags.

## What is already verified

The previous landing's implementation **was committed**, despite its failed agent:

- Python landing: `166e34eae9850e70d1832fc6d607bb151e42f056`.
- Core provenance fix: `4ffe48ced77d21c024ee0df8d4796df1269e59b6`.
- Current core pin at audit: `e3b0907b8d3eafcd3d673b5af0c071d6a9f8370c`, which includes
  that fix and core phases `01b0ad73` and `51b66fdb`.

The renewed lander read the original epic and all notes of all seven closed phases, both
plans above, the Python and Rust implementation, and all commits since the first epic
commit `563f046a85`. Freshly fetched `origin/master` equaled audit HEAD `6ece2ac1bc`. No
primary, plans, or core files were changed by the renewed audit.

The following selected suite passed **179 tests, zero xfails, in 180.88 seconds**:

```sh
.venv/bin/python -m pytest tests/autonomy_contract tests/fakey/test_autonomy_lifecycle_e2e.py tests/test_autonomy_roles.py tests/test_plan_direct_approval_launch.py tests/test_plan_direct_approval_run.py tests/test_plan_direct_approval_recovery_execute.py tests/test_axe_plan_successor_auto_inherit.py tests/test_run_agent_runner_refresh_reconcile.py tests/test_agent_auto_restart_incident_replay.py -q
```

These passes miss the specific boundary cases below. Do not redo the working Rust
record/evaluator, mutation, log store, projection, gate evaluator, or CLI design.

Post-start integration to preserve:

- `274c65ec53` deliberately changes epic phase/land defaults to `standard` through
  `autonomy.roles`, with explicit tale/manual overrides and updated contract rows. Do
  not revert those rows to the original epic's old worker-default expectation.
- Auto-restart uses `agent/_restart_live_autonomy.py` for the same live-record prompt
  rewrite as retries. Preserve A-off behavior and the refresh import firewall.
- Gate creation now lives in `notification_gates/service_creation.py` and
  `service_evaluation.py`; directive metadata is split into build/preserved modules;
  direct approval and recovery share `main/plan_direct_approval_launch.py`.
- The opened core checkout had a newer `6ed9c3d8` removing awareness beyond sase's
  pinned revision. Do not advance the pin just because that unrelated commit exists.
  Review new landed drift before coding, and respect any intentional later changes.

## 1. Repair target-project inheritance at direct approval and recovery

`main/plan_direct_approval_launch.py::_host_composed_attach_env(prompt)` resolves the
predecessor using `ensure_project_file_and_get_workspace_num(create_missing=False)`.
That is the shell CWD project, not necessarily the project targeted by the composed
coder prompt. When CWD is another project or is unrecognized, the helper returns `{}`.
The subsequent correctly-targeted attach resolves successfully but has
`host_composed=False`; `build_agent_meta` resolves the unadorned coder prompt as manual
instead of inheriting the planner's live record.

This was reproduced through the actual sequence
`launch_coder_once -> prepare_agent_session_attach_launch -> build_agent_meta`. Only the
final process launch and project/session lookup were intercepted. A tale predecessor and
the same `+contract-proj %id(code, session=contract-agent) #coder(...)` prompt gave:

| Shell CWD lookup | Coder profile | Source    |
| ---------------- | ------------- | --------- |
| contract-proj    | tale          | inherited |
| other project    | manual        | prompt    |
| no project       | manual        | prompt    |

For a minimal regression setup, reuse the existing `test_landing_gaps.py`
`_host_composed_plan` and `_build_with_prompt` fixtures; set the initially resolved
plan's `host_composed=False`, put a real tale record in a temporary predecessor meta,
and have the session lookup reject projects other than `contract-proj`. Intercept
`launch_agents_from_cwd` only at the process boundary, invoking the actual attach and
metadata functions inside it. Vary the CWD lookup while keeping target context fixed.
The current same-project control passes; the other two expose the bug.

Carry the trusted host-composed intent through the already-resolved target project or
launch placement. Prefer transporting the existing `DirectApprovalPlan.project` and
placement, or using the existing resolved launch context, over adding another
prompt/project parser. Resolve the attach with the correct project and retain the
trusted marker through serialization and fresh attach resolution. Do not silently drop
required inheritance on a lookup failure. Preserve existing actionable error handling
and the standalone/session/transient launch ladder.

Keep these distinctions explicit:

- A direct-approved or recovery coder attached to a planner inherits its **live**
  record, regardless of CWD; no re-emitted `%auto` prefix is needed.
- An explicit agent-authored manual/off selection narrows, and widening remains refused
  by core.
- A normal human-authored session attach still uses its own authored selection.
- A genuinely standalone coder without a predecessor retains its existing behavior.
- An unreadable predecessor cannot grant wider autonomy.

Add regression coverage for direct approval and recovery callers, same/other/no CWD,
tale inheritance, live A-off, human attach and standalone controls. Exercise both
`autonomy_record_only` flag states while that flag remains. If its separately planned
retirement has landed, test the resulting unconditional behavior instead.

## 2. Complete the promised boundary verification

The prior remediation plan required these tests, but its final commit did not implement
their assertions. Replace the weak tests rather than accumulating redundant wrappers.

1. `test_direct_approval_host_composed_env_inherits` currently asserts only that the
   result is a dict, so `{}` passes. Assert the actual serialized attach is trusted and
   the resulting coder record has the expected policy/source/revision/last values. The
   new direct/recovery tests from step 1 should catch the demonstrated regression.
2. `test_contract_suite_publishes_nothing` installs spies but only launches temporary
   metadata and creates follow-up artifacts. It never creates any gate. Make it actually
   exercise automatic tale, epic, and question creation through the contract harness,
   with temporary SASE/SDD roots and fail-fast checks at durable plan/prompt publication
   and real agent/bead launch boundaries. Assert the expected automatic results too, so
   skipping the operation cannot pass. Keep core evaluation real; stub only the intended
   execution side effects. Verify the actual imported aliases are fenced. The historical
   `sase-1ir` fixture escape is evidence for this requirement, not evidence of a
   continuing leak today.
3. The fakey lifecycle still uses `_successor -> adapt_followup_artifacts` for each
   differently named transition. Drive the real coder/successor, monitor member and
   follow-up, and gate member and follow-up entry points. Relevant current modules
   include `axe/run_agent_successor.py`, `monitor/member.py`, `monitor/followup.py`,
   `gate_turn/member.py`, `gate_turn/followup.py`, and `agent/detached_child.py`. Stub
   provider/process execution and ephemeral allocation where necessary, while leaving
   policy decisions and inheritance real. Assert tale survives the chain, its epic gate
   parks, and a live A-off mutation survives the next actual transition.
4. Tighten the existing refresh and idempotent-gate regressions where their assertions
   stop short of the previous repair's acceptance: preserve/build/restore must retain
   manual revision and last=tale across refresh; repeated/recovered gate creation must
   preserve policy identity and exactly one decision-log row, including generated IDs.
   Read the existing tests first and reuse coverage that already proves these cases.

Shared semantics stay in core. These repairs should be Python transport and tests; open
`sase-core` through `/sase_repo` before any work there if a genuine core defect is
found. Do not add a Python policy implementation.

## 3. Verify and review drift

Run the strengthened contract/lifecycle and direct approval/recovery tests, plus the
existing role, refresh, monitor follow-up, and Plan Decisions receipt/default tests
affected by the repair. Keep the contract at zero xfails. Inspect new upstream commits
for overlapping fixes before landing; integrate them instead of restoring old copies.

Read `lint_and_test.md` via `/sase_memory_read`. Run `just fix` and then
`sase tool run check` (the governed `just check`) in each changed code repository. Do
not run `just check-full`. No rendered UI change is planned. Read TUI memory before
editing TUI code if the implementation unexpectedly needs it.

If verification needs `/sase_monitor`, its successor must receive the remaining epic
closeout below. A prepared completion that merely commits would skip the required
closeout. Do not make closeout depend on this turn's own future commit, push, or CI.

Spot-check checkout `sase autonomy explain` for a static tale and a live agent, the
filtered log, and agent AUTO display as in the original acceptance. Use audited artifact
reads for any saved prompt inspection. Report what was actually exercised; do not claim
a real provider-driven chain when the test uses function entry points.

## 4. Finish sase-1ip in this coding turn

All unrelated follow-up triage is complete and recorded in the epic's renewed landing
notes. Carry every outcome into the final close note:

- All seven phases' note #1 memory proposals are consolidated in ready small task
  `sase-1j8`; `decision_record=no` is honored and no memory is edited.
- `sase-1ip.4` note #2 was corroborated on `sase-13a`; `sase-1ip.6` note #2 on
  `sase-120`. No new occurrence was claimed by the renewed audit.
- `sase-1ip.7` note #2 was declined as a new task because `191bc6d2e3` already fixes the
  obsolete monitor-prefix assertion.
- `sase-1ip.5` note #2 is epic-owned; this tale completes its remaining CWD hole.
- The gateway IPC flake in epic note #4 is now corroborated on the exact existing task
  `sase-1gz`, citing the previous failing and passing ToolRuns. It is not remaining
  autonomy work. No duplicate tasks were created.

Re-read `sase-1ip` and all seven children with audited `sase bead read`, review new
notes and linked-plan readiness, confirm each descendant is closed, and verify the
remaining original exit criteria. `sase-1j0` existed at audit; `sase-11g` remains open
for its queue half with the auto-half note. Honor any subsequent flag retirement.

Run `sase bead epic-symbols sase-1ip`. Resolve every entry by wiring, privatizing, using
a valid non-test pragma, or deleting it under the Symvision policy. Re-key only when an
identified still-open later bead actually needs the exemption. Then close normally, with
an evidence-rich note containing actual checks, repaired boundaries, post-start
integration, the confirmed old commit SHAs, and every follow-up outcome:

```sh
sase bead close sase-1ip --note "<what you verified, test results, integration, and follow-up outcomes>"
just symvision
```

If symbols block closure, finish cleanup and retry. If named phases block closure,
finish or reopen them. Never use `--force` merely to succeed, or to advance a successful
nested landing. Force with a reason and canceled/superseded resolution is reserved for
deliberately abandoned scope, not this successful closeout.

Open the plans repository using `/sase_repo`. Set `status: done` in the frontmatter of
`202610/auto_e1_autonomy_record.md`, the original epic's PLAN
`plan:202610/auto_e1_autonomy_record.md`; include the plans repository in the final
declaration. Do this even if the old close note claimed it already happened: the renewed
audit found no plan-status commit in that file's history.

Finally run `sase bead read sase-1ip -r "Need the parent link"`. There was **no parent
bead** at this audit. If that is still true, finish normally. If a parent was added,
apply the original landing rules: close only a completed parent phase normally and leave
its containing epic to its lander; for a directly parented plan, recheck its previous
landing note, descendants and notes, linked plan and post-child drift, retire its
symbols, close normally, run `just symvision`, mark its plan done, and repeat only while
each direct plan ancestor is fully complete. Stop and note any incomplete or ambiguous
parent. Declare changed repositories with `/sase_final` only after this closeout. The
host owns commits after the turn.
