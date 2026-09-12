---
tier: epic
title: Finish weighted-capacity lifecycle and published-package proof
goal: Prove real weighted monitor and gate handoffs on the integrated tree and deliver
  a compatible published SASE, core, and research-plugin cohort.
parent_bead: sase-z4.6.5.4
phases:
- id: lifecycle-proof
  title: Complete production-path weighted lifecycle acceptance
  depends_on: []
  description: 'lifecycle-proof: drive real monitor-next delivery and gate execution
    under contention, preserve zero-valued queue inputs, and prove timeout/crash isolation
    on the current integrated tree.'
  size: medium
- id: package-contract
  title: Repair the research package compatibility contract
  depends_on: []
  description: 'package-contract: reconcile the research plugin''s incompatible core
    window and stale wheel assertions with the actual containing release and the current
    SASE API requirements.'
  size: medium
- id: published-proof
  title: Establish and verify the published minimum-version cohort
  depends_on:
  - lifecycle-proof
  - package-contract
  description: 'published-proof: establish real containing releases through existing
    automation, run clean wheel-only positive and negative smoke checks without skipped
    research acceptance, and preserve the flag-retirement evidence.'
  size: medium
proposed_by: bbugyi200.athena.sase-z4.6.5.4.land
create_time: 2026-09-12 06:29:11
status: wip
bead_id: sase-z4.6.5.4.6
---

- **PROMPT:** [prompts/202609/weighted_capacity_lifecycle_and_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/weighted_capacity_lifecycle_and_release.md)
- **PARENT:** [202609/weighted_capacity_remaining_acceptance.md](https://github.com/sase-org/sase--plans/blob/main/202609/weighted_capacity_remaining_acceptance.md)
- **BEAD:** [sase-z4.6.5.4.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-z4/sase-z4.6.5.4.6.md)

# Finish weighted-capacity lifecycle and published-package proof

The full audit is `file:explicit:d4190118a754296843815658`; read it with
`sase artifact read` for the exact child-note dispositions and verification limits.

This is only the remaining work discovered while landing `sase-z4.6.5.4` at SASE
`96c3877e0` on 2026-09-12. The parent link resumes that interrupted landing after this
child lands. Ancestor closing, post-close Symvision, and linked-plan status updates are
landing actions, not implementation phases here.

Read the parent's landing audit and all subsequent notes, plus these contracts:

```sh
sase bead show sase-z4.6.5.4
sase bead show sase-z4.6.5.4.2
sase bead show sase-z4.6.5.4.5
sase artifact read plan:202609/weighted_capacity_remaining_acceptance.md "Complete the missing lifecycle and release acceptance"
sase artifact read plan:202609/weighted_capacity_landing_repairs.md "Preserve the governing ownership and rollout contract"
```

Use `/sase_repo` to open `gh:sase-org/sase-core` and
`gh:sase-org/sase-research-artifacts`, use only its returned paths, and read each
repository's AGENTS.md. Configure source-based tools with those opened paths. Shared
ownership and eligibility policy belongs in Rust; Python supplies process, marker, lock,
and presentation glue. Release automation owns package/crate release versions.

## Verified work and drift

Do not repeat the pin repair or replace the completed parity suite. Commit `3e32c5cc6`
pins the lineage-bearing core; the current pin
`0a72d7df232a259450d00d044234ad90185dae47` still contains index schema 27 and the
ownership wire. Core tags from `v0.34.0` contain the original pinned `da0a73895ff8`.
SASE currently declares `sase-core-rs>=0.34.15,<0.35.0`; newer continuation and bead
code may need a later floor, so preserve every current binding rather than lowering the
dependency to the first weighted-only release.

Commit `74a4e4282` adds seven shared-snapshot runtime/CLI/TUI tests. Commit `3e39ebdce`
changes exactly 90 PNGs. An independent pixel audit found every change confined to
status-strip rows y=91..116, with no body changes. These are real completed artifacts,
though the eventual combined tree still needs verification.

The relevant subsequent changes include the queue `capacity` contract and bead/gate
plumbing (`e48aa7db0`, `39cc0c4b8`, `41806ee98`, `14ddd4d82`), monitor delivery and
resume (`56ceab3f9`, `4c0d1c216`), and fleet/header changes (`f7a570268`, `e2229a680`).
`tests/test_capacity_gate_to_admission.py` already proves the new threshold measures
weighted load. Keep those tests and the shared-snapshot suite consistent with this
contract. Do not reintroduce headcount semantics under the old internal `wait_runners`
wire name.

The audit environment's `.venv` held core 0.32.61, missing even
`agent_artifact_index_schema_version`; it is not verification evidence. Provision a
current environment before testing, record the actual installed binding identity, and
distinguish source integration from published-wheel proof.

## lifecycle-proof

The six tests introduced by `25b5d4cf7`, now split between
`tests/fakey/test_monitor_capacity_e2e.py` and `test_gate_capacity_e2e.py`, cover useful
portions of the contract, but leave these gaps:

1. The weight-2 monitor test starts a monitor without a next action, kills the parked
   competitor before monitor completion, edits only an in-memory `monitor_next_action`
   afterward, and manually calls `launch_followup_agent`. Its spawn stub hard-codes the
   successor's weight. This bypasses the real supervisor settlement and
   `wait_for_followup_started` boundary. Add acceptance with the next action authored at
   creation and dispatched by the real monitor path into a controlled fakey successor.
   Keep the competing waiter alive across command completion and delayed child
   bootstrap. Observe persisted owner keys, effective weights, and actual occupied
   capacity: one lineage must retain one 2.0 claim without allowing the contender
   through or double-charging the family. A monitor with no next action may correctly
   release; do not confuse that case with a continuing monitor. Exercise the current
   delivery reservation/adoption path rather than bypassing it with fabricated child
   metadata.
2. `src/sase/monitor/continuation_delivery.py::queue_launch_prefix` uses truthiness
   fallbacks for `wait_priority`/`queue_priority` and `wait_runners`/`queue_capacity`. A
   stored zero disappears or is replaced by the alternate key. This was reported to
   active owner `sase-zl.13` with the introducing commit `56ceab3f9`. Recheck that
   owner's work first; consume its fix if landed, otherwise make the narrow integration
   repair and record it there. Verify absent, zero, nonzero, and both-key cases through
   actual directive parsing and successor launch. Preserve authored explicitness and
   effective weight; do not duplicate delivery machinery. Include the canonical
   `capacity` spelling.
3. Keep the existing independently weighted parallel-lineage and pending/manual/
   detached-gate tests. Add the missing timeout and crash cases with an unrelated live
   owner and a parked competitor. Prove both the failed lineage's reclamation and the
   unrelated lineage's continued occupancy, not merely an unchanged owner string. Cover
   failed startup and cancellation through the corresponding real settlement paths,
   retaining existing coverage where adequate.
4. Resolve `.2` proposal #2 explicitly. `notification_gates/service.py` accepts
   `GateSpec.auto.enabled` and `_resolve_auto_gate` calls the executor without gate
   capacity callbacks. Trace the real shell-backed creation route and the creator's
   existing claim. If this is a participating execution, prove reuse/acquisition before
   its command runs; fix any bypass through the existing admission path. If the
   combination is unsupported or intentionally nonparticipating, establish that through
   the actual route/validation and a regression, with a precise scope explanation.
   Merely renaming manual answer coverage to "automatic" is insufficient.
   Nonparticipating procs remain outside this feature's budget.

Read active `sase-zl.13`, `sase-zp`, and `sase-zm.5` before changing shared paths. The
latter plans independent command weights in a later feature; do not absorb its new
command-reservation model. Preserve the currently shipped serial-lineage contract. Run
focused lifecycle, lineage, shared-snapshot, and gate-to-admission tests plus the
required repository checks. Use `/sase_monitor` for long commands.

## package-contract

The research checkout at `5aaa244` already emits `%q(w=0.25, capacity=...)`, but its
`pyproject.toml` requires `sase>=0.17.2` and `sase-core-rs>=0.33.0,<0.34.0`. This is
incompatible with SASE's current core window. Release PR #2's CI run 34664698929 fails
dependency resolution on that exact conflict, before tests.
`tests/test_wheel_contract.py` also hard-codes `<0.34.0` and `0.33.*`.

1. Recheck current source and release metadata, then align the plugin's dependency
   constraints, wheel checks, and minimum-version constants with the actual SASE/core
   cohort. Preserve newer core capabilities. Use supported floor/pin tooling where
   available and never hand-edit release-plz-owned versions.
2. Run plugin source and built-wheel integration checks. Repair stale template-test
   assumptions exposed by the `%wait` to `%queue` and `runners` to `capacity` moves;
   preserve four quarter-weight units and their dependency graph, including zero
   priority/capacity. Do not satisfy these tests by installing an old host or masking
   incompatible dependencies with overrides in the release-proof lane.
3. Make the separate published-minimum lane exercise the exact minimum versions and fail
   clearly for an incompatible cohort. Keep any source-coordination lane clearly
   distinct; it cannot count as positive published acceptance.

Run the plugin's required checks and tests for package metadata and expansion. If SASE's
dependency floor needs to advance for newer bindings, coordinate that with their active
owner and verify the combined binding inventory before the ratchet.

## published-proof

On 2026-09-12 the package index served core 0.34.19, SASE 0.17.1, and research plugin
0.2.0. The host and plugin containing releases did not exist. SASE release PR #299
(0.17.2) had a passing floor smoke; research release PR #2 (0.3.0) failed on the
constraint conflict above. Read current state rather than treating these version numbers
as promises.

1. Establish containing SASE and plugin releases through each repo's normal release
   automation. Verify tag ancestry and exact package-index availability. Prepare
   concrete release changes/checks before any approval required by that workflow. If an
   external publication blocker remains, retain this phase as incomplete with exact
   evidence; a core release alone cannot fulfill it.
2. In a fresh environment install only published wheels for the exact compatible minimum
   SASE, core, and plugin versions. No editable installs, local distribution paths,
   source overrides, or `maturin develop` in this proof. Record versions, wheel
   identities, and commands. Exercise repaired candidate decisions, scanner ownership
   projection, weighted fleet summaries, and all four research segments. Preserve a
   negative smoke against the last incompatible cohort and make the release/floor
   assertion depend on the positive behavioral checks.
3. Commit `9202146ca` added a skip when the installed research template contains retired
   `%wait(priority=...)`. A skipped optional local test does not prove this contract.
   Make the release acceptance require the compatible plugin and execute the
   four-quarter-weight test; no stale-plugin skip may satisfy this phase.
4. `sase-z5` was already closed by another agent on 2026-09-11 after its registry
   entry/Off branch had been removed. Re-read its note and verify those remain absent.
   Append the completed acceptance/release evidence to that existing bead; do not
   recreate the flag or mistake its earlier administrative close for package proof.
   Confirm rollout guidance still addresses draining/replacing old schedulers.

Use the current floor probe and every changed repo's required checks. Before the
combined tree lands, run `just check-full` only through `/sase_monitor` with
TESTING/TESTED. The resumed parent lander must also resolve the historical proposed
follow-up dispositions recorded in its audit, recheck relevant visual drift, and perform
the user's normal descendant/ancestor readiness checks. None of those ancestor closing
actions is a child phase here.
