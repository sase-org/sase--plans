---
tier: tale
title: Concise single-line commit timeline
goal: 'Every commit entry in Artifacts > Commits is a beautiful, stable one-line summary
  at any supported pane width, while the selected commit''s complete useful metadata
  remains immediately available in the detail pane.

  '
create_time: 2026-07-17 12:17:31
status: wip
prompt: 202607/prompts/commits_single_line_timeline.md
---

# Plan: Concise single-line commit timeline

## Product outcome

The Commits timeline should read like a dense, calm history rather than a stack of irregular cards. One commit must
always occupy exactly one physical row, so navigation distance, highlight shape, and chronology remain visually obvious.
The row will optimize for scanning; the adjacent detail pane will provide the complete record for the current selection.

This is a presentation-only TUI change. It does not alter commit collection, filtering, ordering, selection identity,
diff loading, keymaps, the `sase vcs log` CLI output, or the Rust wire model.

## Chosen interaction and visual design

Use progressive disclosure rather than squeezing all metadata into the list. Each row keeps the five fields that
distinguish commits at a glance, in a stable left-to-right hierarchy:

```text
↑ 11:55  2ec405a  plans      chore(beads): update sase-61.3
● 11:17  c9c8131  sase       feat(plan): guide phase description authoring
```

1. The colored presence glyph communicates unpushed, GitHub-only, synced, or unknown state and continues to agree with
   the legend.
2. Local wall-clock time anchors the entry inside its day banner.
3. The gold short SHA gives the commit a compact, copyable identity.
4. The repository keeps the cross-repository timeline understandable and retains its deterministic repository color.
5. The subject gets all remaining horizontal space because it is the best human-readable summary of the change.

Do not append author names or inline SASE footer tags after the subject. Those are valuable, but they are secondary when
scanning and are the main reason the current rows wrap. They already belong naturally in the selected detail pane. The
timeline's existing alignment, palette, day separators, selection highlight, and presence glyphs should be retained so
the redesign feels like a refinement rather than a different component.

Long content must end with a single ellipsis instead of wrapping. The leading status/time/SHA/repository columns remain
visible as space allows, and the subject receives the flexible tail of the row. At extreme widths, ellipsis is an
intentional preview affordance, not data loss: selecting the row reveals the unabridged values immediately to its right.
Jump-mode prefixes such as `[1]` participate in the same one-line budget and must not increase row height.

## Information-preservation contract

Make the selected detail header the authoritative, unabridged companion to the compact row. It should present:

- the full repository name and short SHA;
- the full author name;
- a full localized commit timestamp, so the row's compact time and day banner are not the only timestamp representation;
- both the colored presence glyph and its textual label (`unpushed`, `GitHub-only`, `synced`, or `unknown`);
- the full subject and body;
- every parsed SASE footer tag using the existing full tag lines; and
- the existing change summary and diff.

Keep presence labels and styling sourced from the same presentation mapping as the timeline/CLI legend so states cannot
drift into contradictory wording or colors. Missing optional values should degrade cleanly: omit an empty author, retain
the existing message-unavailable fallback, and always render a valid presence label.

This gives every datum currently visible in a wrapped row a deliberate home: presence, time, SHA, repository, and
subject remain in the row; author and SASE tags move to the detail pane; the full message and diff remain there. The
existing `y` action continues to copy the full commit ID, and Enter continues to open the commit modal.

## Rendering implementation

Refine the interactive timeline builder without changing the normal pretty CLI line:

- Let the timeline call the shared VCS-log row builder with explicit compact inclusion controls (no inline tags and no
  trailing author), while preserving existing defaults for other callers. Keep presence/time/SHA/repository formatting
  centralized rather than recreating it in the widget.
- Mark timeline row `Text` values as `no_wrap=True` with ellipsis overflow and ensure no row builder introduces embedded
  newlines.
- Add `text-wrap: nowrap` and `text-overflow: ellipsis` to `#commits-timeline`. This CSS layer is the actual
  mounted-widget guarantee: Textual 8 converts Rich option prompts to `Content` and drops their Rich wrapping
  attributes. Keeping both layers makes direct renderer use and the real `OptionList` behavior correct.
- Apply the same CSS contract to commit rows with and without transient jump hints. Do not add resize observers or
  rebuild options on navigation; CSS should handle width changes without work on the keypress path.
- Extend the existing commit-detail metadata header with localized timestamp and human-readable presence, reusing
  project time and presence formatting. Continue using the current debounced detail update and background diff load; no
  I/O or parsing should enter render or navigation handlers.

Likely implementation touchpoints are `src/sase/vcs_log/render.py` for compatible compact row/presence presentation,
`src/sase/ace/tui/widgets/artifacts/commits_timeline.py` for selecting the compact row form,
`src/sase/ace/tui/widgets/artifacts/commits_rendering.py` for the complete detail header, and
`src/sase/ace/tui/styles.tcss` for the mounted one-line contract. Keep the change within these presentation boundaries
unless tests expose a smaller shared helper worth reusing.

## Reliability, tests, and visual validation

Strengthen the commit fixtures with deliberately long subjects, author names, repository labels, and multiple SASE tags
so the original failure is always exercised rather than depending on incidental terminal width.

Add or update focused coverage for all of these contracts:

1. Renderer tests assert that a compact row contains presence, time, short SHA, repository, and subject; excludes inline
   author and SASE tags; contains no newline; declares Rich nowrap/ellipsis behavior; and wraps to exactly one line
   under a narrow Rich console.
2. Detail tests assert that the information removed from the row is still rendered in full, including author, localized
   timestamp, textual presence, complete subject/body, and all SASE tags. Exercise at least local-only, remote-only,
   synced, and unknown presence mappings.
3. A mounted `CommitsTimeline` regression clears/rebuilds Textual's line cache at a constrained width and asserts
   `text_wrap == "nowrap"`, `text_overflow == "ellipsis"`, and a cached height of exactly one for every commit option.
   Repeat the assertion with jump hints active, while confirming day banners remain disabled and selection identity is
   preserved.
4. Update the populated and jump-hint PNG snapshots so review captures the intended rhythm: uniform one-line highlights,
   aligned metadata, graceful ellipsis, and a detail pane visibly containing the relocated information. Add a
   constrained-width visual case if the existing snapshot helper can express it cleanly.
5. Run the focused commits behavior and visual suites, then the Artifacts key-to-paint benchmark
   (`tests/ace/tui/bench_artifacts_jk.py`) and preserve its p95-under-16-ms budget. Finally run `just check` after the
   required `just install` workspace setup.

## Acceptance criteria

- Every selectable commit occupies one and only one physical `OptionList` row at supported widths, including unusually
  long metadata and jump mode.
- Overflow is shown as an ellipsis; it never becomes a continuation line or a taller selection highlight.
- Rows retain the presence glyph, time, short SHA, colored repository, and as much subject as fits, with stable
  alignment and existing semantic colors.
- Selecting any truncated row exposes its complete useful metadata in the adjacent detail pane without another action.
- CLI VCS-log output, collection semantics, selection/navigation, copy/view actions, filtering, refresh behavior, and
  diff loading are unchanged.
- Navigation performs no new I/O, width-driven rebuild, or synchronous data work and remains within the existing
  Artifacts paint-latency budget.
