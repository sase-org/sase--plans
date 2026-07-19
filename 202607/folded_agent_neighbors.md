---
tier: tale
title: Reveal neighbors across folded clans and tribes
goal: 'The Agents-tab neighbor command discovers every currently relevant agent hidden
  only by clan or tribe folding, and selecting one expands exactly the required containers
  before focusing it safely.

  '
create_time: 2026-07-19 17:26:45
status: wip
prompt: 202607/prompts/folded_agent_neighbors.md
---

# Plan: Reveal neighbors across folded clans and tribes

## Context

The Agents-tab `~` action currently builds its hood/ancestor/descendant index from `_agents`, the already-folded list
that feeds the rendered tribe panels. A collapsed tribe panel remains represented in that list and is therefore already
searchable, but collapsing a clan removes its real member rows and leaves only the synthetic clan container. Those
members never enter the neighbor index, so the neighbor badge under-counts them and the chooser/direct-jump path cannot
select them. This also masks clan members whose containing tribe panel is collapsed.

Keep this work in the Python TUI layer: the dotted-name relationship rules are presentation/navigation behavior, and the
shared agent-reveal path already knows how to expand a target's tree ancestry, enclosing grouping banners, and tribe
panel by stable identity. The implementation should reuse the in-memory prospective-clan projection semantics already
used by unread-agent jumps rather than introducing disk access, an alternate refresh pipeline, or Rust core behavior.

## Revealable neighbor projection

- Define an identity-stable neighbor target/row contract that can describe both a currently rendered row and a real
  agent that would render after its collapsed outer clan fold is expanded. Preserve a current global index only when one
  exists; carry the agent identity/object, effective tribe panel key, hood metadata, and deterministic display order
  independently so hidden members never require fake `_agents` indices.
- Build the index from the union of the current rendered rows and eligible rows in `_agents_with_children` that are
  hidden only by collapsed clan ancestry. Derive eligibility with a side-effect-free prospective fold projection: relax
  the relevant clan folds in copied in-memory state, then apply the same active search, STARTING-row exclusion,
  split/merged tribe assignment, per-panel grouping folds, and tree visibility rules used by the actual Agents view.
  Continue traversing collapsed tribe panels so a member can be offered even when both its clan and `@tribe` panel are
  collapsed. Do not expose dismissed agents as ordinary neighbors, rows filtered by search, or workflow/family rows
  still hidden behind their own inner fold.
- Extract or generalize the existing prospective-clan helper used by unread jumps so unread and neighbor navigation
  agree about which hidden clan members are revealable. Keep the helper pure and in-memory, deduplicate visible and
  prospective rows by stable identity, and preserve the established panel/render ordering and deepest-first hood
  grouping in the chooser.
- Extend the existing neighbor-index cache rather than recomputing the projection on every info/footer refresh. Its
  signature and invalidation must cover the complete roster plus tree, grouping, panel, query, and dismiss/revive state
  that can change candidate membership. Repeated neighbor-count reads for an unchanged view should reuse one index,
  maintaining the keystroke-path performance contract.

## Stable selection and targeted expansion

- Drive the single-result fast path, modal payloads, panel labels, and counts from the identity-stable target records.
  The info badge/footer count should include revealable folded targets, and modal rows for hidden members should show
  the correct `@tribe`/`(no tribe)` label even though no current global index exists.
- Preserve the stricter current-row preflight for already-visible targets, while allowing a prospective clan target to
  resolve from `_agents_with_children`. Revalidate its identity and revealability when a modal choice is accepted so a
  stale, removed, newly filtered, or ambiguous target cancels without selecting an unrelated row or acknowledging it.
- Use the shared reveal transaction to expand only the selected target's validated clan/tree ancestor chain, perform one
  lightweight in-memory refilter, recompute its actual index and panel, expand only its enclosing collapsed group
  banners if necessary, and finally expand/focus its tribe panel. Leave unrelated clans, groups, and tribes folded.
  Retain one structural repaint for any reveal, the selective repaint path for already-visible jumps, immediate
  highlight behavior, debounced detail updates, unread acknowledgement, and stable back/forward jump history.

## Verification

- Add model tests for mixed visible/prospective target indexing, identity deduplication, deterministic cross-panel hood
  order, and unchanged ancestor/descendant/duplicate-name semantics.
- Extend the real folded-projection navigation harness (using `project_clan_tree` plus `filter_agents_by_fold_state`) to
  prove that a sole hidden clan neighbor is counted and direct-jumped, multiple hidden neighbors appear in the chooser,
  and choosing one expands the exact clan before focus.
- Cover the combined case requested here: the target lives in a collapsed clan inside a collapsed tribe panel. Assert
  that it is listed with the right panel label, both layers expand on selection, the target is focused, one in-memory
  refilter/structural refresh occurs, and unrelated collapsed clans/tribes remain untouched. Include same-name clan
  generations so only the target generation opens, and retain an explicit tribe-only regression proving that a neighbor
  in a collapsed tribe remains discoverable even without clan ancestry.
- Preserve negative boundaries with tests for active search filters, STARTING rows, inner family/workflow folds,
  collapsed grouping banners, dismissed agents, stale modal callbacks, artifact-viewer guards, and split versus merged
  tribe-panel modes. Verify the neighbor badge/footer count uses the same cached candidate set as `~` and that cache
  invalidation follows fold, query, panel, refresh, and dismiss/revive changes.
- Add or update the focused Agents interaction PNG coverage so the chooser and post-jump layout exercise a neighbor in a
  collapsed clan within a collapsed tribe. Run `just install` before the targeted neighbor/model/navigation/visual
  tests, then run the required `just check` before handoff.

## Risks and constraints

- Do not build the prospective projection from the raw roster alone: doing so would leak search-filtered or otherwise
  non-renderable agents into the chooser. The projection must answer the narrower question, “would this row render if
  clan folding alone stopped hiding it?”
- Avoid coupling modal callbacks to numeric positions. Expanding a clan or tribe can repartition panels and replace
  `_agents`, so identity must remain authoritative from discovery through final focus.
- Keep all discovery synchronous but bounded to cached in-memory structures; no filesystem reads, subprocesses, full
  reloads, or slow async work belong on the `~`, footer, or info-panel path.
