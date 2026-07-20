---
tier: tale
title: Information-dense Rich epic clan summaries
goal: 'Epic clan summaries render a launch-stable, markdown-aware overview of the
  epic goal, phase progress and metadata, child epics, and plan reference while remaining
  valid Rich markup within the persisted width and size limits.

  '
create_time: 2026-07-20 11:33:09
status: done
prompt: 202607/prompts/rich_epic_summary.md
---

# Plan: Information-dense Rich epic clan summaries

## Context and boundaries

The built-in `sase_clan_summary_epic` launch script currently persists a compact Rich-markup string containing a header,
a single shortened goal line, and bare numbered phase titles. The preceding `sase-85.1` work already made the bead read
launch-fresh, diagnosable, and tolerant of failures; this change will preserve that refresh/retry/fallback behavior and
alter only the successfully loaded summary data and presentation. The TUI will continue to render the persisted markup
through its existing clan-detail path, so no render-time I/O, new wire fields, sase-core changes, or member-panel
changes are needed.

## Markdown-aware rendering primitives

Add a small presentation helper near the built-in scripts that converts bead Markdown into safe Rich markup at a
caller-selected width. Use Rich's pinned Markdown/code-theme rendering into a deterministic truecolor console capture,
decode the ANSI styling back into `Text`, trim capture padding line by line, and serialize markup only after styles and
literal bracket characters are safe. Keep plain content on a lightweight path when Markdown rendering would add no
value. Provide utilities or structured results that let the epic renderer cap, shorten, and compose content at complete
`Text`/line boundaries rather than truncating serialized markup.

Cover inline code, bold text, lists, literal brackets, trailing-padding removal, and successful `Text.from_markup`
parsing. Pin expectations to semantic styles and rendered plain text instead of fragile escape-sequence internals.

## Rich epic summary document

Extend the epic load result to separate ordered phase children from ordered child plan beads whose tier is `epic`.
Rebuild the successful summary as a 76-column document with:

- a bold magenta `◆ EPIC <id> · <title>` identity header;
- the full Markdown-highlighted goal, wrapped and capped at about six lines with a visible ellipsis only when necessary;
- a `PHASES · <closed>/<total> done at launch` heading;
- one phase row per phase with the existing open/in-progress/closed glyph, launch-time status color, ordinal and
  shortened title, plus a right-aligned small/medium/large chip (using a stable small fallback for legacy sizeless
  phases), followed by one shortened Markdown-aware description line when set;
- an optional `CHILD EPICS · <count>` section for direct epic-tier plan children, including each child's launch-time
  glyph, id, and shortened title; and
- an optional dim `Plan:` line using the epic bead's stored design reference.

Escape directly interpolated ids, titles, paths, and generated labels. Measure visible cell width before serialization,
keep every rendered line at or below 76 cells, and enforce an internal UTF-8 budget below the runner's 32 KiB hard cap
by omitting only complete trailing entries/blocks with an explicit summary line. This keeps large epics useful without
ever producing markup truncated in the middle of a tag. Preserve the existing stdout/stderr and fallback contract.

## Regression and visual coverage

Expand the epic-script tests with representative Markdown-rich goals and phase descriptions, all three statuses and
sizes, mixed child phases/epics, plan-path escaping, absent optional sections, long text, and many phases. Assert
semantic plain text, status/progress values, style presence, deterministic ordering, parseable markup, per-line width,
and total UTF-8 size, while retaining the existing refresh and fallback tests.

Replace the epic clan-panel visual fixture summary with the richer persisted markup, including progress, size chips,
descriptions, a child epic, and a plan reference. Tighten SVG text assertions around those new sections. Regenerate and
inspect only the three epic clan-panel PNG goldens at all fold levels; confirm the swarm goldens remain unchanged.

## Validation

Run `just install` before repository checks. Exercise the Markdown helper and epic renderer tests first, then run the
targeted epic visual snapshot test in normal mode to capture intentional diffs. Regenerate its goldens with
`--sase-update-visual-snapshots`, inspect the PNGs and any `.pytest_cache/sase-visual/` actual/expected/diff artifacts
for color, wrapping, alignment, and section visibility, and rerun `just test-visual` without update mode. Spot-check
`sase_clan_summary_epic` against the existing `sase-85` bead so the real stored plan path and current child statuses
parse correctly without creating or launching any new beads. Finish with `just check`, verify the diff contains only
intended source/tests/goldens, close `sase-85.2`, and explicitly confirm parent epic `sase-85` remains open or in
progress.
