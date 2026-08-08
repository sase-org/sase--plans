---
tier: epic
title: Finish the verified residue and land epic sase-h8.10
goal:
  Epic sase-h8.10 closes only after its contract-budget target is deterministic under
  contention, post-start commits adopt the shared load-tolerant wait convention, the
  linked plan's dangling provenance is repaired, every proposed follow-up has a recorded
  outcome, and the combined tree passes proportionate landing verification.
phases:
  - id: contract-budget
    title: Replace the load-sensitive contract runtime oracle
    depends_on: []
    size: medium
    description:
      "contract-budget: replace the still-load-sensitive normalized-child-CPU
      contract-set guard with the deterministic manifest-entry budget explicitly allowed
      by the original contract-set plan, remove the now-unused calibration machinery,
      and prove the guard and its diagnostic remain useful without any real-time or
      CPU-time correctness oracle."
  - id: post-start-integration
    title: Integrate commits that landed after the epic began
    depends_on: []
    size: small
    description:
      "post-start-integration: migrate the post-epic plan-link concurrency waits to the
      shared load-tolerant timeout, re-audit all non-epic commits after 2e9e1a29c for
      wait-helper conflicts, and repair the linked epic plan's dangling parent-plan link
      while leaving its status wip for the landing phase."
  - id: land
    title: Verify, close sase-h8.10, and complete its plan
    depends_on:
      - contract-budget
      - post-start-integration
    size: small
    description:
      "land: verify every child and epic commit on the combined tree, run the
      contention, full, visual, and flake-gate criteria, record every follow-up outcome,
      close sase-h8.10 with a comprehensive note, run post-close symvision and remove
      its findings, then set status done in plans:202608/flake_class_residue.md."
proposed_by: bbugyi200.athena.sase-h8.10.land
parent_bead: sase-h8.10
create_time: 2026-08-08 13:27:08
status: wip
---

- **PARENT:**
  [202608/flake_class_residue.md](https://github.com/sase-org/sase--plans/blob/main/202608/flake_class_residue.md)

# Plan: Finish and land `sase-h8.10`

## Verified starting point

This is residue discovered by the land audit of epic `sase-h8.10`, not a replay of its
completed phases.

- `sase-h8.10.1` landed as `2e9e1a29c`. The production watchdog now accepts an
  injectable monotonic clock and exposes `_poll_once()`. Its F2 tests advance a fake
  clock and assert exact event sequences, while
  `test_resumed_watchdog_records_one_later_real_stall` retains a real-timer path. The
  current watchdog file passes all 17 tests.
- `sase-h8.10.2` landed as `9360e850c`. It introduced
  `tests._load_tolerant.LOAD_TOLERANT_TIMEOUT`, converted the files-loading and
  residual-freeze deadlock diagnostics to that bound, made the XPrompt jump test wait on
  state, and injected the clan-summary timeout process. The targeted residue nodes pass
  on the current tree.
- `sase-h8.10.3` landed as `3c771b77c`. The AST checker scans all `tests/**/*.py` for
  private `_wait_until` helpers, bounded `pilot.pause()` loops with conditional breaks,
  and raw agent-prompt-panel `Text` injection. The checker and its eight tool tests
  pass, and the migrated call sites use supported helpers.
- `sase-h8.10.4` ran the parent epic's four exit criteria at `9360e850c`. The visual
  lane passed. Its contention run retained three exact tally artifacts:
  `file:explicit:c163965096076ddf2cb31881`, `file:explicit:2936172d6b360805316b93fb`,
  and `file:explicit:c127588f8314583ddb8d68b1`.

The current head is `e368d5756`. A focused audit is green:
`tools/check_test_wait_helpers` and 44 tests covering the watchdog, checker, residue
nodes, and XPrompt checks pass; the five newly observed contention nodes pass together
in isolation (5 passed in 3.63s). `SASE_TEST_SELECTION_HEALTH_DISABLED=1` also lets the
contract-budget node pass alone. A direct quiet measurement at this head reports 35
manifest files, 27.4 normalized seconds against the 35-second ceiling, 25.8 seconds raw
wall, 25.9 seconds child CPU, and calibration probes of 0.884/0.889 seconds against the
0.940-second baseline. The same node failed 1/3 full contention repeats, so the
synthetic probe still under-corrects for real xdist contention. It remains in this
epic's explicit `clock` scope and must be fixed here.

## Phase `contract-budget`: deterministic contract-set budget

The original contract-set plan permits a manifest-entry cap when a runtime assertion is
not reliable. Use that fallback now instead of tuning a third load calibration or
raising the ceiling again.

1. In `tests/test_contract_manifest.py`, replace
   `test_contract_set_serial_runtime_stays_within_budget` with a deterministic guard
   over the committed entries in `tests/contract_manifest.txt`. Calibrate the cap from
   the measured current set and preserve the original "err small" policy: adding a
   contract file should require an explicit value-per-second curation decision rather
   than consume invisible headroom. The failure message must point maintainers to the
   curation procedure in `plans:202608/test_suite_tier1.md` and state the current
   measured serial cost.
2. Remove `tests/_test_contract_budget.py` and
   `tests/test_contract_budget_normalization.py` if they have no remaining caller. Do
   not leave the synthetic probe, `resource` helpers, or historical normalization tests
   as unused machinery. Update module comments to explain why a deterministic
   manifest-entry budget replaced the flaky runtime oracle.
3. Do not increase a time ceiling and do not use wall time, process CPU time, load
   average, or another host-state measurement as the correctness assertion. The
   production contract selection stays unchanged.
4. Verify the manifest drift guard and new budget guard in isolation. Run
   `SASE_CONTENTION_REPEAT=6 just test-contention -- tests/test_contract_manifest.py`
   and require a zero-failure tally. Run `just check`; because this phase changes
   contract-selection tooling/tests, use `just check-full` as well if the selector
   broadens or escalates.

Acceptance: the contract budget still prevents unreviewed growth, the test no longer
executes a nested timed pytest or calibration probe, six contention repeats are green,
and no normalization-only code remains.

## Phase `post-start-integration`: later-commit adoption and plan provenance

The first epic commit is `2e9e1a29c`. The non-epic commits after it on the current
branch are `72ec6aa3a`, `e8028151e`, `f4dfc2626`, `e62959e88`, `125b5c31b`, `4bf78056c`,
and `e368d5756`.

1. `e62959e88` changed fixed 5/10-second waits in
   `tests/test_bead/test_cli_work_from_plan_concurrency.py` to a file-local
   `_CONCURRENCY_TIMEOUT_SECONDS = 10.0`. Those waits are deadlock diagnostics over
   background threads and futures, the exact contract now centralized by
   `tests._load_tolerant.LOAD_TOLERANT_TIMEOUT`. Import the shared value, replace the
   local constant's uses, and remove the duplicate constant. Keep the 0.2-second
   negative lock-exclusion assertion separate: it is an ordering assertion, not a
   diagnostic wait for eventual progress.
2. Verify the full concurrency test file and rerun `tools/check_test_wait_helpers`.
   Re-scan the other listed commits. `f4dfc2626` adds ordinary one-turn pauses plus a
   `wait_for` state wait and is checker-compliant; `e8028151e` injects `now` in its
   timer tests; `72ec6aa3a`'s deterministic XPrompt assertion regression is already
   repaired by `e368d5756`; the remaining docs/config commits do not duplicate or
   conflict with the epic. Change nothing for those commits unless current source
   inspection disproves these findings.
3. In `plans:202608/flake_class_residue.md`, preserve `parent_bead: sase-h8` but replace
   the Markdown `PARENT` link to nonexistent
   `plans:202608/parallel_suite_flake_class.md` with a durable parent-bead reference.
   Confirm with the plans repository history that the target plan file still does not
   exist. Leave this plan's `status` as `wip`; only the final landing phase changes it.
4. Run `just check` for the main-repository change and validate the linked plan through
   the normal SASE validation path.

Acceptance: all post-start commits have an explicit integration disposition, the later
concurrency fix uses the epic's shared load-tolerant convention, the wait-helper checker
remains green, and the plan has no dangling provenance link.

## Follow-up outcomes already established

The final close note must preserve these outcomes and must not create duplicate tasks.

- `sase-h8.10.4`'s XPrompt regression proposal is resolved by unrelated later commit
  `e368d5756`; current `tests/test_bead_xprompt_tags.py` passes. Decline a task as
  already fixed. `sase-hl` separately and correctly tracks the flake classifier's
  deterministic-break blind spot.
- The contract-budget proposal is caused by and explicitly scoped to this epic. The
  `contract-budget` phase resolves it here; do not file a task.
- The five 1/3 contention nodes are not caused by this epic. `/sase_new_task` triage
  found them to be semantic duplicates of in-progress umbrella task `sase-ct` and
  causally owned by active epic `sase-h8`. The land audit added one `+1` to `sase-ct`
  and a `DISCOVERED ISSUE` note to `sase-h8`, referencing all three tally artifacts. No
  new task is warranted.
- The dangling plan-provenance proposal was authored by this epic and is repaired in
  `post-start-integration`; do not file a task.
- Earlier parent-plan candidates already became ready tasks: `sase-hk` covers the
  swallowed XPrompt-selector raiser, and `sase-hl` covers deterministic breaks being
  misclassified as flakes.

## Phase `land`: final verification and closure

1. Re-run `sase bead show sase-h8.10` and all four child shows. Confirm every child is
   closed, every note above has an outcome, and the three original epic commits plus the
   remediation commits are present in the current source. Check current branch history
   once more for commits after `2e9e1a29c` that landed while these phases ran and
   integrate any additional missed adoption before continuing.
2. Run the combined-tree verification:
   - `just test-contention` at the configured baseline and repeat count; inspect the
     generated tally and require no failure caused by `sase-h8.10`.
   - `just check-full`.
   - `just test-visual`.
   - the flake gate used by the parent plan against
     `tests/reproducible_flake_baseline.txt`, confirming it evaluates real post-baseline
     full-run records.
   - `tools/check_test_wait_helpers` and the focused watchdog, residue, checker,
     concurrency, contract-manifest, and XPrompt tests.
3. If a new failure is caused by this epic, keep it in scope and fix it before close.
   For a genuinely distinct unrelated failure, use `/sase_new_task` and record its
   duplicate, active-epic, new-task, or declined outcome. Do not add another report for
   the five nodes already routed above unless there is materially new evidence.
4. Close only this child epic with
   `sase bead close sase-h8.10 --note "<comprehensive verification and integration summary>"`.
   The note must name commits, tests/criteria, post-start integration, all follow-up
   dispositions, and any unrelated remaining parent-epic work. Do not force closure. Do
   not close `sase-h8` or `sase-ct` merely as part of this child landing; they own the
   unrelated flake-class residue recorded above.
5. After the close succeeds, run `just symvision`. Remove expired `sase-h8.10` whitelist
   entries and unused code it reports, then run `just check` for any resulting
   main-repository changes (use `just check-full` if the change hits the broadening
   set). Finally set `status: done` in the frontmatter of
   `plans:202608/flake_class_residue.md`. The status edit happens last, after the close
   and post-close cleanup.

Acceptance: `sase-h8.10` is closed without force with an evidence-rich note, post-close
Symvision is green, its linked plan is `status: done`, and the still-open parent
task/epic accurately retain unrelated residue rather than hiding it inside this closure.
