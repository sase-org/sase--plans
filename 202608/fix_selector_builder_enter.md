---
tier: tale
title: Restore Enter confirmation in the model selector builder
goal:
  Pressing Enter in the focused selector-builder list confirms valid model pool and
  fallback expressions through the existing edit flow.
size: small
proposed_by: bbugyi200.athena.01h
create_time: 2026-08-14 13:21:13
status: done
---

- **PROMPT:**
  [prompts/202608/fix_selector_builder_enter.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/fix_selector_builder_enter.md)
- **AGENTS:**
  - [bbugyi200.athena.01h](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.01h.md)
- **COMMITS:**
  - [ecb5c93](https://github.com/sase-org/sase/commit/ecb5c939df060286609a659680b42ef494ebba41)
    — fix(ace): confirm selector builder on Enter when list is focused

# Restore Enter confirmation in the model selector builder

## Context and root cause

The `sase-lz.3` epic phase added `SelectorBuilderModal` for guided model-pool and
fallback authoring. The modal focuses its child `OptionList` on mount and declares an
`enter -> confirm` screen binding, but Textual's focused `OptionList` consumes Enter
through its own selection action and posts `OptionList.OptionSelected`. The builder has
no handler for that message, so pressing Enter produces no visible result and never
reaches `action_confirm()`.

This differs from the neighboring `DefaultEffortLevelModal`, which has the same
modal-level Enter binding but also handles `OptionList.OptionSelected`, stops the
message, and delegates to its confirmation action. The builder's existing confirmation
tests invoke `action_confirm()` directly, so they verify expression validation and
dismissal without exercising the broken keyboard dispatch path.

## Implementation

1. Update `SelectorBuilderModal` in
   `src/sase/ace/tui/modals/models_panel_selector_builder.py` to handle
   `OptionList.OptionSelected`, stop the handled event, and delegate to
   `action_confirm()`. Keep the existing confirmation action as the single source of
   truth for minimum-member and selector validation checks, and retain the screen-level
   Enter binding for cases where focus is not owned by the list.
2. Add a regression test in `tests/test_models_panel_selector_builder.py` that mounts a
   valid builder and drives the real keyboard path with `pilot.press("enter")`, then
   asserts that the modal callback receives the normalized selector expression. This
   must fail against the current implementation and pass only when the focused
   `OptionList` selection message is bridged to confirmation. Preserve the existing
   direct-action tests for invalid and undersized selectors.

## Verification

1. Run `just install` before project commands because this numbered workspace may have
   stale or missing editable dependencies.
2. Run the focused selector-builder tests, including the new keyboard regression:
   `pytest tests/test_models_panel_selector_builder.py`.
3. Run `just check`, as required for every source change in this repository, and fix any
   failures attributable to this change. The change is event wiring only and does not
   alter rendering, selector semantics, the configured keymap, or the broadening set, so
   visual snapshots and `just check-full` are not expected unless scoped verification
   escalates or reports an unusual selection.

## Expected outcome

With at least two valid members, pressing Enter while the builder's member list is
focused dismisses the builder with the composed pool or fallback expression and
continues into the existing preview/write flow. Invalid or undersized builders remain
open under the same validation rules as before.
