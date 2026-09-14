---
tier: epic
title: Failed monitor member permanently blocks family wait resolution
goal: "A family wait resolves once a lane recovers from a failed shell member via a
  same-kind retry, monitor starts stop minting doomed family members or stealing live
  workspace claims, and permanently blocked waiters surface as notifications.

  "
phases:
  - id: wait-supersession
    title: Superseded failed shell members stop blocking family waits
    depends_on: []
    size: medium
    description:
      "wait-supersession: classify monitor/gate shell members on ArtifactCandidate and
      exclude a terminal-failed, follow-up-less shell member from the family's effective
      generation when a newer same-kind shell member exists in the same generation, with
      reproduction tests for the sase-zt.6.5.3 incident."
  - id: monitor-start-claim
    title: Monitor start stops minting doomed members and stealing live claims
    depends_on: []
    size: medium
    description:
      "monitor-start-claim: pre-flight the lane workspace claim before
      create_monitor_member so a doomed start raises without creating a family member,
      resolve stale transfer pids by adopting only dead holders' claim rows (never
      transferring away from a live process), and make no-op releases record truthfully."
  - id: terminal-block-notify
    title: Surface permanently blocked waiters instead of only logging
    depends_on: []
    size: small
    description:
      "terminal-block-notify: have the wait_checks chop upsert one deduplicated inbox
      notification per terminally-blocked waiter naming the blocker and actionable
      guidance."
proposed_by: bbugyi200.kellys_mbp.0i.f0
create_time: 2026-09-13 21:53:01
status: wip
---

- **PROMPT:**
  [prompts/202609/failed_monitor_blocks_family_wait.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/failed_monitor_blocks_family_wait.md)

# Failed monitor member permanently blocks family wait resolution

## Problem

The `sase-zt.6.5.land` agent on athena never started: its
`%w:sase-zt.6.5.1,sase-zt.6.5.2,sase-zt.6.5.3` wait can never resolve because the
`sase-zt.6.5.3` family contains a monitor member that failed at start and was never
"replaced" in the eyes of the wait resolver, even though retry monitors recovered the
lane and the whole plan chain completed (workers `--2`..`--4` completed, the phase bead
closed).

### Root-cause chain (verified against the athena incident artifacts and the workspace-claim ledger)

1. **Claim-side trigger.** `sase-zt.6.5.3--plan` started the plan-chain check-full
   monitor (`--mon`, artifact `20260913170758`) with `transfer_from_pid` taken from its
   own `agent_meta.json` `pid` (1556836). That pid was stale: the lane's workspace #11
   claim had legitimately moved down the plan chain (gate shell at 17:00:29, then
   `--1`'s runner pid 3917772 at 17:01:48). The transfer correctly failed:
   `could not claim workspace for monitor: workspace #11 with pid 1556836 was not found; conflicting RUNNING claim: #11 pid 3917772 workflow ace(run)-260913_170147`.
2. **Permanent family blocker.** `_teardown_failed_member()`
   (`src/sase/monitor/start.py`) stamped the half-created member with
   `done.json = {outcome: "monitored", monitor_state: "failed", error: ...}`.
   `effective_done_outcome()` maps that to `failed`, so the `ArtifactCandidate` for
   `--mon` has `is_resolved=False` and no `shell_followup_agent` (it never launched a
   follow-up). The family aggregate in
   `src/sase/core/wait_dependency_resolution/_index_entities.py` (`_family_entity` /
   `_family_handoff_state`) only excludes shell members whose _successful follow-up_ is
   present in the generation, so `--mon` stays in the effective generation forever and
   `is_resolved` for the family can never become true. The retry monitors that recovered
   the lane (`--mon-0`, `--mon-1`, `--mon-2`) do not replace it: `--mon-0` also had
   `monitor_state: "failed"` (its `just check-full` exited 1) but is excluded because it
   recorded `monitor_followup_outcome: "launched"` → `--2`; the start-failed `--mon` has
   no follow-up at all.
3. **Consequence.** `dependency_resolution_status()` returns
   `blocked_on=('sase-zt.6.5.3',)` for the land agent's `waiting.json` on every
   `wait_checks` chop run and every runner fallback check, forever. The chop logs
   "Terminal dependency still blocks waiter" but nothing surfaces to the user, so the
   hang sat silent for hours.
4. **Additional claim-ledger findings** (from `~/.sase/logs/workspace_claims.jsonl` on
   the incident host):
   - The teardown's `undo_monitor_claim()` → `release_workspace()` recorded
     `success=True` with identical before/after content (it matched nothing — the claim
     belonged to `--1`). A no-op release should not report as a successful mutation.
   - The retry `--mon-0` (17:10:10) resolved the lane to the newest member `--1` and
     **transferred the workspace claim away from `--1`'s still-running runner** (pid
     3917772, alive until 17:24:08). The monitor then ran `just check-full` in workspace
     #11 concurrently with `--1` working in it.

### Design intent to preserve

Fail-closed semantics are correct and must survive: a family whose newest shell member
failed terminally with no successor **should** keep blocking waits (that state means the
chain died; `test_unsuccessful_monitor_blocks_and_is_reported_as_terminal` in
`tests/test_monitor_wait_dependency.py` codifies this). The bug is only that a failed
shell member which was _superseded by a newer same-kind shell retry in the same
generation_ keeps blocking after the lane recovered. A lane is sequential and only ever
has one active monitor at a time (see `allocate_monitor_suffix()` in
`src/sase/monitor/naming.py`), so a newer same-kind shell member is by construction a
retry that supersedes the failure.

## Non-goals

- Do **not** blanket-exclude failed or start-failed shell members from family
  aggregates; exclusion must require a newer same-kind superseding shell member in the
  same generation.
- Do not change the Rust-core claim-planning primitives
  (`plan_transfer_workspace_claim_from_content` etc.); the core's refusal to transfer
  from a missing pid was correct behavior. The wait-dependency resolver and the
  monitor-start flow are Python-only in this repo; no `sase-core` change is expected.
- No feature flag: these are bug-fix semantics with no old branch that must remain
  reachable.
- The deeper orchestration issue — LaunchApproval launching `--1` while `--plan` was
  still alive in the same workspace, i.e. two family members overlapping in one checkout
  — is out of scope. The final phase should record it as a `PROPOSED FOLLOW-UP:` note on
  its phase bead, not fix it.
- Live remediation of the stuck athena agent (writing `ready.json` into the waiting land
  agent's artifact dir) is an operational action outside this epic.

## Phases

### Superseded failed shell members stop blocking family waits (`wait-supersession`)

In `src/sase/core/wait_dependency_resolution/`:

1. Add a shell-member kind to `ArtifactCandidate` (`_types.py`), e.g.
   `shell_member_kind: str | None` with values `"monitor"` / `"gate"` / `None`. Populate
   it in `WaitDependencyIndex._add_prepared()` (`_index.py`) from the meta mapping using
   the existing classification helpers already used by `_is_family_shell_member_meta()`
   in `_artifact_state.py`: `is_monitor_member_role(agent_family_role, role_suffix)` and
   `is_real_gate_member(agent_family_role, gate_id)`. Both index construction paths pass
   full meta mappings (`build()` reads `agent_meta.json`; `add_scan_record()` receives
   `asdict(AgentMetaWire)` from `sase/agents/_wait_live_rows.py`
   `_index_from_snapshot()` and from `sase/scripts/_chop_incremental_index.py`
   `wait_rows_from_index_records()`), so no wire changes should be needed — verify the
   fields are present in all three paths.
2. Extend the family aggregation in `_index_entities.py` so the effective generation
   additionally excludes a member that is (a) a shell member of kind K, (b) terminal for
   wait purposes with no successful follow-up (`has_done_marker` true, effective outcome
   not in `WAIT_SUCCESS_OUTCOMES`, `shell_followup_agent is None`), and (c) superseded —
   at least one member of the same kind K with a strictly later timestamp exists in the
   same generation. Implement it in `_family_handoff_state()` (or a sibling applied at
   the same call sites) so both `_family_entity()` and `family_candidate_for_root()`
   (`_index_queries.py`) get identical behavior. The `handoffs_present` computation must
   keep working on the pre-exclusion candidates exactly as today.
3. Confirm `terminal_blocking_artifacts_for_name()` stops reporting superseded members
   (its `members` come from the entity's effective generation) so the `wait_checks` chop
   no longer logs them as terminal blockers.
4. Tests (extend `tests/test_monitor_wait_dependency.py`,
   `tests/test_gate_wait_dependency.py`, and the chop coverage in
   `tests/test_axe_chop_wait_checks.py`):
   - Reproduction of the incident: family root + start-failed `--mon` (`done.json`
     exactly in the teardown shape: `outcome: "monitored"`, `monitor_state: "failed"`,
     `error`, no follow-up fields) + later `--mon-0` with
     `monitor_followup_outcome: "launched"` naming a completed follow-up worker present
     in the generation → family `is_resolved`, `is_resolved("lane")` true,
     `terminal_blocking_artifacts_for_name("lane") == ()`, and an incident-shaped
     `waiting.json` gets its `ready.json` written by the chop.
   - A failed shell member that is the newest of its kind still blocks (all existing
     fail-closed tests must keep passing unchanged).
   - A newer shell member of a _different_ kind (e.g. a gate after a failed monitor)
     does not supersede.
   - Gate symmetry: a start-failed gate member superseded by a newer gate member
     resolves the same way.

### Monitor start stops minting doomed members and stealing live claims (`monitor-start-claim`)

In `src/sase/monitor/start.py` and `src/sase/monitor/start_claim.py` (scope: the
monitor-start lane-claim path only; do not change `transfer_workspace_claim()` semantics
for other callers such as the agent-runner retry-transfer flow):

1. **Pre-flight claim feasibility before member creation.** In
   `_start_monitor_locked()`, before `create_monitor_member()`, when the resolved
   `_LaneStart` will claim a nonzero workspace: read the current claim rows
   (`get_claimed_workspaces`) for that workspace. If a claim row exists whose pid
   differs from the intended `transfer_from_pid` and whose process is alive, raise
   `MonitorError` with the same conflict detail `_monitor_claim_error()` produces today
   — **without** creating the member artifact. A doomed start must not mint a
   permanently-failed family member. The existing `after_ack` claim plus
   `_teardown_failed_member()` stay as the authoritative racy backstop.
2. **Stale transfer-pid resolution.** `_resolve_lane_start()` derives
   `transfer_from_pid` from the selected member's `agent_meta.json` `pid`, which goes
   stale once the plan chain hands the claim to a gate shell or a successor runner
   (evidence above: `--plan` meta pid 1556836 vs. actual holder 3917772). When the lane
   workspace's current claim row has a matching `cl_name`/lane but a different pid than
   the meta suggests: transfer from the current row's pid only if that process is
   **dead** (adopting an orphaned claim); if it is alive, fail fast via the pre-flight
   above. Never transfer a claim away from a live process in this path — the 17:10:10
   ledger entry (claim taken from still-running `--1`) is a bug to eliminate, not
   behavior to preserve.
3. **Truthful no-op release.** `undo_monitor_claim()` → `_release_monitor_claim()` must
   not record a ledger mutation with `success=True` and identical before/after when
   nothing matched; record it as a no-op (keep it non-raising).
4. Tests in `tests/monitor/` (`test_monitor_start.py`,
   `test_monitor_start_conflicts.py`, `test_monitor_start_teardown.py`,
   `test_monitor_start_ack.py`): pre-flight conflict with a live holder raises before
   any member artifact exists; orphaned-claim adoption transfers from the dead holder's
   row; live-holder claim is never transferred away; the ack-path teardown behavior is
   unchanged for races that pass pre-flight; no-op release ledger shape.

### Surface permanently blocked waiters instead of only logging (`terminal-block-notify`)

In `src/sase/scripts/sase_chop_wait_checks.py` plus `src/sase/notifications/`:

1. When a waiting marker's blocked dependencies include terminal blockers (the existing
   `_terminal_blockers()` result), upsert a notification via the notifications store
   (`sase.notifications.store.upsert_notification`, following the existing kind/catalog
   patterns in `sase/notifications/catalog.py` and `models.py`) identifying the waiting
   agent, the blocking dependency name, the blocking artifact dir, and its outcome. Key
   the upsert per waiter artifact so repeated chop runs (every ~60s) do not spam the
   inbox; a plus-one/occurrence count is acceptable but a stable single entry per waiter
   is required.
2. Include actionable guidance in the notification body: the wait will never
   self-resolve; the operator can kill/relaunch the waiter or intentionally clear the
   wait.
3. Tests in `tests/test_axe_chop_wait_checks.py`: a terminally-blocked waiter produces
   exactly one inbox entry across repeated chop runs; a waiter that later resolves does
   not produce one; the notification carries the blocker identity.

## Verification

Each phase runs the standard repo verification (`just check` during development; the
landing gate runs `just check-full`). The reproduction tests in `wait-supersession` are
the acceptance test for the incident: the exact artifact shapes from the `sase-zt.6.5.3`
family must yield a resolved family wait.
