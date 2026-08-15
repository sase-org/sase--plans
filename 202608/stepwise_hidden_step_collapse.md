---
tier: tale
title: Restore stepwise H collapse for hidden workflow steps
goal:
  Make uppercase H reverse a selected workflow or family lane's l expansions one level
  at a time before applying group-wide collapse operations.
size: medium
proposed_by: bbugyi200.athena.02l
create_time: 2026-08-15 13:52:00
status: done
---

- **PROMPT:**
  [prompts/202608/stepwise_hidden_step_collapse.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/stepwise_hidden_step_collapse.md)

# Restore stepwise `H` collapse for hidden workflow steps

## Context and root cause

Agent-family and standalone-workflow lanes use one `FoldStateManager` ladder. The first
`l` moves a lane from `COLLAPSED` to `EXPANDED`, revealing ordinary descendants, and the
second moves it to `FULLY_EXPANDED`, revealing hidden workflow steps. The fold manager
already has the correct inverse transition: `FULLY_EXPANDED -> EXPANDED -> COLLAPSED`.

The Agents-tab uppercase-`H` dispatcher currently checks the group-wide SASE-agent
collapse target before the selected structural target. That bulk path calls
`collapse_fully_all()`, so selecting a visible hidden step and pressing `H` drives its
family/workflow lane directly from `FULLY_EXPANDED` to `COLLAPSED`. It bypasses the
existing one-level structural collapse, hides ordinary family members together with the
hidden steps, and makes the footer advertise `collapse sase agents` instead of the
selected family/workflow operation.

## Intended behavior

- With row focus inside an open workflow or sequential-family lane, `H` first owns the
  selected lane and retreats it by exactly one fold level. From a visible hidden step,
  the first press must hide hidden steps while leaving ordinary descendants visible;
  because the selected row disappears, selection must re-anchor to the lane owner. The
  second press must collapse that still-selected lane.
- After the selected lane reaches `COLLAPSED`, later `H` presses may continue through
  the existing group-wide remaining-lane, selected-clan, remaining-clans, grouping-
  banner, and panel behavior. Preserve the existing Tools-detail and whole-panel hint
  precedence. Preserve the binary clan fold semantics rather than treating clans as
  two-level hidden-step lanes.
- The footer, help, keymap metadata, default-configuration commentary, and user
  documentation must describe the same precedence as the action that will run.
- Fold changes must remain an in-memory, single-refilter keystroke path with no disk,
  subprocess, or content-index work.

## Implementation

1. In the Agents folding action routing, resolve the selected structural collapse target
   before the group-wide lane target. If the selected target is a workflow or family,
   apply that already-defined one-level transition and return. Keep clans on the
   established lane-before-clan ladder, then retain the current group-wide lane,
   selected-clan, remaining-clans, structural fallback, and grouping-banner order. Reuse
   a resolved target rather than probing mutable selection twice, retain the
   hidden-child re-anchor to the canonical owner, remember the resulting selection, and
   repaint once through the fold-only fast path (`refresh_content_index=False`). Do not
   change `FoldStateManager` or hidden-row filtering semantics; they already implement
   the required levels.

2. Make contextual footer capability resolution mirror the dispatcher. An open selected
   workflow/family must expose its structural collapse kind before the group-wide
   SASE-agent capability; once it is collapsed, the existing bulk-lane, clan, group,
   panel, and Tools labels resume. Update the footer computation tests, deferred-detail
   resolver tests, command/help metadata, and default keymap commentary so no surface
   promises the old saturating-first behavior.

3. Strengthen transition coverage around the exact regression and the surrounding
   ladder:
   - Exercise a loader-shaped plan family containing ordinary descendants and a hidden
     Python/pre-prompt step. Starting at `FULLY_EXPANDED` with the hidden step selected,
     assert that the first `H` lands at `EXPANDED`, re-anchors to the family owner,
     removes only hidden rows, and leaves ordinary descendants visible; assert that the
     second `H` lands at `COLLAPSED`.
   - Cover the same one-level reversal for a standalone workflow lane, and confirm the
     footer changes from selected `collapse workflow`/`collapse family` to the next
     group-scoped action only after that lane is closed.
   - Adjust the group-wide lane tests so they still prove that other open canonical
     lanes in the resolved group collapse together when the selected row does not own an
     open workflow/family fold, while preserving panel isolation, merged-layout,
     malformed-owner fail-closed behavior, and clan precedence.

4. Extend the existing ACE family-navigation visual regression that already renders a
   hidden Python setup step. After the expanded-state assertion, press `H` and verify
   the setup row disappears, the family remains expanded with its ordinary rows, the
   selection is on the family owner, and the footer advertises the next selected-family
   collapse. Capture a new intentional PNG golden for this intermediate state, then
   press `H` again and assert the family collapses. Update `docs/ace.md` and
   `docs/agent_families.md` to document the selected-lane-first reversal followed by the
   existing broader collapse ladder.

## Verification

1. Run `just install` before repository checks in the ephemeral workspace.
2. Run the focused non-visual fold-transition, footer, and deferred-detail test files,
   including the new loader-shaped family and standalone-workflow cases.
3. Regenerate only the intentional family hidden-step PNG with
   `just test-visual -- <target-test> --sase-update-visual-snapshots`, inspect the
   resulting intermediate image, and rerun that visual test without the update flag to
   prove exact equality.
4. Run `just check` for whole-repository formatting/lint gates and the diff-scoped test
   lane. If scoped selection escalates or reports unusual coverage, follow project
   guidance and run `just check-full` through `/sase_monitor` with a follow-up action.

## Acceptance criteria

- In the supplied family-with-hidden-steps state, one `H` hides hidden steps without
  collapsing the family, and the next `H` collapses the family.
- The disappearing hidden-row selection re-anchors deterministically to the canonical
  family/workflow owner, with remembered panel selection and footer state updated.
- Group-wide lane collapse, clan/group/panel ladders, merged and split panel isolation,
  Tools priority, and malformed-tree safety retain regression coverage.
- Documentation, help text, keymap commentary, footer labels, unit tests, the visual
  golden, and `just check` all agree with and pass for the new precedence.
