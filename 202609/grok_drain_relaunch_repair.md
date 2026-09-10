---
tier: tale
title: Repair provider-drain replacement launches
goal:
  Usage-limit drains launch valid replacements, settle the original trigger, and report
  actionable failures without duplicating recovered work.
size: medium
proposed_by: bbugyi200.athena.0ix
create_time: 2026-09-10 16:40:19
status: wip
---

# Repair provider-drain replacement launches

Make the existing automatic provider drain actually launch replacements after a
usage-limit failure. Repair the shared restart handoff, settle the precise triggering
run, prove successful recovery through the real launch validator, and expose actionable
information when a replacement cannot start.

This is a medium tale: one coding agent can implement and verify this bounded repair to
existing orchestration. No new scheduler, retry service, provider routing policy,
feature flag, or public CLI option is needed.

## Evidence and diagnosis

On September 10, 2026, Grok's usage balance was exhausted. Notification
`6a90d32a-47a4-41b6-ab66-689c9d0a8500`, sender `llm.usage_limit`, at 13:59:04 EDT
reported seven attempted relaunches and zero completed relaunches. Inspect the original
incident with:

```sh
sase notify show --id 6a90d32a-47a4-41b6-ab66-689c9d0a8500
sase proc show smygc1qx8adw --all-lines --output-only
```

The drain proc selected seven agents and resolved every replacement to Claude Sonnet.
All seven results were `partial`: the old processes were killed, `launched` was null,
and the error was `Agent name '<name>' uses forced reuse; confirmation is required.`
Thus this incident was not a failure to detect Grok's limit or choose a fallback.

The specific defect is in `src/sase/agent/_restart_execute.py`:

1. `plan_agent_restart()` stores the restart request, still containing `%id:!name` or
   its parenthesized equivalent, in `AgentRestartPlan.rewritten_prompt`.
2. Its nested `ForceReuseLaunchPlan.rewritten_prompt` already has the active `!` removed
   by the shared parser and carries the per-segment environment needed for bead
   association.
3. Execution stops/dismisses the old row and applies the name wipe, then incorrectly
   calls `launch_agents_from_cwd(plan.rewritten_prompt, ...)`.
4. Normal launch validation correctly rejects that still-forced request. The approved
   `sase run` path in `src/sase/main/query_handler/_launch.py` instead uses the nested
   prepared prompt after applying reuse.

A read-only reproduction using `plan_force_reuse_launch()` and
`preflight_launch_name_requests()` confirmed that the outer prompt raises
`AgentNameReuseConfirmationRequiredError` while the nested execution prompt passes. No
agents or name reservations were changed during planning.

`tests/fakey/test_provider_drain_e2e.py` explicitly documents this known defect and
currently asserts the erroneous result: the old artifacts disappear, the drain fails,
and the notification says no relaunch completed. The test must become a regression test
for successful replacement. Historical context was read through
`plan:202608/provider_drain.md`, `bead:sase-su`, and `bead:sase-su.5`; the original plan
requires a real successful relaunch. Reconcile against current code and tracking state
without reopening or closing unrelated historical work.

The same proc log contains a separate contributing defect: settling trigger
`sase-z4.6.5.1` scans unrelated workflow records and passes `sase/fix_just` to strict
agent-name normalization, which raises `ValueError`. Settlement is then skipped. The
trigger is absent from the seven moves. The traceback alone does not establish why it
was absent; do not claim the forced-reuse error explains that omission.

## Implementation

### 1. Use the prepared prompt at the restart execution boundary

In `_restart_execute.py`, launch `plan.force_reuse_plan.rewritten_prompt` after the
existing successful stop and `apply_force_reuse_launch()` steps. Preserve
`segment_extra_env`, project/VCS refs, model aliases and overrides, family/clan
membership, bead association, and home-CWD launch semantics. Keep the original request
available for preview and audit.

Use the shared parser and preflight for the exact execution prompt before destructive
steps wherever a syntax-only check is needed. Reservation collision checks still run at
launch after the authorized wipe; a syntax preflight must not falsely reject the old
name merely because it is still reserved. Do not add a global `allow_force_reuse=True`
launch bypass, remove validation, or strip `!` with string replacement. The existing
trusted preparation has already consumed that request.

Preserve the current per-move failure isolation: stop failure prevents wipe and launch;
wipe failure prevents launch; launch exceptions or empty results produce an honest
partial result, and subsequent drain moves are still attempted. Do not count a launch
attempt as a successful replacement.

### 2. Settle the original trigger by its concrete run identity

Carry the already-known triggering `artifacts_dir` as an optional
`trigger_artifacts_dir` in the durable operation payload constructed by
`usage_limit_disable._submit_drain()`. Pass it through `src/sase/ops/commands/agent.py`
to `settle_drain_trigger_agent()`.

For new payloads, build the existing `WaitTarget(kind=AGENT, artifact_dir=...)` and use
the existing bounded watcher against the matching scan record. Avoid resolving a known
concrete run through clan/family/workflow-name heuristics. Limit the snapshot supplied
to this settlement to the original artifact identity so the wait classifier's
retry-chain following cannot switch the wait to a later replacement. The existing
60-second bound and best-effort failure behavior remain; a missing original artifact
must not make the proc wait on a newer same-name run.

Keep old payloads without this optional field readable through the existing bounded
name-based fallback. Neither malformed unrelated workflow names nor a newer same-name
run may break the new exact-artifact path. Do not weaken valid agent-name rules to
accept slash-containing workflow labels. A general overhaul of workflow-name lookup is
outside this repair.

This is Python process/payload glue over the existing Rust-backed scan and existing wait
interfaces. The Rust repo was inspected through `sase repo open gh:sase-org/sase-core`;
no new Rust domain policy is required by this design. Any implementation that does
introduce shared identity/selection semantics must implement them in sase-core with
bindings and tests rather than adding a Python fallback.

### 3. Make unsuccessful replacement results recoverable and visible

Audit `_restart_recovery.py`, `_restart_render.py`, `_drain_render.py`, and
`_agent_drain_notify.py` together. Recovery bundles are written before cleanup and
survive removal of the old artifacts. Preserve the authored restart request and the
bead/family information needed by the existing approved force-reuse path.

The current recovery command, a bare `sase run` of `rewritten.md`, repeats the same
forced-reuse rejection. Stop presenting that command as executable recovery for a forced
request. Point the operator to the saved prompt and the existing ACE launch flow, which
reviews forced reuse and reconstructs its scoped bead authorization. Emit a direct CLI
recovery command only for a prompt whose actual launch semantics have been validated;
removing the marker alone is insufficient when scoped metadata or a still-reserved name
is involved. Preserve recovery context for wipe failure as well as post-wipe launch
failure. Do not add a privileged replay bypass.

Extend usage-limit drain notes to report failed moves as well as successful moves and
skips. Name a bounded number of failed agents, summarize their actual errors, and point
to retained recovery context or the durable proc output for the complete result. Mixed
success/failure must not hide failed rows. A planning-error envelope must report its
error rather than claiming there were no candidates. Preserve the single-notification
ownership contract and stable per-agent JSON fields; include available inline recovery
context if bundle persistence failed, instead of dropping it from the drain result.

Update `docs/llms.md`'s provider-drain section with failure inspection and the supported
saved-prompt recovery procedure. Keep existing provider-disable duration, alias routing,
feature-flag state, relaunch limits, and monitor/question/caller exclusions intact.

### 4. Replace assertions of failure with behavioral regression coverage

Update the restart fixtures and tests so the launch stub validates the supplied prompt
with the real name preflight. It must fail against the old implementation and pass with
the prepared execution prompt. Cover both live-stop and completed-dismiss paths,
including a recently failed usage-limit row. Cover explicit and injected identity,
parenthesized/family identity, bead environment propagation, and fenced/literal `!` text
through existing parser tests where appropriate.

Change the flag-on fakey drain drill to require a successful durable proc result, one
actual replacement with a new artifact identity and recorded PID, execution using the
harmless fakey backend, the enabled fallback display route, and one notification
reporting the successful relaunch. Do not merely change the expected counts. Keep
provider execution hermetic and ensure all child processes are cleaned up. Retain the
flag-off case and separate injected-failure coverage for partial drains.

Add settlement tests with a real scan/wait classification fixture containing the trigger
and an unrelated `workflow_name: sase/fix_just`, plus a newer same-name run. Verify the
original trigger reaches its terminal state without a name-normalization exception or
following the replacement. Cover missing artifacts, old payloads, and timeout without
aborting the drain. Add result/notification tests for all-failed, mixed, planning-error,
and recovery-context cases.

Relevant existing suites are `tests/test_agent_restart_plan.py`,
`tests/test_agent_restart_execute.py`, `tests/test_agent_restart_cli.py`,
`tests/agent/test_force_reuse_launch.py`, `tests/test_force_reuse_launch_seam.py`,
`tests/test_agent_provider_drain_*.py`, `tests/test_agent_drain_cli.py`,
`tests/test_agent_drain_usage_limit_report.py`, `tests/test_ops_agent_drain_notify.py`,
`tests/test_llm_provider_usage_limit_disable.py`, and
`tests/fakey/test_provider_drain_e2e.py`.

### 5. Reconcile the incident before proposing operational recovery

After the implementation and verification, inspect current agent state and the original
proc's recovery directories. These were the seven failed moves:

| Agent                  | Recovery bundle under `~/.sase/restarts/` |
| ---------------------- | ----------------------------------------- |
| `sase-xe.16.11.7.14.5` | `20260910135530-sase-xe.16.11.7.14.5`     |
| `bob-cli-1z.3`         | `20260910135600-bob-cli-1z.3`             |
| `sase-xe.16.11.7.14.2` | `20260910135630-sase-xe.16.11.7.14.2`     |
| `0is--code`            | `20260910135659-0is--code`                |
| `sase-z7.3`            | `20260910135734-sase-z7.3`                |
| `bob-cli-1z.2`         | `20260910135804-bob-cli-1z.2`             |
| `sase-za.4`            | `20260910135834-sase-za.4`                |

Five of these names already appeared as later runs during this investigation.
`0is--code` and `sase-z7.3` were absent from the recent listing; absence alone does not
prove outstanding work. The trigger also had a later completed Claude run. Recheck
completed/dismissed history, family continuations, and associated work status using the
audited SASE tools before choosing any retry.

Produce a concrete recovery recommendation for any work still missing, preserving its
original project, prompt, dependency, family/clan, and bead context. Do not replay all
seven saved commands or rerun already completed work. Launching new work must use the
existing reviewed launch surface and applicable `sase_run` workflow; prepare and verify
the code and recovery prompts before that handoff. This plan authoring turn changes no
runtime agent state and launches no recovery agents.

## Verification and acceptance

Read `lint_and_test.md` through `sase memory read`; run `just install` if the workspace
environment needs setup. Run the focused suites above and `just check`. Use
`sase_monitor` for long verification commands, and run `just check-full` through that
skill if the repository's selection/broadening rules require it. If Rust changes turn
out necessary, follow that repo's `AGENTS.md` and run its whole-workspace `just check`,
including binding tests; do not manually change release-owned version pins.

The repair is accepted when a real hermetic usage-limit drain creates the intended
replacement through normal validation, retains its identity metadata, settles the
original trigger despite unrelated workflow records, and reports every successful,
failed, or skipped move accurately. Recovery guidance must work through supported
authorization paths. The implementation report must separate the verified automatic fix
from the current disposition of the seven historical attempts and name any work that
still needs a reviewed recovery launch.
