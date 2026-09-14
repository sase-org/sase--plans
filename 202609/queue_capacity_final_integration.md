---
tier: epic
title: Finish queue-capacity landing integration
goal: The current pinned core and every approved launch path preserve and describe
  authored queue-capacity budgets, and the combined implementation passes the missing
  live and full acceptance evidence.
parent_bead: sase-zt.6
phases:
- id: core-completion
  title: Finish flag-aware queue name completion in the current Rust core
  size: small
  depends_on: []
  description: 'core-completion: select flag-aware queue metadata for directive-name
    completion and cover enabled/disabled documentation through Rust and the LSP.'
- id: pin-and-launch
  title: Pin the integrated core and prove LaunchApproval preserves capacity
  size: medium
  depends_on:
  - core-completion
  description: 'pin-and-launch: ratchet the current core cohort and repair or disprove
    the reported agent-skill LaunchApproval capacity loss through typed request and
    dispatch boundaries.'
- id: acceptance
  title: Complete live and combined-tree capacity acceptance
  size: medium
  depends_on:
  - pin-and-launch
  description: 'acceptance: observe the authorized live capacity scenarios, inspect
    targeted visuals, and complete focused plus full integrated verification.'
proposed_by: bbugyi200.athena.sase-zt.6.land
create_time: 2026-09-13 14:29:52
status: done
bead_id: sase-zt.6.5
---

- **PROMPT:** [prompts/202609/queue_capacity_final_integration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/queue_capacity_final_integration.md)
- **PARENT:** [202609/queue_capacity_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/queue_capacity_landing_repairs.md)
- **BEAD:** [sase-zt.6.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zt/sase-zt.6.5.md)

# Finish queue-capacity landing integration

## Context

This plan contains only work still open after auditing epic `sase-zt.6`. Its four phases
are closed, but the source, commit, note, and post-start-drift review found three
remaining landing obligations:

1. `sase-zt.6.2` note 1 is still true in current sase-core. The
   `queue_capacity_budget`-aware metadata exists, but
   `build_directive_completion_candidates_with_flags` iterates the unflagged
   `DIRECTIVES` entry, so the LSP `%queue` name row still documents the disabled-state
   weighted-load threshold while argument and recipe rows describe the enabled-state
   capacity budget.
2. Main now expects agent-scan wire schema 9 and artifact-index schema 30, while
   `sase-core-revision.txt` remains at `17947a05`, before later core commits
   `1b122287`/`b79accb3` added full-history completeness and schema-30 machine
   provenance and before `23f19f0` added the continuation bindings consumed by main
   commit `a6f6ae5c`. A clean pinned rebuild therefore does not represent the current
   integrated Python tree.
3. The acceptance phase did not complete the approved live TUI smoke. Its retry created
   LaunchApproval gates, but the probes did not yield durable running rows; the phase
   instead combined the earlier `sase-zt.5` admission smoke with a new preconstructed
   PNG fixture. The current Rust typed planner and dispatch-prompt builder do preserve
   canonical capacity in source and unit tests, so the phase's claim that `sase run`
   drops `%queue` is not yet a proven product defect. The production LaunchApproval path
   lacks a regression that settles the question end to end.

Preserve the completed behavior: canonical-only writers with legacy/dual readers,
canonical-wins alias resolution, explicit-zero upgrade semantics, exact fractional
effective-weight drains, both sunset-flag branches, per-segment capacity flooring,
schema-30 full-history/source reconciliation, machine provenance filtering, continuation
exactly-once delivery and ancestry retention, authored row/detail badges, numeric
over-limit styling through `u32::MAX`, and the existing capacity visual golden.

Unrelated follow-ups are not phases here. The gateway seeded-row failure corroborated
ready task `sase-10a`; the broad visual drift corroborated ready task `sase-x5`; the
machines-pane full-lane/pass-isolation failure was recorded on active flake epic
`sase-j7`; the continuation binding landed in core `23f19f0` and released-package
acceptance remains with `sase-zl.13.11.6`; creator handoff remains `sase-106`.

## Phase `core-completion`: Finish flag-aware queue name completion in the current Rust core

Open `sase-core` through `sase repo open` and work on the latest current lineage, not
the old main pin. In `crates/sase_core/src/editor/directive.rs`, make directive-name
completion select `queue_directive_metadata(enabled_feature_flags)` before building the
`%queue` candidate, just as `directive_contract_with_flags` already does. The enabled
row must say that capacity is this launch's budget; the disabled row must retain the
weighted-load-threshold description. Keep aliases, hiding rules, ranking, insertion, and
non-queue metadata unchanged.

Add focused Rust coverage for name candidates in both flag states, not only direct
metadata lookup. Extend the LSP completion coverage so a real directive-name request
asserts the correct documentation in both states. Run the core repository's required
`just check`, land the core change, and record its full SHA on the phase bead for the
next phase.

## Phase `pin-and-launch`: Pin the integrated core and prove LaunchApproval preserves capacity

Advance `sase-core-revision.txt` to the core SHA from `core-completion`. That SHA must
contain the queue scan/editor work plus source reconciliation, schema-30 machine
provenance, `continuation_decide_resume_adoption`, and continuation retention. Update
only validation probes/constants genuinely required by that SHA; do not edit release
versions or published dependency windows. Run `just install` and prove the installed
extension and LSP were rebuilt from the pinned SHA.

Add a production-path regression beginning with an agent-surface launch request that
contains `%queue(capacity=100)` and the enabled typed-launch flag. Exercise request
creation, serialized `typed_plan`, approved dispatch-prompt reconstruction, ordinary
launcher directive extraction, and the resulting metadata/marker payload. Assert the
canonical capacity and explicit marker survive every boundary and that no legacy key is
written. The preview may still say `waits=none`: queue admission is not a dependency
wait, so that text alone is not evidence of loss. If the actual path loses capacity, fix
it at the responsible boundary rather than adding a second parser or journal. Also cover
the flag-off compatibility dispatch path and canonical dispatch output.

Re-run real scanner/index tests after the pin change, including canonical metadata after
waiting-marker removal, rebuild/cached/full-history refresh at schema 30, and machine
candidate filtering. Re-run continuation prefix/admission tests for canonical positive
capacity, dual aliases, explicit priority zero, fractional weight, and persisted legacy
zero. Coordinate any continuation-core overlap with active `sase-zl.13.11`; preserve its
current ownership and do not duplicate its protocol or retention implementation. Run
`just check` and record the verified main and core SHAs.

## Phase `acceptance`: Complete live and combined-tree capacity acceptance

Use only authorized SASE launch workflows. Preserve the current runner-limit override
and restore it in a guaranteed cleanup path. Under a temporary global limit of 1,
observe an ordinary occupied claim, an approved `%queue(capacity=100)` agent admitted
over that limit, and an approved default-weight `%queue(capacity=1)` agent parked until
the other claims drain and then admitted. Verify the real `agent_meta.json`, any
`waiting.json`, scanner/index projection, and ACE presentation: red global `C/L`, gold
`c100`, quiet `c1`, and authored `Capacity: 100 capacity units`. Stop only the smoke
agents. If agent-side gate handoff still hits `sase-106`, answer or continue through the
supported gate shell; do not bypass approval.

Recheck the two invalid authoring cases (`%q:0`, `%q(capacity=1, w=2)`), positive
capacity inheritance through a real serial continuation, and legacy-zero upgrade
behavior. Run the affected Agents and Models visual nodes, inspect actual/expected/diff
artifacts for the capacity-specific golden, and keep unrelated standing drift on
`sase-x5`. Review main and core commits that landed after this plan started for any new
queue-capacity intersections.

Run the focused capacity, scanner/index, LaunchApproval, continuation, completion, and
visual suites. Then run `just check-full` through `/sase_monitor` with `TESTING` and
`TESTED`, using the rebuilt pinned extension. Investigate failures causally. Route only
genuinely unrelated new issues through the established proposal/task workflow; do not
relax budgets, accept uninspected goldens, or suppress tests. Record exact evidence on
the phase bead for the resumed `sase-zt.6` land agent.
