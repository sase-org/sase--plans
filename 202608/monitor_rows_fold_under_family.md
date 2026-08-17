---
tier: tale
title: Reveal family monitor rows with the family fold instead of leaking through it
goal:
  A monitor row is collapsed under its agent family by default and is revealed by
  expanding that family, while it keeps rendering as one gear-glyph node under the agent
  that started it.
size: medium
proposed_by: bbugyi200.athena.04l.f1
create_time: 2026-08-17 10:10:28
status: wip
---

# Reveal family monitor rows with the family fold instead of leaking through it

## Problem

`eefc44983` made every agent-family monitor render as one gear-glyph node under the
agent that started it. It also made monitor rows skip the fold gate entirely, so a
monitor is now visible whenever its starter is visible. That is too permissive in one
direction: when the family root itself started the monitor, the monitor row is the
family's _own_ child, so it leaks through the family's collapsed fold and dangles under
a family that is supposed to be showing nothing but its one-line summary.

Proc shells inside an agent family should be collapsed under that family by default and
revealed on demand — select the family container row and press `l`.

### Reproduced against this tree

Driving `filter_agents_by_fold_state` / `compute_visible_parents` /
`compute_fold_annotation` directly (no TUI), with a family root, one member, and one
monitor, at the default fold state (`FoldStateManager.get` returns `COLLAPSED`):

| Shape                      | Family fold | Visible rows                | Family annotation |
| -------------------------- | ----------- | --------------------------- | ----------------- |
| root started the monitor   | COLLAPSED   | `fam`, **`fam--mon`**       | `''` (no `×N`)    |
| root started the monitor   | EXPANDED    | `fam`, `fam--1`, `fam--mon` | —                 |
| member started the monitor | COLLAPSED   | `fam`                       | —                 |
| member started the monitor | EXPANDED    | `fam`, `fam--1`, `fam--mon` | —                 |

Row 1 is the defect. Row 3 already behaves correctly, and row 4 is the shape the user
approved in the previous screenshot, so it must not change.

This is the common shape, not a corner: of the 211 monitor `agent_meta.json` artifacts
on this machine, **135** have a `parent_timestamp` pointing at a row whose
`agent_family_role` is `root`, and 75 point at a mid-family member.

### Root cause

`filter_agents_by_fold_state` (`src/sase/ace/tui/models/_fold_filter.py:97`) gates on
the row's _immediate_ parent fold and exempts monitors unconditionally:

```python
level = fold_manager.get(parent_key)
if level == FoldLevel.COLLAPSED and not agent.is_monitor:
```

The exemption exists for a real reason: a monitor started by a mid-family member hangs
off that member, and a member's own fold is never openable by the user. Monitors are
excluded from `fold_counts` (`_fold_filter.py:38`), so `_get_workflow_key_for_agent`
(`src/sase/ace/tui/actions/agents/_folding_agent_tree.py:60-65`) never finds the
member's key in `_fold_counts` and routes `l` up to the family instead. Without the
exemption a member-started monitor would be unreachable.

But the exemption is expressed as "ignore whichever fold happens to be the immediate
parent", and when the starter _is_ the family container that fold is the family fold.

Two consequences follow from the same leak:

1. **The collapsed `×N` disappears.** `compute_visible_parents`
   (`src/sase/ace/tui/widgets/_agent_list_build.py:65-77`) records the family fold key
   in `parents_with_visible_children` because a visible monitor claims it as its parent,
   so `compute_fold_annotation`
   (`src/sase/ace/tui/widgets/_agent_list_helpers.py:103-124`) takes the expanded branch
   and drops the ` ×N` badge from a collapsed family. Confirmed in the table above.
2. **A reachability hole is hiding behind the leak.** A family whose only loaded child
   is its monitor — root + `--mon`, which is exactly the shape while a family's first
   monitor runs — produces an _empty_ `fold_counts`, because monitors are excluded. Its
   fold key is therefore absent from `_fold_counts`, `_get_workflow_key_for_agent`
   returns `None`, and `l` on that family row does nothing. Today that is invisible
   because the leak shows the monitor anyway. Any fix that gates monitors must make that
   fold openable or the row becomes permanently unreachable.

## Desired behavior

1. A monitor row is revealed by the fold of the **agent family it belongs to** — the
   fold the user opens by selecting the family container row and pressing `l` — not by
   its starter's own fold and not unconditionally.
2. At the default fold state (family `COLLAPSED`) no monitor row renders. The family and
   clan container rows keep their aggregate `⚙N` badge as the collapsed summary, and the
   family's collapsed `×N` counts the monitor among the rows the fold hides.
3. One `l` on the family (`COLLAPSED` → `EXPANDED`) reveals the family's members _and_
   their monitors, whether the monitor hangs off the family root or off a mid-family
   member. Monitors are **not** deferred to `FULLY_EXPANDED`: the screenshot the user
   approved for `eefc44983` showed the family at `×8 −3` (`EXPANDED`, hidden steps still
   hidden) with the monitor row visible.
4. Placement is unchanged from `eefc44983`: one gear-glyph node at
   `starter.tree_depth + 1` under the agent that started it, no `⚙N` on the starter,
   aggregate `⚙N` retained on family and clan container rows.
5. `l` and `H` on a selected monitor row act on the family fold that governs that row,
   and collapsing the family from a selected monitor reanchors selection to the family
   row rather than leaving the cursor on a row that just disappeared.
6. Collapsing the family container or the clan still hides the monitor (already true).

## Implementation

All work is presentation-side in this repo. No wire, binding, or `../sase-core` change:
this only re-reads fold state that the Python projection already owns. No feature flag —
this is a defect fix to already user-reaching behavior that is ready on landing.

### 1. One shared notion of "the fold that reveals this row"

`src/sase/ace/tui/models/_agent_tree.py` — add a pure helper beside `agent_fold_key` /
`agent_parent_fold_key` and export it in `__all__`:

```python
def agent_gating_fold_key(agent: Agent, owners_by_key: Mapping[str, Agent]) -> str | None:
    """Return the fold key whose expansion reveals *agent*."""
```

- Non-monitor rows: return `agent_parent_fold_key(agent)` unchanged.
- Monitor rows: walk up the immediate-parent chain (`agent_parent_fold_key` resolved
  through `owners_by_key`) and return the first parent key whose owner row is **not**
  `is_child_row` — the family container / workflow root. A monitor started by the family
  root resolves in one step to the family key; a monitor started by `--2` climbs past
  `--2` to the same family key. A family root inside a clan is not a child row (its clan
  link is `tree_parent_key`, not `parent_timestamp`), so the walk stops at the family
  and never climbs to the clan fold.
- Bound the walk by the owner count and cycle-guard on `id(row)`; return `None` when a
  parent is missing or the chain is malformed.

Complexity is O(depth) per monitor row against an index the callers already build, so
the refilter path stays linear.

### 2. Gate and count monitors by that key

`src/sase/ace/tui/models/_fold_filter.py` — `filter_agents_by_fold_state`.

- `children_by_parent`: stop skipping monitor rows; attribute every row to
  `agent_gating_fold_key(...)` instead of `agent_parent_fold_key(...)`. Keep skipping a
  row whose key is unresolvable or absent from `owners_by_key`, as today. Guard the
  degenerate case where a monitor's gating key resolves to a `clan:` key: leave those
  monitors out of `fold_counts`, because a clan's counts are direct-member counts and
  `clan_members` already excludes monitor rows. This closes the reachability hole: a
  root-plus-monitor family now has a `fold_counts` entry, so `l` on it expands and the
  `×N` badge advertises the row.
- `is_visible`: keep the parent-chain recursion keyed on `agent_parent_fold_key` (that
  is structural containment and must stay as-is), but evaluate the `FoldLevel.COLLAPSED`
  gate against the gating key. For a non-monitor row the two keys are identical, so
  nothing changes. For a monitor whose gating key is unresolvable, fall back to today's
  behavior (visible with its parent) rather than hiding a row in a malformed projection.
- The `is_hidden_step` / `FULLY_EXPANDED` rule stays keyed on the immediate parent; a
  monitor is never a hidden step.
- Update the module docstring: `fold_counts` now maps each fold key to the rows that
  fold reveals, which for a monitor is its family rather than its immediate parent.

### 3. Point `l` / `H` on a monitor row at the fold that governs it

`src/sase/ace/tui/actions/agents/_folding_agent_tree.py`.

- `_get_workflow_key_for_agent`: for a monitor row, return its gating key (resolved with
  `tree_parent_lookup(self._agents)`, the same index the neighbouring resolvers use)
  instead of falling through to `agent_parent_fold_key`, which today names the starter's
  fold — a fold that gates nothing and whose expansion is invisible.
- `_resolve_agent_structural_collapse_target`: compute `selected_is_child` from the
  gating key so a selected monitor row counts as a child of the family fold being
  collapsed and `reanchor` moves selection to the family row. Without this, `H` on a
  monitor collapses the family and strands the cursor on a hidden row.
- No change to `_validated_agent_parent_target`: `h` still walks the rendered tree to
  the starter, which is deliberately _not_ the gating fold.

### 4. Documentation

`docs/ace.md`, in the Agents-tab monitor section (around the `⚙` / `⚙N` glyph table at
line 1832): state that a monitor row nests under the agent that started it, that it is
revealed by its family's fold rather than its starter's — so a collapsed family shows
`⚙N` and counts the monitor in `×N` but no monitor row — and that `l` / `H` on a monitor
row act on that family fold.

## Tests

Extend the suites `eefc44983` already added, using both real shapes (root-started and
member-started) so the asymmetry that caused this defect stays covered:

- `tests/test_fold_filtering.py`
  - New: a root-started monitor is **hidden** while the family fold is `COLLAPSED` and
    appears at `EXPANDED` (the defect, directly).
  - New: with that family collapsed, the family fold key is absent from
    `compute_visible_parents(...)[0]` and `compute_fold_annotation` still renders the
    collapsed ` ×N` — the badge the leak was eating.
  - New: a family whose only loaded child is a monitor has a `fold_counts` entry for the
    family key, so the fold is openable and one `expand` reveals the monitor.
  - Update `test_monitor_is_visible_whenever_starter_is_visible` to the new contract
    (member-started monitor visible once the family fold is open; the starter's key
    still absent from `fold_counts`; the family's count now includes the monitor).
  - Keep `test_monitor_hides_when_family_or_clan_is_collapsed` passing unchanged.
- `tests/ace/tui/test_agent_fold_transitions_navigation.py`
  - `l` on a family container reveals a monitor nested under a mid-family starter.
  - `l` on a selected monitor row targets the family fold, not the starter's key.
  - `H` on a selected monitor row collapses the family and reanchors selection to the
    family container row.
  - `test_h_nested_monitor_navigates_to_starter` keeps passing: `h` still goes to the
    starter.
- `tests/ace/tui/models/test_monitor_family_root_projection.py`: the lane/summary
  liveness assertions still hold with the monitor hidden by the fold — visibility is a
  display filter and must not change family or clan status.
- End-to-end through `normalize_loaded_agents`: at the default fold state the screenshot
  shape renders no monitor row; after expanding the clan and then the family, exactly
  one monitor row at depth 3 under `--2`.

## Verification

```bash
just install          # ephemeral workspace: always first
just check            # whole-repo lint gates + scoped tests
```

Then the exhaustive suite through `/sase_monitor` (never inline), because the fold
filter is shared projection code many suites depend on:

```bash
sase monitor start --command 'just check-full' --next '<follow-up action>'
```

Manual confirmation in `sase ace`, Agents tab: a family running a monitor shows no gear
row while collapsed, shows `⚙N` and a `×N` that counts the monitor, and reveals the
gear-glyph row under its starter after a single `l` on the family container row.

Known pre-existing gates that block a fully green run and are not caused by this work:
stale `--epic-symbol` entries in `_lint-symvision` (`sase-o7`), the suite-wide
`just test-cost` budget misses (`sase-j0`), and the flake-baseline nodes (`sase-ob`,
`sase-mv`, `sase-oh`, `sase-j7`).

## Acceptance criteria

- [ ] No monitor row renders while its agent family's fold is collapsed, including when
      the family root is the agent that started the monitor.
- [ ] A single `l` on the family container row reveals every monitor in that family,
      each still nested under the agent that started it.
- [ ] A collapsed family with a monitor keeps its ` ×N` annotation, and that count
      includes the monitor.
- [ ] A family whose only loaded child is a monitor can still be expanded.
- [ ] `l` / `H` on a selected monitor row act on the family fold, and `H` reanchors
      selection to the family row.
- [ ] `h` from a monitor still navigates to its starter; placement, glyph, and the
      family/clan `⚙N` badges are unchanged.
- [ ] `just check` clean; `just check-full` clean via `/sase_monitor` apart from the
      pre-existing gates listed above.

## Non-goals

- Moving monitor rows out from under their starter, or reviving
  `normalize_monitor_family_display_parents`.
- Deferring monitors to `FULLY_EXPANDED` alongside hidden steps.
- Adding a `⚙N` badge to the starter row.
- Changing monitor status, liveness propagation, the Procs tab, or persisted
  monitor/starter linkage.
- Any change under `../sase-core`, and any feature flag.
