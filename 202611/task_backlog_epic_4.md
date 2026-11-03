---
tier: epic
title: 'Task backlog epic 4: stale whitelist rows, import-budget churn, and recurring
  check flakes'
goal: Fix and close the five next-costliest open task beads, then continue the backlog
  chain under the same ranking and closeout rules.
decisions:
  optional_scope:
    ask: Should phase workers include adjacent feature or memory changes discovered
      while fixing these five bug and flake tasks?
    choices:
      keep_five_seeds: Limit work to the five selected task beads; defer adjacent
        feature or memory work to owner review
      required_scope_only: Include a feature or memory change only when a selected
        bead's acceptance requires it
    default: keep_five_seeds
    why: The selected beads are defect and flake work, so optional scope should stay
      under owner control.
    answer: keep_five_seeds
phases:
- id: epic-symbol-lifecycle
  title: Stop --epic-symbol rows from outliving their bead (sase-o7)
  depends_on: []
  size: large
  description: 'epic-symbol-lifecycle: find why Justfile --epic-symbol rows keyed
    to a closed bead still reach master after the 8bea2e8414 close guard (2026-10-30
    recurrences for sase-1pj.4.1, sase-1pj.5.1.3, sase-1pj.5.1.4), and close that
    gap at the lifecycle point that actually creates it, so unrelated agents stop
    discovering stale rows at the symvision lint gate. Keep the closed-bead symvision
    refusal and the epic-whitelist policy. Add regression coverage for the reproduced
    path.'
- id: import-budget-churn
  title: Stop behavior-neutral module splits from turning the TUI import budget red
    (sase-1ic)
  depends_on: []
  size: large
  description: 'import-budget-churn: the first-party TUI import-count guard in tests/ace/tui/test_app_import_budget.py
    has been re-raised after toobig splits and reopened twice (+35). Make the guard
    catch genuine new import-path work while staying neutral to mechanical splits
    of modules already on the path, without weakening the deferred-module list or
    just raising the cap again. Verify against recent split commits and a synthetic
    new eager import.'
- id: core-federation-socket-flake
  title: Fix the sase-core federation_worker socket test flake (sase-1gz)
  depends_on: []
  size: large
  description: 'core-federation-socket-flake: in sase-core, reproduce federation_worker::imp::tests::ipc::listener_creates_private_socket_and_rejects_symlink
    failing under the parallel sase_gateway gate (+15, ten recurrences since 2026-10-18,
    latest 2026-11-01). Find the shared-state or timing cause (for example temp-dir
    or socket-path races in listener recovery) and fix it without weakening the private-socket
    or symlink-rejection assertions. Prove it with a loaded loop.'
- id: zsh-completion-pty-flake
  title: Fix the zsh sbd alias completion PTY flake (sase-13a)
  depends_on: []
  size: large
  description: 'zsh-completion-pty-flake: reproduce tests/completion/test_zsh_smoke.py::test_alias_sbd_completes_static_bead_tree
    timing out in loaded full lanes (+15, latest 2026-10-31). Identify whether PTY
    startup, completion-cache warmup, or shared state causes the stall, and fix the
    cause without dropping the alias completion assertion or simply raising its timeout.'
- id: agy-probe-timeout-flake
  title: Fix the AGY usage-probe rate-limit timeout flake (sase-19m)
  depends_on: []
  size: medium
  description: 'agy-probe-timeout-flake: tests/llm_provider/test_agy_usage_probe.py::test_agy_usage_probe_timeout_with_rate_limit_stderr_is_rate_limited
    fails under the loaded diff-scoped lane (timeout classified instead of rate_limited;
    latest +1 2026-11-01) and passes alone. Make the test and, if needed, the probe
    classification deterministic under load while keeping the rate-limit-wins-over-timeout
    contract.'
proposed_by: bbugyi200.athena.sase-1pu.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-01 14:34:46
status: done
bead_id: sase-1qn
---

- **PROMPT:** [prompts/202611/task_backlog_epic_4.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/task_backlog_epic_4.md)
- **PARENT:** [202610/task_backlog_epic_3.md](https://github.com/sase-org/sase--plans/blob/main/202610/task_backlog_epic_3.md)
- **BEAD:** [sase-1qn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qn/README.md)

# Task Backlog Epic 4: Stale Whitelist Rows, Import-Budget Churn, and Recurring Check Flakes

## Context

This epic continues the fleet-wide task backlog chain after epic sase-1pu
(`202610/task_backlog_epic_3.md`). Its land agent verified and closed that epic's five
seeds (sase-1os, sase-12v, sase-1mv, sase-120, sase-1cc) on 2026-11-01 at sase master
`0547d23ce3`. It then re-listed the 388 non-closed task beads.

- **Closed one duplicate:** sase-rh was closed as `superseded` by canonical sase-o7. Its
  original sase-rd.5 instance was fixed by 4f87eb4b2. Every later reopening was the same
  stale `--epic-symbol` lifecycle defect, and the evidence is carried onto sase-o7.
- **Reconsidered and excluded:**
  - sase-1px, sase-1py, and sase-1pz from sase-1pu note #7's candidate list are already
    closed.
  - sase-1ps (pager view-release flake) and sase-1q0 (footprint microbench) have no
    corroboration beyond their reports. Leave them for a later ranking.
  - The hint/prompt-render cluster (sase-1ia, sase-1hy, sase-1l3) passes on athena at
    `0547d23ce3` (9/9 and 33/33 focused tests). It still needs apollo reproduction.
  - sase-1qj and sase-1qi are owned by active epic sase-1qk
    (`202611/16p_stall_handoff_recovery.md`).
  - sase-1qm (host completion stale-worktree refusal) has been observed once, with no
    root cause yet.
  - sase-j0 is `in_progress` under its own holder. sase-1ot stays excluded after the
    owner rejected it.
- **Ownership check:** none of the five beads below is named as owned work by any active
  epic plan. The 16p recovery plan only says to +1 sase-1gz, not fix it.

These five beads have the highest current fleet-wide cost. Each one breaks or flakes
many agents' verification:

| Bead     | Fleet-wide cost                                                                                                                                                                         |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sase-o7  | +25 (plus sase-rh's three). Stale Justfile `--epic-symbol` rows fail every agent's `just check` at the symvision lint gate before tests run. Recurred after the 8bea2e8414 close guard. |
| sase-1ic | +35, reopened twice. The TUI first-party import-count guard turns red after mechanical toobig splits, so the cap keeps being raised by hand.                                            |
| sase-1gz | +15, ten recurrences since 2026-10-18. A sase-core `sase_gateway` IPC test flakes in the parallel core gate and blocks core-touching agents.                                            |
| sase-13a | +15. A zsh PTY completion smoke test times out in loaded full lanes, including scoped checks that escalate.                                                                             |
| sase-19m | +4, three since 2026-10-18 including 2026-11-01. A timeout-driven AGY probe test flakes in the default diff-scoped lane.                                                                |

The five phases touch disjoint files and have no ordering constraints, so they may run
in parallel.

> [!decision] optional_scope = keep_five_seeds
>
> Implement only the five listed bug and flake tasks. Leave adjacent feature and memory
> work open for later owner review.

> [!decision] optional_scope = required_scope_only
>
> Include a feature or memory change only when a selected task's acceptance requires it.
> Defer all other adjacent work.

## Rules for every phase

- Read the assigned task bead and every note and +1 entry with
  `sase bead read <id> -r "<why>"`. Reproduce the failure on a tree and under a load
  that can exhibit it before changing code.
- Keep assertions, safety refusals, and existing budgets. Do not add skips, xfails,
  flake-baseline entries, timeout increases, or suppressions merely to make a gate pass.
- Shared backend behavior belongs in sase-core. Open it with `/sase_repo` before reading
  or editing it. Keep test-fixture and presentation fixes in this repo. If a Rust
  binding changes, let host completion move the pin; never invent a SHA.
- Before finishing, read the `lint_and_test` reference memory, plus `tui` before TUI
  changes and `symvision` before Symvision changes. Run focused regressions, then
  `sase tool run check` in each changed repository. Do not run `just check-full`.
- Record each failure's base, runtime, before/after results, and limits. Do not create
  task beads. Record out-of-scope findings as `PROPOSED FOLLOW-UP:` notes on the phase
  bead.
- Record `TASK RESOLVED: <task-id> — <evidence>` on the phase bead, resolve any
  `--epic-symbol` entries the phase keyed, and close only the phase bead. The lander
  closes the task beads.

## Phase epic-symbol-lifecycle: stop --epic-symbol rows from outliving their bead (sase-o7)

Read sase-o7 in full, including the notes carried over from closed duplicate sase-rh.
The 8bea2e8414 guard makes `sase bead close` read both the local and default-branch
Justfile. Even so, rows keyed to sase-1pj.4.1, sase-1pj.5.1.3 (both found by
toobig-8u.reclaim.0--1), and sase-1pj.5.1.4 (ToolRun 3ca0ba01382eb58f81f4b5063766ebe1)
reached master after those phases closed on 2026-10-30.

- Reconstruct the exact sequence for at least one of those recurrences from git history
  and bead close times. Find the commit that added each row, the workspace it came from,
  the bead it was keyed to, and when that bead closed relative to when the commit
  landed. Candidate causes:
  - A row keyed to a sibling or later phase.
  - A phase closed before the host committed the row.
  - An auto-close path that skips the guard, such as "delegated work landed".
  - A re-key to a bead that then closed.
- Close the gap at the lifecycle point that actually lets the row through. Examples:
  extend the guard to the close path that skips it, or refuse or flag commits that add
  rows naming an already-closed bead at host completion. Keep the closed-bead refusal in
  symvision and the existing epic-whitelist policy.
- Add regression tests for the reproduced path and the ordinary guarded close.

Acceptance: the reproduced sequence can no longer leave a row keyed to a closed bead on
master without a refusal or a visible remediation at the moment it happens. The existing
close guard and symvision behavior still pass their tests.

## Phase import-budget-churn: stop behavior-neutral splits from turning the import budget red (sase-1ic)

Read sase-1ic in full, including its two close/reopen cycles and its recent +1 entries.
`tests/ace/tui/test_app_import_budget.py` guards the first-party module count
(`_MAX_SASE_MODULE_COUNT`, 2861 at `0547d23ce3`). The toobig split routine keeps
splitting modules already on the TUI import path, so the count grows with no new import
work and agents keep raising the cap by hand.

- Measure the current closure, then replay recent split commits (for example 24561602cc
  and 86bec79ca4) to confirm which count increases were split-only.
- Change the guard so it stays neutral to mechanical splits of modules already on the
  path but still fails for genuinely new eager imports. One option is a measure that
  splits do not change, such as the first-party source size of imported modules or
  facade-attributed counting. Choose the mechanism from your measurements.
- Keep the deferred-module list and the loose total-module bound. Do not just raise the
  cap again.
- Add regression proof that a synthetic split-only change stays green, and that a
  synthetic new eager import of a previously deferred or unrelated module fails.

Acceptance: the guard passes on master, stays green across a replayed split-only change,
fails a real new eager import, and its comment explains how to change the budget.

## Phase core-federation-socket-flake: fix the sase-core federation_worker socket test flake (sase-1gz)

Read sase-1gz and its +1 entries. The failing test is
`crates/sase_gateway/src/federation_worker/imp/tests/ipc.rs::listener_creates_private_socket_and_rejects_symlink`
in sase-core. It fails under the parallel `sase tool run check` gate and passes alone.

- Open sase-core with `/sase_repo`. Reproduce the failure in a loop under parallel load
  and record the exact assertion or panic text.
- Inspect shared temp directories, socket-path length or reuse, the listener's
  socket-target recovery, and permission races. Fix the cause in the test fixture or in
  the listener, whichever owns it.
- Keep the private-socket mode and symlink-rejection assertions.

Acceptance: a loaded loop that previously reproduced the failure passes. The test still
fails if the symlink is accepted or the socket is not private. `sase tool run check`
passes in sase-core.

## Phase zsh-completion-pty-flake: fix the zsh sbd alias completion PTY flake (sase-13a)

Read sase-13a and its +1 entries. The failing node is
`tests/completion/test_zsh_smoke.py::test_alias_sbd_completes_static_bead_tree[sbd show --for-sbd show --format ]`.
It times out in loaded full lanes (for example ToolRun ca21f9a45f6c297a802e8f97a8f40524
on 2026-10-30) and passes alone.

- Reproduce under xdist load and measure where the time goes: zsh startup, compinit or
  cache generation, the completion command, or PTY reads.
- Fix the cause. Examples: prewarm or isolate a per-test completion cache, read until a
  stable marker instead of racing a fixed window, or remove shared-state contention. Do
  not raise the timeout or drop the completion assertion.
- Check the neighboring zsh smoke parameters for the same pattern, and apply the fix
  there when it is the same mechanism.

Acceptance: the node and its module pass repeated loaded runs, with before/after timing
recorded. No assertion is weakened.

## Phase agy-probe-timeout-flake: fix the AGY usage-probe rate-limit timeout flake (sase-19m)

Read sase-19m and its +1 entries, including ToolRun af92a91bfbe6635cd5c072966a9dc112.
The failing node is
`tests/llm_provider/test_agy_usage_probe.py::test_agy_usage_probe_timeout_with_rate_limit_stderr_is_rate_limited`.
Under load it reports `timeout` where `rate_limited` is expected, and it passes alone.

- Trace how the probe decides between a timeout and rate-limit stderr. Check whether a
  loaded host leaves stderr unread or only partly drained when the timeout fires.
- Make classification deterministic. Ensure stderr written before the timeout is
  collected and classified. If the test races a real subprocess clock, give it a
  controlled clock or a synchronization point. Keep the rule that rate-limit evidence
  wins over a timeout.

Acceptance: the node passes under repeated loaded runs. A test proves that rate-limit
stderr captured before a timeout yields `rate_limited`, and that a timeout without
rate-limit stderr is still classified as `timeout`.

## Landing

The lander re-reads the epic and every task, phase, and nested bead and note, checks
each `TASK RESOLVED` claim against the source and commits, and integrates any drift that
landed while the epic ran. It closes each task bead normally with its own commit or test
evidence, and triages every `PROPOSED FOLLOW-UP:` through `/sase_new_task`. Then it:

1. Resolves every `sase bead epic-symbols` entry.
2. Closes the epic normally with the actual follow-up triage.
3. Runs `just symvision`.
4. Marks this plan file done.

Do not use `--force` to advance a successful landing.

Continue the backlog chain while non-closed task beads remain:

1. Re-list every non-closed task bead (`sase bead list -t task -n 0`). Close only beads
   you can verify are duplicates or obsolete:
   - **Duplicate:** resolution `superseded`, naming the canonical bead, with a note on
     the canonical bead.
   - **Obsolete:** resolution `done`, citing the fixing commit, the grep that proves the
     code is gone, or a passing focused test.

   Never close a bead an active epic owns. Never close a snoozed bead early. Beads that
   are merely dormant stay open.

2. Pick the 5 most important remaining task beads. Rank them by fleet-wide cost first,
   then by cost to fix. Fleet-wide cost means:
   - breaks every agent's check or CI;
   - breaks or wastes every landing;
   - security or data-loss risk;
   - heavy +1 corroboration.
3. Write an epic plan with at most 5 phases and at most 5 task beads. One phase may work
   several beads. Use plan decisions sparingly, but always give the owner a way to opt
   out of feature work and memory changes they might not want.
4. That epic's Landing section must carry this same "continue the backlog chain" rule,
   until no non-closed task beads remain.
5. Propose it with `/sase_plan`. Proposing ends the land agent's turn, so do it last.
