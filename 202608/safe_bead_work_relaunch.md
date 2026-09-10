---
tier: tale
title: Preserve active bead workers during deterministic relaunch
goal:
  Re-running `sase bead work` replaces only waiting or failed owners that are provably
  assigned to the beads in the current launch, while active work is preserved and
  omitted from the replacement launch.
size: medium
proposed_by: bbugyi200.athena.02v
create_time: 2026-09-09 20:00:27
status: wip
---

# Plan: Preserve active bead workers during deterministic relaunch

## Why this is a tale

This is one bounded correction to the existing bead-work retry transaction. The
agent-state inspection, cleanup policy, selected prompt/environment projection,
preclaim, launch result, and tests form one serial chain with a single safety invariant.
The Rust bead mutation already accepts a phase subset and an optional land assignment,
so no core-backend feature or cross-repository phase is needed. Splitting the work among
phase agents would add handoff risk precisely where the state and launch views must
remain aligned.

## Objective

Make `sase bead work` safe to repeat while an epic is still underway. A retry may
replace a deterministic phase or land owner only when all of the following are true:

1. the owner is in the exact phase/land launch set just rendered;
2. durable agent metadata associates it with that segment's phase bead (or the epic bead
   for land);
3. the bead store still assigns that bead to the same current owner; and
4. its authoritative lifecycle is `WAITING` or `FAILED` at cleanup time.

A `RUNNING` worker, or any other nonterminal active state, must survive the command and
must not have a second agent launched for its bead. Ambiguous, missing, or contradictory
ownership evidence must stop the retry before cleanup or bead mutation. The direct task
path must retain its existing live-worker idempotence and gain the same bead-aware guard
for any deterministic-name cleanup it still performs.

## Confirmed diagnosis

- `launch_epic_bead_work` builds every non-closed phase plus land, renders every
  segment, and passes every deterministic name to `preview_bead_work_force_reuse` /
  `prepare_bead_work_force_reuse`.
- `preview_bead_work_force_reuse` currently asks only whether a matching process is
  live. Every live exact owner or family member becomes `KILL (running)`; it does not
  inspect waiting/failed status, the metadata `bead_id`, or the bead's assignee.
- `prepare_bead_work_force_reuse` then wipes those owners regardless of state. The
  generic wipe terminates live artifact PIDs before deleting the owner graph, so a
  healthy worker can be killed even though its phase is already in progress.
- After cleanup, the handler preclaims and launches the complete rendered plan. Thus a
  retry both destroys active work and starts a replacement for the same bead.
- The task path has an earlier `already_running` check for a live current assignee, but
  its fallback deterministic-name cleanup is still state- and bead-blind.
- The current launch already has the identity evidence needed for a safe decision:
  `EpicWorkPlan` maps phase names to phase bead ids and land to the epic id;
  `%id(!..., bead=...)` parsing exposes the same mapping through
  `force_reuse_bead_associations_by_prompt`; `agent_meta.json` carries `bead_id`; and
  the bead store carries the current assignee.
- `render_multi_prompt` and `epic_work_segment_env` deliberately use identical full-plan
  ordering. A partial retry must project both from one selected segment list or their
  indexes can drift. It must also retain waits on preserved upstream workers/beads,
  place clan declaration/summary metadata correctly on the first newly launched segment,
  and keep the existing Patch/VCS prefix semantics.
- The Rust `preclaim_epic_work_plan` API already accepts an assignment subset and an
  optional epic agent. Only the Python facade currently types the land name as required.
  Therefore preserved segments can be excluded from preclaim without changing
  `sase-core`.

## Safety contract

Classify each planned segment before asking for confirmation, then revalidate it
immediately before any signal or deletion. Use the canonical artifact scan and lifecycle
markers, not a display-only family rollup and not process liveness alone.

| Observed owner for planned `(name, bead)`                                                             | Decision   | Effect                                                                              |
| ----------------------------------------------------------------------------------------------------- | ---------- | ----------------------------------------------------------------------------------- |
| No concrete owner or reservation                                                                      | `LAUNCH`   | Include the segment; no cleanup.                                                    |
| Matching live owner in exact `WAITING` state                                                          | `REPLACE`  | Preview `KILL`; guarded cleanup; relaunch.                                          |
| Matching terminal `FAILED` owner                                                                      | `REPLACE`  | Preview `REMOVE`; clean artifacts; relaunch.                                        |
| Matching live owner whose own lifecycle is `FAILED`                                                   | `REPLACE`  | Preview `KILL`; guarded cleanup; relaunch.                                          |
| Matching `RUNNING`, `STARTING`, question/answer, monitor, or any other active/nonterminal owner       | `PRESERVE` | Do not signal, delete, preclaim, or relaunch this segment.                          |
| Matching successful/other terminal owner while its bead remains non-closed                            | `BLOCK`    | Require the user to reconcile the bead instead of guessing that work should repeat. |
| Concrete owner has missing/different bead metadata, or bead assignee does not normalize to that owner | `BLOCK`    | Report the conflicting name, artifact, expected bead, observed bead, and assignee.  |
| Name-only orphan reservation with no concrete artifacts                                               | `RELEASE`  | Release the reservation; this is not process termination.                           |

Additional rules:

- Treat the current generation of a deterministic family as one launch segment. If any
  concrete member is active in a protected state, preserve the entire segment. Replace
  it only when every live member is `WAITING`/`FAILED`, every concrete member is
  associated with the planned bead, and the bead assignment identifies the same family
  owner. Mixed or incomplete evidence blocks rather than partially wiping a family.
- Treat the legacy bare epic land name as an alias for the current land segment, not as
  an unassociated extra wipe. A legacy land owner is replaceable only with epic-bead and
  assignee proof; a protected legacy land owner preserves the land segment. If both
  legacy and current land generations have concrete owners, block and describe the
  conflict instead of deleting one opportunistically.
- A populated epic clan remains the joinable container; it is not itself a cleanup
  target. An orphaned clan/family reservation may be released only after proving it has
  no concrete members.
- A recheck may narrow a previously confirmed cleanup (`WAITING` became `RUNNING`, so
  preserve it), but it must not silently add a newly destructive target that was absent
  from the preview. A new eligible collision requires another invocation/confirmation.
- Hold the existing agent-name allocation lock across the final owner lookup, guarded
  cleanup, registry rebuild, and prompt rewrite. Add an opt-in bead-work wipe guard that
  re-reads the exact artifact identity, bead id, PID/liveness, and waiting/failed marker
  immediately before process signaling and again before artifact removal. If the
  fingerprint or state changed, refuse the wipe and preserve/block; do not weaken the
  default semantics of `wipe_agent_name_for_reuse` for other callers.

## Implementation

### 1. Build one state-aware launch inventory

- Add a focused immutable segment/decision model beside the bead-work orchestration.
  Preserve full authored order and record role (`phase`/`land`), planned agent name,
  accepted legacy alias, bead id, dependency names/beads, and the concrete owner
  fingerprint inspected for cleanup.
- Derive the canonical name-to-bead map from the rendered force-reuse directives and
  assert it equals the `EpicWorkPlan` mapping. Do the same for the task's one segment.
  Reuse `force_reuse_bead_associations_by_prompt` and the current-owner normalization
  from `sase.bead.force_reuse`; do not add a second parser or rely on name similarity.
- Load each relevant exact owner/current family generation once from the existing agent
  artifact/index scan, including metadata bead association, waiting marker, done
  outcome, PID/liveness, and concrete artifact path. Keep this targeted and read-only;
  do not load the full ACE/TUI model or add render-path work.
- Resolve the corresponding current `Issue` values from the already-open `BeadProject`,
  and run the decision table above. Return all blockers together before any prompt
  rewrite, cleanup, readiness change, snapshot, preclaim, commit, or spawn.

### 2. Make preview and cleanup enforce the same decision

- Replace cleanup's current liveness-only target construction with the launch inventory.
  Extend the preview vocabulary with a non-destructive `PRESERVE`/`KEEP` row so dry runs
  and interactive retries explicitly show which running workers will remain. Keep
  `has_destructive_targets` true only for `KILL`, `REMOVE`, and `RELEASE`.
- Render expected bead ids and observed states in cleanup summaries. `--dry-run` must
  print the selected replacement prompt rather than the current misleading full prompt,
  and must perform no registry, artifact, bead, snapshot, VCS, or launch mutation.
- Pass approved cleanup identities, expected bead associations, and allowed states into
  `prepare_bead_work_force_reuse`; do not let it rediscover an unrestricted name list.
  Under the name-allocation lock, recompute the inventory and apply only the
  intersection of the previewed replacements and still-eligible replacements.
- Give the generic wipe an optional validation callback/guard (or an equivalently narrow
  bead-work wrapper at the signal/deletion boundary). The guard must run after the wipe
  graph is built so descendants cannot broaden cleanup beyond the authorized current
  generation. An unauthorized descendant, changed PID/artifact, state transition, or
  changed bead association is a structured cleanup refusal, never a best-effort kill.
- Rewrite force-reuse `%id:!` directives only for segments whose prior owners were
  removed/released or that had no owner. Preserved segments are removed from the launch
  query entirely.

### 3. Launch and preclaim only the selected segments

- Project prompt segments, expected names, and per-segment environments from the same
  ordered selected-segment object. Assert their counts and parsed `(name, bead)` pairs
  remain identical before calling the launcher.
- Preserve dependency directives from the full plan. A newly launched downstream phase
  or land agent may still wait on a preserved worker name and its bead; omitting a
  worker from this invocation must not erase the graph edge.
- Re-evaluate clan declaration after selection: join an existing populated epic clan,
  declare it on the first selected segment only when it is genuinely absent, and reject
  a preserved member with a missing/conflicting clan reservation. Move the clan summary
  environment marker to the first selected environment when the original first segment
  is preserved. Preserve the existing first-phase Patch creation/ref targeting rules;
  add assertions/tests for a preserved first phase so filtering cannot retarget later
  segments incorrectly.
- Snapshot/mark ready/checkpoint/publish only when at least one segment will launch.
  Preclaim only selected phase assignments and pass the land name only when land is
  selected. Change the Python project/facade type to permit `land_agent_name=None`,
  matching the existing Rust binding; do not change Rust claim semantics or edit the
  linked `sase-core` repository.
- Scope rollback records, partial-launch termination, agent counts, and workspace output
  to the selected launch set. Never restore or terminate a preserved worker as part of a
  failure in newly spawned segments.

### 4. Represent partial and no-op retries truthfully

- Replace the epic launch helper's ambiguous boolean with a small result value parallel
  to `TaskWorkResult`. It must distinguish `launched`, `partially_launched`,
  `already_running`, `dry_run`, and `declined`, and carry launched, replaced, and
  preserved agent names plus workspace information when present.
- Update direct bead-id JSON and human output to report both preserved and newly
  launched workers. Keep `launched` as “at least one new process spawned”; expose
  `launch_state` so an all-preserved success is not mislabeled as an aborted launch.
- Thread the richer result through linked-plan resume. When every segment is protected,
  return `already_running`, perform no graph push/rollback, and render the epic as
  underway. For a partial retry, publish once and report only the names actually
  launched while retaining the complete authored phase list.
- Keep task output compatibility. Its current matching live assignee remains an
  `already_running` no-op. Any fallback cleanup of the deterministic task name must use
  the same expected task-bead association and may terminate only `WAITING`/`FAILED`; a
  differently named assigned worker is not silently broadened into a cleanup target.

### 5. Add regression coverage around the safety boundary

Build artifact fixtures with real `agent_meta.json` bead ids and authoritative waiting,
run-started, and done markers instead of labeling every live PID “running.” Cover:

- a running phase plus waiting/failed sibling phases: preserve and omit the running
  segment, replace only the eligible siblings, and preclaim/launch only those siblings;
- a running land agent, a waiting land agent, and both legacy/current land-name cases;
- all segments running: no confirmation, cleanup, snapshot, bead write, graph publish,
  or spawn; result/output says `already_running` and the plan-file resume remains
  underway;
- all fresh segments: unchanged full launch, ordering, clan declaration, environment,
  preclaim, checkpoint, and rollback behavior;
- current family generations with running, waiting, failed, mixed, and unrelated-bead
  members, including a protected child beneath a terminal family root;
- same deterministic name with missing metadata, a different bead id, a mismatched bead
  assignee, a successful terminal outcome, and simultaneous legacy/current land owners:
  each blocks before the first mutation and gives actionable evidence;
- a stale name-only reservation: release and launch without claiming that a process was
  killed;
- a state/PID/artifact/bead transition between preview and cleanup: cleanup narrows to
  preserve or aborts, and never signals the changed process; a transition that would add
  a new destructive target is not applied without a new preview;
- partial query filtering with preserved upstream dependencies, preserved original first
  phase under VCS and Patch contexts, selected land only, selected phases with preserved
  land, environment/count alignment, and launch failure rollback limited to selected
  preclaims/processes;
- task-bead `already_running`, waiting/failed exact-owner, unrelated-owner, and stale
  reservation paths so the command-wide invariant cannot regress;
- human/dry-run/JSON/plan-file result rendering for full, partial, already-running,
  declined, and dry-run states.

## Expected files

- `src/sase/bead/cli_work_handler.py` and a focused adjacent selection module for the
  inventory, partial retry transaction, and result value.
- `src/sase/bead/cli_work_cleanup.py` and `src/sase/agent/names/_wipe.py` for
  association-aware preview and opt-in guarded cleanup.
- `src/sase/bead/work.py`, `src/sase/bead/cli_work_launch.py`, and
  `src/sase/bead/cli_work_plan.py` as needed to project aligned selected segments and
  render preserved/replaced decisions.
- `src/sase/bead/_project_mutations.py` and `src/sase/core/bead_mutation_facade.py` for
  the already-supported optional land preclaim at the Python boundary.
- `src/sase/bead/cli_work_entry.py` and the plan-file work result/resume/render modules
  for truthful partial/already-running results.
- Focused tests under `tests/test_bead/test_cli_work_*.py`, especially collision,
  cleanup-confirmation, epic-relaunch/checkpoint, task, and plan-file resume suites; add
  narrow wipe-guard tests under `tests/test_agent_names_*.py` if the generic guard lives
  at that boundary.

No SASE memory, generated instruction, default configuration, documentation, or PNG
snapshot change is expected. No linked repository may be modified.

## Validation

1. Run focused cleanup/collision, epic retry/checkpoint, task work, plan-file resume,
   and guarded-wipe tests while developing. Include assertions that forbidden paths were
   not called, not only that the final launch count is correct.
2. Run the existing force-reuse bead-association and agent-family/name-registry suites
   to catch authorization or current-generation regressions.
3. Run `just install` before repository-wide commands in the ephemeral workspace.
4. Run `just check`; resolve every caused Ruff, mypy, Symvision, or scoped-test failure.
   Re-run the focused regression matrix after any correction.
5. If test selection escalates, reports unusual coverage, or the implementation touches
   a broadening-set file, run `just check-full` through `/sase_monitor` with a follow-up
   action as required by the repository instructions.

## Acceptance criteria

- Re-running an active epic never signals, deletes, reassigns, or relaunches a `RUNNING`
  (or otherwise protected active) phase/land worker.
- Every process the command does terminate is both `WAITING`/`FAILED` at the guarded
  cleanup boundary and durably associated with the exact bead for its current launch
  segment; missing or contradictory evidence fails closed.
- Partial retry launches only fresh/replaced segments while preserving full dependency,
  clan, VCS/Patch, environment, and bead-attribution semantics.
- Preclaim, publication, rollback, counts, and output describe only newly launched
  segments; all-running retries are successful mutation-free `already_running` no-ops.
- Family generations, the legacy land alias, stale reservations, races after preview,
  task work, and plan-file resume obey the same safety invariant.
- A fresh epic launch remains behaviorally unchanged, and the repository's scoped
  verification passes.
