---
tier: tale
title: Show an aggregate ?N unknown-dependency indicator on agent clan rows
goal:
  Agent clan rows render ?N counting the distinct unknown wait dependencies of their
  WAITING members.
size: medium
proposed_by: bbugyi200.athena.0om
create_time: 2026-09-21 09:53:18
status: wip
---

# Show an aggregate `?<N>` unknown-dependency indicator on agent clan rows

## Goal

When one or more members of an agent clan render a `?<N>` wait indicator on their
Agents-tab row (unknown/missing agent or bead dependencies), the clan container row
should also render `?<N>`. `<N>` is the count of **distinct** unknown dependencies
across the clan's members. Example: clan `sase-11y.11` has a `.land` member rendering
`(WAITING ▶1 ⊘2 ✓2 ●3 ?1)`. After this change the clan row reads
`(RUNNING) [R1 W1 D3] ?1 sase-11y.11`.

This is presentation-only TUI aggregation. It builds on the existing Python wait-count
projection in `src/sase/ace/tui/_agent_completion_wait.py`, so no `sase-core` change is
needed.

## Current behavior (for orientation)

- `wait_dependency_status_counts(agent, status_maps, wait_bead_statuses)` in
  `src/sase/ace/tui/_agent_completion_wait.py` tallies one row's dependencies. Agent
  targets whose bucket is not a known `AGENT_STATUS_BUCKETS` value count as `unknown`.
  Clan targets expand via `status_maps.clan_member_statuses`, where an unknown member
  bucket also counts as unknown. Tribe targets are skipped. Bead targets with a non-cold
  snapshot entry whose status is not in `BEAD_STATUS_VALUES` count as `unknown`. Cold
  bead-cache misses are omitted.
- `append_agent_row_status` in
  `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py` renders these counts
  only when `agent.status == "WAITING"`. It goes through
  `format_wait_dependency_summary` in `src/sase/ace/tui/wait_status_presentation.py`.
  Unknowns render as `?N` using `WAIT_UNKNOWN_GLYPH` / `WAIT_UNKNOWN_GLYPH_STYLE`.
- Clan container rows (`agent.is_clan_container`) render the `[R.. W.. D..]` chip in
  `format_agent_option` in `src/sase/ace/tui/widgets/_agent_list_render_agent.py`. The
  chip's counts come from `clan_member_counts` and `clan_members` in
  `src/sase/ace/tui/models/_agent_clan.py`. No wait information from members reaches the
  clan row.
- Per-row context is built in `_row_context` / `build_row_inputs` in
  `src/sase/ace/tui/widgets/_agent_list_build_rows.py`. It is stored in
  `widget._row_render_ctx` and reused by `patch_row` in
  `src/sase/ace/tui/widgets/_agent_list_build_patching.py`.
- `src/sase/ace/tui/actions/agents/_loading_bead_warmup.py` re-patches rows whose
  wait-bead snapshot warmed. Right now it patches only the member row itself.

## Design

### 1. Shared unknown-target classification (`_agent_completion_wait.py`)

Add a public helper next to `wait_dependency_status_counts`:

```python
def wait_dependency_unknown_targets(
    agent: Agent,
    status_maps: AgentWaitStatusMaps,
    wait_bead_statuses: WaitBeadStatusSnapshot | None = None,
) -> frozenset[tuple[str, str]]:
```

It returns stable keys for exactly the dependencies that `wait_dependency_status_counts`
counts as `unknown`:

- `("agent", name)` for a non-tribe, non-clan `waiting_for` name whose bucket is not in
  `_WAIT_COUNT_FIELDS`.
- `("agent", f"{name}:{label}")` for a clan-target member (from
  `status_maps.clan_member_statuses[name]`) whose bucket is unknown.
- `("bead", bead_id)` for a non-cold snapshot entry whose status is not in
  `_BEAD_COUNT_FIELDS`.

The same `wait_display_agent(agent)` resolution and the same early return for "no waits"
must apply. Avoid two divergent copies of the classification. Factor the "is this
bucket/status unknown" predicates (the `else` branches of `_increment_agent_wait_count`
/ `_increment_bead_wait_count`) into small private predicates, and use them in both
functions. Parity invariant, enforced by a test: for any row,
`len(wait_dependency_unknown_targets(...)) == counts.agents.unknown + counts.beads.unknown`.

Also add:

```python
def clan_unknown_wait_dependency_count(
    clan: Agent,
    status_maps: AgentWaitStatusMaps,
) -> int:
```

- Return `0` unless `clan.is_clan_container`.
- Iterate `clan_members(clan)` from `sase.ace.tui.models._agent_clan`, deduplicated by
  `identity`. Use a lazy import inside the function, the same way this module already
  imports `wait_display_agent`, to avoid import cycles.
- Consider only members whose row would render the `?N` badge, which means
  `member.status == "WAITING"`. This mirrors the gate in `append_agent_row_status`.
  Without it, the clan could show a `?` that no visible member explains.
- Union each member's
  `wait_dependency_unknown_targets(member, status_maps, cached_wait_bead_status_snapshot(member))`
  and return the size of the union. `cached_wait_bead_status_snapshot` comes from
  `sase.ace.tui.models.agent_wait_beads` and is memory-only. It is already called per
  row in the render path, so this adds no I/O.
- Direct members only. A sequential-family member already resolves to its current shell
  through `wait_display_agent`. Nested clan containers are not members (they are
  excluded by `clan_members`).

Export both from `__all__` and re-export them through
`src/sase/ace/tui/agent_completion.py`, the way `wait_dependency_status_counts` is
exported.

### 2. Rendering (`_agent_list_render_agent.py`)

- Add a `clan_unknown_wait_count: int = 0` keyword to `format_agent_option` and
  `cached_format_agent_option`. Thread it into `agent_render_key` in
  `src/sase/ace/tui/widgets/_agent_list_render_cache.py`: add a keyword with default `0`
  and include it in the key tuple so a changed count busts the cache.
- In `format_agent_option`, inside the existing `if agent.is_clan_container:` block and
  immediately after the clan chip is appended, add: if `clan_unknown_wait_count > 0`,
  append `" "` and then `f"{WAIT_UNKNOWN_GLYPH}{clan_unknown_wait_count}"` styled with
  `WAIT_UNKNOWN_GLYPH_STYLE`. Import both constants from `..wait_status_presentation` so
  the glyph and color match member rows exactly. The indicator must render even if the
  chip is empty.

### 3. Row context and patching

- `_agent_list_build_rows.py`: in the per-row ctx dict, add
  `"clan_unknown_wait_count": clan_unknown_wait_dependency_count(agent, wait_status_maps) if agent.is_clan_container else 0`,
  and pass `ctx["clan_unknown_wait_count"]` through `format_agent_row` to
  `cached_format_agent_option`.
- `_agent_list_build_patching.py` `patch_row`: for clan container rows, recompute the
  count at patch time rather than reusing a possibly stale ctx value. A member's status
  or bead snapshot may have changed, the same way `clan_member_counts` is recomputed
  live. Use `_agent_wait_status_maps_for_build(widget, widget._agents)` from
  `_agent_list_build_rows.py`; rename it to a non-underscore name, or add a thin public
  wrapper, if Symvision's private-misuse rule requires it. Store the result back into
  `ctx["clan_unknown_wait_count"]`. Non-clan rows pass `0`.
- `_loading_bead_warmup.py`: after computing `changed_identities`, also patch every
  loaded clan container (from `current_by_identity`) where any `clan_members(container)`
  identity is in `results.wait_bead_status_identities`. Call
  `self._try_patch_agent_row(container)`. `patch_row` recomputes the clan count, so no
  new kwarg is needed. Fall back to the existing `needs_rebuild` path if the patch
  fails. Do not patch the same container twice in one pass.

### 4. Scope notes

- The CLAN detail-panel header (`Status: RUNNING [R1 W1 D3]`) is out of scope. The user
  asked for the list node only.
- Existing per-member `?N` rendering is unchanged.
- No config or keymap changes, so `src/sase/default_config.yml` does not change.

## Tests

Add focused tests. Mirror the fixtures and helpers of the existing files named here.

1. `tests/ace/tui/test_agent_wait_dependency_status_counts.py` (or a new sibling
   `test_clan_unknown_wait_dependency_count.py`):
   - Parity: `len(wait_dependency_unknown_targets(...))` equals
     `agents.unknown + beads.unknown` for rows with an unknown agent target, an unknown
     clan-member target, an unknown bead status, and a mix of these.
   - Cold bead-cache entries and tribe targets are not unknown targets.
   - `clan_unknown_wait_dependency_count` returns `0` for non-clan agents and for clans
     with no WAITING members. It counts unknowns from WAITING members only (ignores a
     non-WAITING member that still carries `waiting_for`). It dedupes the same missing
     target shared by two members, and it sums distinct agent and bead unknowns.
2. `tests/ace/tui/widgets/test_agent_list_runtime_clan_rendering.py` (or
   `test_agent_list_wait_dependency_status.py`): a clan whose WAITING member waits on a
   missing agent renders `?1` right after the `[..]` chip, styled with
   `WAIT_UNKNOWN_GLYPH_STYLE`. A clan with no unknowns renders no `?`. Reproduce the
   screenshot scenario: a clan with 1 running, 3 done, and 1 waiting `.land` member with
   one unknown dependency renders `[R1 W1 D3] ?1`.
3. `tests/ace/tui/widgets/test_agent_render_cache_clan.py` /
   `test_agent_render_key_wait_state.py`: `agent_render_key` differs when only
   `clan_unknown_wait_count` changes.
4. A patch/warmup test: after a member's wait-bead snapshot warms to an unknown status,
   the bead warmup path patches the clan container row and the rendered clan row shows
   `?1`. Build it on the existing bead-warmup tests; locate them with
   `grep -rln "_loading_bead_warmup\|wait_bead_status_identities" tests/`.

## Verification

- Run `just fmt`, then `just check`. Do not run `just check-full`.
- If any PNG visual snapshot golden includes a clan whose members have unknown waits, it
  will change. Inspect that intentionally; `just check` does not run PNG goldens, so
  mention any expected golden churn in the final summary instead of regenerating it
  unasked.
