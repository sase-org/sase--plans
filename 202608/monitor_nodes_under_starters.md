---
tier: tale
title: Show one monitor node under the agent that started it
goal:
  Every loaded monitor member renders exactly one gear-glyph node nested directly
  beneath the agent row that started it, and the aggregate gear badge stays on container
  rows (clan / family) instead of appearing on the starter row.
size: medium
proposed_by: bbugyi200.athena.04k
create_time: 2026-08-17 07:44:29
status: wip
---

# Plan: Show One Monitor Node Under the Agent That Started It

## Problem

When a sase agent inside an agent family runs `sase monitor start`, the Agents tab shows
**no monitor node at all**. Instead, the starter row grows a `⚙1` count badge that
duplicates the badge already shown on its ancestors.

Observed in `sase ace` (screenshot `20260817_071832.png`, family `sase-ns.6.6.6.1`
inside clan `sase-ns.6.6.6`):

```
🤖 (WAITING) [W1 D4] ⚙1 sase-ns.6.6.6              <- clan container
  └ sase (TALE DONE) ×8 −3 ⚙1 ◆ sase-ns.6.6.6.1    <- family root
      ├ main (TALE APPROVED) ◆ sase-ns.6.6.6.1--plan
      ├ sase (TALE DONE)     ◆ sase-ns.6.6.6.1--code
      ├ sase (TALE DONE)     ◆ sase-ns.6.6.6.1--1
      ├ sase (TALE DONE) ⚙1  ◆ sase-ns.6.6.6.1--2   <- badge, but no monitor node
      └ ⟩ diff (DONE) ◆ ▼#gh
```

Two live monitors existed at that moment (`--mon-0` under `--1`, `--mon-1` under `--2`)
and neither had a row.

## Goal

```
🤖 (WAITING) [W1 D4] ⚙1 sase-ns.6.6.6
  └ sase (TALE DONE) ×8 −3 ⚙1 ◆ sase-ns.6.6.6.1
      ├ main (TALE APPROVED) ◆ sase-ns.6.6.6.1--plan
      ├ sase (TALE DONE)     ◆ sase-ns.6.6.6.1--code
      ├ sase (TALE DONE)     ◆ sase-ns.6.6.6.1--1
      │   └ ⚙ config-cache check-full retry (MONITORED ✗ 1)  ◆ …--mon-0
      ├ sase (TALE DONE)     ◆ sase-ns.6.6.6.1--2
      │   └ ⚙ check-full after toobig (MONITORING)           ◆ …--mon-1
      └ ⟩ diff (DONE) ◆ ▼#gh
```

- Exactly one node per loaded monitor member, nested directly under its **starter** (the
  row whose `parent_timestamp` the monitor persists), with the existing orange `⚙` glyph
  and monitor status labels.
- The starter row loses its `⚙N` count badge.
- Container rows (the clan container, the family container) keep `⚙N`, so a collapsed
  subtree still advertises live monitors.

## Root Cause (verified — do not re-derive)

Every claim below was confirmed by running the real projection functions in a workspace
venv against a roster shaped like the screenshot, and against the durable
`agent_meta.json` files the screenshot was taken from.

1. **The monitor member persists its starter as its parent, and never carries clan
   metadata.** `sase-ns.6.6.6.1--mon-1`'s `agent_meta.json` has
   `agent_family: sase-ns.6.6.6.1`, `agent_family_role: monitor`,
   `parent_timestamp: 20260817070811` (that is `--2`), **no** `agent_clan`, and **no**
   `agent_clan_generation`. Its starter `--2` has `agent_clan: sase-ns.6.6.6` and
   `agent_clan_generation: 20260817055518`.

2. **`normalize_monitor_family_display_parents()` silently no-ops for clan families.**
   `src/sase/ace/tui/models/_agent_status_family_core.py:255` reroots a monitor onto its
   family root, but keys roots by `(project_file, agent_clan_generation, agent_family)`.
   The monitor's generation is `None` and the root's is `20260817055518`, so the lookup
   misses and the monitor keeps `parent_timestamp = --2`. (In a non-clan family the
   reroot _does_ fire, which is why monitors are visible there today — as siblings of
   the family members, not under their starter.)

3. **A followup of a followup is dropped during ordering.** `sort_and_reorder()`
   (`src/sase/ace/tui/models/_agent_ordering.py:108`) only pops
   `followups_by_parent[suffix]` for rows that are themselves in `sorted_agents` (i.e.
   non-followup rows). `--2` is a followup, so `followups_by_parent['<--2>']` is never
   drained and the monitor row never reaches the output list. Verified: the monitor is
   absent from `sort_and_reorder()`'s result for the clan-shaped roster.

4. **Even if it survived ordering, fold filtering would drop it.**
   `filter_agents_by_fold_state()` (`src/sase/ace/tui/models/_fold_filter.py:21`)
   registers fold owners only for rows where `not agent.is_child_row`. A row whose
   parent fold key resolves to no owner is forced invisible (`_fold_filter.py:69-72`),
   so any grandchild of a child row disappears.

5. **And the clan projection would eject it from its clan block.** `project_clan_tree()`
   resolves a row's clan through `parent_by_suffix`, which is also built from non-child
   rows only, so the monitor resolves to no clan, is emitted after the whole clan block,
   and gets the `is_child_row` fallback depth of 1. Verified output with monitors
   present: `d1 sase-ns.6.6.6.1--mon-0 tree_parent=None` (should be `d3`, parent `--1`).

6. **The stray `⚙1` on `--2` comes from the container predicate.**
   `_attach_runtime_children()` indexes _all_ suffixes, so `--2.runtime_children` does
   contain the monitor even though no row is rendered. That makes
   `is_sequential_family_container(--2)` true via its third branch (`agent_family` + a
   non-parallel family-member child), which is exactly the `is_container_row` gate on
   the badge in `src/sase/ace/tui/widgets/_agent_list_render_agent.py:478-486`.

## Prototype Evidence

The design below was prototyped end-to-end (monkeypatched, no repo edits) over a roster
mirroring the screenshot, then rendered through the real `format_agent_option()`. With
all five changes simulated:

```
d0 | (DONE) [D1] ⚙1 sase-ns.6.6.6
d1 |   └─ sase-ns.6.6.6.1 (TALE DONE) ⚙1 sase-ns.6.6.6.1
d2 |   │  └─ sase-ns.6.6.6.1--plan (TALE APPROVED)
d2 |   │  └─ sase-ns.6.6.6.1--1 (TALE DONE)
d3 |   │  │  └─ ⚙ check-full after toobig (MONITORED ✗ 1)
d2 |   │  └─ sase-ns.6.6.6.1--2 (TALE DONE)
d3 |   │  │  └─ ⚙ check-full after toobig (MONITORING)
```

and, with the family fold collapsed, only the two container rows survive — both still
carrying `⚙1`. The existing renderer needs **no** changes: `_append_tree_indent()`
already handles depth ≥ 3 and monitor rows already render the gear glyph, the label, and
terminal `✗ N` / `⧖` / `⚠` / `⚑` badges at any depth.

## Design

One containment link, used consistently: **a monitor row belongs to the row named by its
persisted `parent_timestamp`** (its starter). Ordering, clan projection, tree depth, and
fold visibility all follow that same chain.

Fold _gating_ stays where it is today: monitors are hidden and shown by their nearest
**fold-owning ancestor** (the family root / clan), not by a new fold on the starter. A
row whose display parent is itself a child row simply inherits its parent's visibility.
This matters: fold state defaults to `COLLAPSED`, so giving `--2` its own fold would
leave the monitor hidden on first paint — the exact symptom being fixed.

### 1. Order nested followups after their parent

`src/sase/ace/tui/models/_agent_ordering.py` — `sort_and_reorder()`

Replace the single-level `followups_by_parent.pop(suffix)` calls in the result loop with
a recursive emitter: after emitting a followup, emit that followup's own followups
(chronological, oldest first), and so on. Requirements:

- Emit each row at most once (guard on `id(row)`); cycles must terminate.
- Preserve today's ordering for existing shapes: for a workflow/`ace-run` parent, the
  order stays `main_agent_steps`, then `followups`, then `other_steps`.
- Groups whose parent never appears stay undrained (unchanged today's behavior); the
  fold filter drops such orphans anyway.

### 2. Project nested rows into the tree

`src/sase/ace/tui/models/_agent_tree.py`

- Build the suffix index over **all** rows, preferring a non-child row on collision
  (same rule `tree_parent_lookup()` already documents — legacy workflow children repeat
  their parent's suffix, and `tests/test_fold_filtering.py::_make_child` constructs
  exactly that shape, so this preference is load-bearing).
- `_clan_for_row()`: walk the display-parent chain (bounded, cycle-guarded) until a row
  with `agent_clan` is found, instead of looking one level up at non-child parents only.
  This puts the monitor in its starter's clan so it is emitted inside the clan block.
- Per-clan depth assignment: resolve each row's display parent; when that parent is in
  the same clan, set `tree_parent_key = parent.raw_suffix` and
  `tree_depth = parent_depth + 1` (memoized recursion, so parents resolve before
  children); otherwise keep today's "direct clan member at depth 1" fallback. Existing
  two-level shapes must keep producing exactly today's depths (1 and 2).
- `_sort_clan_member_units()`: attach a row to its **nearest ancestor** unit by walking
  up `tree_parent_key`, not just the direct `unit_by_parent_key` hit. Otherwise the
  monitor becomes its own unit and sorts away from its starter.
- Non-clan rosters take the `if not row_clans: return real_agents` early return today
  and get no depth at all. Apply the same nested-depth pass before that return so a
  monitor under a family member of a non-clan family renders at depth 2 (root 0, member
  1, monitor 2) instead of colliding with its parent's depth.

### 3. Inherit visibility through non-owner parents

`src/sase/ace/tui/models/_fold_filter.py` — `filter_agents_by_fold_state()`

- Keep `owners_by_key` exactly as-is (non-child rows only), and keep `fold_counts` keyed
  on owners with immediate-child counts, so `×N` annotations do not change.
- Add a `rows_by_key` index over all rows (non-child preferred, as above). When a row's
  parent fold key has no owner but _does_ resolve to a present row, that row's
  visibility is inherited (recursive, reusing the existing `visiting` cycle guard)
  instead of returning `False`.
- A parent key that resolves to no row at all still yields `False` —
  `test_orphan_workflow_child_is_dropped_even_when_expanded` must keep passing
  unchanged.

Consequence, intended: monitors are not counted in any `×N`, and they appear or
disappear together with their starter.

### 4. Reroot only as a fallback

`src/sase/ace/tui/models/_agent_status_family_core.py` —
`normalize_monitor_family_display_parents()`

- Reroot a monitor onto its family root **only when its starter row is not among the
  loaded rows** (dismissed, cleaned up, filtered out). A loaded starter is now a
  renderable parent, so the workaround must not fire.
- While there, drop `agent_clan_generation` from the root identity key: match on
  `(project_file, agent_family)` and keep the existing "must resolve to exactly one
  loaded root, else no-op" rule. The generation component is what made this function
  dead code for every clan-launched family (root cause 2), and it only guards the
  fallback path now.
- Keep every other safety property: no durable metadata is written, monitors without
  `agent_family` are untouched, ambiguity fails safe, and repeated passes over cached
  objects stay a fixed point.
- Update the module docstring and the function docstring to describe the new contract.

### 5. Keep a family with a live monitor reading Running

`src/sase/ace/tui/models/_agent_status_apply.py` — family-root mirroring loop (the
`for parent in parent_by_suffix.values():` block that builds `candidates` / `active`)

Today the reroot is what lets a family root mirror a running monitor
(`tests/ace/tui/models/test_monitor_family_root_projection.py:: test_nested_monitor_family_lane_counts_running_without_extra_agent`
pins `lane.running == 1`). With change 4 the monitor is no longer a direct child, so add
nested monitors to the root's mirror candidates: for each family child of the root,
include that child's monitor rows (one level deeper — monitors are leaves; a monitor's
follow-up agent attaches to the family root, not to the monitor).

`_is_active_root_mirror_candidate()` already treats an in-flight monitor as active, so
no other change is needed.

**Intentional, user-visible consequence:** clan-launched families now behave like
non-clan families — a family root with a live monitor beneath it displays the monitor's
status (e.g. `MONITORING`) and counts as a running lane, where today (screenshot) it
reads `TALE DONE`. That can move a clan row between status groups. This is the same rule
already pinned for non-clan families; if the project owner prefers roots to stay
terminal and rely on the `⚙N` badge alone, drop change 5 and re-pin the lane test
instead — everything else in this plan is independent of that choice.

### 6. Stop treating a monitor child as a family membership

`src/sase/ace/tui/models/agent_family_members.py` — `is_sequential_family_container()`

Add `and not child.is_monitor` to the third branch's `any(...)`, so a row whose only
"family members" are monitors is not a container. That is what removes `⚙1` from `--2`.

Scope note — this deliberately does **not** touch `is_family_container_row`, so a
top-level agent that starts its own monitor keeps its `⚙N` badge exactly as today
(useful when that row is collapsed). The only rows that lose the badge are non-root rows
whose sole members are monitors, which is precisely the reported bug. Tightening the
rule further (never badge a monitor's direct starter) is a follow-up decision, not part
of this change.

### 7. Navigation on a monitor row

`src/sase/ace/tui/actions/agents/_folding_agent_tree.py`

Monitor rows become selectable, so the two structural keys must not dead-end on them:

- `_get_workflow_key_for_agent()`: when a child row's `agent_parent_fold_key()` is not
  in `self._fold_counts`, walk up the display chain (via `tree_parent_lookup`) to the
  nearest key that is, so `H` on a monitor row collapses the owning family fold.
- `_validated_agent_parent_target()`: accept a parent that is itself a child row when
  its suffix identifies exactly one row and the selected row's `tree_parent_key` /
  `tree_depth` agree with it, so `h` on a monitor row selects its starter. Keep every
  other validation gate (single occurrence, exact depth, explicit-edge agreement) intact
  — this relaxes _one_ clause, it does not loosen the invariant.

## Non-Goals

- Do not change monitor row rendering. Glyph, label, status colors, `✗ N` / `⧖` / `⚠` /
  `⚑` badges, and duration ticking already work at any depth.
- Do not change what a monitor _is_ on disk. `parent_timestamp`,
  `monitor_starter_agent`, and follow-up settlement semantics are untouched; this is a
  presentation-layer change only.
- Do not remove monitors from `concrete_family_member_rows()` /
  `family_roster_entries()`. The FAMILY MEMBERS roster intentionally labels monitor
  members (`test_family_roster_labels_monitor_members`). A known, separate cosmetic wart
  rides on that: `_append_model_fields()` calls `build_family_model_lanes()` →
  `concrete_family_member_rows()` unconditionally, so a starter's detail panel shows a
  second `Model:` lane for its monitor (the monitor inherits the starter's model
  metadata and drives no LLM). None of the changes here affect it; file it separately
  via `/sase_new_task` if it is worth fixing.
- Do not count monitor rows as agent nodes. `is_agents_tab_agent_node()` and
  `concrete_agent_statuses()` already exclude them; header counts, clan chips, and lane
  counts must not move because a monitor row became visible.
- Do not touch `../sase-core`. Tree projection, folding, and row rendering are
  presentation state that belongs in this repo per the `rust_core_backend_boundary`
  memory.

## Files to Change

| File                                                     | Change                                           |
| -------------------------------------------------------- | ------------------------------------------------ |
| `src/sase/ace/tui/models/_agent_ordering.py`             | recursive followup placement (1)                 |
| `src/sase/ace/tui/models/_agent_tree.py`                 | clan resolution, nested depth, unit nesting (2)  |
| `src/sase/ace/tui/models/_fold_filter.py`                | visibility inheritance for non-owner parents (3) |
| `src/sase/ace/tui/models/_agent_status_family_core.py`   | reroot becomes a fallback (4)                    |
| `src/sase/ace/tui/models/_agent_status_apply.py`         | nested monitors as mirror candidates (5)         |
| `src/sase/ace/tui/models/agent_family_members.py`        | container predicate ignores monitors (6)         |
| `src/sase/ace/tui/actions/agents/_folding_agent_tree.py` | `h` / `H` on monitor rows (7)                    |

## Tests

Add:

- `tests/ace/tui/models/test_monitor_nested_rows.py` (new) — the end-to-end regression
  for this bug. Build the screenshot's shape (clan container → family root → `--plan` /
  `--1` / `--2`, `--mon-0` under `--1`, `--mon-1` under `--2`, monitors carrying **no**
  `agent_clan` / `agent_clan_generation`), run `apply_status_overrides()` +
  `sort_and_reorder()`, then assert:
  - each monitor appears exactly once in the ordered rows;
  - each monitor immediately follows its own starter;
  - `agent_tree_depth(monitor) == agent_tree_depth(starter) + 1` and
    `tree_parent_key == starter.raw_suffix`;
  - after `filter_agents_by_fold_state()` with the clan and family folds expanded, both
    monitors are visible; with the family fold collapsed, neither is, and the
    family/clan rows still report `running_monitor_count() >= 1`;
  - the same shape without a clan nests the monitor under its starter too.
- `tests/test_fold_filtering.py` — a nested grandchild inherits its parent's visibility;
  `fold_counts` for the owning parent is unchanged by the grandchild; the existing
  orphan test still passes untouched.
- `tests/ace/tui/widgets/test_agent_list_monitor_rows.py` — a family-member starter with
  a running monitor child renders no `⚙N`; a clan container and a family container still
  do; the existing `test_family_container_with_running_monitor_renders_badge` case must
  keep passing.
- `tests/ace/tui/models/test_agent_tree.py` — clan projection keeps a grandchild in its
  ancestor's clan block, at `parent_depth + 1`, adjacent to its parent.
- `tests/ace/tui/test_agent_fold_transitions_tree.py` (or the closest existing
  navigation module) — `h` from a monitor row selects its starter; `H` from a monitor
  row collapses the owning family fold.

Update:

- `tests/ace/tui/models/test_monitor_family_root_projection.py` — rewrite for the
  fallback-only contract: a loaded starter means no reroot; an absent starter still
  reroots to a uniquely resolved root; clan generation no longer blocks the fallback;
  ambiguity, missing family, and idempotence cases stay. Keep
  `test_nested_monitor_family_lane_counts_running_without_extra_agent`'s assertions
  unchanged — change 5 is what makes them pass.

Optional (only if goldens are being regenerated anyway): a PNG case in
`tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py` showing a monitor node
nested under a family member. Skip it if it is the only thing forcing golden churn.

## Verification

From the implementing agent's own workspace directory:

```bash
just install    # workspaces are ephemeral; deps may be stale
just check      # whole-repo lint gates + diff-scoped tests
```

This change touches shared tree/fold projection code that many suites depend on, so
`just check`'s scoped selection is not sufficient on its own. Before landing, run the
exhaustive gate through `/sase_monitor` (never inline):

```bash
sase monitor start --command 'just check-full' --next '<follow-up action>'
```

Manual smoke check: open `sase ace` on the Agents tab with a family that has run a
monitor, expand the family, and confirm one gear node per monitor sits under its
starter, the starter has no `⚙N`, the clan/family rows still do, and collapsing the
family hides the monitor nodes.

## Acceptance Criteria

- Every loaded monitor member renders exactly one node, nested directly under its
  starter row, in both clan-launched and non-clan families.
- The starter row shows no `⚙N` badge; the clan container and family container still do,
  and still do when the subtree is collapsed.
- Monitor nodes keep their gear glyph, monitor label, monitor status labels, and
  terminal badges, and remain killable/stoppable from the row.
- No agent count changes because a monitor row became visible: header totals, clan
  status chips, panel indices, and `×N` fold annotations are unaffected.
- A monitor whose starter row is not loaded still appears under its family root when
  that root resolves uniquely; a monitor that resolves to nothing is dropped, not
  duplicated.
- `h` and `H` behave sensibly on a monitor row (select the starter / collapse the owning
  fold).
- `just check` passes, and `just check-full` passes when run through `/sase_monitor`.
