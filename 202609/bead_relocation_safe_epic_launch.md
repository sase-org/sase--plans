---
tier: epic
title: Relocation-safe bead IDs and epic launches
goal: 'A concurrently minted bead ID never renumbers a bead that is already published,
  and `sase bead work` only acts on bead relocations it can prove moved its own beads.
  When publication does move a freshly created epic graph, the launch rolls back by
  the moved IDs and retries automatically, so an approved epic launches without a
  manual retry and without collateral damage to other agents'' beads.

  '
phases:
- id: core-winner
  title: Published bead creations keep their ID in duplicate-ID merges
  depends_on: []
  size: small
  description: 'core-winner: in the linked sase-core repo, make the merge-base or
    published upstream creation win a duplicate issue_created collision so only the
    local, unpublished bead is ever relocated; replace the orientation-independence
    tests and update the docs.'
- id: launch-guard
  title: Identity-verified relocation handling in bead work launches
  depends_on: []
  size: medium
  description: 'launch-guard: replace the prompt and env text rewrite in launch_epic_bead_work
    with an identity-verified check. Foreign relocations are ignored; a relocation
    of the launch''s own graph rolls back on the moved IDs and raises EpicGraphRelocatedError.
    Also fix the resume callback that drops relocations, guard the task path, and
    make the text rewrite helper token-safe.'
- id: plan-retry
  title: Automatic recovery for approved-plan epic launches
  depends_on:
  - launch-guard
  size: medium
  description: 'plan-retry: when a freshly created epic is relocated, remove it by
    its moved ID, restore the plan''s bead_id, publish the rollback, and retry creation
    (at most 3 attempts); relink a resumed plan to its moved epic and fail with an
    actionable resume command.'
- id: pin-regression
  title: Core pin bump, git-backed regressions, and incident cleanup
  depends_on:
  - core-winner
  - plan-retry
  size: small
  description: 'pin-regression: move sase-core-revision.txt past the core-winner commit,
    add git-backed regressions where the local creation is older than the upstream
    one, and repoint the tool_handoff flag registry entry to sase-17w if it still
    names sase-17v.'
proposed_by: bbugyi200.athena.0qv
create_time: 2026-09-24 11:57:09
status: wip
bead_id: sase-17y
---

- **PROMPT:** [prompts/202609/bead_relocation_safe_epic_launch.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/bead_relocation_safe_epic_launch.md)
- **BEAD:** [sase-17y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17y/README.md)

# Plan: Relocation-safe bead IDs and epic launches

## Incident (2026-09-24): what actually happened

The user approved the `0qs` agent's epic plan `202609/command_line_panel.md` from the
TUI. `sase bead work <plan> --yes-to-all` failed with:

```
Error: agent launch failed for epic sase-17w: bead-work rendered agent names
['sase-17w.1', ..., 'sase-17w.land'] do not match planned names ['sase-17v.1', ..., 'sase-17v.land']
```

Reconstructed from the beads sidecar history, the managed bead sync log, and the chat
transcript:

| Time (EDT) | Event                                                                                                                                                                                                                                                                                                                                            |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 11:20:35   | The epic launch mints `sase-17v` for the epic in its own clone of the beads store. The ID is **not published** yet; creating the phases, dependencies, and plan link, previewing the launch, and preclaiming take about 2 minutes.                                                                                                               |
| 11:20:38   | The agent `sase-17p.2`, in another clone, mints the same counter (`sase-17v`) for flag bead "Retire tool_handoff".                                                                                                                                                                                                                               |
| 11:20:41   | `sase-17p.2` pushes first (`chore(beads): create sase-17v`). Its code already embeds the ID: its in-flight `src/sase/feature_flags/registry.py` change maps `tool_handoff` to `bead="sase-17v"`.                                                                                                                                                 |
| 11:22:36   | The epic's graph checkpoint rebases onto upstream, hits an add/add conflict on `events/streams/sase-17v.jsonl`, and the semantic resolver relocates. The sync log records `bead_relocations: [{old_id: sase-17v, new_id: sase-17w, kind: top_level_duplicate}]`.                                                                                 |
| 11:22:36   | **The relocated bead is the already-published flag bead**, because the Rust merge keeps "the older creation": 15:20:35 (epic) beats 15:20:38 (flag). The epic kept `sase-17v`. The flag bead became `sase-17w`.                                                                                                                                  |
| 11:22:36+  | `launch_epic_bead_work` assumes every relocation moved _its own_ beads. It rewrites the prompt text `sase-17v` → `sase-17w` and sets `epic_id = sase-17w`, but `expected_names` still holds `sase-17v.*`. The name guard in `launch_planned_bead_work_agents` fires. That guard is what stopped 13 agents launching against someone else's bead. |
| 11:23:18   | Zero-spawn rollback runs on `sase-17w`, which is the flag bead. Restoring readiness on it fails only because of a bead-type guard ("failed to restore is_ready_to_work on sase-17w").                                                                                                                                                            |
| 11:23:19   | The plan-file caller rolls back the creation of `sase-17v` (correct in this case) and restores the plan's `bead_id`.                                                                                                                                                                                                                             |
| 11:30:42   | A manual retry succeeds as `sase-17x`.                                                                                                                                                                                                                                                                                                           |

### Verdict on the "race condition" suspicion

**Partly right.** There was a genuine race: two clones minted the same counter-derived
ID 3 seconds apart. That collision is expected, and it is handled by design (the Rust
resolver relocates one side). The launch failure itself was **deterministic, given the
collision**, and comes from three bugs:

1. **The merge winner rule can renumber a published bead.** `losing_creation` in the
   sase-core `crates/sase_core/src/bead/events/merge.rs` keeps the older
   `(timestamp, event_id)` creation "so both clones agree regardless of which side git
   calls ours". But the Python resolver already maps orientation explicitly
   (`merge(base, local, upstream)` via `upstream_and_local_stages`). Only the clone that
   is pushing receives relocation records. So when the _published_ bead loses, its
   creator never learns the ID moved. Collateral damage today: the flag bead is now
   `sase-17w`, while `sase-17p.2`'s code says `bead="sase-17v"`. `sase-17v` no longer
   exists; had the epic launched, it would have pointed at the Command Line epic.
2. **The epic launch misapplies relocation records.** `launch_epic_bead_work`
   (`src/sase/bead/cli_work_handler.py`) resolves `epic_id` and text-rewrites the query
   and segment env through _every_ relocation. The record carries no side, and the
   subject rewrite in the sync worker has the same blind spot: it mislabeled the
   checkpoint commit as "checkpoint approved epic graph sase-17w".
3. **Even a genuine relocation of the launch's own graph cannot work today:**
   - `expected_names` is never remapped, so the name guard always fires.
   - The Rust relocation remaps issue, parent, dependency, and link IDs, but not
     `assignee`. The preclaimed phases therefore stay assigned to the old agent names,
     and `src/sase/bead/claims.py` only accepts a claim when
     `issue.assignee == agent_name`.
   - The plan snapshot lives at `artifacts/epic-plans/<epic_id>.md` under the old ID,
     and the text-rewritten env points at a file that does not exist.
   - `rollback_preclaims` and the plan-file rollback (`_rollback_epic_creation` removes
     `epic.id`) use pre-publication IDs. By then those IDs belong to _another clone's
     published bead_, so a failed launch would clobber or delete it.
   - The resume path's `publish_resumed_graph` returns `None`, so resumed launches drop
     relocations entirely.

   The only test (`test_work_rewrites_launch_query_and_env_after_graph_relocation`) uses
   a fake launcher. It captures `expected_names` but never asserts it, which is how this
   shipped.

### Why "just retry" is not the fix

A blind retry would have made this one launch succeed, and the manual retry did. But:

- It cannot undo the collateral renumbering of the other agent's published bead.
- Its rollback uses pre-publication IDs. Once relocations move the _local_ graph (which
  is the common case whenever the upstream bead is older), the rollback would delete or
  overwrite a foreign bead before retrying.
- It hides a deterministic bug behind 2+ minutes of repeated work and publication churn.

The retry idea survives in a narrow, correct form (phase `plan-retry`): once publication
provably moved _our own freshly created_ graph, roll back by the moved IDs, restore the
plan's `bead_id` frontmatter, publish the rollback, and recreate. We rejected in-place
re-anchoring (re-preclaiming under new agent names, re-snapshotting, relinking, and a
second publication). It has more new states than the proven rollback-and-recreate path,
for an event that should be rare.

## Design invariants

1. **A published bead ID is immutable.** In a duplicate-`issue_created` merge, the
   merge-base creation wins, otherwise the published upstream (`theirs`) creation wins.
   Only the pushing clone's local, unpublished creation is relocated, and that clone is
   the one that receives the relocation record and can repair its references.
2. **A launch acts only on relocations it can prove moved its own beads.** It verifies
   the move by reading the post-publication store and matching the bead's creation
   identity. Timestamps and relocation direction are not trusted. This keeps the Python
   side correct on both the old and the new core rule, so it does not depend on which
   `sase_core_rs` is installed.
3. **Nothing launches under an ID that publication moved.** If the launch's own graph
   moved, it rolls back by the moved IDs and either retries (fresh plan launch) or fails
   with a resume command naming the moved ID.

## Phase `core-winner`: published creations keep their ID (sase-core)

Work in the linked `sase-core` repo (`sase repo open sase-core -r "<why>"`) and follow
its `AGENTS.md`. Never run bare `cargo`. Verify with `sase tool run check` from inside
that checkout; it takes about 5 minutes, so use a tool timeout of at least 10 minutes.

- `crates/sase_core/src/bead/events/merge.rs`, `losing_creation`: choose the winner as
  1. the `BranchTag::Base` creation, if any;
  2. otherwise the `BranchTag::Theirs` (published upstream) creation;
  3. otherwise, defensively, the current oldest `(timestamp, event_id)`.

  Keep the existing ambiguity checks unchanged: a loser tagged `Base` or `Both`, or
  losers from mixed sides, still errors.

- Rewrite the comments that justify the old rule:
  - the `losing_creation` comment ("the older creation keeps the id so both clones
    agree...");
  - the `merge_bead_event_streams_with_relocation` docs;
  - the module and `BeadEventStreamMergeWire` docs, if they mention orientation.

  State the new reason. A published ID may already be embedded by its creator (code such
  as the feature-flag registry, commit trailers, plan links, chats). The creator never
  learns about a relocation performed by another clone's push. So only the pushing
  clone's unpublished creation may move. Keep the existing sentence that `ours` is local
  and `theirs` is upstream.

- Tests in `crates/sase_core/src/bead/events/tests/merge.rs`:
  - `concurrently_minted_bead_id_relocates_instead_of_wedging_the_store`: `theirs`
    ("Theirs") now keeps `sase-ey`, and `ours` relocates to `sase-ez`.
  - Replace `relocation_picks_the_same_loser_whichever_side_git_calls_ours` with tests
    showing that the upstream creation keeps the ID for **both** timestamp orders (local
    older and local newer).
  - `concurrently_minted_child_id_renumbers_to_a_free_sibling`: the upstream child keeps
    `.1`, and the local child renumbers to `.2`.
  - Re-check `relocated_events_are_reminted_onto_their_new_stream`; the relocated events
    are now `ours`.
- Check the PyO3 binding tests (`crates/sase_core_py/src/beads/tests.rs`, around the
  `relocation_records` assertion) and every other test that encodes the old winner:
  `rg -n "older|Ours|Theirs|relocat" crates/sase_core/src/bead crates/sase_core_py/src/beads`.
- No wire change. Use a non-breaking conventional commit subject, e.g.
  `fix(bead): keep published bead ids stable when relocating duplicate creations`.

## Phase `launch-guard`: identity-verified relocation handling (sase)

1. **Helpers in `src/sase/bead/relocation.py`** (keep module imports light):
   - `relocations_for_subtree(root_id, relocations)` returns only records whose `old_id`
     is `root_id` or starts with `f"{root_id}."`.
   - `resolve_own_bead_id(show, before, relocations) -> str`, where `show` is a
     `Callable[[str], Issue]` such as `proj.show` and `before` is the bead as read
     _before_ publication:
     - Let `candidate = resolve_created_bead_id(before.id, relocations)`. If
       `candidate == before.id`, return it.
     - Otherwise compare creation identity (`issue_type`, `title`, `created_at`,
       `created_by`; never mutable fields such as status or assignee).
     - If `show(candidate)` matches `before`, return `candidate` (our bead moved).
     - Elif `show(before.id)` still matches, return `before.id` (the relocation moved
       someone else's bead; ignore it).
     - Otherwise raise a `ValueError` subclass naming both IDs.
   - Make `rewrite_text_for_bead_relocations` token-safe:
     - Rewrite in a single regex pass over a dict, so `{a→b, b→c}` never turns `a` into
       `c`.
     - Match only when the ID is not preceded by `[A-Za-z0-9_-]` and not followed by
       `[A-Za-z0-9_]`. A following `.` stays allowed, so `old.3` and `old.land` are
       rewritten, but `sase-17v` inside `sase-17v1` is not.
     - Its remaining caller is `rewrite_head_subject_for_bead_relocations`.
2. **`launch_epic_bead_work` (`src/sase/bead/cli_work_handler.py`)**, after the
   `graph_publication` stage:
   - Delete the block that resolves `epic_id` and calls
     `rewrite_text_for_bead_relocations` on `query` and on `segment_env`.
   - Compute `own = relocations_for_subtree(epic_id, graph_relocations)` and
     `moved_epic_id = resolve_own_bead_id(proj.show, issue, own)`. `issue` is the epic
     read at the top of the function.
   - Unchanged ID: launch exactly as today with the original query, env, and
     `expected_names`. This alone would have launched the 2026-09-24 epic on its first
     attempt.
   - Changed ID: do **not** launch.
     - Map `rollback_preclaims` through `own` with
       `dataclasses.replace(prior, bead_id=resolve_created_bead_id(prior.bead_id, own))`.
     - Call
       `rollback_work_launch(proj, moved_epic_id, marked_ready_this_run=..., rollback_preclaims=<mapped>, no_push=no_push or defer_push)`.
     - Raise a new `EpicGraphRelocatedError(BeadWorkError)` with `original_epic_id`,
       `relocated_epic_id`, `bead_relocations=own`, `graph_published=True`, and
       `agents_spawned=False`.
     - Message:
       `epic <old> was renumbered to <new> during publication because <old> was already published by another clone; no agents were spawned. Resume with: sase bead work <new>`.
   - If `resolve_own_bead_id` cannot locate the epic, roll back with the unmapped IDs
     only if `show(epic_id)` is still ours. Either way, raise `BeadWorkError` with a
     `sase doctor -v` hint. Never guess.
   - Export `EpicGraphRelocatedError` next to `BeadWorkError`.
3. **Resume callback (`src/sase/bead/cli_work_from_plan_resume.py`)**: make
   `publish_resumed_graph` return the `checkpoint_and_publish_graph(...)` result, so the
   handler sees relocations for resumed launches.
4. **Task path**:
   - Make `checkpoint_task_work_launch` (`src/sase/bead/cli_work_commit.py`) return a
     `LaunchCheckpointResult`. It is already truthy on push, so bool-style callers keep
     working, and it will carry `bead_relocations`.
   - In `launch_task_bead_work` (`src/sase/bead/cli_work_task.py`), after publication,
     run `resolve_own_bead_id` on the task.
   - If the task moved:
     - call `rollback_task_work_launch` on the moved ID (zero-spawn);
     - raise the task-work error with
       `task <old> was renumbered to <new> during publication; resume with: sase bead work <new>`.
   - Why this matters: after `core-winner`, an unpublished task that collides moves, and
     an agent named after the old ID would try to claim someone else's bead.
5. **Tests**:
   - In `tests/test_bead/test_cli_work_epic_checkpoint.py`, replace
     `test_work_rewrites_launch_query_and_env_after_graph_relocation` with two tests:
     - _Foreign relocation (incident regression)._ `before_agent_launch` reports
       `{epic_id → sase-99}`, but the store still holds the epic at `epic_id`. The
       launch proceeds, and `result.epic_id == epic_id`.
     - _Own relocation._ The epic actually lives at the new ID after publication. Build
       that store state for real where practical; otherwise patch `resolve_own_bead_id`.
       Assert `EpicGraphRelocatedError`, that no launch happened, and that the rollback
       ran on the moved IDs.
   - Make the fake `launch_bead_work_agents` in these tests enforce the real invariant.
     Split the query into segments, collect `%name` via
     `sase.agent.multi_prompt_references.extract_static_name_directive`, and assert that
     the set equals `expected_names`. A fake must never again hide a name mismatch.
   - In `tests/test_bead/test_relocation.py`, add unit tests for
     `relocations_for_subtree`, `resolve_own_bead_id` (own move, foreign move, and
     unlocatable), and the token-safe rewrite (child suffixes, no prefix hits, no
     chained rewrites).
   - Add a task-path test for a moved task.
6. Run `sase tool run check`.

## Phase `plan-retry`: automatic recovery for approved-plan launches (sase)

1. **`create_and_launch_epic_from_plan` (`src/sase/bead/epic_from_plan.py`)**:
   - When the launch raised `EpicGraphRelocatedError`, `_rollback_epic_creation` must
     remove `exc.relocated_epic_id` (cascading to its phases). It must never remove
     `epic.id`, which at that point belongs to another clone's published bead.
   - Add `relocated_epic_id: str | None` to `EpicFromPlanError`. Keep
     `graph_published=True` and `rollback_performed=True`.
   - The existing plan restore puts back the original content, which removes the
     `bead_id` frontmatter this attempt added.
2. **`work_from_plan_file_locked` (`src/sase/bead/cli_work_from_plan_launch.py`)**,
   fresh-creation branch:
   - Wrap `create_and_launch_epic_from_plan` in a bounded loop (a module constant of 3
     attempts in total).
   - Retry only when all three hold: the failure carries `relocated_epic_id`, rollback
     reported no errors, and `hooks.publish_epic_rollback(store)` succeeded.
   - Before a retry, render
     `↻ Epic ID <old> collided with a concurrently published bead (moved to <new>); rolled back, retrying (attempt 2/3)`.
   - The retry mints fresh IDs, because the integrated store's counter is already past
     both beads.
   - Reset per-attempt state (`launched_names`, `preserved_names`, `launch_state`, and
     any relocation capture).
   - Keep `stale_epic_id` unchanged across attempts: rollback restores the original plan
     content, including the stale link being replaced.
   - Any other failure, a failed rollback or rollback publication, or exhaustion keeps
     the existing `error_with_resume` path.
   - Replace
     `timer.fields["bead_id"] = resolve_created_bead_id(created.epic.id, published_relocations)`
     with the launched epic's verified ID, and drop the now-unused
     `published_relocations` plumbing.
   - Record a `relocation_retries` timer field so collisions show up in
     `tui_launch_timing.jsonl`.
3. **Resume branch (`resume_linked_epic`)**: this covers an epic created earlier but
   never published (for example under `--no-push`) that moves on first publication. On
   `EpicGraphRelocatedError`:
   - Rewrite the archived plan's `bead_id` frontmatter to `relocated_epic_id`. Use
     `set_frontmatter_fields` and commit through the resume path's plan-commit hook
     (thread `hooks.write_and_commit_plan_file` in if it is not already available).
   - Let the existing `graph_published and not agents_spawned` branch publish the
     rollback.
   - Raise the resume error: re-running `sase bead work <plan>` resumes the moved epic.
     No automatic retry, because this epic predates the current invocation.
4. **Tests**: find the existing plan-file launch tests with
   `rg -ln "create_and_launch_epic_from_plan|work_from_plan_file_locked" tests/`.
   - First attempt relocated: rollback removes the moved ID, never the original. A
     foreign bead seeded at the original ID keeps its title and state. The plan's
     `bead_id` is restored, the rollback is published, and the second attempt launches.
   - Relocated on every attempt: stops after 3 with the resume error, and the plan is
     restored.
   - Resume path: the plan is relinked to the moved epic, and the error names it.
5. Run `sase tool run check`.

## Phase `pin-regression`: core pin, git-backed regressions, incident cleanup (sase)

1. Once the `core-winner` commit is on sase-core's remote default branch, move
   `sase-core-revision.txt` to it or later (`just ratchet-core-revision`). See "The CI
   source revision pin" in `docs/rust_backend.md`. Make sure the local `sase_core_rs` is
   rebuilt from the linked source (setup rebuilds automatically when linked sources
   change).
2. In `tests/test_bead/test_conflict_resolver_streams.py`, add a git-backed test next to
   `test_duplicate_top_level_creations_report_typed_relocation`:
   - the upstream branch creates the bead at `...:05Z` and the local branch at `...:00Z`
     (so the local creation is **older**);
   - expect that the upstream bead keeps the ID and title, the local bead relocates, and
     `relocation.old_id == local.id`.
   - Also sweep `rg -n "relocat" tests/` (e.g. `test_sync_conflict_recovery.py`,
     `test_task_beads.py`) for assumptions about the old rule.
3. If the existing sync fixtures support a bare remote with two clones, add a sync-level
   regression that mirrors the incident:
   - clone A creates `X` at T0 and does not push;
   - clone B creates `X` at T1 > T0 and pushes;
   - clone A publishes. Expect B's `X` unchanged on the remote, A's bead relocated, and
     the relocation record naming A's bead.
4. **Incident cleanup**:
   - If `src/sase/feature_flags/registry.py` on master maps `FeatureFlag.tool_handoff`
     to `bead="sase-17v"`, change it to `bead="sase-17w"`. The flag bead "Retire
     tool_handoff" was renumbered by the incident, and `sase-17v` no longer exists.
     Confirm with `sase bead read sase-17w -r "<why>"`.
   - If the `tool_handoff` entry is not on master yet (its epic `sase-17p` is still in
     flight), record a `PROPOSED FOLLOW-UP:` note on this phase bead with the exact
     change instead.
5. Run `sase tool run check`.

## Out of scope / possible follow-ups

- **`compose_bead_relocations` chain conflation.** A top-level relocation `X→Y`
  followed, in a later rebase round, by `Y→Z` for a _different_ local bead that also
  minted `Y` gets composed into `X→Z`. This needs several unpublished local commits plus
  an allocator collision, and it has not been observed. Phase workers who confirm it
  should record a `PROPOSED FOLLOW-UP:` note.
- **Reserving the epic ID by publishing the bare epic right after creation.** This would
  remove even the retry, at the cost of one extra publication (about 3–9 s) on every
  launch. It is not worth it while collisions are rare. Revisit if `relocation_retries`
  shows up in the launch timing log.
- **Commit-subject rewrite in `src/sase/bead/sync_worker.py`.** It is correct once
  `core-winner` lands, because every relocation is then a local bead. No change is
  needed beyond the token-safe rewrite.
