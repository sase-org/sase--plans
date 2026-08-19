---
tier: tale
title: Dismiss the whole clan subtree instead of only its launch-time rows
goal:
  Pressing `x` once on a clan container dismisses every surviving clan row — including
  post-handoff sequential family members and monitor proc shells — so the clan row
  disappears immediately instead of lingering as an empty DONE shell.
size: medium
proposed_by: bbugyi200.athena.07t
create_time: 2026-08-19 10:22:19
status: wip
---

# Plan: Dismiss the whole clan subtree, not just its launch-time rows

## Symptom

Dismissing an agent clan with the `x` keymap "doesn't always work", and when it does
work the clan shells stay visible for a while. In the reported case, three DONE clans in
the `@epic` tribe panel (`sase-ps`, `sase-pv`, `sase-pq`) were dismissed and stayed on
screen. The lingering `sase-ps` row renders as `(DONE) ×3 ⚙3 sase-ps` while its detail
pane reports `Members: 0 agents` — an empty clan shell whose only remaining children are
monitor and family-member rows.

## Root cause

`plan_agent_cleanup` skips every row that has a `parent_timestamp` as "cascade only",
but the cascade it defers to never runs for clan members.

The planner lives in the Rust core (`crates/sase_core/src/agent_cleanup/planner.rs`) and
is mirrored by the Python reference planner (`src/sase/core/agent_cleanup_python.py`).
Both use one over-broad predicate:

```rust
fn is_workflow_child(target: &AgentCleanupTargetWire) -> bool {
    target.is_workflow_child
        || target.parent_workflow.is_some()
        || target.parent_timestamp.is_some()
}
```

The wire's own `is_workflow_child` flag is no help either: `agent_to_cleanup_target`
(`src/sase/core/agent_cleanup_targets.py`) copies it from `Agent.is_workflow_child`,
which `src/sase/ace/tui/models/agent.py:418` documents as a _historical alias for
`is_child_row`_ — true for family-member children as well as workflow steps. So every
sequential family member and every monitor proc shell reads as a "workflow child".

The main planning loop (`planner.rs:615`, `agent_cleanup_python.py:309`) then does:

```rust
let direct_child_target = is_direct_child_target(...);
if is_workflow_child(target) && !direct_child_target {
    add_skip(&mut skipped_items, target, SKIPPED_WORKFLOW_CHILD_CASCADE_ONLY, None);
    continue;
}
```

and `is_direct_child_target` (`planner.rs:143`) is false whenever the row's own parent
is in the same selection (`parent_selected_for_child`). The assumption is that the
parent's action will cascade to the child. That assumption only holds in two places:

- `planner.rs:713` — cascade to `children_by_parent` when the parent is killed with
  `KILL_KIND_WORKFLOW`.
- `planner.rs:733` — cascade to `parallel_family_members` when the parent is a
  parallel-family root.

A clan member is `agent_type == "run"` with `agent_family_parallel == false`, and a
finished one is _dismissed_, not killed. The dismiss branch (`planner.rs:678`) pushes
one `AgentCleanupDismissItemWire` and `continue`s with no cascade at all, and
`related_workflow_targets` (`planner.rs:456`) only widens side effects when
`target.agent_type == "workflow"`. The children are therefore dropped silently: they are
neither dismissed nor reported as failures.

### Why the shell survives and why it looks empty

Post-handoff rows carry `agent_clan` and `agent_clan_generation` in their own
`agent_meta.json`, so `_clan_for_row` / `project_clan_tree`
(`src/sase/ace/tui/models/_agent_tree.py`) rebuild a synthetic container from the
survivors on the next load. The container reports `×3` direct children and `⚙3` monitor
lanes, but its detail pane says `Members: 0 agents` because `clan_members()`
(`src/sase/ace/tui/models/_agent_clan.py:103`) filters through
`is_agents_tab_agent_node()` (`src/sase/ace/tui/models/agent_nodes.py:50`), which
excludes both monitor rows and family-member children. That is exactly the empty DONE
shell in the screenshot.

### Why it "takes a while"

Each `x` press peels exactly one generation of the parent chain. Round 1 dismisses the
launch-time rows; round 2 dismisses the children whose parents are now gone from
`_agents_with_children`; round 3 dismisses the grandchildren. A clan with a plan-root →
family-root → monitor chain needs three presses before the shell clears.

### Evidence gathered on the reporting host

Dismissal batch of 2026-08-19, 09:42–10:07 (81 revive bundles written under
`~/.sase/dismissed_bundles/202608/`):

- 80 of the 81 dismissed raw suffixes have `parent_timestamp == null` in their
  `agent_meta.json`. **Zero** rows with a `parent_timestamp` were dismissed. (The
  remaining suffix had no readable meta.)
- Clan `sase-ps`, generation `20260818102050`: the five launch-time rows
  (`20260818102050`–`20260818102054`) are in `~/.sase/dismissed_agents.json`; the six
  later rows are not — family roots `--1` / `--2` / `--3` (`20260818114621`,
  `20260818115117`, `20260818120358`) and monitor shells `--mon` / `--mon-0` / `--mon-1`
  (`20260818114457`, `20260818114833`, `20260818115413`).
- All six survivors have `parent_workflow == null`, `agent_family_parallel == null`, and
  dead PIDs — so neither planner cascade could ever have reached them, and
  `runner_is_live` is not involved.
- The same launch-rows-only pattern holds for `sase-pq`, `sase-pt`, and `sase-pv`.

## Fix

The two cascade-only decision points must use the narrow "workflow step child" test
(`AgentChildLinkage.WORKFLOW_STEP`, i.e. `parent_workflow is not None`) rather than the
broad "any child row" test. No wire field is added and
`AGENT_CLEANUP_WIRE_SCHEMA_VERSION` does **not** change, because `parent_workflow` is
already on `AgentCleanupTargetWire`.

### Step 1 — Rust core planner

Open the core repo first; do not clone or path-guess it:

```bash
sase repo open sase-core -r "Fix the agent-cleanup cascade-only skip that drops clan family and monitor rows"
```

In `crates/sase_core/src/agent_cleanup/planner.rs`:

1. Rename the existing broad predicate to `is_child_row` (same body) and add:

   ```rust
   /// Mirrors `AgentChildLinkage::WORKFLOW_STEP`: only a workflow step child is
   /// covered by its parent's cascade. Family members and monitor proc shells
   /// carry a `parent_timestamp` but are independent agent rows with their own
   /// PID, artifacts, and dismissal record.
   fn is_workflow_step_child(target: &AgentCleanupTargetWire) -> bool {
       target.parent_workflow.is_some()
   }
   ```

2. Switch to `is_workflow_step_child` at exactly two sites:
   - `is_direct_child_target` (`planner.rs:156`)
   - the main-loop cascade-only skip (`planner.rs:615`)

3. Leave every other call site on `is_child_row` so tribe inheritance, workspace
   release, and workflow-cascade semantics are unchanged: `effective_tribe` (58),
   `parent_matches_child` (114), `parallel_family_members` (218),
   `workflow_children_by_parent` (246), `parent_tribes_by_suffix` (265),
   `add_workspace_release` (398), `add_held_workspace_release` (430),
   `related_workflow_targets` (458).

4. Tests in the same file's `mod tests`:
   - Update `broad_scopes_keep_child_rows_cascade_only` (`planner.rs:1343`). Its fixture
     is a `run` child with only a `parent_timestamp`; under `all_panels`,
     `focused_panel`, and `tribe` scope that row must now be acted on directly. Add a
     sibling case with `parent_workflow = Some(...)` that still asserts
     `SKIPPED_WORKFLOW_CHILD_CASCADE_ONLY`, so the workflow contract stays covered.
   - Audit the other cascade-only assertions (`planner.rs:1006`, `~1032`, `~1152`,
     `~1431`, `~1494`) — the ones whose fixtures set `is_workflow_child = true` plus
     `parent_workflow` keep passing; any that rely only on `parent_timestamp` need the
     same treatment.
   - Add a new regression test shaped like the real clan: three `run` targets with the
     same `agent_clan` / `agent_clan_generation`, `parent_workflow = None`,
     `agent_family_parallel = false`, status `DONE`, `pid = None` — a plan root
     (`parent_timestamp = None`), a family root whose `parent_timestamp` is the plan
     root, and a monitor shell whose `parent_timestamp` is the family root. Run it under
     both `CLEANUP_SCOPE_CLAN` and `CLEANUP_SCOPE_EXPLICIT_IDENTITIES` with all three
     identities selected, and assert all three appear in `dismiss_items` and in
     `side_effects.dismissed_index_additions`, and that no
     `SKIPPED_WORKFLOW_CHILD_CASCADE_ONLY` entry is produced.

Run `cargo test -p sase_core` in that checkout.

### Step 2 — Python reference planner parity

The facade falls back to `plan_agent_cleanup_python` whenever the binding is missing or
stale, so both implementations must agree.

1. `src/sase/core/agent_cleanup_targets.py`: keep the existing broad predicate available
   under its current exported name (other modules import `is_workflow_child` — confirm
   with `grep -rn "agent_cleanup_targets import" src/ tests/` before renaming anything)
   and add `is_workflow_step_child`.
2. `src/sase/core/agent_cleanup_python.py`: use the narrow predicate at
   `_is_direct_child_target` (line 150) and the main-loop skip (line 309) only. Leave
   lines 71, 117, 205, 268, and 274 on the broad predicate.
3. Mirror the new and updated Rust cases in
   `tests/test_core_facade/test_agent_cleanup_python.py`, and add a facade-level check
   in `tests/test_core_facade/test_agent_cleanup_facade.py` asserting the Rust and
   Python planners agree on the clan fixture.

### Step 3 — Close two TUI parity gaps in the same flow

1. `src/sase/ace/tui/actions/agents/_kill_cleanup_planning.py`:
   `plan_bulk_kill_cleanup_side_effects` builds its `AgentCleanupRequestWire` without
   `include_pidless_as_dismissable=True`, unlike `plan_single_agent_kill_cleanup` in the
   same file and `_plan_clan_cleanup_container` in `_kill_cleanup_clan.py`. That
   asymmetry makes a pidless clan member whose recorded status never settled into a
   terminal value get skipped as `not_killable` instead of dismissed. Add the flag so
   the bulk path matches the focused and clan-chooser paths.

2. `src/sase/ace/tui/actions/agents/_kill_flow.py`, `_do_bulk_kill_agents`: today
   `dismissed_ids = dismissed_identities_from_plan(cleanup_plan)` and the
   `_collect_dismissal_identities(dismiss_candidates)` call only runs when the plan
   returned nothing at all. Make it a union instead of an empty-set fallback, so the set
   the user confirmed in the modal is always the set that gets hidden and persisted even
   if the planner drops a row again. Add a short comment noting that rows contributed
   only by the union get index-only dismissal — the plan remains the source of truth for
   the richer side effects (bundle saves, artifact deletes, workspace releases,
   notification dismissals).

### Step 4 — TUI regression test

Add a test (extend `tests/test_agent_kill_bulk.py`, or add
`tests/ace/tui/test_agent_clan_dismiss_cascade.py` next to the existing clan tests) that
builds the plan-root → family-root → monitor chain under one clan container, drives
`action_kill_agent` on the container, confirms the modal, and asserts:

- all three identities land in `app._dismissed_agents`;
- `project_clan_tree` over the remaining `_agents_with_children` produces no container
  for that clan (the shell is gone);
- the submitted cleanup-proc payload's `dismissed_identities` contains all three, so the
  dismissal is persisted and not just optimistic.

`tests/_agent_cleanup_proc_helpers.py` and `tests/test_agent_clan.py` already carry the
fixtures and proc-capture helpers for this shape.

### Step 5 — Build, verify, and repin

1. `just install` — this builds `sase_core_rs` from the linked `sase-core` checkout, so
   the Rust change is exercised locally.
2. `cargo test -p sase_core` in the `sase-core` checkout, then `just check` here.
3. `just check-full` through `/sase_monitor` before landing, with a `--next` action.
4. Because the wire schema version is unchanged, a published `sase-core-rs` still
   satisfies `_current_rust_cleanup_binding()` and would keep serving the old behavior
   for non-dev installs. Land the `sase-core` change first, let release-plz publish,
   then raise the `sase-core-rs` floor in `pyproject.toml` (currently
   `>=0.29.0,<0.30.0`) and refresh `uv.lock` in this repo. Do not close the work with
   the floor unbumped — the fix is inert for real users until then.

### Recovering rows already stranded

No migration or repair command is needed. With the fix in place, a single `x` on a
leftover clan shell selects the whole surviving subtree in one pass, dismisses it, and
the container disappears because `project_clan_tree` has no rows left to build it from.

## Out of scope

Note these but do not fix them here; file task beads through `/sase_new_task` for any
that reproduce.

- `parent_tribes_by_suffix` (`planner.rs:260`) only publishes a tribe for non-child
  rows, so a monitor nested under a post-handoff family root cannot inherit one. A
  tribe- or panel-scoped `X` cleanup may therefore still miss it even after this fix.
- `add_held_workspace_release` (`planner.rs:428`) skips every child row, so a
  post-handoff family member's held workspace is not released when it is dismissed. This
  overlaps the in-flight `sase-ps` epic on monitor and post-handoff shells versus
  `max_running_agents`; leave it to that epic.
- The `runner_is_live` bypass in `compute_apply_loaded_agents`
  (`src/sase/ace/tui/actions/agents/_loading_compute.py:272`) re-shows a dismissed row
  while its runner record is live. Not implicated here — every survivor PID on the
  reporting host was dead — so leave it alone.
- Rendering a clan container that has zero `is_agents_tab_agent_node` members is left as
  is on purpose: after this fix it correctly surfaces a clan whose only remaining row is
  a live monitor.
- `~/.sase/dismissed_agents.json` has grown to 32,534 entries / 2.0 MB and is fully
  rewritten on every dismissal. Unrelated to correctness here, but worth a separate bead
  if dismissal latency is still noticeable afterward.

## Verification

1. Reproduce first, on the pre-fix build: pick a DONE clan whose members include a
   `--mon` or `--1` row, press `x`, confirm, and observe the empty shell remain with a
   `Members: 0 agents` detail pane.
2. After the fix, repeat: one `x` press must clear the clan row completely.
3. Confirm on disk that the post-handoff suffixes now appear in
   `~/.sase/dismissed_agents.json` and have revive bundles under
   `~/.sase/dismissed_bundles/`.
4. Confirm a workflow parent with real step children still cascades and still reports
   `workflow_child_cascade_only` for those steps — that contract must not regress.
