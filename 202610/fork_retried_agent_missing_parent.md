---
tier: tale
title: Fix forks and monitor continuations of retried agents
goal:
  "Agents whose run was retried in-process can be continued again through #fork and
  monitor follow-ups. New retry attempts no longer list the archived failed attempt as
  their parent, and replay splices that edge out of continuation nodes already on disk."
size: medium
proposed_by: bbugyi200.athena.0xz
create_time: 2026-10-07 14:10:08
status: wip
---

# Fix `#fork` and monitor continuations of retried agents (`missing_parent` refusal)

## Problem

Any agent run that was retried in-process (provider retry or fallback model) and then
finished cannot be continued automatically. Both `#fork:<name>` and monitor follow-up
turns fail at launch in the `load_history` step:

```
ContinuationReplayRefusal: missing_parent: parent id has no continuation record ·
node `agent-delta:<run>:<retry-node>`, parent `agent-delta:<run>:<failed-attempt-node>`
```

Confirmed incidents, all with the same root cause:

- 2026-10-07 `research.3x.image` (`#fork:research.3x.final`, project
  `gh_bobs-org__bob-cli`). The parent's attempt 1 failed on a finalizer git-push 500;
  attempt 2 completed.
- 2026-10-03 `0vv--1` (`#fork:0vv--code`). That parent ran three attempts, archived in
  `attempts/01` and `attempts/02`.
- 2026-09-13 `sase-10h.3--1`, a monitor follow-up whose starter was a retried run.

A scan of `~/.sase/projects/*/artifacts` found 15 retried runs since 2026-09-11 whose
root continuation node is parented on an archived attempt: 11 completed, 3 interrupted
(monitor handoffs), and 1 failed. Every one of them is currently unforkable.

## Root cause

1. When an attempt fails, `handle_workflow_error`
   (`src/sase/axe/run_agent_exec_retry.py`) calls
   `persist_agent_delta_best_effort(..., status="failed")`. That writes the failed node
   and sets the root `agent_meta.json` node pointers: `continuation_node_id`,
   `continuation_node_ref`, `continuation_manifest_ref`, and the rest of the group.
2. `_snapshot("failed")` then calls `snapshot_attempt`
   (`src/sase/axe/run_agent_exec_attempts.py`), which moves the whole `continuation/`
   directory into `attempts/<N>/continuation/`. The root meta pointers are left in
   place, now dangling.
3. When the next attempt finishes, `persist_agent_delta` builds `parent_ids` with
   `_parent_node_ids` (`src/sase/continuation_capture/agent_delta.py`). That function
   merges the on-disk `agent_meta.json` and includes `continuation_node_id`, so the new
   node's parents become `[<archived failed attempt node>, *launch parents]`.
4. Fork replay (`_ReplayBuilder` in `src/sase/history/chat_fork/continuation/replay.py`)
   resolves parents through `ContinuationNodeIndex` (`.../continuation/_source.py`). The
   index only reads `<artifact_dir>/continuation/` (plus same-day sibling runs, starter
   dirs, and portable refs) and never `attempts/*/continuation/`. The parent stays
   unresolved, the Rust planner emits a `missing_parent` omission, and that omission
   kind is in `AUTOMATIC_REFUSAL_OMISSIONS`, so the launch is refused.
5. Making the archived node resolvable would not be enough. It has `status: failed` and
   no checkpoint, so as an essential parent it triggers
   `failed_starter_without_checkpoint` in `_assess.py`.

**The edge itself is wrong.** An in-process retry or fallback starts a fresh provider
session. Only `_RETRY_CONTINUATION_NUDGE` and the on-disk workspace carry over. The
failed attempt's delta (a duplicate of the authored request, with no response) is not
part of the retry's context. It is a superseded sibling within the same run, not a
parent. Its own parents are the run's launch parents, which the retry already inherits
through `continuation_parent_node_ids`.

## Changes

### 1. Writer: a new attempt does not inherit the archived attempt's node pointer

- In `src/sase/continuation_capture/agent_delta.py`, define one module-level tuple of
  the attempt-scoped node-pointer keys that `persist_agent_delta` writes to
  `agent_meta.json`:
  - `continuation_node_id`
  - `continuation_manifest_ref`
  - `continuation_manifest_path`
  - `continuation_agent_delta_ref`
  - `continuation_node_ref`
  - `continuation_status`
  - `continuation_portable_content_ref`
  - `continuation_node_portable_ref`

  Build the existing `fields` dict from those same names so the two lists cannot drift.

- Add a small public helper, for example
  `release_attempt_continuation_pointers(state, artifacts_dir)`, and export it from
  `sase.continuation_capture`. It removes those keys with
  `update_agent_meta_fields(artifacts_dir, {}, remove_keys=...)` and resets
  `state.continuation_node_id` and `state.continuation_manifest_ref` to `None`.
  - Do **not** remove launch-lineage keys (`continuation_parent_node_ids`,
    `continuation_parent_node_id`, `continuation_parent`,
    `continuation_parent_portable_refs`). They are the retry node's correct parents.
  - Do **not** remove workspace or prepared-prompt keys (`continuation_workspace_ref`,
    `continuation_prepared_prompt_ref`, and so on). The retry path re-persists them
    through `persist_workspace_facts_best_effort` and `ensure_prepared_prompt`.
- In `handle_workflow_error`, call the helper immediately after `_snapshot("failed")` in
  the two branches that start another attempt: the wait-and-retry branch, before the
  spawn-on-retry check, and the fallback-model branch. Leave the terminal `raised`
  snapshots alone; failed-run fork behavior is out of scope.
- Leave `_parent_node_ids` unchanged. Once the stale pointer is gone it yields exactly
  the launch parents.

### 2. Reader: splice out legacy superseded-attempt edges during replay

Nodes already on disk are content-addressed, carry recorded digests, and are registered
as portable locators, so they must not be rewritten. The reader has to tolerate them.
This is permanent read-side tolerance for immutable records, not a user-facing behavior
choice, so **no feature flag**. A prototype of exactly this splice, run in memory
against the real `research.3x.final` and `0vv--code` artifacts, cleared both refusals
with no omissions and rendered only the completed node.

- `ContinuationNodeIndex` (`src/sase/history/chat_fork/continuation/_source.py`):
  - When `observe_dir` indexes a run, also read
    `<artifact_dir>/attempts/*/continuation/nodes/*.json`, bounded by
    `MAX_HYDRATION_NODES`.
  - Record each archived node's raw record in a new map, for example
    `_superseded_by_id`. No content loading is needed, because only `parent_ids` and
    `owner` matter.
  - Never put these nodes in `_by_id`. `resolve()` must not return them, and they must
    never be rendered.
- Add `ContinuationNodeIndex.canonical_parent_ids(node) -> list[str]`. For each parent
  id:
  - If the parent is in `_superseded_by_id` **and** its `owner.run_id` equals the child
    node's `owner.run_id`, replace it with that archived node's own canonical parent
    ids. This must be recursive, cycle-safe, deduplicated, and order-preserving, because
    attempt 3 points to attempt 2, which points to attempt 1.
  - Otherwise keep the parent unchanged.

  This keeps the fail-closed guarantee. Only ids proven to be same-run archived attempts
  are spliced; any other unresolved parent still reaches the Rust planner and still
  refuses.

- `_ReplayBuilder.hydrate_parents()` (`replay.py`): after the index has observed the
  seed dirs and siblings, rewrite each record's `parent_ids` through
  `index.canonical_parent_ids` before calling `hydrate_missing_parents`.
- `hydrate_missing_parents` (`_source.py`): canonicalize each hydrated node's
  `parent_ids` before storing it and enqueueing grandparents. A hydrated ancestor that
  was itself a retried run is then handled too.
  - Keep the `_HydratedNode` record consistent with the rewritten parents.
  - `delayed_starter_parent_ids` already rewrites `parent_ids` at load time, so this
    follows existing precedent.
- Do not add a new omission entry. Omissions render into the injected history, and a
  superseded retry attempt is not missing context.
- No `sase-core` or Rust change is needed. The planner keeps receiving well-formed
  records, and node loading and hydration already live in this Python layer.

### 3. Tests

- `tests/axe/test_run_agent_exec_attempts_integration.py`:
  - **Retry branch:** seed `agent_meta.json` with
    `continuation_parent_node_ids: ["agent-delta:launch:parent"]`, then run
    `handle_workflow_error`, which returns `"continue"`.
    - Root `agent_meta.json` no longer has `continuation_node_id`,
      `continuation_node_ref`, or `continuation_manifest_ref`.
    - The launch-parent keys survive.
    - `state.continuation_node_id is None`.
    - A follow-up
      `persist_agent_delta(ctx, state, status="completed", final_response=...)` produces
      a node whose `parent_ids` equals the launch parents and does not include the
      attempt-1 node id, which can be read from
      `attempts/01/continuation/manifest.json`.
  - **Fallback branch:** the same pointer-release assertion.
- `tests/history/test_continuation_replay_hydration.py`. Reuse the existing helpers
  (`_publish_agent_delta`, `_agent_member`, `_write_agent_node`):
  - **Legacy shape heals.** In one run dir, publish a `failed` delta, archive it with
    `snapshot_attempt`, then write a completed root node whose `parent_ids` is
    `[<archived id>]`, matching what the old writer produced.
    - `replay_versioned_continuation_history([...])` has no refusals and no
      `missing_parent` omission.
    - The rendered history contains the completed response and no block for the failed
      attempt.
    - `build_fork_injected_history(sources, automatic=True)` succeeds.
  - **Multi-attempt chain.** Attempt 3 is parented on attempt 2, which is parented on
    attempt 1, and both earlier attempts are archived. The replay still has no refusals.
  - **Launch ancestry survives the splice.** Give the archived attempt a parent in a
    predecessor run, published as a sibling run dir. That predecessor node is hydrated
    and rendered parent-first.
  - **Negative, unknown parent.** A parent id that is not under any `attempts/` dir
    still yields a `missing_parent` refusal.
  - **Negative, cross-run.** An archived node whose `owner.run_id` differs from the
    child's is not spliced and still refuses.
- **End-to-end regression.** Run the retry flow (`handle_workflow_error` →
  `"continue"`), then a completed `persist_agent_delta`, then fork replay with
  `automatic=True`. It launches without refusals.

### 4. Verification

- Read `sase/memory/lint_and_test.md` with `/sase_memory_read` before finishing and run
  the verification it prescribes (`just check` through `sase tool run`).
- Manual check against real data, read-only and run from any directory. Call
  `replay_versioned_continuation_history` with a source like this:

  ```
  {"kind": "agent", "name": "research.3x.final", "outcome": "completed",
   "artifact_dir": "~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202610/07/20261007100742",
   "path": "~/.sase/chats/202610/gh_bobs_org__bob_cli-ace_run-research_3x_final-261007_100742.md"}
  ```

  Expand `~` before calling. Confirm `refusals == ()` and that exactly one node, the
  completed `agent-delta:20261007100742:10aa5a7f0388ae92`, is rendered. Do not modify
  anything under `~/.sase`.

## Out of scope

- Why the parent was retried at all: a host finalizer git-push 500 matched Claude's
  `"Internal server error"` retry pattern and re-ran the whole agent. This is tracked
  separately as task bead `sase-1he`.
- Fork behavior for terminally failed (`raised`) runs, and cross-day ancestors that
  hydrate only through portable refs.
