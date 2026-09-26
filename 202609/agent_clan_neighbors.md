---
tier: tale
title: Add numeric jump targets between neighboring agent clans
goal:
  Selecting an agent clan exposes other clan nodes in the same agent hood as numbered
  jump targets, including reversible jumps between sase-1ao.3 and sase-1ao.
size: medium
proposed_by: bbugyi200.apollo.22
create_time: 2026-09-26 17:24:24
status: wip
---

# Agent clan neighbor jumps

## Current behavior and scope

The Agents tab already gives a selected clan a `CLAN MEMBERS` roster and publishes its
numbers to the sticky jump panel. Ordinary sase agent nodes have a separate `NEIGHBORS`
roster backed by the cached Agents-tab neighbor index. That index deliberately excludes
synthetic clan containers from agent-name kinship, so the visible `sase-1ao` clan cannot
be reached by a number while `sase-1ao.3` is selected. The existing member-jump path
already selects clan containers, reveals targets through folds and panels, and records
jump-back history.

Add a distinct `CLAN NEIGHBORS` section for a selected clan. Two clan nodes are
neighbors when their _presented clan names_ belong to the same dotted hood: `foo`,
`foo.bar`, `foo.baz`, and `foo.bar.deep` share the `foo` hood. Compare normalized names
case-insensitively at dot boundaries. Exclude the selected identity, ordinary
agent/session/turn rows, malformed names, and unrelated prefixes such as `foobar`.
Preserve each target's complete stable clan identity, including its generation, so a
rendered number never resolves merely by name. Show the section only when another
eligible clan node is available in the current Agents-tab navigation projection; respect
the current query and the existing renderable/revealable visibility rules.

## Implementation

1. Extend the cached Agents-tab neighbor discovery in
   `src/sase/ace/tui/models/agent_hoods.py` and its app adapter in
   `src/sase/ace/tui/actions/agents/_neighbors.py` with a clan-specific projection.
   Reuse the existing in-memory row walk and extend its cache fingerprint where needed
   to track presented clan names and identities; do not turn clan containers into
   ordinary sase-agent `NEIGHBORS` targets. Return deterministic render-order clan
   targets across panels, with a cheap identity-based lookup for keypress revalidation.
   Avoid disk, config, or subprocess work in rendering and key handling.

2. Adapt those clan targets into shared `MemberRosterEntry` rows and append
   `CLAN NEIGHBORS` after `CLAN MEMBERS` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`. Use one
   `MemberJumpNumbering` for both sections, merge their `MemberJumpMap`s, and publish
   the merged map to the existing sticky jump panel. Keep the established
   one-digit/two-digit threshold and shared 100-target cap, including an unnumbered
   overflow count. Use clear full-clan labels and a distinct section accent; the
   numbering and roster must agree at every clan fold level and in both detached and
   normal detail rendering. Thread the projection through the immediate header, full
   render, and hint render paths, and include it in the clan hint-cache context so
   refreshing, filtering, or changing neighboring clans cannot leave stale displayed
   numbers.

3. Add a dedicated `clan_neighbor` target role to the shared roster/navigation types in
   `src/sase/ace/tui/widgets/prompt_panel/_member_roster.py` and
   `src/sase/ace/tui/actions/navigation/_member_jump.py`. Before acting on a number,
   verify that the target remains a current clan neighbor of the selected clan. Reuse
   `_reveal_agent_row` for cross-panel/fold navigation and the existing jump history for
   `Ctrl+O`; cancel a stale relation or missing target without moving selection. Keep
   member and ordinary neighbor/revive behavior intact.

4. Update `docs/ace.md` and `docs/agent_sessions.md` to describe the new section, its
   hood rule, section order, numbering/cap, and jump-back behavior.

## Acceptance and verification

- With `sase-1ao.3` selected, `CLAN NEIGHBORS` contains `sase-1ao`; its digit lands on
  that clan, and the reciprocal roster on `sase-1ao` lands back on `sase-1ao.3`.
  `Ctrl+O` also returns to the previous selection.
- A clan with members and neighbors shows unique contiguous numbers across both groups;
  eleven combined entries use two digits, and entries beyond the shared 100 slots are
  explicitly unnumbered. The sticky panel's collapsed, expanded, and narrowed views
  match the published map.
- Tests cover root/sibling/deeper hood membership, dot boundaries, case normalization,
  exclusion of non-clans and self, stable generation identities, cross-panel and
  folded-target reveal, query/fold changes, stale-map cancellation, immediate/full/hint
  rendering, and hint-cache invalidation. Extend the focused model, roster, navigation,
  and panel tests under `tests/ace/tui/`, using a visual snapshot only if the new group
  changes a snapshot's intended contract.
- Run the focused tests for the changed components, then the repository's default
  `just check` gate. Diagnose any failure caused by this change before declaring the
  implementation complete.
