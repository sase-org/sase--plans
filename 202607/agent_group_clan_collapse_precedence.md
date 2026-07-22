---
tier: tale
title: Prefer group-wide agent-clan collapse before grouping collapse
goal: Uppercase H fully collapses every expanded canonical agent clan in the next
  Agents-panel grouping scope after houses are closed and before a later press collapses
  that grouping banner.
create_time: 2026-07-22 12:27:44
status: done
---

- **PROMPT:** [202607/prompts/agent_group_clan_collapse_precedence.md](prompts/agent_group_clan_collapse_precedence.md)

# Plan: Prefer group-wide agent-clan collapse before grouping collapse

## Context and confirmed root cause

The supplied screenshot shows row focus on the collapsed `sase-8l` clan in the `@epic` tribe panel while sibling clan
`sase-8k` remains expanded inside the same `Running` status group. In that state, one uppercase `H` should fully
collapse the open clan layer and leave `Running` expanded. Only a later `H`, after all houses and clans in that grouping
scope are closed, should collapse the `Running` banner.

The Agents action currently has two related but incomplete ladders. Expanded whole-panel focus already resolves and
collapses every open canonical clan in the selected tribe panel between its house and group steps. Ordinary row or
group-banner focus instead runs a group-scoped house resolver, then a selected-row-only structural resolver, then the
grouping resolver. When the selected `sase-8l` clan is already closed, the structural resolver has no target and cannot
see the expanded sibling `sase-8k`; dispatch therefore falls through to the `Running` group even though that group still
contains an open clan. The contextual footer follows the same incomplete probes and incorrectly advertises
`H collapse group` in the screenshot state.

This behavior belongs in the Python TUI presentation layer. It should extend the existing cached, synchronous
fold-target machinery rather than introduce Rust-core behavior, I/O, awaits, background work, or a new refresh path.

## Behavioral contract and precedence

Treat the grouping banner that the existing row-focused resolver would collapse next as the scope for a group-wide
clan-collapse attempt. For an agent row this is the deepest enclosing group; for a focused collapsed child banner it is
the parent reached by the existing repeated-`H` escalation. Limit the scope to the focused split tribe panel. In merged
layout, the single merged panel remains the scope, matching existing group-wide house behavior.

Within that group, identify every unique canonical synthetic clan container whose binary outer fold is open. One `H`
must drive all such clans directly to `COLLAPSED` in a single batch, regardless of which clan or member row is selected.
Clans in sibling grouping buckets or sibling split panels must not change, even when their display group keys are
identical. Already-collapsed clans remain untouched.

The Agents-tab `H` order should be:

1. Preserve the Tools-detail compaction override.
2. Preserve the separate expanded whole-panel ladder: panel houses, panel clans, top-level panel groups, then the panel
   itself.
3. Under row or grouping-banner focus, fully collapse every open canonical workflow/family house in the next grouping
   scope and stop if anything changed.
4. Otherwise, fully collapse every open canonical clan in that same grouping scope and stop if anything changed.
5. Retain the selected structural transition as a defensive fallback for any valid case not covered by the scoped bulk
   resolvers.
6. Only then perform the existing grouping-banner collapse/escalation.

Thus the reported state becomes a two-press transition: first `sase-8k` collapses while `Running` stays open; the next
press collapses `Running`. Lowercase `h`, `l`, numeric fold selection, whole-panel focus and isolation, group-fold
persistence, non-Agents tabs, and configurable key mappings retain their current semantics.

## Scope resolution and canonical ownership

Generalize the focused-panel clan helper boundary so it can also resolve a group-scoped target without duplicating clan
ownership rules. Reuse the focused panel's rendered slice and build a fully expanded grouping projection with an empty
grouping-fold registry. Find the exact `GroupRow` for the next group target and use its anchored `agent_indices` to
enumerate clan containers in stable rendered order. This mirrors the existing house resolver and keeps a clan plus all
of its members in the outer container's status/date/project/name bucket even when members have different statuses or a
nested grouping banner is currently collapsed.

Validate each candidate through synthetic clan metadata and `agent_fold_key`, not through display text or dotted-name
parsing. Because `FoldStateManager` is global across panels, validate fold-key uniqueness against the complete cached
projection (`_agents_with_children` when available), including rows currently hidden by structural or grouping folds. A
malformed, stale, or duplicated owner must fail closed for that candidate without preventing unrelated valid clans in
the same target group from collapsing.

Return an immutable target that records the panel/group scope, ordered fold keys, and any required selection re-anchor.
Keep the existing panel-wide target and transition semantics intact, sharing only the canonical candidate collection
where it makes the scope rules clearer.

## Focus, refresh, and performance safety

Apply every clan-fold mutation before one lightweight in-memory refilter and explicitly disable content-index refresh
for this fold-only action. Do not refilter once per clan and do not write group-fold persistence intent, because the
first press changes only structural state.

If the selected visible row is a direct member of a clan that will disappear, move selection to that clan's canonical
container before refiltering and update normal split-panel selection memory. If the selected row survives, as `sase-8l`
does when only sibling `sase-8k` is open, preserve its identity, attempt selection, group/banner focus, jump history,
focused panel, and panel isolation state. The group banner must remain open, and the normal fold refilter should remain
the only cache/display invalidation path.

The resolver and footer probe must remain bounded to already-loaded in-memory rows. No keypress or render path may
perform filesystem reads, subprocess work, or a full agent-list rebuild beyond the established fold refilter.

## Conditional affordances and documentation

Use the same group-clan resolver for action dispatch and footer capability calculation. After houses are saturated, the
reported state should show the configured `hooks_or_collapse_all` key with `collapse clans`; after the clan batch it
should change to `collapse group`. Preserve the higher-priority `compact tools` and `collapse houses` labels, the
whole-panel clan probe, and custom key rendering. Keep the selected structural singular label only for a fallback that
is genuinely selected-row-specific.

Update the Agents help text, default keymap comments, command/binding wording where needed, and the Agents/clan
documentation so row focus is described as `houses -> clans in the next group -> group`, while whole-panel focus remains
`panel houses -> panel clans -> panel groups -> panel`. Do not change the configured key or add a second action.

## Regression coverage

Add focused transition coverage using projected clan rows and real grouping models:

- Reproduce the screenshot under `BY_STATUS`: focus a collapsed clan in an expanded split tribe-panel `Running` group,
  leave a sibling clan open, press `H`, and assert the sibling clan closes while the status banner and selected clan row
  remain in place; press again and assert only then that `Running` collapses.
- Prove the full precedence ladder with mixed open workflow/family houses and open clans: houses close first, all valid
  clans in the target group close on the next press, and the grouping banner closes only after both structural rungs are
  saturated.
- Cover multiple open clans in one target, an already-closed clan, a clan in a sibling status/date/name group, an
  equal-key group in another tribe panel, and merged layout. Assert exact group/panel isolation and stable target order.
- Cover focus on a direct clan member that disappears, verifying re-anchor to the visible clan container, panel
  selection-memory repair, one refilter with `refresh_content_index=False`, and no group-fold persistence. Also cover a
  surviving sibling selection and focused collapsed subgroup escalation.
- Cover malformed container metadata and duplicate global fold owners. Reject ambiguous candidates without mutating
  their keys, while still collapsing valid siblings; if no valid open clan remains, preserve the existing
  selected-structural/group fallthrough.
- Extend footer integration tests to prove row-focused resolver precedence and `collapse clans` versus `collapse group`,
  while retaining the existing whole-panel, Tools-detail, custom-key, help, and command-catalog assertions.
- Add or extend one exact ACE PNG interaction scenario matching the supplied shape: a selected closed clan plus an
  expanded sibling clan inside a status group. Assert the pre-action `collapse clans` footer, then the collapsed clan
  and still-open group after `H`. Inspect the generated diff and accept only this intentional Agents golden.

Existing selected-panel clan tests remain regression coverage for the distinct panel-wide ladder and should not be
weakened or rewritten to pass the new group-scoped behavior.

## Validation

Run `just install` first in the ephemeral workspace. Then run the focused group, structural-tree, selected-panel-clan,
footer integration/unit, help, keymap, and command-catalog tests plus the affected exact PNG scenario. Run the dedicated
Agents visual suite without update mode after auditing any intentional golden. Finish with the repository-required
`just check` and `git diff --check`; investigate failures individually and do not accept unrelated snapshot drift.

## Non-goals

Do not change clan projection or grouping ownership in Rust core, lowercase parent navigation, clan expansion with `l`,
grouping order or persistence, whole-panel selection/isolation, panel-wide clan batching, or `H` behavior on
ChangeSpecs, Axe, and Tools detail.
