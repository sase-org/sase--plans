---
tier: tale
title: Concise agent-family root count chips
goal: "Parallel agent-family root rows show member status counts as compact,
  consistently highlighted chips that match agent panel header counts.

  "
create_time: 2026-09-09 19:53:08
status: wip
---

# Plan: Concise Agent-Family Root Count Chips

## Context and outcome

ACE now aggregates the statuses of direct members attached through `%family` and shows
them on the family root row. The aggregation is useful, but the current plain annotation
(`4 running · 1 waiting`) is long and is rendered as one dim-cyan span. Agent panel
titles already have a denser visual vocabulary: bracketed, zero-suppressing status
tokens whose counts are highlighted with status-specific colors.

Make the panel-title count chip the single presentation contract for both uses. The
existing panel-header grammar is a status letter followed by its count, in the canonical
`S`, `R`, `W`, `F`, `U`, `D` order. Accordingly, a family with four running members and
one waiting member will render `[R4 W1]`. This deliberately normalizes the prompt's
transposed `[W1 4R]` example to the established panel header syntax instead of
introducing a second shorthand dialect.

This is a presentation-only TUI change. It must not alter `%family` membership, root
status aggregation, family folding, child visibility, or backend data.

## Design

1. Extract the bracketed agent-count rendering from the panel-title code into a small
   shared Rich `Text` formatter. It will own the canonical metric labels, ordering,
   neutral delimiter/letter styling, and the existing bold per-status count styles.
   Zero-valued metrics remain omitted, multi-digit counts remain intact, and the unread
   token retains its full highlighted treatment. Keep the panel-title output and styles
   byte-for-byte equivalent by delegating its current rendering to this formatter.

2. Adapt parallel-family member counts to that shared metric vocabulary at the
   presentation boundary: stopped/awaiting maps to `S`, running (including the existing
   starting bucket) to `R`, waiting to `W`, failed to `F`, and done to `D`. Continue
   counting only already-loaded direct children marked as parallel family members; do
   not count the root itself or serial workflow children.

3. Separate family metrics from the structural fold annotation. Keep `×N`, hidden-child
   `+N`/`−N`, and retry `↻N` text in the existing fold-annotation path, then append the
   independently styled family chip from the agent-row renderer. Preserve the chip in
   both collapsed and expanded family-root rows, preserve spacing around any structural
   annotation, and omit it entirely when there are no parallel members or all counts are
   zero.

4. Preserve the agent-row fast path. Use the immutable family count bundle that already
   participates in the row render-cache signature to drive the visible chip as well, so
   a member status transition invalidates and recolors the root row without disk access,
   subprocess work, or a new full-list rebuild path. Avoid repeated child aggregation
   within one cache/render attempt where the existing interfaces can pass the computed
   bundle through.

The main implementation touchpoints are the panel-title formatter in
`src/sase/ace/tui/actions/agents/_display_panel_titles.py`, the parallel-family count
model in `src/sase/ace/tui/models/_agent_parallel_family.py`, and the fold/row/cache
rendering path under `src/sase/ace/tui/widgets/_agent_list_*`. Keep aggregation in the
model layer and Rich composition in the TUI presentation layer.

## Verification

- Add focused formatter tests for empty, single-status, mixed-status, and multi-digit
  chips, asserting both plain text and Rich spans for neutral syntax and each status
  color. Retain panel-title tests as compatibility coverage.
- Update family-row tests to cover collapsed and expanded roots, combinations with
  fold/retry annotations, all supported family buckets, zero suppression, and non-family
  rows. Assert the concise text and verify that verbose status words and middle-dot
  separators are absent from the family summary.
- Add or extend render-cache coverage so changing a parallel child's status produces a
  new root-row rendering and the new count/color distribution.
- Update the dedicated `agents_parallel_family_counts_120x40` PNG snapshot and its SVG
  text assertions, then inspect the generated actual/expected/diff artifacts to confirm
  the chip is legible and correctly highlighted in the real row layout.
- Run `just install` before the focused unit and visual tests, then use those tests
  while iterating. Before handoff, run the repository-required `just check`.

## Risks and boundaries

- A shared formatter must not create an actions/widgets import cycle; place the
  presentation primitive at a neutral TUI layer that both callers can import.
- Rich `Text` styling must remain explicit rather than embedding markup in plain
  strings, so brackets cannot be misparsed and cell-width/alignment calculations
  continue to use the rendered text accurately.
- Do not broaden this tale into converting the top Agents info strip or unrelated
  verbose summaries; only panel titles and `%family` root rows share this compact chip
  contract.
