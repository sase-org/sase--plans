---
tier: epic
title: Weighted agent capacity with clear queue and status presentation
goal:
  Support positive fractional weights on %queue/%q, enforce and display weighted
  capacity consistently across agent lifecycles, and adopt heavier epic landers and
  lighter research swarm members.
phases:
  - id: core-contracts
    title: Define weighted queue contracts and shared capacity policy in Rust
    depends_on: []
    size: medium
    description:
      "core-contracts: implement the validated weight value, queue and launch wire
      changes, deterministic capacity projection and admission policy, editor metadata,
      and PyO3 contracts."
  - id: launch-plumbing
    title: Preserve weight through prompt editing and durable launch metadata
    depends_on:
      - core-contracts
    size: medium
    description:
      "launch-plumbing: add temporary beta rollout scaffolding and preserve authored and
      effective weight through directive adapters, launch paths, metadata, scan
      adapters, family continuation inputs, and prompt rewrites."
  - id: admission-lifecycle
    title: Enforce weighted claims through admission, handoffs, and cleanup
    depends_on:
      - launch-plumbing
    size: medium
    description:
      "admission-lifecycle: connect Rust capacity policy to the locked runner gate and
      implement family claim transfer, released-family reacquisition, monitor and gate
      handling, and concurrent lifecycle regression coverage."
  - id: capacity-ux
    title: Separate capacity from counts and render quiet weight badges
    depends_on:
      - admission-lifecycle
    size: medium
    description:
      "capacity-ux: feed shared capacity into the TUI and CLI, render global usage
      before status brackets, add non-default agent weight badges and queue
      explanations, and verify refresh and visual behavior."
  - id: presets-rollout
    title: Adopt workload weights and complete the coordinated rollout
    depends_on:
      - capacity-ux
    size: medium
    description:
      "presets-rollout: apply 2.0 to bundled epic landers and 0.25 to all four research
      swarm segments, update documentation and package compatibility, exercise the
      integrated feature, and remove temporary rollout scaffolding."
proposed_by: bbugyi200.athena.0i5
bead_id: sase-z4
create_time: 2026-09-10 12:53:59
status: wip
---

- **PROMPT:**
  [prompts/202609/weighted_queue_capacity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/weighted_queue_capacity.md)
- **BEAD:**
  [sase-z4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-z4/README.md)

# Weighted queue capacity

## Outcome and scope

An agent normally claims 1.0 units of machine-wide capacity. `%q(w=2)` claims 2.0 units;
`%queue(weight=0.25)` claims one quarter. A capacity budget of 8 permits eight ordinary
agents, four weight-2 agents, or thirty-two quarter-weight agents, subject to their
existing dependency, time, priority, and explicitly authored runner conditions. This is
admission accounting, not an OS resource limiter or a provider token quota. Every
admitted agent still needs its own normal workspace.

This is an epic because the implementation crosses Rust core contracts, Python runtime
lifecycle coordination, cached TUI projections, and a separately packaged research
plugin. Each phase has a bounded implementation contract and its own tests. The
dependency chain deliberately settles ownership and wire contracts before their
consumers change. Plan approval authorizes implementation; this authoring turn changes
only this scratch plan and submits it for review.

The repositories are the current SASE checkout, `gh:sase-org/sase-core`, and the linked
`sase-research-artifacts` plugin. Each worker must use `/sase_repo` and
`sase repo open <ref> -r '<specific reason>'` before accessing another repository, then
use only the returned paths. Read each opened repository's AGENTS.md. Configure the
existing `SASE_CORE_DIR`/Justfile override with that opened Rust checkout for local
builds; do not use an unrelated sibling checkout. No memory-file or generated
instruction-file edits are part of this plan.

## Current implementation and why it matters

- Rust owns `%queue` parsing and canonical formatting in
  `crates/sase_core/src/queue_directive.rs`, typed launch units and dispatch prompt
  reconstruction in `agent_launch/{mod,admission}.rs`, editor metadata in
  `editor/{directive,wire}.rs`, and artifact scan wire/index projections in
  `agent_scan/{wire,scanner,index}.rs`. Bindings live in
  `crates/sase_core_py/src/lib.rs`.
- Python's `src/sase/xprompt/queue_directive.py` is already a thin Rust adapter.
  `_directive_types.py`, `_directive_extract.py`, and `_directive_edit_wait.py` carry or
  rewrite runner and priority fields. In particular, editing a wait currently rebuilds
  the entire queue directive: failing to preserve weight here would silently reset an
  agent's resource requirement.
- `src/sase/axe/run_agent_wait_slots.py` scans and claims under `runner_slots.lock`. It
  currently turns the implicit cap into the integer threshold `limit - 1` and allows an
  explicit `runners=` threshold to replace it. Policy in
  `src/sase/core/runner_slots/_admission.py` is still Python, including occupancy,
  family grouping, priority/FIFO ordering, eligibility, and deference.
- Serial families count once, while parallel members count separately. Monitor handoffs
  bridge parent/child liveness. Pending gates and legacy question markers release
  occupancy. Serial successors currently bypass the queue even after their family has
  released its slot; the existing troubleshooting guide documents the resulting
  possibility of exceeding the cap.
- `src/sase/ace/tui/models/agent_runner_slots.py` currently derives integer occupancy
  through display lanes. `widgets/agent_info_panel.py` colors a combined `R/L running`
  metric using the visible running count. Capacity must instead come from authoritative
  global claims, independently of filters and the status count.
- The lander prompt is `xprompts.bd/land_epic.content` in `src/sase/default_config.yml`.
  The authoritative swarm is `src/sase_research_artifacts/xprompts/research_swarm.md` in
  the plugin. It has four launch segments: researcher A, researcher B, lead, and image.
  Each currently authors `runners=16` by default.

## Public contract

### Syntax and numeric values

1. Add parenthesized keyword `weight=` and alias `w=` to both `%queue` and `%q`.
   Examples: `%q(w=2.0)`, `%q(w=2)`, `%queue(weight=0.25)`, and `%q(0, p=20, w=2)`. The
   sole positional/colon argument remains integer `runners`; `%q:2` continues to mean a
   runner-count condition, never weight. `%w` remains the alias for `%wait`.
2. Accept positive finite base-10 floating-point values, including integer shorthand,
   `.25`, `2.`, and scientific notation such as `2.5e-1`. A leading plus is valid for
   weight. Reject empty, zero (including negative zero), negative, boolean, nonnumeric,
   NaN, infinity, overflow, and values that underflow to zero. Use actionable
   diagnostics naming `weight`, its accepted range, and the source span. The existing
   integer-only runner and priority rules stay intact.
3. Omitted weight has effective value 1.0 on a new independent agent. Preserve omission
   separately from an explicitly authored 1.0 for reconstruction and family inheritance.
   Canonical queue wire field: optional `weight`; launch/agent metadata field:
   `queue_weight` with `queue_weight_explicit` provenance where needed. Persist the
   resolved numeric weight before a claim can become visible.
4. Canonical formatting orders present fields as `runners`, `priority`, `weight` and
   uses the long keyword. Preserve explicit defaults in edited/reconstructed prompts;
   avoid inserting a weight directive into untouched default prompts. Bare/empty `%q`
   remains invalid, while weight-only `%q(w=...)` is valid.
5. `w` and `weight` are one field. Reject duplicate assignments across aliases and
   occurrences even if numerically equal. Disjoint fields compose, including
   `%q(w=0.25) %q(p=20)`. Each expanded swarm/fan-out launch unit validates separately.
   Fenced literal text and disabled-xprompt regions keep their established behavior.
6. Weight is a launch-time resource requirement. No new keybinding or live reweighting
   control is introduced. Existing wait/priority edits preserve it. Restarting or
   retrying the same workload preserves it; an independent child workload defaults to
   1.0 unless its own prompt supplies a value.

### Capacity and runner-count conditions

Keep the existing positive-integer `max_running_agents` configuration and temporary
override interface, with the same key, numeric default, and override precedence. Its
meaning becomes the total capacity budget, displayed as a float. Fractional limits and a
second configuration key are outside this change. Update configuration descriptions and
Launch Control's label/help to say capacity, while preserving the configuration key and
editing shortcuts.

Let `C` be currently claimed capacity, `L` the effective limit, `W` the requested
weight, and `R` the number of occupied participating agent lanes. New claims require:

```text
C + W <= L
and, if runners=N was explicitly authored, R <= N
and all existing dependency/time/priority/deference conditions are satisfied
```

`runners=N` remains an integer count of other occupied agent lanes. It is an additional
condition; it cannot bypass the capacity budget. `%q(runners=0, w=2)` waits for a true
drain and enough capacity for its two units. After it starts, it does not impose an
exclusive fence on later agents. Do not derive an implicit integer runner threshold from
`L - W`, round capacity to a count, or impose a second implicit headcount cap.

Two deliberate behavior changes are included in this design: explicitly large `runners=`
values no longer override the global budget, and a serial successor whose family has no
live claim must reacquire capacity. Document both prominently in the queue
troubleshooting and release-facing documentation. Ordinary omitted/default weight
launches keep their previous one-unit behavior apart from those corrections.

Read the effective limit once per locked admission attempt and evaluate every queued
candidate against that same current limit. Temporary limit changes apply to explicit and
implicit runner conditions alike. Lowering the cap never preempts active work; display
actual over-cap usage and admit no additional capacity until it fits. An agent whose
weight exceeds the current limit remains queued with an explicit reason such as
`needs 2.0 capacity; limit is 1.0`; raising the limit can unblock it. Never clamp its
weight or silently let it run alone above the budget.

Keep current lower-priority-number/FIFO scheduling and bounded deference. Select the
first eligible waiter, skipping a heavier or count-blocked waiter when a later one fits.
Weight is not priority. Heavy agents can consequently starve under a continual stream of
fitting work, just as existing priority/threshold scheduling permits; reservation,
aging, and preemption are outside scope. Report queue order as present eligibility, not
an ETA.

Use a validated finite-positive f64 contract in Rust and deterministic compensated
summation of claim weights. Bound any fit-comparison allowance to at most four ULPs of
the compared total/limit to handle binary rounding at fractional boundaries; do not use
a fixed absolute epsilon or round weights to tenths/quarters. Test repeated 0.1 claims,
heterogeneous sums, values on both sides of a meaningful boundary, tiny positive values,
and aggregate overflow. Unrepresentable totals must fail closed.

### Claim ownership and family lifecycle

Capacity is reconstructed from durable claims and liveness using the established
participating-record rules. Keep OS liveness checks, locking, and marker I/O in the host
adapter; move the affected grouping, accounting, eligibility, and queue policy into Rust
so the runtime, CLI, and TUI consume the same answers.

- A standalone agent owns one claim of its effective weight. Each independently admitted
  parallel member owns its own claim, even if nested in a family or clan. A serial
  continuation of a parallel member inherits that member's claim lineage; it cannot
  borrow an unrelated branch's claim merely because the display family name matches.
  Clan/workflow display containers have no extra weight.
- A serial family owns one shared claim across its shells. Serial continuations inherit
  the family's effective weight when their prompt omits it. Overlapping live records
  during handoff count that claim once, using the maximum valid weight in the serial
  group as the conservative projection. They are never added together.
- A serial child can reuse a live family claim without another wait or charge. It must
  not change that active claim's weight: an explicitly different value fails with a
  clear message to use the existing family weight or start an independent agent. This
  prevents a waiting parent/child pair from deadlocking on an incremental upgrade.
  Revalidate this constraint under the claim lock, not just during prompt preview.
- After a family releases capacity, a successor inherits its last effective weight
  unless explicitly supplied, and joins normal admission. A different weight is allowed
  at that new claim boundary. Exclude the successor's own unadmitted record from any
  existing-claim calculation. Queued serial successors must be included in queue ranks
  and cancellation logic; a blanket `parent_timestamp` exemption is no longer correct.
- Monitor handoff transfers the effective weight before killing the starter so a
  weight-2 lander's `just check-full` monitor retains 2.0 units. Continuation must not
  double-charge or create a liveness gap. Preserve the same field through retry, repeat,
  plan, question, pipe, and gate follow-up metadata where those are serial continuations
  of this agent. Do not propagate it to unrelated independently launched workloads.
- Pending human gates and legacy unanswered questions hold zero capacity; requested
  weight remains available as metadata. Resumption and gate command execution that
  participates in the family's runner claim must transfer a still-live claim or acquire
  it through the same locked policy before work starts. Keep standalone nonparticipating
  procs and workflow Python/bash steps outside this budget.
- Keep a claim through the runner's actual work and finalization under the existing
  liveness rules. Completion, failed launch, cancellation, timeout, crash, and stale-PID
  cleanup release the entire weight. Cleanup is idempotent and cannot decrement a
  counter twice because occupancy is derived from records.

All scan-plus-check-plus-claim and claim-transfer decisions must remain atomic relative
to `runner_slots.lock`, including publication of effective weight and started state.
Legacy records with an absent weight mean 1.0. Malformed present weights must not be
silently treated as zero or lose precision through an integer coercer: reject new
invalid metadata, and fail closed with a diagnostic if a live claim cannot be accounted
for. Missing new fields in valid historical records remain readable. Scan projections
must distinguish an absent legacy field from an invalid present field; do not collapse
both into the same optional value during deserialization.

## Presentation design

The status line separates machine-wide resource usage from agent counts:

```text
8  8.0/8.0 [8 running]
4  8.0/8.0 [4 running]
8  2.0/8.0 [8 running]
9  7.25/8.0 [8 running · 1 queued]
```

These examples show the leading agent total followed by capacity immediately before the
first status `[`. Preserve the rest of the current status metrics and ordering. Always
show at least one decimal place in capacity values (`8.0`, `0.0`), retaining meaningful
fractional digits (`7.25`, not `7.2`). Suppress binary summation noise in presentation
only; do not round the accounting value. Extreme values may use readable scientific
notation. If the snapshot is unavailable, show an explicit unknown value such as
`—/8.0`, not a fabricated zero or the filtered running count.

Use the existing capacity pressure palette at 50%, 75%, and 100% of `C/L`, with over-cap
red and a neutral low-usage denominator. Keep the integer running count's current stable
green. Capacity is global to this machine's participating claims: searching, hiding
rows, folding families, switching tribe/project panels, or following remote fleet agents
must not change it. Preserve existing visible status-count scope; explain the
global-versus-visible distinction in help/details. Remote machine claims belong to their
own machine's budget.

For non-default weight, render a small text badge immediately after the agent's name and
before its status parenthetical:

```text
epic.land  w2 (RUNNING)
research.ab.cdx  w0.25 (QUEUED #2/4)
ordinary (RUNNING)
```

Use `w` dim and the number in the existing quiet cyan metadata accent, with no filled
background and no new status color. This echoes the input alias and the existing `p20`
queue notation. Do not use `×2`: folded child counts already use `×N`. Hide the badge
for both implicit 1.0 and explicit `1`/`1.0`; use compact numbers for badges (`w2`) and
minimum-one-decimal numbers for capacity/details (`2.0`). Keep it visible on queued,
waiting, running, and terminal agent nodes so it describes the workload, not current
occupancy. Render on standalone nodes, serial family nodes, and independently weighted
parallel agent nodes. Do not assign synthetic clan totals a weight or duplicate a family
badge on every serial proc/gate child row.

The selected agent detail explains non-default weight in words:
`Weight: 2.0 capacity units`. Queue details distinguish `needs 2.0 · 0.75 free`,
`waiting for ≤N other agents`, `priority N`, and `weight exceeds current limit`. Keep
request weight distinct from currently held capacity, especially for pending gates. Add
the same `wN` badge to non-default queue ladder entries and use shared eligibility for
parked styling/ranks. Order eligible entries by priority/FIFO; order parked entries by
capacity shortfall, then runner-count shortfall, then priority/FIFO. Label these as
current blockers and avoid describing this ordering as a prediction of completion times.

Add `w=`/`weight=` completion and concise documentation from the Rust editor contract
used by ACE and the LSP. Offer representative values `0.25`, `1.0`, and `2.0` without
restricting valid floats to that set. Completion must suppress both aliases when the
canonical weight field is already assigned. Update queue snippets and field hints.

## Implementation phases

### core-contracts

Implement the public grammar and numeric rules in `queue_directive.rs`. Extend
`QueueFieldsWire`, `AgentUnitWire`, dispatch prompt reconstruction, scan metadata and
waiting-marker wire fields, and the appropriate serializer/index projections. Audit Rust
`Eq` derives and struct literals affected by the floating-point field. Expose new
functionality through PyO3 and follow each wire/index's existing versioning rules;
update affected schema fixtures and cache invalidation deliberately.

Introduce a focused Rust capacity-policy module with pure, serializable inputs for
record identity/family, parallel membership, liveness/occupancy, requested weight, queue
timestamp, explicit count threshold, priority, and effective limit. Return integer
occupied-lane count, floating occupied capacity, claim ownership, per-waiter
eligibility/blockers, and stable queue order. Include the existing deference semantics
needed by admission. Python must not implement a second weighted sum or eligibility
formula. Provide binding parity tests for the complete returned snapshot.

Add editor metadata/examples for both weight keywords, with feature availability
controllable by the launch/editor flags used in later phases. The low-level value
contract can exist before public launch surfaces enable it. Verify default-only
snapshots match current grouping/counting behavior except for the explicitly chosen
future admission corrections. Exercise serial overlap, parallel children, hidden
participants, pending gates/questions, stale records, and excluded workflows in pure
fixtures. Run Rust's required `just check`, including PyO3 tests; core-only Cargo tests
are insufficient. Do not manually edit release-plz-owned crate version numbers.

### launch-plumbing

Before introducing temporarily incomplete user-facing behavior, read `sase_flags.md`
through `/sase_memory_read` and create beta scaffold `weighted_queue_capacity` with
`sase flag new` using the required enabled/disabled/removal descriptions. Its removal
condition is this epic's complete lifecycle, UI, preset, and integration coverage. Read
`sase_beads.md` before flag-bead operations. Off uses existing behavior and rejects
weight arguments with a useful unavailable-feature message; it must never accept a
requested weight and run it as 1.0. Test both states. This is temporary epic
scaffolding, not a permanent user preference.

Thread optional authored weight and resolved effective weight through
`_directive_types.py`, `_directive_extract.py`, `queue_directive.py`, typed launch
wires/adapters, launch planning/approval/dispatch reconstruction, and
`axe/run_agent_directives.py`, `run_agent_directive_metadata.py`, runner bootstrap, and
runner start inputs. Ensure a weight-only prompt triggers the required metadata path.
Add matching scan wire fields and both filesystem and cached/indexed loader enrichment;
missing field defaults must not mask a stale binding or stale index.

Preserve weight in `PromptWaitDirective`, `set_prompt_wait_and_queue`,
`set_prompt_queue`, wait-modal persistence, and all related prompt rewrite callers.
Changing priority or clearing dependency waits must not erase weight or accidentally
replace it with an explicit default. Editing independent queue fields preserves
explicitness. Keep weight itself immutable while a queued/running launch is edited.

Audit continuation metadata/allowlists and persist inherited family weight for serial
retry/repeat/monitor/plan/gate/pipe continuations. Keep independent fan-out unit weights
isolated and include weight in launch digests/receipts so a durable replay cannot reuse
approval for a materially different resource requirement. Local and remote dispatch must
carry the field; admission is on the destination machine. Incompatible consumers must
fail clearly rather than silently discard the new field.

Acceptance tests include `%q(w=...)` end-to-end extraction, all aliases and duplicate
errors, typed/untyped launch parity, request serialization/reconstruction, wait-editor
preservation, absent legacy metadata, explicit 1.0 provenance, repeat/fan-out isolation,
and scan/index parity. Extend `tests/test_queue_directive.py`, directive edit tests,
typed launch tests, and `test_core_agent_scan_wire_*` as appropriate. Run `just check`.

### admission-lifecycle

Replace affected policy in `src/sase/core/runner_slots/_admission.py` with thin Rust
adapters. Retain Python host locking, process liveness probes, and marker writes in
`run_agent_wait_slots.py`. Call the shared snapshot/decision under the lock, claim the
full resolved weight, and preserve FIFO requested time and deference continuity while
parked. Limit lookup failures remain fail-closed and release the lock for the next poll.
Do not add integer casts to weight, implicit headcount thresholds, or separate mutable
capacity counters.

Implement the family ownership rules above across initial admission, serial handoffs,
monitor start/claims/followup, pending and executing gate transitions, legacy question
reacquisition, retry, and final cleanup. Start exploration from
`monitor/{start,claims, followup}.py`, gate launch/execution adapters, and existing
runner lifecycle helpers. Exercise the real boundaries; changing a pure counting helper
alone is insufficient. Ensure failed/cancelled pending transfers leave either the old
live claim or no claim, never a phantom reservation. Include serial successors in live
waiters when they must reacquire, while retaining immediate same-weight reuse of an
occupied family claim.

Extend the existing runner-slot unit/integration suites and
`tests/fakey/test_runner_slots_e2e.py` using isolated temporary homes and controlled
process liveness. Required scenarios: four 0.25 agents exactly fill 1.0; weight 2 cannot
start with only 1.0 free; mixed simultaneous contenders never exceed the budget; lighter
eligible waiters can pass a non-fitting heavy waiter; priority/FIFO/deference still
apply; `runners=0` drains; high explicit runners cannot bypass capacity; cap
raise/lower/expiry; oversized requests; killed and crashed weighted agents; parent to
monitor to successor remains 2.0; overlapping family shells count once; pending gate
releases 2.0 and its successor queues if full; explicit conflicting active-family weight
fails; independent parallel members add their own weights. Run `just check` in every
changed code repository.

### capacity-ux

Feed the authoritative global Rust capacity projection into
`ace/tui/models/agent_runner_slots.py` and the shared agent-listing path. Keep the
integer lane count and existing count-valued JSON fields integer; add explicit numeric
`queue_weight`, occupied-capacity, effective-limit, and blocker fields rather than
changing a field named count to a fractional value. Update `agent/_running_listing_*`,
`agents/cli_list.py`, and wait listing context so machine readers and the TUI agree. No
new CLI command or option is required.

Update `AgentInfoPanel.update_state`, compatibility setters, stable-state tuples, and
`actions/agents/_display_detail_info.py` to pass actual global usage. Render the
capacity prefix and pressure styles specified above. Implement the quiet badge in agent
row prefix/styling helpers and the selected-agent metadata. Update queue ladder and wait
descriptions to remove every misleading implicit `threshold + 1` capacity display.
Preserve count-based explicit runner-condition labels.

Carry fields through `_agent_state.py`, loader enrichment, dedup/family/clan
projections, selective refresh merges, render-cache keys, and prompt-panel hint caches.
Changing weight or occupancy must invalidate exactly the affected surfaces; do no disk
reads, config lookups, subprocesses, or synchronous JSON parsing in rendering. Derive
the global capacity snapshot before visible filtering and once per refresh, using the
existing background load path. Weight-only metadata changes must survive the
artifact-delta path. Update help text if Launch Control labels/behavior change.

Verify the four status-line examples, non-default versus explicit-default badges, family
folding, terminal provenance, global usage under search/tribe/hidden filters, remote
rows, queue blockers, unavailable snapshots, 50/75/100% styling, and no metadata I/O on
render. Extend `tests/ace/tui/widgets/test_agent_info_panel.py`, row rendering,
`tests/ace/tui/test_agent_runner_slots.py`, and completion parity suites. Add focused
visual fixtures at a normal and narrow terminal width, inspect actual PNGs, and verify
name/status readability, truncation, and family expansion before accepting goldens. Run
applicable visual tests and `just check`; follow `tui_perf.md` for refresh checks.

### presets-rollout

After scheduling and presentation pass together, enable the full feature in the isolated
integration environment and make these authored prompt changes:

1. Prepend `%q(w=2.0)` once to `xprompts.bd/land_epic.content` in
   `src/sase/default_config.yml`. This covers direct invocation and both ordinary/big
   epic land-model routes. Do not also inject it in `bead/work.py`, which would create
   duplicate canonical fields after expansion. Verify actual expanded lander prompts,
   not only an unexpanded `#bd/land_epic` token. Custom shadowing xprompts retain their
   established ownership of their prompt bodies.
2. In the research plugin's `research_swarm.md`, add `w=0.25` to the existing queue
   directive in each of the four segments. Use one queue occurrence per segment, e.g.
   `%q(w=0.25)` when no other queue controls were supplied, and
   `%q(w=0.25, runners=0, priority=5)` when supplied. Preserve priority=0, optional
   initial waits, the lead's dependencies, image fork, and segment-local templating.
   Change the `runners` input's default from 16 to null and emit that condition only
   when explicitly provided. The old artificial headcount override is replaced by the
   resource weight; retained `runners=` is an optional count condition. Do not add an
   unrelated swarm weight API: the requested preset is 0.25 for every member.
3. Test plugin expansion with defaults, explicit runners=0, priority=0, custom wait, and
   combined inputs; parse each expanded segment through the new runtime and assert
   effective 0.25 with no duplicate weight. Verify packaging includes the updated
   template using the plugin's `just check` and `just test-wheel`.

Update `docs/xprompt.md`, `docs/configuration.md`, `docs/ace.md`,
`docs/troubleshooting/runner-slots.md`, relevant release-facing docs, config/schema
descriptions, and the research plugin README. Explain the float grammar, default, units,
independent priority/count conditions, explicit-threshold change, family
inheritance/reacquisition, oversized-request display, global header versus visible
counts, and workload presets. Keep examples consistent with the emitted prompts.

Coordinate Rust binding and package availability using the repositories' existing
release tooling. Pin/build the Rust revision that contains the new required API for SASE
CI and update dependency compatibility through the established mechanism. Set the
plugin's SASE dependency floor to an actual released version with weighted queue support
before publishing the new template. Do not invent a version or manually change
release-plz-owned crate versions. Verify the declared minimum with an installed wheel
smoke test, not only an editable development checkout. A consumer without the new API
must fail clearly; no Python backend fallback is allowed.

With the full integration green, delete the temporary flag's Off branch, make the On
path unconditional, remove the registry entry and flag-only editor gating, and close the
scaffold flag bead following `sase_flags.md`. The final feature has no opt-in switch.
Coordinate activation with replacement or draining of old runner binaries: they cannot
enforce weighted claims they do not understand. Existing stored records without weights
still read as 1.0; do not claim that storage compatibility makes mixed old/new active
scheduler processes safe.

## Integrated acceptance and landing

The combined feature must demonstrate real weighted admission, not just parsing or row
decoration. Use fakey agents and temporary state to show default agents consume 1.0,
landers consume 2.0 through monitored verification, and all research swarm agents
request 0.25 while respecting their existing dependency graph. Compare runtime
occupancy, CLI JSON, the capacity header, and queue details for the same snapshot.
Verify no double counting, capacity leak, stale cache, or loss of weight through a wait
edit, retry, or family continuation.

Each implementation worker reads `lint_and_test.md` and runs SASE `just check` when
changing its tracked files. Rust changes require the core repository's `just check`,
including binding tests; plugin changes require its `just check`, with wheel testing for
the packaged template. The epic lander verifies the combined tree with SASE
`just check-full` exclusively through `/sase_monitor`, using TESTING/TESTED status, and
rechecks all changed repositories' required gates. Run focused visual tests for the new
and changed scenes and inspect their outputs. Follow the approved-plan and host-owned
finalization workflows; do not manually create commits, branches, or PRs.

Acceptance is complete when both aliases and integer shorthand work, invalid weights
fail clearly, weighted claims remain accurate across all listed lifecycle boundaries,
the capacity prefix and non-default node badges meet the specified presentation, every
requested preset is exercised, the three packages can be installed together, and the
temporary rollout branch is gone.
