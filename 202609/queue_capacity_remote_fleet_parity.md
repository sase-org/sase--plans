---
tier: epic
title: Finish queue-capacity remote parity and landing acceptance
goal:
  Queue capacity survives the remote fleet summary boundary with the same canonical
  semantics and ACE presentation as local agents, the deferred low-capacity launch is
  observed admitting after drain, and the integrated tree passes the complete landing
  gate.
parent_bead: sase-zt.6.5
phases:
  - id: remote-wire
    title: Carry canonical queue capacity through the Rust fleet summary
    size: medium
    depends_on: []
    description:
      "remote-wire: extend the current Rust fleet summary contract and owner projection
      with canonical queue-capacity value and explicitness, preserving the local loader
      precedence, legacy-read, explicit-zero, and schema compatibility contracts."
  - id: remote-consumer
    title: Restore queue-capacity parity in synthesized remote agent rows
    size: medium
    depends_on:
      - remote-wire
    description:
      "remote-consumer: ratchet the supported core cohort, consume the new fleet fields
      through the existing Agent adapter, and prove remote row/detail capacity
      presentation without changing admission accounting."
  - id: integrated-acceptance
    title: Complete drain, remote, and full landing acceptance
    size: medium
    depends_on:
      - remote-consumer
    description:
      "integrated-acceptance: integrate later queue and fleet changes, finish the
      missing drain-then-admit and remote presentation proofs, disposition the filed
      historical flake baseline entries, and pass focused plus full verification."
proposed_by: bbugyi200.athena.sase-zt.6.5.land
create_time: 2026-09-13 22:09:23
status: wip
---

- **PROMPT:**
  [prompts/202609/queue_capacity_remote_fleet_parity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/queue_capacity_remote_fleet_parity.md)

# Finish queue-capacity remote parity and landing acceptance

## Context

This plan contains only work still open after the landing audit of `sase-zt.6.5`. The
three original phases are closed and their implementation is present: core commit
`7f43a996` makes directive-name completion flag-aware, main commit `3224d461` preserves
canonical queue capacity through LaunchApproval, and the combined continuation and Jinja
repairs remain in `1ebcb2f1` and `0eb2bbea` plus their later integrated forms. Focused
and visual coverage passed, and the final full test-cost lane passed roughly 41,413
tests.

Two acceptance obligations remain:

1. Post-start core commit `d0ec62c` and main commit `65f876aa` made remote fleet agents
   real Agent rows with local/remote presentation parity. The current Rust
   `ResolvedAgentSummaryWire` and owner projection carry `queue_weight` but omit
   `queue_capacity` and its explicitness, while `_fleet_agents_rows._agent_from_summary`
   maps weight but never calls the existing `set_queue_capacity` adapter. A remote agent
   authored with `%queue(capacity=100)` therefore loses its `c100` row badge and
   `Capacity: 100 capacity units` detail. This is integration drift created after the
   queue-capacity epic began, not a reason to duplicate fleet rendering or parsing in
   Python.
2. The original acceptance live smoke proved an over-limit `c100` launch and a parked
   canonical `c1` launch, but unrelated live holders occupied the global limit and the
   phase correctly did not stop them. It therefore did not observe that exact `c1`
   launch transition from waiting to admitted after capacity drained.

The phase's `just check-full` test-cost run passed, but its final selection-health gate
reported 19 historical reproducible-flake node IDs. The landing agent completed the
required `/sase_new_task` sweep: `sase-u1`, `sase-10f`, and `sase-lk` received +1
evidence; the two artifact-audit nodes remain filed on closed deterministic repair
`sase-n1`; the known pre-fix incomplete-history record belongs to closed `sase-zu.8.5`;
the machines-pane node was already on `sase-j7`; and the remaining nodes were attached
to active causal epics `sase-j7`, `sase-xe.16`, `sase-xe.16.11.7.14.6`, `sase-n4`,
`sase-z4.6.5.4.6`, `sase-yz`, `sase-yy.8.6`, and `sase-kp`. No generic flake umbrella or
duplicate task was created. The current collectable IDs, not their pre-split file names,
must be used when updating `tests/reproducible_flake_baseline.txt`.

Preserve all completed contracts: canonical-only capacity writers with legacy and dual
readers, canonical-wins resolution, explicit zero, exact fractional effective weight,
flag-off compatibility, capacity flooring, waiting-marker precedence, local occupancy
accounting, LaunchApproval preservation, continuation inheritance, gold `cN` badges,
numeric over-limit styling, and the existing capacity visual golden. Coordinate with the
still-active remote-agent family `sase-xe.16.11.7.15` and gate-capacity landing
`sase-10h`; adopt their latest work and do not duplicate or overwrite it.

## Phase `remote-wire`: Carry canonical queue capacity through the Rust fleet summary

Open `sase-core` through `sase repo open` and start from its latest published lineage,
including the remote renderer and gate-capacity work. Extend `ResolvedAgentSummaryWire`
and the owner-side summary projection with the normalized queue-capacity value and an
explicitness bit. Derive them through the shared queue metadata contract: canonical
`queue_capacity` wins over legacy `wait_runners`, waiting metadata wins where the local
loader gives it precedence, absence remains distinct from explicit zero, and legacy
input may be read but is never emitted as the authoritative field. Do not introduce a
Python parser or duplicate admission arithmetic.

Advance the fleet wire/schema version if the serialized contract requires it and update
all Rust, PyO3, gateway, and compatibility fixtures that legitimately consume that
version. Cover local owner projection, serialized/deserialized remote summaries,
canonical-versus-legacy conflicts, waiting-over-meta precedence, explicit zero, a
positive value such as 100, and absent capacity. Verify older payloads without the new
fields remain readable. Run the core repository's required `just check`, land the core
change, and record the full core SHA and release/package availability on the phase bead
for the consumer phase.

## Phase `remote-consumer`: Restore queue-capacity parity in synthesized remote agent rows

Begin from current main after integrating the latest remote-agent work from
`sase-xe.16.11.7.15` and the current gate-capacity/cohort work; do not revert their
family synthesis, host chips, refresh behavior, or zero-weight semantics. Ratchet
`sase-core-revision.txt` and the published dependency floor only when the new core
surface is available according to the repository's cohort policy, then rebuild the
installed extension.

Teach `_fleet_agents_rows._agent_from_summary` to apply the fleet summary's canonical
capacity and explicitness through the existing `Agent.set_queue_capacity` path. Keep the
renderer and detail panel shared with local rows. Remote capacity is display and
metadata parity only: a remote row must not be counted as a local occupied runner or
change local launch-admission decisions.

Add contract tests that begin with an owner-side serialized summary rather than a
hand-built Python Agent. Prove positive and explicit-zero capacity survive owner,
gateway/client decoding, remote row synthesis, family/clan grouping, refresh/cache
replacement, and detail rendering; prove absence stays quiet; prove legacy-only owner
metadata normalizes to canonical output; and prove a remote `c100` row does not alter
the local global `C/L` denominator. Exercise the existing real row renderer so the gold
badge and authored Capacity detail cannot drift independently. Run focused core binding,
fleet contract, Agents row/detail, grouping, cache, and capacity suites plus
`just check`, and record exact main/core SHAs.

## Phase `integrated-acceptance`: Complete drain, remote, and full landing acceptance

Review main and core commits since this child plan started, especially active remote
fleet and gate-capacity landings, and integrate every queue-capacity intersection. Use
only authorized SASE launch workflows and isolate the acceptance cohort so unrelated
live agents are neither stopped nor counted as controlled test members. Preserve and
restore any runner-limit override in a guaranteed cleanup path. Demonstrate an authored
default `capacity=1` launch first parks with canonical waiting metadata and then admits
after the controlled occupying claim drains. Recheck the over-limit `capacity=100` case,
exact fractional weight, explicit zero, flag-off compatibility, invalid mixed authoring,
and a real serial continuation. Inspect real metadata, waiting-marker removal,
scanner/index projection, and ACE `C/L`, `c1`, `c100`, and Capacity detail. Stop only
agents launched by this smoke.

Also prove the new remote boundary with an owner-produced fleet payload or authorized
live remote row: canonical `capacity=100` must remain visible after transport, refresh,
family/clan synthesis, and detail selection, while local admission totals remain
unchanged. Inspect the capacity-specific visual actual/expected/diff artifacts and do
not accept unrelated golden drift.

Update `tests/reproducible_flake_baseline.txt` using the filed owners from the landing
audit and these exact current node IDs:

- `tests/ace/tui/actions/test_agent_search_history_split.py::test_async_bounded_agents_search_load_rejects_stale_query`
- `tests/ace/tui/test_agents_fleet_refresh_laziness.py::test_fleet_catalog_refresh_requests_legal_pages_and_logical_keys`
- `tests/ace/tui/test_machines_pane.py::test_status_check_is_user_triggered_and_records_observation`
- `tests/dispatch/test_machine_bootstrap_real_gateway.py::test_bootstrap_issue_enroll_hello_round_trip_through_real_gateway`
- `tests/fakey/test_provider_drain_e2e.py::test_provider_drain_e2e_flag_on_relaunches_stranded_agent`
- `tests/fakey/test_runner_slots_e2e.py::test_installed_research_swarm_quarter_weights_fill_one_fakey_capacity_unit`
- `tests/llm_provider/test_codex_usage_probe.py::test_registered_hook_runs_through_isolated_probe`
- `tests/main/test_artifact_cli_link_health.py::test_inspect_fix_repairs_historical_research_rename`
- `tests/monitor/test_monitor_proc_settlement.py::test_settle_monitor_artifacts_leaves_stopped_at_unpersisted`
- `tests/monitor/test_monitor_resume.py::test_checkpoint_resume_preserves_concurrent_acknowledgment`
- all three current collectable nodes in
  `tests/monitor/test_monitor_supervise_timeout.py`
- both current collectable nodes in
  `tests/test_agent_artifact_directory_operation_audit.py`
- `tests/test_agent_loader_incomplete_history_dedup.py::test_incomplete_load_after_complete_history_keeps_non_workflow_suffix_guard`
- `tests/test_config_schema.py::test_default_config_matches_public_schema`
- `tests/test_fleet_contract_counts_sase_core_rs.py::test_count_contract_deduplicates_current_instances_and_buckets`
- `tests/test_scratch_tmpdir_leak_regression.py::test_prepare_pytest_tmpdir_leak_does_not_break_a_later_scratch_read`

Use raw filed-debt entries for unresolved flakes and a `# fixed-at:` entry only where
commit ancestry proves every eligible incomplete-history failure predates the
`ef254fd6dc` repair. Do not invent a current-date fix, suppress collection, loosen the
gate, or reopen closed deterministic work without a reproduced post-fix defect. Rerun
the selection-health gate and preserve every routing outcome on the phase note.

Run focused queue, LaunchApproval, continuation, fleet wire/gateway, remote row,
scanner/index, completion, and visual suites. Run `just check`, then `just check-full`
through `/sase_monitor` with `TESTING` and `TESTED` after rebuilding the pinned
extension. Investigate failures causally and route only genuinely unrelated new work
through `/sase_new_task`. Record exact test counts, artifacts, current main/core SHAs,
live cleanup, and remaining limitations for the resumed `sase-zt.6.5` land agent.
