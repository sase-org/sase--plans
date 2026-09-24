---
tier: tale
title: Deck panel core behind the agent_decks beta flag
goal:
  With the new agent_decks beta flag on, the Agents tab detail area shows a DeckArea of
  pre-composed deck panels fed by a hidden Main source instead of the metadata panel
  plus one Files or LLM Calls panel. Each panel renders its deck paged with tab-strip
  titles, a deck-switcher subtitle and empty-state cards. Files and Tools views load
  only when a panel shows them. With the flag off, the Agents tab behaves exactly as it
  does today.
size: medium
proposed_by: bbugyi200.athena.sase-17d.3
bead: sase-17d.3
status: done
---

- **PARENT:**
  [202609/agents_tab_decks_and_cards.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_decks_and_cards.md)
- **BEAD:**
  [sase-17d.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17d/sase-17d.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-17d.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-17d.3.md)
- **COMMITS:**
  - [00ee519](https://github.com/sase-org/sase/commit/00ee51996d109f2715701b4140f7b528520764de)
    — feat(agents-tui): deck panel core behind the agent_decks beta flag

# Plan: Deck panel core behind the `agent_decks` beta flag (phase `deck-panel-core`)

This is phase `deck-panel-core` of epic sase-17d ("Agents tab agent data decks and
cards", plan `plan:202609/agents_tab_decks_and_cards.md`, §4 and §7). Its phase bead is
**sase-17d.3**. The prior phases `llm-calls-subject-guard` (sase-17d.1) and
`main-card-partition` (sase-17d.2, plan `202609/main_card_partition.md`) have landed.
Where this plan and the epic plan disagree about this phase, **this plan wins**. It
refines §7 of the epic plan using the current code.

- All paths are relative to `src/sase/ace/tui/` unless they start with `src/`, `tests/`,
  `tools/` or `docs/`.
- No `sase-core` change is needed, because decks are presentation-only Textual state.
- Before you start, read these with `/sase_memory_read`: `tui.md`, `tui_perf.md`,
  `tui_screenshot.md`, `sase_flags.md`, `symvision.md`, `lint_and_test.md`.

## 1. Scope

**In scope:**

- the `agent_decks` flag
- the `widgets/decks/` package (model, Main document, shared section-view base,
  `MainDeckView`, `DeckPanel`, `DeckArea`, title/subtitle helpers, empty state,
  availability probes)
- the flag-gated `AgentDetail` compose and delegation
- per-panel Files and Tools views, with their scrolls resolved through the ancestor
- lazy loading
- disabling `p`, `Z`, the `view:` chip and metadata search while decks are on
- tests for both flag states
- the before/after j/k perf bench

**Out of scope** (later phases; do not start them):

- the Ctrl+J/K and Ctrl+N/P keys and any keymap registration (`deck-navigation-keys`)
- splits, focus switching and ratio (`deck-splits-focus`)
- fold, hint, search, `E` and `l/h` retargeting (`deck-action-retarget`)
- node-panel collapse and in-place zoom (`node-panel-collapse-zoom`)
- spread mode (`deck-spread-mode`)
- persistence
- PNG goldens: the first flag-on goldens land in `deck-navigation-keys`

**Invariant:** with the flag off, the widget tree, rendering and behavior are the same
as today. The existing suite is the flag-off regression net, and it must pass unchanged
except for tests touched by the mechanical refactors in D5 and D6.

## 2. Design decisions

### D1. Feature flag `agent_decks` (beta)

1. Run exactly (it creates the flag bead; this is the one bead this phase is authorized
   to create):

   ```
   sase flag new agent_decks \
     --when-enabled "The Agents tab shows one or two deck panels of agent data decks (Main, Files, Tools) and cards instead of the metadata panel plus one Files or LLM Calls panel." \
     --when-disabled "The Agents tab keeps the legacy metadata panel plus one secondary Files or LLM Calls panel, the p view picker, and the Z zoom modal." \
     --remove-when "The agent data decks epic's cut-over phase lands: every Agents detail action targets deck panels and the legacy panels, picker, and zoom modal are deleted."
   ```

2. Paste the printed enum member and registry entry into
   `src/sase/feature_flags/registry.py`.
3. Run `just sync-feature-flags-schema` (`tools/sync_feature_flags_schema --write`) to
   update the generated schema block in `src/sase/config/sase.schema.json`.
4. Record the flag bead id as a note on sase-17d.3:
   `sase bead note sase-17d.3 'agent_decks flag bead: <id> (deck-cutover closes it)'`.

**Reading the flag:**

- Add `widgets/decks/flag.py` with `agent_decks_enabled() -> bool`
  (`current_flags().enabled(FeatureFlag.agent_decks)`). It is the only non-test flag
  reference, which satisfies `tools/check_feature_flags` rule 3. Never call it at import
  time or in a class body (rule 4).
- `AgentDetail.__init__` sets `self._decks_enabled = False`. `AgentDetail.compose` reads
  `agent_decks_enabled()` **once** and caches the result.
- Expose the cached value as the read-only property `AgentDetail.decks_enabled`.
  Everything else asks this property, never the flag, so behavior always matches the
  composed tree even if the flag changes at runtime.
- Add `agent_decks_active(app) -> bool` to `widgets/decks/flag.py`. It returns
  `query_one("#agent-detail-panel", AgentDetail).decks_enabled` and returns `False` on
  `NoMatches` or any other exception. Import `AgentDetail` lazily inside the function.
  App-side gates (D12) use it. `_app_action_availability.py:224/239` already queries
  `AgentDetail` in the same way.

### D2. Package layout (`widgets/decks/`)

Keep `widgets/decks/__init__.py` docstring-only, with no re-exports. `prompt_panel`
imports `decks.card_part`, and the new deck modules import `prompt_panel` internals, so
eager re-exports would create an import cycle.

New modules (each well under the ~500-line `toobig` limit):

| Module             | Contents                                                                |
| ------------------ | ----------------------------------------------------------------------- |
| `card_part.py`     | Existing. Add `split_card_parts(content) -> tuple[CardPart, ...]` (D4). |
| `flag.py`          | D1                                                                      |
| `model.py`         | Pure model (D3)                                                         |
| `main_document.py` | `MainDeckDocument` and `build_main_deck_document` (D4)                  |
| `titles.py`        | Deck accents and glyphs; pure title and subtitle helpers (D9)           |
| `empty_state.py`   | The empty-state renderable and its messages (D10)                       |
| `availability.py`  | No-I/O Files and Tools probes (D11)                                     |
| `main_view.py`     | `MainDeckView` (D6)                                                     |
| `panel.py`         | `DeckPanel` (D7)                                                        |
| `area.py`          | `DeckArea` (D7)                                                         |

The shared section-view base lives next to the prompt panel as
`widgets/prompt_panel/_section_view.py` (D5).

Do not touch `widgets/__init__.py` / `.pyi` unless something outside `widgets/` needs a
lazy export. Tests import the modules directly.

### D3. Pure model (`model.py`)

Only what this phase needs. Later phases add `DeckLayout`, `RenderMode`, focus and ratio
transitions, and zoom snapshots. Do not add those symbols now, because Symvision rejects
unused public symbols.

- `class DeckId(StrEnum)`: `MAIN = "main"`, `FILES = "files"`, `TOOLS = "tools"`.
  `DECK_CYCLE: tuple[DeckId, ...] = (MAIN, FILES, TOOLS)`. The subtitle switcher uses
  this order.
- `@dataclass(frozen=True) class DeckPanelState`: `deck: DeckId`,
  `preferred_card: str | None = None`.
- `@dataclass(frozen=True) class DeckAreaState`:
  - `panels: tuple[DeckPanelState, ...]`, where the default is one Main panel (`SINGLE`)
  - `focused: int = 0`
  - Pure helpers `with_panel_deck(state, index, deck)` and
    `with_preferred_card(state, index, card_id)` return new states. An out-of-range
    index raises `IndexError`.
- `default_card_id(card_ids: Sequence[str]) -> str | None`: returns `"context"` when
  present, otherwise the first id, otherwise `None`. Context is the default card; a
  Summary-only document defaults to `summary`.
- `resolve_active_card(card_ids, preferred, *, partial) -> str | None`:
  - returns `preferred` if it is present
  - else, if `partial` is true and `preferred` is not `None`, returns `None`, meaning
    "keep the tab strip, render an empty body until the full document lands, and do not
    flash Context" (§3.7)
  - else returns `default_card_id(card_ids)`

`DeckAreaState` holds at most the two pre-composed panels. This phase only ever
populates index 0 (SINGLE). Index 1 exists only as a composed, hidden widget (D7).

### D4. Main document and `split_card_parts`

`card_part.split_card_parts(content) -> tuple[CardPart, ...]`:

- A card part at the top level returns `(part,)`.
- A `Group` returns its card-part children in order. Loose (non-card) top-level
  renderables are collected, in order, into the `context` card:
  - If a Context card exists, they are appended after its renderables, as a new
    `CardPart` (never mutate the original).
  - Otherwise a new Context card is created at the front.
- Any other non-empty renderable (`Text`, `str`) returns `(context_card(content),)`.
- `""` or `None` returns `()`.
- Empty card parts (no renderables) are dropped.

`main_document.py`:

- `@dataclass(frozen=True) class MainDeckDocument`:
  - `cards: tuple[CardPart, ...]`
  - `subject: object | None`, which is `AgentDetail.metadata_identity`
  - `partial: bool`
  - `digest: str | None`, which is the source's `renderable_content_digest`
  - Properties `card_ids` and `card(card_id) -> CardPart | None`.
- `EMPTY_MAIN_DOCUMENT` has `cards=()`, `subject=None`, `partial=False` and
  `digest=None`.
- `build_main_deck_document(content, *, subject, partial, digest)`:
  - When `subject is None`, it returns a document with no cards.
    `AgentPromptPanel.show_empty` still emits its cardless "No agent selected" text, and
    the deck shows its empty state instead.
  - Otherwise it returns `split_card_parts(content)`.

### D5. Shared section-view base (`prompt_panel/_section_view.py`)

Extract the **view-side** features from `AgentPromptPanel` (`prompt_panel/__init__.py`)
into `SectionViewMixin`, and make `AgentPromptPanel` inherit it. This is a
behavior-neutral move for the flag-off tree. It also brings `prompt_panel/__init__.py`
(518 lines) under the limit.

**What moves:**

- the class-level section/pin attributes
- `prepare_section_document`, `reset_section_document`,
  `preserve_missing_section_on_next_update`
- the generation/anchor-invalidation half of `update()`. Factor it into
  `_apply_section_content(content, digest, *, layout)`, which does the digest-skip
  bookkeeping, bumps the generation, resets anchors, calls `Static.update`, and
  reschedules the pin. It returns `False` when it skipped.
- the bottom pin: `is_pinned_to_bottom`, `pin_to_bottom`, `release_bottom_pin`,
  `bottom_scroll_target`, `_bottom_pin_container`, `_schedule_bottom_pin_reapply`,
  `_reapply_bottom_pin`, `on_resize`
- `render()` (`SectionTrackingVisual`) and `get_content_height` (layout reserve)
- `enable_section_layout_reserve`, `_publish_section_layout`, `resolve_section_target`,
  `resolve_section_at_row`, `queue_section_retry` / `consume_section_retry`,
  `active_section_identity`, `section_layout_reserve`

**What stays in `AgentPromptPanel`:**

- the identity-header, jump-map and main-document sinks
- `inline_document_renderable`
- `prepare_section_document_for_agent`
- the slow-tool tick
- `update()` itself: sinks → digest →
  `_apply_section_content(flatten_card_document(content), …)` → the main-document sink
  (D8)

**Replace the three `getattr(self, "id", None) == "agent-prompt-panel"` gates** (in
`pin_to_bottom`, `get_content_height` and `enable_section_layout_reserve`) with a
capability method `_section_view_features_enabled() -> bool`:

- The base default is `True`.
- `AgentPromptPanel` overrides it to `self.id == "agent-prompt-panel"`, so the zoom
  modal's `#zoom-metadata-panel` keeps today's behavior.

**Typing:** retype `SectionTrackingVisual`'s `owner` (`_section_navigation.py`) from
`AgentPromptPanel` to `SectionViewMixin`, or to a small `Protocol` exposing
`_publish_section_layout`.

**Checks:** the existing prompt-panel section, fold and pin tests are the regression
net. Run them before and after the move; they must pass unchanged. Update any test that
patches a moved attribute on `AgentPromptPanel` only if it breaks, and keep its
assertions.

### D6. `MainDeckView` (`main_view.py`)

`class MainDeckView(SectionViewMixin, Static)` is a Main card view that lives inside a
`VerticalScroll`.

**API:**

- `show_document(document: MainDeckDocument, *, preferred_card: str | None) -> str | None`
  - Computes
    `active = resolve_active_card(document.card_ids, preferred_card, partial=document.partial)`.
  - Calls `prepare_section_document((document.subject, active))`.
  - Renders the active card's `Group(*card.renderables)`, or `Text("")` when `active` is
    `None` or the document has no cards. The DeckPanel shows the empty state in the
    no-cards case (D10).
  - Returns `active`.
- `active_card_id` property.

**Digest skip:**

- The render key is `(document.digest, active, document.partial)`. When it equals the
  last key, return without touching the widget.
- Otherwise call
  `_apply_section_content(renderable, f"{document.digest}:{active}", layout=True)`. This
  gives the section strip caches a unique per-card digest without recomputing a digest;
  fall back to `None` when `document.digest` is `None`.
- Switching cards recomposes from the stored document and never rebuilds it (§4.2).

**Scroll on subject change:** when `document.subject` differs from the view's previous
subject, scroll the view's parent `VerticalScroll` to `y=0` (no animation). Paged mode
means a new subject starts at the top of its card.

### D7. `DeckPanel` and `DeckArea` widgets

#### `DeckPanel(Vertical)`

It is constructed with `panel_index: int` and gets id `agent-deck-panel-{i}`, classes
`deck-panel`. It composes these children, all pre-composed:

1. `VerticalScroll(id=f"agent-deck-panel-{i}-main-scroll", classes="deck-scroll -main")` >
   `MainDeckView()`
2. `VerticalScroll(id=f"agent-deck-panel-{i}-files-scroll", classes="deck-scroll -files")` >
   `AgentFilePanel()`, with no id
3. `VerticalScroll(id=f"agent-deck-panel-{i}-tools-scroll", classes="deck-scroll -tools")` >
   `AgentLLMCallsPanel()`, with no id
4. `Static(classes="deck-empty-state")`, hidden by default

**Showing a deck:**

- Only the active deck's scroll is displayed, via a `-shown` class and CSS `display`.
  Per-deck hosts keep each deck's scroll position across deck switches.
- When the active deck has no content, the empty-state `Static` is shown and the deck
  scroll is hidden. The frame and its geometry stay: the panel never collapses.
- The panel carries its deck accent class (`-deck-main`, `-deck-files` or `-deck-tools`)
  and a `-focused` class. This phase always focuses panel 0.

**Public methods:**

- `set_deck(deck)`
- `show_main_document(document, preferred_card)`
- `file_view` / `tools_view` / `main_view` accessors
- `active_scroll()`, which returns the displayed `VerticalScroll`
- `set_availability(availability)` (D11)
- `refresh_chrome()`, which recomputes the title and subtitle

The panel owns its chrome state: the Main card ids and active card, the Files
count/index/source label/line counts, and the Tools content flag.

**Message handling:** handle the bubbling `FileListChanged`, `FileLineCountChanged`,
`FileVisibilityChanged` and `LLMCallsVisibilityChanged` (with `@on(...)`, because of the
Textual handler-name quirk; see `_agent_detail_panels.py:356`). Each handler updates
chrome and empty state, then calls `message.stop()` so `AgentDetail`'s legacy handlers
never see deck-mode messages.

On `on_resize`, recompute the title tier.

#### `DeckArea(Vertical)`

It has id `agent-deck-area`, classes `-single`. It composes `DeckPanel(0)` and
`DeckPanel(1)`; panel 1 has a `hidden` class (`display: none`) in this phase. It holds a
`DeckAreaState`.

**API** (tests drive it through `AgentDetail`; see D8):

- `state` property
- `panel(index) -> DeckPanel`
- `visible_panels() -> tuple[DeckPanel, ...]`, which is panel 0 only in SINGLE
- `focused_panel() -> DeckPanel`
- `set_panel_deck(index, deck)`, which updates the state and the panel's deck
- `set_preferred_card(index, card_id)`, which updates the state and re-shows the stored
  Main document on that panel
- `panels_showing(deck) -> tuple[DeckPanel, ...]`, visible panels only

**Rules:**

- Layout never remounts panels or views. This phase mounts both panels at compose time
  and only toggles classes.
- Never call `mount`/`remove` on deck widgets after compose.

### D8. `AgentDetail` compose and deck-mode delegation

Put the deck-mode logic in a new mixin, `widgets/_agent_detail_decks.py`
(`AgentDetailDeckMixin`). Keep `agent_detail.py` and the existing mixins short: each
public entry point gets an early `if self.decks_enabled: return self._deck_…(...)`
branch, or a split at the shared prompt-source step. Leave the flag-off code paths
unchanged.

#### Compose (flag on)

```
Vertical#agent-detail-layout
  AgentHeaderPanel#agent-header-panel.hidden          shared chrome (unchanged)
  Vertical#agent-deck-source-host                     display: none, always
    VerticalScroll#agent-prompt-scroll > AgentPromptPanel#agent-prompt-panel.-deck-source
    VerticalScroll#agent-search-scroll.hidden > Static#agent-search-panel
    Static#agent-search-command.hidden
  DeckArea#agent-deck-area
  AgentJumpPanel#agent-jump-panel.hidden              shared chrome (unchanged)
```

**Deviation from epic §4.4, on purpose:** the hidden Main source stays inside a hidden
`#agent-prompt-scroll`, and the search ids are kept inside the always-hidden host. About
90 legacy consumers (`actions/navigation/_basic.py`, `_fold.py`, `_metadata_search.py`,
`_panel_detail.py`, `_agent_detail_jump.py`, …) resolve these ids with a bare
`query_one`. Keeping them resolvable, but invisible, means nothing raises `NoMatches`
with the flag on before `deck-action-retarget` gives each consumer a deck path.
`deck-cutover` deletes the host. The legacy `#agent-file-scroll` /
`#agent-llm-calls-scroll` / `#agent-file-panel` / `#agent-llm-calls-panel` are **not**
composed in deck mode. Their `AgentDetail` accessors get deck paths instead (see
"Deck-mode accessors" below).

#### Main-document sink

- `AgentPromptPanel.attach_main_document_sink(sink)` stores a
  `Callable[[object, bool, str | None], None]` (content, partial, digest).
- `update()` calls the sink after the identity and jump sinks, and only when
  `_apply_section_content` did **not** skip. On a digest skip the views already hold the
  document.
- **Partial flag:** override `update_header_only` in `AgentPromptPanel`. It sets
  `self._main_document_partial = True`, calls `super().update_header_only(agent)`, and
  resets the flag in `finally`. `update()` passes the flag to the sink.
- `AgentDetail.on_mount` attaches the sink only in deck mode.
- **Handler:** `AgentDetail._on_main_document(content, partial, digest)`
  1. builds
     `build_main_deck_document(content, subject=self.metadata_identity, partial=partial, digest=digest)`
  2. stores it as `self._main_deck_document`
  3. pushes it to every panel in `panels_showing(DeckId.MAIN)`, with that panel's
     `preferred_card`
  4. refreshes each panel's chrome

  `metadata_identity` is already updated at this point, because every entry point sets
  `_current_agent` / `_current_tribe_identity` before it calls the source.

#### Delegation (flag on)

| Entry point                                                                                                                                                      | Deck-mode behavior                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `update_display` / `_update_display_impl`                                                                                                                        | Run the shared prompt-source part unchanged: `_current_*`, render context, workflow async or `prompt_panel.update_display`. Factor it out as `_update_main_source(agent, attempt_number)` and call it from both branches. Then call `_deck_refresh_views(agent, stale_threshold_seconds, attempt_number)` and `_deck_refresh_availability()`. Skip every legacy step: panel modes, scroll classes, `_update_panel_indicators`, and the hidden LLM probe at `_agent_detail_display.py:226-229`. |
| `update_display_immediate`                                                                                                                                       | Unchanged. The source's `update_header_only` → partial document → Main views. Files/Tools views and probes wait for the debounced path (§4.8: a j/k paints only the header and the partial Main document).                                                                                                                                                                                                                                                                                     |
| `update_display_with_hints`, `hint_document_is_current`, `detail_header_summary_complete`                                                                        | Unchanged. The hinted document flows through the sink to Main views.                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `show_empty`                                                                                                                                                     | Bump the generation and clear `_current_*`, then `prompt_panel.show_empty()`. The sink produces an empty document because the subject is `None`. Call `show_empty()` on every panel's file and tools views, set availability to empty, and refresh chrome. Skip the legacy scroll-class code.                                                                                                                                                                                                  |
| `show_tribe_summary`                                                                                                                                             | Source `update_tribe_display(...)` → Summary card. Files/Tools views `show_empty()`; availability is empty for Files/Tools, with tribe wording (D10). Skip the legacy scroll-class code.                                                                                                                                                                                                                                                                                                       |
| `_publish_metadata_identity_change`                                                                                                                              | Unchanged; the header and jump scroll resets still apply. The Main views reset their own scroll on a subject change (D6).                                                                                                                                                                                                                                                                                                                                                                      |
| `on_linked_deltas_refreshed`                                                                                                                                     | Fan out to the file view of every panel in `panels_showing(DeckId.FILES)` through the new public `reconcile_linked_pages(agent)` (D13), then `message.stop()`.                                                                                                                                                                                                                                                                                                                                 |
| `toggle_header_expanded`                                                                                                                                         | Also re-apply the bottom pin on every visible Main view that is pinned (base `_schedule_bottom_pin_reapply`).                                                                                                                                                                                                                                                                                                                                                                                  |
| `_update_panel_indicators`, `_apply_detail_layout_classes`, `_expand_prompt_only`, `set_panel_mode`, `set_detail_layout`, `cycle_detail_layout`, `toggle_layout` | No-op in deck mode, returning `False` where a bool is expected.                                                                                                                                                                                                                                                                                                                                                                                                                                |

#### `_deck_refresh_views(agent, stale, attempt_number)`

For each panel in `visible_panels()`:

- **Files:**
  `load_deck_file_view(panel.file_view, agent, attempt_number=…, stale_threshold_seconds=…)`
  (D13).
- **Tools:**
  - If `attempt_number is None`, the agent is not a clan container or proc shell, and
    `supports_slow_tool_sources(agent)`: call `panel.tools_view.update_display(agent)`.
  - Otherwise: `tools_view.show_empty()` and mark Tools empty.
- **Main:** nothing; the sink handles it.

Panels not showing Files or Tools never touch those views: only shown decks load.

#### Switching decks

`AgentDetail.show_deck(panel_index, deck)` is the public entry point that tests (and
`deck-navigation-keys`) use:

1. `deck_area.set_panel_deck(...)`
2. For Main, re-show `self._main_deck_document`. For Files or Tools, run the same
   per-view load as `_deck_refresh_views` for the current subject right away. The views
   already serve from caches first (`AgentFilePanel` cache paths and
   `AgentLLMCallsPanel._cached_fetch_result`).
3. Refresh chrome.

With no current agent (empty, or a tribe), the view shows its empty state.

`AgentDetail.set_deck_preferred_card(panel_index, card_id)` delegates to
`DeckArea.set_preferred_card`.

#### Deck-mode accessors (so flag-on hot paths never raise)

- `is_file_visible()`: `_current_agent` is set, some visible panel shows Files, and that
  panel is not showing its empty state.
- `is_llm_calls_visible()`: the same for Tools.
- `is_metadata_visible()`: some visible panel shows Main.
- `is_info_mode()`: keeps its definition.
- `effective_detail_scroll_id()`: returns
  `f"#{deck_area.focused_panel().active_scroll().id}"`. The generic Ctrl+D/U path in
  `actions/navigation/_basic.py:244-256` then scrolls the focused panel.
  `deck-navigation-keys` owns the pin and `g`/`G` semantics.
- `get_editor_file_info()` / `get_current_image_path()`: use the focused panel's file or
  tools view when that panel shows Files or Tools. Otherwise return today's empty
  results.
- `refresh_current_file`, `cycle_next_file`, `cycle_prev_file`: use the focused panel's
  file view.
- `_llm_calls_panel_or_none()`: the focused panel's tools view if it shows Tools, else
  the first visible panel showing Tools, else `None`.
- `panel_mode_label`: returns `""` (the chip is hidden; D12).

### D9. Titles, subtitles and visual language (`titles.py`)

**Constants:**

| Deck  | Accent (Rich color) | Glyph | Name    |
| ----- | ------------------- | ----- | ------- |
| Main  | `$secondary`\*      | `◆`   | `MAIN`  |
| Files | `green`             | `▤`   | `FILES` |
| Tools | `#87D7FF`           | `λ`   | `TOOLS` |

\* Rich cannot resolve Textual `$` variables. The `DeckPanel` resolves `$secondary` from
the app theme (`app.theme_variables["secondary"]`, falling back to a fixed hex) and
passes the resolved color into the pure helpers. The helpers take `accent: str` as a
parameter and never read the app.

**Pure helpers** (no widget or app access; exhaustively unit-tested):

- `CardTab(card_id: str, title: str)`
- `deck_title(deck, tabs, active_index: int | None, *, width: int, accent: str, focused: bool) -> Text`

It picks the widest tier whose `cell_len(plain) <= width`, falling back to `micro`. This
reuses `PanelTabStrip._reflow_tier_for_width`'s fit-ladder idea
(`widgets/panel_tab_strip.py:195`). The tiers are:

- **full:** `◆ MAIN ┃ Context │ Reply  1/2`
  - The active tab is a bold pill: `reverse bold <accent>`. Inactive tabs are muted
    `#888888`, and separators are `#444444`.
  - `i/n` appears only when `n > 1`.
- **compact:** `▤ FILES ┃ ‹ 3/27 › diff · src/foo.py`
  - The active title only, with `‹ i/n ›` only when `n > 1`.
- **micro:** `FILES 3/27`

With `active_index=None` (a partial paint whose active card is missing), all tabs are
muted and there is no `i/n`. Unfocused styling (muted title text) is parameterized now
but only exercised by tests; splits arrive later.

Files cards come from the panel's file state: the count, the index and
`AgentFilePanel.current_source_label()`. In compact form, Files titles use only the
active label; the full tier lists all tabs only when there are 4 or fewer, otherwise it
falls back to the compact form. Tools has one tab, `LLM Calls`.

- `deck_subtitle(active: DeckId, availability: DeckAvailabilitySet, *, status: Text | None, width: int, accent_for: Mapping[DeckId, str]) -> Text`

Its parts:

- The deck switcher, `main · files 3 · tools`, in `DECK_CYCLE` order.
  - The active deck is bold in its accent. Decks with content use normal text, and empty
    decks are dim. Unknown availability uses normal text with no count.
  - Counts appear only when known.
- The optional status comes first, separated by two spaces. When the full string does
  not fit `width`, drop the switcher first, then truncate the status.
- Files status helper:
  `file_line_status(visible, total, capped, *, editor_key: str) -> Text | None`
  - capped: `1-120 of 693 lines · E editor`
  - otherwise: `693 lines`
  - `total == 0`: `None`
  - `editor_key` comes from the live keymap if a display helper exists for the editor
    action; otherwise pass `"E"`.

### D10. Empty-state card (`empty_state.py`)

`deck_empty_state(deck, *, subject_kind: Literal["agent", "tribe", "node", "attempt", "none"], hint: str | None) -> RenderableType`
renders two centered lines: a muted message and a dim hint.

| Situation                   | Message                                              |
| --------------------------- | ---------------------------------------------------- |
| Main, no selection          | `No agent selected`                                  |
| Files: agent / node / tribe | `No files for this agent` / `… node` / `… tribe`     |
| Files: attempt-pinned       | `Files are not shown for attempt views`              |
| Tools: agent / node / tribe | `No LLM calls for this agent` / `… node` / `… tribe` |
| Tools: attempt-pinned       | `LLM calls are not shown for attempt views`          |

"node" covers proc shells, monitors, gates, workflow steps and clan containers. The
`deck-navigation-keys` phase fills `hint` with `^N next deck · ^P previous deck` from
the live keymap. This phase always passes `hint=None`, because those actions do not
exist yet. Center it with the `.deck-empty-state` CSS (`content-align: center middle`)
or `Align.center` in Rich. Pick whichever keeps geometry stable.

### D11. No-I/O availability probes (`availability.py`)

- `@dataclass(frozen=True) class DeckAvailability`: `has_content: bool | None` (where
  `None` means unknown) and `count: int | None`.
- `DeckAvailabilitySet`: a mapping from `DeckId` to `DeckAvailability`.
- **Main:** known from the stored document (`bool(cards)`, `count=len(cards)`).
- `probe_files_deck(agent, *, attempt_number) -> DeckAvailability`
  - Empty (`False, 0`) for attempt-pinned, clan containers, proc shells, and bash/python
    workflow steps.
  - `pages = desired_file_pages(agent)[0]` (D13). If `pages` is non-empty, return
    `(True, len(pages))`.
  - Otherwise, for a non-active agent (`_ACTIVE_STATUSES` from
    `_agent_detail_helpers.py`) with `agent.all_files`, return `(True, len(all_files))`.
  - Otherwise, when a fetch could still find content (an active agent, or a local
    `workspace_num` without `fleet_origin_alias`), return unknown `(None, None)`.
  - Otherwise return `(False, 0)`.
- `probe_tools_deck(agent, *, attempt_number) -> DeckAvailability`
  - Empty for attempt-pinned, clan containers, proc shells, or
    `not supports_slow_tool_sources(agent)`.
  - Otherwise `count = cached_tool_call_count(agent)` (D13): `None` is unknown, `0` is
    empty, and `n` is `(True, n)`.
- Neither probe stats, globs, reads or spawns anything. The only side effect allowed is
  the existing LRU touch in `get_cached_linked_delta_groups`.
- `_deck_refresh_availability()` runs on the debounced path, after
  `_deck_refresh_views`, and in `show_deck`. A panel's own `FileVisibilityChanged` /
  `LLMCallsVisibilityChanged` overrides the probe with the observed truth for that
  panel's deck.

### D12. Disabled while decks are on

- **`p` (`choose_agent_view`):**
  - `_agent_view_picker_block_reason` (`actions/agents/_agent_view_picker.py:85`)
    returns `"Agent decks replace the view picker"` when `agent_decks_active(self)`. One
    check covers `check_app_action`, the chip's `(p)` hint and the toast.
  - Also make `commands/_availability_agents.py` report `app.choose_agent_view`
    unavailable.
- **`Z` (`zoom_panel`):**
  - In `check_app_action` (`_app_action_availability.py`, before the shared `zoom_panel`
    block near line 461): return `False` when `agent_decks_active(app)`.
  - Make `action_zoom_panel` (`actions/agents/_panel_detail.py:167`) a no-op when
    active.
  - Make the palette predicate for `app.zoom_panel` in
    `commands/_availability_agents.py` return `False` when active.
- **`view:` chip:** in `_update_agents_info_panel_impl`
  (`actions/agents/_display_detail_info.py:193`), force `view_mode = ""` when decks are
  active (including the tribe case). `view_picker_available` already becomes `False`
  through the block reason.
- **Metadata search (`,` `/`):**
  - `_agent_metadata_search_can_start` (`actions/agents/_metadata_search.py:~55-68`)
    returns `False` when `detail.decks_enabled`. Otherwise search would run invisibly
    inside the hidden source host.
  - `deck-action-retarget` replaces this with the per-panel overlay. Add a comment
    saying so.

Record each of these temporary guards in the handoff note (§5) so
`deck-action-retarget`, `node-panel-collapse-zoom` and `deck-cutover` find them.

### D13. File and LLM Calls panel changes (multi-instance safe; the flag-off tree is equivalent)

**Scroll resolution:**

- `file_panel/_content.py:129` `_get_scroll_container`: return `self.parent` when it is
  a `VerticalScroll`. Otherwise fall back to today's
  `self.app.query_one("#agent-file-scroll", VerticalScroll)`.
- Do the same for `llm_calls_panel.py:237` with `#agent-llm-calls-scroll`.
- The zoom modal's overrides (`modals/zoom_panel_widgets.py:54,134`) stay as they are.
- In the flag-off tree the parent _is_ `#agent-file-scroll`, so behavior is identical.

**Lift pure helpers:**

- Lift `AgentFilePanel._desired_file_list` into a module-level
  `desired_file_pages(agent)` in `file_panel/_file_list.py`, with the body unchanged.
  The method delegates to it, so `ZoomFilePanel`'s override keeps working.
- Lift the no-I/O half of `AgentLLMCallsPanel._cached_fetch_result` into a module-level
  `cached_tool_call_count(agent) -> int | None` in `_llm_calls_panel_fetching.py`. It
  uses `build_cached_slow_tool_sources` / `peek_tool_calls_cache_entry`; `None` means
  cold.

**Public methods replacing private writes:**

- `AgentFilePanel.invalidate_subject()` sets `_current_agent = None` and
  `_file_list = []`. Use it in `_agent_detail_panels.py:318-320` (flag off) and in deck
  mode.
- `AgentFilePanel.reconcile_linked_pages(agent)` wraps
  `_reconcile_file_list(agent, allow_initial_display=True)`. Use it in
  `_agent_detail_state.py:151` (flag off) and in the D8 fan-out.

**Extract the legacy Files dispatch:**

- Move it into `widgets/_agent_detail_files.py`:
  `dispatch_file_view(file_panel, agent, *, stale_threshold_seconds) -> bool`. This is
  the branch from "Bash/python workflow steps" onward in
  `_agent_detail_display.py:237-270`: active → `update_display`; commit diffs →
  `update_display`; `all_files` → `set_file_list(files, start_index=0)`; local workspace
  → `update_display`; else → return `False`.
- The legacy code calls it and runs `_expand_prompt_only()` on `False`. Keep the exact
  statement order.
- The deck path wraps it as
  `load_deck_file_view(file_panel, agent, *, attempt_number, stale_threshold_seconds) -> bool`:
  - For attempt-pinned, clan, proc shell or bash/python steps, call
    `file_panel.show_empty()` and return `False`.
  - Otherwise return `dispatch_file_view(...)`. On `False`, call
    `file_panel.show_empty()`.
  - The deck caller marks Files empty for that panel.

### D14. CSS (`styles.tcss`, next to the `#agent-detail-layout` rules near line 3853)

- `#agent-deck-source-host { display: none; }`
- `#agent-deck-area { height: 1fr; }`
- `#agent-deck-area .deck-panel { height: 1fr; border: solid $secondary; padding: 0; border-title-align: left; border-subtitle-align: right; }`
- `.deck-panel.hidden { display: none; }`
- Accent classes set the border color:
  - `.deck-panel.-deck-main`: `border: solid $secondary`
  - `.deck-panel.-deck-files`: `green`
  - `.deck-panel.-deck-tools`: `#87D7FF`
  - Unfocused panels get the same accent at about 35%, e.g.
    `.deck-panel.-deck-main:not(.-focused) { border: solid $secondary 35%; }`. Check
    that Textual's TCSS accepts the alpha form.
- Focus changes never alter border width or padding, so geometry is stable.
- `.deck-scroll { height: 1fr; padding: 1 2; scrollbar-gutter: stable; display: none; }`
  and `.deck-scroll.-shown { display: block; }`
- `.deck-empty-state { height: 1fr; content-align: center middle; display: none; }` and
  `.deck-empty-state.-shown { display: block; }`

Border titles and subtitles are Rich `Text` set on `DeckPanel.border_title` /
`border_subtitle`, the same approach as today's file title.

## 3. Implementation steps

1. **Baseline perf.** Before any change, run the j/k bench:
   `pytest -s -m slow tests/ace/tui/bench_tui_jk.py -k agents` (use `/sase_monitor` if
   it may outrun the turn). Save the p50/p95 table in a bead note.
2. **Flag (D1).** Create it, paste the registry entry, sync the schema, and note the
   bead id.
3. **D13 refactors, flag off.** Scroll resolution, lifted helpers, public methods and
   the extracted dispatch. Run
   `pytest tests/ace/tui/widgets/file_panel tests/ace/tui -k "file_panel or llm_calls or zoom_panel or agent_detail"`
   and keep it green.
4. **D5 extraction.** Run the prompt-panel, section-navigation, fold and pin tests
   before and after. They must pass unchanged.
5. **Pure modules (D3, D4, D9, D10, D11).** Write each with its unit tests.
6. **Widgets (D6, D7) and CSS (D14).**
7. **`AgentDetail` compose, sink and delegation (D8).**
8. **Gates (D12).**
9. **Tests (§4)**, then `just fix` and `sase tool run check`.
10. **Live screenshots**, flag on. Capture with
    `SASE_FEATURE_FLAGS='{"agent_decks": true}'` in the environment (check how
    `sase screenshot` forwards the env to the TUI; pass it through `-- ` args or an env
    prefix as the command supports):
    - an agent with a reply (Main on Context)
    - no selection (Main empty state)
    - a tribe summary

    Inspect each PNG for the frame accent, the title tab strip, the subtitle switcher,
    empty-state centering, and header and jump chrome. Capture a flag-off screenshot as
    well and confirm it is unchanged. You may take a Files or Tools screenshot only if a
    test hook exists. Otherwise the pilot tests cover those decks, because the keys
    arrive in the next phase.

11. **Perf after.** Rerun the bench with the flag off and with the flag on
    (`SASE_FEATURE_FLAGS` in the env, or an `override_flags` wrapper if the bench
    supports it). Record both tables in a bead note. §4.8 targets j/k p95 < 16 ms for
    SINGLE; report any regression honestly rather than hiding it.

## 4. Tests

New tests go under `tests/ace/tui/widgets/decks/`, with pure tests kept separate from
pilot tests. Pilot tests use the `_DetailApp` harness from
`tests/ace/tui/widgets/test_agent_header_panel.py:41-64` and wrap `run_test` in
`with override_flags(agent_decks=True):`. When a test needs real CSS (display, borders),
use the `CSS_PATH` pattern from `tests/ace/tui/test_agents_zoom_panel_files.py:25-38`.
Use `make_agent` from `tests/ace/tui/widgets/_agent_display_helpers.py`.

**Pure unit tests:**

- `split_card_parts`: card, Group of cards, loose renderables before, between and after
  cards, a missing Context, Text, empty, and dropping empty parts.
- `build_main_deck_document`: `subject=None` gives no cards; the digest and partial
  flags pass through.
- `default_card_id` and `resolve_active_card`: preferred present or missing, partial vs
  full, Summary-only, and empty.
- The model: the `DeckAreaState` helpers and out-of-range index errors.
- `deck_title`: each tier at boundary widths, active pill styling, `active_index=None`,
  single-card decks without `i/n`, and the Files compact form.
- `deck_subtitle`: active, content, empty and unknown styling; counts only when known;
  the switcher dropped before the status is truncated.
- `file_line_status`.
- `deck_empty_state`: messages per deck and subject kind.
- The probes: each empty kind, commit-diff pages, cached pages, `all_files` for
  non-active agents, unknown for active agents, and Tools cold (`None`) / 0 / n. Assert
  no I/O by patching `os.stat` / `open` / `Path.read_text` to raise.

**Pilot tests (flag on):**

- **Compose:** `#agent-deck-area` exists with two panels, and panel 1 is hidden. The
  legacy `#agent-file-scroll` / `#agent-llm-calls-scroll` are absent. The
  `#agent-prompt-panel` source is present but not displayed. The header and jump panels
  are present.
- **j/k:** `update_display_immediate(agent)` puts the partial document in the Main view
  with the Context card active and updates the header. Then `update_display(agent)`
  replaces it with the full document; digest skip holds on repeat. Switching to another
  agent resets the Main scroll to the top.
- **Preferred card:** `set_deck_preferred_card(0, "reply")` shows Reply. A new agent
  with a reply keeps Reply. A partial document without a Reply card renders an empty
  body with the tab strip kept (no Context flash). A node without Reply falls back to
  Context.
- **Lazy loading:** with panel 0 on Main, `update_display` never calls the file view's
  or tools view's `update_display`; spy on the methods. `show_deck(0, DeckId.FILES)`
  loads the file view for the current agent, and `DeckId.TOOLS` does the same for the
  tools view. The hidden panel 1's views are never touched.
- **Empty states:**
  - `show_empty` shows the Main empty state.
  - A tribe summary shows the Summary card on Main, and Files shows
    `No files for this tribe`.
  - Proc shell and attempt-pinned Files/Tools empty states.
- **Messages:** `FileListChanged` / `FileLineCountChanged` / `FileVisibilityChanged` /
  `LLMCallsVisibilityChanged` update the panel title and subtitle. They do not reach
  `AgentDetail`'s legacy handlers; assert that `_file_count` is unchanged.
- **Title and subtitle tiers** at two terminal widths.
- **Accessors** do not raise: `is_file_visible`, `is_llm_calls_visible`,
  `is_metadata_visible`, `effective_detail_scroll_id`, `get_editor_file_info`,
  `panel_mode_label`, `llm_calls_detail_level`.
- **Gates:**
  - `check_app_action` is `False` for `zoom_panel` and `choose_agent_view` with decks
    on, and unchanged with decks off (follow the fakes at
    `test_agent_header_panel.py:280-310`).
  - The palette availability is updated.
  - The `view:` chip is hidden.
  - Metadata search does not start.
- **Linked deltas:** `LinkedDeltasRefreshed` reconciles only the file views of panels
  showing Files.

**Flag off:**

- The existing suite passes; that is the invariant.
- Add one explicit test that the flag-off compose tree has the legacy ids and no
  `#agent-deck-area`.
- Add unit tests for `invalidate_subject`, `reconcile_linked_pages`,
  `dispatch_file_view` (each branch) and the scroll resolution: parent scroll vs.
  fallback.

## 5. Verification and close-out

- `just fix`, then `sase tool run check`, via `/sase_monitor` if it may outrun the turn.
  Do not run `just check-full`.
- **No PNG golden changes are expected:** the flag is off by default. If
  `sase tool run check` reports visual diffs, investigate them as regressions. Do not
  regenerate goldens to hide them.
- Run `sase bead epic-symbols sase-17d.3` and resolve every leftover before closing.
  Re-key to the epic only symbols that a later phase will consume, and record why.
- **Bead notes on sase-17d.3:**
  - the flag bead id
  - the bench tables from before and after
  - a **handoff note** for the later phases, listing:
    - the temporary D12 guards (p, Z, chip, metadata search) and the files they live in
    - the hidden `#agent-deck-source-host` compatibility host (the D8 deviation)
    - the `AgentDetail` deck API (`show_deck`, `set_deck_preferred_card`, `deck_area`)
    - that the empty-state `hint` is still `None`
    - that `deck_title` already supports unfocused styling
- Record discovered out-of-scope work as `PROPOSED FOLLOW-UP:` notes on sase-17d.3. Do
  not create beads other than the flag bead.
- Close with `sase bead close sase-17d.3 --note "<what you verified>"`. Do not close the
  epic or the flag bead.
