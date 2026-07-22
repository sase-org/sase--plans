---
tier: tale
title: Prefer group-wide agent-house collapse before grouping collapse
goal: 'Uppercase H fully collapses every open agent house in the next Agents-panel
  grouping scope before a later press collapses that grouping banner.

  '
create_time: 2026-07-22 07:27:44
status: done
---

- **PROMPT:** [202607/prompts/agents_group_house_collapse_precedence.md](prompts/agents_group_house_collapse_precedence.md)

# Plan: Prefer group-wide agent-house collapse before grouping collapse

## Context and confirmed root cause

The two reference screenshots define a two-press contract in the split `@default` panel while grouping by status. In the
first screenshot, focus is on the already collapsed standalone `hu` house inside `Running`, while sibling houses such as
`ht` and `hs` are expanded. One `H` should leave the `Running` banner expanded and fully collapse all of those open
houses. With no open house folds left in that group, the next `H` should perform today's grouping-fold transition and
collapse `Running`.

The current dispatcher in the Agents fold actions cannot express that contract. After Tools-detail precedence, it asks
the selected-row-only structural resolver for one workflow/family/clan fold. Because `hu` owns no open fold, that
resolver returns no target and dispatch immediately falls through to the enclosing `Running` grouping fold. The grouping
model already knows the exact focused panel, deepest enclosing/next-parent group, and complete member indices, but the
`H` path never uses that scope to inspect sibling house owners. This is why the behavior depends on which row happens to
be selected even though the requested operation is group-wide.

This is presentation-only Agents-tab behavior and remains in the Python TUI boundary. It must stay a pure in-memory
keypress path: no disk reads, subprocesses, awaits, or full data reloads.

## Behavioral contract and precedence

Treat the grouping banner that the existing group resolver would collapse next as the scope for one group-wide
house-collapse attempt. For an agent row this is its deepest visible grouping container; for an already-collapsed child
banner it is the parent banner reached by the existing repeated-`H` ladder. Scope is always limited to the focused tribe
panel (or the single merged panel) and must not affect equal group keys in sibling tribe panels.

Within that scope, enumerate canonical **agent house** owners: standalone workflow/agent roots and sequential-family
roots that own a real descendant fold. Include houses nested as direct clan members, but do not treat synthetic clan
containers, workflow-step suffix aliases, family members, or malformed duplicate owners as independent houses.
Deduplicate by validated canonical fold key. A house is open whenever its fold level is not `COLLAPSED`.

Make one `H` atomic and saturating for those houses: drive every open house in the scoped group directly to `COLLAPSED`,
whether it began `EXPANDED` or `FULLY_EXPANDED`. Do not use the legacy level-wise `collapse_all()` semantics, which
intentionally stops fully-expanded folds at the intermediate level. Apply all mutations before one lightweight in-memory
refilter, avoid refreshing the content-search index for a fold-only change, and never repaint once per house.

The Agents-tab `H` order should become:

1. Preserve the existing Tools-detail override and whole-panel-focus behavior.
2. If the next grouping scope contains any open canonical house, fully collapse all such houses and stop; do not
   collapse the grouping banner in that press.
3. If no house changed, retain any distinct selected structural transition that is not subsumed by house collapse
   (notably the binary clan layer).
4. Only then run the existing grouping-banner collapse/escalation.

This preserves the workflow/family → clan → group hierarchy while changing a workflow/family step from selected-house,
one-level behavior to group-wide, fully-collapsed behavior. It also makes selection-independent cases such as the
screenshots deterministic. Lowercase `h`/`l`, numeric fold selection, `Z` panel isolation, grouping-fold persistence,
and other tabs keep their current contracts.

## Scope resolution, focus, and refresh safety

Add a shared resolver/result for the next group-wide house-collapse action and use the same result for key dispatch and
footer availability. Derive membership from the focused panel's grouping tree and its `GroupRow.agent_indices`, using
the existing anchored grouping rules so workflow steps and clan/family members cannot drift into a different
status/date/project group from their outer presentation root. Resolve the full membership of the target even when a
child group banner is currently collapsed, without mutating grouping state during the probe.

If the selected row would disappear because its house is being collapsed (including agent, Bash, Python, hidden
pre-prompt, and family-member rows), move selection to that house's visible canonical owner before the single refilter.
Otherwise preserve the selected identity, current panel, grouping context, jump history, and panel selection memory.
After refiltering, invalidate only the normal navigation/display caches already invalidated by fold changes; the group
banner remains open and no group-fold persistence intent is emitted on the first press.

Malformed or incomplete ancestry must fail closed for that candidate rather than mutating an unrelated fold. Other valid
houses in the same group may still collapse. If no validated open house remains, fall through to the existing
structural/group behavior.

## Affordances and documentation

Drive the conditional footer from the shared resolver so the first screenshot's state advertises `H collapse houses`,
while the post-collapse state advertises `H collapse group`. Preserve `H compact tools` and the distinct clan label
where those actions have precedence. Update the Agents help text, command/binding metadata, and the Agents folding
documentation to describe the saturating group-scoped house step and repeated-`H` ladder. Do not change the configurable
key itself.

## Regression coverage

Add focused action tests using loader-shaped rows and real grouping projection:

- Reproduce the screenshots under `BY_STATUS`: select a closed standalone house in `Running`, leave sibling workflow and
  sequential-family houses at mixed `EXPANDED`/`FULLY_EXPANDED` levels, press `H` once, and assert every valid house is
  fully collapsed while `Running` remains open; press again and assert only then that `Running` collapses.
- Cover selected agent, Bash, Python, hidden pre-prompt, and family-member rows; verify disappearing selections
  re-anchor to the correct visible house owner and the bulk action still collapses sibling houses.
- Cover a direct clan-member house plus the separate binary clan layer, nested name/date/project groups, a focused
  collapsed subgroup escalating to its parent, split and merged layouts, and identical group keys in another tribe
  panel. Assert only the exact target group/panel changes.
- Cover mixed fold levels, already-collapsed houses, suffix aliases, duplicate or malformed owners, and a scope with no
  open houses. Assert one refilter for a changing bulk action, no content-index refresh request, no group persistence on
  the first press, and unchanged fallthrough behavior when there is nothing to bulk-collapse.
- Preserve Tools-detail priority, whole-panel `H` no-op/isolation behavior, lowercase navigation/expansion, numeric fold
  selection, and non-Agents-tab dispatch through existing regression suites.
- Update footer/help/command metadata assertions for `collapse houses` versus `collapse group` and add one focused
  Agents PNG interaction snapshot matching the supplied before/after layout. Generate or refresh only intentional Agents
  goldens and inspect every diff before acceptance.

## Validation

Run `just install` before repository checks, then execute the focused structural and grouping transition tests,
footer/help/metadata tests, panel/Tools precedence tests, and the dedicated Agents PNG snapshot. Run the complete Agents
visual snapshot suite without update mode after any audited golden change. Finish with `just check` and
`git diff --check`; investigate failures in isolation and do not accept unrelated visual drift.
