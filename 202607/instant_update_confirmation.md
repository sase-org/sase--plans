---
create_time: 2026-07-11 12:32:25
status: done
prompt: 202607/prompts/instant_update_confirmation.md
tier: tale
---
# Make `,U` Open Update Confirmation Immediately After Refresh

## Context

The global `,U` shortcut opens SASE Admin Center on the Updates tab with `auto_update_on_load=True`. The tab correctly
does its catalog, installed-version, latest-version, and editable-checkout remote work in a background worker. Once that
worker finishes, however, the pane calls the normal comprehensive update action. For editable installs, that action
builds a fresh `DevUpdatePreview`, and `plan_dev_update()` fetches every unique editable checkout upstream again before
opening the y/n confirmation modal.

That second network pass was made necessary for standalone/manual update planning so stale remote refs cannot hide
updates, but it is redundant for the `,U` path when the immediately preceding Updates-tab load has already fetched the
same checkout successfully. This explains the visible pause after the tab has finished loading. The fix should remove
that duplicate work without weakening freshness guarantees for manual updates or exceptional roots.

## Goals

- Open the comprehensive update confirmation on the next UI turn after the `,U` Updates-tab load completes in the normal
  editable-install case.
- Preserve the existing rule that update actionability is based on freshly fetched upstream refs.
- Preserve current behavior for the manual `u` action, refresh failures, offline/unavailable sources,
  detached/no-upstream checkouts, and installed editable packages that are not represented in the catalog.
- Keep all blocking inventory, git, receipt, and planning work off the Textual event loop.
- Avoid a time-based global cache whose validity or invalidation would be ambiguous.

## Proposed Design

### 1. Return explicit remote-freshness evidence from the Updates-tab loader

Extend the immutable Updates-tab load result with the canonical git roots whose editable latest-version checks completed
a successful online fetch during that specific load.

Derive this set from the enriched core/package results only for states that prove the fetch completed and the post-fetch
status was classified (for example current, update available, dirty, diverged/ahead). Do not mark roots fresh when the
check was offline, unavailable, detached, missing an upstream, or reported a fetch failure. Deduplicate shared roots.

This keeps freshness scoped to one load result and lets the UI distinguish “we just fetched this root” from “we merely
have a cached upstream ref.”

### 2. Thread the one-shot freshness set through only the auto-update path

When the initial `,U` load succeeds and updates are available, schedule the comprehensive update preview with the load
result's fresh-root snapshot. Continue rendering the completed Updates tab before scheduling the preview so the event
loop can paint normally.

The ordinary `u` action must continue to invoke preview planning without a freshness snapshot. Likewise, consume the
snapshot only for the auto-update triggered by that exact load; do not retain it as a pane-wide cache for later user
actions or later reloads.

Keep the existing all-current toast and load-error behavior unchanged.

### 3. Let dev-update planning reuse only proven-fresh roots

Add an optional explicit collection of already-refreshed canonical roots to the editable update preview/planner call
chain. The planner should still classify local state for every root, but skip its network refresh step for roots in this
collection because the immediately preceding loader already refreshed their remote-tracking refs. It should continue to
fetch every other eligible root using the existing error handling and post-fetch reclassification.

Default the new planner input to empty so CLI updates, manual TUI updates, plugin-specific updates, and all existing
callers preserve their current fetch-before-plan semantics. Plan contents, blocking rules, reconcile steps, and
execution behavior remain unchanged; this is orchestration/freshness reuse, not a new update policy.

This selective contract is intentionally safer than a blanket `refresh=False`: an uncatalogued editable plugin or a root
whose tab fetch failed will still get a planning-time fetch and cannot be silently evaluated from stale refs.

### 4. Preserve responsive modal behavior

Continue running runtime inventory and preview construction in the existing threaded worker. Once the local-only preview
for already-refreshed roots completes, open the existing `PluginActionConfirmModal` through the current UI-thread
completion path. The modal's incoming-commit section can continue loading asynchronously from its seeded cache and must
not delay the y/n controls.

## Test Plan

- Add planner unit coverage showing that:
  - default calls still fetch each unique eligible root exactly once;
  - explicitly fresh roots skip the network fetch while producing the same actionable/skipped classification from the
    refreshed tracking refs;
  - roots absent from the freshness set still fetch;
  - fetch-failed/offline/unavailable roots are not incorrectly treated as fresh.
- Add Updates-tab loader coverage for extracting and deduplicating only successfully refreshed editable roots from core
  and plugin results.
- Extend the `,U`/auto-update TUI tests to prove the load result's freshness snapshot reaches preview planning, no
  second fetch is attempted for those roots, and the confirmation modal opens immediately after load completion.
- Retain or add regression coverage proving manual `u` does not reuse the auto-load snapshot, all-current still shows
  its informational toast without a modal, and load errors clear the one-shot auto-update state.
- Prefer deterministic call-order/fetch-count assertions over fragile wall-clock thresholds. Manually profile the real
  `,U` path before and after to confirm the post-load network span is gone and the modal appears on the next event-loop
  turn.

## Validation

- Run the focused dev-update planner and Updates-pane TUI tests while iterating.
- Run `just install` followed by the required full `just check` before handoff.
- Confirm the worktree contains only the intended source/test changes and that no visual snapshot update is needed
  because the UI layout is unchanged.

## Non-Goals

- Do not change the update command, confirmation contents, incoming-commit rendering, or task execution/restart flow.
- Do not introduce a persistent/TTL git-fetch cache.
- Do not move the delay earlier by performing the same duplicate fetch inside the initial loader; the redundant network
  operation should actually be removed.
- Do not broaden this focused responsiveness fix into a migration of the existing dev-update backend.
