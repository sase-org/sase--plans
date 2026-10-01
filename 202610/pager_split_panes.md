---
tier: epic
title: Pager split panes (`\` below, `|` beside)
goal: 'The SASE pager can show two independent reading panes, split below with `\`
  or beside with `|`, using Agents-tab-style toggle/rotate/focus semantics, framed
  accent-colored panes, link labels painted only in the focused pane, ctrl+w to follow
  a link into the other pane, and a reading position that never jumps. Single-pane
  rendering stays pixel-identical to today.

  '
phases:
- id: view-extract
  title: Extract a per-pane PagerView from PagerScreen
  depends_on: []
  size: medium
  description: 'view-extract: move all per-document pager state, chrome rows, lifecycle
    and key handling into a mountable PagerView widget. PagerScreen becomes a thin
    host that routes bindings to the focused view. No user-visible change, and every
    existing pager PNG golden stays byte-identical.'
- id: reading-anchor
  title: Keep the reading line fixed across width changes
  depends_on:
  - view-extract
  size: small
  description: 'reading-anchor: add a pure (section, line, row-offset) reading anchor.
    A pane that recomposes at a new width keeps its top logical line, and trail back/forward
    lands on the recorded line even when the pane width has changed since the visit.'
- id: split-panes
  title: Split panes with framed chrome and focus-scoped labels
  depends_on:
  - reading-anchor
  size: medium
  description: 'split-panes: pure split model; `\` / `|` / ctrl+f / `+` / `-` keys;
    q, Esc and exhausted backspace close a pane; pane clone on split; link labels
    only in the focused pane; framed panes with the subject in the border title/subtitle
    and accent focus colors; split footer verbs, small-window guard, help rows, and
    async-safe pane teardown.'
- id: other-pane-follow
  title: Follow a link into the other pane with ctrl+w
  depends_on:
  - split-panes
  size: small
  description: 'other-pane-follow: ctrl+w arms an ''other pane'' follow. A painted
    label then opens its target in the other pane (opening a split when single) while
    focus stays put, and doubled ctrl+w focuses the other pane.'
- id: polish-docs
  title: Split-view goldens, visual polish, and docs
  depends_on:
  - other-pane-follow
  size: small
  description: 'polish-docs: add split-view PNG goldens, review the result against
    the look spec and fix visual nits, then document split panes in docs/pager.md
    and the help sheet.'
proposed_by: bbugyi200.athena.0v1
create_time: 2026-10-01 15:39:28
status: wip
bead_id: sase-1eg
---

- **PROMPT:** [prompts/202610/pager_split_panes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/pager_split_panes.md)
- **BEAD:** [sase-1eg](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eg/README.md)

# Plan: Pager split panes

## Context

The SASE pager (`src/sase/pager/`) is the link-traversing reading surface. It runs
standalone (`SasePager` in `src/sase/pager/app.py`, used by `sase pager`, `cli_pager.py`
and memory history) and as a modal pushed inside the ACE TUI (`_files.py` hints,
`_metadata_pager.py`, `memory_pane_history.py`, `command_line/screen_navigation.py`). It
paints jump-hint labels (`0-9a-zA-Z` minus the reserved `qjkgGyErnN`) over links, keeps
a bounded back/forward trail, and supports search, goto-line, syntax highlighting, and
the memory-history time axis.

Today `PagerScreen` (`src/sase/pager/screen.py`) is a single `ModalScreen` built from
eleven mixins (`_screen_*.py`). All per-document state lives on the screen object:
document, composed body, label layer, trail, search controller, goto, history, diff,
time band and syntax state. A second pane is impossible without first moving that state
into a widget that can be mounted twice.

The Agents tab's deck split was the inspiration (`src/sase/ace/tui/widgets/decks/`:
`layout.py`, `area.py`; CSS under `#agent-deck-area` in `src/sase/ace/tui/styles.tcss`):

- From a single deck, `\` opens a top/bottom split and `|` opens a left/right split. The
  new panel takes focus at a 50/50 ratio.
- Pressing the same key again unsplits back to panel 0. Pressing the other key rotates
  the split.
- `ctrl+f` focuses the other panel, and `{` / `}` step the ratio through 30/50/70.
- Panels are framed in their deck's accent color, at 35% strength when unfocused.

Facts verified while planning:

- Textual 8.0.1 accepts Rich `Text` for `border_title` / `border_subtitle`.
- Duplicate widget IDs are legal in different subtrees. However, `query_one("#id")` from
  a base node caches its result keyed only on the base node's direct children, so a
  host-level ID query can return a removed pane's widget.
- Under a `ModalScreen`, non-priority app bindings are cut from the binding chain. ACE's
  `\`, `|` and `ctrl+f` deck bindings therefore never fire while the pager is open, but
  no test covers this yet.
- No production caller reads `PagerScreen` internals or `PagerExit.trail_exhausted`.
- Tests reach into screen privates heavily: about 205 accesses across about 25 files,
  plus 22 monkeypatches by string path into `sase.pager.screen`.
- Pager keys are hard-coded in `screen.py`, not in the configurable keymaps.
  `src/sase/default_config.yml` is untouched by this epic.
- This is presentation-only Textual work. Nothing crosses the `sase_core` boundary.

## UX design (the contract every phase implements)

### Keys

| Key                          | One pane                                                           | Two panes                                                                                         |
| ---------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `\`                          | Split below: a clone of this pane opens underneath and takes focus | Panes stacked: close the **other** pane and keep the focused one. Side by side: rotate to stacked |
| `\|`                         | Split beside: a clone opens to the right and takes focus           | Side by side: close the other pane. Stacked: rotate to side by side                               |
| `ctrl+f`                     | (nothing)                                                          | Focus the other pane                                                                              |
| `+` / `-`                    | (nothing)                                                          | Grow / shrink the focused pane (30/50/70 steps, clamped)                                          |
| `q` / `Esc`                  | Close the pager (unchanged)                                        | Close the **focused** pane; the other pane fills the screen and takes focus                       |
| `Backspace` past trail start | Close the pager (unchanged)                                        | Close the focused pane                                                                            |
| `ctrl+w` `<label>`           | Open the link in a new pane below; focus stays here                | Open the link in the other pane; focus stays here                                                 |
| `ctrl+w` `ctrl+w`            | (nothing; clears the arm)                                          | Focus the other pane (vim alias of `ctrl+f`)                                                      |
| Mouse click in a pane        | —                                                                  | Focus that pane                                                                                   |

Every other key acts on the focused pane only.

The two split keys together express every arrangement. The **same** key means "just this
pane" (like vim's `ctrl-w o`), `q` means "not this pane", and the **other** key rotates.

Divergences from the Agents tab, chosen deliberately:

- **The same split key keeps the focused pane, not pane 0.** Pager panes are peers, and
  the pane you are reading should survive. After splitting and following links in the
  new pane, `\` gives you that pane full-screen. `q` covers the converse.
- **`+` / `-` instead of `{` / `}`.** The pager already uses `{` / `}` for history first
  and now.
- **No zoom key.** Uppercase letters are link labels. Reserving `Z` would shrink the
  label alphabet and change every labeled golden. Also, the same split key already gives
  "only this pane".

### What a new pane shows

A split opens a faithful **clone** of the focused pane, like vim's `:split`. The clone
carries:

- the same document, including pinned history versions and the read/diff view;
- the same top-of-viewport reading line (see reading-anchor);
- the same goto line mark;
- a copy of the back/forward trail.

History state is copied independently, so stepping versions in one pane never moves the
other. Already-prepared syntax highlighting is copied, so the clone's first paint is
already colored. Search state, an open goto prompt, a pending label prefix, and `y` /
`E` / `ctrl+w` arms are **not** copied.

Because the clone shows the same links, `\` followed by a label naturally means "open
this link in a split below". Walking back in the new pane returns to where you split
from.

### Focus

Exactly one pane is focused, and keys go to it.

**Only the focused pane paints link label badges.** The unfocused pane shows its links
without keys, so a label keystroke can never be ambiguous, and the yellow badges show
where your keys will land.

Losing focus cancels a pane's transient input: label prefix, armed `y` / `E` / `ctrl+w`,
the goto prompt, and search typing. Committed search highlights stay. `?` shows the
focused pane's trail and keys.

### Look (beauty spec)

**One pane: unchanged, pixel for pixel.** All current goldens must stay byte-identical.

**Two panes:** each pane is framed with a `round` border.

- **Frame color:** the accent of the pane's current section kind, from
  `section_accent()` in `_chrome.py`, which is the same color as its title glyph. Full
  strength when focused, about 35% when unfocused (the Agents deck treatment).
- **Subject line moves into the frame.** The left half (glyph, title, section title,
  history chip) becomes the border title, top-left. The right half
  (`n/N · % · ⌘ chars · syntax hint`) becomes the border subtitle, bottom-right, where
  vim keeps its ruler.
- **Title styling:** normal in the focused pane, dim in the unfocused pane.
- **Rows inside a framed pane:** the `#pager-subject` and `#pager-chrome-rule` rows are
  hidden, so a pane spends the same two chrome rows as today's single view. The trail
  band, time band, search prompt and goto prompt stay inside the frame.
- **Shared footer:** one footer sits at the bottom. In split mode its rule is hidden
  because the lower frame edge already separates it.

Sketches (Textual's exact title padding may differ):

```
Stacked, bottom pane focused (120 cols)
╭ ◆ docs/pager.md ──────────────────────────────────────────────────────────╮   accent @35%
│ ## Keys                                                                    │
│ | Key | Action | ...                                                       │
╰────────────────────────────────────────────────────── 1/3 · 42% · ⌘ 12.3Kc ╯
╭ ◆ src/sase/pager/screen.py ───────────────────────────────────────────────╮   accent @100%
│ from sase.pager._help import PagerHelpScreen [a]                           │
╰──────────────────────────────────────────────────────── 10% · ⌘ 8.1Kc · py ╯
0-9a-z follow · y copy · E edit · ^F pane · / search · ? keys · q close pane

Side by side, left pane focused
╭ ◆ docs/pager.md ────────────────╮╭ ◆ screen.py ────────────────────╮
│ ...                             ││ ...                             │
╰─────────────── 42% · ⌘ 12.3Kc ─╯╰──────────── 10% · ⌘ 8.1Kc · py ╯
0-9a-z follow · … · ^F pane · / search · ? keys · q close pane
```

### Footer and help

The footer follows the existing convention: only verbs that sometimes do nothing earn a
slot.

- **Split mode:** add `^F pane` before `/ search`, and show `q close pane` instead of
  `q close`.
- **`ctrl+w` armed:** the footer shows `^W… other pane`, like the existing `y…` and `E…`
  arm states.
- **Single-pane footer:** unchanged.

`\`, `|` and `+` / `-` appear only in `?`. The help sheet gains an always-visible
"Panes" group.

### Small windows

A split opens or rotates only when both panes get at least about 7 rows (frame plus 5
body rows) when stacked, or about 32 columns when side by side, at 50/50.

Otherwise the pager shows a toast that suggests the orientation that would fit, for
example "Not enough room to split below — try | for side by side" or "Window too small
to split".

Ratio steps that would push a pane below the minimum are clamped silently. Shrinking the
terminal never auto-closes a pane.

### Reading position

Splitting, rotating, resizing the terminal, closing a pane, and walking the trail all
keep the logical line at the top of the viewport. A width change never makes the text
jump.

### Non-goals

- More than two panes, or nested splits.
- Zoom.
- Persisting the split across pager sessions; the split is per-session.
- Synchronized scrolling.
- Drag-to-resize.
- Configurable pager keymaps.
- Any Rust/`sase_core` change.

## Architecture

**`PagerView`** (new `src/sase/pager/view.py`) is a `Vertical` widget that owns
everything per-document:

- **Mixins:** the full stack currently on `PagerScreen`, in the same MRO order.
- **State:** everything `PagerScreen.__init__` initializes today.
- **Chrome rows:** `#pager-subject`, `#pager-trail`, `#pager-time`,
  `#pager-chrome-rule`, `#pager-body-scroll` > `#pager-body`, `#pager-search-command`
  and `#pager-goto-command`.
- **Mount work:** body, trail, footer and subject; the theme watch; syntax-after-paint;
  history discovery.
- **`on_unmount`:** calls `cancel_pump_free_tasks(self)`.
- **`handle_key(event) -> bool`:** keeps today's precedence: goto prompt, then search,
  then label keys.
- **Mixin modules:** keep their `_screen_*.py` names. Renaming them is churn without
  value.

**`PagerScreen`** (`src/sase/pager/screen.py`) becomes the host:

- **Constructor:** same signature and keyword arguments.
- **Layout:** composes `#pager-root` → `#pager-panes` (a container holding one or two
  `PagerView`s) → `#pager-footer-rule` → `#pager-footer`.
- **View references:** holds direct references to its views. It never re-queries pane
  internals by ID from the screen, because of the stale `query_one` cache noted above.
- **Bindings:** `BINDINGS` keep the same keys and route per-document actions to the
  focused view. Either use a small router (for example
  `Binding("j,down", "view('scroll_down')", ...)` plus `action_view(name)`, with a test
  asserting every routed name exists on `PagerView`) or use explicit delegators.
- **`on_key`:** keeps the `self.app.screen is not self` modal guard and the existing
  order: view goto, then `?` help (unless search is typing), then view search, then view
  labels.
- **`action_close_pager` and `action_show_help`:** stay on the screen. Help reads the
  focused view's label count and trail snapshot.
- **Public conveniences:** `views`, `focused_view`, and a read-only `document` property
  that returns the focused view's document.
- **Teardown:** still calls `cancel_pump_free_tasks(self)` on unmount, because ACE's
  `_files.py` schedules copy tasks on the screen.

**Host protocol** (`PagerViewHost`, a `typing.Protocol`). Views never import
`PagerScreen`.

- In view-extract: `paint_footer(view, legend)` paints only when `view` is focused.
  `close_view(view, *, trail_exhausted)` dismisses with `PagerExit(trail_exhausted=...)`
  when single.
- Later phases add focus requests, other-pane display, and other-pane focus.
- `_update_footer` builds the legend as today but hands it to the host instead of
  querying `#pager-footer`.

**Body widgets** `PagerBodyScroll` / `PagerBody` (`_screen_widgets.py`) resolve their
**owning view** (constructor-injected, or the nearest ancestor), not `self.screen`. Each
pane's scroll height must come from its own composed body. `self.size` inside mixins
(trail and time-band row budgets, history-chip folding) becomes the pane's size. That is
intended: a short pane compacts its own chrome.

**Seams that must keep working**, because tests monkeypatch them by string path:

- `sase.pager.screen.resolve_ref`: `PagerScreen` evaluates
  `resolve_ref if resolve_ref_fn is None else resolve_ref_fn` at construction and passes
  the callable to every view.
- `sase.pager.screen.suspend_for_external_tool`, `sase.pager.screen.subprocess` and
  `sase.pager.screen.view_artifact_files`: keep `_screen_actions._screen_module()`
  pointing at `sase.pager.screen`, and keep those names imported in `screen.py` and
  listed in its `__all__`.
- `attached_handlers`: pass the **same** mapping object to every view, never a copy.
  `ace/tui/actions/hints/_files.py` inserts a handler after constructing the screen.

**CSS** (`_styles.py`): per-pane selectors become `PagerView #pager-…`, still namespaced
so ACE's stylesheet is not polluted. Footer selectors stay on `PagerScreen`.

## Extract a per-pane PagerView from PagerScreen

Phase `view-extract` is a pure refactor with no user-visible change.

1. Create `PagerView` and the host protocol as described in Architecture. Move the
   per-document code out of `screen.py`, and turn `PagerScreen` into the host. Replace
   the mixins' `self.dismiss(...)` paths (`action_close_pager`, the exhausted trail in
   `_screen_trail.py`) with host calls. Replace `self.app.screen is not self` checks
   with view-appropriate guards.
2. Re-point `PagerBodyScroll` / `PagerBody` and the `_PagerBodyHost` protocol to the
   owning view. Re-scope CSS. Keep the single-pane DOM row-for-row identical.
3. Keep every seam listed above.
4. Tests:
   - Add `pager_view(app)` (the focused view) to `tests/pager/_app_helpers.py` and
     `tests/pager/_rendered_link_pilot.py`. Switch private-state access to the view:
     `_label_layer`, `_search`, `_back_trail`, `_forward_trail`, `_body`, `_body_width`,
     `_apply_resolution`, `_goto_*`, `_syntax_*`, `_history_*`, `_trail_snapshot`,
     `_update_subject`, and the long tail.
   - Fix the local `_pager_screen` copies in `tests/pager/test_goto.py`,
     `tests/pager/test_app_goto.py`, `tests/pager/test_pager_context_identity.py`,
     `tests/pager/visual/test_app_png_snapshots.py` and
     `tests/pager/visual/test_syntax_png_snapshots.py`.
   - Fix inline `app.screen` uses in the other pager visual tests,
     `tests/pager/test_bead_live_links.py`,
     `tests/ace/tui/actions/test_view_files_pager_*.py`,
     `tests/test_bead/test_bead_show_pager.py`, and the ACE agent-metadata conversation
     test.
   - `screen.document` reads keep working through the property.
     `tests/pager/test_screen_syntax.py` (fake mixin host) should need nothing.
   - Add a test that a view's pump-free tasks are cancelled when the view unmounts.
5. Done when:
   - `sase tool run check` passes.
   - `sase tool run test-visual -- tests/pager/visual` reports **no drift**. Run it
     through `/sase_monitor` if it may outlast the turn.
   - A hand run of `sase pager <file>` behaves exactly as before: labels, `y`/`E`,
     search, goto, trail, history keys, `?` and `q`.

## Keep the reading line fixed across width changes

Phase `reading-anchor`. This is independently valuable: today a terminal resize leaves
the raw `scroll_y` row in place, so rewrapped text jumps.

1. Add a pure `ReadingAnchor(section_index, line, row_offset)` with
   `reading_anchor_at_row(body, row)` and `row_for_reading_anchor(body, anchor)`. Put
   them in `src/sase/pager/_layout.py` or a small sibling module, built on
   `ComposedBody.section_offsets` / `section_line_rows`.
   - A row on a section rule anchors to that section's start.
   - `row_offset` is clamped to the line's wrapped rows at the new width.
2. In the view's `_ensure_body`, when the composed width changes and a body already
   existed:
   - Capture the anchor at the current `scroll_y` from the old body, then recompose.
   - Restore the anchor's row via `call_after_refresh`, because `max_scroll_y` only
     reflects the new height after layout.
   - Guard the restore so a later document swap, resize or user scroll wins.
   - Track the last composed width separately from the `_body_width = None`
     force-recompose sentinel, so same-width recomposes for label repaints never move
     the scroll.
3. `PagerTrailEntry` (`src/sase/pager/trail.py`) gains an optional `reading_anchor`,
   captured in `_current_view_state()`. Trail restore prefers it over raw `scroll_y`.
   Raw `scroll_x` / `scroll_y` stay as the fallback.
4. Tests:
   - Unit tests for the helpers: wrapped lines, multi-section rules, clamping, and an
     empty document.
   - A pilot test that narrows and then widens the terminal and asserts the top logical
     line is unchanged.
   - A pilot test that a trail visit recorded at one width restores to the same line at
     another width.
   - Goldens must stay unchanged.

## Split panes with framed chrome and focus-scoped labels

Phase `split-panes` makes the feature real. Implement the UX contract above for every
key except `ctrl+w`.

1. **Pure model** in new `src/sase/pager/split.py` (no Textual imports):
   - `PagerSplitLayout` (`single` / `below` / `beside`), a frozen
     `PagerSplitState(layout, focused, ratio)`, and `RATIO_STEPS = (30, 50, 70)`.
   - Transitions: toggle (open / close-other / rotate, as in the key table), close
     focused pane, toggle focus, and step ratio.
   - Step-ratio semantics match the Agents tab: the ratio is the first pane's share,
     "grow" grows the focused pane, and the ends clamp.
   - A `split_fits(layout, ratio, width, height)` feasibility check.
   - Mirror the shapes in `decks/layout.py`, but do not import the Agents deck model.
2. **Host wiring** in `PagerScreen`:
   - Bindings: `backslash`, `vertical_line`, `ctrl+f`, `plus` and `minus`.
   - Split and close are async actions that `await` mount / `remove()` behind an
     in-flight guard, so key repeat can never double-mount or race.
   - One `_apply_split_state()` does everything:
     - the container class (`-single` / `-below` / `-beside`);
     - pane `fr` sizes from the ratio;
     - `view.set_pane_role(framed=…, focused=…)`;
     - footer-rule visibility;
     - Textual focus on the focused pane's body scroll;
     - the footer repaint.
3. **Close:** `q` / `Esc` and trail-exhausted backspace close the focused pane in a
   split. With one pane they behave exactly as today.
4. **Focus:**
   - `ctrl+f` focuses the other pane.
   - A click anywhere in a pane focuses it, via `on_click` / `on_mouse_down` in
     `PagerView` → host.
   - A `DescendantFocus` into a pane also syncs logical focus.
   - `set_pane_role(focused=False)` builds an empty label layer, exactly like
     `links_enabled=False`, and cancels transient input. `focused=True` rebuilds the
     label layer.
5. **Clone:**
   - `PagerView.split_seed()` returns a frozen `PagerViewSeed` containing:
     - the document, reading anchor and goto mark;
     - back/forward trail tuples;
     - independent copies of `_history_states` (copy each `SectionTimeState` with fresh
       `body_cache` / `comparison_cache` dicts and a fresh `expanded_folds` set);
     - `_history_supported` and `_history_view_sticky`;
     - `_syntax_prepared` and `_syntax_attempted`;
     - a copy of `_dangling_refs`.
   - Never share `SyntaxResultCache` / `StyledTextCache`: worker threads mutate those
     LRUs.
   - The new view applies the seed on mount, scrolls to the anchor after first layout,
     then starts syntax preparation and history discovery. Discovery keeps copied pins
     because it only sets `current_pin` when it is `None`.
6. **Framed chrome:**
   - Split `subject_line()` in `src/sase/pager/_chrome.py` into a public
     `subject_parts(...) -> tuple[Text, Text]` (left, right). `subject_line` keeps
     byte-identical output, and `tests/pager/test_chrome.py` keeps passing.
   - In framed mode `_update_subject` sets `border_title` (left half, fitted to the pane
     width) and `border_subtitle` (right half), dimmed when unfocused.
   - Set the border inline: `round`, the current section's accent, alpha about 0.35 when
     unfocused. Touch border, title or subtitle only when their signature changes,
     because `_update_subject` runs on every scroll.
   - Hide `#pager-subject` / `#pager-chrome-rule` when framed. The trail and time-band
     code that toggles the chrome rule (`_screen_trail.py`, `_screen_time_band.py`) must
     leave it hidden in framed mode.
7. **Footer:** `footer_legend(..., split=...)` adds `^F pane` and shows `q close pane`.
8. **Small windows:** a min-size guard and ratio clamp in the host, sized from
   `#pager-panes`, with the toast wording from the UX section.
9. **Help:** add the "Panes" group to `_binding_rows` in
   `src/sase/pager/_trail_chrome_help.py`, and update the `q, escape` row to "Close the
   focused pane (the pager when single)".
10. **Async safety:** any callback that can land on a view after an await or thread hop
    must no-op once the view is detached. That covers:
    - the `r` refresh worker's `call_from_thread`;
    - the resolve, history and copy tasks;
    - deferred `call_after_refresh` lambdas.

    Closing a pane mid-resolve, mid-refresh or mid-history-step must not raise.

11. **Tests:** add `tests/pager/test_split_model.py` (pure transitions and fit checks)
    and `tests/pager/test_app_split.py` (pilot). The pilot tests cover:
    - open below and beside;
    - the same key keeps the focused pane;
    - rotate keeps both panes' documents, focus and ratio;
    - `ctrl+f` moves focus and labels (the unfocused layer is empty);
    - `q` closes a pane, then the pager;
    - exhausted backspace closes a pane;
    - the clone matches document, anchor line, trail, pins and syntax-prepared state;
    - following a label in one pane leaves the other untouched;
    - `+` / `-` steps and clamps;
    - the too-small refusal;
    - a click focuses a pane;
    - footer verbs in both modes;
    - closing a pane with an in-flight resolve/refresh does not raise.

    Add an ACE test in the style of
    `tests/ace/tui/actions/test_view_files_pager_screen.py`: with the Agents tab under
    the pager modal, `\`, `|` and `ctrl+f` split and focus the pager, and the Agents
    deck layout stays `SINGLE`.

12. **Self-check the look before finishing:** render a 120x40 stacked split and a side
    by side split, for example through the pager visual fixture or
    `app.export_screenshot`, and inspect the PNGs against the beauty spec. Goldens are
    added in the polish-docs phase, but existing goldens must still show no drift.

## Follow a link into the other pane with ctrl+w

Phase `other-pane-follow`.

1. Add an internal arm value for "other pane" alongside `copy` / `edit`. The public
   `PendingAction` used by `AttachedTargetHandler` stays unchanged. Attached handlers
   receive `"follow"` for an other-pane arm, and never a new value.
2. Bind `ctrl+w` to arm. `normalize_jump_key` yields `"ctrl+w"`, so the doubled-press
   check in `_handle_label_key` works like `yy` / `EE`. Doubled `ctrl+w` asks the host
   to focus the other pane, and does nothing when single.
3. `ctrl+w` + label resolves exactly like follow. That includes the historical-link path
   for pinned sections, dangling caching, URL → copy, and media → viewer. The difference
   is that a document result goes to
   `host.show_in_other_view(source, document, line, end_line)`:
   - **One pane:** open a split. Use stacked if it fits, else side by side if it fits,
     else follow in place with an information toast "No room for a split — opened here".
     The new pane is a clone of the source with focus left on the source. After mount,
     it pushes its current view onto its trail and navigates, so backspace in it returns
     to the source document.
   - **Two panes:** the other pane pushes a trail entry and navigates.
   - Focus never moves.
4. Footer armed state is `^W… other pane`. The help rows are `ctrl+w…` ("Follow a
   painted link in the other pane; opens a split when single") and `ctrl+w ctrl+w`
   ("Focus the other pane").
5. Tests (pilot):
   - single: `ctrl+w` + label opens a stacked split showing the target with focus on the
     source, and backspace in the new pane returns to the source document;
   - split: the other pane navigates and gains a trail entry;
   - doubled `ctrl+w` swaps focus;
   - a URL target copies;
   - attached handlers receive `"follow"`;
   - an unresolvable target notifies and changes nothing;
   - `Esc` cancels the arm;
   - the no-room fallback.

   Extend the ACE modal test to cover `ctrl+w`.

## Split-view goldens, visual polish, and docs

Phase `polish-docs`.

1. **Goldens.** Add `tests/pager/visual/test_split_png_snapshots.py` with deterministic
   fixtures:
   - stacked at 120x40 with the bottom pane focused and labels painted. Use two
     different-kind documents, reached via a label follow, so the frames show different
     accents;
   - side by side at 120x40 with the left pane focused;
   - stacked at 60x30, for narrow-title truncation;
   - the `^W…` armed footer at 120x40.

   Generate with
   `just fix-tui-screenshots -- tests/pager/visual/test_split_png_snapshots.py` and
   verify with `sase tool run test-visual -- tests/pager/visual`. Use `/sase_monitor`
   for long runs.

2. **Beauty review against the look spec.** Fix anything off:
   - title and subtitle truncation at narrow widths;
   - frame accent legibility and unfocused contrast on dark and light themes;
   - corners where stacked frames meet;
   - no stray chrome or footer rule;
   - trail band, time band, and search/goto prompts sit inside the frame;
   - footer verb order.
3. **Docs:**
   - `docs/pager.md`: add a "Split panes" section after "Keys" covering keys, clone,
     close, focus and labels, `ctrl+w`, small windows, and position keeping. Add the new
     rows to the Keys table. Update the `Backspace` row ("an empty back trail closes the
     pane, or the pager when single") and the `q` / `Esc` row.
   - `docs/ace.md`: where it introduces the pager (`v` / `V`), add one sentence pointing
     to split panes.
   - Run `just fmt`.
4. Done when `sase tool run check` passes and the visual lane shows only the new
   goldens.

## Verification (all phases)

- **Every phase:** `sase tool run check`.
- **Phases that can affect rendering** (all except `other-pane-follow`):
  `sase tool run test-visual -- tests/pager/visual` must show no unexpected drift.
- Never run `check-full` unless explicitly instructed.
- **Manual smoke** after `split-panes` and again after `polish-docs`:
  - in `sase pager docs/pager.md`, press `\`, a label, `ctrl+f`, `|`, `+` and `-`, `q`,
    then `q` again;
  - in ACE, open a file with `v` and check that splitting inside the modal never
    disturbs the Agents tab underneath.

## Risks and gotchas

- **Pixel-identical single-pane rendering is the regression oracle** for `view-extract`.
  Any golden drift there is a bug, not an update.
- **Stale `query_one` ID cache.** Host-level queries into a pane subtree can return
  removed widgets. Hold view references instead.
- **Detached-view callbacks.** Workers and tasks can land on a closed pane. Guard them
  (split-panes step 10).
- **Shared mutable state.** Never share syntax LRU caches or `SectionTimeState` objects
  between panes. Copy them.
- **Footer ownership.** Only the focused view may paint the shared footer. Repaint it on
  every focus change.
- **Test churn in `view-extract`** is wide but mechanical. Prefer the `pager_view(app)`
  helper over ad-hoc `app.screen` digging.
- **No feature flag.** Every landed phase is complete for users: view-extract and
  reading-anchor are invisible or pure improvements, split-panes ships a finished split
  with help rows, and later phases add on top.
