---
tier: epic
title: Agents tab agent data decks and cards
goal: 'The Agents tab replaces its metadata panel and its Files and LLM Calls panels
  with one deck-panel type. It shows one or two panels, stacked top and bottom or side
  by side, and each panel shows an agent data deck (Main, Files or Tools) made of agent
  data cards. A deck renders all of its cards on one page when they fit a configurable
  threshold and one card per page otherwise. Keys cycle cards (Ctrl+J/K) and decks
  (Ctrl+N/P), split and unsplit panels (\ and |), move focus (Ctrl+F), collapse the node
  panel (Ctrl+S) and zoom a deck in place (Z). The p view picker, the Z zoom modal and
  the legacy panel modes are deleted, and glossary strands describe the new vocabulary.

  '
phases:
  - id: llm-calls-subject-guard
    title: LLM Calls stale-worker fix and split-key display
    depends_on: []
    size: small
    description:
      "llm-calls-subject-guard: stop LLM Calls worker results from painting onto a
      different agent, and teach key validation and display about backslash and
      vertical_line so split keys can be overridden and shown."
  - id: main-card-partition
    title: Card-partitioned Main documents
    depends_on: []
    size: large
    description:
      "main-card-partition: every metadata builder wraps its output in
      Context/Reply/Output/Summary card parts, error tracebacks move to the top of the
      Reply card, and tree walkers learn the card wrapper. With the flag off the panel
      renders as before, except the traceback fix."
  - id: deck-panel-core
    title: Deck panel core behind the agent_decks beta flag
    depends_on:
      - llm-calls-subject-guard
      - main-card-partition
    size: large
    description:
      "deck-panel-core: create the agent_decks beta flag and the widgets/decks package:
      DeckArea with two pre-composed DeckPanels, a hidden Main source feeding
      MainDeckViews, per-panel Files and Tools views, tab-strip titles, empty-state
      cards, lazy loading and no-I/O availability probes. Paged rendering only."
  - id: deck-navigation-keys
    title: Card and deck cycling keys
    depends_on:
      - deck-panel-core
    size: medium
    description:
      "deck-navigation-keys: Ctrl+J/K cycle cards and Ctrl+N/P cycle decks in the
      focused panel, with sticky preferred cards. Ctrl+D/U, g/G and the bottom pin
      target the focused deck panel. Includes full keymap registration and the first
      flag-on PNG goldens."
  - id: deck-splits-focus
    title: Split layouts, focus and split ratio
    depends_on:
      - deck-navigation-keys
    size: large
    description:
      'deck-splits-focus: \ and | toggle top-bottom and left-right splits through a pure
      layout state machine. Ctrl+F moves logical focus and {/} resize the focused panel.
      Duplicate decks fan out to both panels, with no remounts on layout change.'
  - id: deck-action-retarget
    title: Retarget detail actions to the focused deck panel
    depends_on:
      - deck-splits-focus
    size: large
    description:
      "deck-action-retarget: inventory every consumer of the legacy panel ids and
      visibility helpers, then give each one a deck-mode path. Covers folds, hints,
      search, E, LLM detail levels, clipboard, footer, palette context and auto-refresh,
      and makes the SLOW TOOL CALLS hint accurate."
  - id: node-panel-collapse-zoom
    title: Node panel collapse and in-place zoom
    depends_on:
      - deck-splits-focus
    size: medium
    description:
      "node-panel-collapse-zoom: Ctrl+S hides the node panel without unmounting it and
      shows a slim node spine plus an info-row chip. With decks on, Z becomes an
      in-place zoom of the focused deck panel that a second Z restores."
  - id: deck-spread-mode
    title: Spread versus paged rendering
    depends_on:
      - deck-action-retarget
    size: large
    description:
      "deck-spread-mode: adds the ace.agent_decks.spread_max_screens config and a pure
      decide_render_mode with hysteresis. Measures cheap lower bounds first, renders
      Main and Files spread with titled separators and card anchors, and keeps the
      reading position stable across transitions."
  - id: deck-state-persistence
    title: Persist the deck layout across restarts
    depends_on:
      - node-panel-collapse-zoom
    size: small
    description:
      "deck-state-persistence: persist the layout, ratio, focus, node-panel collapse and
      each panel's deck and preferred card to ~/.sase/ace_agents_deck_state.json.
      Loading fails open and saves are coalesced off the event loop."
  - id: deck-cutover
    title: Cut over to decks and delete the legacy UI
    depends_on:
      - deck-spread-mode
      - deck-state-persistence
    size: large
    description:
      "deck-cutover: delete the flag's Off branch and the flag, the p picker, the zoom
      modal, the panel enums and the legacy panel ids and CSS. Retire or rename the
      affected action ids, migrate or delete tests, and regenerate every affected PNG
      golden."
  - id: deck-docs-glossary
    title: Docs, glossary strands and key-change notice
    depends_on:
      - deck-cutover
    size: medium
    description:
      "deck-docs-glossary: rewrite the Agents detail docs around decks and cards, sweep
      stale zoom, picker and section-stop references, add a one-time post-update key
      notice, and add and edit the listed glossary strands, then run sase memory init."
proposed_by: bbugyi200.athena.0qd
create_time: 2026-09-23 19:16:44
status: wip
---

- **PROMPT:**
  [prompts/202609/agents_tab_decks_and_cards.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_tab_decks_and_cards.md)

# Plan: Agent Data Decks and Cards for the Agents Tab

## 1. Context

Today the Agents tab detail area (`src/sase/ace/tui/widgets/agent_detail.py`) is one
"metadata" panel (`AgentPromptPanel` inside `#agent-prompt-scroll`) plus at most one
secondary panel: Files (`AgentFilePanel`) or LLM Calls (`AgentLLMCallsPanel`). Two enums
(`DetailPanelMode`, `DetailLayoutMode` in `widgets/_agent_detail_panels.py`), the `p`
view picker (`modals/agent_view_modal.py`, `actions/agents/_agent_view_picker.py`) and
the `Z` zoom modal (`modals/zoom_panel_*.py`) choose what is visible.

This epic replaces all of that with one model: **deck panels showing agent data decks
made of agent data cards.**

Background research, read with `sase artifact read`:
`research:202609/agents_tab_decks_and_cards/agents_tab_decks_and_cards.md`. That report
inventories the current code (§2) and measures content sizes (§2.6). **This plan wins
wherever the two disagree.** The user settled these points:

- Cards cycle on **Ctrl+J/Ctrl+K**, replacing the old metadata section stops, which are
  retired.
- Decks cycle on **Ctrl+N/Ctrl+P**.
- A paged deck keeps a **sticky Reply** (the chosen card) across j/k.

All paths below are relative to `src/sase/ace/tui/` unless they start with `src/`,
`tests/`, `docs/` or `sase/`.

Rust core boundary: deck and card composition, render modes, layout, focus, keys and
persistence are presentation-only Textual state. **No `sase-core` change is needed.**

## 2. Vocabulary

| Term                         | Meaning                                                                                                                                                                                                                             |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Agent data deck** ("deck") | A named, ordered set of cards about the current selection. Built-ins in cycle order: **Main**, **Files**, **Tools**.                                                                                                                |
| **Agent data card** ("card") | One titled unit of detail in a deck. It has a stable id (`context`, `reply`, `summary`, `llm-calls`) or a per-node id (a file page slot).                                                                                           |
| **Deck panel**               | One detail region showing one deck. The Agents tab shows one or two.                                                                                                                                                                |
| **Deck layout**              | `SINGLE`, `TOP_BOTTOM` (a "horizontal split", `\`) or `LEFT_RIGHT` (a "vertical split", `\|`). This follows vim's `:split`/`:vsplit` naming. Code uses `TOP_BOTTOM`/`LEFT_RIGHT` and never the ambiguous horizontal/vertical words. |
| **Spread / paged**           | The two ways a panel renders a multi-card deck. **Spread:** every card on one scrollable page, with titled separators. **Paged:** only the active card.                                                                             |
| **Active card**              | Paged: the card being shown. Spread: the card whose header is at or above the viewport top.                                                                                                                                         |
| **Preferred card**           | The card the user last chose explicitly with Ctrl+J/K in a panel. It is kept across selections.                                                                                                                                     |
| **Node panel**               | The left column of sase node rows (`#agent-list-container`). It can be collapsed.                                                                                                                                                   |
| **Shared chrome**            | `AgentHeaderPanel` (identity header, `d`) and `AgentJumpPanel` (numbered roster, `.`). They span all deck panels and belong to no deck.                                                                                             |

## 3. UX specification

### 3.1 Main deck cards by node kind

| Selection                                                  | Cards (`id`: title)                                                                       |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Agent node, agent shell, queued agent                      | `context`: Context · `reply`: Reply                                                       |
| Family container                                           | `context`: Context · `reply`: Reply (`AGENT REPLY · N` with per-shell phases)             |
| Attempt-pinned view (`D`)                                  | `context`: Context (banner, `ATTEMPT ERROR`, prompt) · `reply`: Reply (`ATTEMPT N REPLY`) |
| Proc shell, monitor, gate, bash/python step, parallel step | `context`: Context · `reply`: **Output** (log tail / `OUTPUT` / `STEP OUTPUT`)            |
| Top-level workflow row                                     | `context`: Context only                                                                   |
| Clan, tribe summary (whole-panel focus)                    | `summary`: Summary only (a triage digest meant to be read whole)                          |
| No selection                                               | none, so the Main deck shows its empty state                                              |

- **Context** is everything before the reply boundary that is not detached into shared
  chrome. That includes the metadata fields, `❖ QUEUE`, `MEMBERS`, `OUTPUT VARIABLES`,
  `WORKFLOW VARIABLES`, `SASE CONTEXT`, `SLOW TOOL CALLS`, the short `ERROR` summary,
  `AGENT XPROMPT` and `AGENT PROMPT`. Any leftover renderable that a builder did not tag
  also lands in Context, so nothing is lost.
- **Reply** is `AGENT REPLY` / `AGENT CHAT` with its phases, attempt dividers and the
  "Waiting for agent response…" placeholder. A failed agent's **error traceback moves to
  the top of the Reply card under a `TRACEBACK` section heading**. This fixes today's
  quirk where the traceback renders between the `AGENT PROMPT` heading and the prompt
  body (`widgets/prompt_panel/_agent_display_render.py:441-521`; the same pattern
  appears in the family, bash/python and parallel builders).
- **Output** reuses card id `reply`, so a sticky Reply also follows onto process nodes.
- **Context is the default card.**

### 3.2 Files and Tools decks

- **Files:** one card per existing file page, in today's `_desired_file_list` order:
  commit diffs, the live diff, linked-repo diffs, then `extra_files`.
  - The default card is today's default page.
  - Across selections, the active page is kept by path, as today.
  - Card titles come from `source_label_for_slot`.
  - Images and videos are **solo cards**: when one is present, the Files deck always
    renders paged.
- **Tools:** one card, `llm-calls` ("LLM Calls"). A future Tool Runs card will join it.
  With one card, spread and paged look the same.

### 3.3 Deck panels, layouts and transitions

- There are at most **two** deck panels in one orientation. The model stays a list of
  panels plus a layout, so a third panel stays possible later.
- **Transitions:**

```
SINGLE      --\-->  TOP_BOTTOM   new panel below; it takes focus
SINGLE      --|-->  LEFT_RIGHT   new panel to the right; it takes focus
TOP_BOTTOM  --\-->  SINGLE       closes the second (bottom) panel; the first panel keeps its deck, card and scroll
LEFT_RIGHT  --|-->  SINGLE       closes the second (right) panel
TOP_BOTTOM  --|-->  LEFT_RIGHT   rotates: both decks, focus, cards and scroll are kept
LEFT_RIGHT  --\-->  TOP_BOTTOM   rotates
```

- **Why unsplit keeps the first panel and not the focused one.** The split keys act as
  involutions: `|` then `|` always returns to where you started, even though the new
  panel took focus. To keep only the _focused_ panel, use `Z` (§3.6).
- **Deck for a new panel.** Walk the deck cycle forward from the current panel's deck.
  Pick the first deck that is **not currently shown** and **has content** for the
  selection (availability probe, §4.5).
  - If none qualifies, **duplicate the current deck** and open it on the card after the
    current panel's active card.
  - For an agent with no files and no tool calls, `|` therefore produces **Context |
    Reply**.
  - This refines the literal "next not-shown deck" rule. Without it the fallback could
    never trigger, since three decks always leave one unshown.
- **Deck cycling.** Ctrl+N/P cycle the focused panel through Main → Files → Tools
  (wrapping). Nothing is skipped: empty decks and decks shown in the other panel are
  both visited, so the order stays predictable. Duplicate decks are allowed, and each
  panel has its own active card and scroll position.
- **Ratio.** `}` grows the focused panel and `{` shrinks it, stepping the first panel's
  share through 30/50/70. Every new split starts at 50/50.
- **Stable geometry.** A panel whose deck is empty for the current selection shows an
  empty-state card. It never reflows or collapses, so the layout never jumps during j/k.
- **Focus is logical.** It is not Textual widget focus, so j/k keep driving the node
  selection. Clicking inside a panel focuses it. In `SINGLE` the only panel is always
  focused.
- **Layout changes never remount DeckPanels or their views.** They only change CSS
  classes, plus `move_child` where DOM order matters. Scroll positions and active cards
  survive every transition.

### 3.4 Keymap (Agents tab, decks enabled)

| Key             | Textual name                                 | Action id                               | Behavior                                                                                                                                                | Other tabs, unchanged                                 |
| --------------- | -------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Ctrl+J / Ctrl+K | `ctrl+j` / `ctrl+k`                          | `next_deck_card` / `prev_deck_card`     | Next/previous card in the focused panel (wraps). Paged: swaps the page. Spread: scrolls that card's header to the top. Sets the panel's preferred card. | Artifacts: `artifacts_load_more` / `artifacts_unload` |
| Ctrl+N / Ctrl+P | `ctrl+n` / `ctrl+p`                          | `next_deck` / `prev_deck`               | Focused panel to the next/previous deck                                                                                                                 | Services: chop-run stepping                           |
| `\`             | `backslash`                                  | `toggle_deck_split_below`               | §3.3                                                                                                                                                    | —                                                     |
| `\|`            | `vertical_line`                              | `toggle_deck_split_right`               | §3.3                                                                                                                                                    | —                                                     |
| Ctrl+F          | `ctrl+f`                                     | `toggle_deck_focus`                     | Move focus to the other panel. Split layouts only.                                                                                                      | Services/Artifacts: `scroll_prompt_down`              |
| `}` / `{`       | `right_curly_bracket` / `left_curly_bracket` | `grow_deck_panel` / `shrink_deck_panel` | Split ratio (§3.3). Split layouts only.                                                                                                                 | Artifacts: split cycling                              |
| Ctrl+S          | `ctrl+s`                                     | `toggle_node_panel`                     | Collapse/expand the node panel (§3.5)                                                                                                                   | —                                                     |
| `Z`             | `Z`                                          | `zoom_panel` (id kept, new behavior)    | Zoom the focused deck panel in place; `Z` again restores (§3.6)                                                                                         | Artifacts Files `Z`                                   |
| `p`             | —                                            | `choose_agent_view` **retired**         | —                                                                                                                                                       | Artifacts `p`                                         |

- **Ctrl+B** has no Agents behavior while decks are on. Ctrl+D/U already half-page
  scroll the focused panel.
- **Retargeted keys:**
  - Ctrl+D/U, `g`/`G` and the bottom pin act on the focused panel.
  - Folds, `v` hints, `,/` search, `E`, `l`/`h`/`L`/`H` follow §6.
  - `d`, `.`, `D` and `V` are unchanged.
- **Every new action needs:**
  - an `AppKeymaps` field (`keymaps/app_keymaps.py`)
  - a default in `src/sase/default_config.yml` (the core gotcha)
  - a `_BINDING_META` row (`keymaps/metadata.py`)
  - the fallback `bindings.py` row
  - a `check_app_action` gate (`_app_action_availability.py`): Agents tab, decks
    enabled, `_prompt_input_owns_keys` respected
  - command-palette metadata and availability (`commands/_app_metadata_*.py`,
    `commands/_availability_agents.py`)
  - a help-modal row (`modals/help_modal/agents_bindings.py`: 57-char boxes,
    descriptions ≤ 32 chars)
  - a footer entry only when the binding is conditional (see `src/sase/ace/CLAUDE.md`)
  - a `_CONTEXTUAL_APP_DUPLICATES` pair for each shared key (`keymaps/registry.py`),
    including the flag-disjoint Agents pairs while the flag exists
- **Enforcement tests:** `tests/test_command_catalog.py`,
  `tests/test_keymaps_defaults.py`, `tests/test_keymaps_app_bindings.py` (fallback
  parity) and `tests/test_command_catalog_guards.py`.

### 3.5 Collapsing the node panel (Ctrl+S)

- **Mechanism.** Add a `-nodes-collapsed` class on `#agents-content` that sets
  `#agent-list-container { display: none; }`.
  - The lists stay mounted, so selection, folds and incremental row patches keep
    running.
  - j/k keep working because `action_next_patch` → `_navigate_agents_panel` walks the
    app's stop model, not the widget.
  - The same mechanism is already proven by onboarding and the artifact-file viewer
    (`styles.tcss:3725-3731`).
- **Representation.** This is the answer to "how should a collapsed node panel look". It
  follows the collapsed-sidebar convention of IDE activity bars:
  - **Node spine.** A new slim widget, about 2 cells wide, at the left edge of
    `#agents-content`, shown only while collapsed:
    - top cell: an accent `»` affordance
    - below it: a dim vertical track with an accent **thumb**, positioned and sized in
      proportion to the selected node's index among the visible navigation stops (a
      tribe panel's index under whole-panel focus), like a minimap scrollbar
    - clicking the spine expands the panel; its tooltip names Ctrl+S
    - it renders from in-memory selection state in O(height) and updates on the
      immediate j/k paint
  - **Info-row chip.** It takes the slot freed by the removed `view: … (p)` chip:
    `nodes 12/47 · ^S`, or `zoom · Z · nodes 12/47` while zoomed. It appears only while
    collapsed, and the key text comes from the live keymap.
- **Whole-panel focus is not cleared** by collapsing. j/k under tribe focus keep
  stepping tribes, and the Main deck's Summary card shows each one. That is exactly how
  the old "zoom a tribe" workflow becomes Z plus j/k.
- **No auto-expand.** List actions (folds, isolate, filter edits) still apply to the
  hidden list, and the chip and spine reflect the result.
- After collapsing, verify that no hidden `AgentList` keeps Textual focus in a way that
  swallows keys. If one does, move focus off it.
- The collapse must compose with onboarding and the artifact-file viewer layouts.

### 3.6 In-place zoom (`Z`)

- **`Z` zooms in.** It snapshots the deck-area state (layout, panels, focus, ratio,
  collapse), then shows only the **focused** panel (SINGLE) and collapses the node
  panel. That panel keeps its widget, card and scroll. A second `Z` restores the
  snapshot exactly.
- **Layout keys end the zoom.** `\`, `|` or Ctrl+S while zoomed drop the snapshot and
  apply to the current state.
- This replaces the zoom modal. Search (`,/`), `E`, cards and decks all work normally
  while zoomed, which is everything the modal offered.

### 3.7 Spread vs paged (algorithm)

- **Configuration.** One field: `ace.agent_decks.spread_max_screens`, default `1.5`. It
  is measured in the panel's viewport heights ("screens"), not lines, because a 25-row
  top/bottom panel and a 60-row single panel must behave alike. `0` means always paged.
- **Decision.** A pure function decides the mode, with hysteresis as an internal
  constant:

```python
def decide_render_mode(*, card_count, has_solo_card, total_rows, viewport_rows,
                       spread_max_screens, previous, same_subject) -> RenderMode:
    if card_count <= 1:
        return RenderMode.SPREAD                  # trivially one page
    if has_solo_card or spread_max_screens <= 0:
        return RenderMode.PAGED
    if total_rows is None:                        # sizes not known yet (Files probes)
        return previous if (same_subject and previous) else RenderMode.PAGED
    budget = spread_max_screens * max(1, viewport_rows)
    if same_subject and previous is RenderMode.SPREAD:
        return RenderMode.PAGED if total_rows > budget * 1.10 else RenderMode.SPREAD
    if same_subject and previous is RenderMode.PAGED:
        return RenderMode.SPREAD if total_rows <= budget * 0.90 else RenderMode.PAGED
    return RenderMode.SPREAD if total_rows <= budget else RenderMode.PAGED
```

- **Measurement.**
  - Compute a cheap **lower bound** first: sum each card's logical line count, and stop
    early once it passes `budget * 1.10`, which settles the deck as PAGED without
    rendering.
  - Only when the lower bound falls inside the band is the **exact** rendered height
    measured, with a Rich render at the panel's content width. Results are cached by
    `(card digest, width)`.
  - Small documents are cheap to render, and large ones never reach the exact step.
- **Subject.** The subject is the node identity plus the attempt pin. A selection change
  is a new subject. Hint-mode toggles, content refreshes (a streaming reply, SASE
  CONTEXT lanes resolving) and geometry changes (resize, split, ratio, `d`/`.` toggles,
  collapse) keep the same subject.
- **Partial documents never decide the mode.** A partial document is the immediate
  header-only paint during j/k.
- **Reading position is anchored.** Every mode transition keeps the active card's header
  at the same viewport row. Going spread → paged activates the card at the viewport top
  and keeps the offset within it. Going paged → spread scrolls so the active card's
  header lands where it was. This replaces the research's one-way rule.
- **New subjects.** In spread mode, start at the top. In paged mode, show the panel's
  preferred card if the new node has it (sticky Reply), otherwise the default card; the
  preference itself is kept.
- **During a partial paint.** If a paged panel's active card is missing from the partial
  document, keep the tab strip and render an empty body until the full document lands.
  Do not flash Context.
- **Each panel decides independently** from its own viewport.

### 3.8 Visual language

The target is "intuitive, reliable, beautiful". Every UI phase verifies with
`sase screenshot` PNGs and inspects them; `SASE_FEATURE_FLAGS='{"agent_decks": true}'`
enables the flag for a live TUI.

- **Deck accents** reuse today's colors: Main `$secondary` (the old metadata border),
  Files `green`, Tools `#87D7FF`. Each deck also gets a single-cell, non-emoji glyph
  that matches the existing TUI glyph vocabulary.
- **Frame.** Each deck panel is a bordered frame in its deck's accent. In a split, the
  **focused** panel uses the full-strength accent with a bold title. The unfocused panel
  uses the same accent at reduced opacity (about 35%) with muted title text. Geometry is
  identical in both states, so focus changes never shift content.
- **Border title (top-left): where am I.** Render it with a pure helper that has full,
  compact and micro tiers (reuse `PanelTabStrip`'s tiering ideas):
  - Full: `◆ MAIN ┃ Context │ Reply  2/2`. The active card is a bold accent pill;
    inactive cards are muted.
  - Compact: `▤ FILES ┃ ‹ 3/27 › diff · src/foo.py`.
  - Micro: `FILES 3/27`.
- **Border subtitle (bottom-right): what else.**
  - A deck switcher: `main · files 3 · tools`. The active deck is bold accent, decks
    with content use normal text and empty decks are dim. Counts appear only when a
    no-I/O probe knows them.
  - Deck-specific status: Files line counts `1-120 of 693 lines · E editor`, and a
    subtle `spread` tag in spread mode.
- **Spread separators.** Between consecutive cards (not before the first), leave one
  blank line, then a full-width heavy rule (`━`) in the deck accent with the card title
  at the left, e.g.
  `━━ ◆ Reply ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`. Files
  separators carry the page label.
- **Empty-state card.** Two centered lines in muted and dim text, such as
  `No files for this agent` and `^N next deck · ^P previous deck`. Key text comes from
  the live keymap.
- **Mock.** A left-right split with focus on the right, node panel expanded:

```
╭─ ◆ MAIN ┃ Context │ Reply ──────────╮╭─ ▤ FILES ┃ ‹ 1/3 › diff · src/foo.py ───╮
│ (dim border)                        ││ (full-strength green border)             │
│ SASE CONTEXT …                      ││  12 │ -old line                          │
│ AGENT PROMPT …                      ││  12 │ +new line                          │
╰───────────── main · files 3 · tools ╯╰──── 1-120 of 693 lines · E editor ───────╯
```

### 3.9 Retirements and removals (at cut-over)

- **Retired action ids** go into `_RETIRED_APP_KEYS`: `choose_agent_view`,
  `next_agent_metadata_section`, `prev_agent_metadata_section`.
- **Renames.** `next_agent_file`/`prev_agent_file` become
  `next_chop_run`/`prev_chop_run` (Services only), with `LEGACY_APP_KEY_ALIASES`
  entries.
- **Behavior changes.** `scroll_prompt_down/up` lose their Agents branch. `zoom_panel`
  keeps its id with the §3.6 behavior.
- **Delete:**
  - the `p` picker
  - the zoom modal and its CSS
  - `DetailPanelMode`, `DetailLayoutMode`, `DETAIL_LAYOUT_CYCLE`
  - the `view:` chip and `_VIEW_MODE_STYLES`
  - the legacy panel ids and CSS
  - the section-stop actions
  - the retired redirect handlers (`action_toggle_layout`, `action_toggle_thinking*`)
- **Keep** the section-anchor machinery: folds and spread mode use it.

## 4. Architecture

### 4.1 Card-partitioned Main documents

- **The wrapper.** Add a `CardPart` Rich renderable: `card_id`, `title` and
  `renderables`. It delegates `__rich_console__` and `__rich_measure__` to
  `Group(*renderables)`, so rendering a `Group` of card parts is byte-identical to
  rendering the children directly.
- **Builders** wrap their sections and still call `self.update(Group(...))`, so builder
  churn stays minimal.
- **Tree walkers must descend into `CardPart`:**
  - `util/renderable_digest.py`: card id and title are part of the digest
  - `find_identity_header`, `find_member_jump_map`, `find_member_roster`
    (`widgets/prompt_panel/_identity_header.py`)
  - `renderable_to_text`
  - `inline_document_renderable`
  - the zoom seed, until it is deleted
- **Splitting.** `split_card_parts(content)` returns the ordered card parts. Loose
  top-level renderables fall into `context`.
- **Hint mode and parallel steps** currently put the reply inside one `Text`
  (`_agent_display_hint_body.py`, `_agent_display_step_render.py:83-94`). Split each
  into context and reply parts, and keep hint numbering continuous across them.

### 4.2 Main source and Main views

- **Builders have no geometry dependencies** (verified: no `self.size` or region reads
  under `widgets/prompt_panel/`), so the builder can run hidden.
- **Deck mode:**
  - `AgentPromptPanel#agent-prompt-panel` stays mounted but hidden (`display: none`) as
    the single **Main source**. It keeps every builder, worker, timer, generation guard
    and the identity/jump sinks, so shared chrome is fed even when no panel shows Main.
    Its existing `update()` override is the choke point.
  - After the sinks and the digest skip, `update()` calls a new attached **main-document
    sink**. That sink carries the content plus a `partial` flag, which is set by the
    `update_header_only` path.
  - `AgentDetail` turns the content into a `MainDeckDocument` (cards, subject identity,
    partial) and pushes it to every panel whose active deck is Main.
- **Shared view base.** Extract the view-side features from `AgentPromptPanel` into a
  shared base or mixin:
  - the `SectionTrackingVisual` render and anchor publish/resolve
  - the layout reserve
  - the bottom pin
  - the section-at-row lookup

  Replace the `id == "agent-prompt-panel"` gates with a class-level capability.
  - Flag off: `AgentPromptPanel` keeps using the base unchanged.
  - Flag on: `MainDeckView` uses it to compose the visible cards (the active card when
    paged, all cards with separators when spread), with a digest skip. Switching cards
    recomposes from the stored document; nothing is rebuilt.

### 4.3 Files and Tools views

- **Per-panel instances.** Each deck panel owns one `AgentFilePanel` and one
  `AgentLLMCallsPanel`. Duplicate Files/Tools decks share the module-level caches
  (`file_cache`, `_inflight_diff_tasks`, `fetch_tool_calls_cached`).
- **Scroll containers** are resolved through the ancestor (`self.parent`), never through
  app-wide `query_one("#agent-file-scroll")` / `("#agent-llm-calls-scroll")`. This is
  equivalent for the flag-off tree.
- **Public methods** replace `AgentDetail`'s private-field writes into the file panel.
- **File/LLM messages** (`FileListChanged`, `FileLineCountChanged`,
  `FileVisibilityChanged`, `LLMCallsVisibilityChanged`) bubble to their `DeckPanel`. It
  updates its own title and subtitle and stops the message. `AgentDetail`'s legacy
  handlers stay for flag off.

### 4.4 Composition

```
AgentDetail#agent-detail-panel            (deck mode)
  Vertical#agent-detail-layout
    AgentHeaderPanel#agent-header-panel   shared chrome
    AgentPromptPanel#agent-prompt-panel   .-deck-source (hidden Main source)
    DeckArea#agent-deck-area              .-single | .-top-bottom | .-left-right, .-ratio-30|50|70
      DeckPanel#agent-deck-panel-0        .-focused, deck accent class
        VerticalScroll (main)  > MainDeckView
        VerticalScroll (files) > AgentFilePanel
        VerticalScroll (tools) > AgentLLMCallsPanel
        (per-panel search overlay + command line, added by deck-action-retarget)
      DeckPanel#agent-deck-panel-1        hidden in SINGLE
    AgentJumpPanel#agent-jump-panel       shared chrome
```

- Only the active deck's scroll is displayed in each panel. Per-deck scroll hosts keep
  each deck's scroll position while you switch decks.
- The flag is read once at `AgentDetail` compose time and cached as `decks_enabled`,
  never at import time (`tools/check_feature_flags`). The flag-off compose tree stays
  exactly as it is today.

### 4.5 Lazy loading and availability probes

- **Debounced updates.** The debounced update always refreshes the Main source. It
  refreshes a Files or Tools view only when a panel shows that deck.
- **Switching decks.** A panel switching to a deck updates that view for the current
  subject at once, from caches first.
- **No hidden probes.** The unconditional hidden LLM Calls `update_display` probe
  (`_agent_detail_display.py:226-229`) goes away in deck mode.
- **Availability (tab strip, split rule) is decided with no I/O:**
  - Files: a pure page-list function over in-memory state (commit diffs, `file_cache`,
    the linked-delta cache, `extra_files`).
  - Tools: `build_cached_slow_tool_sources` / `peek_tool_calls_cache_entry`.
  - Unknown means "show the deck without a count".

### 4.6 Pure state model

- `widgets/decks/` holds frozen dataclasses and pure transition functions, each
  exhaustively unit-tested:
  - `DeckId`
  - `DeckLayout`
  - `RenderMode`
  - `DeckPanelState(deck, preferred_card)`
  - `DeckAreaState(layout, panels, focused, ratio, nodes_collapsed, zoom_snapshot)`
- Widgets only render state and dispatch actions. Every transition in §3.3, §3.5 and
  §3.6 is a pure function.

### 4.7 Feature flag

- The `deck-panel-core` phase creates a **`beta` flag `agent_decks`** with:

  ```
  sase flag new agent_decks \
    --when-enabled "The Agents tab shows one or two deck panels of agent data decks (Main, Files, Tools) and cards instead of the metadata panel plus one Files or LLM Calls panel." \
    --when-disabled "The Agents tab keeps the legacy metadata panel plus one secondary Files or LLM Calls panel, the p view picker, and the Z zoom modal." \
    --remove-when "The agent data decks epic's cut-over phase lands: every Agents detail action targets deck panels and the legacy panels, picker, and zoom modal are deleted."
  ```

- Paste the printed registry entry. Record the flag bead id in the phase notes, because
  `deck-cutover` closes it.
- It is epic scaffolding: phases up to `deck-state-persistence` land behind it, and
  `deck-cutover` removes it.
- Every flagged phase tests **both states**. The existing suite covers Off, and new
  tests use `override_flags(agent_decks=True)`.

### 4.8 Performance rules (read `tui_perf.md`)

- **j/k.** A j/k paints only the shared header and the partial Main document at once.
  Deck content and mode decisions ride the existing `DetailPanelDebouncer` path.
- **No remounts on layout change.** Layout and ratio are CSS classes, and every
  DeckPanel and view is pre-composed.
- **Only shown decks load.** Probes do no I/O.
- **Measurement.** Only small content is measured exactly, and results are cached by
  `(digest, width)`. Files size probes run off-thread, are bounded, and stop once they
  pass the band.
- **Workers check subject identity.**
- **Persistence I/O stays off the event loop.** Pump callbacks stay thin (rule 2).
- **Gates.** With `SASE_TUI_PERF=1`, j/k p95 must stay under 16 ms in the SINGLE and
  LEFT_RIGHT layouts. Run `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` before and
  after `deck-panel-core`, `deck-spread-mode` and `deck-cutover`.
- **Module size.** New modules stay under the repo's ~500-line module limit (the
  `toobig` lint); split into focused modules instead.

## 5. Phase: `llm-calls-subject-guard`

- **Stale-worker bug.** In `widgets/llm_calls_panel.py`, `_update_display_impl` returns
  early while a worker runs, and `on_worker_state_changed` only compares the worker, so
  agent A's calls can paint onto agent B. They also post a wrong
  `LLMCallsVisibilityChanged`.
  - Carry the subject identity in the fetch result, and drop any result whose identity
    is not `_current_agent`'s.
  - When a stale worker finishes while a different agent is current, start that agent's
    fetch, which was skipped by the early return.
  - Keep the cache, throttle and scroll-restore behavior.
- **Split-key display.** In `keymaps/key_validation.py`, add `backslash` → `\` and
  `vertical_line` → `|` to `_KEY_DISPLAY`. Confirm that `left_curly_bracket` and
  `right_curly_bracket` validate and display too. User overrides to these keys then
  validate, and help and footer show real glyphs.
- **Tests:**
  - Select A, then select B while A's fetch is in flight: B never shows A's calls and
    B's fetch runs.
  - Validation and display tests (`tests/test_keymaps_validation.py`).

## 6. Phase: `main-card-partition`

- **`CardPart`.** Add it plus the card id/title constants (§4.1) in a small module under
  `widgets/decks/`, or next to the prompt panel if that avoids import cycles.
- **Builders.** Wrap every builder branch per §3.1. Relevant code:
  - `_agent_display_render.py` (regular agent, queued agent, no-prompt case)
  - `_agent_display_family_render.py`
  - `_agent_display_attempts.py`
  - `_agent_display_step_render.py` (proc shell, monitor, gate, bash/python, parallel)
  - `_workflow_render.py` / `_workflow_display.py`
  - `_agent_display_clan.py` and `_agent_display_tribe*.py` (a single Summary card)
  - `update_header_only` (Context only)
  - `show_empty`
  - the hint-mode renderers (`_agent_display_hint_render.py`,
    `_agent_display_hint_body.py`, `_agent_display_hint_sections.py`)
- **Traceback.** Move the error traceback to the top of Reply under a `TRACEBACK`
  heading in every branch that has the quirk.
- **Parallel step.** Its `STEP OUTPUT` heading moves out of `header_text` into the
  Output card.
- **Walkers.** Teach the tree walkers `CardPart` (§4.1).
- **Symvision.** If `split_card_parts` has no non-test consumer yet, land it in
  `deck-panel-core` instead, or put the test helper under `src/sase/ace/testing/`. Do
  not add unused public symbols.
- **With the flag off, output is unchanged except for the traceback move.**
- **Tests:**
  - A partition test per node kind: ids, titles, order, the traceback's position.
  - A plain-text equivalence test against the pre-change document (modulo the
    traceback).
  - Hint numbering continuity across cards.
  - Digest stability: unchanged documents still skip.
- **Goldens.** Regenerate the goldens this phase changes (failed-agent views) with a
  targeted `just fix-tui-screenshots -- <selectors>`, and inspect the report.

## 7. Phase: `deck-panel-core`

- **Flag.** Create `agent_decks` (§4.7).
- **Package.** Build `widgets/decks/` with the model (§4.6: only the decks, cards and
  preferred-card parts needed here), `MainDeckDocument`, `split_card_parts`, the shared
  view base extracted from `AgentPromptPanel` (§4.2), `MainDeckView`, `DeckPanel` and
  `DeckArea`.
  - Pre-compose two panels; the second stays hidden.
  - Update the `widgets/__init__.py` / `.pyi` lazy exports if anything is exported.
- **Compose and delegation.** Add the flag-gated `AgentDetail.compose` and the deck-mode
  delegation for:
  - `update_display`
  - `update_display_immediate`
  - `update_display_with_hints`
  - `show_empty`
  - `show_tribe_summary`
  - identity-change scroll resets
- **Views and data flow.** Fan out the main-document sink, add the per-panel Files and
  Tools views with ancestor-resolved scrolls (§4.3), lazy loading and the no-I/O
  availability probes (§4.5).
- **Rendering.** Build the pure title/subtitle helpers and the empty-state card (§3.8).
  Rendering is **paged only** (spread arrives in `deck-spread-mode`). Show the partial
  paint behavior of §3.7.
- **Disabled in deck mode:** hide the `view:` chip, disable `choose_agent_view` (`p`)
  and disable `zoom_panel` (`Z`) until `node-panel-collapse-zoom` restores it.
- **Tests (flag on, driving the DeckArea API):**
  - compose
  - j/k updates the Main view and the header
  - Files/Tools views load only when shown
  - empty states
  - title and subtitle tiers
  - `split_card_parts`
  - the flag-off suite still passes
- **Perf.** Record the j/k bench before and after.

## 8. Phase: `deck-navigation-keys`

- **Actions.** Add `next_deck_card`/`prev_deck_card` (Ctrl+J/K) and
  `next_deck`/`prev_deck` (Ctrl+N/P) with full registration (§3.4).
  - Flag off: Ctrl+J/K keep the section stops and Ctrl+N/P keep file cycling.
  - Services chop runs and Artifacts load-more/unload are unchanged in both states.
- **Preferred card.** Explicit card selection sets it. On a new subject, show the
  preferred card if the node has it, otherwise the default; keep the preference.
- **Files cards.** They cycle files with wrap, and the `(agent, slot)` anchor store
  keeps working.
- **Scrolling.** Ctrl+D/U, `g`/`G` and the bottom pin target the focused panel's visible
  scroll. The pin follows the active card; for Main, re-apply it through the view base.
- **Footer.** Show `^J/^K` only while the focused deck has more than one card.
- **Tests:**
  - Pilot key tests in both flag states.
  - Keymap parity and default tests, and contextual-duplicate resolution per tab.
  - Services and Artifacts regressions: `tests/ace/tui/test_axe_chop_run_nav.py` and
    `test_artifacts_limit_keys.py`.
- **Goldens.** Add the first flag-on PNG goldens: single Main paged on Context and on
  Reply, a Files card, the Tools card, and an empty state.

## 9. Phase: `deck-splits-focus`

- **Actions.** Add `toggle_deck_split_below` (`backslash`), `toggle_deck_split_right`
  (`vertical_line`), `toggle_deck_focus` (Ctrl+F), `grow_deck_panel` (`}`) and
  `shrink_deck_panel` (`{`) with full registration.
  - The Services and Artifacts owners of Ctrl+F/`{`/`}` stay.
  - Flag off, Agents Ctrl+F/B keep their half-page scroll.
  - Flag on, Agents Ctrl+B does nothing.
- **State machine.** Implement the pure transitions of §3.3 (split, unsplit, rotate,
  choosing a new panel's deck, duplicate-deck opening on the next card, ratio steps,
  focus) with a test for every transition row.
- **Widgets.**
  - `DeckArea` layout and ratio classes, with no remounts.
  - Focus styling and click-to-focus.
  - The new panel takes focus.
  - Fan the Main document out to both views.
  - Duplicate Files/Tools views must not double fetches: rely on in-flight dedupe and
    verify it.
  - Views re-render on resize (image cards resize).
- **Help and footer.** `^F` and `{/}` appear in the footer only while split.
- **Tests:** pilot tests for every key in both flag states. **Goldens:** TOP_BOTTOM,
  LEFT_RIGHT, focus styling, and Context | Reply.

## 10. Phase: `deck-action-retarget`

- **Inventory first.** Grep every consumer of:
  - `#agent-prompt-scroll`, `#agent-prompt-panel` (as a _view_), `#agent-search-scroll`
  - `#agent-file-panel`, `#agent-file-scroll`
  - `#agent-llm-calls-panel`, `#agent-llm-calls-scroll`
  - `DetailPanelMode`, `DetailLayoutMode`
  - `is_file_visible`, `is_llm_calls_visible`, `is_metadata_visible`
  - `effective_detail_scroll_id`, `get_editor_file_info`

  The research counted about 216 source references in 17 files. Give each consumer a
  deck-mode path, or mark it for deletion at cut-over, and list the inventory in the
  phase's final notes.

- **Folds (`z…`).** Fold-level keys stay document-global. `za`/`zA` resolve the section
  at the viewport top of the focused panel's Main view, or the other panel's Main view
  if the focused panel isn't Main; with no Main shown they are a no-op with a hint.
- **Hints (`v`).** Hints render in Main views. If no panel shows Main, the focused panel
  switches to Main first.
- **Search (`,/`).** A per-panel overlay searches the focused panel's deck:
  - Main: the text of all cards, with card titles as separators.
  - Files: the active card (all cards in spread).
  - Tools: the LLM Calls text.

  Salvage the corpora and structural-exit logic from `modals/zoom_panel_search.py`, and
  keep today's search semantics otherwise.

- **`E`.** Files opens the real path. Tools and Main open the active card's text as a
  temporary `.md`, salvaging `modals/zoom_panel_content.py` `editor_info`.
- **`l`/`h`/`L`/`H`.** While the focused deck is Tools they step and set LLM Calls
  detail levels, fixing today's unreachable `h`/`L`. Otherwise they keep their current
  behavior.
- **Clipboard file-path targets** (`actions/clipboard/_agents.py`,
  `_palette_helpers.py`) use the focused panel's Files view, or any visible one.
- **Visibility consumers** treat "Files visible" as "any panel shows Files": the footer,
  `commands/context.py`, `event_refresh/_auto_refresh_surfaces.py` and
  `_display_detail_footer.py`.
- **Chrome toggles.** `d`/`.` re-apply the pin on the Main views.
- **SLOW TOOL CALLS overflow hint**
  (`widgets/prompt_panel/_agent_slow_tools.py:243-250`). Make it accurate in both
  states, built from the live keymap: with decks on, point at the Tools deck and its
  Ctrl+N/P keys; with the flag off, drop the wrong `]` reference. Update
  `tests/ace/tui/widgets/test_agent_slow_tools.py`.
- **Tests:** a pilot test per retargeted action with the flag on; flag-off behavior
  unchanged.

## 11. Phase: `node-panel-collapse-zoom`

- **Collapse.** Add `toggle_node_panel` (Ctrl+S) with the §3.5 mechanism, the node spine
  widget, the info-row chip and the focus safety check. Register it fully; its help row
  reads "Collapse/expand node panel".
- **Zoom.** `zoom_panel` implements §3.6 when decks are enabled; the old modal stays for
  flag off. Update the palette label ("Zoom focused deck panel"). The chip shows the
  zoom state.
- **Tests:**
  - Collapse and expand in SINGLE and LEFT_RIGHT.
  - j/k while collapsed updates the header, the spine thumb and the chip.
  - Z round-trips; layout keys end zoom.
  - Tribe focus plus Z plus j/k steps tribe summaries.
  - Composition with onboarding and the artifact viewer.
- **Goldens:** collapsed single, collapsed split, zoomed.

## 12. Phase: `deck-spread-mode`

- **Config.**
  - Add `ace.agent_decks.spread_max_screens: 1.5` to `src/sase/default_config.yml` under
    `ace:` next to `tool_calls:`, with a comment explaining screens and that `0` means
    always paged.
  - Add the schema entry in `src/sase/config/sase.schema.json`: `ace` has
    `additionalProperties: false`, and the value is a number ≥ 0 with default 1.5.
  - Add a typed parser: a frozen dataclass plus a coercing parse function, following
    `prompt_submission_settings.py`, read in `actions/_state_init_late.py`.
  - Document it in `docs/configuration.md` and update the schema/default parity tests
    (`tests/test_config_schema*.py`).
- **Decision.** Add `decide_render_mode` (§3.7) with exhaustive unit tests: boundaries,
  both hysteresis edges, unknown totals, solo cards, `0`, single card and same versus
  new subject.
- **Measurement.** Cheap lower bounds with early exit, then the exact measurement for
  small decks only, cached by `(card digest, width)`. Each panel uses its own viewport
  width and height.
- **Main spread.**
  - Titled separators (§3.8) carry a hidden `sase_deck_card=<id>` style meta, so
    `SectionTrackingVisual` publishes card anchors next to section anchors.
  - The active card is the last card anchor at or above `scroll_y`, and the title pill
    tracks it as you scroll.
  - Ctrl+J/K scroll a card anchor to the top, using the existing layout reserve so the
    last card can reach the top.
  - Transitions anchor the reading position; a new subject starts at the top.
- **Files spread.**
  - An off-thread, bounded size probe covers all pages. It is cached by
    `(path, mtime, size)` for files and uses the in-memory text for the live and linked
    diffs, and it stops once the total passes the band.
  - Solo cards force paged.
  - Spread renders all pages in order with titled separators, through the existing page
    rendering (line numbers, per-page render caps).
  - A new subject starts paged on the default page until sizes are known. The later
    paged → spread switch is viewport-stable because the default card stays anchored.
- **Tests:**
  - A streaming reply crossing the threshold keeps the reading position.
  - Ctrl+J in spread scrolls to the Reply header.
  - A small multi-diff agent spreads, and an image forces paged.
  - Resize and split re-decide with hysteresis.
- **Goldens:** spread Main, spread Files, paged-after-threshold. **Perf:** j/k bench
  before and after.

## 13. Phase: `deck-state-persistence`

- **Store.** Add a model module following `models/agent_fold_persistence.py`, with the
  lifecycle following `actions/agents/_fold_persistence.py`:
  - File: `~/.sase/ace_agents_deck_state.json`, schema v1.
  - Contents:
    `{layout, ratio, focused, nodes_collapsed, panels: [{deck, preferred_card}]}`. A
    zoomed session persists its pre-zoom snapshot.
  - Loading fails open, ignoring unknown decks or layouts.
  - Writes are atomic, and saves are coalesced off-thread.
  - Flush on exit next to `_flush_agents_fold_state` (`actions/lifecycle.py`).
  - Load in a startup worker and apply once it lands.
- **Scope.** Deck mode only.
- **Tests:** round-trip, corrupt file, unknown values, save coalescing, flush on exit.

## 14. Phase: `deck-cutover`

- **Flag.** Delete every `agent_decks` Off branch, make the On branch unconditional,
  remove the registry entry, sync the generated schema block
  (`tools/sync_feature_flags_schema`) and close the flag bead in this change.
- **Delete the legacy UI and its tests** (§3.9):
  - `modals/agent_view_modal.py`
  - `actions/agents/_agent_view_picker.py` and its mixin wiring
  - `modals/zoom_panel_*.py`
  - the zoom seed code in `actions/agents/_panel_detail.py`
  - the `ZoomPanelModal` CSS and its `_export_table` entry
  - the panel enums and cycle
  - the legacy scroll ids and CSS
  - the `view:` chip
  - the section-stop actions
  - the retired redirect handlers
  - any view code left dead on the hidden Main source
- **Keymap registry.** Apply the retirements and renames in §3.9 (`_RETIRED_APP_KEYS`,
  `LEGACY_APP_KEY_ALIASES`). Remove the `choose_agent_view`/`pick_artifacts_project`
  pair and the flag-disjoint pairs. Update `default_config.yml`, including its stale "Z
  zooms agent detail…" comment, `_BINDING_META`, the fallback bindings, palette metadata
  and help rows.
- **Tests.** Migrate tests that used the enums or legacy ids only to set up visibility
  so they use the deck API. Delete tests of deleted code. Remove the
  `override_flags(agent_decks=True)` wrappers.
- **Goldens.** Run the full `just fix-tui-screenshots` through `/sase_monitor`. Inspect
  every creation, removal and update group before finalizing, then run the final j/k
  perf bench.

## 15. Phase: `deck-docs-glossary`

- **`docs/ace.md`.** Replace these sections with one "Agent data decks and cards"
  section covering vocabulary, the §3.1 and §3.2 tables, layouts and transitions, keys,
  spread/paged and its config, node-panel collapse, zoom and persistence:
  - the view picker (about `:5123-5146`)
  - the zoom panel (about `:4784-4823`)
  - the metadata-panel layout text

  Then fix the key tables (about `:1368-1376`, `:2050`, `:2198`, `:2307`), the File
  Panel and LLM Calls sections (about `:4777`, `:5427-5456`) and the slow-tools overflow
  text (about `:5353`).

- **`docs/configuration.md`.** Update the new and retired keymap ids and document
  `ace.agent_decks`.
- **Doc sweep.** Search the remaining docs for stale zoom, `p` picker, Ctrl+J
  section-stop and "metadata panel" references and fix each hit: `agent_families.md`,
  `agent_images.md`, `llms.md`, `development.md`, `memory.md`, `query_language.md` and
  `artifacts_pane_contract.md`.
- **Post-update notice.** Add a one-time post-update toast summarizing the Agents key
  changes, following `_keymap_unification_notice.py` and its startup hook. Keep it
  silent on fresh installs and tests.
- **Glossary.** The user asked for glossary strands, and approving this plan authorizes
  exactly these memory changes. Use `/sase_memory_write`, then run `sase memory init`.

  **Create** `sase/memory/glossary/agent-data-deck.md`:

  ```markdown
  ---
  keyword: Agent Data Deck
  aliases:
    - deck
    - agent data decks
  ---

  An agent data deck is a named, ordered set of agent data cards about the selected sase
  node, shown in an Agents-tab deck panel. The built-in decks are Main (a Context card
  and a Reply card — titled Output for proc shells, monitors, gates, and workflow steps
  — or one Summary card for agent clans and tribe panels), Files (one card per diff or
  attached file), and Tools (the LLM Calls card). A deck is presentation only and owns
  no agent data; `Ctrl+N`/`Ctrl+P` cycle the focused deck panel through the decks.
  ```

  **Create** `sase/memory/glossary/agent-data-card.md`:

  ```markdown
  ---
  keyword: Agent Data Card
  aliases:
    - card
    - agent data cards
  ---

  An agent data card is one titled unit of detail inside an agent data deck, such as
  Context, Reply, one file's diff, or LLM Calls. A deck panel shows a multi-card deck
  spread — every card on one scrollable page, separated by titled rules — when the cards
  fit within `ace.agent_decks.spread_max_screens` panel heights, and paged — one card at
  a time — otherwise. `Ctrl+J`/`Ctrl+K` move to the next or previous card in both modes,
  and a paged deck keeps the chosen card (for example Reply) as the selection moves
  between nodes.
  ```

  **Create** `sase/memory/glossary/deck-panel.md`:

  ```markdown
  ---
  keyword: Deck Panel
  aliases:
    - deck layout
  ---

  A deck panel is one Agents-tab detail region that shows one agent data deck. The
  Agents tab shows one deck panel, or two in a horizontal split (`\`, one above the
  other) or a vertical split (`|`, side by side), following vim's split naming; pressing
  the same split key again closes the second panel, and the other split key rotates the
  layout. `Ctrl+F` moves the logical focus that deck, card, scroll, search, and fold
  keys act on, and `Z` zooms the focused deck panel in place. The identity header and
  the jump panel span every deck panel and belong to none.
  ```

  **Create** `sase/memory/glossary/node-panel.md`:

  ```markdown
  ---
  keyword: Node Panel
  ---

  The node panel is the Agents tab's left column of sase node rows, one agent list per
  tribe panel. `Ctrl+S` collapses it into a slim node spine without unmounting it, so
  j/k still move the selection and the identity header still follows it; the spine and
  an info-row chip show the selected position and restore the panel.
  ```

  **Edit** these existing strands:
  - `sase/memory/glossary/llm-calls.md`: "LLM Calls is the Agents detail and zoom view
    of normalized provider tool-call artifacts" → "LLM Calls is the Agents-tab card, in
    the Tools agent data deck, that shows normalized provider tool-call artifacts". Keep
    the rest.
  - `sase/memory/glossary/tool-run.md`: "it is not the Agents-tab [[glossary:llm-calls]]
    view" → "it is not the Agents-tab [[glossary:llm-calls]] card".
  - `sase/memory/glossary/agent-relation-jump-target.md`: "the sticky footer below the
    Agents detail panels" → "the sticky footer below the deck panels". Remove the
    retired "`Ctrl+J`/`Ctrl+K` metadata section stops," item from the ambiguity list.

  If the final implementation's key or config names differ from this plan, adjust the
  strand text to match the shipped behavior. Never describe unshipped behavior.

- **Help modal.** Give it a final consistency pass.

## 16. Verification (every phase)

- Run `just fix` (or at least `just fmt`), then `sase tool run check`. Use
  `/sase_monitor` when the check may outrun the turn. Do not run `just check-full`
  unless explicitly instructed.
- Phases that change rendered TUI output run targeted `just fix-tui-screenshots -- …`,
  via `/sase_monitor` when long, and inspect the report and every golden change.
  `deck-cutover` runs the full form.
- UI phases capture live `sase screenshot` PNGs with the flag on and inspect them for
  layout, focus styling, tab strips, separators and empty states.
- Phases `deck-panel-core` through `deck-state-persistence` test both flag states.
- Run the perf gates in §4.8 where listed.
