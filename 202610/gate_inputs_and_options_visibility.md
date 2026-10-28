---
tier: tale
title: Make every gate input and gate option reachable in the ACE gate UI
goal:
  Reviewers can see, reach, and fill every declared gate input and every gate decision
  option from the TUI.
size: medium
decisions:
  body_fit:
    ask:
      Grow the gate input panel body to fit the shell instead of the fixed 32-row cap?
    choices:
      fit:
        Fit body to shell on mount and resize; tall terminals show more fields, footer
        never clips
      cap:
        Keep the fixed 32-row cap; smallest diff, but the footer still clips under ~46
        rows
    default: fit
    why:
      The reported 80-row terminal left half the modal unused; the cap clips short ones
    answer: fit
proposed_by: bbugyi200.athena.15a
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 14:50:58
status: wip
---

# Make every gate input and gate option reachable in the ACE gate UI

## Problem

On the custom gate `agents-history-purge`, the reviewer could neither reach most of the
typed inputs nor the gate's other decisions.

- **Input panel.** Branch `attest AND approve` opens `GateInputPanel` for 12 shared
  inputs: 9 required 5-choice enums, a required bool, an optional text field, and an
  optional secret text field. The panel showed only F-01..F-06, F-14, F-09..F-13 and the
  F-15..F-20 header. The 6 other required fields, the bool, and both text fields never
  appeared, and there was no scrollbar. The panel reported "4 of 10 required filled", so
  the branch could not be submitted from the TUI. Below the footer hint, the modal was
  empty space.
- **Decision rail.** In the `CustomGateModal` behind the panel, the left rail cuts off
  after branch `1 Attest and run fail-closed preflight`. Branches 2–4 (`rotation_only`,
  `defer`, `reject`) sit below the fold. j/k moves focus onto them, but they never
  scroll into view.

## Root causes (reproduced headlessly with the real `styles.tcss`)

### 1. `GateInputSection` inherits Textual's `Vertical` defaults

`GateInputSection` (`src/sase/ace/tui/modals/gate_input_panel_sections.py`) subclasses
`textual.containers.Vertical`. In Textual 8, `Vertical.DEFAULT_CSS` sets
`height: 1fr; overflow: hidden hidden`. No ACE rule overrides the section's height:
`styles.tcss` only styles `.gate-input-section-title`. Its children are already
`height: auto`: `TypedInputForm` and `.input-field-block`.

The body `#gate-input-body` is an auto-height `VerticalScroll` with `max-height: 32`.
Inside it, each section resolves to a fraction of the body's leftover height. For this
gate that is 32 minus the Note title and editor, about 22 rows. `overflow: hidden` then
clips the rest silently. Each section reports its clipped height, so the body never
overflows:

- Before: the section's `region.height` was 22 and its `virtual_size.height` was 79. The
  body's `max_scroll_y` was `0`.
- Result: no scrollbar, and Tab focused invisible widgets.

The same `1fr` explains two side effects:

- The container stretched to its `max-height: 90%`, leaving blank space under the
  footer.
- The current goldens `gate_input_panel_{single,group}_120x45.png` and
  `gate_input_panel_note_120x40.png` show the same blank gaps.

With `height: auto` on the section, the reproduction gives section height 79/79, body
`max_scroll_y` 57, and a compact container. Tab then walks all 12 fields plus the Note,
and each one auto-scrolls into view.

### 2. j/k gate navigation focuses without scrolling

`GateActionsMixin.focus_gate_control` in `src/sase/ace/tui/modals/gate_action_runner.py`
ends with `.focus(scroll_visible=False)`. That flag dates from the July single-row gate
layout, where it kept the on-mount focus from scrolling the view. Today the Actions and
Decision controls stack at the bottom of a scrolling rail, under possibly long Context
and Attachments sections:

- the preview layout uses `.gate-review-actions`;
- the compact layout uses `#custom-gate-review-scroll`.

A 260x80 reproduction with this gate's shape focused `gate-singleton-1..3` at rows 78–86
while the rail's viewport ended at row 75, and the rail's `scroll_y` stayed `0`. With a
post-focus `scroll_visible(animate=False)`, every j/k step kept the focused control
inside the rail.

`GateBranchControls.expand_group`, in `gate_branch_controls.py`, has the same pattern:
expanding a lower AND group focuses its first member with `scroll_visible=False`.

`PlanApprovalModal` overrides `focus_gate_control` (`plan_approval_modal_controls.py`)
and scrolls its decision cards itself. It is unaffected, and out of scope.

## Changes

### A. Size gate input sections to their content (the core fix)

In `src/sase/ace/tui/styles.tcss`, next to the existing `GateInputPanel` rules, add:

```tcss
GateInputPanel .gate-input-section {
    height: auto;
}
```

Put it immediately before `GateInputPanel .gate-input-section-title`. Leave
`TypedInputForm` and `.input-field-block` unchanged; they are already `height: auto`.

### B. Fit the input panel body to the shell

> [!decision] body_fit = fit Implement this section.

> [!decision] body_fit = cap Skip this section. Change A alone makes every field
> reachable inside the existing 32-row window. Drop the B-specific tests below.

Mirror `CustomGateModal._fit_compact_review_scroll` (`custom_gate_modal.py`) in
`GateInputPanel` (`src/sase/ace/tui/modals/gate_input_panel.py`):

- Add a module constant `_BODY_MIN_ROWS = 6` and an instance field
  `self._body_max_height: int | None = None`.
- At the end of `on_mount`, call `self.call_after_refresh(self._fit_body_scroll)`. Add
  `on_resize(self, _event)`, which returns early when not mounted and otherwise calls
  `self.call_after_refresh(self._fit_body_scroll)`.
- Write `_fit_body_scroll()` to match the custom gate arithmetic:
  1. Query `#gate-input-container` (shell), `#gate-input-body` (body) and
     `#gate-input-footer`, and return when the shell has no width.
  2. Set `shell_max = int(shell.styles.max_height.resolve(self.size, self.app.size))`.
     Fall back to `self.size.height` when the shell has no `max_height`.
  3. Set `top = body.region.y - shell.region.y`.
  4. Set
     `bottom = footer.region.bottom - body.region.bottom + shell.styles.gutter.bottom`.
  5. Set `new_max = max(_BODY_MIN_ROWS, shell_max - top - bottom)`.
  6. If `new_max` differs from the cached value, cache it, assign
     `body.styles.max_height = new_max`, and re-queue itself with `call_after_refresh`
     so the layout settles.
- Keep `max-height: 32` on `GateInputPanel #gate-input-body` in `styles.tcss` as the
  pre-fit first-frame cap. Add a one-line comment saying runtime fitting overrides it.
- The body keeps `height: auto`, so short forms stay compact. Only the cap moves.

Do not refactor `CustomGateModal` to share the helper. The duplication is about 20
lines, and touching the custom gate sizing path is out of scope.

### C. Scroll the focused gate control into view on j/k

In `src/sase/ace/tui/modals/gate_action_runner.py`:

- Change the signature to
  `def focus_gate_control(self, delta: int, *, scroll_visible: bool = True) -> None`.
- Keep `.focus(scroll_visible=False)` on the target. When `scroll_visible` is true, then
  call `target.scroll_visible(animate=False)`, the repo's usual non-animated idiom.
- In `CustomGateModal.on_mount` (`custom_gate_modal.py`), call
  `self.focus_gate_control(1, scroll_visible=False)` so an opening gate still shows the
  top of its Context. `action_next_control` and `action_previous_control` keep the
  default, which now scrolls.
- In `GateBranchControls.expand_group`, keep `first.focus(scroll_visible=False)` and
  follow it with `first.scroll_visible(animate=False)`, because expanding is a user
  action on a possibly lower group.
- Leave `PlanApprovalControlsMixin.focus_gate_control` unchanged. Its signature has no
  keyword, but it overrides the method and is never called with one; mypy's override
  check is already silenced there with `# type: ignore[override]`.

## Tests

### New module `tests/ace/tui/test_gate_input_panel_layout.py`

Keep `test_gate_input_panel.py` under the toobig thresholds. Use `StyledGateTestApp`
from `tests/ace/tui/_custom_gate_modal_helpers.py`, which loads the real `styles.tcss`.
Build the fixture with `GateOption.from_mapping` and `build_gate_input_request`, as the
existing panel tests do. It should mirror the reported gate:

- two options, `attest` and `approve`, both declaring the same 12 inputs:
  - 9 required `enum` fields with 5 choices each and a `help` string;
  - 1 required `bool`;
  - 1 optional `text`;
  - 1 optional `text` field with `secret: true`;
- both options selected in one branch, with `feedback_mode="optional"`.

Tests:

1. `test_tall_section_is_not_clipped_and_body_scrolls`: at 140x80 and 120x40, for every
   `.gate-input-section`, assert `section.size.height == section.virtual_size.height`
   (no clipping). Also assert `#gate-input-body` has `max_scroll_y > 0`. Before change
   A, this fails with 22 vs 79 and `max_scroll_y == 0`.
2. `test_tab_reaches_every_field_and_scrolls_it_into_view`: press `tab` through
   `len(panel._control_ids())` steps; scope to the panel's public behavior if mypy or
   symvision objects to the private access. After each step, assert that the focused
   widget's region lies inside the body's region whenever the focused id is a body
   control. Also assert every one of the 12 field input ids was focused at least once.
3. Only for `body_fit = fit`, `test_body_fits_shell_and_footer_stays_inside`:
   - For sizes (80, 24), (120, 40), (140, 80), resize and wait for the layout. Assert
     `footer.region.bottom <= shell.region.bottom - shell.styles.gutter.bottom`.
   - At 140x80, assert the body is taller than 32 rows.
   - Reuse the `_wait_for_terminal` resize-settle approach from
     `tests/ace/tui/test_custom_gate_modal_sizing.py`. If both modules need it, move it
     into `_custom_gate_modal_helpers.py` as a public `wait_for_terminal` and import it
     in both.
4. `test_short_panel_stays_compact`: a single option with one required `line` field at
   140x80 keeps `#gate-input-container` under 25 rows. This guards against the fit, or
   any future rule, stretching small panels.

### Custom gate navigation test (add to `tests/ace/tui/test_custom_gate_modal_sizing.py`)

`test_jk_navigation_keeps_lower_branches_visible`, parametrized over
`preview in (False, True)`:

- Build the gate with the module's `_LONG_NOTES`, a grouped `attest`+`approve` branch
  (`GateGroup`), and singleton branches `rotation_only`, `defer`, `reject`, using the
  `data()` and `option()` helpers.
- At 160x50, first assert the rail's `scroll_y == 0` after mount, so Context stays at
  the top. The rail is `#custom-gate-review-scroll` in the compact layout and
  `.gate-review-actions` in the preview layout.
- Then press the default `next_control` key, j, once per control id. After each press,
  assert the focused control's region lies inside the rail's region, and that the
  singleton branch buttons were each focused. Before change C, this fails on
  `gate-singleton-*` while `scroll_y` stays 0.

## Visual goldens

Change A, and change B with `fit`, alter the rendered panel. The blank gaps and the
stretched container disappear. Regenerate only the gate-input-panel goldens:

```bash
just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_gate_input_panel.py
```

Run it through `/sase_monitor` if it risks exceeding the turn. Then inspect all three
updated PNGs (`gate_input_panel_single_120x45`, `gate_input_panel_group_120x45`,
`gate_input_panel_note_120x40`). Confirm that:

- the fields render contiguously with no stretched gaps;
- the progress line, buttons, and footer sit directly under the body.

Following the golden-maintenance rule, prove host independence with two targeted
`just fix-tui-screenshots --check -- <same path>` runs, one with `TZ=UTC`, before
committing images. No custom-gate golden presses j/k, and on-mount focus does not
scroll, so custom-gate goldens should not change. If the targeted check of
`tests/ace/tui/visual/test_ace_png_snapshots_custom_gate.py` reports drift, stop and
investigate instead of accepting it.

## Verification

- Run `just fmt`, then `sase tool run check`. Do not run `check-full`; nothing here
  explicitly requires it.
- Run the targeted visual commands above.
- Optionally confirm in a live TUI with `sase screenshot`: open any custom gate whose
  AND branch carries 10 or more inputs. Tab through the input panel, and j/k through the
  Decision rail.

## Out of scope

- Enum headers show `enum (5)` instead of listing the choices once there are more than a
  few. That label comes from the Rust `macro_input_type_label` binding, and cycling the
  enum button reveals each choice, so it is not part of this bug.
- Plan approval modal navigation.
- Refactoring the custom gate's compact-scroll fit into a shared helper.
