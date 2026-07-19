---
tier: tale
title: Traverse prompt panes when repeating searches
goal: Prompt NORMAL-mode n/N repeats follow the shared search register across the
  entire prompt stack, focusing the pane that owns the next match and preserving Vim-style
  direction, count, wrap, and feedback behavior.
create_time: 2026-07-19 17:26:52
status: wip
prompt: 202607/prompts/prompt_search_cross_pane_navigation.md
---

# Plan: Traverse Prompt Panes When Repeating Searches

## Context

The prompt input bar now owns the last confirmed `/` or `?` search register, so a newly focused or rebuilt prompt pane
can use `n` and `N`. The repeat operation itself still runs entirely inside the focused `PromptTextArea`, however, and
the local match selector wraps within that text area before sibling panes can be considered. As a result, a repeat
either wraps locally or reports `pattern not found` even when the next directional match belongs to another prompt pane.

Treat the prompt stack as one ordered search space for repeat commands while leaving interactive `/` and `?` previews
pane-local. Forward traversal follows top-to-bottom pane order and increasing offsets within each pane; reverse
traversal mirrors that order. A repeat searches beyond the current cursor in the active pane first, crosses into
adjacent panes as necessary, and wraps only at the boundary of the whole stack. `N` continues to invert the direction
recorded by the original search, and a numeric count advances that many matches through this global ordering.

## Search Coordination

Move repeat-search coordination to the prompt input bar, alongside the shared search register and stack selection state.
Have the originating prompt text area delegate `n`/`N` repeats to that coordinator rather than allowing its local
selector to wrap prematurely.

Build a deterministic snapshot of mounted panes in stack order and their literal smartcase match spans for the
registered query. Resolve the destination from the originating pane and cursor without rebuilding panes or performing
I/O: choose the next or previous match according to the effective direction, skip panes with no matches, apply the count
across pane boundaries, and record whether traversal crossed the global top/bottom boundary. Keep the single-pane result
equivalent to the current Vim behavior.

When the destination belongs to another pane, use the existing prompt-stack focus machinery to update the selected item,
active/inactive styling, keyboard focus, NORMAL mode, visibility, and height. Clear transient search highlighting from
the departing pane, then position the destination cursor and render that pane's complete local match set with the
selected match marked current. Emit the existing top/bottom wrap notification only when the whole-stack traversal wraps.
If no pane contains the query, retain the current pane, cursor, and shared register, clear stale search highlighting,
and report `pattern not found`.

Keep this path synchronous, bounded, and UI-local: it is a keystroke navigation operation over already-mounted text, so
it must not introduce disk access, subprocesses, awaits, stack rebuilds, or backend behavior. Reuse the existing
literal/smartcase matching primitives and add only the narrow helper interfaces needed for the bar to resolve and apply
a prompt-search destination cleanly.

## Regression Coverage

Extend the interactive prompt-search tests to exercise the behavior through real `n`/`N` keypresses:

- forward repeats exhaust later matches in the active pane, focus a later pane at its first match, and skip intervening
  panes without matches;
- reverse repeats focus an earlier pane at its last match, while `N` traverses opposite to the recorded `/` or `?`
  direction;
- numeric counts advance across multiple panes and select the correct local match;
- crossing the bottom/top of the entire stack wraps to the opposite edge and preserves the existing Vim-style feedback,
  without reporting a wrap for an ordinary adjacent-pane transition;
- moving panes clears the old pane's transient highlights, focuses the destination in NORMAL mode, and highlights the
  destination pane with the correct current index;
- a query absent from every current pane keeps focus/cursor/register stable and reports `pattern not found`;
- existing single-pane repeats, shared-register behavior after manual pane navigation, and stack-rebuild persistence
  remain intact.

Run the focused prompt search and prompt-stack navigation tests first, then run `just check` for the repository-wide
required verification.

## Risks and Boundaries

The main risk is conflating a local buffer wrap with a whole-stack wrap, especially for reverse searches and counts
larger than one. Model the candidate ordering explicitly and test both directions at stack edges. Focus changes can also
trigger the bar's descendant-focus cleanup, so apply the transition through one controlled stack-level path and verify
that it does not erase the destination state after it is installed. Do not expand interactive `/` and `?` previews into
other panes or make the shared register depend on transient pane widgets; those behaviors are outside this fix and would
weaken the existing rebuild guarantee.
