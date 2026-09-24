---
tier: tale
size: medium
title: Deck spread versus paged rendering (sase-17d.8)
goal: Agents-tab deck panels render a multi-card Main or Files deck spread (all cards
  on one page with titled separators and card anchors) when it fits ace.agent_decks.spread_max_screens
  viewport heights and paged otherwise, deciding with hysteresis and keeping the reading
  position stable across transitions.
proposed_by: bbugyi200.athena.sase-17d.8
bead: sase-17d.8
status: done
---

- **PARENT:**
  [202609/agents_tab_decks_and_cards.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_decks_and_cards.md)
- **BEAD:**
  [sase-17d.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17d/sase-17d.8.md)

# Plan: Spread versus paged deck rendering (phase `deck-spread-mode`, bead sase-17d.8)

## 1. Context

This is phase `deck-spread-mode` of epic sase-17d ("Agents tab agent data decks and
cards"). The epic plan is `plan:202609/agents_tab_decks_and_cards.md`; §3.7 (algorithm),
§3.8 (visual language) and §12 (this phase) are normative, and this plan refines them
against the code that exists after phases .1–.6 landed. **Where this plan and the epic
disagree on implementation detail, this plan wins; where this plan is silent, the epic
wins.**

Everything is behind the `agent_decks` beta flag (read via `widgets/decks/flag.py` /
`AgentDetail.decks_enabled`). Flag-off behavior must not change. No `sase-core` change
is needed (presentation-only state).

All paths are relative to `src/sase/ace/tui/` unless they start with `src/`, `tests/` or
`docs/`.

Relevant existing code (read before editing):

- `widgets/decks/model.py` (`DeckId`, `DeckAreaState`, `resolve_active_card`,
  `cycle_card_id`), `layout.py` (`choose_new_panel`, `toggle_split`), `main_document.py`
  (`MainDeckDocument` with `cards`, `subject`, `partial`, `digest`), `main_view.py`
  (`MainDeckView`: paged-only `show_document` / `show_card`), `panel.py` (`DeckPanel`,
  **485 lines — must be split, see §6**), `area.py`, `titles.py` (`deck_title`,
  `deck_subtitle`, `file_line_status`), `card_part.py`.
- `widgets/_agent_detail_decks.py` (`AgentDetailDeckMixin`: `_on_main_document` sink,
  `show_deck`, `cycle_focused_deck_card`, `_deck_refresh_views`).
- `widgets/prompt_panel/_section_view.py` (`SectionViewMixin`: anchors, layout reserve,
  bottom pin) and `widgets/prompt_panel/_section_navigation.py`
  (`SectionTrackingVisual`, `SECTION_MARKER_META_KEY`, `PromptPanelSectionRole`,
  `_segment_section_identity`).
- `widgets/file_panel/` (`_file_list.py` page slots, `desired_file_pages`,
  `source_label_for_slot`, `_display_file_at_current_index`; `_content.py`
  `_render_full_content`, scroll-anchor store; `_display.py` static/linked/image/video
  display; `_messages.py` `file_cache`, slot helpers, `_LIVE_DIFF_SENTINEL`).
- `prompt_submission_settings.py` + `actions/_state_init_late.py` (typed config
  pattern).

## 2. Configuration

- `src/sase/default_config.yml`: under `ace:` next to `tool_calls:` add

  ```yaml
  agent_decks:
    # A multi-card deck renders "spread" (every card on one scrollable page) when
    # its cards fit within this many panel viewport heights ("screens"), and
    # "paged" (one card at a time) otherwise. 0 means always paged.
    spread_max_screens: 1.5
  ```

- `src/sase/config/sase.schema.json`: `ace.properties.agent_decks` object,
  `additionalProperties: false`, property `spread_max_screens`: `type: number`,
  `minimum: 0`, `default: 1.5`, with a description.
- New `agent_decks_settings.py` (sibling of `prompt_submission_settings.py`): frozen
  slotted dataclass `AgentDecksSettings(spread_max_screens: float = 1.5)`,
  `DEFAULT_AGENT_DECKS_SETTINGS`, `parse_agent_decks_settings(ace_cfg)`: non-mapping
  blocks, bools, non-numbers and negatives fall back to 1.5; ints coerce to float.
- `actions/_state_init_late.py`: set `self._agent_decks_settings` next to
  `_prompt_submission_settings`. Declare the attribute wherever the app's other typed
  settings attributes are declared (grep `_prompt_submission_settings`).
- Panels read it through one helper (`agent_decks_settings_for(widget)` in the new
  settings module or in the decks package) that returns
  `getattr(widget.app, "_agent_decks_settings", DEFAULT_AGENT_DECKS_SETTINGS)` and fails
  open to the default.
- `docs/configuration.md`: document `ace.agent_decks.spread_max_screens` (screens, not
  lines; `0` = always paged; only affects the Agents tab when agent decks are enabled).
- Tests: parser unit tests; update the schema/default parity tests
  (`tests/test_config_schema_ace.py` and any `tests/test_config_schema*.py` that
  enumerates `ace` keys).

## 3. Pure decision and measurement (`widgets/decks/render_mode.py`, new)

- Add `RenderMode(StrEnum)` with `SPREAD = "spread"`, `PAGED = "paged"` to
  `widgets/decks/model.py` (the epic's §4.6 model home).
- `render_mode.py`:
  - `SPREAD_HYSTERESIS = 0.10` (internal constant).
  - `spread_budget_rows(spread_max_screens, viewport_rows) -> float` =
    `spread_max_screens * max(1, viewport_rows)`.
  - `decide_render_mode(*, card_count, has_solo_card, total_rows, viewport_rows, spread_max_screens, previous, same_subject) -> RenderMode`
    — **exactly** the epic §3.7 function (card_count ≤ 1 → SPREAD; solo or `<= 0` →
    PAGED; `total_rows is None` → previous if same subject and previous else PAGED;
    hysteresis edges `> budget
    - 1.10`and`<= budget \* 0.90`; otherwise `<= budget`).
  - `lower_bound_rows(renderables, *, stop_after: float) -> int`: a cheap walker that
    never renders. `Text` → `plain.count("\n") + 1` (0 for empty); `Syntax` → code line
    count; `CachedRenderable`/lazy syntax renderables (`util/lazy_syntax.py`) → their
    source line count if exposed, else 1; `Group` and `CardPart` → sum of children;
    anything else → 1. Stops summing as soon as the running total exceeds `stop_after`.
    It must be a true lower bound of rendered rows (wrapping and padding only add rows).
  - `measure_main_rows(cards, *, width, console, options, budget, cache_key_prefix) -> int`:
    separator overhead is `2 * (len(cards) - 1)` rows (blank line + rule). First sum the
    lower bounds (with `stop_after = budget * (1 + SPREAD_HYSTERESIS)`); if it passes
    that bound return the lower bound (settles PAGED with no render). Otherwise render
    each card exactly with
    `console.render_lines(Group(*card.renderables), options.update_width(width), pad=False)`
    and sum the line counts plus separators. Cache exact per-card heights in a small
    module LRU keyed by `(card digest, width)` where card digest is
    `f"{document.digest}:{card_id}"` when the document digest exists, else
    `renderable_content_digest(card)`.
- Tests (`tests/ace/tui/widgets/decks/test_deck_render_mode.py`): exhaustive
  `decide_render_mode` table — exact boundary `total == budget`, both hysteresis edges
  (just inside/outside 1.10 and 0.90 for previous SPREAD/PAGED), unknown totals with and
  without a same-subject previous, solo cards, `spread_max_screens == 0`, single card,
  zero viewport, and same versus new subject. Lower-bound walker early exit and
  lower-bound ≤ exact on wrapped text. Measurement cache hit (render not called twice).

## 4. Card anchors in the section tracker

- `_section_navigation.py`: add `DECK_CARD_META_KEY = "sase_deck_card"` and
  `PromptPanelSectionRole.CARD`. `_segment_section_identity` checks the card key first
  and returns `(f"card:{card_id}", CARD)` so card identities can never collide with
  section identities. Export the new key.
- `_section_view.py`:
  - `resolve_section_at_row` and `resolve_section_target` ignore `CARD` anchors (fold
    and section behavior of `AgentPromptPanel` must be byte-for-byte unchanged; it never
    emits card meta).
  - `get_content_height`'s layout reserve uses the max row over `TITLE` **and** `CARD`
    anchors, so the last card header can be scrolled to the top.
  - New `card_anchor_rows(*, width) -> tuple[tuple[str, int], ...] | None`: `None` when
    anchors are not ready for the current generation/width; otherwise the ordered
    `(card_id, row)` pairs from `CARD` anchors (strip the `card:` prefix).
- Tests: extend the section-navigation tests: card meta yields CARD anchors; section
  target/at-row ignore them; reserve covers the last card anchor.

## 5. Spread separators (`widgets/decks/separators.py`, new)

- `CardSeparator(card_id, title, *, glyph, accent)`: a Rich renderable whose
  `__rich_console__` yields a blank line, then one full-width line (use
  `options.max_width`): `━━ {glyph} {title} ` followed by `━` to the width, styled in
  the deck accent, with `Style(meta={DECK_CARD_META_KEY: card_id})` on the whole rule
  line so `SectionTrackingVisual` publishes a CARD anchor at the rule row.
  `__rich_measure__` returns full width. Degrade gracefully at tiny widths (truncate the
  title, never raise).
- Separators are placed between consecutive cards only (not before the first). The first
  card's anchor is implicitly row 0: views prepend `(first_card_id, 0)` to the published
  anchors.
- Card body start rows: first card → 0; other cards → `anchor_row + 1`.
- Accents: Main uses the resolved `$secondary` value (the same value `DeckPanel`
  computes for its border; Rich cannot resolve `$secondary`), Files `green`. Glyphs come
  from `titles.DECK_GLYPHS`. Files titles use the page label from
  `source_label_for_slot(slot).label`.
- Tests: rendered text at several widths, meta on exactly one row, no separator before
  the first card.

## 6. DeckPanel split and mode state

`panel.py` is at 485 lines and the repo's `toobig` lint caps modules near 500. Before
adding logic, extract cohesive parts into new modules (for example chrome/tab helpers
into `widgets/decks/panel_chrome.py` as a mixin, and the new mode logic into
`widgets/decks/panel_spread.py` as `DeckPanelSpreadMixin`). Keep every new module under
~500 lines, keep `DeckPanel`'s public API unchanged, and update imports/tests.

Per panel, per deck (Main and Files; Tools has one card so it is trivially spread and
needs no change):

- `_render_mode: dict[DeckId, RenderMode]` (default PAGED) and
  `_mode_subject: dict[DeckId, object]`.
- Viewport rows = the active scroll's content-region height; width = the view's content
  width (scroll width minus the vertical scrollbar gutter). If either is 0 (hidden
  panel, not laid out yet) skip the decision and keep the current mode; the next resize
  decides.
- Decision triggers (each calls `decide_render_mode` with this panel's own geometry and
  the configured `spread_max_screens`):
  1. A **full** (non-partial) Main document arrives (`show_main_document`).
     `same_subject` = document subject equals the subject the mode was last decided for.
  2. A Files spread probe result lands (§8).
  3. `on_resize` (resize, split, ratio, `d`/`.` toggles, node-panel collapse all arrive
     as resizes): re-decide with `same_subject=True`, debounced to one decision per
     refresh via `call_after_refresh`.
  4. `show_deck` switching this panel to Main or Files.
- **Partial documents never decide the mode.** A partial paint renders with the panel's
  current mode (paged: existing preferred-card / empty-body behavior, no Context flash;
  spread: the available cards).
- **Subject** is the node identity plus the attempt pin. Verify that
  `AgentDetail.metadata_identity` already includes the attempt pin; if it does not, make
  `_on_main_document` build the subject as `(metadata_identity, attempt pin)`. Hint-mode
  toggles and content refreshes (streaming reply, SASE CONTEXT lanes) keep the same
  subject.
- Chrome: `deck_subtitle` gains `spread: bool = False`, rendering a subtle dim `spread`
  tag between the status and the switcher (dropped first when width is tight). Update
  `tests/ace/tui/widgets/decks/test_deck_titles.py`.
- Spread active card drives the title pill: in spread the pill shows the active card,
  derived from scroll (§7). The tab strip itself is unchanged.

## 7. Main spread (`MainDeckView`)

- `MainDeckView.show_document(document, *, preferred_card, mode)`:
  - PAGED: today's behavior.
  - SPREAD: compose
    `Group(*card0.renderables, CardSeparator(card1), *card1.renderables, …)` and apply
    with digest `f"{document.digest}:spread"` (digest skip keeps j/k and idle refreshes
    cheap). Render key includes the mode. The active card in spread is **not** taken
    from `preferred_card`; it is derived from scroll.
  - `prepare_section_document` identity includes the mode so section state resets across
    mode transitions.
- **Active card in spread** = last card anchor at or above `scroll_y`
  (`spread_active_card()`), falling back to the first card while anchors are not ready.
  `DeckPanel` watches the Main scroll's `scroll_y`
  (`self.watch(scroll, "scroll_y", …, init=False)`) and, when the derived card changes,
  updates `_main_active_card` and refreshes chrome. Scrolling never sets the preferred
  card.
- **Ctrl+J/K in spread** (`DeckPanel.cycle_card` for Main): compute the next/previous
  card from the spread active card (wraps), call `enable_section_layout_reserve()`, then
  scroll that card's anchor row to the top after refresh. If anchors are not ready,
  retry after the next refresh (bounded, e.g. 3 attempts; never loop). Return the card
  id so `AgentDetail.cycle_focused_deck_card` still sets the preferred card.
  `DeckArea.set_preferred_card` → `show_main_document` must be a no-op re-render in
  spread (same digest/mode) and must not reset scroll.
- **Reading position is anchored across transitions:**
  - Spread → paged: active card A = spread active card;
    `offset = max(0, scroll_y - body_start(A))`; render paged with A as the shown card
    (A becomes the panel's displayed card but the preferred card is unchanged); after
    refresh scroll to `offset` (clamped).
  - Paged → spread: `offset = scroll_y`; render spread; after anchors are ready scroll
    to `body_start(A) + offset` (bounded retries as above).
  - Keep the bottom pin working: if the view was pinned to the bottom before the
    transition, re-pin instead of anchoring.
- **New subjects:** spread starts at the top; paged shows the preferred card if present,
  else the default (existing `resolve_active_card`). Exception: a duplicate Main panel
  opened by the split rule (`choose_new_panel` returns a preferred card) records a
  one-shot "scroll to card" that the panel applies on its first spread render.
- `show_card` in spread delegates to the scroll-to-card path instead of recomposing.
- Keep the existing consumers that ask a Main view for its visible text/active card
  (search, `E`, folds, hints from phase .6) correct in spread: grep for
  `active_card_id`, `main_view`, `_main_document` under `widgets/`, `actions/agents/`
  and `modals/`, and make spread return the scroll-derived card where a single card is
  expected and all cards where the phase-.6 contract says "all cards".

## 8. Files spread

- **Probe (`widgets/file_panel/_spread_probe.py`, new, pure + I/O, off-thread):**
  `probe_files_spread(agent, slots, *, width, stop_after_rows, text_cache) -> FilesSpreadProbe(pages, total_rows, has_solo, exceeded, bound)`.
  - Walk slots in order. Image/video paths (`is_supported_image_path`,
    `is_supported_video_path`) → `has_solo=True`, stop.
  - Live diff sentinel → `file_cache` entry's `diff_output`; linked slots → cached
    linked group `diff_text`; commit slots → read the commit diff file; plain paths →
    read the file. File reads are cached by `(path, mtime, size)` in a bounded module
    LRU and read line-by-line only until the running total passes the bound (never read
    a huge file whole).
  - Rows per page = page header rows (match what `_render_full_content` /
    `_build_linked_banner` emit: header + blank line; linked banner may be two lines) +
    body rows + 2 separator rows for pages after the first. Body rows: check whether
    `lazy_renderable(..., line_numbers=True)` wraps or crops long lines; if it crops,
    logical lines are exact, otherwise estimate wrapped rows from cell widths against
    `width - gutter`.
  - Stop once the total passes `stop_after_rows` (`budget * 1.10`) → `exceeded=True`
    (PAGED). Otherwise return the page texts (bounded by the band, so small) so later
    resizes can recompute rows on the UI thread without I/O.
- **Worker:** `DeckPanel` runs the probe with
  `run_worker(thread=True, exclusive=True, group=f"deck-files-spread-{panel_index}")`
  when its Files deck is shown and the file list or subject changes (`FileListChanged` /
  `FileVisibilityChanged`). The result carries `(agent identity, slots)` and is dropped
  if it no longer matches. On resize, recompute rows from stored page texts; if the last
  probe was `exceeded` and the new band exceeds `bound`, re-probe. Duplicate Files
  panels share the text cache.
- **Rendering:** add `FilesSpreadView(SectionViewMixin, Static)` inside each panel's
  Files scroll, next to `AgentFilePanel` (compose it in `DeckPanel.compose`; hidden
  unless spread). In spread, `AgentFilePanel` is hidden (it keeps loading and posting
  messages, since it owns the page list) and `FilesSpreadView` shows every page in
  order: `CardSeparator(f"file-{i}", label, glyph=▤, accent="green")` between pages,
  each page's header and
  `lazy_renderable(text, lexer, line_numbers=True, max_render_lines=FILE_PANEL_MAX_RENDER_LINES, …)`,
  reusing the file panel's header builders (extract small pure helpers from
  `_content.py` / `_display.py` if needed so both views produce identical page headers).
  Card anchors come from the separators as in §4.
- **Solo cards force paged** (image/video present anywhere in the deck).
- **New subject:** start PAGED on the default page (existing behavior) until the probe
  lands. When it decides SPREAD, anchor the current page: scroll to
  `body_start(current page) + paged scroll_y`, so the switch is viewport-stable.
- **Spread → paged** (resize/growth): select the spread active page in `AgentFilePanel`
  without an extra render hop (add a public `select_file_index(index)` that updates the
  index, notes the slot change and displays it), and seed that slot's scroll-anchor
  store with the in-card offset before display so the async static read restores to it
  (add a small public seeding method rather than writing private fields from the deck).
- **Ctrl+J/K in Files spread:** scroll the next/previous page anchor to the top (with
  the layout reserve) and update the file panel's current index silently (no re-render
  of the hidden paged view) so `E`, clipboard targets and chrome follow the active page.
  Scroll-derived active page updates the same way (via the `scroll_y` watch).
- Make sure the Files search corpus from phase .6 covers all pages when spread (grep the
  deck search corpus builder for Files; extend it if it only reads the current page).

## 9. Tests

Existing flag-on tests and PNG goldens (`tests/ace/tui/widgets/decks/*`,
`tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py`) were written for
paged-only rendering, and their tiny fixture documents will now spread by default. For
tests whose point is paged behavior, pin `spread_max_screens=0` (set
`app._agent_decks_settings = AgentDecksSettings(spread_max_screens=0)` via a small
shared helper/fixture) so they keep testing paged mode and their goldens do not churn.
Only regenerate a golden when the change is intended, and inspect it.

New tests (flag on via `override_flags(agent_decks=True)`; Pilot where noted):

- Pure: §3 table, §4 anchors, §5 separators, settings parser, subtitle `spread` tag.
- Main (Pilot): a small Context/Reply agent renders spread with a titled separator;
  Ctrl+J in spread scrolls the Reply header to the viewport top and sets the preferred
  card; scrolling updates the title pill but not the preferred card; a streaming reply
  that grows past `budget * 1.10` flips to paged showing the card that was at the top
  with the same in-card offset (reading position kept); shrinking below `budget * 0.90`
  flips back; values inside the band do not flip (hysteresis); a new subject in spread
  starts at the top; partial documents never change the mode; `spread_max_screens=0` is
  always paged.
- Files (Pilot, with patched/`tmp_path` files and a populated `file_cache`): a small
  multi-diff agent spreads with page separators; an image page forces paged; a new
  subject starts paged and switches to spread after the probe without moving the default
  page's viewport row; Ctrl+J in Files spread scrolls to the next page and updates the
  current index; oversized content stops the probe early (assert bounded reads).
- Resize and split re-decide with hysteresis (Pilot: shrink the terminal / split a panel
  so a spread deck crosses the upper edge → paged; unsplit → back to spread only below
  the lower edge).
- Each panel decides independently (split with two Main panels of different heights).
- Flag-off: the existing suite passes unchanged (no config read, no card anchors).

PNG goldens (add to the decks visual module; generate with a targeted
`just fix-tui-screenshots -- <selectors>` and inspect every image): spread Main, spread
Files, and a paged-after-threshold Main deck.

## 10. Performance

- j/k must stay cheap: partial paints never measure; full-document decisions use the
  lower bound first and exact rendering only inside the band, cached by
  `(card digest, width)`; Files probes are off-thread and bounded; the `scroll_y` watch
  does O(cards) work.
- Run `SASE_TUI_PERF=1 pytest -s -m slow tests/ace/tui/bench_tui_jk.py` (through the
  guarded runner as the lint/test memory instructs) before and after, with the flag on
  in SINGLE and LEFT_RIGHT layouts; p95 must stay under 16 ms. Record both numbers in
  the bead close note.

## 11. Verification and close-out

- Read the `lint_and_test` reference memory (via `/sase_memory_read`) and run the
  required checks (`just check` through `sase tool run`); fix every failure, including
  `toobig` and Symvision (no unused public symbols — every new public helper needs a
  non-test consumer).
- Screenshot a live TUI with `SASE_FEATURE_FLAGS='{"agent_decks": true}'` via
  `sase screenshot`: single spread Main, a split with one spread and one paged panel,
  spread Files; inspect the PNGs for separator styling and title-pill tracking.
- Run `sase bead epic-symbols sase-17d.8`; resolve or re-key any leftover
  `--epic-symbol` entries to a still-open bead.
- Record discovered follow-ups with `sase bead note sase-17d.8 'PROPOSED FOLLOW-UP: …'`
  (do not create beads).
- Close only this phase:
  `sase bead close sase-17d.8 --note "<what was verified, including bench numbers>"`. Do
  not close the epic or any other bead.
