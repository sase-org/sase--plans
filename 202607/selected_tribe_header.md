---
tier: tale
title: Restore the selected Agent Tribe panel header
goal: 'Selected expanded Agent Tribe panels retain a clear, beautiful identity header
  above their focus frame without reintroducing layout churn or weakening the existing
  whole-panel selection cues.

  '
create_time: 2026-07-19 07:18:03
status: wip
prompt: 202607/prompts/selected_tribe_header.md
---

# Plan: Restore the selected Agent Tribe panel header

## Context and diagnosis

The `sase-6z` epic made expanded tribe panels first-class selection targets. The title model still produces the intended
selected label (`❖ @<tribe> · <count> [<status counts>]`), and the panel refresh paths still move that label with
whole-panel focus. The header disappears only at paint time: the selected state uses Textual's four-sided heavy
`outline`, which is rendered after the ordinary border and overwrites every cell in the top border, including the
`border_title`.

That outline was introduced deliberately to keep lower-case `j`/`k` panel cycling paint-only. Returning to a double
border, mounting a second header widget, or reserving another row would reintroduce layout work and/or reduce scarce
vertical space. The repair should therefore change the composition of the selected chrome, not duplicate tribe-title
state or alter panel sizing.

## Product and visual contract

Treat the selected expanded panel's top edge as a **label rail** and its body as a **three-sided heavy focus frame**:

- Keep the existing left-aligned title grammar and styles: `❖ @epic · 59 [D56]`, with the diamond and tribe identity in
  tribe gold and the count/status chip retaining its established metric colors.
- Let the normal solid gold top border render the title and its continuation rule. Disable only the selected heavy
  outline's top edge; retain the heavy gold outline on the left, right, and bottom. This places the identity directly
  above the emphasized body while preserving one compact panel and one source of truth.
- Keep the row cursor suppressed during whole-panel focus. The visible header, gold side/bottom frame, and detail
  document together should make “the tribe is selected” unmistakable without suggesting that an interior agent row is
  selected.
- Preserve existing title prefixes and ordering. Jump/fold hints remain before the selection marker, selected expanded
  panels retain `❖`, and collapsed panels retain `▸`. Untagged identity, status counts, and dynamic title-width
  negotiation remain unchanged.
- Do not change ordinary expanded panels, row-focused panels, collapsed selected panels, merged `All agents` mode, or
  the right-side tribe document.

The intended state matrix is:

| Panel state                     | Header                                        | Body selection cue                                |
| ------------------------------- | --------------------------------------------- | ------------------------------------------------- |
| Expanded, row selected          | Existing tribe title                          | Existing focused border plus row cursor           |
| Expanded, whole panel selected  | Visible `❖` tribe title on the top label rail | Heavy gold left/right/bottom frame; no row cursor |
| Collapsed, whole panel selected | Existing visible `▸` tribe title              | Existing compact focused border                   |

## Implementation

Update the selected-panel rule in `src/sase/ace/tui/styles.tcss` so the heavy outline is applied per edge, leaving the
top outline unset while keeping the other three edges heavy and gold. The existing solid focused border should remain
the top-edge geometry; changing only paint styles preserves the panel's two border rows and avoids a collection relayout
when focus moves.

Keep `agent_panel_border_title` and the panel refresh/selection paths as the authoritative title pipeline. They already
carry the selection marker, hints, scoped counts, and width updates. Only adjust Python code if a regression test proves
that a specific refresh path fails to publish the current title; do not introduce a parallel `Static`, header row,
overlay, or selected-title cache.

## Reliability and regression coverage

- Strengthen the selected-expanded interaction coverage so a whole-panel focus transition and a lower-case `j`/`k` hop
  prove that the destination title gains `❖`, the departed expanded title loses it, and collapsed destinations keep
  their `▸` identity instead of inheriting expanded styling.
- In the selected-expanded tribe PNG scenario, assert that the rendered screen contains the exact `❖ @epic` header, not
  merely `@epic` from the metadata document, then update the golden snapshot to capture the label rail, unhighlighted
  rows, and three-sided heavy frame together.
- Retain the existing pure title-builder/count/hint tests and width-negotiation tests; they protect long titles,
  multi-digit hints, untagged panels, and transient prefixes without adding new layout behavior.
- Re-run the selected-tribe `j`/`k` benchmark at all four tribe fold levels and retain its current p95 budget. The fix
  must not reintroduce double-border geometry changes, full panel rebuilds, extra detail renders, or event-loop work.

## Validation

Before testing in the implementation workspace, run `just install` as required for an ephemeral SASE checkout. Then:

1. Run the focused panel-title, panel-display, collapse/navigation, and selected tribe visual tests.
2. Run `just test-visual` and inspect the updated selected-expanded PNG at normal and constrained terminal sizes for
   title legibility, clean frame joins, no overlap, and no extra vertical row.
3. Run `pytest -s -m slow tests/ace/tui/bench_tui_jk.py -k selected_tribe` and confirm every fold level remains within
   the established selected-tribe p95 threshold with no stall log.
4. Run the mandatory `just check` suite and leave unrelated snapshots and behavior unchanged.

## Risks and guardrails

- Textual paints outlines after borders, so any future four-sided `outline` shorthand on the selected class can regress
  the header. Keep the per-edge intent documented beside the CSS and enforce visibility through the rendered-screen
  assertion.
- A separate title widget would consume height and create synchronization risks during panel reordering; keep identity
  in `border_title`.
- A border-style swap can trigger relayout on every panel hop; preserve the paint-only focus path and use the existing
  benchmark as the acceptance gate.
