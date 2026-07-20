---
tier: epic
title: Integrate and land priority-aware runner-slot queues
goal: 'Existing agent-list and ACE queue-position projections match priority-aware
  runner-slot admission, then epic sase-8c is closed and its post-close Symvision
  and plan-state cleanup is completed.

  '
phases:
- id: queue-projections
  title: Align queue projections with priority-aware admission
  depends_on: []
  size: small
  description: '''Queue projection integration'' section: carry wait priority through
    presentation-neutral agent models and make existing queue-position calculations
    follow backend priority ordering.'
- id: land-epic
  title: Close and clean up epic sase-8c
  depends_on:
  - queue-projections
  size: small
  description: '''Epic landing'' section: close sase-8c, remove post-close Symvision
    findings, verify the repository, and mark the original epic plan done.'
create_time: 2026-07-20 15:01:35
status: done
bead_id: sase-8e
---

# Plan: Integrate and land priority-aware runner-slot queues

## Context

Epic `sase-8c` added `%wait(priority=<non-negative integer>)`, with lower values admitted before higher values and the
existing FIFO order retained for equal priorities. Its three implementation commits are present at the current
default-branch heads:

- `sase-core` commit `82c7efa` carries `wait_priority` through the Rust scan wire and adds editor/LSP completions.
- `sase` commit `46c2f0622` parses, validates, persists, and applies priority to backend runner-slot admission,
  including default priority 10 and marker overrides.
- `bugyi-chops` commit `21babe3` launches every `toobig_split` proposal with `%wait(priority=20)`.

The audit also reviewed commits landed after the epic's 2026-07-20 14:00 start. A concurrently added
presentation-neutral agent-list projection and the ACE runner-capacity projection still calculate queue position using
FIFO only. Consequently, their reported positions can disagree with the backend's actual priority-aware admission order.
These projections also discard `WaitingMarkerWire.wait_priority`. Correcting existing queue-position output is
integration work; it does not add priority rendering or a priority editor, both of which remain explicit non-goals of
the original epic.

## Queue projection integration

Update both current queue-position projections to use the backend semantics: priority first, then invalid timestamp
status, parsed `slot_requested_at`, the existing stable timestamp/suffix tie breaker, and artifact path. Missing,
negative, boolean, or otherwise invalid priority values must behave as the backend default of 10. Preserve the current
rule that an ineligible high-priority waiter does not block an eligible lower-priority waiter.

For the presentation-neutral integration API, carry the marker's priority into `AgentWaitInfo`, populate it in the
agent-list entry builder, expose it from `sase agent list --json`, and use it in `_attach_runner_slot_context`. Keep the
field additive and backward-compatible for old markers.

For ACE, carry the marker's priority through `AgentState` from both wire-backed and filesystem-backed enrichment,
preserve it during row deduplication, and use it in `agent_runner_slots` when computing existing queue positions. Do not
add new priority text, wait-modal inputs, or keybindings.

Add focused regression coverage for:

- a newer priority-1 waiter receiving position 1 ahead of an older priority-20 waiter in both projections;
- equal and missing priorities retaining FIFO/default-10 behavior;
- marker-to-model and JSON projection of `wait_priority`;
- preservation across the ACE enrichment/dedup paths touched by the change.

Run focused tests while iterating. Because this phase changes files in the main SASE repository, run `just install`
before the final `just check`, as required by repository policy.

## Epic landing

Only after the queue projection integration and `just check` pass:

1. Close the original epic with `sase bead close sase-8c`.
2. Run `just symvision` after the close, because epic-symbol whitelist entries for `sase-8c` expire at that point.
   Remove stale whitelist entries and any unused code Symvision reports, then rerun `just symvision`. If this cleanup
   changes main-repository files, rerun `just check` as well.
3. Open the plans sidecar with `sase repo open plans` and change only the frontmatter `status` of
   `${SASE_SDD_PLANS_DIR}/202607/wait_priority_directive.md` from `wip` to `done`.
4. Verify `sase bead show sase-8c` reports the epic closed and the original plan file reports `status: done`.

Do not close the epic or mark its original plan done before the integration tests and repository checks succeed.

## Risks and boundaries

- Reuse `DEFAULT_WAIT_PRIORITY` rather than introducing a second numeric default, so projections cannot drift from
  admission.
- Treat Python booleans as invalid priorities even though `bool` is an `int` subclass, matching the backend's existing
  strict type guard.
- Do not broaden the original epic into priority display/edit UI. This landing only makes already-present queue-position
  output truthful and exposes the marker field through the existing presentation-neutral JSON model.
