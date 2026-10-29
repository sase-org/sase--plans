---
tier: epic
title:
  "Task backlog epic 3: publication safety, handoffs, check coverage, and state
  freshness"
goal:
  Fix and close the five selected fleet-costly task beads, then continue the backlog
  chain under the same ranking and closeout rules.
decisions:
  optional_scope:
    ask:
      Should the lander include adjacent feature or memory changes discovered while
      fixing these five bug and flake tasks?
    choices:
      keep_five_seeds:
        Limit work to the five bug/flake beads; defer adjacent feature or memory work
      required_scope_only:
        Include feature or memory changes only when required by a selected bead's
        acceptance
    default: keep_five_seeds
    why:
      The selected beads are defect work, so optional scope should remain under owner
      control.
    answer: keep_five_seeds
phases:
  - id: prompt-migration-protection
    title: Scan and safely recover prompt migrations into the agents sidecar (sase-1os)
    depends_on: []
    size: medium
    description:
      "prompt-migration-protection: route the one-time legacy prompt migration through
      the shared publication-protection scanner before staging, and stop recommending a
      raw push after a GH013 rejection. Preserve the existing private/quarantine
      behavior and sanitized diagnostics. Add regression coverage for clean migration, a
      detected secret, and a rejected push; ensure no secret value is emitted or
      replayed."
  - id: waiter-wake-recovery
    title: Keep long-waiting bead agents alive when their dependency wakes (sase-12v)
    depends_on: []
    size: large
    description:
      "waiter-wake-recovery: reproduce the wake-time NameCollisionError after a long
      wait, identify the reservation or retry failure, and make the wake path recover or
      retry safely without launching duplicate successors. Preserve the original
      dependency and workspace state. Review the related sase-1el monitor-start report,
      but do not assume a shared root cause from the exception class alone."
  - id: marker-audit-selection
    title: Select agent-artifact marker audits for marker-writer changes (sase-1mv)
    depends_on: []
    size: medium
    description:
      "marker-audit-selection: make scoped test selection include both AST audit modules
      when a changed source file writes a tracked agent-artifact marker, without
      exceeding the contract-set serial budget or weakening the audits. Add selector
      regression tests and show that a marker-writer change selects both audits."
  - id: sdd-identity-check-flake
    title: Remove the full-check timeout for the empty-home SDD identity test (sase-120)
    depends_on: []
    size: large
    description:
      "sdd-identity-check-flake: reproduce the repeated full-lane timeout and passing
      isolated run for test_sdd_git_identity_survives_empty_home_subprocess. Determine
      whether the nested subprocess or the temporary-directory leak guard causes the
      slow/failing result, then fix isolation or cleanup without weakening the test or
      simply extending its timeout. Verify the node, its file, and the scoped/full
      check."
  - id: published-run-state-freshness
    title:
      Publish terminal agent-run state instead of leaving thousands of rows active
      (sase-1cc)
    depends_on: []
    size: large
    description:
      "published-run-state-freshness: ensure a completed or failed run in a published
      hood eventually updates the agents-sidecar state snapshot. Choose a reliable
      terminal-publication or reconciliation path, account for concurrent hood
      publication, and provide a safe bounded backfill plan for existing stale rows.
      Verify terminal-state convergence and that live runs are not marked terminal."
status: wip
proposed_by: bbugyi200.athena.sase-1pn.land
proposed_with: codex/gpt-6-luna
decided_by: auto
create_time: 2026-10-29 19:18:57
---

- **PROMPT:**
  [prompts/202610/task_backlog_epic_3.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/task_backlog_epic_3.md)
- **PARENT:**
  [202610/task_backlog_epic_2.md](https://github.com/sase-org/sase--plans/blob/main/202610/task_backlog_epic_2.md)

# Task Backlog Epic 3: Publication Safety, Handoffs, Check Coverage, and State Freshness

## Context

This epic continues the fleet-wide task backlog after epic sase-1pn
(202610/task_backlog_epic_2.md), which closed after landing its five selected tasks. The
lander re-listed the non-closed task beads on 2026-10-29: 385 remain. No additional bead
was closed as a duplicate or obsolete because the current review did not establish the
required canonical link or a fixing commit/test for another candidate.

The five seeds below were selected for security, blocked work, check/CI reliability, and
published-state correctness:

| Bead     | Fleet-wide cost                                                                                                                   |
| -------- | --------------------------------------------------------------------------------------------------------------------------------- |
| sase-1os | A public-sidecar prompt migration bypasses the publication scanner and recovery advice could replay a GH013-rejected commit.      |
| sase-12v | A long-waiting bead agent can fail with NameCollisionError at wake and leave its phase stalled without retry.                     |
| sase-1mv | Scoped check omits marker-writer AST audits, allowing a green scoped check before deterministic Master Gate and Full CI failures. |
| sase-120 | A repeated full-check-only failure delays the full suite after tens of thousands of passing tests; the node passes in isolation.  |
| sase-1cc | A measurement found 8,659 of 15,457 published run-state records still marked active or waiting after more than two days.          |

The five tasks have no parent epic and no phase dependencies. sase-1el remains open: its
gate-turn half was fixed, while the monitor-start collision remains undiagnosed. It is
referenced by sase-12v for comparison, but this plan does not treat the shared exception
class as proof that the reports have the same cause. The separate historical
secret-purge task sase-1ot is excluded because its latest recorded gate outcome is
owner_reject; renewed remediation requires a fresh owner request.

> [!decision] optional_scope = keep_five_seeds
>
> Implement only the five listed bug/flake tasks. Leave adjacent feature and memory work
> open for later owner review.

> [!decision] optional_scope = required_scope_only
>
> Include a feature or memory change only when it is required by a selected task's
> acceptance; defer other adjacent work.

## Phase prompt-migration-protection: scan and safely recover prompt migrations into the agents sidecar (sase-1os)

Read sase-1os in full. The one-time prompt migration copies files into the public agents
sidecar without the shared publication-protection scanner. Its failure guidance can tell
the operator to re-push a commit that GH013 rejected.

- Scan migrated prompt content through the shared publication-protection adapter before
  staging or publishing.
- Route secret-rejected publication through the existing recovery path, or stop with
  sanitized guidance that never recommends replaying the rejected commit.
- Preserve clean migration behavior and the existing private/quarantine safeguards.

Acceptance: tests cover a clean migration, a detected secret withheld from publication,
and a GH013 rejection that does not advise or execute a raw re-push. Logs and artifacts
contain no matched secret values.

## Phase waiter-wake-recovery: keep long-waiting bead agents alive when their dependency wakes (sase-12v)

Read sase-12v and inspect its related task sase-1el. Reproduce a waiter waking after a
long delay and failing with NameCollisionError. Find the source of the collision and
determine whether it is a stale reservation, duplicate successor name, or another
launch-lifecycle condition.

- Make the wake transition recover or retry safely after a transient collision, while
  ensuring only one successor agent starts.
- Preserve the waiter's dependency, project, and workspace context across recovery.
- Keep the fix scoped to evidence from the waiter path; handle the separate sase-1el
  monitor-start report only if a shared cause is demonstrated.

Acceptance: regression tests cover the collision, successful retry/recovery, no
duplicate successor, and the ordinary non-collision wake path.

## Phase marker-audit-selection: select agent-artifact marker audits for marker-writer changes (sase-1mv)

Read sase-1mv in full. The marker audits parse src/sase text rather than importing the
writer functions, so the current import-graph selector misses them. Adding both audits
to the contract set exceeds its 30-second serial budget.

- Select both AST audit modules when a changed source file can write a tracked marker,
  or provide an equally reliable bounded selection mechanism.
- Keep the audits unchanged and keep the contract-set budget.
- Add a selector regression proving that a representative marker-writer change selects
  both audits, and that unrelated changes do not broaden the suite unnecessarily.

Acceptance: the scoped check selects the audits for marker-writer changes and reports
their failures before landing; the contract manifest budget remains within its limit.

## Phase sdd-identity-check-flake: remove the full-check timeout for the empty-home SDD identity test (sase-120)

Read sase-120 and its corroborating evidence. The node failed in repeated full just
check lanes after more than 42,000 tests passed, then passed in isolation and as part of
its file. One report also implicated the system temporary-directory leakage guard after
the nested test itself passed.

- Reproduce the full-lane failure and distinguish subprocess runtime from temp-leak
  guard failure.
- Fix the identified fixture or cleanup contention without weakening assertions or
  increasing timeouts solely to hide contention.
- Confirm the exact node, its file, and the scoped/full check on the resulting tree.

Acceptance: the nested identity test and cleanup guard pass without a full-lane-only
failure; no broader timeout relaxation is introduced.

## Phase published-run-state-freshness: publish terminal agent-run state (sase-1cc)

Read sase-1cc and its corroborating measurements. Published agents/\*/state.json records
are generally snapshots taken while a hood is still running; without a later
publication, terminal state remains stale for history readers.

- Choose a reliable path to publish terminal transitions or reconcile published hoods,
  accounting for concurrent publication and sidecar write ownership.
- Define a bounded backfill or convergence procedure for existing stale records; do not
  rewrite history or publish unverified terminal outcomes.
- Add tests for completion/failure convergence, a still-live run, and concurrent
  publication.

Acceptance: an eligible terminal run converges to its terminal state on the published
sidecar, live runs remain live, and the repair procedure does not overwrite a newer
snapshot.

## Landing

Each phase records TASK RESOLVED: <task-id> — <actual evidence> on its phase bead. The
lander re-reads the epic and all task/phase beads, verifies the resolved claims, closes
the task beads normally with test or commit evidence, and closes the epic normally with
the actual follow-up triage. Do not use --force to advance a successful landing. Run
just symvision after the epic closes and resolve any remaining symbols before marking
this plan done.

Continue the backlog chain while non-closed task beads remain:

1. Re-list every non-closed task bead (`sase bead list -t task -n 0`). Close only beads
   you can verify are duplicates (resolution `superseded`, naming the canonical bead,
   with a note on the canonical) or obsolete (resolution `done`, citing the fixing
   commit or the grep that proves absence, or a passing focused test). Never close a
   bead an active epic owns. Never close a snoozed bead early. Beads that are merely
   dormant stay open.
2. Pick the 5 most important remaining task beads. Rank them by fleet-wide cost first
   (this means: breaks every agent's check or CI, breaks or wastes every landing,
   security or data-loss risk, heavy +1 corroboration), then by cost to fix.
3. Write an epic plan with at most 5 phases and at most 5 task beads. One phase may work
   several beads. Use plan decisions sparingly, but always give the owner a way to opt
   out of feature work and memory changes that might not be wanted.
4. That epic's Landing section must carry this same "continue the backlog chain" rule,
   until no non-closed task beads remain.
5. Propose it with `/sase_plan`. Proposing ends the land agent's turn, so do it last.
