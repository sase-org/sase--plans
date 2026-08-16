---
tier: tale
title: Count and mark only agent nodes on the Agents tab
goal:
  The Agents tab counts and marks agent nodes exactly once without leaking family-member
  shell status or unread state.
size: medium
proposed_by: bbugyi200.athena.046
create_time: 2026-08-16 15:17:28
status: wip
---

# Count and mark only agent nodes on the Agents tab

## Goal

Make the Agents tab honor the node taxonomy consistently: an agent family is one agent
node, a standalone agent shell is an agent node, and a shell inside a family is only an
agent-shell node. Status summaries and unread state must therefore count or mark agent
nodes exactly once. Family rows must never render member-status chips such as `[W4 U1]`,
and family-member shell rows must never render unread check/X markers.

Keep clan and panel aggregation: a clan row, tribe/panel title, or top info strip may
show a status-count chip, but each value in that chip must come from direct agent nodes
(families count once; their member shells do not contribute separately). Completion
notifications may remain shell-addressed in the Notifications UI; this change only
normalizes how those notifications project onto the Agents tab.

## Diagnosis

- The list renderer computes `clan_member_counts()` for every row. The helper accepts
  any row with `agent_clan` metadata, so a family root inside a clan accidentally
  aggregates its own `runtime_children`. That turns family shells into a status chip on
  the family row and lets a shell-level unread identity become `[U1]`.
- `_unread_completed_agent_ids` is reconciled against every loaded real row. It can
  therefore contain family-member shell identities, and render, manual-toggle, bulk
  read, and unread-jump paths treat those identities as if they belonged to agent nodes.
- A shell that completed before its row became a non-terminal family container can also
  leave stale unread state behind: reconciliation skips non-terminal rows instead of
  normalizing unread state at the owning agent-node boundary.
- The tribe detail snapshot deliberately builds concrete-shell status counts for
  sequential families and propagates unread state to nested shell entries, extending the
  same taxonomy violation beyond the main list.

## Implementation

1. Add one pure, presentation-independent Agents-tab predicate/projection for agent
   nodes. It must classify family containers and standalone root agents as agent nodes,
   while excluding clan containers, family-member shells, workflow/step children, and
   monitor/proc-shell rows. Base the decision on semantic ownership/linkage rather than
   visual `tree_depth`, because a standalone agent nested under a clan is still an agent
   node. Add a truth-table unit test covering these row kinds.

2. Reconcile completion notifications into node-level unread state.
   - Build the projection from the complete loaded roster, not the current folded or
     filtered rows. For a standalone agent node, match its own completion key. For a
     family node, match the completion keys of the concrete shells it owns, using the
     existing family-member projection rather than reconstructing family membership.
   - Store only owning agent-node identities in `_unread_completed_agent_ids` and
     `_manual_unread_agent_ids`. A node may be notification-unread only while its
     effective node status is terminal; clear stale member-shell and non-terminal
     identities during reconciliation while preserving the existing manual suppression
     and manual-unread rules for eligible nodes.
   - Make manual toggle, bulk read/undo, automatic acknowledgement, and unread jump
     candidate discovery reject non-agent nodes. Entering or expanding a family-member
     shell must not expose or consume a hidden shell-level unread marker.
   - Acknowledging or bulk-reading a family node must dismiss every active completion
     notification that backs that family projection and update the in-memory
     notification snapshot. Keep selective repainting: patch the owning node and any
     affected visible clan ancestor, then refresh scoped titles/info/summary; retain the
     existing full-rebuild fallback only when a row patch cannot be applied.

3. Restrict status-count chips to real aggregate surfaces.
   - Make clan membership/count helpers accept actual clan containers (plus the existing
     legacy parallel-family-as-clan compatibility case), never an ordinary family root
     merely because it carries clan metadata.
   - In the agent-row formatter and render cache, calculate and fingerprint member
     counts only for clan container rows. Rename remaining `parallel_family_*`
     presentation variables/tests to clan terminology where they are no longer true
     compatibility APIs. Family rows keep their structural fold annotation and their
     single effective status, but no `[S/R/Q/W/F/U/D]` chip.
   - Gate the right-hand unread marker on the semantic agent-node predicate so member
     shell, workflow-step, and monitor rows cannot render `✅`/`❌`, even if stale input
     is supplied by a caller.

4. Align summary/detail projections with the same boundary.
   - Keep the top info strip, panel/tribe border titles, clan rows, and clan detail
     headers on the deduplicated agent-node projection: each standalone agent is one
     count and each family is one count with its effective family status/unread state.
   - Stop attaching concrete-shell status-count bundles to family units in tribe detail
     rosters. Clan units may retain aggregate chips, but their counts must be computed
     from direct agent nodes. Family units may show their one node-level unread marker;
     nested family-shell entries must never show unread markers.
   - Remove or isolate any concrete-shell summary helper that is no longer used by an
     Agents-tab presentation path so future callers cannot accidentally reintroduce
     shell counts.

5. Add regression coverage around the reported shape and the state transitions that can
   recreate it.
   - Render a family inside a clan with waiting/done member shells and shell-addressed
     completion notifications; assert that the family has no status-count chip, member
     rows have no unread marker, and a non-terminal family contributes no unread agent
     node.
   - Finish the family and assert that it becomes one unread agent node (not one per
     shell), appears once in clan/panel/global counts, can be jumped to and acknowledged
     at the family row, and dismisses all backing family completion notifications.
   - Preserve standalone-agent unread behavior, collapsed-clan ancestor patching, manual
     unread/undo behavior for valid agent nodes, queued/waiting partitioning, and legacy
     clan projection coverage.
   - Update cache tests to prove shell unread/count changes do not invalidate or widen
     unrelated family rows, while node/clan changes still invalidate the exact cached
     rows that display them.
   - Add or update the focused Agents-tab PNG snapshots for collapsed/expanded clan and
     family views. Inspect actual/expected/diff artifacts before accepting only the
     intentional removal of family chips and shell unread glyphs; retain clan/panel
     aggregate chips.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral SASE
   workspace.
2. Run focused model, notification/unread, navigation, row-render/cache, panel-summary,
   tribe-detail, and family/clan roster tests while iterating.
3. Run the relevant visual cases first, then `just test-visual`; use
   `--sase-update-visual-snapshots` only after inspecting the generated actual,
   expected, and diff images for the intended changes.
4. Run `just check` and resolve every lint, typing, and diff-scoped test failure. If
   test selection escalates or reports an unusual selection, follow with
   `just check-full` through `/sase_monitor` and provide its required `--next` action.

## Acceptance criteria

- No agent-family row displays a member status-count chip, including `[U…]`.
- No family-member shell, workflow-step, or monitor/proc-shell node displays or accepts
  an Agents-tab unread marker.
- A terminal family with active member completion notifications is represented by at
  most one unread agent-node identity and one unread contribution in every aggregate; a
  non-terminal family contributes none.
- Acknowledging the family node clears its projected unread state and all backing family
  completion notifications without requiring a normal full-list rebuild.
- Clan rows and panel/tribe/global summaries retain compact status chips, but count
  direct agent nodes only and never inflate totals/statuses from family shells.
- Focused and visual regression tests cover collapsed and expanded family-in-clan
  layouts, and `just check` passes.
