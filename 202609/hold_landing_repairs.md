---
tier: epic
title: Complete hold admission and visibility after the landing audit
goal: 'Holds select the same targets across CLI, directives, admission, and display;
  arming is ordered with admission; capture and release evidence stays visible; and
  observable wait cycles are reported on a supported released core.

  '
parent_bead: sase-11l
phases:
- id: selector-parity
  title: Unify hold selectors and effective tribe identity
  depends_on: []
  size: medium
  description: 'selector-parity: share Rust selector expansion across CLI and directives,
    integrate stored and contextual tribe membership into admission and display, and
    correct required CLI operands.'
- id: admission-ordering
  title: Order hold arming with agent and proc admission
  depends_on:
  - selector-parity
  size: medium
  description: 'admission-ordering: serialize hold publication with the final pre-run
    admission transition, cover both race orders, and preserve running-work immunity
    and fail-open recovery.'
- id: capture-lifecycle
  title: Persist capture summaries and report expiry releases
  depends_on:
  - admission-ordering
  size: medium
  description: 'capture-lifecycle: retain effective arm-time counts across rebind,
    render them in CLI and Holds pane, and carry validated expiry and liveness prune
    outcomes to deduplicated notifications.'
- id: deadlock-integration
  title: Complete deadlock detection and supported-core acceptance
  depends_on:
  - capture-lifecycle
  size: medium
  description: 'deadlock-integration: traverse every relevant wait branch including
    hood dependencies through shared core policy, finish released-core adoption, and
    prove the repaired hold composition across its production paths.'
proposed_by: bbugyi200.athena.sase-11l.land
create_time: 2026-09-18 18:08:37
status: wip
bead_id: sase-11l.11
---

- **PROMPT:** [prompts/202609/hold_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/hold_landing_repairs.md)
- **PARENT:** [202609/hold_directive.md](https://github.com/sase-org/sase--plans/blob/main/202609/hold_directive.md)
- **BEAD:** [sase-11l.11](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11l/sase-11l.11.md)

# Plan: complete hold landing repairs

## Scope and evidence

This is only the remaining work discovered while landing `sase-11l`. Its ten phases and
nine nested descendants are closed, but the approved contract is not complete. The
original design is `plan:202609/hold_directive.md`; the nested plans are
`plan:202609/hold_directive_surface.md` and `plan:202609/hold_launch_arming.md`. Read
them with `sase artifact read`. Open the research sidecar before reading
`research:202609/reverse_wait_hold_barrier/reverse_wait_hold_barrier.md`; an absent
local checkout was the reason that earlier workers reported this artifact missing.

The audit read every descendant and note, inspected epic and intervening commits, and
tested source at `fa61906da` with the published `sase-core-rs 0.34.53`. The checkout was
then fast-forwarded to fetched base `bb332b5aa`; that commit defers TUI startup work and
does not change hold semantics. The linked core was clean at `b7531db`. No stranded
phase-10 Rust changes were found: the flag removal landed in core `8d5341a`, mixed with
finalizer work, and Python `a60210501`.

The focused hold/hood/deadlock/temp-guard and two previously failing test nodes passed:
**77 tests**. Separate isolated probes using the real binding nevertheless showed:

- CLI `names=["team"]` does not block `team--code` in family `team`; directive expansion
  of the same selector does.
- A stored posthoc `ops` tribe assignment is recognized by the wait index, while the
  same runner admission record has `tribe=None`.
- A matching hold can publish after the admission snapshot read but before `claim()`,
  and the candidate still starts.
- Persisted hold records contain no capture counts; reading an expired hold removes it
  without invoking a release notification.
- An armer waiting on `[safe, bridge]`, where `bridge` waits on the held candidate,
  produces no deadlock notification because only the first branch is traversed.

The parent bead's landing audit, `file:explicit:50d20b6062acfc5207bafbbb`, contains the
complete isolated probe script, output, source findings, and all nine proposed-follow-up
dispositions. Read it with `sase artifact read`. These defects remain epic work, not
standalone feature tasks.

## Common constraints

- Shared selector, identity, graph, and store policy belongs in `sase-core`, with PyO3
  bindings and thin Python adapters. Open that repo through `/sase_repo`. Do not
  implement a competing Python policy or a Python fallback.
- Retain the pull model: never write a blocking dependency into another launch's
  markers. Holds remain host-local, project-scoped by default, TTL-bounded, fail-open,
  and irrelevant to already running agents or dispatched procs.
- Preserve pre-arm idempotency, rollback, coordinator/runner re-anchor, original
  timestamps on rebind, armer/kin exclusion, terminal proc release, and implied
  non-authored priority. Do not reintroduce the retired `agent_holds` Off branch.
- The phases are ordered to avoid overlapping edits to the facade, store wire, and
  bindings. Each is bounded direct implementation work; use its evidence and acceptance
  cases rather than replanning the original feature.
- Read the relevant reference memories through `/sase_memory_read`, especially
  `lint_and_test.md`, `cli_rules.md`, `xprompts.md`, and `tui_perf.md`.
- Each phase runs the applicable focused regressions and `just check` in SASE; Rust
  changes require the core's whole `just check`, including PyO3 tests. Update binding
  indexes and validation tools when the API changes. Keep required checks through the
  documented monitor handoff when they are long-running.
- Do not edit memory. Ready memory task `sase-134`, dependent on `sase-11l`, owns the
  explicit documentation follow-up. Child workers record unrelated discoveries as
  `PROPOSED FOLLOW-UP:` notes on their own phases.

## 1. selector-parity

`src/sase/core/agent_hold_facade.py::_hold_selectors_wire` currently puts CLI names only
into `names`, while Rust `hold_directive::hold_fields_to_selectors` also fills families,
clans, and workflows. Route both entry points through one Rust-owned expansion and
validation contract. Cover exact agent, family, clan, workflow, proc shell, hood, tribe,
and future selectors; retain exact role-suffixed name behavior and arm-time self/kin
validation.

Integrate the concurrent tribe changes (`edde28a8d`, `9759e5afe`, `bb839b4ea`,
`c05aa3a94`): use the existing contextual tribe resolver and stored evidence for both
selector normalization and candidate membership. CLI's current @ stripping and the
directive's context-free public-name canonicalization are insufficient for the job/chop
ambiguity contract. Do not equate a user's independently stored `job` tribe with the
automation `chop` identity without the shared evidence decision.

Admission currently uses raw `meta.tribe or meta.clan_tribe` in
`core/runner_slots/_admission_capacity_records.py` and `axe/run_agent_wait_slots.py`. It
must honor posthoc assignments and effective clan generation precedence as wait/display
resolution does. Update any necessary Rust wire fields and Python producers together.
Cover synthetic admission candidates, scan-backed snapshots, and the cached TUI roster
path in `ace/tui/models/agent_runner_slots.py`, without adding full-history scans per
poll or disk work to rendering. Reuse one snapshot of identity evidence per decision.

Correct `main/parser_agent_hold.py` and `agents/cli_hold.py` to support the approved
positional operands: selector names/@tribes for `create`, required armer key for `show`,
optional armer key for `release`. Preserve existing optional `-n/-t/-k` spellings where
unambiguous; `show -k` must no longer be the sole required form. Keep sorted help,
aliases, selector-free `run` semantics, and completion snapshots consistent. These are
corrections to the original CLI contract, not new commands.

Acceptance: real-store CLI and directive parity for family/clan/workflow/name selectors;
posthoc tribe and conflicting clan-member metadata; canonical automation alias versus
independently stored `job`; self/kin rejection; project/host scope; CLI help/parser/JSON
and ACE versus actual-admission parity.

## 2. admission-ordering

The runner holds `runner_slots.lock` across its hold snapshot and claim, but
`arm_agent_hold` only takes the independent Rust hold-store lock. Thus an arm can
complete inside that interval. Make publication and the final admission transition share
a documented ordering boundary. Reuse the existing runner/admission locks where
possible; do not introduce a global completion graph or a second scheduler.

Cover all shared arming paths (CLI, runner bootstrap, typed pre-arm) and both target
kinds. Inspect `launch_admission_engine.py`: hold blocks are computed before the action
batch, so a stale proc dispatch action must not bypass a newly published hold. Define
the proc's committed pre-run dispatch transition explicitly, and recheck at that
transition; once committed, the proc is immune.

Document lock order among bundle admission, runner slots, and the hold store. Pending
capture calls `agent_list_entries`, which itself reads holds: compute expensive
capture/liveness data outside nested critical sections or use an explicit non-recursive
snapshot API. Never send notifications or run slow process-launch work under the global
runner lock. Preserve bounded store lock acquisition and fail-open reads; report an
explicit arm failure without leaving a partial hold.

Acceptance uses deterministic barriers/events, not probabilistic sleeps: arm wins before
the final check and the candidate parks; claim/dispatch wins first and later arm never
reblocks running work; an arm attempted between snapshot and claim cannot complete
unnoticed; multiple holds release independently; proc action batches cannot use stale
hold eligibility; malformed/dead/expired holds still fail open. Exercise the actual arm
facade and actual admission transition, not only mocked predicates.

## 3. capture-lifecycle

The original plan sections 4.4 and 4.8 explicitly require persistent capture counts. Add
an optional typed arm-time summary to the Rust record and binding. Keep older records
readable with an honest unknown/not-recorded display rather than invented zeros.
Preserve counts and frozen identity sets across every rebind. Compute counts from
effective capturable identities after scope and armer/kin exclusion, and make
preview/confirmation, arm output, and the stored summary use the same definitions.
`pending` still does not capture undispatched procs.

Render the historical waiting/queued/skipped-running summary in `hold list`, `hold show`
(including JSON), and `ace/tui/modals/holds_pane.py`. Continue using the existing worker
and cached presentation paths. Counts describe arm time, not a fresh scan each time the
panel paints.

Fix expiry evidence at the store boundary. Currently the first read in
`_list_and_reconcile_holds` removes TTL-expired rows before Python can notify. Have the
locked Rust prune operation return validated removed records plus reason (expiry versus
dead armer) in a versioned/defaultable result. Route that result through the shared
Python lifecycle adapter; avoid reparsing raw JSON before the lock or depending on
whichever frontend happened to read first. Notifications remain best effort, deduped by
hold lifetime/armer as appropriate, and must never turn fail-open admission into a
failure. Ensure no-liveness reads, doctor, rebind, and background readers cannot
silently consume the only expiry evidence.

Acceptance: restart/read and launch-to-agent/proc rebind preserve summaries; legacy
records render honestly; excluded kin do not inflate capture thresholds; exact TTL
boundary emits the release notification once across repeated/concurrent reads;
dead-armer release keeps the right reason; malformed records do not generate bogus
notifications or block admission. CLI/pane tests verify the presented counts.

## 4. deadlock-integration

Replace the single-path `next(...)` walk in
`axe/run_agent_wait_slot_candidate.py::hold_deadlock_armer_record`. Build bounded facts
from the already available relevant wait set and delegate complete graph reachability to
Rust. Visit every branch with a visited set. Match dependencies through the shared
identity rules; include the new hood dependencies instead of reading only `waiting_for`.
Honor the waiter's launch cutoff and self exclusion. Do not report unrelated, running,
or already settled branches as a mutual block, and do not auto-release holds to break
cycles. Keep the existing deduped admission notification and TTL backstop.

Acceptance: direct cycle, branched transitive cycle whose first branch is harmless,
longer cycle with repeated vertices, hood-mediated cycle, no-cycle graph, running armer
immunity, and notification deduplication. Include CLI/directive-produced records in the
cases so the earlier phases' identity facts are exercised.

Complete released-core adoption for this repair set using the repository's ratchet
tools. The original git pin `8d5341a` covers hood/unconditional contracts, but the
package minimum `0.34.48` predates them (first release `0.34.53`). The final minimum
must include every new repair API as well. Validate installed binding and LSP behavior
at the supported floor, and leave no unpublished binding or stale-core override as the
only evidence of success. Do not manually change Rust release versions; follow the core
release workflow.

Run the integrated production-path regression for
`%proc ... %q:1 %hold(pending, future)`: existing load drains, later matching agents and
undispatched procs remain held, and terminal proc settlement releases them. Retest all
four armer/target kind combinations, malformed/expired stores, killed and failed armers,
family handoff, future arrivals after settlement, authored priority overrides,
`%repeat`/`%dispatch` rejection, and ACE/LSP parity. Retain the existing proc queue
weight-0 capacity semantics and running-work immunity.

## Landing evidence and parent handoff

The child land agent must review the combined tree and new base drift, run the whole
core check and SASE `just check-full` via `/sase_monitor` (after `just fix`), and
resolve any failures caused by this work. Preserve the parent's follow-up ledger in its
final landing evidence; do not create duplicates for historical proposals that already
have resolutions or task owners.

There is no phase for closing `sase-11l`, running its post-close Symvision check, or
changing its original plan status. The direct `parent_bead: sase-11l` link is the
handoff: after this child is complete, its land agent rechecks the parent's descendants,
linked plans, notes, and post-child drift before resuming the interrupted normal landing
under the user's instructions.
