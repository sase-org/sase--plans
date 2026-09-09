---
tier: tale
title: A clear and beautiful pager breadcrumb trail
goal: "Make the pager's navigation history immediately legible, with a prominent current
  visit, useful surrounding breadcrumbs, accurate back and forward context, and a
  readable complete trail at every supported terminal size.

  "
size: medium
proposed_by: bbugyi200.athena.0he
create_time: 2026-09-09 09:49:20
status: wip
---

# A clear and beautiful pager breadcrumb trail

## Outcome and scope

Redesign the breadcrumb display shared by the standalone `SasePager` and embedded
`PagerScreen`. Readers should immediately understand where they are, how they arrived,
and which visits remain available in either direction. The reading surface should feel
deliberate and polished without taking excessive space from the document.

This is a **medium tale**: one implementation agent can deliver the renderer, Textual
integration, complete-history view, documentation, and focused regression coverage. The
work is substantial but bounded to pager presentation and its existing transient view
state. It needs no separate implementation phases.

Implement after plan approval. Preserve the existing history operations, resolution,
search restoration, label allocation, history limits, and host exit contract. Direct
jumps from breadcrumbs, new pager commands, persistent history, and changes to the ACE
link rail or Memory/Snippets presentation are outside this design.

## Findings that drive the design

- `src/sase/pager/_screen_trail.py::_trail_strip_entries()` includes the back stack and
  current section, but excludes the forward stack. It returns nothing whenever the back
  stack is empty, so returning to the earliest visit hides the trail even while Forward
  remains available.
- `src/sase/ace/tui/modals/trail_strip.py::build_trail_strip()` collapses kind-aware
  paths longer than three entries to `TRAIL  ⟨ …N › current ⟩`, regardless of available
  width. A five-entry path renders identically at 28, 60, and 120 columns. Shorter paths
  do not receive a strict width-fitting pass either.
- Current and previous entries use the same label styling. The one-row trail has no
  background separation and inherits muted text from `src/sase/pager/_styles.py`.
- `src/sase/pager/_help.py` appends a numbered trail after the key guide inside a
  height-constrained, non-scrollable container. It has no explicit current marker or
  forward history. A sufficiently long list cannot be inspected completely.
- The state already exists: `_back_trail`, `_forward_trail`, and the current document.
  `PagerTrailEntry` retains document/section titles and identities, scroll position,
  search state, and label anchor. Each stack uses `PAGER_TRAIL_LIMIT = 32`.
- `_screen_widgets.py::PagerBodyScroll.on_resize()` currently updates the body and
  subject, but not the trail. `_screen_body.py::_update_chrome_position()` updates both
  during explicit pager scrolling. The new trail must also follow native scroll, search
  jumps, goto, resize, and restoration paths.
- Existing PNG fixtures cover the base pager, syntax, and goto surfaces but have no
  breadcrumb-history examples. Use the existing pager PNG infrastructure for this work.

## Visual design

### A compact, prominent trail band

Keep the subject line at the top. When either history stack is nonempty, show a two-row
trail band immediately below it:

1. **Orientation row:** bold `TRAIL`, a compact `position/total` badge, available
   navigation keys with visit counts, and a right-aligned `? trail` hint.
2. **Path row:** retained visits in chronological order, separated by `›`, with a
   distinctly highlighted current visit and counted gaps only where entries are hidden.

The following are text wireframes; actual styling supplies the surface background, badge
fill, emphasis, and quieter forward entries. The eight-visit examples have the same
state: four back visits, the current fifth visit, and three forward visits.

Wide terminal, with all eight short labels visible:

```text
TRAIL 5/8    ^O back 4 · ^I forward 3                                    ? trail
▤ overview › ◈ problem › ▤ notes › ◈ design › [● ✎ pager.md] › ▤ app.py › ▤ tests › ◈ review
```

60-column terminal, retaining the immediate back and forward destinations:

```text
TRAIL 5/8    ^O back 4 · ^I forward 3              ? trail
…3 › ◈ design › [● ✎ pager.md] › ▤ app.py › …2
```

Returning to the earliest retained visit still shows the remaining route:

```text
TRAIL 1/3    ^I forward 2                          ? trail
[● ▤ overview.md] › ◈ design › ✎ pager.md
```

The brackets and `●` explicitly mark the current visit even without color. There is
exactly one current marker. Forward visits sit to its right and use quieter styling; the
header names their direction and count. These are visit-history breadcrumbs, not a
filesystem hierarchy: repeat visits remain distinct entries.

### Styling and space budget

- Use a subtle theme-derived surface background across the band, aligned to the
  subject's existing horizontal padding. Use theme foreground for readable labels, a
  restrained accent for `TRAIL` and the position badge, and a stronger filled background
  plus bold foreground for the current crumb. Keep the whole current label readable in
  both light and dark themes. Do not use the yellow painted-link key style for the
  current crumb.
- Reuse the existing artifact kind glyph/accent vocabulary. Color the small kind glyph;
  keep label text neutral so the band does not become a sequence of competing colors.
  Earlier visits have normal readable text; forward visits and separators are quieter,
  but essential text must not become illegible through `dim` styling on a light theme.
- Hide `#pager-chrome-rule` while the two-row band is visible. This reuses the existing
  trail-plus-rule space: subject plus active trail still occupies three rows. A fresh
  pager with both stacks empty keeps its current subject-plus-rule layout and does not
  display a lonely `1/1` breadcrumb.
- For pager screen heights of **12 rows or fewer**, use one trail row and hide the
  chrome rule. Preserve orientation, current title, direction counts, and the help hint;
  the full path remains available through `?`. Example:

  ```text
  TRAIL 5/8  ‹4 [● ✎ pager.md] 3›                   ? trail
  ```

- Fix the row count for each layout mode; do not wrap the compact trail, marquee it,
  animate it, or grow it vertically as history deepens. The active two-row layout
  consumes only one more row than a fresh pager; compact mode consumes no extra row.
- Breadcrumbs are informational. Do not add focus stops, underlined link styling,
  clickable-looking numeric jump labels, or new bindings to follow them. The existing
  Backspace/Ctrl+O and Ctrl+I operations remain the navigation controls.

### Width fitting is part of the contract

Measure the actual trail content width after padding and any borders, independently of
body scrollbar width. Use Rich terminal-cell measurement and style-preserving clipping.
`len()` and a final widget crop are not sufficient layout algorithms.

For the two-row path:

1. Show the full ordered path if it fits. Never collapse merely because there are four
   or more entries.
2. Under pressure, prioritize the current crumb, then the immediate back destination,
   then the immediate forward destination, then the oldest retained visit. Use sensible
   label budgets so one very long title does not evict every other useful breadcrumb.
3. Allocate remaining space to nearby visits. Preserve chronological order regardless of
   allocation priority. Each contiguous omitted run becomes one `…N` token at that
   position, where `N` is precisely the number of omitted visits. A truncated label does
   not count as an omitted visit. Never render duplicate copies of an entry that
   occupies two priority roles.
4. Shorten labels with an ellipsis, preserving a useful filename suffix/extension when
   possible. If the preferred set still cannot fit, relinquish the oldest retained
   entry, then the forward neighbor, then the back neighbor, replacing omissions with
   accurately counted gaps. The current marker and some current title have priority over
   every other crumb and every kind glyph.
5. At very narrow widths, reduce the path to the marked current label. Direction counts
   and position remain in the orientation row; hidden-history access remains `?`.

For the orientation row, prefer the full wording in the examples. When it cannot fit,
shorten `^O back B · ^I forward F` to `‹B F›`, dropping zero-count directions, then omit
that redundant direction summary. Shorten `? trail` to `?` before sacrificing the
position badge. `TRAIL p/N` plus `?` is the narrow fallback. In the one-row short-height
mode, preserve `p/N`, current marker/title, and `?` first, then the direction counts.

All rows must satisfy `cell_len(row.plain) <= available_width`, including widths 0 and

1. Below the space needed for meaningful text, render the highest-priority token that
   fits (or empty text), without negative budgets or an exception. Test Unicode wide
   characters and combining marks; clipping must not leave half a wide glyph or orphan a
   combining mark. Treat labels as literal `Text`, not Rich markup; normalize embedded
   newlines, tabs, and terminal control characters to safe single-line display text.

## Complete history through the existing help key

Upgrade `PagerHelpScreen` into a responsive, scrollable **Trail & keys** sheet whenever
history exists. Keep ordinary key help when there is no history. Use the existing `?`
entry point, and retain `?`, `q`, and Escape dismissal.

The sheet has a fixed title/header with `position/total`, one scrollable content area,
and a small fixed legend for scrolling and dismissal. Put the complete history before
the existing key guide. Each visit has its chronological number, kind glyph, full human
label, and an explicit state: `back`, `current`, or `forward`. Use a vertical connector
and the same prominent current treatment as the compact band:

```text
Trail & keys                                      5/8

  1  ▤ overview                         back
  │
  2  ◈ problem                          back
  │
  3  ▤ notes                            back
  │
  4  ◈ design                           back
  │
● 5  ✎ pager.md                         current
  │
  6  ▤ app.py                           forward
  │
  7  ▤ tests                            forward
  │
  8  ◈ review                           forward

Keys
  backspace / ctrl+o   Walk back
             ctrl+i   Walk forward
  ...existing key guide...
```

Wrap long titles and identities here instead of permanently truncating them. Show the
already-stored section identity/ref/path as secondary text when it adds information, and
include differing document/section titles for a multi-section visit. This lets readers
distinguish identical short filenames without resolving or reopening anything. Unknown
kinds and empty documents receive the existing neutral glyph/title fallback.

The viewport must fit narrow and short terminals; replace the hard-coded 64-column
assumption with a preferred width of 88 columns constrained by the available screen.
Keep a modest outer margin when space allows, and reserve a usable scrollable area on
small screens. At opening, reveal the current visit with neighboring context. Every
retained entry and the complete key guide must be reachable with keyboard scrolling and
the mouse wheel. Support `j/k`, arrows, Ctrl+D/Ctrl+U, and `g/G` within this sheet. The
help footer documents these controls. There is one scroll target, no row selection or
Enter-to-jump behavior, and no keyboard events leak to the underlying pager or host.

Opening and dismissing the sheet must leave document scroll, committed search state,
pending label state, and both history stacks untouched. Preserve input-mode priority:
while typing a search, `?` remains search text; an active goto prompt continues to
consume its own keys. Outside those input modes, route `?` to help before the generic
committed-search exit and invalid-label handling in `PagerScreen.on_key()`. This lets
the reader inspect history without clearing the current search or a pending link key.
Use a local pager key-routing change; do not change the shared search controller.
Explain in the sheet/documentation that positions count retained visits, `…N` denotes
hidden visits, and `●` marks the current visit. Older discarded visits are not
recoverable through this view.

## State and implementation boundaries

### One lightweight presentation snapshot

Create a pager-owned, immutable presentation snapshot and pure rendering helpers,
preferably in `src/sase/pager/_trail_chrome.py` (split narrowly if file-size gates need
it). Its ordered entries are:

```text
back oldest-to-newest + current + reversed(forward stack)
current_index = len(back)
position = current_index + 1
total = len(back) + 1 + len(forward)
visible = bool(back or forward)
```

The forward list's last element is the next Forward destination; displaying it in
storage order would be wrong. Build compact and complete-history views from this same
snapshot. Keep identities separate from shortened labels. The current entry reflects the
live current section; historical entries reflect their captured section. Scrolling
between sections updates the current crumb without adding a history entry. Use only
stored titles/identities and the live section metadata; do not call
`_current_view_state()` and copy search data merely to paint chrome.

The displayed total is the actual retained history length, not a lifetime counter. After
bounded eviction, the earliest displayed item is the oldest retained visit; do not label
it as the original root or imply lost visits can be recovered. Following a new document
after Back clears forward context through the existing navigation path. A failed
resolution, copy, edit, or media action does not invent a breadcrumb.

### Files and integration

- `_screen_trail.py`: replace the back-only strip projection/rendering with the
  snapshot; retain the existing push/pop, restore, generation checks, and limit logic in
  `_screen_trail.py`, `_screen_actions.py`, and `trail.py`.
- `screen.py` and `_styles.py`: integrate the band and responsive row modes, coordinate
  chrome-rule visibility, and pass the snapshot to help. Keep `PagerScreen` as the
  shared surface so CLI and embedded ACE callers receive identical behavior.
- `_screen_widgets.py`, `_screen_body.py`, `_screen_search.py`, and `_screen_goto.py`:
  audit and connect relevant scroll/resize paths to the existing chrome-update route.
  Observe actual scroll position so mouse-wheel/native scrolling and search jumps also
  update the live current-section crumb. Do not depend solely on j/k action handlers.
- `_help.py`: implement the scrollable sheet, full history renderer, and help-local
  scroll bindings. Split a helper only if needed to keep the module focused.
- `_screen_chrome.py`/`_chrome.py`: keep footer history availability correct. When
  history exists, label the existing `?` footer hint `trail/keys`; otherwise keep
  `keys`. Preserve current conditional Back/Forward hints.
- Reuse the marker vocabulary from `ace/tui/modals/trail_strip.py`; a small public
  marker accessor is acceptable if necessary to style icons separately from titles. Keep
  `build_trail_strip()`'s output unchanged for Memory/Snippets and its existing shared
  tests. Do not replace it globally with pager-specific behavior.
- `docs/pager.md`: document the trail band, position/count/gap semantics, compact
  layout, complete-history sheet, and unchanged navigation keys. No new CLI options or
  pager configuration values are needed; the existing default keymap configuration does
  not require a new binding for this design.

This is presentation-only Textual state and rendering, so it belongs in this Python repo
under the Rust-core boundary. Do not add a backend navigation/history API or reimplement
artifact resolution. Any unexpected shared-domain requirement should be resolved through
the existing Rust binding rather than expanding this display change.

### Update and performance rules

Refresh the trail after follow/back/forward, final restored scroll, current-section
changes, relevant viewport changes, and theme changes. First visibility must use a real
content width, not a hidden widget's zero-size placeholder; a cheap post-layout refresh
is appropriate. Cache the presentation signature (entry metadata, current index/section,
dimensions/mode, and resolved styles) and skip identical widget updates. Ordinary
scrolling within the same section must not rebuild the band or document.

All projection/fitting work is bounded by the existing stacks and displayed label
budgets. Keep render/update handlers free of I/O, artifact resolution, subprocesses, and
synchronous waits. Theme changes must refresh the new chrome independently of
syntax-highlight cache decisions, including when syntax is disabled. Reuse Textual theme
colors and the project's established color handling; verify both theme extremes and
monochrome output. Do not add timers or background work for breadcrumb rendering.

## Verification and acceptance

Read `lint_and_test.md` and `tui_perf.md` through `sase memory read` before
implementing. Use existing deterministic Textual wait helpers rather than adding timed
sleeps.

### Focused behavioral tests

Add pure renderer/snapshot tests covering width fitting, order, and omission accounting.
Exercise fresh, back-only, forward-only, middle-of-history, and long histories; repeated
visits and identical labels; unknown kinds; empty documents; multi-section labels; long
paths; literal Rich-looking text; control characters; wide and combining characters. For
widths from 0 through 160, assert bounded row width, no accidental wrapping, and correct
retained position/counts. At useful widths, assert exactly one visible current marker,
current label priority, correct gap totals, and expansion back to the complete path when
it fits. Check both short-height and normal modes.

Extend `tests/pager/test_app_history.py` and appropriate navigation/help tests:

- Follow A → B → C → D, back to B and then A, and forward again. Verify both displayed
  orders and counts at each step; update the old expectation that A hides the trail when
  forward history exists.
- From B, follow E and verify the forward branch disappears from both band and sheet.
  Exercise enough follows to evict the oldest retained visit and verify honest
  numbering.
- Preserve Backspace/Ctrl+O on an empty back stack returning
  `PagerExit(trail_exhausted=True)`, even when the forward trail is nonempty. Preserve
  Ctrl+I with no forward entries being inert, and ordinary q/Escape close behavior.
- Verify existing search, scroll, label-anchor, and syntax restoration behavior after
  history travel. Check failed resolution and non-navigation actions leave history
  alone.
- Resize a mounted active trail 120 → 60 → 28 → 120 columns and cross the 12-row height
  threshold. Assert repaint without an extra keypress, stable counts, no body overlap,
  and no repeated relayout cycle.
- Scroll a multi-section document with pager keys, native scroll, search, and goto. The
  current label tracks the live section; historical labels remain captured labels.
- Open `?` with a deep retained history. Reach its earliest/latest entries and key guide
  by keyboard and wheel, keep the current marker visible on entry, dismiss without
  changing underlying state, and verify help-local keys do not reach the host. Exercise
  committed search and a pending link prefix, and separately verify that `?` still obeys
  search-typing/goto input priority.
- Include an embedded `PagerScreen` case using
  `tests/ace/tui/actions/test_view_files_pager_screen.py`'s host pattern. Check resize,
  help dismissal, and the existing host-return contract.
- Verify runtime theme changes with syntax disabled. Verify repeated chrome updates for
  the same section/dimensions do not mutate widgets or rebuild body layout.

Retain shared trail-strip regression tests if exposing a marker helper.

### Visual review

Add breadcrumb-focused PNG tests under `tests/pager/visual/`, using its existing pinned
renderer and fixture. Cover a short trail, a deep trail, a current visit with history on
both sides, the forward-only state, and the complete-history sheet at **120×40 and
60×30**, in dark and light themes. Add representative **60×10** compact-band and
**28×12** narrow-help cases; use behavioral width tests for the remaining tiny sizes.
Add a representative no-color case proving that position, direction, and the current
marker remain clear. Keep fresh-pager snapshots stable unless an intentional change is
explained by this design.

Inspect rendered PNGs, including actual/expected/diff artifacts on failures. The visual
acceptance standard is: the eye finds the current page first, sees nearby destinations
second, and can distinguish navigation context from document text without reading the
footer. The band must have balanced spacing, readable contrast, aligned padding, and no
clipped markers. The sheet must visibly afford scrolling and show wrapped identities.
Iterate on styling and width budgets until these checks pass; do not accept snapshots
solely because they were regenerated.

Run the focused behavioral selection through the project test recipes, then the required
`just check`. Run the visual lane explicitly:

```bash
just test -- tests/pager tests/ace/tui/actions/test_view_files_pager_screen.py tests/ace/tui/modals/test_trail_strip.py
just check
just test-visual -- tests/pager/visual
```

For intentional/new PNG goldens, use a targeted
`just test-visual -- --sase-update-visual-snapshots <breadcrumb-test-file>` invocation,
inspect the results, then rerun the affected visual selection without the update flag.
Do not refresh unrelated goldens. Follow the memory's `sase_monitor` workflow if a
verification command takes a long time; exhaustive pre-landing `just check-full` is
host/landing verification and must use that workflow when required.

The implementation is complete when the new band and sheet meet these contracts in both
pager hosts, all retained history is inspectable, navigation/restoration behavior is
preserved, the docs match, the required checks pass, and the new visual examples have
been reviewed by eye.
