---
tier: epic
title: Capacity inspector redesign — answer "can new work start, and who is using
  it?" at a glance
goal: 'Opening the capacity inspector (`,l`) answers three questions in under a second:
  can a new agent start right now, who is using the capacity, and who is waiting for
  it and why. A slot gauge, a plain-English verdict, tiered sections, and a detail
  strip replace the flat wall of dim text. The noise of inactive records is folded
  away. Every number still comes straight from the authoritative report, so nothing
  is estimated and nothing unknown ever reads as free.

  '
decisions:
  cli_parity:
    ask: Should `sase agent load` adopt the same verdict, gauge, and section order
      as the inspector?
    choices:
      adopt: CLI gets the verdict line, a text gauge, and the inspector's tiered sections
      minimal: CLI keeps today's layout; only its mislabeled idle section is split
        correctly
    default: adopt
    why: The view model is shared, and the CLI, chip, and inspector promise to agree
    answer: adopt
  inactive_records:
    ask: How should the inspector show inactive records (not live, done, not agent
      records)?
    choices:
      collapsed: One dim, collapsed row per reason under Not running; Enter expands
        it
      hidden: No rows; counts appear only in the Not running header; `y` JSON still
        has them
    default: collapsed
    why: Keeps the list one keypress away without letting 65 dead rows bury live ones
    answer: collapsed
phases:
- id: view-model
  title: 'Phase view-model: tiered capacity view, verdict, and gauge geometry'
  depends_on: []
  size: medium
  description: 'view-model: additively extend the shared stdlib-only capacity view
    with a verdict, limit provenance, tiered sections, queue tones, durations, and
    gauge cell geometry. Existing renderers keep their output.'
- id: inspector
  title: 'Phase inspector: redesigned TUI capacity inspector'
  depends_on:
  - view-model
  size: medium
  description: 'inspector: rebuild the modal around the gauge, verdict, tiered sections,
    and detail strip, with stable selection, h/l folding, a freshness ticker, and
    a verdict-first chip tooltip. Update goldens and TUI docs.'
- id: cli
  title: 'Phase cli: `sase agent load` parity and legacy view cleanup'
  depends_on:
  - inspector
  size: small
  description: 'cli: move `sase agent load` onto the tiered view per the cli_parity
    decision, delete legacy view fields with no remaining consumer, and update the
    runner-slots docs.'
proposed_by: bbugyi200.athena.14s
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 10:48:19
status: done
bead_id: sase-1ox
---

- **PROMPT:** [prompts/202610/capacity_inspector_redesign.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/capacity_inspector_redesign.md)
- **BEAD:** [sase-1ox](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ox/README.md)

# Plan: Capacity inspector redesign

## Why the current inspector is hard to read

This analysis uses the user's live screenshot (athena, 5 of 8 units) and the current
code (`src/sase/ace/tui/modals/capacity_inspector_modal.py`,
`src/sase/core/runner_slots/_capacity_view.py`):

1. **The main question goes unanswered.** "Can my next agent start?" is only implied by
   `3 headroom` at the end of the first of four equally dim lines of text.
2. **The live section's heading is wrong.** "LIVE, HOLDING NOTHING" also gathers records
   that Rust accounting classifies as `not_live` (dead process, no done marker), `done`,
   and `not_agent_record` (workflow steps, procs, hidden records). In the screenshot, 65
   of the 83 rows under that heading are not live at all. Accounting entries in state
   `unknown` with no reason code also fall through to `not started`.
3. **Noise buries the signal.** About 100 rows follow the holders, so the most
   actionable section, **Queued for capacity**, sits below the fold, after the idle
   lists.
4. **"Holding capacity" mixes charged and zero-weight claims.** It shows 7 rows for 5
   charged units: wait monitors at `+0` sit beside workers at `+1`. The list is sorted
   alphabetically, not by who uses the most or has held capacity longest.
5. **Rows are hard to scan.** Names reach 48 characters
   (`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.land`), times are raw ISO timestamps,
   and `default`/`explicit_zero` is internal jargon.
6. **Freshness is opaque.** `stale (token changed)` and `18s old` describe the scan at
   compute time. Nothing tells you whether the inspector itself is still receiving
   reports.
7. **The layout is framed twice.** The list sits inside its own border, inside a
   `VerticalScroll` with top and bottom rules, inside the modal's double border. The
   OptionList is `height: auto` within the scroller, so the scroller, not the list, owns
   scrolling.
8. **Selection resets.** Each new report (at least every 10 s) clears and re-adds the
   options with counter-based ids, so the highlight jumps back to the top while you
   browse.

## Design principles

- **Answer first, explain second.** The top of the modal is a gauge plus one sentence.
  Details live one keypress away in the detail strip.
- **Truth over comfort.** Every number comes straight from `MachineCapacityReport`. The
  verdict restates facts the report proves, such as free units and the queue head; it
  never predicts admission order or start times. Unknown accounting renders as unknown
  (`╱` cells, `? free`), never as free. The loading state shows no numbers.
- **One visual language with the rest of the TUI.** The gauge fill uses the same
  usage-window color gradient as the `⌂athena cap 5/8` chip
  (`_usage_indicator_palette.usage_percent_color`, keyed on free percent), so a color
  means the same headroom in both places. Occupancy marks reuse the Agents list's `▰N`
  and `▱?` glyphs (`docs/reference/tui/agent-row-glyphs.md`). Over budget uses the
  chip's `▲`.
- **Calm by default.** Inactive records stay folded away, empty optional sections are
  hidden, and only problems use strong color.

## The new inspector

The target layout is roughly 100 columns wide. Here the busy fixture has a 2-unit holder
highlighted, so its two gauge slots are drawn in the accent color.

```
╔═══════════════════════════════ CAPACITY · athena ══════════════════════════════════╗
║  8 / 8 units in use                                      0 free · updated 3s ago   ║
║  ████████ ████████ ████████ ████████ ████████ ████████ ████████ ████████           ║
║  ■ Full — all 8 units in use                                                       ║
║    2 queued · next up #1 sase-1nf.15: needs 1 · 0 free · budget 8                  ║
║  limit 8 · ACE override for 44d ⚠ (config 10)     9 running · 7 charged · 2 at 0   ║
║                                                                                    ║
║  USING CAPACITY ─────────────────────────────────────────── 8 units · 7 agents     ║
║  ▰2  sase-1oa.4                      worker          explicit weight     3h12m     ║
║  ▰1  sase-1nf.14.1.2                 worker                              2h01m     ║
║  ▰1  bob-cli-an.6.6.6.5…6.6.6.land   worker                              50m       ║
║  ▰1  sase-1lou.5.1.3                 verify monitor  inherited · +1 rider 18m      ║
║  QUEUED ────────────────────────────────────────────────────────── 2 waiting       ║
║  #1  sase-1nf.15                     requests 1      needs 1 · 0 free    4m        ║
║  #2  sase-1oc.2                      requests 1      held by sase-1ob    1m        ║
║  RUNNING AT ZERO WEIGHT ─────────────────────────────────────────── 2 agents       ║
║  ▰0  sase-1n6.10--mon                wait monitor                        1h15m     ║
║  ⇢   sase-1lou.2                     rider           shares sase-1lou.5.1.3's claim║
║  NOT RUNNING ──────────────────────────────── holds nothing · 18 live · 65 inactive║
║  ▸ 18 waiting to start     sase-1n6.land, sase-1nf.10.1.land, sase-1nf.11, …       ║
║  ▸ 31 not live             05h, 05i, 20260914090802, …                             ║
║  ▸ 34 not agent records    20260812125020, …                                       ║
║ ────────────────────────────────────────────────────────────────────────────────── ║
║  sase-1oa.4 · worker · sase                                                        ║
║  charges 2 units (explicit weight) · running 3h12m, since 2026-10-27 11:20 UTC     ║
║  claim held alone · Enter jumps to this agent                                      ║
║        j/k move · Enter jump / expand · h/l fold · y copy JSON · Esc close         ║
╚════════════════════════════════════════════════════════════════════════════════════╝
```

### 1. The header: headline, gauge, verdict, facts

- **Title** (in the border or the first line): `CAPACITY · <machine>`.
- **Headline row.** On the left, `<occupied> / <limit> units in use`: occupied in bold
  fill color, limit in bold. On the right, `<free> free`. Over budget, this reads
  `<n> over` in red. With invalid accounting it reads `? free`. A freshness label
  follows:
  - `updated 3s ago` (dim): live age, computed from an injectable clock minus
    `report.computed_at`, ticked by a 1 s timer.
  - `⟳ updated 41s ago` (yellow) once the age passes the chip's stale threshold (25 s).
  - `⟳ records changed · rescanning` (yellow) when `report.scan_current` is false.
- **Gauge** (one row, full content width). The view model computes its geometry and the
  renderer only styles it:
  - _Slot mode_ (up to 24 units, and at least 2 cells per slot): one slot per unit of
    the limit, separated by single spaces.
  - _Continuous mode_ otherwise: a proportional bar.
  - `█` is used capacity, in the fill color. `░` is free (dim). `╱` is unknown, used
    when accounting is invalid: the known subtotal is filled and the rest is unknown,
    never free.
  - Over budget, the bar spans the occupied units: a bold `┃` limit marker replaces the
    gap at the limit, and the overflow slots are red.
  - Each used cell knows which charged row owns it, from cumulative weights in display
    order. The highlighted holder's cells render in the theme accent color, so moving
    through **Using capacity** visibly lights up that agent's share of the machine.
- **Verdict.** A tone glyph and a title, plus an optional dim detail line. The tone sets
  the color: ok `✓` green, tight `◔` yellow, full `■` orange, over `▲` red, blocked `!`
  red, unknown `?` yellow. Exact rules and strings are in the view-model phase.
- **Facts row.** On the left, limit provenance:
  `limit 8 · ACE override for 44d (config 10)`. An override active for more than 14 days
  gets a yellow `⚠`; this is the chip's existing `_OVERRIDE_WARNING_AFTER_SECONDS` rule,
  and it surfaces the user's forgotten 44-day override. Without an override it reads
  `limit 10 · from config`. On the right, `9 running · 7 charged · 2 at 0`; the running
  count is omitted when `running_nodes` is `None`.

### 2. Sections and rows

Sections appear in this order. Each header is a bold `#87D7FF` label, a dim `─` rule
filling the width, and a dim right-aligned summary.

1. **NEEDS ATTENTION** appears only when non-empty and is always expanded. It holds
   accounting entries in state `unknown` (`no_pid`, `invalid_weight`, unknown rider
   holder) as `▱?` rows in yellow, then each report diagnostic as a non-selectable row.
   When accounting is invalid, this section explains why admission is blocked.
2. **USING CAPACITY** is always shown. It holds claims with `charged_weight > 0`, sorted
   by weight (descending), then running time (longest first), then name. The empty state
   reads `Nothing holds capacity — all <limit> units are free`.
3. **QUEUED** is always shown. Waiters appear in queue order, with the blocker colored
   by tone: ready green, capacity yellow, held magenta, never-fits red, deferred dim.
   The empty state reads `Nobody is queued`.
4. **RUNNING AT ZERO WEIGHT** appears only when non-empty. It holds zero-weight claims
   (wait monitors, explicit `w=0`) as `▰0` rows, and serial riders as `⇢` rows with
   `shares <holder>'s claim`.
5. **NOT RUNNING** appears only when non-empty, with the header summary
   `holds nothing · N live · M inactive`. Each tier group is one collapsible row:
   `▸ <count> <label>`, followed by a dim preview of the first names that fit.
   - Live waiting groups (`waiting to start`, `waiting on a question`,
     `waiting on a gate`, `zero weight`) come first, in normal color.
   - Inactive groups (`not live`, `done`, `not agent records`) follow in dim color.
   - Enter, `l`, or `→` expands a group to its full, indented member list. `h` or `←`
     collapses it; on a member row, `h` collapses the parent group and moves the
     highlight to it.

> [!decision] inactive_records = hidden Inactive tier groups render no rows. The NOT
> RUNNING header still reports `M inactive`. If no live waiting groups exist either, the
> section renders a single non-selectable dim row:
> `M inactive records hidden · y copies the full report`.

**Row grid.** Rows use a column layout computed from the OptionList's content width and
rebuilt on resize:

- Width ≥ 96: a 4-cell mark (`▰1`, `#1`, `⇢`, `▱?`); then the name; then the kind (16
  cells); then a note (dim tags such as `inherited`, `explicit weight`, `budget 2`,
  `+1 rider`, or the blocker for queued rows); then a right-aligned age (running time
  for holders, waiting time for queued rows).
- The name is middle-truncated with `…` so the head and tail stay readable
  (`bob-cli-an.6.6.6.5…6.6.6.land`), capped at 40 cells.
- Width < 96: the kind column is dropped (it remains in the detail strip), and names cap
  at 28 cells.
- Ages use minute granularity, so values do not flicker between 10 s report refreshes:
  `45s`, `4m`, `1h06m`, `3d04h`.

### 3. The detail strip

The detail strip is three lines with a top rule, above the footer. It always describes
the highlighted row in full, so rows can stay terse. It is pure in-memory text,
recomputed on `OptionHighlighted`, with no I/O.

- **Holder:** the full name, kind, and project label; then
  `charges N unit(s) (<weight-source sentence>) · running <dur>, since <YYYY-MM-DD HH:MM UTC>`;
  then `shared with <riders>` or `claim held alone`, plus
  `launched with %queue(capacity=N)` when budgeted, plus `Enter jumps to this agent`.
- **Rider:**
  `rides the claim charged through <holder>; serial session members share one claim`.
- **Queued:** `queue #N · requests W · waiting <dur>`, then every blocker in full.
- **Group or member row:** the tier explanation sentence from the view model, then the
  Enter hint (expand, collapse, or jump).

Weight-source sentences:

| Source          | Sentence                               |
| --------------- | -------------------------------------- |
| `default`       | `default weight`                       |
| `inherited`     | `weight inherited from its starter`    |
| `explicit`      | `explicit weight`                      |
| `explicit_zero` | `explicit zero weight — never charges` |
| `invalid`       | `invalid weight`                       |

Project labels come from the memory-only
`sase.project_display_names.project_display_name_for(project_key)`, which is the same
sanctioned lookup `Agent.project_label` uses. Never derive a label from the key format
or a path (decision `project-labels-never-keys`).

### 4. Interaction

| Key                                 | Action                                                                           |
| ----------------------------------- | -------------------------------------------------------------------------------- |
| `j`/`k`, `↑`/`↓`, `ctrl+n`/`ctrl+p` | Move (existing)                                                                  |
| Enter                               | Jump to the agent row, or toggle a group                                         |
| `l`/`→`                             | Expand a group                                                                   |
| `h`/`←`                             | Collapse a group (from a member row: collapse the parent group and highlight it) |
| `y`                                 | Copy the exact `sase agent load -j` JSON (unchanged)                             |
| Esc / `q`                           | Close                                                                            |

- Option ids are stable identities instead of counters: `agent:<artifact_dir>` for
  holders, riders, queued, and attention rows; `group:<reason_code>` for group rows;
  `member:<reason_code>:<artifact_dir>` for member rows.
- On each report refresh, re-select the previously highlighted id. If that row is gone,
  select the row now at the same index within its section, falling back to the first
  selectable row. Expanded groups survive refreshes.
- The initial highlight is the first selectable row.
- Drop the `VerticalScroll` wrapper. The OptionList itself is `height: 1fr` and owns
  scrolling, so `scroll_to_highlight` always keeps the cursor visible.
- No extra framing: no inner border around the list. The modal keeps its single double
  border.

### 5. States

| State                      | What the inspector shows                                                                                                                                                                                                   |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Loading (`report is None`) | A dim `░` placeholder gauge; the verdict reads `Waiting for the first authoritative report…`, then a dim `The inspector never estimates capacity from visible rows.`; the list holds one disabled row; no numbers anywhere |
| Accounting unavailable     | The gauge still shows the totals; verdict `unknown`; the sections collapse to one notice: `This sase-core build reports totals only.`                                                                                      |
| Accounting invalid         | Gauge: subtotal plus `╱`; `? free`; verdict `blocked`; NEEDS ATTENTION is populated                                                                                                                                        |
| Over budget                | Overflow slots in red past `┃`; `<n> over`; verdict `over` with the per-launch budget detail                                                                                                                               |
| Nothing holds capacity     | Gauge all `░`; verdict `ok`; empty-state rows                                                                                                                                                                              |

## Guardrails

- **Rust boundary** (`rust_core_backend_boundary`, decision `rust-core-required`). This
  epic adds no capacity arithmetic. Admission, accounting, waiters, and blockers stay in
  sase-core and arrive through `MachineCapacityReport`. `_capacity_view.py` is already
  the shared, stdlib-only presentation projection consumed by the CLI, the chip, and the
  inspector. The verdict, tiers, and gauge geometry are presentation over report fields:
  free = limit − occupied, queue head = waiters[0]. Mapping reason codes to tiers is
  labeling, not admission logic. No sase-core change and no `sase-core-revision.txt`
  move are needed. If a future non-Python frontend needs the same tiers, that is the
  trigger to move this projection into sase-core.
- **TUI performance** (`tui_perf.md`). Rendering is pure and in-memory: no stat, read,
  subprocess, or project-record load in compose, render, highlight, or tick paths. The 1
  s freshness timer is a thin synchronous callback that updates a single Static. A
  report arrival rebuilds options once; highlight changes re-render only the gauge and
  the detail strip.
- **Determinism.** Row ages are relative to `report.computed_at`; only the freshness
  label uses the live clock, which the modal takes as a `clock: Callable[[], float]`
  seam (default `time.time`). Displayed times are UTC. Visual tests pin the clock, the
  agents visual "now", and TZ.
- **File size.** `toobig` limits are 700, 850, and 1000 lines. Keep every new or
  modified module under 700 lines by splitting rendering out of the screen and gauge
  geometry out of the projection.

## Phase view-model: tiered capacity view, verdict, and gauge geometry

Additively extend `sase.core.runner_slots._capacity_view`. Keep it stdlib-only; split it
into `_capacity_view.py` (projection entry point and dataclasses),
`_capacity_view_rows.py` (row and tier builders), and `_capacity_gauge.py` (geometry) as
needed to stay under 700 lines. Re-export the new public names lazily through
`sase/core/runner_slots/__init__.py`, the way the existing names are. **Do not change
any existing field's value or order.** The current modal, CLI, and chip must render
byte-for-byte the same, so the `capacity_inspector_120x40` golden must not move. The one
exception is a bug fix: an accounting entry in state `unknown` with no reason code gets
reason code `unknown_holder` (label `claim holder unknown`) instead of falling through
to `not_started`.

**New types and fields.**

- `CapacityTone = Literal["ok", "tight", "full", "over", "blocked", "unknown"]`
- `CapacityVerdict(tone, title, detail: str | None)`
- `CapacityLimitInfo(effective, configured, override_source, override_created_at, override_expires_at, override_age_seconds, override_long_lived)`.
  Here `override_source` uses the existing `_override_source()` uppercasing and is
  `None` without an active override.
- Public constants `CAPACITY_REPORT_STALE_AFTER_SECONDS = 25.0` and
  `CAPACITY_OVERRIDE_WARNING_AFTER_SECONDS = 14 * 24 * 3600`. Switch
  `src/sase/ace/tui/widgets/agent_load_indicator.py` to import them in place of its
  private copies; this gives them a non-test consumer now.
- `format_capacity_duration(seconds: float | None) -> str`: `—` for `None`; `45s` under
  1 m; `4m` under 1 h; `1h06m` under 1 d; `3d04h` otherwise. Negative values clamp to 0.
- `CapacityHoldingRow` gets `project_key: str = ""` (the accounting `project_name`),
  `running_seconds: float | None = None` (`computed_at` − parsed `run_started_at`;
  `None` when unparsable), and `rider_names: tuple[str, ...] = ()`.
- `CapacityRiderRow` gets `holder_artifact_dir`, `project_key`, and `running_seconds`,
  all with defaults.
- `CapacityQueuedRow` gets the following, all with defaults:
  - `project_key`.
  - `waiting_seconds`: from `slot_requested_at`.
  - `tone`: `ready` | `capacity` | `held` | `deferred` | `never_fits` | `other`. It
    comes from the first blocker code (`insufficient-capacity` → capacity,
    `hold-barrier` → held, `deference-window` → deferred, `weight-exceeds-limit` →
    never_fits); with no blockers, eligible → ready, otherwise capacity.
  - `blockers: tuple[str, ...]`: one `_one_blocker` text per blocker.
- `CapacityIdleGroup` gets
  `tier: Literal["attention", "waiting", "inactive"] = "waiting"` and
  `explanation: str = ""`.
- `MachineCapacityView` gets:
  - `occupied`, `limit`, `free` (floats; `free` may be negative), `verdict`,
    `limit_info`, `computed_at`, `scan_age_seconds`, `scan_current`.
  - `charged` and `zero_weight`: holdings split on `charged_weight > 0`, both in the new
    sort order (weight descending, then running time descending with `None` last, then
    name).
  - `attention_groups`, `waiting_groups`, `inactive_groups`: tuples of
    `CapacityIdleGroup` with **complete** membership (`truncated=False`, `hidden=0`;
    renderers own previews and folding).
- `capacity_gauge_cells(view, width) -> tuple[CapacityGaugeCell, ...]`. Each
  `CapacityGaugeCell` is `(char, kind, owner)` with kind in
  `used | over | free | unknown | gap | limit | beyond`, and `owner` is an index into
  `view.charged` or `None`.

**Tier mapping** for non-holding, non-riding, non-waiting accounting entries:

- **attention:** state `unknown`. That covers reason `no_pid` (label
  `no process evidence`), `invalid_weight`, and the new `unknown_holder`.
- **waiting:** `not_started` (label `waiting to start`), `pending_question`,
  `pending_gate`, `explicit_zero_weight` (label `zero weight`). Any **unrecognized**
  reason code goes here, so a new code is visible and never silently hidden. Its label
  is the code with spaces, and its explanation is `Holds no capacity (reason: <code>).`
- **inactive:** `not_live` (`not live`), `done`, `not_agent_record`
  (`not agent records`).

Within each tier, groups keep `_REASON_ORDER`, and members sort by name, then path.

**Explanations.** These match the Rust accounting in
`sase-core/crates/sase_core/src/runner_capacity/accounting.rs` and `records.rs`:

| Reason                 | Explanation                                                                                                       |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `not_started`          | `Live, but its run has not begun (waiting on dependencies, a schedule, or launch). Holds nothing until admitted.` |
| `pending_question`     | `Paused on a question while a human decides. Holds nothing; the follow-up reacquires capacity when answered.`     |
| `pending_gate`         | `Paused on a gate while a human decides. Holds nothing; the follow-up reacquires capacity when decided.`          |
| `explicit_zero_weight` | `Launched with an explicit weight of 0, so it never charges capacity.`                                            |
| `not_live`             | `No live process and no done marker (killed, crashed, or stranded). Never holds capacity.`                        |
| `done`                 | `Finished. Done records never hold capacity.`                                                                     |
| `not_agent_record`     | `Not an agent run (workflow step, proc, or hidden record). Only agent runs charge capacity.`                      |
| `no_pid`               | `A monitor member with no recorded process. Its charge is unknown — never counted as free.`                       |
| `invalid_weight`       | `Its queue weight cannot be parsed, so its claim is dropped and accounting is invalid.`                           |
| `unknown_holder`       | `Shares a claim whose holder could not be determined. Its charge is unknown — never counted as free.`             |

**Verdict rules.** The first match wins. Numbers use the existing `_fmt`. Pluralize
`unit`/`units` and `agent`/`agents`. The queue summary used below is:
`<n> queued · next up #<pos> <name>: <blocker of the lowest queue_position>`.

| #   | Condition                  | Tone    | Title                                                                   | Detail                                                                                                                                                |
| --- | -------------------------- | ------- | ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `not accounting_available` | unknown | `Holder accounting unavailable — totals only`                           | `This sase-core build cannot explain individual holders.`                                                                                             |
| 2   | `not accounting_valid`     | blocked | `Admission blocked — capacity accounting is invalid`                    | `<occupied> is a known subtotal, not free capacity · see Needs attention`                                                                             |
| 3   | `over_limit`               | over    | `Over budget by <over> — no new work starts until load drops`           | The per-launch-budget clause of today's `_over_limit_note`                                                                                            |
| 4   | `free ≤ 1e-9`              | full    | `Full — all <limit> units in use`                                       | The queue summary if anything is queued, else `The next launch will queue until a unit frees up.`                                                     |
| 5   | `free < 1`                 | tight   | `<free> unit free — a default-weight agent would queue`                 | The queue summary or `None`                                                                                                                           |
| 6   | otherwise                  | ok      | `<free> units free — room for <floor(free)> more default-weight agents` | `<n> still queued · #<pos> <name>: <blocker>` when anything is queued, which explains holds, deference, and never-fits despite free room; else `None` |

**Gauge geometry.** All rules below are pure and deterministic:

- `span = max(limit, occupied)` and `units = ceil(span - 1e-9)`.
- Return `()` when `width < 8`, `limit <= 0`, or a value is non-finite.
- **Slot mode** applies when `units <= 24` and `(width - (units - 1)) // units >= 2`.
  Each unit is a slot of `slot_w` cells with one `gap` cell between slots. When over the
  limit, the gap after slot `ceil(limit) - 1` becomes a `limit` cell (`┃`). Leftover
  width becomes trailing `beyond` cells.
- **Continuous mode** uses `width` cells, or `width - 1` plus one `limit` cell inserted
  at `round(limit / span * cells)` when over the limit.
- A cell's unit position is its midpoint `p`:
  - `p ≤ occupied` and `p > limit` → `over` (`█`).
  - `p ≤ occupied` → `used` (`█`).
  - Invalid accounting and `p ≤ limit` → `unknown` (`╱`).
  - `p ≤ limit` → `free` (`░`).
  - Otherwise → `beyond` (a space).
- `owner` is the index `i` with `cum[i] < p ≤ cum[i+1]` over `view.charged` weights; it
  is `None` past the cumulative sum or when accounting is invalid.

**Symvision.** New public symbols consumed only by later phases (for example
`capacity_gauge_cells`, `format_capacity_duration`, `CapacityVerdict`) need
`--epic-symbol` entries. Read `sase/memory/symvision.md` with `/sase_memory_read` and
follow its hierarchy; the inspector and cli phases remove the entries once they consume
the symbols.

**Tests.** Use a new `tests/test_capacity_view.py`, building reports with
`tests/_runner_slots_helpers._record` and `machine_capacity_report_from_records`
(`is_live` callables and `active_holds` cover not-live and held cases). Cover:

- Every verdict branch and its exact strings, including pluralization and fractional
  free capacity.
- Tier assignment for every reason code, the unrecognized-code fallback, and the
  `unknown_holder` fix.
- Charged versus zero-weight split and sort order; riders attached to holders.
- Queue tones and blocker tuples, durations, and limit info: override, config only, an
  expired override, and a long-lived override.
- Gauge cells in slot and continuous modes, over limit with the marker, invalid
  accounting producing unknown cells and no owners, owner ranges for weights 2 and 0.5,
  and widths below 8.
- The unchanged-legacy contract: existing `tests/test_agent_load_cli.py` and
  `tests/ace/tui/modals/test_capacity_inspector_modal.py` pass untouched, apart from the
  `unknown_holder` label.

Run `sase tool run check`.

## Phase inspector: redesigned TUI capacity inspector

Rebuild `CapacityInspectorModal` to the design above, reading only the new view fields.
Read `sase/memory/tui.md`, `tui_perf.md`, `tui_screenshot.md`, and `docs_writing.md`
with `/sase_memory_read` first.

**Files:**

- `src/sase/ace/tui/modals/capacity_inspector_modal.py`: the screen. It owns compose,
  bindings, report install, stable selection, fold state, the `clock` seam, the 1 s
  freshness timer, and detail and gauge updates on highlight. Keep the public surface
  (`CapacityInspectorModal(report, *, running_nodes, clock=None)`,
  `update_capacity_report`, `CapacityInspectorSelection`, `report`, `running_nodes`) so
  `actions/axe_service.py` and `actions/event_refresh/_auto_refresh.py` need no changes.
- `src/sase/ace/tui/modals/_capacity_inspector_render.py`: pure functions from view (and
  width, dark flag, and highlighted owner) to Rich `Text`. It renders the headline,
  gauge runs (grouping consecutive cells by kind and highlight into spans), verdict,
  facts, section headers with rules, rows for each layout breakpoint, collapsed-group
  previews, detail-strip text, and the middle-truncation helper.
- A small `CapacityGauge` widget whose `render()` uses `self.size.width`, so it reflows
  on resize.
- `src/sase/ace/tui/styles.tcss`, `CapacityInspectorModal` block: keep the container
  (96% / max 150 by 90% / max 52, `border: double $primary`). Add header widgets
  (`height: auto`); `#capacity-inspector-gauge` (height 1); the OptionList at
  `height: 1fr` with no border; `#capacity-inspector-detail` (height 3,
  `border-top: solid $primary-darken-2` or the nearest existing token); and the footer.
  Remove the scroll wrapper rules.

**Implementation notes:**

- Gauge fill: `usage_percent_color(free_percent, dark=…)`, where `free_percent` follows
  the chip's `_free_capacity_percent`. Over and invalid use the chip's rejected/red
  style. Free uses dim neutral. The highlighted owner uses the theme accent. Get `dark`
  the way `AgentLoadIndicator._current_dark_theme` does.
- Footer: `j/k move · Enter jump / expand · h/l fold · y copy JSON · Esc close`. Render
  keys bold and the text muted, matching other modals.
- Chip tooltip (`src/sase/ace/tui/widgets/agent_load_indicator.py`
  `_agent_load_tooltip`): put the verdict title on the line right after the headline,
  and build the `Held by:` block from `view.charged` plus a `+N at zero weight` line.
  Keep the `,l opens the capacity inspector` hint.
- Remove this phase's now-consumed `--epic-symbol` entries.

> [!decision] inactive_records = hidden Skip inactive group rows and their member and
> fold handling; render only the header count (and the single dim notice row when no
> live groups exist).

**Tests:**

- Update `tests/ace/tui/modals/test_capacity_inspector_modal.py`:
  - The holder row jumps.
  - Enter, `l`, and `h` fold groups, including `h` from a member row returning to its
    group.
  - The highlight survives `update_capacity_report`, both when the row still exists and
    when it vanishes.
  - The detail strip follows the highlight.
  - The freshness label crosses the stale threshold with a fake clock and shows
    `rescanning` when `scan_current` is false.
  - The loading state contains no digits in the gauge or verdict.
  - Copy still matches the CLI JSON.
  - Invalid and unavailable accounting render their verdicts.
- Add `tests/ace/tui/modals/test_capacity_inspector_render.py` for the pure renderers:
  middle truncation, the two layout breakpoints, gauge highlight spans, and section
  header summaries.
- Keep `tests/ace/tui/test_capacity_inspector_openers.py` and
  `tests/ace/tui/widgets/test_agent_load_indicator.py` green; update tooltip
  expectations.

**Visual goldens** (`tests/ace/tui/visual/test_ace_png_snapshots_capacity_inspector.py`,
with fixtures in a new `tests/ace/tui/visual/_capacity_inspector_fixtures.py`). Pin the
modal clock to `computed_at + 3`.

| Golden                                    | Fixture                                                                                                                                                                                                                                                                                                                                                                  |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `capacity_inspector_120x40` (regenerated) | The existing `_screenshot_records()` report: the ok tone                                                                                                                                                                                                                                                                                                                 |
| `capacity_inspector_busy_120x40`          | Full at 8 of 8: one weight-2 explicit holder, default workers with staggered start times, a verify monitor with a rider, a wait monitor, two queued (one capacity-blocked, one held via `active_holds`), 3 not started, 4 not live, 5 not agent records, and a long-lived ACE override. Press `j` so the weight-2 holder is highlighted and its gauge cells are accented |
| `capacity_inspector_over_limit_120x40`    | Over budget, with a per-launch budget                                                                                                                                                                                                                                                                                                                                    |
| `capacity_inspector_blocked_120x40`       | An invalid weight, so NEEDS ATTENTION and the blocked verdict show                                                                                                                                                                                                                                                                                                       |
| `capacity_inspector_busy_80x32`           | The narrow breakpoint                                                                                                                                                                                                                                                                                                                                                    |

Run
`just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_capacity_inspector.py`
through `/sase_monitor`. Inspect every created and updated golden image. Prove host
independence with the targeted `--check` run twice, once with `TZ=UTC`, as
`tui_screenshot.md` requires.

**Live check.** Run
`sase screenshot -o /tmp/capacity_inspector.png -p , -p l -w CAPACITY`, inspect the PNG
against real host data, and confirm that the noisy groups are folded and the queue is
visible above the fold.

**Docs.** Rewrite `### Capacity inspector` in
`docs/reference/tui/agents-tab-grouping.md` to cover the gauge, the verdict tones, the
sections and their order, folding keys, the detail strip, freshness, and copy. Check the
help modal entry (`src/sase/ace/tui/modals/help_modal/agents_mode_sections.py`); `,l` is
unchanged, so update only its description if the wording drifts.

Run `sase tool run check`.

## Phase cli: `sase agent load` parity and legacy view cleanup

> [!decision] cli_parity = adopt Re-render `src/sase/agents/cli_load.py` with Rich in
> the inspector's order:
>
> - The headline `<machine> · <occupied> / <limit> units in use · <free> free`.
> - A text gauge from `capacity_gauge_cells(view, min(console.width - 2, 64))`, styled
>   with the same kinds (no highlight).
> - The verdict line with its glyph and the detail line.
> - The facts line: limit provenance and `report <UTC time> · scan <age> old`, plus
>   `· records changed since scan` when not current.
> - Then the sections:
>   - Needs attention (if any).
>   - A `Using capacity (<units> units · <n> agents)` table: UNITS, NAME, KIND, RUNNING,
>     NOTE.
>   - A `Queued for capacity (<n>)` table: #, NAME, REQUESTS, WAITING, BLOCKER (tone
>     colored).
>   - `Running at zero weight` lines.
>   - One `Not running` line per live waiting group, previewing 10 names then `+N more`.
>   - One dim line:
>     `Ignored <M> inactive records: <a> not live · <b> not agent records · <c> done (sase agent load -j lists them)`.
>   - Diagnostics.
>   - The existing footer.

> [!decision] cli_parity = minimal Keep today's CLI layout, but build its idle panel
> from `waiting_groups` under the heading `Live, holding nothing` and print inactive
> groups as the single dim `Ignored <M> inactive records…` line. This fixes the mislabel
> without restyling.

Then, in both cases:

1. Delete every legacy `MachineCapacityView` field, `capacity_view` parameter, and
   `CapacityIdleGroup` field that no longer has a non-test consumer. Candidates are
   `idle_groups`, `expanded_idle_groups`, `truncated`, `hidden`, `holdings`, `headline`,
   `limit_line`, `snapshot_line`, `running_line`, and `over_limit_note`; grep `src/`
   first and keep anything still read.
2. Delete each removed field's private helpers and tests.
3. Remove the remaining `--epic-symbol` entries.

Update `tests/test_agent_load_cli.py` to the new output: sections in order, the verdict
present, inactive records counted and not listed, and JSON unchanged.

Docs: first load the docs-writing reference note with `/sase_memory_read` (read only).
Then revise `docs/reference/runner-slots.md` › _What holds capacity_ to describe the
verdict and the tiered sections, and `docs/reference/cli/agent.md` if it describes the
`load` output. No CLI flags change.

Run `sase tool run check`.
