---
tier: tale
title: Unified clan/family row identity with distinct icon colors
goal: 'Agent clan and agent family rows present one shared identity grammar on the
  Agents tab: the grouping icon sits directly left of the group''s name, the name
  renders in the same gold and trailing position as every agent name, and each icon
  carries its own reserved color that no other agent-row element uses — so a row''s
  kind is readable from icon shape + color + status counts alone.

  '
create_time: 2026-07-18 09:54:48
status: wip
prompt: 202607/prompts/clan_family_row_identity_unification.md
---

# Plan: Unified clan/family row identity with distinct icon colors

## Context

The previous change (`4c20b1bdb`) gave grouping rows their glyphs — `◫` for clan containers, `⌘` for real multi-member
family roots — but left three inconsistencies that this plan resolves:

1. **Both icons share one color, and it isn't theirs.** `_CLAN_GLYPH_STYLE` and `_FAMILY_GLYPH_STYLE`
   (`src/sase/ace/tui/widgets/_agent_list_styling.py:116`) are both `bold #D7AFFF` — the same lavender used by the
   `parallel` step-type accent, so neither icon has a unique identity and the pair cannot be told apart peripherally.
2. **Clan names use a different grammar than agent names.** Every agent row reads
   `<teal ChangeSpec name> (STATUS) ×N [chip] <gold agent name>` — the gold `presented_agent_name` annotation
   (`_agent_list_render_agent.py:378`, style `_AGENT_NAME_ANNOTATION_STYLE = #FFD700`) is the _agent identity_ slot.
   Clan rows instead put their name at the front, in lavender, right of the provider badges
   (`_agent_list_render_agent.py:204`), and leave the gold slot empty. Two different systems for "what is this row?".
3. **The icons are detached from the names they mark.** Both glyphs render in the leading type-badge slot
   (`_agent_list_render_agent.py:186`), before the provider badges — up to a full row-width away from the gold family
   name they describe.

Concretely, today's rows (right-aligned runtime elided):

| Row kind    | Today                                                    |
| ----------- | -------------------------------------------------------- |
| Agent       | `🤖 audit (DONE) ×2 +1 research.audit`                   |
| Family root | `⌘ 🤖 visual-real-family (PLAN DONE) visual-real-family` |
| Clan        | `◫ 🤖 research @epic (RUNNING) ×2 [R1 W1 D1]`            |

## Design

### 1. One identity grammar for every grouping row

Every row ends with the same **identity block**: `<icon?> <gold name>`, placed where agent names already live — after
the status, fold annotation `×N`, count chip, and bead badge. The icon, when present, sits directly left of the name it
marks. Nothing else about row anatomy changes.

| Row kind    | After                                                    |
| ----------- | -------------------------------------------------------- |
| Agent       | `🤖 audit (DONE) ×2 +1 research.audit`                   |
| Family root | `🤖 visual-real-family (PLAN DONE) ⌘ visual-real-family` |
| Clan        | `🤖 (RUNNING) ×2 [R1 W1 D1] ◫ research @epic`            |

Reading rule the user learns once: _the gold text at the end is the thing's name; the mark in front of it says what kind
of thing it is_ (`⌘` family, `◫` clan, bare = single agent). Clans keep their aggregate status, `×N`, and rolled-up
count chip — which, per the user's direction, is what distinguishes a clan at a glance now that its name looks like any
agent name.

Specifics:

- **Family roots** (`agent.is_family_container_row`): the `⌘` moves out of the leading type-badge slot into the identity
  block, immediately before the gold `presented_agent_name`. The bead badge `◆`, when present, stays to the _left_ of
  the icon (`×N ◆ ⌘ name`) — the icon+name pair is never split. If the gate holds but the row somehow has no presented
  name, render the icon alone in the slot so the container mark never silently disappears.
- **Clan containers** (`agent.is_clan_container`): the leading lavender name is removed entirely. The clan's name
  renders in the identity block from `agent.display_name` (clan containers are synthesized without an `agent_name` — see
  `_container_for_clan` at `src/sase/ace/tui/models/_agent_tree.py:191` — so the annotation source must be
  `display_name`, which returns `agent_clan`). `@tribe` labels (`clan_tags` and the panel `tag_label`) move with the
  name and keep their existing `bold #FFD75F` style: identity stays one contiguous block, `◫ research @epic`.
- **Lone planners / plain agents / workflows / ChangeSpec rows**: unchanged. The lone-planner-vs-family contrast from
  the previous change is preserved — only the position of the family mark changes.
- The clan name no longer has a bold-when-selected variant; like agent names, selection is conveyed by the cursor
  highlight. `is_selected` remains in the render-cache key regardless.

### 2. Reserved icon colors

Each icon gets a saturated jewel tone that is used by **no other element on agent panel rows**. The full row palette was
audited (status colors, chip metrics `agent_count_chip.py`, runtime/activity suffix styles
`_agent_list_render_layout.py`, tier-gutter banner accents, step-type colors, badge glyphs, `STOPPED_COLOR`) before
choosing:

| Icon | Element      | New style      | xterm name       | Why                                                                                                                                        |
| ---- | ------------ | -------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `◫`  | Agent clan   | `bold #D75FFF` | 171 MediumOrchid | Inherits the clan's purple heritage at full saturation — instantly recognizable as "the clan color" without colliding with pastel lavender |
| `⌘`  | Agent family | `bold #00AFFF` | 39 DeepSkyBlue   | Vivid azure; reads as "linked" and pops against the gold name it prefixes                                                                  |

Nearest neighbors, checked deliberately:

- `#D75FFF` vs WAITING amethyst `#AF87FF` (status + chip `W`) and parallel-step lavender `#D7AFFF`: the new orchid is
  far more saturated and magenta-leaning; at one bold cell they do not read as the same family. `#D75FFF` is also qwen's
  provider brand color, but that renders only in the status bar and detail surfaces — never as an agent-row element,
  which is the constraint the user set.
- `#00AFFF` vs STARTING sky `#87D7FF`, ANSWERED/chip-done cyan `#5FD7FF`, and the dim `#5FAFFF` banner gutters: those
  are all pastels or dimmed; fully-saturated azure is clearly distinct, and it never appears inside dim parens the way
  status colors do.
- The two icons are unmistakable from _each other_ (purple-pink vs cyan-blue), and shape remains the primary channel —
  color is reinforcement, so the design stays colorblind-safe.

Both hexes are otherwise unused across `src/` row surfaces (verified by grep). Carry the existing one-terminal-cell
comment onto the constants and note both hexes are reserved for these icons.

No glyph changes — `◫`/`⌘` were already verified present with non-empty outlines in both bundled Fira Code weights, and
colors do not affect the PNG harness's font constraints.

### 3. Render mechanics and gotchas

All in `src/sase/ace/tui/widgets/_agent_list_render_agent.py`:

- Delete the clan/family branches from the leading type-badge chain (`:186-189`). **Gotcha:** clan containers are not
  `appears_as_agent` and not tree children, so without an explicit suppression they would now fall through to the
  `[agent]` debug fallback (`:195`) — the badge chain must explicitly skip clan containers (family roots are
  `appears_as_agent` and skip naturally).
- Skip the leading `display_name` append and tag rendering for clan containers (`:204-223`); drop the clan branch of
  `name_style`.
- **Spacing:** the status opener is appended as `" ("` (`:227`). With no leading name, a clan row would render a double
  space after provider badges (which already emit trailing spaces) or a leading space on badge-less rows. Append `"("`
  without the space when the accumulated text is empty or already ends in whitespace. Pin this with a test — no double
  or leading spaces in any clan-row fold state.
- Build the identity block after the bead badge (`:374-381`): icon (per §1 gates) then gold name, then clan tags for
  clan rows. Keep the existing `if presented_name and not agent.is_clan_container` behavior for plain agents.
- Net row width is approximately unchanged (the icon+space and name+space move within the row rather than being added),
  so no responsive-pane surprises are expected; the golden sweep confirms.

**Render cache** (`_agent_list_render_cache.py:169`): no key changes needed, and the plan's correctness depends on
saying why: every input the new layout reads — `is_clan_container`, `is_family_container_row`, `display_name`,
`clan_tags`, `agent_name`, `presented_agent_name`, `tag_label`, `panel_tag`, provider tuple, chip counts — is already in
`agent_render_key`. Colors and positions are compile-time constants, invisible to a per-process cache.

**Performance:** pure presentation change. No new I/O, stats, or scans on the render path; the family gate remains the
existing O(children) in-memory property.

### 4. Consistency ripple: clan detail panel

`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py` is the one off-row surface that must follow, per the
codebase's own lockstep rule (`_AGENT_NAME_ANNOTATION_STYLE` is documented as "shared by the list row trailing name
annotation and the metadata header Name: value so the two stay in lockstep", and the clan panel already renders member
labels in that gold at `:193`):

- The `Name:` value in the CLAN header (`:281`, currently `bold #D7AFFF`) becomes `_AGENT_NAME_ANNOTATION_STYLE` — clan
  names are gold everywhere now.
- `_CLAN_HEADING_STYLE` (`:29`) adopts the new orchid (`bold #D75FFF underline`), and the heading renders as `◫ CLAN` —
  the panel teaches the icon's meaning on first selection.

The rest of the panel (tribes, member statuses, chips) is untouched, as is the family root's detail header, which
already presents its gold name.

### 5. Docs and help

- `docs/ace.md` — rewrite the clan/family paragraph (~`:578-584`) for the new grammar (identity block, per-icon colors,
  clan rows relying on icon + counts); refresh the glyph table rows for `◫`/`⌘` (~`:660`); touch the CLAN panel bullet
  (~`:1625`) for the `◫ CLAN` heading.
- `docs/agent_families.md` — update the rendering paragraph (~`:50-53`) that still says "related lavender `⌘` glyph" and
  describes the old clan row shape.
- Help modal (`src/sase/ace/tui/modals/help_modal/agents_bindings.py:285`): the `◫`/`⌘` legend entries remain accurate
  ("Agent clan container" / "Agent family container") — verify wording, no change expected. The 57-character box rule
  applies if any edit is made.
- `sase/memory/*` files must not be touched.

## Testing

- **Model-level** (`format_agent_option(...).plain`, `tests/ace/tui/models/test_agent_tree.py`): the fixture helper sets
  `agent_name = cl_name = name`, so expected strings are:
  - Clan container (`:301`): `(RUNNING) ×2 [R1] ◫ research @epic` — also asserts the leading-name removal, the
    no-double-space rule, and that no `[agent]` fallback badge appears.
  - Nested family root (`:302`): starts `  └─ research.family` and ends with the identity block `⌘ research.family`.
  - Family/lone-planner contrast (`:356`): family root starts with its teal name and _ends_ `⌘ cx`; lone planner and
    plain agent rows contain no `⌘`.
  - Style-level: assert via the rendered `Text` spans that `◫` carries `#D75FFF`, `⌘` carries `#00AFFF`, and the moved
    clan name carries `#FFD700`.
- **Cache** (`tests/ace/tui/widgets/test_agent_render_cache.py:79`): the gains-first-member transition test currently
  pins the _leading_ position (`startswith("⌘ ")`); repoint to position-independent membership (`"⌘" not in` →
  `"⌘" in`). `test_agent_render_cache_patching.py:138` is already position-independent.
- **Visual**: regenerate with `just update-visual-snapshots` on Linux and eyeball every diff. Expected set: the nine
  `agents_clan_*` goldens (name moved to the identity block, orchid icon, panel heading changes in
  `agents_clan_panel_*`), the family set (`agents_renamed_plan_family_root`, `agents_parallel_family_counts`,
  `agents_waiting_family_child`, `agents_retry_e2e_plan_family_countdown`) shifting the `⌘` into the identity block
  (family rows shift 2 cells left at the front), and `agents_family_and_lone_planner_glyph`, whose at-rest contrast must
  remain obvious. The existing `assert_page_svg_contains(page, "⌘"/"◫")` assertions
  (`tests/ace/tui/visual/test_ace_png_snapshots_agents.py:729`, `:755`) stay valid; if the SVG serializer emits hex
  fills, additionally assert both new hexes appear in the contrast/clan frames.
- Confirm no golden overflows or clips at the 120-column snapshot width.
- `just check` and `just test-visual` must pass.

## Risks and compatibility

- **Clan rows lose their leading anchor.** With the name in the identity block, a clan row's left edge is provider
  badges + aggregate status. This is the user's explicit direction (icon + status counts carry the distinction), and the
  trailing identity block keeps clans scannable — but the golden eyeball pass should specifically judge collapsed clans
  inside dense project groups.
- **Gold-on-gold adjacency.** `@tribe` tags (`#FFD75F`) now follow a gold name (`#FFD700`). The `@` sigil and the tags'
  bold weight keep them parseable; accepted deliberately to keep the identity block contiguous. If the eyeball pass
  disagrees, dimming the tags is the fallback — not restyling the name.
- **Muscle memory.** The leading `⌘`/`◫` position shipped very recently, so converging on the final grammar now is
  cheaper than retraining later; docs and help stay in sync.
- **Glyph overload.** `⌘` also serves as the xprompt-workflow glyph in the detail panel (`_agent_xprompts.py:16`).
  Different surface; the reserved azure now visually separates the row usage. No action, noted for awareness.
- **Bundle-restored family roots** still degrade to badge-less when `followup_agents` is empty (accepted in the previous
  change); the icon's new position neither improves nor worsens that.
