---
tier: epic
title: 'Deck scroll scope: tabs mean pages, a view chip, and a layout lock'
goal: 'From the deck panel''s chrome alone, a user can tell whether the focused deck
  is one scrollable document or split into pages, which way the rest of it lies, and
  whether its layout is locked; the auto/fixed words give way to a lock icon with
  a real lock/unlock key.

  '
decisions:
  lock_key:
    ask: Which default key should toggle the deck view lock (lock what auto chose
      / return to auto)?
    choices:
      shift_b: Free on every tab today and one Shift press away, but not mnemonic
      ctrl_shift_l: Mnemonic, but terminals without kitty key reporting send Ctrl+L
        (dismiss toasts)
      p_wraps: No new key; P gains automatic as a fourth stop, so unlocking can take
        three presses
    default: shift_b
    why: Every mnemonic letter is taken on the Agents tab; B is the only unbound letter
    answer: shift_b
  glossary_tabs_mean_pages:
    ask: Should the docs phase add the "tabs mean pages" rule to the Agent Data Card
      glossary strand?
    memory:
    - glossary:agent-data-card
    requested: Review the deck_scroll_scope_tabs_and_layout_lock.md file in the research
      sidecar repo for context and inspiration before planning. I agree with all of
      the requirements recommended in that research file
    default: true
    answer: true
phases:
- id: tabs-chip
  title: Tabs mean pages, the view chip, and the lock
  depends_on: []
  size: medium
  description: 'tabs-chip: card tabs only for paged multi-card decks, a solid view
    chip with a lock replacing auto/fixed, a spread you-are-here title with structural
    ▴/▾ cues, the jump/cards footer label, and the "locked" vocabulary.'
- id: lock-key
  title: Lock-toggle key and lock copy
  depends_on:
  - tabs-chip
  size: medium
  description: 'lock-key: an Agents-only action that locks the layout auto chose or
    returns to automatic, wired through keymaps, availability, palette, footer, help,
    the first-lock toast, and generated keymap docs.'
- id: viewport-cues
  title: Viewport-precise spread cues and the paged continuation cue
  depends_on:
  - lock-key
  size: medium
  description: 'viewport-cues: one refresh-on-change scope key per panel drives in-view-aware
    spread cues, the "all in view" state, and a bottom-border +N cue for paged decks
    that turns bold at the bottom.'
- id: docs-acceptance
  title: Docs, glossary, and acceptance review
  depends_on:
  - viewport-cues
  size: small
  description: 'docs-acceptance: rewrite the Deck Views reference and the Ctrl+J/K
    paragraph, apply the glossary decision, and verify the five acceptance criteria
    with live screenshots and goldens.'
proposed_by: bbugyi200.athena.research.52.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 17:07:58
status: done
bead_id: sase-1n7
---

- **PROMPT:** [prompts/202610/deck_scroll_scope.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/deck_scroll_scope.md)
- **BEAD:** [sase-1n7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n7/README.md)

# Plan: Deck scroll scope — tabs mean pages, a view chip, and a layout lock

## Context

Today a deck panel cannot answer "can I scroll down to see the whole deck?". The loudest
thing in every deck title is a tab strip — a reverse-video active tab plus `N/M`, or
`‹ 2/2 › Reply` when narrow — and it says "paged" in every mode. The only thing that
says "spread" is one muted word in the `{layout} · {auto|fixed}` badge. Narrow panels
show undecodable letters (`MAIN C·F 2/2`). A spread deck that lands on Reply hides
Context _above_ while the title still looks like tabs.

This epic implements the recommended solution of the research report
`research:202610/deck_scroll_scope_tabs_and_layout_lock/deck_scroll_scope_tabs_and_layout_lock.md`
(all of its requirements A1–A9 are accepted). It supersedes two parts of
`plan:202609/deck_views.md`: D5's "quiet by default" badge styling and `auto`/`fixed`
words, and D2's "only the palette returns to automatic". Everything else from that plan
(the `P` widening order, D3 equivalence, D4 persistence, the D5 label rule for vacuous
requests, partial-paint holding) stays.

All of this is presentation-only Textual chrome under `src/sase/ace/tui/`; nothing
crosses the `sase_core` boundary. No feature flag: each phase lands a complete, coherent
improvement, and no phase exposes a half-built control.

## The rule (what users learn)

> **Tabs mean pages.** A deck panel shows card tabs only when other cards are on other
> pages. With no tabs, the whole deck is one scrollable document, and `▴`/`▾` show which
> way the rest of it is. A `+N` cue at the bottom of a paged deck says what is on other
> pages. `🔒` in the view chip means auto is off for this panel's layout.

There are three nested questions, and the design answers each with chrome at the edge
where the reader needs it:

1. **Deck level** — are the other cards in this scroller? → tabs vs. no tabs (title).
2. **Direction** — where are the other cards? → `▴`/`▾` cues (spread title) and the `+N`
   continuation cue (paged bottom border).
3. **Policy** — is this layout held? → the view chip, with `🔒` when locked.

## Design spec

This section is the single source of truth for copy, styling, and ladders. Phases
reference it; implement it exactly unless a live screenshot proves a detail wrong (then
record why in the phase's commit message).

### Paged vs. spread derivation (every deck)

A deck title is **paged** when its _applied_ deck render mode is paged **and** it has
two or more cards. Derive it:

- **Main and Files** from the same held chrome view that drives the chip
  (`DeckPanelChromeMixin._chrome_view`), using its _effective_ layout
  (`effective is not DeckView.SPREAD`; add an `effective` field to `ResolvedView` if
  needed, since `shown` follows the D5 vacuous-request rule). This keeps the chip and
  the tabs in agreement during partial `j`/`k` paints. When no view is held yet, fall
  back to `is_spread(deck)`.
- **FINAL and RECORD** from `is_spread(deck)` — never from the deck type, because both
  the count-based main path and the measured redecision path can apply either mode.
- **Tools** is always paged.
- **One card** (any deck) is never paged in the tab sense.

### Title grammar

`[X]` below marks the solid chip (the brackets are not drawn); `«X»` marks today's
active-tab pill. Widths are cells measured with `rich.cells.cell_len`.

**Spread, two or more cards** — no `┃`, no tabs, no `N/M`, no `‹ ›`. A plain
you-are-here card name, then the cards off-screen in each direction:

```text
◆ MAIN  [SPREAD]  Reply · ▴ Context          landed on Reply; Context above
◆ MAIN  [SPREAD]  Context · ▾ Reply          at the top
◆ MAIN  [SPREAD 🔒]  Reply · ▴ Context       locked
▤ FILES  [SPREAD]  b.py · ▴ a.py · ▾ 3 files  several cards in one direction: a count
◆ MAIN  [SPREAD]  both cards in view          (viewport-cues) no scroll range
▤ FILES  [SPREAD]  all 4 files in view        (viewport-cues) no scroll range
◆ FINAL  Overview · ▾ 3 cards                 FINAL/RECORD: same grammar, no chip
```

- Cue naming: a direction with exactly one card names it (`▴ Context`); a direction with
  several shows a count with the deck's card noun (`▾ 3 files`, `▾ 2 cards`). Nouns:
  `file`/`files` for Files, `card`/`cards` for every other deck (do not reuse
  `DeckSpec.count_noun`, which counts calls/finalizers for Tools/FINAL).
- All-in-view copy: `both cards in view` / `both files in view` for two, otherwise
  `all N cards in view` / `all N files in view`; compact form `all in view`.
- Spread ladder (first rung that fits wins): (1) chip + here + named cue; (2) chip +
  named cue; (3) chip + counted cue (`▴ 1 · ▾ 3`); (4) tiny chip + arrows (`▴▾`); (5)
  micro: `NAME` + tiny chip + arrows; (6) `NAME` + arrows. The all-in-view phrase
  degrades `both cards in view` → `all in view` → nothing. The spread chip's long and
  short forms are both `SPREAD`, so there is no separate short rung.

**Paged** — the existing tab rungs, with the chip in place of the badge. Tabs now always
mean pages:

```text
◆ MAIN  [PAGE CARDS]  ┃ Context │ «Reply»  2/2
◆ MAIN  [PAGE BLOCKS 🔒]  ┃ Context │ «Reply»  2/2
◆ MAIN  [CARDS]  ┃ ‹ 2/2 › Reply                compact
MAIN [C] 2/2   ·   MAIN [B🔒] 2/2                 micro
▤ FILES  [PAGE CARDS 🔒]  spreading…  ┃ ‹ 2/5 › app.py
▤ FILES  [PAGE CARDS 🔒]  spread unavailable  ┃ ‹ 1/3 › shot.png
```

Paged ladder (unchanged shape): full tabs + long chip, full tabs + short chip, compact +
long, compact + short, compact + tiny, micro + tiny, micro alone. Files and FINAL with
more than four tabs keep skipping the full rungs. Dropping `· auto`/`· fixed` pays for
the chip padding.

**One card** (any deck): `◆ MAIN  Summary` — no `┃`, no pill, no chip (a one-card deck
has a single distinct layout, so no choice exists).

**Other decks:** FINAL and RECORD follow the tab rule with no chip and no lock (they
have no policy); a paged FINAL keeps its status-glyph tabs. Tools is unchanged apart
from the one-card rule. ZOOM still leads every rung and wins as the rarer state.

### The view chip

- Text: long `SPREAD` / `PAGE CARDS` / `PAGE BLOCKS`; short `SPREAD` / `CARDS` /
  `BLOCKS`; tiny `S` / `C` / `B`. Padded one cell each side (`PAGE CARDS`), like the
  footer's status pills.
- Lock: when the policy is not `DeckView.AUTO`, `🔒` sits inside the chip before the
  trailing pad (`SPREAD 🔒`, tiny `S🔒`). Auto shows nothing. Never show `🔓`.
- Focused panel: `bold #1a1a1a on #D0D0D0` (a `VIEW_CHIP_STYLE` constant next to
  `ZOOM_CHIP_STYLE`). Unfocused panel: `dim` text, no fill, same padding, so the title
  width never depends on focus. No state colors, no animation. The chip stays visually
  distinct from both the gold ZOOM chip and the accent-colored active-tab pill.
- Pending/blocked words sit right after the chip, outside it: `spreading…` (muted) and
  `spread unavailable` (`#D7AF5F`); short `spreading…` / `no spread`; tiny `…` / `!`.
  They describe a request, not the policy; the lock still shows because both states only
  arise from a locked spread.
- Visibility: Main and Files only, when a held view exists and either a layout choice
  exists (two or more D3-distinct layouts — compute it in `resolve_view` and expose it
  on `ResolvedView`) or the status is pending/blocked.
- Emoji check: confirm `🔒` keeps the border aligned in a live `sase screenshot` and in
  the visual renderer. If it does not, use the word `LOCK` inside the chip instead.

### Styling of the title text

Here-name: `bold {accent}` on the focused panel. Separators `·`: `#444444`. Cue arrows:
`bold {accent}`; cue names and counts: `#888888`. Unfocused: everything `dim`. Tune by
screenshot within these roles.

### Bottom-border continuation cue (paged decks only)

A leading status segment in the subtitle, before the Files line status:

```text
└─ +1 card · Ctrl+J   main 2 · files 0 · tools · final 0 · record ─┘
└─ +3 blocks · (/) · +1 card · Ctrl+J   main 2 · … ─┘                 page blocks
└─ +4 files · Ctrl+J   1-400 of 2000 lines · E editor ─┘               Files paged
```

- Counts only what is not on this page: `cards - 1`; in page blocks also
  `active card blocks - 1`. Keys come from the keymap registry (`next_deck_card`;
  `prev_card_block`/`next_card_block`), resolved like `_deck_switch_hint()`; an unbound
  key drops its segment.
- Spread gets no bottom cue; its absence is part of the signal.
- Resting style: counts `#888888`, key in `{accent}` (like the zoom restore hint). When
  the paged scroller is at its bottom (`scroll_y >= max_scroll_y`, which includes "no
  scroll range"), the whole cue turns `bold {accent}` on the focused panel, so at the
  "am I done?" moment the panel says "not yet". Unfocused: `dim`.
- Width: the switcher counts drop first, then the switcher (existing behavior), then the
  cue's key segments; the counts are the last thing to truncate. A zoomed subtitle keeps
  the cue after the restore hint.

### Copy

- Footer: `Ctrl+J/Ctrl+K jump` when the focused panel's deck is spread with two or more
  cards, `Ctrl+J/Ctrl+K cards` when paged. `P view` is unchanged. The lock key shows as
  `<key> lock` (auto) or `<key> unlock` (locked).
- Palette: `Deck view: automatic (unlock)`, `Deck view: spread (locked)`,
  `Deck view: page cards (locked)`, `Deck view: page blocks (locked)`; keep the old
  `(fixed)` labels and the word `fixed` as search aliases.
- First-lock toast (a `P` press from auto): `Main view locked 🔒 · <lock key> unlocks`
  with the live key display (`B` under the default); with the key unbound,
  `Main view locked 🔒 · palette "Deck view: automatic" unlocks`. Keep it at most 70
  cells.
- Help rows: `P` — `Lock a wider deck view (<lock key> unlocks)`; lock key —
  `Lock / unlock the deck view as shown`; `Ctrl+J / Ctrl+K` —
  `Next / prev card (jumps in spread, pages when paged)`; plus one explanatory row:
  `Tabs in a deck title mean one card per page; no tabs means one document, and ▴/▾ show where the rest is`.
- "fixed" becomes "locked" everywhere users read it.

### Performance contract (all phases)

Every cue rides refreshes that already exist or that fire only on a state change:
scroll-spy changes, document applies, section-layout publication, resize, and the paged
at-bottom flip. No filesystem reads, no card measurement, and no `refresh_chrome()` per
scroll tick: compute a cheap hashable key over cached anchors (O(cards)) and refresh
only when it changes. Read `tui_perf.md` before touching scroll watchers.

## Phase tabs-chip: Tabs mean pages, the view chip, and the lock

Delivers the structural fix as a complete, coherent change.

1. **Chip variants** — rewrite `widgets/decks/view_badge.py` (rename it `view_chip.py`
   if that reads better) to return the long/short/tiny chip `Text`s and the separate
   pending/blocked word variants per the spec. Delete the `auto`/`fixed` words and
   `_status_style`'s quiet/loud logic.
2. **Policy facts** — in `view_policy.py`, have `resolve_view` expose the effective
   layout and whether a choice exists (two or more distinct layouts) on `ResolvedView`.
   Keep the D5 label rule for `shown`.
3. **Title rungs** — `titles.py` `deck_title(...)` gains `paged: bool` (default `True`
   for non-chrome callers) and a structural spread cue input (here-name plus the card
   titles before and after it). Implement the spread, paged, and one-card ladders from
   the spec. Put the new spread/chip rung builders in a new module (for example
   `widgets/decks/title_scope.py`) so `titles.py` stays well under the 700-line `toobig`
   tier.
4. **Chrome wiring** — `panel_chrome.py` `refresh_chrome()` computes `paged` per the
   derivation rules, builds the structural cue (above = cards before the active card,
   below = cards after it; the active card is `_active_tab_index()`, already
   pending-aware for Main), and passes chip visibility. The existing scroll-spy hooks
   (`_on_main_scroll_y`, `_on_files_scroll_y`, `_on_document_scroll_y`) already refresh
   chrome exactly when the active card changes, which is when the structural cue
   changes; add no new refresh points.
5. **Footer label** — add a cached panel predicate (for example
   `DeckPanel.deck_cards_spread`) updated in `refresh_chrome()` that pokes the footer
   only when it flips, mirroring `_sync_view_cycle_available`. Use it in both footer
   builders in `widgets/_keybinding_bindings_agents.py` (via
   `actions/agents/_display_detail_footer.py`) for `jump` vs `cards`.
6. **Locked vocabulary** — palette labels and aliases (`commands/catalog.py`
   `_iter_deck_view_commands`), the first-lock toast in
   `actions/agents/_panel_detail.py` (palette form, since no key exists yet), and the
   `P` help row (`Lock a wider deck view (palette unlocks)`) in
   `modals/help_modal/agents_main_sections.py`.
7. **Tests** — `test_deck_titles.py`: spread rungs never contain `┃`, `│`, `N/M`, or
   `‹`; paged rungs keep tabs; one-card heading; chip text per layout and tier; lock
   only when locked; no `auto`/`fixed` text anywhere; pending/blocked words; ladder
   selection at several widths; ZOOM still leads. `test_deck_chrome.py`: the paged
   derivation for every deck, the held view during partial Main paints, chip visibility.
   `test_deck_view_policy.py`: the new `ResolvedView` facts. A pilot test: an auto flip
   from spread to paged shows tabs, `j`/`k` partial paints keep the title stable,
   `Ctrl+J` in spread moves the here-name and cue, and the footer label flips. Delete
   tests of removed symbols (Symvision will flag dead code; see `symvision.md`).
8. **Goldens** — re-baseline the affected `agents_deck_view_*` and `agents_decks_*`
   goldens and add: a split with spread and paged panels side by side, a narrow split at
   the micro tier, a one-card heading, and a locked chip. Follow the `tui_screenshot`
   inspection contract (run `just fix-tui-screenshots` with selectors via
   `/sase_monitor`, inspect every changed golden).

Done when `sase tool run check` passes, goldens are inspected and current, and a live
`sase screenshot` of a spread and a paged Main panel shows the grammar above.

## Phase lock-key: Lock-toggle key and lock copy

> [!decision] lock_key = shift_b

Add an Agents-only app action `toggle_deck_view_lock`, default key `B`.

> [!decision] lock_key = ctrl_shift_l

Add the same action with default key `ctrl+shift+l`. Note in the config comment that
terminals without kitty keyboard reporting deliver `Ctrl+L` instead.

> [!decision] lock_key = p_wraps

Add no action. Instead `P` gains automatic as a fourth stop: from locked `spread` (the
widest layout), `P` returns to automatic; from automatic it locks the next wider layout
as today. In `cycle_focused_deck_view()`, when `P` would wrap from the widest distinct
layout back to the narrowest, set automatic instead (D3 equivalence is unchanged).
Update the help row (`Cycle deck view; past spread returns to auto`), the toast
(`… · P past spread unlocks`), and the footer (`P view` unchanged). Skip the
action-wiring steps below.

For a new key:

1. **Semantics (AE-L)** — from automatic, set the focused panel's policy to its current
   effective layout (`panel.effective_layout(deck)`), so the picture does not change and
   only `🔒` appears; this also stops a streaming spread deck from flipping to paged at
   1.65 screens. From locked, set automatic (the auto deciders re-run). Add
   `AgentDetail.toggle_focused_deck_view_lock()` beside `cycle_focused_deck_view()` in
   `widgets/_agent_detail_deck_view.py`, applying through
   `area.set_panel_view(..., user_initiated=True)`. `P` is unchanged (widen and lock,
   per D2).
2. **Wiring** — follow every `cycle_deck_view` touchpoint: `default_config.yml` (app
   keymaps, with a comment), `keymaps/app_keymaps.py`, `keymaps/metadata.py`,
   `bindings.py`, `commands/_app_metadata_nav.py` (label `Toggle deck view lock`,
   aliases `lock`, `unlock`, `hold view`, `fixed`), `_app_action_availability_agents.py`
   and `commands/_availability_agents.py` (available on Main/Files when the deck is
   non-empty, a Main document is not partial, and either the view cycle is available or
   the policy is locked), and `actions/agents/_deck_search_host.py` if it gates deck
   keys.
3. **Footer** — show `<key> lock` / `<key> unlock` next to `P view` in both footer
   builders, gated like `P view` plus the locked case; refresh via
   `_refresh_agent_footer_bindings_only` after the action, as `action_cycle_deck_view`
   does.
4. **Copy** — first-lock toast and help rows exactly as in the spec's Copy section; the
   toast stays silent for the lock key itself (the chip changes in place).
5. **Generated docs** — run `just fix` to regenerate `docs/reference/tui/keymaps.md`.
6. **Tests** — extend `test_deck_view_keys.py`: toggling from auto keeps the effective
   layout and sets that policy; toggling from locked returns to auto; availability; the
   footer label; palette labels plus the old aliases; toast text with the key and with
   the key unbound; help rows. Run any keymap/metadata consistency tests.
7. **Goldens** — the footer change re-baselines Agents-tab goldens that show `P view`;
   inspect them all.

## Phase viewport-cues: Viewport-precise spread cues and the paged continuation cue

Serialized after `lock-key` only because both re-baseline the same Agents-tab PNG
goldens.

1. **Scope key** — per panel, a cheap hashable key for the shown deck: spread →
   `(here, above_ids, below_ids, all_in_view)`; paged → `(at_bottom,)`. Recompute it in
   the existing scroll watchers (`_on_main_scroll_y`, `_on_files_scroll_y` — which today
   returns early when paged — and `_on_document_scroll_y`; add the Tools scrollers if
   they are not watched), in a new hook on section-layout publication
   (`SectionViewMixin._publish_section_layout`, overridden in `CardDocumentView` and
   reachable from `FilesSpreadView`), and on resize. Call `refresh_chrome()` only when
   the key changes. Layout publication matters: `refresh_chrome()` after a document
   apply runs before Textual lays out, so scroll extents read there can be stale, and
   streaming content growth never moves `scroll_y`.
2. **Pure cue functions** — a pure module (extend the tabs-chip title module or add
   `widgets/decks/scope_cues.py`) with: a spread cue from ordered cards, cached anchor
   rows (`card_anchor_rows(width=...)`), `scroll_y`, viewport height, and
   `max_scroll_y`; and the continuation cue `Text` builder. Spread rules: the here-card
   is the scroll-spy active card; every card before it is fully above the viewport;
   cards after it are listed only when not fully in view
   (`card end > scroll_y + viewport height`); `max_scroll_y == 0` means all in view.
   When anchors are not ready, fall back to the structural cue. If the section view
   exposes the trailing layout reserve added by `scroll_to_card()`, exclude it from the
   last card's end; otherwise accept the conservative `▾`.
3. **Spread title** — feed the precise cue and the all-in-view phrase into the spread
   ladder from the spec.
4. **Paged bottom cue** — build the continuation cue in `refresh_chrome()` and pass it
   to `deck_subtitle()` ahead of the Files line status, with the spec's width tiers and
   at-bottom emphasis. Block counts come from the active card's block mode
   (`block_mode_for_active_card()`), cards from the deck's tab count.
5. **Tests** — pure: fully visible later cards are dropped; all-in-view;
   anchors-not-ready fallback; reserve handling; continuation strings, tiers, and
   emphasis styles. Pilot: a spread Main landed on Reply shows `▴ Context`; streaming
   growth past the viewport replaces `both cards in view` with a cue without any scroll
   event; scrolling within one cue state does not call `refresh_chrome()` (count calls);
   the paged at-bottom flip refreshes once per flip. Optionally confirm j/k latency with
   `pytest -s -m slow tests/ace/tui/bench_tui_jk.py`.
6. **Goldens** — re-baseline spread and paged deck goldens; add an all-in-view spread
   and a paged card scrolled to its bottom (bold cue).

## Phase docs-acceptance: Docs, glossary, and acceptance review

1. **Docs** — read the `docs_writing` memory note first. Rewrite
   `docs/reference/tui/agent-statuses.md` § Deck Views (chip, lock, the tab rule, spread
   cues, the bottom cue, the lock key, the palette labels) and its Tools/FINAL/RECORD
   badge mentions; update the `Ctrl+J`/`Ctrl+K` paragraph in
   `docs/reference/tui/agents-main-deck.md` (jumps in spread, pages when paged; tabs
   mean pages). Confirm the generated keymap reference is current.
2. **Glossary**

   > [!decision] glossary_tabs_mean_pages

   Through `/sase_memory_write`, add to the Agent Data Card strand
   (`glossary:agent-data-card`): "A deck panel's title shows card tabs only when the
   deck is paged and has two or more cards — tabs mean pages; with no tabs the whole
   deck is one scrollable document, and the title's ▴/▾ cues name the cards above and
   below." Then run `sase memory init`.

   > [!decision] glossary_tabs_mean_pages = no

   Record the skipped strand change as a `PROPOSED FOLLOW-UP:` note on this phase bead.

3. **Acceptance** — with `sase screenshot` (spread, paged, split, narrow, locked) and
   the goldens, confirm that a reviewer who has never used decks can, from a screenshot
   alone: (1) say "Context is above" for a spread Main landed on Reply; (2) say "I have
   to change pages" for a paged Main; (3) _not_ conclude Context is below in a
   page-cards deck with a scrollbar; (4) see no policy mark on an auto panel and `🔒`
   after `P` or the lock key; (5) see no tabs and no chip on a one-card deck. Fix any
   miss in place (small chrome tweaks) or file it through `/sase_new_task`.

## Non-goals

- No change to auto thresholds (1.5 screens + 10% hysteresis), spread landing (Reply is
  still top-aligned over a trailing reserve), the `P` widening order, or block paging.
- No state-colored chips, no bare padlock outside the chip, no `🔓`, no restyled tabs in
  spread, no new docked badge row, and no hiding of `Ctrl+J`/`Ctrl+K` in spread.
- Deferred, each with its own reopen condition: clickable paged tabs; landing a fitting
  spread at the top; scope-worded chips (`Scroll: deck/card/block`) if confusion
  persists; an in-body end-of-card marker; a modeless single document.
