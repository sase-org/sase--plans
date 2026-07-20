---
tier: tale
title: Preserve agent tribe expansion across TUI updates
goal: Keep each agent tribe panel's explicit expanded or collapsed preference across
  update-driven TUI restarts and transiently incomplete agent projections, without
  regressing deliberate clears or nested fold reconciliation.
create_time: 2026-07-20 10:17:24
status: wip
prompt: 202607/prompts/preserve_tribe_expansion_after_update.md
---

# Preserve Agent Tribe Expansion Across TUI Updates

## Diagnosis

The persisted fold-state snapshot and the update restart path are both functioning: update completion enters the same
controlled-exit path as an ordinary restart, waits for the fold-state flush, and re-execs ACE against the same state
file. The loss happens after restart when the Agents tab reconciles saved whole-panel intents against the panels present
in the current agent projection.

`reconcile_panel_fold_registries` currently intersects `_collapsed_panel_keys` and `_expanded_panel_keys` with the panel
keys visible in each non-merged projection, then schedules the pruned state for persistence. Startup is allowed to
expose a partial agent projection while artifact/history data is still converging. If a tribe is absent from that
projection, its durable preference is therefore mistaken for stale data and erased; when the tribe reappears, it falls
back to the default expanded state. An update-driven restart makes this transient state especially visible, but the
faulty lifetime assumption is in fold reconciliation rather than in the updater or restart plumbing.

## Implementation

1. In the Agents-tab fold-scope reconciliation, separate durable whole-panel preference from projection-local cleanup.
   - Stop pruning explicit collapsed and expanded panel keys merely because a tribe is absent from the current
     projection.
   - Continue reconciling nested group-fold state against the active layout, since that state describes concrete agent
     groupings rather than the stable tribe-level preference.
   - Preserve the existing explicit clearing paths, including merged-layout transitions and user toggles, so obsolete
     intent can still be removed deliberately.
   - Do not add synchronous I/O or any extra data-scaled work to the UI refresh/startup path.

2. Update the fold-persistence regression coverage to encode the intended lifetime.
   - Replace the assertion that a missing panel is automatically pruned with assertions that both explicit collapsed and
     explicit expanded tribe intents survive a partial projection.
   - Exercise a disappear/reappear sequence and verify the returning tribe receives its previously persisted state
     rather than the default expansion.
   - Retain coverage proving stale nested group-fold entries are reconciled and saved when appropriate.
   - Retain or strengthen coverage that an intentional merged-layout clear removes both collapsed and expanded panel
     intent.

3. Cover the update-triggered lifecycle boundary without performing a real self-update or process exec.
   - Use the existing update/restart test seams to confirm a successful changed update still requests the controlled
     restart path.
   - Combine that invariant with the fold regression so the original failure sequence—saved preference, restart,
     incomplete projection, full projection—is represented deterministically.

## Validation

1. Run `just install` so the ephemeral workspace has current development dependencies.
2. Run the focused fold-persistence and group-fold tests, plus the self-update and controlled-restart tests affected by
   the scenario.
3. Run any visual test directly implicated by an Agents-tab rendering behavior change; no snapshot change is expected
   because this corrects restored state rather than presentation.
4. Run `just check` for the repository-wide required validation.
5. Re-run the focused disappear/reappear regression after the full check to confirm no ordering or shared-state
   dependency masks the fix.

## Acceptance Criteria

- A collapsed tribe remains collapsed after a successful TUI update and restart, even if the tribe is absent from an
  intermediate startup projection.
- An explicitly expanded tribe likewise retains its preference instead of being overwritten by configuration defaults.
- Ordinary restarts and update-driven restarts use the same persisted state without a schema migration or alternate file
  path.
- Nested group folds still reconcile against the current layout, and explicit panel-intent clearing still works.
- The focused regression suite and `just check` pass.
