---
tier: epic
title: Count monitors and post-handoff family shells against max_running_agents
goal: "A running sase agent holds exactly one runner slot for as long as any of its
  shells is alive — including its monitor proc shells and every agent shell launched
  after a monitor handoff — so max_running_agents is the real ceiling on concurrent
  agents on this host, and every surface that reports occupancy agrees with the
  admission gate.

  "
phases:
  - id: count
    title: Occupancy rule and live admission gate
    depends_on: []
    size: medium
    description: "count: separate slot occupancy from slot admission in the pure
      runner-slot core, make a serial agent family hold one slot while any of its shells
      (agent or monitor) is live, and wire the live admission gate to the new count.

      "
  - id: display
    title: Occupancy parity across ACE and agent listings
    depends_on:
      - count
    size: medium
    description: "display: make the ACE capacity chip, statistics methodology text, wait
      modal, and agent listings report the same occupancy the admission gate enforces,
      including rows for post-handoff family shells that are invisible today.

      "
  - id: stats
    title: Rust core parity for historical runner occupancy
    depends_on:
      - count
    size: medium
    description: "stats: update the duplicated runner-eligibility predicate in the
      sase-core Rust crate so the Statistics runner-occupancy analysis uses the same
      rule as live admission, then release the binding and repin it here.

      "
  - id: docs
    title: Documentation sweep and cross-surface consistency check
    depends_on:
      - count
      - display
      - stats
    size: small
    description:
      'docs: retire the "serial family follow-ups do not consume slots" claim everywhere
      it appears, document the sase-agent occupancy rule and the monitor handoff, and
      verify every surface agrees end to end.'
proposed_by: bbugyi200.athena.063
status: done
bead_id: sase-ps
create_time: 2026-09-09 19:50:57
---

- **PROMPT:**
  [prompts/202608/monitor_runner_slots.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/monitor_runner_slots.md)
- **BEAD:**
  [sase-ps](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ps/README.md)

# Plan: Count Monitors and Post-Handoff Family Shells Against `max_running_agents`

## Verdict on the reported suspicion

**Confirmed.** Monitors do not consume a runner slot, and neither does anything a family
runs after a monitor handoff. The leak is larger than monitors alone: once a family
starts its first monitor, that family disappears from runner-slot accounting for the
rest of its life.

Live evidence taken from this host while writing this plan (effective cap 10):

```
effective limit: 10
admission count (today): 4
running sase agents (lanes with a live shell): 5
   05t.f0 -> [('...', monitor=True,  counted=False)]     <-- running monitor, counts 0
   062    -> [('...', monitor=False, counted=True ),
              ('...', monitor=False, counted=False)]     <-- serial pair, counts 1 (correct)
   <three standalone agents, each counted=True>
```

Retrospective scan of `ace-run` artifacts since 2026-08-10 on this host, charging each
family at most one slot:

- **471 monitor shells** and **432 family agent shells** ran without contributing to the
  count.
- Roughly **80 hours** in which at least one sase agent was running while the admission
  gate counted nothing for it, and about **181 uncounted agent-hours** in total.
- Worst observed instant: **12 sase agents running while the gate counted 0**, against a
  configured cap of 10.

## True root cause

One predicate answers two different questions.

`is_runner_slot_user_agent_record()` in `src/sase/core/runner_slots/_admission.py`
returns `False` for any record whose `agent_meta.parent_timestamp` is set and whose
`agent_family_parallel` is not `True`. That single predicate is used for both:

1. **Admission** — who may be made to park at the gate (`live_runner_slot_waiters`,
   `better_priority_agent_pending`, and the `wait_for_runner_slot` early return in
   `src/sase/axe/run_agent_wait_slots.py`), and
2. **Occupancy** — who counts as holding a slot (`running_root_agent_count`, consumed by
   `_try_claim_runner_slot`).

The original design (`sase-5u`, landed 2026-07-12) excluded child agents from _both_ on
one stated ground: "prevents parent-holds-slot-waits-on-child deadlocks". That
justification rests on an invariant — _a live parent is already holding the slot on the
child's behalf_ — which was true for every family member that existed in July 2026,
because a serial child only ever ran while its parent's runner process was alive.

Monitors landed a month later (2026-08-12) and reused the same family-member metadata
shape without that invariant holding:

- `sase monitor start` inside an agent writes the monitor member with `parent_timestamp`
  set and `agent_family_parallel` unset (`src/sase/monitor/member.py`), **then kills the
  starter's runner group** (`src/sase/monitor/handoff.py`). The starter stops being live
  and gets a done marker, so it stops counting; the monitor was never counted. The
  family drops to zero.
- When the monitor settles, `launch_followup_agent()` (`src/sase/monitor/followup.py`)
  spawns the `--next` agent as a brand-new detached runner whose `parent_timestamp`
  still points at the long-dead starter. That agent reaches `wait_for_runner_slot()`,
  matches the serial-follow-up exemption, and claims RUNNING immediately without ever
  being counted. It can start another monitor, and the cycle repeats indefinitely.

So the exemption is applied on _lineage_ (`parent_timestamp` is set) when the property
it was designed to encode is _coverage_ (some live ancestor is already holding this
family's slot). Monitors are the first shell kind for which lineage and coverage came
apart, which is why the symptom shows up as "monitors are not counted".

Two consequences worth stating explicitly, because they shape the phases:

- **In-process successors are fine and must stay fine.** `sase pipe`, `#fork`, and
  `%repeat` continue in the same runner process via `continue_as_successor()`
  (`src/sase/axe/run_agent_successor.py`), so the family's root record keeps a live pid
  and no done marker and keeps holding the slot. The live snapshot above shows family
  `062` correctly holding exactly one slot for two live shells. Any fix must not turn
  that into two.
- **The same broken predicate is duplicated in Rust.** `is_runner_eligible_record()` in
  `sase-core/crates/sase_core/src/agent_runtime.rs` implements the identical
  `parent_timestamp.is_some() && !agent_family_parallel` rule and feeds the Statistics
  tab's runner-occupancy analysis (`agent_stats/runner.rs`, `agent_stats/run.rs`). The
  historical concurrency numbers under-report for exactly the same reason.

## Corrected semantics

Adopt the definition ACE already uses for its sase-agent total and the glossary already
uses for **Sase Agent** ("an agent family or a single agent that does not belong to a
family"), and make the cap mean what its name says:

> A **runner slot is held by one running sase agent.** A serial agent family holds one
> slot for as long as _any_ of its shells is live — agent shell or monitor proc shell —
> regardless of which shell that is and whether earlier shells have exited. A standalone
> agent holds one slot. Each live parallel family member holds its own slot. A shell
> parked at `QUESTION` yields its family's slot exactly as a root does today.

This is a strict repair of the existing model, not a new one:

- Family with a live root and a live serial child → still **1** (unchanged).
- Family whose root was killed by a monitor handoff, monitor live → **1** (was 0).
- Family whose monitor settled and whose `--next` agent is live → **1** (was 0).
- Standalone agent → **1** (unchanged). Clan member launched independently → **1** each
  (unchanged). Parallel family members → **1** each (unchanged).
- Workflow Python/bash steps (`workflow_state.appears_as_agent` false) and axe Patch
  runners → **0** (unchanged; axe keeps its own `axe.max_*_runners` limits).

**Admission behavior is deliberately unchanged.** Serial family members — monitors and
monitor follow-ups included — must still never park at the gate. They inherit the slot
their family already holds; making them wait would reintroduce the deadlock the original
exemption prevented, and would be incoherent for a monitor whose starter is killed no
matter what the gate decides. Splitting the predicate is what makes the fix safe: no new
wait edges are created, only the count changes.

## Phases

### Occupancy rule and live admission gate

Own the semantic split and the live enforcement path. This phase alone fixes the
reported bug; the rest bring other surfaces into agreement.

In `src/sase/core/runner_slots/_admission.py` (and its `__init__.py` re-exports):

- Keep `is_runner_slot_user_agent_record()` exactly as it is and make its docstring say
  it answers the _admission_ question only: may this run be parked and queued at the
  gate. Keep `live_runner_slot_waiters()`, `better_priority_agent_pending()`, and
  `may_start()` on it.
- Add a slot-**occupancy** API that takes the whole record set, because the rule is
  per-family and cannot be decided from one record: a public counting function that
  returns the number of runner slots held, plus a helper that exposes the per-family
  grouping so display code can reuse it rather than reimplement it. Group by
  `(record.project_name, agent_meta.agent_family or record.timestamp)`; a record with no
  `agent_family` is its own group, which keeps standalone agents and clan members
  counting individually.
- Within a group, count 1 if any member is _occupying_, and count each live parallel
  member (`agent_family_parallel is True`) individually on top of that. A member is
  occupying when all of the following hold: `workflow_dir_name == "ace-run"`; no done
  marker; `workflow_state` is absent or `appears_as_agent`; `pending_question` is
  absent; `is_live(record)` is true; and it has been started.
- **"Started" must be monitor-aware.** For an agent shell, started means
  `agent_meta.run_started_at` is set, as today. For a monitor member
  (`agent_meta.monitor_id` set), accept a recorded pid as sufficient. The supervisor
  writes `run_started_at` only after the launch barrier releases
  (`src/sase/monitor/supervise.py`), whereas `start_monitor()` records the supervisor
  pid on the member and returns _before_ the starter's runner group is killed
  (`src/sase/monitor/start.py`). Requiring `run_started_at` would open a window in which
  the starter is already dead and the monitor does not yet count, letting a queued agent
  slip in. Occupancy must be continuous across the handoff.
- `running_root_agent_count()` is now a misleading name for what the gate needs. Either
  rename it to a slot-occupancy name and update its callers, or keep it as a thin
  deprecated alias; do not leave two live definitions of "how many slots are in use".

In `src/sase/axe/run_agent_wait_slots.py`:

- `_try_claim_runner_slot()` computes `running_count` from the new occupancy function.
  Nothing else in that function changes; the `_marker_threshold` /
  `_marker_priority_state` / `may_start` / deference logic is untouched.
- `wait_for_runner_slot()` keeps its serial-family early return verbatim. Extend the
  docstring to record _why_ the exemption is now only about waiting: the family's slot
  is already counted, so an exempt member is riding a slot rather than escaping the cap.

Tests:

- Extend `tests/test_runner_slots.py` with a table over the pure occupancy function
  covering, at minimum: standalone agent; root plus live serial child (exactly 1); root
  dead with live monitor member (1); monitor settled with live `--next` agent whose
  `parent_timestamp` names the dead original root (1); two independent families (2);
  parallel family members (individual); clan members launched independently
  (individual); `pending_question` on the family's only live shell (0); done-marker and
  dead-pid members (0); a monitor member with a pid but no `run_started_at` (1); a
  `workflow_state.appears_as_agent` false record (0); records from two projects that
  share a family name (2, not 1).
- Extend the gate tests (`tests/test_run_agent_runner_slot_capacity.py` and the
  `tests/_runner_slot_fixtures.py` helpers) with: a running monitor occupying the last
  slot parks a new implicit-cap launch; a monitor follow-up still claims immediately
  without parking; releasing the monitor releases the parked waiter.
- Add end-to-end coverage in `tests/fakey/test_runner_slots_e2e.py`: with the cap set to
  1, an agent that starts a monitor keeps the host at capacity for the monitor's whole
  lifetime and its `--next` agent, and a second launch stays `QUEUED` until the family
  finishes.

Do not add a feature flag. Per `sase/memory/sase_flags.md`, a flag is a temporary route
for behavior that is not ready to be unconditional; this is a correctness repair that
must take effect the moment it lands, and `max_running_agents` is already the
user-facing knob for how much concurrency they want.

### Occupancy parity across ACE and agent listings

Every surface that shows "how many slots are in use" reimplements the predicate, so each
one under-reports the same way and must be moved onto the shared rule. Read
`sase/memory/tui_perf.md` through the `/sase_memory_read` skill before touching loaders
or rendering, and derive everything from the snapshot the loaders already hold — no
extra artifact scans.

- `src/sase/ace/tui/models/agent_runner_slots.py`: `_holds_runner_slot()` and
  `_participates_in_runner_slots()` currently exclude every child row that is not
  `agent_family_parallel`, so the `[R/L · Q queued]` capacity chip shows the same wrong
  number the gate used to. Reproject them onto the per-family occupancy rule. The Agents
  tab already computes a "one standalone agent or one sequential family is one sase
  agent" projection in `src/sase/ace/tui/models/_agent_clan.py`
  (`_lane_summary_projections` / `sase_agent_status_counts`); reuse that lane grouping
  instead of writing a third one. `R` must equal the number the gate would compute for
  the same snapshot.
- `src/sase/agent/running_listing.py`: `_is_visible_runner_slot_child()` gates
  visibility on `is_runner_slot_user_agent_record()`, so a post-handoff family agent
  shell — a full agent doing real work — can fail every visibility branch
  (`is_root_user_agent_record` false, not a slot-participating child, and not a monitor
  record). Confirm this against a live post-handoff family and, if reproduced, make
  occupancy the visibility rule so a running shell is never omitted from
  `sase agent list`. Keep `_is_visible_monitor_record` behavior intact.
- Surfaces whose text or values state the participation rule:
  `src/sase/ace/tui/modals/statistics_help_modal.py` (the "Eligibility" methodology
  row), `src/sase/ace/tui/modals/statistics_pane_legends.py`,
  `src/sase/ace/tui/modals/wait_modal_values.py`, and the agent-list entry builders
  under `src/sase/integrations/`.
- Tests: extend `tests/test_agent_list_runner_slots.py` and the ACE runner-slot model
  tests with a monitor-holding-a-slot case and a post-handoff-follow-up case; refresh
  any PNG snapshot whose capacity chip changes with `just test-visual` and
  `--sase-update-visual-snapshots`, and state in the phase notes which snapshots moved
  and why.

### Rust core parity for historical runner occupancy

The Statistics tab's runner-occupancy analysis is computed in the sibling Rust core, not
here, so it keeps under-reporting until the same rule lands there. Per the
`rust_core_backend_boundary` instructions, do the Rust work first, then the Python side.
Open the repository with the `/sase_repo` skill (`sase repo open sase-core`) and use
only the path it prints.

- `crates/sase_core/src/agent_runtime.rs`: `is_runner_eligible_record()` is the exact
  duplicate of the Python admission predicate. Split it the same way — an
  admission/eligibility predicate and a slot-occupancy grouping — so the per-family rule
  from the `count` phase is what the statistics builder applies.
- `crates/sase_core/src/agent_stats/runner.rs` (`RunnerStatsBuilder::add_record`) and
  `agent_stats/run.rs` are the two call sites. Occupancy is interval-based there, so the
  grouping must merge a family's shell intervals into a single occupancy interval rather
  than summing them; overlapping shells during an in-process handoff must not
  double-count, and a gap between a starter's exit and its monitor's start must not
  reopen the family's interval.
- Add Rust unit tests mirroring the Python table from the `count` phase case for case,
  so the two implementations are reviewable against one list. Follow the crate's
  existing release and versioning conventions for `sase_core_rs`.
- In this repo: repin the binding, run `just install`, and add a parity check that a
  synthetic artifact index produces the same occupancy through `sase stats` as the pure
  Python function produces for the same records. Note in the phase notes that historical
  statistics for windows before this change now read higher than they did — the numbers
  were always wrong, and the analyzer recomputes from stored artifacts rather than from
  a frozen series.

### Documentation sweep and cross-surface consistency check

The current behavior is documented as intended in several places, so the fix is not
complete until those statements are retired. Every one of these says some variant of
"serial family follow-ups do not consume these slots":

- `docs/configuration.md` — the `max_running_agents` field section.
- `docs/xprompt.md` — the runner-slot semantics subsection under `%wait`.
- `docs/ace.md` — the `[R/L · Q queued]` capacity chip description and the wait-modal
  section.
- `docs/troubleshooting/runner-slots.md` — the participants paragraph and the exemption
  paragraph.
- `docs/llms.md` — the effective-cap description.
- `src/sase/default_config.yml` — the comment above `max_running_agents`, which is the
  first thing a user reads.

Replace them with the sase-agent occupancy rule, and state the two halves separately so
the next reader does not re-conflate them: which runs _hold_ a slot (per-family
occupancy) and which runs _wait_ for one (roots and parallel members only). Add a short
subsection to `docs/monitors.md` saying that a monitor holds its family's runner slot
for its whole lifetime and hands it to the `--next` agent, so a monitor is not a way to
free capacity — that is the single most likely surprise for an existing user.

Also call out the operational consequence in `docs/configuration.md`: on a host that
uses monitors heavily, the same `max_running_agents` value now admits fewer new agents
than it did before, and raising the value is the supported response. Do not change the
packaged default of `10` as part of this epic; that is the project owner's call, not a
silent side effect of a correctness fix.

Finish with a consistency pass on a live host: compare the ACE capacity chip, the
Statistics runner view, `sase agent list`, and the gate's own count while at least one
monitor is running, and record the four numbers in the phase notes.

## Risks and mitigations

- **Effective concurrency drops.** This is the intended effect, but it is user-visible
  on the first monitor-heavy run after landing. Mitigated by documenting it prominently
  and by leaving `max_running_agents` as the knob; the `docs` phase says so explicitly.
- **Double-counting a family during an in-process handoff.** A family whose root record
  and successor record are both live must count once. Covered by an explicit unit case,
  by the interval-merge requirement in the Rust phase, and by the live `062` snapshot in
  this plan as the regression baseline.
- **A zero-occupancy window at the monitor handoff.** Addressed by the monitor-aware
  "started" rule; a queued agent must not be able to slip in between the starter's death
  and the supervisor's first `run_started_at` write.
- **Python and Rust drifting apart again.** The two implementations already drifted into
  the same bug by copying. Mitigated by the mirrored test tables and the `sase stats`
  parity check, and by both docstrings naming the other implementation.
- **Stale or unreadable family metadata.** A record with no `agent_family` must fall
  back to counting individually rather than collapsing unrelated records into one group;
  a malformed record must not be able to make the gate under-count, since under-counting
  is the failure mode being fixed.

## Out of scope

- Making monitors or monitor follow-ups _wait_ at the admission gate. They inherit their
  family's slot by design; changing that risks the deadlock the exemption exists to
  prevent and would strand a handoff whose starter is killed regardless.
- Any change to `axe.max_hook_runners` / `axe.max_agent_runners` or the axe `RunnerPool`
  / `SharedRunnerPool`. Those govern a separate world and are already documented as
  excluded.
- Changing the packaged `max_running_agents` default, adding a per-project or
  per-provider cap, or adding preemption of running agents.
- Adding a feature flag for the new accounting.
