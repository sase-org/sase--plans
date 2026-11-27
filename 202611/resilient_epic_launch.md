---
tier: epic
title: Resilient epic launches under concurrent bead-ID minting
goal: 'Approved epics launch successfully even when another machine or agent mints
  and publishes beads in the same store at the same time. ID collisions are relocated
  automatically, removed IDs are never minted again, a launch never reuses another
  plan''s epic, and a graph that cannot be merged is re-minted from the plan instead
  of wedging the workspace store.

  '
decisions:
  conflict_replay:
    ask: Auto-discard the launch's own unpublished graph and re-mint from the plan
      when publication cannot merge?
    default: true
    why: nothing has spawned and the plan is the source of truth, so re-minting is
      safe
    answer: true
phases:
- id: merge
  title: Published-ID collision rule in the sase-core stream merge
  depends_on: []
  size: medium
  description: 'merge: replace the raw issue_created count in the Rust three-way stream
    merge with a rule that relocates any unpublished creation of a published ID and
    accepts create/remove/create history, plus a Python resolver regression for the
    incident''s stream shape.'
- id: counter
  title: Advance next_counter past every relocation and stream ID
  depends_on: []
  size: small
  description: 'counter: make the bead conflict resolver write config.json next_counter
    past every relocation ID and every stream ID in the store, even when config.json
    itself merged cleanly, so a rollback-and-retry never re-mints a relocated ID.'
- id: premint
  title: Pre-mint store refresh and linked-epic identity check
  depends_on: []
  size: medium
  description: 'premint: integrate the remote bead store right before each epic-creation
    attempt (fail open on transient errors, fail early on conflicts), record the launch
    base commit, and treat a plan bead_id link whose epic belongs to a different plan
    as stale.'
- id: replay
  title: Conflict replay for unmergeable zero-spawn graph publication
  depends_on:
  - premint
  size: medium
  description: 'replay: carry the integration status out of the managed sync worker,
    raise a typed publication-conflict error, and in the plan-launch retry loop discard
    the launch''s own unpublished graph commit under a recovery ref and re-mint, with
    cause-first error text.'
- id: regress
  title: Incident regression suite and docs
  depends_on:
  - merge
  - counter
  - premint
  - replay
  size: medium
  description: 'regress: reproduce the full three-clone collision chain end to end
    with real git clones, assert the launch succeeds under fresh IDs, and update the
    bead-storage and epic-launch reference docs.'
proposed_by: bbugyi200.athena.1hj
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 15:24:20
status: wip
bead_id: sase-1vt
---

- **PROMPT:** [prompts/202611/resilient_epic_launch.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/resilient_epic_launch.md)
- **BEAD:** [sase-1vt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vt/README.md)

# Plan: Resilient epic launches under concurrent bead-ID minting

## Incident (2026-11-27, bob-cli project)

The `1hg` agent's approved epic (`ctrl_jk_section_map`) failed to launch with:

```text
epic graph publication failed before agent launch for bob-cli-ra: git rebase failed: Rebasing (1/1)
error: could not apply 35f518644... chore(beads): checkpoint approved epic graph bob-cli-ra
...
Could not apply 35f518644...; semantic conflict resolution failed: validation: duplicate issue_created event for bob-cli-ra
```

The managed sync logs (`~/.sase/bead_push_logs/sync-*.log` on athena and apollo) and
`sase bead history` show a three-step chain:

1. **15:00:04 (apollo).** Apollo minted epic `bob-cli-r9` ("Tree-note link walks"). At
   15:00:25 athena's `1hh` agent minted and published task bead `bob-cli-r9`.
2. **15:01:19 (apollo).** Graph publication rebased onto that task. The bead resolver
   relocated apollo's epic from `r9` to `ra`, which worked. The plan launcher then
   rolled back the moved epic and published `issue_removed bob-cli-ra` at 15:01:31. At
   15:01:37 its retry minted **`bob-cli-ra` again**. Both clones had bumped
   `config.json` `next_counter` to the same value, so `config.json` merged cleanly. The
   resolver only advances the counter when `config.json` itself conflicts
   (`src/sase/bead/conflict_resolver.py`, `merged_config is not None`), and the
   allocator ignores removed beads. The comment in
   `src/sase/bead/cli_work_from_plan_launch.py` ("The integrated store's counter is
   already past both beads") is therefore false. Upstream
   `events/streams/bob-cli-ra.jsonl` now held create → remove → create.
3. **15:02:56 (athena).** The `1hg` workspace's bead-store clone had seen the `r9` task
   but not apollo's re-minted `bob-cli-ra`, which was published around 15:02:22. The
   `ctrl_jk_section_map` launch therefore also minted `bob-cli-ra`. Its last managed
   sync was at 14:56:43, and no fetch runs between plan archive and mint. Publication
   conflicted on that stream. `losing_creation` in sase-core
   `crates/sase_core/src/bead/events/merge.rs` counts raw `issue_created` events per ID.
   It found two upstream creations and one local creation. Because the non-winning
   creations had mixed provenance, it called the case "ambiguous" and returned
   `duplicate issue_created event`, when it should have relocated the local graph.

Fallout beyond the dead launch:

- **Workspace store blocked.** The workspace's bead store kept an unpublishable commit.
  The next agent run's sync in that workspace (15:05:25) failed the same way.
- **Resume repeats the failure.** The printed "Resume with" command would fail
  identically.
- **Plan link points at the wrong epic.** The archived plan's `bead_id: bob-cli-ra`
  header now names apollo's unrelated epic. `resolve_linked_epic`
  (`src/sase/bead/cli_work_from_plan_helpers.py`) only checks type and tier, so a resume
  would treat that epic as its own.
- **Misleading error.** The error's first line was git progress noise ("Rebasing
  (1/1)"), not the cause.

## Design

Four independent defenses, each sufficient for part of the chain, plus an end-to-end
regression:

| Layer   | Stops                                                            | Repo      |
| ------- | ---------------------------------------------------------------- | --------- |
| merge   | a re-created published ID making a collision "unresolvable"      | sase-core |
| counter | a rollback-and-retry re-minting the ID it was relocated to       | sase      |
| premint | minting from a minutes-stale clone; resuming another plan's epic | sase      |
| replay  | any future merge gap killing the launch and blocking the store   | sase      |

The governing boundary is `decisions:rust-core-required`. Stream-merge semantics are
shared, deterministic backend behavior, so the rule change belongs in `sase_core`, with
no Python fallback or reimplementation. Launch orchestration, git workspace handling,
and rendering stay in Python. Allocation itself stays in Rust and unchanged: the counter
floor is restored through `config.json`, which the Rust allocator already honors.

## Phase merge: published-ID collision rule (sase-core + a sase test)

Open the linked core with `sase repo open sase-core` and read its `AGENTS.md`
(build/test only through `just`; `sase tool run check` before finishing).

In `crates/sase_core/src/bead/events/merge.rs`, replace the collision detection in
`losing_creation` / `split_duplicate_creations` with this rule:

- **Published IDs.** Let _published_ be the IDs that have an `issue_created` event
  tagged `Base`, `Theirs`, or `Both`.
- **Collision.** A collision is any `issue_created` tagged `Ours` whose ID is published,
  whether that published bead is live or removed. Only `Ours` ever moves. This matches
  the existing doc comment: only the pushing clone's local, unpublished creation is
  relocated, and published IDs are never reused by an unpublished creation.
- **Published re-creation is not a collision.** Several creations of one ID that are all
  published are valid history: `Theirs` and `Base` are validated together, since
  `theirs` extends `base`. Multiple `Ours`-only creations are also valid. Neither may
  raise `duplicate issue_created`.
- **What moves.** Move the `Ours` subtree events from the first colliding `Ours`
  creation onward. Earlier `Ours` events on that ID, which act on an older published
  incarnation, stay in the merged stream.
  - Top-level IDs: extract into the relocated stream, as `extract_subtree` and
    `relocated_stream` do now.
  - Child IDs: renumber in place to the next free sibling, as `remap_subtree` and
    `next_sibling_issue_id` do now. `next_sibling_issue_id` already counts removed
    siblings, so tombstoned child IDs are never reused.
- **Order and loop.** Process shallow IDs before their children (BTreeMap order already
  does this). Keep the existing one-collision-per-round rescan loop.
- **Remaining errors.** Keep `relocation_issue_id == None` →
  `duplicate issue_created event for <id>` for a top-level collision. Keep
  `merged.validate()` and `relocated.validate()` as the final guard.

Rust tests (`crates/sase_core/src/bead/events/tests/merge.rs`):

- **Incident shape.** `theirs` = create X + children, remove X (cascade), create X +
  different children. `ours` = create X + its own children. Base is empty. Expect ours
  relocated to the supplied ID with its whole subtree, theirs' create/remove/create kept
  intact, and the merged and relocated streams reducing cleanly.
- **Published-only re-creation.** Base has create + remove of X; `theirs` re-creates X;
  `ours` appends something unrelated in the same stream. Expect no relocation and no
  error.
- **Unpublished reuse of a tombstone.** Base has create + remove of X; `ours` re-creates
  X; `theirs` appends to the stream. Expect ours relocated.
- **Ours-only re-creation.** `ours` alone has create/remove/create of X with no
  published creation of X. Expect no relocation.
- **Tombstoned child.** A child ID that a published `issue_removed` tombstoned and that
  `ours` re-creates is renumbered.
- **Regression.** Every existing merge test still passes, including
  `merging_without_a_relocation_id_still_reports_the_duplicate`.

In this sase checkout, add `tests/test_bead/test_conflict_resolver_reincarnation.py`. It
drives `sase.sdd._semantic_conflict_resolver.resolve_semantic_conflicts` on a real
rebase conflict with the incident's stream shape: two clones off one bare remote, built
with the helpers in `tests/test_bead/conflict_resolver_test_helpers.py` /
`sync_conflict_regression_helpers.py`. Assert success and a
`relocated duplicate beads: X -> <fresh>` message. Build the local `sase_core_rs` from
the linked checkout as described in `docs/contribute/rust-backend.md`. Committing both
repos in this phase's declaration moves `sase-core-revision.txt` automatically. Use a
Conventional Commit subject in sase-core (`fix(bead): ...`). The wire is unchanged, so
this is not a breaking change.

## Phase counter: advance next_counter past every relocation and stream ID

In `src/sase/bead/conflict_resolver.py` `_resolve_bead_conflict_paths`:

- **Raise the counter.** After the stream loop, whenever any event stream was merged,
  ensure `config.json` `next_counter` is at least one more than the highest top-level
  counter among the stream IDs now in the store. Compute it with
  `max_counter_in_ids(prefix, set(streams))`; that set includes streams created by
  relocation and the streams of removed (tombstoned) beads.
- **Do not burn IDs.** Do not use `relocation_ids.next_counter` for this floor.
  `_RelocationIds.allocate()` reserves an ID for every conflicted stream even when no
  relocation happens, so copying its cursor would skip IDs on every ordinary same-stream
  conflict.
- **Clean merges too.** Today this only happens when `config.json` itself conflicted.
  When it merged cleanly, load the worktree `config.json` (it has no conflict markers),
  apply `_config_with_allocated_counter` with that floor, save it with `save_config`,
  and include `config.json` in the staged paths and in `resolved_paths`. The conflicted
  path takes the max of the floor and the current pool cursor.
- **Write only when it rises.** Skip the write when the value would not increase, to
  avoid gratuitous `config.json` churn.
- **Keep the conflicted path.** The existing `merged_config` path stays as-is.

Also fix the false comment in `src/sase/bead/cli_work_from_plan_launch.py` ("The
integrated store's counter is already past both beads"). It becomes true after this
phase; reword it to cite the resolver guarantee.

Tests (extend `tests/test_bead/test_conflict_resolver.py` or add
`tests/test_bead/test_conflict_resolver_counter.py`):

- **Clean config merge.** Both sides bump `next_counter` to the same value (clean
  `config.json` merge) and collide on a top-level stream. After resolution,
  `next_counter` exceeds the relocated ID.
- **Fresh ID after rollback.** A `BeadProject` create after removing the relocated bead
  mints a fresh ID, not the relocated one.
- **No relocation, no rewrite.** A same-stream conflict with no relocation and an
  already-high counter leaves `config.json` untouched and skips no IDs.

## Phase premint: pre-mint store refresh and linked-epic identity check

All changes are in the plan-file launch path (`work_from_plan_file_locked` in
`src/sase/bead/cli_work_from_plan_launch.py` and helpers).

1. **Refresh before minting.** Run at the top of each attempt in the
   `for attempt in range(...)` creation loop, before `create_and_launch_epic_from_plan`,
   and once before `resolve_linked_epic` so link resolution sees published state:
   - Integrate the bead store with its upstream using
     `sase.bead.sync.refresh_bead_store` with a bounded `lock_timeout`, under a new
     timer stage `pre_mint_refresh`. It already no-ops for in-tree and remote-less
     stores.
   - Confirm the lock order against the launch's store lock. The refresh takes the store
     git write lock, so it must not run while this process already holds an incompatible
     lock. Document the ordering you verify in a short comment.
   - Make `BeadStoreRefreshError` (`src/sase/bead/_sync_refresh.py`) carry the
     `SddIntegrationStatus` of the failed integration, and choose the policy from that
     status:
     - **Transient (proceed).** `REMOTE_UNAVAILABLE`, `TIMED_OUT`, and
       `LOCK_UNAVAILABLE` print a yellow `!` warning and continue with minting.
       Relocation remains the backstop.
     - **Conflict or unrecoverable (stop).** Any other failure, such as
       `ABORTED_UNSUPPORTED_CONFLICTS`, means the store already holds commits that
       cannot publish. Stop before minting with a cause-first `error_with_resume`
       message. It names the store path, the integration error, and that nothing was
       created.
   - After a successful or skipped refresh, record the beads repo `HEAD` as the
     attempt's **launch base**. Expose it to the replay phase, for example as a field on
     a small per-attempt context object or a `timer.fields` entry plus a local variable.
2. **Check the linked epic's identity.**
   - When the archived plan already links `bead_id: <id>` and that bead exists, compare
     the bead's `design` with `plan_ref_for_store(archived_path, store, ...)`.
     Canonicalize both sides the same way, through the existing plan-ref helpers.
   - If the bead names a different plan, treat the link as stale, exactly like a missing
     bead: set `stale_epic_id`, let the existing stale-link replacement relink the plan,
     and mint fresh. Render a warning naming the other epic's title and plan ref.
   - Never resume, preclaim, or launch another plan's epic. An epic with no `design`
     keeps today's behavior.

Tests (extend `tests/test_bead/test_cli_work_from_plan_concurrency.py` and
`test_cli_work_from_plan_resume_links.py`):

- **Remote mint becomes visible.** A remote clone mints and publishes a top-level bead.
  The launching clone's first epic gets the next ID and needs no relocation.
- **Refresh status policy.** Transient refresh statuses proceed. A conflicting refresh
  stops before any bead is created and leaves the store unchanged.
- **Foreign link.** A linked `bead_id` that belongs to a different plan's epic is
  replaced, not resumed.
- **Own link.** A matching link still resumes.

## Phase replay: conflict replay for unmergeable zero-spawn graph publication

> [!decision] conflict_replay

1. **Carry the integration status.**
   - Add `integration_status: str | None` to `_ManagedSyncOutcome`
     (`src/sase/bead/_sync_worker_run.py`) and `PushOutcome`
     (`src/sase/bead/_sync_publication.py`), set from the failing
     `SddIntegrationOutcome`.
   - Populate it on every `_failure` path that has an integration result, and in
     `push_bead_work_launch`.
2. **Raise a typed conflict error.**
   - In `publish_epic_graph_before_launch_result`
     (`src/sase/bead/cli_work_from_plan_store.py`), when `outcome.integration_status` is
     `aborted_unsupported_conflicts`, raise a new `EpicGraphPublicationConflictError`
     (in `src/sase/bead/cli_work_handler_errors.py`). It carries the semantic cause and
     the sync log path.
   - Keep every other failure on today's `RuntimeError` path, so transient failures
     still preserve the local checkpoint as the retry point.
   - `checkpoint_and_publish_graph` must re-raise this error as a `BeadWorkError`
     subclass with `publication_conflict=True` instead of the generic wrapper.
3. **Roll back for replay.** In `create_and_launch_epic_from_plan`
   (`src/sase/bead/epic_from_plan.py`), a publication-conflict error with zero agents
   spawned:
   - restores the plan link through `commit_plan_update` with the original content, like
     the existing rollback;
   - does **not** call `proj.remove`, because the store is about to be reset and a
     removal would only add more unpublished events;
   - raises `EpicFromPlanError(publication_conflict=True, ...)`.
4. **Replay in the creation loop.** In the creation retry loop
   (`work_from_plan_file_locked`), on `publication_conflict` with attempts remaining,
   discard the attempt's own unpublished graph:
   - **Lock and fetch.** Under the store git write lock, fetch the beads repo.
   - **Safety checks.** All of the following must hold:
     - the launch base recorded by the premint phase is an ancestor of `@{upstream}`;
     - every commit in `launch_base..HEAD` was created by this attempt (compare against
       the checkpoint SHA captured right after `commit_epic_graph_checkpoint`, plus any
       commits the attempt itself recorded);
     - the worktree has no other tracked changes. If any check fails, do not reset: fail
       with the cause-first message from step 5, the manual recovery steps, and
       `sase doctor -v`.
   - **Preserve, then reset.** Preserve `launch_base..HEAD` with the existing
     recovery-ref helpers (`src/sase/sdd/_repository_recovery_git.py` /
     `src/sase/workspace_provider/_rescue_git.py`), as a ref under `refs/sase/recovery/`
     naming the epic ID. Then hard-reset only that sidecar clone to `@{upstream}`. Honor
     the bead-store write sandbox (`assert_bead_store_write_sandboxed`) and the
     workspace ownership guards. If they refuse, fail without resetting.
   - **Retry.** `continue` the loop. The next attempt re-runs the premint refresh, opens
     a fresh `BeadProject`, and mints a fresh ID.
   - **Report.** Render
     `↻ Epic ID <old> could not be merged with concurrently published history; discarded the unpublished launch graph (recovery ref <ref>), retrying (attempt n/3)`.
     Count replays in `timer.fields["publication_conflict_replays"]`.
   - Attempts stay bounded by `_EPIC_CREATION_MAX_ATTEMPTS`. When they are exhausted,
     fail as today, with the step 5 message.

> [!decision] conflict_replay = no Skip steps 3–4. Keep the local checkpoint on
> publication conflicts, but still do steps 1, 2, and 5. The typed error's message must
> say that resuming cannot succeed until the store is rescued, and print the exact
> manual recovery: rescue the commit to a recovery ref, reset the beads clone to
> upstream, remove the plan's `bead_id` header, and rerun `sase bead work <plan>`.

5. **Put the cause first.**
   - When a rebase fails and semantic repair fails, lead the integration error with the
     semantic cause, for example
     `bead-ID collision with concurrently published history could not be merged: duplicate issue_created event for bob-cli-ra`.
   - Follow it with the managed sync log path. Drop git `hint:` lines and
     `Rebasing (n/m)` progress from the summary; the full git output stays in the sync
     log.
   - Implement this in `repair_or_abort_rebase` / `format_git_error` callers
     (`src/sase/sdd/_repository_integration_repair.py`), without changing the
     `SddIntegrationStatus` values.

Tests:

- **Typed error.** `tests/test_bead/test_cli_work_epic_launch.py` and
  `test_cli_work_from_plan_publication_flow.py` cover the typed error and the zero-spawn
  replay. Stub `launch_bead_work_agents`. Seed an unmergeable conflict with a
  monkeypatched `resolve_semantic_conflicts` that returns failure, so the test does not
  depend on the merge phase.
- **Replay path.** Assert the recovery ref exists, the store equals upstream before the
  retry, the retry mints a fresh ID, and the plan's `bead_id` names the launched epic.
- **Refusal path.** A foreign unpublished commit below the launch base makes the replay
  refuse without resetting.
- **Message.** The cause-first message omits `hint:` lines.

## Phase regress: incident regression suite and docs

1. **End-to-end incident replay.** Add
   `tests/test_bead/test_epic_launch_collision_chain.py` using the real-git helpers
   (`sync_conflict_regression_helpers.py`: bare remote plus three clones):
   - **Force the race.** Monkeypatch the pre-mint refresh off for clones B and C, so
     both mint from stale counters as in the incident.
   - **Clone A** creates and publishes task bead `p-1`.
   - **Clone C** integrates now, so it sees `p-1` and none of B's publications.
   - **Clone B** creates an epic with phases (minted `p-1`) and publishes through the
     plan-launch path with `launch_bead_work_agents` stubbed. Expect relocation to
     `p-2`, rollback, and a retry that mints **`p-3`**, not `p-2`.
   - **Clone C** launches another plan's epic:
     - It mints `p-2` from its stale counter.
     - It publishes against the create → remove → create `p-2` stream.
     - Expect the launch to succeed under a fresh ID through relocation.
   - **Replay variant.** With relocation forced to fail, the conflict-replay path
     succeeds instead.
   - **Convergence.** All three clones converge, and the reduced stores match a fresh
     reduction (`_assert_store_matches_fresh_reduction`).
2. **Docs.** Read `docs_writing.md` with `/sase_memory_read` first. Update:
   - `docs/reference/work-units/bead-storage.md` "Duplicate Bead IDs":
     - the published-ID rule;
     - create/remove/create history is valid;
     - removed IDs are never minted again, because the resolver advances `next_counter`
       past every stream ID;
     - only the unpublished side moves.
   - `docs/reference/work-units/cli-workflows.md`, the `sase bead work <plan>` launch
     section. Add the pre-mint refresh and its transient-vs-conflict policy, the
     linked-epic identity check, and the publication-conflict replay with its recovery
     ref and attempt bound. If `conflict_replay = no` was chosen, document the
     cause-first failure and the manual recovery instead.
   - Run `just fix` if any generated reference changes.

## Out of scope / observed separately

- **Plans clone conflict.** At 15:08:46 the bob-cli primary plans clone failed to
  integrate an `Archive approved plan ctrl_jk_section_map` commit against the upstream
  copy (`non-bead conflicts remain: 202611/ctrl_jk_section_map.md`). The actor that
  re-archived the plan there was not identified. It is a plans-store double-archive
  issue, not a bead-ID collision. Record it as a follow-up task during this epic's land
  if it recurs; do not widen these phases for it.
- **Task-bead minting.** Pre-mint refresh is deliberately limited to epic launches.
  Ordinary `sase bead create` stays offline-fast; its collisions are handled by the
  merge and counter phases.
