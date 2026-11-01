---
tier: tale
title: Make the TUI import budget tolerate mechanical module splits
goal:
  Catch new eager implementation imports while keeping mechanical module splits within
  an unchanged startup budget.
size: medium
proposed_by: bbugyi200.athena.sase-1qn.2
proposed_with: codex/gpt-6-astra
bead: sase-1qn.2
create_time: 2026-11-01 14:49:49
status: wip
---

- **PARENT:**
  [202611/task_backlog_epic_4.md](https://github.com/sase-org/sase--plans/blob/main/202611/task_backlog_epic_4.md)
- **BEAD:** sase-1qn.2

# Make the TUI import budget tolerate mechanical module splits

Implement phase `sase-1qn.2` of `plan:202611/task_backlog_epic_4.md`, repairing
`sase-1ic`. The final epic decision is `optional_scope = keep_five_seeds`: this tale
covers the import-budget defect only. No memory edits are authorized. Close only
`sase-1qn.2` when its implementation is verified; the parent lander owns `sase-1ic` and
`sase-1qn`.

## Evidence and scope

The phase has no earlier implementation or remaining-work notes. A single coding agent
can complete this bounded developer-tool/test change; it needs no child epic, production
TUI behavior change, Rust binding change, or new CLI option.

At clean planning base `078e3da1d6a784cfc8fd2ab783273246318660c2`, Python 3.14.7:

- The existing five tests in `tests/ace/tui/test_app_import_budget.py` pass in 4.76s.
  `tools/tui_import_closure` measures 2,860 first-party modules, 3,570 total modules,
  and 2.76s import CPU. The strict first-party cap is still 2,861, so the defect has not
  been repaired even though this particular tree currently passes.
- Replaying the reported failing tree `d3d26a5b55` measures 2,863 first-party modules
  against that same cap. Invoking its real startup-budget test in an isolated source
  snapshot exits 1 with `TUI app import loads 2863 sase.* modules against the 2861 cap`;
  no import was introduced by this phase.
- `24561602cc` introduced first-party counting, and `86bec79ca4` raised its cap. Those
  are policy/history anchors, not split-only commits. Use actual split commits for the
  before/after proof:

| Split commit                         | First-party modules before → after | Raw imported source bytes before → after |
| ------------------------------------ | ---------------------------------- | ---------------------------------------- |
| `eef22a0124` (filesystem enrichment) | 2,860 → 2,863                      | 23,092,385 → 23,093,748                  |
| `bffbc8effd` (artifact loader)       | 2,855 → 2,860                      | 23,076,489 → 23,079,665                  |

These closure replays used the same installed dependencies and isolated historical `src`
snapshots. Their import timings include cold snapshot effects and are not CPU budget
calibration data.

A read-only prototype removed import statements, docstrings, and literal `__all__`
assignments, then summed UTF-8 bytes of significant source tokens. Across all changed
production files in each split, it measured:

| Commit                                 | Normalized token bytes before → after | Delta |
| -------------------------------------- | ------------------------------------- | ----- |
| `eef22a0124`                           | 19,318 → 19,606                       | +288  |
| `bffbc8effd`                           | 29,711 → 29,770                       | +59   |
| `362fcb2e7f` (prompt artifact staging) | 15,470 → 15,539                       | +69   |

These last numbers are changed-file measurements, not the whole import closure.
Recompute the whole closure with the implementation before setting its new cap. The
small residuals are source changes accompanying the splits, rather than a per-module
charge. Raw source bytes alone still count duplicated boilerplate; a normalized source
footprint removes that mechanical charge.

## Implementation

1. Add one small, typed developer-only measurement helper under `tools/`, for example
   `tools/_tui_import_footprint.py`. Share its implementation between
   `tools/tui_import_closure` and the import-budget test. Keep it out of the product
   import graph. This is test/developer measurement infrastructure, not a new shared
   backend domain.

   Define the first-party footprint as the sum of UTF-8 byte lengths of significant
   Python tokens from the unique source files of imported `sase` / `sase.*` modules. Use
   `ast` to identify excluded source spans and `tokenize` to measure the original
   source. Do not use `ast.unparse`, whose spelling/formatting can vary by interpreter.
   Ignore whitespace, indentation, newlines, comments, encoding/end markers, import
   statements, actual module/class/function docstrings, and plain assignments of a
   literal string list/tuple to `__all__`. Keep ordinary literals, annotations,
   decorators, function bodies, executable statements, and computed `__all__` values.
   Avoid broader name-based exclusions or directory/facade allowlists.

   Correctly translate AST byte columns and tokenizer character columns for Unicode.
   Excluding an import on a semicolon-separated line must not discard its neighboring
   executable statement. Do not add per-file overhead or count a source twice when two
   module names alias it. Package `__init__.py` files count. Missing, unreadable, or
   unparseable first-party sources must produce an actionable failure instead of
   silently contributing zero; explicitly handle source-backed `.pyc` paths and any
   genuine namespace package discovered during implementation.

2. Keep measuring `sase.ace.tui.app` in a fresh interpreter. Capture import CPU,
   `sys.modules`, and the deferred-module results immediately after the app import. Only
   then load the footprint helper and read/parse sources. Measurement machinery must not
   inflate the captured module counts or the existing startup stopwatch. Resolve sources
   from the modules actually imported, including the historical tree when measuring
   `--diff REV`, never from a guessed current-checkout path.

   Replace the tight `_MAX_SASE_MODULE_COUNT` assertion with a strict `<` normalized
   source-footprint assertion. Keep first-party module counts as diagnostics. Preserve
   the deferred-module list exactly, `_MAX_TOTAL_MODULE_COUNT = 3904`, the 5.0s CPU
   budget, and the existing two-attempt CPU policy.

   Set the new fixed cap from the measured clean implementation baseline plus 4,096
   normalized token bytes. This small absolute allowance covers the measured split
   residuals; it is not a percentage of the 23 MB raw source tree. Record the exact
   baseline, revision, interpreter, metric version, allowance, and resulting cap in the
   test comment. Verify both replayed split pairs pass with that same cap. Do not
   automatically recalibrate from HEAD/merge-base, adjust the cap during a test, or
   increase the allowance if a replay fails: attribute and resolve that discrepancy.

   Explain the policy: split/re-export boilerplate does not consume the budget, while
   additional imported implementation does. Genuine increases require named sources and
   a first-paint justification after considering deferral. Keep strict `<` and the
   existing ratchet discipline; never raise the old module-count cap. Be explicit that
   the small allowance admits small code growth, just as the previous guard had module
   headroom; deferred-module exclusion remains absolute.

3. Extend `tools/tui_import_closure` to report the same normalized footprint along with
   first-party count, total count, and CPU. Its existing `--diff REV` must report signed
   footprint changes and identify source contributors, including changed contents of
   modules already present on both sides. Keep the existing added/removed module
   diagnostics. No new CLI option or budget-update command is needed. Reuse the
   measurement logic instead of maintaining separate probe definitions.

4. Correct only the existing “TUI import-budget ratchet” section in
   `docs/contribute/perf-runbook.md` and the tool/test descriptions to explain the new
   unit, allowance, attribution output, and retained guards. Follow the docs-writing
   memory and `docs/contribute/writing-docs.md`. Do not edit memory or unrelated docs.

## Regression proof and verification

Extend `tests/ace/tui/test_app_import_budget.py` and `tests/test_tui_import_closure.py`;
place focused helper cases in a separate test module if necessary to stay within the
repository's file-size limits.

- Use real temporary source packages and fresh subprocess imports to prove a module
  containing two functions can be split into a facade and helper files without changing
  its normalized footprint or failing an unchanged budget. Include repeated import
  boilerplate and docstrings. Do not substitute fabricated module-name lists for the
  end-to-end measurement proof.
- Import an unrelated fixture module that was previously outside the closure and assert
  that its real source is counted and trips an unchanged fixture cap. Also exercise
  growth larger than the production allowance so the 4,096-byte allowance cannot
  accidentally turn the positive control green. Include a previously deferred module
  case that fails independently of size. Assert the useful diagnostic names the source
  or deferred module responsible.
- Cover third-party growth remaining outside the first-party metric, strict cap equality
  failing, comments/formatting/docstrings being neutral, executable code on an import's
  line still counting, Unicode offsets, aliases counted once, empty facades,
  missing-source failure, and computed export expressions still counting. Keep
  assertions focused on actual measurement and failure behavior.
- Test diagnostic output for footprint growth inside an existing imported module,
  added/removed modules, and an unchanged footprint despite a higher module count.
- Reproduce `d3d26a5b55` with the old test, then use the final helper to measure both
  sides of `eef22a0124` and `bffbc8effd`. Use isolated snapshots/worktrees inside the
  current workspace and the same interpreter/dependencies; remove scratch snapshots.
  Record actual module and footprint deltas and results under one unchanged cap.
  Historical compatibility errors are not successful replay evidence: diagnose them or
  use another verified split-only commit and document why.
- Run the focused tests and current real TUI import guard. Inspect measurement overhead
  separately from app import CPU; avoid redundant full-closure parsing in the small
  fixture tests. Run `just fix`, then `sase tool run check`. Do not run
  `just check-full`. If final verification needs a monitor, use `/sase_monitor` and wait
  for its handoff command to exit.

Acceptance requires the current real app guard passing, both replayed mechanical splits
passing without cap edits, a new eager source import tripping the footprint guard, a
deferred import still failing, and unchanged CPU/loose-total protections.

## Phase completion

Record measurements, command results, and any limitations on `sase-1qn.2`, including
`TASK RESOLVED: sase-1ic — <implementation and verification evidence>` when complete.
Record unrelated discoveries only with
`sase bead note sase-1qn.2 'PROPOSED FOLLOW-UP: <summary — detail>'`; do not create or
close other beads. A check failure reproduced identically on the clean base is a
follow-up with evidence, not a reason to leave the completed phase open.

Run `sase bead epic-symbols sase-1qn.2` immediately before closing. Resolve any entries
or re-key them to a still-open owner. The planning check found none. Then close only
`sase-1qn.2` with `sase bead close sase-1qn.2 --note '<what was verified>'` and finish
through `/sase_final`. Never close the parent epic or its source task on this phase's
authority.
