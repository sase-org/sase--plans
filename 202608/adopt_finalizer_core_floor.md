---
tier: tale
title: Adopt the finalizer integrity core floor
goal:
  Published installs contain every shared authenticated-plan API required by sase-rr.5.
size: small
proposed_by: bbugyi200.athena.sase-rr.5.land
bead: sase-rr.5
create_time: 2026-08-21 23:28:07
status: wip
---

- **PARENT:** [202608/finalizer_integrity_closeout.md](finalizer_integrity_closeout.md)
- **BEAD:**
  [sase-rr.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rr/sase-rr.5.md)

# Adopt the finalizer integrity core floor

## Goal

Raise the primary repository's `sase-core-rs` dependency floor from 0.29.6 to the
published 0.29.9 release that contains `validate_finalizer_plan` and
`authenticate_finalizer_plan`, then refresh the committed lock metadata so an install
without the linked core checkout has every shared contract required by closed child epic
`sase-rr.5`.

## Implementation

1. Update the `sase-core-rs` requirement in `pyproject.toml` to `>=0.29.9,<0.30.0` and
   refresh only the corresponding dependency metadata in `uv.lock` using the
   repository's supported lock workflow.
2. Verify the core-floor probe no longer reports the two finalizer-plan capabilities as
   stale/actionable, and run the focused finalizer facade/plan-integrity tests that
   exercise those bindings.
3. Run `just check`. Preserve unrelated work and do not change finalizer behavior,
   feature flags, plans, memories, or generated skills.

## Acceptance criteria

- A normal install resolves a published core release containing both authenticated-plan
  APIs without relying on the linked checkout.
- `tools/probe_core_floor` reports the declared floor as current for the required
  capabilities.
- Focused finalizer tests and `just check` pass, or any unrelated failure is recorded
  through the required task-bead workflow.
