---
tier: tale
title: Isolate clan-member fold state in the Agents tab
goal: 'Expanding a clan reveals only its direct agents and agent families, while each
  member independently owns the expansion and collapse of its ordinary and hidden
  child rows.

  '
create_time: 2026-07-18 06:31:51
status: done
prompt: 202607/prompts/clan_member_fold_isolation.md
---

# Plan: Isolate clan-member fold state in the Agents tab

## Context and diagnosis

The `sase-6n` clan work introduced the intended three-level Agents-tab shape, but the current projection flattens every
row in a clan onto the synthetic clan's fold key. As a result, one `l` on a clan reveals ordinary descendants of every
member, and another `l` on any selected member advances the shared clan key and reveals hidden/family descendants across
the entire clan. The same flattening makes the clan fold annotation count grandchildren that belong to individual
members.

The tree already has the identities needed for independent folds: a synthetic clan key, each direct member's
`raw_suffix`, and each child row's `parent_timestamp`. Keep fold state in the existing in-memory `FoldStateManager`;
correct which node owns each edge and make visibility depend recursively on every ancestor. No wire, artifact,
configuration, or persistence-format change is needed.

## Interaction contract

Treat the clan container as a binary outer fold and each foldable member as its own existing three-state fold:

1. With the clan collapsed, only the clan row is visible.
2. `l` on the clan reveals only its direct agents/family roots. All member child rows remain hidden, regardless of
   previously stored member fold levels.
3. `l` on one selected member changes only that member from `COLLAPSED` to `EXPANDED`, revealing its ordinary child rows
   while peer members remain folded.
4. A second `l` on that member changes only it to `FULLY_EXPANDED`, adding its hidden workflow steps. Family/follow-up
   rows are ordinary children of their family and therefore appear at `EXPANDED`, not as clan-wide hidden rows.
5. `h` reverses the selected member one level at a time. If a disappearing child was selected, focus returns to its
   immediate member parent. An additional `h` from a folded direct member may collapse the enclosing clan and re-anchor
   on the clan row, preserving the existing walk-up behavior before outer group folds.
6. A second `l` while the clan row itself remains selected must not reveal member descendants; only selecting a member
   can change that member's fold.

Refreshing, filtering, switching panels, or rebuilding the synthetic projection must preserve these independent
in-memory fold states and must never expose a descendant while its clan or member ancestor is collapsed.

## Tree ownership and visibility

Update the pure clan projection in `src/sase/ace/tui/models/_agent_tree.py` so `tree_parent_key` identifies the
immediate rendered parent rather than always the clan: direct members point to the clan key, while depth-two
workflow/family rows point to their real member parent's suffix. Keep `tree_depth` and artifact linkage unchanged, and
make the fold-key helper distinguish a row's own descendant fold from the fold that controls the row as a child. Ensure
`tree_parent_lookup`, `tree_parent`, and query-driven `filter_tree_rows` still traverse the complete clan -> member ->
child chain, including ancestor retention for descendant matches.

Refactor `src/sase/ace/tui/models/_fold_filter.py` around immediate-child buckets and recursive ancestor gating:

- clan state controls only direct members;
- member state controls only that member's immediate children;
- `EXPANDED` admits ordinary children and `FULLY_EXPANDED` additionally admits `is_hidden_step` children;
- a child's visibility requires every ancestor edge to be open, so a remembered expanded member cannot leak through a
  collapsed clan;
- fold counts are immediate and per owner, so the clan annotation describes direct members and each member annotation
  describes its own ordinary/hidden children;
- preserve existing orphan and hidden-only-workflow behavior outside clans.

Keep the filtering path pure and linear-time over the already loaded rows so both the synchronous `_refilter_agents()`
fast path and the background fold-snapshot path continue to share exactly the same result without I/O or event-loop
work.

## Fold actions, annotations, and focus

Adjust `src/sase/ace/tui/actions/agents/_folding.py` to choose a fold target by the selected node's role: the clan's
outer key for a clan row, the member's own suffix when it owns children, and the immediate parent's suffix for a
selected child. Clamp clan expansion at its direct-member state instead of advancing it to a clan-wide `FULLY_EXPANDED`
state. Update collapse walk-up and re-anchoring so `h` on member descendants lands on the member, while collapsing the
outer clan lands on its synthetic container; do not fall through prematurely to project/ChangeSpec group folding.

Make `_compute_visible_parents` and fold-annotation inputs in `src/sase/ace/tui/widgets/_agent_list_build.py`
level-aware. Hidden indicators and `+N`/`-N` suffixes belong to the member whose hidden steps they describe, not to the
clan. Retain current indentation, rendering, detail-panel, runtime, kill, dismiss, panel `L`/`H`, and keymap behavior
unless a regression test demonstrates that a tree-ownership correction requires a narrowly scoped adjustment.

## Regression and visual coverage

Replace the shared-clan fold assumptions in `tests/ace/tui/models/test_agent_tree.py` with assertions for the immediate
parent chain, per-node counts, recursive visibility, and independent peer state. Cover at least two foldable members so
the tests prove that expanding or fully expanding one does not expose the other's children, and prove that collapsing
the clan masks remembered expanded member state until the clan is reopened.

Extend `tests/ace/tui/test_agent_fold_transitions.py` with action-level sequences for clan `l`, member `l/l`,
child/member `h/h`, outer-clan walk-up, and cursor re-anchoring. Include both a sequential family/follow-up child and a
workflow with an ordinary agent step plus a hidden bash/python step. Add or update query/tree tests if needed to lock
ancestor retention after the immediate-parent correction.

Revise the clan fixture and PNG sequence in `tests/ace/tui/visual/test_ace_png_snapshots_agents.py` so the goldens
visibly prove:

- collapsed clan;
- expanded clan with direct members only;
- one selected member expanded with only its ordinary children;
- that member fully expanded with hidden steps while a peer remains folded.

Inspect generated actual/expected/diff artifacts before accepting intentional PNG changes. Preserve unrelated clan-panel
goldens.

## Validation

Run `just install` before repository checks, then exercise the focused model, transition, query (if changed), and visual
snapshot tests. Regenerate only the intentional clan-tree PNG goldens, rerun `just test-visual`, and finish with the
required `just check`. Confirm from the focused tests that `l`/`h` use only cached in-memory refiltering and add no new
refresh, disk, subprocess, timer, or async work to the keystroke path.

## Non-goals

Do not change clan launch/runtime aggregation, wire schemas, artifact metadata, agent-family naming, kill/dismiss
semantics, uppercase panel folding, default keymaps, documentation, memory files, or generated provider instruction
shims.
