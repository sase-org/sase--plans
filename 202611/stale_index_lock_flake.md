---
tier: tale
title: Stabilize stale index-lock recovery caller tests under load
goal:
  Resolve sase-1vy.4 with deterministic owner observations, preserved lock safety, and
  five passing loaded repeats.
size: medium
proposed_by: bbugyi200.athena.sase-1vy.4
proposed_with: codex/gpt-6-astra
bead: sase-1vy.4
create_time: 2026-11-28 06:42:30
status: wip
---

- **PARENT:**
  [202611/backlog_epic_2.md](https://github.com/sase-org/sase--plans/blob/main/202611/backlog_epic_2.md)
- **BEAD:**
  [sase-1vy.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vy/sase-1vy.4.md)

# Make stale index-lock caller tests independent of host process churn

Implement the remaining work for assigned phase `sase-1vy.4`, which addresses
`sase-1kr`, under `plan:202611/backlog_epic_2.md`. Keep real Git commands, filesystem
locks, Rust classification, and Python retry/removal execution in the regression tests.
Make the synthetic locks' owner observations and stale ages explicit, then demonstrate
the affected nodes passing repeatedly under CPU contention.

## Scope and evidence

The planning checkout was clean at `a4bbd29cba`. The phase is in progress, has no
previous implementation or remainder notes, and has no epic-symbol entries. Recheck the
phase and recent history before implementing, because other work may land.

The seven corroborations on `sase-1kr` identify the shared recovery path across
workspace commands, dev-update/reporters, agent reverts, commit finalization, bare-Git
initialization, managed sidecar sync, and direct Git integration tests. Several logs
explicitly show `permission denied reading /proc/<pid>/fd`; another shows the recovery
budget expiring after waiting for a freshly created fixture lock to reach stale age.

Current code explains both test dependencies:

- `tests/test_sase_utils.py::test_run_workspace_git_command_recovers_stale_index_lock`
  creates and ages its own temporary lock but lets recovery scan the whole host.
- `src/sase/git_lock_observe.py::_linux_process_fact` reports incomplete observation
  when a Git process's descriptors cannot be inspected. Preserving a lock when owner
  absence cannot be established is the intended safety behavior.
- `src/sase/git_lock_retry/_policy.py` caches its observation callbacks. Patching the
  original observation function after that cache initializes is order dependent.
- `tests/git_lock_test_helpers.py` already supplies `absent_owner_facts`,
  `active_owner_facts`, `incomplete_owner_facts`, and `age_git_index_lock`.
- The bare-Git and managed-sync regression nodes still create young locks with
  `touch()`. They unnecessarily exercise the real 15-second age wait inside a 30-second
  recovery budget; deterministic young-lock coverage already exists in
  `tests/test_git_lock_retry.py`.
- The later `4a7ce0f330` change skips probing when no lock exists. It does not fix tests
  that deliberately plant a lock, and these caller tests still lack explicit owner
  facts.

This is one medium coding unit: a test helper, explicit adoption by the reported
callers, focused safety coverage, and a bounded repeated validation run. No earlier
phase remainder requires a child epic.

`sase-1s4` is related context, not the same defect: it tracks a test leaving a lock in
the real checkout. This plan must keep all synthetic locks in temporary repos and must
not add blanket lock cleanup or claim that `sase-1s4` is resolved.

## Governing decisions and boundaries

- `decisions:rust-core-required`: shared recovery decisions remain in Rust. This plan
  changes tests, not production policy, wire formats, or the pinned core revision.
- `decisions:check-full-is-explicit`: use `sase tool run check`; do not invoke
  `check-full`. Targeted contention verification is required by this phase.
- `decisions:guarded-recipes`: route the check recipe through the named tool.
- `decisions:host-owned-completion` and
  `decisions:handoffs-end-through-host-completion`: stage long work with `/sase_monitor`
  when necessary and finish every root turn through `/sase_final`.

The parent plan has no additional reviewer decisions for this phase. No memory edits,
feature flags, public API changes, global timeout increases, skipped tests, new task
beads, or further backlog epic are included. Read `lint_and_test.md` before finishing.
If deterministic evidence instead exposes a product defect, establish its cause and
respect the Rust boundary; do not turn unknown ownership into permission to unlink.

## Implementation

1. **Make the observation dependency explicit at the existing test seam.** Extend
   `tests/git_lock_test_helpers.py` with a small opt-in helper for caller tests. Patch
   the caller module's imported `run_with_git_lock_retry` to a forwarding callable that
   invokes the real public executor with a supplied `owner_probe`. Preserve all other
   arguments and the original result/outcome. Direct executor tests can pass
   `owner_probe` themselves. Use the existing fact builders; never mock a successful Git
   result, skip the executor, or replace Rust classification. The helper should check
   that observed lock/context paths belong to the temporary repo supplied by the test
   and expose enough observation evidence to assert the probe ran. Keep the patch
   function-scoped and avoid changing the cached global policy or installing an autouse
   fixture across unrelated tests.

2. **Apply the helper to the reported caller family.** Preserve each test's existing
   command, output, staging/HEAD, convergence, retry, and removal assertions. Include
   stderr or recovery logs in failure assertions where currently only a Boolean is
   reported. Cover these files and caller seams:

   | Test file                                                     | Imported executor to wrap                                                                                                |
   | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
   | `tests/test_sase_utils.py`                                    | `sase.core.shell.run_with_git_lock_retry`                                                                                |
   | `tests/ace/test_revert_agent_execute.py`                      | `sase.ace.revert_agent_git.run_with_git_lock_retry`                                                                      |
   | `tests/llm_provider/test_commit_finalizer_auto_sdd_status.py` | `sase.llm_provider.commit_finalizer_git_status.run_with_git_lock_retry`                                                  |
   | `tests/ace/tui/test_session_proc_reporter_git_lock.py`        | `sase.dev_update.command.run_with_git_lock_retry` (capture, streaming, reporter)                                         |
   | `tests/test_bare_git_init.py`                                 | `sase.workspace_provider.plugins.bare_git_init.run_with_git_lock_retry`                                                  |
   | `tests/test_bead/test_sync_remote_worker.py`                  | Follow the managed-sync call into `sase.sdd._git_contention.run_with_git_lock_retry` and verify the actual imported seam |
   | `tests/test_git_lock_recovery_integration.py`                 | Explicit `owner_probe` on the existing direct executor calls                                                             |

   For integration tests expecting absence, provide deterministic absent facts while
   retaining real `git add`, `commit`, and `checkout` operations. Cover the churned
   identity case with absent facts too, so it proves identity protection rather than
   incidentally passing due to an unknown owner. Do not override observation in the real
   open-descriptor owner test. Age the bare-Git and managed-sync synthetic locks with
   `age_git_index_lock` before invoking their commands. Keep the existing deterministic
   young-lock wait tests as the age-wait contract.

3. **Prove that isolation preserves safety.** At the workspace-command caller seam,
   exercise an aged temporary lock with absent, active, and incomplete owner facts.
   Absent facts must allow real staging and remove the lock; active/incomplete facts
   must return failure and retain the same lock and its contents. Exercise the
   incomplete case with a permission-denied reason matching the recorded failure. Keep
   existing live-descriptor, symlink, replacement/churn, cancellation, and
   budget-exhaustion coverage in `tests/test_git_lock_retry.py` intact. If adding
   observation-adapter coverage, call the public `probe_index_lock_owner` against a
   synthetic proc tree rather than depending on a privileged host process. Make
   assertions distinguish safe preservation from merely receiving any failure.

4. **Check isolation and test ordering.** Run a caller test after the retry policy has
   already initialized; the injected facts must still be used. Confirm the temporary
   patch is restored and the genuine descriptor-owner test still runs with its real
   observation path. Keep test changes focused on this family; record unrelated findings
   on the phase instead of expanding scope.

## Verification and acceptance

First run the changed caller modules plus `tests/test_git_lock_retry.py` and
`tests/test_git_lock_observe.py` with the checkout's pytest. Capture outcomes for
permission-denied/unknown and live ownership as well as successful recovery.

Then use the existing `just test-contention -- <explicit nodes/files>` harness. Choose
two allowed CPU IDs from the process affinity and set `SASE_CONTENTION_CPUS`
accordingly, `SASE_CONTENTION_WORKERS=8`, and `SASE_CONTENTION_REPEAT=5`. Do not run a
whole-suite soak. Include at least:

- `tests/test_sase_utils.py::test_run_workspace_git_command_recovers_stale_index_lock`
- `tests/ace/test_revert_agent_execute.py::test_revert_git_runner_recovers_stale_index_lock`
- `tests/llm_provider/test_commit_finalizer_auto_sdd_status.py::test_auto_commit_git_runner_recovers_stale_index_lock`
- `tests/ace/tui/test_session_proc_reporter_git_lock.py`
- `tests/test_bare_git_init.py::TestInitBareGitProject::test_git_runner_removes_persistent_index_lock_after_retries`
- `tests/test_bead/test_sync_remote_worker.py::test_managed_sync_worker_converges_sidecar_store_mutations`
- `tests/test_git_lock_recovery_integration.py`
- The new caller-level owner-preservation tests.

Run the contention command through `/sase_monitor` if it will outlast a synchronous
turn. Preserve the exact selection, CPU/worker/repeat settings, elapsed time, and
per-node tally. Acceptance requires all affected nodes green in all five loaded repeats,
without relaxing production safety or budgets. A single isolated pass is insufficient.
If a changed node fails, inspect its diagnostic reason and fix the mechanism before
repeating; do not treat it as the historical flake automatically.

Run formatting for the edited files and `sase tool run check`. Preserve the ToolRun ID
and result. Investigate UNKNOWN failures. A failure reproduced identically on the clean
base is a `PROPOSED FOLLOW-UP:` note, with an existing task ID if known, and does not
block this phase's close. Follow `/sase_repo` rules if a separate checkout is needed for
that comparison.

## Completion and ownership

Record the mechanism, changed callers, five-repeat tally, safety results, and check
ToolRun on `sase-1vy.4`. Provide enough evidence for the parent land agent to settle the
source task `sase-1kr`. The current user instruction authorizes closing only the
assigned phase: leave `sase-1kr`, `sase-1s4`, parent `sase-1vy`, and all other beads
unchanged in status, despite the older phase description's instruction to close the
source task.

Before closing, run `sase bead epic-symbols sase-1vy.4`; resolve any listed symbols or
re-key them to a still-open owning bead, following `symvision.md` if needed. Close only
`sase-1vy.4` with `sase bead close sase-1vy.4 --note "<verification evidence>"`. Use
`sase bead note sase-1vy.4 'PROPOSED FOLLOW-UP: <summary — evidence>'` for discovered
work; do not create task beads. Finish through `/sase_final` with bead action `close`
only after implementation and verification are complete; intermediate handoffs use
`keep`.
