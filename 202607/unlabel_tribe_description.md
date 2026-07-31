---
tier: tale
title: Unlabel the agent tribe description summary
goal: Render tribe descriptions as full-width prose without a field prefix or hanging indentation.
create_time: 2026-07-31 10:38:19
status: wip
---

- **PROMPT:** [202607/prompts/unlabel_tribe_description.md](prompts/unlabel_tribe_description.md)

# Unlabel the agent tribe description summary

## Goal

Render the agent tribe panel's metadata summary description as a standalone, full-width prose block: keep its blank-line
separation and styling, but remove the `Description:` prefix and the corresponding hanging indentation from every
wrapped line.

## Context and scope

- The screenshot shows the tribe metadata document rendered by
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe_header.py`. Its `_append_description` helper currently
  emits a `Description: ` field label, subtracts that label's cell width from the shared 80-cell prose budget, and
  indents continuation lines to the label width.
- This is a presentation-only TUI change. The existing snapshot/configuration model already provides the correct
  description, so no configuration schema, tribe aggregation, persistence, or Rust core API changes are needed.
- Apply the unlabeled, unindented layout consistently to configured descriptions and to the fallback
  `not set · add ace.tribes.<tribe>.description` guidance for missing descriptions. Preserve the existing blank line
  after `Fold:`, literal handling of markup-like characters, Rich styles for normal/missing text and the configuration
  key, and rendering in cheap mode.
- Keep the render path pure and in-memory; do not introduce file/config reads or other work into the UI render path.

## Implementation

1. In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe_header.py`, simplify `_append_description` so it no
   longer appends the field label or label-width indentation. Wrap configured description text against the full
   `PROMPT_PANEL_LINE_CELL_LIMIT` and append every wrapped line at column zero with `_DESCRIPTION_STYLE`. Render
   missing-description guidance from column zero as well, retaining its current semantic pieces and styles; when its
   configuration key must wrap, wrap it to the full cell limit without a hanging indent. Remove constants and imports
   that become unused with the label-width calculation.
2. In `tests/ace/tui/widgets/test_agent_display_tribe.py`, update the tribe-description contracts to assert the new
   standalone placement after `Fold:`, absence of `Description:`, full-width cell-safe wrapping with no continuation
   indentation, unchanged styles, correct missing-description configuration keys (including `default` and unconfigured
   tribes), cheap-mode availability, and literal markup behavior. Update the all-fold expected prefix so each fold level
   continues to share the same unlabeled header block.
3. In `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py`, replace label-dependent assertions with
   assertions that the description follows `Fold:` and that the removed prefix is absent. Regenerate only the affected
   ACE PNG goldens under `tests/ace/tui/visual/snapshots/png/`, then inspect the actual/expected/diff artifacts to
   confirm the description starts at the left metadata margin and uses the newly available width without unrelated
   layout changes.

## Validation

1. Prepare the workspace as required before running checks: `just install`.
2. Run the focused renderer tests: `pytest tests/ace/tui/widgets/test_agent_display_tribe.py`.
3. Run `just test-visual`; for the expected tribe-summary failures, inspect `.pytest_cache/sase-visual/`, accept only
   intentional goldens with `just test-visual -- --sase-update-visual-snapshots`, and rerun `just test-visual` without
   the update flag.
4. Run the repository-required complete verification: `just check`.

## Acceptance criteria

- Agent tribe metadata summaries display configured descriptions with no `Description:` prefix.
- The first and every continuation line begin at the same unindented metadata margin and respect the shared 80-cell
  limit.
- Missing-description guidance is also unlabeled and unindented while still naming the correct configuration key with
  its existing visual treatment.
- Description placement, blank-line separation, styles, cheap-mode rendering, literal text behavior, and fold-level
  availability remain intact.
- Focused unit tests, reviewed PNG snapshots, the visual suite, and `just check` pass.
