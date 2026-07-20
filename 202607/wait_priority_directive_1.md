---
tier: tale
title: Priority-aware wait directive admission
goal: 'SASE accepts %wait(priority=<non-negative integer>), persists the resolved
  priority through agent metadata and waiting markers, and admits eligible runner-slot
  waiters by ascending priority before preserving FIFO order.

  '
create_time: 2026-07-20 14:23:53
status: done
prompt: 202607/prompts/wait_priority_directive_1.md
---

# Plan: Priority-aware `%wait` directive admission

## Context and boundaries

Bead `sase-8c.2` implements the main-SASE phase of the approved priority-aware runner-slot design. Its prerequisite Rust
wire/completion phase is already closed, so this work consumes the additive `wait_priority` scan field without changing
the linked Rust core. The later bugyi-chops consumer remains outside this bead. No new beads will be created, and
completion will close only `sase-8c.2`, leaving parent epic `sase-8c` open.

The new `%wait(priority=<int>)` keyword and `%w` alias must compose with positional wait targets plus `bead=`, `time=`,
and `runners=`. Lower integer values run first; an omitted or invalid legacy marker field resolves to the constant
default `10`. Within one priority band, the current oldest-waiter FIFO ordering and deterministic tie-breakers remain
unchanged. Eligibility remains independent of priority, so an earlier high-priority waiter whose `runners=` threshold is
not satisfied cannot block the first eligible lower-priority waiter.

## Directive and launch metadata

Extend the xprompt directive collection, value resolution, and typed result to recognize one `priority=` occurrence
across all `%wait` directives, reject duplicates, empty/non-integer/negative values with targeted `DirectiveError`
messages, and update the supported-keyword diagnostic. Thread the parsed value through `AgentInfo`; persist it to
`agent_meta.json` only when explicitly provided, matching the existing optional directive fields.

Treat a priority-only wait as sufficient deferred-start metadata. Pass the value from the runner into the runner-slot
gate so a prompt such as `%wait(priority=20)` can remain in the placeholder workspace until admission and claim its real
workspace only afterward. Update shared test fixtures that construct `AgentInfo` explicitly.

## Marker resolution and admission

Add `DEFAULT_WAIT_PRIORITY = 10` to the runner-slot package and expose it to the gate. Mirror the existing threshold
resolver with priority-specific semantics: a parked marker containing a real non-negative integer wins, otherwise the
directive value wins, otherwise the default applies. Booleans must not count as integers. Always write the resolved
`wait_priority` into a parked `waiting.json`, retaining its original `slot_requested_at`; this both makes the queue
inspectable and ensures a live hand-edit is retained and used on the next locked poll.

Mirror the additive Rust scan field in Python's tolerant `WaitingMarkerWire`. Extend `RunnerSlotWaiter` with the
defaulted priority, populate it defensively from scanned markers, and lead the queue sort key with priority before the
existing invalid-date/FIFO/timestamp/path components. Do not alter `may_start`: its first-eligible selection already
provides the required interaction between priority ordering and per-waiter runner thresholds.

## User-facing completion

Add `priority=` to the Ace prompt-bar `%wait` usage hint and keyword completion catalog with concise semantics that
lower values start first and the default is 10. Preserve keyword filtering and selected-value exclusion behavior,
updating the exact candidate-order and metadata assertions affected by the new entry. No queue-priority rendering or
wait-modal editor is included.

## Verification and completion

Expand directive tests for explicit values, alias support, composition, duplicates, invalid inputs, and the revised
unsupported-keyword message. Expand admission tests for priority-over-FIFO, FIFO within equal priority, defaulting for
missing/invalid values, deterministic ordering, and skipping an ineligible higher-priority waiter. Extend locked-gate
tests for marker persistence and live marker override, plus deferred-workspace coverage for a priority-only prompt.
Update scan-wire round-trip/fixture coverage where the waiting projection is asserted, and add a fakey end-to-end case
whose park order differs from its priority admission order.

Before repository checks, run `just install` as required for an ephemeral SASE workspace. Run focused pytest selections
while iterating, then run `just check` as the final repository-wide validation. Reinspect the diff and bead state,
record concise implementation notes if useful, and close `sase-8c.2` only after all checks pass.
