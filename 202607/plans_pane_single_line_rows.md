---
tier: tale
title: Single-line concise rows for the Artifacts Plans pane
goal: "Every row in the Plans sub-tab's left pane renders on exactly one line at any
  pane width, with compact metadata ordered so the most useful information survives
  truncation.

  "
create_time: 2026-09-09 19:53:17
status: wip
---

# Plan: Single-line concise rows for the Artifacts Plans pane

## Problem

Rows in the "Plan pipeline" list (Artifacts → Plans) wrap onto two or more lines
whenever `id + title + phase count + state + age + project badge` exceeds the pane
width. With long epic titles this is the common case, which makes the list ragged and
hard to scan.

Root cause: `single_line_text()` in
`src/sase/ace/tui/widgets/artifacts/plans_rendering.py` builds
`rich.text.Text(no_wrap=True, overflow="ellipsis")`, but under Textual 8 an `OptionList`
option prompt of type `rich.text.Text` is converted through
`Content.from_rich_text(...)`, which discards the Text-level `no_wrap`/`overflow`
attributes. Wrapping of `Content` is governed solely by the widget's `text-wrap` /
`text-overflow` CSS rules (see `textual/content.py`, `get_rule("text_wrap", ...)`), and
`#plans-list` has no such rules in `src/sase/ace/tui/styles.tcss`. So the "never wraps"
contract is silently broken — the existing unit test asserts `label.no_wrap is True` and
passes while the real UI wraps.

## Design

Two complementary changes: a hard single-line guarantee, and more concise labels ordered
so that ellipsis truncation eats the least valuable content.

### 1. Hard no-wrap guarantee (CSS)

Add to the `#plans-list` block in `src/sase/ace/tui/styles.tcss`:

```tcss
text-wrap: nowrap;
text-overflow: ellipsis;
```

This is the established repo pattern for single-line OptionLists (e.g.
`PromptHistoryModal OptionList` and `BgCmdList` use `text-wrap: nowrap`). With this rule
every option renders exactly one line tall regardless of content length.

Keep `single_line_text()` returning `no_wrap=True` Text (harmless, and unit tests assert
label intent through it), but update its docstring to note that the widget CSS is what
actually enforces single-line rendering under Textual 8.

### 2. Concise, truncation-ordered row labels

All changes live in `src/sase/ace/tui/widgets/artifacts/plans_rendering.py` (plus the
section header in `plans_list.py`). Principle: with end-of-line ellipsis, whatever sits
after a long title is lost — so short, high-value metadata moves before the title, and
only low-value tail metadata (age, project badge) remains after it.

**Epic rows** — replace the wordy state token with a one-cell state glyph column and
move progress + state ahead of the title:

- Before: `▸ ● sase-6g <title>  8/8  LAUNCHED  12h  [sase]`
- After: `▸ ● sase-6g 8/8 ▶ <title>  12h  [sase]`

State glyph column (always exactly one cell, so titles stay roughly aligned):

| state                              | today      | new glyph | style                |
| ---------------------------------- | ---------- | --------- | -------------------- |
| blocked                            | `BLOCKED`  | `⊘`       | bold red (#FF5F5F)   |
| ready                              | `READY`    | `▷`       | bold green (#5FD787) |
| launched (`epic.is_ready_to_work`) | `LAUNCHED` | `▶`       | bold teal (#00D7AF)  |
| none                               | (absent)   | `·`       | dim placeholder      |

This trims ~11 cells per row and guarantees id, progress, and state are never truncated;
only the title tail, age, and badge can be ellipsized, and those are the pieces the
Details panel and preview already show in full.

**Phase rows** — same treatment, minimal version:

- Shrink the indent prefix from `  ↳` to `↳`.
- Drop the `active` suffix entirely — it is redundant with the `◐` in-progress status
  glyph already on the row.
- Replace the `blocked`/`ready` suffix words with the same `⊘`/`▷` glyph in a one-cell
  column between id and title (dim `·` placeholder when no state), matching epics.

**Proposal and archive rows** — already compact; leave content as-is. They inherit the
CSS single-line guarantee.

**Legend for discoverability** — since state words become glyphs, extend the Epics
section header built by `_section_option(...)` in
`src/sase/ace/tui/widgets/artifacts/plans_list.py` with a dim one-line legend, e.g.
`── Epics (229) · ⊘ blocked ▷ ready ▶ launched ──` (only for the Epics header; keep it
short enough for the pane's 56-cell minimum width). Check the `?` help modal for any
Plans row documentation and keep it in sync per the ace help-popup rule.

### Explicitly considered and rejected

- **Width-aware label building with resize-triggered option rebuilds** (exact-fit title
  truncation with right-pinned metadata): needs pane width at build time, a new
  resize→rebuild trigger, and startup width-0 fallbacks. The TUI perf memory forbids new
  refresh code paths without strong cause; the CSS + reordering approach achieves the
  same user outcome with zero new refresh machinery.
- **Rich `Table.grid` per option** for right-aligned metadata: `RichVisual` height
  measurement renders every option at rebuild time, adding per-option render cost on a
  ~250-row list for purely cosmetic alignment.
- **Python-side fixed title cap** (e.g. 60 chars): wastes space on wide panes and still
  requires the CSS guarantee on narrow ones; metadata-first ordering makes it
  unnecessary.

## Scope of changes

- `src/sase/ace/tui/styles.tcss` — `#plans-list` gains
  `text-wrap: nowrap; text-overflow: ellipsis;`.
- `src/sase/ace/tui/widgets/artifacts/plans_rendering.py` — `epic_text()`,
  `phase_text()` reordered/compacted per the spec above; `single_line_text()` docstring
  updated.
- `src/sase/ace/tui/widgets/artifacts/plans_list.py` — `_section_option()` legend for
  the Epics header.
- Tests as described below.

This is presentation-only Textual rendering, so per the Rust core backend boundary it
stays entirely in this repo — no `sase-core` changes.

## Testing

1. **Update label unit tests** in `tests/ace/tui/test_artifacts_plans.py`
   (`test_plan_list_rows_are_compact_single_line_labels`,
   `test_project_badges_render_only_for_all_projects_scope`): new expected orderings
   (`8/8 ▶` before the title), glyph selection for blocked/ready/launched/none, absence
   of `active` on in-progress phase rows, and the badge still terminating the row.
2. **Add a real wrap-regression test** — the existing unit assertions passed while the
   UI wrapped, so add a widget-level test that mounts the pane (AcePage or pilot) with
   an epic title far wider than the pane, then asserts every option occupies exactly one
   line (e.g. the OptionList's per-option content height is 1 for all options / virtual
   size height equals the option count). This is the test that would have caught the
   Textual-8 `no_wrap` regression.
3. **PNG visual snapshots** — regenerate `artifacts_plans_populated_120x40` and
   `artifacts_plans_all_projects_populated_120x40` goldens via `just test-visual` +
   `--sase-update-visual-snapshots`. If the shared fixture's epic titles are all short,
   lengthen one so the golden visibly locks in single-line ellipsis behavior.
4. `just check` before finishing (run `just install` first in a fresh workspace).

## Risks / notes

- Perf-neutral: pure label construction changes; no new refresh paths, no work added to
  render or keystroke paths (TUI perf memory reviewed).
- Glyphs `⊘ ▷ ▶ ·` render deterministically under the pinned Fira Code visual fixtures;
  the pane already uses `◆ ▸ ▾ ● ◐ ○ ▤ ↳`.
- Information loss is bounded: state words become color-coded glyphs with a legend in
  the Epics header, and the Details panel continues to show full status/readiness for
  the selected row.
- Sibling artifact lists (Bugs, Commits) are out of scope; if they exhibit the same wrap
  behavior they can get the same one-line CSS treatment in a follow-up.
