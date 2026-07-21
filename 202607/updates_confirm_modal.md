---
tier: tale
title: Readable, scrollable Updates-tab confirm modal
goal: 'The y/n confirm popup opened by `u` on the SASE Admin Center Updates tab reads
  at a glance (component-first "what will update" list, de-noised commands, wider
  panel that fits one page), and its main preview scrolls with ctrl+d/ctrl+u when
  it overflows.

  '
create_time: 2026-07-21 07:41:24
status: done
prompt: 202607/prompts/updates_confirm_modal.md
---

# Plan: Readable, scrollable Updates-tab confirm modal

## Problem

Pressing `u` on the Updates tab of the SASE Admin Center opens `PluginActionConfirmModal`
(`src/sase/ace/tui/modals/plugin_action_confirm_modal.py`) with the comprehensive-update preview (see
`.sase/home/tmp/screenshots/20260721_073124.png`). Three problems:

1. **Unreadable wall of text.** The panel is 76 columns wide and every command is a full shlex-joined argv with absolute
   paths (`/home/bryan/projects/github/sase-org/...`), so each command wraps 3–4 lines. Sections, commands, and skipped
   items blur together.
2. **No scrolling.** The main preview is a plain `Static` inside a `max-height: 80%` container. The existing
   `ctrl+d`/`ctrl+u` bindings only scroll the optional incoming-commits pane (`#plugin-action-commits`), which the
   comprehensive flow does not even mount — overflow content is simply unreachable.
3. **Unclear what will actually update.** The answer ("sase + sase-core via 12 incoming commits; bugyi-chops upgrade;
   sase-github/sase-telegram already current") is buried in raw argv and a trailing "Manual / skipped" list.

The same modal is shared by install, batch-install, uninstall, single-plugin update, mode-switch, agent-CLI update,
managed SASE update, and dev SASE update flows, so the work must improve all flows uniformly without breaking any of
them.

## Approach

All work is presentation-only Textual/Python (no Rust core boundary crossing). The preview data is already built
off-thread; rendering changes stay synchronous-light per the TUI perf rules (scroll-hint syncing copies the existing
`call_after_refresh` pattern used for the commits pane).

### 1. Scrollable main preview (`plugin_action_confirm_modal.py` + `styles.tcss`)

- Wrap the `#plugin-action-preview` `Static` in a new `VerticalScroll` (`#plugin-action-preview-scroll`).
- Generalize the `ctrl+d`/`ctrl+u` actions: scroll the preview scroll when it has overflow, otherwise fall back to the
  incoming-commits scroll (preserves today's behavior for the dev-update flow, where the preview is short and the
  commits box is the long part). Keep half-page steps with `animate=False`.
- Mirror the existing `_sync_commits_scroll_hint` mechanism for the preview: when the preview overflows, add a
  `has-scrollable-preview` class (CSS gives the container a bounded height like `has-scrollable-commits` does) and
  surface a `ctrl+d/u scroll` hint (preview-scroll border subtitle, matching the commits pane's hint style).

### 2. Larger, better-proportioned panel (`styles.tcss`)

- Keep the default 76-column width for the simple flows, but widen the panel when the active variant carries preview
  _sections_ (the comprehensive flow): add a modal CSS class (e.g. `has-sections`, set in `__init__` from the variants)
  that bumps width to ~110 (`max-width: 95%`) and allows `max-height: 90%`. The extra width alone removes most command
  wrapping.

### 3. Component-first "what will update" content

Extend the preview data model and builders so each section leads with a scannable component list and demotes commands to
secondary detail:

- **`PluginActionPreviewSection`** gains a structured `components` field: small frozen dataclass rows of (name, detail,
  state) where state is one of `update` / `current` / `skipped`. The modal renders them as aligned rows with a state
  glyph and color: `↑` (cyan) will update, `✓` (dim green) already current, `•` (yellow) manual/skipped.
- **`sase_preview_section`** (`plugins_browser_comprehensive_update_preview.py`) builds components from the
  `DevUpdatePlan`:
  - one row per actionable git root: short repo name (`_short_root`), upstream ref, and "N incoming commits" from
    `behind`;
  - one row per actionable package: `name  current → latest` when versions are known;
  - one row per reconcile step using its short `label` (not the argv);
  - skipped packages as `skipped` rows with their reasons.
- **`provider_preview_section`** builds one component per provider entry: `display_name  installed → latest` for
  runnable entries, and `skipped` rows carrying the skip reason for the rest (dropped candidates and provider errors
  stay as skipped rows too).
- The managed/dev SASE update and agent-CLI flows keep working unchanged: the new field defaults to empty and existing
  `summary`/`details`/`items`/`skipped` rendering remains.

### 4. De-noised command rendering (keep exact argv — epic decision D5)

The confirmation must still show the exact commands, but readably:

- Display-shorten the user's home directory prefix to `~` (display-only transformation in the modal's command renderer,
  shared by section commands and the top-level `Would run` line).
- Render each command as its own block: dim `$` prefix, cyan command text, hanging indent on wrapped continuation lines
  (e.g. via `Padding`/indented `Text`) so distinct commands are visually separable; render `fallback:` repair commands
  as an indented dim sub-line of their parent step.
- Group commands under a dim "commands" sub-heading _after_ the component list within each section.

### 5. Visual polish

- Replace bare bold section titles with full-width rules (`rich.rule.Rule`) carrying the section title plus counts (e.g.
  `SASE, core & plugins · 1 checkout · 4 steps`, `Agent CLIs · 1 command · 2 skipped`), so sections separate cleanly.
- Tighten the intro and per-variant summary spacing; keep the bold one-line summary.
- Keep the border subtitle keymap legend (`y confirm · n/esc cancel [· g source]`).

## Files expected to change

- `src/sase/ace/tui/modals/plugin_action_confirm_modal.py` — layout, scrolling, hint syncing, component/command
  rendering, home-path display shortening, `has-sections` class.
- `src/sase/ace/tui/modals/plugins_browser_comprehensive_update_preview.py` — component rows for both legs; command
  lists unchanged in content.
- `src/sase/ace/tui/modals/plugins_browser_dev_update.py` — reuse/expose `_short_root` and any shared formatting
  helpers.
- `src/sase/ace/tui/styles.tcss` — preview-scroll styles, wide/`has-sections` sizing, `has-scrollable-preview` height
  rule.
- Tests (see below).

## Testing

- Unit tests in `tests/ace/tui/test_plugin_action_confirm_modal.py`: ctrl+d/u scrolls the preview when it overflows;
  falls back to the commits pane when the preview fits; the `has-scrollable-preview` hint appears only on overflow;
  component rows render with the right glyph/state; home prefix renders as `~`.
- `tests/ace/tui/test_plugins_browser_pane_comprehensive_update.py`: assert the new component rows
  (repo/package/provider entries with versions and skip reasons) for both section builders; existing command-content
  assertions stay valid.
- PNG visual snapshots (`tests/ace/tui/visual/test_ace_png_snapshots_config_center_plugin_actions.py`): add a
  comprehensive-update confirm snapshot (long preview exercising the wide panel and scroll hint) and regenerate the
  existing install/update/uninstall confirm goldens that the restyling intentionally changes (`just test-visual` with
  `--sase-update-visual-snapshots` to accept, inspecting `.pytest_cache/sase-visual/` diffs first).
- `just install` then `just check` before finishing.

## Risks / constraints

- The modal is shared by eight flows; the new section/component fields must be optional and default-empty so untouched
  callers only pick up the intentional restyling. Visual goldens for those flows will change and must be reviewed, not
  blindly accepted.
- Keymap conflict: none — `ctrl+d`/`ctrl+u` are already bound in this modal; their scope just widens. The footer/help
  conventions are unaffected (modal-local keys live in the modal's border subtitle).
- Per TUI perf rules, no I/O or planning work moves onto the render path; scroll-hint reflow uses `call_after_refresh`
  exactly like the commits pane already does.
