---
tier: tale
title: Grey settled-monitor gear count on clan and family container rows
goal:
  A clan or family container row shows an amber gear count for its running monitors and
  a quieter grey gear count for its finished ones, with the two lanes partitioning that
  subtree's monitors exactly.
size: medium
proposed_by: bbugyi200.athena.067
create_time: 2026-08-18 10:42:03
status: done
---

# Settled-monitor gear chip on clan and family container rows

## Goal

Agent clan container rows and agent family container rows in the ACE Agents tab today
carry a single amber `⚙N` badge counting the **running** monitors anywhere in their
subtree. Add a second, quieter **grey** `⚙N` badge on those same rows counting the
monitors in that subtree that have **finished**, so a collapsed node tells you both "how
much monitored work is still in flight" and "how much monitored work already ran"
without expanding anything.

The two badges must partition the monitor rows beneath a node exactly: every monitor is
counted in exactly one of the two lanes, never both, never neither.

## Design

### The visual

A container row grows at most two gear badges, rendered in this order immediately after
the clan status chip and before the trailing agent-name identity block:

```
▶ ◆ acme.1 [S1 R2] ⚙1 ⚙4 acme.1 @tribe
                   │  └── grey: 4 finished monitors in this subtree
                   └───── amber: 1 running monitor in this subtree
```

Design decisions and why:

- **Same glyph, hue carries the lane.** The gear is already SASE's monitor mark
  everywhere (top-bar `MonitorIndicator`, the Procs tab rows, monitor rows themselves).
  Two adjacent gears differing only in hue is an idiom this codebase already ships: the
  Procs tab header renders `⚙ 2  ⚙ 1` (blue procs lane, orange monitors lane) for
  exactly this "two lanes of the same object" purpose. Reusing it means the new badge
  needs no new vocabulary from the user.
- **Live first, history second.** Amber (running) keeps the leftmost, stable slot so the
  live signal never shifts position as history accumulates behind it. This also matches
  `AGENT_COUNT_CHIP_METRICS` ordering in `src/sase/ace/tui/agent_count_chip.py`, where
  `done` is the last metric in the chip.
- **Grey recedes, amber pops.** The settled badge renders `#9E9E9E` **non-bold**; the
  running badge keeps its existing `bold #FFAF5F`. `#9E9E9E` is already this codebase's
  neutral-grey token (`_IMPLICIT_TAG_STYLE` in
  `src/sase/ace/tui/model_alias_styles.py`), so no new palette value is introduced.
  Encoding the lane in **both** hue and weight keeps the two badges distinguishable on
  low-color terminals and for red/green color vision deficiency; it also means the eye
  lands on the live count first.
- **Zero-suppressed, both lanes.** A lane with no monitors renders nothing. Rows are
  dense and most nodes have no monitors at all; a dim `⚙ 0` belongs in the Procs tab
  header (where a fixed two-lane header must never read as "unknown") but would be pure
  noise on a list row. A lone badge is unambiguous because the hue names its lane.
- **Same rows as the existing badge.** Both badges are gated on the one existing
  container predicate
  (`agent.is_clan_container or is_sequential_family_container(agent)`). One predicate
  for both lanes means a node can never show one badge while being ineligible for the
  other.
- **Rendered whether folded or expanded**, matching the current amber badge. An expanded
  family with a dozen members still benefits from the aggregate, and a badge that
  appears and disappears with fold state would read as a fold artifact rather than as a
  property of the node.
- **No cap on the count.** A three-digit count costs five cells and is honest; a `99+`
  form would hide information the badge exists to show.

### The lane rule (the "no double counting" contract)

The two lanes are defined as a strict partition over the monitor rows beneath a node:

```
settled(row) := monitor_state_is_terminal(row.monitor_state) or row.stop_time is not None
running(row) := not settled(row)
```

where terminal means the monitor state buckets to something other than `Running` in
`MONITOR_STATE_BUCKETS` — that is, `completed`, `stopped`, `failed`, `timeout`, `lost`.
An unrecognized or missing `monitor_state` is **not** terminal, matching the doctrine
already written into `monitor_state_bucket`: a monitor that has not yet reached a
terminal state must never read as finished.

Consequences worth stating explicitly, because they are the whole point of the rule:

| `monitor_state` | `stop_time` | lane  | vs. today                      |
| --------------- | ----------- | ----- | ------------------------------ |
| `running`       | unset       | amber | unchanged                      |
| `running`       | set         | grey  | today: counted in neither lane |
| terminal        | any         | grey  | today: counted in neither lane |
| unknown/`None`  | unset       | amber | today: counted in neither lane |
| unknown/`None`  | set         | grey  | today: counted in neither lane |

The first row is the only case the existing amber badge counts today, so the amber
badge's only behavior change is that a monitor that is mid-launch (row created, state
not yet enriched from `agent_meta.json`, no stop time) now counts amber instead of
vanishing from both lanes. That is a strict improvement and is required for the
partition to hold; it also matches what the rest of ACE already does with that row,
since `apply_monitor_meta` gives it `status_bucket = "Running"`.

Note that `running(row)` as defined above subsumes the existing
`agent_row_is_in_flight(row)` test for monitor rows
(`monitor_state == "running" and stop_time is None` implies not-terminal and no stop
time), so the new predicate is a superset, not a redefinition, and
`agent_row_is_in_flight` itself must **not** change — other callers (family bucket
projection) depend on its current meaning.

Both lanes are counted in **one** traversal that keeps the existing cycle guard
(`id(row)`) and identity dedupe (`row.identity`), so a monitor attached to both
`runtime_children` and `followup_agents` — which real family rows do — is counted once.

### Deliberate non-goals

- **No third, red "failed monitors" lane.** A failed, timed-out, or lost monitor counts
  grey along with a clean completion: the badge answers "still running or not". A
  separate failure lane is a real idea (a collapsed clan currently hides monitor
  failures entirely, since `concrete_agent_statuses` excludes monitor rows from clan
  status chips) but it is a different feature with a different question behind it, and a
  third gear on an already-dense row needs its own design pass. Out of scope here.
- **No change to the top bar, the Procs tab, or the FAMILY MEMBERS roster panel.** The
  top-bar `MonitorIndicator` is an ambient "is anything running right now" signal and
  the Procs tab already reports done totals in its own header.
- **No Rust core change.** Per the Rust core backend boundary rule: the classification
  derives from the already-shared `monitor_state` string and adds no persisted field or
  wire shape, and its data table (`MONITOR_STATE_BUCKETS`) already lives in
  `src/sase/monitor_state.py`. The new predicate goes next to that table. Do not touch
  `../sase-core`.

## Implementation

### Step 1 — terminal-state predicate (`src/sase/monitor_state.py`)

Add, beside `MONITOR_STATE_BUCKETS` and `monitor_state_bucket`:

- `monitor_state_is_terminal(monitor_state: str | None) -> bool`, returning
  `monitor_state_bucket(monitor_state) != "Running"`. Implementing it in terms of the
  existing bucket map (rather than a second hardcoded state set) guarantees the two
  cannot drift when a monitor state is added.
- Export it from `__all__`.

Docstring must state the unknown-state doctrine: an unrecognized or missing state is not
terminal, so a monitor that has not yet reported never reads as finished.

### Step 2 — two-lane counts (`src/sase/ace/tui/models/agent_family_members.py`)

- Add `monitor_row_is_settled(row: Agent) -> bool` implementing the rule above, next to
  `agent_row_is_in_flight`. Its docstring should explain why `stop_time` alone settles a
  row (the family member row is over even if its monitor record was never reconciled).
- Add a frozen, slotted `MonitorLaneCounts` dataclass with `running: int = 0` and
  `settled: int = 0`, mirroring `ClanStatusCounts` in
  `src/sase/ace/tui/models/_agent_clan.py`. It must be hashable — it becomes part of the
  agent render cache key.
- Add a module-level `NO_MONITOR_LANES = MonitorLaneCounts()` zero sentinel.
- Replace `running_monitor_count(agent) -> int` with
  `monitor_lane_counts(agent) -> MonitorLaneCounts`: one traversal, same cycle guard and
  identity dedupe as today, incrementing exactly one lane per distinct monitor row. Do
  not keep `running_monitor_count` as a wrapper — Symvision flags unused symbols, and
  every call site is updated in this plan.
- Update `__all__`.

### Step 3 — settled badge style (`src/sase/ace/tui/widgets/_agent_list_styling.py`)

Add `_MONITOR_SETTLED_COUNT_GLYPH_STYLE = "#9E9E9E"` beside the existing
`_MONITOR_COUNT_GLYPH_STYLE`, with a comment recording that the settled lane is
deliberately non-bold so hue _and_ weight both separate it from the live amber lane.

### Step 4 — render both badges (`src/sase/ace/tui/widgets/_agent_list_render_agent.py`)

In `format_agent_option`:

- Replace the `running_monitors: int | None = None` keyword parameter with
  `monitor_lanes: MonitorLaneCounts | None = None`, mirroring how `clan_counts` is
  threaded.
- Compute `is_container_row` **first**, and only resolve the lane counts when the row is
  a container; otherwise use `NO_MONITOR_LANES`. This removes a per-row subtree
  traversal for the large majority of rows that can never render the badge, and — more
  importantly — stops descendant monitor churn from invalidating the render-cache entry
  of non-container rows (see Step 5). Before landing this gating, grep the row renderer
  for any other output on a non-container row that depends on descendant monitor state;
  there should be none (the starter row renders nothing about its monitor, and its `×N`
  fold annotation and runtime signature already enter the cache key separately). If such
  a dependency does exist, drop the gating and keep the traversal unconditional — the
  rest of the plan is unaffected.
- Emit the amber badge when `lanes.running`, then the grey badge when `lanes.settled`,
  each as `" "` + `f"{_MONITOR_GLYPH}{count}"` in its lane style, preserving the current
  position in the row (after the clan status chip, before the bead glyph and trailing
  identity block).

In `cached_format_agent_option`, compute the lanes once (gated on container-ness the
same way) and pass the same value into both `agent_render_key` and
`format_agent_option`, exactly as it does today for `running_monitors`.

### Step 5 — render cache key (`src/sase/ace/tui/widgets/_agent_list_render_cache.py`)

- Swap the `running_monitors: int | None` parameter of `agent_render_key` for
  `monitor_lanes: MonitorLaneCounts | None`, apply the same container gating and
  `monitor_lane_counts` fallback, and put the `MonitorLaneCounts` value in the key where
  `monitor_count` sits today.

This is load-bearing, not cosmetic: a monitor moving running → settled changes both
lanes, but a monitor row that is _loaded already settled_ changes only the settled lane.
Without the settled count in the key, that node's row would keep serving a stale cached
render.

### Step 6 — help legend (`src/sase/ace/tui/modals/help_modal/agents_bindings.py`)

The Agents help legend currently lists `("⚙N", "N running monitors in subtree")`. Split
it into the two lanes so the legend explains the colors, e.g. a running entry and a
finished entry. Keep the wording short enough for the legend column.

### Step 7 — docs

- `docs/ace.md`: the glyph table near the `⚙` / `⚙N` rows — replace the single `⚙N` row
  with the two lanes and name the colors. Then the paragraph that reads "a collapsed
  family shows the aggregate `⚙N` badge" must describe both badges and say that the two
  counts partition the subtree's monitors, with the unknown-state tie-break stated ("a
  monitor that has not reported a terminal state counts as running").
- `docs/monitors.md`: the "In the ACE TUI" paragraph that reads "A collapsed family or
  clan container row carries a `⚙N` badge for its running monitors" gets the same
  treatment, including that a failed, timed-out, or lost monitor counts in the finished
  lane.

Do not touch the top-bar indicator or Procs-tab sections of `docs/ace.md`.

### Step 8 — tests

`tests/ace/tui/models/test_agent_family_members.py` — update the four
`running_monitor_count` tests to `monitor_lane_counts` and add:

- the partition invariant: for a subtree mixing running, completed, stopped, failed,
  timeout, and lost monitors, `running + settled` equals the number of distinct monitor
  rows beneath the node, and the running lane counts only the running one;
- every terminal state (`completed`, `stopped`, `failed`, `timeout`, `lost`) lands in
  the settled lane;
- unknown/`None` state with no `stop_time` counts **running**, not settled;
- unknown/`None` state with a `stop_time` counts settled;
- `monitor_state == "running"` with a `stop_time` counts settled;
- clan aggregation at depth two counts both lanes across member families;
- the existing overlap-dedupe + cycle test asserts both lanes (a monitor attached to
  both `runtime_children` and `followup_agents` is counted once);
- a plain agent projects `MonitorLaneCounts()`.

`tests/ace/tui/widgets/test_agent_list_monitor_rows.py`:

- **`test_family_container_with_only_settled_monitors_renders_no_badge` asserts the
  behavior this feature intentionally changes** — rewrite it to assert the grey badge is
  present and no amber badge is, and rename it accordingly.
- Add: a family container with one running and three settled monitors renders `⚙1` then
  `⚙3` in that order, with the amber style on the first and
  `_MONITOR_SETTLED_COUNT_GLYPH_STYLE` on the second (assert via `left.spans`, not just
  `left.plain`, so a silent hue regression fails).
- Add: a clan container aggregates the settled lane across two member families.
- Add: a non-container row with settled monitors beneath it renders neither badge
  (mirror the existing parallel-family-root isolation trick, asserting the lane counts
  are nonzero so the test isolates the container gate itself).
- Keep `test_family_container_badge_does_not_alter_status_chip` passing and extend it to
  cover the settled badge coexisting with `[S1 R2]`.

`tests/ace/tui/widgets/test_agent_render_cache.py` (helpers in
`_agent_render_cache_helpers.py`): add a test that a container row's cached render is
invalidated when a monitor beneath it moves from running to settled, and another for a
monitor that arrives already settled (settled lane 0 → 1 with the running lane
unchanged) — the second is the case that fails if the settled count is left out of the
key.

### Step 9 — visual snapshot

Add a fixture and a PNG snapshot so the two-badge row is reviewed as pixels, not as
assertions about styles:

- Add a fixture to `tests/ace/tui/visual/_ace_agents_png_snapshot_fixtures.py` building
  a collapsed family container with one running monitor and three finished monitors (mix
  at least one failed one in, to prove failures read grey).
- Add a test to `tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py`
  following the shape of `test_waiting_family_child_row_png_snapshot`, with the
  container row **highlighted/selected** so the snapshot exercises the grey badge
  against the selection background — the hardest legibility case.
- Pin `now` with `pin_agents_visual_now` so elapsed text is deterministic.
- Generate the golden with `just test-visual --sase-update-visual-snapshots` and **look
  at the resulting PNG** before accepting it: confirm the grey reads as clearly quieter
  than the amber but is still legible, and that the two gears do not visually merge at
  120x40.

No existing PNG golden should change — there is no Agents-tab snapshot with monitors
today (`grep -l monitor tests/ace/tui/visual/_ace_agents_png_snapshot*.py` is empty). If
any existing golden does move, stop and investigate rather than accepting it.

## Verification

1. `just install` first — this workspace may be cold.
2. `just check` after the source and test edits.
3. `just test-visual` for the new snapshot (and to confirm no other golden moved).
4. `just check-full` through `/sase_monitor` before landing, since this touches a hot
   render path and the render-cache key.
5. Manual: launch `sase ace`, find a family that has run monitors, and confirm the
   collapsed row shows both badges, that the amber count drops and the grey count rises
   by one as a monitor finishes, and that the two counts always sum to the number of
   monitor rows revealed by pressing `l` on that node.

## Risks

- **Stale rows.** The single highest-risk item is Step 5. If the settled lane is left
  out of the render cache key, badges will be right on first paint and silently stale
  afterwards — the exact failure mode the cache key's docstring warns about ("adding a
  new visible field is a deliberate edit here rather than a silent cache desync"). The
  two cache tests in Step 8 are the guard.
- **Amber semantics shift.** The mid-launch (unknown-state) monitor now counts amber. It
  is transient and strictly better than counting nowhere, but it is a real behavior
  change to an existing badge and is called out here so it is reviewed rather than
  discovered.
- **Row width.** Two badges add at most ~8 cells to container rows. Clan rows are the
  widest rows in the list (tier gutter, status, bead, clan chip, name, tribes); the
  120x40 snapshot in Step 9 is what confirms this stays comfortable.
