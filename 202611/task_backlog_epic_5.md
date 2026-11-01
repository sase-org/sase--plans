---
tier: epic
title:
  "Task backlog epic 5: apollo-only prompt-render red, agents-sync drain flake, and the
  sase-core sudo_runner ETXTBSY flake"
goal:
  Fix and close the five next-costliest open task beads, then continue the backlog chain
  under the same ranking and closeout rules.
decisions:
  optional_scope:
    ask:
      Should phase workers include adjacent feature or memory changes discovered while
      fixing these five bug and flake tasks?
    choices:
      keep_five_seeds:
        Limit work to the five selected task beads; defer adjacent feature or memory
        work to owner review
      required_scope_only:
        Include a feature or memory change only when a selected bead's acceptance
        requires it
    default: keep_five_seeds
    why:
      The selected beads are defect and flake work, so optional scope should stay under
      owner control.
    answer: keep_five_seeds
phases:
  - id: apollo-prompt-render
    title:
      Fix the apollo-only ACE prompt-render and hint failures (sase-1ia, sase-1l3,
      sase-1hy)
    depends_on: []
    size: large
    description:
      "apollo-prompt-render: the ACE prompt-render and hint widget tests behind
      sase-1ia, sase-1l3 and sase-1hy fail deterministically on apollo (7-9 failed of 30
      across four files at sase 31a28d9618) and pass 30/30 on athena. Bisect the apollo
      host input that changes the result, fix it where it is owned (test isolation or
      product code), and prove the four files pass on both hosts without weakening an
      assertion."
  - id: agents-sync-drain-flake
    title: Fix the agents-sync bounded-drain publication flake (sase-xo)
    depends_on: []
    size: large
    description:
      "agents-sync-drain-flake:
      tests/agents_sync/test_commit_publication_bounded_drain.py::test_blocked_render_is_bounded_and_leaves_the_request_queued
      still fails in loaded full lanes after the 1a69c8947d/4adee846d3 fix (recurrences
      2026-10-27 and 2026-10-31; one report measured 15.63s after git index-lock
      backoff). Find what still consumes the wall-clock budget under load and fix it
      without raising the elapsed bound or dropping the queued-request assertion."
  - id: core-sudo-runner-etxtbsy
    title: Fix the sase-core sudo_runner ETXTBSY fixture flake (sase-15h)
    depends_on: []
    size: medium
    description:
      "core-sudo-runner-etxtbsy: in sase-core, sase_gateway sudo_runner tests
      intermittently fail with ETXTBSY (os error 26) when exec'ing fixture scripts under
      the parallel core check (+7, latest 2026-10-31). Confirm the fork-inherited
      write-fd race on freshly written fixture executables and fix the fixture so no
      test can exec a script while any process still holds it open for writing, keeping
      every sudo_runner assertion."
proposed_by: bbugyi200.athena.sase-1qn.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-01 18:08:16
status: wip
---

- **PROMPT:**
  [prompts/202611/task_backlog_epic_5.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/task_backlog_epic_5.md)
- **PARENT:**
  [202611/task_backlog_epic_4.md](https://github.com/sase-org/sase--plans/blob/main/202611/task_backlog_epic_4.md)

# Task Backlog Epic 5: Apollo-Only Prompt-Render Red, Agents-Sync Drain Flake, and the sase-core sudo_runner ETXTBSY Flake

## Context

This epic continues the fleet-wide task backlog chain after epic sase-1qn
(`202611/task_backlog_epic_4.md`). Its land agent verified and closed that epic's five
seeds (sase-o7, sase-1ic, sase-1gz, sase-13a, sase-19m) on 2026-11-01 at sase master
`31a28d9618` with sase-core pin `e60f3e1658`. It then re-listed the 387 non-closed task
beads and ranked them with the selection-health full-run records
(`~/.sase/test-selection/gh_sase-org__sase/*-full-run.json`; 630 full runs since
2026-10-18), counting only runs with at most five failures and only failures outside
each run's own changed files.

- **No duplicate or obsolete closures this round.** Every examined candidate still
  reproduces or is merely dormant:
  - sase-zo: the configured `sase repo open <name>` half is fixed, but
    `sase repo list -p sase` still exits 2 ("no WORKSPACE_DIR") at `31a28d9618`. Last +1
    was 2026-10-16. Next ranking.
  - sase-1q3: `SASE_MODEL_OVERRIDE=sonnet` still yields 7 failed / 44 passed on the
    cited nodes at `31a28d9618`. It hits only fallback-model retry agents, has no +1,
    and is an xsmall fix. First in line for the next epic.
  - sase-lk: dormant. The supervisor-timeout nodes have failed about four times since
    2026-10-09, and the last +1 was 2026-09-26. It stays open.
- **Master-red bursts already fixed:** the highest raw full-run counts
  (`test_contract_manifest`, `test_monitor_start_ack`, `test_monitor_start_supervisor`,
  completion `test_snapshot`, the slot-scan capacity tests, the artifact-directory and
  marker-path audits, the macro-terminology audits) all pass at `31a28d9618`: 110/110
  under `-n 6`. Those were transient master-red episodes, not open flakes.
- **Reconsidered and excluded:**
  - sase-j0 is `in_progress` under its own holder.
  - sase-1em and sase-1oy each had about four recent full-run failures. Leave them for
    the next ranking.
  - sase-1ps and sase-1q0 still have no corroboration.
  - sase-1qj and sase-1qi are owned by active epic sase-1qk.
  - sase-1i9 (a hint-mode node) was verified passing on athena on 2026-10-26. The apollo
    phase rechecks it as related evidence. It is not a seed.
- **Ownership check:** no active epic plan owns any of the five beads below. The wip
  plans that mention sase-1hy, sase-1ia, sase-xo, or sase-15h list them as KNOWN
  failures and say not to repair them.

| Bead                 | Fleet-wide cost                                                                                                                                                                                                                                                                                                                                |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sase-1ia, -1l3, -1hy | +9, +8, +7, each reopened once. Every apollo agent whose check selects the ACE prompt-render and hint widgets goes red. The sase-1qn lander reproduced this on apollo at `31a28d9618`: 7–9 failed of 30 across four files. Earlier landers excluded the cluster because it passes on athena; this epic carries the apollo reproduction recipe. |
| sase-xo              | +7, reopened twice. The bounded-drain test recurred after the 2026-10-21 fix: full-lane failures on 2026-10-27 at `daa96a5bd0` and 2026-10-31 at `eec20f6ad5`.                                                                                                                                                                                 |
| sase-15h             | +7, latest 2026-10-31. A sase-core `sase_gateway` sudo_runner fixture exec fails with ETXTBSY under the parallel core check and blocks core-touching agents.                                                                                                                                                                                   |

The three phases touch disjoint files and have no ordering constraints, so they may run
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

- Read every assigned task bead, including every note and +1 entry, with
  `sase bead read <id> -r "<why>"`. Reproduce the failure on a tree and host that can
  exhibit it before changing code.
- Keep assertions, safety refusals, and existing budgets. Do not add skips, xfails,
  flake-baseline entries, timeout or elapsed-bound increases, or suppressions merely to
  make a gate pass.
- Shared backend behavior belongs in sase-core. Open it with `/sase_repo` before reading
  or editing it. Keep test-fixture and presentation fixes in this repo. If a Rust
  binding changes, let host completion move the pin; never invent a SHA.
- Before finishing, read the `lint_and_test` reference memory, plus `tui` before TUI
  changes. Run focused regressions, then `sase tool run check` in each changed
  repository. Do not run `just check-full`.
- Record each failure's base, host, runtime, before/after results, and limits. Do not
  create task beads. Record out-of-scope findings as `PROPOSED FOLLOW-UP:` notes on the
  phase bead.
- Record `TASK RESOLVED: <task-id> — <evidence>` on the phase bead, resolve any
  `--epic-symbol` entries the phase keyed, and close only the phase bead. The lander
  closes the task beads.

## Phase apollo-prompt-render: fix the apollo-only prompt-render and hint failures (sase-1ia, sase-1l3, sase-1hy)

Read sase-1ia, sase-1l3, and sase-1hy in full, and sase-1i9 as related evidence. These
four test files fail on apollo and pass on athena at the same commit:

- `tests/ace/tui/widgets/test_agent_clan_aggregation.py` (sase-1ia: the member loader
  keeps the raw `#research first segment` prompt instead of the expanded body)
- `tests/ace/tui/widgets/test_agent_display_agent_session_render.py` (sase-1l3: the
  `--plan` turn prompt renders as `...`, and `plan prompt line 15` is missing at every
  fold level)
- `tests/ace/tui/widgets/test_identity_header_raw_prompt.py` (sase-1hy: a duplicate
  file-hint mapping)
- `tests/ace/tui/widgets/test_agent_display_raw_prompt.py`
  (`test_done_agent_renders_raw_prompt`)

The beads also name sibling nodes, such as the
`test_agent_display_agent_session_hints.py` and `test_agent_header_panel_*` hint rows.
Include them in the reproduction.

**Reproduction recipe (the lander used this; apollo is reachable as `ssh apollo`).**
Never run tests inside an apollo SASE workspace, and clean up every scratch path
afterwards:

1. Copy a scratch snapshot:
   `git archive HEAD | ssh apollo 'mkdir -p /tmp/<scratch> && tar -x -C /tmp/<scratch>'`.
2. Create a scratch venv with `~/.local/bin/uv venv -p 3.12` on apollo. Install
   `-e ".[dev]"` with an `--override` that pins `sase-core-rs` to the current cached
   abi3 wheel from athena's `~/.sase/cache/sase-core-artifacts/` (copy it with `scp`).
3. Run the four files with `-n 4 -p no:cacheprovider`.

**Inputs the lander already ruled out on 2026-11-01:**

- Python version: athena with a Python 3.12.13 venv and the current wheel passes 30/30.
- The Rust binding: apollo with athena's current wheel still fails.
- User config: apollo with an empty `HOME` and `XDG_CONFIG_HOME` still fails.
- Environment variables and PATH: apollo under `env -i` with a minimal PATH still fails.
- The git-archive layout: an athena git-archive snapshot passes.
- Timezone: athena with `TZ=UTC` passes.

Remaining differences to bisect: CPU count (16 on apollo vs 64 on athena) and any timing
or concurrency the renderers depend on, system git (2.43.0 vs 2.47.3), filesystem and
temp-path layout, locale and glibc, and kernel. Find the input that changes the rendered
prompt, file-hint mapping, or prompt precedence.

- When the dependency is a real product behavior, for example rendering that depends on
  git output or a host-dependent path or file check, fix the product code so ACE renders
  the same on every host. Then add a regression that pins the input.
- When the dependency is test-only, fix the test or fixture isolation.
- Either way, keep the expanded-prompt precedence, the full-section rendering, and the
  single file-hint assertions exactly as they are.

Acceptance: the four files and the sibling nodes the beads name pass on apollo and
athena at the phase's tree. A regression fails if the identified host input is
reintroduced. sase-1i9's node is rechecked on apollo, and its result is recorded for the
lander.

## Phase agents-sync-drain-flake: fix the agents-sync bounded-drain publication flake (sase-xo)

Read sase-xo in full, including its two close/reopen cycles. 1a69c8947d stopped the
publication drain alarm from being swallowed by broad `except Exception` guards, and
4adee846d3 bounded cleanup. Even so, the node failed again in loaded full lanes on
2026-10-27 and 2026-10-31. One relayed report measured 15.63s after git index-lock
backoff. The test runs a real 30s blocked render with a short drain timeout and asserts
`elapsed < 5.0`.

- Reproduce under load, for example with repeated `-n` runs of the module next to a CPU
  and git-heavy neighbor such as the agents_sync suite. Record per-stage timings: render
  start, drain timeout, alarm delivery, cleanup, index-lock backoff, and process
  teardown.
- Find the stage that can still exceed the budget after the timeout fires. Likely
  candidates are blocking git index-lock retries, a cleanup path that waits on the
  blocked render, or a subprocess teardown that is not bounded. Fix that cause in the
  publication code or the test fixture, whichever owns it.
- Keep the elapsed bound and the queued-request assertion. Do not raise the timeout.

Acceptance: the node and its module pass repeated loaded runs, with before/after timings
recorded. A regression proves the identified stage stays bounded after the drain
timeout.

## Phase core-sudo-runner-etxtbsy: fix the sase-core sudo_runner ETXTBSY fixture flake (sase-15h)

Read sase-15h and its +1 entries. The failing tests include
`dispatch::apt_get_shaped_command_uses_requested_cwd_without_sudo_chdir_option`,
`dispatch::successful_run_uses_expected_sudo_order_and_cleared_environment`,
`command_literal_chdir_flag_after_boundary_is_allowed`, and
`worker::output_policy_none_keeps_ledger_empty`. They live under
`crates/sase_gateway/src/sudo_runner/tests/` in sase-core and fail with ETXTBSY
(`Text file busy`, os error 26) under the parallel core check.

- Open sase-core with `/sase_repo`. Confirm the mechanism: a fixture writes an
  executable with `fs::write` or a similar call, and another test thread forks a child
  before that write fd is closed. The child inherits the descriptor until its own exec,
  so the kernel refuses to exec the script. Reproduce it with a loaded loop or a
  deterministic test that holds a forked child open across the write.
- Fix the fixture so no script is exec'd while any process could still hold it open for
  writing. For example, write and close a temp file and then atomically rename it into
  place, or serialize fixture writes against spawns with a shared lock. A bounded retry
  on ETXTBSY is acceptable only as a documented backstop behind a real fix. Apply the
  fix to every sudo_runner script writer, including `support.rs`, the script writers in
  `dispatch`, and `worker.rs`.
- Keep every sudo_runner assertion, including argv, cwd, environment, and ledger checks.

Acceptance: a loaded loop that reproduced ETXTBSY (or a deterministic regression for the
race) passes, the sudo_runner tests pass repeatedly under parallel load, and
`sase tool run check` passes in sase-core.

## Landing

The lander re-reads the epic and every task, phase, and nested bead and note. It checks
each `TASK RESOLVED` claim against the source and commits, and integrates any drift that
landed while the epic ran. It closes each task bead normally with its own commit or test
evidence, and triages every `PROPOSED FOLLOW-UP:` through `/sase_new_task`. When the
apollo phase shows sase-1i9 passing on apollo, the lander may close sase-1i9 as obsolete
with that evidence. Then it:

1. Resolves every `sase bead epic-symbols` entry.
2. Closes the epic normally, with the actual follow-up triage in its note.
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
   - breaks every agent's check or CI, on any fleet host;
   - breaks or wastes every landing;
   - security or data-loss risk;
   - heavy +1 corroboration.

   The selection-health full-run records give objective recurrence counts. Ignore runs
   with many failures and failures inside a run's own changed files.

3. Write an epic plan with at most 5 phases and at most 5 task beads. One phase may work
   several beads. Use plan decisions sparingly, but always give the owner a way to opt
   out of feature work and memory changes they might not want.
4. That epic's Landing section must carry this same "continue the backlog chain" rule,
   until no non-closed task beads remain.
5. Propose it with `/sase_plan`. Proposing ends the land agent's turn, so do it last.
