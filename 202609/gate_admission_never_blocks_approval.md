---
tier: epic
title: Gate approval never blocks on weighted capacity
goal: "Answering a tale/epic plan gate on a machine at full weighted runner capacity
  completes promptly: the coder agent launches and parks as QUEUED, and the epic-launch
  monitor starts immediately with an explicit queue weight of 0.

  "
phases:
  - id: core-zero-weight
    title: Allow explicit zero-weight capacity records in the Rust core
    depends_on: []
    size: medium
    description:
      "core-zero-weight: in sase-core's runner_capacity.rs, accept explicit queue_weight
      0.0 records as valid non-occupying, non-reusable-lineage capacity records while
      limits and user-authored %q weights stay strictly positive, with fail-closed
      tests."
  - id: gate-exec-nonblocking
    title: Make gate-shell execution admission non-blocking
    depends_on: []
    size: medium
    description:
      "gate-exec-nonblocking: replace the wait_for_runner_slot loop in gate_shell/log.py
      with one locked claim attempt that degrades to unclaimed execution with no phantom
      claim or waiting marker, so answers complete on every surface and successors
      self-acquire capacity."
  - id: epic-monitor-zero-weight
    title: Epic-launch monitor carries explicit weight 0 and full-capacity acceptance
    depends_on:
      - core-zero-weight
      - gate-exec-nonblocking
    size: medium
    description:
      "epic-monitor-zero-weight: author queue_weight 0 on the epic-launch monitor member
      only, keep general monitor claim lineage intact, check capacity presentation, and
      add fakey end-to-end acceptance for tale/epic approval and rejection at a full
      weighted limit."
proposed_by: bbugyi200.athena.fa
create_time: 2026-09-13 19:13:22
status: wip
---

- **PROMPT:**
  [prompts/202609/gate_admission_never_blocks_approval.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/gate_admission_never_blocks_approval.md)

# Gate Approval Never Blocks On Weighted Capacity

## Context

Approving (or rejecting) a tale/epic plan gate on a machine at full weighted runner
capacity hangs indefinitely. Reproduced live on apollo (2026-09-13): tale gate
`84a425e2` (`w--gate`, plan `mac_capture_route_plus_commit.md`) journaled
`attempt_started` at 22:42:08, wrote a `waiting.json` runner-slot marker at 22:42:09
(`queue_weight: 1.0`, `queue_capacity: 0`), and then sat forever with the flock on the
bundle's `.response.lock` held by the ACE process. No option command child was ever
spawned, no `response.json` was written, `sase gate list` still showed the gate as
pending, and the coder agent was never launched. The same block occurs before the epic
`sase bead work` monitor/proc is created, and it also affects reject/feedback answers
and the detached `sase gate answer` proc path (`gate-answer-detach`), which is the "hung
proc" visible in the Procs tab.

Root cause: commit `7da379ea28` (bead sase-z4.6.2, "make weighted shell admission
atomic") added `_claim_gate_shell_execution_capacity()` to the `on_command_start`
callback in `src/sase/gate_shell/log.py`. For every shell-backed gate it calls
`wait_for_runner_slot(...)` (`src/sase/axe/run_agent_wait_slots.py`), which loops with
backoff until admitted or killed. That wait is correct inside a launched agent-runner
process (the agent parks and renders as QUEUED/WAITING), but inside gate execution it
blocks a human-facing approval on capacity that may not free for hours, while holding
`.response.lock` (so even `cancel_gate` fails with `lock_timeout` after 5s).

This violates the project decision that a gate never blocks (`gates-never-block`): the
gate's option commands are short trusted host commands, and the heavy work (the coder
agent, the epic's phase workers) already queues correctly through its own runner
process. The intent of sase-z4.6.2 — a successor must not bypass admission by reusing a
claim that was never acquired — must be preserved: Rust's `is_waiting_record` /
`serial_continuation_reuses_claim` in `runner_capacity.rs` already forces a successor to
queue when its lineage claim is not live, so an unclaimed gate execution is safe as long
as no phantom claim or stale waiting marker is left behind.

Desired behavior (confirmed with the owner):

1. Answering a gate always completes promptly. If capacity is free, the gate still
   acquires/transfers the claim atomically (today's happy path) so the follow-up coder
   agent inherits it with no gap. If capacity is full, the gate executes without a claim
   and the follow-up coder agent launches and parks as QUEUED, exactly like any other
   launch at capacity.
2. The epic-launch monitor (`start_epic_launch_monitor` in
   `src/sase/bead/epic_launch.py`, e.g. `v--mon`) is pure supervision around
   `sase bead work`; it must carry an explicit queue weight of `0` so it always starts
   and never consumes a runner lane. Rust currently rejects non-positive weights
   (`queue_weight_is_valid` in `queue_directive.rs` requires `> 0`), so zero-weight
   records need a core policy change. Zero weight stays an internal shell-metadata
   value: user-authored `%q`/`%queue` weights remain strictly positive.

## Remediation note (not a phase)

The currently wedged apollo gate/ACE thread predates this fix; it unsticks when capacity
frees or ACE restarts. No code in this epic needs to handle already-wedged processes.

## Phases

### Phase `core-zero-weight` — Allow explicit zero-weight capacity records in the Rust core

size: medium

In the sibling Rust core repo (`sase-core`, crate `sase_core`; open it through
`sase repo` tooling — note `sase repo open sase-core` currently mis-resolves the project
and errors, use the checkout path printed by `sase repo list`):

- Extend the runner-capacity policy in `runner_capacity.rs` so a record whose
  `queue_weight` is exactly `0.0` and explicit (`queue_weight_explicit: true`) is valid,
  occupies `0.0` lanes, and is never a capacity blocker. Effective limits and admission
  limits must remain strictly positive; do not loosen `queue_weight_is_valid` for
  limits. Introduce a separate record-weight predicate (e.g. `record_weight_is_valid`,
  `>= 0` when explicit) rather than widening the shared one.
- Zero-weight records must not become reusable claim lineages: a successor whose lineage
  resolves to a zero-weight claim must acquire its own capacity (treat a zero-weight
  claim as not reusable in the claim-lineage logic).
- Keep `%q`/`%queue` directive parsing/validation in `queue_directive.rs` unchanged:
  user-authored weights stay `> 0`. Only wire-level record metadata may carry zero.
- Add unit tests: zero-weight monitor-shaped record occupies nothing at a full limit; a
  waiter still admits when the only other records are zero-weight; invalid cases
  (implicit zero, negative, NaN) still fail closed; zero-weight lineage is not reusable
  by a successor.
- Update the Python binding surface only if a signature changes (the snapshot API should
  not need one), and run the core repo's own checks.

Dependencies: none.

### Phase `gate-exec-nonblocking` — Make gate-shell execution admission non-blocking

size: medium

In `src/sase/gate_shell/log.py` (and `src/sase/axe/run_agent_wait_slots.py` only if a
small helper export is needed):

- Replace the `wait_for_runner_slot(...)` call in `_claim_gate_shell_execution_capacity`
  with a single locked claim attempt (the existing `_try_claim_runner_slot` semantics):
  if the decision is `acquire_capacity` or `reuse_existing_claim`, publish ownership and
  claim exactly as today; if the decision is blocked, do NOT loop, do NOT park — remove
  any `waiting.json` this attempt wrote for the gate's artifacts dir, leave no phantom
  claim and no `slot_requested_at` marker, and return so command execution proceeds
  unclaimed.
- When execution proceeds unclaimed, the gate shell record must stay non-occupying (its
  `gate_state` remains `pending` until settlement, which Rust's `is_pending_gate`
  already excludes) and its successor must self-acquire: verify the follow-up coder
  agent launched by the gate handoff goes through the agent-runner
  `wait_for_runner_slot` path and renders as QUEUED with rank in ACE.
- Preserve pid/process-identity recording for `sase gate` runaway reporting on both the
  claimed and unclaimed paths.
- The change must cover every execution surface that binds these callbacks: in-process
  ACE answers, `sase gate answer` (detached and `--no-detach`), telegram, and
  auto-resolution.
- Add regression tests: at a fully occupied weighted limit, executing an approve (and a
  reject) on a shell-backed plan gate completes without blocking, writes
  `response.json`, settles the gate, and leaves no waiting marker for the gate shell;
  with free capacity the claim/transfer behavior is unchanged (assert ownership
  publication still happens under the lock); a successor after an unclaimed execution
  parks instead of reusing the dead lineage claim.

Dependencies: none (this phase must not require zero-weight support).

### Phase `epic-monitor-zero-weight` — Epic-launch monitor carries explicit weight 0 and full-capacity acceptance

size: medium

Depends on: `core-zero-weight`, `gate-exec-nonblocking`.

- In `src/sase/bead/epic_launch.py` (`start_epic_launch_monitor`) and the monitor member
  creation path (`src/sase/monitor/member.py` / `src/sase/monitor/start.py`), author
  `queue_weight: 0` with `queue_weight_explicit: true` on the epic-launch monitor member
  instead of inheriting the planner's weight. Scope this to the epic-launch monitor; do
  not change test/general monitors, whose deliberate claim-holding lineage behavior
  (parent-to-monitor-to-successor transfer) must stay intact — add a test proving a
  general monitor still holds/transfers its inherited claim.
- Confirm the fallback proc path (`_submit_epic_launch_task`) needs no weight: procs
  outside the agent budget must stay outside it.
- Verify presentation: a zero-weight monitor must not distort ACE capacity strips or
  weight badges (`0.0/10.0`-style rendering and `_queue_weight_badge`); adjust rendering
  expectations/goldens only for rows this change actually affects.
- End-to-end acceptance at a fully occupied weighted limit, using isolated temporary
  state plus the existing fakey integration infrastructure:
  1. Approving a tale gate completes immediately; the coder agent exists and is QUEUED;
     when capacity frees it is admitted.
  2. Approving an epic gate completes immediately; the epic-launch monitor starts at
     once with effective weight 0; `sase bead work` runs; the launched phase workers
     park as QUEUED; total occupied weight never exceeds the limit.
  3. Rejecting a gate at full capacity completes immediately.

## Verification

Each phase runs the focused tests it adds plus `just check` in its repo; the final phase
also exercises the fakey end-to-end acceptance above. Follow `lint_and_test.md` (read
via `sase memory read`) for the SASE repo phases and the core repo's own check recipes
for the Rust phase.
