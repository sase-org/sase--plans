---
tier: tale
title: Statistics contextual help and keymap integration
goal: 'Complete bead sase-8a.3 by making every Statistics view, control, metric, scope
  value, and freshness rule discoverable in a fast pane-local help overlay opened
  through a configurable key, while synchronizing the wider Admin Center help surfaces.

  '
create_time: 2026-07-20 14:53:36
status: wip
prompt: 202607/prompts/statistics_help_overlay.md
---

# Plan: Statistics contextual help and keymap integration

## Context

The preceding `sase-8a` phases have already established the Statistics view catalog, descriptions, scope chips, verified
per-view metric legends, actionable states, and clickable Overview tiles. This bead connects those existing sources of
truth into a contextual help experience without changing statistics queries, the Rust core, the
worker/debounce/coalescing load path, or the 30-second active-tab refresh path. Visual-golden regeneration remains the
responsibility of the dependent `sase-8a.4` phase.

## Implementation

1. Add a dedicated `StatisticsHelpModal` under the ACE TUI modals package. Its constructor will receive an immutable
   snapshot of the Statistics pane's already-resolved state: effective pane keymaps, active view, resolved range,
   Runtime and Projects grouping selections, current project filter display, and the last generated timestamp when
   available. The modal will do no disk, subprocess, query, refresh, or other data-scaled work when it opens.

2. Build the modal's scrollable content from the existing metadata rather than duplicating it:
   - enumerate `VIEW_ORDER` with `VIEW_LABELS` and `VIEW_DESCRIPTIONS`;
   - enumerate every effective Statistics binding from `statistics_help_bindings`, showing the active view and current
     range/group/project values where state applies, and naming Projects and Runtime as the grouping-capable views;
   - organize the complete `VIEW_LEGENDS` glossary by view;
   - explain that data comes from durable agent-run records through the Rust statistics bindings, refreshes every 30
     seconds while the tab is visible, and uses the displayed inclusive-start/exclusive-end timestamps, plus the last
     generated time when one exists.

   Render those sections with fixed-width bordered help conventions inside a vertically scrollable, Statistics-accented
   modal. Add a concise footer and close actions for Escape, `q`, and the configured Statistics help key. The content
   and bindings must remain usable when the help key is remapped.

3. Add a pane-local `action_help` that pushes the modal using only current in-memory pane state. Extend
   `StatisticsPaneKeymaps`, `_STATISTICS_BINDING_META`, `src/sase/default_config.yml`, and the public config schema with
   the `help` action, defaulting to `question_mark`. Let the existing Statistics binding builders, scoped validation,
   duplicate-key handling, and human-readable key formatting carry that action to runtime. Update the pane hint builder
   to append the effective help key without adding any footer-global binding.

4. Synchronize the three global ACE help binding surfaces so their Admin Center summary advertises the Statistics `?`
   help control alongside the existing view/range/group/project/refresh controls. Keep this summary informational; the
   Statistics binding remains active only while that pane has focus.

5. Add focused regression coverage:
   - modal content exhaustively covers every `VIEW_ORDER` entry, every `VIEW_LEGENDS` definition, and every Statistics
     binding action, including configured keys and current scope/freshness values;
   - pilot tests prove the default and remapped help keys open the overlay, configured help/Escape/`q` close it, and the
     Statistics help binding is inert on other Admin Center tabs;
   - pane hint and global help copy tests include the new action;
   - keymap default, registry override, schema acceptance, invalid-key fallback, and duplicate-key regression tests
     include `help`.

## Validation

Run `just install` first because this is an ephemeral workspace. Then run the focused Statistics/help/keymap/schema
tests while iterating, followed by `just check` as the required repository-wide validation. Re-run the focused
Statistics auto-refresh soak coverage to ensure modal plumbing did not alter or block the event loop/message pump.
Confirm the working diff contains no query, Rust-core, snapshot-golden, or bead-parent changes. Close only `sase-8a.3`
after all validation passes, leaving epic `sase-8a` open.
