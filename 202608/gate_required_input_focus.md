---
tier: tale
title: Focus required gate inputs after an invalid decision shortcut
goal:
  ACE focuses the first actionable invalid required input when a reviewer activates a
  gate branch that cannot yet submit, without emitting the generic invalid-input toast
  or changing valid submission behavior.
proposed_by: bbugyi200.athena.vo
create_time: 2026-08-08 10:16:50
status: wip
---

# Focus required gate input when a decision shortcut cannot submit

## Goal

When an ACE reviewer activates a gate branch whose selected option still needs input,
keep the gate pending and move keyboard focus to the first actionable invalid input for
that branch. In the task-triage example, pressing `3` for **Snooze** must focus the
required **Wake after** enum instead of raising the generic “Fix the highlighted inputs
before submitting” toast. Once the inputs are valid, the same decision action must
submit normally.

This is one shared ACE interaction change. It applies to custom, task-triage, bead
snooze, plan, and epic review modals through `GateBranchControls`; it does not change
the gate wire format, validation rules, option commands, or any non-ACE surface.

## Current behavior and root cause

- `GateBranchControls` disables a singleton/group submit button while its visible
  `GateBranchInputSection` is invalid. Numbered shortcuts, the primary-action binding,
  and active-branch submission intentionally call `_resolve_branch()` directly, so a
  keyboard user can still choose a branch without first navigating to its button.
- `_resolve_branch()` already gives missing required feedback an actionable response: it
  focuses `#gate-feedback-input`. The declared-input failure path only emits the warning
  toast and returns, leaving focus on the prior decision control.
- `TypedInputForm` knows field declaration order, visibility, requiredness, and typed
  validity, but exposes only `focus_first()`, which can select a valid or optional
  field. `GateBranchInputSection` knows which raw-schema YAML editors are visible and
  invalid, but exposes no focus helper. The resolution layer therefore has no safe way
  to locate the control the reviewer must fix.
- The screenshot case is the generic path, not a task-triage special case: Snooze is the
  third singleton branch, and its declared enum starts without a submitted value.
  Pressing `3` reaches `_resolve_branch(2)`, detects the invalid form, and takes the
  toast-only branch.

## Interaction contract

1. Resolve and activate the requested branch exactly as today; do not change selected
   AND members, typed values, feedback state, or primary/numbered shortcut semantics.
2. If the selected branch has invalid visible declared inputs, focus the first visible
   missing or invalid **required** typed field in declaration/render order. This makes
   the common empty required enum/line case deterministic.
3. If all required typed fields are valid but another visible typed field is invalid,
   focus the first such field. If the branch uses the raw-schema escape hatch, focus the
   first visible invalid YAML editor. Hidden inputs belonging only to deselected AND
   members are never candidates.
4. When an actionable invalid control is focused, return without posting a warning
   toast. `Widget.focus()` should perform the normal scroll-into-view behavior so a
   field below the fold becomes immediately editable.
5. Retain a defensive warning when branch validity is false but no focusable invalid
   control can be found. Keep the existing conflict, empty-selection, required-feedback,
   draft-block, and submission-error behaviors unchanged; those are distinct states with
   their own reviewer guidance.
6. A branch whose defaults/current values are already valid submits immediately. The new
   focus path must never emit `Resolved`, dismiss a modal, or execute a gate command.

## Implementation

### 1. Let the typed form focus its own invalid field

In `src/sase/ace/tui/widgets/typed_input_form.py`, add a small public
`focus_first_invalid() -> bool` helper beside the existing focus methods.

- Consider only fields whose `_visible` entry is true.
- First scan invalid required fields in form order, then scan any remaining invalid
  visible field in form order. Reuse `_field_ok()` and `focus_field()` so conversion,
  enum sentinel handling, secret inputs, path inputs, and Vim-mode display updates stay
  centralized.
- Return `True` only when focus moved; return `False` when the form has no invalid
  visible control. Keep `focus_first()` and the input-collection launch modal behavior
  unchanged.

This is an additive widget API. It must not introduce I/O, timers, or `call_later` work
on the keystroke path.

### 2. Give each branch input section one focus target

In `src/sase/ace/tui/modals/gate_branch_input_section.py`, add
`focus_first_invalid() -> bool` as the host-facing counterpart to `is_valid()` and
`collect()`.

- Ask the nested `TypedInputForm` first, preserving the rendered typed-form-before-raw
  order and its required-first policy.
- If no typed field needs attention, walk the raw options in render order and focus the
  first editor that is both visible in `_visible_option_ids` and false in `_raw_valid`.
- Return `False` for an empty/valid section or for a non-actionable conflict. Conflicts
  continue through the existing pointed warning rather than focusing an arbitrary
  widget.

Keep ownership at this layer: `GateBranchControls` should not inspect private form
fields, reconstruct field IDs, or duplicate raw-schema validity rules.

### 3. Replace the normal invalid-input toast with focus

In `src/sase/ace/tui/modals/gate_branch_controls.py`, change only the invalid branch-
input arm of `_resolve_branch()`:

- after `_set_active_branch()` and selected-option/feedback checks, call the active
  section's `focus_first_invalid()` when `_branch_inputs_valid()` is false;
- if it returns `True`, stop resolution silently with the gate still pending;
- if it returns `False`, retain the current warning as a defensive fallback so an
  unexpected invalid state never becomes a silent dead end.

Do not make the numbered shortcut preselect a value, do not re-enable invalid submit
buttons, and do not route this through modal-specific action methods. Keeping the
behavior in `_resolve_branch()` ensures mouse/button activation, numbered branches, the
configured primary key, and active-branch submission converge on one rule in both shared
gate modals.

## Regression coverage

Extend the existing focused ACE tests rather than adding a kind-specific implementation
fixture.

1. In `tests/ace/tui/test_typed_input_form.py`, cover the new helper directly:
   - it focuses the first invalid required visible field even when an earlier optional
     field is also invalid;
   - after required fields are valid, it focuses the first remaining invalid optional
     field;
   - it returns false and does not disturb focus when every visible field is valid;
   - hidden fields are skipped.
2. In `tests/ace/tui/test_custom_gate_modal.py`, strengthen the numbered-shortcut
   regression to mirror the reported interaction: use at least three branches, give the
   third branch a required enum followed by an optional line field, press `3` from
   another decision control, and assert that:
   - no modal result is produced;
   - the required enum has focus (and is scrolled into view by normal focus behavior);
   - the generic warning toast is not emitted;
   - choosing a value and activating the branch again submits the third option with its
     typed `option_inputs`. Preserve the existing tests proving numbered shortcuts
     select canonical branches independently of current focus and valid branches submit
     immediately.
3. In `tests/ace/tui/test_gate_branch_inputs.py`, cover section-level edge cases:
   - for an AND branch, a required field belonging only to a deselected member remains
     hidden and is skipped in favor of the selected member's first invalid field;
   - an invalid raw-schema YAML editor is focused when it is the actionable invalid
     control;
   - collecting and distributing values after correction is unchanged.
4. Keep the existing required-feedback focus test and action/control traversal tests in
   the focused run. They guard the neighboring behavior and ensure the new field focus
   does not alter feedback handling or the `j`/`k` focus ring.

No CSS, footer/help text, or PNG golden update is expected: the DOM and rendered
controls do not change, only focus after an attempted invalid submission. If a golden
changes, inspect it as an unintended regression rather than accepting it automatically.

## Verification

1. Run `just install` first, as required in an ephemeral SASE workspace.
2. Run the focused interaction suite:

   ```bash
   just test \
     tests/ace/tui/test_typed_input_form.py \
     tests/ace/tui/test_gate_branch_inputs.py \
     tests/ace/tui/test_custom_gate_modal.py \
     tests/ace/tui/test_gate_action_controls.py \
     tests/ace/tui/test_notification_plan_gate.py
   ```

3. Run the existing custom-gate input snapshot without snapshot-update mode to confirm
   the static layout is unchanged:

   ```bash
   just test-visual \
     tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py::test_custom_gate_inputs_section_png_snapshot
   ```

4. Run the repository-mandated `just check`. Because this is a local presentation and
   focus-state change, use the scoped lane selected by `just check`; escalate to
   `just check-full` only if selection broadens or reports an unusual result.

## Acceptance criteria

- Pressing `3` for a Snooze-like third branch with an empty required **Wake after** enum
  leaves the gate pending, emits no generic invalid-input toast, focuses that enum, and
  lets the reviewer choose its value immediately.
- Multiple required fields are visited deterministically by correction need: the first
  visible invalid required field wins, then any other invalid typed field; deselected
  AND-member fields never steal focus.
- Raw-schema-only branches focus their invalid YAML editor rather than becoming a
  toast-only dead end.
- Correcting the focused input and activating the same branch submits the same option
  IDs and per-option values as before; already-valid branches still submit on the first
  activation.
- Required feedback, conflicts, empty selections, draft blocks, action traversal, and
  gate execution semantics are unchanged, and all focused tests, the unchanged visual
  snapshot, and `just check` pass.
