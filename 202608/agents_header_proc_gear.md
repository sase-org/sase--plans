---
tier: tale
title: Render the Agents header proc count as a trailing blue gear badge
goal: 'Replace the Agents-tab header''s leading "N procs" text with a compact blue

  `⚙N` badge placed immediately after the bracketed agent-status counts, while

  preserving the existing proc-shell count semantics and hiding the badge when

  the count is zero.

  '
size: small
proposed_by: bbugyi200.athena.0c8
---

- **AGENTS:**
  - [bbugyi200.athena.0c8](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0c8.md)
- **COMMITS:**
  - [5dc8973](https://github.com/sase-org/sase/commit/5dc8973956dbd2b82635cfb836be712ae9fa8439)
    — feat(tui): render agent header proc count as gear badge

# Plan: Render the Agents header proc count as a trailing blue gear badge

## Problem

The Agents tab currently renders a nonzero stand-alone proc-shell count before the
status strip, for example:

```text
21 agents · 1 procs  [6/10 running · 8 waiting · 7 done]
```

That long label interrupts the agent metrics and is inconsistent with the compact blue
gear notation already used for proc-shell counts in Agents-tab panel titles. The desired
header keeps the agent headline and status strip together, then shows the proc count to
the right of the first closing bracket:

```text
21 agents  [6/10 running · 8 waiting · 7 done] ⚙1
```

The gear and its following count should use the existing bold light-blue proc-shell
styling. A zero proc-shell count should continue to render no proc badge at all.

## Implementation

1. Update `AgentInfoPanel._build_display_text()` in
   `src/sase/ace/tui/widgets/agent_info_panel.py` so that a nonzero `_proc_shell_count`
   still qualifies the leading total as `N agents`, but no longer appends `· N procs`
   before the status strip. Build the bracketed status strip first, then append a
   single-space-separated inline `⚙N` badge. Keep the glyph before the number, style the
   complete badge with the existing bold light-blue proc-shell color (`#5FD7FF`), and
   preserve the current zero-count suppression. Do not change `_agent_info_metrics()`,
   runner/status counts, caching, loading-state rendering, or the independent top-bar
   proc and monitor indicators.

2. Strengthen the info-panel rendering tests in
   `tests/ace/tui/test_startup_loading_indicators.py` and/or
   `tests/ace/tui/widgets/test_agent_info_panel.py` to pin the new order and
   presentation. Cover a nonzero multi-digit proc-shell count so the test proves that
   the first `]` precedes `⚙`, the gear precedes the count, the word `procs` and the old
   dot separator are absent, and both glyph and digits use the intended blue style.
   Retain or add a zero-count assertion showing that no gear badge is emitted and the
   ordinary agent-only prefix remains unchanged.

3. Update the existing proc-shell Agents-tab PNG goldens exercised by
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_proc_shells.py`. Add a direct
   header-text assertion there if needed so the test identifies this contract
   independently of the many gear glyphs already present in proc rows. Regenerate only
   the snapshots changed by this header layout, inspect them to confirm the badge sits
   immediately after the status bracket at both 120x40 and 90x30 sizes (and in the
   detail view if that golden changes), then rerun the focused visual test without
   update mode.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral SASE
   workspace.
2. Run the focused non-visual tests for
   `tests/ace/tui/test_startup_loading_indicators.py` and
   `tests/ace/tui/widgets/test_agent_info_panel.py`.
3. Run the focused proc-shell visual test once with `--sase-update-visual-snapshots`,
   inspect every changed PNG, and rerun the same test without the update option to prove
   the new goldens are stable.
4. Run `just check` for the repository-wide lint gates and diff-scoped test lane.

## Non-goals

- Do not alter which rows count as proc shells or agents.
- Do not change the filled blue/orange chips in the application top bar or the Procs tab
  header.
- Do not change monitor badges, agent-panel border-title badges, keymaps, configuration,
  or refresh behavior.
