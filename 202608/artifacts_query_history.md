---
tier: tale
title: Query history for every Artifacts pane
goal: "Make the configured previous/next query keys navigate an independent, durable
  query history on every capable Artifacts sub-tab, and show that pane's history in
  Help.

  "
size: medium
proposed_by: bbugyi200.athena.0cq
---

# Plan: Query history for every Artifacts pane

## Outcome and current gap

The Artifacts host already derives `PaneCapability.QUERY_HISTORY` for healthy built-in
and provider-backed panes with an inventory and query fields, and durable history is
already namespaced by pane id. The runtime does not fulfill that contract: `^` / `_` are
implemented in `PatchQueryMixin`, explicitly return outside `patches`, are excluded from
the non-Patch action allowlist, and only the Patch history bucket is loaded into app
state. Stitches, Beads, provider documents (including Plan), and Files also commit
queries through separate filter-session adapters without recording the displaced query.
Finally, `HelpModal` hard-codes the Query History box to the Patch pane and reads it
from disk during rendering.

Implement one host-owned query-history path that keeps Patch's specialized filtering
behavior but makes every active Artifacts pane that earns `QUERY_HISTORY` participate in
the same previous/next stack semantics. Histories, selections, profile validation, and
help content remain isolated by the exact pane id, including arbitrary `ref:<kind>`
provider panes.

## Design constraints

- Treat the active `ArtifactsPaneContract` as the availability boundary. Healthy panes
  with `QUERY_HISTORY` get the actions and help box; suppressed, degraded, unknown, and
  non-Artifact surfaces do not. Do not infer support from a fixed pane-id list or a
  `ref:` prefix.
- Persist `QueryRecord(source, canonical, profile_digest)` values produced from the
  active pane's live compiled query profile. A record from a changed dialect must fail
  visibly without consuming or corrupting either stack. Provider panes must use their
  contract profile rather than the built-in Plan profile.
- Record only committed query transitions. Live typing and invalid submissions must not
  mutate history; a successful new branch pushes the displaced query and clears the
  forward stack. Route all existing committed query rewrites through that rule,
  including inline submission, host `limit:` paging, project/quick-filter controls, and
  explicit clear/cycle actions, while deduplicating canonical no-ops.
- Preserve the selected `ArtifactEntryTarget` per canonical query and restore it when
  still present after history navigation. Fall back through each pane's established
  selection behavior when the remembered row is filtered out. Keep an open filter bar
  synchronized if a programmatic history or paging action changes its query.
- Apply the in-memory transition and repaint immediately. Query-history persistence must
  run off the Textual event loop through the existing pump-free persistence machinery,
  with latest-value coalescing so rapid `^` / `_` presses cannot let an older write
  overwrite newer stacks. Help rendering consumes the app's in-memory snapshot and
  performs no disk I/O.
- Preserve legacy flat-file migration under `patches`, the 50-record stack bound,
  configured/rebound query keys, Patch saved-query behavior, and the existing per-pane
  query dialects. Saved-query support for non-Patch panes is outside this change.

## Implementation

### 1. Add the shared contract-gated history coordinator

- Add a focused Artifacts query-history action/helper module and compose it into
  `ArtifactsMixin`. Move top-level `action_prev_query` / `action_next_query` ownership
  there, leaving Patch parsing, reload, last-query persistence, and selection handling
  behind a Patch adapter method instead of duplicating it.
- Resolve the active contract and pane adapter once per action. For non-Patch panes,
  expose a small common host protocol from Stitches, Beads, Documents, and Files that
  can return the committed source/canonical/profile record, validate and apply a record
  through the pane's existing commit/query-session path, synchronize its `FilterBar`,
  and save/restore stable entry targets. Reuse the shared Documents implementation for
  every provider kind.
- Navigate against a copy of the pane-local stacks and publish the mutation only after
  the target record validates and applies successfully. Report the existing boundary
  warnings when a stack is empty and a precise stale/invalid-query error when a stored
  record cannot be used, without moving either stack.
- Register `prev_query` and `next_query` as the host actions implementing
  `PaneCapability.QUERY_HISTORY`, admit them through the non-Patch action allowlist, and
  gate them in `check_app_action` by the active contract. Extend the contract
  conformance checks so every capable built-in and synthetic provider resolves the
  configured keys to reachable actions and every incapable/degraded pane rejects them.

### 2. Populate pane-local history from every committed query change

- Add one transition helper that compares the old and new canonical forms, captures the
  old source text and live profile digest, saves the old selection target, pushes the
  record onto that pane's previous stack, clears its forward stack, and schedules
  persistence. Make it safe for pane widgets to call through their app without knowing
  storage details.
- Wire the helper into the successful commit funnels for Patch, Stitches, Beads,
  Documents/providers, and Files. Account for Stitches' debounced live preview by using
  the filter-session restore value as the pre-edit committed query rather than the
  preview value. Route `limit:` changes and each pane's programmatic project,
  toggle/cycle, and clear-filter mutations through the same funnel so `^` always returns
  to the view the user just left.
- Load the complete pane-keyed history and selection maps once into app state rather
  than initializing only `patches`, while retaining empty lazy buckets for newly
  discovered provider ids. Add snapshot/copy helpers in the persistence layer as needed
  so background writes cannot observe mutable lists or lose another pane's bucket. Reuse
  the existing pump-free task lifecycle and cancel/drain behavior at app teardown.

### 3. Render the active pane's Query History box in Help

- Pass the active contract capability, pane-local in-memory stack snapshot, and current
  query metadata into `HelpModal` from `action_show_help`. Generalize
  `add_query_history_section` to render a supplied stack (with its storage-loading
  fallback retained only for standalone callers/tests), and remove Patch-only comments
  and conditions from the modal and forwarded navigation actions.
- Show the Query History box for every active Artifacts pane with `QUERY_HISTORY`, using
  the configured previous/next key labels and the exact active pane id. Keep it absent
  on Agents/Axe and incapable or degraded Artifacts panes. Make help filtering and
  column balancing account for the box exactly as they already do for Patch.
- Update the Artifacts query documentation and contract capability/action table to say
  that histories are pane-local across all capable subtabs, including provider panes,
  and that Help displays the active pane's stack.

## Verification

1. Run `just install` before repository checks.
2. Add focused persistence tests for whole-map loading/snapshotting, pane isolation,
   legacy Patch migration, maximum size, and rapid coalesced writes.
3. Add parameterized TUI coverage for Stitches, Beads, Plan/a synthetic provider, and
   Files that commits two queries, walks `^` then `_`, verifies canonical query and
   idle/open filter-bar text, restores a stable selection when possible, and proves the
   Patch and sibling-pane buckets are unchanged. Cover no-op/invalid commits, empty
   stack warnings, stale profile digests, forward-stack clearing, paging/quick-filter
   transitions, and stale async query results.
4. Extend HelpModal tests for each capable pane, configured key labels, in-memory
   history rendering, title filtering/column balance, and omission on non-Artifact or
   degraded panes. Update or add a focused Help visual snapshot only if the existing
   golden set exercises a non-Patch Artifacts pane.
5. Run the focused query-history, filter-session, Artifacts contract/conformance,
   action-availability, help-modal, and Artifacts limit suites, then run `just check`.
   If scoped selection escalates or reports an unusual selection, run `just check-full`
   only through `/sase_monitor` with a follow-up that inspects and handles the result.

## Completion criteria

- `^` and `_`, including user-rebound equivalents, walk backward and forward on every
  active Artifacts pane whose contract enables `QUERY_HISTORY`.
- Each pane has an independent durable stack containing only successful committed
  queries with the correct profile digest; failed/stale navigation leaves state intact.
- History navigation keeps filter UI, async results, and stable selection aligned with
  the restored query, and introduces no synchronous disk work on the keystroke or Help
  render paths.
- Help shows the active pane's Query History box and keys on every supported sub-tab,
  while incapable/degraded and non-Artifact surfaces do not advertise the feature.
- Focused verification and `just check` pass with no unrelated changes.
