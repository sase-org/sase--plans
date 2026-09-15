---
tier: epic
title: Finish fleet acceptance evidence and phase handoff
goal: Fresh Athena-to-Apollo evidence is durable and the two acceptance phases explicitly
  owned by the fleet ghost-row plan are closed only after every live gate passes.
parent_bead: sase-xe.16.11.7.16.5
phases:
- id: capture-and-close-live-acceptance
  title: Capture durable fleet evidence and close the acceptance phases
  depends_on: []
  size: medium
  description: 'capture-and-close-live-acceptance: repeat the fresh Athena-to-Apollo
    fleet proof, register and attach its complete evidence, and normally close the
    two still-open acceptance phases.'
proposed_by: bbugyi200.apollo.sase-xe.16.11.7.16.5.land
create_time: 2026-09-15 16:01:46
status: done
bead_id: sase-xe.16.11.7.16.5.5
---

- **PROMPT:** [prompts/202609/finish_fleet_acceptance_evidence.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_fleet_acceptance_evidence.md)
- **PARENT:** [202609/fleet_ghost_rows_remaining.md](https://github.com/sase-org/sase--plans/blob/main/202609/fleet_ghost_rows_remaining.md)
- **BEAD:** [sase-xe.16.11.7.16.5.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.7.16.5.5.md)

# Plan: Finish fleet acceptance evidence and phase handoff

## Context and boundaries

The implementation under parent epic `sase-xe.16.11.7.16.5` is present and its focused
regression suites pass on the integrated repository heads. The Rust owner now suppresses
unprotected terminal family-member rows, the gateway publishes `gateway_version`, the
viewer preserves feed issues and consumes that version, and the display adapter uses
logical agent identity before exact attempt identity. The Rust changes shipped in
`sase-core` 0.34.32 and the primary repository currently requires and locks 0.34.35.

The remaining gap is acceptance durability and bead state, not a request to redo those
features. Phase `sase-xe.16.11.7.16.5.4` reported clean live Athena TUI and raw fleet
checks, but it registered no durable artifact and intentionally closed only its own
phase. Consequently `sase-xe.16.11.7.15.7` and `sase-xe.16.11.7.16.1` remain open with
only earlier failing ghost-row evidence. Their descriptions and the parent plan
explicitly authorize this remaining work to attach passing evidence and close them
normally.

Do not close `sase-xe.16.11.7.15`, `sase-xe.16.11.7.16`, or `sase-xe.16.11.7.16.5`;
their waiting land agents own those closures. Do not mark any linked plan file done. Do
not treat exact locator fields such as `attempt-0` in raw identity keys as a display
failure; the forbidden-label check applies to display-relevant labels and rendered rows.

## Phase: Capture durable fleet evidence and close the acceptance phases

Re-run the live acceptance from Athena viewing Apollo on the current installed builds.
First inspect both checkouts and installed/runtime versions without overwriting dirty
remote work. Restart or update only through the supported deployment path if the
installed command and Apollo gateway process do not already contain the landed fleet
changes.

Create one timestamped Markdown transcript containing enough command output and pane
captures for an independent reviewer to verify all of the following:

- Athena's human and JSON machine status reaches Apollo and reports the real
  `sase-gateway` package version, with match/skew/unknown text derived honestly from the
  live hello response.
- A fresh, non-cached authenticated projection and both presentation and explicit
  terminal-history catalog checks agree with Apollo-local listing semantics. No display
  label or rendered row contains the forbidden ghost names `lane`, `attempt-0`, or
  `y--plan`.
- Athena TUI captures filtered to Apollo cover both by-project and by-machine grouping.
  Legitimate remote family/clan/host chips, project labels, timing, and liveness remain
  present where applicable, while remote rows do not acquire local-only `here` state.
- The healthy live Apollo feed stays quiet. Also record the focused invalid-host and
  stale-cache diagnostic regressions (or an equally safe deliberate diagnostic probe)
  showing the normalized code/message and cache age remain reachable even when a host
  produces zero agent rows. Do not leave a persistent invalid machine configuration.

Register the transcript with `sase artifact create`, attach the resulting canonical
reference to `sase-xe.16.11.7.15.7`, and append a root-cause/result note that contrasts
the earlier failing evidence with the new owner-side terminal-family suppression and
logical-label fallback. Confirm the reference appears in `sase bead show`.

Only after every gate above passes, close `sase-xe.16.11.7.15.7` normally with a note
naming the artifact reference. Then close `sase-xe.16.11.7.16.1` normally with a note
confirming its delegated target phase is closed and the live proof passed. If any live
gate fails, leave both phases open, record the exact failure and evidence, and repair
the causally related regression within this nested epic before retrying.

Run the focused Rust presentation/gateway tests and the focused primary fleet
status/projection/display tests on the final integrated heads. If source changes become
necessary, follow each repository's required lint/test policy before closing this nested
epic's phase.
