---
tier: tale
title: Put Agent first in the Artifacts sub-tab order
goal:
  Make Agent the first Artifacts sub-tab and bind it to numeric shortcut 1 while keeping
  every rendered, cycled, and fallback shortcut synchronized with visual order.
size: small
proposed_by: bbugyi200.athena.0dx
create_time: 2026-08-26 05:02:29
status: wip
---

# Plan: Put Agent first in the Artifacts sub-tab order

## Context and intent

Artifacts panes are resolved as an ordered descriptor sequence. That sequence drives the
tab strip, forward/reverse cycling, descriptor/contract order metadata, quick-start
copy, and the runtime numeric bindings. Today Agent is appended after provider panes,
while a separate static `DEFAULT_BINDINGS` table supplies the pre-registry fallback
digits.

Move Agent to the front of the visual and cycle order and make its digit `1` in both
binding paths. Keep `DEFAULT_ARTIFACTS_SUBTAB` as Stitch: this request changes ordering
and shortcuts, not which potentially data-backed pane activates during startup. Continue
to place configured document-provider panes after Bead and immediately before File, and
preserve the existing rule that File receives the highest available numeric shortcut.

## Implementation

1. Update the canonical fixed Artifacts ordering metadata and the provider-aware
   resolver so their fixed-pane sequence is Agent, Stitch, Patch, Bead, File, with
   providers still inserted between Bead and File. Ensure the generated descriptor
   digits and contract `order`/`digit` values follow that resolved order, yielding
   Agent=`1`, Stitch=`2`, Patch=`3`, Bead=`4`, then provider digits, with File
   last/highest.
2. Update the class-level zero-provider fallback bindings to expose the same fixed-pane
   mapping (Agent=`1` through File=`5`). Leave runtime binding generation descriptor-
   driven so installed providers continue to shift only provider/File positions without
   displacing Agent from `1`.
3. Bring deterministic TUI fixtures and order-dependent presentation expectations in
   line with production: include Agent first in the fast Artifacts descriptor fixture,
   update quick-start labels/keycaps, cycle/wrap expectations, public fixed-order
   assertions, degraded-provider ordering, and any intentional tab-strip visual
   snapshots. Replace positional literals in interaction tests with the existing live
   digit helper where the test is about behavior rather than the exact mapping.
4. Add or strengthen focused regressions that explicitly prove both keymap paths map `1`
   to Agent, the resolved order remains correct with healthy or degraded providers, the
   fixed panes retain consecutive digits in the no-provider case, cycling crosses the
   new first position correctly, and Stitch remains the initial active pane.

## Verification

- Run the focused Artifacts registry/digit, keymap-binding, quick-start, scaffold, and
  Agent-pane tests that cover ordering, digit dispatch, provider insertion, and cycling.
- Run the dedicated ACE PNG snapshot suite for the intentionally changed Artifacts tab
  strip and update only snapshots whose tab order/number labels changed.
- Run `just check` as the required repository-wide lint plus diff-scoped test gate. If
  it reports stale environment dependencies, run `just install` and repeat `just check`.

## Acceptance criteria

- Agent renders as the leftmost Artifacts sub-tab and is the first destination in the
  sub-tab cycle order.
- Pressing `1` while Artifacts is active selects Agent in both runtime-generated and
  class-level fallback bindings; every other displayed digit agrees with its visual
  position, including provider panes and the last/highest File shortcut.
- Provider discovery success or degradation does not reorder Agent away from first or
  File away from last.
- The Artifacts tab still initially activates Stitch, and no new synchronous work is
  added to startup, input handlers, or rendering paths.
- Focused tests, intentional visual snapshots, and `just check` all pass.
