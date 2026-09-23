---
tier: tale
title: Move the numbered roster sections into the Agents jump panel
goal:
  FAMILY SHELLS, NEIGHBORS, CLAN MEMBERS, and TRIBE MEMBERS leave the Agents metadata
  body; the jump panel stays unchanged when collapsed and, when expanded with `.`, shows
  those sections exactly as the metadata panel rendered them.
size: medium
proposed_by: bbugyi200.athena.0qa
create_time: 2026-09-23 15:21:25
status: wip
---

# Plan: Move the numbered roster sections into the Agents jump panel

## Context

Epic `sase-16y` added `AgentJumpPanel`, the sticky footer at the bottom of the Agents
detail column (`src/sase/ace/tui/widgets/agent_jump_panel.py`, wired by
`_agent_detail_jump.py`). The epic misread the request. The four numbered roster
sections were supposed to **move** out of the scrolling metadata body into this footer:

- `FAMILY SHELLS` (`_agent_display_family.py`)
- `NEIGHBORS` (`_agent_display_neighbors.py`)
- `CLAN MEMBERS` (`_agent_display_clan.py`)
- `TRIBE MEMBERS` (`_agent_display_tribe.py`)

All four render through `append_member_roster()` in `prompt_panel/_member_roster.py`.
The epic instead left them in the metadata body and made the footer a separate mirror.

What the user wants:

- **Collapsed footer: keep it exactly as it is.** This is the packed two-row legend
  (`JumpLegendRenderable` in collapsed mode) with its title legend and `▴ . more`
  subtitle.
- **Expanded footer (`.`): show the same content the metadata panel shows today for
  these sections.** Today it shows `_expanded_lines()`, a grid of number/label/glyph
  cells with its own headings and tail lines. The new content is the real roster block
  that `append_member_roster()` builds:
  - the accent rule and the `❖ TITLE · N` heading, with its fold glyph and any heading
    suffix;
  - the dim group labels;
  - each numbered row with chip, marks, unread and dismissed markers, label, owner
    badge, kind, status, model, duration, count chip, and fold-driven annotations;
  - nested child rows and the tribe `❯` entry cursor;
  - the `… +N more …` tail and `… +N also listed under FAMILY SHELLS` tail.
- **The metadata body no longer contains these sections.**

The change is presentation-only Textual/Rich code, so it stays in this repo (no
`sase-core` change).

## Design

### The builder writes each roster into its own `Text`

Only detached documents change: the Agents `AgentDetail` prompt panel, where
`detach_identity` is true because `AgentDetail` attaches both the identity sink and the
jump sink. There, each builder appends its roster(s) to a separate `roster_text` instead
of the body. The roster code keeps its exact rendering; only the destination `Text`
changes.

The document carrier (`AgentHeaderRenderable`) carries that text next to the jump map it
already carries. Responsive-section offsets (BEAD, PLAN, slow tools) are recorded
against the body as it is built, so they stay correct because the roster never enters
the body.

Non-detached documents keep their rosters inline, exactly as today. These are the `Z`
zoom modal (`publish_member_jump_map=False`) and direct builder callers. The `V`
metadata pager builds its own document and is untouched.

### The jump panel's expanded mode renders that text verbatim

`AgentJumpPanel` now has three modes:

- **Collapsed:** unchanged `JumpLegendRenderable` legend.
- **Narrowed:** after the first digit of a two-digit jump, the unchanged narrowed legend
  grid, whether the panel was collapsed or expanded.
- **Expanded:** the carried roster `Text`.

Title, subtitle, border accent, visibility, toggling, the bottom-pin behavior,
`max-height: 40%` with mouse-wheel scroll, and the per-session collapsed/expanded state
all stay the same. The legend's old expanded grid, and the section fields only it used,
are deleted.

### Hint mode

File-hint documents (`hint_state` set) also keep rosters out of the body. Otherwise the
body would reflow under the user the moment hints appear. The carrier still carries no
map or roster in hint mode, so the panel stays hidden, as today: digits select `[N]`
hints there, not jumps.

### Deliberate consequences, to state in docs

- Roster headings are no longer `Ctrl+J`/`Ctrl+K` stops.
- `za`/`zA`, which act on the section at the metadata viewport top, can no longer target
  a roster section or roster row. Rosters follow the global panel fold keys (`zz`, `zZ`,
  and the direct level keys). Any leftover roster override still applies until a global
  fold key clears overrides.
- `,/` metadata search no longer covers roster rows.

## Implementation

1. **`src/sase/ace/tui/widgets/prompt_panel/_member_roster.py`**
   - Remove `hidden_count` and `hidden_hint` from `_MemberJumpSection` and stop
     computing `hidden_hint` in `append_member_roster()`. Keep rendering the tail lines
     into the text exactly as now; the roster text now carries them verbatim.
   - Add `detached_roster_text(text: Text) -> Text | None`. It returns `None` when
     `text.plain.strip()` is empty. Otherwise it returns the text with leading and
     trailing blank lines removed. Keep spans: slice the `Text` or rebuild spans the way
     `strip_leading_document_chrome()` does, and never go through `.plain`. This way the
     panel does not start or end on an empty row.
2. **`_agent_display_header_renderable.py`**
   - Add a `_member_roster` slot and a `member_roster` property.
   - Change `with_member_jump_map(member_jump_map, *, roster: Text | None = None)` to
     set both fields together.
   - The roster must not affect `plain`, `spans`, `__rich_console__`, or the digest.
3. **`_identity_header.py`**
   - Add `find_member_roster(content) -> Text | None` over the existing `_find_carrier`
     walker, and export it.
   - Change `MemberJumpMapSink` to
     `Callable[[MemberJumpMap | None, Text | None], None]`.
4. **Builders.** In every builder below, `roster_text` exists only when
   `detach_identity` is true, and the destination is chosen with an explicit `is None`
   test: `roster_text if roster_text is not None else header_text`. **Never use `or`
   here:** an empty Rich `Text` is falsy, so `roster_text or header_text` would silently
   send the roster back into the body.
   - `build_header_text()` (`_agent_display_header.py`)
     - Build `roster_text` as above.
     - Pass the destination to both `append_family_member_roster()` and
       `append_lane_neighbors_section()`. Family comes first, then neighbors, which
       matches the shared numbering order.
     - On the detached return, keep the existing `hint_state is None` and
       targets-or-sections guard, and call
       `carrier.with_member_jump_map(jump_map, roster=detached_roster_text(roster_text))`.
   - `build_clan_detail_text()` (`_agent_display_clan.py`): same pattern for the
     `CLAN MEMBERS` roster.
   - `build_tribe_detail_text()` / `_append_tribe_body()` (`_agent_display_tribe.py`)
     - Add a `roster_text: Text | None` parameter to `_append_tribe_body()`, used only
       for the `append_member_roster()` call.
     - `CLAN SUMMARIES` and `PROMPTS` keep their `unit_numbers` references in the body.
     - The detached non-cheap path passes a fresh `Text` and attaches it with the map.
     - The cheap path and the non-detached path are unchanged.
5. **`prompt_panel/__init__.py` (`AgentPromptPanel`)**
   - In `update()`, still before the digest early return, store
     `_member_roster_last_published = find_member_roster(content)` next to
     `_jump_map_last_published`, and call `jump_sink(jump_map, roster)`. A roster-only
     change can leave the body digest unchanged; it must still reach the panel.
   - `inline_document_renderable()` (the zoom-modal seed) appends the last published
     roster after the content, separated by one blank line, when there is one. That
     keeps the zoom modal's first paint from losing the rosters.
6. **`_agent_detail_jump.py`**: `_on_member_jump_map(jump_map, roster=None)` forwards
   the roster to `panel.show_jump_map(jump_map, roster=roster)`.
7. **`agent_jump_panel.py`**
   - `show_jump_map(jump_map, roster: Text | None = None)` stores both before any digest
     early return. That way a roster that changes while the panel is collapsed is
     current the moment the user expands it.
   - Choose what the content `Static` shows:
     - a pending prefix → `JumpLegendRenderable(map, mode=prefix)`;
     - expanded with a roster → the roster `Text`;
     - otherwise → `JumpLegendRenderable(map, mode="collapsed")`. This is also the
       defensive fallback for a map with no roster.
   - Paint digest:
     - always include the legend digest, which covers the title inputs;
     - in expanded mode, add `renderable_content_digest(roster)`, so status and duration
       changes repaint;
     - keep mode, subtitle, and prefix in it as now.
   - `toggle_expanded()` and `set_pending_prefix()` keep forcing a repaint.
   - Rename `_effective_mode()` if its values change, and keep it private.
8. **`_agent_jump_legend.py`**
   - Delete the expanded mode: `_expanded_lines()`, `"expanded"` in `_NARROW_MODES`, and
     the section `hidden_*` parts of `plain`.
   - Drop constants that become unused (for example `_DIM_ITALIC_STYLE`).
   - Update the docstrings: the renderable now has a collapsed mode and a narrowed mode.
9. **`_agent_display_neighbors.py`**: change `hidden_tail_hint` from
   `"zz / za to show more"` to `"zz to show more"`. `za` can no longer reach this
   section from the Agents detail.
10. **Docs (`docs/ace.md`)**
    - The `0`–`9` keybinding row (currently "mirrored in the jump panel") and the
      numbered-roster paragraph after it (currently "Every live numbered target is also
      listed in the sticky jump panel…"): the rosters now **live** in the jump panel.
    - The **Jump panel** bullet under "Agents Tab Metadata Panel":
      - collapsed is unchanged;
      - `.` expands to the full roster sections exactly as they used to render in the
        metadata body;
      - the metadata body no longer contains them;
      - add the three consequences from Design (no `Ctrl+J` stops, global folds only,
        not searched by `,/`).
    - "Sase Agent Neighbors Section"
      - Replace the "carries a numbered `NEIGHBORS` roster in its metadata region" and
        "sits directly below `WORKFLOW VARIABLES`…" placement sentences with the jump
        panel placement: after `FAMILY SHELLS` when both exist.
      - Update the tail text to `zz to show more`.
    - Grep `docs/` for other `FAMILY SHELLS` / `CLAN MEMBERS` / `TRIBE MEMBERS`
      placement claims about the Agents metadata body and fix them.
    - The help modal's existing "Expand / collapse jump panel" row stays accurate. There
      are no keymap or `default_config.yml` changes.

## Tests

Update the builder and carrier tests,
`tests/ace/tui/widgets/test_member_jump_sections.py` and `test_identity_header.py`:

- Detached documents have no roster in the body. Check each roster kind: family
  container, family member shell, agent with neighbors (including a dismissed and a
  suppressed-sibling tail), clan, and full tribe. Each body `plain` has none of
  `❖ FAMILY SHELLS`, `❖ NEIGHBORS`, `❖ CLAN MEMBERS`, `❖ TRIBE MEMBERS` or numbered
  roster rows, and `find_member_roster()` returns them.
- Parity with the inline document:
  - For single-roster documents (clan, tribe, member shell), the non-detached document's
    `plain` contains `roster.plain`.
  - For family plus neighbors, each section's block (from its rule through its tail)
    appears in the non-detached `plain`, family before neighbors.
  - The roster keeps the `bold black on {accent}` chip spans.
- A detached family document with a bead or plan summary still renders its responsive
  sections at the right offsets, and its body contains no roster.
- Hint-mode detached documents carry neither a map nor a roster, and their body has no
  roster.
- The tribe cheap paint carries no roster. Non-detached documents still render rosters
  inline (existing tests).
- `test_clan_detached_body_starts_at_roster` is rewritten: the detached clan body no
  longer starts at the roster.
- Delete or rewrite the `hidden_count`/`hidden_hint` assertions to check the tail lines
  in `roster.plain` instead.

Prompt-panel sink test (mirror
`test_panel_sink_receives_jump_map_before_digest_return`):

- The sink receives `(map, roster)` on each update, and `(None, None)` for empty
  documents.
- A second update whose body is identical but whose roster differs (for example a
  neighbor status change) still delivers the new roster.
- `inline_document_renderable()` includes the roster.

Legend tests (`test_agent_jump_legend.py`):

- Remove `test_expanded_shows_every_target_with_headings_and_tails` and
  `test_expanded_single_section_has_no_heading`.
- Update the digest test for the removed section fields.
- Collapsed and narrowed tests stay unchanged.

Panel pilot tests (`test_agent_jump_panel.py`, `_DetailApp` pattern):

- The collapsed view is unchanged, and all existing collapsed, visibility, and narrowing
  tests stay green.
- `.` expands to the roster: the jump content equals the carried `roster.plain`,
  including headings, row fields, tails, and the dismissed `⊘` and `dismissed`
  annotation. `.` again restores the legend.
- The metadata prompt panel contains no roster for every roster kind.
- When expanded, a roster change with an unchanged body repaints the panel. A change
  made while collapsed is visible as soon as the panel expands.
- A global fold change (`panel_fold_level`) is reflected in the expanded panel, for
  example NEIGHBORS going from 3 rows to all rows.
- Narrowing from the expanded view shows the grid, and cancelling restores the roster
  view.
- A hint document hides the panel and its body has no roster.

Other suites that read roster text from the metadata body or navigate to it must now
read it from the jump panel, expanding it where they need rows or tails:

- `tests/ace/tui/test_agents_panel_fold_mounted.py`: the `ctrl+j` → `members` and `za`
  steps on roster anchors. Drop those steps and keep the non-roster fold assertions.
- `tests/ace/tui/widgets/test_agent_tribe_summary.py` (`TRIBE MEMBERS · 12`).
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py`
  - the `NEIGHBORS` before `SASE CONTEXT` order assertion is gone;
  - `also listed under FAMILY SHELLS` needs the expanded panel.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py` (`TRIBE MEMBERS · 2`).
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_jump_panel.py`: the expanded
  snapshots now show the roster.
- Run the scoped suite and fix any other section-navigation or body-content assertion
  that assumed detached rosters.

## Visual verification

Read `sase/memory/tui_screenshot.md` first.

1. **Live screenshots.** Take a `sase screenshot --keep` session and drive it through:
   - a family container;
   - a family member shell;
   - an agent with many neighbors (two-digit numbering);
   - a clan and a tribe panel;
   - `.` expanded;
   - a first digit pressed while expanded;
   - the LLM Calls and file layouts;
   - a narrow terminal.

   In each PNG, confirm:
   - the metadata body has no roster;
   - the collapsed footer looks exactly as before;
   - the expanded footer matches the old in-body roster rendering (rules, headings,
     chips, colors, tails), aligned with the header and body text columns, and starts
     without an empty row;
   - the 40% cap and wheel scrolling behave for long rosters.

2. **Goldens.** Run `just fix-tui-screenshots` through `/sase_monitor`.
   - Expected changes:
     - every Agents golden whose selected node has a roster loses those sections from
       the metadata body;
     - the expanded jump-panel goldens show the roster;
     - collapsed jump-panel rows are unchanged.
   - Inspect every creation, removal, and update group before finalizing. Any other diff
     is a bug.

## Verification

Run `sase tool run check` (after `just fix`). Do not run `just check-full`.

## Out of scope

- The collapsed legend layout and the narrowing grid.
- Keyboard focus or keyboard scrolling of the jump panel.
- Per-row fold toggles inside the footer.
- The `Z` zoom modal's own inline document and the `V` metadata pager.
