---
tier: epic
status: done
title: Make approved-plan persistence single-writer and epic launches source-swap safe
goal:
  Approved tales produce exactly one canonical plan commit before their runner resumes,
  artifact-link finalization cannot be poisoned by a competing plan writer, and an
  approved epic waits safely through an in-progress developer update instead of ending
  as a failed launch with no work started.
phases:
  - id: archive-ownership
    title: Make plan approval one atomic publication boundary
    depends_on: []
    size: medium
    description:
      "archive-ownership: establish one host-owned canonical plan archive operation,
      publish its durable path in the terminal gate response before a planner can
      resume, retain an explicit compatibility fallback for genuinely old responses, and
      prove that concurrent approval and runner writers cannot recur."
  - id: swap-safe-epic-launch
    title: Hold approved epic launches through developer source swaps
    depends_on: []
    size: medium
    description:
      "swap-safe-epic-launch: keep direct bead work fail-fast while making the detached
      host-owned epic launcher wait outside the editable SASE import boundary, then
      execute exactly once from a fresh process under the existing code-swap reader
      protection."
  - id: reliability-integration
    title: Prove the combined approval-to-launch lifecycle
    depends_on:
      - archive-ownership
      - swap-safe-epic-launch
    size: small
    description:
      "reliability-integration: exercise tale approval, sidecar link finalization, epic
      approval during a developer update, and exhaustive repository checks together so
      neither fix leaves a false failed agent or a duplicate durable mutation."
proposed_by: bbugyi200.athena.0an
bead_id: sase-s2
create_time: 2026-09-09 19:51:03
---

- **PROMPT:**
  [prompts/202608/plan_approval_launch_reliability.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/plan_approval_launch_reliability.md)
- **BEAD:**
  [sase-s2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-s2/README.md)

# Plan: Make approved-plan persistence single-writer and epic launches source-swap safe

## Confirmed diagnosis

The supplied screenshot is evidence of a real duplicate-writer defect, but the two named
agents did not fail for the same reason.

For `0aj`, `sase plan propose` completed and the tale was approved. The plans sidecar
then recorded two sibling commits from the same base for the same new path:

- `14190c58` (`Archive approved plan family_shell_metadata`) was made by the host
  approval archive operation. It stamped `create_time: 2026-08-22 11:33:24` and
  `status: wip`.
- `ba513715` (`Add SDD files for family_shell_metadata`) was made two seconds later by
  the resumed planner/implementation runner. It independently stamped
  `create_time: 2026-08-22 11:33:26` and `status: wip`, and projected the prompt link.

This is not two copies of one function accidentally running in a single process. It is
two distinct lifecycle owners in separate leased clones. The neutral gate executor
currently writes its terminal `response.json` before the plan adapter runs approval side
effects. The blocked runner sees that response immediately; because `saved_plan_path`
has not yet been appended, it takes the compatibility fallback in
`run_agent_exec_plan_accept.py` and writes the plan itself. The August 20
`saved_plan_path` single-writer change therefore has a check-then-publish race: it works
only when the late response rewrite wins before the poller reads the already-terminal
file.

The differing `create_time` values make the race visible, but preserving one timestamp
or adding a merge driver would only hide the ownership bug. The divergent local plan
commit later made `0aj`'s artifact-link commit repeatedly fail to rebase. Its commit
finalizer consequently reported that `links/202608/family_shell_metadata.md.json` had
vanished without an attributable commit. The implementation itself was not lost: the
primary-repository change later landed as `015557337`. Thus `0aj` was a false failed run
after successful implementation, caused by the poisoned plans-sidecar history rather
than by failure to launch the coder.

For `0al`, no plan conflict was involved. Its host-owned epic monitor started the
canonical `sase bead work ...restore_github_actions.md` command while `sase dev update`
held the exclusive code-swap lock. The monitor exited with code 1 after about two
seconds, and its saved response says, correctly, that no work was started. The update
had begun before the monitor and completed roughly forty-six seconds after it failed.
The fail-fast reader guard is intentional and must remain: a process that has already
imported editable SASE code must not wait and continue across a source-tree swap. The
missing behavior is at the detached host launcher, which should defer creation of a
fresh `sase bead work` process until the writer lock is released rather than treating
that transient safety condition as a terminal epic-launch failure.

## Required invariants

- A current-generation tale approval that includes plan persistence has one canonical
  writer. The runner consumes the published path and never races the approval host.
- A success response is not observable by the waiting runner until every durability
  result needed to interpret that response, especially `saved_plan_path`, is final.
- Absence of a late field is not used to infer ownership. Truly old response formats
  retain an explicit, tested runner-owned compatibility route.
- Archive failure is represented deliberately. It must not silently publish a success
  response that causes a second writer to repair the host's work behind its back.
- `create_time` describes one plan creation event and is stable across replay;
  `status: wip` is inserted once without duplicate or clock-dependent rewrites.
- Direct, user-invoked `sase bead work` remains fail-fast during a developer update.
  Only the detached host-owned epic launch may wait, and it waits before importing the
  editable SASE package.
- A deferred epic launch executes exactly once, retains its logical command/reason and
  workspace claim, remains visible as pending/running, and still respects the existing
  launch timeout and active-plan deduplication.

## Phase `archive-ownership` — Make plan approval one atomic publication boundary

Refactor the plan-specific notification-gate lifecycle so durable archive preparation
and response publication form one ordered transaction. The generic executor in
`src/sase/notification_gates/executor.py` currently persists the response and only then
calls `adapter.apply_side_effects`; introduce the narrow pre-terminalization hook or
equivalent plan-adapter split needed to prepare durability-derived response fields
before the exclusive response write. Keep unrelated gate kinds on their existing
semantics unless the new hook is a no-op for them.

Split the plan adapter's responsibilities in `src/sase/notification_gates/adapters.py`
and `src/sase/plan_approval_actions.py`:

1. Validate the selected tale/epic action, synchronize reviewed plan content, persist
   approval metadata, and run the single host archive operation when the selected choice
   promises plan persistence.
2. Put the resulting canonical path into the in-memory primary option result (and the
   translated runner protocol) before `response.json` becomes visible. Write that file
   once atomically; remove the late read/modify/write response patch as an ownership
   mechanism.
3. Leave notification dismissal, UI refresh, and host-owned epic-launch submission as
   post-terminal effects only when they do not supply data the runner needs. Preserve
   idempotence and the notification-gate partial-attempt/retry journal contract.

Make archive failure fail closed for a selected commit operation: retain the existing
reset-and-replay recovery and error notification, but do not emit a successful terminal
response without an archive result. A retry must resume or restart through the gate's
recorded attempt rules without creating another canonical plan. Rejection, feedback, and
tale approval without the commit option must remain fast and must not archive.

Apply the same ownership rule to the legacy plan-response path in
`src/sase/ace/tui/actions/agents/_notification_modals.py` and
`_notification_plan_background.py` without blocking Textual's message pump. Either run
archive-plus-response as one session worker operation or designate the old runner as the
sole writer for that explicitly versioned compatibility path; do not retain a path where
ACE publishes an archive-less success and then races a background archive against the
runner.

Harden the runner contract in `src/sase/llm_provider/_plan_utils.py` and
`src/sase/axe/run_agent_exec_plan_accept.py`. Current responses must carry explicit
archive ownership/state and a verified canonical path. The runner-owned write remains
only for responses positively identified as predating the host-owned protocol, not for
any current response that happens to omit a path. Validate that the saved path belongs
to the resolved plan store and exists before treating the plan as committed.

Stamp managed `create_time`/`status` at one stable boundary and make replay preserve the
source or existing canonical creation time rather than calling the clock again. Keep
`archive_plan_file` idempotent, including reviewed edits and prompt-link projection;
this is defense in depth and not a substitute for the single-writer protocol.

Add deterministic tests that pause the host archive after the option commands have
completed. While paused, assert that the gate poller cannot observe a terminal success
and the runner cannot enter `write_sdd_files`. After release, assert one response write,
one `Archive approved plan ...` commit, a populated canonical path, no
`Add SDD files ...` sibling commit, stable managed frontmatter, and idempotent replay.
Cover ACE, headless/CLI or auto approval, and the explicitly supported legacy branch.
Include a hermetic two-clone sidecar regression modeled on the `0aj` chronology and
prove that a subsequent artifact-link commit rebases/publishes without conflict.

Acceptance criteria:

- No current approval surface can expose a successful commit-bearing response before the
  canonical archive result is present.
- The `0aj` race fixture produces one reachable plan commit and identical plan bytes in
  every consumer clone.
- Archive errors are retryable gate failures, not successful responses followed by a
  runner fallback.
- The coder/commit-only follow-up still starts with the canonical plan, and rejection,
  feedback, no-commit approval, and historical response compatibility remain covered.

## Phase `swap-safe-epic-launch` — Hold approved epic launches through source swaps

Keep `code_swap_reader_lock`'s fail-fast behavior in
`src/sase/dev_update/code_swap_lock.py` and the direct CLI regression in
`tests/test_bead/test_cli_work_code_swap_lock.py`. Build a separate guarded-exec path
for host-owned epic launches: a tiny stable bootstrap must acquire the same shared lock
before importing SASE, then `exec` a fresh canonical `sase bead work` argv while the
shared lock remains held. Waiting inside an already-imported `sase bead work` process or
retrying by matching human-readable stderr is not acceptable.

Wire the guarded execution into `src/sase/bead/epic_launch.py` for both the family
monitor path and the leased fallback proc path. If monitor/proc request data needs to
distinguish the logical command from its execution argv, extend that internal contract
so agent metadata, labels, reasons, active-launch deduplication, fingerprints, restart
hints, and user-facing resume commands continue to show the canonical `sase bead work`
command rather than bootstrap machinery. Persist enough execution intent that proc
reconciliation cannot restart an unguarded version after a supervisor interruption.

The monitor should be acknowledged promptly and remain in its `EPIC APPROVED` running
state while the update owns the writer lock. Once released, the fresh process must
acquire reader protection and create the epic exactly once. Existing four-hour timeout,
workspace claim transfer, active-plan deduplication, host origin, prompt-snapshot, and
artifact-directory attribution must survive unchanged. A genuinely failing post-update
`sase bead work` command remains a failed monitor with its real output; only lock
contention is deferred.

Add unit coverage for guarded argv construction, logical-versus-execution command
serialization, request fingerprinting, monitor and fallback-proc parity, and restart
reconstruction. Add a subprocess-level regression that holds the writer lock, submits an
approved epic launch, verifies that the monitor stays active and no bead/plan mutation
occurs, releases the writer, then observes one successful epic creation. Keep the direct
CLI test proving an ordinary `sase bead work` invocation still exits before mutation
while the writer is held.

Acceptance criteria:

- Replaying the `0al` timing leaves one running/pending monitor during the update and
  creates the epic after the update finishes; it never records the transient guard as a
  terminal failed agent.
- No editable SASE module is imported by the waiting bootstrap before the writer lock is
  released.
- Direct bead work remains fail-fast, and real launch failures remain visible and
  actionable.

## Phase `reliability-integration` — Prove the combined approval-to-launch lifecycle

Build one end-to-end regression that submits and approves a tale while deliberately
delaying archive publication, resumes its coder, records an artifact link, and verifies
that every relevant repository has a linear, published history with no conflict and no
false finalizer failure. Build the corresponding epic journey with the code-swap writer
held across approval: the response may complete, the launch monitor must wait safely,
and exactly one epic/phase DAG must appear after release. Assert durable agent-family
states and saved transcripts so a successful implementation cannot be reported as failed
merely because plan-sidecar bookkeeping raced.

Review the final diff for accidental changes to generated plans, agent artifacts,
sidecar repositories, or user work. Run `just install`, the focused gate/approval,
plan-archive, finalizer, code-swap, monitor, and epic-launch suites, then run
`just check-full` through `/sase_monitor` with `TESTING`/`TESTED` statuses because this
is an epic's combined tree and changes the shared gate executor. Re-run the two
deterministic race tests several times with inverted scheduling to ensure they prove
ordering rather than merely winning it on one machine.

Acceptance criteria:

- Focused tests reproduce both historical failures before their owning fix and pass
  afterward.
- The combined lifecycle has one canonical plan writer, one epic launcher, no merge
  conflict, no discarded-dirty-work diagnostic, and no terminal failure for transient
  source-swap contention.
- Monitored `just check-full` passes on the combined tree.
