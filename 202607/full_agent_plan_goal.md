---
tier: tale
goal: 'Every associated plan goal is visible in full in the ACE Agents metadata panel,
  with responsive, polished wrapping that keeps every rendered line at or below 80
  terminal cells without slowing navigation.

  '
create_time: 2026-07-15 09:11:16
status: wip
prompt: 202607/prompts/full_agent_plan_goal.md
---

# Plan: Show complete plan goals in ACE agent metadata

## Context

The Agents-tab metadata header already resolves an agent's associated plan goal off the Textual event loop and renders
it immediately after `Name`/`Bead`. The renderer currently passes the value through `_truncate_plan_goal`, however,
which replaces everything beyond 72 characters with an ellipsis. Rich may then wrap that shortened value to the visible
panel width, but the omitted suffix is unrecoverable even though the metadata panel is vertically scrollable.

This change is presentation-only. Plan association, bead fallback, normalized frontmatter reading, mtime-aware caching,
and background enrichment should stay unchanged.

## User experience

Render the complete normalized goal as a distinct responsive metadata row:

```text
Goal: Make the selected agent's intended outcome immediately legible while
      preserving fast keyboard navigation across every Agents-tab entry.
```

- Keep `Goal:` in the existing bold blue metadata-label style and keep the goal value in the existing warm italic style.
- Keep the field directly below `Bead`, or directly below `Name` when there is no bead.
- Use hanging indentation: wrapped continuation lines begin beneath the first character of the value, not beneath the
  label or at the panel's left edge.
- Reflow against the available metadata content width and cap the goal block at 80 terminal display cells. A narrow
  panel may use fewer than 80 cells; a wide panel must not stretch the goal beyond 80.
- Prefer word boundaries. Hard-fold an individual token only when it cannot fit within the available value column, using
  terminal-cell width rather than Python character count so wide Unicode remains within the budget.
- Never add an ellipsis, character cap, line cap, collapsed state, or hidden suffix. Arbitrarily long goals remain
  reachable through the existing metadata panel's vertical scrolling.
- Preserve the current behavior for short goals and omit the row when no goal has been resolved. The cheap immediate
  navigation render may remain goal-free until the cached/background header summary is applied.

## Technical design

1. Replace the truncation path in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py` with a small, pure
   goal-row renderable. Model the row as a fixed label column and a flexible value column, constrained to
   `min(available_width, 80)`. Rich's width-aware layout primitives (for example a borderless grid under a width
   constraint) should perform responsive wrapping and preserve hanging indentation on both initial render and terminal
   resize. Configure the value to fold oversized tokens rather than crop or ellipsize them.

2. Adjust metadata-header composition only as much as needed to place that renderable between the existing Name/Bead
   prefix and the remaining metadata. Keep one shared composition path for normal agent details, cached refreshes,
   cheap/header-only updates, file-hint rendering, and bash/python/parallel child displays so the field does not
   disappear or move depending on how the panel was reached. Preserve existing `Text` styling and append behavior for
   all non-goal metadata and prompt/chat sections; avoid duplicating the large header builder merely to insert the
   wrapped row.

3. Keep rendering entirely in memory. The goal renderable consumes only `DetailHeaderSummary.plan_goal`; it must not
   read the plan, query beads, launch work, rebuild the agent list, or introduce a resize handler. Rich should reflow
   the retained renderable from its current console options, so resizing remains a normal paint rather than a detail
   refresh.

4. Update the Agents metadata documentation in `docs/ace.md` to state that the complete goal is displayed and
   responsively wrapped to at most 80 terminal cells. Do not change plan-goal lookup semantics or promise a Goal row for
   agents whose association cannot be resolved.

## Tests and visual acceptance

- Replace the truncation unit test with rendering assertions that use Rich at controlled widths and prove the complete
  goal is present with no ellipsis or dropped suffix.
- Cover a wide rendering context to verify no physical goal line exceeds 80 terminal cells, and a narrow context to
  verify reflow plus hanging alignment.
- Cover a single oversized token and wide Unicode so hard folding obeys the terminal-cell budget and makes forward
  progress without data loss.
- Retain coverage for row ordering, short single-line goals, absent goals, and the existing label/value styles. Validate
  the actual composed renderable rather than relying only on the logical backing text.
- Lengthen the deterministic Agents-tab PNG fixture so the final words would have been lost by the old 72-character
  truncation. Assert those final words appear in the exported SVG, update the golden, and inspect the PNG for label
  alignment, comfortable wrapping, unchanged colors, and a clean transition into the next metadata row.
- Run the focused prompt-panel/plan-goal tests and the dedicated visual suite, then run `just install` followed by
  `just check` as required for repository changes. If the header-composition adjustment touches shared paths, include
  the existing prompt-panel header, hint, workflow-child, and navigation tests in the focused pass.

## Risks and guardrails

- Rich renderables can measure characters differently from terminal cells. Tests must assert `cell_len`, not `len`, and
  the implementation must let Rich fold wide or unbroken content rather than pre-slicing strings.
- A fixed-width table could overflow a narrow panel; the 80-cell value is an upper bound, while the current render width
  remains the lower constraint.
- Splitting the header into multiple renderables can accidentally add blank rows or detach later prompt/chat content.
  Snapshot and ordering assertions should cover boundaries before and after `Goal` on both short and wrapped values.
- Goal resolution is intentionally absent from the hot render path. Do not move any filesystem or bead work into
  rendering while changing presentation.
