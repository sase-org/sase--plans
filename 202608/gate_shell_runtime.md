---
tier: tale
title: Exclude gate-shell windows from family and clan accumulated runtime
goal:
  An agent family's or clan's accumulated runtime counts only time agents and monitors
  actually ran, never the human-decision window a gate shell owns.
size: medium
proposed_by: bbugyi200.athena.0g1
create_time: 2026-08-29 09:18:49
status: wip
---

# Plan: Exclude gate-shell windows from family and clan accumulated runtime

## Problem

The Agents-tab runtime suffix on a sequential-family container row (and on the clan
container above it) is an aggregate over the row's `runtime_children`. That aggregate
currently treats a **gate shell** exactly like any other family member: it hands the
gate's `run_started_at` → `stopped_at` interval to the Rust interval-union aggregator
along with the real agent shells.

A gate shell's interval is not agent runtime. `run_started_at` is stamped when the gate
member's artifacts are created (`create_followup_artifacts`, i.e. the moment the gate
goes `pending`), and `stopped_at` is stamped at settlement
(`src/sase/gate_shell/settlement.py`). Everything between those two marks is a human
reading a plan and clicking a button. So the number that is supposed to answer "how long
did agents run for this family?" silently absorbs review latency, which on a plan gate
is routinely longer than the agents' own work.

### Reproduced

With the shape real artifacts actually produce (agent shells attach to the family root;
a gate shell attaches to the shell that created it), a plan family whose planner ran 30
minutes and whose plan then sat in review for 90 minutes:

| Situation                                       | Family row shows today | Should show |
| ----------------------------------------------- | ---------------------- | ----------- |
| Planner done (30m), plan gate `pending` for 90m | `2h`                   | `30m`       |
| Gate answered, coder running 10m                | `2h10m`                | `40m`       |

Two aggravating details found while reproducing this:

1. During the pending window the family total **grows without the `🏃‍♂️` live marker**.
   `runtime_suffix_ticks()` correctly declines to tick (a `pending` gate is not
   `settling`, and every agent shell has stopped), but `_aggregate_runtime()` still
   reports the gate as live, so each repaint renders a larger number with no marker
   explaining why. The two code paths already disagree about whether a pending gate is
   running; this plan makes them agree that it is not.
2. Once the gate settles, its terminal timestamp can win `max(terminal_times)` and
   become the family's displayed _finish_ time, so a settled family is stamped with the
   moment a human clicked rather than the moment its last agent stopped.

## Doctrine this plan establishes

**A gate shell owns no agent runtime.** It never contributes its own interval to any
ancestor's accumulated runtime, at either the family or the clan level, in any gate
state.

This is deliberately different from a monitor shell, which keeps contributing: a monitor
is one supervised OS command doing machine work for the family, and `docs/monitors.md`
already documents that it counts. A gate shell contains no LLM and no command until a
human answers; its duration measures the human, not the machine.

The gate shell's _own row_ is unchanged — it keeps its own suffix, and it keeps ticking
while `gate_state == "settling"`. Only the propagation of that interval into an
ancestor's total is removed. The follow-up agent a gate launches is an ordinary agent
shell and keeps contributing normally.

### Why not exclude only the pending half

The alternative considered was to keep the `settling` window (option commands executing)
and drop only the `pending` window. It was rejected: SASE records no settling-start
timestamp today (`settlement.py` writes `gate_state = "settling"` with no mark of its
own), so it would require a new metadata field plumbed through the artifact marker, the
scan wire, the loader, and the `Agent` model, plus a fallback for every gate already on
disk — a large change for a window that is normally seconds long, and one whose machine
work is already represented by the follow-up agent's own row.

## Where the change goes

All of it lands in `src/sase/ace/tui/models/agent_time.py`. That module is the only
caller of `sase.core.agent_runtime_facade.aggregate_clan_runtime`, and the
include/exclude policy for family shells already lives there
(`_aggregates_family_shells`, `_is_family_shell`).

**On the Rust-core boundary:** the interval-union _math_ stays in
`sase_core::agent_runtime`; what changes here is only which rows the caller submits,
which is a property of the TUI's loaded `Agent` tree and has no non-TUI consumer today.
If a CLI or web frontend later needs family totals, the right move is to carry a
shell-kind discriminator on `ClanRuntimeMemberWire` and drop gate members inside
`aggregate_clan_runtime` — do **not** duplicate this filter in a second Python caller.

## Implementation

### 1. One shared child projection

Three traversals in `agent_time.py` walk `runtime_children` with the same filter and
must keep agreeing: `_aggregate_runtime` (line ~399), `runtime_suffix_ticks` (line
~580), and `row_runtime_or_wait_ticks` (line ~655). Replace the three copies of
`if _is_family_shell(child) and not include_*_shells: continue` with one helper:

```python
def _runtime_child_rows(
    agent: "Agent", *, include_monitor_shells: bool
) -> tuple["Agent", ...]:
    """Return the child rows whose runtime an ancestor row may absorb.

    A gate shell owns a human decision window rather than agent runtime, so
    it never contributes its own interval to an ancestor at any level. Its
    own children are yielded in its place, so an agent a gate started is not
    dropped along with the gate. Monitor shells still contribute, but only to
    family and clan container rows.
    """
```

Requirements for the helper:

- Expand a gate child into that gate's own `runtime_children`, recursively (a gate
  chained to a gate must not resurrect either interval).
- Cycle-guard on `id(row)`; `Agent` is mutable and unhashable, and `runtime_children`
  can contain cycles — mirror the guards already used in `_ShellLaneTally` in
  `agent_family_members.py`.
- Keep monitor handling byte-identical to today: excluded when `include_monitor_shells`
  is false, and excluded _without_ being expanded into its children (that is the
  existing behavior fixed by `20f0d1395`, "stop settled monitor starters from borrowing
  monitor runtime"; do not change it here).
- Use `row.is_gate` / `row.is_monitor` rather than reusing `_is_family_shell`, which no
  longer expresses one rule. If `_is_family_shell` ends up with no callers, delete it —
  symvision will fail the build on a dead private helper
  (`sase memory read symvision.md`).

### 2. `_aggregate_runtime`

Drive `append_runtime_member` from `_runtime_child_rows(...)` at both the top-level loop
and the grandchildren recursion. Verify after the change that a gate contributes neither
a `ClanRuntimeMemberWire` entry nor a `terminal_times` entry, so a settled gate can no
longer become the family's displayed finish timestamp.

Note the existing early return: a child that has `runtime_children` is treated as an
aggregate proxy and is not appended itself. Leave that rule alone (see _Out of scope_).

### 3. `runtime_suffix_ticks` and `row_runtime_or_wait_ticks`

Drive both child loops from the same helper. A gate must not make an ancestor tick, so
the ancestor must never call `runtime_suffix_ticks(gate)` — it evaluates the gate's own
row and returns `True` while settling. Consulting the gate's expanded children instead
is exactly what the helper provides.

Leave the self-decision branch `if agent.is_gate and agent.gate_state == "settling"`
(line ~590) untouched: that is the gate's own row deciding about itself, and it stays
correct.

### 4. `compute_lowest_row_runtime` (clan lanes)

No code change expected. It already falls back to `_leaf_runtime_interval` and skips a
row whose interval is not active, so a lane parked on a gate is dropped from the
`<lowest-running-lane-runtime>` half rather than pinning it to a human wait. Add a test
that pins this rather than assuming it.

### 5. Render-cache check

No change expected in `src/sase/ace/tui/widgets/_agent_list_render_cache.py`:
`_runtime_signature` recurses into `runtime_children` and already folds `gate_state`
into every child's signature, so a family row still repaints when its gate settles even
though that row is no longer time-sensitive during the wait. Confirm this with the
existing `test_agent_render_cache.py` suite plus one new case rather than by inspection.

## Tests

Add a `gate_shell(...)` factory to
`tests/ace/tui/widgets/agent_list_runtime_helpers.py`, mirroring the existing
`monitor_shell(...)`: `agent_family_role="gate"`, `role_suffix="--gate"`, a non-empty
`gate_id`, and a caller-chosen `gate_state` (`is_gate` requires both the role and a
durable id).

`tests/ace/tui/widgets/test_agent_list_runtime_compute.py`:

- A family container with a done planner and a `pending` gate shows the planner's total,
  not the planner-plus-review span. Use the reproduced numbers above: `30m`, not `2h`.
- The same family after the gate is answered and a coder is running shows `40m`, not
  `2h10m`.
- A family whose last settled member is a gate reports its last _agent_ stop as the
  finish timestamp, not the gate's `stopped_at`.
- A gate shell's own row still reports its own runtime (`compute_leaf_row_runtime`),
  proving the exclusion is about propagation only.
- An agent row attached beneath a gate still contributes to the family total (defends
  the recursion in `_runtime_child_rows`).
- A regression guard that a monitor shell still contributes to a family container total.

`tests/ace/tui/widgets/test_agent_list_runtime_ticks.py`:

- A family container does not tick while its gate is `pending`, and does not tick while
  its gate is `settling`, for both `runtime_suffix_ticks` and
  `row_runtime_or_wait_ticks` (the existing `_TICK_DECISIONS` parametrization covers
  both).
- The gate row itself still ticks while `settling` and still does not tick while
  `pending`.

`tests/ace/tui/widgets/test_agent_list_runtime_clan_rendering.py`:

- A clan whose family lane is parked on a pending gate: the clan total excludes the gate
  window, and the `<lowest-running-lane-runtime>` half does not pin to the gate.

`tests/ace/tui/widgets/test_agent_render_cache.py`:

- A cached family row is re-rendered after its gate settles (proves the signature still
  invalidates once the row stops being time-sensitive).

### PNG snapshots

`tests/ace/tui/visual/_ace_agents_png_snapshot_family_panel_fixtures.py::_gate_family_agents`
builds gate rows with `parent_timestamp=starter.raw_suffix` and runs them through
`sort_and_reorder`, so its container row's runtime suffix is expected to change. Run
`just test-visual`, inspect the artifacts under `.pytest_cache/sase-visual/`, confirm
the only deltas are the container runtime suffix, then accept them with
`--sase-update-visual-snapshots`. Affected goldens are likely
`agents_family_panel_shells_gate_90x40.png` and
`agents_family_panel_shells_gate_120x40.png`; regenerate whatever the run actually
reports rather than trusting this list.

## Docs

- `docs/ace.md` (runtime-suffix section, near the
  `🏃‍♂️ <current-shell-runtime> / <family-total-runtime>` description): state that
  gate-shell windows are excluded from the family and clan totals.
- `docs/agent_families.md`: the runtime paragraph currently reads "The runtime is the
  union of member run intervals, with human-wait windows excluded" — say explicitly that
  a gate shell's window is one of those excluded human waits. Also add the exclusion to
  the `--gate` suffix description, which already notes that a pending gate occupies no
  runner slot; the same reasoning applies to accumulated runtime.
- `docs/monitors.md` states monitor shells "contribute to the family's total runtime".
  Add the contrast so the two shell kinds are not read as one rule.

Do not edit `sase/memory/` notes; nothing there states the old behavior.

## Verification

```bash
just install          # ephemeral workspace clones need this before anything else
just check            # required after any file change in this repo
just test-visual      # PNG goldens; regenerate as described above
```

Run `just check-full` through `/sase_monitor` (status pair `TESTING` / `TESTED`) before
landing — this changes a shared display primitive consumed by the agent list, the clan
roster, the member roster digest, the tribe summary, and the cleanup modal, and the
scoped lane may not select all of them.

## Out of scope

While reproducing the reported bug, a **separate pre-existing defect** was confirmed in
the same function and is deliberately not fixed here: `_aggregate_runtime`'s
`append_runtime_member` skips any child that has `runtime_children` of its own, treating
it as an aggregate proxy for those children. That heuristic is correct for a plan-family
root represented by its planner workflow step, but wrong for a real family chain: a
family member that started a monitor has the monitor as a child, so **the member's own
interval is dropped from the family total**.

Measured: a family of planner (30m) + coder (60m) + a monitor the coder started (6m,
overlapping) reports `36m` instead of the correct `1h35m` union. Every `--code` member
that starts a monitor disappears from its family's total. This is a much larger
under-count than the gate over-count; folding it in here would change nearly every
family total in the same commit and make the gate fix impossible to review.

It is recorded as a `DISCOVERED ISSUE:` note on epic `sase-kp` ("sase monitor —
long-running commands as first-class agent family members"), whose phase `sase-kp.7`
nested monitor shells under their starter and so created the shape that triggers the
latent heuristic. Read that note before touching `append_runtime_member`'s early return:
a fix there and this plan's gate filter both live in the same function, so whichever
lands second must rebase rather than re-derive the child projection.

Also out of scope: the `_aggregates_family_shells` container rule, monitor contribution,
`sase stats` agent counts (gate shells are already excluded there), and the runner-slot
occupancy rules in `sase_core::agent_runtime` (a pending gate is already correctly
non-occupying).

## Risks

- **Silent double removal.** If the implementer excludes gates in the aggregate but
  forgets one of the two tick predicates, the family row will freeze at a stale number
  while a gate settles, or keep ticking on a total that no longer moves. The shared
  `_runtime_child_rows` helper exists specifically to make that impossible; do not
  hand-write the filter three times.
- **Dropping a gate's descendants.** Returning early on a gate without expanding its
  children would delete a real agent's runtime from the family total. Today gate
  follow-ups attach to the family root rather than to the gate, so the bug would not
  show up in current artifacts — the test named above is the only thing defending it.
- **Monitor regression.** `20f0d1395` fixed settled monitor starters borrowing monitor
  runtime by _not_ expanding an excluded monitor into its children. Preserve that
  asymmetry: gates expand, monitors do not.
