---
tier: tale
title: Keep Artifacts selections inside their list viewports
goal: Fast and programmatic navigation in every non-PR Artifacts sub-tab keeps the
  selected row visible without reintroducing selection-event echoes or eager detail
  rendering.
create_time: 2026-07-21 15:10:05
status: done
---

- **PROMPT:** [202607/prompts/artifacts_viewport_follow.md](prompts/artifacts_viewport_follow.md)

# Plan: Keep Artifacts selections inside their list viewports

## Context and root cause

The shared non-PR Artifacts navigation path routes `Ctrl+F`, `Ctrl+B`, `g`, `G`, and adaptive entry jumps through
`select_relative_entry()` and each pane's stable-target `select_entry_target()` implementation. This correctly moves the
selected model entry and avoids opening it, but it does not uniformly preserve the `OptionList` viewport invariant that
the highlighted row remains visible.

Textual 8.0.1 implements viewport following inside `OptionList.watch_highlighted()` by calling `scroll_to_highlight()`
before emitting `OptionHighlighted`. `CommitsTimeline` and `BugIssueList` deliberately bypass that watcher while
`_programmatic_update` is true so direct highlight assignments cannot echo back into navigation or detail updates. The
guard is necessary, but it also suppresses Textual's scroll side effect. A headless reproduction with 50 entries shows
Commits and Bugs advancing to options 41 and 40 while their 24–25-row viewports remain at `scroll_y=0`. Plans uses a
pane-level event guard instead, still reaches Textual's watcher, and correctly advances its viewport to `scroll_y=17`
for an equivalent jump.

This is presentation-only Textual behavior and remains in the Python TUI; no Rust core or wire/API changes are needed.

## Implementation

### Restore viewport following under guarded highlights

Keep the existing synchronous programmatic-update guards, stable-target selection, focus behavior, disabled
heading/day-banner handling, and detail debouncing. In the guarded highlight helpers and rebuild paths for Commits and
Bugs, explicitly reveal the assigned highlight with Textual's non-animated `scroll_to_highlight()` behavior before
clearing the guard. Centralize the operation inside each custom `OptionList` rather than teaching the shared navigator
about widget row geometry.

Apply the behavior consistently to:

- direct stable-target selection in `CommitsTimeline`, which serves fast distance navigation, first/last navigation, and
  adaptive entry jumps;
- commit timeline option rebuilds that preserve a stable selected commit across refreshes or jump-hint repainting;
- `BugIssueList.set_highlight()`, which serves ordinary Bugs `j/k` movement as well as stable-target navigation;
- bug option replacement that preserves selection across refreshes or jump-hint repainting.

Do not emit `OptionHighlighted` during these assignments, open an artifact, or synchronously render expensive detail
content. Leave Plans on its already-correct native watcher path unless the regression test exposes a separate issue.

## Regression coverage

Extend `tests/ace/tui/test_artifacts_list_navigation.py` with enough selectable rows to exceed the actual list viewport
and assert both halves of the contract:

- repeated `Ctrl+F` moves the stable selection beyond the initial visible page and scrolls until its highlighted option
  is inside the viewport;
- repeated `Ctrl+B` moves back above the current viewport and scrolls upward to reveal the highlight;
- Commits still skip disabled day banners, Plans still skip non-selectable headings, and Bugs still restore issue-list
  focus;
- fast navigation still changes selection without opening an artifact or producing duplicate selection/detail effects;
- Commits, Bugs, and Plans all satisfy the same visibility assertion, using Plans as coverage for the native watcher
  path and guarding against future sub-tab divergence.

Prefer a visibility assertion derived from the highlighted option's laid-out line region and the list's current scroll
region over brittle exact scroll offsets.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE workspace.
2. Run the focused Artifacts navigation test module and any directly affected Commits/Bugs widget tests while iterating.
3. Run `just check` as the required full repository validation, including formatting, lint/type checks, SASE validation,
   the complete test suite, and visual snapshots.
4. Re-run the focused viewport regression after the full check if formatting or test-driven adjustments changed the
   guarded highlight code, and confirm the working tree contains only the intended implementation and test changes.

## Risks and constraints

Calling viewport alignment after every programmatic row replacement can change stale scroll offsets when the data model
is repainted; that is intentional only insofar as the preserved selection must remain visible. Keep the call synchronous
and non-animated to preserve keystroke responsiveness. The event-suppression flag must still be cleared in `finally` so
an exception cannot leave user navigation muted.
