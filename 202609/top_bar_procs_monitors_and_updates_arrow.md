---
tier: tale
title: Fold monitors into the top-bar procs group and restore the updates arrow
goal:
  The top-bar indicator cluster drops its separate `monitors:` group, showing the orange
  monitor gear chip directly beside the blue proc gear chip inside `procs:`, and the
  `updates:` chip carries the `⬆` arrow again in its SASE and agent-CLI segments.
size: medium
proposed_by: bbugyi200.athena.0qf
create_time: 2026-09-23 20:45:57
status: wip
---

# Plan: Fold monitors into the top-bar `procs:` group and restore the `⬆` updates arrow

## Goal

The recent labeled top-bar cluster (commits `f4d70c452` "labeled dot-separated top-bar
indicator cluster" and `c44f69618` "provider-priority icon chip") gave the right-hand
indicator row eight `<type>: <body>` groups:
`procs · monitors · updates · overrides · priority · disabled · stash · inbox`.

Two follow-up changes:

1. **Remove the `monitors:` group.** Monitors are procs, so the orange monitor gear chip
   moves into the `procs:` group, directly right of the blue proc gear chip — exactly
   the pairing the Procs tab header already renders
   (`gear_chip(proc) + gear_chip(monitor)` in
   `src/sase/ace/tui/modals/procs_pane_selection.py::_title_text`). The cluster goes
   from eight groups / seven separators to seven groups / six separators.
2. **Restore the `⬆` arrow in the `updates:` chip.** `f4d70c452` dropped `UPDATE_GLYPH`
   (`⬆`, U+2B06) from the badge body because the label now names the group. Put it back
   inside the fill in both segments, exactly as it was before that commit: SASE segment
   `⬆ N`, agent-CLI segment `CLI ⬆ N` (the inset `core` tag is unchanged and still
   follows the SASE segment). This also makes the updates chip follow the cluster's own
   rule that "count chips carry their identity glyph inside the fill" so compact density
   (labels dropped) still identifies the group.

Target renders (full density):

| State                         | Before                           | After                           |
| ----------------------------- | -------------------------------- | ------------------------------- |
| 2 procs, 1 monitor            | `procs: [⚙ 2] · monitors: [⚙ 1]` | `procs: [⚙ 2][⚙ 1]`             |
| 0 procs, 1 monitor            | `monitors: [⚙ 1]`                | `procs: [⚙ 1]` (orange only)    |
| 2 procs, 0 monitors           | `procs: [⚙ 2]`                   | unchanged                       |
| 3 SASE + core + 2 CLI updates | `updates:  3  core  CLI 2 `      | `updates:  ⬆ 3  core  CLI ⬆ 2 ` |

(`[...]` = filled chip; blue `#48CAE4` for procs, orange `MONITOR_GEAR_HUE` for
monitors.)

## Design decisions

- **Chip adjacency:** blue then orange, no space or separator between them — the same
  concatenation the Procs tab header uses. Each chip already carries its own one-cell
  padding and a distinct fill, so they read as two segments of one group (same grammar
  as the updates badge's adjacent SASE / `core` / CLI segments).
- **Zero handling:** each chip keeps `gear_chip`'s default `hide_at_zero=True`; the
  group hides only when both counts are zero. A monitors-only state shows the orange
  chip alone under the `procs:` label.
- **Click target:** unchanged — `procs:` already opens the Procs tab
  (`CLICK_ACTION = "open_tasks_panel"`), which is where monitors live too.
- **Widget id:** keep `#proc-indicator`; delete `#monitor-indicator` entirely (no
  compatibility shim — it is an internal widget with one production caller).
- **Label:** stays the fixed string `procs` (labels never pluralize/change).

## Implementation

### 1. `src/sase/ace/tui/widgets/proc_indicator.py`

- Delete the `MonitorIndicator` class. Update the module docstring (no longer "proc and
  monitor indicator widgets"; it is the single proc indicator that also counts
  monitors).
- `ProcIndicator` tracks both counts (`self._count`, `self._monitor_count`, both `0`
  initially) and replaces `set_count(count)` with one method that updates both at once
  and repaints once:

  ```python
  def set_counts(self, proc_count: int, monitor_count: int) -> None:
  ```

  No-op when both are unchanged; otherwise `_set_body(...)` and refresh the tooltip
  (keep the existing "only assign tooltip when it changed" pattern).

- `_build_content(proc_count: int, monitor_count: int) -> Text`: return
  `gear_chip(proc_count, PROC_GEAR_HUE)` with
  `gear_chip(monitor_count, MONITOR_GEAR_HUE)` appended (`Text.append_text`), so either
  chip can be absent and both-zero yields `Text("")` (hidden group).
- `_build_tooltip(proc_count: int, monitor_count: int) -> str`:
  - both zero: `"No running procs\nClick to open the Procs tab"` (unchanged text)
  - otherwise join the nonzero parts with `", "`: `"2 running procs"`,
    `"1 running monitor"` (singular/plural as today), then
    `"\nClick to open the Procs tab"`. E.g.
    `"2 running procs, 1 running monitor\nClick to open the Procs tab"`,
    `"1 running monitor\nClick to open the Procs tab"`.
- Rewrite the class docstring: renders `procs: ⚙ N` with the blue chip for ACE-owned
  procs and appends the orange chip for running monitor shells (`sase monitor start`),
  the same pair the Procs tab header shows; hidden only when both are zero; click opens
  the Procs tab.

### 2. `src/sase/ace/tui/actions/_proc_action_observer.py`

- Drop the `MonitorIndicator` import.
- `_update_proc_indicator`: a single `query_one("#proc-indicator", ProcIndicator)` then
  `indicator.set_counts(gear_eligible_count(projection), projection.active_monitor_count)`
  inside the existing `try/except Exception: pass`. Update the docstring (one widget
  now; the "missing indicator never blocks the other" sentence goes away).

### 3. `src/sase/ace/tui/widgets/top_bar.py`

- Remove the `MonitorIndicator` import, the `"monitor-indicator"` entry in
  `_TOP_BAR_GROUP_IDS`, and the `MonitorIndicator(id="monitor-indicator")` compose
  entry.
- Fix the count words in docstrings: "Yield the seven groups…", "Return the seven
  indicator groups…", "Return the six separator widgets…".

### 4. Widget exports

- `src/sase/ace/tui/widgets/__init__.py`: remove the `"MonitorIndicator"` entry from the
  lazy-export map and from `__all__`.
- `src/sase/ace/tui/widgets/__init__.pyi`: remove the `MonitorIndicator` re-export.

### 5. `src/sase/ace/tui/widgets/top_bar_group.py`

- `icon_count_chip` docstring: "Shared body for procs, monitors, and prompts" → keep
  accurate (e.g. "Shared body for the proc/monitor gear chips and the stash chip").
  Behavior unchanged.

### 6. `src/sase/ace/tui/proc_gear_chips.py`

- Module docstring: the blue/orange chip pair is shared by the Procs tab header and the
  top bar's single `procs:` group (`ProcIndicator`), not `ProcIndicator` /
  `MonitorIndicator`. No code change.

### 7. `src/sase/ace/tui/widgets/updates_indicator.py`

- Import `UPDATE_GLYPH as _UPDATE_GLYPH` from `.update_accents` again.
- `_build_content`: SASE segment `f" {_UPDATE_GLYPH} {count} "`, CLI segment
  `f" CLI {_UPDATE_GLYPH} {agent_cli_count} "`; styles, `core` tag placement, and
  ordering unchanged.
- Class docstring: "Renders as `updates: ⬆ N`…".

### 8. `src/sase/ace/tui/widgets/update_accents.py`

- Replace the paragraph that says the top-bar badge "no longer carries the `⬆` identity
  glyph" with the pre-`f4d70c452` rationale: the glyph is `⬆` (U+2B06), a solid
  pictogram matching the row's `⚙ ≡ ★` neighbors instead of punctuation, keeps the "up =
  upgrade" direction, is in both bundled Fira Code weights, and is always one cell wide;
  it is shared by the top-bar badge, the Update panel, and the plugins browser.

### 9. Docs

- `docs/ace.md` → "Top-Bar Indicators": order list becomes `procs`, `updates`,
  `overrides`, `priority`, `disabled`, `stash`, `inbox`; glyph sentence becomes "`⚙` for
  procs (a blue chip for sase's TUI procs, plus an orange chip for monitor shells), `⬆`
  for updates, `≡` for the stash, `★` for priority"; click sentence drops "and
  monitors".
- `docs/ace.md` → "Proc Indicator": describe the combined group — blue `⚙ N` chip for
  the TUI's own procs, followed by an orange `⚙ N` chip for running monitor shells
  (`sase monitor start`), the same pair the Procs tab header shows; monitors are counted
  separately (a detached supervisor that survives TUI exit and never blocks TUI procs)
  but live in the same group; hides only when both are zero; click opens the Procs tab.
  Keep the existing sentences about excluding service-host rows. Delete the "### Monitor
  Indicator" subsection and its `[Monitor Indicator](#monitor-indicator)` cross-link
  (rephrase that sentence to point at the orange chip in the same group).
- `docs/ace.md` → Procs tab "Monitors on this tab": "matching the Agents tab and the
  top-bar monitor indicator" → "matching the Agents tab and the orange chip in the top
  bar's `procs:` group".
- `docs/ace.md` (Updates section, "The top bar renders the `updates:` group…") and
  `docs/configuration.md` ("The persistent `updates:` top-bar group…"): segments are now
  lime `⬆ N` and sage `CLI ⬆ N`.
- Grep `docs/` once more for `monitors:`, `monitor-indicator`, `Monitor Indicator`, and
  `CLI N` to catch stragglers.

### 10. Tests

Unit / mounted tests:

- `tests/ace/tui/widgets/test_proc_indicator.py`: replace the monitor tests with
  `ProcIndicator` coverage for: procs-only (`" ⚙ 2 "`, blue style), monitors-only
  (`" ⚙ 1 "`, orange `bold #1a1a1a on #FFAF5F`), both (plain `" ⚙ 2  ⚙ 1 "`, blue span
  before orange span), both-zero hidden, and the four tooltip shapes above. Assert
  `_build_content(p, m)` equals `gear_chip(p, PROC_GEAR_HUE)` +
  `gear_chip(m, MONITOR_GEAR_HUE)` concatenated. Add a `set_counts` no-op-on-unchanged
  check if cheap.
- `tests/ace/tui/test_proc_actions_session_workers.py`: `_FakeIndicator` records
  `set_counts(proc_count, monitor_count)` tuples; the split test asserts
  `proc_indicator.counts == [(2, 1)]` with only `#proc-indicator` registered; replace
  the "missing widget does not block the other" test with "missing proc indicator is a
  no-op" (no exception).
- `tests/ace/tui/test_top_bar_order.py`: drop `"monitor-indicator"` from
  `EXPECTED_TOP_BAR_CLUSTER_ORDER`; comment says seven groups; update
  `"updates:  3  core  CLI 2 "` → `"updates:  ⬆ 3  core  CLI ⬆ 2 "`.
- `tests/ace/tui/test_top_bar_indicators.py`: drop the `MonitorIndicator` import;
  `_drive_busy` calls `set_counts(2, 1)` on `#proc-indicator`; remove `"monitors:"` from
  the wide label list and add `assert "monitors:" not in text` plus a check that the
  procs group renders both chips (e.g. `"procs:  ⚙ 2  ⚙ 1 "` in text); compact test also
  asserts `"⬆"` survives compact density; the click test drops the monitor click and its
  expected `"open_tasks_panel"` entry.
- `tests/test_updates_indicator.py`: every expected plain string gains the arrow
  (`" ⬆ 3 "`, `" ⬆ 3  core "`, `" CLI ⬆ 2 "`, `" ⬆ 3  CLI ⬆ 2 "`,
  `" ⬆ 3  core  CLI ⬆ 2 "`, and any `rendered` comparisons further down).
- `tests/ace/tui/test_top_bar_palette.py`: no change expected (it uses `gear_chip`
  directly and color/contrast only) — confirm it still passes.

Visual (PNG) tests:

- `tests/ace/tui/visual/test_ace_png_snapshots_top_bar_indicators.py`: remove
  `MonitorIndicator`; `set_counts(2, 1)` on `#proc-indicator` in both drivers; module
  and test docstrings say "seven groups".
- `tests/ace/tui/visual/test_ace_png_snapshots_updates_indicator.py`: all
  `render().plain` waits gain the arrow (`"updates:  ⬆ 3 "`, `"updates:  ⬆ 3  core "`,
  `"updates:  CLI ⬆ 2 "`, `"updates:  ⬆ 3  CLI ⬆ 2 "`,
  `"updates:  ⬆ 3  core  CLI ⬆ 2 "`, compact `" ⬆ 3  core  CLI ⬆ 2 "`); the
  with-neighbors test uses `ProcIndicator.set_counts(1, 1)` and waits on the proc
  indicator only; update its docstring ("next to the proc/monitor gears in the procs
  group").
- Grep `tests/` for `MonitorIndicator`, `monitor-indicator`, `monitors:`, and
  `.set_count(` on `ProcIndicator` to be sure nothing is left.

## Verification

1. `just fix` (or at least `just fmt`) inline, then `sase tool run check` (install first
   with `just install` if the workspace venv is stale). Symvision must not flag the
   removed `MonitorIndicator` or the new `set_counts`.
2. Refresh PNG goldens with a **full** `just fix-tui-screenshots` run through
   `/sase_monitor` (`TESTING` / `TESTED`, generous timeout). Many goldens show the top
   bar's updates badge or monitor chip, so expect updates across (at least) the
   `updates_indicator_*`, `top_bar_indicators_*`, `top_bar_*usage*`,
   `config_center_updates_*`, `config_center_plugins_*update*`,
   `config_center_procs_tab*`, `agents_proc_shell*`,
   `agents_settled_monitor_lane_badge`, `*update_toast*`, and `update_panel_*` families.
   The monitor follow-up must inspect the retained report
   (`.pytest_cache/sase-visual/latest-report.json`) and confirm **every** diff is
   confined to the top-bar row: the `monitors:` label and its separator are gone with
   the orange chip now adjacent to the blue chip under `procs:`, and the `⬆` appears
   inside the moss updates chip. Any diff outside the top-bar band is a bug to
   investigate, not a golden to accept. There should be no golden creations or removals.
3. Optional live confirmation: `sase screenshot -o /tmp/top_bar.png` with a monitor
   running and updates pending, and eyeball the top-right row.

## Out of scope

- No change to the Procs tab header, gear hues, `gear_chip`, or the Update panel /
  plugins browser use of `UPDATE_GLYPH`.
- No change to the status rows' `load: · model: · project:` launch-context cluster.
- No Rust-core change: this is presentation-only Textual rendering.
