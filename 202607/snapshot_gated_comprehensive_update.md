---
tier: tale
title: Snapshot-gated comprehensive update flow
goal: 'ACE''s global ,U action captures the provider candidates from the latest completed
  automatic update result, safely replans only those providers against live state,
  and runs provider plus SASE work as one conflict-safe tracked flow with truthful
  completion and restart behavior.

  '
create_time: 2026-07-20 11:01:27
status: wip
prompt: 202607/prompts/snapshot_gated_comprehensive_update.md
---

# Plan: Snapshot-gated comprehensive update flow

## Context and boundaries

The preceding phase made automatic update results composite: an `UpdateStatus` now carries immutable provider
candidates, separate source freshness, and SASE/provider counts, and the existing worker applies completed results on
the UI thread. The Updates pane already loads live core/plugin/provider inventories and exposes independent `u` (SASE,
core, and plugins) and `A` (agent CLIs) actions backed by the existing SASE and provider planners/executors.

This phase connects those systems only for the global `,U` action. It will not change provider-specific update logic,
the direct `u` or `A` semantics, badge-click behavior, cadence, indicator styling, startup-toast language, public CLI
commands, or Rust-core behavior; the parent epic's final phase owns the distinct badge/help/docs polish. All inventory,
planning, subprocess, receipt, and snapshot work remains in worker threads or tracked tasks, while key dispatch and
render paths remain in-memory and synchronous.

## Keypress-owned one-shot request

- Add immutable in-memory automatic-update state to ACE and replace it only when a completed automatic result is applied
  on the UI thread. Store the stable provider names from that exact result, including an empty tuple after a successful
  completed result; failed or still-running checks do not become new authority.
- At `,U` dispatch, synchronously copy the current provider-name tuple and pass that value through the Config Center
  constructor into a one-shot comprehensive request owned by the Updates pane. Preserve the distinction between no prior
  completed automatic result and a completed result with no provider candidates so both correctly avoid provider
  mutation while keeping existing SASE behavior.
- Consume the request once after the initial pane load succeeds or fails. The pane's foreground load may update its
  displayed inventory and the shared durable snapshot, but it must not mutate or broaden the captured names. Closing or
  cancelling consumes only the pane request and leaves ACE's latest automatic projection available for the next `,U`.
- Keep opening the Updates tab directly or via the badge mutation-free, and leave pane-wide `u` and `A` as deliberate
  current-inventory actions independent of the automatic candidate projection.

## Typed comprehensive planning and confirmation

- Introduce focused typed models for one comprehensive preview and terminal result. The preview composes the existing
  managed/editable SASE preview with an agent-CLI plan built from the captured provider names and the live statuses
  already loaded by the pane; durable candidates supply identity only and never executable commands.
- Revalidate by intersection: plan only captured names that still exist in the live inventory, represent removed or
  current candidates as dropped/already-current outcomes as appropriate, and never include a newly discovered outdated
  provider that was absent at keypress time. Preserve exact planner-produced commands, manual suggestions, skip reasons,
  and vendor documentation.
- Build the comprehensive preview off the UI thread, reusing the pane load's fresh editable roots. Isolate the SASE leg
  so an unavailable or blocked SASE plan does not discard a runnable provider leg; likewise, provider planning trouble
  must not suppress valid SASE work.
- Present one confirmation with separate “SASE, core & plugins” and “Agent CLIs” sections. It must support SASE-only,
  provider-only, mixed, and SASE-blocked/provider-runnable confirmations; show every runnable command and every
  manual/skipped provider reason; route manual-only provider cases to the Agent CLIs sub-tab with guidance; and emit one
  accurate all-current result when live revalidation leaves no work. Cancellation performs no mutation.

## Conflict-safe execution and aggregate outcomes

- Extend tracked-task deduplication with the smallest reusable notion of mutually exclusive update scopes, covered at
  the task-queue/action layer. A comprehensive update must claim both `sase-update` and `agent-cli-update`, and each
  existing direct action must conflict with that claim, without changing unrelated task deduplication behavior.
- After confirmation, submit exactly one tracked comprehensive task. Execute safe provider commands sequentially with
  the existing agent-CLI executor, then execute the selected managed/editable SASE leg with the existing runners. Record
  provider updated/current/manual/skipped/failed outcomes and the SASE outcome independently; a failure in one leg must
  not silently prevent the other independent leg from being attempted.
- Produce one task log and concise aggregate completion summary that remains truthful for full success, partial success,
  all-current, manual-only, and fully failed results. Keep provider result details available to the pane and refresh the
  composite update snapshot/indicator immediately off-thread after completion rather than waiting for the periodic tick.
- Close the Admin Center after a confirmed runnable comprehensive task, matching the existing SASE update flow. For a
  provider-only result, stay in the current ACE process and refresh state in place. No-op and fully failed flows do not
  restart.

## Restart receipt and completion handoff

- Define “code changed” solely from the SASE/core/plugin leg. Run all provider commands before considering restart, and
  use the existing restart gate so no tracked work is abandoned.
- Extend the one-shot update receipt schema/model to carry bounded provider result summaries alongside the existing SASE
  transitions. Preserve updated, already-current, manual/skipped, and failed provider details needed for an honest
  post-restart message; decode old receipts compatibly and reject malformed/future formats safely.
- When SASE code changed, persist one combined receipt and restart only after the comprehensive task has completed. When
  SASE did not change, render the same aggregate outcome in place. A partial provider failure must remain visible even
  when a successful SASE leg causes a restart.

## Verification

- Add automatic-check/action tests for atomic projection replacement, no-prior-result behavior, completed-empty
  behavior, in-flight and later-result race boundaries, immutable keypress capture, Config Center handoff, cancellation,
  and load-error consumption. Assert key dispatch performs no disk I/O, subprocess calls, or provider discovery.
- Add comprehensive planner/confirmation tests for SASE-only, provider-only, mixed, blocked-SASE/provider-runnable,
  stale removed/current candidates, non-broadening despite newly discovered live updates, manual-only guidance, exact
  commands/docs, all-current messaging, and preservation of direct `u`, direct `A`, and badge-click behavior.
- Add task-queue and execution tests for mutual exclusion with both existing update scopes, sequential provider
  execution before the SASE leg, continuation across independent leg failures, accurate aggregate results, immediate
  snapshot/indicator revalidation, and one tracked task per accepted comprehensive action.
- Extend receipt and post-update-toast tests for combined SASE/provider round trips, legacy compatibility, partial
  success/failure rendering, provider-only no-restart, SASE-change restart-after-provider ordering, and no-op/fully
  failed no-restart behavior.
- Run `just install`, then focused task-queue, update-receipt, post-update-toast, automatic-check, keymap, Updates-pane
  loading, SASE-update, agent-CLI, and new comprehensive-flow tests. Finish with the repository-required `just check`.

## Completion criteria

- `,U` can update a provider CLI if and only if that provider name was present in the last completed automatic result at
  keypress time and remains safely actionable in the pane's live inventory; no later discovery can broaden the request.
- One confirmation and one tracked task truthfully cover all eligible SASE and provider work, show manual guidance, and
  prevent overlap with either direct update scope while allowing independent legs to finish after failures.
- Provider-only changes refresh without restart; a real SASE/core/plugin change restarts only after provider execution
  and preserves the full combined outcome across the restart; no-op and fully failed flows do not restart.
- Direct `u`, direct `A`, badge-click, startup/periodic responsiveness, and existing update services remain compatible,
  and all focused tests plus `just check` pass.
