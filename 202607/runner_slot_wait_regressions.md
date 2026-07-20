---
tier: tale
title: Fix runner-slot wait regressions
goal: 'Parked runner-slot agents respond correctly to Run Now, preserve authored admission
  priority after question yields, and retain unrelated waiting-marker conditions while
  slot polling refreshes its owned fields.

  '
create_time: 2026-07-20 16:37:32
status: done
prompt: 202607/prompts/runner_slot_wait_regressions.md
---

# Plan: Fix runner-slot wait regressions

## Context and invariants

Runner-slot admission is a process-local polling loop coordinated by the global runner-slot lock. The ACE wait editor
already persists edits in a tracked worker under the same lock, but its Run Now branch currently writes a dependency
`ready.json` marker that the slot poll never consumes. The question continuation path also drops the persisted
`%wait(priority=...)` value when it reacquires a slot, and the slot poll replaces `waiting.json` with a fixed payload
that removes conditions or extension fields owned by other wait mechanisms.

Keep the existing admission model intact: the shared priority normalization and priority-then-FIFO queue remain
authoritative; Run Now restores the implicit global-cap threshold rather than bypassing capacity; TUI disk work remains
in the tracked background task; and all waiting-marker edits serialize through the existing runner-slot lock. Preserve
backward compatibility for markers and metadata that omit priority.

## Implementation

1. Route ACE Run Now for an agent carrying `slot_requested_at` through the live runner-wait persistence path. Remove the
   prompt's explicit runner threshold, clear the matching metadata field, and rewrite the live marker to an implicit
   global-cap wait so the runner's next poll can admit it. Do not create the dependency-only `ready.json` marker for
   this case. Keep optimistic Agent-model state and rollback behavior aligned with the persisted outcome.
2. Change the runner-slot marker refresh to start from the currently locked `waiting.json` object and update only
   slot-owned fields (`cl_name`, timestamp, effective threshold/explicitness, normalized priority, and request time).
   Preserve dependency, bead, timing, and unknown extension keys across polls, while continuing to retain the original
   FIFO request time and remove the whole marker only after a successful claim or kill.
3. Thread the interrupted phase's persisted `wait_priority` from `base_meta` into question-answer slot reacquisition.
   Extend `AgentMetaWire` in both the Python adapter and the Rust core scan contract, including Rust marker decoding, so
   scanners and frontends retain that metadata even before or without a live waiting marker. Keep Python filesystem and
   wire enrichment behavior consistent where metadata fallback is exposed.
4. Document that slot admission sorts lower numeric priority first and FIFO within a priority, with no aging; sustained
   higher-priority arrivals can therefore starve default/lower-priority waiters. Clarify that question continuations
   keep their authored priority while reacquiring under the current global cap.

## Verification

- Replace the misleading ACE Run Now assertion with an integration-style regression that exercises the live marker edit
  and a subsequent runner-slot claim, proving the parked agent proceeds and no stray `ready.json` is created.
- Extend runner-slot unit coverage to prove repeated polls preserve foreign waiting-marker keys and still update owned
  threshold/priority fields.
- Add question-marker coverage that captures the reacquisition call and asserts explicit, default, and invalid/missing
  priorities follow the existing normalization contract without changing the global-cap threshold behavior.
- Add Python and Rust scan-wire parity tests for `agent_meta.json` priority, including legacy omission/default behavior
  and frontend enrichment fallback.
- Run focused Python tests for ACE wait resume, runner-slot polling, question continuation, metadata enrichment, and
  scan-wire conversion; run the relevant Rust `sase_core` tests; then run `just install` followed by `just check` in the
  SASE checkout, fixing every failure before closing only `sase-8g.3`.

## Risks

The primary race is a TUI edit landing between runner polls; reusing the shared lock and merge-updating the freshly read
marker prevents either writer from silently winning with stale state. Preserving foreign keys must not change slot
eligibility itself: the poll still evaluates only its slot-owned threshold and priority, leaving dependency resolution
to the existing wait machinery. Cross-repo wire changes must remain field-for-field compatible so deployments with older
metadata continue to deserialize safely.
