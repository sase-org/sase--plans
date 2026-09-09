---
tier: tale
title: Fix Agents-tab h navigation for plan families and single tribe panels
goal: "Lowercase h follows the rendered hierarchy from real plan-family members to their
  family and then to the containing tribe panel, including a sole @default panel, while
  uppercase H retains structural-collapse priority and malformed ancestry remains safely
  rejected.

  "
create_time: 2026-09-09 19:52:57
status: wip
---

# Plan: Fix Agents-tab h navigation for plan families and single tribe panels

## Confirmed root cause and interaction contract

The failure is deterministic and has two independent causes; it is not a Textual
highlight, refresh, or key-binding problem:

1. A real plan-family workflow loads the family root and each workflow step with the
   same run timestamp as `raw_suffix`. The canonical tree lookup already handles this
   compatibility shape by retaining the non-child root as the parent, but
   `_resolve_agent_left_navigation_target()` adds a stricter scan that counts the child
   workflow-step aliases as competing fold owners. The screenshot's `main` planner row
   therefore makes the valid `he--code -> he` edge look ambiguous, so the resolver
   returns no target before selection bookkeeping can run. The current navigation
   fixture gives its workflow-agent step a distinct suffix and misses the production
   shape.
2. The approved implementation explicitly gates whole-panel activation on there being
   more than one split panel, and `_activate_focused_panel()` repeats the same
   restriction. The screenshot contains a sole split `@default` panel, so the top-level
   `he` family is intentionally classified as having no selectable tribe parent under
   the current code. That restriction no longer matches the requested hierarchy.

Apply the user's clarified key contract (which supersedes the accidental uppercase
reference in the report):

| Focus                                                                          | Lowercase `h`                                                              | Uppercase `H`                                                                       |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `he--code` or the concrete `main` agent step                                   | Select the rendered `he` family row without changing folds                 | Collapse the containing `he` family according to the existing structural-fold rules |
| Top-level `he` family in the sole split `@default` panel                       | Select the whole `@default` panel                                          | Collapse the open `he` family; do not navigate to the panel                         |
| Any top-level agent/family/clan or grouping banner in a sole split tribe panel | Select that whole panel                                                    | Preserve the current structural-then-group collapse priority                        |
| Selected expanded tribe panel, including a sole panel                          | Collapse the panel on `h`; retain the existing `H` selected-panel handling | Keep panel isolation/restoration precedence                                         |
| Merged `All agents` layout                                                     | No fabricated tribe-parent target                                          | Preserve current merged-layout behavior                                             |
| Stale, self-referential, or genuinely ambiguous structural edge                | No-op without skipping to the panel                                        | Preserve the next valid uppercase collapse behavior                                 |

The navigation stays independent of grouping strategy, including `by status`. All
resolution and selection work must remain in-memory and prompt-free, with no full list
rebuild for either member-to-family or row-to-panel navigation.

## Resolve canonical parent owners without rejecting workflow aliases

Keep `agent_parent_fold_key()` and `tree_parent_lookup()` as the source of the immediate
rendered relationship, but align the navigation validation with the lookup's documented
legacy workflow-child semantics:

- Resolve the parent through the canonical lookup and locate that exact object in the
  loaded Agents projection.
- When checking uniqueness, distinguish structural fold owners from child rows that
  merely repeat their parent's `raw_suffix`. A root family row and its `main`/script
  workflow children sharing a timestamp represent one owner, not multiple owners. Still
  reject two distinct root/fold-owner rows for the same key, a missing canonical owner,
  a self-reference, or an owner that appears inconsistently in the projection.
- Retain the existing endpoint checks after ownership is established: only real agent
  entries may navigate to a sequential-family container, only a sequential-family/direct
  clan member may navigate to its clan as appropriate, and hidden python/bash/script
  rows do not gain a navigation shortcut.

Keep this ownership validation local to Agents navigation unless exploration during
implementation shows an existing pure tree helper is the canonical reusable seam. Do not
weaken the global tree projection, deduplication, panel inheritance, or fold-key
behavior to accommodate this action.

## Treat a sole split panel as a real navigation parent

Generalize whole-panel selection eligibility from "multiple split panels" to "a valid
split tribe panel." Keep merged mode excluded because its `All agents` surface is not a
tribe panel. Update both the pure left-target capability and the selection mutation so
action dispatch, footer computation, and actual activation agree.

Once a sole panel can be selected, preserve the established selected-panel exception to
navigation-only `h`: allow that selected expanded panel to collapse, retain its
remembered in-panel row/banner, and let the existing `l` path expand and then re-enter
it. Keep panel-fold persistence, collapsed-panel handling, jump history, attempt reset,
unread departure behavior, detail routing, and selected border/title rendering on their
existing paths. Align any panel-fold capability enumeration that still declares an
expanded sole split panel uncollapsible so numeric folding and footer affordances do not
contradict the key action. Do not make the merged panel collapsible.

The `he--code -> he -> @default` ladder must save reversible jump anchors and update
ordinary panel selection memory, unread acknowledgement/departure, attempt state, and
detail/footer state exactly as the existing multi-panel ladder does. Parent and panel
selection must not mutate tree folds, grouping folds, or panel fold state until `h` is
pressed again on the selected panel to request collapse.

## Keep conditional affordances truthful

Continue deriving the footer from `_resolve_agent_left_navigation_target()` so the
screenshot-shaped child advertises `h: parent family` and the top-level row in a sole
split panel advertises `h: parent tribe`. After the panel is selected, show the existing
selected-panel enter/collapse controls, now backed by a real single-panel collapse
action. Uppercase `H` must continue to advertise and perform `collapse family` on the
`he` row before any group collapse; no keymap action ids, configured defaults, general
help contract, or non-Agents behavior changes.

Expect broad Agents PNG changes where a top-level row in a single split panel gains the
conditional `h: parent tribe` chip. Inspect actual/expected/diff artifacts and accept
only this intentional footer change plus focused selected-panel state coverage; do not
refresh unrelated goldens.

## Regression coverage and verification

Add a loader-shaped regression fixture matching the screenshot: a plan-family workflow
root, a concrete `main` agent step and hidden script steps whose `raw_suffix` equals the
root timestamp, and a later `--code` family member whose `parent_timestamp` points to
that root. Exercise it in `BY_STATUS` within a sole `@default` panel and prove:

- `h` on the coder and concrete planner selects the family root despite legitimate child
  aliases;
- a second `h` on the top-level family selects `@default`, without any tree/group/panel
  fold mutation;
- `H` on the open family still collapses it and reanchors according to existing rules;
- duplicate non-child owners remain ambiguous, while hidden script rows, stale links,
  and self-links remain no-ops.

Extend whole-panel tests for both the reserved default and a sole named tribe: top-level
row/banner to selected panel, selected panel to collapsed panel on `h`, `l`
expansion/re-entry, remembered selection and jump restoration, persistence, and the
unchanged merged-layout no-op. Update test harnesses that currently duplicate the
obsolete `len(panel_keys) > 1` guard, but include coverage through the real
`AgentSelectionMixin`/panel-folding path so a synchronized stub cannot mask another
production mismatch.

Cover footer resolution for the exact aliased plan family, sole-panel `parent tribe`,
selected sole-panel controls, uppercase family-collapse precedence, and Tools-detail
precedence. Run the focused navigation, structural-fold, panel-collapse/isolation,
footer, and grouping tests first; run the complete Agents/Tools PNG visual suite and
inspect every changed golden. After `just install`, finish with the repository-required
`just check` and `git diff --check`.

This is a focused Python TUI presentation/navigation correction and requires no
`sase-core` wire or API change.
