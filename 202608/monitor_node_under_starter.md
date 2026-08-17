---
tier: tale
title: Render every agent-family monitor as one node under the agent that started it
goal:
  Every monitor an agent runs inside an agent family shows up as exactly one gear-glyph
  node nested under the agent row that started it, that starter row no longer carries
  the orange monitor count badge, and ancestor clan/family container rows keep their
  aggregate count.
size: medium
proposed_by: bbugyi200.athena.04l
create_time: 2026-08-17 08:21:29
status: wip
---

# Render every agent-family monitor as one node under the agent that started it

## Problem

When a SASE agent inside an agent family runs a monitor, the Agents tab sometimes shows
no monitor node at all. Instead, the agent that started the monitor grows a small orange
`⚙1` count badge, and the monitor's own row is missing from the tree entirely. It works
in other cases, which is why the failure reads as inconsistent.

Observed shape (real session, `sase ace` Agents tab):

```
@epic
  sase-ns.6.6.6 (WAITING) [W1 D4] ⚙1                 <- clan container
    sase (TALE DONE) ×8 −3 ⚙1 ◆ sase-ns.6.6.6.1      <- family container (depth 1)
      main (TALE APPROVED) ◆ sase-ns.6.6.6.1--plan   <- depth 2
      sase (TALE DONE)     ◆ sase-ns.6.6.6.1--code
      sase (TALE DONE)     ◆ sase-ns.6.6.6.1--1
      sase (TALE DONE) ⚙1  ◆ sase-ns.6.6.6.1--2      <- started the monitor; badge, no node
                                                     <- the --mon-1 monitor row is missing
```

## Root cause (verified against live artifacts and by reproduction)

The monitor row is loaded, is linked to its starter in memory, and is then dropped by
the display pipeline before it can be rendered.

1. **Durable metadata.** `create_monitor_member` (`src/sase/monitor/member.py`) builds
   the monitor member through `create_followup_artifacts`
   (`src/sase/axe/run_agent_helpers_artifacts.py:123`), which records
   `parent_timestamp = <starter artifact timestamp>` and copies a fixed key list that
   does **not** include `agent_clan` / `agent_clan_generation`. Live evidence from the
   screenshot's session:
   - `.../ace-run/202608/17/20260817071511/agent_meta.json` (the monitor):
     `name=sase-ns.6.6.6.1--mon-1`, `agent_family=sase-ns.6.6.6.1`,
     `agent_family_role=monitor`, `parent_timestamp=20260817070811`, `agent_clan` and
     `agent_clan_generation` **absent**.
   - `.../202608/17/20260817070811/agent_meta.json` (the starter `--2`):
     `agent_clan=sase-ns.6.6.6`, `agent_clan_generation=20260817055518`,
     `parent_timestamp=20260817055518` (the family root), so the starter is itself a
     child row.

2. **The existing workaround cannot fire inside a clan.**
   `normalize_monitor_family_display_parents`
   (`src/sase/ace/tui/models/_agent_status_family_core.py:255`) rewrites the monitor's
   in-memory `parent_timestamp` to its family root, but only when
   `(project_file, agent_clan_generation, agent_family)` resolves to exactly one loaded
   family root. The root carries a clan generation and the monitor carries `None`, so
   the key never matches and the monitor keeps pointing at `--2`. This is exactly the
   "sometimes it works" split: monitors started by a family root, or started inside a
   family with no clan generation, still resolve and render; monitors started by a
   mid-family continuation inside a clan do not.

3. **Ordering drops grandchild follow-ups.** `sort_and_reorder`
   (`src/sase/ace/tui/models/_agent_ordering.py:108-146`) removes every family-member
   child from `sorted_agents` into `followups_by_parent`, then walks only the remaining
   top-level rows and splices in **one** level of follow-ups. `--2` is itself a
   follow-up, so `followups_by_parent["20260817070811"]` is never popped and the monitor
   row never reaches the output list.

4. **Two more layers would drop it even if it were emitted.** `project_clan_tree`
   (`src/sase/ace/tui/models/_agent_tree.py:357-394`) builds its parent index from
   non-child rows only and therefore caps nesting at depth 2, and
   `filter_agents_by_fold_state` (`src/sase/ace/tui/models/_fold_filter.py:18-22`)
   registers fold owners from non-child rows only, so a row whose parent is a child row
   resolves to `parent is None` and is filtered out.

5. **The badge is the visible symptom of the same link.** `_attach_runtime_children`
   (`src/sase/ace/tui/models/_agent_ordering.py:180-196`) indexes _every_ row by
   `raw_suffix`, including child rows, so the monitor is attached to
   `--2.runtime_children`. That makes `running_monitor_count` return 1 and makes
   `is_sequential_family_container`
   (`src/sase/ace/tui/models/agent_family_members.py:62`) classify `--2` as a container
   (its only "family member child" is the monitor), which is what renders `⚙1` at
   `src/sase/ace/tui/widgets/_agent_list_render_agent.py:478-486`.

Reproduction (already confirmed; keep as the starting regression test):

```python
# root (family root, in clan) -> --2 (family member child) -> --mon-1 (monitor)
ordered = sort_and_reorder([root, code, mon], [])
# ordered rows: clan container, sase-ns.6.6.6.1 (depth 1), sase-ns.6.6.6.1--2 (depth 2)
# the monitor row is absent
running_monitor_count(code) == 1          # badge source
is_sequential_family_container(code) is True
```

## Desired behavior

1. Exactly one monitor node per monitor that runs in an agent family, rendered with the
   orange gear glyph (`⚙`, `_MONITOR_GLYPH`) instead of an agent icon.
2. That node is nested directly under the agent row that started it, at
   `starter.tree_depth + 1` — under `sase-ns.6.6.6.1--2` in the example, not under the
   family root and not at clan-member depth.
3. The starter row does **not** render the `⚙N` count badge; the node replaces it.
4. Ancestor container rows (the clan container, and the family container row) keep the
   aggregate `⚙N` badge, which is what the user sees while those containers are
   collapsed.
5. A monitor node is visible whenever its starter row is visible. It must not hide
   behind a per-row fold the user has never opened, and it disappears with its starter
   when an ancestor container is collapsed.
6. Family/clan liveness is preserved: while a nested monitor is still running, the
   family root and clan must not read as finished.
7. Monitors whose artifacts already exist on disk (no clan metadata, `parent_timestamp`
   pointing at a mid-family continuation) render correctly with no migration.

## Implementation

Work is confined to the Python Agents-tab projection plus one narrow durable-metadata
change. Verified: this is not Rust-core work — `crates/sase_core/src/agent_scan/wire.rs`
already carries `monitor_*`, `monitor_starter_agent`, `agent_clan`, and
`agent_clan_generation`; the clan/family/fold tree projection is presentation-side and
lives in this repo. If a new wire field seems necessary, stop and re-check the design
before touching `../sase-core`.

No feature flag: this is a defect fix to already user-reaching behavior that is ready on
landing, not a staged beta.

### 1. Emit nested follow-ups in the ordered row list

`src/sase/ace/tui/models/_agent_ordering.py` — `sort_and_reorder`.

Replace the single-level splice (`result.extend(followups_by_parent.pop(suffix))`, both
the workflow branch and the `elif suffix in followups_by_parent` branch) with a
transitive emit: after appending a row, recursively append that row's own follow-ups in
the same chronological order, then their follow-ups, and so on. Requirements:

- Pop each parent's bucket as it is consumed so no row is emitted twice.
- Cycle-guard on `id(row)` (rows are mutable and unhashable; malformed
  `parent_timestamp` chains must not loop).
- Preserve today's ordering for the existing one-level case exactly: main workflow agent
  steps, then follow-ups, then other steps.

### 2. Let the clan tree projection nest deeper than two levels

`src/sase/ace/tui/models/_agent_tree.py` — `project_clan_tree`, `_clan_for_row`,
`_sort_clan_member_units`.

- Resolve clan membership for rows whose parent is itself a child row, so a monitor
  started by `--2` joins clan `sase-ns.6.6.6` (walk `parent_timestamp` up through child
  rows until a row with `agent_clan` is found, cycle-guarded). This is what makes
  existing monitors — which have no clan metadata on disk — land in the right clan and
  therefore the right panel.
- Assign `tree_parent_key` / `tree_depth` transitively: a row whose parent is loaded in
  the same clan gets `tree_parent_key = parent.raw_suffix` and
  `tree_depth = parent.tree_depth + 1`, resolved in dependency order rather than the
  current one-shot depth-1/2 branch. Rows with no loaded parent stay direct clan members
  at depth 1 (unchanged fail-safe).
- `_sort_clan_member_units` must attach a descendant to the unit of its nearest
  **direct** clan member (walk up `tree_parent_key` to that member) instead of only
  matching immediate direct-member keys, so a depth-3 row is not split off into its own
  unit and reordered away from its parent.
- `_append_tree_indent` already handles arbitrary depth; no renderer change needed for
  indentation.

### 3. Make monitor nodes visible with their starter

`src/sase/ace/tui/models/_fold_filter.py` — `filter_agents_by_fold_state`.

- Register fold owners from every row that owns a fold key, not only non-child rows, so
  a child row can be resolved as a parent. Keep the existing collision rule that a
  non-child row wins a repeated key (legacy workflow children can repeat their parent's
  suffix — see the comment in `tree_parent_lookup`).
- A monitor row is visible exactly when its resolved parent row is visible: skip the
  `FoldLevel.COLLAPSED` gate for `agent.is_monitor` rows. Collapsing the family
  container or the clan still hides the monitor, because its parent becomes invisible.
- Exclude monitor rows from `fold_counts` so they do not inflate a parent's `×N`
  annotation for a fold that does not gate them.
- `_is_foldable_parent` (`src/sase/ace/tui/widgets/_agent_list_helpers.py:57`) can stay
  as-is: with monitors out of the fold counts, a starter that owns only a monitor has no
  fold annotation to render.

### 4. Stop the starter from rendering the container badge

`src/sase/ace/tui/models/agent_family_members.py` — `is_sequential_family_container`.

Require at least one **non-monitor** family-member child before a row counts as a
sequential family container. A monitor child alone must not promote its starter to a
container row. Effects to confirm: `--2` renders no `⚙1`; the family container row and
the clan container keep their aggregate `⚙N` from `running_monitor_count`; the
`is_sequential_family_container` consumers that follow from this
(`agent_nodes._agent_node_owned_rows`, `concrete_agent_statuses`, and the `h`-navigation
parent-kind classification in `src/sase/ace/tui/actions/agents/_folding_agent_tree.py`)
keep treating `--2` as a plain agent node.

### 5. Replace the display reroot with liveness-only propagation

`src/sase/ace/tui/models/_agent_status_family_core.py`,
`src/sase/ace/tui/models/_agent_status_family.py`,
`src/sase/ace/tui/models/_agent_status_apply.py`.

- Delete `normalize_monitor_family_display_parents` and its call in
  `apply_status_overrides`. Rewriting the display parent now actively fights the desired
  containment: the monitor must stay under its starter.
- Preserve the property that motivated it — a family root must not read as finished
  while one of its descendants' monitors is still running. Implement it as status/
  liveness propagation instead of a link rewrite: when the root-summary loop in
  `apply_status_overrides` (the `for parent in parent_by_suffix.values()` block) looks
  for active work, include in-flight monitor rows found transitively beneath the root's
  family children (reuse the traversal shape of
  `agent_family_members.running_monitor_count`, which already walks `runtime_children` +
  `followup_agents` with cycle and identity guards). `agent_row_is_in_flight` already
  defines "in flight" for a monitor row.
- Persisted metadata is untouched in both directions: `parent_timestamp` on disk keeps
  pointing at the starter, which monitor follow-up settlement
  (`src/sase/monitor/followup.py`) depends on for forking the starter.

### 6. Record clan identity on new monitor members

`src/sase/monitor/member.py` — `create_monitor_member`.

Copy `agent_clan` and `agent_clan_generation` from the starter's `base_meta` onto the
monitor member's `agent_meta.json` (narrowly, in this function — do **not** widen the
inherited key list in `create_followup_artifacts`, which every follow-up kind shares).
This makes new monitor rows self-describing for clan-aware consumers. The TUI must not
depend on it: step 2's derivation is what fixes monitors already on disk, and both paths
must produce the same tree. `clan_members` already filters monitor rows out of clan
counts through `is_agents_tab_agent_node`, so adding clan identity must not change any
clan count.

### 7. Audit the remaining depth assumptions

Check and fix (or explicitly confirm correct) these known depth-aware sites once nesting
can exceed two levels:

- `src/sase/ace/tui/actions/agents/_folding_agent_tree.py` (`tree_depth == 1` clan
  collapse target; the `selected.tree_depth != parent.tree_depth + 1` navigation
  validation should now accept the monitor -> starter edge).
- `src/sase/ace/tui/models/agent_panel_index.py` and
  `src/sase/ace/tui/models/agent_panels.py` (panel assignment flows through
  `presentation_anchor_lookup`, so the monitor should inherit the clan anchor's panel —
  confirm with a test rather than by inspection).
- `src/sase/ace/tui/actions/navigation/_member_jump.py`,
  `src/sase/ace/tui/actions/agents/_neighbors.py`,
  `src/sase/ace/tui/widgets/_agent_list_render_cache.py` (cache key already includes
  `tree_depth`).

## Tests

Add or extend, using the real failing shape (clan -> family root -> `--2` -> monitor,
with the monitor carrying no `agent_clan` / `agent_clan_generation`):

- `tests/ace/tui/models/test_agent_family_members.py`: a row whose only family-member
  child is a monitor is not a sequential family container; a row with a real member plus
  a monitor still is.
- `tests/ace/tui/models/test_agent_tree.py`: the monitor row projects to
  `tree_parent_key == starter.raw_suffix` and `tree_depth == starter.tree_depth + 1`,
  joins the starter's clan without clan metadata of its own, and stays adjacent to its
  parent through `_sort_clan_member_units`.
- New coverage for `sort_and_reorder`: a follow-up of a follow-up is emitted exactly
  once, immediately after its parent; ordering for the existing one-level case is
  unchanged.
- New coverage for `filter_agents_by_fold_state`: the monitor is visible whenever its
  starter is visible (including when the starter's own fold key was never expanded), is
  hidden when the family container or clan is collapsed, and never appears in the
  starter's `fold_counts`.
- `tests/ace/tui/widgets/test_agent_list_monitor_rows.py`: the starter row renders no
  `⚙N`; the monitor row renders the gear glyph and its label; a clan container with a
  nested running monitor still renders `⚙1`.
- `tests/ace/tui/models/test_monitor_family_root_projection.py`: retarget from the
  deleted reroot to the new contract — the monitor keeps its starter link, renders under
  the starter, and
  `test_nested_monitor_family_lane_counts_running_without_extra_agent`'s assertions
  (`summary.total == 2`, `summary.done == 2`, `lane.total == 1`, `lane.running == 1`)
  still hold with the monitor nested one level deeper.
- One end-to-end projection test through `normalize_loaded_agents`
  (`src/sase/ace/tui/models/_agent_loader_normalization.py`) asserting the full
  screenshot shape yields exactly one monitor row at depth 3 under `--2`.

## Verification

```bash
just install          # ephemeral workspace: always first
just check            # whole-repo lint gates + scoped tests
```

Then run the exhaustive suite through `/sase_monitor` (never inline) because this change
touches shared projection code that many suites depend on:

```bash
sase monitor start --command 'just check-full' --next '<follow-up action>'
```

Manual confirmation in `sase ace`: with a family whose mid-family continuation is
running a monitor, the monitor appears as a single gear-glyph node indented under its
starter, the starter shows no `⚙N`, and the clan container still shows the aggregate
count.

## Acceptance criteria

- [ ] One gear-glyph monitor node per monitor, nested under the agent that started it.
- [ ] No `⚙N` badge on the starter row; aggregate badge retained on clan and family
      container rows.
- [ ] Monitors already on disk (no clan metadata) render identically to newly started
      ones.
- [ ] A running nested monitor still keeps its family root and clan from reading as
      finished.
- [ ] `just check` clean; `just check-full` clean via `/sase_monitor`.

## Non-goals

- Changing persisted monitor/starter linkage or the monitor follow-up settlement path.
- Reworking the procs pane, monitor detail panel, or monitor status semantics.
- Any change under `../sase-core`.
- A feature flag for this behavior.
