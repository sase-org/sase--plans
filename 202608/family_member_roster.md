---
tier: tale
title: Show FAMILY MEMBERS on family-member detail panels
goal:
  Selecting an agent family member in ACE renders the enclosing family's numbered FAMILY MEMBERS roster with every
  member except the selected one, and its 0-9 member jumps work exactly as they do on family container panels.
proposed_by: bbugyi200.athena.u9
create_time: 2026-08-06 13:28:32
status: done
---

- **PROMPT:**
  [prompts/202608/family_member_roster.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/family_member_roster.md)

# Show `FAMILY MEMBERS` on family-member detail panels

## Goal

When an agent **family member** row is selected in ACE's Agents tab, its metadata panel must render the same numbered
`FAMILY MEMBERS` roster that a family **container** row already renders, listing every concrete member of that family
**except the selected member itself**, with fully working `0`–`9` member jumps.

Today the section appears only when the family container row is selected, so a member panel gives no way to see or jump
to its siblings.

## Background

### Where the roster is rendered today

`build_header_text()` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py:70`) gates the whole family
roster on one flag:

```python
family_fold_enabled = agent.is_family_container_row and lane_fold_level is not None
...
family_entries = family_roster_entries(agent) if family_fold_enabled else ()
```

`family_fold_enabled` does double duty: it enables the roster _and_ switches the panel to the two-level
`FAMILY_FOLD_SCALE` for every foldable section (`OUTPUT VARIABLES`, `WORKFLOW VARIABLES`, `SASE CONTEXT`, `ERROR`, the
`Fold: 1/2` header line). A family member row is an ordinary agent row on the three-level `AGENT_FOLD_SCALE`
(`lane_fold_scale()` in `src/sase/ace/tui/models/fold_scale.py:58`), so the new roster must be enabled **without**
turning on `family_fold_enabled`.

Rendering itself is already generic: `append_family_member_roster()`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_family.py:100`) accepts precomputed `entries` and a shared
`MemberJumpNumbering`, and returns the `MemberJumpMap` the panel publishes.

### Which rows are "family members"

The authoritative list is `concrete_family_member_rows(container)`
(`src/sase/ace/tui/models/agent_family_members.py:94`). It covers both real family shapes:

- **Rename-on-attach families** — the container row itself is member 0 (`alpha--plan`, presented as `alpha`) and each
  follow-up (`alpha--code`) is a `is_family_member_child` row in `container.followup_agents`.
- **Plan-workflow families** — the container is a workflow aggregate row; the concrete planner is a
  `is_workflow_step_child` row in `container.runtime_children` and continuations are follow-ups.

Synthetic planner rows and parallel-family rows are excluded by that function and must stay excluded here.

Family chains are **flat**: every member's `parent_timestamp` is the family root's timestamp (verified against local
artifact metadata, including a three-member family: `tu.f0--0` is the root and both `tu.f0--1` and `tu.f0--code` point
at it). So one container row owns the whole chain and there is no grandchild-member case to handle.

### What blocks numeric jumps today

`MemberJumpNavigationMixin` (`src/sase/ace/tui/actions/navigation/_member_jump.py`) refuses to treat a member row as a
jump container:

- `_selected_member_jump_container()` returns `None` unless the selection is a clan container or `agent_owns_lane()` is
  true, and `agent_owns_lane()` (`src/sase/ace/tui/models/agent_hoods.py:45`) explicitly returns `False` for
  `is_family_member_child` and workflow-child rows.
- `_current_member_target_is_valid()` revalidates `role="member"` targets against
  `concrete_family_member_rows(container)`; with the selected member as `container` that returns the member itself, so
  every sibling jump would be rejected as stale.
- The footer only advertises `("0-9", "member")` for clan/family container rows
  (`src/sase/ace/tui/widgets/_keybinding_bindings.py:261`).

### Why a back-pointer

A member row has no reference to its container; only the container holds `followup_agents` / `runtime_children`.
Resolving upward on the render path would mean building a suffix→row index per render, which violates the TUI perf rule
that render paths never rebuild structures per keypress (the cheap header path in
`src/sase/ace/tui/widgets/prompt_panel/_agent_display.py:248` runs on every `j`/`k` highlight move).

`Agent` already carries a runtime-only, non-serialized `Agent` pointer for exactly this kind of presentation plumbing:
`wait_display_source` (`src/sase/ace/tui/models/_agent_state.py:339`). This plan follows that precedent with a
`family_container` back-pointer populated once per load, giving O(1) resolution at render time.

## Non-goals

- **Container panels do not change.** A selected family container keeps listing every member, including the root's own
  entry when the root represents member 0, on the two-level family scale with its `Fold: 1/2` header line.
- **No `NEIGHBORS` section on member rows.** `agent_owns_lane()` keeps excluding member/workflow-child rows; the
  documented rule that family member child rows have no `NEIGHBORS` stays true.
- No changes to the `~` neighbor chooser, the info-panel `neighbors:` badge, agent completion candidates, fork/wait
  targets, or clan/tribe rosters.
- No change to which rows are considered family members (`concrete_family_member_rows` is reused verbatim).

## Design decisions

- **D1 — Back-pointer, not a lookup.** Add a runtime-only `Agent.family_container` set by `sort_and_reorder()`, the one
  place where `followup_agents` (from `apply_status_overrides`) and `runtime_children` are both final. It is derived
  from `concrete_family_member_rows()` itself, so the roster and the pointer can never disagree. The field **must** be
  `compare=False, repr=False`: it closes a cycle with `followup_agents`, and the dataclass `__repr__`/`__eq__` (and
  therefore the hint cache's `repr()`-based digest) would recurse forever otherwise.
- **D2 — Member panels keep their own three-level scale.** The roster is rendered with `lane_fold_scale(agent)`
  (`AGENT_FOLD_SCALE`), so `z1`/`z2`/`z3`, `zz`, `za` behave exactly as they do for the panel's other foldable sections:
  level 1 compact rows, level 2 adds activity/wait/retry, level 3 adds workspace/timestamps/attempts.
  `family_fold_enabled` stays `False`, so no `Fold: 1/2` line appears and every other section keeps its current
  member-panel rendering.
- **D3 — Heading names the family.** On a member panel the heading count is the number of _listed siblings_, which is
  one less than the family size, so the heading gets a dim ` · <family name>` suffix (via the roster's existing
  `heading_suffix` parameter) to make it obvious that this is the enclosing family's roster minus the current row.
  Container panels get no suffix and are byte-identical to today.
- **D4 — Section placement and id are unchanged.** The roster keeps section id `members` and its current position
  (immediately after the runner-queue section, above `OUTPUT VARIABLES`), so `Ctrl+J`/`Ctrl+K` section cycling and
  `z`-mode section overrides work with no extra wiring.
- **D5 — No entry limit.** Unlike `NEIGHBORS`, the family roster shows every member at every level (capped by the shared
  100-target numbering ladder), matching container panels.

## Implementation

### 1. Model: `Agent.family_container`

**`src/sase/ace/tui/models/_agent_state.py`** — add beside `wait_display_source` (~line 339):

```python
    # Family container row whose FAMILY MEMBERS roster lists this row. Runtime
    # presentation plumbing; not serialized. ``compare``/``repr`` must stay off:
    # this pointer closes a cycle with ``followup_agents``/``runtime_children``
    # and dataclass eq/repr (and the repr-based hint digest) would recurse.
    family_container: Agent | None = field(default=None, compare=False, repr=False)
```

**`src/sase/ace/tui/models/agent_bundle.py`** — add `"family_container"` to the skipped-field tuple in
`to_bundle_dict()` (~line 20), next to `"wait_display_source"`.

**`src/sase/ace/tui/models/agent_family_members.py`** — add the shared resolver used by every consumer, and export it in
`__all__`:

```python
def family_roster_container(agent: Agent) -> Agent | None:
    """Return the container row whose FAMILY MEMBERS roster lists ``agent``.

    Container rows render their own roster and are never members of another
    row's roster, so they resolve to ``None``.
    """
    if agent.is_family_container_row:
        return None
    container = agent.family_container
    if container is None or container is agent:
        return None
    return container
```

**`src/sase/ace/tui/models/_agent_ordering.py`**:

- `_clear_runtime_children()` also resets the pointer (`agent.family_container = None`) for every row it visits, so a
  member that left its family cannot keep a stale container across reloads.
- Add `_attach_family_containers(rows)` and call it on the projected result in **both** `return` paths of
  `sort_and_reorder()` (assign `ordered = project_clan_tree(...)`, attach, then return `ordered`):

```python
def _attach_family_containers(rows: list[Agent]) -> None:
    """Point each concrete family member at the container row that lists it."""
    for row in rows:
        if not row.is_family_container_row:
            continue
        for member in concrete_family_member_rows(row):
            if member is not row:
                member.family_container = row
```

Import `concrete_family_member_rows` from `.agent_family_members` (no import cycle: that module imports only `.agent`).
Both live load paths already funnel through `sort_and_reorder` — `normalize_loaded_agents()`
(`src/sase/ace/tui/models/_agent_loader_normalization.py:83`) and `_normalize_relationships_after_merge()`
(`src/sase/ace/tui/actions/agents/_loading_compute_merge.py:183`) — so no other call site needs changing. Rows that
never pass through it (e.g. agents rebuilt straight from a dismissed bundle) simply keep `family_container = None` and
render exactly as they do today.

### 2. Roster helpers

**`src/sase/ace/tui/widgets/prompt_panel/_agent_display_family.py`**:

- `family_roster_entries(agent, *, now=None, exclude: Agent | None = None)` — skip a member when
  `member is exclude or member.identity == exclude.identity`. Everything else (labels, kinds, effective status buckets,
  digests) is unchanged, and labels still strip the container's presented family name so entries read `--plan`,
  `--code`.
- `append_family_member_roster(..., fold_scale: FoldScale = FAMILY_FOLD_SCALE, heading_suffix: Text | None = None)` —
  pass both through to `append_member_roster()`. The default keeps container panels identical.
- Add `family_roster_heading_suffix(container: Agent) -> Text` returning `" · "` (dim) plus the container's
  `presented_agent_name` in `FAMILY_IDENTITY_COLOR`, for D3.

### 3. Header rendering

**`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`**, inside `build_header_text()`:

```python
family_fold_enabled = agent.is_family_container_row and lane_fold_level is not None
roster_container = family_roster_container(agent)          # None on container rows
...
if family_fold_enabled:
    family_entries = family_roster_entries(agent)
elif roster_container is not None:
    family_entries = family_roster_entries(roster_container, exclude=agent)
else:
    family_entries = ()
```

The existing `document_numbering` block is unchanged: a member row owns no lane, so `shown_neighbor_count` is `0` and
the ladder is exactly the sibling count (single digits up to 10 entries, two-key `00`–`99` beyond that).

Replace the render block so the `Fold:` header line stays container-only while the roster becomes shared:

```python
if family_fold_enabled:
    append_fold_header_line(header_text, level=resolved_lane_fold_level, scale=FAMILY_FOLD_SCALE)
if family_entries:
    family_map = append_family_member_roster(
        header_text,
        agent,
        panel_level=resolved_lane_fold_level,
        section_fold_overrides=lane_overrides,
        entries=family_entries,
        numbering=document_numbering,
        fold_scale=FAMILY_FOLD_SCALE if family_fold_enabled else lane_scale,
        heading_suffix=(
            None if family_fold_enabled else family_roster_heading_suffix(roster_container)
        ),
    )
```

`append_family_member_roster()` keeps passing `container_identity=agent.identity` — i.e. the **selected member's**
identity — which is exactly the key `_member_jump_map_for()` looks the published map up under.

### 4. Publish the jump map from member panels

Three call sites compute the same gate and all three must learn about member rows:

- `src/sase/ace/tui/widgets/prompt_panel/_agent_display.py:230`
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_render.py:237`
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_render.py:179`

```python
lane_summary_enabled = (
    agent.is_family_container_row
    or lane_owner
    or family_roster_container(agent) is not None
)
```

Leave `lane_neighbors` gated on `lane_owner` alone — member rows still get no `NEIGHBORS` projection.

### 5. Numeric keymaps

**`src/sase/ace/tui/actions/navigation/_member_jump.py`**:

- `_selected_member_jump_container()` — accept the selection when it renders a roster:

```python
if agent is None or not (
    agent.is_clan_container
    or agent_owns_lane(agent)
    or family_roster_container(agent) is not None
):
    return None
return agent
```

- `_current_member_target_is_valid()` — the trailing family branch must revalidate against the right roster, and must
  not accept a jump back to the selected member itself:

```python
if container.is_family_container_row:
    return any(
        member.identity == target_identity
        for member in concrete_family_member_rows(container)
    )
roster_container = family_roster_container(container)
if roster_container is None:
    return False
return any(
    member.identity == target_identity and member.identity != container.identity
    for member in concrete_family_member_rows(roster_container)
)
```

Everything else in the mixin already generalizes: two-digit buffering keys on the container identity (the member's own
identity), `_reveal_agent_row()` reveals targets through family/clan/grouping folds, and a roster that changed since the
panel was drawn still cancels with `Member roster changed; jump cancelled`.

**`src/sase/ace/tui/widgets/_keybinding_bindings.py:261`** — advertise the digits in the footer:

```python
if (
    not panel_focused
    and not group_focused
    and (
        agent.is_clan_container
        or agent.is_family_container_row
        or family_roster_container(agent) is not None
    )
):
    bindings.append(("0-9", "member"))
```

The `elif ... lane_neighbor_jump_available` branch stays as the fallback for ordinary lanes.

### 6. Hint-render cache correctness

**`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_cache.py`** — `agent_hint_render_cache_key()` digests a
non-clan row with `_digest_parts(agent)`, which is `repr()`-based and therefore blind to `family_container`
(`repr=False`) and to sibling rows. Without this, a member panel rendered in `v` hint mode could keep showing stale
sibling statuses. Mirror the clan branch: when `family_roster_container(agent)` is not `None`, fold
`tuple((member.identity, member.display_status) for member in concrete_family_member_rows(roster_container))` into
`agent_state_digest`.

### 7. Documentation

- **`docs/ace.md`**
  - Keymap table row for `0`–`9` (~line 707): include a selected family member as a numbered-roster surface.
  - The numbered-jump paragraph (~line 730): state that a selected family member numbers its `FAMILY MEMBERS` roster
    (siblings only) from the same ladder, and that it has no `NEIGHBORS` rows to follow it.
  - §"Clan and Family Detail Panels" (~line 806): add that selecting a family member shows the enclosing family's roster
    minus that member, with a heading suffix naming the family, folded by the member's own three-level agent scale
    rather than the family's two-level scale.
  - §"Lane Neighbors Section" (~line 930): keep the statement that family member child rows have no `NEIGHBORS`, and
    clarify that such a panel carries only the `FAMILY MEMBERS` roster.
- **`docs/agent_families.md`** §"Family detail folding" (~line 266): document the member-panel roster — every member in
  stable chain order except the selected one, numbered from `0`, `z`-foldable on the member's three-level scale, with
  the same digit-jump semantics as the container roster.

## Testing

Add or extend (module names are suggestions; follow existing layout):

1. **`tests/ace/tui/widgets/test_agent_display_family_member_roster.py`** (new)
   - Rename-on-attach family: selecting the follow-up child renders `▾ ❖ FAMILY MEMBERS · 1 · alpha`, lists `--plan`
     only, and the published `MemberJumpMap` has `container_identity == child.identity` and one target `0` pointing at
     the root row.
   - The selected member never appears in the roster text or in the jump map.
   - Plan-workflow family (mirror `test_loaded_plan_family_roster_uses_concrete_member_state_and_content`): selecting
     the concrete planner step child lists the coder and vice versa.
   - Three-member family: selecting the middle member lists the other two in chain order numbered `0`,`1`.
   - Member panels stay on the agent scale: no `Fold: 1/2` line; at `COLLAPSED` the entries carry no activity
     annotation, at `EXPANDED` they do, and at `FULLY_EXPANDED` they add `ws N`.
   - A row with `family_container = None` (ordinary agent, synthetic planner, parallel-family member) renders no roster,
     and the container panel's own output is unchanged (assert the existing container expectations still hold).
2. **`tests/ace/tui/widgets/_agent_display_family_helpers.py`** — `make_family()` sets `child.family_container = root`
   after wiring `followup_agents`, with a comment that `sort_and_reorder()` does this in production.
3. **Ordering/model test** (extend `tests/ace/tui/widgets/test_agent_list_runtime_ordering.py` or add a focused module):
   `sort_and_reorder()` sets `family_container` for both family shapes, leaves it `None` on the container row, on
   synthetic planner rows, and on parallel-family rows, and **clears** a stale pointer when a re-run no longer lists
   that member.
4. **`tests/ace/tui/test_member_jump_navigation.py`** — with a family member selected: `0` jumps to the listed sibling;
   a target that is no longer a member of the resolved container cancels as stale; a two-digit family (12+ members)
   buffers the first digit against the member's own container identity; the existing container-panel cases still pass
   (including a root that is its own member 0).
5. **`tests/test_keybinding_footer_agent.py`** — a selected family member row advertises `("0-9", "member")`.
6. **Visual**: add a member-panel case to `tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel.py` (select a
   member row, assert the SVG contains `FAMILY MEMBERS` and does not contain the selected member's own label), and
   regenerate goldens with `just test-visual -- --sase-update-visual-snapshots` after visually confirming the new panel.

Run `just install` first (ephemeral workspace), then `just check`; run `just check-full` before landing since this
touches shared model, panel-render, and navigation code.

## Risks and mitigations

- **Reference cycle in the model.** Mitigated by `compare=False, repr=False` plus the `to_bundle_dict` skip entry; a
  missed flag surfaces immediately as `RecursionError` in the hint digest or agent equality. Cover it with the ordering
  test asserting `repr(member)` and `member == member` still work after attachment.
- **Stale pointer after a partial refresh.** `_clear_runtime_children()` resets the field on every reorder, and both
  live load paths reorder. A pointer that somehow survives a family dissolution still cannot produce a bad jump:
  `_current_member_target_is_valid()` re-derives members from the container at keypress time.
- **Panel-render cost.** Resolution is a single attribute read; roster building is the same pure, disk-free work the
  container panel already does for the same family (`agent_roster_digest` reads in-memory fields only).
