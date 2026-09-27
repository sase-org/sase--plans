---
tier: epic
title: "Deck views: see and choose how a deck panel pages its cards and blocks"
goal: "Every Main and Files deck panel names its effective view (spread, page cards, or
  page blocks) and whether it is automatic or fixed, in a stable, text-first badge in
  the top border. `P` cycles the focused panel through the valid views without losing
  the reader's place. Palette commands pick a view directly or return to automatic.
  Choices persist per panel and per deck across agents, splits, zoom, and restarts.

  "
phases:
  - id: model
    title: Deck view policy model, pure resolution, and persistence
    depends_on: []
    size: medium
    description: "model: add the DeckView policy enum and per-panel policies to the pure
      deck state. Add the pure view_policy resolution/cycle/equivalence module and the
      pure badge-text helper. Persist views as an additive optional field in the v1
      deck-state file. Unit tests only; no visible change.

      "
  - id: main-engine
    title: Main deck honors view policies with anchor-preserving transitions
    depends_on:
      - model
    size: medium
    description: "main-engine: store policies on DeckPanel through a new panel_view
      mixin. Force deck/block modes in the Main deciders, and add one view-change
      transition that keeps card, block, offset, pin, and following in every direction,
      including rapid presses. Add the DeckArea/AgentDetail set/cycle API, the cached
      cycle-availability predicate, and resolved_view(). Pilot tests.

      "
  - id: files-engine
    title: Files deck honors view policies with a complete spread probe
    depends_on:
      - main-engine
    size: medium
    description: "files-engine: make fixed Files views skip or complete the spread
      probe. Add a complete probe with per-page line caps and truncation hints. Track
      pending and media-blocked spread states for resolved_view, honor policy in the
      probe result path, and add a user-initiated media toast. Tests.

      "
  - id: chrome
    title: Top-border view badge, rail cue, and subtitle cleanup
    depends_on:
      - main-engine
    size: medium
    description: "chrome: render the effective view badge after the deck name with a
      width-tier ladder, held across partial paints. Add the page N/M / all N block-rail
      cue, remove the old bottom-border spread tag, and regenerate and inspect the
      affected deck goldens.

      "
  - id: controls
    title: P key, palette view commands, footer, help, and search exits
    depends_on:
      - files-engine
      - chrome
    size: medium
    description: 'controls: add the Agents-only cycle_deck_view action on P with gating,
      footer "P view", help row, search passthrough, and the first-fix toast. Add
      palette commands for the three fixed views and "Deck view: automatic", with
      availability context and tests. Regenerate footer-affected goldens.

      '
  - id: verify
    title: View goldens, live inspection, and forced-spread benchmarks
    depends_on:
      - controls
    size: medium
    description: "verify: add new deck-view PNG scenarios and inspect live screenshots
      at wide and narrow widths. Benchmark P transitions on the 5,000-line Reply, a
      pathological Reply, and a forced Files spread against stated budgets. Mitigate
      only by measured rules, then run the acceptance checklist.

      "
  - id: docs
    title: User docs for deck views
    depends_on:
      - controls
    size: small
    description:
      "docs: document deck views, the badge legend, P, palette reset, persistence, and
      the Auto-only scope of the spread thresholds in docs/ace.md and
      docs/configuration.md. Record a proposed glossary follow-up."
proposed_by: bbugyi200.athena.0sx
create_time: 2026-09-27 05:45:12
status: wip
---

- **PROMPT:**
  [prompts/202609/deck_views.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/deck_views.md)

# Deck views: see and choose how a deck panel pages its cards and blocks

## Background

Card blocks (epic `sase-19x`, closed) gave the Main deck a second paging level: a deck
is _spread_ (every card on one scrollable page) or _paged_ (one card at a time), and a
paged card with two or more blocks is _block-spread_ (all blocks inline) or
_block-paged_ (one block per page). Both decisions are automatic
(`ace.agent_decks.spread_max_screens` and `block_spread_max_screens`, 1.5 viewport
heights, 10% hysteresis). The user can neither see which state they are in nor choose
one:

- `deck_subtitle()` shows a dim `spread` word only for a spread deck. It is the first
  thing dropped when space runs short, and paged is signaled only by its absence.
- Nothing names the block mode. The block rail appears for both block modes, so a long
  newest block looks the same whether older blocks are inline below it or on other
  pages.
- There is no way to override the automatic choice.

The research synthesis `research:202609/deck_card_paging_ux/deck_card_paging_ux.md` (and
its source report `deck_card_paging_ux__cdx.md`) recommends a single ordered view depth,
a persistent text-first title badge, one cycle key, palette direct choices, and
per-panel/per-deck persistence. This plan adopts that direction. Below it settles the
points the research left open or contradictory.

This is presentation-only Textual state, keymaps, and chrome. Nothing here belongs in
`sase_core`: no other frontend needs to match a TUI panel's paging preference.

## Design

### D1. One view depth: three layouts, one policy

A deck panel's **view** is one of three layouts, ordered by paging depth:

| Layout (`DeckView`) | Deck   | Active card's blocks | What one page holds |
| ------------------- | ------ | -------------------- | ------------------- |
| `SPREAD`            | spread | inline               | every card          |
| `PAGE_CARDS`        | paged  | inline               | one whole card      |
| `PAGE_BLOCKS`       | paged  | paged                | one block           |

The **policy** is `AUTO` or one of the three layouts (a _fixed_ view). `AUTO` is today's
behavior, unchanged: it resolves to a layout through the existing measurements,
thresholds, and hysteresis. A fixed view bypasses measurement entirely, so it is also
cheaper. Deck-spread with block-paging stays unrepresentable.

User-facing name: **deck view**. The layout names `spread`, `page cards`, and
`page blocks` are used verbatim in the badge, palette, help, and docs.

Invariants that a fixed policy never overrides:

- A deck with one card or fewer renders spread (today's rule).
- A card with fewer than two blocks has no block mode.
- Files containing an image or video can never spread (hard renderer constraint).
- A partial Main document never decides a mode.

### D2. `P` widens the view, wrapping

The research is internally inconsistent: it lists the cycle as
`spread → page cards → page blocks`, yet says the first press from an automatic
page-blocks Reply should yield page cards. **Decision:** `P` steps _wider_ (less paging)
and wraps:

```text
page blocks → page cards → spread → page blocks
```

Why: Auto already picks the widest layout that fits the budget. So an override almost
always means "show me more together", as in reading a whole long Reply. Widening reaches
that in one press. It also never routes a big Reply through the most expensive layout
(full spread) on the way to page cards.

- **From Auto:** the first `P` fixes the next distinct layout after the _effective_ one.
  It never just pins the current picture.
- **From fixed:** `P` moves to the next distinct layout after the fixed one.
- **Back to Auto:** only the palette command **Deck view: automatic**. The key cycles
  exactly the three layouts. A dedicated reset key is a future option if resets prove
  frequent (see D9 for the one-time teaching toast).

### D3. Equivalence, skipping, and availability

A press must always change what is on screen, so the cycle skips layouts that render
identically for the current content. A pure
`ViewContent(deck, card_count, active_block_count, spread_blocked)` gives each layout a
signature `(deck_mode, block_state)`, where `block_state` is `NONE` when the card has
fewer than two blocks, `INLINE`, or `PAGED`:

- deck mode: `SPREAD` if `card_count <= 1`, or if the layout is `SPREAD` and
  `spread_blocked` is false; otherwise `PAGED`.
- block state: `NONE` if `active_block_count < 2`; `INLINE` if the deck is spread or the
  layout is `PAGE_CARDS`; `PAGED` for `PAGE_BLOCKS`.

**Distinct layouts** are the signature classes, each represented by its **shallowest**
member (spread < page cards < page blocks). For example, on a blockless Context card,
page blocks collapses into page cards. Files never offers `PAGE_BLOCKS`; Tools offers
nothing. The cycle walks the distinct layouts in widening order, starting from the
distinct class that contains the current layout. The cycle is **available** only when
there are at least two distinct layouts, the deck is Main or Files and not empty, and a
Main document is not partial. The active card for `active_block_count` is the card the
next paged render would show: the active card when paged; in deck-spread, the reading
anchor's card (scroll-derived, or the sticky block card).

Examples: a Main deck on Reply with 5 blocks has 3 distinct layouts. On Context it has 2
(spread, page cards). A one-card Summary deck has 1, so the action is unavailable. Files
with three text files has 2. Files with media has 1.

### D4. Scope and persistence

- The policy belongs to a **panel and deck**: `DeckPanelState.views` holds a Main policy
  and a Files policy. It is never keyed by card, agent, or subject, so it survives
  `j`/`k`, deck switches, Ctrl+J/K, splits, zoom, and restart. The active block, scroll,
  and following stay ephemeral and per subject, as today.
- New panels (split open, other-panel picks) start `AUTO`. Unsplitting keeps panel 0's
  policies.
- A change made while zoomed is written to both the live state and the zoom snapshot, so
  it survives unzoom and is what gets persisted (persistence unwraps the zoom).
- **File format decision:** add an optional `views` object to each panel entry of the
  existing schema-version-1 `ace_agents_deck_state.json` rather than bumping the
  version. Today's decoder already ignores unknown panel keys, so older builds keep
  reading the file (they just drop `views`), and no migration code is needed. A missing,
  non-object, or unknown-valued `views` decodes to `AUTO`. A Files `page_blocks` decodes
  to `AUTO`. Unknown keys are ignored. Always serialize both keys, for example
  `{"deck": "main", "preferred_card": "reply", "views": {"main": "page_blocks", "files": "auto"}}`.

### D5. The badge: effective view plus policy, beside the deck name

The badge lives in the **top border title, immediately after the deck name**, before the
card tabs. Placing it after the fixed-width deck name keeps its position constant while
card and file titles change width, so the eye learns where to look. The layout name
encodes both levels at once. Showing two independent fields
(`DECK PAGED · BLOCKS SPREAD`) would imply two independent switches, which is the very
model this design rejects.

```text
◆ MAIN  page blocks · auto ┃ Context │ Reply  2/2
◆ MAIN  spread · fixed ┃ Context │ Reply  2/2
◆ MAIN  blocks · auto ┃ ‹ 2/2 › Reply                (compact)
MAIN B·A 2/2                                         (micro)
▤ FILES  page cards · spreading… ┃ ‹ 2/5 › app.py     (fixed spread, probe in flight)
▤ FILES  page cards · spread unavailable ┃ ‹ 1/3 › shot.png   (fixed spread, media)
```

**Label rule.** The badge shows the effective layout, with one exception. When the
policy is fixed and the requested layout has the same D3 signature as the effective one
(a vacuous request, such as fixed page blocks while viewing blockless Context), it shows
the requested name. A fixed badge therefore stays stable while you Ctrl+J between cards,
without ever claiming a picture that is not on screen. When a hard constraint changes
the picture (Files media) or the change is still pending (Files probe), the badge shows
the effective layout and names the request in the status segment. It never labels an
unrendered request as on screen.

**Variants** (pure helper, built in the model phase):

| Part           | long                 | short        | tiny |
| -------------- | -------------------- | ------------ | ---- |
| `SPREAD`       | `spread`             | `spread`     | `S`  |
| `PAGE_CARDS`   | `page cards`         | `cards`      | `C`  |
| `PAGE_BLOCKS`  | `page blocks`        | `blocks`     | `B`  |
| auto           | `auto`               | `auto`       | `A`  |
| fixed          | `fixed`              | `fixed`      | `F`  |
| pending spread | `spreading…`         | `spreading…` | `…`  |
| blocked spread | `spread unavailable` | `no spread`  | `!`  |

long/short join as `{layout} · {status}` and tiny joins as `{layout}·{status}`.

**Width ladder.** The first rung that fits the chrome budget wins. The badge outranks
inactive card tabs:

1. full tabs + long badge
2. full tabs + short badge
3. compact tabs + long badge
4. compact tabs + short badge
5. compact tabs + tiny badge
6. micro + tiny badge (`MAIN B·A 2/2`)
7. micro alone

Files decks with more than 4 tabs already skip the full rungs.

**Style.** The badge is quiet by default and loud only when the user has changed it.
Focused panel: the layout word uses the deck accent (not bold), `·` uses the separator
color, `auto` and `spreading…` are muted, `fixed` is bold accent, and
`spread unavailable`/`no spread` use a muted warning tone (`#D7AF5F`). Unfocused panel:
everything is dim, like the rest of the title. The meaning never depends on style:
`auto`, `fixed`, `A`, and `F` are text, so monochrome captures stay legible.

**Where no badge appears:** the Tools deck, an empty deck (empty state shown), and a
panel that has never painted a full Main document.

**Partial paints.** The badge is recomputed only from full documents. While a partial
Main document is shown (header-only paints during `j`/`k`), the panel keeps the badge
from its last full document, so fast navigation never flickers between labels.

**Old subtitle tag.** Remove the `spread` tag and the `spread=` parameter from
`deck_subtitle()`. The bottom border keeps the Files line status and the deck switcher.

### D6. Block-rail cue

The rail (shown for a paged Main deck whose active card has two or more blocks) gets a
small muted cue in its widest tier, just before the key hint: `page 3/5` when blocks are
paged, and `all 5` when they are inline. Ladder: entries + cue + hint, then entries +
hint (today's full-hint tier), then the unchanged remaining tiers. The rail stays a
navigation control; the title badge is the source of truth.

### D7. Transitions keep the reader's place

A view change is a layout change, never navigation:

- Capture one `_ReadingAnchor` (card, block, offset, pin) with the old modes before
  recomposing, and restore it in **every** direction. Today `_apply_main_transition` and
  `_refresh_main_mode_for_shown` do not restore anchors for paged→spread; they land on
  the preferred card or do a sticky landing. The view-change path must restore spread
  targets too: pinned → `pin_to_bottom()`; block anchor → block row in the spread
  document plus offset (deck-spread publishes `block_anchor_rows`); otherwise
  `spread_body_start(card)` plus offset. Every restore retries after layout, bounded.
- When entering page blocks from an inline layout, show the anchor's block, not the
  newest. If the card's cursor differs from the anchor block, align it with
  `select_block_cursor(card, anchor.block_id)`. That is exactly what the spread
  scroll-sync would have done, so `following` stays equal to "the reader is on the
  newest block".
- Do not change the preferred (sticky) card, do not step blocks, and do not touch
  `following` otherwise.
- **Rapid presses:** a per-panel view-transition generation counter makes every deferred
  restore callback a no-op once it is stale. While a restore is still pending, the next
  change reuses that pending anchor instead of capturing from a scroll offset that has
  not been restored yet. A subject change clears the pending anchor.
- **Fresh auto decision:** switching the policy (including back to `AUTO`) decides with
  `same_subject=False` for the deck mode, and clears `_block_mode_key` so the block mode
  is also decided without hysteresis. The result is deterministic: exactly what Auto
  would pick for this content.
- New subjects under a fixed policy use today's landing rules (sticky Reply, newest
  block). Anchors are only for view changes on the same subject.
- The keypress stays in memory: no I/O and no rebuild of Main source data. Only the
  Files spread probe does I/O, and it runs off-thread (D8).

### D8. Files specifics

- `PAGE_CARDS` fixed: always paged; skip the spread probe entirely for new subjects.
- `SPREAD` fixed: spread needs **complete** pages. Today's probe stops at the row
  budget, so its pages can be truncated. Add a complete mode that reads every page,
  capping each page's text at the paged view's render cap
  (`FILE_PANEL_MAX_RENDER_LINES`) and marking truncated pages. Truncated pages render
  the same truncation hint the paged view uses, so forced spread never silently drops
  content. Auto-probe pages count as complete only when the probe did not exceed its
  bound.
- While the complete probe is in flight, the deck stays paged and the badge reads
  `page cards · spreading…`.
- If the probe finds media (`has_solo`), the deck stays paged and the badge reads
  `page cards · spread unavailable`. The fixed preference is kept for the next
  compatible subject. When the user caused the change (P or palette), post one concise
  toast: "Files stays paged: images and videos can't spread". Never toast on `j`/`k`.
- Stale probe results (slots, subject, or policy changed) are dropped, as today, and
  re-evaluated against the current policy.

### D9. Controls

- **Key:** `cycle_deck_view: "P"` in `ace.keymaps.app`. It is Agents-only, acts on the
  focused panel's current deck, and is gated by D3 availability. `P` has no app-level
  binding today (Statistics, Memory, and Snippets use it inside their own modals; the
  prompt editor's vim `P` owns keys while typing), so no contextual-duplicate entry is
  needed. Verify this with tests.
- **Footer:** `P view` when the focused panel's cycle is available. It is conditional,
  per the footer convention.
- **Help (Agents › Navigation):** row `P`, "Cycle deck view (auto: palette)" (within the
  32-character limit).
- **Palette:** the app command `Cycle deck view` (aliases: view, layout, spread, paged,
  page, cards, blocks, P), plus four palette-only direct commands with executor
  `set_deck_view_at(index)` over
  `DECK_VIEW_CHOICES = (AUTO, SPREAD, PAGE_CARDS, PAGE_BLOCKS)`:
  - `agents.deck_view.auto` "Deck view: automatic"
  - `agents.deck_view.spread` "Deck view: spread (fixed)"
  - `agents.deck_view.page_cards` "Deck view: page cards (fixed)"
  - `agents.deck_view.page_blocks` "Deck view: page blocks (fixed)"

  Direct commands are available on Agents when the focused deck is Main or Files, is not
  empty, and (for Main) is not partial. The command equal to the current policy is
  hidden, so "automatic" appears only when fixed. `page_blocks` is Main-only. A direct
  choice may be vacuous for the current card; that is deliberate, since it applies to
  future Reply cards.

- **Committed deck search:** add `cycle_deck_view` to `deck_structural_exit_keys`.
- **Toasts:** none for routine presses, because the badge changes in place. Exactly one
  teaching toast when a `P` press turns `AUTO` into fixed: "Main view fixed · palette
  “Deck view: automatic” undoes" (adapt the deck name; keep it at most 70 characters).
  Plus the Files media toast (D8).

### D10. Performance

Fixed views skip measurement, so they are cheaper than Auto on every subject change. The
risk is one large recomposition: forcing spread or page cards on a huge Reply. That
resembles the pre-blocks whole-Reply render. Do not add an arbitrary size refusal. The
verify phase measures against these budgets on the existing 5,000-line Reply fixture (10
turns × 500 lines):

- p50 at most 150 ms and p95 at most 300 ms key-to-paint for each `P` transition.
- A new pathological 14,000-line Reply fixture must stay under 1,000 ms max, with no
  stall-watchdog row.
- A forced Files spread of 20 files × 2,000 lines must never block the event loop on the
  keypress (the probe is off-thread) and must paint within 1,000 ms of probe completion.

Mitigations, in this order, only when a budget is missed: remove redundant work (double
render or measurement); follow the established pump-safe patterns from `tui_perf.md` so
the badge paints first; and only as a last resort, an explicit measured guard that the
UI explains.

### Non-goals

- No Tools-deck views, no per-card or per-agent policies, and no config key for a
  default policy.
- No change to the automatic thresholds, hysteresis, or `spread_max_screens` semantics
  (they now apply only under `AUTO`; document this).
- No deck-picker modal changes and no dedicated reset key.
- No `sase_core` changes.

## Conventions for every phase

- Read `tui.md` and `tui_perf.md` with `/sase_memory_read` before changing TUI code, and
  `lint_and_test.md` before finishing. Verify with `sase tool run check`.
- Keep every touched file under the `toobig` 1,000-line hard limit. Put new logic in new
  modules (named below) rather than growing `panel_blocks.py` (703 lines),
  `panel_transitions.py`, or `_agent_detail_decks.py`. Run `just toobig` before
  finishing.
- Mirror the defensive style of the deck widgets: panel sync helpers never raise and
  fail open to today's behavior. Keystroke and check_action paths stay O(1) by using
  cached predicates.
- When rendered TUI output changes, update goldens with `just fix-tui-screenshots`
  (targeted selectors after `--`; hand long runs to `/sase_monitor`). Inspect every
  created or updated golden before finishing; generation is not approval.
- Record anything out of scope as `PROPOSED FOLLOW-UP:` notes on your own phase bead.

## Phase: model

Pure code and persistence; no widget changes.

1. `src/sase/ace/tui/widgets/decks/model.py`:
   - Add `class DeckView(StrEnum)` with `AUTO="auto"`, `SPREAD="spread"`,
     `PAGE_CARDS="page_cards"`, `PAGE_BLOCKS="page_blocks"`.
   - Add a frozen `DeckViewPolicies(main: DeckView = AUTO, files: DeckView = AUTO)` with
     `for_deck(deck)` (Tools → `AUTO`) and `with_deck(deck, view)` (rejects invalid
     combinations: Tools, Files `PAGE_BLOCKS`).
   - Add `views: DeckViewPolicies = DeckViewPolicies()` to `DeckPanelState`.
   - Fix `with_panel_deck` and `with_preferred_card` to use `dataclasses.replace` so
     they keep `views`. Today they rebuild `DeckPanelState(deck, preferred)`
     positionally, which would silently reset views.
   - Add `with_panel_view(state, index, deck, view)`. It updates `state.panels[index]`
     and, when zoomed, the same index in `state.zoom_snapshot`.
   - Add `DECK_VIEW_CHOICES` (palette order: AUTO, SPREAD, PAGE_CARDS, PAGE_BLOCKS).
2. New `src/sase/ace/tui/widgets/decks/view_policy.py` (pure, no Textual):
   - `ViewContent`, `BlockState`, and `layout_signature(layout, content)` per D3.
   - `forced_deck_mode(policy, card_count) -> RenderMode | None` (`None` means decide
     automatically; `card_count <= 1` → `SPREAD`).
   - `forced_block_mode(policy, block_count) -> RenderMode | None`.
   - `distinct_layouts(content)` (shallowest representative per class, depth order).
   - `next_view(current_layout, content) -> DeckView | None` (widening with wrap per D2;
     `None` when fewer than two distinct layouts).
   - `ResolvedView(deck, policy, shown, status)` with `status` in {`ok`, `pending`,
     `blocked`}, and
     `resolve_view(deck, policy, effective_layout, content, *, pending, blocked)`
     implementing the D5 label rule.
3. New `src/sase/ace/tui/widgets/decks/view_badge.py` (pure):
   `badge_variants(resolved, *, accent, focused) -> tuple[Text, Text, Text]` (long,
   short, tiny) with the D5 text and styles.
4. `src/sase/ace/tui/models/agent_deck_persistence.py`: add `views` to
   `_DeckPanelSnapshot` and wire decode, encode, `snapshot_from_area_state`, and
   `area_state_from_snapshot` per D4. Keep `SCHEMA_VERSION = 1`, and update the module
   docstring to describe the optional field.
5. Tests: a new `tests/ace/tui/widgets/decks/test_deck_view_policy.py` covering the full
   signature/equivalence table, every D3 example, and widening with wrap from each
   layout. It must include first-from-Auto from each effective layout, fixed vacuous
   page blocks on Context → spread, and single-card/Tools/media returning `None`. It
   also covers `resolve_view` label rules and `badge_variants` plain text for every row
   of the D5 table. Extend `test_deck_model.py` (replace helpers keep views;
   `with_panel_view` while zoomed updates both). Extend
   `tests/ace/tui/models/test_agent_deck_persistence.py`: round trip, a legacy file
   without `views` → AUTO, garbage values → AUTO, Files `page_blocks` → AUTO, and zoom
   unwrap carrying views.

## Phase: main-engine

1. New `src/sase/ace/tui/widgets/decks/panel_view.py` with a `DeckPanelViewMixin`
   composed into `DeckPanel`, with `_init_view_state()` called from `__init__`. It
   holds:
   - `_view_policies`, `view_policy(deck)`, and `sync_view_policies(policies) -> bool`
     (store only; returns whether the shown deck's policy changed).
   - `set_view_policy(deck, view)` (store, then apply when that deck is shown).
   - `view_content(deck)`, `effective_layout(deck)` (from `_render_mode` and
     `main_view.block_mode_for_active_card()`), `next_view()`, and `resolved_view()`
     (Files `pending`/`blocked` come from attributes that default to `False` and are
     populated by files-engine).
   - The cached `deck_view_cycle_available` property and `_sync_view_cycle_available()`.
     It pokes `app._refresh_agent_footer_bindings_only()` when the value flips, like
     `_sync_block_navigable`. Call it wherever `_sync_block_navigable()` is called.
2. Deciders:
   - `_decide_main_mode` (`panel_spread.py`): return `forced_deck_mode(...)` before
     measuring when it is not `None`.
   - `_decide_block_mode` (`panel_blocks.py`): same with `forced_block_mode`. Keep this
     edit to a few lines.
   - `AUTO` code paths stay byte-for-byte equivalent.
3. `_apply_main_view_change()` in `panel_view.py`, per D7:
   - Take the pending-or-captured anchor; clear `_block_mode_key`.
   - Decide with `same_subject=False`; set `_render_mode`/`_mode_subject`.
   - Recompose: spread → `main_view.show_document(..., mode=SPREAD)`; paged →
     `_show_main_paged(document, anchor.card_id or current, PAGED)`, after the
     page-blocks cursor alignment.
   - Restore with generation-guarded callbacks: reuse `_restore_block_transition` for
     paged targets, and add a spread-target restore beside it (or in `panel_view.py` if
     `panel_transitions.py` would grow past its budget).
   - Then `_sync_files_views()`, `refresh_chrome()`, `_sync_block_navigable()`,
     `_sync_block_rail()`, and `_sync_view_cycle_available()`.
   - Partial or empty documents: store the policy and refresh chrome only. The next full
     document applies it through the normal deciders.
4. `DeckArea` (`area.py`): `set_panel_view(index, deck, view)` updates state via
   `with_panel_view`, then calls `panel.set_view_policy`. `apply_state` calls
   `sync_view_policies` for each panel, applying only when changed and the panel has a
   full document.
5. New `src/sase/ace/tui/widgets/_agent_detail_deck_view.py` mixin composed into
   `AgentDetail`:
   - `cycle_focused_deck_view() -> tuple[DeckView, bool] | None` returns the new view
     and whether it was a first fix from Auto, for the D9 toast.
   - `set_focused_deck_view(view) -> bool`.
   - Both call `area.set_panel_view` and then `_notify_deck_state_changed()`, so
     persistence saves through the existing coalesced writer.
6. Pilot tests in a new `tests/ace/tui/widgets/decks/test_deck_view_main_pilot.py`
   (build fixtures from `test_deck_block_paged_pilot.py` and
   `test_deck_block_spread_pilot.py`):
   - All six ordered layout transitions on a multi-block Reply keep block and offset.
   - Pinned-bottom stays pinned.
   - Entering page blocks from spread shows the scroll-derived block, not the newest.
   - `following` is unchanged except for the D7 alignment rule.
   - A fixed policy survives `j`/`k`, Ctrl+J/K, deck switch, split and unsplit, and zoom
     in/out.
   - Resize under a fixed policy never transitions.
   - Reset to Auto matches a fresh Auto decision.
   - Rapid P-P-P ends at the correct block and offset.
   - Partial documents neither apply nor break the policy.
   - A persisted snapshot with views restores the view after a simulated restart.

## Phase: files-engine

1. `src/sase/ace/tui/widgets/file_panel/_spread_probe.py`:
   `probe_files_spread(..., complete: bool = False)`. In complete mode there is no total
   bound, and each page reads at most `FILE_PANEL_MAX_RENDER_LINES` lines (+1 to detect
   truncation) for plain paths and commit diffs; cached live and linked text is sliced
   the same way. Add `truncated: bool` to `FilesSpreadPage`. It must never read a whole
   huge file into memory.
2. `files_spread.py`: render a truncation hint for truncated pages that matches the
   paged file view's wording (check `lazy_renderable`'s `truncation_hint` support
   first).
3. Panel Files paths (`panel_files.py`, `_decide_files_mode` in `panel_spread.py`,
   helpers in `panel_view.py`):
   - Honor the policy in `_schedule_files_probe`: skip for `PAGE_CARDS`; complete mode
     for `SPREAD`; today's bounded probe for `AUTO`.
   - Honor it in `_decide_files_mode` and `_on_files_probe_result` (fixed `SPREAD`
     spreads only with complete pages and no media).
   - Track `_files_probe_complete`, `_files_spread_pending`, and `_files_spread_blocked`
     per subject, feeding `view_content` (`spread_blocked`) and `resolved_view`.
   - `_apply_files_view_change()`: dispatch from `set_view_policy(FILES, ...)`. It
     transitions through the existing `_apply_files_transition`, which keeps the page
     and offset.
   - Post the D8 media toast only for user-initiated changes. Pass a flag from the
     AgentDetail API; never toast on subject changes.
4. Tests:
   - Probe unit tests: complete mode reads all pages; per-page cap with the truncated
     flag; media short-circuit; bounded mode unchanged.
   - Pilot tests: fixed spread on a new subject spreads after the probe; fixed page
     cards never probes; `spreading…` while in flight; a media deck keeps the preference
     and shows blocked, and the next text subject spreads; stale results are dropped;
     reset to Auto re-decides.

## Phase: chrome

1. `titles.py`: `deck_title(..., view: ResolvedView | None = None)` implements the D5
   ladder using `badge_variants`. Remove `spread` from `deck_subtitle()` and its
   candidate list.
2. `panel_chrome.py`: `refresh_chrome()` passes the badge view. Main holds its last
   full-document `ResolvedView` per deck while `_main_document.partial` (keep the hold
   here so this phase does not edit `panel_view.py`). No badge for Tools, an empty deck,
   or before the first full paint. Drop the `spread=` argument.
3. `block_rail.py`: add an optional `mode_cue` argument to `set_rail` and
   `_render_block_rail`, with the D6 ladder. `panel_blocks._sync_block_rail` computes
   `page N/M` or `all N` from the view's block mode and the active block index.
4. Tests: extend `test_deck_titles.py` (every rung at chosen widths, never exceeding the
   budget, badge surviving before inactive tabs, focused and unfocused styles, `.plain`
   legibility for every status, and a subtitle without a spread tag), plus
   `test_deck_chrome.py` (partial hold: no flicker across a header-only paint; Tools and
   empty decks have no badge) and `test_deck_block_rail.py` (cue tiers and fallback).
5. Goldens: every Agents-deck golden changes (title badge; the Files spread golden also
   loses its subtitle tag). Regenerate the `agents_decks*`, `agents_deck_blocks*`,
   `agents_deck_picker*`, and any other golden showing a deck title. Inspect each one.

## Phase: controls

1. Keymaps:
   - `src/sase/default_config.yml` `ace.keymaps.app`: add `cycle_deck_view: "P"` with a
     comment.
   - Add the `AppKeymaps` field, the `_BINDING_META` row
     `("cycle_deck_view", "Cycle Deck View", False)`, and the `DEFAULT_BINDINGS`
     fallback in `src/sase/ace/tui/bindings.py`.
   - Update the binding-count and parity tests.
2. `actions/agents/_panel_detail.py`:
   - `action_cycle_deck_view()`: Agents only; it calls
     `AgentDetail.cycle_focused_deck_view()`, posts the D9 first-fix toast, and
     refreshes footer bindings.
   - `action_set_deck_view_at(index)`: palette.
3. `_app_action_availability.py`: add a `_DECK_VIEW_ACTIONS` set that is blocked while
   the prompt owns keys and outside Agents. Otherwise it returns the focused panel's
   cached `deck_view_cycle_available`.
4. Footer (`_display_detail_footer.py`, `_keybinding_modes.py`,
   `_keybinding_bindings_agents.py`): add a `deck_view_cycle_available` kwarg that
   renders `(kd("cycle_deck_view"), "view")`.
5. Help (`modals/help_modal/agents_bindings.py`): add the D9 row next to the card-block
   row.
6. Palette:
   - Add a `NAV_COMMAND_META` row for `cycle_deck_view`.
   - Add a `_iter_deck_view_commands()` generator in `commands/catalog.py` (mirror
     `_iter_deck_picker_commands`) and register it in `build_command_catalog`.
   - Add `CommandContext` fields `deck_view_deck`, `deck_view_policy`, and
     `deck_view_cycle_available` (`types.py`, `context.py`).
   - Add availability rules in `commands/_availability_agents.py` per D9.
7. `actions/agents/_deck_search_host.py`: add `cycle_deck_view` to the exit ids.
8. Tests: a new `tests/ace/tui/widgets/decks/test_deck_view_keys.py` modeled on
   `test_deck_card_block_keys.py`. It covers:
   - the default key and the no-conflict check;
   - that `P` inside the Statistics/Memory/Snippets modals still reaches their actions,
     and that the prompt editor owns `P` while typing;
   - gating: Tools, empty deck, partial, single-card deck, and prompt-owns-keys;
   - handler dispatch, the footer flip, the help row, and the search exit;
   - palette specs, availability (current policy hidden, `page_blocks` only on Main,
     automatic only when fixed), and execution;
   - that the first-fix toast fires once, and the Files media toast.
9. Goldens: regenerate and inspect the goldens whose footer now shows `P view`.

## Phase: verify

1. New `tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_views.py`, with
   deterministic fixtures reused from the deck-blocks visual file:
   - `agents_deck_view_auto_page_blocks_120x40` (badge `page blocks · auto`, rail
     `page N/M`)
   - `agents_deck_view_fixed_page_cards_120x40` (one `P`)
   - `agents_deck_view_fixed_spread_120x40` (two `P`)
   - `agents_deck_view_split_narrow_120x40` (left-right split: focused fixed vs
     unfocused auto, compact rungs)
   - `agents_deck_view_files_fixed_spread_160x40`
   - `agents_deck_view_files_media_blocked_120x40`
2. Live check: capture with `sase screenshot` (`--keep`, drive `P`, Ctrl+J, `|`, `Z`,
   recapture). At wide and narrow widths, confirm the D5 visual test: with the body
   covered, the title alone tells which panel and deck, whether the deck is spread or
   paged, whether the active card's blocks are inline or paged, and whether that is
   automatic or fixed.
3. New `tests/ace/tui/bench_tui_deck_view.py` (`slow` marker; reuse
   `_bench_tui_jk_helpers.py` and the `bench_tui_jk_blocks.py` fixture builders).
   Measure `P` key-to-paint p50/p95/max for each transition on the 5,000-line Reply, a
   new 14,000-line pathological Reply fixture, and forced Files spread against the D10
   budgets. Record the numbers in the phase bead notes. Apply D10 mitigations only when
   a budget is missed, and re-measure.
4. Walk the acceptance checklist below and fix any gaps.

## Phase: docs

1. `docs/ace.md`:
   - Add `P` to the navigation key table.
   - In "Agent Data Decks and Cards" and "Card Blocks", add a "Deck views" subsection:
     the three layouts, Auto versus fixed, the badge legend (long, short, and tiny
     forms), `P` widening order and skipping, palette direct choices and "Deck view:
     automatic", per-panel/per-deck persistence in `ace_agents_deck_state.json`, Files
     constraints (media, pending, truncation), and the rail cue.
   - Note that `spread_max_screens` and `block_spread_max_screens` apply only under
     Auto.
2. `docs/configuration.md`: add the `cycle_deck_view` keymap; note in `ace.agent_decks`
   that fixed views bypass the thresholds.
3. Do not edit SASE memory. Record `PROPOSED FOLLOW-UP:` on the phase bead: add a "Deck
   View" glossary strand, and mention the view in the Agent Data Card / Card Block
   strands.

## Acceptance checklist

- Each layout and Auto is correct at wide and narrow widths, focused and unfocused, and
  legible as plain text (monochrome).
- A blockless active card under fixed page blocks shows `page blocks · fixed` and pages
  the card. Files media shows `spread unavailable`; Files in flight shows `spreading…`.
- The first `P` from each Auto-resolved layout lands on the next distinct wider layout.
  Skipping never produces a no-op press. The action is hidden/disabled for Tools, empty,
  partial, and single-layout decks.
- Card, block, offset, and bottom pin survive every transition. `following` never
  changes except by the D7 alignment rule, and rapid presses stay correct.
- Views persist per panel and per deck across agents, deck switches, splits, zoom, and
  restart. A legacy file loads as Auto, and older builds still read the new file.
- The badge never flickers during `j`/`k` partial paints, and the subtitle no longer
  carries `spread`.
- Benchmarks meet the D10 budgets (or the documented, measured mitigation is in place).
