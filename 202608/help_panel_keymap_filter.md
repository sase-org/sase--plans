---
tier: tale
title: Live keymap filter bar for the ACE Help panel
goal:
  Pressing `/` inside the `?` Help panel opens a filter bar that live-filters the Keymaps view as the user types, with
  matched text highlighted, sections preserved for context, and a match counter — making it easy to find the keymaps for
  one pane, sub-tab, or action without scrolling the full reference.
proposed_by: bbugyi200.athena.uk
create_time: 2026-08-07 09:02:57
status: wip
---

- **PROMPT:**
  [prompts/202608/help_panel_keymap_filter.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/help_panel_keymap_filter.md)

# Plan: Live keymap filter bar for the ACE Help panel

## Goal

Add a `/` keymap to the ACE Help panel (`?`) that opens a filter bar. Typing into it live-filters the **Keymaps** view
so only matching sections and keybinding rows render, with the matched characters highlighted. The filter makes it easy
to answer "what are the keymaps for the Beads pane / Copy Mode / the Commits sub-tab?" without scrolling a reference
that is currently ~40 sections and several hundred rows long.

## Why this design

The Help panel's Keymaps view is already tab-scoped, but within a tab it is enormous: `cls_bindings()` alone composes
`artifact_sections()` (Artifact Views, Commits Pane, Beads Pane, Bugs Pane, Plans Pane, Chats Pane, Other Pane, Preview
Reader), PR Navigation, PR Actions, Fold/Bang/Leader mode sections, seven `Copy Mode · <pane>` sections, Queries,
Grouping, Prompt Input, Admin Center Tasks, Admin Center Updates, and General. Section names are already named after the
panes and sub-tabs they document, so **matching the section name and showing that whole section** is exactly the "filter
to one tab" behavior the request asks for — it falls out of a single matching rule rather than needing a separate
tab-picker UI.

## Current architecture (verified)

- `src/sase/ace/tui/modals/help_modal/modal.py` — `HelpModal(CopyModeForwardingMixin, ModalScreen[None])`. Composes
  `#help-title`, `PanelTabStrip#help-tabs`, `ContentSwitcher#help-content-switcher` with two children
  (`VerticalScroll#help-keymaps-view` containing `Horizontal#help-columns` → `Static#help-left-column` +
  `Static#help-right-column`; and `Container#help-guide-view`), then `Static#help-footer`.
- Section data is plain Python: `Sections = list[tuple[str, list[tuple[str, str]]]]` (section name → list of
  `(key_display, description)`), built by `cls_bindings()` / `agents_bindings()` / `axe_bindings()` in
  `help_modal/*_bindings.py` from the live `KeymapRegistry`.
- `HelpModal._add_section()` renders one boxed section into a `rich.text.Text`. Box geometry is fixed by
  `BOX_WIDTH = 57` / `CONTENT_WIDTH = 50` in `help_modal/binding_common.py`; keys are padded to 16 columns and
  descriptions truncated to 32 with `...`. `src/sase/ace/CLAUDE.md` mandates these widths stay consistent.
- `COLUMN_SPLITS` (in `binding_common.py`) is a static per-tab index that splits sections between the two columns.
- The left column additionally prepends `add_saved_queries_section()` and (ChangeSpecs only)
  `add_query_history_section()` from `help_modal/query_sections.py`. These write directly into the `Text`; they are not
  `Sections` data.
- `refresh_for_tab()` rebuilds the mounted content in place; `_app_watchers.watch_current_tab` calls it when the user
  switches ACE tabs while Help is open.
- Modal-local keys (`escape`, `q`, `?`, `[`, `]`, `ctrl+d`, `ctrl+u`) are hardcoded in `BINDINGS` and rebuilt in
  `on_mount()`; only the app-tab and query-history keys come from the registry. `/` is `search_forward: "slash"` in
  `src/sase/default_config.yml`, so `/` already means "search" app-wide.

### Textual behaviors this design depends on (verified against the pinned `textual==8.0.1` in `.venv`)

1. `Input._on_key` calls `event.stop()` + `event.prevent_default()` for every printable key. So while the filter input
   has focus, `q`, `?`, `[`, `]`, `%` are typed as text and never reach `HelpModal.BINDINGS` or
   `CopyModeForwardingMixin.on_key`. No defensive work is needed for those.
2. Non-printable keys still reach the binding chain, and the **focused widget's bindings win first**. `Input` binds
   `ctrl+u` → `delete_left_all` and `ctrl+d` → `delete_right`, so the modal's `ctrl+d`/`ctrl+u` scroll bindings are
   shadowed while the input is focused. `escape` has no `Input` binding, so it would otherwise reach the modal's
   `action_close` and close the whole panel mid-filter.
3. Priority bindings are checked before the focused widget. `on_mount()` already registers `next_tab` / `prev_tab` with
   `priority=True`, so app-tab switching keeps working while typing in the filter.
4. `App.AUTO_FOCUS = "*"` and `Screen._update_auto_focus()` focuses the first `focusable` widget after compose.
   `Widget.focusable` checks `self.visible` (the `visibility` rule), **not** `display`. A `display: none` filter input
   can therefore still be auto-focused. `HelpModal` must set `AUTO_FOCUS` explicitly.
5. `sase.core.fuzzy_facade.fuzzy_match(query, text)` (the shared Rust matcher already used by the recursive file finder
   and completion surfaces) is case-insensitive and returns `tier`, `score`, and character-indexed `runs`. Probed
   behavior: `tier=0` prefix, `tier=2` contiguous substring, `tier=3` scattered subsequence, `None` no match. A query
   containing a space is matched literally, so tokens must be split before matching.
6. `sase.ace.tui.widgets._completion_match_highlight.append_highlighted()` renders a string with highlighted runs,
   normalizing/merging/clamping runs to the string length. It changes styles only, never characters — so box alignment
   is preserved for free.

## Design

### Interaction model

| State                           | Keys                                                                                                                                                                                                                                                                                                                                           |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Help open, results focused      | `/` opens + focuses the filter bar. All existing keys unchanged.                                                                                                                                                                                                                                                                               |
| Filter bar focused              | Type → live re-render. `Enter` applies + returns focus to results. `Esc` cancels the filter entirely and returns focus to results. `↑`/`↓`/`PageUp`/`PageDown` scroll the results without leaving the bar. `Ctrl+U` / `Ctrl+W` / `Ctrl+A` / `Ctrl+E` keep native `Input` editing. `Tab`/`Shift+Tab` still switch ACE tabs (priority bindings). |
| Filter applied, results focused | The bar stays visible showing the active query + counter. `/` re-focuses it with the cursor at the end. `Esc` clears the filter and hides the bar (does **not** close the modal). `q` / `?` still close the modal. `Ctrl+D`/`Ctrl+U` scroll normally.                                                                                          |
| Filter applied, no filter text  | Bar hidden; `Esc` closes the modal, exactly as today.                                                                                                                                                                                                                                                                                          |

Rationale for the two Escape meanings: Escape must never destroy more than one layer of state at a time. With a filter
active it removes the filter; only with no filter does it close the panel. This mirrors `ConfigPane.cancel_input()`, the
existing house pattern for `/`-filtered panes.

`↑`/`↓` scroll rather than `Ctrl+D`/`Ctrl+U` because `Input` already owns `Ctrl+D`/`Ctrl+U` for text editing (see
Textual behavior 2), and a single-line `Input` has no use for the vertical arrows. Nothing is stolen from the user.

Pressing `/` while the **Guide** tab is active switches to the Keymaps tab first, then opens the bar — `/` always means
"find a keymap".

### Matching rule (one rule, no special cases)

The query is split on whitespace into tokens. For a keybinding row inside a section, the haystacks are the **section
name**, the **key display**, and the **description**.

> A row matches when **every** token matches at least one of its three haystacks.

Consequences that make this feel right without extra machinery:

- Typing `beads` matches the "Beads Pane" section name for every row in it → the whole section renders. That is the
  "filter to one pane / sub-tab" behavior.
- Typing `copy beads` matches rows in `Copy Mode · Beads` (first token hits the section name, second token also hits the
  section name) and nothing else.
- Typing `kill agent` matches rows where `kill` and `agent` appear across the key/description/section, in any order.
- A section renders only when at least one of its rows matched; its header renders with matched runs highlighted.

**Two-pass relaxation.** Pass 1 accepts only contiguous matches (`fuzzy_match` tier ≤ 2). If pass 1 produces at least
one matching row, that is the result. Otherwise pass 2 re-runs accepting scattered subsequence matches (tier 3) and the
status chip appends `~ fuzzy`. This keeps everyday queries precise (`mark` will not match `Metadata Search` through
scattered letters) while still finding something for initialisms like `bp` → `Beads Pane`. The `~ fuzzy` chip reuses the
existing vocabulary from `_prompt_input_bar_completion_panel.py`.

Empty/whitespace-only query ⇒ no filter (identical to today's render, see the invariant below).

### Rendering

- **Highlight**: matched runs render with `MATCH_STYLE` (`bold #FFD700`) from `_completion_match_highlight`, via
  `append_highlighted()`, applied to the _already truncated and padded_ display strings so box geometry is untouched. A
  match that lands entirely inside a truncated description tail simply renders unhighlighted; the row still shows. This
  is an accepted, documented limitation, not a bug to chase.
- **Column balance**: with no filter, keep `COLUMN_SPLITS` exactly as-is. With a filter active, compute the split by
  rendered line count (each section is `len(rows) + 3` lines) choosing the prefix index that minimizes
  `abs(left_lines - right_lines)`, counting the surviving query panels in the left column. A static index would dump
  every result into one column once sections drop out.
- **Query panels**: `Saved Queries` and `Query History` are contextual panels, not keymaps. While a filter is active
  they render only when every token matches their title (`"Saved Queries"` / `"Query History"`), using the same matching
  rule. Otherwise they are omitted.
- **Status chip** (right side of the filter bar): `N keymaps · M sections`, plus ` · ~ fuzzy` when pass 2 was used, or
  `no matches` when nothing matched.
- **Empty state**: when nothing matches, hide `#help-columns` and show a centered dim `Static#help-filter-empty`:
  `No keymaps match "<query>"` over a dim hint line (`Esc clears the filter`).
- **Footer**: two variants. Normal keeps its existing prefix and gains the new key —
  `? / q / esc close  |  / filter  |  [ / ] panel tabs  |  <next> / <prev> app tabs  |  Ctrl+D/U scroll`. While the bar
  is focused it becomes filter-scoped: `type to filter  |  ↑ / ↓ scroll  |  enter apply  |  esc clear filter`. Keeping
  the literal `? / q / esc close` prefix in the normal variant preserves the existing assertion in
  `test_help_guide_agents_png_snapshot`.

### State lifetime

- The filter **persists across ACE tab switches** while Help stays open (`refresh_for_tab` re-applies it). Switching
  from Artifacts to Agents with `bead` typed shows the Agents-tab bead keymaps — this is the feature at its best. If the
  new tab has no matches, the empty state renders and the counter says so.
- The filter **does not persist across close/reopen**. Pressing `?` always shows the complete reference.
- Switching to the Guide tab leaves the filter applied but moves focus off the input; returning to Keymaps restores the
  filtered view.

### Invariant that protects the existing goldens

**With an empty filter, the rendered `Text` of both columns must be byte-identical to today's output.** The filtered
path is a separate branch layered over the same `_add_section` geometry; the unfiltered path keeps calling the existing
code with the existing `COLUMN_SPLITS`. Only the footer string and the (hidden) filter bar are new.

## Implementation

### 1. New: `src/sase/ace/tui/modals/help_modal/filter_model.py` (pure, no Textual)

```python
Token = str

@dataclass(frozen=True, slots=True)
class FilteredRow:
    key: str
    description: str
    key_runs: tuple[tuple[int, int], ...]
    description_runs: tuple[tuple[int, int], ...]

@dataclass(frozen=True, slots=True)
class FilteredSection:
    name: str
    name_runs: tuple[tuple[int, int], ...]
    rows: tuple[FilteredRow, ...]

@dataclass(frozen=True, slots=True)
class FilterResult:
    query: str
    active: bool           # False for an empty/whitespace-only query
    sections: tuple[FilteredSection, ...]
    keymap_count: int
    section_count: int
    relaxed: bool          # True when pass 2 (tier 3) produced the result

def tokenize(query: str) -> tuple[str, ...]: ...
def filter_sections(sections: Sections, query: str) -> FilterResult: ...
def matches_title(title: str, query: str) -> tuple[tuple[int, int], ...] | None: ...
def balance_split(sections: Sequence[FilteredSection], *, lead_lines: int = 0) -> int: ...
```

- `filter_sections` runs `_filter_pass(sections, tokens, max_tier=2)`, falls back to `_filter_pass(..., max_tier=3)`
  when the first pass yields zero rows, and sets `relaxed` accordingly.
- Matching goes through `sase.core.fuzzy_facade.fuzzy_match` — reuse the shared Rust matcher, do not hand-roll one. Tier
  is used only as the precision gate; **no re-ranking**: authored section and row order is preserved so the panel stays
  visually stable as the user types.
- `matches_title` is the same rule applied to a single string, used for the Saved Queries / Query History panels.
- `balance_split` returns the section index where the right column starts.

### 2. New: `src/sase/ace/tui/modals/help_modal/filter_bar.py`

- `HelpFilterInput(Input)` with `BINDINGS` for `up`/`down` (`scroll_results_up` / `scroll_results_down`, half-page like
  the modal's own scroll), `pageup`/`pagedown` (full page), and an `on_key` intercept for `escape` that calls the
  modal's cancel path (mirroring `ConfigFilterInput`). Walk `self.parent` upward to find the owning `HelpModal`,
  matching the existing `_pane()` idiom, or use `self.screen`.
- `build_filter_status(result: FilterResult) -> Text` — the counter / `~ fuzzy` / `no matches` chip.
- `build_filter_empty_state(query: str) -> Text` — the centered empty-state message.

### 3. New: `src/sase/ace/tui/modals/help_modal/sections_render.py`

Move `HelpModal._add_section` here as `add_section(text, name, rows, *, name_runs=(), content_width=CONTENT_WIDTH)` so
both the filtered and unfiltered paths share one renderer and `modal.py` stays under the `toobig` thresholds
(`just _lint-toobig` runs `toobig src 1000 850 700`; `modal.py` is 469 lines today). Highlighting is opt-in via the runs
arguments; with empty runs the output is character- and style-identical to today. Keep the existing
`HelpModal._add_section` as a thin delegating wrapper only if something outside the class calls it — otherwise delete it
and update callers.

### 4. `src/sase/ace/tui/modals/help_modal/modal.py`

- Add `AUTO_FOCUS = "#help-keymaps-scroll"` to `HelpModal` (see Textual behavior 4). Without it the new `Input` can
  steal initial focus and `q` would type a character instead of closing the panel.
- Restructure the Keymaps switcher child: `Vertical(id="help-keymaps-view")` → `Horizontal(id="help-filter-bar")`
  (`Static#help-filter-prompt` with the `/` glyph, `HelpFilterInput#help-filter-input`, `Static#help-filter-status`) +
  `VerticalScroll(id="help-keymaps-scroll")` → `Horizontal#help-columns` (unchanged) + `Static#help-filter-empty`. The
  `ContentSwitcher` child id stays `help-keymaps-view` so `HelpPanelTab` / `_switch_help_tab` are untouched.
- Update `_active_scroll()` to return `#help-keymaps-scroll` for the Keymaps tab.
- New state: `self._filter_query: str = ""`, `self._filter_result: FilterResult | None`.
- New binding `("slash", "focus_filter", "Filter")` in both `BINDINGS` and the `on_mount()` rebuild list (the `on_mount`
  list currently re-declares the static bindings — keep the two lists in sync).
- Actions:
  - `action_focus_filter()` — switch to the Keymaps tab if needed, show the bar, focus the input, put the cursor at the
    end of any existing text (set `select_on_focus=False` on the input so re-entering `/` continues editing rather than
    silently replacing).
  - `action_clear_filter()` — clear text, hide the bar, re-render unfiltered, focus `#help-keymaps-scroll`.
  - `action_apply_filter()` (Enter / `Input.Submitted`) — keep the filter, focus `#help-keymaps-scroll`.
  - Override `action_close()`: if a filter is active **and** the input is focused, clear the filter instead of
    dismissing. (When the input is not focused, `escape` with an active filter also clears — handle that in
    `action_close` by checking `self._filter_query`.) With no filter, dismiss as today.
- `on_input_changed` — guard on `event.input.id == "help-filter-input"`, store the query, re-render both columns and the
  status chip, toggle `#help-columns` / `#help-filter-empty`, and scroll the results back to the top. The app's
  `_event_keyboard.on_input_changed` still sees the bubbled message and keeps input-quiescence timing accurate for the
  visual suite — no change needed there.
- `_build_left_column` / `_build_right_column` — compute `filter_sections(self._get_bindings_for_tab(), query)` once per
  render (store it, do not compute twice), then render from `FilteredSection`s using `balance_split`; keep the existing
  unfiltered code path untouched when `not result.active`.
- `_build_footer()` — return the filter-scoped variant when the input has focus, otherwise the normal variant with the
  new `/ filter` hint. Call it from the input's focus/blur handlers as well as the existing refresh points.
- `refresh_for_tab()` — re-apply the current filter and refresh the status chip and footer.
- Guard every `query_one` the same way the file already does (`except (NoMatches, LookupError): return`), so a partially
  mounted modal never raises.

### 5. `src/sase/ace/tui/modals/help_modal/query_sections.py`

Give `add_saved_queries_section` / `add_query_history_section` an optional `title_runs` argument so a title-matched
panel can highlight its header consistently with keymap sections. Callers decide whether to call them at all.

### 6. `src/sase/ace/tui/styles.tcss` (Help Modal block, ~line 4287)

- `HelpModal #help-keymaps-view` → `height: 1fr;` only (it is now a plain `Vertical`).
- New `HelpModal #help-keymaps-scroll { height: 1fr; scrollbar-gutter: stable; }` — the `scrollbar-gutter: stable` must
  move here or the columns will shift by one cell when the scrollbar appears, which would break PNG equality.
- New `HelpModal #help-filter-bar` — `height: 1; display: none;`, `layout: horizontal`, `margin-bottom: 1`, and a
  `&.-active { display: block; }` (or a `-visible` class toggled from Python, matching the repo's `hidden`-class idiom).
- `#help-filter-prompt` fixed `width: 2`, accent-colored; `#help-filter-input` `width: 1fr` with `border: none`,
  `background: transparent`, `padding: 0`, `height: 1`; `#help-filter-status` `width: auto`, `color: $text-muted`,
  `text-align: right`.
- `#help-filter-empty` — `display: none` by default, `height: auto`, `content-align: center middle`,
  `color: $text-muted`, `padding: 2 0`.
- Use concrete hex colors for the match/accent styling (the visual fixtures pin colors for deterministic PNGs), and tint
  the bar with the per-tab accent already available from `HelpModal._tab_accent()` so the bar matches the panel's
  Artifacts/Agents/Axe border color.

### 7. `docs/ace.md`

Extend the Help paragraph at ~line 66 with the `/` filter: what it matches (section name, key, description), that tokens
are ANDed, that section-name matches show the whole section, that the filter follows ACE tab switches, and that `Esc`
clears the filter before it closes the panel. Add `/` to the Help-panel key list if one is present nearby.

## Testing

### Unit — `tests/ace/tui/test_help_modal_filter_model.py`

Pure `filter_model` coverage, no Textual:

- Empty and whitespace-only queries return `active=False` and the sections untouched.
- Section-name match (`beads`) returns every row of `Beads Pane` and drops unrelated sections.
- Multi-token AND (`copy beads`, `kill agent`) — order-independent, all tokens required.
- A token matching only the key display (e.g. the configured `ctrl+d`) and only the description.
- Two-pass relaxation: a query with a contiguous match anywhere returns `relaxed=False` and does **not** include
  scattered-subsequence rows; an initialism with no contiguous match returns `relaxed=True` with results.
- Zero matches: `sections == ()`, `keymap_count == 0`, `section_count == 0`.
- Runs are within bounds and non-overlapping after `append_highlighted` normalization; feed real
  `cls_bindings(load_keymap_registry({}))` data so the test exercises production section shapes.
- `balance_split` balances line counts and handles 0/1 section and the `lead_lines` offset.
- `matches_title` for `Saved Queries` / `Query History`.

### Integration — `tests/ace/tui/test_help_modal_filter.py` (`AcePage`, following `test_popup_panel_tab_switch_keymaps.py`)

- `?` then `/` shows the bar and focuses the input; assert `page.app.focused` is `#help-filter-input`.
- Typing `beads` live-updates `#help-left-column` / `#help-right-column` plain text: `Beads Pane` present, an unrelated
  section (e.g. `Prompt Input`) absent, and the status chip shows a non-zero count.
- **Focus-safety regression**: with the input focused, pressing `q` inserts `q` into the filter and the modal stays open
  (`isinstance(page.app.screen, HelpModal)`). This is the highest-value test in the file — it locks in Textual behaviors
  1 and 4.
- `enter` applies the filter and moves focus to `#help-keymaps-scroll`; the filtered content is unchanged.
- `escape` with an active filter clears it and keeps the modal open; a second `escape` closes it.
- `/` on the Guide tab switches to Keymaps and focuses the input.
- A nonsense query renders the empty state and hides `#help-columns`.
- Filter persists across an ACE tab switch: filter on Artifacts, press `tab`, assert the Agents-tab content is still
  filtered by the same query.
- Reopening `?` after closing starts with no filter and the bar hidden.
- **Byte-identity guard**: capture `#help-left-column` / `#help-right-column` plain text before filtering, filter,
  clear, and assert the text is identical to the captured value.

### Visual — `tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py`

- Add `test_help_panel_filter_png_snapshot`: open Help on Artifacts, press `/`, type `beads`, wait for
  `wait_for_visual_idle`, assert the SVG contains `Beads Pane`, and add golden
  `tests/ace/tui/visual/snapshots/png/help_keymaps_filter_120x40.png`.
- The footer text change means the four existing help goldens (`help_keymaps_changespecs_120x40`,
  `help_guide_agents_120x40`, `help_guide_axe_120x40`, `help_guide_changespecs_120x40`) will need regeneration. Run
  `just test-visual`, confirm from `.pytest_cache/sase-visual/` diffs that **only the footer line changed**, then accept
  with `just test-visual -- --sase-update-visual-snapshots`. Do not blanket-accept: a diff anywhere other than the
  footer means the DOM restructure shifted the layout and the CSS move of `scrollbar-gutter: stable` needs revisiting.

### Gates

1. `just install` first — this is an ephemeral numbered workspace and dependencies may be stale.
2. `just check` during development.
3. `just check-full` plus `just test-visual` before landing, because this change touches shared TUI CSS and the
   help-panel goldens.

## Constraints and non-goals

- **Do not edit any memory file** (`sase/memory/*.md`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`,
  or `src/sase/ace/CLAUDE.md`). This plan does not carry user permission for that. If the implementer believes a memory
  update is warranted, file a task bead through `/sase_new_task` instead.
- Box geometry stays at `BOX_WIDTH = 57` / `CONTENT_WIDTH = 50` per `src/sase/ace/CLAUDE.md`. Highlighting changes
  styles only, never the character stream.
- `/` stays a hardcoded modal-local binding, consistent with the panel's other local keys (`q`, `[`, `]`, `ctrl+d`,
  `ctrl+u`) and with `search_forward: "slash"` already meaning "search" app-wide. No new
  `AppKeymaps`/`default_config.yml` field.
- Matching reuses `sase.core.fuzzy_facade` — the shared Rust matcher. Do not add a matcher to this repo, and do not add
  anything to `sase-core`: the section data being filtered is TUI presentation data that no other frontend renders, so
  only the existing matching primitive is shared.
- No re-ranking of results, no filter history, no filter persistence across panel close/reopen, and no filtering of the
  Guide tab. Those are deliberate scope cuts that keep the panel stable and predictable.
