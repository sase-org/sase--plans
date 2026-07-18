---
tier: tale
title: Color-matched clan and family name-first identities
goal: 'Agent clan and real multi-member family rows present a name-first identity
  pair whose name and trailing kind icon share the grouping kind''s reserved color,
  while ordinary agent names remain gold and all existing row context, fallback behavior,
  cache correctness, and detail-panel consistency are preserved.

  '
create_time: 2026-07-18 10:42:04
status: wip
prompt: 202607/prompts/clan_family_name_first_identity.md
---

# Plan: Color-matched clan and family name-first identities

## Context

The recently landed clan/family row identity unification established one trailing identity slot after status, fold
counts, aggregate chips, and bead context. It currently renders the kind icon immediately before a gold name:

| Row kind       | Current trailing identity                                             |
| -------------- | --------------------------------------------------------------------- |
| Ordinary agent | `research.audit` in gold                                              |
| Family root    | `⌘ research.family` with an azure icon and gold name                  |
| Clan           | `◫ research @epic` with an orchid icon, gold name, and gold tribe tag |

That layout still makes grouping names look like ordinary agent annotations and puts the kind marker before the thing it
classifies. The requested follow-up is a presentation-only refinement: make the grouping kind's color apply to the name
as well as the icon, and make the icon a trailing mark after the name.

This work is a `tale`: one coding agent can update the shared renderer and style constants, keep the two detail headers
consistent, adjust their focused tests and documentation, and review the bounded visual snapshot delta as one coherent
change. It does not require ordered backend/frontend phases. The behavior is presentation-only and remains in the
Python/Textual layer; no Rust core API, configuration default, memory file, or persisted data format changes are needed.

## Identity grammar

Change only the trailing identity grammar for grouping rows:

| Row kind       | Resulting trailing identity                                                                                   |
| -------------- | ------------------------------------------------------------------------------------------------------------- |
| Ordinary agent | `research.audit` in gold (unchanged)                                                                          |
| Family root    | `research.family ⌘`, with both cells using the family azure                                                   |
| Clan           | `research ◫ @epic`, with the name and icon using the clan orchid and tribe tags retaining their existing gold |

The name and icon form the contiguous semantic pair: `<name> <kind icon>`. For clans, deduplicated/split-panel-aware
`@tribe` tags remain decorators after that pair rather than being moved ahead of the name. This satisfies the icon's
directly-right-of-name placement without changing tag suppression or forcing users to scan a new tag order. Existing
status, `×N`, aggregate count chip, provider badge, tree indentation, and bead ordering remain untouched, so examples
such as `×2 [R1] research ◫ @epic` and `◆ research.family ⌘` keep context to the left of identity.

Use the existing kind gates as the sole authority:

- `agent.is_clan_container` selects the orchid clan identity and sources its synthesized name from `agent.display_name`.
- `agent.is_family_container_row` selects the azure family identity only for a real multi-member family root. Lone plan
  proposers, plain agents, workflows, and family-shaped records without a real member stay unbadged and retain the
  ordinary gold trailing-name treatment.
- When a family-container gate is true but both presented-name sources are absent, retain the existing icon-only
  fallback. It should still append exactly one separated azure `⌘`; there is no name to precede it, but the row must not
  silently lose its container identity.

Keep the current grouping-container suppression in the leading type-badge chain. Reordering the trailing pair must not
reintroduce the `[agent]` debug badge for synthetic clans or fixture/bundle-restored family containers, and it must not
recreate the provider-spacing/pane-width edge case fixed by the prior change.

## Styling and rendering design

In `src/sase/ace/tui/widgets/_agent_list_styling.py`, represent orchid `#D75FFF` and azure `#00AFFF` as the canonical
per-kind identity colors, and derive both the name and glyph styles for each kind from those shared values. This makes
the requested color equality structural instead of relying on duplicated literals. Preserve the glyphs' bold weight and
the names' existing normal weight; the requirement is a shared hue, while the heavier one-cell mark should remain the
visual punctuation. Update the comments that currently reserve those colors only for icons so they describe the whole
grouping identity pair.

In `src/sase/ace/tui/widgets/_agent_list_render_agent.py`, have the trailing identity block determine three independent
pieces from the existing row kind: the presented name, its style, and the optional glyph/style. Append the name first,
then one separating space and the glyph. Ordinary rows continue using `_AGENT_NAME_ANNOTATION_STYLE`; do not globally
recolor that gold token, because it also styles ordinary agent names and member labels. Clan tags continue to append
after the completed name/icon pair.

The reorder should neither add nor remove cells for named grouping rows: `glyph + space + name` becomes
`name + space + glyph`. Preserve the current single leading separator before the identity block and the icon-only
fallback's spacing. No render-cache key change is expected: row-kind gates, both name sources, clan tags, panel/tag
context, provider tuple, fold state, and count inputs are already represented by `agent_render_key`, while the colors
and append order are process-static presentation code. Document that conclusion in the implementation rather than adding
redundant cache inputs.

The render path must stay memory-only and allocation-light. Do not add I/O, config lookup, filesystem inspection,
subprocess work, or a second family aggregation. Reuse the already-computed `is_family_container_row` decision and
existing `Text` construction so row patching and cache behavior remain unchanged.

## Detail-panel consistency

The selected-row detail surface should reinforce the same color semantics without treating headings as list-row
identities:

- In `prompt_panel/_agent_display_clan.py`, color the clan header's `Name:` value with the canonical clan orchid. Keep
  the `◫ CLAN` section heading in its current icon-first form: it is a legend-like section heading, not the clan-name
  pair the user asked to reorder. Keep individual member labels gold.
- In `prompt_panel/_agent_display_header.py`, use the canonical family azure for the `Name:` value when the selected
  agent satisfies `is_family_container_row`; ordinary agents, lone planners, workflow rows, and family members retain
  the shared gold name annotation. Do not add an icon to this generic metadata row.

This avoids a row/detail mismatch after the list name changes, while keeping the scope focused and preserving the
existing detail layouts.

## Documentation

Update the user-facing identity descriptions in `docs/ace.md` and `docs/agent_families.md`:

- Replace the icon-first/gold grouping grammar with the color-matched name-first examples `name ◫` and `name ⌘`.
- Explain that plain agent annotations remain gold, while clan and family identity pairs use their reserved orchid and
  azure respectively.
- Keep clan `@tribe` tags documented after the `name ◫` pair and update the glyph table from “prefixes the gold name” to
  “follows the same-colored name.”
- Update the clan detail-panel note to say its `Name:` value is orchid. If family detail metadata is described nearby,
  keep it consistent with the azure family name; do not add unrelated documentation.

The help modal's `◫` “Agent clan container” and `⌘` “Agent family container” labels remain semantically correct, so
verify them but do not edit them merely to create churn. Do not touch `sase/memory/*` or generated agent-instruction
files.

## Verification

Add or adjust focused tests so plain text and Rich spans both pin the new contract:

1. In `tests/ace/tui/models/test_agent_tree.py`, update the clan assertion to end in `research ◫ @epic`, assert the last
   clan-name span and `◫` share `#D75FFF`, and retain the no-leading-space, no-double-space, count-chip, and
   no-`[agent]` checks. Update the family root to end in `research.family ⌘`, and use the final occurrence of the
   duplicated family name when asserting its azure span so the leading teal ChangeSpec/display name is not mistaken for
   the trailing family identity. Assert both that name and `⌘` use `#00AFFF`.
2. Keep the family-vs-lone-planner test explicit: only a real multi-member root gets the trailing `name ⌘` pair; lone
   planners, plain agents, and anonymous workflows stay unbadged, and their ordinary trailing names remain gold.
   Preserve the missing-name icon-only assertion.
3. Update exact clan-tag/cache expectations in `tests/ace/tui/widgets/test_agent_render_cache.py` from `◫ research @...`
   to `research ◫ @...`. Retain the transition test proving that adding the first real member invalidates the cached row
   and introduces `⌘`; no cache-key expansion is needed for static styles/order.
4. In `tests/ace/tui/widgets/test_agent_display_clan.py`, assert the clan `Name:` value is orchid while the `◫ CLAN`
   heading remains unchanged and member labels remain gold. Add focused coverage around `build_header_text` for a real
   family root whose `Name:` value is azure, alongside the existing ordinary-agent gold assertion.
5. Run the relevant model, renderer/cache, detail-header, and row-patching tests before the visual suite. Include the
   provider-badge clan case and bundle/fixture family-container case so spacing and leading-badge suppression remain
   protected.

Regenerate PNG snapshots only after focused textual/style tests pass. The expected visual surface is the same bounded
grouping-row set affected by the prior identity move: the nine `agents_clan_*` frames and the six family-root frames
(`agents_family_and_lone_planner_glyph`, `agents_output_variables_multi_agent`, `agents_parallel_family_counts`,
`agents_renamed_plan_family_root`, `agents_retry_e2e_plan_family_countdown`, and `agents_waiting_family_child`). Detail
name recoloring may affect frames where a clan or family root is selected. Inspect every actual diff rather than
assuming the set: verify `name icon` order, exact matching hues, tag placement, selected-row readability, dense clan
scannability, and no clipping or pane-width change at 120 columns. Preserve unrelated antialias-only goldens rather than
accepting a broad renderer-drift rewrite.

Before handoff, run `just install` as required for the ephemeral workspace, then `just check` and `just test-visual`
without snapshot-update mode. All retained changed goldens must pass exact comparison; any unrelated pre-existing
renderer drift or timing flake should be isolated and reported rather than hidden by weakening feature assertions.

## Risks and acceptance criteria

- **Grouping names are no longer gold.** This is intentional: kind now reads as one colored `name icon` unit. Ordinary
  agent names remain gold, preserving contrast between a single agent and a group.
- **Azure text can resemble nearby blue UI accents, and orchid can resemble waiting/parallel colors.** The hues were
  already reserved for these row-grouping identities; applying each to several name cells increases their visual weight.
  Review selected/unselected and dense rows in the snapshots, retaining the exact existing saturated colors so the icons
  do not lose their established identity.
- **A trailing icon could visually drift toward tribe tags or suffix content.** A single space joins name to icon, and
  clan tags follow with their existing ` @tag` separators and gold style. Unit tests pin the complete token order.
- **Detail headers could accidentally recolor non-container agents.** Gate family styling on the same real-container
  predicate used by the row; explicitly test a lone planner and a normal agent as gold.

The change is accepted when clan rows show orchid `name ◫`, real family roots show azure `name ⌘`, clan tribe tags
remain after the icon, ordinary agents and lone planners retain gold unbadged names, detail `Name:` values agree with
the selected grouping kind, row/cache behavior is unchanged, all intentional visual diffs are reviewed without clipping,
and the required checks pass.
