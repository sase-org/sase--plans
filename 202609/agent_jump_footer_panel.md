---
tier: epic
title: Sticky collapsible jump footer panel on the Agents tab
goal: 'On the Agents tab, every live numbered roster target (family shells, neighbors,
  clan members, tribe members) is listed in a sticky panel at the bottom of the detail
  column, below the LLM Calls/file panel. The panel appears only when digit jumps
  are live. It is collapsed by default to at most two packed rows, where every visible
  number carries a label that unambiguously identifies its target. `.` expands it
  to the complete list, and the first digit of a two-digit jump narrows it to the
  matching candidates. The Agents show/hide non-run agents toggle moves from `.` to
  `I`.

  '
phases:
- id: legend
  title: Jump-map sections, document carrier, and pure legend renderer
  depends_on: []
  size: medium
  description: 'legend: enrich MemberJumpMap with per-target labels and status buckets
    plus ordered roster sections, carry the exact published map on detached documents
    (not hint documents or cheap tribe paints) through a new AgentPromptPanel sink,
    and build the width-responsive collapsed/expanded/narrowed legend renderable with
    its uniqueness-preserving packer and unit tests; no visible change.'
- id: keymap
  title: Dot jump-panel toggle plumbing and the non-run toggle move
  depends_on: []
  size: small
  description: 'keymap: add the inert Agents-only toggle_agent_jump_panel action on
    full_stop, move the Agents show/hide non-run agents toggle to a new toggle_hide_non_run_agents
    action on I, narrow toggle_hide_reverted to Services, and update availability,
    registry pairs, palette, help, docs, and keymap tests.'
- id: panel
  title: Jump panel widget, layout, toggle, narrowing, and visual verification
  depends_on:
  - legend
  - keymap
  size: medium
  description: 'panel: add the AgentJumpPanel widget as the last child of the detail
    column, wire the sink, visibility, dot toggle, bottom-pin handling, and first-digit
    narrowing, add CSS, help and docs, pilot/reliability/visual tests, live screenshot
    review, and the Agents golden refresh.'
proposed_by: bbugyi200.athena.0q0
create_time: 2026-09-23 10:49:52
status: done
bead_id: sase-16y
---

- **PROMPT:** [prompts/202609/agent_jump_footer_panel.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_jump_footer_panel.md)
- **BEAD:** [sase-16y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-16y/README.md)

# Plan: Sticky collapsible jump footer panel on the Agents tab

## Context

The Agents-tab metadata document numbers every row that a digit key can reach. Those
numbered rows appear in four rosters, all rendered by `append_member_roster()` in
`src/sase/ace/tui/widgets/prompt_panel/_member_roster.py`:

- `FAMILY SHELLS` (`_agent_display_family.py`), on family containers and family member
  shells.
- `NEIGHBORS` (`_agent_display_neighbors.py`), on any row that owns a sase agent. It
  includes dismissed neighbors, and a digit on one of those **revives** the agent
  instead of jumping to it.
- `CLAN MEMBERS` (`_agent_display_clan.py`), on clan containers.
- `TRIBE MEMBERS` (`_agent_display_tribe.py`), on a selected whole tribe panel.

`build_header_text()` numbers the family and neighbor rosters from one shared
`MemberJumpNumbering` ladder. The ladder uses `0`–`9` when a document has at most ten
numbered rows and `00`–`99` otherwise; entries beyond fold limits or the 100-number
capacity get an unnumbered `… +N more` tail. Each roster returns a `MemberJumpMap`, and
the builder publishes it into `app._member_jump_maps` (keyed by container identity)
through `member_jump_map_publisher_for(app)`. Digit keys resolve through
`MemberJumpNavigationMixin._handle_member_jump_key()`
(`src/sase/ace/tui/actions/navigation/_member_jump.py`), which reads that registry.
After the first digit of a two-digit number, the keybinding footer shows a `<noun> N▁`
pending indicator.

The problem: these numbers live inside a scrolling document. As soon as the user
scrolls, or picks a layout that hides the metadata body (file-only or LLM-Calls-only),
there is no on-screen answer to "what will `4` do?".

Epic `sase-16k` solved the same problem for the identity header. It split the header out
of the document into a separate sticky `AgentHeaderPanel` above the metadata scroll:
collapsed by default, toggled with `d`, and fed by a sink that
`AgentPromptPanel.update()` calls with the `IdentityHeader` carried on the rendered
document (`AgentHeaderRenderable.identity_header`, found by `find_identity_header()`).
This epic adds the mirror image: a sticky **jump panel** at the bottom of the detail
column that lists every live numbered target. It uses the same carrier/sink architecture
and the same visual language.

Everything here is presentation-only Textual/Rich code plus keymap plumbing, so it stays
in this repo (no `sase-core` change).

## Design

### Key decision: `.` is already taken on the Agents tab

Today `full_stop` is bound to `toggle_hide_reverted`, which on the Agents tab shows and
hides non-run agents (on Services it shows and hides axe commands; Artifacts uses `.`
for `toggle_relation_panel`). A key that toggles the jump panel only when the panel
exists and otherwise toggles non-run agents would be unpredictable, so this epic **moves
the Agents non-run toggle to a new Agents-only action, `toggle_hide_non_run_agents`, on
`I`** (mnemonic: _inactive_). `I` is unbound at the app level today. `.` on Agents then
belongs only to the jump panel: it is a clean no-op when no panel is shown, and Services
keeps `.` for axe commands. Users can rebind either key in config as usual.

### What the user sees

```
│  … scrolling metadata body / LLM Calls panel …                          │
╰─────────────────────────────────────────────────────────────────────────╯
╭─ JUMP · FAMILY SHELLS 00–03 · NEIGHBORS 04–23 ──────────────────────────╮
│  00  --plan ✓       01  --code ▶        02  --review …   03  .1 ✓       │
│  04  sase-16h.2 ▶   05  sase-16h.3 ⏳   06  sase…6h.10 ✓   +17         │
╰────────────────────────────────────────────────────────────── ▴ . more ─╯
```

- **Its own panel, last in the detail column.** It is the final child of
  `#agent-detail-layout`, so it always sits below the metadata scroll, the search
  command line, and the file or LLM Calls (tools) panel when one is visible, in every
  layout (including file-only and LLM-Calls-only).
- **Mirror of the header panel.**
  - A `round` border in the first section's accent at 50% alpha.
  - Horizontal padding `0 2` and `scrollbar-gutter: stable`, so text columns line up
    with the body and the header.
  - The border subtitle (bottom right) names the toggle key from the live keymap:
    `▴ . more` when collapsed and `▾ . less` when expanded. It is omitted when the key
    is unbound. The arrows are inverted relative to the header's `▾ d more` because this
    panel grows **upward**.
  - Like the header's `d`, the key is advertised in the panel's own subtitle rather than
    in the keybinding footer.
- **The border title is the color legend:** `JUMP` (bold, light neutral), then one entry
  per section with numbered targets: `·` + the roster title in its accent
  (`FAMILY SHELLS` `#00AFFF`, `NEIGHBORS` `#00D7AF`, clan and tribe identity colors)
  - its dim number range (`0–5`, `04–23`, or a single number). Chip colors in the body
    map straight back to these titles.
- **Target cells echo the roster rows.** Each cell is:
  - the number chip with the roster's exact style (`bold black on {accent}`, e.g. `04`),
    so the chip matches the one printed in the body;
  - a space, then the roster label in the roster name style;
  - a space, then the status-bucket glyph (`AGENT_STATUS_BUCKET_GLYPHS`) in the roster's
    status color.

  A dismissed neighbor (whose digit revives it) gets the roster's dim `⊘ ` prefix and a
  dim label. Marks, unread markers, and annotations are left out: the panel answers
  "where does this digit go", and the body keeps the detail.

- **Collapsed (default): at most two rows, packed as tightly as clarity allows.**
  - Cells go in a row-major, column-aligned grid (like `ls -x`), so the number chips
    stack in neat columns. The grid uses one row when everything fits.
  - When not every target fits, the last slot becomes a dim `+K` overflow token, where K
    is the number of targets not shown.
  - Labels shrink with a middle ellipsis that keeps the tail. Sibling names in SASE
    differ at the end (`sase-16h.2` / `sase-16h.3`, `--plan` / `--code`), so for example
    `sase…6h.10`.
  - Labels never go below a minimum readable width (`MIN_LABEL_CELLS = 10`, or the full
    label when shorter).
  - Two targets whose full labels differ never display the same truncated text; the
    packer shows fewer targets rather than ambiguous ones. This makes the user's
    requirement concrete: every visible number is followed by a label that uniquely
    identifies its target.
- **Expanded (`.`): every numbered target, no folding.**
  - The same grid uses full labels, with no row limit. A label is clamped only if it
    alone is wider than the panel.
  - When the map has two or more sections, each section opens with a quiet heading
    (`❖ NEIGHBORS · 20` in its accent).
  - Every section that has an unnumbered tail ends with the roster's own dim hint, for
    example `… +3 more neighbors (zz / za to show more)`, so the panel is honest about
    rows that no digit reaches.
  - Dismissed cells append a dim `revive`.
  - Height is `auto` with `max-height: 40%` of the detail column. Past that the panel
    scrolls with the mouse wheel and is not focusable.
- **Type-ahead narrowing for two-digit numbers.** After the first digit of a two-digit
  jump (for example `1`), the panel temporarily shows only the targets numbered
  `10`–`19` with full labels in the grid, whether it is collapsed or expanded.
  - The title becomes `JUMP · 1▁` and the subtitle becomes `esc cancel`.
  - If no target starts with that digit, the panel shows one dim line:
    `no targets start with 1`.
  - When the jump completes or is cancelled, the user's collapsed or expanded view
    returns.

  This is what makes large two-digit rosters usable from a two-row panel.

- **Visibility: shown if and only if the current document has live numbered targets.**
  The panel is hidden in these cases:
  - "No agent selected";
  - nodes without rosters (gates, monitors, workflows, proc shells, attempt-pinned
    views, and so on);
  - rosters whose every entry is unnumbered;
  - the cheap first paint of a tribe document, which has no roster yet, so the panel
    appears in the same paint as the roster it mirrors;
  - **file-hint documents**, where the digits select `[N]` hints instead of jumping.

  It stays visible in every layout and during `,/` metadata search, because the digits
  keep working there.

- **State.** Collapsed/expanded is per session, defaults to collapsed, and holds across
  j/k, tribe focus, and layout changes.
  - Toggling does not rebuild the document, and a bottom-pinned body stays pinned.
  - The panel's own scroll resets to the top when the metadata document identity
    changes.
  - `.` is unavailable (a no-op) while the panel is hidden or the prompt input bar owns
    keys.

### Architecture: the jump map travels with its document

The panel must never disagree with what a digit does. The digit handler reads the
`MemberJumpMap` that the document builder published, so the panel renders **that same
map object**, carried on the same document:

1. **Enrich the map, not a parallel model.**
   - `_MemberJumpTarget` gains `label: str = ""` and `status_bucket: str | None = None`.
   - New frozen dataclass `MemberJumpSection` has `title`, `accent`, `numbered_count`,
     `hidden_count`, and `hidden_hint`.
   - `MemberJumpMap` gains `sections: tuple[MemberJumpSection, ...] = ()`, ordered so
     that section _i_ owns the next `numbered_count` targets. The invariant is that
     `sum(numbered_count) == len(targets)`.
   - `append_member_roster()` fills all of this from the entries it renders.
   - `merged_member_jump_map()` concatenates sections as well as targets.
   - Defaults keep existing constructors and tests working.
2. **Carry it.** `AgentHeaderRenderable` (the detached-document carrier that already
   holds `identity_header`) gains a `member_jump_map` slot and `with_member_jump_map()`.
   In detached mode, `build_header_text()`, `build_clan_detail_text()`, and
   `build_tribe_detail_text()` attach **the exact map object they publish to the
   registry**. The one exception is documents built with a `hint_state`, which carry no
   map, so the panel hides in hint mode. The map is also attached when the publisher is
   `None`, since the publisher and the carrier are independent.
3. **Publish it.** `AgentPromptPanel` gains `attach_member_jump_map_sink(sink | None)`.
   `update()` calls `find_member_jump_map(content)` and passes the result (or `None`) to
   the sink **before** the unchanged-digest early return, next to the existing identity
   sink call. That call runs on the UI thread and in the same call as the body update,
   so the panel, the body chips, and the registry all come from one build.

The renderer is a width-responsive Rich renderable (`__rich_console__` packs the grid at
`options.max_width`, the same approach `AgentHeaderRenderable` uses for responsive
lanes). Textual therefore re-packs on resize with no resize handling.

The work is pure in-memory arithmetic over at most 100 targets:

- at most ~7 label budgets;
- column counts bounded by `width // min_cell_width`;
- layout memoized per width on the renderable instance;
- no disk I/O (`sase/memory/tui_perf.md` rules 1 and 8).

## Phase `legend`: jump-map sections, document carrier, and pure legend renderer

Touch only `src/sase/ace/tui/widgets/prompt_panel/`, one new
`src/sase/ace/tui/widgets/_agent_jump_legend.py`, and tests. With no jump sink attached,
nothing visible changes. Existing tests and every PNG golden must pass unchanged.

1. **Model** (`_member_roster.py`):
   - Add `MemberJumpSection` and the target and map fields described in Architecture.
   - In `append_member_roster()`:
     - set each target's `label=entry.label`;
     - set
       `status_bucket=entry.effective_bucket or status_bucket_for_values(entry.status)`;
     - append one section with the roster `title` and `accent`, the numbered count, the
       `hidden_count` it already computes for the `… +N more` tail, and a hint string
       that matches that tail line (`hidden_tail_label` / `hidden_tail_hint`).
   - A roster that numbers nothing because the shared capacity is spent still records
     its section (with `numbered_count=0`), so the expanded view can report its tail.
   - Export a small public helper for the roster status style (for example
     `member_status_style(bucket)`, backed by `_MEMBER_STATUS_STYLES`) so the renderer
     reuses the roster colors instead of copying them.
2. **Carrier** (`_agent_display_header_renderable.py`, `_identity_header.py`):
   - Add the `member_jump_map` slot and `with_member_jump_map()` to
     `AgentHeaderRenderable`. It does not affect `plain`, `spans`, or the digest.
   - Refactor `find_identity_header()` over a shared carrier walker (the content itself,
     the first carrier among a `Group`'s renderables, or `.renderable`).
   - Add `find_member_jump_map(content)` and
     `MemberJumpMapSink = Callable[[MemberJumpMap | None], None]`.
3. **Builders**, detached mode only:
   - `build_header_text()`: compute
     `merged_member_jump_map(agent.identity, family_map, neighbors_map)` once whenever
     either roster rendered. Publish that object when a publisher exists, and attach it
     to the returned carrier unless `hint_state` is set.
   - `build_clan_detail_text()`: attach the clan roster map (non-hint only).
   - `build_tribe_detail_text()`: have `_append_tribe_body()` return its map and attach
     it on the non-`cheap` detached path. The cheap path carries none.
   - Audit every detached return path in these builders and in the render mixins that
     wrap them (`_agent_display.py`, `_agent_display_render.py`, the family/proc-shell
     wrappers). A carrier built for a roster document must not be rebuilt or re-wrapped
     in a way that drops the map; `find_member_jump_map` must find it inside the final
     `Group`.
4. **Panel plumbing** (`prompt_panel/__init__.py`): add `attach_member_jump_map_sink()`,
   store `_jump_map_last_published`, and call the sink in `update()` right after the
   identity sink, before the digest early return.
5. **Pure renderer** (`src/sase/ace/tui/widgets/_agent_jump_legend.py`):
   - `JumpLegendRenderable(jump_map, *, mode)` where `mode` is `collapsed`, `expanded`,
     or a narrowing prefix digit.
   - `jump_legend_title(jump_map, *, prefix) -> Text` and
     `jump_legend_border_accent(jump_map) -> str` (the first section accent, with a
     neutral fallback).
   - Collapsed packing algorithm:
     - For each candidate label budget in descending order (the longest label, then 32,
       24, 18, 14, 12, 10, keeping values ≥ `MIN_LABEL_CELLS` and ≤ the longest label),
       skip any budget at which two _distinct_ full labels truncate to identical text.
     - For each remaining budget, find the largest column count _C_ whose row-major,
       column-aligned grid of `min(n, 2C)` slots fits the width (per-column max cell
       width, two-space gutters). When `n > 2C`, the last slot is the `+K` token with
       `K = n - (2C - 1)`.
     - Choose the budget that shows the most targets; ties go to the larger budget.
     - If even one minimum cell cannot fit, show a single cell clamped to the width.
   - Measure and truncate by terminal cells (`rich.cells.cell_len`,
     `Text.truncate`/`set_cell_size`), never by `len()`.
   - Middle ellipsis keeps roughly 40% head and 60% tail.
   - Expanded and narrowed modes use the same grid routine with full labels and no row
     limit. Expanded adds section headings (only when ≥ 2 sections) and hidden-tail
     lines. Narrowed filters by `number.startswith(prefix)`, has no headings, and shows
     the empty-prefix line when nothing matches.
   - Give the renderable a stable content digest (for example a `plain`/`spans` view or
     a `content_digest` over targets, mode, and prefix) so `renderable_content_digest()`
     can skip unchanged paints.
6. **Tests** (new, for example `tests/ace/tui/widgets/test_member_jump_sections.py` and
   `tests/ace/tui/widgets/test_agent_jump_legend.py`):
   - For each roster kind (family container, family member shell, neighbors with a
     dismissed entry, clan, tribe full): the map carries correct labels, buckets, and
     sections; `sum(numbered_count) == len(targets)`; shared numbering across family and
     neighbors produces two sections in order; the neighbor fold limit and 100-slot
     capacity produce the right `hidden_count` and hint.
   - Detached builders attach the **same object** they publish (capture it with a fake
     publisher and assert `is`). There is no map for tribe cheap paints, hint-mode
     documents, clans without members, or non-roster nodes.
   - A mounted `AgentPromptPanel` with a fake jump sink receives the map on each update,
     `None` for empty documents, and is called before the digest early return (mirror
     `test_panel_sink_receives_identity_before_digest_return`).
   - Renderer, at widths 30/48/64/80/120/200:
     - collapsed output is at most two rows;
     - every shown chip number is followed by its target's (possibly truncated) label;
     - visible truncated labels are unique;
     - shown count plus K equals n;
     - the one-row case works, and ties prefer the larger budget;
     - two-digit chips are handled;
     - wide characters measure correctly;
     - dismissed marker;
     - expanded shows every target at every width, with headings and tails;
     - narrowed filtering and the empty-prefix line;
     - title ranges for single and multiple sections.
   - Non-detached documents are unchanged (existing tests cover this; add a spot check
     if none does).
   - If `symvision` flags a public symbol that only the `panel` phase consumes (for
     example the renderable, title helper, or sink attach method), add an
     `--epic-symbol <epic_bead_id>(<symbol>)` entry to the `Justfile` Symvision
     invocation per `sase/memory/symvision.md`.

## Phase `keymap`: `.` jump-panel toggle plumbing and the non-run toggle move

Add the app action `toggle_agent_jump_panel` ("Toggle Jump Panel"), default `full_stop`.
It stays inert until the `panel` phase lands (the `toggle_agent_header` pattern), and
this phase also moves the Agents non-run toggle to `I`. No feature flag is needed:
keymaps are user config, and no old branch has to stay reachable.

- `src/sase/default_config.yml`, `keymaps.app`:
  - Next to `toggle_agent_header`, add `toggle_agent_jump_panel: "full_stop"` with a
    comment: Agents-only; shares `.` with `toggle_hide_reverted` (Services) and
    `toggle_relation_panel` (Artifacts).
  - Add `toggle_hide_non_run_agents: "I"` with a comment explaining the move.
  - Update the `toggle_hide_reverted` and `toggle_relation_panel` comments: `.` is now
    Services axe commands / Artifacts relations / Agents jump panel.
- Keymap tables:
  - add both fields to `keymaps/app_keymaps.py` and entries to `keymaps/metadata.py`;
  - in `bindings.py`, add
    `Binding("full_stop", "toggle_agent_jump_panel", "Toggle Jump Panel", show=False)`
    and
    `Binding("I", "toggle_hide_non_run_agents", "Show/Hide Non-Run Agents", show=False)`.
- `keymaps/registry.py` `_CONTEXTUAL_APP_DUPLICATES`: add the tab-disjoint pairs
  `{toggle_agent_jump_panel, toggle_hide_reverted}` and
  `{toggle_agent_jump_panel, toggle_relation_panel}`, each with a short comment.
- `_app_action_availability.py` `check_app_action`:
  - `toggle_agent_jump_panel` is available only when `current_tab == "agents"`, the
    prompt input does not own keys, and `AgentDetail.jump_panel_toggle_available()`
    exists and returns true. Look the method up with `getattr` so the action stays
    unavailable until `panel` adds it.
  - `toggle_hide_reverted` becomes available only on the Services tab.
  - `toggle_hide_non_run_agents` is available only on Agents.
- Actions:
  - Add `action_toggle_hide_non_run_agents` (Agents only) that calls the existing
    `_toggle_hide_non_run_agents()` (`actions/agents/_filter_actions.py`).
  - Delete the now-unreachable Agents branch from `action_toggle_hide_reverted`
    (`actions/patch/_core.py`) and update its docstring.
  - Add `action_toggle_agent_jump_panel` on `AgentPanelDetailMixin`
    (`actions/agents/_panel_detail.py`), which calls
    `agent_detail.toggle_jump_panel_expanded()` when present.
- Command palette:
  - Register `toggle_agent_jump_panel` in `commands/_app_metadata_actions.py` and gate
    it in `commands/_availability_agents.py` with the same availability rules.
  - Narrow the `toggle_hide_reverted` entry in `commands/_app_metadata_display.py` from
    Agents+Services to Services, and add a `toggle_hide_non_run_agents` entry for
    Agents.
- Help modal (`modals/help_modal/agents_bindings.py`):
  - Move the "Show/hide non-run agents" row from "General" into the Agents rows using
    `a.toggle_hide_non_run_agents`.
  - Keep a Services-accurate row for `.` wherever Services bindings are listed. Relabel
    it "Show/hide axe commands" if it stays in "General".
  - Leave the jump-panel row to `panel`, so nothing user-facing advertises `.` on Agents
    early.
- Docs:
  - `docs/ace.md`: in the Global Keybindings `.` row, drop "Agents: show/hide non-run
    agents". Add an `I` row to "Keybindings: Agents Tab".
  - `docs/configuration.md`: document `toggle_hide_non_run_agents`, and add
    `toggle_agent_jump_panel` to the shared-`.` row of the shared-key allowlist table.
- Tests:
  - Extend `tests/test_keymaps_defaults.py`, `tests/test_keymaps_app_bindings.py`, and
    `tests/test_keymaps_validation.py`. A user override of `.` must not conflict with
    the tab-disjoint actions.
  - Extend `tests/test_command_availability_agents_actions.py` and
    `tests/test_command_palette_wiring.py`.
  - Update `tests/ace/tui/test_artifacts_relation_key_resolution.py`:
    - on Agents, `full_stop` resolves to nothing until the panel exists, then to
      `toggle_agent_jump_panel`;
    - Services still resolves to `toggle_hide_reverted`;
    - Artifacts still resolves to `toggle_relation_panel`.
  - Add a pilot test that `I` toggles `hide_non_run_agents` on Agents and `.` no longer
    does.

## Phase `panel`: widget, layout, toggle, narrowing, and visual verification

1. **Widget** `AgentJumpPanel(VerticalScroll)`:
   - Lives in the new `src/sase/ace/tui/widgets/agent_jump_panel.py`, with
     `can_focus = False`, and composes one `Static(id="agent-jump-content")`.
   - API:
     - `show_jump_map(jump_map | None)`;
     - `set_pending_prefix(prefix | None)`;
     - `toggle_expanded()`;
     - `has_targets`;
     - `is_expanded`.
   - The panel renders `JumpLegendRenderable` in the effective mode: the narrowing
     prefix if one is set, else expanded or collapsed.
   - Title, subtitle, and border come from the `legend` helpers. The subtitle key is
     `key_display_name` of the live `toggle_agent_jump_panel` binding, falling back to
     `.` and omitted when unbound. The border is set as
     `("round", Color.parse(accent).with_alpha(0.5))` only when the accent changes;
     confirm the color visually.
   - Unchanged paints are skipped with a digest over the map content, mode, prefix,
     title, and subtitle, so 5 s slow-tool ticks and idle refreshes stay free.
2. **`AgentDetail`**:
   - Put the wiring in a new mixin module (for example `_agent_detail_jump.py`) so
     `agent_detail.py` stays lean.
   - Compose the panel (`id="agent-jump-panel"`, initially `hidden`) as the **last**
     child of `#agent-detail-layout`, after `#agent-llm-calls-scroll`.
   - On mount, attach the prompt panel's jump sink to `_on_member_jump_map(map)`, which
     shows the map and then syncs visibility.
   - Add `_sync_jump_panel_visibility()`: hidden unless `has_targets`. Like the header,
     visibility does not depend on the layout, so syncing on publish is enough.
   - Add `jump_panel_toggle_available()` (panel shown) and
     `toggle_jump_panel_expanded()`. The latter flips the state, repaints from the
     stored map, and reapplies a bottom-pinned prompt panel's pin, like
     `toggle_header_expanded()`.
   - Add `set_jump_panel_prefix(prefix | None)`.
   - Reset the panel's scroll to the top when the metadata identity changes (the same
     hooks the header uses).
3. **Narrowing hooks** (`actions/navigation/_member_jump.py`):
   - `_update_member_jump_footer(first_digit)` also calls
     `set_jump_panel_prefix(first_digit)`.
   - `_cancel_member_jump_pending()` always clears the prefix, including the
     `refresh_footer=False` and modal-screen paths.
   - Grep every assignment to `_member_jump_pending_digit` and route each clear through
     that method, so the panel can never stay narrowed.
4. **CSS** (`src/sase/ace/tui/styles.tcss`, next to `#agent-header-panel`):
   ```
   #agent-jump-panel { height: auto; max-height: 40%; border: round $secondary;
     padding: 0 2; scrollbar-gutter: stable; border-title-align: left;
     border-subtitle-align: right; }
   #agent-jump-panel.hidden { display: none; }
   #agent-jump-content { width: 100%; height: auto; }
   ```
   The fr-sized scrolls (`.expanded`, `layout-priority`, `layout-equal`, the default
   3fr/7fr) already share the remaining height with auto-height siblings. Verify each of
   the five `DetailLayoutMode`s.
5. **Help and docs**:
   - Help modal: add `(d(a.toggle_agent_jump_panel), "Expand / collapse jump panel")`
     next to the header row in `modals/help_modal/agents_bindings.py`.
   - `docs/ace.md`:
     - add the `.` row to "Keybindings: Agents Tab";
     - extend the `0`–`9` row and the numbered-roster paragraph to mention the panel and
       first-digit narrowing;
     - set the Global Keybindings `.` row to "Agents: expand/collapse the jump panel";
     - add a **Jump panel** bullet after **Header panel** in "Agents Tab Metadata
       Panel", covering placement below the LLM Calls/file panel, the collapsed and
       expanded contents, the uniqueness guarantee, the title legend, dismissed-revive
       cells, narrowing, visibility rules (including hint mode), and per-session state.
6. **Tests**:
   - Pilot tests (new `tests/ace/tui/widgets/test_agent_jump_panel.py`, using the
     `_DetailApp` pattern from `test_agent_header_panel.py`):
     - hidden for empty, non-roster, and all-unnumbered documents;
     - shown for family, neighbors, clan, and tribe with the right title;
     - collapsed at no more than two rows by default;
     - `.` expands to every target and back;
     - the state persists across j/k and tribe focus;
     - the panel's `region.y` is at or below the bottom of the visible LLM Calls or file
       scroll, and its bottom is at or below `detail.region.bottom`, in every layout;
     - visible during `,/` search;
     - hidden while a hint document is shown and back afterwards;
     - a bottom-pinned body stays pinned across a toggle;
     - `.` does nothing while the prompt input owns keys or the panel is hidden;
     - with a two-digit roster, the first digit narrows the panel and `Esc`, the second
       digit, and a non-digit key each restore it.
   - Reliability test: for each roster document kind, pressing any number shown in the
     panel lands on (or revives) that cell's target through the real
     `_handle_member_jump_key` path, reusing
     `tests/ace/tui/_member_jump_navigation_helpers.py`.
   - Visual tests (new
     `tests/ace/tui/visual/test_ace_png_snapshots_agents_jump_panel.py`):
     - collapsed single section;
     - collapsed family plus neighbors with two-digit overflow;
     - expanded;
     - narrowed after a first digit;
     - the LLM-Calls layout showing the panel below the tools panel.
   - Update existing Agents pilot and visual wait conditions or geometry assertions that
     assume the prompt or secondary scroll reaches the column bottom (for example the
     `SECONDARY_ONLY` and equal-layout assertions in
     `test_ace_png_snapshots_agents_panel_layout.py`).
   - Once `panel` consumes them, remove any `--epic-symbol` entries that `legend` added.
7. **Visual verification** (read `sase/memory/tui_screenshot.md` first):
   - Capture live screenshots with `sase screenshot`, using a `--keep` session driven
     through:
     - a family container;
     - a family member shell;
     - an agent with many neighbors (two-digit numbering);
     - a clan and a tribe panel;
     - `.` and a first digit;
     - the LLM Calls and file layouts;
     - a narrow terminal.
   - Inspect each PNG for alignment with the body and header text columns, chip and
     title colors, truncation quality, and the up/down symmetry with the header. Polish
     until it is clean; tune `MIN_LABEL_CELLS`, the gutters, the border alpha, and
     `max-height` here if needed.
   - Then run the full `just fix-tui-screenshots` through `/sase_monitor`. Every Agents
     golden whose selected node has a numbered roster gains the panel. Inspect every
     creation, removal, and update group per the golden-maintenance rules before
     finalizing.

## Out of scope

- Clicking a cell to jump.
- Persisting the expanded state across sessions.
- Showing the panel inside the `Z` zoom modal: digits do not act while a modal is open,
  and the zoomed document keeps its inline numbered rosters.
- A keybinding-footer entry for `.`.

## Verification

Each phase runs `just check` (via `sase tool run check`) before finishing.

- `legend` and `keymap` must leave every PNG golden unchanged.
- `panel` owns the golden refresh and the live screenshot review.
- Phase dependencies:
  - `legend` and `keymap` are independent and can run in parallel.
  - `panel` depends on both.
