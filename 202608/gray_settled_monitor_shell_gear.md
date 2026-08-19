---
tier: tale
title: Gray the monitor-shell gear glyph once its monitor settles
goal:
  A monitor shell row on the Agents tab renders a plain grey gear once its monitor
  reaches a terminal state, matching the grey settled-monitor count badges already used
  on container rows and panel titles, while a running monitor keeps its bold amber gear.
size: small
proposed_by: bbugyi200.athena.076
create_time: 2026-08-18 20:11:24
status: wip
---

# Gray the monitor-shell gear glyph once its monitor settles

## Goal

On the Agents tab, a monitor shell row renders a leading `⚙` glyph that is **always**
bold amber (`#FFAF5F`), whether that monitor is still running or finished hours ago.
Make the row glyph switch to the already-established settled-monitor grey (`#9E9E9E`,
non-bold) once the monitor reaches a terminal state, so a monitor shell row matches the
grey `⚙N` lane badges this codebase already renders for finished monitors on container
rows and panel titles.

After this change the Agents tab has exactly one gear contract:

- **bold amber `⚙`** — this monitor is still running.
- **plain grey `⚙`** — this monitor is finished.

## Background: what exists today

### The row glyph (the thing being changed)

`src/sase/ace/tui/widgets/_agent_list_render_agent.py` renders the monitor row glyph at
two sites inside `format_agent_option`, both hardcoded to the running style:

- the tree-child branch (`if tree_depth > 0:` → `if agent.is_monitor:`), around line 197
- the top-level branch (`elif agent.is_monitor:`), around line 204

Both call `text.append(f"{_MONITOR_GLYPH} ", style=_MONITOR_GLYPH_STYLE)`.

`src/sase/ace/tui/widgets/_agent_list_styling.py` defines the styles:

```python
_MONITOR_GLYPH = MONITOR_GLYPH                             # "⚙"
_MONITOR_GLYPH_STYLE = f"bold {MONITOR_GLYPH_COLOR}"       # bold #FFAF5F
_MONITOR_ROW_STYLE = MONITOR_GLYPH_COLOR                   # #FFAF5F
_MONITOR_COUNT_GLYPH_STYLE = f"bold {MONITOR_GLYPH_COLOR}"
_MONITOR_SETTLED_COUNT_GLYPH_STYLE = MONITOR_SETTLED_GLYPH_COLOR   # #9E9E9E
```

The comment above `_MONITOR_SETTLED_COUNT_GLYPH_STYLE` records the existing doctrine and
MUST be honored by the new style: the settled lane is _deliberately non-bold_ so hue
**and** weight both separate it from the live amber lane, keeping the two
distinguishable on low-color terminals and under red/green color vision deficiency.

### The grey gear that already exists ("elsewhere")

Two surfaces already partition monitors into an amber running lane and a grey settled
lane:

1. **Container-row count badges** — `_agent_list_render_agent.py` near line 490 renders
   `⚙{lanes.running}` in `_MONITOR_COUNT_GLYPH_STYLE` and `⚙{lanes.settled}` in
   `_MONITOR_SETTLED_COUNT_GLYPH_STYLE`.
2. **Panel border titles** — `src/sase/ace/tui/actions/agents/_display_panel_titles.py`
   renders `⚙{counts.running_monitors}` in `_PANEL_MONITOR_RUNNING_STYLE` and
   `⚙{counts.settled_monitors}` in `_PANEL_MONITOR_SETTLED_STYLE`
   (`MONITOR_SETTLED_GLYPH_COLOR`).

Both lane partitions are computed by `_MonitorLaneTally` in
`src/sase/ace/tui/models/agent_family_members.py`, which calls the module-private
predicate:

```python
def _monitor_row_is_settled(row: Agent) -> bool:
    return monitor_state_is_terminal(row.monitor_state) or row.stop_time is not None
```

This is the exact predicate the row glyph must reuse. Reusing it — rather than writing a
fresh `monitor_state in {...}` check — is the whole point: a monitor row that renders a
grey gear must be the same monitor row that increments the grey `⚙N` badge on its parent
container and its panel title. Any second, independently-written predicate can drift and
produce an amber row inside a grey count (or the reverse).

Note the `or row.stop_time is not None` clause. It is load-bearing: a monitor
family-member row whose `monitor_state` was never reconciled to a terminal value is
still over, and the lane partition deliberately settles it rather than stranding it. The
glyph must agree.

### "Completes" means "settled", not "succeeded"

`sase/monitor_state.py` defines the terminal set through `MONITOR_STATE_BUCKETS` /
`monitor_state_is_terminal`: `completed`, `failed`, `timeout`, `stopped`, and `lost` are
all terminal; `running` and any unrecognized/missing state are not. The existing grey
lane already covers all five — the visual fixture docstring in
`tests/ace/tui/visual/_ace_agents_png_snapshot_fixtures.py` states outright that its mix
of a clean completion, a failure, and an explicit stop exists so "the grey settled badge
is proven to read as 'finished', not merely 'succeeded'".

So the grey gear means **finished**, in any terminal state. The _outcome_ of a finished
monitor is already carried by other, unchanged parts of the row: the green/red status
word, the `✗ <code>` exit badge, the `⧖` timeout badge, the `⚠` stalled badge, and the
`⚑` follow-up-error badge. The gear carries lifecycle only, exactly as the lane badges
already do.

### Render cache — already correct, do not change

`agent_render_key` in `src/sase/ace/tui/widgets/_agent_list_render_cache.py` already
folds in `agent.monitor_state` (explicitly, around line 232) and `agent.stop_time` (via
`_runtime_signature`, around line 120). Both inputs to the new predicate are therefore
already part of the memoization key, so a monitor that settles naturally invalidates its
cached row and re-renders grey. **No cache-key change is needed.** Add a regression test
asserting this rather than editing the key.

## Approach

### Step 1 — Promote the settled predicate to public API

In `src/sase/ace/tui/models/agent_family_members.py`:

- Rename `_monitor_row_is_settled` to `monitor_row_is_settled` and update its in-file
  caller in `_MonitorLaneTally.visit`.
- Add `"monitor_row_is_settled"` to the module's `__all__` (alphabetical order, between
  `is_sequential_family_container` and `monitor_lane_counts`).
- Keep the existing docstring and extend it to say the predicate is now shared by the
  lane counts _and_ the per-row glyph style, so the two can never drift.

Do **not** leave a `_monitor_row_is_settled` alias behind. Symvision reports a dead
private symbol
(`Private functions/classes must be used in the file where they are defined`), and there
is no back-compat consumer to protect.

Symvision note: promoting a private symbol to public requires a real non-test consumer,
and Step 3 supplies one (`_agent_list_render_agent.py`). Test references alone would not
satisfy the linter.

### Step 2 — Add the settled row-glyph style

In `src/sase/ace/tui/widgets/_agent_list_styling.py`, next to `_MONITOR_GLYPH_STYLE`,
add:

```python
_MONITOR_SETTLED_GLYPH_STYLE = MONITOR_SETTLED_GLYPH_COLOR
```

Non-bold, matching `_MONITOR_SETTLED_COUNT_GLYPH_STYLE` exactly. Add a short comment
pointing at the existing weight-plus-hue rationale block so the next reader does not
"helpfully" bold it. `MONITOR_SETTLED_GLYPH_COLOR` is already imported in this file — no
new import is required.

### Step 3 — Pick the style per row

In `src/sase/ace/tui/widgets/_agent_list_render_agent.py`:

- Import `monitor_row_is_settled` from `..models.agent_family_members` (that module is
  already imported here for `NO_MONITOR_LANES`, `MonitorLaneCounts`,
  `is_sequential_family_container`, and `monitor_lane_counts`), and import
  `_MONITOR_SETTLED_GLYPH_STYLE` from `._agent_list_styling`.
- Add one module-level helper above `format_agent_option`, alongside the other small
  `_should_render_*` predicates:

```python
def _monitor_glyph_style(agent: Agent) -> str:
    """Return the row gear style for a monitor shell.

    Shares ``monitor_row_is_settled`` with the ``⚙N`` lane counts so a grey
    gear on a row and the grey count it feeds can never disagree.
    """
    return (
        _MONITOR_SETTLED_GLYPH_STYLE
        if monitor_row_is_settled(agent)
        else _MONITOR_GLYPH_STYLE
    )
```

- Replace the style argument at **both** gear sites (the `tree_depth > 0` child branch
  and the top-level `elif agent.is_monitor:` branch) with
  `style=_monitor_glyph_style(agent)`. Leave the glyph text itself untouched.

Leave `_MONITOR_GLYPH_STYLE` in place: it is still used for the running row glyph and
for the `MONITORING` status word (around line 286).

### Step 4 — Update the Agents tab help legend

`src/sase/ace/tui/modals/help_modal/agents_bindings.py` (around line 443) lists, under
"Agent Row Glyphs":

```python
("⚙", "Monitor shell (label)"),
("⚙N", "N running monitors (amber)"),
("⚙N", "N finished monitors (grey)"),
```

The bare `⚙` entry now describes two colors, so split it to match the wording already
used by the two count entries directly beneath it:

```python
("⚙", "Monitor shell, running (amber)"),
("⚙", "Monitor shell, finished (grey)"),
("⚙N", "N running monitors (amber)"),
("⚙N", "N finished monitors (grey)"),
```

Keep the entries in that order so row glyphs precede count glyphs. If a help test
asserts the old legend text, update that assertion too.

## Tests

### `tests/ace/tui/widgets/test_agent_list_monitor_rows.py`

This file already has everything needed: a `_monitor(...)` factory that sets `stop_time`
for terminal states, a `_family_container(...)` factory, and a
`_style_at(text, position)` span-style probe. Import `_MONITOR_GLYPH_STYLE` and
`_MONITOR_SETTLED_GLYPH_STYLE` from `sase.ace.tui.widgets._agent_list_styling` alongside
the existing style imports.

Add:

1. A running monitor row still renders the bold amber gear — assert
   `_style_at(left, left.plain.index("⚙")) == _MONITOR_GLYPH_STYLE` for
   `_monitor(status="MONITORING", monitor_state="running")`.
2. Every terminal state renders the grey gear — parametrize over `completed`, `failed`,
   `timeout`, `stopped`, and `lost` (with the matching `MONITORED` status), asserting
   `_style_at(...) == _MONITOR_SETTLED_GLYPH_STYLE`. This is the test that pins
   "completes" to the full terminal set.
3. A monitor row with `monitor_state=None` and no `stop_time` renders the amber gear —
   an un-reported monitor must not read as finished.
4. A monitor row with a non-terminal (or missing) `monitor_state` **and** a `stop_time`
   renders the grey gear — pins the `stop_time` fallback clause and keeps the row in
   step with `test_monitor_lane_counts_running_state_with_stop_time_counts_settled`.
5. A tree-child monitor row (`tree_depth > 0`, i.e. a monitor nested under its starter)
   that is settled renders the grey gear — proves both call sites were changed, not just
   the top-level one. Build it the way `_family_container` does: a root with
   `followup_agents` containing the monitor, then render the monitor row itself.
6. Row-and-badge agreement: render a settled monitor row and its family container, and
   assert the row gear style equals `_MONITOR_SETTLED_COUNT_GLYPH_STYLE` (the container
   badge style) — one assertion that fails loudly if the two lanes are ever
   re-implemented apart.

Also confirm the existing
`test_monitor_starter_row_uses_agent_rendering_not_monitor_rendering` still passes: the
starter row is not `is_monitor`, so it renders no gear at all in either color.

### `tests/ace/tui/models/test_agent_family_members.py`

Update the import of the predicate to the new public name if the file imports it. Every
existing `monitor_lane_counts` / `panel_monitor_lane_counts` test must keep passing
unchanged — the rename must not alter behavior. Optionally add one direct
`monitor_row_is_settled` unit test covering `running` → `False`, each terminal state →
`True`, unknown-without-`stop_time` → `False`, and unknown-with-`stop_time` → `True`.

### `tests/ace/tui/widgets/test_agent_render_cache.py`

Add a regression test: build a settled and a running monitor row that are otherwise
identical and assert their `agent_render_key` values differ, so a future cache-key
refactor cannot silently freeze a monitor row on its stale amber gear.

### Visual (PNG) snapshots

Two Agents-tab PNG goldens involve monitors:
`agents_settled_monitor_lane_badge_120x40.png` and
`agents_family_conversation_monitor_120x40.png`. Both are rendered with the family
**collapsed** (their tests assert `agent_count == 1` and check the container/panel `⚙N`
badges), so the monitor member rows themselves are most likely not on screen and the
goldens should be unaffected.

Do not assume it. Run `just test-visual` and:

- If both pass, done — say so.
- If a golden diffs, inspect the artifacts under `.pytest_cache/sase-visual/` and
  confirm the **only** change is a monitor row gear going amber → grey. If so, accept it
  with `--sase-update-visual-snapshots` and note the regenerated file in the commit
  message. If anything else moved, stop and investigate instead of blessing the diff.

## Non-goals

Keep the diff to the row gear. Each of these is deliberately excluded:

- **The monitor row's label color.** `_MONITOR_ROW_STYLE` (`#FFAF5F`) tints the
  monitor's `monitor_label` on every monitor row regardless of state. Graying it is a
  separate, larger visual decision about row legibility, and
  `test_monitor_starter_row_uses_agent_rendering_not_monitor_rendering` asserts against
  that style today. Leave it amber.
- **The MONITOR phase divider** in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_monitor_section.py`
  (`build_monitor_phase`, `accent=MONITOR_GLYPH_COLOR, glyph=MONITOR_GLYPH`). Its accent
  colors an entire divider rule, not just a glyph, so graying it is a visibly bigger
  change than the one requested.
- **The Procs tab gear** in `src/sase/ace/tui/modals/procs_pane_render.py`
  (`_append_monitor_marker`). That surface marks "this proc row _is_ a monitor shell"
  and already carries its own `●`/`✓`/`✗` status glyph; it is a different tab and out of
  the stated scope.
- **The `⚙` in the detail panel's `_STATE_DISPLAY` block** and the `⧖`/`✗`/`⚠`/ `⚑`
  outcome badges. Outcome stays where it already lives.
- **`agent_render_key`.** It already covers `monitor_state` and `stop_time`; changing it
  would be churn.
- **A feature flag.** This is a finished, user-requested cosmetic correction with no old
  branch that must stay reachable and nothing for a user to choose permanently — per the
  feature-flag rules it is neither a beta, an early landed path, nor a deprecation, so
  no flag and no flag bead.

## Verification

```bash
just install                 # ephemeral workspaces may have drifted deps
just check                   # whole-repo lint gates + diff-scoped tests
just test-visual             # PNG snapshot suite (see "Visual" above)
```

`just check` covers ruff, mypy, and Symvision — the last of these is the gate that
confirms Step 1's promotion landed with a real non-test consumer.

Targeted runs while iterating:

```bash
just test tests/ace/tui/widgets/test_agent_list_monitor_rows.py
just test tests/ace/tui/models/test_agent_family_members.py
just test tests/ace/tui/widgets/test_agent_render_cache.py
just test tests/ace/tui/test_agent_panel_titles.py
```

If `just check` or `just test-visual` runs long, hand it to `/sase_monitor` with a
`--next` action rather than blocking a turn on it.

## Done when

- A monitor shell row on the Agents tab renders a plain grey `⚙` once its monitor is in
  any terminal state (`completed`, `failed`, `timeout`, `stopped`, `lost`) or has a
  `stop_time`, and a bold amber `⚙` while it is still running or has not yet reported.
- The grey used is `MONITOR_SETTLED_GLYPH_COLOR` (`#9E9E9E`), non-bold — byte-identical
  to the grey already used by the `⚙N` settled count badges on container rows and panel
  titles.
- Row glyph and lane badge are driven by the single shared predicate
  `monitor_row_is_settled`; no second copy of the settled test exists.
- Both gear call sites in `format_agent_option` changed (top-level and tree-child),
  proven by tests.
- The Agents tab help legend documents both gear colors.
- `just check` and `just test-visual` pass; any regenerated PNG golden is explained.
