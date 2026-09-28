---
tier: tale
title: Labeled node rail for the Agents tab
goal:
  The collapsed Agents sidebar shows every agent's and group's name in a compact,
  stable, fixed-width rail.
size: medium
proposed_by: bbugyi200.apollo.2l
create_time: 2026-09-28 07:42:53
status: wip
---

# Plan: A labeled node rail for the Agents tab

## Context

`Ctrl+S` on the Agents tab switches the node sidebar into the **node rail** (epic
`sase-1bn`, plan `plan:202609/agents_node_rail_and_zoom.md`). The rail is a paint-time
projection inside `AgentList`: the same tribe panels and the same visible rows, row for
row, at a fixed 9-cell width. Each row carries a single glyph and no text. In practice
that went too far. A column of `✓`s and `▶`s answers "where is everything and what
needs me?" but not "which agent is that?", so the user has to hover, move the selection,
or expand the sidebar just to read a name.

The user asked for:

1. every sase agent's name and every group's name (for example `Running`) to be visible
   while the sidebar is collapsed;
2. any other change that clearly helps without costing much width;
3. a design that is intuitive, reliable, and beautiful.

The epic's architecture stays; only what a rail row _draws_ changes:

- the `_get_visual` projection, never writing `option._visual`, so a toggle is a cache
  clear rather than a rebuild;
- position is identity: rail row _k_ is expanded row _k_;
- calm by default: no runtimes, emoji, provider or machine badges;
- color means urgency;
- `Z` zoom (HIDDEN mode) is unchanged.

The epic explicitly rejected "a 14-cell rail with name tokens" as cryptic and churning.
This plan revisits that with a different answer. It shows real names, makes nested names
short by writing them **relative to their parent row**, and uses a fixed width, so
nothing churns.

This is presentation-only work. It does not cross the `sase-core` boundary, and no
keymap or config default changes.

### Evidence gathered while planning

- Real agent names from the last 7 days: top-level names are mostly 2–16 cells (`2l`,
  `bob-cli-29`, `sase-1bn.land`, `sase-1aq.10.7.5`); a few `dispatch-<hash>` names run
  to 40+. Nested names extend their parent's name at a `.` or `--` boundary
  (`research.9` → `research.9.cld`, `1h` → `1h--plan`, `sase-1aq.10.7.5` →
  `sase-1aq.10.7.5.land`). All 67 session members in the sample extend their session's
  name this way; written relative to it, each shrinks to 3–6 cells.
- Live bug, visible in the user's screenshot and in a live capture: session containers
  render a member count of `0` (`✓ 0`). The expanded row for the same node reads `×4`.
  `_container_member_count` counts `followup_agents` instead of the source behind the
  expanded `×N` fold annotation, contrary to what the epic specified.
- The expanded name-root banner prefix is already `▸` (`_NAME_ROOT_BRANCH_GLYPH`), so
  the rail's `▸` fold mark on banners collides with it. The rail's `▸` fold mark also
  sits next to the `▶` running glyph.

## The design

### Geometry

`NODE_RAIL_WIDTH = 22`: 2 border cells, 1 selection gutter (where the thick accent bar
lands), and **19 content cells**. Derive the other constants from it:

- `RAIL_CONTENT_CELLS = NODE_RAIL_WIDTH - 3`
- `RAIL_TITLE_CELLS = NODE_RAIL_WIDTH - 4`

The rail is a bit over a third of the expanded minimum (60). It leaves 98 of 120 columns
to the decks.

The width is **fixed**, not fitted to content. The rail is a stable minimap: the deck
edge never jumps when an agent with a longer name launches, and a deck reflow re-wraps
every card. Long names use a middle ellipsis. Relative names keep nested rows short. The
identity header and the hover tooltip always carry the full name.

### Row anatomy (19 content cells)

```text
agent row:       [guides][G] [name…………]   [×N] [p]
                  0–3     1 1   elided    right-aligned count ends at cell 16; pip at 18
unfolded banner: [prefix][label] [rule━━━━━━━━━━━━━━]     (rule fills to the edge)
folded banner:   [prefix][label] [rule━━] [×N] [r]        (count column = agent rows')
tribe title:     [hint ][mark ][icon ]@label[ ×N][ r]     left-aligned, ≤ 18 cells
overflow:        ▴3 ▾12                                    right-aligned bottom border
```

Mockup (BY_STATUS, two tribes, one collapsed tribe):

```text
╭ ⌂ @default ────────╮
│ ▶ Running ━━━━━━━━━│
│▌▶ 2l           ×4  │   ← selected; folded session container with 4 members
│                    │
│ ◷ Waiting ━━━━━━━━━│
│ ◷ 2k               │
│ ✓ Done ━━━━━━━━━━━━│
│ ✓ 2j           ×7  │   ← dim: done and read
│ ✓ 2i           ×7 •│   ← unread pip
╰────────────────────╯
╭ ▲ @epic ───────────╮
│ ? Stopped ━━━━━━━━━│
│ ? research.b       │   ← needs-you chip on a clan (purple name)
│ └✗ .cld           •│   ← relative name: research.b.cld
│ ▶ Running ━━━━━━━━━│
│ ▶ bob-cli-29       │   ← clan, unfolded, so no count; members follow
│ └✓ .1             •│   ← bob-cli-29.1
│ └▶ .2              │
│ │└⚙ --mon          │   ← bob-cli-29.2--mon
│ ◷ dispa…c674c69ae  │   ← middle-elided long name
│ ✓ Done ━━━━━━━ ×8 •│   ← folded group: 8 inside, unread roll-up
╰──────────────── ▾4 ╯
╭ ▸ ◆ @research ×14 ?╮   ← collapsed tribe: fold mark, count, urgency
╰────────────────────╯
```

### Vocabulary

**Glyph cell `G`.** Unchanged:

- hint chip, else kind glyph (`⚙ ⋔ ≡ ❯ ❑` and the proc gear), else status glyph
  (`? ✗ ▶ ◐ ○ ◷ ✓ Ø`);
- 1-character hints replace `G`; 2-character hints replace `G` and the following space.
  The name column never moves.

**Tree guides.** `│` per ancestor level and `└` for the row's own branch, each in
`TREE_DEPTH_COLORS`. `RAIL_MAX_DEPTH` rises from 2 to 3: guides are 1 cell per level,
and relative names are short. Rows deeper than 3 draw depth-3 guides.

**Names are the expanded row's identity token** (new pure `rail_row_name(agent)`).
Resolution order:

1. Named procs and bash/python workflow steps: their left title (`agent_tree_title`).
   Their trailing token is a language tag, not a name.
2. Clan containers: `agent.display_name`, in the clan color.
3. `presented_agent_name or agent_name`. Session containers use the session color;
   everything else uses the gold name-annotation color.
4. Fallback: `agent_tree_title(agent) or agent.display_name` in the expanded title teal
   `#00D7AF`. An empty result leaves the name blank.

Name colors match what the expanded row uses for the same token, so identity color is
consistent across densities.

**Attention dimming.** A row that is settled and read renders its name `dim`. Settled
and read means the `Done` bucket and not unread, or raw `STOPPED`. This mirrors the
existing dim-`✓` rule. Running, waiting, failed, needs-you, and unread rows stay at full
strength, so the eye lands on what matters.

**Relative names.** A tree-child row shows its name relative to its anchor: the row its
guides point to.

- If the name starts with the anchor's full name and the remainder starts with `.` or
  `--` and is longer than that separator, show the remainder (`.cld`, `--plan`).
- Otherwise show the full name.
- This is the same rule as the jump legend's `_neighbor_label`. Extract one pure public
  helper into a models module, for example `relative_agent_label(name, prefix)`, and
  make `_neighbor_label` delegate to it, so the two surfaces cannot drift.

How the anchor is chosen:

- The anchor is the nearest preceding agent row, since the last banner or spacer, at
  depth `min(depth, RAIL_MAX_DEPTH) - 1`.
- For depth ≤ 3 that is the true parent.
- For deeper rows it is the ancestor the clamped guides visually point to. A depth-4 row
  under `.7` then reads `.5.2`, not a misleading sibling `.2`.
- Top-level rows have no anchor.

**Elision.** Names and labels use a cell-aware middle ellipsis that keeps about 40% of
the head and 60% of the tail. Sase names differ at the end (`.land`, `--plan`, `.cld`).

**Container count `×N`.**

- Shown only when the expanded row shows `×N`, that is, when the container is folded.
  The value is the same total.
- Right-aligned in a shared count column that ends at cell 16. Capped at `×99`.
- Tinted dim in the container color: clan purple, session blue, otherwise dim.
- Unfolded containers show no count, because their members are the following rows.
- This fixes the `0` bug.

**Pip (cell 18).** Marked `▪` wins over unread `•`. Unchanged.

**Banners** mirror the expanded banner's own prefix vocabulary:

- STANDARD L0 `▌`;
- middle tier `▎`;
- name-root `▸`;
- BY_STATUS L0 bucket glyph;
- BY_MACHINE L1 `▎` plus bucket glyph;
- BY_DATE L0 has no prefix.

Then comes the `banner_label`, then the heavy (L0) or light rule in the expanded rule
style. The rule is always at least 1 cell, so a banner always reads as a banner.

**Folded banners** keep their prefix and add:

- the top-level member count `×N` in the shared count column;
- the 1-cell urgency roll-up in the pip column: `?` chip, then red `✗`, then gold `•`,
  else blank.

**`×N` means "folded, N inside"** everywhere in the rail: containers, banners, and
collapsed tribes. The `▸` fold mark on banners is retired, which removes both glyph
collisions.

**Hinted banners.** A hint chip plus a space replaces the prefix, or is prepended when
the prefix is empty. The label budget shrinks to match.

**Tribe titles** are left-aligned, as expanded titles are. Pieces:

- `[hint]` chip;
- the `❖` whole-panel-focus or `▸` collapsed-tribe mark, matching the expanded title's
  marks;
- the tribe icon when `cell_len ≤ 2`;
- `agent_panel_label(key)` (for example `@default`) in the tribe identity style, or
  `All agents` for the merged panel;
- on collapsed tribes only, ` ×{lane_count}` in dim, then the urgency roll-up.

The budget is `RAIL_TITLE_CELLS`:

- never dropped: the hint, the label (middle-elided, minimum 3 cells), and the roll-up;
- dropped in this order when over budget: mark, then icon, then `×N`;
- after drops, the label is elided to the remaining budget.

**Overflow subtitle.** `▴N ▾M` (with a space), within `RAIL_TITLE_CELLS`, degrading to
`▴▾` and then to a single arrow. It is right-aligned, like the deck panels' subtitles.

**Unchanged:** the hover tooltip (full expanded text), jump hints, the node finder,
runtime-tick pause, and the footer and info-row affordances.

### Reliability: cache soundness with anchors

A child row's relative name depends on its anchor row, which is not an input to the
child's own prompt. Validate that dependency explicitly:

- **Cache entries.** `_rail_visual_cache` entries become
  `(prompt, anchor_prompt, visual)`. An entry is valid only while
  `entry.prompt is option.prompt` and `entry.anchor_prompt` is the anchor option's
  current prompt (or both are `None`).
  - A patch that re-renders an anchor row therefore busts its children's cells.
  - Structural paths already call `_rail_rows_changed()`.
- **Anchor map.** `_rail_anchor_rows: dict[int, int] | None` maps row index to anchor
  row index.
  - It is built lazily in one O(rows) pass over `_row_entries` with a depth stack that
    resets at banner and spacer rows.
  - `_rail_rows_changed()` (unconditionally) and `set_rail()` set it to `None`.
  - Building it must never raise. On inconsistent maps it yields "no anchor"; the
    rows-changed hook repaints once the maps land.
- **Count source.** Extract
  `folded_member_total(agent, fold_counts, parents_with_visible_children) -> int | None`
  from `compute_fold_annotation` in `src/sase/ace/tui/widgets/_agent_list_helpers.py`.
  - It returns the total exactly when the folded ` ×N` form renders, honoring the
    anonymous-single-child exception.
  - `compute_fold_annotation`'s folded branch uses it.
  - `agent_row_context` stores it as `ctx["folded_total"]`.
  - It derives from the same inputs as the prompt, so prompt identity stays a sound
    validator. All three ctx creation sites go through `agent_row_context`.

## Implementation

Files:

- `src/sase/ace/tui/widgets/_agent_list_render_rail.py`
- a new `src/sase/ace/tui/widgets/_agent_list_render_rail_names.py`
- `src/sase/ace/tui/widgets/_agent_list_rail_mode.py`
- `src/sase/ace/tui/widgets/_agent_list_render_banner.py`
- `src/sase/ace/tui/widgets/_agent_list_helpers.py`
- `src/sase/ace/tui/widgets/_agent_list_build_rows.py`
- a models module for the relative-label helper
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_neighbors.py`
- `src/sase/ace/tui/actions/agents/_display_panel_collection.py`
- `src/sase/ace/tui/styles.tcss` (the `AgentList.-rail` block)
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py` (via `RAIL_LEGEND`)
- `docs/ace.md`

Steps:

1. **Names module.** Create `_agent_list_render_rail_names.py` with pure functions and
   no Textual imports:
   - `rail_row_name(agent) -> tuple[str, str]`;
   - `rail_middle_elide(text, budget) -> str`, which is cell-aware;
   - the dimming predicate.

   Keep `_agent_list_render_rail.py` under the `toobig` 700-line tier by putting new
   helpers here.

2. **Relative label helper.** Add `relative_agent_label(name, prefix)` in a models
   module and make `_neighbor_label` delegate to it. Its behavior stays identical.
3. **Shared banner prefix.** Extract a pure
   `banner_prefix_segments(group, agents, mode)` from `format_banner_option`. The
   expanded banner calls it, and so does the rail, so the two densities cannot drift,
   just as `row_kind_glyph` already works.
4. **Fold total.** Add `folded_member_total` and `ctx["folded_total"]` as described in
   "Reliability" above.
5. **Rail vocabulary.** Rewrite the builders in `_agent_list_render_rail.py` to the
   anatomy above, all returning exactly `RAIL_CONTENT_CELLS` cells:
   - `rail_agent_cells(agent, ctx, *, depth, anchor_name=None)`;
   - `rail_banner_cells(...)`;
   - `rail_panel_title(...)`, which gains lane-count use for collapsed tribes;
   - `rail_overflow_subtitle(...)`.

   Other changes in the same module:
   - update the constants;
   - delete `_container_member_count`;
   - update `RAIL_LEGEND`: replace the `▸ folded group` entry with `×N` "folded — N
     inside", and add a relative-name entry (`.x` "name continues its parent's") and a
     dim-name entry ("done and read").

6. **Projection.** In `_agent_list_rail_mode.py`:
   - add the lazy anchor map;
   - pass `anchor_name` (the anchor agent's full `rail_row_name`) into
     `rail_agent_cells`;
   - switch the cache to the 3-tuple validation;
   - invalidate the anchor map in `_rail_rows_changed()` and `set_rail()`;
   - keep the blank-cells fallback and the `widget.agent_list.rail_fallback` trace.
7. **CSS.** In the `AgentList.-rail` block, set `border-title-align: left` and
   `border-subtitle-align: right`. The width clamp in `_sync_rail_projection` picks up
   the new `NODE_RAIL_WIDTH` automatically.
8. **Docs.** Rewrite the rail paragraphs of `docs/ace.md` › "Agents Zoom and Node Rail".
   Cover:
   - the 22-cell width;
   - names and relative names;
   - dimming;
   - the `×N` vocabulary;
   - left-aligned tribe titles with the `×N` roll-up;
   - `▴N ▾M`;
   - the depth-3 clamp.

   Remove the stale "9-cell", "5 cells", and "`▸` fold mark" wording.

## Tests

- `tests/ace/tui/widgets/test_agent_list_render_rail.py`: rewrite to the new anatomy.
  - **Exhaustive widths.** Every bucket, raw `STOPPED`, and every kind (plain, clan,
    session container, monitor, gate, named proc, workflow, bash/python step, Patch)
    must yield `cell_len == RAIL_CONTENT_CELLS`. Cover depth 0–5, 1- and 2-char hints,
    marked, unread, folded and unfolded, empty, short, long, wide-char, and 44-cell
    names, and every grouping mode and banner level.
  - **Name parity.** For a fixture matrix, `rail_row_name`'s name appears verbatim in
    the expanded row's `format_agent_option(...)[0].plain`.
  - **Relative names.** `.` and `--` boundaries strip; non-boundary prefixes such as
    `sase-1` vs `sase-10` do not; a remainder that is just the separator does not;
    `None` anchors give the full name.
  - **Elision.** Budget respected, cell-aware, tail-biased.
  - **Dimming.** Only settled and read rows dim.
  - **`×N`.** Present iff `ctx["folded_total"]` is set; capped at `×99`; aligned to the
    shared column. A regression test pins that a folded session container with 4 members
    reads `×4`, never `0`.
  - **Banners.** Every prefix register, the BY_DATE empty prefix with a hint, folded
    count and roll-up, and `rule ≥ 1` with a maximal label.
  - **Titles.** `≤ RAIL_TITLE_CELLS` under long labels, wide icons, 2-char hints, and
    merged panels. Check the drop order, and that the hint and roll-up are never
    dropped.
  - **Subtitle.** Format and degradation.
  - **Legend.** Entries match the vocabulary.
- `tests/ace/tui/widgets/test_agent_list_rail_mode.py`:
  - the anchor map's depth-stack semantics, including reset at banners and spacers and
    the clamp-anchor rule for depth > 3;
  - patching an anchor row's prompt re-renders its children's rail cells, while
    unrelated patches leave them cached;
  - structural paths reset the anchor map;
  - the fallback still never raises.
- `_agent_list_helpers` tests: `folded_member_total` agrees with
  `compute_fold_annotation` across its branches (folded, unfolded with hidden rows,
  anonymous single child, retries).
- The neighbors legend tests keep passing after the delegation.
- `tests/ace/tui/test_agents_mode_affordances.py`: the help legend box-width test still
  passes with the new entries.
- `tests/ace/tui/test_agents_node_rail_wiring.py`: width assertions follow the constant.
  Check that none hard-codes 9.

## Goldens and verification

- Run `sase tool run check` (the wrapped `just check`). Do not run `just check-full`.
- Regenerate goldens with targeted `just fix-tui-screenshots -- <selectors>` through
  `/sase_monitor`, then inspect every created, removed, and updated PNG before
  finalizing. Expected changes:
  - the five `agents_node_rail_*` goldens;
  - `agents_decks_collapsed_single_120x40` and `agents_decks_collapsed_split_120x40`;
  - `agents_deck_blocks_split_rails_120x40`. Its test railed the sidebar to widen the
    split, so confirm it still shows the windowed rails with overflow counts at the
    narrower deck. If not, widen that fixture's terminal or zoom instead of railing.
  - any others the run reports.
- When inspecting the rail goldens, check:
  - names and relative names are legible;
  - the selected row stays clearly readable even when its name is dimmed. If
    dim-under-highlight reads poorly, use a muted explicit color instead of `dim`.
  - the `×N` column and pips align;
  - titles are left-aligned, with no clipped chips.
- Spot-check `j`/`k` in rail mode with `SASE_TUI_PERF=1` against the 16 ms p95 target.
  Keep all render and toggle paths free of disk I/O.
- Take one live `sase screenshot` in rail mode (`-p ctrl+s`) on real data as a final
  sanity pass.

## Rejected alternatives

- **Content-fitted width** (for example clamped to 16–26): more compact for short names,
  but the deck edge moves whenever the longest visible name changes.
- **Full names everywhere:** `sase-1aq.10.7.5.land` under its clan is noise, and deep
  rows would elide into mush. Relative names follow the precedent of the TUI's own jump
  legend.
- **Keeping the `▸` banner fold mark:** it collides with the expanded name-root prefix
  and sits next to `▶`. `×N` is already how the expanded list says "folded".
- **Status-colored names:** status already owns the glyph cell. Name color stays
  identity (kind), as in the expanded list.
- **Showing runtimes, retry badges, or `@tribe` chips on rail rows:** they cost width
  and contradict "calm by default". The tooltip and identity header already carry them.
