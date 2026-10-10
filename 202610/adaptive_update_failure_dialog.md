---
tier: tale
size: medium
title: Fit update failure reports to their content and terminal
goal:
  Update failure reports automatically use the space needed to minimize wrapping and
  scrolling, up to nearly the whole terminal, while staying readable, navigable, and
  visually calm at small sizes.
proposed_by: bbugyi200.apollo.69
create_time: 2026-10-10 12:14:58
status: wip
---

# Adaptive update failure dialog

The report should feel like opening a readable document: a small failure stays compact,
a long command gets more width, and a long transcript gets more height. When the report
needs almost the whole terminal, let it have it automatically.

## Evidence and scope

The user's screenshot, `~/tmp/screenshots/20261010_090913.png`, shows the **Update
failed** modal occupying roughly a third of a large terminal's width, with wrapped Git
commands and a scrolling output pane despite substantial unused space.
`UpdateFailureModal` in `src/sase/ace/tui/modals/update_failure_modal.py` is the exact
surface. Its rules in `src/sase/ace/tui/styles.tcss` fix the container at 90 columns and
cap `#update-failure-scroll` at 20 rows; container percentage caps also prevent full use
of the viewport. These are the immediate causes.

This is one medium tale: bounded presentation work in the modal, its styles,
documentation, and tests, implementable by one coding agent. All geometry belongs in
this Python/Textual frontend. The `UpdateFailure` data, journal, update actions, and
copy delivery already exist. Their domain behavior and Rust APIs need no changes. The
report displays the existing recorded tail, currently bounded to 200 lines and 16 KiB;
enlargement cannot recover output absent from that record.

Apply this design to `UpdateFailureModal`, including interrupted updates and both entry
paths (red gear and Update panel failure row). Do not generalize every modal, change
logging limits, add settings, or add a fullscreen/wrap mode. Automatic sizing solves the
requested workflow without another control to discover.

## Design contract

### Content determines the footprint

1. Keep a 90-column preferred minimum width for ordinary reports, clamped to the
   available terminal. This preserves a familiar compact starting point without imposing
   a maximum. Use natural content height, with no 20-row log ceiling and no
   percentage-based height floor for short reports.
2. Determine width before height. Measure the widest displayed logical line across
   metadata, failure explanation, and output, plus actual border, padding, and scrollbar
   space. Include the title and chosen footer's width requirements. Use terminal display
   cells, including wide and combining characters and a consistent explicit tab
   expansion in both measurement and rendering.
3. Choose the smallest width that satisfies those requirements and the preferred
   minimum, capped at the available width. A single long diagnostic line is enough to
   widen this report; do not use the preview panel's 95th-percentile heuristic, which
   could discard the one line that explains the failure.
4. At that width, measure the actual Rich/Textual wrapped rows of the entire report and
   add the real vertical chrome. Choose the smallest height that shows them all, capped
   at the available height. Short-wide and tall-narrow reports should grow on the needed
   axis independently.
5. On normal terminals, the maximum outer rectangle leaves two terminal columns on each
   side and one row above and below. Keep it centered with the existing dimmed backdrop.
   This produces an almost full-screen view with a visually balanced gutter, matching
   the preview panel's existing margin convention.
6. For cramped terminals (width below 60 columns or height below 16 rows), use the full
   available rectangle and reduce optional padding/blank lines. Bounds always win over
   preferred sizes. Handle transient zero dimensions by deferring layout; never compute
   negative dimensions. At 40x12 all actions and at least one report row must remain
   visible. Smaller sizes must still allow closing and scrolling without exceptions,
   even when the complete hints cannot fit.
7. When a line still exceeds the maximum inner width, wrap it normally. When the report
   still exceeds the maximum inner height, scroll vertically. Keep all recorded content
   reachable without horizontal scrolling, clipping, or ellipsizing the error/output.
   Preserve explicit line breaks and literal markup-like text.

### One report, one scroll region

Keep the existing red double outer border and failure/interrupted title. Place metadata,
bold red explanation, the subdued **last output** separator, and output inside a single
`VerticalScroll`, in that order. Remove the cyan inner output frame. The hierarchy
becomes one frame, a clear error, quiet metadata, and readable output. Use modest
spacing between sections; avoid an extra blank row after every element.

Keep the footer outside the scroll region, with the border title always visible. A
lengthy explanation must never consume all the space allocated to output or push the
actions offscreen: the entire report can scroll. Retain existing DOM IDs where useful
for tests; move their containment deliberately rather than introducing two competing
scroll regions. The modal initially shows the beginning of the report, including its
explanation; `G` reaches the last recorded output line.

Use a stable vertical scrollbar gutter, included in width measurement, and show the
scrollbar only when overflow exists. No outer-container scrolling. Pin the action footer
and give the report all remaining rows, allowing it to shrink.

The normal footer keeps the familiar actions:
`u open Update · d dismiss · y copy · q/Esc close`. Give key tokens normal contrast and
descriptions muted contrast. When scrolling is necessary, add a subdued second hint row
for `Ctrl+u/d scroll · g/G top/bottom`. Include that row in geometry calculations.
Resolve overflow in at most two deterministic passes: first with the action footer; if
content cannot fit at maximum height, reserve the scroll hint and finalize. Avoid
visibility/geometry feedback loops. At narrower widths, shorten labels and wrap between
complete action groups; at short heights omit the optional scroll-hint row before
sacrificing the action row or report viewport. Keep Update, dismiss, copy, and close
discoverable at the specified 40x12 minimum.

### Predictable interaction and resize

Preserve `u`, `d`, `y`, `q`, `Escape`, `Ctrl+d`, `Ctrl+u`, `g`, and `G`, including their
return values and the distinction between closing and dismissing a recorded failure.
Keep `CopyModeForwardingMixin`. Copy the original report text, unaffected by wrapping,
tab display, or scroll position. Focus the report viewport so wheel, arrow, and page
navigation work through Textual's normal scroll behavior. Half-page scroll steps must be
at least one row, even in a tiny viewport.

Compute the initial geometry before the first settled frame and recompute on actual
terminal-size changes. Growing and shrinking the terminal must both work. Do not remount
the report, steal focus, reset to the top, or animate resize. Preserve the existing
scroll offset, clamped only as necessary; a reader already at the bottom should remain
at the bottom after reflow. A screen resize must not be mistaken for the child layout
events caused by applying its new dimensions.

## Implementation sequence

1. Introduce a small, focused geometry helper adjacent to the modal, such as
   `src/sase/ace/tui/modals/update_failure_geometry.py`. Keep its calculation
   independently testable. Separate width measurement, wrapped-height measurement,
   footer selection, and viewport clamping enough to make their inputs explicit. Prefer
   existing Rich measurement facilities over `len()` or a hand-written approximation
   such as character-count divided by width. Keep chrome constants aligned with the
   actual TCSS and verify rendered geometry in integration tests.
2. Update `update_failure_modal.py` composition, initial sizing, resize handling, focus,
   and responsive footer. Build display text once from the immutable failure. Cache
   measurements by relevant width/terminal size and avoid doing this work on keystrokes,
   scrolling, or unrelated refreshes. The bounded in-memory report needs no disk reads,
   journal reloads, subprocesses, timers, or background workers for sizing. Guard
   pre-mount resize events and coalesce any necessary layout callback. Apply styles only
   when dimensions change; use a bounded settle correction if needed, never an unbounded
   resize/layout loop.
3. Replace only the `UpdateFailureModal` TCSS rules that impose the fixed width,
   percentage maxima, and 20-row ceiling. Implement the single scroll region, pinned
   footer, stable gutter, and reduced spacing at cramped dimensions. Ensure fallback
   TCSS cannot override the calculated caps or cause an initial oversized frame. Use the
   existing failure accent and theme surface/text tokens.
4. Update the red-gear/failure-report paragraph in `docs/ace.md` with automatic
   expansion, resize behavior, remaining scrolling, and close versus dismiss. Keep the
   `?` popup accurate per `src/sase/ace/AGENTS.md`: add a concise failure report
   reference to the appropriate help sections under
   `src/sase/ace/tui/modals/help_modal/`, including the existing report navigation keys.
   Preserve help box widths and short description limits. Use shared section
   construction if the same reference is exposed across help tabs. No keymap or
   `src/sase/default_config.yml` change is needed unless implementation actually changes
   a configured binding; this plan adds none.
5. Extend `tests/ace/tui/test_update_failure_modal.py` and add focused geometry tests.
   Add a dedicated visual module, such as
   `tests/ace/tui/visual/test_ace_png_snapshots_update_failure_modal.py`, using the
   established `AcePage`, startup stubs, deterministic timestamps, visual-idle wait, and
   canonical PNG renderer. The existing updates-indicator visual tests exercise the
   badge, not this modal, so they are insufficient coverage for the new layout.

## Verification and acceptance

Assert visible behavior and layout relationships, not merely copies of the sizing
formula. Use real mounted widgets to catch border, padding, scrollbar, and wrapping
differences. These cases must pass:

| Case                                                                            | Required result                                                                                                                     |
| ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Two-line failure at 240x70                                                      | Preferred compact width, natural short height, no needless scrollbar                                                                |
| A 160-cell command and 30 short output lines at 240x70                          | Width exceeds 90; command fits unwrapped; all output fits without vertical scrolling                                                |
| A wide transcript longer than the terminal at 160x50                            | Outer rectangle reaches 156x48; only the report scrolls; title/actions remain visible                                               |
| A fitting report at 120x40                                                      | No overflow introduced by scrollbar/chrome measurement; trailing output line visible                                                |
| Long report at 80x24 and 60x16                                                  | In-bounds layout, wrapping only at available width, footer usable, all report text reachable                                        |
| Long metadata/error at 40x12                                                    | No crushed/zero-height body; compact footer visible; beginning and end reachable                                                    |
| Resize 240x70 to 80x24 and back                                                 | Geometry reconverges, focus retained, offset clamped sensibly, bottom anchoring retained when applicable                            |
| Unicode, tabs, blank lines, trailing newline, very long token, literal `[bold]` | Measured and rendered widths/heights agree; content stays literal and reachable                                                     |
| Empty output and interrupted attempt                                            | Existing explanatory text and placeholder remain readable in a compact report                                                       |
| Exact fit, one-row overflow, repeated same-size resize                          | Stable scrollbar/footer decision; no oscillation or repeated style churn                                                            |
| Navigation and actions                                                          | Half-page/top/bottom reach their targets; wheel/keyboard scrolling works; copy stays original; close/dismiss/open results unchanged |

Use synthetic, deterministic update output based on the screenshot's structure: long Git
commands and paths, an index-lock explanation, and enough output rows to exercise both
axes. Do not depend on reading the user's live journal or on producing a real failed
update. Add a representative fixture at the journal's existing 200-line/ 16-KiB bounds
to verify responsive, finite sizing.

Capture and inspect PNGs for compact, large expanded, small-terminal overflow, and 40x12
fallback states. Include a large screenshot-inspired case where the available screen is
used effectively. Check the actual images for a clean single frame, balanced margins,
readable key hints, preserved red error hierarchy, and no stranded blank area that could
have prevented wrapping or scrolling.

Run focused behavior/geometry tests while developing. Before finishing, read the current
`lint_and_test.md`, `tui.md`, `tui_perf.md`, and `tui_screenshot.md` through
`sase memory read`. Run formatting and `sase tool run check` (the governed `just check`
recipe). Run `just fix-tui-screenshots --` with selectors for the new modal tests and
any changed help snapshots. Inspect the retained visual report and every new/updated
golden; an exit-zero partial update alone does not prove all selected snapshots current.
Use `/sase_monitor` for known-long verification and ensure its continuation inspects
visual results before completion. `just check-full` is not requested.

For a real workflow check, use `sase screenshot` against the implementation checkout and
open an already-recorded failure if available; inspect the resulting PNG. If no recorded
failure is available, report that limit and rely on the deterministic mounted modal
captures without modifying the live journal or triggering an update.

The work is complete when the mounted acceptance cases and required checks pass, the
images have been reviewed, both existing opening paths still use this modal, and the
report expands automatically while keeping its controls reliably accessible.
