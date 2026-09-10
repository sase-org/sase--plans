---
tier: epic
title: Finish weighted-capacity acceptance
goal: Weighted capacity uses authoritative durable lineage end to end, passes integrated
  lifecycle acceptance, and ships with verified published package floors.
parent_bead: sase-z4.6
phases:
- id: admission-authority
  title: Make Rust candidate lineage authoritative at admission
  size: medium
  depends_on: []
  description: 'admission-authority: connect the Rust runner-capacity candidate decision
    and durable claim lineage through scan metadata and the locked Python admission
    path, then integrate the later capacity-only scan mode without losing predecessor
    ownership.'
- id: integrated-acceptance
  title: Add the missing integrated weighted workload acceptance
  size: medium
  depends_on:
  - admission-authority
  description: 'integrated-acceptance: exercise weighted claims, parallel-lineage
    handoffs, monitor transfer, research swarm expansion, cleanup, and runtime/CLI/TUI
    parity through real lifecycle fixtures rather than isolated projections.'
- id: published-floors
  title: Prove actual released floors and retire the rollout flag
  size: medium
  depends_on:
  - admission-authority
  - integrated-acceptance
  description: 'published-floors: establish releases that contain the repaired core,
    host, and research contracts; verify exact published wheels in a clean environment;
    ratchet dependency floors and pins; and close the existing weighted_queue_capacity
    flag bead only after the release proof succeeds.'
proposed_by: bbugyi200.athena.sase-z4.6.land
create_time: 2026-09-10 13:23:36
status: wip
bead_id: sase-z4.6.5
---

- **PROMPT:** [prompts/202609/weighted_capacity_final_acceptance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/weighted_capacity_final_acceptance.md)
- **PARENT:** [202609/weighted_capacity_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/weighted_capacity_landing_repairs.md)
- **BEAD:** [sase-z4.6.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-z4/sase-z4.6.5.md)

# Finish weighted-capacity acceptance

This child epic contains only work still missing from `sase-z4.6`. Its `parent_bead`
relationship is the handoff back to that interrupted land agent. Do not close
`sase-z4.6` or `sase-z4`, run either plan bead's post-close Symvision pass, or mark
their linked plan files done from a child phase.

Read the governing contract and audit evidence first:

```sh
sase artifact read plan:202609/weighted_capacity_landing_repairs.md "Implement only the remaining acceptance work found by the sase-z4.6 land audit"
sase artifact read file:explicit:5a03b708fc9d4e94c358413c "Retain the original sase-z4 landing failures and proposal dispositions"
sase bead show sase-z4.6
sase bead show sase-z5
```

Open `sase-core` and `sase-research-artifacts` with `/sase_repo` before reading or
changing them, use only the paths returned by that skill, and read their `AGENTS.md`
files. The audit baseline is SASE `4f6eb2b17`, core `dc3d0a8`, and research plugin
`8f00896`; recheck later drift before each phase lands.

The prior phase commits are real but do not satisfy the combined contract:

- Core `8b672ab` added schema-v2 candidate decisions and predecessor-based lineage, and
  core `8e491c3` added fleet weight fields. SASE `7da379ea2` never sends a distinct
  candidate to Rust or consumes `candidate_decision`; it merges the candidate into
  `records`, then reconstructs `project:family` and weight compatibility in Python. A
  serial successor of a live weight-2 parallel member therefore returns parked at limit
  2 instead of transferring the member's `parallel_member` claim. The production wrapper
  returns `candidate_decision: null` while the claim owner is
  `proj:fam:parallel:<parent timestamp>`.
- Core/main `161206b` / `ae07c41` later added `capacity_only`, but the runner-slot scan
  does not opt into it. Enabling it naively drops done predecessor directories, so the
  new fast path must be integrated with durable lineage and legacy migration rather than
  merely toggled on.
- Core tag `v0.33.0` points to `b53d15e`, before `8b672ab`, `8e491c3`, and `161206b`. No
  core release tag contains those repairs. SASE has no `v0.17.2` tag; its current
  `0.17.2` release branch split before `4f6eb2b17`, so it does not carry the declared
  core floor. The research `0.3.0` release branch split before `8f00896`, so it does not
  contain its new floors or published-minimum smoke. Treat all three as unreleased until
  tags and clean package installs prove otherwise.
- `tests/fakey/test_runner_slots_e2e.py` still drives only default weight 1.0. Its
  monitor case explicitly simulates a monitor record instead of exercising the real
  monitor handoff. The required four-quarter-weight fill, weight-2 land/monitor
  transfer, research expansion dependency graph, and same-snapshot runtime/CLI/TUI
  comparison are absent.

## admission-authority

Keep ownership, compatibility, effective weight, and admission decisions in Rust. Python
owns the host lock, liveness probes, marker I/O, and process start callback only.

1. Extend the capacity adapter so `runner_capacity_snapshot` sends the candidate in the
   Rust request's `candidate` field instead of overlaying it into the ordinary record
   list. Consume `candidate_decision.decision`, `owner_key`, `effective_weight`,
   `explicit_weight_compatibility`, and blockers directly under `runner_slots.lock`.
   Remove `_serial_family_owner_key`, `_active_serial_claim`, and the duplicate Python
   weight-compatibility policy once no caller needs them. A missing, malformed, or
   unknown decision must fail closed with an actionable error.
2. Persist the returned lineage owner on successful acquisition or transfer before the
   claim callback exposes work. Add `runner_claim_owner_key` to `agent_meta.json`, Rust
   agent-scan/index projections, Python wire records/conversion, capacity projection,
   and required schema/version fixtures. Propagate it through retry, repeat, monitor,
   gate, question, plan, pipe, and ordinary family follow-up creation. Independently
   launched parallel members must receive distinct owners; serial descendants must keep
   their actual predecessor lineage. Define a deliberate migration for legacy records
   with no owner key using existing predecessor/family facts, then publish the resolved
   owner at the next locked admission boundary.
3. Integrate `capacity_only=True` into the production runner-slot scan after proving it
   retains every fact needed for active claims, invalid-weight failure, candidate
   migration, and lineage transfer. Do not reintroduce terminal marker parsing merely
   for display data. If released-predecessor inheritance still needs a terminal fact,
   preserve that fact durably on the successor or adjust the scan contract explicitly;
   do not silently collapse a parallel lineage to its display family.
4. Add Rust, binding, scan/index, and Python admission regressions for a serial
   successor of a live parallel member, unrelated same-family branches, nested
   parallel/monitor/gate successors, pre-stamped candidates, released-lineage explicit
   reweighting, invalid legacy metadata, and candidate-decision wire failures. Assert
   the claim callback sees the effective weight and owner already persisted.

Run focused Rust and PyO3 tests, focused SASE runner/scan tests, core `just check`, and
SASE `just check`. Advance `sase-core-revision.txt` through the supported tooling when
the host consumes a newer core commit; do not edit release-plz-owned crate versions.

## integrated-acceptance

Build the missing combined acceptance on the authoritative path from the first phase.

1. Upgrade the real fakey lifecycle harness to author queue weight and explicitness and
   add a concurrency case where four independent `0.25` claims exactly fill capacity 1,
   the next fitting/non-fitting contenders park in shared Rust order, cap changes are
   reread, and kill/crash cleanup leaves no phantom claim. Cover default and explicit
   `runners=0`, `priority=0`, dependency waits, and mixed weights.
2. Exercise a weight-2 land-style family through actual monitor creation, supervision,
   and `--next` handoff rather than hand-authoring a monitor record. Verify the starter,
   monitor, and serial successor retain one owner and one 2.0 claim; an independently
   weighted parallel member and its serial successor retain their own lineage; pending
   human gates hold zero; approved automatic and detached gate routes transfer or
   reacquire before command execution; cancellation, failed startup, timeout, and crash
   do not release another owner's claim.
3. Load the installed research plugin's `research_swarm` xprompt and prove all four
   `0.25` launch units, their dependency graph, waits, omitted/default conditions, and
   explicit zero runner/priority values through the real expansion and launch-planning
   path. Connect those units to the lifecycle harness sufficiently to prove they can
   fill one capacity unit without being rounded to headcount.
4. From one captured source snapshot, compare runtime admission, `sase agent list -j`,
   ACE capacity header, queue ranks/blockers/details, local and remote non-default
   badges, unknown usage, and filtering/folding behavior. Preserve remote ownership:
   remote weights are presentation metadata and never charge the local snapshot.

Run the focused fakey, monitor/gate, listing, fleet, TUI, and visual suites; inspect any
changed PNGs. Run research-plugin `just check` and `just test-wheel`, core `just check`
if its contracts changed, and SASE `just check`. Phase notes must name exact commands,
outcomes, and any remaining gap; do not substitute parser-only tests for lifecycle
acceptance.

## published-floors

Do not infer compatibility from branch names, source overrides, or the presence of a
binding that predates its repaired behavior.

1. Harden `tools/probe_core_floor` and its contract tests so the probe exercises the
   required runner-capacity schema/decision behavior, parallel-lineage transfer,
   capacity-only scan fields, and weighted fleet summary fields. A published wheel that
   only exposes the old binding names must fail the probe.
2. Through each repository's established release automation, establish actual releases
   containing the completed core, then SASE host, then research-plugin work. Never edit
   release-plz/release-please-owned versions manually or guess the next number. Verify
   ancestry with release tags and verify package indexes supply the exact versions. If
   publication is not yet available, keep this phase open with concrete tag/workflow
   evidence rather than weakening the checks or claiming a source checkout as proof.
3. After publication, ratchet SASE's core dependency and source revision to the first
   containing core release, then ratchet the research plugin to the first containing
   SASE/core releases. Replace every provisional `0.33.0` / `0.17.2` assertion with the
   actual first compatible versions and keep upper bounds coherent.
4. Run a genuinely clean, wheel-only minimum-version smoke with no editable checkout,
   source override, `maturin develop`, or local package path. Install the exact
   published SASE and core floors plus the built/published research wheel; exercise the
   repaired candidate decision, scan projection, weighted fleet summary, all four
   research swarm segments, presets, waits, runners/priority zeros, and dependency
   graph. Retain a clear negative install/behavior check for the last incompatible
   floors, and make publication depend on the positive smoke.
5. Re-read `sase-z5` and verify the registry and Off branch for
   `weighted_queue_capacity` remain absent after all drift. Close that existing flag
   bead normally with a note naming the completed lifecycle, presentation, package, and
   rollout proof. This resolves `sase-z4.6.4`'s sole `PROPOSED FOLLOW-UP`; do not create
   a duplicate task bead.

Run the floor probe against the installed published wheel, research `just check` and
`just test-wheel`, all changed-repository checks, and the combined SASE
`just check-full` only through `/sase_monitor` with `TESTING` / `TESTED`. Record exact
tags, package versions, smoke commands, and outcomes in the phase note so the child land
agent can revalidate them before handing control back to `sase-z4.6`.
