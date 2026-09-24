---
tier: tale
title: Green update gear in the top bar while SASE is updating
goal:
  While SASE is updating itself, the top-bar updates badge shows a green proc gear just
  left of its arrow instead of counting the update in the blue procs gear. Hovering
  explains what is running, clicking opens the running update in the Procs tab, and the
  Procs tab uses the same green gear.
size: medium
proposed_by: bbugyi200.athena.0qn
create_time: 2026-09-24 10:05:05
status: wip
---

# Plan: Green update gear in the top bar while SASE is updating

## Goal

While SASE is updating itself, show a small **green proc gear** at the left edge of the
top-bar `updates:` badge, just before the `⬆` arrow. Show it instead of counting the
update proc in the blue `procs:` gear. Today an update is just "one more blue proc",
which you can't tell apart from a sync or a mail run. After this change, the update
shows up right where you look for updates, in the update color family, and nowhere else.

## Design

### What it looks like

The `updates:` group gets a leading **update-gear inset**: a single `⚙` cell in bold
dark ink (`#1a1a1a`) on the update lime (`UPDATES_ACCENT`, `#AFFF87`). It sits directly
against the deep-moss `⬆ N` segment, so the two read as one two-part pill:

| State                                               | Top bar (right cluster)                         |
| --------------------------------------------------- | ----------------------------------------------- |
| Idle, 3 updates available                           | `procs: [⚙ 1] · updates: [⬆ 3]` (unchanged)     |
| Updating, no other procs                            | `updates: [⚙][⬆ 3]` (the blue group disappears) |
| Updating + 1 other proc                             | `procs: [⚙ 1] · updates: [⚙][⬆ 3]`              |
| Updating, nothing counted (mode switch, dev update) | `updates: [⚙]`                                  |
| Everything                                          | `updates: [⚙][⬆ 3][core][CLI ⬆ 2]`              |

Design decisions and why:

- **The gear is the update identity color, filled in.** The badge's existing rule
  (`update_accents.py`) is "hue is identity; state lives inside the chip". The gear uses
  the same recipe as the blue/orange proc gears and the `core` tag: a bright fill with
  dark ink, in lime. So it reads as "the updates lane's proc gear". Lime sits about 90°
  of hue away from the blue proc gear and is much lighter, and it always touches the
  moss chip. That keeps it distinct even when the blue gear is visible in the next
  group.
- **Bare glyph, never a count, in the top bar.** The gear shows a state ("an update is
  running"), not a quantity. `⚙ 1 ⬆ 3` would put two numbers side by side and blur which
  one is the update count. The rare case of several concurrent update procs (for
  example, two plugin updates) is listed in the tooltip and counted in the Procs tab
  header.
- **Static, not animated.** Nothing in the top bar animates. A timer would add pump
  work, make visual goldens nondeterministic, and compete with agent spinners. A bright
  lime inset on the only dark chip in the row is already the most eye-catching thing in
  the cluster.
- **The group stays visible while an update runs**, even when both update counts are
  zero (`updates: [⚙]`). That way an update started from the plugins browser or a mode
  switch still shows.

### What counts as "SASE is updating"

A row is in the **update lane** when it is an active, gear-eligible row (not a monitor
shell, not a service row) that plans or applies a change to the installed SASE stack:

- session-worker `proc_type` in
  `{"update-preview", "comprehensive-update", "sase-update", "dev-update", "agent-cli-update", "mode-switch"}`,
  or
- any row whose `exclusive_scopes` contains `"sase-update"` or `"agent-cli-update"`, or
  a scope starting with `"plugin-update:"`. This covers durable plugin updates. Their
  store rows have a generic kind, and their pending placeholder uses
  `proc_type="plugin.update"`.

`update-preview` (the `,U` planning proc) is included on purpose. Pressing `,U` should
light the green gear right away, and an auto-approved update should never flip from blue
to green halfway through.

These are explicitly **not** in the update lane: plugin/agent-CLI installs and
uninstalls (`plugin.install`, `plugin.install_many`, `plugin.uninstall`,
`agent-cli-install`, `agent-cli-plugin-install`), the automatic background update check
(it is not a proc), monitors, and service rows.

### Counting rules (these must hold)

- Blue = gear-eligible rows minus update rows. Green = update rows. Orange = unchanged.
- `gear_eligible_count()` keeps its current meaning (it still includes update rows). The
  quit confirmation (`LifecycleMixin._count_running_tasks`) and the post-update restart
  drain (`update_restart.running_background_procs`) must still see a running update.
  Quitting mid-update must still warn. **Do not change these callers.**
- Totals are preserved:
  `lanes.procs + lanes.updates == gear_eligible_count(projection)`.

### Interaction

- **Tooltip while updating** (tooltips are computed only when state changes, never on a
  timer):
  - Line 1: `Update in progress: <label>`, or `N updates in progress: <label>, <label>`
    (labels are `ObservedProc.label`, oldest first).
  - Line 2: `Click to watch it in the Procs tab.`
  - Then, if any counts are nonzero, the existing availability sentence (domain counts,
    the sase-core Rust-rebuild note, the manual agent-CLI note). Drop the "Click to open
    Updates, or press ,U …" guidance, because clicking now goes to the Procs tab and
    `,U` would only be rejected as a duplicate.
  - When not updating, the tooltip stays byte-for-byte what it is today.
- **Click while updating** runs a new app action `open_update_procs`. It opens the Admin
  Center **Procs** tab with the oldest running update row preselected, so the live
  update log is one click away. When no update is running, the click keeps opening the
  Updates tab as it does today.
- **Procs tab.** The header gains a green `⚙ N` chip right after the orange monitor
  chip, only while update procs run (`gear_chip(n, UPDATE_GEAR_HUE)`, hidden at zero).
  The blue header chip excludes update rows, so it matches the top-bar blue gear. Update
  rows get a green `⚙` marker, mirroring how monitor rows get the orange one. Clicking
  the green gear therefore lands on a row wearing the same green gear.

## Implementation

This is Python/TUI presentation over TUI-session proc rows. Session workers exist only
in the TUI process, and the classifier sits next to the existing `gear_eligible_count()`
read-model helper. No `sase-core` change is needed.

### 1. Lane classifier and aggregate — `src/sase/ace/tui/_proc_observer_models.py`

- Add module constants `UPDATE_PROC_TYPES` (the six session proc types above),
  `UPDATE_EXCLUSIVE_SCOPES = frozenset({"sase-update", "agent-cli-update"})`, and
  `PLUGIN_UPDATE_SCOPE_PREFIX = "plugin-update:"`, with a short comment on why each
  signal exists (session proc types vs durable concurrency keys).
- `is_update_row(row) -> bool`: `is_gear_eligible_row(row)` and (proc_type in
  `UPDATE_PROC_TYPES`, or any scope in `UPDATE_EXCLUSIVE_SCOPES` or starting with the
  plugin prefix). Like the neighboring predicates, callers still gate on the row being
  active.
- `GearLane = Literal["proc", "update", "monitor"]` and
  `proc_gear_lane(row) -> GearLane | None`: monitor shells → `"monitor"`, service rows →
  `None`, update rows → `"update"`, everything else → `"proc"`. This is the one row
  classifier that both the top bar and the Procs header use.
- A frozen dataclass `ProcGearLanes` with `procs: int = 0`, `monitors: int = 0`,
  `update_rows: tuple[ObservedProc, ...] = ()` (oldest `started_at` first), plus
  `updates` (len) and `update_labels` (tuple of `row.label`) properties.
- `proc_gear_lanes(projection, *, all_sessions=False) -> ProcGearLanes`: one pass over
  `projection.active_rows(all_sessions=...)`. It is pure and in-memory (no I/O; it runs
  on every observer snapshot).
- Update the `gear_eligible_count` docstring to say it includes update rows and that the
  top bar splits them out with `proc_gear_lanes`.
- Re-export the new public names from `src/sase/ace/tui/proc_observer.py` the same way
  it re-exports `is_gear_eligible_row`, and add them to `__all__` wherever that module
  pattern expects.

### 2. Green gear chip — `src/sase/ace/tui/proc_gear_chips.py`

- `UPDATE_GEAR_HUE = UPDATES_ACCENT` (import from `widgets.update_accents`).
- `update_gear_chip(active: bool) -> Text`: returns
  `Text(" ⚙ ", style="bold #1a1a1a on <UPDATE_GEAR_HUE>")` when active, else `Text("")`.
  Reuse the module's `_GEAR` glyph and the same dark-ink constant/recipe as
  `icon_count_chip`, and don't hard-code a second copy of `#1a1a1a` if a shared constant
  is practical.
- Update the module docstring: the gear chip now has three lanes. Blue is the session
  proc lane, orange is monitors, and green (the update identity accent) is procs that
  are updating SASE.
- Export the new names in `__all__`.

### 3. Updates badge — `src/sase/ace/tui/widgets/updates_indicator.py`

- Add state `_running_labels: tuple[str, ...] = ()`, a `running_count` property, and
  `set_running(labels: Sequence[str]) -> None`. The count is `len(labels)`. It no-ops
  when the tuple is unchanged and otherwise re-renders body and tooltip.
- Move the body/tooltip refresh into one private `_render()` used by both
  `set_available` and `set_running`. The two inputs stay independent, so the periodic
  update-check refresh and the proc observer can never overwrite each other.
- `_build_content(count, *, core=False, agent_cli_count=0, running=False)`: when
  `running`, prepend `update_gear_chip(True)` before the existing segments. The existing
  output for `running=False` must not change.
- `_build_tooltip(..., running_labels=())`: when non-empty, build the updating tooltip
  described above. Otherwise return exactly today's text.
- Override `on_click`: when `running_count > 0`, run
  `app.run_action("open_update_procs")`, otherwise defer to `TopBarGroup.on_click`
  (Updates tab).
- Refresh the class docstring and the `update_accents.py` module docstring (the design
  narrative) to describe the gear inset. That docstring currently says the badge is the
  only deep chip and describes the `core` inset; add the gear inset with the same
  reasoning.

### 4. One push point — `src/sase/ace/tui/actions/_proc_action_observer.py`

`_update_proc_indicator()` is already called on submit, on every observer snapshot, on
session-worker completion/error, and on durable completion. Make it compute
`lanes = proc_gear_lanes(projection)` once, then:

- `ProcIndicator.set_counts(lanes.procs, lanes.monitors)` (monitor count now comes from
  the lanes, so the top bar and the header use one classifier), and
- `UpdatesAvailableIndicator.set_running(lanes.update_labels)` via
  `query_one("#updates-indicator", UpdatesAvailableIndicator)`.

Wrap each widget update in its own `try` so a missing widget can't block the other. Keep
the import cheap. `updates_indicator` is already imported by `top_bar.py` at startup, so
importing it here does not grow the startup import closure. Check this against the
import-budget test if one flags it.

### 5. Click action — `src/sase/ace/tui/actions/base.py`

Add `action_open_update_procs()` next to `action_open_updates_panel()`:

- Recompute `proc_gear_lanes(self._effective_proc_projection())` at click time. Don't
  trust a value captured at render time.
- If there is an update row, seed the Procs bookmark with it before opening: make sure
  `self._admin_center_session_state` is an `AdminCenterSessionState` (create and assign
  one if missing, as `_open_config_center` does), then set
  `state.procs.task = SelectionBookmark(identity=row.durable_proc_id or row.proc_id)`.
  This matches `ProcsPaneSelectionMixin._task_identity`.
- Always finish with `self._open_config_center("procs")`, so a click that races the
  update finishing still lands on the Procs tab.
- The action is click-only: no new keybinding and no `default_config.yml` change.

### 6. Procs tab — `src/sase/ace/tui/modals/procs_pane_selection.py`, `procs_pane_render.py`

- `_title_text()`: compute the lanes for active tasks with `proc_gear_lane`. The blue
  chip counts `"proc"`, the orange chip counts `"monitor"` (same value as today), and a
  green `gear_chip(update_running, UPDATE_GEAR_HUE)` (hidden at zero) goes after the
  orange chip. Update the inline comment: the blue chip still equals the top-bar blue
  gear.
- `procs_pane_render.py`: add
  `_append_update_marker(text, task, *, prefix="", suffix="")`, which appends `⚙` in
  `bold <UPDATE_GEAR_HUE>` when `proc_gear_lane(task) == "update"`. Call it everywhere
  `_append_monitor_marker` is called: the row label (`task_row_label`) and the detail
  header (around the second call site).

### 7. Docs

- `docs/ace.md`:
  - "Top-Bar Indicators": mention the green gear inset in the `updates` group and that
    clicking it while updating opens the Procs tab on the running update.
  - "Proc Indicator": update procs move to the green gear in `updates:` and no longer
    count in the blue chip.
  - Procs pane header paragraph (the `Procs · this session ⚙ 2 ⚙ 1 …` example): the
    green `⚙ N` chip appears only while updates run, update rows carry a green `⚙`
    marker, and the "blue plus orange" sentence becomes blue + green + orange.
  - The updates-badge sentence near "The top bar renders the `updates:` group as its
    only dark chip…".
- `docs/configuration.md`: the `updates:` badge paragraph (≈ line 435). Add one sentence
  on the green gear inset shown while SASE is updating.

## Tests

- **New `tests/ace/tui/test_proc_gear_lanes.py`**
  - Truth table for `is_update_row` / `proc_gear_lane`: each session update proc type; a
    store-backed row (`proc_type="command"`,
    `exclusive_scopes={"plugin-update:sase-github"}`) → update; the pending placeholder
    (`proc_type="plugin.update"`) → update; `plugin-install:*` / `agent-cli-install` /
    plain `sync` → proc; a monitor row carrying an update scope → monitor; service rows
    → `None`.
  - `proc_gear_lanes`: counts, oldest-first `update_rows`, `update_labels`, and that
    inactive or dead-session rows are ignored.
  - Totals: `lanes.procs + lanes.updates == gear_eligible_count(projection)` over a
    mixed projection.
  - **Registry guard** (keeps the classifier from drifting): for every site in
    `sase.ace.tui.proc_producer_sites.PRODUCTION_PRODUCERS` whose `result_kind` is in
    `{"sase.update", "sase.update.preview", "agent-cli.update", "plugin.update", "plugin.mode-switch"}`,
    build a synthetic active row and assert `proc_gear_lane(row) == "update"`.
    Session-worker sites use `proc_type=site.proc_type`. Durable sites use
    `proc_type="command"` and `exclusive_scopes` from the site's `concurrency_keys`,
    with `{placeholders}` filled with a sample value. Assert that every other
    session-worker/durable site does not classify as `"update"`. The AST-backed
    inventory test already ties the registry to live source, so a new update producer
    can't silently land in the blue lane.
- **`tests/ace/tui/test_proc_gear_chips.py`**: `update_gear_chip(True).plain == " ⚙ "`
  with the lime-fill/dark-ink style, `update_gear_chip(False).plain == ""`, and
  `UPDATE_GEAR_HUE == UPDATES_ACCENT`.
- **`tests/test_updates_indicator.py`**:
  - running-only renders `⚙` and the group is visible;
  - running with counts renders the gear first (`" ⚙  ⬆ 3 "`);
  - the full combination (`" ⚙  ⬆ 3  core  CLI ⬆ 2 "`);
  - `running=False` output is unchanged;
  - `set_running` no-ops on an unchanged tuple;
  - `set_available` and `set_running` don't overwrite each other;
  - updating tooltip: single and multiple labels, the Procs-tab line, and the dropped
    `,U` guidance;
  - the tooltip when not updating is unchanged;
  - `on_click` runs `open_update_procs` while running and `open_updates_panel` otherwise
    (stub `app.run_action`).
- **`tests/ace/tui/test_top_bar_palette.py`**: the gear inset's dark-on-lime text meets
  the 4.5:1 AA floor, and the full-density probe
  (`test_full_density_chip_text_is_not_dimmed`) includes an updating badge body.
- **Push-point test** (next to the existing proc-indicator/observer tests; reuse their
  app harness): with one running `sase-update` session row, `#proc-indicator` is hidden
  and `#updates-indicator` shows the gear. Adding a plain `sync` row gives blue `⚙ 1`
  plus the green gear. Completing the update clears the gear.
- **`tests/ace/tui/test_procs_pane_header_counts.py`**: an update row is excluded from
  the blue chip and shown in a green `⚙ 1` chip after the orange chip. The title's plain
  text is exact. With no update rows the green chip is absent, so existing expectations
  stay the same. Add a row-label test that an update row carries the green `⚙` marker.
- **Action test** (pattern:
  `tests/ace/tui/test_log_panel_keymap.py::test_open_updates_panel_action_pushes_admin_center_on_updates`):
  `action_open_update_procs` pushes the Admin Center on `"procs"` and seeds
  `state.procs.task.identity` with the running update's identity. With no running
  update, it still opens `"procs"` and leaves the bookmark alone.
- **Visual goldens** in
  `tests/ace/tui/visual/test_ace_png_snapshots_updates_indicator.py`, following the
  existing tests there:
  - `updates_indicator_updating_120x40`: `set_available(3)` +
    `set_running(("comprehensive update",))`, waiting for
    `render().plain == "updates:  ⚙  ⬆ 3 "`.
  - `updates_indicator_updating_no_counts_120x40`: running with zero counts
    (`"updates:  ⚙ "`).

  Generate and approve them per the TUI screenshot memory: run the targeted
  `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`, then inspect every
  created or updated golden in the retained report. Also regenerate any existing Procs
  goldens that shift because of the new marker or chip, and inspect those too.

## Verification

- Read the `lint_and_test` memory and run the agent verification recipe it prescribes
  (`just check` through the sanctioned runner). Fix Symvision unused-symbol findings for
  any new public name.
- Look at both new PNG goldens yourself. The lime gear must touch the moss `⬆ 3` segment
  with no gap, sit at the left edge of the badge, and the group label must stay dim
  while the chip does not.

## Out of scope

- Keeping the gear lit through a _queued_ post-update restart. The restart toast already
  says "restart queued until N procs finish".
- Showing running state inside the Updates panel.
- Animating the gear.
- Treating installs or the automatic update check as "updating".
