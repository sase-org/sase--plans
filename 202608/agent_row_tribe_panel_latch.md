---
tier: tale
title: Stop the Agents tab latching a pre-metadata row into the wrong tribe panel
goal:
  A live agent row can never be stranded in a tribe panel that contradicts its own
  agent_meta.json, because the Tier-1 merge identity is stable across metadata
  completion and a PID collapse never discards the fresher row's structural placement
  fields.
size: medium
proposed_by: bbugyi200.athena.0fz
---

# Plan: Stop the Agents tab latching a pre-metadata row into the wrong tribe panel

## 1. The Defect

A running agent shell (`toobig-4j.test_workflow_executor.0--1`) rendered as an
untethered top-level row in the `@default` Agents-tab panel while its own clan
(`toobig-4j`) and family (`toobig-4j.test_workflow_executor.0`) rendered correctly in
the `@chop` panel. The stranded row still displayed the agent's real name, so it read as
a second, homeless copy of an agent that visibly belonged somewhere else.

This is not cosmetic drift that self-heals on the next refresh. Once the bad row is in
the cached list it is never replaced, so the misplacement persists until a full-history
reload.

## 2. Root Cause (Confirmed, Not Suspected)

### 2.1 The projection math is correct — this was ruled out first

Building `Agent` rows from the five real `agent_meta.json` files of that clan generation
and running the real projection (`project_clan_tree` -> `panel_keys_for`) puts every
row, including `...0--1`, in the `chop` panel at the right tree depth. The tree/panel
code in `src/sase/ace/tui/models/_agent_tree.py` and
`src/sase/ace/tui/models/agent_panels.py` needs no change. The bad row therefore did not
have the metadata those files read.

Two properties decide placement, and both come from the row, not the projection:

- `src/sase/ace/tui/models/_agent_tree.py:246` (`_clan_for_row`) puts a row in a clan
  subtree only when `agent.agent_clan` is set. Without it the row never joins the clan
  container, so `_panel_key_for_agent` (`src/sase/ace/tui/models/agent_panels.py:99`)
  resolves the row's presentation anchor to itself, reads `tribe is None`, and files it
  under `@default`.
- `src/sase/ace/tui/models/agent.py:433` (`is_child_row`) is driven purely by
  `parent_timestamp` / `parent_workflow`. Without `parent_timestamp` the row renders as
  a titled root instead of a family-member child.

So the rendered row had `agent_clan is None` and `parent_timestamp is None` while its
`agent_meta.json` on disk had both.

### 2.2 How a pre-metadata row survives forever

Both defects are in the Tier-1 patch path and the dedup safety net.

**Defect A — the Tier-1 merge identity is derived from mutable relationship state.**
`_tier1_merge_key` (`src/sase/ace/tui/actions/agents/_loading_compute_merge.py:71`)
buckets a row as `("identity", agent_type, cl_name, raw_suffix)` when it has no parent
link, but as `("artifact-followup", agent_type, raw_suffix, parent_timestamp)` once
`parent_timestamp` appears. The same artifact directory therefore changes merge identity
the moment the runner finishes writing its metadata. In
`merge_incomplete_load_after_complete_history`:

- line 369, `replacement = incoming_by_key.get(_tier1_merge_key(cached), cached)` misses
  and keeps the **stale** cached row;
- line 338's new-row loop sees the fresh row's key as unknown and emits it _as well_.

Result: two rows for one artifact directory.

**Defect B — the PID collapse keeps the stale row and drops its placement.**
`dedup_by_pid` (`src/sase/ace/tui/models/_dedup.py:433`) then finds both rows under the
same `(pid, raw_suffix)`. `_collapse_pid_duplicate`
(`src/sase/ace/tui/models/_dedup.py:415`) has no RUNNING-vs-RUNNING preference, so it
falls through to `return existing, agent` and keeps whichever row came first — the stale
one, because the cached list is sorted newest-first and the stale row is the same
(newer) artifact. `_merge_agent_fields` (`src/sase/ace/tui/models/_dedup.py:53`) then
merges the fresh row into it, and its whitelist copies `agent_name`, `model`,
`workspace_num`, `runner_is_live` and friends but **none** of `parent_timestamp`,
`agent_family`, `agent_family_role`, `role_suffix`, `agent_clan`,
`agent_clan_generation`, `clan_tribe`, `clan_summary`, `clan_context` or `tribe`.

That is exactly the observed row: correct name, correct RUNNING status, no clan, no
family, no tribe.

### 2.3 Reproduction

This is confirmed, not inferred. Two rows for one artifact directory, one captured
before the runner wrote its family/clan metadata and one after:

```python
stale = Agent(agent_type=AgentType.RUNNING, cl_name="gh_sase-org__sase", ...,
              raw_suffix="20260829072911", pid=3473413, runner_is_live=True)
fresh = Agent(..., raw_suffix="20260829072911", pid=3473413, runner_is_live=True,
              agent_name="toobig-4j.test_workflow_executor.0--1",
              parent_timestamp="20260829061545",
              agent_family="toobig-4j.test_workflow_executor.0",
              agent_family_role="root", role_suffix="--1",
              agent_clan="toobig-4j", agent_clan_generation="20260829061525")

_tier1_merge_key(stale)  # ('identity', RUNNING, 'gh_sase-org__sase', '20260829072911')
_tier1_merge_key(fresh)  # ('artifact-followup', RUNNING, '20260829072911', '20260829061545')
# -> keys differ, so the cached row is never replaced

kept = dedup_by_pid([stale, fresh])
# len(kept) == 1 and kept[0] is stale
# kept[0].agent_name   == 'toobig-4j.test_workflow_executor.0--1'   (copied)
# kept[0].agent_clan   is None                                       (dropped)
# kept[0].parent_timestamp is None                                   (dropped)
# kept[0].tribe        is None                                       (dropped)
# kept[0].is_child_row is False
```

The upstream race that produces the pre-metadata snapshot (a workspace `RUNNING` claim
row from `load_agents_from_running_field`, or an artifact-index row indexed before the
runner's `extract_directives_and_write_meta` write, both of which build an `Agent` first
and enrich from whatever `agent_meta.json` exists at that instant) is inherent to
starting an agent and is _supposed_ to be transient. The bug is that the TUI latches it
permanently. That latch is what this plan removes.

## 3. Implementation

### 3.1 Make the Tier-1 merge identity stable across metadata completion

In `src/sase/ace/tui/actions/agents/_loading_compute_merge.py`:

1. Add a stable secondary key for non-workflow-step rows:
   `("artifact-row", agent.agent_type, agent.project_file, agent.raw_suffix)`, returning
   `None` when `raw_suffix` is `None` or `parent_workflow` is not `None`. Workflow-step
   rows keep the existing `"artifact-step"` key untouched — several step rows
   legitimately share a parent's `raw_suffix`, so their discriminators are load-bearing.

   Scope the key by `project_file`, not `cl_name`. `project_file` is fixed when the row
   is constructed, whereas `cl_name` is transient for plan-chain children (see
   `test_incomplete_merge_replaces_plan_chain_child_with_transient_cl_name`). Project
   scoping is also the established dedup doctrine — see the module docstring of
   `tests/test_agent_loader_dedup_cross_project_collision.py`, which exists because two
   projects can launch in the same clock second.

2. Build an `incoming_by_stable_key` index alongside `incoming_by_key` (line 252) and a
   `cached_stable_keys` set alongside `cached_keys` (line 301). Drop any stable key that
   more than one row in the same collection claims, so an ambiguous key never drives a
   replacement.

3. In the new-row loop (line 338), also skip an incoming row whose stable key is already
   present in `cached_stable_keys`. This stops the duplicate being emitted at all.

4. In the cached loop (line 369), fall back to `incoming_by_stable_key` when the primary
   key misses, so the cached pre-metadata row is _replaced by_ the fresh row rather than
   surviving beside it.

Keep the primary key semantics exactly as they are; the stable key is a recovery path,
not a replacement. This keeps the change surgical and lets every existing merge test
stand as a guard.

### 3.2 Never drop structural placement on a PID collapse

In `src/sase/ace/tui/models/_dedup.py`, extend `_merge_agent_fields` (line 53) to carry
the row's structural placement fields from `source` to `target` when the target lacks
them, using the same "only fill what is empty" rule the rest of the function uses:

- `parent_timestamp`
- `agent_family`, `agent_family_role`, `role_suffix`, `plan_chain_root`
- `agent_clan`, `agent_clan_generation`, `clan_tribe`, `clan_summary`, `clan_context`
- `tribe`

Do **not** copy `parent_workflow` or the step fields: those identify a workflow step
row, and copying them across a collapse would change a row's kind rather than complete
its placement.

This is defence in depth. After 3.1 the duplicate should not reach `dedup_by_pid` on
this path, but `_merge_agent_fields` is shared by the axe, patch, VCS-claim and
RUNNING/WORKFLOW collapse paths (lines 245, 340, 421, 427, 429, 482, 496), so any future
duplicate source gets the same guarantee for free.

Placement is recomputed after dedup — `merge_incomplete_load_after_complete_history`
runs `_normalize_relationships_after_merge` -> `sort_and_reorder` ->
`project_clan_tree`, and `normalize_loaded_agents` runs `sort_and_reorder` after
`_deduplicate` — so a row that gains `parent_timestamp` / `agent_clan` during the
collapse is re-bucketed as a family child and re-projected into its clan in the same
pass. No extra re-projection call is needed; confirm this holds rather than adding one.

### 3.3 Tests

Add regression coverage at three levels. Prefer extending the existing files; create a
new focused file only if a target file would exceed the repo's `toobig` line limits.

1. **Merge-key stability** — `tests/test_agents_tab_incomplete_merge.py`: a cached row
   with no `parent_timestamp` and a fresh Tier-1 row for the same
   `(agent_type, project_file, raw_suffix)` that has `parent_timestamp`, `agent_family`
   and `agent_clan` must merge to exactly one row, and that row must carry the fresh
   metadata. Assert the count, so a regression that re-introduces the duplicate fails
   here rather than silently downstream.

2. **Lossy-collapse guard** — `tests/test_agent_loader_dedup_pid_safety_net.py`: pass
   the stale/fresh pair from section 2.3 through `dedup_by_pid` and assert the surviving
   row has `agent_clan`, `agent_clan_generation`, `parent_timestamp`, `agent_family`,
   `role_suffix` and `tribe` populated and `is_child_row is True`, in **both** input
   orders.

3. **The user-visible invariant** — the strongest test, and the one that would have
   caught this: after the Tier-1 merge, feed the merged rows through `project_clan_tree`
   and `panel_keys_for` and assert the agent lands in the clan's tribe panel
   (`"chop"`-style) and that `None` (`@default`) is not among the returned panel keys.
   Place it with the other merge tests so it exercises the real merge output rather than
   hand-built rows.

Also confirm the cross-project guard still holds: two rows with the same `raw_suffix`
but different `project_file` must not be merged by the new stable key. Extend
`tests/test_agent_loader_dedup_cross_project_collision.py` if the new key is not already
covered there.

## 4. Out Of Scope

- Do not change `project_clan_tree`, `_clan_for_row`, `panel_keys_for` or
  `_panel_key_for_agent`. They were verified correct against the real metadata and are
  the oracle this fix is measured against.
- Do not try to eliminate the upstream start-up race that produces a pre-metadata row. A
  claim row that briefly precedes `agent_meta.json` is legitimate; making the TUI stop
  latching it is the correct fix and the smaller one.
- Do not add a new refresh path or a forced full reload as a workaround. Per
  `sase/memory/tui_perf.md` rule 5, refreshes go through the existing fast path, and a
  full agent-list rebuild is the most expensive UI operation there is.

## 5. Verification

- The three test groups in section 3.3 must fail before the fix and pass after. Write
  them first and watch them fail — the whole point is that the current code passes its
  existing suite while producing this row.
- Run `just check` (see `sase/memory/lint_and_test.md`). Run `just install` first if
  this workspace clone's virtualenv is stale.
- These files sit in the ACE agent-loading hot path, so run `just check-full` through
  the `/sase_monitor` skill before landing, using the `TESTING` / `TESTED` status pair.
- The change is pure in-memory list bookkeeping with no new I/O, no new render-path work
  and no new refresh path, so it introduces no TUI performance risk. Adding a per-row
  disk read, a re-projection pass or an extra reload to "repair" placement would, and is
  explicitly not the fix.
