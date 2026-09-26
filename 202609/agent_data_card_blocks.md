---
tier: epic
title: Agent data card blocks - per-shell blocks for the session Reply card
goal: 'Agents-tab deck panels gain a third level, deck -> card -> block. An agent
  session''s Reply card is split into one block per concrete sase shell. A card shown
  alone spreads its blocks when they fit `ace.agent_decks.block_spread_max_screens`
  and pages them one shell at a time otherwise. Every node lands on its newest block,
  `[` / `]` step to older / newer blocks, and a one-row block rail shows the session
  timeline. The result is intuitive, reliable, fast, and beautiful.

  '
phases:
- id: card-block-model
  title: CardBlock data model, walkers and block anchors
  depends_on: []
  size: medium
  description: 'card-block-model: add the transparent CardBlock and BlockSpreadOnly
    wrappers, validated CardPart preamble/blocks accessors, one is_card_container
    helper that every renderable walker uses, salted per-block hint caching, the BLOCK
    section-anchor role, and block_id divider meta. There is no visual change.'
- id: block-cursor-model
  title: Pure block cursor, block-mode decision and config key
  depends_on: []
  size: small
  description: 'block-cursor-model: add the pure block_model module (BlockCursor land/reconcile/step/select,
    arrivals, cycle_block_id, derive_spread_block, decide_block_mode) and the ace.agent_decks.block_spread_max_screens
    setting with its default-config, schema and parity tests.'
- id: session-reply-blocks
  title: Session Reply cards emit one block per sase shell
  depends_on:
  - card-block-model
  size: medium
  description: 'session-reply-blocks: make the agent-session Reply builder wrap each
    concrete shell phase in a CardBlock whose BlockMeta matches the JUMP roster, in
    both hint and non-hint modes. Remove the vestigial blank + rule + blank prefix
    that opens every Reply/Output card, update the test walkers, and regenerate the
    affected goldens.'
- id: legacy-followup-blocks
  title: Blocks for the legacy followup_agents Reply path
  depends_on:
  - session-reply-blocks
  size: small
  description: 'legacy-followup-blocks: give the still-reachable non-session followup_agents
    Reply path the same per-phase blocks in both modes. This splits its single-Text
    hint twin per phase and adds the missing gate branch.'
- id: block-paged-view
  title: Block-paged projection, newest landing and the card_blocks flag
  depends_on:
  - session-reply-blocks
  - block-cursor-model
  size: medium
  description: 'block-paged-view: create the card_blocks beta flag. Add DeckPanelBlocksMixin
    and the MainDeckView block mixin, which decide the block mode for a card shown
    alone, render one block per page, and land on the newest block. They also follow
    new shells, keep the reader''s block by id, and expose cycle/select and a cached
    navigable predicate. Add the sticky-Reply bench fixture.'
- id: block-spread-view
  title: Block-spread and deck-spread block navigation and transitions
  depends_on:
  - block-paged-view
  size: medium
  description: 'block-spread-view: add chat-log newest landing and anchor-motion navigation
    for block-spread cards and spread decks, the scroll-derived block cursor, and
    a block-aware layout reserve. Replace the ad hoc spread/paged anchoring with one
    hierarchical ReadingAnchor capture/restore that also covers block-mode transitions.'
- id: block-keys
  title: The [ and ] card-block keys, gating, footer, help and palette
  depends_on:
  - block-paged-view
  size: medium
  description: 'block-keys: register prev_card_block / next_card_block (defaults [
    and ]) through the whole keymap pipeline, including contextual duplicates with
    the Artifacts sub-tab keys, check_app_action gating, a conditional footer entry,
    a help row, palette metadata/availability, deck-search exit keys and the parity
    tests.'
- id: block-rail
  title: The one-row block rail
  depends_on:
  - block-spread-view
  size: medium
  description: 'block-rail: add the pure tiered block_rail_text renderer and the pre-composed
    BlockRail widget, docked under the Main deck panel''s top border. It uses roster
    numbers, glyphs and status colors, an accent pill for the active block, arrival
    dots, a key hint at the widest tier, click-to-select, focus dimming and theme
    updates.'
- id: card-blocks-cutover
  title: Remove the flag, add goldens, inspect live, and bench
  depends_on:
  - legacy-followup-blocks
  - block-keys
  - block-rail
  size: medium
  description: 'card-blocks-cutover: record the flag-off vs flag-on j/k bench, then
    remove the card_blocks flag (delete the Off branch, close the flag bead). Add
    and inspect the block-state PNG goldens, inspect live sase screenshot captures,
    and leave just check green.'
- id: card-blocks-docs
  title: User docs for card blocks
  depends_on:
  - card-blocks-cutover
  size: small
  description: 'card-blocks-docs: document the deck -> card -> block hierarchy, the
    newest-block landing and triage loop, the rail, the [ / ] keys (and the Ctrl+Shift+J/K
    override caveat) and block_spread_max_screens in docs/ace.md and docs/configuration.md.
    Record proposed glossary-strand text as a follow-up note, without editing memory.'
proposed_by: bbugyi200.athena.0s4
create_time: 2026-09-25 20:37:35
status: wip
bead_id: sase-19x
---

- **PROMPT:** [prompts/202609/agent_data_card_blocks.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_data_card_blocks.md)
- **BEAD:** [sase-19x](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19x/README.md)

# Plan: Agent data card blocks

## 1. Context

Epic `sase-17d` (closed) replaced the Agents-tab metadata panel with **deck panels**. A
deck panel shows an **agent data deck** (Main, Files, Tools), and each deck is made of
**agent data cards**. The shipped code lives under `src/sase/ace/tui/widgets/decks/`:

- `card_part.py`: the transparent `CardPart` wrapper
- `main_document.py`, `main_view.py`, `panel*.py`, `render_mode.py`, `separators.py`,
  `titles.py`

A multi-card deck is _spread_ (every card on one page) when its cards fit
`ace.agent_decks.spread_max_screens` panel heights, and _paged_ otherwise. `Ctrl+J/K`
cycles cards and a paged deck keeps the chosen card (for example Reply) as the selection
moves between nodes.

The Reply card of an agent session is the worst reading experience left. It concatenates
every sase shell's output in chronological order:

- plan and code agents, `⚙ MONITOR`, `⋔ GATE`
- p50 3 / p90 5 / max 15 shells, and p90 738 / max 14,013 raw reply lines

What you almost always want is the **newest** shell's output, and today it is the most
expensive thing to reach.

This epic adds **agent data card blocks** ("card blocks"), a strict third level: **deck
→ card → optional, non-nesting blocks**. The first and only wired use is the session
Reply card, which gets one block per concrete sase shell.

**Research.** Read `research:202609/agent_data_card_blocks/agent_data_card_blocks.md`
with `sase artifact read` for the full evidence base: the key-delivery probes, the
walker hazards, the size data and the alternatives that were rejected. The user reviewed
it and accepted every recommended requirement change. They are recorded here as binding
decisions.

**Rust boundary.** This is presentation-only Agents-tab behavior: Textual state, layout,
rendering and keybindings. The shell order and roster facts it consumes already come
from the Python `concrete_agent_session_shell_rows()` / `agent_session_roster_entries()`
projections. There is **no `sase-core` change and no `sase-core-revision.txt` bump**. If
a web or editor frontend ever needs the same session timeline, move the roster adapter
into `sase_core`, not the blocks.

## 2. Binding decisions (accepted by the user)

| #   | Original ask                                           | Decision                                                                                                                                                                                                                                          | Why                                                                                                                                                                                                                                                                                                       |
| --- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | `Ctrl+Shift+J/K` cycle blocks                          | Defaults are **`[` (older) / `]` (newer)**, both wrapping. `Ctrl+Shift+J/K` is not shipped in defaults or help. The docs describe it as a personal override for terminals that deliver it.                                                        | In the user's tmux 3.5a + kitty chain, `Ctrl+Shift+J/K` arrives as `Ctrl+J/K` and would **cycle cards, not blocks**. That was verified by two independent probes, and no tmux setting fixes it. `[`/`]` are printable, free on the Agents tab, and already mean "previous/next sub-thing" across the TUI. |
| D2  | Reverse the Reply's block order                        | **Blocks stay chronological. Every node lands on its newest block, and `[` steps back**, so the first `[` reaches the second-to-last shell (the requested behavior).                                                                              | This keeps one ordering everywhere: the JUMP roster, digit keys, hint numbers, search, `V`, `E` and streaming.                                                                                                                                                                                            |
| D3  | Spread by default, paged past a configurable threshold | Kept, with one invariant: **only a card shown alone pages its blocks.** A spread deck always spreads its blocks. The threshold is its own key, `ace.agent_decks.block_spread_max_screens` (default **1.5**; `0` means always one block per page). | Block navigation must never flip the deck's mode.                                                                                                                                                                                                                                                         |
| D4  | (unspecified)                                          | A **one-row block rail** shows the card's blocks whenever they are navigable in a paged deck.                                                                                                                                                     | Most of the "intuitive" and "beautiful" lives here.                                                                                                                                                                                                                                                       |
| D5  | "one block per `AGENT CHAT` sub-section"               | **One block per concrete sase shell**, taken from `concrete_agent_session_shell_rows()`. Block ids are shell identities and are never parsed from headings.                                                                                       | This is correct terminology and robust to repeated labels (`--mon`, `--mon-0`, feedback rounds).                                                                                                                                                                                                          |
| D6  | (open question 4)                                      | A **running** newest block lands at its top. `G` tails it, as today.                                                                                                                                                                              | This matches today's paged cards.                                                                                                                                                                                                                                                                         |
| D7  | (open question 5)                                      | The legacy non-session `followup_agents` Reply path **gets blocks too**. It is still reachable, for pre-marker roots and partially loaded or orphaned member shapes.                                                                              | The user accepted "legacy path only if still reachable", and investigation confirmed it is reachable.                                                                                                                                                                                                     |
| D8  | (cosmetic)                                             | Remove the vestigial "blank + dim 50-column rule + blank" prefix that opens every Reply/Output card.                                                                                                                                              | In spread mode it doubles the `━━ ◆ Reply ━━` separator, and in paged mode it wastes three rows.                                                                                                                                                                                                          |

Always say **card block**, `CardBlock`, `next_card_block`. Bare "block" collides with
`command_line.block_*`.

## 3. Design specification (all phases implement against this)

### 3.1 Vocabulary and invariants

| Term                           | Meaning                                                                                        |
| ------------------------------ | ---------------------------------------------------------------------------------------------- |
| **Card block**                 | One titled, stably identified unit inside a card. A session Reply card has one per sase shell. |
| **Block rail**                 | The one-row timeline of a card's blocks, docked in a deck panel.                               |
| **Block-spread / block-paged** | How a card shown alone renders its blocks: all inline, or one per page.                        |
| **Active block**               | Block-paged: the block shown. Spread: the scroll-derived block (§3.4).                         |
| **Following**                  | The panel is on the newest block, so a newly started shell becomes active automatically.       |

Invariants:

- A card has zero blocks or one or more. Block controls need **two or more**.
- Blocks are trailing and contiguous inside a card. They never nest, and ids are
  non-empty and unique.
- Block ids are data identity (a string derived from the shell's `Agent.identity`). They
  are never positions or role labels.
- Block state is **ephemeral** and **panel-local**, scoped to the current subject. It
  lives on each panel's pre-composed `MainDeckView`, so it survives deck switches, split
  rotation and zoom. It is never persisted: there is no change to
  `ace_agents_deck_state.json`.
- Navigation acts on the **focused** deck panel.
- There is **one `VerticalScroll`** per deck. Blocks are Rich structure plus anchors,
  never nested scroll widgets.
- The builders **reorder nothing**, and hint numbering stays computed over the full
  card.

### 3.2 Which cards get blocks

| Selection                                      | Card  | Blocks                                                                                          |
| ---------------------------------------------- | ----- | ----------------------------------------------------------------------------------------------- |
| Agent session container                        | Reply | One per shell from `concrete_agent_session_shell_rows()`: `AGENT (role)`, `⚙ MONITOR`, `⋔ GATE` |
| Non-session root with legacy `followup_agents` | Reply | Root phase plus one per followup                                                                |
| Everything else                                | —     | None; renders exactly as today                                                                  |

Out of scope, though the model supports them later:

- clan/tribe Summary (one block per member)
- Tools / LLM Calls (one per shell)
- attempt history
- single-agent `AGENT CHAT` turns (these are time-based, not semantic)

### 3.3 Mode matrix (per deck panel)

| Deck mode                         | What is shown          | Block mode                                                                                                                                          | Rail                                                                                      |
| --------------------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| spread                            | every card on one page | always inline                                                                                                                                       | hidden (the phase dividers suffice)                                                       |
| paged, active card has ≥ 2 blocks | that card alone        | block-spread if card rows ≤ `block_spread_max_screens × viewport`, else block-paged, with the deck's ±10% hysteresis for the same `(subject, card)` | **shown in both block modes**, so it never pops in or out when streaming crosses the band |
| paged, active card has < 2 blocks | that card alone        | —                                                                                                                                                   | hidden                                                                                    |

- Measurement reuses `measure_main_rows([card], …)` with cache key prefix
  `document.digest`, so its per-card exact height is shared with the deck-level
  decision. The cheap lower bound short-circuits huge cards.
- **Partial documents never decide a mode and never touch block state.**

### 3.4 Block cursor, landing, navigation and live updates

Each panel's `MainDeckView` keeps `{card_id: BlockCursor}` for the current subject.
`BlockCursor(block_id, following, known_ids)` is defined in §4 `block-cursor-model`.

1. **New subject** (j/k, attempt toggle, first paint): cursor = newest block,
   `following=True`, `known_ids` = all.
   - Block-paged: show the newest block's page from its top.
   - Block-spread: scroll to `min(header_row(newest), real_bottom)`. `real_bottom`
     excludes the section layout reserve (`bottom_scroll_target`). A card that fits
     therefore lands at the bottom like a chat log, with no blank tail.
   - Deck-spread with the panel's preferred card = a card with blocks (the sticky
     Reply): the same clamp replaces today's `scroll_to_card(reply)`. A Reply with < 2
     blocks keeps today's behavior.
2. **Same subject, content update:** keep the cursor's block **by id** and keep the
   scroll offset. A bottom pin (`G`) persists.
3. **New shell while following:** advance to the new newest block.
   - Paged swaps the page.
   - Spread re-lands with the §3.4.1 clamp.
   - If the view was bottom-pinned, re-pin after the swap. The page identity changes, so
     capture the pin before `prepare_section_document` and restore it after.
4. **New shell while not following:** stay put. The new rail entries get an **arrival
   dot** until the reader navigates. **Never yank a reader out of history.**
5. **`[` / `]`:** step to the older / newer block, wrapping. `following` becomes
   `target == newest`, and `known_ids` becomes all.
   - Block-paged: swap the page and reset to its top.
   - Block-spread and deck-spread: an anchor motion that top-aligns the target header,
     like spread `Ctrl+J/K`. It enables the block-aware layout reserve and uses the same
     bounded `call_after_refresh` retry.
   - In a spread deck, the current block comes from scroll. When the viewport is above
     the first block header, `]` goes to the oldest block and `[` to the newest.
   - Block navigation makes the owning card the panel's preferred card, exactly as
     `Ctrl+J/K` does via `set_deck_preferred_card`.
6. **Vanished block id:** land on the newest.
7. **Partial paint** (the header-only paint during j/k): it never touches block state.
   The rail is **cleared on subject change** and redrawn only from a full document of
   the current subject, so it never shows the previous node's shells.
8. **Scroll-derived cursor in spread modes:** the explicit cursor wins until the user
   scrolls. On a user-driven `scroll_y` change, recompute with
   `derive_spread_block(anchor_rows, scroll_y, at_real_bottom)`:
   - at the real bottom: the newest block
   - otherwise: the last block whose header row ≤ `scroll_y`
   - otherwise: none (above the first header)

   Content growth does not move `scroll_y`, so streaming never silently flips
   `following`.

9. **Mode transitions** can come from streaming crossing the band, resize, split, ratio,
   or header/jump toggles, for both deck spread↔paged and block spread↔paged. Capture
   one hierarchical `ReadingAnchor(card_id, block_id, offset_rows, pinned)` and restore
   the most specific target that survives:
   - same block + offset (clamped ≥ 0), then
   - card top, then
   - card default, then
   - document default

   This replaces the ad hoc branches in `_apply_main_transition` and
   `_refresh_main_mode_for_shown`; it does not add a layer.

### 3.5 Block-paged page content

- A block page is the block's own renderables, which begin with its phase divider (the
  block header).
- The card **preamble** is the leading non-block renderables. Items wrapped in
  `BlockSpreadOnly` are dropped on block pages. The `AGENT REPLY · N` heading is wrapped
  this way, because the rail replaces it.
- The remaining preamble (a card-level `TRACEBACK`) is shown **only above the newest
  block's page**, which is the landing page.
- Block-spread and deck-spread render the whole card exactly as the builder produced it:
  traceback, heading, then every block.

### 3.6 Keys and gating

| Key | Textual name           | Action            | Behavior            | Contextual duplicate             |
| --- | ---------------------- | ----------------- | ------------------- | -------------------------------- |
| `[` | `left_square_bracket`  | `prev_card_block` | older block (wraps) | `cycle_artifacts_subtab_reverse` |
| `]` | `right_square_bracket` | `next_card_block` | newer block (wraps) | `cycle_artifacts_subtab`         |

- Bindings are non-priority, so text inputs keep `[`/`]`.
- Available only when **all** of these hold:
  - the tab is Agents
  - the prompt input does not own keys
  - the focused panel's cached `card_blocks_navigable` is true, which requires:
    - the deck is Main and the document is not partial, and
    - either the deck is paged and the active card has ≥ 2 blocks, or the deck is spread
      and some card has ≥ 2 blocks
- A block keypress is a pure in-memory path: no file reads, stats, subprocesses, JSON or
  Main-source rebuild.

### 3.7 Visual design

**Hierarchy.** Blocks add no new separator style.

| Level | Marker                                                                                                                                                                                                             |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Deck  | panel border in the deck accent                                                                                                                                                                                    |
| Card  | title tab pill (paged) or heavy `━━ ◆ Reply ━━` rule (spread)                                                                                                                                                      |
| Block | the existing phase divider `─── AGENT (code) ─── 07:23:10 ───` in the shell's accent (purple agent `#AF87FF`, amber `⚙` monitor `#FFAF5F`, lifecycle-colored `⋔` gate), now carrying `sase_card_block` anchor meta |
| Turn  | dim `─── 07:31:02 ───`                                                                                                                                                                                             |

The border title stays deck + card only. The subtitle is unchanged.

**The block rail.**

Layout and data:

- One row, pre-composed in `DeckPanel.compose` directly above the Main scroll, and
  toggled by a CSS class. It is never remounted.
- Its horizontal padding matches the scroll's `padding: 1 2`, so entries align with the
  content. The scroll's top padding gives one blank row of breathing room below it.
- Entries run chronologically left to right and are joined by `─` in separator gray
  `#444444`.
- An entry is `{number} {glyph }{label} {status}`, all from `BlockMeta`:
  - `number`: the shell's JUMP-roster index (0-based, unpadded), in dim
  - `glyph`: `⚙`/`⋔` in the shell accent, or empty for agent shells
  - `label`: the roster label (`--plan`, `--mon-0`), in muted `#888888`
  - `status`: the roster status glyph (✓ ▶ ✗ ▲ …) in `member_status_style(bucket)`

Active and unfocused states:

- The active entry is a pill: `▐` + the entry text (no padding spaces) in
  `reverse bold <deck accent>` + `▌`, with the half-block caps in the deck accent. The
  deck accent is the same one the active card tab uses (`_resolve_accent`).
- An unfocused panel dims the whole rail and renders the pill `reverse dim`, matching
  its dim title.

Markers and interaction:

- **Arrival dot:** entries in `arrived_ids` get a `●` prefix in `#5FD7FF`, the roster's
  unread color. When an arrived entry is hidden by windowing, the dot moves onto that
  side's overflow indicator.
- **Key hint:** at the widest tier only, the right edge shows
  `{prev} older · newer {next}` from the live keymap, so it reads `[ older · newer ]` by
  default. The keys are bold in the deck accent and the words are muted. The hint's own
  brackets document the keys.
- **Click:** a click on an entry selects that block (`@click` meta) and focuses the
  panel.

Tiers come from a pure helper, and the widest one that fits wins; the rail never
overflows:

1. full + key hint
2. full
3. windowed: the active entry ± k neighbors (k = 3, 2, 1) with `‹N older` / `N newer›`
   overflow counts
4. compact: neighbors show only `{number}{status}`, with bare `‹N` / `N›` counts
5. micro: the active pill plus bare counts; the label is middle-ellipsized as a last
   resort

Target renderings (focused; the pill is shown as `▐…▌`):

```text
Landing (paged deck on Reply, newest block):
0 --plan ✓ ─ 1 ⋔ --gate ✓ ─ ▐2 --code ▶▌                            [ older · newer ]

After one [ (the second-to-last shell):
0 --plan ✓ ─ ▐1 ⋔ --gate ✓▌ ─ 2 --code ▶                            [ older · newer ]

Reading history while a new monitor starts (not following):
0 --plan ✓ ─ ▐1 ⋔ --gate ✓▌ ─ 2 --code ✓ ─ ●3 ⚙ --mon ▶              [ older · newer ]

Windowed (12 shells, narrow split):
‹7 older ─ 8 ⚙ --mon-3 ✓ ─ 9 --code ✗ ─ ▐10 --code ▶▌ ─ 11 ⋔ --gate ▲

Micro:
‹10 ▐10 --code ▶▌ 1›
```

Full panel landing mock:

```text
┌─ ◆ MAIN ┃ Context │ Reply  2/2 ─────────────────────────────────────────────┐
│  0 --plan ✓ ─ 1 ⋔ --gate ✓ ─ ▐2 --code ▶▌                [ older · newer ]  │
│                                                                             │
│  ─── AGENT (code) ─── 07:23:10 ─────────────────                            │
│  ─── 07:23:41 ──────────────────────────────────                            │
│  Reading the sticky-header code to see how the collapsed preview is built…  │
└──────────────────────────────────────────── main 2 · files 1 · tools 109 ───┘
```

### 3.8 Module map

| Module                                                      | Owner phase          | Purpose                                                                                     |
| ----------------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------- |
| `widgets/decks/card_block.py` (new)                         | card-block-model     | `CardBlock`, `BlockMeta`, `BlockSpreadOnly`, `is_card_container`, `card_block_id(identity)` |
| `widgets/decks/card_part.py`                                | card-block-model     | validated `preamble` / `blocks` accessors; `flatten` and `split` are block-aware            |
| `widgets/decks/block_model.py` (new)                        | block-cursor-model   | pure cursor and mode helpers                                                                |
| `agent_decks_settings.py`                                   | block-cursor-model   | `block_spread_max_screens`                                                                  |
| `widgets/prompt_panel/_agent_session_reply_blocks.py` (new) | session-reply-blocks | the shared phase→block assembly and `BlockMeta` adapter                                     |
| `widgets/decks/flag.py` (new, deleted at cutover)           | block-paged-view     | `card_blocks_enabled()`                                                                     |
| `widgets/decks/panel_blocks.py` (new)                       | block-paged-view     | `DeckPanelBlocksMixin`                                                                      |
| `widgets/decks/main_view_blocks.py` (new)                   | block-paged-view     | the `MainDeckView` block mixin                                                              |
| `widgets/decks/panel_transitions.py` (new)                  | block-spread-view    | `ReadingAnchor` plus the transition code moved out of `panel.py`                            |
| `widgets/decks/block_rail.py` (new)                         | block-rail           | `block_rail_text()` and `BlockRail`                                                         |

`panel.py` (744 lines) and `main_view.py` (352) only gain mixin includes and thin
delegations. New logic goes in new modules so `toobig` stays green, and
`_agent_display_agent_session_render.py` (432) must not grow.

### 3.9 Feature flag scaffolding

Phases `block-paged-view` through `block-rail` land user-visible behavior piecemeal, so
they run behind a **`card_blocks` beta flag** (default off). It is created by
`block-paged-view` with `sase flag new card_blocks`, following
`sase memory read sase_flags.md`.

- **On** (`--when-enabled`): block paging, newest landing, the rail and the `[`/`]`
  availability.
- **Off** (`--when-disabled`): cards with blocks render as one undivided card exactly as
  before (no block paging, no newest landing, no rail, and `[`/`]` unavailable).
- **Remove when** (`--remove-when`): the card-blocks epic's cutover phase has verified
  the goldens, live screenshots and j/k bench with the flag on.

Phases `card-block-model`, `session-reply-blocks` and `legacy-followup-blocks` are
unconditional. They render identically apart from D8, which is a standalone cosmetic
fix. Every flag-gated behavior gets on and off tests. `card-blocks-cutover` deletes the
Off branch, removes the registry entry and `decks/flag.py`, and closes the flag bead in
the same change. This follows the `agent_decks` precedent from `sase-17d`.

## 4. Phases

### 4.1 `card-block-model` — CardBlock data model, walkers and block anchors

This phase is infrastructure only. No builder emits blocks yet, so there is **no visual
change** and no golden churn.

1. **`widgets/decks/card_block.py`**
   - `BlockMeta`: a frozen slots dataclass with:
     - `number: str`
     - `label: str`
     - `glyph: str` (`""`, `⚙` or `⋔`)
     - `accent: str`
     - `status_bucket: str` (a roster bucket such as `Done` or `Running`)
     - `kind: str` (`agent` / `monitor` / `gate`)
   - `CardBlock(block_id, title, *renderables, meta)`:
     - a transparent Rich wrapper modeled on `CardPart`, with `__slots__`, the marker
       `__sase_card_block__ = True`, and `__rich_console__` / `__rich_measure__`
       delegating to `Group`
     - a `repr` that does not embed child object ids
   - `BlockSpreadOnly(*renderables)`: a transparent wrapper (marker
     `__sase_block_spread_only__`) for card-level chrome that block-paged pages drop.
   - `is_card_container(node)`: true for `CardPart`, `CardBlock` and `BlockSpreadOnly`.
     Every walker descends containers through this one helper.
   - `card_block_id(identity)`: a stable string from an `Agent.identity` tuple,
     `f"{type.value}|{name}|{suffix or ''}"`.
2. **`CardPart`**
   - Compute `preamble` and `blocks` once in `__init__` (extend `__slots__`) using a
     pure `partition_card_children()`.
   - If the structure is invalid (a non-block after the first block, or empty or
     duplicate ids), log a warning and treat the card as block-less. Rendering is
     unaffected because the wrappers are transparent.
   - Accessors:
     - `block_ids`
     - `block(block_id)`
     - `newest_block_id`
     - `has_block_navigation` (≥ 2 blocks)
     - `block_page(block_id) -> tuple[RenderableType, ...]`, implementing §3.5: preamble
       minus `BlockSpreadOnly`, only for the newest block, then the block's renderables
   - Constructing `CardPart` from existing children must preserve blocks. Audit
     `split_card_parts`' context merge.
3. **Walkers.** Teach each through `is_card_container`:
   - `render_mode._lower_bound_node`: descend. Otherwise each block counts as one row,
     the early exit is lost, and huge sessions get exact-rendered on every decision.
   - `util/renderable_digest._update_digest`: a `b"B"` branch hashing `block_id`,
     `title`, every `BlockMeta` field and the children. `BlockSpreadOnly` gets its own
     tag. Otherwise the `repr()[:2048]` fallback hides streaming and ▶→✓ status changes.
   - `prompt_panel/_agent_display_hints._plain_renderable_content`: descend.
   - `prompt_panel/_identity_header._find_carrier`: use the helper.
   - `flatten_card_document`: recursively unwrap `CardBlock` / `BlockSpreadOnly` too, so
     legacy consumers (the hidden source panel, `inline_document_renderable`, test
     helpers) see today's flat shape.
   - `split_card_parts`: keep top-level `CardPart` objects as-is, with blocks intact.
4. **Hint cache** (`_prepare_cached_hint_renderable`). For a card with blocks:
   - Wrap each preamble child in its own `CachedRenderable`, but keep `BlockSpreadOnly`
     as the outer wrapper.
   - Wrap each block as
     `CardBlock(same id/title/meta, CachedRenderable(Group(*children), plain, digest_salt=block_id))`.

   Add an optional `digest_salt` keyword to `util/lazy_syntax.CachedRenderable` that
   salts `_digest` while keeping `plain`/`code` unchanged. Without it, the global
   segment cache (keyed on plain-content digest) can serve another document's segments
   carrying a **different block id meta**. Cards without blocks keep today's
   one-cache-per-card shape.

5. **Anchors** (`prompt_panel/_section_navigation.py`, `_section_view.py`)
   - Add `DECK_BLOCK_META_KEY = "sase_card_block"` and `PromptPanelSectionRole.BLOCK`.
   - `_segment_section_identity` returns `(f"block:{id}", BLOCK)`. Check the CARD key
     first, then BLOCK, then section markers.
   - Add
     `SectionViewMixin.block_anchor_rows(*, width) -> tuple[tuple[str, int], ...] | None`,
     mirroring `card_anchor_rows`.
   - `resolve_section_at_row` skips BLOCK. Card anchors, TITLE navigation and the
     default layout reserve (TITLE + CARD) ignore BLOCK anchors.
6. **Divider meta.** Add a keyword-only `block_id: str | None = None` to:
   - `render_phase_divider` (`_agent_display_content.py`), which stylizes the whole
     divider line with `Style(meta={DECK_BLOCK_META_KEY: block_id})`, in the
     `_mark_section_heading` idiom
   - `build_monitor_phase` and `build_gate_phase`, which pass it through

   A block never gets two headers.

**Tests:**

- partition validity and the fallback
- `block_page` rules
- digest changes on status/meta and on content changes past 2 KB
- the lower-bound early exit still triggers with blocks
- flatten equivalence
- a hinted card keeps its blocks, with per-block salted caches, and two documents with
  identical block text but different ids publish different anchor ids
- the BLOCK anchor is published at the divider row

### 4.2 `block-cursor-model` — Pure block cursor, block-mode decision and config key

1. **`widgets/decks/block_model.py`** (pure; no Textual imports):
   - `BlockCursor(block_id: str, following: bool, known_ids: frozenset[str])`, frozen.
   - `land_cursor(ids)`: newest, following, all known; `None` when there are no ids.
   - `reconcile_cursor(ids, cursor, *, new_subject)`: lands on a new subject, a `None`
     cursor, `cursor.following`, or a vanished id; otherwise it keeps the cursor
     unchanged.
   - `step_cursor(ids, cursor, direction)`: uses `cycle_block_id`. It sets
     `following = target == ids[-1]` and all ids known.
   - `select_cursor(ids, block_id)`: for rail clicks and scroll derivation.
   - `arrived_ids(ids, cursor)`: ids not in `known_ids`; empty while following.
   - `cycle_block_id(ids, active, direction)`: a wrapping step. When `active` is unknown
     or `None`, `direction > 0` returns the oldest and `direction < 0` the newest.
   - `derive_spread_block(anchor_rows, *, scroll_y, at_real_bottom)`: implements the
     §3.4.8 rule.
   - `decide_block_mode(*, block_count, card_rows, viewport_rows, block_spread_max_screens, previous, same_card) -> RenderMode`:
     a thin call to `decide_render_mode`. Fewer than 2 blocks means SPREAD, `0` means
     PAGED, `card_rows=None` keeps `previous` for the same card, and the hysteresis is
     shared.
2. **Config**
   - Add `AgentDecksSettings.block_spread_max_screens` (default
     `DEFAULT_BLOCK_SPREAD_MAX_SCREENS = 1.5`) and parse it with the same coercion as
     `spread_max_screens`: bool / non-number / negative → default, and int → float.
   - Add `ace.agent_decks.block_spread_max_screens: 1.5` to
     `src/sase/default_config.yml`.
   - Add a `src/sase/config/sase.schema.json` property (number, minimum 0, default 1.5).
     `ace.agent_decks` has `additionalProperties: false`.
   - Extend the parity tests in `tests/ace/tui/widgets/decks/test_deck_spread_pure.py`.
     They currently assert the exact `agent_decks` dict.
   - Leave `docs/configuration.md` to `card-blocks-docs`.

**Tests:** exhaustive pure tests for:

- landing, following vs not following, arrivals, vanished ids and wrap
- unknown-anchor stepping
- derive at the bottom, mid-scroll and above the first header
- the hysteresis band
- `0`
- config coercion

### 4.3 `session-reply-blocks` — Session Reply cards emit one block per sase shell

1. **Shared roster facts.** In `_agent_display_agent_session.py`, factor the per-shell
   facts of `agent_session_roster_entries` into one helper that both it and blocks use.
   The facts are:
   - label (`agent_session_member_label`)
   - kind + glyph
   - status bucket (monitor/gate/agent bucket logic)

   The helper skips the duration/digest work blocks do not need, so the rail matches the
   JUMP roster by construction. The number is the shell's chronological index, which is
   its JUMP roster number.

2. **`prompt_panel/_agent_session_reply_blocks.py`** holds the per-phase loop, extracted
   from `_update_agent_session_display`. One loop serves hint and non-hint modes, and
   hint numbering is unchanged. Each phase's parts are wrapped in
   `CardBlock(card_block_id(phase.identity), get_phase_label(phase), *parts, meta=BlockMeta(...))`,
   where:
   - the parts come from `build_monitor_phase(..., block_id=)`,
     `build_gate_phase(..., block_id=)`, or divider + reply renderables
   - `accent` is `PHASE_DIVIDER_ACCENT`, `MONITOR_GLYPH_COLOR`, or
     `phase.gate_accent or "#0BCDEC"`

   The `AGENT REPLY · N` heading goes in `BlockSpreadOnly`, and the card is
   `reply_card(*traceback_parts, BlockSpreadOnly(heading), *blocks)`. Expose a small
   `phase_card_block(...)` helper for `legacy-followup-blocks` to reuse.

3. **D8 rule sweep.** Remove the leading blank + dim 50-column rule + blank from every
   card with id `reply` (titled Reply or Output) whose first renderable opens with it.
   The sites include:
   - session and legacy followup
   - single-agent `AGENT CHAT`
   - attempts
   - step output
   - proc/monitor/gate Output
   - the matching hint twins in `_agent_display_hint_body.py` and
     `_agent_display_hint_sections.py`

   Rules that separate sections _inside_ a card stay, and the Context card is untouched.
   Grep for `"─" * 50` / `"─" * 50` and verify each site. Leave a site alone when the
   rule does not open a Reply/Output card.

4. **Test walkers.** Update every test helper that walks card documents by type so it
   uses `is_card_container` or `flatten_card_document`. Known ones:
   - `_section_ids` in `test_agent_display_agent_session_render.py`
   - `_logical_plain`, `plain_of` ×2, `_plain`, `_iter_texts`
   - `test_prompt_panel_card_partition.py`'s per-card `CachedRenderable` assertion

   Grep `tests/` for `CardPart`, `card_part` and `.renderables` before landing, because
   sase-17d once left `Text`/`Group` isinstance failures behind.

5. **Goldens.** D8 changes every golden showing a Reply/Output card. Run
   `just fix-tui-screenshots` (the full run via `/sase_monitor`). Inspect every
   creation/removal and each update group in the retained report, and confirm the only
   difference is the removed three-row prefix.

**Tests:**

- a 4-shell session (plan, gate, monitor, code) yields 4 blocks in order, with the right
  ids, meta, numbers and status buckets matching `agent_session_roster_entries`
- the heading is `BlockSpreadOnly`
- hint mode yields the same block ids with monotonic hint numbers
- plain-text equivalence with the pre-change output, modulo D8
- a digest change when one shell's status changes

### 4.4 `legacy-followup-blocks` — Blocks for the legacy followup_agents Reply path

- **Non-hint** (`_agent_display_render.py` followup branch): wrap the root phase and
  each followup through `phase_card_block`. The heading goes in `BlockSpreadOnly`. Meta
  numbers are the chronological index, labels come from the shared facts helper
  (fallback: role suffix / display name), and buckets use the same bucket logic.
- **Hint twin** (`_agent_display_hint_body.py` + its caller in
  `_agent_display_hint_render.py`):
  - Stop appending every phase into one `reply_text`. Build one `Text` per phase
    (sharing `hint_counter`) and return block parts, so the caller builds
    `reply_card(<traceback text>, BlockSpreadOnly(heading), *blocks)`.
  - Add the missing `is_gate` branch, which renders gates through
    `build_gate_phase(followup, annotate=<hint annotator>, block_id=…)` like the session
    path (today hint mode renders gates as agent phases).
- **Tests:** extend the `_starter_with_monitor`-style fixtures in
  `tests/ace/tui/widgets/test_agent_prompt_panel_monitor.py`. Assert:
  - blocks in both modes, with identical ids
  - the gate renders as a gate in hint mode
  - hint numbering is monotonic
  - existing starter tests pass

### 4.5 `block-paged-view` — Block-paged projection, newest landing and the card_blocks flag

1. **Flag**
   - Run `sase flag new card_blocks -k beta` with the three sentences from §3.9, and
     paste the printed registry entry into `src/sase/feature_flags/registry.py`.
   - Add `widgets/decks/flag.py:card_blocks_enabled()`
     (`current_flags().enabled(FeatureFlag.card_blocks)`), following the removed
     `agent_decks` helper. Confirm `current_flags()` is cached, with no per-call disk
     I/O (tui_perf rule 8); cache it if it is not.
2. **`DeckPanelBlocksMixin`** (`panel_blocks.py`, mixed into `DeckPanel`)
   - Holds `_block_mode: RenderMode` and `_block_mode_key: (subject, card_id)`.
   - `_decide_block_mode(document, card)` implements §3.3 with `block_model`, reading
     `block_spread_max_screens` via `agent_decks_settings_for`.
   - Decide after the deck-mode decision on these paths, but never for partial documents
     and only when the flag is on:
     - `show_main_document`
     - `_refresh_main_mode_for_shown` (resize, deck switch)
     - `cycle_card`, when the newly active card has blocks
   - `cycle_block(direction) -> bool` and `select_block(block_id) -> bool` for the paged
     deck. They return False (a no-op) otherwise, until `block-spread-view`.
   - `card_blocks_navigable` is a cached boolean recomputed whenever the document,
     active card, deck, deck mode or flag state changes. When it flips, call
     `app._refresh_agent_footer_bindings_only` (guarded `getattr`, as `area.py` does).
     `check_app_action` reads this cache, so it must stay O(1).
3. **`MainDeckView` block mixin** (`main_view_blocks.py`)
   - Holds `_block_cursors: dict[str, BlockCursor]` and `_block_cursor_subject`.
   - On each full document, reconcile the cursors of **every** card with blocks, so
     `Ctrl+J` to Reply shows the landed block. Clear them on subject change.
   - In paged deck mode, `show_document(..., block_mode=…)` projects:
     - block-paged: `Group(*card.block_page(cursor.block_id))`, with digest
       `f"{digest}:{card}:{block}"` and identity `(subject, card, "paged", block)`
     - block-spread: the whole card (today's render, and today's top landing until
       `block-spread-view`)
   - The render key includes block mode and block id. The §3.4 rules are implemented for
     paged mode: new subject, same subject, following advance with pin carry, not
     following, vanished, and partial.
   - Expose `active_block_id(card_id)`, `arrived_block_ids(card_id)` and
     `block_mode_for_active_card` for the rail and footer.
4. **`AgentDetail.cycle_focused_card_block(direction)`** and
   **`select_focused_card_block(block_id)`** (`_agent_detail_decks.py`) delegate to the
   focused panel and then set the owning card as preferred.
5. **Flag off:** `show_document` renders exactly as today. `navigable` is always False
   and cursors are neither reconciled nor used.
6. **Bench fixture.** In `tests/ace/tui/bench_tui_jk.py`, add a sticky-Reply case over ≥
   3 sessions of 10 shells / ~5,000 reply lines, and a block-cycle (`cycle_block`) case.
   Run `SASE_TUI_PERF=1 pytest -s -m slow tests/ace/tui/bench_tui_jk.py` (via
   `/sase_monitor` if long) with the flag off and on, in SINGLE and LEFT_RIGHT. p95 must
   be < 16 ms and j/k should improve with the flag on. Record the numbers on the bead.

**Tests:** flag-on and flag-off pilot tests covering:

- landing on the newest block in block-paged mode
- `cycle_block(-1)` reaching the second-to-last block, with wrap
- a streaming new shell while following (advance) and not following (stay + arrival)
- subject reset on j/k
- partial paint never touching cursors
- split independence (two Main panels, independent cursors)
- hysteresis at the band
- `0` = always paged
- hidden blocks still searchable via `deck_search_corpus`
- `E` staying card-wide

### 4.6 `block-spread-view` — Block-spread and deck-spread block navigation and transitions

1. **Block-aware reserve.** `enable_section_layout_reserve(*, include_blocks=False)`.
   Block navigation passes True, so BLOCK anchors join TITLE/CARD in
   `get_content_height`'s reserve and the newest header can top-align. Card navigation
   keeps today's reserve.
2. **Block-spread** (paged deck, card fits):
   - Landing: `min(header_row(newest), real_bottom)`, using a deferred bounded retry
     until anchors publish. Apply it in the same cycle when anchors are already cached,
     to avoid a flash of the oldest block.
   - `[`/`]`: top-align the target header with the reserve.
3. **Deck-spread:**
   - Sticky-Reply landing per §3.4.1.
   - `cycle_block` / `select_block` operate on the card with blocks, using the
     scroll-derived current block, and make it preferred.
   - `card_blocks_navigable` includes spread decks where some card has ≥ 2 blocks.
4. **Scroll-derived cursor.** Extend `_on_main_scroll_y`. In block-spread and
   deck-spread, recompute via `derive_spread_block` using `block_anchor_rows` and
   `bottom_scroll_target`, then update the cursor with `select_cursor`. Keep it
   O(blocks) over cached anchors. The explicit cursor set by landing or navigation must
   agree with the recompute its own programmatic scroll triggers.
5. **`ReadingAnchor`** (`panel_transitions.py`):
   - Move `_apply_main_transition` / `_refresh_main_mode_for_shown`'s anchoring out of
     `panel.py`, and reimplement both deck spread↔paged and block spread↔paged with one
     capture/restore per §3.4.9.
   - With `block_id=None` it must reproduce today's math exactly. **All existing
     `test_deck_spread_pilot.py` / split / zoom tests pass unchanged**, and the
     duplicate one-shot preferred-card behavior is preserved.
   - Offsets clamp at 0, so a bottom-landed newest block becomes the newest page's top.

**Tests:**

- block-spread landing at the bottom and at the newest header when the card is taller
- `[` top-aligning with a blank tail
- deck-spread sticky landing and `]` from Context → oldest / `[` → newest
- scroll-derived cursor after a user scroll, and streaming growth not flipping
  `following`
- block-spread → block-paged on growth keeping the reader's block and offset
- deck spread → paged into a block page
- flag off unchanged

### 4.7 `block-keys` — The [ and ] card-block keys, gating, footer, help and palette

Use `next_deck_card` / `prev_deck_card` and the `{`/`}` `grow_deck_panel` ↔
`cycle_artifacts_split` pair as templates at every site:

1. `keymaps/app_keymaps.py`: add `prev_card_block: str` and `next_card_block: str` in
   the Navigation block.
2. `src/sase/default_config.yml` under `ace.keymaps.app`:
   `prev_card_block: "left_square_bracket"` and
   `next_card_block: "right_square_bracket"`, with a comment noting the Artifacts
   sub-tab sharing. This is the core gotcha.
3. `keymaps/metadata.py` `_BINDING_META`: add
   `("prev_card_block", "Older Card Block", False)` and
   `("next_card_block", "Newer Card Block", False)` after the deck-card rows.
   `ace/tui/bindings.py` `DEFAULT_BINDINGS` gets matching fallback `Binding`s.
4. `keymaps/registry.py` `_CONTEXTUAL_APP_DUPLICATES`: add
   `{next_card_block, cycle_artifacts_subtab}` and
   `{prev_card_block, cycle_artifacts_subtab_reverse}`.
5. `_app_action_availability.py`: a card-block action set with the Agents-tab and
   prompt-ownership gates (like `_DECK_NAV_ACTIONS`) **and** the focused panel's
   `card_blocks_navigable`.
6. Palette:
   - `commands/_app_metadata_nav.py`:
     `("prev_card_block", "Older card block", "Navigation", AGENTS_ONLY, ("block", "shell", "reply", "card", "["))`
     and its `next` twin
   - `commands/types.py` / `commands/context.py`: a `card_blocks_navigable` context
     field
   - `commands/_availability_agents.py`: return it
7. Handlers: `actions/agents/_panel_detail.py` gets `action_prev_card_block` /
   `action_next_card_block`, each with an Agents guard, calling
   `AgentDetail.cycle_focused_card_block(∓1)`.
8. Help: `modals/help_modal/agents_bindings.py` gets the row `"{[} / {]}"` → "Older /
   newer card block" (≤ 32 chars, 57-column box; `src/sase/ace/CLAUDE.md`).
9. Footer: thread `card_blocks_navigable` through
   `actions/agents/_display_detail_footer.py` → `widgets/_keybinding_modes.py` (the stub
   and `update_agent_bindings`) → `_keybinding_bindings_agents._compute_agent_bindings`,
   which appends `("[/]", "blocks")` only when true. It is conditional per
   `src/sase/ace/CLAUDE.md`.
10. `actions/agents/_deck_search_host.deck_structural_exit_keys`: add both actions.
11. Tests:
    - `test_keymaps_app_bindings.py`: per-key order `]` →
      `["cycle_artifacts_subtab", "next_card_block"]`, and `by_action` / fallback
    - `test_keymaps_defaults_panels.py`
    - `test_command_catalog*.py`
    - a `test_keymaps_validation.py` pair test (a user override sharing the key is
      accepted)
    - key resolution per tab, modeled on
      `test_artifacts_paging_chords_resolve_only_on_artifacts`
    - gating (`test_deck_split_keys.py` `_check` harness)
    - footer on/off
    - palette availability
    - `test_keymaps_display_help_agents.py`

    Update any help-modal PNG goldens the new row changes (targeted
    `just fix-tui-screenshots`, inspected).

### 4.8 `block-rail` — The one-row block rail

1. **`widgets/decks/block_rail.py`**
   - A pure
     `block_rail_text(entries, *, active_id, arrived_ids, width, accent, focused, key_hint) -> Text`,
     implementing every §3.7 tier and style. The widest fitting tier wins, and
     `cell_len(result.plain) <= width` always holds, including for wide glyphs.
   - `BlockRail(Static)`:
     - one row, `can_focus=False`
     - it subscribes to theme changes like the chrome
     - it re-renders on resize
     - `@click` meta on entries dispatches to an action that posts `BlockRailSelected`
       (block id), which `DeckPanel` routes to `select_block`
2. **Compose + CSS**
   - Yield `BlockRail(classes="deck-block-rail")` in `DeckPanel.compose` directly before
     the Main scroll.
   - In `styles.tcss`, `.deck-block-rail { height: 1; padding: 0 2; display: none; }`
     and `.deck-block-rail.-shown { display: block; }`.
3. **Sync** (`DeckPanelBlocksMixin._sync_block_rail()`)
   - Show the rail only when all of these hold: the flag is on, the deck is MAIN, the
     deck mode is paged, the active card has ≥ 2 blocks, the document is full and for
     the current subject, and neither the search overlay nor the empty state is shown.
   - Otherwise hide it and clear its content.
   - Call it from these paths:
     - Main document arrival
     - card and block navigation
     - scroll-derived cursor changes
     - focus change
     - theme change
     - resize
     - deck switch
     - search overlay show/hide
     - subject change (clear)
   - The rail is O(blocks), built from `BlockMeta`, with no render.
4. **Pill caps.** Verify with a live screenshot that the `▐`/`▌` caps render cleanly in
   the bundled font. If they do not, fall back to a space-padded pill; the rest of the
   design is unchanged.

**Tests:**

- pure tier selection across widths 20–200
- the active pill at the first, middle and last position
- the arrival dot inline and on an overflow indicator
- unfocused styling
- the key hint only at the widest tier, using live keymap names
- label ellipsis
- pilot tests: visible exactly per the rule, clears on subject change / partial, follows
  `[`, click selects, and a dual-panel split keeps independent pills

With `SASE_FEATURE_FLAGS='{"card_blocks": true}'`, capture `sase screenshot` PNGs of a
real session. Cover a single panel, a LEFT_RIGHT split (compact tiers, one unfocused
panel) and a narrow width. Inspect them, and fix any spacing, color or alignment defect.

### 4.9 `card-blocks-cutover` — Remove the flag, add goldens, inspect live, and bench

1. **Bench first.** With the flag still present, run the j/k bench (including the
   `block-paged-view` fixtures) flag-off vs flag-on in SINGLE and LEFT_RIGHT. p95 must
   be < 16 ms. A regression gets fixed, and an environment problem gets worked around or
   explained. Record both numbers in the phase close note.
2. **Remove the flag.**
   - Delete every Off branch and make the On branch unconditional.
   - Delete `decks/flag.py` and the registry entry.
   - Collapse on/off tests to single-state.
   - Close the flag bead in the same change.
   - `sase flag show card_blocks` and `tools/check_feature_flags` must be clean.
3. **PNG goldens.** Build a deterministic session fixture with fixed times and plan,
   gate, monitor and code shells, some running. Add goldens in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py` (or a sibling file)
   for:
   - a block-paged Reply on the newest block (120×40)
   - the same after `[`
   - a block-spread chat-log landing
   - a LEFT_RIGHT split with windowed/compact rails, independent cursors and an
     unfocused dim panel
   - a spread deck with sticky-Reply landing and no rail
   - the not-following arrival dot
   - a narrow micro tier

   Generate them with targeted `just fix-tui-screenshots -- <selectors>`, and open and
   inspect **every** PNG. Re-baseline existing goldens only where the flag removal
   legitimately changes them, and inspect those too.

4. **Live inspection.** Use `sase screenshot` on real sessions: landing, `[`, the split,
   a streaming new shell, and j/k across several sessions (the triage loop). Fix
   defects.
5. `sase tool run check` must be green. Do not run `just check-full` unless explicitly
   instructed.

### 4.10 `card-blocks-docs` — User docs for card blocks

Describe only shipped behavior.

- **`docs/ace.md`**
  - The hierarchy (deck → card → block).
  - Which cards have blocks.
  - Spread vs paged blocks and the "shown alone" rule.
  - Newest-block landing, and the **triage loop** tip: with Reply sticky, j/k shows each
    session's newest shell in one keypress.
  - Following and arrival dots.
  - The rail and its tiers, and clicking.
  - `[`/`]` in the Agents Navigation table.
  - A note that `Ctrl+Shift+J/K` is not a default, because tmux delivers it as
    `Ctrl+J/K`, plus how to bind it where it works
    (`ace.keymaps.app.prev_card_block: "ctrl+shift+j"`,
    `next_card_block: "ctrl+shift+k"`).
- **`docs/configuration.md`**
  - The `block_spread_max_screens` row next to `spread_max_screens`.
  - The `[`/`]` pairs in the shared-key allowlist table.
- **Glossary memory.** This plan does not authorize memory edits, so **do not edit
  `sase/memory/`**. Instead, record a `PROPOSED FOLLOW-UP:` note on this phase's bead
  with ready-to-apply text for:
  - a new `glossary` strand, **Agent Data Card Block** (aka card block, card blocks)
  - the matching update to the `glossary:agent-data-card` strand

## 5. Verification

- Per phase:
  - targeted pytest for the touched areas
  - `sase tool run check` before finishing (run `just fix` first)
  - UI phases: flag-on `sase screenshot` inspection
- Goldens change only in `session-reply-blocks` (D8), `block-keys` (help rows, if any)
  and `card-blocks-cutover`. These are ordered by dependencies, so binary goldens never
  conflict. Always inspect the retained visual report; generation is not approval.
- The epic is done when:
  - all block states behave per §3.4 in pilot tests
  - the goldens are committed and inspected
  - live captures are inspected
  - the j/k bench p95 is < 16 ms with flag-off/on numbers recorded
  - the flag bead is closed
  - `sase tool run check` is green
