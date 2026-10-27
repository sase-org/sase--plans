---
tier: tale
title: Finish stranded-wait alert reconciliation and land sase-1ok
goal:
  Resolved stranded-bead alerts dismiss after every waiter release path, and sase-1ok
  closes with verified integration and follow-up dispositions.
size: medium
proposed_by: bbugyi200.athena.sase-1ok.land
proposed_with: codex/gpt-6-astra
bead: sase-1ok
create_time: 2026-10-26 22:13:45
status: wip
---

- **PARENT:**
  [202610/stranded_phase_detection_and_wrong_bead_guard.md](https://github.com/sase-org/sase--plans/blob/main/202610/stranded_phase_detection_and_wrong_bead_guard.md)
- **BEAD:**
  [sase-1ok](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ok/README.md)

# Finish stranded-wait notification reconciliation and land sase-1ok

Complete the remaining work in epic **sase-1ok**. This tale's coder must finish the epic
landing in the same turn as the fix: a tale has no land agent, and no earlier lander
resumes afterward. Do not wait for this work's own commit, push, SHA, or CI result
before closing. The host commits after the coder's turn.

## Verified scope and remaining defect

The parent epic plan is `plan:202610/stranded_phase_detection_and_wrong_bead_guard.md`;
its file is `sase/repos/plans/202610/stranded_phase_detection_and_wrong_bead_guard.md`.
All four phase beads are closed. The landing audit reviewed every phase note and these
implementation commits: `538736f0f2` (closed-parent stamping), `98cb07ebb8` (single-turn
collector), `9cb35859c9` (assigned-ID guard), and `fabfff06e0` (waiter alerts), plus
sase-core `03dce1f2` (AgentName filter). The core revision pin includes that filter. The
approved decision remains **near_miss_mutation = refuse**; preserve it and add no
override flag.

The audit also reviewed concurrent changes through fetched `e7d9f7a479`: Codex shadow
sessions, publication retry/protection, failed-close recovery, published-core pinning,
and monitor-capacity projections. They do not replace the new collector or CLI guard.
Recheck later commits for relevant drift before closing. Failed-close recovery can
reopen beads after a failed host finalizer; continue to derive stranded state through
the existing Rust reducer.

The late `e7d9f7a479` capacity-scan adoption was fast-forwarded during the audit. Its
wire-comment change preserves AgentName and the revision pin. A focused run of its four
capacity-related test files passed 36 tests with one pre-existing fixture failure,
recorded on its owning epic `sase-1oj` (details below).

The seven focused test files listed below passed **89 tests** in the audit. However, the
new alert reconciliation has a reproducible release-path hole:

1. Create a completed single-turn phase, leave its bead in progress, and put a live
   waiter on that bead. One tick creates one stranded-bead notification.
2. Close the bead and put `ready.json` on the waiter before the next tick.
3. The next tick skips the already-ready waiter and leaves the alert active.
4. Remove `waiting.json` and tick again: the alert still remains.

The audit reproduced counts **1 -> 1 -> 1**, using the existing stranded/wait test
helpers and a temporary isolated SASE home. Expected counts are **1 -> 0 -> 0**. This is
reachable when another writer releases the waiter, and when
`src/sase/axe/run_agent_wait.py` resolves dependencies through its runner fallback and
removes its waiting marker.

`_record_stranded_bead_waits` only records pending live waiters;
`_process_stranded_bead_waits` only evaluates those recorded projects and IDs. Existing
notifications therefore lose their reconciliation target when their waiters stop being
pending. The current close test only exercises release by the same `wait_checks` tick,
so it misses this case.

## Implementation

1. Read `sase-1ok` and its landing notes, and the approved parent plan through audited
   reads. Inspect these files:
   - `src/sase/scripts/_chop_wait_checks_run.py`
   - `src/sase/scripts/_chop_wait_checks_run_waiters.py`
   - `src/sase/scripts/_chop_wait_checks_stranded.py`
   - `src/sase/scripts/_chop_wait_checks_terminal.py`
   - `src/sase/bead/stranded_facts.py`
   - `tests/test_axe_chop_wait_checks_stranded_beads.py`
2. Seed reconciliation targets from existing undismissed `wait_checks` stranded-bead
   notifications as well as this tick's pending bead waits. Group the union by project
   and bead ID. Use the canonical `wait_checks:stranded-bead:<project>:<bead_id>`
   identity; ignore unrelated or malformed notification keys. A released/dead/removed
   waiter must not prevent evaluating an existing notification's bead.
3. Query closure for those exact IDs through the existing Rust-backed bead-read facade
   (for example `statuses_for_ids`), including when an ID is absent from the collector's
   in-progress candidates. Use the existing targeted collector and Rust reducer for
   owned/delegated/settling/stranded state. Dismiss existing notifications only on
   positively established closed, owned, delegated, or settling state. Unknown/missing
   status or a collection/read failure is not proof of resolution; leave that project's
   rows intact and log the failure.
4. Preserve one targeted collection per project per TTL, with sorted-ID-set and
   store-fingerprint invalidation. Keep the target set bounded to pending waits plus
   outstanding notifications; do not restore a full-project artifact scan on each
   two-second tick. Reuse loaded notifications and status facts where practical. With
   neither pending waits nor outstanding rows, do no extra stranded collection. New
   alerts still require actual blocked live waiters; reconciliation alone must not
   create an alert or release/park a runner.
5. Add behavioral regressions for already-ready and removed waiting markers, and for an
   outstanding row whose waiter died. Cover notification-only reconciliation to
   owned/delegated/settling, closed beads absent from reducer candidates, retention
   after collection/status failures, and no work when there are neither pending waits
   nor active alerts. Preserve the TTL test and assert that reconciliation targets only
   the required IDs across projects.

Keep this as Python orchestration over existing Rust-backed status and stranded
classification APIs. Do not duplicate the reducer or introduce a new backend policy in
Python. If a core API change proves necessary, open `sase-core` via `/sase_repo`, obey
its instructions, test both repos, and include its pin update.

## Verification and final landing

1. Run the focused tests, including the added release-path regressions:
   `tests/test_axe_chop_wait_checks_stranded_beads.py`,
   `tests/test_stranded_single_turn.py`, `tests/test_stranded_collector.py`,
   `tests/test_stranded_collector_statuses.py`,
   `tests/test_bead/test_assigned_near_miss.py`,
   `tests/test_bead/test_cli_assigned_near_miss.py`, and
   `tests/test_plan_command_handler_associations.py`. Read `lint_and_test.md` through
   `/sase_memory_read`, run `just fix`, and run `sase tool run check` (the guarded
   `just check`). Do not run `just check-full`. If a command must hand off, use
   `/sase_monitor` and carry these closeout instructions forward.
2. Recheck source and relevant commits landed since `e7d9f7a479`. Reconfirm phase
   readiness and the parent plan's acceptance criteria. Consult the landing note for the
   completed live old/new collector comparison: `gh_bobs-org__bob-cli`
   unknown/no_records fell **8 -> 4** and `gh_sase-org__sase` fell **149 -> 18**. An
   independent enumeration of 3,030 bob-cli and 17,699 sase local artifact directories,
   reading names/sessions from meta and done markers, found no records for any of the
   remaining assignees and no unreadable JSON. These are genuinely missing local
   records; the precise IDs are in the epic note. Record these timestamped audit counts
   honestly rather than claiming them as a fresh sample.
3. Follow-up triage is already complete and recorded on sase-1ok: capacity proposals
   `sase-1ok.1#1`, `.2#1`, and `.3#2` are one issue attached as `DISCOVERED ISSUE` to
   causally responsible active epic `sase-1oj`. Its `e7d9f7a479` adoption fixed two of
   the three; the remaining
   `test_capacity_blocked_waiter_does_not_park_the_queue_behind_it` still uses
   nonexistent PIDs 100/101/102 and fails at line 90 (`drained is None`). That fresh
   clean-tree result is also recorded on `sase-1oj`; no duplicate task. Gateway proposal
   `.1#2` was corroborated on exact-node task `sase-1gz` (+9), and Machines-pane
   proposal `.3#1` on `sase-1ax` (+1). Do not duplicate these tasks or treat them as
   remaining sase-1ok work. Include every outcome in the final close note.
4. **Finish sase-1ok closeout in this turn.** Run `sase bead epic-symbols sase-1ok`.
   Resolve every listed symbol according to Symvision policy, or re-key an exemption
   only to a still-open later bead that actually needs it. The landing audit found no
   entries, but recheck. Close normally with
   `sase bead close sase-1ok --note "<verification>"`, describing all four phase checks,
   integration review, release-path fix, test results, live collector evidence, and
   follow-up dispositions. Never use `--force` to make a successful landing advance;
   resolve stale symbols or unfinished phases instead. After the close, run
   `just symvision`. Open the plans sidecar with `/sase_repo` and set `status: done` in
   `sase/repos/plans/202610/stranded_phase_detection_and_wrong_bead_guard.md` using the
   printed repository path. Include that sidecar change in final declarations. Finally
   inspect `sase bead read sase-1ok -r "Need the parent link"`: this audit confirmed
   `parent_id: null`, so there is no ancestor to close. If that link has changed, follow
   the original land prompt's normal parent-readiness rules; do not force-close any
   ancestor. Submit `/sase_final` as the last normal action.
