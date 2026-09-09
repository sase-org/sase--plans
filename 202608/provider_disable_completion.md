---
tier: epic
title: Complete provider-disable Models-panel correctness and acceptance
goal: "Provider disabling remains off the Textual event loop, rejects disabled custom
  targets, reports only real routing changes, and satisfies the original behavior and
  visual acceptance matrix.

  "
parent_bead: sase-mc
phases:
  - id: runtime
    title: Make provider-routing state safe and exact
    depends_on: []
    description:
      "runtime: remove synchronous provider-state reads from rendering, close
      custom-target bypasses, and emit routing changes only for effective mutations."
    size: medium
  - id: acceptance
    title: Complete behavior and visual acceptance coverage
    depends_on:
      - runtime
    description:
      "acceptance: exercise the full provider manager lifecycle, failure and concurrency
      edges, and the missing provider-specific PNG states."
    size: medium
proposed_by: bbugyi200.athena.sase-mc.land
bead_id: sase-mc.5
create_time: 2026-09-09 19:51:16
status: wip
---

- **PROMPT:**
  [prompts/202608/provider_disable_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/provider_disable_completion.md)
- **PARENT:** [202608/temporary_provider_disabling.md](temporary_provider_disabling.md)
- **BEAD:**
  [sase-mc.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-mc/sase-mc.5.md)

# Plan: Complete provider-disable Models-panel correctness and acceptance

## Context and constraints

The existing Rust store, Python facade, routing filters, default resolution,
Models-panel manager, and top-bar indicator are present and their focused 43-test suite
passes. This plan addresses only the completion gaps found while landing `sase-mc`.
Preserve the Rust store as the authoritative cross-process state and preserve the
asynchronous snapshot model already established by the provider manager. Rendering and
event handlers may use an in-memory snapshot, but they must not acquire the
provider-disable file lock or touch disk on the Textual event loop.

Recent Models-panel and route-selection changes on the current branch are integration
inputs. Re-read the current implementations before editing, preserve unrelated behavior,
and keep provider availability orthogonal to model aliases, effort, and route sizing.

## Phase `runtime`: Make provider-routing state safe and exact

Remove authoritative provider-disable reads from synchronous Models-panel composition
and row refresh. Build alias views from the panel's loaded in-memory provider snapshot,
and restrict store reads and writes to workers or other explicitly off-thread paths.
Keep the loading state conservative and refresh rows once a valid snapshot arrives. Add
a regression that fails if compose or synchronous refresh reaches the authoritative
facade.

Validate free-form custom model targets at the final submission boundary in both
override and alias-edit flows. When an explicit provider prefix is disabled, reject the
target with clear user-visible feedback before preview or persistence; do not rely only
on filtering the predefined selector. Preserve the existing treatment of provider-less
values, hidden providers, aliases, and enabled explicit providers.

Separate snapshot synchronization from effective user mutations. Opening the Provider
Routing modal and receiving its initial snapshot must not mark the Models panel changed,
invalidate defaults, or request a routing refresh. A successful disable, replacement,
enable, or observed expiry should publish the appropriate snapshot and change signal
exactly once when effective state changed. Preserve stale-worker generation guards and
safe close/unmount behavior.

## Phase `acceptance`: Complete behavior and visual acceptance coverage

Expand focused tests beyond the current five provider-manager cases. Cover exact-map
replacement rather than additive writes; every duration choice including custom and
until-cleared; enable and idempotent enable/disable behavior; Back and cancel paths;
disabled-provider rejection for free-form override and edit targets; initial-load versus
mutation change signaling; countdown expiry and automatic re-enable; worker failure;
stale completion; close/unmount races; cursor preservation; and routing/default refresh
effects. Tests must prove UI-thread code consumes an in-memory snapshot while worker
paths remain authoritative.

Complete the provider-specific PNG matrix. Retain the existing manager, disabled-row,
and single-indicator snapshots, and add enough intentional states to meet the original
minimum: provider duration copy, multiple disabled-provider pills, a narrow manager, and
at least one additional state that demonstrates until-cleared or disabled custom-target
feedback. Inspect PNG diffs before accepting goldens and do not absorb the unrelated
Artifacts query-migration visual failures already routed to `sase-m6.6.1`.

Run `just install`, the focused provider-disable and Models-panel suites, the focused
provider PNG snapshot subset, and `just check`. If the scoped lane broadens or reports
an unusual selection, follow the repository rule for monitored `just check-full`. Record
any new unrelated failure as a proposed follow-up with exact evidence.

## Completion criteria

- No synchronous Models-panel render or refresh path reads authoritative
  provider-disable state or waits on its file lock.
- Free-form override and edit submissions cannot select a disabled explicit provider.
- Initial snapshot loading is not a user-visible routing change; effective mutations and
  expiry propagate exactly once.
- The original provider behavior matrix and at least seven provider-specific PNG states
  are covered, inspected, and green.
- Focused verification and the required repository check lane pass without weakening the
  provider-disable contracts.
