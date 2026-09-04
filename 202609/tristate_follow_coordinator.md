---
tier: tale
title: Complete tri-state artifact-link follows
goal:
  Artifact-link requests remain pending until authoritative selection, absence, or
  failure, and only successful selections finalize navigation history.
size: medium
proposed_by: bbugyi200.apollo.sase-w3.3
bead: sase-w3.3
---

- **PARENT:** [202609/link_follow_reliability.md](link_follow_reliability.md)
- **BEAD:**
  [sase-w3.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w3/sase-w3.3.md)

# Complete tri-state artifact-link follows

## Objective

Implement Phase 3 (`sase-w3.3`) of the approved artifact-link reliability epic so a link
follow has an explicit `SELECTED`, `PENDING`, `MISSING`, or `FAILED` outcome. Pending
follows must remain generation-tagged until their destination pane selects the target,
link-trail history and rail refreshes must happen only after selection, and only the
app-level coordinator may report an authoritative absence or acquisition failure.

Preserve the Phase 2 canonical-ref resolution behavior and the existing link-follow
ladder apart from the Phase 3 changes below. Do not implement the generic reveal ladder,
identity queries, or targeted hydration assigned to later phases.

## Implementation

1. Add a shared link-request state/transaction model and change the
   `ArtifactEntryNavigator.request_entry_target` contract from Boolean completion to the
   four-state result. Give deferred pane requests an optional generation token, retain
   that token beside `_pending_entry_target`, and provide one shared completion seam
   that clears pending state and reports `SELECTED`, `MISSING`, or `FAILED` back to the
   host. Update non-link callers and test doubles to compare the explicit state rather
   than relying on enum truthiness.

2. Refactor `LinkFollowMixin` into the host-owned coordinator. Capture the complete
   `LinkTrailHop` origin before any tab, pane, scope, query, selection, or fold changes;
   increment a generation for each artifact follow; supersede the prior transaction; and
   register the transaction before invoking pane code so synchronous refreshes can
   safely report completion. A `SELECTED` completion records the origin and refreshes
   the rail exactly once. A `PENDING` completion keeps the transaction open. A stale
   generation is ignored. An authoritative `MISSING` result proceeds through the
   existing Phase 3 host fallback (including the head-limit rewrite) before emitting one
   warning; `FAILED` emits one distinct error and is never described as deletion or
   absence. Ensure a second follow and subsequent user navigation/input cancel the old
   generation without producing a stale trail hop or toast.

3. Wire every asynchronous pane at its existing pending-target resolution and worker
   error points. Commits/Stitches, Beads, Agents, Plans, and Files must report selection
   only after the matching row is actually selected; incomplete snapshots and active
   work stay `PENDING`; complete authoritative coverage without the row reports
   `MISSING`; and load/query/collection errors report `FAILED`. Remove the pane-local
   “no longer visible”/“not found” notifications so absence and failure copy is owned
   centrally. Preserve current fold expansion and temporary filter-clearing behavior for
   now because the generic reveal ladder replaces those paths in Phase 4.

4. Extend `_pane_is_loading` so Stitches collection work and asynchronous query
   evaluation count as loading, including queued/in-flight query-session work, while
   keeping the test duck types supported. A follow into any loading pane must return
   `PENDING` rather than silently dropping the transaction.

5. Add the Patches host-query adapter to `ArtifactsPatchesPane`: expose the live or
   committed query through `host_limit_query`, and apply rewritten host queries through
   the existing `_commit_patch_query` seam so query history and selection restoration
   remain canonical. Normalize every `apply_host_limit_query(query, *, grow=False)`
   implementation (especially Beads and Stitches), remove the link-follow `TypeError`
   compatibility shim, and make `LinkTrailHop` origin capture include the Patches query
   so `Ctrl+O` can restore it.

6. Add focused regressions for the state machine and protocol boundaries: partial or
   loading snapshots remain pending without warnings; a matching generation selects and
   finalizes one trail hop; a second follow supersedes the first; stale success,
   missing, and failure callbacks do nothing; acquisition failure uses distinct error
   copy; Stitches collection/query workers are recognized as loading; the normalized
   grow keyword works for every pane; and a Patches-origin follow followed by `Ctrl+O`
   restores the exact prior Patch query and selection. Update existing navigator/loading
   tests for explicit request states and retain the Phase 2 canonical-ref regressions.

## Verification

Run `just install` first because this is a fresh ephemeral checkout. Run the focused
link-follow, link-trail, navigator, pane-loading, query-history, and artifact-limit test
modules affected by the contract change. Then run the repository-required `just check`
lane. Inspect `git diff --check`, re-run `sase bead epic-symbols sase-w3.3`, resolve or
re-key any ownership markers, and close only `sase-w3.3` with a note naming the focused
tests and `just check` result.
