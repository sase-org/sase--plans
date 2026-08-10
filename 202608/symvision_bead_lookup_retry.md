---
tier: tale
title: Make Symvision bead status lookup resilient to transient store races
goal:
  Existing in-progress epic beads are not misclassified as missing by transient
  Symvision status-probe failures.
size: medium
proposed_by: bbugyi200.athena.sase-it
bead: sase-it
create_time: 2026-08-10 10:43:24
status: wip
---

- **BEAD:**
  [sase-it](https://github.com/sase-org/sase--beads/blob/main/pages/sase-it/README.md)

# Make Symvision bead status lookup resilient to transient store races

## Objective

Prevent an existing in-progress epic bead from being misclassified as missing when
Symvision's one-shot `--epic-symbol` status probe overlaps a transient bead-store
refresh or publication window.

## Context

The `_lint-symvision` recipe supplies `tools/sase_bead` as `BD_COMMAND` and enables its
status-only mode. Symvision invokes that command once for each epic symbol and treats
every nonzero result as authoritative proof that the bead does not exist. Two
independent `just check` runs instead observed the same pattern: the probe failed for
`sase-i8`, then the wrapper and complete Symvision command both succeeded immediately
afterward. Symvision also captures and discards the wrapper's stderr, so a wrapper-only
diagnostic cannot correct the misleading upstream message.

## Implementation

1. Add bounded retry behavior to the status-only `show` path in `tools/sase_bead`.
   Capture each attempt's output, return the first successful primary status line, and
   retry only this opt-in Symvision path after a short delay. Preserve the final
   underlying exit status and diagnostic if every attempt fails so genuinely missing or
   unreadable beads still fail the lint gate.
2. Keep all other wrapper invocations as a direct `exec` of `sase bead`, preserving
   their existing arguments, output, and exit behavior.
3. Add executable-level tests that run a copied wrapper against a controlled fake
   checkout-local `sase` binary. Cover a transient failure followed by success, a
   persistent failure that exhausts the bounded attempts without being converted to
   success, status-line truncation, and unchanged non-Symvision delegation.

## Verification

1. Run the focused wrapper regression tests.
2. Run `just install`, as required for an ephemeral SASE workspace.
3. Run the exact previously failing path, `just _lint-symvision`, repeatedly enough to
   exercise the live status lookup.
4. Run the repository-required `just check` gate and investigate any failures.
5. Re-run the focused regression tests after the whole-repository gate, then inspect the
   final diff and working-tree status before closing `sase-it` with a note naming the
   checks that passed.
