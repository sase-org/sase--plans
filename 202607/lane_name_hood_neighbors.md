---
tier: tale
title: Match agent lanes against hoods that share their name
goal: An agent lane whose name equals another agent's hood is matched as a neighbor
  in both directions, so the NEIGHBORS metadata-panel section and the `~` neighbors
  keymap relate agent-family lanes to their dotted hood-mates exactly the way single-agent
  lanes already are.
create_time: 2026-07-25 13:43:02
status: done
---

- **PROMPT:** [202607/prompts/lane_name_hood_neighbors.md](prompts/lane_name_hood_neighbors.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-99.land.f2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-99.land.f2/README.md)
  - [bbugyi200.athena.sase-99.land.f2--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-99.land.f2.md#member-code)
- **COMMITS:**
  - [9ea6edc](https://github.com/sase-org/sase/commit/9ea6edc3ec5de0c221223ec61b7f508ec73ea146) — fix(ace): key agent lanes by the name they present

# Plan: Match agent lanes against hoods that share their name

## Problem

Per the SASE glossary, an agent hood is the group of agents sharing a `<name>.` prefix, **and the agent named `<name>`
itself is part of that hood**. So `foo`, `foo.baz`, and `foo.bar.1` are all neighbors.

ACE honors that for single-agent lanes, but **not for agent-family lanes**. A family lane renders under its bare family
name (`fam`) while the ACE neighbor index keys the row by the root member's raw name (`fam--plan`). The lane therefore
never joins hood `fam`, and neither the NEIGHBORS section nor `~` relates it to `fam.helper`.

### Reproduction (verified against current `master`)

Rows: family `fam` (container row plus members `fam--plan`, `fam--code`) and single agents `fam.helper`, `fam.other`.
Driving `AgentNeighborMixin.lane_neighbor_projection_for` through the real Agents-tab harness:

| Selected lane  | Today             | Expected                              |
| -------------- | ----------------- | ------------------------------------- |
| `fam` (family) | _no rows at all_  | descendants `fam.helper`, `fam.other` |
| `fam.helper`   | `fam.other` only  | ancestor `fam`, neighbor `fam.other`  |
| `fam.other`    | `fam.helper` only | ancestor `fam`, neighbor `fam.helper` |

The control case proves the intent: replace the family with a _single agent_ named `fam` and all three lanes immediately
relate correctly (`fam` → descendants `fam.helper`/`fam.other`; `fam.helper` → ancestor `fam` + neighbor `fam.other`).

A nested family is mismatched rather than unmatched. For family `a.b` plus `a.b.helper` plus `a.other`, the family and
`a.b.helper` today only meet as co-members of the _outer_ `a` hood; `a.b.helper` should see the family lane as its
**ancestor** in hood `a.b`, exactly as a single agent named `a.b` would be seen.

## Root cause

`src/sase/ace/tui/models/agent_hoods.py::agent_name_key` keys every indexed row by `Agent.presented_identity_name`. For
a family root entry that attribute is the raw agent name (`fam--plan`), while `Agent.presented_agent_name` holds the
lane name the Agents tab actually displays (`fam`) — see `Agent.refresh_raw_presented_agent_name`, which branches on
`is_family_root_entry`.

Because the key carries the `--<suffix>` member part:

- `_agent_hood_chain("fam--plan")` is `("fam--plan",)`, so the lane never joins hood `fam`.
- `_agent_boundary_prefixes("fam.helper")` yields `"fam"`, which matches no indexed key, so the lane is never found as
  an ancestor.

### The canonical rule already exists in the Rust core

`sase.core.agent_identity_facade` exposes Rust-backed helpers whose semantics are the target behavior:

```
agent_name_ancestors("fam--plan")            == ("fam",)
agent_name_ancestors("visual.worker--plan")  == ("visual", "visual.worker")
agent_name_in_hood("fam--plan", "fam")       is True
agent_name_in_hood("visual.helper", "visual.worker") is False
```

`crates/sase_core/src/agent_identity/identity.rs::agent_name_in_hood` parses the name, reduces it to
`parsed.family_name` (dropping `--<member>`), and then applies "equals the hood or starts with `<hood>.`".
`crates/sase_core/src/machine_hood.rs` documents the same boundary rule. **The core already treats the family (lane)
name as the hood-membership key; only ACE disagrees.**

No Rust change is required. The neighbor index is a presentation projection over _rendered Agents-tab rows_ (fold state,
panels, prospective clan reveals, dismissed rows) and has no counterpart in `sase-core`; the `agent_identity` crate
covers name/identity semantics only. This plan aligns the ACE projection with the core rule rather than moving it.

## Design

**Key every indexed row by the lane name it presents.** For a family root entry that is the bare family base; for every
other row it is unchanged. This is a naming fix, not a new relation: once the family lane is keyed `fam`, the existing
ancestor / descendant / hood-group machinery produces the correct result in both directions, and it makes ACE's hood
chain agree with `agent_name_ancestors` for every lane.

Deliberately **not** changed: family _member_ rows (`fam--code`) keep their raw `--` key. They are not lanes; they
attach to their lane through the existing `--` descendant edge, and re-keying them would move rows between the FAMILY
MEMBERS and NEIGHBORS rosters for no user-visible gain.

### Blast-radius findings (measured, not assumed)

Running the real fixtures through the harness with the re-key simulated:

1. **Both existing lane-neighbor PNG fixtures are byte-identical in projection output.**
   `_family_lane_neighbor_agents()` (family `visual.worker` + `visual.helper` + `visual.notes`) yields the same visible
   rows (`visual.helper`, `visual.notes` under `visual hood`) and the same `suppressed_lane_member_count == 2` before
   and after. `_single_lane_neighbor_agents()` is likewise unchanged. **Do not regenerate any golden**; if a golden does
   move, stop and report it rather than accepting it with `--sase-update-visual-snapshots`.

2. **The lane self-drop guard breaks unless it is extended.** An expanded family renders its root member as a _second_
   row: in `_family_lane_neighbor_agents()`, row index 2 is the container (`presented_agent_name='visual.worker'`,
   `presented_identity_name='visual.worker--plan'`) and row index 3 is a distinct member-child row with the same
   `presented_identity_name` and a different identity. `build_agent_lane_neighbor_projection` currently drops it because
   `lane_name` is `presented_identity_name`. Once `lane_name` becomes the lane name `visual.worker`, that row leaks into
   NEIGHBORS — confirmed by direct measurement:

   ```
   lane_name='visual.worker--plan' -> rows=['visual.helper', 'visual.notes']                        # today
   lane_name='visual.worker'       -> rows=['visual.worker--plan', 'visual.helper', 'visual.notes']  # leak
   ```

   The projection must therefore drop rows matching **either** the lane name **or** the lane owner's
   `presented_identity_name`. The `suppressed_identities` path does not cover this row (its identity differs from every
   entry `concrete_family_member_rows` returns).

3. The lane's real family members become `--` descendants instead of outer-hood neighbors. They are still suppressed
   into the `… +N also listed under FAMILY MEMBERS` tail by the existing `suppressed_identities` argument, so the
   rendered section is unchanged.

## Implementation

### 1. `src/sase/ace/tui/models/agent_hoods.py`

Add a public helper next to `agent_name_key` and make the key derive from it:

```python
def agent_lane_name(agent: Agent) -> str | None:
    """Return the lane name a row presents on the Agents tab.

    A family root entry renders under its bare family base while its raw name
    keeps the ``--<suffix>`` member part, so kinship must key on the lane name
    to agree with ``sase.core.agent_identity_facade.agent_name_in_hood``.
    """
    if agent.is_clan_container:
        return None
    if agent.is_family_root_entry:
        return agent.presented_agent_name or agent.presented_identity_name
    return agent.presented_identity_name
```

`agent_name_key` keeps its existing validation (reject empty dotted segments, casefold) but reads
`agent_lane_name(agent)` instead of `agent.presented_identity_name`. Export `agent_lane_name` from the module's public
surface.

Notes for the implementer:

- Branch on `is_family_root_entry`, not `is_family_container_row`, so the index key always equals the name the row
  displays — `refresh_raw_presented_agent_name` uses the same predicate.
- Fall back to `presented_identity_name` when `presented_agent_name` is empty, and keep the clan-container `None`
  short-circuit so `agent_owns_lane` is unaffected.
- `agent_hood()` derives from `agent_name_key` and its results are unchanged for every case checked
  (`visual.worker--plan` and `visual.worker` both give `visual`; `fam--plan` and `fam` both give `None`).
- A key collision between a family `fam` and a hypothetical single agent named `fam` is harmless: both rows are listed,
  and target resolution is identity-based (`_unique_key_for_identity`).

### 2. `src/sase/ace/tui/models/agent_lane_neighbors.py`

Extend `build_agent_lane_neighbor_projection` so the "a lane is never its own neighbor" guard accepts more than one self
key — e.g. a new keyword-only `lane_row_names: Collection[str] = ()` whose casefolded members join `lane_name` in the
drop test. Update the existing comment to explain both cases (the lane's own row and the expanded family's duplicate
root-member row). Keep `lane_name` as the source of `label_prefix`.

### 3. `src/sase/ace/tui/actions/agents/_neighbors.py`

Four sites currently read `presented_identity_name` and must use the lane name:

- `_start_agent_neighbor_navigation` — `lane_name=` (line ~277); also pass the new
  `lane_row_names=(selected.presented_identity_name,)`.
- `lane_neighbor_projection_for` — `lane_name=` (line ~556); also pass
  `lane_row_names=(agent.presented_identity_name,)`.
- `_dismissed_descendant_agents` — `selected_name` (line ~567), so a dismissed `fam.helper` is offered for revival under
  the family lane `fam`.
- `_agent_neighbor_display_hoods` — the `parts` split (line ~588), so hood labels for a family lane read `visual.worker`
  rather than `visual.worker--plan`.

Import `agent_lane_name` from `...models.agent_hoods` and use it at all four sites.

### 4. Tests

Add coverage that fails before the fix and passes after:

- `tests/ace/tui/models/test_agent_neighbors.py`
  - `agent_lane_name` / `agent_name_key` truth table: family root entry → family base; family member child, single
    agent, and malformed names unchanged; clan container → `None`.
  - Index relations for a top-level family `fam` with `fam.helper` / `fam.other`: family is an ancestor of each
    hood-mate; both hood-mates are descendants of the family.
  - Nested family `a.b`: `a.b.helper` sees the family lane as an **ancestor**, not just an `a` hood neighbor.
  - A parity test against the core rule: for the lane rows in a fixture, the index's hood membership agrees with
    `sase.core.agent_identity_facade.agent_name_in_hood(lane_name, hood)`. Scope it to lane rows — family member
    children intentionally keep their raw key.
- `tests/ace/tui/models/test_agent_lane_neighbors.py`
  - A top-level family lane projects its hood-mates.
  - Regression guard for finding 2: with `lane_name` set to the family base and `lane_row_names` carrying the root
    member's raw name, the duplicate root-member row is still dropped. Extend or mirror
    `test_projection_never_lists_the_lane_as_its_own_neighbor` rather than deleting it.
- `tests/ace/tui/test_agent_neighbor_navigation_targets.py` (harness `NeighborApp` in
  `tests/ace/tui/_agent_neighbor_navigation_helpers.py`)
  - `~` on a top-level family lane offers `fam.helper` / `fam.other` and jumps to the chosen row.
  - `~` on `fam.helper` offers the family lane and lands on the family container row.
- `tests/ace/tui/widgets/test_agent_display_neighbors.py`
  - The NEIGHBORS section renders for a top-level family lane (it renders nothing today).

### 5. Docs

Behavior changes, so keep the documentation in sync:

- `docs/ace.md` — the `~` row in the Agents keymap table (~line 399) and the **Lane Neighbors Section** (~lines
  600–615): state that an agent lane joins the hood matching its own name, so a family lane and its dotted hood-mates
  are related in both directions.
- `docs/agent_families.md` — the hood paragraph (~lines 11–12) and the family lane / NEIGHBORS paragraph (~lines
  273–278): a family lane participates under its bare family name, not the root member's `--` name.
- `src/sase/ace/CLAUDE.md` requires the `?` help popup to stay in sync with any `sase ace` behavior change. Review
  `src/sase/ace/tui/modals/help_modal/agents_bindings.py` (the `0-9` and `~` entries). The current wording ("Jump
  ancestor/neighbor/desc") is expected to remain accurate — update it only if it becomes misleading, and respect the
  32-character description limit and 57-character box width if you do.

**Do not edit any file under `sase/memory/`, `AGENTS.md`, or the generated provider shims.** The glossary already states
the rule this plan implements, and memory edits require explicit user permission that this plan does not carry.

## Out of scope

- **Clan containers stay unmatched.** A clan name is reserved and is never itself an agent, so `agent_name_key`
  continues to return `None` for `is_clan_container` rows.
- **Family member rows still appear individually in hood neighbor lists.** With family `visual.worker` and single agent
  `visual.helper`, `visual.helper` lists `visual.worker`, `visual.worker--plan`, `visual.worker--impl`, and
  `visual.worker--review`. That is pre-existing (the row count is identical before and after this change), and
  collapsing member rows into their lane is a separate design question.
- **The `neighbors:` info-panel badge keeps its current formula** (`neighbor_count + ancestor_count + descendant_count`
  in `src/sase/ace/tui/actions/agents/_display_detail_info.py`). Measured on both PNG fixtures, the badge value is
  unchanged by this fix. For a _top-level_ family lane it rises from `0` to the newly reachable rows, which is correct:
  those rows are now reachable from `~`.
- No Rust / `sase-core` changes.

## Validation

Run from the workspace checkout (a stale workspace needs `just install` first so its virtualenv matches the current
dependency set):

```bash
just install
just check
just test-visual
```

Targeted runs while iterating:

```bash
.venv/bin/pytest tests/ace/tui/models/test_agent_neighbors.py \
                 tests/ace/tui/models/test_agent_lane_neighbors.py \
                 tests/ace/tui/test_agent_neighbor_navigation.py \
                 tests/ace/tui/test_agent_neighbor_navigation_targets.py \
                 tests/ace/tui/test_agent_neighbor_index_cache.py \
                 tests/ace/tui/widgets/test_agent_display_neighbors.py
```

Acceptance:

- `just check` and `just test-visual` are green.
- Every lane-neighbor PNG golden is unchanged without `--sase-update-visual-snapshots`
  (`agents_family_lane_neighbors_160x50`, `agents_lane_neighbors_section_first_level_160x50`,
  `agents_lane_neighbors_section_expanded_160x50`, `agents_neighbor_badge_120x40`,
  `agents_neighbor_jump_expanded_panel_120x40`, `agent_neighbor_modal_60x30`,
  `agent_neighbor_modal_descendants_dismissed_60x30`, `agent_neighbor_folded_clan_modal_70x32`).
- `just symvision` reports no unused public symbol for `agent_lane_name`.
- The reproduction table at the top of this plan matches the Expected column for both the NEIGHBORS section and the `~`
  chooser.
