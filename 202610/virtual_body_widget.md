---
tier: tale
title: Swap the pager Static body for a Line-API ScrollView
goal: "PagerView paints only visible rows from the virtual line model through one
  ScrollView, keeps every existing pager PNG golden byte-identical, and drops the giant
  Static compose path.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1es.6
bead: sase-1es.6
create_time: 2026-10-02 15:34:09
status: wip
---

- **PARENT:**
  [202610/pager_performance.md](https://github.com/sase-org/sase--plans/blob/main/202610/pager_performance.md)
- **BEAD:**
  [sase-1es.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1es/sase-1es.6.md)

# Swap the pager Static body for a Line-API ScrollView

Implement bead `sase-1es.6` (phase `virtual-body-widget` of epic `sase-1es`, design
`plan:202610/pager_performance.md`). One agent does the whole swap. Do not add a feature
flag, do not move wrap or layout into `sase-core`, and do not change
`VimSearchController`.

## Outcome

`PagerView` no longer mounts a `PagerBody(Static)` inside a `VerticalScroll`. One
`PagerBodyScroll(ScrollView)` with id `pager-body-scroll` renders row `scroll_y + y`
from the line model, sets `virtual_size` from that layout, and caches strips. Paint-only
changes (syntax publish, pending label prefix, goto emphasis, change marks, rail styles,
theme) bump a paint epoch and refresh visible rows. Layout changes (document, width,
label set, removal anchors) rebuild or relabel the integer layout. Search still receives
a full styled `Text` from the unchanged controller, but the widget splits that `Text`
into logical lines and paints only the visible ones. Every PNG under
`tests/pager/visual` stays byte-identical. Scrolling, keys, trail, history, split, goto,
and labels behave as they do today.

## Prerequisite: the line model is not on master

Phase `sase-1es.5` is closed, but `HEAD` (`3691b88ae7`) does not contain its files. They
exist only as dangling commit `cca2d9f8863a620c476deccfe75dd7bd96a633fe` in workspace
checkout `sase_13` (parent `34079430e3`, subject
`feat(pager): add Textual-free virtual body line model with parity oracle (sase-1es.5)`).
Master has moved by two commits since that parent (`e34386fee4`, `3691b88ae7`). Those
commits do not touch `src/sase/pager/_gutter.py`. They only delete three `sase-1eu`
`--epic-symbol` lines from the `Justfile`.

Before any widget work, confirm the modules are absent (`src/sase/pager/_body_lines.py`
does not exist). If they are already on the tree, skip the copy. If they are absent,
copy these paths byte-for-byte from that commit with
`git --git-dir=<sase_13>/.git show cca2d9f886:<path>`:

- `src/sase/pager/_body_lines.py` (new)
- `src/sase/pager/_body_layout.py` (new)
- `src/sase/pager/_body_rows.py` (new)
- `src/sase/pager/_gutter.py` (replace the worktree file only after
  `git diff 34079430e3 HEAD -- src/sase/pager/_gutter.py` is empty)
- `tests/pager/_reference_compose.py` (new)
- `tests/pager/test_body_model_parity.py` (new)

Do not apply the commit's `Justfile` hunk. Do not `git cherry-pick` and do not create a
git commit. The host finalizer commits the tree. Do not rewrite the model. If the object
is gone, search other `sase_*` workspace reflogs for that subject before giving up. If
it cannot be recovered, stop, record a `PROPOSED FOLLOW-UP:` note on `sase-1es.6`, and
do not invent a second line model.

After the copy, `tests/pager/test_body_model_parity.py` must pass. That file imports
`compose_body`. Keep `compose_body` until the deletion step below, then point the parity
tests at `reference_compose_body` only.

### API the widget consumes

`build_body_layout(document, width, *, label_layer, prepared_sections, removal_anchors) -> BodyLayout`
in `_body_layout.py`. `BodyLayout` carries `section_offsets`, `total_height`,
`section_line_counts`, `section_line_rows`, `width`, `content_width`, and
`lines_laid_out`. `relabel(new_layer) -> BodyLayout` rebuilds only lines whose label key
changed and sets `lines_laid_out` to that count. `locate(row)` resolves a row. Field
names match what `reading_anchor_at_row` and `row_for_reading_anchor` already read off
`ComposedBody`.

`BodyPaintState` and `BodyRenderer` live in `_body_rows.py`. Construct one renderer per
paint epoch. `BodyRenderer.render_row(row) -> Text` is O(that line) after staging and
increments `rows_rendered`. The module-level `render_row()` helper restages the whole
epoch on every call. Do not use it from the widget.

Paint state holds the pending label prefix, prepared syntax text, goto mark and accent,
change marks, rail styles, history styles, and surface. The model documents that those
do not change row counts. Label-set changes and removal anchors do.

`BodyRenderer.__init__` stages every section's labeled text once per epoch. That cost is
accepted. The win is that Textual no longer renders every row.

## Widget

Replace `PagerBody` and the `VerticalScroll` subclass in
`src/sase/pager/_screen_widgets.py`. Keep the class name `PagerBodyScroll` and the id
`pager-body-scroll`. Subclass `textual.scroll_view.ScrollView`. Delete `PagerBody`.
`view.compose` yields the scroll view alone, not a child Static.

`ScrollView` defaults to `overflow-x: auto`. Today's `VerticalScroll` is
`overflow-x: hidden` and `overflow-y: auto`. Set that on `PagerBodyScroll` and on
`PagerView #pager-body-scroll` in `src/sase/pager/_styles.py`. Move `padding: 0 1` from
`#pager-body` onto `#pager-body-scroll` and delete the `#pager-body` rule. The padding
columns must still be drawn by Textual's style cache around `render_line` content, which
is how the old Static got its one-column inset.

`render_line(y)` paints absolute row `scroll_offset.y + y`:

- Ask the owning `PagerView` for the row `Text` (model row, or overlay line while search
  is painted).
- Convert that `Text` to a `Strip` with the app console at the paint width, `no_wrap`
  and `overflow="crop"`, then `Strip.from_lines`, the same Line-API shape as
  `textual.widgets.RichLog`.
- Crop with `crop_extend` to the content width and `apply_style(self.rich_style)`.
- On any exception, paint that row's plain text (the logical line, cropped) and do not
  raise. `BodyRenderer.render_row` already swallows exceptions into an empty `Text`, so
  the widget test must patch `BodyRenderer.render_row` to raise and still observe the
  plain row.

Cache strips in a `textual.cache.LRUCache` whose max size is
`max(512, 4 * viewport_height)`. Key by
`(absolute_row, paint_epoch, paint_width, scroll_x)`. A cache hit must not call
`render_row` and must not increment `rows_rendered`. Clear the cache on layout
invalidation, paint-epoch bump, overlay switch, document swap, and widget unmount.

`virtual_size` is `Size(paint_width, max(layout.total_height, 1))` for the model, and
`Size(paint_width, max(overlay_line_count, 1))` during search. Do not grow the width
past the content box. Horizontal scrolling stays off.

Override `watch_scroll_x` and `watch_scroll_y` to call `super()` and then
`view._after_scroll()`, preserving today's chrome and window-label refresh. `on_resize`
calls `view._ensure_body_layout()` (relayout only when the paint width changed) and
`view._update_chrome_position()`.

Keep `scroll_relative`, `scroll_to`, `max_scroll_y`, `scrollable_content_region`, focus,
mouse wheel, and scrollbar dragging on the inherited `ScrollableContainer` path.
`_screen_split.py` focuses `#pager-body-scroll` with an expected type of `Static`.
Change that expected type to `PagerBodyScroll`. A `Static` expectation throws, the `try`
swallows it, and the pane never takes focus.

### Paint width

Today `_body_paint_width` subtracts the child Static's horizontal padding from
`scrollable_content_region.width`, because the padding lived on the child. After the
padding moves onto the `ScrollView`, the scrollable content width is already inside that
padding. Use that width as the layout width. Do not subtract the padding a second time.
Prove it with the headless parity test below before running the PNG suite: the first
visible row's plain text, including the gutter, matches the oracle row, and the widget's
content origin is one column in from the pane edge.

## Invalidation

Replace the `self._body_width = None; self._ensure_body()` idiom. Keep `_body` as the
`BodyLayout` (it already exposes `section_offsets`, `total_height`,
`section_line_counts`, and `section_line_rows`, so goto, trail, history, diff, and
reading-anchor callers stay). Store the current `BodyRenderer` and a paint epoch on the
view.

Widen `reading_anchor_at_row` and `row_for_reading_anchor` to a small `Protocol` with
those three fields so `BodyLayout` type-checks. Do not keep building a `ComposedBody`
just to satisfy the annotation.

`_invalidate_body_paint()` builds a new `BodyPaintState` and `BodyRenderer` from the
current prefix, prepared sections, goto mark, change marks, rails, history styles, and
surface, bumps the epoch, clears the strip cache, and `refresh()`es without
`layout=True`. It does not change `scroll_y` and does not increment `lines_laid_out`.

`_invalidate_body_layout()` rebuilds with `build_body_layout` when the document, width,
or removal anchors changed. A label-set change at the same width calls
`layout.relabel(new_layer)` instead of a full rebuild. A real width change captures
`reading_anchor_at_row` first and restores it through the existing
`_restore_reading_anchor` path (top logical line stays put). Same-width relayouts never
call `scroll_to`. Then set `virtual_size`, clear the strip cache, and refresh.

`_ensure_body_layout()` is the "layout exists" call used by
`_current_section_line_count` and mount. It builds only when the layout is missing or
the paint width changed.

Route every current call site:

| Location                                                                               | What changed                                       | API                                                       |
| -------------------------------------------------------------------------------------- | -------------------------------------------------- | --------------------------------------------------------- |
| `view.on_mount`                                                                        | first layout                                       | `_ensure_body_layout`                                     |
| `view.set_pane_role`                                                                   | badges dropped or restored                         | `relabel` via layout invalidation, scroll stays           |
| `view.apply_split_seed`                                                                | body cleared before mount                          | next mount lays out                                       |
| `_screen_body.action_refresh` (no provider)                                            | dangling set cleared, suffix characters can change | `relabel`, scroll stays                                   |
| `_apply_refreshed_document`                                                            | new document                                       | layout, then the existing section-identity scroll restore |
| `_refresh_window_scoped_labels_if_needed`                                              | window labels moved                                | `relabel`, scroll stays                                   |
| `_screen_actions_labels._repaint_label_state`                                          | pending prefix                                     | paint                                                     |
| `_screen_syntax._publish_syntax_update` and the theme path that rewrites `#pager-body` | prepared text and surface                          | paint, `lines_laid_out` unchanged                         |
| `_screen_goto._submit_goto`                                                            | goto emphasis                                      | paint, then the existing `_scroll_to_line_mark`           |
| `_screen_goto._current_section_line_count`                                             | layout must exist                                  | `_ensure_body_layout`                                     |
| `_screen_history_swap._recompose_for_history_marks`                                    | marks and rails                                    | paint                                                     |
| `_screen_diff_view` section body replace                                               | document text                                      | layout, keep `_restore` / search reapply                  |
| `_screen_diff_folds` fold expand                                                       | document text                                      | layout, keep `_restore_history_anchor`                    |
| `_screen_trail._restore_view_state`                                                    | new document                                       | layout, then the existing scroll restore                  |
| `_screen_actions_resolve._navigate_to_document`                                        | new document, then a goto mark                     | layout, then paint for the mark                           |
| `_screen_time_band` when the band target count changes                                 | body label set may change                          | `relabel`, scroll stays                                   |
| `PagerBodyScroll.on_resize`                                                            | width                                              | layout only when `_body_paint_width()` changed            |

Search methods stop calling `query_one("#pager-body")`. There is no such node.

Add any new per-view attribute that code assigns through the screen facade to
`_VIEW_STATE_ATTRS` in `_screen_host.py`. Methods defined on `PagerBodyMixin` are
forwarded already.

## Search overlay in this phase

Leave `VimSearchController` alone. `styled_search_base` stays in `_layout.py`. The
controller already sets `no_wrap=True` and `overflow="crop"` on the overlay `Text`, and
it scrolls by logical line index.

`vim_search_paint_overlay(content)` stores that `Text` on the scroll widget and switches
`render_line` to an overlay source. Split on newlines lazily (reuse
`logical_line_starts` and `slice_styled_line` from `_body_lines.py`). Each logical line
is one row. Crop to the content width starting at column 0. Do not wrap, and do not
honor `scroll_x`: today's container is `overflow-x: hidden`, so a horizontal `scroll_to`
must not reveal more columns. `virtual_size` height is the logical line count.

`vim_search_hide_overlay` drops the overlay, restores the model `virtual_size`, and
refreshes. It does not rebuild the layout.

## Delete the dead compose path

After the widget is the only painter, delete src symbols with no non-test consumer. Test
imports do not count for Symvision.

- Delete `PagerBody`.
- Delete `compose_body` and `apply_gutter`, plus private helpers that only they used.
  Keep `styled_search_base`, `search_corpus`, `reading_anchor_at_row`,
  `row_for_reading_anchor`, `current_section_index`, and `render_section_with_labels`
  (the line model and the search host still use them). Delete `ComposedBody` if nothing
  in `src/` constructs it.
- Update `tests/pager/test_body_model_parity.py` so it compares the line model to
  `reference_compose_body` and does not import `compose_body`.
- `RowLocation` and `renderable_rows` are public and referenced only inside
  `_body_layout.py`. Rename them to `_RowLocation` and `_renderable_rows`, and drop them
  from `__all__`.
- Delete the module-level `render_row()` in `_body_rows.py` if the widget uses
  `BodyRenderer.render_row` and no src caller remains. Keep the method.
- The widget's src imports of `build_body_layout`, `BodyPaintState`, and `BodyRenderer`
  are the real consumers. Do not add `--epic-symbol` lines for `sase-1es` or
  `sase-1es.6`. `sase bead epic-symbols sase-1es.6` must print no entries before close.

`render_section_with_labels` and `effective_section_source` stay public because another
src module already imports them.

## Tests

Update queries that expect `VerticalScroll` or `#pager-body`. `PagerBodyScroll` is not a
`VerticalScroll` and not a `Static`, so `query_one(..., VerticalScroll)` raises.

- `tests/pager/_app_helpers.py` `body_scroll`: expect `PagerBodyScroll`.
- `tests/pager/test_app_goto.py`, `tests/pager/test_goto.py`,
  `tests/pager/test_syntax_activation.py`,
  `tests/pager/visual/test_app_png_snapshots.py`,
  `tests/ace/tui/actions/test_view_files_pager_screen.py`: same type change.
- `tests/pager/test_app_navigation.py`, `test_app_goto.py`, `test_goto.py`, and
  `test_rendered_link_navigation.py` read `screen._body.renderable.renderables[0]` and
  index `plain.split("\n")`. Add a view helper `row_text(row) -> Text` that returns the
  model row (the overlay line while search is painted) and assert on `row_text(n).plain`
  with the same prefixes (`"12┃ "` and neighbors).
- `tests/pager/test_app_split.py` asserts `query("#pager-body")`. Assert
  `#pager-body-scroll` instead. That node is the body.
- `tests/pager/test_screen_syntax.py` `_FakeHost.query_one` asserts the selector
  `#pager-body`. Point the theme and publish paths at `_invalidate_body_paint` and teach
  the fake to record that call.

New tests:

- Headless parity against `reference_compose_body`: visible rows, text and style, at
  several scroll positions, after a resize, after a label prefix, and after a syntax
  publish. Several scroll positions means the top, a mid-document row, and the last
  viewport. Style means the Rich span styles on the row `Text`, not only `.plain`.
- Counters at 120x40. Opening a 50k-line document leaves `BodyRenderer.rows_rendered` at
  most `2 * body_viewport_height` after first paint. `j` increases `rows_rendered` only
  by the newly exposed rows (one, when the cache holds the rest). A label-prefix key on
  a document-mode label layer relayouts 0 lines and re-renders at most one viewport. A
  syntax publish relayouts 0 lines.
- A patched `BodyRenderer.render_row` that raises paints the plain row and does not
  crash `render_line`.

Keep each edited `src/sase/pager` file under the 700-line toobig warning threshold. The
scroll widget belongs in `_screen_widgets.py` (67 lines today). Invalidation stays on
`PagerBodyMixin`.

## Verification

Run, in order:

1. `tests/pager/test_body_model_parity.py` after the copy.
2. The new widget tests and the updated pager tests listed above.
3. `sase tool run check`. A failure that also fails on the clean base tree (the
   directive-vocabulary mismatches and the full-suite-only flakes already noted on
   `sase-1es.5`) does not keep this bead open. Record it as `PROPOSED FOLLOW-UP:` on
   `sase-1es.6` if it is not already noted, and continue.
4. `just test-visual -- tests/pager/visual` in check mode. Zero golden changes. If the
   run would outlast the turn, hand it to `sase monitor` (`/sase_monitor`) instead of
   blocking the turn. Do not run `just check-full`. Do not update goldens to force a
   pass. A mismatch is a widget bug (width, padding, overflow, or crop) until proven
   otherwise.
5. Bench, headless, above the trivial-app floor. Record before/after in a bead note.
   Command:
   `just bench-pager --cases code-sparse,log-dense --ladder 2000,20000,100000 --no-cold`.
   Targets: 2k-line open ≤ 250 ms (baseline 14 s), 20k-line open ≤ 600 ms (baseline 145
   s), 100k `code-sparse` ≤ 2 s, 100k `log-dense` ≤ 5 s, `j` / `k` / `ctrl+d` / `G` ≤ 5
   ms above the floor at every size, label-prefix key at 20k lines ≤ 10 ms, peak RSS
   opening 20k lines ≤ 350 MB (baseline 5.7 GB). If a number misses, record why and a
   `PROPOSED FOLLOW-UP:` line. Do not drop a guardrail to hit a number.

Then `sase bead epic-symbols sase-1es.6`. Close only `sase-1es.6` with
`sase bead close sase-1es.6 --note "<what you verified>"`. Do not close `sase-1es` or
any ancestor. Do not create beads. Discovered work goes on this bead as
`PROPOSED FOLLOW-UP:` via `sase bead note`.

## Guardrails

- No visible behavior change. Rendering, keys, navigation, trail, history, split,
  search, goto, and labels stay.
- No second full copy of a document's text beyond the model's per-epoch staged lines and
  the search `Text` the controller already builds.
- The strip cache is per widget, bounded, and dropped on document swap and unmount. No
  disk cache, no new thread, no I/O on the key path.
- `render_line` never raises.
- Read `tui_perf.md`, `lint_and_test.md`, and `symvision.md` with `sase memory read`
  before editing if those rules are not already in context.
- Follow the surrounding pager style. Do not add comments that restate the design.
