---
tier: tale
title: Highlight selected tribe panel summary chrome
goal: 'Make selected, expanded agent tribe panels easier to identify by carrying the
  panel''s existing yellow focus accent into the total count and status-summary chrome
  without weakening the status counts'' semantic colors.

  '
create_time: 2026-07-19 16:20:02
status: wip
prompt: 202607/prompts/selected_tribe_panel_title_highlight.md
---

# Plan: Highlight selected tribe panel summary chrome

## Context and intended result

An expanded tribe panel already has an explicit whole-panel selection state: the panel receives the `-whole-panel-focus`
class, its outline and border-title color use `#FFD75F`, and its title is prefixed with `❖`. The remaining summary text
in the title is still mostly neutral gray, so a selected title such as `❖ @epic · 52 [R3 W7 D42]` does not carry the
focus accent through the most useful at-a-glance fields.

Keep the title's text, ordering, spacing, and width exactly the same, but when and only when the panel is selected and
expanded:

- render the total agent count (`52`) with the existing `#FFD75F` selected-panel accent;
- render `[` and `]` and every emitted status letter (`S`, `R`, `W`, `F`, `U`, and `D`) with that same accent;
- preserve each status count's current semantic styling, including the unread count's emphasized background, so the
  numeric values remain quickly distinguishable; and
- leave the separator before the total, all unselected panels, and collapsed panel titles on their current styles.

This is presentation-only TUI behavior. It does not change tribe membership, status projection, counts, navigation
state, layout geometry, or the Rust core.

## Implementation

### Selected title styling

Update the agent panel title builder in `src/sase/ace/tui/actions/agents/_display_panel_titles.py` to derive the summary
chrome style from the existing `selected` flag. Reuse the established selected-panel accent rather than introducing
another yellow. Split the neutral separator from the total count so only the numeric total is accented, and pass the
selected accent into status-chip construction only for selected titles. The caller already sets `selected` exclusively
for a non-collapsed focused panel, so keep that state boundary intact instead of duplicating focus logic.

Extend the shared formatter in `src/sase/ace/tui/agent_count_chip.py` with an opt-in chrome/letter style override. The
default path must reproduce the current chip exactly on every other ACE surface. When the override is supplied, apply it
to brackets, inter-token spacing, and every status letter, including `U`, while leaving metric digits under
`AGENT_COUNT_CHIP_METRIC_STYLES`. This keeps the selected-panel title compositional and avoids rebuilding or parsing the
chip in the panel-specific code.

No TCSS change should be necessary: `-whole-panel-focus` already supplies the desired `#FFD75F` outline and border-title
accent. Preserve the existing Rich plain text so dynamic width negotiation and panel layout do not change.

## Tests and visual verification

Add focused assertions in `tests/ace/tui/test_agent_count_chip.py` for the opt-in chrome override. Cover the full
canonical status sequence so the brackets, spaces, and all status letters use the override, the unread letter follows
the override in this mode, and every numeric count retains its semantic metric style. Retain the existing default-style
assertions as regression coverage for all other chip consumers.

Strengthen `tests/ace/tui/test_agent_panel_titles.py` around a selected expanded title to assert the exact accent spans
for the total, brackets, and status letters, as well as the unchanged semantic styles for status digits. Also rely on
the existing unselected and collapsed-title cases to prove that the accent is selection-scoped and the plain title
remains unchanged.

Regenerate only the intentional `agents_tribe_panel_selected_expanded_120x40.png` golden exercised by
`tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py`, inspect the result to confirm the selected title
reads clearly against the yellow panel chrome, and rerun that visual test without snapshot-update mode. The other tribe
fold-level goldens should remain byte-for-byte unchanged.

Before final handoff, run `just install`, the focused chip/title unit tests, and the selected tribe-panel visual test.
Use `--sase-update-visual-snapshots` only to accept the intentional selected-expanded golden change, then rerun without
that flag. Finally run the repository-required `just check`; if visual rendering differs unexpectedly, inspect the
actual/expected/diff artifacts under `.pytest_cache/sase-visual/` before accepting any golden.
