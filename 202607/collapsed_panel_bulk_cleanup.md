---
tier: tale
title: Collapsed agent panel bulk cleanup
goal: 'Pressing the existing kill/dismiss action while a collapsed Agents-tab tribe
  panel is selected safely targets every agent in that panel, with accurate confirmation,
  discoverability, scoping, and regression coverage.

  '
create_time: 2026-07-18 06:24:23
status: done
prompt: 202607/prompts/collapsed_panel_bulk_cleanup.md
---

# Plan: Collapsed agent panel bulk cleanup

## Context and outcome

Collapsed whole-agent panels are first-class navigation targets. Their summary view intentionally exposes no hidden row
through `_get_selected_agent()`, so the existing `x` action currently falls through to “No agent selected” even though
the focused `@tribe` panel is an unambiguous bulk-cleanup scope. The cleanup panel already supports “Kill Panel,”
collapsed in-panel group banners already route `x` through the confirmed bulk transaction, and the selected panel can
already be resolved without I/O. The missing piece is to connect the collapsed-panel selection contract to that
established operation.

Make `x` mean “kill/dismiss this panel” only while a whole collapsed panel owns focus. Preserve the summary design's
core rule that the backing `current_idx` is merely an expansion anchor, never an implicit single-agent selection. The
new panel action is instead an explicit, discoverable, destructively confirmed bulk command. It must kill active
members, dismiss completed or pidless members, and leave every other expanded or collapsed panel untouched.

This is a Python TUI tale. Membership, confirmation, optimistic removal, process termination, tracked persistence,
notification cleanup, wait-dependency handling, and archive/index maintenance already have shared implementations; do
not add a parallel bulk-cleanup pipeline or move presentation-only focus logic into the Rust core.

## Selection, precedence, and scoping

Extend `action_kill_agent()` to resolve the existing `CollapsedAgentPanelFocus` after the marked-set branch and before
in-panel group-banner or single-row handling. This establishes an explicit precedence:

1. marks continue to target exactly the marked identities, even if a collapsed panel is focused;
2. an unmarked collapsed whole-panel selection targets the focused panel;
3. an expanded panel's focused collapsed group/tribe banner keeps its current group-scoped behavior; and
4. normal agent and synthetic clan rows keep their existing single/clan flows.

Resolve the panel targets from the memoized focused-panel slice through the existing selection/index helpers, not from
the stale backing row and not by rescanning files or persisted state. Keep tagged and untagged panel keys distinct, and
fail closed with a warning if focus or membership becomes stale before the modal is built. Use the complete focused
panel scope and the same workflow-child/clan expansion and identity de-duplication semantics as current panel cleanup;
the implementation must not accidentally include agents that share a project, status group, or tribe-like group key in
another panel.

Route the resolved candidates through the established bulk confirmation and `_do_bulk_kill_agents()` transaction. The
confirmation subject should identify the panel (`@name` or `(untagged)`), show the number/split of affected agents, and
retain the current danger behavior: dismiss-only panels use the dismiss confirmation, while any live process requires
the existing final kill confirmation. Cancellation performs no mutation. Confirmation produces one optimistic
removal/refresh and one tracked persistence task rather than a loop of single-agent operations, keeping destructive
disk/process work off the UI event loop.

## Discoverability and command consistency

Teach the Agents footer about collapsed whole-panel focus independently of group-banner focus. With no marks, the
summary view should advertise the existing configured `x` key as `kill/dismiss panel`; marked-set wording remains higher
priority, and no hidden-agent actions reappear. Update the Agents help description to include panels alongside agent,
group/tribe, clan, and marked targets where appropriate. No keymap or default-config value changes are needed because
this extends the contextual meaning of the existing `kill_agent` binding.

Carry the same focus fact into command-palette context and availability so `app.kill_agent` is offered when a collapsed
panel is selected. Context extraction must honor the authoritative selected-agent resolver (or otherwise explicitly
clear the raw backing agent) so the palette does not expose retry, edit, tmux, tag, or other single-agent commands for a
hidden row. Keep group banner and marked-set command behavior unchanged.

## Refresh and post-confirm behavior

Rely on the current bulk transaction's in-memory identity removal, selective row/panel refresh, recent-dismissal
grouping, and tracked background persistence. After confirmation, allow the existing panel synchronization to remove an
emptied panel, prune its collapsed key, and re-anchor focus to the next surviving panel/row; do not manually mutate fold
persistence or introduce a second focus-restoration path. If only some targets fail process signaling, preserve the
established bulk failure semantics and keep failed targets visible.

The keystroke and footer paths must remain synchronous and in-memory. Add no subprocesses, disk reads, timers, awaited
callbacks, or full-list reloads to focus resolution or modal construction. Any persistence and cleanup side effects
remain in the tracked background task already used by bulk kills.

## Verification

Add focused tests around the action and shared presentation contracts:

- action routing proves marked identities beat panel focus; a collapsed tagged panel targets its mixed
  running/done/pidless membership; an untagged panel is handled without confusing `None` with absent focus; unrelated
  panels and workflow cascades are scoped and de-duplicated correctly; stale/empty focus warns without opening a
  destructive modal; and cancel versus confirm has the expected mutation boundary;
- the confirmation description names the selected panel and partitions targets correctly, while confirmation delegates
  once to the existing bulk executor;
- footer tests cover `kill/dismiss panel`, marks-over-panel precedence, and the absence of the panel label for
  expanded/single-agent or in-panel group focus;
- command-context and availability tests cover collapsed-panel selection, suppress the backing hidden agent and its
  agent-only commands, and still expose `app.kill_agent`;
- panel-removal/focus tests verify that cleaning the last member of one panel leaves other panels intact and lands on a
  valid surviving selection; and
- the existing collapsed-panel interaction visual test semantically asserts the new footer affordance and updates only
  the intentionally changed PNG goldens. Add an interaction-level assertion that pressing `x` from the collapsed `@chop`
  state opens a panel-scoped confirmation whose subject does not list agents from neighboring panels.

Before handoff, run `just install`, targeted agent kill/panel-scope/selection, footer, command-palette, navigation, and
collapsed-panel visual tests. Inspect any updated 120x40 visual artifacts before accepting their goldens, then run
`just test-visual` and the repository-required `just check`. Because the new keypress path is only cached in-memory
selection and modal construction, a benchmark run is not expected; if implementation changes navigation or causes new
list reconstruction, use `SASE_TUI_PERF=1` or the Agents j/k benchmark to reconfirm the documented p95 target.

## Safeguards and non-goals

- Do not change panel folding, ordering, persistence formats, grouping modes, tribe directives, or collapse/expand
  keybindings.
- Do not reinterpret an expanded panel's collapsed in-panel group banner as a whole-panel selection, and do not make
  ordinary expanded agent rows panel-destructive.
- Do not bypass the existing confirmation modals, optimistic transaction, tracked task queue, or Rust-backed cleanup
  planning/side effects.
- Do not add a new keybinding or revive the retired global kill-all command; global and cleanup-panel actions remain
  separate explicit scopes.
- Do not make unrelated summary-widget layout or roster changes beyond the footer affordance and required visual golden
  update.
