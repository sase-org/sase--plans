---
create_time: 2026-07-11 09:59:59
status: done
prompt: 202607/prompts/models_panel_no_wrap_width.md
tier: tale
---
# Plan: Widen the Models Panel for Single-Line Content

## Goal

Make the ace **Models** panel (leader `,m`) wide enough that its built-in alias descriptions, bucket descriptions, model
badges, state labels, and key hints render on their intended lines at normal and wide terminal sizes. Keep the modal
contained on narrower terminals, where some wrapping or ellipsis is unavoidable because the viewport itself is smaller
than the content.

The motivating screenshot shows the panel centered in a roughly 256-column terminal while retaining its fixed 84-column
width. The panel has ample screen space available, but does not use it.

## Verified cause

- `#models-panel-container` in `src/sase/ace/tui/styles.tcss` has a fixed width of 84 cells and no viewport-relative
  maximum. Its border and horizontal padding leave 78 cells for the description/footer widgets; the `OptionList` has its
  own gutter and leaves about 74 cells for each model row.
- The current built-in `default` description is 96 cells, the built-in `coder` description is 87 cells, and the
  configured `research` bucket description in the reported environment is 93 cells. These necessarily wrap inside the
  current 78-cell description region even on a very wide terminal.
- Model rows are already created as Rich `Text(no_wrap=True, overflow="ellipsis")`, so they remain one physical row.
  However, `PROVIDER_MODEL_CELL_MAX` in `src/sase/ace/tui/modals/models_panel_rendering.py` is capped at 20 cells to fit
  the old 84-cell budget. That truncates the 25-cell `CLAUDE(claude-fable-4-10)` badge visible in the screenshot even
  though the terminal has room for it.
- The fixed width also overflows an 80-column viewport today. A larger fixed width without a responsive maximum would
  make that existing narrow-terminal behavior worse.
- The description strip intentionally reserves two stable content rows. Bucket details use both rows (description plus
  effective-model mix), and the fixed height prevents layout shifts while navigating. Its vertical sizing should not be
  reduced as part of this change.

This is presentation-only Textual layout/rendering behavior. It remains in the Python TUI layer and does not cross the
Rust core backend boundary.

## Design

### 1. Give the modal a larger preferred width with a narrow-screen cap

In `src/sase/ace/tui/styles.tcss`, change `#models-panel-container` to:

- prefer a width of **110 cells**, yielding 104 cells in the description/footer region and approximately 100 usable
  cells in the `OptionList` after its gutter;
- cap itself at **95% of the viewport width**, following the responsive modal pattern already used elsewhere in the
  stylesheet.

At full preferred width, 104 cells accommodates the measured 96-cell built-in description without wrapping and leaves
some room for future wording changes. At 120 columns (the standard visual-test viewport), the modal retains a visible
margin. At narrower widths, the percentage cap keeps both horizontal edges on-screen rather than allowing the modal to
extend beyond the viewport.

Do not change the modal height budget: the list cap, stable two-row description strip, and footer calculations are
independent of this horizontal expansion.

### 2. Spend the new row width on readable model badges

In `src/sase/ace/tui/modals/models_panel_rendering.py`, raise `PROVIDER_MODEL_CELL_MAX` from its old 20-cell budget to a
new cap sized for the 110-cell modal. A 32-cell cap is sufficient for the reported 25-cell Claude badge while retaining
comfortable space for the longest state label and future copy within the roughly 100-cell option-row region.

Update the nearby budget documentation so it describes the new preferred-width geometry instead of the obsolete
84/78-cell calculation. Preserve the current behavior that:

- aligns the state/provenance column across rows;
- measures badges with Rich cell widths;
- ellipsizes genuinely pathological model identifiers;
- marks the complete row as `no_wrap`, providing a safe single-line fallback when the viewport is narrower than the
  preferred modal width.

Avoid adding `on_resize` handlers or rebuilding option data during layout changes. CSS containment plus the existing
Rich overflow behavior solves this without adding work to the Textual event loop.

### 3. Add focused regression coverage

Extend `tests/test_models_panel.py` with layout and rendering checks that load the production stylesheet:

1. At a 120x40 viewport, assert the Models container reaches the 110-cell preferred width and a production-length
   96-cell description fits within the description content region. This guards the actual no-wrap width budget rather
   than merely checking that the `Static` contains the right text.
2. At an 80-column viewport, assert the container stays within the screen bounds. This guards the new `max-width`
   fallback and improves on the current overflow behavior.
3. Add a representative provider/model rendering case for `CLAUDE(claude-fable-4-10)` and assert it is preserved without
   an ellipsis under the new cap, while keeping the existing extremely-long-model test to prove ellipsis and
   state-column alignment still work beyond the cap.
4. Keep the existing assertion that the description strip has two visible content rows; widening must not regress the
   stable bucket-detail layout.

### 4. Update and inspect Models-panel visual goldens

Update the deterministic fixtures in `tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py` so at least one alias
uses a production-length description and one row uses a model badge that exceeded the old cap. Regenerate the four main
Models-panel 120x40 PNG goldens:

- `models_panel_default_120x40.png`
- `models_panel_overrides_120x40.png`
- `models_panel_bucket_120x40.png`
- `models_panel_bucket_drilled_in_120x40.png`

Inspect the regenerated images to confirm the wider modal remains centered, the representative descriptions no longer
wrap, the long model badge is readable, row/state columns remain aligned, bucket details still occupy their deliberate
two-line strip, the footer remains on-screen, and the modal retains reasonable side margins.

The alias edit-preview, model picker, duration picker, and exact-time modals are separate surfaces with their own sizing
and are out of scope.

## Verification

1. Run `just install` first, as required for an ephemeral workspace.
2. Run the focused non-visual Models-panel tests (`just test -- tests/test_models_panel.py`).
3. Regenerate only the intentional Models-panel goldens with
   `just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py`,
   then inspect the PNG diffs.
4. Re-run the same visual test file without the update flag to prove the accepted goldens are stable.
5. Run mandatory full verification with `just check`.
6. Manually open `sase ace`, invoke `,m`, and verify both a wide terminal comparable to the supplied screenshot and a
   constrained terminal: wide content should remain on its intended lines, while the narrow modal must stay inside the
   viewport and preserve single-line/ellipsis behavior for model rows.

## Acceptance criteria

- At the preferred 110-cell width, all current built-in alias descriptions fit on one line; bucket descriptions retain a
  separate second line only for their intentional effective-model summary.
- `CLAUDE(claude-fable-4-10)` and similarly sized badges render in full, with the state column still aligned.
- The Models modal is centered and contained at both the 120-column visual-test size and an 80-column constrained size.
- No resize-time I/O, data reload, or option-list rebuild path is introduced.
- Focused tests, regenerated visual snapshots, and `just check` pass.

## Risks and boundaries

- No finite modal can guarantee unwrapped arbitrary user descriptions or model identifiers on every terminal size. The
  guarantee is therefore scoped to current built-in/representative content at the preferred width; narrow viewports
  degrade safely, and unusually long model identifiers continue to ellipsize.
- Increasing the provider/model cap shifts the state column to the right. The unit alignment assertions and all four
  main visual states are the controls against clipping or inconsistent columns.
- The wider panel changes four intentional visual goldens. Review those image diffs closely and do not accept unrelated
  snapshot churn.
