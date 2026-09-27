---
tier: epic
title: Agents node rail and unmistakable deck zoom
goal: "Ctrl+S turns the Agents-tab node sidebar into a fixed-width, row-for-row node
  rail that still shows every tribe, group, and node as glyphs. Z zoom hides the left
  column entirely and looks unmistakably different from the rail. Zoom never leaks into
  the saved Ctrl+S preference, and no key silently drops a split.

  "
phases:
  - id: sidebar-modes
    title: Three sidebar modes and the zoom state fixes
    depends_on: []
    size: medium
    description:
      "sidebar-modes: derive EXPANDED / RAIL / HIDDEN from deck-area state. Stop zoom
      from writing nodes_collapsed, and make Ctrl+S while zoomed restore the snapshot
      like Z. Route every deck-state change through one sync choke point, un-nest the
      info-row zoom chip, and update the tests, docs, and zoom goldens."
  - id: rail-vocabulary
    title: Pure node-rail vocabulary module
    depends_on: []
    size: medium
    description:
      "rail-vocabulary: add one pure module that owns the rail geometry constants, the
      glyph and color vocabulary, and the fixed-width cell builders for rows, banners,
      tribe titles, and overflow. Also add the tooltip text helper and the legend
      entries, with exhaustiveness and cell-width tests. There is no widget wiring."
  - id: rail-projection
    title: Paint-time rail projection inside AgentList
    depends_on:
      - rail-vocabulary
    size: medium
    description:
      "rail-projection: give AgentList a set_rail() render mode that overrides
      _get_visual with a cache validated by prompt identity. Record an all-banner
      _group_at_row map, add a rows-changed hook to every structural path (fixing the
      insert path's ordering), and paint the overflow subtitle. Add unit and guard
      tests, still unwired."
  - id: zoom-chrome
    title: Structural zoom chrome on the zoomed deck panel
    depends_on:
      - sidebar-modes
    size: medium
    description:
      'zoom-chrome: DeckArea marks the zoomed panel -zoomed with ZoomChrome context. The
      panel then gets a heavy border in its own deck accent, a reverse-gold ZOOM chip
      leading its title, and a split-position plus "Z restore" hint in its subtitle.
      Includes title/subtitle ladder tests and zoom goldens.'
  - id: rail-wiring
    title: Wire the rail into the Agents tab and delete NodeSpine
    depends_on:
      - sidebar-modes
      - rail-projection
    size: medium
    description:
      "rail-wiring: RAIL mode shows the tribe lists in rail form at a fixed width. Focus
      stays on the list, titles and newly mounted panels become rail-aware, and runtime
      ticks pause with a catch-up on expand. The info-row nodes chip becomes clickable,
      and NodeSpine, its handler, CSS, and tests are deleted. Includes pilot tests and
      rail goldens."
  - id: banner-glyphs
    title: Align expanded status banners with the rail glyphs
    depends_on:
      - rail-vocabulary
    size: small
    description:
      "banner-glyphs: the TUI's expanded BY_STATUS and BY_MACHINE banners adopt the rail
      bucket glyphs (? chip, ◷, ○), leaving the shared AGENT_STATUS_BUCKET_GLYPHS
      untouched. Also fix the banner width math that uses len() instead of cell_len."
  - id: mode-affordances
    title: Info-row, footer, tooltip, help, palette, and docs affordances
    depends_on:
      - zoom-chrome
      - rail-wiring
    size: medium
    description:
      "mode-affordances: add a clickable reverse ZOOM info-row chip and conditional
      footer entries (Z restore, Ctrl+S expand nodes). Rail rows get hover tooltips
      showing the full expanded row, and the help modal gets a node-rail legend
      generated from the vocabulary. Renames the binding, metadata, and palette labels,
      rewrites the docs, and records the glossary follow-up."
proposed_by: bbugyi200.apollo.2h
create_time: 2026-09-27 17:33:06
status: wip
---

- **PROMPT:**
  [prompts/202609/agents_node_rail_and_zoom.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_node_rail_and_zoom.md)

# Plan: Agents node rail and unmistakable deck zoom

## Context

Today `Ctrl+S` on the Agents tab collapses the node sidebar (`#agent-list-container`,
one `AgentList` per tribe panel) into `NodeSpine`, a 2-cell strip holding only a `»` and
a scroll thumb. `Z` zooms one deck panel. Both leave the left edge looking almost
identical: the `agents_decks_collapsed_single_120x40` and `agents_decks_zoomed_120x40`
goldens differ only by a small `zoom · Z` word in the info row.

Background research, which every phase worker should skim before starting:

```bash
sase artifact read research:202609/agents_sidebar_node_rail_and_zoom/agents_sidebar_node_rail_and_zoom.md "Design context for the node rail epic"
```

The research directory also holds prototype PNGs
(`agents_node_rail_and_zoom_chrome__cld_rail.png`, `..._zoom.png`, `..._hints.png`).
This plan adopts the research's recommended shape and corrects several of its
implementation assumptions that turned out to be false against the current code. Those
corrections are listed in "Design decisions" below.

This is presentation-only work: glyph choice, layout, and Textual chrome. It reads the
existing status-bucket semantics and does **not** cross the `sase-core` boundary. The
`ctrl+s` and `Z` keymap defaults do not change, so `src/sase/default_config.yml` only
needs a wording touch if its comments describe the old spine.

## The design

### Three sidebar presentations

| Mode         | When                            | What the left column shows                                    | Question it answers                                   |
| ------------ | ------------------------------- | ------------------------------------------------------------- | ----------------------------------------------------- |
| **EXPANDED** | default                         | today's full node list                                        | "Which agent is which, and what exactly is it doing?" |
| **RAIL**     | `Ctrl+S` (persisted preference) | the same tribe panels and the same visible rows, 9 cells wide | "Where is everything, what needs me, where am I?"     |
| **HIDDEN**   | while a deck is zoomed (`Z`)    | nothing; the deck starts at the left edge                     | "Let me read this one thing."                         |

`SidebarMode = HIDDEN if is_zoomed(state) else RAIL if state.nodes_collapsed else EXPANDED`.
`nodes_collapsed` becomes purely the persisted `Ctrl+S` rail preference. Zoom never
writes it. The persistence schema (v1) is unchanged; a saved `true` now simply means
"rail".

| From                   | `Ctrl+S`                                       | `Z`                      | `\` / `\|`                                                  |
| ---------------------- | ---------------------------------------------- | ------------------------ | ----------------------------------------------------------- |
| EXPANDED               | → RAIL (saved)                                 | → HIDDEN (zoom)          | split as usual                                              |
| RAIL                   | → EXPANDED (saved)                             | → HIDDEN (zoom)          | split as usual; rail stays                                  |
| HIDDEN (zoomed from X) | **restore the snapshot → X, exactly like `Z`** | restore the snapshot → X | end zoom without restore; sidebar returns at the preference |

The rule is "in zoom, any sidebar key gives your layout back". No split is ever silently
lost. That fixes the leak the research reproduced: `expanded split → Z → \` currently
persists `nodes_collapsed=True`, and `collapsed split → Z → Ctrl+S` drops the split and
flips the preference.

### The rail: a row-for-row minimap

**Geometry.** The rail is 9 cells: 2 tribe-panel border cells, 1 selection gutter, and 6
content cells. The gutter is where the existing `border-left: thick $accent` highlight
bar lands. In rail mode the list has no padding and no scrollbar.

**Position is identity.** Rail row _k_ is expanded row _k_, at the same y and the same
scroll offset. Panel heights and the 1-row gaps between panels are unchanged, because
every `AgentList` row is one line either way. Toggling `Ctrl+S` changes only the width;
your row never moves vertically.

**Every visible row is represented, and folds stay authoritative.** Every tribe panel is
present. A folded group is its one collapsed-banner row. A folded clan or session is its
container row. A collapsed tribe (`h`) is a title-only strip. Folded descendants stay
absent in both densities.

**Glyphs are redundant cues, not labels.** Each row carries exactly one semantic glyph.
Identity comes from:

1. position;
2. the identity header, which always names the selection, so `j`/`k` is never blind;
3. a hover tooltip showing that row's full expanded text;
4. jump hints (`'`) and the node finder (`"`).

**Calm by default.** The rail shows no emoji, runtimes, provider badges, machine chips,
or `⚡ ◌ ↻ ↺`. Color means urgency. _Needs you_ is the rail's only reverse-video chip
outside of transient hint mode.

Content cells (0–5):

```text
agent row, depth 0:   G c c · · p        ▶3   •
agent row, depth 1:   └ G c c · p        └✓   •
agent row, depth 2+:  │ └ G c c p        │└▶
expanded L0 banner:   L ━ ━ ━ ━ ━        s━━━━━   (BY_STATUS: L = bucket glyph)
expanded L1+ banner:  L ─ ─ ─ ─ ─        p─────   (thin rule in the tier color)
folded banner:        ▸ L ━ … n r        ▸b━━4?   (hidden count, then roll-up)
spacer:               blank (kept, so heights match)
```

- `G` is the glyph, or the jump-hint chip.
- `c` is the container member count, capped at `99`.
- `p` is the pip: marked `▪` wins over unread `•` (gold `#FFD700`).
- `L` is the lead: the label's first character as written (projects are lowercase), in
  the banner's tier color.
- `n` is the folded group's top-level agent count, right-aligned to end at cell 4.
- `r` is the folded group's urgency roll-up: the `?` chip if any member needs you, else
  a red `✗` if any failed, else a gold `•` if any is unread, else blank.

Tree guides mirror the expanded list's own connector vocabulary. Ancestor levels are `│`
and the row's own branch is `└`, each in `TREE_DEPTH_COLORS` for its depth. There is no
last-child logic, so a row's cells never depend on its neighbours. Depth is clamped
at 2.

**Vocabulary** (single source of truth: the new rail module):

| Row                                                              | Glyph                           | Style                                                                                                                                |
| ---------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Needs you (bucket `Stopped`: QUESTION and plan/tale/epic review) | `?`                             | `bold #1a1a1a on #FFAF00`, a reverse chip in the same amber as the tribe chip's `S` metric, so it never reads as the gold unread `U` |
| Failed (bucket `Failed`)                                         | `✗`                             | `bold #FF5F5F`                                                                                                                       |
| Running (bucket `Running`)                                       | `▶`                            | `#FFD700` (`RUNNING_COLOR`)                                                                                                          |
| Starting                                                         | `◐`                             | `#87D7FF`                                                                                                                            |
| Queued                                                           | `○`                             | `#5F87FF` (`QUEUED_STATUS_COLOR`)                                                                                                    |
| Waiting                                                          | `◷`                             | `#AF87FF`                                                                                                                            |
| Done, unread / read                                              | `✓`                             | `bold #5FD75F` + pip / `dim #5FD75F`                                                                                                 |
| User-stopped (raw status `STOPPED`, bucket Done)                 | `Ø`                             | `STOPPED_COLOR` `#8787AF`                                                                                                            |
| Monitor / named proc                                             | `⚙`                            | the expanded row's glyph styles (monitor lane colors, settled grey; proc `bold #5FD7FF`)                                             |
| Gate turn                                                        | `⋔`                             | the expanded gate glyph style (accent, settled grey, failed red)                                                                     |
| Workflow / bash-python step / Patch                              | `≡` / `❯` / `❑`                 | the expanded type glyph styles                                                                                                       |
| Clan / session container                                         | its status glyph + member count | count tinted clan `#D75FFF` / session `#00AFFF`                                                                                      |
| Jump hint active                                                 | the hint letter(s)              | `bold #1a1a1a on #FFFF00` chip. A 1-character hint replaces `G`; a 2-character hint also takes the first count cell                  |

Glyph precedence: hint > kind glyph (non-agent nodes) > status glyph. Every glyph above
is 1 cell under Rich `cell_len` and present in the bundled screenshot fonts; the
research checked this. `⏳`, `⚡`, `⤢`, `⧖`, and emoji are banned from the rail.

**Tribe titles** are centered in the top border and at most 5 cells. Pieces in display
order:

1. `[hint]` while panel hints are up;
2. `❖` for whole-panel focus, or `▸` for a collapsed tribe (gold when selected),
   matching today's semantics;
3. the configured tribe icon when its `cell_len ≤ 2`, else the tribe's **uppercase bold
   initial in its identity color**; the merged panel shows `All`;
4. on collapsed tribes only, an urgency mark (the same roll-up as folded banners).

When over budget, drop pieces in this order: mark, urgency, identity. The hint is never
dropped. Existing borders keep working: the gold focused-panel border and the heavy gold
whole-panel outline.

**Overflow.** With no scrollbar, the bottom border shows `▴N▾M` for the rows above and
below the viewport, degrading to `▴▾` and then a single arrow within 5 cells. It is
empty when everything fits.

**No animation, no hover expansion.** The switch is instant; an animated width would
reflow the deck spreads every frame. Rows never expand on hover or focus.

### Zoom: loud and structural

1. **Flush left edge.** There is no rail and no spine. A rail means compact; no left
   column means zoom.
2. **Heavy border in the zoomed deck's own accent.** Main uses `$secondary`, Files
   `green`, Tools `#87D7FF`, Final `#FF87D7`. It is geometry-neutral, so there is no
   reflow. It avoids gold, which already means tribe focus, and `double`, which this TUI
   reserves for modal dialogs.
3. **A reverse-gold `ZOOM` chip** (`bold #1a1a1a on #FFD700`) leads the deck border
   title. `deck_title()` gets its width budget reduced by the chip, and the chip is
   never dropped from the width ladder. A word is more reliable than a maximize glyph
   (`⤢` is tofu in the bundled fonts).
4. **A restore hint leads the bottom border.** From a split it reads
   `◧ 1 of 2 · Z restore`. The glyph names the zoomed half: `◧`/`◨` for left/right,
   `⬒`/`⬓` for top/bottom. From a single deck it reads `Z restore`. Both use the
   configured `zoom_panel` key display; the key segment is omitted if unbound. The
   existing deck-switcher segment shrinks first.
5. **The info row** shows the same reverse `ZOOM` chip, then `Z restore`, then
   `node i/N` (because `j`/`k` still moves the selection). The chip is clickable.
6. **The footer** gets conditional entries: `Z restore` while zoomed and
   `Ctrl+S expand nodes` in rail mode. Both have real on/off conditions, as
   `src/sase/ace/CLAUDE.md`'s footer rule requires.

### Design decisions, including corrections to the research

- **The rail is a paint-time projection inside `AgentList`, not a new widget.** Textual
  8.0.1's `OptionList` routes both height arrangement (`_update_lines`) and painting
  (`_get_option_render`) through `_get_visual(option)`. Overriding it renders a rail
  visual for each existing `Option`. The build, patch, and insert paths,
  `AgentRenderCache`, and incremental refresh all stay untouched, and a toggle is a
  cache clear, not a rebuild. The override must **never** write `option._visual`,
  because the expanded visual stays cached for the toggle back.
- **Correction: patches do not replace `Option`s.** `patch_row` calls
  `replace_option_prompt_at_index`, which mutates the _same_ `Option`'s prompt. A cache
  keyed only by `Option` would serve stale rail cells. The rail cache therefore stores
  `(prompt, visual)` per `Option` and is valid only while
  `entry.prompt is option.prompt`. The strong reference to the prompt prevents id reuse.
- **Rail cells must derive only from inputs that also determine the expanded row's
  prompt.** Those inputs are the agent fields, `_row_render_ctx`, the tier styles, and
  the group row. That rule is what makes prompt identity a sound validator.
- **Correction: the in-place insert path assigns row maps after `install_options`.**
  `try_insert_rows` in `_agent_list_build_patching.py` calls `install_options()`, which
  synchronously runs `_update_lines()` and therefore `_get_visual`, _before_ it assigns
  `_agents`, `_row_entries`, `_row_render_ctx`, and the other maps. Every structural
  path (`build_list`, `try_insert_rows`, `try_remove_rows`, `render_collapsed`)
  therefore ends by calling one `AgentList._rail_rows_changed()` hook. When the rail is
  on, the hook drops the rail cache and calls `_clear_caches()`. `_get_visual` must
  never raise; while the maps are mid-update it falls back to blank cells and emits a
  trace event.
- **Correction: split keys can end a zoom without re-syncing the sidebar.** Today that
  is harmless only because zoom forces `nodes_collapsed=True`. A single
  `_apply_deck_area_state()` choke point in the deck-layout mixin re-syncs the sidebar
  whenever `sidebar_mode` changes.
- **The shared `AGENT_STATUS_BUCKET_GLYPHS` is not changed.** The CLI `sase agents wait`
  live rows and `wait_status_presentation.py` use it. The TUI rail and TUI banners use
  the rail vocabulary module instead.
- **Tooltips cost nothing to format.** In rail mode `option.prompt` is still the full
  expanded row, so the tooltip is that prompt with its right-alignment pad run collapsed
  to two spaces.
- **Runtime ticks pause in rail mode,** since the rail shows no runtimes. Expanding runs
  one catch-up `_patch_agent_runtime_rows(now)` so the first expanded frame is current.
- **NodeSpine is deleted.** The rail supersedes it, and the clickable info-row chip
  replaces its click target.
- **Rejected alternatives:**
  - a separate `NodeRail` widget: two sources of truth for panels, heights, scroll,
    folds, hints, and patches;
  - a group-aggregate rail with its own cursor: breaks "every node" and `j`/`k`
    continuity;
  - a 14-cell rail with name tokens: cryptic, churns as siblings appear, and gives back
    less width;
  - provider emoji: 2 cells and noise;
  - a hover overlay: jitter and accidental activation;
  - a double or gold zoom frame: reads as a modal or as tribe focus;
  - a third "hidden without zoom" key: duplicates `Z` without its chrome.

## Verification rules for every phase

- Run `sase tool run check`, the wrapped `just check`. Do not run `just check-full`.
- Regenerate affected PNG goldens with targeted
  `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`. Then inspect every
  created, removed, and updated golden image before finalizing; generation is not
  approval.
- Keep new or grown modules under the `toobig` line gate. Put the new widget code in its
  own mixin module rather than growing `_agent_list_widget.py`.
- Export any helper you need across modules under a public name, because Symvision flags
  cross-module private imports.
- Hot paths stay cheap, per the TUI performance note:
  - no disk I/O in render or toggle paths;
  - `j`/`k` in rail mode stays under the 16 ms p95 target (spot-check with
    `SASE_TUI_PERF=1`);
  - a toggle is a cache clear plus a repaint and is wrapped in a `tui_trace` span.

## Phase: sidebar-modes

Files:

- `src/sase/ace/tui/widgets/decks/layout.py`
- `src/sase/ace/tui/widgets/_agent_detail_deck_layout.py`
- `src/sase/ace/tui/styles.tcss` (the Agents section around `-nodes-collapsed`)
- `src/sase/ace/tui/widgets/agent_info_panel.py`
- `src/sase/ace/tui/actions/agents/_display_detail_info.py`
- `src/sase/ace/tui/actions/agents/_deck_persistence.py`

Steps:

1. In `decks/layout.py`:
   - add `class SidebarMode(Enum): EXPANDED, RAIL, HIDDEN` and a pure
     `sidebar_mode(state) -> SidebarMode`;
   - `_enter_zoom` stops setting `nodes_collapsed` (update its docstring: zoom snapshots
     the state and shows only the focused panel);
   - `toggle_nodes_collapsed(state)` returns `_exit_zoom(state)` when zoomed, else flips
     the bool;
   - `toggle_split` already drops the snapshot and now naturally keeps the pre-zoom
     preference.
2. In the deck-layout mixin:
   - add `_apply_deck_area_state(new_state)`. It reads the old `sidebar_mode`, calls
     `area.apply_state(new_state)`, and calls `_sync_sidebar_chrome()` iff the mode
     changed;
   - route all seven mixin `area.apply_state(...)` calls through it;
   - `toggle_node_panel()` delegates to `toggle_deck_zoom()` when zoomed, so the
     restore, the focused-panel `refresh_chrome()`, and the save are identical to `Z`;
   - add an `is_node_rail` / `sidebar_mode` property next to `is_nodes_collapsed` and
     `is_deck_zoomed`.
3. Replace `_sync_nodes_collapsed_chrome` with `_sync_sidebar_chrome()` and update its
   callers (`show_deck_in_other_panel` and `_deck_persistence.py`). It:
   - toggles `-nodes-rail` (RAIL) and `-nodes-hidden` (HIDDEN) on `#agents-content`, and
     removes the legacy `-nodes-collapsed`;
   - records the mode on the app as `_agents_sidebar_mode`, which later phases read;
   - refreshes the info row or footer as today;
   - wraps the work in a `tui_trace("agents.sidebar.sync", mode=...)` span.
4. Interim presentation, until `rail-wiring` lands:
   - CSS hides `#agent-list-container` under both `-nodes-rail` and `-nodes-hidden`;
   - the spine is shown only in RAIL, so zoom is already flush-left;
   - focus moves off the hidden list in both modes.
5. Info row:
   - replace the `nodes_collapsed` / `nodes_zoomed` pair in
     `AgentInfoPanel.update_state` (and its stable tuple and caller) with one
     `sidebar_mode`;
   - un-nest the zoom chip: HIDDEN renders `zoom · Z · nodes i/N` with no `Ctrl+S` hint,
     because `Ctrl+S` now also restores; RAIL renders `nodes i/N · Ctrl+S`;
   - `mode-affordances` restyles these later.
6. Tests:
   - update the assertions that zoom sets `nodes_collapsed` in
     `tests/ace/tui/widgets/decks/test_deck_collapse_zoom.py`, `test_deck_layout.py`,
     `test_deck_other_panel.py`, and `tests/ace/tui/test_zoomed_tribe_deck.py`, plus the
     info-chip test;
   - rewrite the pinned `test_collapse_key_ends_zoom_then_toggles` to the new
     restore-exactly rule;
   - add a table test covering every (mode, key) transition above;
   - add the three reproduced leak cases;
   - add persistence tests: zoom is never saved, and the preference survives zoom, split
     keys, and restart;
   - add a pilot test: a split key that ends a zoom from EXPANDED brings the list back.
7. Docs: in `docs/ace.md`, section "Agents Zoom and Node-Panel Collapse" plus the `Z`
   and `Ctrl+S` rows in the key tables, state the three modes and the new
   `Ctrl+S`-in-zoom rule.
8. Goldens: regenerate the ones zoom touches, which now lose the spine:
   - `agents_decks_zoomed_120x40`
   - `agents_named_proc_detail_120x40`
   - `agents_phase_bead_and_plan_context_120x40`
   - any others the targeted run reports

## Phase: rail-vocabulary

Add `src/sase/ace/tui/widgets/_agent_list_render_rail.py`: pure functions only, no
Textual widget imports. It is the single source of truth for everything the rail draws.

- **Constants:**
  - `NODE_RAIL_WIDTH = 9`
  - `RAIL_CONTENT_CELLS = 6`
  - `RAIL_TITLE_CELLS = 5`
  - `RAIL_MAX_DEPTH = 2`
  - `RAIL_COUNT_CAP = 99`
  - `RAIL_NEEDS_YOU_STYLE`
  - `RAIL_HINT_STYLE`
  - `RAIL_BUCKET_GLYPHS`: bucket name → `(glyph, style)` for all seven buckets, per the
    vocabulary table.
- **`rail_agent_cells(agent, ctx, *, depth) -> Text`:**
  - returns exactly `RAIL_CONTENT_CELLS` cells, built from the agent and its
    `_row_render_ctx` entry (`hint_char`, `is_unread`, `is_marked`);
  - classifies the bucket with `agent_status_bucket`; raw `STOPPED` maps to `Ø`;
  - for kinds, extract one pure `row_kind_glyph(agent) -> tuple[str, str] | None` from
    the glyph selection in `_agent_list_render_agent_prefix.py` (~L183–255, including
    the monitor and gate glyph-style helpers). Make the expanded prefix call it too, so
    the two densities cannot drift;
  - container member counts come from the same source as the expanded row's member or
    fold annotation.
- **`rail_banner_cells(group, agents, *, mode, unread, hint, mark_state) -> Text`:**
  - exactly 6 cells, covering expanded L0 (heavy rule), L1+ (thin rule, tier color), and
    folded (fold mark, lead, rule, count, roll-up);
  - BY_STATUS L0 and BY_MACHINE L1 leads use `RAIL_BUCKET_GLYPHS`;
  - a hint chip replaces the fold mark and lead.
- **`rail_urgency(agents, unread) -> Text`:** the shared roll-up used by folded banners
  and collapsed tribe titles.
- **`rail_panel_title(...) -> Text`:** at most `RAIL_TITLE_CELLS`, with the drop order
  from the design. Its inputs are those `agent_panel_border_title` already receives:
  key, hint, selected, collapsed, merged, the tribe icon and color, and the counts
  object behind the `[S R Q W F U D]` chip for urgency.
- **`rail_overflow_subtitle(above, below) -> Text`:** at most 5 cells.
- **`rail_tooltip_text(prompt: Text) -> Text`:** the prompt with its right-alignment pad
  run collapsed to two spaces; spacer and blank prompts give `None`.
- **`RAIL_LEGEND`:** ordered `(glyph Text, meaning)` pairs for the help modal.

Tests (new `tests/ace/tui/widgets/test_agent_list_render_rail.py`):

- **Exhaustiveness.** Every status bucket, raw `STOPPED`, and every node kind, grouping
  mode and level, folded and unfolded, hinted (1 and 2 characters), marked, unread, and
  depth 0–4 must produce Text whose `cell_len` equals `RAIL_CONTENT_CELLS`.
- Titles never exceed `RAIL_TITLE_CELLS`, including emoji or wide tribe icons, long
  names, and 2-character hints.
- The roll-up precedence.
- `row_kind_glyph` parity with the expanded prefix.
- Every rail glyph has `cell_len == 1` and is not one of the banned glyphs.

## Phase: rail-projection

Add `src/sase/ace/tui/widgets/_agent_list_rail_mode.py` with an `AgentListRailMixin`
mixed into `AgentList`. It is not wired to any key yet.

- **`set_rail(enabled: bool)`:**
  - no-op when unchanged;
  - stores the flag, toggles a `-rail` class on the widget, drops the rail cache, and
    calls `_clear_caches()`;
  - wraps the work in `tui_trace("widget.agent_list.set_rail")`.
- **`_get_visual(option)` override:**
  - when the rail is off, returns `super()._get_visual(option)`;
  - when on, maps option → `_option_to_index` → `_row_entries`:
    - banner rows go through `_group_at_row`;
    - agent rows go through `_agents` + `_row_render_ctx` + depth (`agent_tree_depth`);
    - spacers render blank;
  - builds the rail cells from the vocabulary module and returns a cached
    `visualize(...)` result;
  - the cache is per widget, keyed by `Option`, and holds `(prompt, visual)`; an entry
    is valid only while `entry.prompt is option.prompt`;
  - never writes `option._visual` and never raises: inconsistent maps give blank cells
    plus a `widget.agent_list.rail_fallback` trace event.
- **Banner maps.** Record a `_group_at_row` map (every banner row: expanded and
  collapsed), plus per-row banner hint and mark state:
  - in `emit_tree_rows` (`_TreeRows` gains the fields);
  - carried through `build_list`, `try_insert_rows`, and `try_remove_rows`;
  - reset in `render_collapsed` and `AgentListBase.__init__`.
- **`_rail_rows_changed()` hook.** Call it at the end of each structural path, after all
  maps are assigned. When the rail is on it drops the rail cache and calls
  `_clear_caches()`.
- **Overflow subtitle.** `_refresh_rail_overflow()` sets `border_subtitle` from
  `rail_overflow_subtitle(above, below)`, computed from `scroll_offset.y`, the virtual
  height, and the viewport height. Set it only when the text changes, and clear it when
  the rail is off. Call it from a `watch_scroll_y` override (call `super()`), on resize,
  from `set_rail`, and from the rows-changed hook.
- **CSS** keyed on `AgentList.-rail`:
  - no widget padding;
  - `scrollbar-size-vertical: 0; scrollbar-gutter: auto`;
  - option padding `0 0 0 1`, highlighted option padding `0` (the thick accent bar fills
    the gutter);
  - `border-title-align: center; border-subtitle-align: center`;
  - selectors must out-rank `#agent-list-container AgentList` (for example
    `#agent-list-container AgentList.-rail`).

Tests (new `tests/ace/tui/widgets/test_agent_list_rail_mode.py`, mounted in a tiny test
app):

- `set_rail` toggles visuals without a rebuild: the same `Option` objects, highlight,
  and scroll offset, with `option._visual` untouched.
- Every rendered line is 1 row, and the gutter plus 6 content cells line up.
- `patch_row` in rail mode repaints the patched row's glyph (status change), proving the
  prompt-identity validator.
- `try_insert_rows` and `try_remove_rows` in rail mode give correct cells for shifted
  rows.
- Folded descendants are absent, and the row count is identical in both densities.
- The overflow subtitle tracks scrolling.
- **Guard test.** A monkeypatched counter proves Textual still routes `_update_lines`
  and `_get_option_render` through `_get_visual`, so a Textual upgrade that bypasses the
  hook fails loudly.

## Phase: zoom-chrome

Files: `src/sase/ace/tui/widgets/decks/area.py`, `decks/panel.py`,
`decks/panel_chrome.py`, `decks/titles.py`, and `styles.tcss` (the deck-panel section,
~L4027–4069).

1. Add a small frozen
   `ZoomChrome(from_layout: DeckLayout, panel_index: int, panel_count: int)` context.
   `DeckArea.apply_state`:
   - in the zoomed branch, sets `-zoomed` and the `ZoomChrome` on the zoomed panel
     **before** `set_focused()` (which refreshes the chrome), taking `from_layout` and
     `panel_count` from `zoom_snapshot`;
   - in every other branch, clears both from all panels.

   That makes every zoom entry and exit correct by construction, including split keys,
   `exit_zoom_keeping_panels`, and persistence loads.

2. CSS: `.deck-panel.-zoomed.-deck-main { border: heavy $secondary; }` and the
   Files/Tools/Final equivalents in their existing accents. The zoomed panel is always
   the focused one, so there are no dimmed variants. Check specificity against the
   `-unfocused` rules.
3. `deck_title(..., zoomed: bool = False)`:
   - when zoomed, prepend `ZOOM` (`bold #1a1a1a on #FFD700`) plus a space to every rung,
     and pick rungs against `width - chip_width`;
   - the chip is never dropped; below the smallest rung, the chip alone remains;
   - `refresh_chrome()` passes the flag.
4. `deck_subtitle(..., zoom: ZoomChrome | None = None)`:
   - when set, lead with the restore segment: `◧ 1 of 2 · Z restore` from a split, or
     `Z restore` from single;
   - glyph mapping: LEFT_RIGHT gives `◧`/`◨` and TOP_BOTTOM gives `⬒`/`⬓`, by the zoomed
     panel's index;
   - styling: the key in bold accent and the words dim;
   - width ladder: the switcher counts drop first, then the switcher, then the position
     (`Z restore` alone), and only then truncate;
   - read the key display from the `zoom_panel` keymap, omitting the key segment if it
     is unbound.
5. Tests:
   - title and subtitle ladder unit tests at many widths: the chip is always present,
     the budget is respected, and the correct glyph appears per half;
   - area tests: `-zoomed` is on only while zoomed, and is cleared by `Z`, `Ctrl+S`, a
     split key, and "show deck in other panel".
6. Goldens:
   - regenerate `agents_decks_zoomed_120x40` (zoom from a split);
   - add `agents_decks_zoomed_single_120x40`;
   - regenerate any other zoom goldens the run reports.

## Phase: rail-wiring

1. `_sync_sidebar_chrome()` in the RAIL state:
   - calls `set_rail(True)` on every non-retiring `AgentList` in `#agent-list-container`
     (`agent_list_widgets_in`), and `False` otherwise;
   - sets the container's inline `min_width` and `max_width` to `NODE_RAIL_WIDTH`,
     clearing both to `None` otherwise. The existing width writers
     (`_settle_agent_list_container_width`, `on_agent_list_width_changed`) stay
     unchanged and keep tracking the negotiated expanded width underneath, so expanding
     snaps back instantly;
   - refreshes the panel titles;
   - on expand, calls `_settle_agent_list_container_width()` explicitly after the titles
     are restored.

   Keep one clamp site: verify in a pilot test that Textual's `max-width` clamps the
   inline `width`.

2. CSS:
   - `-nodes-rail` no longer hides `#agent-list-container`;
   - `-nodes-hidden` still does;
   - onboarding and the artifact-file viewer keep hiding the whole column in every mode.
3. Focus: only HIDDEN calls `_move_focus_off_hidden_list()`. The list keeps focus in
   RAIL, so every key works unchanged: `j`/`k`, `h`/`l`/`H`/`L`/`z` folds, `'` hints,
   `"` finder, marks, and whole-panel focus.
4. Titles:
   - `_agent_panel_title` in `actions/agents/_display_panel_collection.py` returns
     `rail_panel_title(...)` when `app._agents_sidebar_mode` is RAIL;
   - make it the only title builder: the fallback at `_try_patch_agent_row`
     (`_display_panel_patches.py` ~L481) must go through it too.
5. New tribe panels mounted while the rail is on start in rail mode
   (`_sync_mounted_panel_widgets`, `actions/agents/_display_panel_widgets_mount.py`).
6. `_patch_agent_runtime_rows` (`_display_panel_patches.py`) returns early in RAIL mode.
   The expand path runs one catch-up call.
7. Info row: the `nodes i/N · Ctrl+S` chip becomes clickable, using the existing
   `_search_query_click_span` pattern. It posts a new
   `AgentInfoPanel.SidebarChipClicked` message, and the handler in
   `actions/agents/_deck_layout_actions.py` expands the rail.
8. Delete `widgets/decks/node_spine.py` and everything that exists only for it:
   - the compose in `_app_layout.py`;
   - the `#agent-node-spine` CSS;
   - `on_node_spine_expand_requested`;
   - `_refresh_node_spine` / `_node_spine_selection` in `_display_detail_info.py`;
   - the spine tests in `test_deck_collapse_zoom.py`.

   Then make sure Symvision is clean.

9. Pilot tests:
   - `Ctrl+S` gives a container region width equal to `NODE_RAIL_WIDTH` and restores the
     negotiated width on the second press;
   - focus stays on the list, and `j`/`k` moves the highlight;
   - rows keep their y positions across the toggle, with the same scroll offset;
   - a split in rail keeps the rail;
   - `Z` from rail is flush left, and the rail returns on restore;
   - a panel mounted in rail starts in rail;
   - collapsed tribes show urgency;
   - the chip click expands;
   - the runtime tick is skipped in rail.
10. Goldens:
    - regenerate `agents_decks_collapsed_single_120x40`,
      `agents_decks_collapsed_split_120x40`, and
      `agents_deck_blocks_split_rails_120x40`;
    - add `agents_node_rail_by_project_120x40`, `agents_node_rail_by_status_120x40`,
      `agents_node_rail_jump_hints_120x40`, `agents_node_rail_collapsed_tribes_120x40`,
      and `agents_node_rail_80x24`.

    The by-project scenario should mix these rows:
    - a needs-you agent;
    - a failed unread agent;
    - a running clan expanded to depth 2;
    - done rows, both read and unread;
    - a monitor and a gate row;
    - a folded project banner;
    - a second tribe.

    The collapsed-tribes scenario should include whole-panel focus. Judge the structure
    of each image; do not rubber-stamp. Confirm there that a 5-cell title renders
    unclipped in the 9-cell border; if Textual's border-title padding clips it, lower
    `RAIL_TITLE_CELLS` and keep the drop order.

## Phase: banner-glyphs

- In `src/sase/ace/tui/widgets/_agent_list_render_banner.py` and its styles, the
  expanded BY_STATUS L0 banners and BY_MACHINE L1 banners take their bucket glyph and
  style from `RAIL_BUCKET_GLYPHS`, so one visual language spans both densities:
  - `▲` becomes the amber `?` chip;
  - `⏳` becomes `◷`;
  - `…` becomes `○`.

  Bucket labels ("Stopped") and the shared `src/sase/agent/status_buckets.py` map stay
  unchanged.

- Fix the width math (~L171–189) to use `cell_len` for the prefix, glyph, label, chip,
  and hint (a 2-character hint is 5 cells, not 4), so banners never overrun the suffix
  column or inflate the panel's requested width.
- Tests:
  - exact `cell_len` for every bucket banner, wide-character labels, and 1- and
    2-character hints;
  - the banner glyph equals the rail glyph for each bucket.
- Regenerate the by-status and by-machine goldens the targeted run reports.

## Phase: mode-affordances

1. Info row (`agent_info_panel.py`):
   - HIDDEN renders a reverse `ZOOM` chip (`bold #1a1a1a on #FFD700`), then `Z restore`
     (configured key), then `node i/N`;
   - the chip reuses the click-span pattern and `SidebarChipClicked`, whose handler
     restores the zoom (`toggle_deck_zoom`);
   - RAIL keeps `nodes i/N · Ctrl+S`.
2. Footer: thread `deck_zoomed` and `node_rail` booleans through
   `actions/agents/_display_detail_footer.py`, `widgets/_keybinding_modes.py`
   (`update_agent_bindings`), and `widgets/_keybinding_bindings_agents.py`
   (`_compute_agent_bindings`). Add `(zoom_panel key, "restore")` while zoomed and
   `(toggle_node_panel key, "expand nodes")` in RAIL. Make sure `_sync_sidebar_chrome()`
   refreshes the footer on every mode change.
3. Tooltips: in the rail mixin, override `_on_mouse_move` (call `super()`). While the
   rail is on, set `self.tooltip = rail_tooltip_text(option.prompt)` for the hovered
   option, change-guarded by hovered index, and clear it on leave and on
   `set_rail(False)`. Follow the per-region pattern in `widgets/panel_tab_strip.py`.
4. Help modal (`modals/help_modal/agents_bindings.py`):
   - relabel the `Ctrl+S` row "Toggle node rail" and the `Z` row "Zoom deck (hides
     nodes)";
   - add a "Node Rail" box generated from `RAIL_LEGEND`, keeping to the 57-character box
     rules in `src/sase/ace/CLAUDE.md`;
   - remove the stale "Zoom Modal" section left over from the retired modal.
5. Labels:
   - `bindings.py` `toggle_node_panel` becomes "Toggle Node Rail";
   - `keymaps/metadata.py` becomes "Toggle node rail";
   - the palette entry in `commands/_app_metadata_nav.py` becomes "Toggle node rail",
     with aliases gaining `rail` and `sidebar`;
   - the zoom palette text in `commands/_app_metadata_actions.py` mentions hiding the
     node rail;
   - update the startup notice in `actions/_startup_loads_maintenance.py`;
   - touch up any `default_config.yml` comment that still says "collapse".
6. Docs: rewrite `docs/ace.md` "Agents Zoom and Node-Panel Collapse" as "Agents Zoom and
   Node Rail":
   - the three presentations;
   - the rail anatomy and legend;
   - the tooltip, hints, and finder as identity;
   - the `Ctrl+S`-in-zoom rule;
   - the zoom chrome.

   Also fix the key-table rows (~L1321, ~L1397).

7. Tests:
   - the info-row text for each mode;
   - click spans post the message, and the handler restores or expands;
   - the footer entries appear iff zoomed or in rail;
   - a pilot hover test sets the tooltip to the collapsed prompt;
   - the help-legend line widths;
   - the palette entries.

   Regenerate the goldens whose info row or footer changed.

8. Record a `PROPOSED FOLLOW-UP:` note on this phase's bead. The glossary strand "Node
   Panel" still describes the slim node spine. It should say that `Ctrl+S` toggles a
   fixed-width, row-for-row node rail and that `Z` zoom hides the column entirely. Route
   it through `/sase_memory_write`; do not edit memory files in this epic.
