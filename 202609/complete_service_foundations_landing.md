---
tier: epic
title: Complete service-foundation landing integration
goal: 'Service status derives boot-scoped stops and duplicate observations correctly,
  the shared Python supervision library delegates restart accounting to the Rust core
  decision function now that it exists, and sase pins the integrated core revision.

  '
parent_bead: sase-11y.2.1
phases:
- id: status-runtime-scope
  title: Correct service-status runtime scoping
  depends_on: []
  size: small
  description: 'status-runtime-scope: make sase-core honor the request boot id when
    projecting stops, deduplicate configured and orphan observations consistently,
    and add Rust and binding-level regressions.'
- id: supervision-core-delegation
  title: Delegate shared restart accounting and ratchet core
  depends_on:
  - status-runtime-scope
  size: medium
  description: 'supervision-core-delegation: preserve the shared supervision API while
    delegating restart accounting to the Rust-backed service restart facade, prove
    AXE compatibility, ratchet sase-core-revision.txt past the completed core work,
    and run repository verification.'
proposed_by: bbugyi200.athena.sase-11y.2.1.land
create_time: 2026-09-17 19:38:44
status: done
bead_id: sase-11y.2.1.5
---

- **PROMPT:** [prompts/202609/complete_service_foundations_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/complete_service_foundations_landing.md)
- **PARENT:** [202609/core_service_foundations.md](https://github.com/sase-org/sase--plans/blob/main/202609/core_service_foundations.md)
- **BEAD:** [sase-11y.2.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11y/sase-11y.2.1.5.md)

# Plan: Complete service-foundation landing integration

This plan contains only work uncovered while landing `sase-11y.2.1`. The four original
child phases are closed, but source and post-start drift review found two requirements
that cannot be treated as follow-up work:

1. `ServiceStatusRequestWire.boot_id` is serialized but unused. Consequently,
   `build_service_status` projects a stop recorded for a different machine boot as
   active whenever it receives an unpruned `ServiceStateWire`, contradicting the
   boot-scoped-stop contract. The same review found that duplicate observations are
   documented as "using the last", but orphan observations are still emitted once per
   duplicate input row instead of using the already-built last-by-name map.
2. The concurrently landed `sase-11y.3` supervision extraction deliberately retained its
   local restart arithmetic until the core-service pure decision function existed. Both
   now exist, so leaving `src/sase/supervision/restart.py` as a second accounting
   implementation would violate the parent plan's explicit integration requirement and
   the Rust core boundary.

The repeated phase proposals to advance `sase-core-revision.txt` are also completed
here, after the corrective core commit exists. Closing the parent bead, cleaning its
epic-symbol entries, and marking its original plan done remain the waiting land agent's
responsibility and are intentionally outside these phases.

## Phase `status-runtime-scope`: Correct service-status runtime scoping

Work in the linked `sase-core` repository and follow its `AGENTS.md` verification rules.

- In `crates/sase_core/src/service/status.rs`, treat a `ServiceStopWire` as active only
  when its optional `boot_id` equals `ServiceStatusRequestWire.boot_id`; two absent ids
  match, exactly as in the state store. Use the filtered stop for `desired`, state,
  summary, and the emitted `stop` field. A stop from another boot must not produce
  `stopped until next boot` or force `desired: stopped`.
- Reuse the existing last-observation-by-name map for both configured entries and
  orphans. Preserve configured order, emit each orphan name once in deterministic name
  order, retain the last supplied observation, and keep the duplicate diagnostic
  truthful.
- Add focused Rust tests for matching, mismatched, and `None` boot-id stop semantics,
  and for duplicate configured/orphan observations selecting the last row without
  duplicate output.
- Extend the PyO3 status binding round-trip test as needed so the corrected wire
  behavior is covered through the public binding. Do not change wire schema versions;
  this corrects derivation semantics without changing shapes.
- Run focused service-status tests and the repository-required `just check`.

## Phase `supervision-core-delegation`: Delegate shared restart accounting and ratchet core

Work in the primary `sase` repository after `status-runtime-scope` has landed in the
linked core repository.

- Refactor `src/sase/supervision/restart.py` so `schedule_restart` converts the existing
  mutable `RestartState` and `RestartPolicy` into `ServiceRestartHistory` and
  `ServiceRestartTuning`, calls the Rust-backed `decide_service_restart` facade with
  policy `always`, and copies the returned history/deadline back into the existing
  mutable state. Preserve `RestartState`, `RestartPolicy`, `record_started`, and
  `schedule_restart` signatures and preserve `schedule_restart`'s boolean meaning:
  return true only for the decision's one-per-crash-loop `notify` edge. Continue
  recording `last_exit_code` for AXE callers.
- Keep the AXE orchestrator behavior and messages stable. Add focused tests that mock or
  wrap the facade boundary to prove the shared layer delegates, plus parity tests for
  capped backoff, healthy-run reset, failure-window pruning, and one notification per
  episode. Run the existing AXE restart/outage tests as regression coverage.
- Run `just ratchet-core-revision` only after confirming the remote core head contains
  the `status-runtime-scope` commit. The pin may advance over intervening compatible
  core commits, but it must not remain behind any service-foundation binding or the
  status correction. Do not manually alter the published `sase-core-rs` version window;
  release automation owns it.
- Run `just install` so the workspace uses the ratcheted linked core, then `just fix`,
  the focused supervision/AXE/service tests, `just check`, and the project-required full
  verification path. Confirm `sase bead epic-symbols sase-11y.2.1` is empty; do not
  remove or re-key symbols owned by still-open downstream beads such as `sase-11y.4`.
