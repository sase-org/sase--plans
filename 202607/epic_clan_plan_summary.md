---
tier: tale
title: Plan-first epic clan summary rendering
goal: Epic clan declarations render their committed authored plan with the shared
  PLAN-lane presentation before falling back safely to bead-store or identity-only
  summaries.
create_time: 2026-07-20 16:18:43
status: wip
prompt: 202607/prompts/epic_clan_plan_summary.md
---

# Plan: Plan-first epic clan summary rendering

## Context

Epic clan summaries are generated during directive extraction, before a newly assigned workspace is fully prepared. The
existing `sase_clan_summary_epic` script therefore races bead-store refreshes and can persist an identity-only fallback
even though the authored epic plan is already committed and its reference is present in `SASE_EPIC_PLAN_REF`. The shared
plan display and generic plan-summary machinery from the prerequisite phases are now available, so this phase should
make the epic-specific script consume that stable source while keeping launch failure-proof.

## Implementation

Update `src/sase/scripts/sase_clan_summary_epic.py` to attempt a plan-backed document first whenever
`SASE_EPIC_PLAN_REF` is present. Resolve absolute references directly; resolve relative references against the summary
subprocess working directory first and then against the project primary checkout discovered through the existing
project/managed-workspace metadata. Deduplicate candidates, require a readable and valid launch-mode plan through
`load_plan_display`, and retain the original reference as the displayed `Path:` value.

Render a successful plan as a compact `◆ EPIC <id>` identity header followed by the shared width-aware plan fields and
phase blocks, excluding the shared `PLAN` header because the epic identity replaces it. Treat the shared renderer as
authoritative for current PLAN-lane phase styling and wrapping instead of recreating presentation logic. Fit output
under the existing internal UTF-8 budget by retaining the complete intro and only removing whole phase blocks from the
tail, with a styled omission line. The output must remain valid Rich markup and every rendered line must stay within the
76-column summary width.

Preserve the current bead-store renderer without behavioral changes as the second stage. A missing, unreadable, or
invalid plan reference should continue into bead-store loading (including its one-refresh retry); only failure of both
sources should emit diagnostics and the escaped `[bold]EPIC <id>[/]` fallback. Epic plans without a plan reference
therefore continue to work exactly as before.

## Tests and visual coverage

Extend `tests/test_bead/test_clan_summary_epic_script.py` with valid plan-first rendering, argument/environment-driven
path behavior, current-workspace precedence, primary-checkout fallback, invalid-plan-to-bead fallback, identity
fallback, Rich-markup safety, width bounds, and whole-phase byte-budget omission. Keep focused assertions for the legacy
bead-store renderer so its fallback shape remains pinned.

Confirm in the clan-summary launch smoke coverage that the epic plan reference survives directive metadata consumption
and reaches the summary subprocess. Refresh the existing epic clan-panel visual fixture to use the plan-lane-style
persisted summary, assert the title/goal/path and phase metadata/description content, and update only the affected PNG
goldens.

## Validation

Run the focused epic-summary, clan-summary persistence/smoke, plan-display, and clan-panel visual tests first. Inspect
any visual diff artifacts before accepting intentional golden changes. Then run `just install` followed by the required
full `just check`, re-run the focused plan-first tests after any fix, verify the worktree contains only intended
source/test/snapshot changes, close only `sase-8d.3`, and confirm the parent epic remains open.

## Risks and constraints

Plan resolution must not clone, refresh, or write repositories during directive extraction; it only probes the launch
workspace and already-known primary checkout. Rendering remains launch-time work and adds no TUI render-path I/O.
Candidate failures must not bypass the established bead refresh fallback, while error reporting must remain useful when
every source fails. No memory files, generated instruction shims, parent-epic state, or unrelated PLAN-lane behavior are
changed.
