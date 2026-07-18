---
tier: tale
title: Preserve agent-tree membership across grouping and tag panels
goal: Keep every clan and agent-family subtree together under its owning root when
  workflow steps are revealed, regardless of project, status, date, or tag-panel grouping.
create_time: 2026-07-18 07:55:29
status: wip
prompt: 202607/prompts/agent_tree_grouping_precedence.md
---

# Plan: Preserve agent-tree membership across grouping and tag panels

## Context and diagnosis

The clan-member fold isolation change correctly made the clan own only its direct-member edges and each member own its
own ordinary and hidden children. The remaining defect is downstream of fold filtering: after a member is expanded to
show hidden steps, the Agents-tab panel and banner builders derive presentation grouping from only each row's immediate
parent.

That one-hop rule is sufficient for a root workflow and its direct step, but not for a deeper
`clan -> member/agent-family -> workflow step` chain. The clan and its direct members inherit the synthetic clan
container's grouping identity, while the grandchildren inherit the member's status, date, ChangeSpec/name root, and tag.
The flat grouping sort can then emit a second family/name banner, a different ChangeSpec or status/date bucket, or even
a separate tag panel for the revealed steps. Under date grouping it can also sort all depth-one members ahead of
depth-two steps, breaking parent adjacency without adding a banner.

Treat structural membership as the stronger presentation invariant: grouping modes and tag panels may arrange whole root
subtrees relative to one another, but they must never partition or reorder rows inside a clan or agent-family subtree.
Preserve the existing fold ownership, indentation, and artifact relationships; this is a Python/TUI presentation fix
with no Rust-core, wire, artifact, or persistence-format change.

## Presentation contract

For every visible rendered tree:

1. Resolve a row's presentation anchor by following its immediate rendered-parent chain to the outermost available root.
   A clan container anchors all direct members and every revealed descendant beneath them. A standalone workflow or
   agent-family root likewise anchors its member/step descendants.
2. Derive project/ChangeSpec, status, date/time subgroup, dotted-name grouping, and effective tag-panel membership from
   that anchor. Do not insert a banner or panel boundary between an anchored parent and any descendant.
3. Order each anchored subtree as an atomic cluster. Group and date ordering compare roots; after the root is placed,
   retain the existing projected parent/child preorder so a member's ordinary and hidden steps stay directly beneath
   that member rather than collecting after peer members.
4. Keep unrelated roots subject to the selected grouping mode's existing deterministic ordering and folding rules.
   Standalone one-level workflow children should retain their current inherited grouping behavior.
5. Keep the approved clan fold interaction unchanged: clan `l` reveals direct members, member `l/l` reveals that
   member's ordinary/hidden rows, `h` walks back through the same owners, and remembered member state remains masked by
   a collapsed clan.

In split tag-panel mode, the complete subtree uses the root's effective panel. Multi-tag clan containers remain in their
existing aggregate/untagged presentation and continue to render their `clan_tags`; grandchildren must not leak into a
member-tag panel. In merged mode, per-row tag labels must follow the same root-anchored result so switching panel modes
cannot visually detach the steps.

## Shared structural-anchor model

Extend the pure tree utilities in `src/sase/ace/tui/models/_agent_tree.py` with one shared way to resolve presentation
anchors from the already-built `tree_parent_lookup`. Make the batch path memoize resolved anchors so callers remain
linear in the number of loaded rows rather than repeatedly walking the same ancestry. Guard malformed cycles and
missing/orphan parents deterministically, falling back to the highest safely resolved row without hiding otherwise
renderable agents.

Keep this presentation anchor distinct from fold ownership: `agent_fold_key` and `agent_parent_fold_key` continue to
describe the row's own fold and its immediate controlling edge. Do not flatten `tree_parent_key`, `tree_depth`, or the
recursive fold filter back onto the clan. Query ancestor retention and focus walk-up must continue to use immediate
parents.

## Group-tree precedence and ordering

Update `src/sase/ace/tui/models/agent_groups/_keys.py` and the tree builder in
`src/sase/ace/tui/models/agent_groups/_tree.py` to use the outer presentation anchor consistently for grouping keys,
ChangeSpec-level detection, status/date buckets, and time anchors. Ensure `build_agent_tree()` and
`enumerate_group_keys()` share the same anchored keys so visible banners, persisted group-fold keys, jump targets, and
fold enumeration cannot disagree.

Make the ordering operate on anchored root clusters. Preserve the established bucket order between roots, including
terminal stop-time behavior in date mode, but use the existing projected tree order within a cluster. A clan with
multiple direct members and multiple hidden steps under one selected member must therefore render as one contiguous
sequence in `STANDARD`, `BY_STATUS`, and `BY_DATE`; descendant metadata must not create a name-root/prefix group or move
the steps to another root's position. Existing registry entries for descendant-derived groups that no longer render can
remain harmlessly ignored; no fold-state migration is needed.

Keep all of this computation pure and in-memory. Reuse the current parent index during a build, add no disk reads,
subprocesses, timers, asynchronous callbacks, or full data reloads, and retain `_refilter_agents()` as the `l`/`h` fast
path.

## Tag-panel precedence

Update `src/sase/ace/tui/models/agent_panels.py` so effective tags and panel membership use the same outer presentation
anchor as banner grouping. Apply the rule uniformly in panel discovery, per-agent key generation, merged-panel labels,
and `agents_for_panel()`; do not create a parallel ancestry implementation. This ensures the panel index and widget
builder always receive a complete structural subtree and can resolve the same anchor locally.

Preserve current panel ordering, collapse state, focus restoration, tag editing, and unrelated-agent membership. A
missing structural parent should continue to fall back to the row's own tag rather than making the row disappear.

## Regression and visual coverage

Add focused model coverage that reproduces the supplied `sase-6r` shape with a clan container, at least two direct
phase/family members, and multiple ordinary/hidden steps under one member:

- in `tests/ace/tui/models/test_agent_tree.py`, lock the immediate-parent chain separately from the new outer
  presentation anchor and prove fold ownership/counts remain per member;
- in the `tests/ace/tui/models/test_agent_groups_*` suites, assert that all three grouping modes derive descendant keys
  from the outer root, emit no descendant-derived ChangeSpec/status/date/name banner, preserve member-step adjacency,
  and still order whole subtrees correctly against unrelated roots;
- in `tests/ace/tui/models/test_agent_panels.py`, prove a multi-tag clan and all revealed grandchildren occupy one root
  panel in both split and merged calculations, while standalone workflows and unrelated tagged roots retain existing
  behavior;
- retain or extend the action-level `l/l` and `h/h` assertions so the fix cannot regress independent clan/member fold
  state or replace one synchronous refilter with a reload.

Strengthen `tests/ace/tui/visual/test_ace_png_snapshots_agents.py` with the user-visible sequence: expand a clan, select
one member, press `l` twice, and render under `BY_STATUS` with enough revealed steps to trigger the former duplicate
family banner. The final PNG must show those steps directly beneath their member inside the clan, with no second
family/name banner or tag panel. Update the existing fully-expanded clan golden where the corrected root-panel
inheritance intentionally consolidates the subtree. Inspect actual/expected/diff artifacts before accepting any PNG; do
not update unrelated snapshots.

## Validation

Run `just install` before repository checks. Exercise the focused tree, grouping-mode, panel, fold-transition, and clan
visual tests first; regenerate only intentional clan PNG goldens. Then run `just test-visual` and the required
`just check`. Re-run the focused status/date ordering tests after any golden update so visual acceptance cannot mask a
model regression. Confirm the implementation adds no I/O or async work to `l`/`h` and that the synchronous refilter
still performs only bounded in-memory indexing, anchor resolution, sorting, and rendering.

## Non-goals

Do not change clan launch/runtime aggregation, fold levels or key semantics, agent-family naming, artifact metadata,
status calculation, query semantics, kill/dismiss behavior, tag editing, panel/group keymaps, Rust bindings, memory
files, or generated provider instruction shims.
