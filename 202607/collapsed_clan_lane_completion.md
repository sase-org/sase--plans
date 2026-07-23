---
tier: tale
title: Complete lanes nested under collapsed agent clans
goal: Prompt agent-target completion includes every otherwise-visible agent lane hidden
  only by a collapsed outer clan, while preserving existing ordering, filtering, grouping,
  and prompt responsiveness.
create_time: 2026-07-23 14:34:59
status: done
---

- **PROMPT:** [202607/prompts/collapsed_clan_lane_completion.md](prompts/collapsed_clan_lane_completion.md)

# Plan: Complete lanes nested under collapsed agent clans

## Context

ACE builds the shared agent-target catalog for `%wait` and agent-typed xprompt arguments such as `#fork` from
`visible_agent_completion_agents()`. That function currently snapshots only rows rendered by each `AgentList` (or the
equivalent `_agents` fallback). A collapsed clan renders its synthetic container but not the real direct lane roots
beneath it. Candidate construction can therefore still derive the aggregate clan target from the container's
`runtime_children`, but it never receives the hidden lane roots needed to emit their individual agent or family targets.
The same omission reaches both prompt syntaxes because they intentionally share `visible_agent_completion_candidates()`
and the prompt text area's per-menu candidate snapshot.

The Agents tab already has a read-only `prospective_clan_projection()` for navigation. It temporarily relaxes only
collapsed outer clan folds in a copied fold-state snapshot, then reapplies active search, grouping folds, split/merged
tribe assignment, panel state, inner workflow/family folds, dismissal state, and STARTING-row exclusion. Reusing that
projection gives completion the intended "visible if the clan alone were opened" semantics without mutating the UI or
inventing a second clan traversal. This is presentation-only TUI behavior and does not cross the Rust core backend
boundary.

## Implementation

1. Refactor `src/sase/ace/tui/_agent_completion_visibility.py` so every existing rendered-row source (mounted panel
   widgets, `_agents_visible_order()`, and the renderable `_agents` fallback) first produces one identity-deduplicated
   base snapshot instead of returning early from separate branches. Preserve the current all-panel behavior and graceful
   fallback when widgets are absent or stale.
2. Enrich that base snapshot with `prospective_clan_projection(app, complete)` using `_agents_with_children` as the
   complete projected tree and `_agents` only as a compatibility fallback. Add only prospective real rows hidden by
   collapsed clan ancestry, retain the synthetic clan containers already present in the base snapshot, and deduplicate
   by stable agent identity.
3. Merge rendered and prospective rows according to the projection's `display_order_by_identity`, with stable base-order
   fallback for identities absent from the projection. This keeps lane, clan, family, tribe, and agent candidate
   ordering consistent with the Agents tab after opening only the relevant clan. Treat a missing fold manager, empty
   complete roster, or empty projection as a no-op so lightweight callers and existing test doubles retain today's
   behavior.
4. Keep the prospective calculation purely in memory and on the existing per-menu snapshot path: do not add filesystem
   reads, subprocesses, resolver calls, fold mutations, list refreshes, or per-character recomputation. Update
   docstrings to distinguish physically rendered rows from completion-eligible rows hidden only by an outer clan fold.
   Leave `_agent_completion_candidates.py` and the `%wait`/`#fork` parsing and insertion code unchanged; once lane roots
   reach the shared catalog, their existing agent/family classification and presented-name handling apply to both
   consumers.

## Tests

- Extend `tests/ace/tui/test_agent_completion.py` with a regression built from the real `project_clan_tree()` and
  `FoldStateManager` projection. Start with a collapsed clan containing both a standalone named agent lane and a
  sequential family lane plus an unrelated visible agent. Assert the completion-visible roster includes the clan
  container, both direct lane roots, and the unrelated row in prospective render order.
- Feed that roster through `build_agent_completion_candidates()` and assert that the collapsed clan still appears
  exactly once as a clan target, the standalone lane appears as an agent target, and the family root appears once under
  its family reference. Keep the family's inner member folded and assert that member-specific completion does not leak
  through the outer-clan relaxation.
- Add coverage for identity deduplication when a clan is already expanded and for preservation of non-clan visibility
  constraints (at minimum active search and STARTING/dismissed exclusion) when prospective rows are considered. Retain
  the existing no-widget and `_agents_visible_order()` fallback tests to prove the enrichment is additive and
  compatibility-safe.
- Run the shared-catalog test plus the focused `%wait` directive and `#fork` xprompt argument interaction suites. These
  consumer tests should pass without consumer-specific production changes, demonstrating that both syntaxes use the
  corrected source catalog.

## Verification

1. Run the focused tests:
   - `pytest tests/ace/tui/test_agent_completion.py -q`
   - `pytest tests/ace/tui/widgets/test_directive_completion_interactions.py tests/ace/tui/widgets/test_directive_arg_completion.py -q`
   - `pytest tests/ace/tui/widgets/test_xprompt_arg_value_completion.py tests/ace/tui/widgets/test_xprompt_optional_spacer.py -q`
2. Run `just install` and then the repository-required `just check`.
3. Review `git diff --check`, the final diff, and `git status --short` to confirm that only the shared completion
   visibility logic and focused tests changed, no fold state is mutated, no memory/provider-shim files changed, and no
   documentation, configuration, or visual snapshot update is needed for the already-documented completion feature.

Planning baseline: `just install` succeeds. `just check` currently passes Python/Markdown formatting plus keep-sorted,
Ruff, mypy, and pyscript lint, then stops at Symvision because the repository's configured `sase-8v(...)` epic-symbol
exemptions reference the already-closed `sase-8v` bead. Re-run the full check after implementation and report that
baseline separately if it remains; do not broaden this focused completion fix into unrelated symbol cleanup.

## Risks and safeguards

- A naive recursive expansion of `container.runtime_children` would expose rows suppressed by search, dismissal,
  STARTING status, group folds, or an inner family/workflow fold. Use the established prospective projection so only the
  collapsed outer clan is relaxed.
- Mounted widgets and the copied projection can contain the same identity when a clan is already expanded or while UI
  state is transitioning. Merge by identity and use stable projected display order to prevent duplicate completion rows
  and aggregate over-counting.
- Completion is a typing path. Reuse the prompt text area's existing one-snapshot-per-menu cache and keep projection
  work bounded to loaded in-memory agents; do not introduce blocking I/O or side-effectful reference resolution.
- Synthetic clan containers must remain in the roster because clan and tribe aggregate candidates derive generation and
  membership from them. Enrichment should add lane roots, not replace containers with descendants.
