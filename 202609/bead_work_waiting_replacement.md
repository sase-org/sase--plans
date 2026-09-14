---
tier: tale
title: Stop waiting bead workers before reusing their names
goal:
  Ensure sase bead work retires every superseded waiting shell before launching a
  replacement with the same name.
size: medium
proposed_by: bbugyi200.athena.0ko
create_time: 2026-09-14 12:04:35
status: wip
---

# Plan: Stop waiting bead workers before reusing their names

## Assessment and scope

I agree with the requested behavior: when `sase bead work` actually replaces a phase or
land agent, its previous `WAITING` shells must stop before the new shell can acquire
that name. Apply the same guarantee to task workers, which share the cleanup pipeline.
Keep a matching actively running worker and omit its replacement, as the command already
does. Only consider names selected for launch; this is not a cleanup of the whole epic
clan or of closed phases.

This is a medium tale because one coding agent can implement and verify the bounded
cleanup correction, including the Rust binding and Python integration. Do not launch
agents or kill real user agents to reproduce it. Use isolated fixtures and disposable
subprocesses.

## Findings from the current implementation

- `src/sase/bead/cli_work_handler.py` selects workers, confirms cleanup, revalidates the
  selection, and calls `prepare_selected_bead_work_force_reuse` before snapshot,
  preclaim, checkpoint, and launch. `cli_work_task.py` shares this cleanup path.
- `cli_work_cleanup_targets.py:classify_artifact_record` already classifies a live
  `WAITING` owner as `KILL`. Running owners are preserved. Tests in
  `tests/test_bead/test_cli_work_epic_relaunch.py` cover this classification and a
  waiting-to-running transition during confirmation.
- `cli_work_cleanup_apply.py` eventually calls `wipe_force_reuse_owners` by name.
  `src/sase/agent/names/_wipe_plan.py` expands a selected name into its artifact and
  related-history closure, including multiple artifacts with that name. Adding another
  name-based kill call would duplicate existing selection without fixing its completion
  guarantee.
- The confirmed defect is in `src/sase/agent/names/_wipe_execute.py`:
  `_terminate_artifact_process` counts a successful `os.killpg(pid, SIGTERM)` call as a
  kill. `execute_wipe_plan` then releases workspaces, removes artifacts, and rebuilds
  the registry without waiting for process exit. It also proceeds with removal when
  termination reports an error.
- The runner installs a soft SIGTERM handler in
  `src/sase/axe/run_agent_runner_signals.py`; `runner_signals.py` sets a killed flag and
  lets the runner finish its shutdown path. Signal delivery therefore does not prove
  that the old runner has stopped writing state.
- A read-only, in-memory simulation of the real `execute_wipe_plan`, with process and
  filesystem operations mocked, kept the target alive after SIGTERM. The function
  nevertheless released its workspace, deleted its artifacts, returned
  `killed_processes=1` with no errors, and reported its name released. This proves the
  ordering defect; it does not establish the exact history of the user's affected clan.
  Existing wipe tests assert signal delivery rather than exit.

## Implementation

### 1. Reproduce the failure at the real cleanup boundary

Extend `tests/test_agent_name_wipe.py` and the bead-work relaunch tests so that the
actual selection, closure, and cleanup code runs. Mock only external process, workspace,
and launch effects where appropriate. Represent two distinct artifact directories and
PIDs with the same fully qualified phase name, and repeat for a land worker. Include
realistic `waiting.json`, bead association, clan name, and clan generation. The current
helper's name-derived directory is insufficient for duplicates; extend the fixture
locally or give it an optional timestamp.

Require tests to fail when cleanup returns or the launcher runs while either old process
can still execute. Include delayed graceful exit and SIGTERM-resistant targets. Record
the selected targets and operation ordering to distinguish an enumeration failure from
the confirmed termination race.

### 2. Introduce a bounded stop-before-removal contract

Open `sase-core` using `/sase_repo` and `sase repo open sase-core` in the implementing
agent's own workspace. Follow that checkout's `AGENTS.md`. Shared lifecycle policy
belongs under `crates/sase_core/src/agent_cleanup`, with its wire/API exposed by
`crates/sase_core_py` and a thin adapter under `src/sase/core` in this repo.

Make Rust own the decision that a concrete cleanup batch can proceed to destructive
removal only after all required process stops are confirmed, together with bounded
timeout and failure outcomes. Keep the operation narrow: Python can collect fresh
process observations, execute signals through existing helpers, and perform the existing
filesystem/index effects. Do not introduce a parallel Python policy or migrate unrelated
agent cleanup functionality. Update wire schema fixtures if the existing schema changes;
require the binding for the new operation.

Use the existing `src/sase/agent/user_kill.py:request_user_kill` semantics for kill
intent, identity checks, SIGTERM, and SIGKILL escalation. Reuse its implementation
through the adapter where possible. It currently reports `force_killed` after SIGKILL
delivery, so explicitly confirm termination afterward as well. Preserve the existing
process identity protections; verify the captured target, not just a reused numeric PID,
and never infer exit solely from a removed marker or released registry name. An already
exited target is success. An unverified termination, permission failure, or timeout is
an error that blocks name reuse.

Use bounded polling and a batch-wide deadline rather than a full grace period multiplied
by the number of waiting phases. Stop every selected live process before removing any
selected artifact or releasing any selected name. In a failed batch, retain the
artifacts and reservations needed to diagnose and retry the cleanup; already stopped
processes cannot be rolled back, and the error should say which targets stopped and
which remain unresolved.

### 3. Connect confirmed stops to bead-work replacement

Update `_wipe_execute.py` and the necessary `_wipe.py` / `_forced_reuse.py` result
handling to enforce the stop barrier before workspace release, artifact deletion,
notification cleanup, and registry rebuild. Keep the existing batched catalog and single
final registry rebuild. Propagate failures through `ForcedReuseCleanupError` so epic and
task work abort before preclaim or launch.

Build the concrete artifact closure during preview and validate it again before
signalling, so every destructive target appears in the confirmation. A single registered
owner is not sufficient evidence about every artifact that a name wipe expands to. Carry
artifact path/timestamp and process identity through preview, validation, and execution;
do not collapse distinct shells merely because their names match. In particular, replace
the name-only stability key in `cli_work_cleanup_apply.py` with a concrete-shell key
when matching duplicate targets between snapshots. Use existing canonical owner-name
normalization for matching, not prefix or display-label comparisons. Require every
same-name waiting shell being replaced to be included, including older shells in the
same clan. Keep the populated clan container intact.

Preserve the existing owner-association and confirmation rules. If a selected waiting
shell becomes running during normal selection revalidation, preserve it and omit its
replacement. If a closure member or generation changes after the last validation, fail
without starting replacements. A running duplicate hidden behind the selected registry
entry must prevent a broad name wipe from killing active work; report the conflicting
identity rather than silently broadening cleanup. Keep these additional decision rules
in the Rust operation with Python supplying observations. Do not add a global uniqueness
service or launch daemon.

After confirmed termination, complete existing removal/index maintenance and verify no
replaceable same-name waiting shell remains before returning cleanup success. Then allow
the existing launch path to proceed. Retain registry admission checks so a concurrent
new owner causes a collision failure, not an extra kill. Do not retry by killing a newly
observed generation automatically.

### 4. Preserve the command's established behavior

Keep dry runs and declined confirmations free of signals, cleanup, bead writes, and
launches. Preserve the meaning of `--yes` and `--yes-to-all`; no new CLI options are
needed. Preserve closed-phase exclusion, live-worker idempotency, family handling,
legacy clan handling, and the existing spawn-failure rollback.

Exercise both the generic CWD launcher and the planned bead-work adapter through their
shared prelaunch cleanup. Add a task-worker regression because it uses the same
primitive. Document the confirmed-stop-before-replacement behavior in `docs/beads.md`
without changing memory files or generated skills.

## Acceptance and verification

- Restarting an epic with waiting phase and land workers stops all superseded shells
  before replacement launch, including two shells sharing a name. The resulting active
  clan has one replacement per selected logical slot.
- Graceful and escalated termination both complete before cleanup and launch. A
  disposable subprocess that delays shutdown or ignores SIGTERM demonstrates this with
  real process behavior. Start it in its own process group, wait for an explicit ready
  signal, and always terminate/reap it in test teardown. Never signal fixture PIDs
  copied from the test runner.
- Stop failure leaves diagnostic state available and prevents preclaim, checkpoint, and
  spawning. No target is considered stopped merely because a signal was sent. Re-running
  after a successful cleanup is safe.
- Running workers, foreign or mismatched owners, unrelated waiting names, closed phases,
  and the clan container survive. Waiting-to-running and changed-generation races do not
  kill new or active work.
- Dry-run/confirmation tests still pass. Keep the existing structural bounds in
  `tests/test_bead/test_cli_work_launch_scale.py`; avoid a history scan per target.
- Run Rust contract and PyO3 tests through the core repo's `just check` (or
  `./scripts/check.sh`), including both crates. Do not substitute
  `cargo test -p sase_core`, and do not manually change release versions.
- Build/install the updated binding using the repositories' supported development
  workflow before testing the Python caller. Run focused wipe, user-kill, bead-work
  relaunch, cleanup failure, confirmation, task, and launch-scale tests. Read
  `lint_and_test.md` through `/sase_memory_read` and run `just check` in sase. Use
  `/sase_monitor` for checks that need a long-running handoff; run `just check-full`
  through that skill if the verification rules require it.

Implementation begins only after this plan is submitted and approved. This planning turn
changes only the scratch plan, then validates and proposes it.
