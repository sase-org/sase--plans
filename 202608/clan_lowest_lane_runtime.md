---
tier: tale
title: Show the lowest running-lane runtime on agent clan nodes
goal:
  An agent clan node in the ACE Agents tab renders `🏃‍♂️ <lowest-running-lane-runtime> /
  <clan-total-runtime>`, mirroring the current-shell prefix that sequential family nodes
  already show.
size: medium
proposed_by: bbugyi200.athena.0d4
create_time: 2026-08-24 19:08:16
status: wip
---

# Plan: Show the lowest running-lane runtime on agent clan nodes

## Background

Commit `184fa9aed` ("feat(ace): show current family shell runtime") added a live prefix
to the runtime suffix of an **agent family** container row. When a sequential family is
ticking, its row now renders:

```
🏃‍♂️ <current-shell-runtime> / <family-total-runtime>
```

The moving parts that shipped there are the template this plan follows:

- `src/sase/ace/tui/models/agent_family_members.py` :: `current_family_shell_row()` — a
  pure projection that returns the family's currently in-flight concrete shell (agent
  shell _or_ monitor shell), or `None`.
- `src/sase/ace/tui/models/agent_time.py` :: `compute_leaf_row_runtime()` — that row's
  own interval, excluding descendants.
- `src/sase/ace/tui/widgets/_agent_list_render_layout.py` :: `build_runtime_suffix()` —
  computes the prefix only when `elapsed is not None and is_ticking`, then appends
  `<prefix>` + `" / "` before the existing elapsed value.

An **agent clan** node gets no such prefix today. It renders only the aggregate clan
runtime, which `agent_time._aggregate_runtime()` derives from
`sase.core.agent_runtime_facade.aggregate_clan_runtime` (a union of member run intervals
with human-wait windows excluded).

A clan differs from a family in exactly one way that matters here: a family has at most
one shell executing at a time, while a clan can have many lanes executing in parallel.
The user's decision is to collapse that set to its **minimum**.

## Desired behavior

On an agent clan container row that is currently ticking, the live runtime suffix
becomes:

```
🏃‍♂️ <lowest-running-lane-runtime> / <clan-total-runtime>
```

### The exact rule

1. Project the clan container into its **lanes** — the Agents-tab agent nodes it
   contains. This is the same projection `_agent_clan.clan_members()` already feeds to
   `clan_member_counts()` (the `[R… W… F…]` chips on the same row) and to
   `sase_agent_status_counts()` (the runner-slot lane count). One sequential family is
   one lane no matter how many shells it owns.
2. For each lane, resolve the row whose runtime represents that lane's _current_ work:
   - a sequential family lane resolves to `current_family_shell_row(lane)` — the exact
     row whose duration that family renders on the left of its own `/`;
   - any other lane represents itself, but only when `agent_row_is_in_flight(lane)`;
   - a lane with nothing executing contributes nothing.
3. Compute each representative's active elapsed seconds and take the **minimum**, then
   format it once with `format_compact_duration`.
4. Render the result exactly like the family prefix: same style constant
   (`_RUNTIME_ELAPSED_STYLE`), same bare `" / "` separator, same position after the `🏃‍♂️`
   marker.

### Deliberate consequences

- **Reduces to the family rule.** A clan whose single running lane is a family shows
  that family's current-shell runtime, so the clan row and the family row underneath it
  agree on the left value.
- **No suppression when the two halves are equal.** The family feature already renders
  `🏃‍♂️ 3m05s / 3m05s` (see
  `test_format_agent_option_active_family_shows_current_root_and_total`); clans stay
  consistent with that and do not special-case a redundant-looking pair. In the
  screenshot attached to the request, clan `sase-sq.7.1` has exactly one running lane
  (`sase-sq.7.1.1`, a single-shell agent whose own row reads `16m46s`), so that capture
  would render `16m46s / 16m46s`; the `15m30s /` in the request illustrates the shape,
  and the rule above is what decides the number.
- **Compare seconds, never formatted strings.** `format_compact_duration` output does
  not sort correctly (`"1h05m" < "45m"` lexically). The minimum must be taken on
  `_RuntimeInterval.elapsed_seconds`.
- **No prefix when nothing qualifies.** A ticking clan whose lanes yield no computable
  active interval renders the plain, unchanged suffix — never a dangling `" / "`.

## Design decisions

**Where the code lives.** The lane projection is pure presentation over already-loaded
TUI `Agent` rows, exactly like `current_family_shell_row`, so it stays in Python
alongside its family sibling rather than crossing into `../sase-core`. Applying the
`rust_core_backend_boundary` litmus test: another frontend would need
`aggregate_clan_runtime` (already in the Rust core, unchanged by this plan), but "which
loaded Agents-tab row represents this lane right now" is a property of the TUI's row
projection, not of the durable agent domain. No wire, binding, or `sase-core` change is
required, and none should be made.

**Import layering.** `agent_time.py` must not import `_agent_clan.py`: `_agent_clan` →
`agent_family_members` → `agent` → `agent_time` is an existing edge, so the reverse
would cycle. The split below keeps time math in `agent_time` and row selection in
`_agent_clan`, which is also how the family feature is split.

**No feature flag.** Per `sase/memory/sase_flags.md`, flags cover user-reaching behavior
that is not ready. This is a finished refinement of an already-shipped suffix, and the
family half shipped unflagged; do not create one.

## Implementation

### 1. `src/sase/ace/tui/models/_agent_clan.py` — lane representative projection

Add a pure, time-free projection:

```python
def clan_current_lane_rows(agent: Agent) -> tuple[Agent, ...]:
    """Return one representative in-flight row per running lane of a clan."""
```

- Return `()` when `not agent.is_clan_container`.
- Iterate `clan_members(agent)`. For each member, skip it unless
  `is_agents_tab_agent_node(member)` (already imported in this module).
- Resolve the representative as `current_family_shell_row(member)`, falling back to
  `member` itself when that is `None` **and** `agent_row_is_in_flight(member)`; skip the
  lane when neither applies.
- Dedupe on `row.identity` (durable identity, not `id()`), preserving `clan_members`
  order.
- Import `agent_row_is_in_flight` and `current_family_shell_row` from
  `.agent_family_members` — this module already imports from there, so no new
  module-level dependency is introduced.
- Add `"clan_current_lane_rows"` to the module's `__all__`.

The docstring must state that a sequential family lane resolves to the same row the
family row shows on the left of its `/`, and that nested clan containers are not clan
members (matching `clan_members` and `_lane_summary_projections`), so they are not
walked.

### 2. `src/sase/ace/tui/models/agent_time.py` — minimum active elapsed

Add, next to `compute_leaf_row_runtime`:

```python
def compute_lowest_row_runtime(
    rows: Sequence["Agent"],
    now: datetime | None = None,
) -> str | None:
    """Return the smallest still-active elapsed duration among *rows*."""
```

- Resolve `reference = now if now is not None else local_now()`.
- Skip any row where `should_display_runtime_suffix(row)` is `False`, matching
  `compute_row_runtime` / `compute_leaf_row_runtime`.
- Per row, prefer `_leaf_runtime_interval(row, reference)`; when that is `None` or not
  `active`, fall back to `_runtime_interval(row, reference)` so a lane that is a
  workflow aggregate row (no leaf interval of its own) still contributes.
- Keep only intervals with `active` true and `terminal_time is None`.
- Track the minimum `elapsed_seconds` and return `format_compact_duration(lowest)`, or
  `None` when no row qualified.
- Add `Sequence` to the `collections.abc` import.

### 3. `src/sase/ace/tui/widgets/_agent_list_render_layout.py` — one new branch

In `build_runtime_suffix`, extend the existing prefix block with a single `elif`; the
append block that writes `current_shell_elapsed` + `" / "` is unchanged:

```python
    if elapsed is not None and is_ticking:
        current_shell = current_family_shell_row(agent)
        if current_shell is not None:
            _shell_ts_pair, current_shell_elapsed = (
                agent_time_model.compute_leaf_row_runtime(current_shell, now=reference)
            )
        elif agent.is_clan_container:
            current_shell_elapsed = agent_time_model.compute_lowest_row_runtime(
                clan_current_lane_rows(agent), now=reference
            )
```

`current_family_shell_row` already returns `None` for clan containers, so the two
branches are mutually exclusive by construction. Import `clan_current_lane_rows` from
`..models._agent_clan` (the sibling `_agent_list_render_cache.py` already imports from
that module, so this is an established edge).

No change is needed in `_agent_list_build.patch_row` / `AgentList._format_agent_option`:
live patching re-runs `format_agent_option`, so the clan prefix ticks for free once
`row_runtime_or_wait_ticks` reports the clan row as ticking (it already does, because it
recurses `runtime_children`).

### 4. `src/sase/ace/tui/widgets/_agent_list_render_cache.py` — close the cache desync

`_runtime_signature` already recurses `runtime_children` and `followup_agents` and
captures each row's `status`, `run_start_time`, `stop_time`, `agent_family*`,
`role_suffix`, and monitor fields, which covers step 2 of the rule. It does **not**
capture the fields `clan_members` and `is_agents_tab_agent_node` read to decide step 1.
Add to the returned tuple (the key is deliberately explicit, per its own docstring):

- `agent.agent_clan`
- `agent.agent_clan_generation`
- `agent.is_clan_container`
- `agent.child_linkage`
- `agent.agent_name`

Before finishing, re-read `clan_current_lane_rows` and confirm every field it reads is
either newly added or already present; a missing field is a silent stale-row bug, which
is exactly what that docstring warns about.

### 5. Tests

Repo convention (see commit `f2f0bd977`, "test: split agent-list runtime rendering tests
under 500-line files") keeps test modules under ~500 lines.
`test_agent_list_runtime_rendering.py` (291) and `test_agent_list_runtime_patching.py`
(481) are both too close to that ceiling, so the widget-level clan cases go in a **new**
module.

**a. `tests/test_agent_clan.py`** (126 lines) — unit-test `clan_current_lane_rows`
directly, following the `_agent(...)` + `container.is_clan_container = True` fixture
style already in that file:

- a running single-shell lane represents itself;
- a family lane resolves to its currently executing shell, not the family root;
- a family lane whose current shell is a running monitor resolves to that monitor
  (parity with `current_family_shell_row`);
- settled / waiting / failed lanes contribute nothing;
- members of another clan or another `agent_clan_generation` are excluded;
- a non-clan row returns `()`.

**b. `tests/ace/tui/widgets/test_agent_list_runtime_clan_rendering.py`** (new) — reuse
`.agent_list_runtime_helpers` (`agent`, `workflow_child`, `AgentListHarness`,
`agent_row_index`) and assert on `format_agent_option(...)[1].plain`:

- two running lanes at different durations → the clan suffix carries the **lower** one:
  `"🏃‍♂️ 45m / <clan-total>"`;
- an explicit ordering guard: lanes at `1h05m` and `45m` must yield `45m`, proving the
  minimum is taken on seconds rather than on the formatted string;
- a clan whose only running lane is a family shows that family's current-shell value,
  and the clan row's left value equals the family row's left value;
- a clan with no running lane (all `DONE` / `WAITING`) renders no `" / "`;
- a live-patching case through `AgentListHarness` (mirroring
  `test_patch_active_runtime_rows_advances_family_current_and_total`) proving both
  halves advance on a tick, and that the clan value follows the _new_ minimum when a
  fresh lane starts.

**c. `tests/ace/tui/widgets/test_agent_render_cache.py`** (362 lines) — add a case
proving `agent_render_key` changes when a clan member's status flips a lane between
running and settled, and another proving it changes when a member's `agent_clan` moves
it out of the clan. Keep the module under ~500 lines; split if it would not fit.

**d. PNG visual snapshot.** Add `running_clan_runtime_agents()` to
`tests/ace/tui/visual/_ace_agents_png_snapshot_clan_fixtures.py` (252 lines): a clan
container plus two running lanes with different run starts (one of them a sequential
family so the family-inside-clan case is visible) and one settled lane. Add a test to
`tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py` (212 lines) modeled on
`test_running_family_current_runtime_png_snapshots`: pin `now` with
`pin_agents_visual_now`, assert both durations appear in the SVG, and capture
`agents_running_clan_runtime_collapsed_120x40` plus an expanded variant after `l`.
Generate the goldens with `just test-visual --sase-update-visual-snapshots` and confirm
the diff artifacts under `.pytest_cache/sase-visual/` before accepting them.

### 6. Docs

- `docs/ace.md` — the runtime-suffix paragraph (currently near line 1990) already
  documents the family case. Add the clan sentence right after it: on an active clan
  container the live suffix is
  `🏃‍♂️ <lowest-running-lane-runtime> / <clan-total-runtime>`, where the left value is the
  smallest current runtime among the clan's running lanes (a sequential family lane
  contributes its currently executing shell) and the right value is the aggregate clan
  interval.
- `docs/agent_families.md` — the clan paragraph that ends with the family
  `<current-shell-runtime> / <family-total-runtime>` sentence (near line 190) gets the
  matching clan sentence, stating that a clan collapses its parallel lanes with a
  minimum because more than one lane can be live at once.

Run `just fmt-md` (or `just fmt`) so prettier reflows the edited paragraphs.

## Verification

```bash
just install          # ephemeral workspace: always first
just check            # whole-repo lint gates + diff-scoped tests
just test-visual      # PNG snapshot suite (new clan golden)
```

`just check-full` must run before landing, and it routinely outruns one turn, so hand it
to a monitor rather than running it inline:

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED --next '...'
```

## Out of scope

- The `CLAN` detail-panel header and its member roster
  (`widgets/prompt_panel/_agent_display_clan_roster.py`, `_member_roster_digest.py`),
  which render per-member runtimes through `compute_row_runtime` and are unchanged.
- Tribe panel titles and grouping-banner chips.
- Any change to `aggregate_clan_runtime` or anything else in `../sase-core`; the clan
  total on the right of the `/` keeps its current value and derivation.
- Suppressing the prefix when the two halves are equal — explicitly rejected above for
  parity with the shipped family behavior.
