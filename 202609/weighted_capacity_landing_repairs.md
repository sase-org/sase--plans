---
tier: epic
title: Complete weighted capacity ownership, presentation, and release acceptance
goal: Repair the remaining sase-z4 acceptance failures and integrate weighted capacity
  with the newer fleet UI so its parent landing can resume.
parent_bead: sase-z4
phases:
- id: capacity-policy
  title: Correct numeric fitting and explicit claim lineage in Rust
  size: medium
  depends_on: []
  description: 'capacity-policy: fix decimal-boundary admission, model serial claim
    ownership through parallel predecessors, and expose authoritative admission and
    parked-order decisions to host consumers.'
- id: lifecycle-boundaries
  title: Make weighted continuation and shell admission atomic
  size: medium
  depends_on:
  - capacity-policy
  description: 'lifecycle-boundaries: exclude unadmitted successors from their own
    claims, preserve authored versus inherited weights, and acquire or transfer capacity
    before monitor and gate work starts.'
- id: capacity-presentation
  title: Correct queue projection and integrate weighted fleet rows
  size: medium
  depends_on:
  - lifecycle-boundaries
  description: 'capacity-presentation: project global local claims before display
    transformations, include serial waiters, consume shared parked ordering, carry
    remote weight metadata, and finish capacity labels and visual acceptance.'
- id: release-acceptance
  title: Prove packaged compatibility and integrated weighted workloads
  size: medium
  depends_on:
  - capacity-presentation
  description: 'release-acceptance: establish real published package floors, add wheel-only
    compatibility checks and end-to-end workload coverage, and resolve the existing
    retired rollout flag bead.'
proposed_by: bbugyi200.athena.sase-z4.land
create_time: 2026-09-10 08:15:36
status: wip
bead_id: sase-z4.6
---

- **PROMPT:** [prompts/202609/weighted_capacity_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/weighted_capacity_landing_repairs.md)
- **PARENT:** [202609/weighted_queue_capacity.md](weighted_queue_capacity.md)
- **BEAD:** [sase-z4.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-z4/sase-z4.6.md)

# Remaining weighted-capacity work

This child epic completes defects and omitted acceptance criteria found while landing
`sase-z4`. Its parent is a plan bead. The `parent_bead` relationship is the handoff back
to the interrupted landing. This plan does not repeat completed queue syntax, ordinary
prompt-edit plumbing, presets, or documentation work. Do not close sase-z4, run its
post-close whitelist pass, or change its linked plan's status as a child implementation
phase. The child land agent must follow the normal parent-plan revalidation and landing
procedure after this child lands.

Read the original contract through:

```sh
sase artifact read plan:202609/weighted_queue_capacity.md "Understand the remaining weighted-capacity acceptance contract"
sase bead show sase-z4
```

The review evidence is `file:explicit:5a03b708fc9d4e94c358413c`. Read it with
`sase artifact read` and inspect the parent landing note and any later notes before
implementation. It records both proposed follow-up dispositions. All five original phase
notes were reviewed. Their successful test reports do not cover the failures below.

## Repositories and baseline

Work in the current SASE checkout. Open `sase-core` and `sase-research-artifacts` using
`/sase_repo` and `sase repo open` before accessing them. Use only the returned paths,
read each AGENTS.md, and configure `SASE_CORE_DIR` and plugin test source overrides to
those opened checkouts. Never use another agent's or an inferred sibling checkout.

Review baseline: SASE `0afe85be4`, core `41bec95`, plugin `526604b`. Original feature
commits are core `63bb275` and SASE `7c31d9aba`, `71c3df748`, `81064c144`, `0afe85be4`.
The source pin already names `63bb275`, which contains weighted APIs. Advance it through
established compatibility tooling as repairs require; it is not currently missing
weighted support.

Post-start integration includes SASE `5b330b242` (fleet contracts), `296b21b88` (unified
list), `d598883d8` (machines pane), and `66773a2b3` (remote row actions and details).
Core also gained native PatchWire and usage indicator contracts. Preserve those
contracts and recheck intervening changes before landing.

Shared ownership, numerical accounting, eligibility, ordering, and wire validation
belong in Rust. Python owns process liveness, lock and marker I/O, and presentation
adapters. Do not replace core decisions with new Python policy. Do not change
release-plz-owned version numbers manually. No memory or generated instruction edits are
part of this plan.

## capacity-policy

Start in `crates/sase_core/src/runner_capacity.rs` and its PyO3 exports. Current direct
reproductions:

- 99 started claims of 0.1 at limit 10 yield occupied 9.9. A 0.1 waiter is incorrectly
  blocked because subtracting free capacity gives 0.09999999999999964 and the tolerance
  is evaluated at that smaller scale.
- A started parallel member of weight 2 and its started serial successor of weight 2,
  with the same family and an explicit predecessor timestamp, yield two claims totaling
  4.0. They must share the predecessor's 2.0 claim.
- An unadmitted successor of a parallel member disappears from waiters if an unrelated
  serial branch with the same display family holds capacity.

1. Compare the deterministically summed proposed total with the effective limit.
   Preserve the original at-most-four-ULP bound at the compared total/limit, positive
   finite validation, and fail-closed overflow. Do not use a fixed epsilon, a rounded
   headcount, or weight-versus-subtracted-free as the final fitting decision. Add
   repeated-0.1, heterogeneous, meaningful over/under boundary, tiny positive, and
   overflow tests that exercise waiter admission.
2. Represent claim lineage independently of display family and the current record's
   parallel boolean. Resolve serial successors through their actual predecessor lineage,
   including nested parallel members and monitor/gate successors. Independently admitted
   parallel members each own a claim; serial overlap within one lineage projects one
   conservative maximum weight. Use durable explicit ownership metadata if necessary,
   with a deliberate legacy-record migration rule. Do not collapse unrelated branches.
3. Expose a candidate decision that distinguishes acquiring new capacity,
   transferring/reusing a real existing lineage claim, and invalid or blocked requests.
   Exclude the unadmitted candidate from evidence for its own claim. Return the
   effective inherited weight and explicit-weight compatibility outcome. The host should
   not reconstruct owner keys or implement a second family compatibility policy. Active
   claims cannot be reweighted; released lineages can reacquire at a new explicitly
   authored weight.
4. Put the required display ordering in the shared projection: eligible queue entries by
   priority/FIFO, then parked entries by capacity shortfall, runner count shortfall,
   priority/FIFO. Retain bounded deference and distinguish queue-order from resource
   blockers. Keep count fields integer and account for unavailable/invalid snapshots
   explicitly.
5. Update wire versions, scan/index projections, bindings, and schema fixtures
   deliberately for any new ownership or decision fields. Old records with absent weight
   remain 1.0; invalid present values stay distinguishable and fail closed. No stale
   binding may silently downgrade the new contracts.

Verify Rust unit and binding parity tests and core `just check` (including PyO3). The
core repository requires Python >=3.12; use its script's interpreter selection and
resolve library-path requirements rather than skipping bindings.

## lifecycle-boundaries

Connect the repaired Rust decision through `src/sase/core/runner_slots/_admission.py`
and `src/sase/axe/run_agent_wait_slots.py`. Audit actual helper-created records, not
only hand-authored unstamped fixtures.

1. Repair the confirmed capacity bypass: `create_followup_artifacts` stamps
   `run_started_at` before admission, the candidate override leaves it intact, and
   `_active_serial_claim` accepts that same candidate as a live family claim. With an
   unrelated weight-2 holder filling limit 2, a pre-stamped weight-2 successor currently
   returns ADMITTED when this record shape is passed directly to the adapter. Trace
   bootstrap as part of the regression: normal metadata rebuilding may remove the early
   stamp before this call, but the helper publishes it earlier and the adapter must not
   trust a candidate as its own existing claim. Initial artifact creation must not grant
   ownership; publish admission/transfer evidence under `runner_slots.lock`, excluding
   the candidate's unadmitted record. Preserve legitimate same-lineage handoffs without
   introducing a liveness gap.
2. Separate authored input from inherited effective weight. Currently `build_agent_meta`
   overlays preserved metadata after authored weight, and `create_followup_artifacts`
   copies the parent's explicitness. Preserve a child's own explicit value so lock-time
   conflict detection sees it, inherit weight only when omitted, and permit a new
   explicit weight once the lineage has released. Cover direct family attachment as well
   as retry, repeat, monitor, plan, question, pipe, and gate continuation routes.
   Independent children retain their own default or authored weight.
3. Publish the resolved effective weight, ownership, and started state atomically before
   work becomes visible. Revalidate against the current live snapshot under the lock. Do
   not accept an inherited value only in a local variable while the claim callback
   persists an older value.
4. Reject invalid present metadata and wait-marker weights. Current marker and
   preservation helpers fall back to defaults, and candidate overrides can clear scan
   invalidity. Add absent, null, bool, negative, nonfinite and corrupt-record boundary
   cases; preserve legacy absence separately. Report actionable failures and prevent
   invalid new work from running.
5. Trace monitor creation/supervision/handoff and all approved gate execution routes,
   including detached and automatic answers. Copying queue_weight is insufficient.
   `notification_gates/cli_answer.py` calls the executor without capacity acquisition,
   and `gate_shell/log.py` records PID after subprocess start. Before participating
   command execution, transfer a still-live claim or acquire it through the same locked
   policy. Pending human gates hold zero. Ensure cancellation, failed startup, timeout,
   crash, and partial transfers leave no phantom claim and do not release another
   lineage's claim. Keep nonparticipating procs and workflow script steps outside this
   budget.

Add real lifecycle regression tests for all confirmed cases, mixed concurrent weights,
cap changes, explicit runner-count conditions, and invalid claims. Exercise
parent-to-monitor-to-successor with weight 2, pending gate release and reacquisition
while full, and a serial continuation of an independently weighted parallel member. Use
isolated temporary state and controlled liveness plus existing fakey integration
infrastructure. Run focused tests and SASE `just check`; run core checks if the adapter
work requires a core amendment.

## capacity-presentation

Read `tui_perf.md` through `/sase_memory_read` before changes.

1. Correct `_participates_in_runner_slots` and related projection so a queued serial
   family child remains QUEUED with rank, blockers, and cancellation support after its
   old claim releases. A live WAITING child with parent_timestamp and slot_requested_at
   currently projects zero waiters.
2. Use authoritative local scan claims for global usage before display family
   aggregation, filtering, or folding. The current path calls
   `refresh_runner_slot_context(prep.filtered_agents)` after display preparation. Carry
   the shared snapshot through the background load and delta merge; never recompute
   ownership from mirrored display statuses. Preserve global usage across
   hidden/search/tribe/project/fold changes, serial overlap and pending gates. Keep all
   render paths free of disk/config/subprocess work.
3. Remove duplicated threshold-based parked ordering from TUI and CLI adapters. Consume
   the core order and blocker data. At full limit 1, weight 0.5 must appear ahead of
   weight 2 among parked entries even when the latter has better priority. Preserve
   visible status-count scope separately from global capacity and make unknown usage
   explicit.
4. Integrate the newer fleet contracts. Rust `ResolvedAgentSummaryWire` and Python
   `_fleet_agents_rows.py` omit weight. Carry validated workload weight and required
   provenance from the owning machine through summary/detail projections and row cache
   invalidation, so remote non-default workloads have the same badge/detail explanation.
   Remote capacity belongs to its owner; adding, following, filtering, or refreshing
   remote rows must not charge the local snapshot. Keep existing integer fleet count
   semantics and capability/remote-action behavior. Handle legacy remote payloads
   deliberately.
5. Finish Launch Control and associated edit/help labels. In
   `models_panel_runner_limit_cards.py` the title still says Max Running Agents and
   `_format_agents` formats the capacity budget as agent counts; the edit modal and
   validation messages also retain count wording. Explain capacity units while
   preserving max_running_agents, integer editing, overrides, and keybindings. Integrate
   the new machines pane's local-limit label as needed.
6. Add the original missing normal/narrow PNG scenes and inspect their actual outputs.
   Cover the four specified header examples; implicit/explicit 1.0 with no badge; w2 and
   w0.25 on standalone, terminal, queued and family nodes; family expansion; no
   duplicated gate/proc badges; local/remote rows; unknown usage and pressure styling;
   truncation/readability. Add cache and same-snapshot runtime/CLI/TUI parity tests
   rather than decorating isolated rows without verifying their source data.

Run focused TUI, listing, completion and federation tests, applicable visual tests with
image inspection, and SASE `just check`. Verify refresh behavior according to the
performance note. Core wire changes require core `just check`.

## release-acceptance

Both proposed follow-ups from `sase-z4.5` are original epic scope. Do not create new
task beads for them:

- Note 1 asks for weighted package floors. The original plan explicitly requires
  compatible published versions and a minimum-version installed-wheel smoke, so this
  cannot be deferred as unrelated work.
- Note 2 asks to close retired flag `sase-z5`. That existing flag bead is the correct
  bookkeeping target; do not duplicate it.

1. Use `tools/probe_core_floor` and the existing release tooling to determine actual
   containing published versions. Current SASE and plugin floors allow core 0.32.61; the
   plugin allows SASE 0.17.1. At review, the opened core had no release tag containing
   weighted commit 63bb275. Establish and verify the actual released compatibility
   floors for core, SASE and the research template. Preserve newer APIs from intervening
   epics. Update manifests, locks, source pin, and existing release verification through
   their supported mechanisms. Never invent a version or manually edit release-plz-owned
   versions. If publication is still unavailable, keep this requirement open with
   concrete evidence; source overrides are not proof of package readiness.
2. Add a genuine clean wheel-only minimum-version smoke. The plugin's current
   `test_wheel_contract.py` overrides SASE with an editable checkout and builds core
   using maturin develop. Retain that useful source integration lane, but independently
   install published minimum SASE/core wheels plus the built plugin wheel without
   editable/source/dependency overrides. Parse and expand all four swarm segments and
   prove the installed APIs and presets work. Verify clear failure for incompatible
   installations.
3. Complete integrated fakey acceptance: four quarter-weight independent claims fill
   limit 1, lander expansion requests 2 and retains it through a real monitor handoff,
   and research expansion produces four quarter-weight launch units with its dependency
   graph. Exercise default and explicit runners=0, priority=0, waits, crash/kill cleanup
   and mixed contenders. Compare runtime, CLI JSON, capacity header and queue details
   from the same snapshot. Keep all fixtures isolated from live user workloads.
4. Verify the Off branch and registry entry for weighted_queue_capacity remain removed,
   read sase_flags.md and sase-z5, and close existing flag bead sase-z5 with a note
   describing completed removal and acceptance. This is flag cleanup, not a child phase
   for closing the parent epic. Record both proposal outcomes in this phase note for the
   eventual parent close note.
5. Verify rollout instructions address draining/replacing old scheduler processes.
   Reading legacy records as weight 1 does not make mixed old/new admission binaries
   safe. Document the operational transition; do not stop unrelated live agents to
   satisfy a test.

Run research-plugin `just check` and `just test-wheel`, the independent minimum-version
wheel smoke, and required checks in every changed repository. Use SASE's monitor skill
for long commands.

## Completion evidence and handoff

Each phase records exact commands, outcomes, changed contracts, and any remaining issue.
Original-epic failures stay in this child; no unrelated task should replace them. Phase
workers record genuinely unrelated discoveries as PROPOSED FOLLOW-UP notes for their
land agent.

The child land agent must verify all descendants and new drift, run the combined SASE
`just check-full` exclusively via `/sase_monitor` with TESTING/TESTED, and recheck the
required core/plugin gates. Do not claim completion from parser tests alone. The parent
landing review contains confirmed failures and release acceptance gaps that each need
explicit resolution evidence.

The parent link lets the land agent then re-review sase-z4, its original phase notes,
this child's descendants, linked plan readiness and post-child drift. The original
parent close note must account for both sase-z4.5 proposals and the repaired failures.
Follow the user's normal whitelist/close/status procedure only when that parent is
actually complete; never force a successful nested landing.
