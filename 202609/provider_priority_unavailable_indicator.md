---
tier: tale
title: Finish unavailable provider-priority presentation
goal: Provider-priority intent is truthful in the top bar and orphaned modal states.
size: small
proposed_by: bbugyi200.athena.sase-xf.land
bead: sase-xf
status: done
---

- **PARENT:** [202609/provider_priority.md](provider_priority.md)
- **BEAD:**
  [sase-xf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xf/README.md)

# Finish unavailable provider-priority presentation

The `sase-xf` landing audit found one narrow acceptance gap in the otherwise-complete
temporary provider-priority feature. The Provider Routing modal rows can show
`priority unavailable` or `priority soft-disabled`, but the ACE top-bar provider-routing
indicator always describes an active priority as `preferred`. This is false when the
priority provider is hard-disabled, soft-disabled, missing its CLI, no longer
registered, or hidden from user-facing provider routing. The modal summary also fails to
mark an orphaned priority unavailable when there are no visible provider rows.

## Implementation

1. Extend the top-bar provider-routing indicator to derive the priority provider's
   effective state from one lock-free cached routing snapshot plus cached provider
   facts. Use the existing Rust-backed availability classifier; do not duplicate the
   precedence policy in Python and do not add synchronous filesystem work to render or
   timer paths. Render and describe active, soft-disabled, and unavailable priority
   intent truthfully, while preserving the current real-disable count/composition and
   click route.
2. Make the Provider Routing summary label an active priority unavailable when its
   provider has no visible status row, including the empty-provider-list recovery case;
   keep `c` clear-priority reachable.
3. Add focused unit coverage for hard-disabled, soft-disabled, missing-CLI, and orphaned
   priority presentation. Add or update a representative 120x40 visual snapshot for the
   unavailable top-bar indicator only if the rendering change is not already exercised
   by an existing fixture, and inspect any changed PNG.

## Verification

Run the focused provider-indicator and Provider Routing rendering tests, the relevant
visual snapshot case when changed, `just symvision`, and the repository-required
`just check`. Keep periodic UI callbacks limited to cached/stat-gated reads and pure
Rust classification.
