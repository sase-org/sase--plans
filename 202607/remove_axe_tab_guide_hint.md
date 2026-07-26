---
tier: tale
title: Remove the persistent AXE tab-guide hint
goal: Remove the `,?  ]  tab guide` hint from the AXE tab header without changing
  AXE status information or Help/Guide access.
create_time: 2026-07-26 11:01:51
status: done
---

- **PROMPT:** [202607/prompts/remove_axe_tab_guide_hint.md](prompts/remove_axe_tab_guide_hint.md)

# Remove the persistent AXE tab-guide hint

## Context

The AXE tab's `AxeInfoPanel` currently appends a cyan Help/Guide keycap and the label `tab guide` to every header
rendering:

- startup loading (`AXE …`);
- daemon/countdown overview;
- lumberjack selection;
- chop-run selection; and
- background-command selection.

The hint is produced by `src/sase/ace/tui/widgets/axe_info_panel.py::_append_help_guide_hint()`. It was added as a
discoverability affordance for the tab guide and later changed to show the current leader-mode Help chord followed by
`]`. The guide itself now lives in the Help panel and remains reachable through the existing `show_help` command and
Help panel Guide tab. This change removes only the persistent header affordance; it must not remove or change the Help
action, Guide panel, keymap defaults, onboarding content, or footer bindings.

Because the header hint is the only reason `AxeInfoPanel` owns a `KeymapRegistry`, mount-time setup also contains
AXE-specific registry wiring that becomes dead code when the hint is removed.

## Implementation

1. Simplify `src/sase/ace/tui/widgets/axe_info_panel.py`.
   - Stop appending the Help/Guide hint from both the loading and normal render paths.
   - Delete `_append_help_guide_hint()`.
   - Remove the panel's `_registry` state, `set_keymap_registry()` method, and now-unused keymap imports.
   - Preserve all existing AXE header content and styling for the loading ellipsis, selected background-command project,
     lumberjack/chop identity and run position, and auto-refresh countdown. Do not collapse or resize the two-row panel
     as part of this change.

2. Remove obsolete setup in `src/sase/ace/tui/actions/_startup_mount.py`.
   - Stop importing `AxeInfoPanel` solely for keymap wiring in `on_mount()`.
   - Remove the guarded lookup/call that injects the registry into the AXE info panel.
   - Retain the separate `AxeInfoPanel` import and lookup in `_apply_startup_loading_state()`, which still drives the
     loading indicator.

3. Update focused rendering coverage in `tests/ace/tui/test_startup_loading_indicators.py`.
   - Remove the keymap fixture/import and the test for configured Help-key rendering, since the header will no longer
     render any Help key.
   - Change the loading-state test to assert the retained `AXE …` copy and the absence of `tab guide`.
   - Change the normal countdown test to assert the retained `(auto-refresh in 5s)` copy and the absence of `tab guide`.
   - Keep the existing background-command display-name and loading short-circuit coverage.

4. Refresh the AXE-tab PNG goldens exercised by `tests/ace/tui/visual/test_ace_png_snapshots_axe.py`.
   - Regenerate only the snapshots whose visible AXE header changes.
   - Inspect the generated expected/actual/diff artifacts and representative wide and constrained-width images. The
     visual delta must be limited to removal of the keycap/`tab guide` text; sidebar, dashboard, footer, borders,
     countdown, and selected-item status must remain unchanged.
   - Do not update Help-panel guide snapshots unless an actual test failure demonstrates that the underlying header is
     visible in that snapshot.

## Verification

Run installation first because this is an ephemeral SASE workspace:

```bash
just install
```

Run the focused non-visual rendering tests:

```bash
just test -- tests/ace/tui/test_startup_loading_indicators.py
```

Regenerate and then re-run the focused AXE visual suite:

```bash
just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_axe.py --sase-update-visual-snapshots
just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_axe.py
```

Finally run the required repository-wide validation:

```bash
just check
```

## Acceptance criteria

- No AXE info-panel render contains the persistent Help keycap, closing `]`, or `tab guide` label shown in the reported
  screenshot.
- Loading, daemon/countdown, lumberjack, chop-run, and background-command header information continues to render as
  before, aside from the removed hint and its adjacent separator spacing.
- The Help command and the AXE Guide tab remain available through the existing Help panel, with no keymap/default-config
  changes.
- `AxeInfoPanel` has no dead keymap state or keymap-injection API, and startup mount no longer performs the
  corresponding lookup.
- Focused unit tests, focused AXE visual snapshots, and `just check` all pass.
