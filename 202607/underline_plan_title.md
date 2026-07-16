---
tier: tale
title: Underline plan titles in agent metadata
goal: 'The Agents-tab SASE PLAN section underlines each available plan title so it
  remains visually distinct from the italic goal beneath it without changing responsive
  layout, fallback states, or metadata behavior.

  '
create_time: 2026-07-16 08:07:38
status: done
prompt: 202607/prompts/underline_plan_title.md
---

# Plan: Underline plan titles in agent metadata

## Context

The Agents detail panel now renders a plan title immediately above its goal. The title and goal share the same
foreground color, with only the goal's italics distinguishing the two values. In the current visual presentation that
difference is too subtle. The title value should gain an underline while the existing field label, goal treatment, and
compact responsive structure remain unchanged.

This is a presentation-only change in the existing cached Agents metadata path. It must not introduce plan reads, new
refresh work, or any other synchronous work on the Textual event loop.

## Implementation

- Adjust the available-title value style in the responsive SASE PLAN renderer so it retains its current color and adds
  an underline. Apply the emphasis to the title value only: do not underline the `Title:` field label, the italic goal,
  or the quiet `unavailable` fallback used for missing or damaged plans.
- Preserve the existing Rich `Text`-based construction and grid wrapping so the style follows the complete title across
  word wrapping, oversized-token folds, and wide Unicode content at narrow and wide panel widths. Do not change row
  order, indentation, width caps, cached metadata, or the title data model.
- Strengthen the Agents plan-section unit coverage to assert the concrete title emphasis independently of the production
  style constant, while retaining the existing assertions for goal italics, fallback styling, field order, and lossless
  responsive wrapping.
- Regenerate only the tale and epic Agents SASE PLAN PNG goldens affected by the title style. Inspect the rendered
  actual/expected/diff artifacts to confirm that the underline appears beneath title values and that no spacing,
  wrapping, color, goal styling, phase layout, or unrelated pixels changed.

## Validation

- Run the focused Agents plan-section widget tests, including narrow/wide and long-title cases.
- Run the dedicated visual snapshot cases for tale metadata and the epic phase roadmap, accepting only the intentional
  underline changes after inspection.
- Run `just install` followed by the repository-required `just check` gate to cover formatting, linting, the full test
  suite, and deterministic PNG snapshots.

## Non-goals and risks

- Do not alter plan validation, authoring templates, generated skills, metadata extraction, or Rust bindings; the
  required title contract is already in place.
- Terminal underline rendering is the only intended visual delta. Retaining the current title color and using the
  established Rich style path minimizes theme and renderer risk.
