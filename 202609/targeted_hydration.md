---
tier: tale
title: Targeted hydration for never-fetched artifact rows
goal:
  Valid artifact links fetch and reveal one missing row without broad inventory growth
  or stale async side effects.
size: medium
proposed_by: bbugyi200.apollo.sase-w3.7
bead: sase-w3.7
---

- **PARENT:** [202609/link_follow_reliability.md](link_follow_reliability.md)
- **BEAD:**
  [sase-w3.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w3/sase-w3.7.md)
- **AGENTS:**
  - [bbugyi200.apollo.sase-w3.7](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.apollo.sase-w3.7.md)
- **COMMITS:**
  - [ae196a3](https://github.com/sase-org/sase/commit/ae196a367bf2f0a48533ce3af48e735c73ee44ff)
    — feat(ace): add targeted hydration for artifact link-follow panes

# Targeted hydration for never-fetched artifact rows

## Goal

Complete phase `sase-w3.7` by extending the existing tri-state link-follow coordinator
and host-owned reveal ladder with one final, targeted acquisition step. A valid link
whose row was never included in a capped pane snapshot must remain `PENDING` while the
single row is fetched off the Textual pump, install that row into the destination pane,
and then re-enter the normal ladder. A malformed or known-dangling reference must still
fail immediately, and an acquisition error must end as `FAILED`, never as an inventory
miss.

The implementation must preserve the existing finalize-on-select behavior: no link trail
hop, rail refresh, or success notice is emitted until the hydrated row is actually
selected. It must also preserve the reveal ladder's query-history and `LinkReveal`
invariants.

## Implementation

1. Add an opt-in targeted-hydration contract beside `ArtifactEntryNavigator` that
   cleanly separates blocking lookup from UI-thread installation. Use a typed outcome
   that distinguishes unsupported refs, authoritative absence, a fetched row, and a load
   failure. Blocking resolvers may only return immutable pane payloads; they must not
   mutate Textual widgets or pane snapshots. Default behavior remains unsupported so
   panes without a direct source keep their current inventory-miss result.

2. Extend `_LinkFollowTransaction` and the missing-target path in
   `src/sase/ace/tui/actions/link_follow.py` so hydration is attempted exactly once,
   only after fold/limit/identity/widen/neutral rungs are exhausted. Parse the canonical
   ref already stored on the transaction, reject malformed or unroutable refs without
   scheduling work, and use `spawn_pump_free_task` plus `asyncio.to_thread` for the
   blocking lookup. Keep a small in-flight map keyed by destination pane and canonical
   ref so repeated requests coalesce; a newer generation supersedes the old waiter.
   After every await, re-read the live transaction and pane and apply a result only when
   generation, ref, and destination still match. Cancellation, user navigation, a second
   follow, and teardown must detach stale waiters and prevent stale installs,
   notifications, or trail hops.

3. On a fetched result, install it on the UI thread, ask the pane's canonical
   `entry_target_for_ref` resolver for the newly available stable identity, replace the
   transaction target, and restart selection plus the reveal ladder from its first rung
   while marking hydration attempted to prevent a loop. While the worker is live, leave
   the transaction visibly pending through the pane/app's existing loading-status seam
   rather than emitting a warning toast. Map an authoritative direct miss to the
   existing `No such artifact` warning and map exceptions or provider failures to the
   coordinator's distinct `FAILED` error.

4. Implement direct lookup and merge adapters for every phase-supported destination,
   reusing source-specific loaders and row builders rather than growing full
   inventories:
   - Stitches: resolve the repository and abbreviated SHA through the configured VCS
     provider, request exactly that revision, and merge the resulting commit row into
     the current `VcsLogResult` without changing the collection window.
   - Files: query the artifact-file index for the exact logical id (including the
     established first-part compatibility spelling) and merge the returned artifact row
     into the files snapshot/index.
   - Plans and plugin document panes: resolve the exact provider identity/path using the
     pane's configured document roots/provider, parse one document with the existing
     plan/provider row builders, and merge it into proposal/active/archive data without
     running the deep-archive scan.
   - Agents: use the persistent agent index/direct name lookup for the canonical and
     alias candidates, materialize one `Agent`, and merge it into the unfiltered agent
     snapshot.
   - Beads: resolve the owning bead store and exact id through the read facade, build
     the correct task/flag/epic/phase row, and merge it with the current snapshot while
     preserving phase-parent grouping.

   Each installer must rebuild only the pane's derived target/query/relation indexes
   needed for the new row, preserve the user's query, selection, scope, and folds, and
   let `request_entry_target` plus the existing ladder decide visibility.

5. Add focused tests around the duck-typed harness in
   `tests/ace/tui/test_link_follow.py` for rung exhaustion before hydration, a slow
   hydration remaining pending without a miss toast, successful install followed by
   ladder re-entry and finalize-on-select, duplicate coalescing, second-follow/user-nav
   supersession, exception-to-`FAILED` mapping, and malformed/dangling fast failure. Add
   pane-level tests for exact stitch, file, document/provider, agent-alias, and bead
   lookups, proving capped snapshots gain only the requested row and that query history
   remains unchanged except for any normal reveal rewrite after installation.

## Verification

Run `just install` before tests in the ephemeral workspace. Run the targeted link-follow
and affected pane suites first, then run `just check` as the required whole-repository
agent gate. Before closing the phase, run `sase bead epic-symbols sase-w3.7`, resolve or
re-key every remaining symbol, and close only `sase-w3.7` with a note naming the tests
and checks that passed.
