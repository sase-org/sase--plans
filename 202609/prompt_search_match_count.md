---
tier: tale
title: Prompt search match-count pill
goal:
  While prompt-input search is active, the prompt bar always shows a reliable,
  theme-aware stack-global match count (for example 3/10) that tracks n/N and disappears
  with the highlights.
size: medium
proposed_by: bbugyi200.athena.0lz
create_time: 2026-09-16 12:16:59
status: wip
---

# Plan: Prompt Search Match Count Pill (`3/10`)

## Goal

Make prompt-input search (`/`, `?`, `n`, `N`, `*`, `#`, `g*`, `g#`) always show how many
matches exist and which one is selected, for example `3/10`. The indicator must be:

- **Intuitive:** it sits where a vim user looks for a ruler, it uses the same colors as
  the match highlights it describes, and its numbers match what `n`/`N` will do.
- **Reliable:** it is derived from the pane's live highlight state, so it can never
  outlive the highlights or show a count for text that has changed.
- **Beautiful:** a compact two-tone pill that echoes the in-text highlight colors and
  degrades gracefully on narrow terminals.

## Current State (what exists today)

- `src/sase/ace/tui/widgets/_prompt_search.py` (`PromptSearchMixin`) owns the typing
  phase. `/` opens the `#prompt-search-command` panel above the prompt stack. It
  previews matches in the **active pane only** and renders `[i/N]` through
  `render_search_command_line` (`src/sase/ace/tui/widgets/search_command_line.py`),
  using `len(self._search_match_spans)`, so the count is **pane-local**.
- On `Enter` the panel hides and the highlights stay, but **no counter is shown
  anywhere**. `n`/`N`/`*`/`#` go through
  `PromptInputBarSearchMixin.repeat_prompt_search`
  (`src/sase/ace/tui/widgets/_prompt_input_bar_search.py`), which resolves the
  destination **across every pane in the stack** and gives no count feedback. Only wrap
  toasts appear.
- Highlight state lives in `SearchHighlightMixin`
  (`src/sase/ace/tui/widgets/_search_highlight.py`): `_search_match_spans` and
  `_search_current_match_index`. `_set_search_highlights` and `_clear_search_highlights`
  are the only mutators. The overlay styles are `search.match` (theme `accent`
  background) and `search.current` (theme `warning` background, bold).
- The bar's bottom border subtitle is composed by `_render_subtitle`
  (`src/sase/ace/tui/widgets/_prompt_input_bar_completion_panel.py`) as
  `<mode hints>  ·  Ln <l>, Col <c>`. It is width-aware (the readout wins, hints
  truncate first) and re-rendered on every cursor move through
  `refresh_cursor_readouts()`.
- Existing bug: `show_search_command_line` sets
  `panel.border_subtitle = "[enter] accept  [esc/^c] cancel"` as a markup string. Rich
  parses and drops `[enter]`, so the panel shows only `accept  [esc/^c] cancel` (visible
  in the `prompt_search_highlight_120x40` PNG golden).
- Known staleness hole: a programmatic `load_text()` does not clear search highlights
  (see `test_repeat_search_reports_not_found_after_buffer_changes`), so spans can
  outlive their text.

## Design

### 1. Where it lives: the bottom-border ruler, next to `Ln, Col`

After a search is committed, the active pane's count appears as a pill on the bar's
bottom border, just left of the cursor readout:

```
╰── [Esc] clear  [i] insert  [g<enter>] send  [^S] stash  [^C] cancel  ·  ▌/alpha ▌ 2/3 ▌  ·  Ln 2, Col 13 ─╯
```

Why this spot:

- It is the vim/statusline convention: the search count sits in the ruler area, beside
  the cursor position.
- It is always visible and costs no vertical space. Keeping the 4-row search panel open
  after `Enter` would cost that space.
- It rides an existing width-aware, per-cursor-move render path, so no new refresh
  machinery is needed.

**Single-owner rule:** while the `#prompt-search-command` panel is visible (typing
phase), the panel owns the count and the border pill is hidden. When the panel hides on
`Enter`, the same numbers move to the border pill. The two surfaces never show a count
at the same time.

### 2. What it looks like: a two-tone pill that mirrors the highlights

Two adjacent segments with no gap between them. Mockups were rendered in flexoki (the
ACE default), textual-dark, and textual-light.

- **Query segment** `/alpha`: background is the theme `accent` darkened 25%
  (`textual.color.Color.darken(0.25)`); foreground is that background's
  `get_contrast_text(1.0)`; the sigil is bold. It echoes the purple/orange "other match"
  highlight (`search.match`). The darkening keeps the two segments distinct in themes
  where `accent == warning` (textual-light).
- **Count segment** `2/3`: background is the theme `warning`; foreground is
  `get_contrast_text(1.0)`; bold. It echoes the gold "current match" highlight
  (`search.current`), so the eye links "2" to the gold word in the text.
- **Sigil** tells how the search was made: `/` forward, `?` reverse, `*` whole-word
  forward (`*`), `#` whole-word reverse (`#`). Non-whole-word word searches (`g*`, `g#`,
  VISUAL `*`/`#`) use `/` / `?`. The sigil comes from the recorded search, not from the
  last `n`/`N` key.
- **Query display:** newline → `↵`, tab → `⇥`, other control characters → `·`. End-elide
  with `…` to at most 24 terminal cells (use `rich.cells.cell_len`, so CJK text is
  measured correctly).
- Theme colors are read at render time from `app.current_theme`. Missing attributes fall
  back the way `PromptInputBarTitleMixin._theme_color` does (warning `#FFA62B`, accent
  `#6B4FBB`).
- Add a short comment tying these roles to `SearchHighlightMixin`'s `search.match` /
  `search.current` registration, so a future palette change updates both.

The typing panel's right side uses the **same count segment** (count only, since the
query is already on the panel's left) instead of `[i/N]`. The count visibly "moves" from
the panel to the border on `Enter`. The other shared `render_search_command_line` hosts
(pager, zoom panel, agent metadata search) keep `[i/N]` unchanged.

### 3. What the numbers mean: stack-global, matching `n`/`N`

`n`/`N` walk every non-auxiliary pane in stack order, so the count is **stack-global**
in both phases:

- **Committed / repeat** (`n`, `N`, `*`, `#`): ordinal is the 1-based
  `destination_index` in `_resolve_prompt_search_destination`'s candidate list; total is
  `len(candidates)`.
- **Typing:** the preview still moves only within the active pane (unchanged). Its local
  selection is converted to a stack ordinal: matches in earlier panes plus
  `local_index + 1`. The total is the sum over all panes. Count the other panes with the
  same matcher (`find_search_matches`, `smartcase=True`, `whole_word=False`) and reuse
  the origin pane's spans instead of rescanning it.
- If the active pane has no match but other panes do, the typing panel shows
  `no match in this pane · N in stack` in a dim warning color, because `Enter` will
  still cancel (existing behavior, unchanged). If nothing matches anywhere, it keeps
  `pattern not found`.
- If the origin pane is not in the stack snapshot (for example an auxiliary
  snippet/mini-xprompt pane), fall back to pane-local numbers.
- For a single pane, stack-global equals pane-local, so current single-pane numbers do
  not change.
- The pill keeps describing the highlighted current match even after the cursor moves
  away with `j`/`w`/etc. The gold highlight stays on that match, so the two stay
  consistent. Do not recompute from the cursor.
- No wrap marker in the pill; the existing wrap toasts remain the wrap feedback.

### 4. Reliability model: derive from highlight state, never sync separately

- Add a frozen dataclass `PromptSearchReadout` holding:
  - `query: str`
  - `direction: SearchDirection`
  - `whole_word: bool`
  - `ordinal: int | None` (1-based stack ordinal; `None` means no selected match)
  - `total: int` (stack total)
  - `pane_text: str`, declared `field(compare=False, repr=False)`: the exact pane text
    the spans were computed from.
- The readout is stored **on the pane** next to its spans, in `SearchHighlightMixin`, as
  `_search_readout: PromptSearchReadout | None`:
  - `_set_search_highlights(spans, current_index=None, *, refresh=True, readout=None)`
    stores it.
  - `_clear_search_highlights` resets it to `None`.
  - Because these two methods are the only span mutators, the pill cannot outlive the
    highlights. Esc, insert-mode entry, keyboard edits, deletes, undo, format, pane
    switches, stack rebuilds and a new `/` already clear highlights through them.
- When the readout value changes (dataclass equality, ignoring `pane_text`), the mixin
  calls a host hook `_on_search_readout_changed()`.
  - Add an inert default on `VimTextArea` next to the existing
    `_clear_search_highlights` default ("Default: no-op"). Do not define it on
    `SearchHighlightMixin`: that would shadow later mixins in the MRO.
  - Override it in `PromptTextAreaBarMixin` (`_prompt_text_area_bar.py`, the home for
    bar-routed hooks) to call `bar.refresh_search_readout()` when that method exists.
- `SearchHighlightMixin._app_theme_changed` also fires the hook when a readout is
  present, so pill colors follow theme switches.
- **Text-divergence safety net:** in
  `PromptInputBarStackLifecycleMixin.on_text_area_changed`
  (`_prompt_input_bar_stack_lifecycle.py`), when `event.text_area` is a `PromptTextArea`
  with a non-`None` `_search_readout` and `text_area.text != readout.pane_text`, call
  `text_area._clear_prompt_search(clear_highlights=True)`.
  - Compare against the snapshot text, not "any Changed means stale". A `Changed`
    message queued by an edit made _before_ the search started can reach the bar after
    the search commits. With identical text it must be a no-op.
  - This also fixes the `load_text()` staleness hole.
- **Pill visibility rule** in `_render_subtitle`: show it only when
  `active_text_area()._search_readout` is not `None`, its `ordinal` is not `None`, and
  `not self._search_command_visible`. Highlights only ever live on the active pane,
  because `focus_item` clears the old active pane first.

### 5. Narrow terminals: degradation ladder

`_render_subtitle` composes `base · pill · readout`, using the existing dim
`_CURSOR_READOUT_DIVIDER` between parts. The cursor readout keeps its current top
priority. Try each rung in order:

1. Everything fits: `base · pill · readout`.
2. Otherwise truncate `base` with `…` while at least 1 base cell remains.
3. Otherwise drop `base`: `pill · readout`.
4. Otherwise use the count-only pill: `2/3 · readout`.
5. Otherwise fall back to today's readout-only behavior.

## Implementation Steps

1. **New pure module** `src/sase/ace/tui/widgets/_prompt_search_readout.py`. No widget
   access, so it is unit-testable. It contains:
   - `PromptSearchReadout` (above).
   - `readout_sigil(direction, whole_word) -> str`.
   - `display_query(query, max_cells=24) -> str` (sanitize and elide).
   - `stack_match_position(counts_by_pane: Sequence[int], origin_position: int, local_index: int | None) -> tuple[int | None, int]`,
     returning `(ordinal, total)`.
   - `search_readout_colors(theme) -> tuple[Style, Style, Style]` (query segment, sigil
     emphasis, count segment), with fallbacks.
   - `format_search_count_segment(ordinal, total, *, theme) -> Text`.
   - `format_search_readout(readout, *, theme, include_query=True) -> Text`.

   Keep this presentation logic in Python. It is TUI-only prompt-stack rendering, not
   shared backend logic, so it does not cross the sase-core boundary.

2. **`_search_highlight.py`:**
   - Add the `_search_readout` field and the `readout=` kwarg.
   - Reset the readout in `_clear_search_highlights`.
   - Fire `_on_search_readout_changed()` on change.
   - Fire it from `_app_theme_changed` when a readout is present.
   - `_preview_search_query_highlights` keeps working without a readout.
   - Existing tests that call `_set_search_highlights(spans, 0)` positionally must keep
     passing.

3. **`vim_text_area.py`:** add the inert `_on_search_readout_changed` default.
   **`_prompt_text_area_bar.py`:** add the override that routes to
   `bar.refresh_search_readout()`.

4. **`_prompt_search.py` (typing phase):**
   - In `_update_prompt_search_preview`, capture `text = self.text` once and compute the
     local spans/selection from it.
   - Ask the bar for the stack position with a new
     `bar.prompt_search_stack_position(origin, local_spans, local_index, query, *, whole_word, smartcase)`.
     If the method is missing, fall back to local numbers.
   - Build a `PromptSearchReadout` and pass it through
     `_apply_prompt_search_result(..., readout=)` and
     `_set_search_highlights(..., readout=)`.
   - Make `_render_prompt_search_command_line` pass the readout, not local index/total,
     to `bar.show_search_command_line(direction=..., query=..., readout=...)`.
   - `_apply_prompt_search_result` gains a keyword-only `readout` parameter.

5. **`_prompt_input_bar_search.py`:**
   - Add `prompt_search_stack_position`. Scan the non-origin panes like
     `_prompt_search_pane_snapshot` does, but don't rescan the origin. Delegate the
     arithmetic to `stack_match_position`. Fall back to local numbers when the origin
     isn't in the snapshot.
   - Add `ordinal` and `total` fields to `_PromptSearchDestination`. In
     `repeat_prompt_search`, build the readout from the register (`query`, `direction`,
     `whole_word`) and the destination, with `pane_text` from the snapshot. Add `text`
     to `_PromptSearchPaneSnapshot` so the spans and snapshot text match exactly. Pass
     it to `target._apply_prompt_search_result(..., readout=)`.
   - In `show_search_command_line`:
     - Take `readout`.
     - Build the right-hand status: the count segment; `pattern not found` (existing dim
       red) when the total is 0; or `no match in this pane · N in stack` when the
       ordinal is `None` and the total is greater than 0.
     - Pass it to `render_search_command_line(..., status=...)`.
     - Set `panel.border_subtitle` to a `rich.text.Text` so `[enter]` renders literally
       (fixes the bug).
     - Re-render the bar subtitle, so an old pill hides.
   - `hide_search_command_line` re-renders the bar subtitle after clearing
     `_search_command_visible`, so the pill appears on `Enter`.
   - Add `refresh_search_readout()`, which just sets
     `self.border_subtitle = self._render_subtitle(self._subtitle_base)`. Keep it cheap
     and synchronous, with no I/O (tui_perf rules 1 and 11).

6. **`search_command_line.py`:** add an optional keyword `status: Text | None = None`.
   When it is given and the query is non-empty, it replaces the default right-hand
   `[i/N]` / `pattern not found` segment. Default behavior for the other hosts must stay
   byte-identical.

7. **`_prompt_input_bar_completion_panel.py`:** extend `_render_subtitle` with the pill
   and the degradation ladder (Design §5). Keep the docstring's priority contract
   updated.

8. **`_prompt_input_bar_stack_lifecycle.py`:** add the text-divergence safety net to
   `on_text_area_changed` (Design §4).

9. **Docs:**
   - In `docs/ace.md`, add a `### Prompt Search` subsection right after
     `### Cursor Readout` under `## Prompt Input Widget`. Describe:
     - `/` / `?` incremental search with the live count in the search panel.
     - `Enter` / `Esc` / `Ctrl+C`.
     - `n` / `N` repeating across every pane in stack order, with wrap toasts.
     - `*` / `#` / `g*` / `g#` and VISUAL `*` / `#`.
     - The two-tone match-count pill: position, colors, sigils, stack-global numbering,
       when it disappears (Esc, INSERT, edits, pane switch, new search), and the
       narrow-terminal ladder.
   - Update the `### Cursor Readout` paragraph so it says the readout still wins width
     over the pill.
   - Run `just fmt` (it formats Markdown).

## Tests

Put new tests in a new file, `tests/ace/tui/widgets/test_prompt_search_readout.py`.
`test_prompt_search_interactive.py` is already at 664 lines, and the `toobig` gate warns
at 700. Reuse the `_PromptSearchApp` harness pattern from that file: copy a small local
harness rather than importing a test-private class.

**Unit tests (pure module):**

- `readout_sigil` for each (direction, whole_word) pair.
- `display_query` sanitizes `\n`/`\t`/control characters and elides by cell width,
  including a CJK case.
- `stack_match_position`: single pane; origin in the middle of three panes; local `None`
  with matches elsewhere; all-zero counts.
- `format_search_readout` plain text is `/alpha  2/3` (segments adjacent);
  `include_query=False` gives `2/3`.
- The count segment's background equals the theme warning color.
- The query segment's background differs from the count segment's in `textual-light`,
  where `accent == warning`.

**Integration tests (Textual `run_test`):**

- **Typing:** after `/alpha`, the panel shows a `2/3`-style count (no brackets) and the
  bar subtitle has no pill. After `Enter`, `bar.border_subtitle.plain` contains `/alpha`
  and `2/3` before `Ln `. Then `n` gives `3/3`, `N` gives `2/3`, and a wrapping `n`
  gives `1/3` plus the existing toast.
- **Clearing:** normal-mode `Esc` removes the pill (and a following `n` restores it);
  `i` removes it; `x` removes it; starting a new `/` hides it while the panel is open.
- **Safety net:** `text_area.load_text("beta only")` removes the pill and the
  highlights. A synthetic `TextArea.Changed` posted while the text is unchanged keeps
  the pill.
- **Word searches:** `*` shows `*alpha`, `#` shows `#alpha`, `g*` shows `/alpha`, `?`
  typing then `Enter` shows `?alpha`.
- **Multi-pane stack** (`initial_panes=["alpha x alpha", "alpha"]`, focus pane 1):
  - `/alpha` gives panel `3/3`; `Enter` gives pill `3/3`.
  - `n` moves focus to pane 0 with pill `1/3`; another `n` gives `2/3`.
  - Typing a query found only in pane 0 while pane 1 is active shows
    `no match in this pane · 2 in stack`.
- **Narrow bar** (for example width 48): the count and `Ln … Col …` remain visible;
  hints are truncated or dropped first.
- **Panel subtitle bug fix:** the search panel subtitle plain text contains
  `[enter] accept`.

**Existing tests:**

- Update the `"[2/2]"` / `"[1/2]"` / `"[1/3]"` panel assertions in
  `tests/ace/tui/widgets/test_prompt_search_interactive.py` to the new bracket-less
  count text. The single-pane numbers are unchanged.
- `tests/ace/tui/widgets/test_vim_search_controller.py` and the pager, zoom, and
  metadata search tests must pass untouched. They prove that
  `render_search_command_line` defaults are unchanged.

**PNG snapshots**
(`tests/ace/tui/visual/test_ace_png_snapshots_prompt_highlighting.py`):

- Regenerate `prompt_search_highlight_120x40`: the typing panel now shows the count pill
  and the fixed `[enter] accept` subtitle.
- Add committed-state goldens after `/alpha`, `Enter`, `n`:
  `prompt_search_count_pill_dark_120x40` (default theme) and
  `prompt_search_count_pill_light_120x40` (`textual-light`).
- If the file would pass 700 lines, put the new test in a new sibling module.
- Inspect the regenerated PNGs yourself before accepting them. The pill must be legible
  and its two segments visibly distinct in both themes, and the count background must
  read as the same gold/amber as the current-match highlight.

## Verification

1. `just install` if the workspace venv is stale.
2. Run the focused suites:
   `pytest tests/ace/tui/widgets/test_prompt_search_readout.py tests/ace/tui/widgets/test_prompt_search_interactive.py tests/ace/tui/widgets/test_prompt_star_search.py tests/ace/tui/widgets/test_prompt_search_highlight.py tests/ace/tui/widgets/test_vim_search_controller.py tests/ace/tui/test_agents_zoom_panel_search.py tests/ace/tui/test_agent_metadata_search.py`
3. `just test-visual` for the prompt highlighting module. Accept the intended changes
   with `--sase-update-visual-snapshots` only after viewing the actual PNGs.
4. `just fix`, then `just check`. If `just check` runs long, hand it to a verify
   monitor.

## Out of Scope

- Letting the typing preview jump across panes.
- Changing `Enter`-on-no-local-match semantics.
- Restyling the pager, zoom, or metadata search command lines.
- Adding a wrap marker.
- Changing keymaps (no `default_config.yml` changes are needed).
