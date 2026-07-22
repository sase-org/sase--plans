---
tier: tale
title: Add clan collapse to the selected tribe-panel H ladder
goal: 'Whole-panel H collapses every expanded agent clan in the selected tribe panel
  after houses are fully collapsed and before top-level groups or the panel itself,
  with accurate contextual hints and complete regression coverage.

  '
create_time: 2026-07-22 09:03:47
status: done
---

- **PROMPT:** [202607/prompts/tribe_panel_h_clan_collapse.md](prompts/tribe_panel_h_clan_collapse.md)

# Plan: Add clan collapse to the selected tribe-panel `H` ladder

## Context

Uppercase `H` is action-driven through `hooks_or_collapse_all`, so custom keymaps receive the same behavior as the
default key. On row focus, the action already walks the scoped house, clan, and group hierarchy. On expanded whole-panel
focus, it currently collapses every open canonical workflow/family house in the selected tribe panel, then collapses
expanded level-0 grouping banners one at a time in reverse rendered order, and finally delegates to lowercase `h` to
collapse the panel.

That panel-wide ladder skips the structural clan layer. In the reported `@chop` state, the `toobig-g` clan is expanded,
its member houses are already collapsed, and `Running` is still open; pressing `H` therefore targets the grouping banner
instead of first hiding the clan members. Add a distinct clan rung between houses and grouping banners without changing
the established row-focused ladder.

## Behavioral contract

Keep Tools detail compaction as the action's highest-priority context. For an expanded selected tribe panel, handle
exactly one rung per invocation in this order:

1. If any canonical workflow, agent, or sequential-family house in the selected panel is open, fully collapse all such
   houses using the existing panel-wide house transition and stop.
2. Otherwise, if any canonical agent clan in the selected panel has an open outer fold, drive every such clan directly
   to collapsed in one batch and stop. This is panel-wide like the house rung, not dependent on the remembered row or on
   which clan happens to render last.
3. Otherwise, collapse only the last expanded level-0 grouping banner in that panel's actual rendered order and stop.
4. Otherwise, collapse the selected panel through the existing lowercase-`h` path.

The clan probe must include clans that belong to the panel but are currently hidden by a collapsed grouping banner;
group folds are presentation state and must not conceal an open structural fold from a panel-wide operation. Equal or
malformed fold keys must fail closed: only unique, canonical synthetic clan containers owned by the selected panel may
be mutated, and a global fold-state key shared ambiguously with another loaded row or panel must be skipped. Valid clans
may still collapse when another candidate is rejected.

Preserve the selected panel, dormant row index, per-panel selection memory, grouping state, and any armed
panel-isolation restore record. The clan rung changes only in-memory structural fold state, performs the whole batch
before one fold-only refilter, and does not trigger content indexing, disk I/O, persistence, or a background reload. An
already collapsed selected panel remains the existing terminal no-op. Merged layout and all row-focused `H` behavior
remain unchanged.

## Runtime design

Add a small panel-scoped clan target resolver alongside the existing fold target helpers. Derive panel membership from
the current cached/rendered panel slice and identify clan owners from canonical synthetic-container metadata and
`agent_fold_key`, rather than parsing display labels or dotted agent names. Validate ownership and fold-key uniqueness
against the complete loaded projection because `FoldStateManager` is global across panels. Return only open clan keys in
stable panel order, or no target when every valid clan is already collapsed.

Add a transition that applies the resolved clan keys as one saturating bulk collapse and then reuses the selected-panel
inner-fold refilter/restore path. Insert it in `action_hooks_or_collapse_all` only after the existing house transition
declines and before resolving a group target. Keep this synchronous and bounded to cached in-memory state, in accordance
with the TUI responsiveness contract. Organize the helper in a focused module or mixin boundary so the already-large
group-folding implementation does not accumulate another unrelated ownership algorithm.

## Conditional UI and discoverability

Probe the same clan target while computing the Agents footer, after house availability and before group availability.
Thread a distinct availability flag through the footer API and render the configured `hooks_or_collapse_all` key with a
`collapse clans` label when this rung is next. Higher-priority `compact tools` and `collapse houses` labels must win;
`collapse group` must appear only after no open valid clan remains. Custom action bindings must display their configured
key, not a literal `H`.

Update all user-facing action descriptions and discovery surfaces to describe houses, clans, groups, and the final panel
step consistently: fallback/runtime binding metadata, command catalog label and search aliases, Agents help content,
default keymap comments, and the Agents/clan documentation. Keep the help text within its existing width constraints.

## Regression coverage

Extend the focused fold-transition tests with projected clans in one or more tribe panels and cover the full precedence
contract:

- an open house is collapsed first while open clans and groups remain unchanged;
- the next press collapses all valid open clans in the selected panel in one transition, including a clan hidden by a
  grouping banner, while sibling-panel clans and group registries remain untouched;
- after clans are saturated, status/date groups retain their existing reverse-render-order, one-banner-per-press
  behavior, followed by the existing panel collapse;
- mixed valid and malformed or globally duplicated clan owners fail closed per candidate without mutating an ambiguous
  key;
- whole-panel focus, remembered selection, current index, nested group folds, and isolation restore state survive the
  clan refilter, which occurs exactly once with content-index refresh disabled;
- collapsed-panel, merged-layout, and row-focused clan behavior do not regress.

Extend footer unit/integration tests to verify the precedence matrix, `collapse clans` label, custom-key rendering, and
that footer refresh invokes the panel resolver rather than a row-scoped structural resolver. Update help, binding, and
command-catalog assertions, including a clan search alias.

Add or extend an ACE PNG scenario with an expanded clan and whole-panel focus, matching the reported shape: the footer
must advertise `H collapse clans`, and pressing the action must collapse the clan before its top-level group. Inspect
visual diffs and accept only intentional footer/list changes; do not refresh unrelated goldens.

## Validation

Run `just install` first for the ephemeral workspace, then run the focused folding, footer, help, command-catalog, and
affected exact PNG tests. Run Ruff, mypy, and Symvision while iterating, paying particular attention to private helper
visibility and file-size boundaries. Finish with the repository-required `just check`, and re-run the affected PNG tests
without snapshot-update mode to prove the accepted goldens compare exactly.

## Non-goals

Do not change lowercase `h`, clan expansion with `l`, grouping order, panel isolation with `Z`, fold persistence, clan
projection in the Rust core, or the semantics of `H` on ChangeSpecs, Axe, Tools, merged Agents layout, or ordinary row
focus.
