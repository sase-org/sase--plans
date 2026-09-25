---
tier: tale
title: Cap the collapsed agent-header xprompt preview at 3 rows
goal:
  The collapsed Agents-tab sticky header previews at most 3 rows of the selected agent's
  prompt (configurable), truncating with a word-boundary ellipsis and a correctly
  pluralized hidden-line hint, while d still expands to the full prompt.
size: medium
proposed_by: bbugyi200.athena.0rv.w0.f0
create_time: 2026-09-25 10:25:35
status: wip
---

# Collapsed agent header: cap the xprompt preview at 3 rows

## Problem

The collapsed sticky header above the Agents-tab data decks previews the selected
agent's xprompt. It fills whatever `ace.agent_header.collapsed_max_share` of the column
allows (about a third). On a tall terminal that is 10 to 30 rows of prompt, pushing the
decks down. The user wants a glance, not a document: at most **3 rows** of prompt while
collapsed, truncated after that. `d` still expands to the whole prompt.

## Relationship to the XPROMPT card

This builds on the XPROMPT card (`plan:202609/agent_header_xprompt_card.md`). The card
puts the preview in a card with an `XPROMPT` tab row above a Monokai surface. When this
plan was written, the card was implemented but **not yet on master**. The row cap does
not depend on the card, so implement it against whatever master has:

- **Card on master** (`preview_card` / `PREVIEW_TAB_ROWS` exist in
  `agent_header_preview.py`): the cap limits the card **body** rows. The tab row is a
  label, not prompt, so it does not count toward the 3. Collapsed header: border 2 +
  chips 2 + tab 1 + body ≤ 3 = at most 8 rows.
- **Card not on master**: the cap limits the quote-barred preview rows. Do **not**
  implement the card here.

Line numbers and exact doc wording below follow pre-card master. Apply each change by
what it means, not by the literal text.

## Design (decided)

- **Unit**: 3 on-screen rows of reflowed prompt (what the user sees as "lines"), not 3
  source lines. The reflow joins soft breaks, so a short multi-line directive prompt
  still fits in one row.
- **Two caps; the smaller one wins**: body rows =
  `min(collapsed_preview_max_rows, share-derived budget)`. The share budget still
  protects short columns: a 20-row column stays under its share, and the at-least-1-row
  floor stays. On any normal terminal the 3-row cap is the one that applies.
- **Configurable, default 3**: new key `ace.agent_header.collapsed_preview_max_rows`. It
  is an integer, at least 1, default `3`, and has no upper bound. A huge value brings
  back the old fill-the-share behavior, because the share still caps it. `0` is **not**
  an off switch here: `collapsed_max_share: 0` stays the one way to turn the preview
  off, so the two keys have separate jobs. The name says "max" because it is a ceiling
  the preview may stay under, not a fixed height.
- **Truncation keeps the existing language**: the fit already ends the last shown row
  with a dim `…` after the last whole word that fits, and the border subtitle reads
  `+N lines · ▾ d more`. Truncation now happens on most real prompts, so fix the
  subtitle grammar: `+1 line` (singular) vs `+N lines`. The count keeps its current
  meaning (source lines not fully shown).
- **Short prompts don't grow**: a prompt that needs 1 or 2 rows shows 1 or 2 rows. A
  prompt that needs exactly 3 rows shows all 3 with no `…` and no `+N` count.
- **Unchanged**: the expanded state (`d`), hint-forced expansion, the pending-hold
  geometry (it is already `min(last rows, budget)`, so it never exceeds the cap), nodes
  without an xprompt (two chip rows), and `collapsed_max_share` semantics and default.
- Presentation-only Textual/Rich behavior, so it stays in this repo. No `sase-core`
  change.

## Implementation

### 1. Settings: `src/sase/ace/tui/agent_header_settings.py`

- Add `DEFAULT_COLLAPSED_PREVIEW_MAX_ROWS = 3` and field
  `collapsed_preview_max_rows: int = DEFAULT_COLLAPSED_PREVIEW_MAX_ROWS` on
  `AgentHeaderSettings`.
- Parse it in `parse_agent_header_settings` with a new `_coerce_rows`. Accept a non-bool
  `int >= 1`, or an integral `float >= 1` coerced to `int` (JSON Schema `integer`
  accepts `3.0`). Anything else falls back to the default: bools, strings, `None`, `0`,
  negatives, non-integral floats, NaN, inf. Each key falls back on its own, so a bad
  `collapsed_max_share` does not reset a good row cap, and the reverse holds too. Update
  the docstring and `__all__`.

### 2. Budget: `src/sase/ace/tui/widgets/agent_header_preview.py`

- Change `preview_row_budget(column_rows, share)` to
  `preview_row_budget(column_rows: int, share: float, *, max_rows: int) -> int`. Make
  `max_rows` a required keyword so every caller passes the cap explicitly. Keep the
  current share math, then return `min(max_rows, max(1, cap - _CHROME_ROWS))`. A
  non-positive `max_rows`, `share`, or `column_rows` returns `0`. Update the docstring:
  the smaller of the two caps wins, and the share alone gives the at-least-one-row
  floor.
- Leave `fit_xprompt_preview` alone. A smaller `max_rows` only makes it cheaper.

### 3. Panel: `src/sase/ace/tui/widgets/agent_header_panel.py`

- `_preview_budget()`: pass `max_rows=settings.collapsed_preview_max_rows`.
- `_subtitle_for()`: use `line` when `hidden_lines == 1`, otherwise `lines`
  (`+1 line · ▾ d more`, `+12 lines · ▾ d more`).
- Nothing else changes. The budget is already in the repaint digest, so changing the
  setting or resizing repaints correctly. `_rendered_rows` and the Main-view pin reapply
  already follow the shown row count.

### 4. Config surface

- `src/sase/default_config.yml`, `ace.agent_header`: add `collapsed_preview_max_rows: 3`
  with a comment. The collapsed header previews at most this many rows of the selected
  agent's xprompt, and fewer when `collapsed_max_share` leaves less room. Minimum 1. `d`
  shows the whole prompt.
- `src/sase/config/sase.schema.json`, `ace.agent_header.properties`: add
  `collapsed_preview_max_rows` with `"type": "integer"`, `"minimum": 1`, `"default": 3`,
  and a matching description. Keep `additionalProperties: false`. Update the
  `collapsed_max_share` description to say the preview also stops at
  `collapsed_preview_max_rows`.

### 5. Docs

- `docs/configuration.md`, `#### ace.agent_header`: add the key to the YAML example and
  a table row: `collapsed_preview_max_rows` | integer | `3` | "Most xprompt preview rows
  the collapsed header shows (at least 1). The `collapsed_max_share` budget can lower it
  on short columns. `d` expands to the full prompt." Update the section intro so it
  covers both keys. Re-align the Markdown table (`just fix` / mdformat).
- `docs/ace.md`, the "Header panel" bullet of the Agents-tab metadata panel section:
  replace "takes as many rows as the `collapsed_max_share` budget allows (about a third
  of the column by default…)". The new text: the preview shows at most three rows
  (`ace.agent_header.collapsed_preview_max_rows`), fewer when the `collapsed_max_share`
  height budget of a short column is tighter (`0` still turns the preview off). Change
  the subtitle example to `+N lines · ▾ d more` (`+1 line` when one line is hidden).

### 6. Tests

- `tests/ace/tui/test_agent_header_settings.py`:
  - Default is `3`.
  - Valid values are kept: `1`, `3`, `12`, `3.0` → `3` as `int`.
  - Invalid values fall back to `3`: `0`, `-1`, `2.5`, `True`, `"3"`, `None`, NaN, inf.
  - Each key falls back on its own.
  - Schema/default-config parity: `default_config["ace"]["agent_header"]` equals both
    keys; the property is `integer`, has minimum 1, and its default equals the Python
    default. The validator accepts 1 and 3 and rejects 0, `"3"`, `2.5`, and `True`.
- `tests/ace/tui/widgets/test_agent_header_preview.py`: `test_preview_row_budget` now
  passes `max_rows`. Keep the existing share-geometry rows with a large `max_rows` (e.g.
  `1000`) so the share math stays covered. Add capped rows: a tall column with
  `max_rows=3` gives `3`. A short column whose share gives fewer than 3 gives the share
  value. `max_rows=1` gives `1`. `max_rows=0` or a negative value gives `0`. Share `0`
  gives `0` even with a positive cap.
- `tests/ace/tui/widgets/test_agent_header_panel.py`:
  - `test_preview_row_count_matches_budget_and_short_prompt_fits`: expected is
    `preview_row_budget(..., 0.35, max_rows=3)`, and assert it is `<= 3`.
  - Rework `test_column_resize_changes_budget`. It currently expects a height-100 column
    to show more rows and no `lines` subtitle, which is now wrong. New test: on a tall
    column the long prompt shows exactly 3 body rows, the last ends in `…`, and the
    subtitle has `lines · `. On a very short column it drops below 3 but not below
    1.
  - New: a prompt that needs exactly 3 rows at the test width shows all 3, with no `…`
    and no `lines` in the subtitle.
  - New: `AgentHeaderSettings(collapsed_preview_max_rows=5)` on a tall column shows 5
    rows; `=1` shows 1 row.
  - New: the subtitle is singular for one hidden line (`+1 line · `) and plural for
    more. Test `_subtitle_for` directly or through a prompt built to hide exactly one
    line.
  - All other row math stays as is. It is written in terms of `_preview_rows(panel)`,
    which counts body rows. If the card is on master, the `+ 1` tab-row terms stay.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_header_preview.py`: in the
  `truncated` case, also assert that the preview shows exactly 3 body rows
  (`panel._last_preview_rows == 3`, with `# noqa: SLF001` as elsewhere). Re-check the
  token tuple: with only 3 rows at 160 columns, `¶` may no longer be visible. Keep only
  tokens the 3-row preview really shows (`▎`, `sticky header`, `…`; plus `XPROMPT` if
  the card is on master).

### 7. PNG goldens

Any golden with a collapsed header previewing a prompt longer than 3 rows changes, in
the header region only. The preview gets shorter, so the decks below move up. The
`agents_header_preview_truncated_160x50` golden changes for sure. The `fits` golden
should not.

1. Run the targeted capture first:
   `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_header_preview.py`.
   Inspect it: 3 body rows, `…` on row 3, and `+N lines · ▾ d more` on the border.
2. Run the full `just fix-tui-screenshots` through `/sase_monitor`, because it is long.
   Read the retained visual report and inspect every creation, removal, and update
   group. The only acceptable differences are a shorter collapsed-header preview and the
   decks shifting up by the freed rows. Expand any group that shows something else.
   Generating goldens does not approve them.

## Verification

- Run `just fix`, then `sase tool run check`. If check would exceed the synchronous
  limit, hand it to a monitor. Fix everything it reports.
- Take a live `sase screenshot` on the Agents tab on a tall terminal, with an agent
  selected that has a long multi-paragraph xprompt. Confirm: the preview stops at 3
  rows; the last row ends in `…` at a word boundary; the subtitle reads
  `+N lines · ▾ d more`; `d` expands to the full `AGENT XPROMPT` and collapses back to 3
  rows. Take a second capture with a short prompt (1 to 2 rows) and confirm the header
  does not reserve empty rows.
