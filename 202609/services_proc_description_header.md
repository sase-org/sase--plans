---
tier: tale
title: Sticky, collapsible description header for Services-tab service proc rows
goal:
  Selecting a service proc row on the Services tab shows its description in the same
  sticky, gutter-accented panel that routine and job rows use. The panel has a service
  teal theme, and `d` expands or collapses it using the shared session state. Nothing is
  duplicated in the output card, and authored text is never dropped.
size: medium
proposed_by: bbugyi200.athena.0qq
create_time: 2026-09-24 10:45:28
status: wip
---

# Sticky, collapsible description header for Services-tab service proc rows

## Goal

When a top-level **service proc** row (a `ServiceProcItem` in the Services tab's
**Service Procs** panel: Scheduler, Gateway, Telegram receiver, and so on) is selected,
show its description in the same sticky, gutter-accented description panel that routine
and job rows already get. The panel sits between the status line and the scrolling
output, stays put while output scrolls, and `d` expands or collapses it.

This reuses the existing routine/job panel (`AxeDescriptionBanner` in
`src/sase/ace/tui/widgets/axe_description_banner.py`, driven by
`src/sase/ace/tui/widgets/axe_dashboard.py`). It does not add a second widget.

## UX specification

### Layout (expanded, the session default)

```
[Scheduler] │ State: running │ Desired: running │ Enabled: enabled        ← status line (unchanged)
▌ Run SASE's background automation: routines and their scheduled jobs      ▾ d
▌
▌ Runs the scheduler orchestrator, which starts every configured routine and runs
▌ each routine's jobs on its own interval. Routines, jobs, and their recorded runs
▌ appear in the Scheduled Routines panel.
▌
▌ • Stopping it pauses all scheduled automation on this machine.
▌ • Configure the work it does under axe.routines, not here.
──────────────────────────────────────────────────────────────────────────  ← output scroll (border-top)
  SERVICE PROC
  ──────
  Scheduler
  Host: running …
```

### Collapsed (after `d`)

```
▌ Run SASE's background automation: routines and their scheduled jobs      ▸ d
```

### Visual language

- The gutter (`▌ `) uses the **service teal** already used for service rows in the
  sidebar (`_SERVICE_ACCENT_STYLE = "bold #00D7AF"` in
  `src/sase/ace/tui/widgets/_bgcmd_list_styles.py`). The summary row uses the full
  accent and body rows use `dim` + accent, exactly like routine (gold) and job (copper)
  panels. The row's own hue then ties the panel to the selected sidebar row.
- Summary and body text get a **cool palette** that matches the teal accent, where
  routine/job panels use a warm one. Start from summary `italic #AFD7D7` and body
  `#87AFAF`. Adjust these by eye against a live screenshot (see Verification). The
  summary must still read as the brightest text in the panel, and the body must stay
  legible on `$surface`.
- The `▸ d` / `▾ d` disclosure hint, bullet reflow, paragraph reflow, and height budget
  (`_description_max_lines()`: 45% of dashboard height, clamped 3–16) behave exactly as
  they do for routines and jobs.
- The overflow row for service procs reads `… +N more`, with no trailing `· e`.
  Routine/job rows keep `… +N more · e`, because `e` opens their config editor. On a
  service proc row `e` does nothing, so advertising it there would be a lie.

### Behavior rules

1. **One Services-tab description state.** `d` flips the existing session boolean
   `axe_description_expanded`, which routine and job rows also read. Collapsing on a
   service proc row also collapses routine/job panels, and the reverse is true too. The
   Services tab stays one mental model ("descriptions are expanded or collapsed"), and
   the existing `ace.axe_description_expanded` config seed covers service procs with no
   new config key.
2. **`d` acts only where a panel exists.** `action_toggle_axe_description` becomes a
   no-op unless the selected row is a service proc, routine, or job row. Today it flips
   invisible state on oneshot rows and empty selections, which then surprises the user
   on the next row. The footer and command palette follow the same rule.
3. **Missing description.** If a service proc's `description` is `None` or blank, show
   the panel with the existing dim-italic fallback `No description configured`, in the
   service palette. This matches routine/job rows and keeps `d` meaningful on every
   service proc row.
4. **Missing status.** When the selected proc is not in the current snapshot
   (`proc is None`: status unavailable or a transient refresh race), **hide** the panel.
   The output card already says "Status unavailable for this service proc", and "No
   description configured" would be false there.
5. **Oneshot rows (background commands) get no panel.** They have commands, not
   descriptions. `update_bgcmd_display` keeps hiding the panel.
6. **No duplication.** Remove the inline `" — <description>"` suffix after the proc
   label in the SERVICE PROC output card
   (`src/sase/ace/tui/widgets/_axe_dashboard_output.py`,
   `AxeOutputSection.update_service_proc`). The sticky panel is now the single home for
   the description. Keep the label line itself.
7. **The disclosure hint uses the real key.** The panel's `▸ d` / `▾ d` hint and the
   `· e` overflow suffix are hardcoded today. Pass the configured display keys for
   `toggle_axe_description` and `edit_spec` into the panel, using
   `key_display_name(self.app._keymap_registry.app.<action>)` read in the dashboard with
   defensive fallbacks to `"d"` / `"e"`. A user who rebinds either key then sees a
   truthful hint. The defaults stay `d` / `e`, so existing routine/job goldens stay
   byte-identical.

## Description grammar for service procs

Service proc descriptions adopt the documented AXE description grammar (see
`docs/axe.md` → "Description Grammar"): a summary line, a blank line, then an optional
body. The body may hold paragraphs and `-` / `*` / `•` bullet blocks, and the renderer
reflows it.

**Split ownership and reliability.** The split is owned by the Rust core
(`split_axe_description`, exposed through `sase.core.axe_chop_facade`). Service
descriptions are **not** shape-validated in the core the way routine/job descriptions
are. The core split ignores line 2 by design (`lines[2..]`), so a service description
written without the blank separator (`"Line one\nLine two\nLine three"`) would silently
lose `Line two`. To stay reliable without a cross-repo change:

- Add a thin adapter module `src/sase/service/description.py` exposing
  `split_service_description(description: str | None) -> tuple[str, str]`.
  - Return `("", "")` for `None` or blank input.
  - Normalize `\r\n` / `\r` to `\n`. If the normalized text has a non-blank line 2,
    insert one blank line after line 1 before delegating, so the whole remainder becomes
    the body and no authored text is dropped. Otherwise delegate unchanged.
  - Delegate to `split_axe_description` from `sase.core.axe_chop_facade`. Do not
    reimplement the split.
  - Memoize with `functools.lru_cache(maxsize=128)` keyed on the raw string. The core
    binding then runs once per distinct description, never per keystroke or refresh
    tick. This follows the "computed once per entity, never on a render path" rule in
    `docs/axe.md` and rule 8 of the TUI perf note.
- Both the TUI and the `sase service proc show` CLI (below) call this adapter, so they
  always agree on where the summary ends.

## Implementation steps

### 1. Adapter: `src/sase/service/description.py` (new)

Add `split_service_description` as specified above, with a short module docstring that
explains the lenient line-2 rule and why it exists. Keep it pure and import-light.

### 2. Panel: `src/sase/ace/tui/widgets/axe_description_banner.py`

- Introduce a small frozen `_DescriptionTheme` dataclass with fields `accent`,
  `summary`, `body`, and `target`. Define three module-level themes:
  - `_LUMBERJACK_THEME` and `_CHOP_THEME` keep today's exact styles (`bold #FFD700` /
    `#D7AF87` accents, `italic #D7D7AF` summary, `#AFAF87` body, `dim #B87333` target
    chip). Existing routine/job output must not change by a single cell or style.
  - `_SERVICE_THEME`: accent `bold #00D7AF`, cool summary and body as described in the
    visual language section.
- `_DescriptionBlock` carries `theme: _DescriptionTheme` in place of the lone
  `accent_style`, plus `toggle_key: str = "d"` and `overflow_key: str | None = "e"`. The
  disclosure hint becomes `f"▾ {toggle_key}"` / `f"▸ {toggle_key}"`. Measure its cell
  width with the Rich cell length, not `len`, so wide or multi-character key names still
  align. The overflow row appends `f" · {overflow_key}"` only when `overflow_key` is
  set. The fallback row keeps `_FALLBACK_STYLE`.
- `_ShownDescription` stores `theme`, `target_key`, and `overflow_key`.
- Add `show_service_proc(self, name: str, summary: str, body: str) -> None`. It mirrors
  `show_lumberjack`: `del name`, service theme, no target chip, `overflow_key=None`.
- Add `set_keys(self, *, toggle_key: str, edit_key: str) -> None`. It stores both keys
  and rerenders only on change. `edit_key` feeds `overflow_key` for routine/job shows.
- **Idle-refresh short-circuit:** in `_show`, if the panel is already displayed and the
  new `_ShownDescription` equals the current one, return without calling `_rerender()`.
  Every Services auto-refresh calls `_refresh_axe_display()`, and repainting an
  identical panel with `refresh(layout=True)` on every tick is wasted layout work (TUI
  perf rule 14). `set_expanded` and `set_max_lines` already short-circuit.
- Update the module and class docstrings to "selected service proc, routine, or job
  description".

### 3. Dashboard: `src/sase/ace/tui/widgets/axe_dashboard.py`

- Add a private `_description_keys()` helper that returns
  `(toggle_display, edit_display)` from the app keymap registry via `key_display_name`,
  falling back to `("d", "e")` on any exception, the same way `_description_expanded()`
  tolerates test doubles. Call `banner.set_keys(...)` before each `show_*` in
  `update_lumberjack_overview`, `update_chop_run_display`, and
  `update_service_proc_display`.
- In `update_service_proc_display`, replace `self._hide_description_banner()` with:
  - `proc is None`: hide the panel (rule 4).
  - Otherwise: `set_expanded(self._description_expanded())`,
    `set_max_lines(self._description_max_lines())`, then
    `summary, body = split_service_description(proc.description)` and
    `banner.show_service_proc(name, summary, body)`.

### 4. Output card: `src/sase/ace/tui/widgets/_axe_dashboard_output.py`

Remove the two lines that append `f" — {proc.description}"` after the label (rule 6).
This file is near the `toobig` threshold, so do not add code here.

### 5. Toggle action: `src/sase/ace/tui/actions/axe.py`

- Add a small helper, for example `_axe_description_row_selected()`. It returns whether
  `self._axe_items[self.current_idx]` is a `ServiceProcItem`, `LumberjackItem`, or
  `ChopItem`, with a bounds check. Put it where both the action and the footer code can
  use it (for example in `axe_display/_render_dashboard.py` or a nearby mixin). `axe.py`
  is already over 500 lines, so prefer not to grow it much.
- `action_toggle_axe_description` returns early unless on the Services tab **and**
  `_axe_description_row_selected()` is true (rule 2). The rest of the action (flip,
  `refresh_description_banner`, cache-only `_refresh_axe_display`) stays unchanged.

### 6. Footer: `src/sase/ace/tui/widgets/_keybinding_bindings_axe.py` and `axe_display/_render_dashboard.py`

- In `_compute_axe_bindings`, keep `e edit config` gated on `config_row_selected`. Show
  the `d` `collapse desc` / `expand desc` binding when
  `config_row_selected or service_selected`. Update the docstring.
- In `_render_dashboard.py`, keep passing `config_row_selected` as today. Only the
  binding computation changes, so the `update_axe_bindings` signature stays as it is.

### 7. Command palette: `src/sase/ace/tui/commands/_availability_axe.py` and `commands/_app_metadata_actions.py`

- `app.toggle_axe_description` is available when
  `_is_lumberjack(item) or _is_chop(item) or _is_service_proc(item)`.
- Add palette keywords `"service description"` and `"proc description"` to the
  `toggle_axe_description` metadata entry. Keep the title unchanged unless a test pins
  it; if you rename it to "Toggle Services description", update the pinned tests.

### 8. Help modal: `src/sase/ace/tui/modals/help_modal/axe_bindings.py`

Change the `toggle_axe_description` label to
`"Expand / collapse service, routine, or job description"`, or a shorter wording that
fits the column. Check the help-guide golden (`help_guide_axe_120x40`) afterward.

### 9. CLI: `src/sase/main/service_handler.py` → `handle_service_proc_show`

Replace `console.print(f"  description: {proc.description}")` with
`split_service_description`. Print `  description: <summary>`. When a body exists,
follow it with `  details:` and each body line indented four spaces, mirroring
`sase axe routine list -v` (`src/sase/axe/cli.py` around the `details:` label). Keep
`--json` output unchanged (raw `description`). `service_handler.py` is over 600 lines,
so put the rendering in a tiny helper and keep net growth small. Use `rich.text.Text` so
bracketed text in descriptions is never parsed as Rich markup.

### 10. Builtin descriptions: `src/sase/default_config.yml` (`service.procs`)

Rewrite the two builtin descriptions as `|-` block scalars that follow the grammar and
the "Authoring Style Guide" in `docs/axe.md`: summary ≤ 80 characters, present tense,
sentence case, no trailing period, and a short body. Verify every claim against
`docs/axe.md` and `docs/mobile_gateway.md` before committing. Drop any sentence you
cannot confirm. Suggested starting points:

```yaml
scheduler:
  builtin: scheduler
  description: |-
    Run SASE's background automation: routines and their scheduled jobs

    Runs the scheduler orchestrator, which starts every configured routine and runs each routine's
    jobs on its own interval. Routines, jobs, and their recorded runs appear in the Scheduled
    Routines panel.

    - Stopping it pauses all scheduled automation on this machine; `sase scheduler start` resumes it.
    - Configure the work it does under `axe.routines`, not here.
gateway:
  builtin: gateway
  description: |-
    Serve the mobile gateway HTTP API for remote agent dispatch

    Serves the HTTP API that paired mobile clients call to list, launch, and manage agents and to
    receive notification events. It exposes fixed product operations only, never a generic file,
    shell, or RPC surface.

    - Disabled by default; enable it on this machine with `sase service proc enable gateway`.
    - Pair a phone with `sase mobile gateway pair` once it is running.
```

Keep the existing `enabled: false` and the comment on `gateway`. Check that no test pins
the old one-line strings; grep `tests/` for `"SASE's background automation"` and
`"Mobile gateway HTTP API"` and update any hits.

### 11. Schema: `src/sase/config/sase.schema.json`

- `serviceProc.properties.description.description`: document the grammar. For example:
  "Summary line, then a blank line and an optional body. Shown in the Services tab
  description panel (`d` expands or collapses it) and `sase service proc show`."
- `ace.axe_description_expanded.description`: say it seeds the Services-tab description
  panel for service procs, routines, and jobs.

Run the schema tests (`tests/test_config_schema.py`,
`tests/test_config_schema_validity.py`).

### 12. Docs

- `docs/ace.md` → "Description Panel": the panel covers the selected **service proc**,
  routine, or job. Document the service teal gutter, the shared session state, the
  service-proc fallback and hidden-on-unavailable behavior, and the `… +N more` overflow
  without `· e` on service rows. Oneshot rows get no panel.
- `docs/ace.md` → Services tab intro (the paragraph that starts "The right-hand panel
  shows the selected proc's …"): mention that the description sits in the sticky panel
  above it.
- `docs/ace.md` → Axe Commands table `d` row: "Expand / collapse the description panel
  for the selected service proc, routine, or job".
- `docs/configuration.md`: update `ace.axe_description_expanded` and
  `toggle_axe_description` wording, and document `service.procs.<name>.description` with
  the summary/blank/body grammar. Link to `axe.md#description-grammar` and mention the
  lenient line-2 handling.
- `docs/axe.md` → "Description Grammar": add one sentence saying service proc
  descriptions use the same grammar for display, are not shape-validated, and have their
  body start at line 2 when the blank separator is missing.
- Do **not** hand-edit `CHANGELOG.md`. It is generated by release-please from
  conventional commit subjects and validated by `tools/validate_changelog`.

## Tests

### Unit

- `tests/service/test_description.py` (new; `tests/service/` already holds the
  `src/sase/service` tests):
  - `None`, `""`, and whitespace-only input give `("", "")`.
  - A single line gives `(summary, "")`.
  - Summary, blank line, and body split normally, with trailing blank lines trimmed.
  - A non-blank line 2 keeps **every** line: `"a\nb\nc"` → `("a", "b\nc")`, and `"a\nb"`
    → `("a", "b")`.
  - CRLF input splits the same as LF.
  - Memoization: a second call with the same string does not re-invoke the binding
    (patch the facade function and count calls).
- `tests/ace/tui/test_axe_description_banner.py`:
  - `show_service_proc` renders the teal-themed gutter, summary, blank row, and reflowed
    body, including bullets.
  - Collapsed mode is exactly one line with `▸ d`.
  - The service overflow row reads `… +N more` with **no** `· e`, while the chop
    overflow row still ends with `· e`.
  - `set_keys(toggle_key="D", edit_key="E")` changes the hint to `▾ D` and the chop
    overflow suffix to `· E`.
  - The fallback text appears for a blank summary.
  - Existing routine/job rendering assertions still pass unchanged (theme refactor
    regression).
  - An identical second `show_*` call does not trigger a rerender (spy on `_rerender`),
    while a changed description does.
- `tests/ace/tui/test_axe_navigation.py`:
  - Extend the fake dashboard so service proc selection records its banner state.
    Selecting a service proc row shows the panel, a bgcmd row hides it, and a service
    proc missing from the snapshot hides it.
  - Update `_ToggleProbe` with `_axe_items` / `current_idx`. `d` toggles on service
    proc, routine, and job rows, and is a no-op (no flip, no repaint) on bgcmd rows and
    empty selection.
- Dashboard-level test (for example in `tests/ace/tui/widgets/`): call
  `AxeDashboard.update_service_proc_display` with a proc whose description is
  multi-line. The mounted `#axe-description-banner` is displayed with the split summary
  and body. The SERVICE PROC output text no longer contains `" — "` plus the
  description.
- `tests/test_command_availability_axe.py`: `app.toggle_axe_description` is available on
  service proc rows and unavailable on bgcmd rows.
- Footer binding test (next to the existing `_compute_axe_bindings` coverage):
  `service_selected=True` yields `collapse desc` / `expand desc` but not `edit config`.
- CLI test for `handle_service_proc_show`: a multi-line description prints the summary
  on the `description:` line and the indented body under `details:`. A one-line
  description prints no `details:`. Bracketed text is printed literally.

### Visual (PNG goldens)

- Give the Services-panel fixture procs realistic descriptions in
  `tests/ace/tui/visual/_ace_axe_png_snapshot_tree_fixtures.py` (`_services_panels_proc`
  gains a `description` kwarg). Scheduler gets a multi-line summary/body with one
  paragraph and two bullets, Telegram gets a one-liner, and `web` keeps none so the
  fallback path stays covered.
- Add to `tests/ace/tui/visual/test_ace_png_snapshots_services_panels.py`, or a new
  `test_ace_png_snapshots_services_descriptions.py` if that file would get crowded:
  - `services_proc_description_120x40`: Scheduler selected, expanded.
  - `services_proc_description_collapsed_120x40`: same after pressing `d`.
  - `services_proc_description_missing_120x40`: the description-less proc selected,
    showing the fallback.
- Expected golden **updates**: `services_panels_120x40`,
  `services_panels_empty_routines_120x40`, and `services_panels_narrow_70x36` (Scheduler
  is selected by default, so the panel now appears), plus `help_guide_axe_120x40` if its
  label changed. Other Services-tab goldens (`launch_context_bar_services_120x40`,
  `link_rail_axe_*`) may shift if their fixtures select a service proc. Inspect every
  update.
- **All existing `axe_*` routine/job description goldens must be unchanged.** That is
  the regression proof that the theme/keys refactor is style-identical. If any changes,
  treat it as a bug in step 2, not a golden to accept.

## Verification

1. `just fix` (or at least `just fmt`), then `sase tool run check`.
2. Visual goldens: run `just fix-tui-screenshots` with targeted selectors after `--` for
   the services/axe/help visual tests, through `/sase_monitor` if it may exceed the turn
   budget. Inspect the retained report and **every** created or updated golden: the
   panel must sit between the status line and the output border, the teal gutter must
   run unbroken on every row, and the disclosure hint must be right-aligned on the
   summary row.
3. Live look: `sase screenshot -o /tmp/services_desc.png -- -t services` (Scheduler is
   selected by default), then capture again after pressing `d` (`--keep` +
   `tmux send-keys`, per the screenshot note). Read the PNGs and tune the service
   summary/body colors if they clash with the teal accent or the `$surface` background.
4. Manually confirm: `j`/`k` from Scheduler to a routine and back keeps the panel stable
   and swaps the accent, `d` on a oneshot row does nothing, and the footer shows
   `d collapse desc` on service proc rows.

## Non-goals

- No new config key or per-kind expanded state. One Services-tab description state is
  intentional.
- No core (sase-core) change. Core-side shape diagnostics for service proc descriptions
  (the routine/job `description_*` codes) would be a separate follow-up if the lenient
  adapter ever proves insufficient.
- No panel for oneshot/background-command rows.
- No changes to plugin-declared descriptions in other repos (for example the Telegram
  receiver). They render as their existing one-line summaries.
