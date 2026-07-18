---
tier: tale
title: Agent family row icon and a better clan icon
goal: 'The Agents tab marks a real multi-member agent family with its own container
  glyph so it is instantly distinguishable from a single-agent row, and the clan container
  glyph is replaced with a more legible mark that reads as a sibling of the new family
  mark.

  '
create_time: 2026-07-18 09:04:10
status: done
prompt: 202607/prompts/agent_family_row_icon.md
---

# Plan: Agent family row icon and a better clan icon

## Context

The Agents tab renders a one-glyph _type badge_ immediately before the provider badge and display name
(`src/sase/ace/tui/widgets/_agent_list_render_agent.py:184`). Today that slot is populated for clan containers (`⌂`),
top-level workflows (`≡`), ChangeSpec rows (`❑`) and unknown types (`[X]`), but **never for an agent family root** —
family roots are `appears_as_agent` rows, so they fall through every branch and render bare.

That leaves two structurally different things looking identical at rest:

- a **real agent family** — a promoted root plus one or more member agents (`cx` presenting `cx--plan`, `cx--code`, …),
  which after `fbe165baf` presents the bare family name on its top row; and
- a **single agent row** — one agent with no real members. Crucially this includes a lone plan proposer, which
  `ensure_synthetic_planner_children` (`src/sase/ace/tui/models/_agent_status_family.py:550`) gives exactly one
  _synthetic_ child row. That makes it a foldable parent with one agent child that is not a family at all.

Both render as a bare teal name with a `×N` fold annotation. The only way to tell them apart today is to press `l` and
read the child names. This plan gives the family its own glyph so the distinction is available at a glance, without
expanding, and takes the invitation to improve the clan glyph at the same time.

## Design

### 1. The glyph system

The two grouping rows become a deliberate pair: **hue says "this row groups agents", shape says how.**

| Row          | Glyph | Codepoint                                        | Reading                                                             |
| ------------ | ----- | ------------------------------------------------ | ------------------------------------------------------------------- |
| Agent clan   | `◫`   | U+25EB WHITE SQUARE WITH VERTICAL BISECTING LINE | a container split into parallel compartments — members side by side |
| Agent family | `⌘`   | U+2318 PLACE OF INTEREST SIGN                    | interlocked loops — members linked in a chain                       |

Both render in the existing clan lavender `#D7AFFF` (`_CLAN_GLYPH_STYLE`), bold. The **name** styling is unchanged and
stays meaningful: a clan container is not an agent, so its name keeps lavender; a family root _is_ an agent, so its name
keeps agent teal `#00D7AF`. So the glyph marks "grouping", the name colour marks "is/isn't a real agent".

`⌂` (house) is retired for the clan. It is small and low-slung at cell size, and "home" only relates to clans
incidentally through the hood namespace rule; `◫` reads as a container of parallel members and pairs visually with the
new family mark.

**Glyph selection was constrained, not free.** The PNG harness renders through resvg with `skip_system_fonts=True`
against only `tests/ace/tui/visual/fonts/FiraCode-{Regular,Bold}.ttf` (Fira Code 6.2). Both chosen glyphs were verified
to be (a) East-Asian-width `N`, so they occupy exactly one cell and cannot disturb the right-aligned runtime column that
`assemble_padded_option` computes from `left.cell_len`; and (b) present with non-empty outlines in **both** weights.
Most obvious alternatives (`⧉`, `⋔`, `⇉`, `⑂`, `⛓`, `❖`, `⋮`) are absent from that font and would rasterize as tofu — as
several already-shipped glyphs (`✗`, `⏳`, `❑`, `⚡`, `↻`, `◌`, `▸`) silently do. Do not substitute a glyph without
re-running that two-part check. Rejected after rendering: `⎇` (illegible at cell size), `▢` (reads as an empty
checkbox), `☰` (too close to the workflow `≡`), `◉` (reads as a status dot).

### 2. When the family glyph appears

This gating rule is the substance of the feature; the glyph is just its expression.

Render `⌘` on a row when **both** hold:

1. `agent.is_family_root_entry` (`src/sase/ace/tui/models/agent.py:60`) — the row anchors a persisted family; and
2. the family has **at least one real member** — i.e. at least one entry in `agent.followup_agents` that is not the
   synthetic planner row.

Condition 2 is what separates a family from a lone planner. `_agent_status_apply` populates `followup_agents` for every
family-member child (`:315-320`), and the synthetic planner row lands there too, so an unqualified emptiness check would
wrongly badge every lone plan proposer. Introduce a marker for synthetic rows and exclude it:

- Add `is_synthetic_planner: bool = field(default=False, init=False, compare=False)` to `AgentState`, set to `True` in
  `ensure_synthetic_planner_children`. The `init=False`/`compare=False` shape matches `presented_agent_name` and is
  already handled correctly by the bundle serializer, which skips non-init fields.
- Expose the gate as a derived property on `Agent` (for example `is_family_container_row`) combining the two conditions.

A property rather than a precomputed field is the right call here: it is an O(children) list scan over already-resident
objects with no I/O, so it respects the render-path rules, and unlike a cached field it cannot go stale independently of
the `followup_agents` list it is derived from.

Resulting behaviour, which is what the tests should pin:

| Row                                                           | Glyph                        |
| ------------------------------------------------------------- | ---------------------------- |
| Family root with real members (`cx` → `cx--plan`, `cx--code`) | `⌘`                          |
| Lone plan proposer with only a synthetic planner child        | none                         |
| Plain single agent, no children                               | none                         |
| Anonymous single-step workflow that renders as an agent       | none                         |
| Family root nested inside a clan                              | `⌘` (and the clan keeps `◫`) |
| Clan container                                                | `◫`                          |

Parallel families (`agent_family_parallel`) are still families and get `⌘`. Accordingly, document the glyph as "agent
family container" rather than "sequential chain", so the label stays true for both execution shapes.

### 3. Wiring

- **Styling** — add `_FAMILY_GLYPH` / `_FAMILY_GLYPH_STYLE` beside the clan constants in
  `src/sase/ace/tui/widgets/_agent_list_styling.py:114`, and repoint `_CLAN_GLYPH` to `◫`. Carry the existing
  one-terminal-cell comment onto the new constant; it is a real constraint, not a note.
- **Render** — add a branch to the badge chain in `_agent_list_render_agent.py:184`, after the clan branch and before
  the `not (is_appears_as_agent or agent_is_tree_child(agent))` fallback, so family roots stop falling through. Leave
  the clan row's other suppressions (bead badge, name annotation, file-change glyph) alone — the family row is a real
  agent and should keep all of them.
- **Render cache** — add the new gate to `agent_render_key` in
  `src/sase/ace/tui/widgets/_agent_list_render_cache.py:134`. That key is deliberately explicit so a new visible input
  is a conscious edit; omitting it would let `patch_agent_row` serve a stale badge when a row gains its first real
  member and becomes a family without any other keyed field changing. This is the same correctness step `fbe165baf` took
  for `presented_agent_name`.
- **Help modal** — register the new glyph and update the clan entry in
  `src/sase/ace/tui/modals/help_modal/agents_bindings.py:285`. `src/sase/ace/CLAUDE.md` makes this mandatory, and the
  57-character box-width rule there applies.

The detail panel is intentionally out of scope: a selected family root already gets its identity from the header and the
members list, and the clan aggregate panel has its own `CLAN` heading. This change is about scanning the list.

## Testing

- **Model-level** (`format_agent_option(...).plain`, the established pattern): family root with real members renders
  `⌘ `; lone planner with only a synthetic child renders no badge; plain agent renders no badge; clan container renders
  `◫ `. Put the clan assertion with the existing one in `tests/ace/tui/models/test_agent_tree.py:298`, which currently
  pins `"⌂ research @epic (RUNNING) ×2 [R1]"` and must be updated.
- **Cache**: a row that gains its first real member re-renders rather than serving a cached badge-less row — extend the
  `tests/ace/tui/widgets/test_agent_render_cache*.py` suites.
- **Visual**: add one golden showing a family root and a lone planner in the same frame, since the contrast between them
  is the whole point of the feature. Both `assert_page_svg_contains(page, "⌂")` at
  `tests/ace/tui/visual/test_ace_png_snapshots_agents.py:673` and the goldens themselves need updating.
- Regenerate affected goldens with `just update-visual-snapshots` on Linux and **eyeball each diff** rather than
  accepting blind. Expect changes to the clan set (`agents_clan_tree_*`, `agents_clan_unread_*`, `agents_clan_panel_*`)
  from the glyph swap, and to the family set (`agents_renamed_plan_family_root`, `agents_parallel_family_counts`,
  `agents_waiting_family_child`, `agents_retry_e2e_plan_family_countdown`) from the 2-cell left shift.
- `just check` and `just test-visual` must pass.

## Risks and compatibility

- **Two-cell shift on family rows.** Family roots gain a badge they never had, shifting their name right by two cells
  and pulling the right-aligned runtime suffix accordingly. This is intended — it is what makes the row scannable — but
  it is why the family goldens change, and it is worth confirming no row overflows at the 120-column snapshot width.
- **Dismissed/restored rows.** `followup_agents` is excluded from bundle serialization
  (`src/sase/ace/tui/models/agent_bundle.py:20`), so a family root rehydrated straight from a bundle without passing
  through `_agent_status_apply` would have an empty list and lose its glyph. Verify which path dismissed rows take; if
  they bypass status-apply, either accept the degradation deliberately and note it, or gate on family metadata that does
  survive the bundle.
- **Muscle memory.** Changing `⌂` retrains existing users. It is a deliberate, user-invited swap; the docs updates below
  are what make it discoverable.
- **Docs.** Update the glyph table and the clan paragraph in `docs/ace.md` (around `:578` and `:659`) and the clan
  rendering sentence in `docs/agent_families.md`. `sase/memory/glossary.md` needs no change, and memory files must not
  be edited without explicit user permission.
