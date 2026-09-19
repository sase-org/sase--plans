---
tier: epic
title: Complete remote Agents parity from real owner state
goal: The production owner roster and fleet feed produce the same visible Agents nodes,
  family statuses, shells, tribes, and runtimes, with truthful version diagnostics
  and matching live apollo evidence.
parent_bead: sase-133
phases:
- id: owner-roster
  title: Share the real owner roster and retain visible family shells
  size: large
  depends_on: []
  description: 'owner-roster: reconcile gateway selection with the actual local loader
    and modern family-shell lifecycle using one production fixture oracle.'
- id: owner-facts
  title: Resolve production family presentation facts
  size: large
  depends_on:
  - owner-roster
  description: 'owner-facts: derive topology, rich statuses, tribe inheritance, shell
    facts, and runtime anchors from real owner records and consume them through the
    existing viewer renderer.'
- id: version-diagnostics
  title: Distinguish capability and fleet data versions
  size: medium
  depends_on: []
  description: 'version-diagnostics: replace the false capability-versus-fleet comparison
    with explicit fleet contract evidence and preserve genuine version-skew diagnostics.'
- id: parity-acceptance
  title: Prove production and live cross-machine parity
  size: medium
  depends_on:
  - owner-facts
  - version-diagnostics
  description: 'parity-acceptance: exercise the production path, validate ordered
    screenshot driving, deploy consistent builds, and save reviewed parity evidence
    from both hosts.'
proposed_by: bbugyi200.athena.sase-133.land
create_time: 2026-09-19 08:06:06
status: wip
bead_id: sase-133.5
---

- **PROMPT:** [prompts/202609/remote_parity_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/remote_parity_landing_repairs.md)
- **BEAD:** [sase-133.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-133/sase-133.5.md)

# Complete Remote Agents Parity From Real Owner State

This is only the remaining work found while landing `sase-133`. The original plan is
`plan:202609/remote_dispatch_agents_tab_parity.md`. All four original phases are closed,
but the production path still fails its acceptance criteria. This child must remain
parented to `sase-133`; its normal landing is the handoff for resuming the interrupted
parent landing. Do not add parent closing, a parent symvision pass, or a parent plan
status update as an implementation phase.

## Evidence and constraints

The landing audit reviewed primary commits `fa61906da`, `67614ee2b`, `2ec00fe68` and
core commits `b7531db`, `11b8e06`, against primary `8989d0a72` and core `39602c9`. The
original phase notes and the new parent landing note document the full audit and
proposal dispositions. Read those through `sase bead show` before implementing.

The authenticated apollo catalog is already schema v4. It returns `0n--plan` and
`0k--plan` as unparented `DONE` gate rows; apollo itself renders the families as
`TALE DONE ×7` and `EPIC CREATED √ ×7`. The settled owner screenshot reports 20 agents
with panels `@default=8`, `@epic=6`, and `@research=6`. The near-contemporaneous viewer
capture filtered to `machine:apollo` reports 30 agents, including one running and four
waiting, and panel counts 1/23/6. These surfaces count different objects today; do not
equate raw catalog rows, visible nodes, and shell counts when building the oracle.

Failure evidence: live catalog `file:explicit:d1d475c9a14db30b47241095`, settled owner
capture `file:explicit:dfc574b713144c3e3c3636ec`, and viewer capture
`file:explicit:6afea0b63a45b9273428ffc2`. Use audited `sase artifact read` to consume
those artifacts. Both capture installs include the original epic; the final acceptance
must additionally use consistent builds and record their versions. These captures are
not final signoff.

Confirmed source gaps:

- `sase_core::fleet_presentation` drops dead family members from presentation scope. The
  viewer `_fleet_refresh.py` requests only presentation scope; pagination never changes
  that scope into history. Missing shells cannot contribute local-equivalent nesting,
  counts, chips, or family status.
- Gateway `PresentationContext` only considers served artifact records. It assumes roots
  and tracked parent timestamps that modern owner family records do not always supply in
  that shape. A real root and its concrete shell are different objects.
- Gateway `display_status_for_record()` derives coarse statuses and generic done labels;
  it does not implement the owner's actual plan/monitor/gate/family pipeline.
- Phase-1's named parity test checks a hand-written expected gateway set, without
  calling the local loader. The Python display-parity fixture supplies the missing
  statuses and full tree by hand. These remain useful unit tests but are not a
  production parity oracle.
- `machine_handler._version_skew()` compares hello capability schema v1 with fleet data
  schema v4. A matching local/remote core version still produces a false warning.

Open `sase-core` through `/sase_repo`; never assume a sibling checkout path. Shared
selection and presentation decisions belong in `crates/sase_core`, with thin gateway
observations and Python bindings/adapters. Keep Textual rendering and layout in Python.
Preserve dismissal, recycled-PID protection, resource capabilities, content identity,
and older-host tolerant readers. Pending gate shells can legitimately outlive their
creator PID; do not infer that every dead-PID shell is obsolete.

Integrate the post-start Agents-list projection (`13a8efbb4` / core `8e1b8b6`), bounded
history reconciliation, startup scheduling, metadata-only detail default, and current
hold/wait lifecycle. Keep shared state decisions out of the event loop and preserve
incremental refresh. Retain the remote login-shell helper from `d4c028772`.

## Phase owner-roster

1. Build a fixture from the real current persisted lifecycle shape: project claims,
   running/waiting/question markers, done records, dismissal state, family roots and
   concrete plan/code/monitor/gate/proc shells. Include a pending gate with a dead
   creator PID, a settled monitor, an active coder, a completed family with history, an
   old dismissed identity, a recycled PID, and a newly launched shell.
2. Run that fixture through the real owner Agents loader and the real gateway catalog
   builder. Define separate normalized signatures for visible nodes, nested shells, and
   grouping. Do not replace either side with a hand-authored expected roster. Trace the
   observed live extra/missing identities to the rule that admits/removes them,
   including cache invalidation and newly launched rows.
3. Put the shared inclusion/retirement decision in Rust core and invoke it from both
   paths, or persist a needed lifecycle fact once where the original approved plan
   permits it. Eliminate duplicated incompatible policy. Keep host observations
   injectable and retain unknown-liveness safety. Preserve valid pending shells.
4. Publish enough bounded family context and visible historical shell data for parity.
   Do not fetch the full archive on every refresh or expose hidden/dismissed history
   merely to obtain counts. Preserve explicit history pagination as a separate API. Keep
   timestamps/project identities collision-safe and deduplicate real/synthetic roots
   without confusing roots with concrete shells.

Acceptance: the real loader and real presentation catalog produce equal normalized
visible-node and shell sets for the fixture, with current and compact-index projection
paths tested. Stale rows are absent, pending shells remain visible, fresh launches
appear, and counts are not inflated by duplicate family containers.

## Phase owner-facts

1. Starting from the corrected roster, resolve owner topology and presentation facts in
   shared core: stable family/node/shell identities and linkage, role, direct and
   inherited tribe/clan tribe, rich status with semantic bucket, shell kind/state and
   chip facts, and family/run/completion timestamps.
2. Reuse or extract the current owner's status rules rather than copying a list of
   desired labels into the gateway. Cover active and settled monitor/gate/proc shells,
   approved tale coding, completed tales, approved epics, question/wait details, and
   stopped/failed outcomes. Verify that real plan and family metadata is sufficient;
   persist bounded facts through lifecycle/index mutation hooks only when needed.
3. Extend/version the wire and bindings additively where required, normalize revisions
   and cache invalidation when presentation facts change, and carry the data through the
   Python adapters. Current hosts should supply complete facts; legacy omissions retain
   a deliberate degraded rendering without exceptions.
4. Use the existing shared viewer renderer and grouping path. Preserve concrete
   action/content identity when a family container summarizes a shell. Do not treat
   terminal process liveness as transport-offline health. Compare actual production
   payloads, not fixture-only fields, for rich status, `×N`, chips, runtime columns,
   done-inclusive banners, and machine-filter counts.

Acceptance: the production fixture's owner and remote normalized rendered rows match
apart from the machine chip and explicitly preserved remote affordances. Regression
cases reproduce and fix the observed `0n`/`0k` family failures and active/waiting
families. No viewer filesystem access is needed to reconstruct remote facts.

## Phase version-diagnostics

1. Separate protocol, hello envelope, capability-set, fleet-data, and package versions.
   Advertise an explicit optional fleet-contract version through hello or consume an
   existing authoritative fleet-data version; do not infer it from capabilities.
2. Update Rust wire/route tests and the Python `MachineStatus` adapter/CLI comparison.
   Keep older hello payloads readable and represent absent fleet version as unknown.
   Preserve genuine package/schema mismatch messages and actionable upgrade guidance.
3. Replace tests that expect the current false warning with same-build/no-warning,
   independently versioned capability/data, old-host/unknown, and real mismatch cases.

Acceptance: core 0.34.63 versus core 0.34.63 with capability v1 and fleet v4 produces no
fleet-schema warning; an actual incompatible data version still does. Use the current
schema version if earlier repair phases advance it.

## Phase parity-acceptance

1. Extend the production fixture oracle through serialization, federation, viewer
   projection, grouping, folding, filtering, and rendering. Assert visible-node count,
   panel count, nested-shell count, status, chips, and timestamps independently. Add
   fixed-time visual coverage derived from the production fixture and retain the
   older-host compatibility cases. Avoid a second synthetic oracle that bypasses the
   owner derivation again.
2. Prove local and `--host apollo` ordered screenshot driving from a fresh TUI. Use a
   meaningful wait after opening the query input and before submitting typed text; the
   audit succeeded with literal `/`, `-w INSERT`, `--type machine:apollo`, a wait for
   that text, then `Enter`. Check the published `slash` spelling against tmux and
   correct CLI/docs examples if it does not send `/`. Handle startup notification
   overlays explicitly; never submit or resolve an unrelated gate to obtain a capture.
3. Run the required focused tests and core canonical checks. Follow `lint_and_test.md`:
   `just fix` before monitored `just check-full`, then inspect the complete retained
   visual report and every changed golden. Include generated golden updates through the
   host finalizer, with required attribution for unrelated ones.
4. Deploy the completed changes through the normal installed-update flow on athena and
   apollo, restart the affected gateway/AXE services, and record exact installed
   versions and truthful `sase machine status apollo` output. Reuse the original
   approved deployment scope and `docs/remote_dispatch.md`; do not silently replace a
   newer editable development install with an older published release.
5. Capture both settled views close together using the new text-driving command. Record
   grouping/filter configuration and freshness so the comparison is meaningful. Compare
   identities as well as counts, panels, one active/waiting family, and one completed
   family. Use controlled short-lived fixture activity if there is no live running
   family, and clean it up. Fix discovered parity failures within this scope.
6. Inspect both PNGs, save them as labeled SASE artifacts, and record their refs,
   production-oracle results, version evidence, and all remaining limitations on this
   phase. Failure screenshots alone do not satisfy acceptance.

Acceptance: equal settled owner/viewer identities and counts, matching representative
rich family rendering, no false skew, passing production parity tests, and reviewed live
evidence. This phase does not close the parent epic or mark its plan done.

## Landing handoff

The child lander must check descendant readiness and post-child drift, then resume
`sase-133` through its `parent_bead` link as the original land prompt directs. The
parent audit records an already-resolved unused-symbol proposal and an outstanding
screenshot-reference-memory proposal. Settle that proposal under `/sase_new_task` and
the memory authorization policy before the parent closes; this plan does not authorize
memory edits. Do not file the parity defects as unrelated follow-up tasks.
