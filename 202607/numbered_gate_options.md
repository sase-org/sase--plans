---
tier: tale
title: Numbered notification-gate branch shortcuts
goal: Every top-level OR branch in an ACE gate review is visibly numbered and directly
  triggered by its matching digit, while AND-member toggles and Cancel keep their
  existing controls.
create_time: 2026-07-23 13:01:46
status: wip
---

- **PROMPT:** [202607/prompts/numbered_gate_options.md](prompts/numbered_gate_options.md)

# Number top-level gate branches and bind their digit shortcuts

## Goal

Make every actionable top-level OR branch in an ACE notification-gate review visibly numbered in canonical branch order,
and let the matching digit key submit that branch directly. The modal-level `Cancel` control remains unnumbered and
continues to use `q`/Escape. An AND branch receives one number on its group action (`Tale`, for example); its individual
optional command toggles remain unnumbered and continue to use focus plus Space.

For the plan gates shown in the request, this yields:

- Epic review: `1` Epic, `2` Reject, `3` Send Feedback.
- Tale review: `1` Tale, `2` Reject, `3` Send Feedback; the launch-coder and commit-plan members inside Tale do not
  receive selectors.

The same branch-order behavior applies to generic custom gates, including singleton branches and configured AND groups.

## Implementation

1. Extend the shared branch control in `src/sase/ace/tui/modals/gate_branch_controls.py` so the ordered
   `GateBranchData.branches` projection is the single source of truth for both display numbers and shortcut targets.
   Prefix singleton button labels and both the collapsed and submit labels for an AND group with the branch's one-based
   digit. Do not prefix `gate-option-*` toggle labels, feedback inputs, or the separate modal Cancel button. Add a small
   public indexed-submit method that bounds-checks the requested branch and routes through the existing
   `_resolve_branch()` path, thereby preserving selected AND members, feedback validation/focus, and the existing
   `Resolved` message contract.

2. Add a shared builder for scoped, hidden Textual digit bindings for the supported one-key selector range and install
   those bindings in both `PlanApprovalModal` and `CustomGateModal`, including each modal's instance-local
   `BindingsMap`. Route the common `submit_numbered_branch(index)` action to the branch control's indexed-submit method.
   Keep these ordinal selectors fixed rather than adding configurable `GateModalKeymaps` fields: the displayed number
   and its key must never diverge, and the bindings are local to an open gate modal. Do not give the bindings priority
   over the feedback `Input`, so digits remain typeable after a feedback branch has activated its field. Existing
   Enter-primary, Ctrl+S-active, j/k navigation, Space toggling, and `q`/Escape cancellation remain intact.

3. Update focused interaction coverage in `tests/ace/tui/test_custom_gate_modal.py`,
   `tests/test_plan_approval_modal_title.py`, and/or `tests/ace/tui/test_notification_plan_gate.py` to prove the shared
   semantics end to end:
   - singleton and AND-group action labels receive consecutive branch numbers while nested toggles and Cancel do not;
   - `1`, `2`, and `3` dispatch the corresponding branch independent of current focus;
   - a numbered AND branch submits its current selected subset rather than resetting defaults or assigning shortcuts to
     its members;
   - epic and tale plan gates both map branch 1 correctly to their tier-specific primary action;
   - a required-feedback shortcut activates validation and focuses the feedback field instead of bypassing it, and
     digits can still be entered into that field before Enter/Ctrl+S completes the branch;
   - `q` still cancels without producing a gate result, and unassigned/out-of-range digits are harmless.

4. Update `tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py` expectations and intentionally regenerate the
   affected gate-review PNG goldens under `tests/ace/tui/visual/snapshots/png/`. Cover at least the epic plan, tale plan
   with its expanded AND controls, custom singleton choices, and custom AND-group layouts so visual regression tests
   lock in numbering only at the branch level. Inspect the generated actual/expected/diff artifacts before accepting the
   new goldens.

5. Update the ACE gate-review documentation in `docs/notifications.md` (and the matching configuration/keymap
   description if needed) to state that top-level OR branches have fixed one-based digit selectors, AND members remain
   Space-toggled, Enter submits the primary branch, Ctrl+S submits the active branch, and `q` cancels the modal.

## Validation

1. Run the focused non-visual gate modal, plan-gate, keymap, and notification tests covering the files above.
2. Run `just test-visual` and verify the intentional gate snapshot updates while ensuring unrelated PNG snapshots stay
   unchanged.
3. Run the repository-required `just check` after `just install`, fixing any formatting, lint, typing, unit, or visual
   failures before handoff.

## Acceptance Criteria

- The screenshot's Epic gate renders and dispatches `1` Epic, `2` Reject, and `3` Send Feedback.
- A tale plan gate renders and dispatches `1` Tale, `2` Reject, and `3` Send Feedback.
- Every displayed number is derived from the top-level OR-branch order, and no AND-member toggle receives a number or
  digit binding.
- Numeric dispatch uses the same selection and feedback-validation path as clicking/submitting the corresponding branch.
- The separate Cancel action remains unnumbered and mapped to `q`/Escape.
- Plan and custom gate behavior, feedback typing, and the updated visual layout are protected by automated tests.
