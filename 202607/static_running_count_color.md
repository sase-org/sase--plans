---
tier: tale
title: Keep the Agents-tab running count one static color
goal: The running-agent count in the Agents-tab capacity chip always renders in the
  same green style, leaving the runner-limit number as the sole capacity-pressure
  color signal.
create_time: 2026-07-25 07:43:08
status: done
---

- **PROMPT:** [202607/prompts/static_running_count_color.md](prompts/static_running_count_color.md)
- **AGENTS:**
  - [bbugyi200.athena.kc](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.kc/README.md)
  - [bbugyi200.athena.kc--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.kc.md#member-code)
- **COMMITS:**
  - [1892887](https://github.com/sase-org/sase/commit/18928870295811986d816146b9af5f4679ed91ca) — fix(ace): keep running count color stable

# Plan

## Problem

The Agents tab header renders an always-visible capacity chip, e.g. `[9/10 running · 7 waiting · 59 done]`.

Today **both** numbers in `R/L` encode capacity pressure:

- `R` (the running count) is green below the limit, gold **at** the limit, and red **above** it.
- `L` (the effective `max_running_agents` limit) escalates dim → gold (≥50%) → orange (≥75%) → red (≥100%) as `R` climbs
  toward and past it.

That makes `R`'s color redundant: the same "we are at/over capacity" information is already carried by `L`. The user
wants `R` to keep one static color at all times, and wants `L` to remain the pressure indicator.

## Current implementation

All of the affected rendering lives in `src/sase/ace/tui/widgets/agent_info_panel.py`.

`AgentInfoPanel._append_status_strip()` (~lines 318-348) builds the chip and currently picks the running-count style
with an inline three-way branch:

```python
text.append("  [", style="dim")
if self._runner_limit > 0 and self._running_count > self._runner_limit:
    running_style = "bold #FF5F5F"
elif self._runner_limit > 0 and self._running_count == self._runner_limit:
    running_style = "bold #FFD700"
else:
    running_style = "bold #00D7AF"
text.append(str(self._running_count), style=running_style)
text.append("/", style="dim")
text.append(str(self._runner_limit), style=self._runner_limit_style())
text.append(" running", style="dim")
```

Two further facts matter:

- `AgentInfoPanel._runner_limit_style()` (~lines 304-316) is the limit's escalation ladder. It **stays exactly as is** —
  it is the signal we are keeping.
- `AgentInfoPanel._COUNT_STYLES` (~lines 279-287) already holds `"running": "bold #00D7AF"`, which is byte-identical to
  the neutral branch above. That entry is currently unreferenced, because `_metric_counts()` deliberately omits
  `running` (the running count is rendered by the capacity chip instead of the metric strip). Reusing it removes the
  duplicated literal and makes the dict entry live again.

## Changes

### 1. `src/sase/ace/tui/widgets/agent_info_panel.py`

In `_append_status_strip()`, delete the three-way `running_style` branch and render the running count with the shared
constant:

```python
text.append("  [", style="dim")
# The runner-limit style already escalates with capacity pressure, so the
# running count itself stays a single stable color at every occupancy.
text.append(str(self._running_count), style=self._COUNT_STYLES["running"])
text.append("/", style="dim")
text.append(str(self._runner_limit), style=self._runner_limit_style())
text.append(" running", style="dim")
```

Notes for the implementer:

- Do **not** touch `_runner_limit_style()`, `_NEUTRAL_RUNNER_LIMIT_STYLE`, the `"/"` separator style (`dim`), the
  `" running"` label style (`dim`), the queued segment, or the metric strip.
- Do **not** add `("running", self._running_count)` to `_metric_counts()`; that would double-render the running count.
- Keep the resulting expression a plain `str` style so the surrounding `text.append(...)` calls stay type-consistent
  (`_runner_limit_style()` returns `str | Style`; the running count needs only `str`).
- After the branch is gone, `self._runner_limit` is still read for the `L` segment, so no attribute becomes unused.

### 2. `tests/ace/tui/widgets/test_agent_info_panel.py`

`test_status_strip_styles_running_capacity_and_positive_queue()` (~lines 257-321) is the test that pins the old
behavior. Its case table currently asserts a pressure-dependent running style:

```python
(10, 10, "bold #FFD700", "bold #FF5F5F"),
(12, 10, "bold #FF5F5F", "bold #FF5F5F"),
...
(7, 7, "bold #FFD700", "bold #FF5F5F"),
```

Update all three of those rows so the expected running style is `"bold #00D7AF"`. Every other row in the table already
expects `"bold #00D7AF"` and must keep its existing expected **limit** style unchanged — the limit ladder
(`not bold dim` / `bold #FFD700` / `bold #FF8700` / `bold #FF5F5F`) is still under test and must not be weakened.

Then add a focused regression test next to it, so the invariant is stated directly rather than only implied by a table.
Something equivalent to:

```python
def test_running_count_style_is_constant_across_capacity_pressure() -> None:
    """The running count never encodes pressure; only the limit does."""
    panel = AgentInfoPanel()
    running_styles = set()
    limit_styles = set()
    for running_count, runner_limit in [(0, 10), (5, 10), (8, 10), (10, 10), (13, 10)]:
        panel._running_count = running_count
        panel._runner_limit = runner_limit
        text = _collect_rich_text(panel)
        occupancy_index = text.plain.index(f"{running_count}/{runner_limit}")
        slash_index = text.plain.index("/", occupancy_index)
        running_styles.add(_style_at_plain_index(text, occupancy_index))
        limit_styles.add(_style_at_plain_index(text, slash_index + 1))

    assert running_styles == {"bold #00D7AF"}
    # The retained pressure signal still varies across the same inputs.
    assert len(limit_styles) > 1
```

Use the module's existing `_collect_rich_text()` and `_style_at_plain_index()` helpers rather than writing new ones.
`(0, 10)` is included on purpose: a zero running count keeps the same green style (the chip is always rendered, per
`test_agent_count_strip_keeps_zero_running_when_all_counts_are_zero`).

Also confirm the plain-text assertions elsewhere in the module (`"12  [8/10 running]"`,
`"[10/10 running · 1 queued · 4 waiting]"`, `"5  [0/0 running]"`, etc.) still pass untouched — this change alters styles
only, never the chip's plain text or layout.

### 3. `docs/ace.md`

The capacity-chip paragraph (~lines 905-911) currently ends:

> Occupancy is green below the limit, gold at the limit, and red above it; a nonzero queue count is violet.

That sentence documents exactly the behavior being removed, and it never documented the limit's own ladder. Replace it
with a description of the new split, e.g.:

> The occupancy count `R` always renders green, so it reads as a plain count; capacity pressure is carried by `L`, which
> escalates from dim through gold at half the limit, orange at three quarters, and red once `R` reaches or passes it. A
> nonzero queue count is violet.

Keep the rest of the paragraph (slot-participant rules, `%wait(runners=N)` exclusion) as is. Grep `docs/` for other
copies of the old wording before finishing; at the time of writing `docs/ace.md` is the only doc that describes these
colors.

## Explicitly out of scope

- `AgentInfoPanel._runner_limit_style()` and its thresholds — this is the signal being kept.
- `StatisticsPane._runner_occupancy_color()` in `src/sase/ace/tui/modals/statistics_pane_runners.py` (and
  `tests/ace/tui/test_statistics_runners.py`). That is the Config Center → Statistics → Runners historical "Occupancy by
  runner count" table, a different surface with no adjacent limit number to carry the signal. The request was
  specifically about the Agents-tab header count, so leave it alone.
- The per-group/per-clan `"N agents · M running"` banner summaries — they have no limit denominator and no pressure
  coloring.
- Any `../sase-core` change. Picking a Rich style for a Textual widget is presentation-only under the Rust core backend
  boundary rule, so this stays entirely in the Python repo. No wire/API/binding change is needed.

## Verification

```bash
just install                                      # ephemeral workspace may have stale deps
just check                                        # required: source files changed
pytest tests/ace/tui/widgets/test_agent_info_panel.py -q
pytest tests/ace/tui/test_statistics_runners.py -q # confirm the untouched surface still passes
just test-visual                                  # PNG goldens
```

Expected PNG-snapshot outcome: **no golden regeneration.** The Agents-tab visual fixtures stub
`sase.config.core.get_max_running_agents` to `10` (see `tests/ace/tui/visual/test_ace_png_snapshots_agents.py:133` and
`tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py:39`) while running well under five agents, so those
captures were already rendering the running count in `bold #00D7AF` and the limit in the neutral dim style. If any
golden does shift, that means a fixture sits at ≥100% occupancy — treat the diff as expected for that one capture,
inspect the artifacts under `.pytest_cache/sase-visual/`, and only then accept it with `--sase-update-visual-snapshots`.
Do not blanket-regenerate the corpus.

## Manual check (optional but cheap)

Launch `sase ace`, open the Agents tab, and confirm the leading count stays the same green whether occupancy is below,
at, or above the configured limit, while the number after the `/` still shifts toward red under pressure.

## Done when

- The running count in the Agents-tab capacity chip renders `bold #00D7AF` at every occupancy, including at and above
  the limit.
- The runner-limit number still escalates dim → gold → orange → red, with its thresholds still covered by tests.
- A regression test asserts the running count's style invariance across pressure levels.
- `docs/ace.md` describes the new split instead of the removed running-count coloring.
- `just check` passes and the PNG golden corpus is unchanged.
