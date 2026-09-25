---
tier: tale
title: Separate the xprompt from the metadata in the collapsed agent header
goal:
  In the collapsed sticky header on the Agents tab, the xprompt preview renders as a
  distinct card with an XPROMPT tab and a uniform surface, clearly set apart from the
  metadata chip rows, while the total header height stays the same.
size: medium
proposed_by: bbugyi200.athena.0rv.w0
create_time: 2026-09-25 07:23:04
status: wip
---

# Collapsed agent header: XPROMPT tab + card to separate the prompt from the metadata

## Problem

When the sticky header above the Agents-tab data decks is collapsed, it shows two chip
rows (metadata) followed by the agent's xprompt preview. Nothing sets the two apart. The
preview starts on the row right after the chips, and its only marker is the thin purple
`▎` bar, which is easy to miss. The prompt text also has a patchy background. The
Monokai highlighter gives every token a `#272822` background, but the reflow's joining
spaces, the dim `¶` hard-break marks, and the ragged end of each row stay on the panel
background. So the preview reads as broken gray pieces glued to the metadata, not as one
block.

## Design (decided)

Turn the preview into a labeled **card** with a folder-style **tab**. This is the
callout idiom used by GitHub alerts and Obsidian callouts:

```
╭─ AGENT SHELL ────────────────────────────────────────────────────────╮
│ 0qp.f0 · CLAUDE(opus) @ xhigh · ⚡ PLAN                               │
│ ⌘ #gh · ⌘ #fork · ■ #beau · +1 · ⏳ 0qp                               │
│ [▎ XPROMPT  ]                                                          │
│ [▎ +sase #fork:0qp Can you now help me add a similar entry for be   ] │
│ [▎ glance) that a sase agent closed a bead. #beau ¶ #plan %m:opus…  ] │
╰───────────────────────────────────────────────── +3 lines · ▾ d more ─╯
```

`[ … ]` marks the card surface, a solid `#272822` background. The tab's surface covers
only `▎ XPROMPT  ` (bar, space, label, two trailing spaces for optical balance). Every
body row's surface covers the full content width, so the card is a clean rectangle with
a tab on its top-left. It sits under the chip rows, which stay on the panel background.

Details and rationale:

- **Tab label is `XPROMPT`**: bold `#AF87FF`, the xprompt accent already used by the `▎`
  bar and by `AGENT XPROMPT` in clan views. The label matches the heading that appears
  when `d` expands the header. We avoid plain `PROMPT` on purpose: the detail panel
  directly below has an `AGENT PROMPT` section, which is a different thing (the expanded
  prompt).
- **Card surface is `#272822`**: the Monokai surface that `highlight_prompt_text`
  already paints under every token. The card therefore matches the token backgrounds
  exactly, and the patchy gaps disappear. It also matches the `AGENT PROMPT` code-block
  surface in the detail panel below. Define it once as a constant, and lock it to the
  highlighter with a unit test.
- **The `▎` bar stays** in `#AF87FF` in column 0 of every card row, including the tab.
  On the surface it becomes the card's accent edge.
- **Row cost is neutral**: the tab takes one row, which comes out of the preview budget.
  The collapsed header keeps its total height cap
  (`ace.agent_header.collapsed_max_share`). The preview still gets at least one body row
  whenever the share is positive, so the tightest possible header grows by one row. That
  is acceptable.
- **The pending hold uses the same geometry**: while an xprompt paint is pending, show
  the tab plus the same number of held card rows (dim `⋯` on the first). The layout does
  not jump when the real text arrives.
- **Unchanged**: the chip rows, border title and subtitle (`+N lines · ▾ d more`), `¶`
  and `…` semantics, the expanded state (`d`), hint-forced expansion, and nodes without
  an xprompt (still exactly two chip rows). When `collapsed_max_share: 0`, no tab or
  card is shown.

This is presentation-only Textual/Rich rendering, so it stays in this repo. No
`sase-core` change is needed.

## Implementation

### 1. `src/sase/ace/tui/widgets/agent_header_preview.py` (pure module; owns all card geometry)

- Add constants: `PREVIEW_CARD_STYLE = "on #272822"` (comment: must match the Monokai
  surface `highlight_prompt_text` paints under tokens), `PREVIEW_TAB_LABEL = "XPROMPT"`,
  `PREVIEW_TAB_LABEL_STYLE = "bold #AF87FF"`, and `PREVIEW_TAB_ROWS = 1`.
- Leave `fit_xprompt_preview()` unchanged: same rows, gutters, ellipsis, and
  `hidden_lines`. Existing fit tests compare `fit.text.plain` exactly and must keep
  passing.
- Move the pending placeholder here from the panel as a pure helper, e.g.
  `pending_preview_rows(rows: int) -> Text`: gutter rows with a dim `⋯` on row one,
  matching today's `_pending_placeholder`.
- Add `preview_card(body: Text, *, width: int) -> Text`. It returns the tab row followed
  by each `\n`-separated row of `body` on the card surface:
  - Build each body row as a `Text` whose base style is `PREVIEW_CARD_STYLE`. Append the
    row's existing styled content, which already starts with the `▎ ` gutter. Then pad
    with spaces to exactly `max(width, _MIN_WIDTH)` cells, measured with `cell_len`,
    never negative, so wide glyphs work. The base style sits under the token spans, so
    token styles still win and the joins, `¶`, `…` and padding get the surface.
  - Tab row: `▎` (`PREVIEW_BAR_STYLE`) followed by `XPROMPT `
    (`PREVIEW_TAB_LABEL_STYLE`), both on the card surface. Not padded to full width.
    Crop it to the width so a very narrow panel never ellipsizes or wraps it.
  - Return a single `Text(no_wrap=True, overflow="ellipsis")`. No row may exceed the
    width.
- Make `preview_row_budget()` also subtract `PREVIEW_TAB_ROWS`: `max(1, cap - 4 - 1)`.
  Update its docstring and the `_CHROME_ROWS` comment (border + chip rows + XPROMPT
  tab).
- Update the module docstring to describe the tab + card, and export the new public
  names in `__all__`.

### 2. `src/sase/ace/tui/widgets/agent_header_panel.py`

- `_collapsed_content()`: whenever a preview is shown (pending hold or `fit.rows > 0`),
  append `"\n"` plus `preview_card(body, width=width)` to the compact rows. `body` is
  `fit.text` or `pending_preview_rows(hold)`. Remove `_pending_placeholder` and the
  now-unused `PREVIEW_BAR_*` / `_PENDING_GLYPH*` imports and constants.
- Keep `_last_preview_rows` counting **body** rows only, because the hold logic uses it.
  Set
  `_rendered_rows = _COMPACT_ROW_COUNT + (PREVIEW_TAB_ROWS + preview_rows if preview_rows else 0)`,
  so the Main-view pin-reapply in `agent_detail.py` sees the real height.
- Resize: padding depends on width. `on_resize` already repaints when collapsed, and the
  digest already includes `width`. Confirm that a width change repaints the card to the
  new width.

### 3. Tests

- `tests/ace/tui/widgets/test_agent_header_preview.py`: add card tests:
  - The tab row's plain text is `▎ XPROMPT  `, and the label is bold `#AF87FF` on the
    surface.
  - Every body row is exactly `width` cells (parametrize widths, including one with wide
    glyphs) and starts with the gutter.
  - Every character in the body rows (joins, `¶`, `…`, padding) resolves to the card
    background, and token foreground styles survive.
  - A very narrow width crops the tab without an ellipsis.
  - `pending_preview_rows` shows `⋯` on row one only.
  - `preview_row_budget` accounts for the tab row (and still returns at least 1, or 0
    when off).
  - A lock test: the `PREVIEW_CARD_STYLE` background equals the background that
    `highlight_prompt_text` puts under its tokens.
- `tests/ace/tui/widgets/test_agent_header_panel.py`: update the row math from
  `2 + _preview_rows(panel)` to `2 + 1 + _preview_rows(panel)` whenever a preview is
  shown. Assert row `[2]` is the `▎ XPROMPT` tab and rows `[3:]` start with `▎ `. This
  covers the collapsed, short-fit, overflow, toggle-back, exact-rows, pending-hold
  (tab + held rows + `⋯`), and cheap-path tests. Keep the no-preview cases (exactly two
  rows) and the `rendered_row_count` change test.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_header_preview.py`: add `XPROMPT`
  to both parametrized token tuples.

### 4. Docs and config text

- `docs/ace.md`, "Header panel" bullet (Agents-tab metadata panel section): describe the
  `XPROMPT` tab and card (Monokai surface, purple `▎` accent edge). Say that the tab row
  counts against the `collapsed_max_share` budget.
- `docs/configuration.md`, `ace.agent_header` table row: "cap minus the border, chip
  rows, and XPROMPT tab row, at least 1 row".
- `src/sase/default_config.yml` comment (around `collapsed_max_share`) and the
  `collapsed_max_share` description in `src/sase/config/sase.schema.json`: mention the
  tab row next to the border and chip rows. Change the text only; the default value
  stays the same.

### 5. PNG goldens

Many Agents-tab goldens show a collapsed header with an xprompt preview. They will
change in the header region only: the tab row is inserted, the card surface fills in,
and there is one fewer body row. Steps:

1. Run the targeted capture first:
   `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_header_preview.py`.
   Inspect both updated goldens against the design above.
2. Run the full `just fix-tui-screenshots` through `/sase_monitor` (`TESTING` /
   `TESTED`) so every affected golden is refreshed. Read the retained report. Inspect
   every creation, removal, and update group. Confirm that the only differences are in
   the collapsed header region. Generation is not approval.

## Verification

- Run `just fix` inline, then `sase tool run check`. Hand it to a verify monitor if it
  would exceed the synchronous limit.
- Take a live check with `sase screenshot` on the Agents tab, with an agent selected
  that has a multi-line xprompt. Confirm three things. The tab and card read as one
  block clearly separated from the chip rows. The surface is uniform, with no gaps at
  joins, `¶`, or row ends. `d` still expands to `AGENT XPROMPT` and collapses back to
  the card. Also take one capture at a narrow width (about 80 columns) to check the tab
  and the padding.
