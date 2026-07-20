---
tier: tale
title: Finish and land plan-lane clan summaries
goal: 'Long absolute plan references preserve a contiguous filename in bounded clan
  summaries, and epic sase-8d is fully verified, integrated, closed, cleaned by Symvision,
  and marked done in its authored plan.

  '
create_time: 2026-07-20 17:12:22
status: wip
prompt: 202607/prompts/land_sase_8d.md
---

# Plan: Finish and land plan-lane clan summaries

## Context

The landing audit for epic `sase-8d` confirmed that its three intended layers are present in the current source:

- `src/sase/sdd/plan_display.py` owns validated plan loading and shared logical/width-aware Rich rendering, while the
  ACE PLAN lane delegates to it through the associated-plan model.
- `summary_script=` accepts quoted argv without invoking a shell, `sase_clan_summary_plan` renders generic tale or epic
  plans, launch environment variables reach the subprocess, and the public documentation describes the contract.
- `sase_clan_summary_epic` resolves `SASE_EPIC_PLAN_REF` from the launch workspace and primary checkout before using the
  legacy bead-store and identity fallbacks; its focused, smoke, byte-budget, and clan-panel visual coverage is present.

The shared-renderer phase was delivered in the `sase-8d.2` commit together with generic machinery, and the plan-first
epic wiring was delivered in `sase-8d.3`. Reviewing every non-epic commit after the first epic commit found no missed
consumer migration: the associated-plan model split retained delegation into `plan_display`, while subsequent receipt,
runner-slot, family-retry, dashboard, and launch-claim changes do not duplicate or conflict with launch-time plan
summary rendering.

Focused verification nevertheless found a remaining width-dependent regression. When a valid absolute plan path is long
enough, the shared fixed-width renderer folds the basename itself (for example, `ab` and `solute.md` on separate lines).
The authored `sase-8d.3` test requires the filename to remain contiguous, so the suite currently reports one failure in
`test_epic_summary_resolves_absolute_plan_reference_directly`; all other selected epic, generic, persistence, model,
widget, and smoke tests pass.

## Phase 1: Preserve plan basenames in bounded rendering

Update the shared width-aware path-field rendering so it prefers a line boundary before the basename whenever the
basename fits within the continuation width. Long directory prefixes may still fold, and a basename that is itself wider
than the available cells must still fold without character loss. Keep the logical PLAN text unchanged, preserve all path
styles, missing-path annotations, optional hint prefixes, and the fixed rendered-width contract. Avoid an
epic-script-only workaround: both `sase_clan_summary_plan` and `sase_clan_summary_epic` must receive the correction from
the common display module, and ACE must continue consuming the same canonical field values.

Add deterministic shared-renderer coverage for a long directory plus a basename that fits on one continuation line,
including reconstruction and per-line cell limits. Retain coverage for spaces, missing paths, long individual tokens,
Rich-markup escaping, and UTF-8 byte-budget omission. Re-run the focused suites covering plan display, both summary
scripts, summary persistence, epic launch smoke behavior, associated-plan models, responsive PLAN widgets, and clan
summary smoke behavior. Run `just test-visual` to confirm the clan panel and PLAN lane snapshots remain intentional.

## Final phase: Land epic sase-8d

Recheck the diff and post-fix history review to ensure no later consumer now duplicates or bypasses the shared plan
renderer. Close the still-in-progress child `sase-8d.3` once its implementation and tests pass, then close the parent
with `sase bead close sase-8d`.

Only after the epic closes, load the required Symvision memory guidance and run `just symvision` if the recipe exists.
Remove expired `sase-8d` whitelist entries and any genuinely unused code it reports, rerunning Symvision until clean.
Set `status: done` in the frontmatter of `sase/repos/plans/202607/clan_summary_plan_lane.md`, preserving the rest of the
authored plan. Finish with the repository gate required for source changes (`just check`, after `just install` as
needed), inspect both the primary worktree and the durable plan/bead state, and report the exact verification and
landing results.

## Risks and guardrails

The path fix must not introduce display-only whitespace or zero-width characters into copied logical paths, and it must
not make an overlong basename exceed the requested cell width. Closing the epic before the regression is fixed would
expire Symvision allowances too early, so landing remains strictly last. Do not edit SASE memory files; the Symvision
memory is read-only guidance for this task.
