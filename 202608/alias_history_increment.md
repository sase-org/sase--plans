---
tier: tale
title: Use ten-run steps in model alias history
goal:
  Make Ctrl+J and Ctrl+K adjust only the Launch Control model-alias history window by 10
  runs while preserving the global 100-row ACE page size everywhere else.
size: small
proposed_by: bbugyi200.athena.08i
create_time: 2026-08-20 07:40:45
status: wip
---

# Plan: Use ten-run steps in model alias history

## Context

The Launch Control alias-history modal currently reads `ace.page_size` through
`get_ace_page_size()` and passes that value to `adjusted_alias_history_limit()`. Because
the bundled global page size is 100, `Ctrl+J` and `Ctrl+K` change the per-alias history
window by 100. That coupling was introduced when load-more panels were standardized, but
alias history needs a deliberately smaller, fixed increment of 10. The initial window
must remain controlled independently by `llm_provider.model_alias_history_limit`, and
`ace.page_size` must remain 100 by default for Artifacts, prompt history,
dismissed-agent revival, and its other existing consumers.

## Implementation

1. In `src/sase/ace/tui/modals/alias_history_state.py`, make the alias-history limit
   adjustment own an explicit 10-run step rather than accepting the global ACE page
   size. Keep the existing semantics in both directions: load-more adds exactly 10,
   unload subtracts exactly 10, and unload clamps at the independently configured
   initial history limit. Name and document the local invariant so it cannot be mistaken
   for `ace.page_size` or the initial limit.
2. In `src/sase/ace/tui/modals/alias_history_modal.py`, remove the `get_ace_page_size()`
   dependency and any modal state derived from it. Route both priority `Ctrl+J` /
   `Ctrl+K` actions through the alias-history-specific adjustment, while preserving
   cached worker-backed reloads, the no-op at the initial window, selection retention,
   and the last-row fallback after unloading.
3. Update `tests/test_alias_history_state.py` and `tests/test_alias_history_modal.py` to
   assert 10-run load/unload transitions, including a configurable initial limit that is
   distinct from the step. Remove monkeypatches that imply alias history follows
   `ace.page_size`. Retain coverage for cached reload freshness, the initial-window
   no-op/floor, selection behavior, and usage-strip recomputation.
4. Correct the alias-history key descriptions in `docs/ace.md` and clarify the
   `ace.page_size` entry in `docs/configuration.md`: the history modal uses a fixed
   10-run step, while `ace.page_size` continues to default to 100 and govern the other
   paged ACE surfaces and the default Artifacts limit. Do not change
   `src/sase/default_config.yml`, `src/sase/config/sase.schema.json`, prompt-history
   paging, revive-history paging, or Artifacts paging.

## Verification

1. Run the focused alias-history state and mounted-modal tests to verify
   `10 -> 20 -> 10`, a non-default initial window such as `25 -> 35 -> 25`, cached
   reload requests, and the unload floor/no-op behavior.
2. Run the ACE page-size schema/default tests and the existing prompt-history and
   revive-history paging tests as regression guards that `ace.page_size` still defaults
   to 100 and remains in use outside alias history.
3. Run `just install` if the workspace dependencies are not current, then run
   `just check` for the required whole-repository lint gates and diff-scoped test lane.

## Acceptance Criteria

- In the model alias history panel, each `Ctrl+J` press increases the per-alias query
  limit by exactly 10 and each effective `Ctrl+K` press decreases it by exactly 10.
- `Ctrl+K` at the initial `model_alias_history_limit` remains a no-op, and unloading
  never drops below that configured initial limit even when it is not 10.
- Alias-history reloads remain off the UI thread and preserve the existing cached
  freshness and highlight behavior.
- The bundled `ace.page_size` default remains 100, and all non-alias-history consumers
  retain their existing page-size behavior.
- User documentation describes the local 10-run exception without presenting it as a
  change to the global ACE setting.
