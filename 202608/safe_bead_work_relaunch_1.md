---
tier: tale
title: Preserve active bead workers during sase bead work retries
goal:
  Rerunning sase bead work replaces only waiting or failed owners that are
  authoritatively associated with this invocation's beads, while preserving active work
  and launching only the missing retry segments.
size: medium
proposed_by: bbugyi200.athena.02v
create_time: 2026-09-09 20:00:28
status: wip
---

# Make `sase bead work` retries state- and bead-aware

## Goal

Make `sase bead work` a safe recovery operation. A retry must never terminate an agent
that is actively doing work, and it must never wipe an agent merely because its name
collides with a deterministic phase, land, or task name. It may replace a waiting or
failed owner only when that owner's durable bead association matches the exact bead for
the launch role being retried. Any surviving active owner must remain in place, and the
command must launch only the roles that still need a worker.

## Current behavior and root cause

- `launch_epic_bead_work` in `src/sase/bead/cli_work_handler.py` always renders every
  non-closed phase plus the land agent, then passes all deterministic names to
  `preview_bead_work_force_reuse` and `prepare_bead_work_force_reuse`.
- `src/sase/bead/cli_work_cleanup.py` classifies every live exact-name or family-member
  collision as `KILL`. Its cleanup contract explicitly wipes old owners “regardless of
  state,” and `wipe_agent_name_for_reuse` in `src/sase/agent/names/_wipe.py` sends
  `SIGTERM` to every live artifact in the transitive wipe plan before deleting the
  artifacts. Neither path checks `agent_meta.json::bead_id`.
- The prompt already contains the authoritative mapping needed for this decision: every
  forced `%id(..., bead=...)` identifies one concrete owner name and one phase, epic, or
  task bead. `force_reuse_bead_associations_by_prompt` already parses that mapping with
  the normal directive, clan, family, protected-region, and identity rules. Epic
  metadata also records `bead_id`, `epic_bead_id`, and `phase_bead_id` in each agent
  artifact.
- The Rust-backed artifact scanner already exposes agent metadata, `waiting.json`, and
  `done.json` for exact artifact directories. The Rust bead mutation already accepts a
  partial phase-assignment list and an optional land assignment. No new `sase-core` wire
  or mutation is required; the missing behavior is the Python bead-work launch
  transaction that selects and cleans a retry set.
- Standalone tasks avoid one common duplicate by returning early for any live assignee,
  but that test does not distinguish `WAITING` from `RUNNING`, and the later
  deterministic-name wipe still has no bead-association guard. The common cleanup
  contract must therefore protect both epic and task entry points.

## Safety contract

Resolve every current launch role to an expected `(owner name, bead ID)` pair before
performing cleanup. For the land role, also recognize the legacy exact land name
`<epic_id>` as an alias for `<epic_id>.land`; a populated `<epic_id>` clan container is
the joinable epic clan, not a legacy land agent.

Apply this disposition matrix to the newest concrete owner generation and, for a family,
to the aggregate of all concrete members that the wipe would affect:

| Existing owner state                                                          | Matching current bead association | Disposition                                                                   |
| ----------------------------------------------------------------------------- | --------------------------------- | ----------------------------------------------------------------------------- |
| No owner                                                                      | n/a                               | Launch the role.                                                              |
| Orphaned reservation with no concrete artifacts                               | n/a                               | Release the reservation, then launch.                                         |
| Live `WAITING`/runner-queued owner                                            | yes                               | Preview `KILL`, revalidate, terminate, wipe, and relaunch.                    |
| Terminal `FAILED` or dead/interrupted failed owner                            | yes                               | Preview `REMOVE`, wipe without signaling, and relaunch.                       |
| `RUNNING`, `STARTING`, plan/question input, or another non-waiting live state | yes                               | Preserve the owner and omit this role from the retry.                         |
| Successful `DONE` owner                                                       | yes                               | Preserve it; do not silently erase successful artifacts or relaunch the role. |
| Any concrete owner                                                            | missing or different bead         | Fail before cleanup, bead mutation, checkpoint, or launch.                    |

Only a live waiting process may receive a termination signal from this workflow. A
failed owner is terminal/dead and is removed without signaling. Status labels should be
derived from the existing snapshot/status helpers so queued waits, family handoffs,
failure outcomes, and current-owner name normalization agree with agent-list behavior;
do not invent a second user-visible status system.

For a family-container collision, preserve the whole launch role if any member in the
current generation is active in a non-waiting state. Replace the family only when every
live member is waiting and all members that will be wiped carry the expected bead, or
when the current generation is failed/dead and carries the expected bead. Never kill a
waiting family member as part of a wipe that would also destroy a running member.

## Implementation

1. Add a read-only, association-aware owner inspection layer to
   `src/sase/bead/cli_work_cleanup.py`.
   - Represent the current invocation's roles explicitly, including the canonical launch
     name, expected bead ID, and any legacy alias. Derive epic mappings from the full
     `EpicWorkPlan` and task mappings from the task bead/current assignee rather than
     inferring a bead from a dotted name.
   - Resolve registry owners, exact agents, current family generations, and the joinable
     epic clan as today, but scan the concrete artifact directories through
     `sase.core.agent_scan_facade.scan_agent_artifact_dirs`. Require the concrete
     owner's persisted `bead_id` to equal the expected bead. When the bead store has a
     non-empty current assignee, also reject a conflicting assignee instead of treating
     a same-name artifact as authorization.
   - Return a structured selection containing destructive preview targets, roles to
     launch, roles preserved by an existing owner, stale reservations to release, and
     any hard ownership/status conflict. Keep preview rendering read-only and show
     enough state/bead detail for the user to understand why a role is replaced or
     preserved.
   - Validate the prompt's parsed force-reuse `(name, bead)` associations against this
     selected launch mapping. A malformed, duplicate, beadless, extra, or missing
     directive must remain a pre-mutation error.

2. Make the destructive wipe honor the selection at the last responsible moment.
   - Extend the name-wipe path in `src/sase/agent/names/_wipe.py` with an optional
     bead-work guard (or an equivalent non-mutating wipe-plan preflight) while
     preserving the existing unrestricted behavior for other explicitly confirmed
     force-reuse surfaces.
   - Before signaling or deleting anything, inspect every concrete artifact in the
     transitive wipe plan and require the expected bead plus the approved waiting or
     failed disposition. For a live process, re-read its metadata, liveness, and
     `waiting.json` immediately before `SIGTERM`; if it is now running, its bead
     changed, its PID identity changed, or the artifact generation no longer matches,
     abort without signaling or deleting that artifact.
   - Prevalidate the whole family/batch before the first wipe so a later conflict does
     not cause avoidable partial cleanup. If state changes after preview, be
     conservative: an owner previously selected for preservation stays preserved for
     this invocation, while an owner selected for replacement that is no longer eligible
     aborts the retry before any bead mutation. The user can rerun against a fresh
     snapshot.
   - Keep stale container release and populated-clan handling, but do not use them to
     bypass concrete-owner bead checks. Update cleanup errors and docstrings to state
     the guarded contract instead of claiming that live owners are always replaced.

3. Project an epic retry down to only the selected launch roles.
   - Add selection-aware helpers in `src/sase/bead/work.py` (or a small adjacent module)
     so prompt rendering, `epic_work_segment_env`, expected names, phase preclaims, and
     the optional land preclaim all use one ordered selection. Do not independently
     filter those collections at multiple call sites.
   - Omit preserved phase and land segments from the rendered multi-prompt, but retain
     their original name and bead dependencies in the waits of relaunched dependents. A
     relaunched land agent must still wait on every non-closed phase name and bead,
     including phases whose running owners were preserved.
   - Treat the first _selected_ segment as the first launch segment. It must declare a
     new epic clan when none exists, or join the existing epic clan when one exists;
     land-only retries and Patch-backed launches must receive the same correct first
     segment VCS/`#pr` prefix and clan-summary environment as an ordinary first phase.
   - In `launch_epic_bead_work`, perform guarded cleanup before snapshot, readiness,
     preclaim, or checkpoint mutations. Preclaim only selected phases and pass the land
     name only when the land role is selected. Preserve existing assignments and avoid
     redundant bead events for running roles. Rollback only the preclaims made by this
     invocation.
   - If every role is preserved, return a successful no-op without snapshotting,
     preclaiming, committing/pushing the graph, or launching a runner. If only some
     roles are selected, checkpoint and launch that partial set normally.

4. Make epic results describe what actually happened.
   - Introduce a structured epic-work result parallel to `TaskWorkResult`, carrying the
     launch state, actual launched names, and preserved names, with a `launched`
     convenience property. Update the bead-ID CLI, linked-plan resume path,
     newly-created-plan path, human summaries, and JSON envelopes to consume it.
   - Report an all-preserved invocation as an idempotent success and a partial retry as
     launching only its selected names. Do not claim that every phase and land agent
     launched merely because they were present in the full work plan.
   - Preserve existing dry-run and confirmation semantics: dry run performs only the
     read-only classification and prints the projected prompt; `--yes` still does not
     approve destructive waiting/failed cleanup, while `--yes-to-all`/JSON may do so.

5. Apply the same guard to standalone task retries without weakening contention.
   - Replace the task path's “any live assignee means already running” shortcut with the
     same state-aware disposition: preserve a matching running/input-active assignee,
     replace a matching waiting or failed assignee, and reject missing/different bead
     metadata.
   - If an older task assignee name differs from the canonical task launch name, guard
     and clean that actual assignee before launching the canonical name. Independently
     reject a canonical-name owner associated with another bead. Keep task status,
     preclaim, checkpoint, rollback, feedback, size/model routing, and one-result
     validation unchanged.

6. Add focused regression coverage.
   - Add cleanup-selection tests using real marker-shaped snapshots for matching
     `RUNNING`, `STARTING`, `WAITING`/queued, `QUESTION`/plan wait, `FAILED`, `DONE`,
     dead/stale, beadless, and wrong-bead owners. Assert only matching waiting owners
     reach `SIGTERM`, only matching waiting/failed owners are wiped, and every mismatch
     fails before mutation.
   - Cover current family generations with completed planner plus running, waiting, or
     failed coder members; mixed running/waiting families; stale family/clan
     reservations; the populated epic clan; and legacy exact land owners. Verify a
     running legacy land owner suppresses the new land segment rather than allowing a
     duplicate.
   - Add race regressions where a previewed waiting owner becomes running, changes bead,
     or is replaced by another generation before cleanup. Assert the guarded wipe does
     not signal or delete the changed owner and the command does not mutate beads or
     launch.
   - Extend work-rendering and epic lifecycle tests for a preserved upstream phase plus
     relaunched dependent, preserved land plus relaunched phases, selected land plus
     preserved phases, land-only fresh-clan/Patch launches, partial environment order,
     partial preclaims/rollback, and the all-preserved no-op.
   - Extend linked-plan/JSON/result tests so launched and preserved names are truthful,
     and task tests for matching running, waiting, and failed assignees plus a
     deterministic-name collision assigned to another bead.

## Validation

1. Run `just install` before project commands, as required for an ephemeral SASE
   workspace and its Rust-backed scan binding.
2. Run focused non-visual tests for the cleanup guard, rendering/segment environment,
   epic lifecycle, plan-file resume/result paths, and task lifecycle. At minimum include
   the new focused test module plus:

   ```bash
   .venv/bin/python -m pytest -q \
     tests/test_bead/test_cli_work_cleanup_confirm.py \
     tests/test_bead/test_cli_work_epic_launch_cleanup.py \
     tests/test_bead/test_cli_work_epic_lifecycle.py \
     tests/test_bead/test_work_rendering.py \
     tests/test_bead/test_cli_work_from_plan_resume.py \
     tests/test_bead/test_cli_work_task.py
   ```

3. Run `just check` and resolve every lint, type-check, and diff-scoped test failure
   caused by the change. If scoped selection escalates or reports unusual selection, run
   `just check-full` through `/sase_monitor`, as required by the repository verification
   policy.
4. Review the final diff and test output to confirm there are no `sase-core`, default
   configuration, keymap, memory, or unrelated TUI changes.

## Acceptance criteria

- Rerunning `sase bead work <epic>` while one matching phase agent is `RUNNING` leaves
  that process and its artifacts untouched, does not preclaim or relaunch that phase,
  and launches only missing/matching waiting-or-failed roles.
- A live process is signaled only when its current artifact is still waiting and its
  persisted bead equals the exact current launch bead. A failed/dead matching owner may
  be removed without a signal. A wrong-bead, beadless, changed-generation, or newly
  running owner is never killed or wiped.
- Partial retries retain all original dependency semantics: relaunched dependents and
  land agents wait for preserved phase names/beads, and an existing land agent is not
  duplicated.
- Family/container and legacy-land behavior cannot hide a running member or bypass the
  bead association check.
- An invocation with all roles already active succeeds without bead-store mutations,
  commits, pushes, or launches; human and JSON output report the actual launched and
  preserved names.
- Standalone task retries enforce the same state-and-bead cleanup safety while retaining
  existing claim contention, feedback, routing, checkpoint, and rollback behavior.
- Focused tests and the mandatory `just check` gate pass.

## Out of scope

- Adding a new durable launch-generation ID. The current prompt and artifact metadata
  already provide the required exact owner-to-bead authorization for this fix.
- Changing Rust bead preclaim semantics or the Rust agent-scan wire.
- Changing epic DAG construction, phase naming, task naming, xprompt contents, agent
  status labels, or general ACE kill/edit force-reuse behavior outside `sase bead work`.
- Automatically resuming a role that changes from running to waiting/failed after this
  invocation's conservative selection; a later `sase bead work` retry handles it.
