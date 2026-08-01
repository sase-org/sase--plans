---
tier: tale
title: Number the Statistics sub-tabs and add 0<N> selection keymaps
goal:
  Every Admin Center Statistics sub-tab is labeled with its 1-based number, and pressing the configurable prefix key `0`
  followed by that digit activates the corresponding view without disturbing the Admin Center's own numbered tab keys.
create_time: 2026-07-29 14:06:31
status: done
---

- **PROMPT:** [prompts/202607/statistics_numbered_subtabs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/statistics_numbered_subtabs.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-au.5.w1--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-au.5.w1.md#member-code)
  - [bbugyi200.athena.sase-au.5.w1--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-au.5.w1.md#member-plan)
- **COMMITS:**
  - [216d027](https://github.com/sase-org/sase/commit/216d027d8ba439f6156b45557750e292c029311b) — feat(ace): add numbered Statistics subtabs

# Plan: Numbered Statistics sub-tabs with `0<N>` selection keymaps

## Goal

The SASE Admin Center's **Statistics** tab (`ConfigCenterModal` tab `4`) hosts nine sub-tabs ("views"). Today they can
only be reached with `[` / `]` cycling or a mouse click. This plan:

1. Labels each sub-tab with its 1-based number (`1 Overview`, `2 Runs`, … `9 Plans & Questions`) in the tab strip and in
   the contextual help modal.
2. Adds a new configurable Statistics-pane keymap `select_view` (default `0`) that arms a one-shot digit prefix, so
   `01`…`09` jump directly to the matching sub-tab.

## Background — what already exists

- `src/sase/ace/tui/modals/statistics_pane_data.py:31` — `VIEW_ORDER` is the canonical nine-view order:
  `overview, runs, runners, projects, providers, runtime, activity, xprompts, plans_questions`. `VIEW_LABELS` and
  `VIEW_COMPACT_LABELS` sit next to it.
- `src/sase/ace/tui/modals/statistics_pane.py:52` — `_VIEW_TABS` builds `PanelTab` entries;
  `src/sase/ace/tui/modals/statistics_pane.py:158` mounts the `PanelTabStrip` as `#statistics-views`.
- `src/sase/ace/tui/widgets/panel_tab_strip.py:40` — `PanelTabStrip` **already supports `show_numbers=True`**; the Admin
  Center's own tab strip (`config_center_modal.py:153`) and the Artifacts strip (`widgets/artifacts/view.py:75`) already
  use it. The Statistics strip does not.
- `src/sase/ace/tui/keymaps/app_keymaps.py:174` — `StatisticsPaneKeymaps` holds the fourteen focused-pane actions;
  `src/sase/ace/tui/keymaps/metadata.py:151` — `_STATISTICS_BINDING_META` gives each one its help description;
  `src/sase/ace/tui/keymaps/bindings.py:47` — `build_statistics_bindings` turns them into instance-local `Binding`s;
  `src/sase/ace/tui/keymaps/scopes.py:25` — `load_statistics_keymaps` validates overrides against
  `src/sase/default_config.yml:148`.
- `src/sase/ace/tui/modals/config_center_modal.py:86` — the Admin Center screen binds `0`–`9` to `focus_center_tab(N)`.
  `1`–`7` open working tabs; `0`, `8`, and `9` are swallowed no-ops.

## Key mechanics that make this work (verified against Textual 8.0.1)

`App.on_event` (`textual/app.py:4047`) first resolves **priority** bindings across the whole chain, then forwards the
key to the focused widget. Non-priority bindings are only resolved after the event bubbles all the way back to the App
(`App._on_key` → `_check_bindings`, `textual/app.py:4267`), and `Screen._binding_chain` (`textual/screen.py:404`) is
built from `focused.ancestors_with_self`, i.e. **focused-widget-first**. Two consequences the implementation relies on:

- A `StatisticsPane.on_key` handler runs **before** `ConfigCenterModal`'s non-priority `1`–`9` bindings. That is the
  only reliable way to intercept the _second_ digit of the `0<N>` sequence.
- The prefix key itself can stay an ordinary pane `Binding`, because the focused pane precedes the modal screen in the
  binding chain — the pane's `0` wins over the modal's `0` → `focus_center_tab(0)` (already a no-op).

`ConfigCenterModal`'s `tab` / `shift+tab` are the only priority bindings in play, so they are unaffected.

## Part 1 — Number the sub-tabs

### The width problem this must solve

Pane content width is `floor(terminal_width * 0.95) - 2 (thick border) - 4 (padding 1 2)`
(`src/sase/ace/tui/styles.tcss:5184`): **108 columns** at a 120-column terminal, **79** at 90 columns.

Measured strip widths for the nine Statistics tabs:

| Tier                                           | Without numbers | With numbers |
| ---------------------------------------------- | --------------- | ------------ |
| Full labels, `" │ "` separator, `" N "` prefix | 118             | **136**      |
| `VIEW_COMPACT_LABELS`, `"│"` separator, `"N "` | 74              | **92**       |

`_VIEWS_COMPACT_BELOW_WIDTH = 108` (`statistics_pane.py:51`) means a 120-column terminal renders the _full_ 118-column
strip into 108 columns. That is a **pre-existing clipping bug**: the committed golden
`tests/ace/tui/visual/snapshots/png/config_center_statistics_overview_120x40.png` shows the last tab truncated to
`Plans &`. `#statistics-views` (`styles.tcss:5375`) is `height: 1` with no `text-wrap`/`text-overflow` rule, unlike
`#config-center-tabs`, so the overflow silently disappears. Numbering makes this strictly worse (136 into 108), and the
numbered compact strip (92) also overflows the 79-column narrow pane exercised by
`config_center_statistics_narrow_90x30.png`.

### Chosen approach: a third "micro" tier on `PanelTabStrip`

Add an opt-in third responsive tier so every supported width shows all nine numbers:

- `PanelTab` gains `micro_label: str | None = None`.
- `PanelTabStrip.__init__` gains `micro_below: int | None = None` (and a `micro_separator: str = "│"`). Replace the
  internal `_compact: bool` with a tier value (`"full" | "compact" | "micro"`) recomputed in `on_resize`; check
  `micro_below` before `compact_below`. With `micro_below=None` the behavior is byte-identical to today, so the Admin
  Center and Artifacts strips and `tests/ace/tui/test_config_center_tabs.py` are unaffected.
- `_build_content` selects separator, number format, label source, and suffix from the tier. `_tab_ranges` and the
  centered click hit-testing keep working unchanged.

Statistics then mounts the strip with `show_numbers=True`, `compact_below=136` (was `108`), and `micro_below=92`, and a
new `VIEW_MICRO_LABELS` map in `statistics_pane_data.py`:

`Ovr, Runs, Rnrs, Proj, Prov, Rtm, Act, XP, P&Q` → micro strip width ≈ 56 columns, comfortably inside the 79-column
narrow pane.

Resulting tiers: ≥136 columns → full numbered labels; 92–135 → compact numbered (a 120-column terminal, which also
**fixes** today's clipped golden); <92 → micro numbered.

Also add `text-wrap: nowrap; text-overflow: ellipsis;` to `StatisticsPane #statistics-views` in `styles.tcss` so any
future overflow degrades visibly instead of dropping tabs silently.

_Alternative considered and rejected:_ keeping two tiers and shrinking `VIEW_COMPACT_LABELS` from 66 to ≤53 characters.
It requires the same abbreviations but loses the readable labels at 108 columns, where they still fit.

### Help modal

`StatisticsHelpModal._views_text` (`statistics_help_modal.py:131`) must prefix each row with its 1-based number so the
help mirrors the strip (e.g. `● 1 Overview — Totals and trends…`).

## Part 2 — The `0<N>` keymaps

### New configurable action

Add `select_view` (default `"0"`) everywhere a Statistics action is declared:

- `src/sase/ace/tui/keymaps/app_keymaps.py` — `StatisticsPaneKeymaps.select_view: str = "0"`.
- `src/sase/ace/tui/keymaps/metadata.py` — `("select_view", "Select View by Number")` in `_STATISTICS_BINDING_META`,
  placed immediately after `next_view` so the help "Controls" section groups the view controls together. This tuple's
  order is asserted by tests, so position matters.
- `src/sase/default_config.yml` — `select_view: "0"` under `ace.keymaps.statistics` (required: `load_statistics_keymaps`
  raises if a dataclass field has no bundled default).
- `src/sase/config/sase.schema.json:745` — a `select_view` property (the object is `additionalProperties: false`).

`"0"` passes `is_valid_key` (single alphanumeric) and collides with no other Statistics action.

### Pane behavior (`statistics_pane.py`)

- New `self._pending_view_select: bool = False` in `__init__`.
- `action_select_view()` — arm the prefix and repaint the hints.
- New `on_key(self, event: Key)`:
  - Return immediately if `#statistics-custom-range` has focus (same guard as `_scroll_body`, `statistics_pane.py:413`)
    so digits typed into the range editor are never stolen.
  - Return immediately if not armed — normal binding dispatch is untouched.
  - While armed, `event.stop()` + `event.prevent_default()` and:
    - the configured prefix key → stay armed (re-arm);
    - `"1"`–`"9"` → disarm and, when the digit is within `len(VIEW_ORDER)`, `self._set_view(VIEW_ORDER[digit - 1])`;
      out-of-range digits are swallowed with no view change (mirrors `action_focus_center_tab`). All nine digits map to
      a view today; the bound check is future-proofing for a shorter `VIEW_ORDER`.
    - any other key → disarm, repaint hints, and **let the event continue** (do not stop it), so `q` / `Esc` still close
      the Admin Center and `g` / `G` still reach `ConfigCenterModal.on_key`.
- Bare digits with no prefix keep switching Admin Center tabs — unchanged.

### Discoverability

- `_hints_text` (`statistics_pane_rendering.py:364`): render the leading segment as `[ / ] · 0N views` using
  `_effective_key("prev_view")`, `next_view`, and `select_view`. This costs ~5 columns and still fits the 79-column
  narrow pane.
- While armed, replace the hints line with a pending indicator (e.g. `0… press 1-9 to select a view`, accent-styled),
  restored when the sequence resolves or is cancelled.
- `StatisticsHelpModal._control_value` (`statistics_help_modal.py:167`): return
  `"press <key> then 1-9; current view: <Label>"` for `select_view`. No `_controls_text` filtering change is needed —
  the action always applies.

## Files to change

**Source**

- `src/sase/ace/tui/widgets/panel_tab_strip.py` — micro tier (`PanelTab.micro_label`, `micro_below`, tier state).
- `src/sase/ace/tui/modals/statistics_pane_data.py` — `VIEW_MICRO_LABELS` (+ `__all__`).
- `src/sase/ace/tui/modals/statistics_pane.py` — numbered/tiered strip, `_VIEWS_COMPACT_BELOW_WIDTH` → 136, new micro
  threshold, `_pending_view_select`, `action_select_view`, `on_key`.
- `src/sase/ace/tui/modals/statistics_pane_rendering.py` — hints line and armed-state indicator.
- `src/sase/ace/tui/modals/statistics_help_modal.py` — numbered view list, `select_view` control value.
- `src/sase/ace/tui/keymaps/app_keymaps.py`, `src/sase/ace/tui/keymaps/metadata.py`.
- `src/sase/ace/tui/styles.tcss` — `#statistics-views` nowrap/ellipsis.
- `src/sase/default_config.yml`, `src/sase/config/sase.schema.json`.

**Docs** (`src/sase/ace/CLAUDE.md` requires the help popup and docs stay in sync with `sase ace` options)

- `docs/configuration.md` — Statistics keymap table row for `select_view`; the `ace.keymaps.statistics` sample YAML
  (~line 496); the "Statistics tab" prose (~line 209) which currently says "eight views" and omits **XPrompts** — make
  it nine numbered views and document `0<N>`.
- `docs/ace.md` — "Remapping Statistics Pane Keys" (~line 2683): add `select_view` to the sample YAML and describe the
  prefix behavior.
- `docs/telemetry.md` — "Admin Center Statistics tab" (~line 151): number the view table, add the missing **XPrompts**
  row so all nine appear in canonical order, and add a `0` + digit row to the default-keys table.

## Tests

**Update**

- `tests/ace/tui/test_statistics_pane_bindings.py` — add `select_view` to the remap dict and to the expected
  `statistics_help_bindings` list (order follows `_STATISTICS_BINDING_META`), and update the exact hints assertion at
  line 74.
- `tests/test_keymaps_defaults.py::test_default_config_covers_all_statistics_keymaps` — add `"select_view": "0"`.
- `tests/test_keymaps_registry_loading.py` — assert `reg.statistics.select_view == "0"`.

**Add**

- Numbered-strip rendering: nine numbers appear in order; assert the rendered plain width at each tier is within the
  pane width for 136 / 108 / 79-column panes (this is the regression guard for the clipping bug).
- `tests/ace/tui/test_config_center_tabs.py` — `micro_below=None` leaves rendering unchanged; a micro-tier strip renders
  numbers plus micro labels.
- `0<N>` dispatch (new `tests/ace/tui/test_statistics_view_number_select.py`, using
  `tests/ace/tui/_statistics_pane_helpers.py::_open_statistics`):
  - `0` then `3` selects `runners` and leaves `modal._active_tab == "statistics"`;
  - `0` then `9` selects `plans_questions`;
  - a bare `3` still switches the Admin Center tab away from Statistics (regression guard for the screen bindings);
  - `0` `0` `2` selects `runs` (re-arm);
  - `0` then `q` closes the Admin Center (disarm passes the key through);
  - `0` typed while `#statistics-custom-range` has focus is inserted into the input and does not arm;
  - a non-default configured prefix (e.g. `f4`) arms the same way.
- Help modal: `_views_text` shows `1`…`9`, and the Controls section lists the `select_view` key and description.

**Visual**

All eleven `config_center_statistics_*` goldens change — the ten pane goldens from numbering, the tier change at 120
columns, and the newly untruncated last tab, plus `config_center_statistics_help_120x40` from the numbered view list.
Regenerate with `just test-visual --sase-update-visual-snapshots`, then **visually inspect each PNG** to confirm the
strip is complete and untruncated at 120x40 and 90x30 before committing the goldens. Only the Statistics goldens should
change; if any other golden moves, the `PanelTabStrip` default must have regressed.

## Verification

```bash
just install     # ephemeral workspace: dependencies may be stale
just check
just test-visual
```

## Out of scope

- The Admin Center's own `1`–`7` top-level tab keys.
- Numbering the Projects, Artifacts, or Updates panes' sub-tabs.
- Any change to `PanelTabStrip` behavior for strips that do not opt into `micro_below`.
