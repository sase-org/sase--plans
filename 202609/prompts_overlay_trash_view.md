---
tier: tale
title:
  Prompts overlay — split-button Stash tab with a Trash view, @@ pop-newest, and a
  100-row Trash
goal:
  The Prompts overlay has two top-level tabs (Stash, History) in a polished pill-style
  tab bar. Trash is a view inside the Stash tab, opened with t or a 🗑️ chip. @ on Stash
  restores the newest draft, so @@ reliably pops the last stash. The Trash limit
  defaults to 100. Docs and glossary match.
size: medium
proposed_by: bbugyi200.apollo.28
create_time: 2026-09-27 06:53:19
status: wip
---

# Plan: Prompts overlay — split-button Stash tab, Trash view, `@@`, and a 100-row Trash

## Why

The merged Prompts overlay (`PromptsModal`) works, but its chrome is plain and uses
space badly. It has a bold `Prompts` label on its own row and a blank row under it.
Below that is a centered, grey `Stash 3 │ History │ Trash 0/20` strip and another blank
row. The Stash list is fixed at 18 rows and the whole pane at 23, so most of a tall
overlay is empty, and long prompts clip in the preview. Trash is a peer tab even though
it only holds drafts discarded from Stash. The user asked for these changes:

1. Default `ace.prompt_stash.trash_limit` becomes **100** (was 20).
2. Trash stops being a sub-tab. It becomes a **little trash icon on the Stash tab**, and
   **`t`** opens a Stash Trash view (`t` is free in every overlay pane, so `T` is not
   needed).
3. The sub-tab title bar should look **much** nicer.
4. **`@`** on the Stash sub-tab restores the first (newest) stash entry. From the main
   tabs, `@@` then pops the most recently stashed prompt when several are stashed.
5. The stash glossary entries are updated.
6. The design must be intuitive, reliable, and beautiful.

## Design

### Information architecture

- Two top-level tabs: **Stash** and **History**. `[`/`]` toggle between them (wraparound
  kept; the History filter's bracket forwarding keeps working unchanged).
- The Stash tab has two views: the **list view** (today's Stash pane) and the **Trash
  view** (today's Trash pane). Trash is presented as part of Stash, because it only ever
  holds drafts discarded from Stash.
- Keep the `PromptsTab` enum values `STASH`, `HISTORY`, `TRASH` as the overlay's
  _surfaces_. This keeps `PromptsResult.tab` routing, the app's result dispatch, and the
  `initial_tab` mapping unchanged. Rewrite the enum and module docstrings to say that
  `STASH`/`HISTORY` are the top-level tabs and `TRASH` is the Stash tab's Trash view.
  Replace `_TAB_ORDER` with `_TOP_LEVEL_TABS = (STASH, HISTORY)`.
- The Stash tab remembers its last view (`self._stash_view`, either `STASH` or `TRASH`).
  `[`/`]` returning to Stash restore that view, matching the overlay's existing rule
  that tabs keep their state across switches. Clicks are explicit: clicking the Stash
  label always shows the list, and clicking the 🗑️ chip always shows Trash.

### Keys (all hardcoded overlay keys, as today; no `default_config.yml` keymap changes)

| Key       | Stash list view                                                                                   | Trash view             | History                                    |
| --------- | ------------------------------------------------------------------------------------------------- | ---------------------- | ------------------------------------------ |
| `t`       | open Trash view                                                                                   | back to Stash list     | no-op (typing in the filter is unaffected) |
| `Esc`     | close overlay (unchanged)                                                                         | **back to Stash list** | close overlay (unchanged)                  |
| `q`       | close overlay                                                                                     | close overlay          | close overlay (outside text input)         |
| `@`       | restore the **newest** row (pin-aware, ignores staged marks like digit keys do; no-op when empty) | no-op                  | no-op (typing in the filter is unaffected) |
| `[` / `]` | switch tab                                                                                        | switch tab             | switch tab                                 |

Bind `t` and `@` (Textual key name `at`) on **`PromptsModal` itself**, next to `[`/`]`,
not on the panes. Screen-level bindings still fire when nothing inside the new screen
has focus yet. That can happen when a fast `@@` delivers the second `@` right after
`push_screen` but before `on_mount` focuses the list. The focused History filter `Input`
still consumes printable `t`/`@` before screen bindings, so typing is never hijacked.
The standalone `StashedPromptsModal` stays untouched.

`Esc` in the Trash view goes back instead of closing, which is the standard rule for
nested views. Change `TrashPane`'s bindings so `escape` maps to a new `back` action that
posts a new `TrashPane.BackRequested` message. `q` keeps `cancel`. `TrashPane` is only
hosted by the overlay, so this is safe.

### The tab bar (the "much nicer" title bar)

Replace the `Prompts` label and the generic `PanelTabStrip` with a dedicated
**`PromptsTabBar`** widget. It is one content row plus a hairline rule, laid out like
this (`[...]` = filled pill; mockups are at 120 columns):

```
Stash list view (stash 3, trash 2 of 100):
[ ≡ Stash 3 ][ 🗑️ 2 ]   ↺ History                                           t trash · [ ] tabs
──────────────────────────────────────────────────────────────────────────────────────────────

Trash view:
[ ≡ Stash 3 ][ 🗑️ Trash 2/100 ]   ↺ History                            t/esc back · [ ] tabs
──────────────────────────────────────────────────────────────────────────────────────────────

History:
 ≡ Stash 3   🗑️ 2   [ ↺ History ]                                                  [ ] tabs
──────────────────────────────────────────────────────────────────────────────────────────────
```

- **The title moves into the frame.** Set `border_title = "Prompts"` on
  `#prompts-modal-container` (as `deck_picker_modal.py` does) with
  `border-title-align: left; border-title-style: bold;`, and delete the
  `#prompts-modal-title` label and its CSS. This frees a row and reads as a finished
  window.
- **Split-button Stash tab.** The Stash tab is two attached segments, `≡ Stash N` and
  `🗑️ …`. The filled segment is the active view:
  - List view: the Stash segment is the active pill (`bold #1a1a1a on #FF87D7`). The
    trash segment is a quiet attached secondary: `#bcbcbc on #303030`, with its badge
    colored as described below.
  - Trash view: the fill moves to the trash segment (`bold #1a1a1a on #EBC04F`, the
    existing Trash amber). The label expands to `🗑️ Trash N/LIMIT` (`🗑️ Trash off` when
    the limit is 0). The Stash segment becomes the parent crumb:
    `bold #FF87D7 on #303030`.
  - History active: both Stash segments are flat, with no background and `#8a8a8a` text.
- **Stash pink matches the top bar.** Use `#FF87D7`, the same hue as the top-bar
  `stash: ≡ N` chip (`stashed_prompts_indicator._STASH_ACCENT`), and the same `≡` glyph.
  The chip the user clicked visually becomes the active tab. History uses `#5FD7FF` with
  glyph `↺`. Hex values replace the old theme-dependent `"orchid"`/`"cyan"` names.
- **Trash badge states** (inactive trash segment): limit 0 shows `off` in `#6c6c6c`.
  Count 0 shows the icon alone. `0 < N < limit` shows `N` in amber `#EBC04F`.
  `N >= limit` shows `N` in bold `#FF5F5F`, because the Trash is full and the next
  discard permanently deletes the oldest draft.
- **Right-aligned navigation hints** (widest tier only): `t trash · [ ] tabs` in Stash,
  `t/esc back · [ ] tabs` in Trash, and `[ ] tabs` in History. Keys are `#bcbcbc`,
  labels `#6c6c6c`, and separators `#4e4e4e`. The header covers "where can I go" and the
  footer keeps covering "what can I do here". The footer never listed `[`/`]` before.
- **Hairline rule.** Draw it with CSS on the bar
  (`height: 2; border-bottom: solid #3a3a3a; margin-bottom: 1;`) rather than as rendered
  text.
- **Hover and tooltips.** An inactive segment's text brightens to `#d0d0d0` under the
  pointer. Tooltips:
  - Stash: `Stash — N stashed drafts`.
  - Trash: `Trash — N of LIMIT discarded drafts kept · t`. When full:
    `Trash is full — the next discard permanently deletes the oldest draft · t`. When
    off: `Trash recovery is off (ace.prompt_stash.trash_limit: 0)`.
  - History: `History — submitted prompts`.
- **Width tiers.** Pick the widest tier whose rendered `cell_len` fits:
  - `full`: segments plus hints, with a gap of at least 2 cells before the hints.
  - `compact`: segments only.
  - `micro`: words dropped, giving `≡ N`, `🗑️ N` (active: `🗑️ N/LIMIT`), and `↺`, with
    1-space gaps.

  Until the width is known (width ≤ 0), render `full`.

- Segment text padding is one space inside each pill (`" ≡ Stash 3 "`). Separate the
  Stash split button from History with a 3-space gap (1 in `micro`). Everything is
  left-aligned from column 0.

### Body polish

Let the Stash and Trash views fill the overlay height, as History already does, instead
of the fixed 18/23-row boxes. The empty space goes away and the preview shows far more
of a long prompt. In `styles.tcss`:

- `PromptsModal #stashed-prompts-panels` and `#trash-panels` → `height: 1fr`.
- `#stashed-prompts-list` and `#trash-list` → `height: 1fr`, and drop `max-height: 18`.

List border colors stay as they are (Trash is already amber).

### Copy changes

- Stash empty-state pointer (`stash_pane_widget._stash_empty_text`): replace
  `— switch with ].` with `— press t to view.`
- Overlay Stash footer (override `_hint_text` on `StashPane`, the overlay-only pane, and
  leave the shared mixin text for the standalone picker alone): prefix line 1 with
  `@ newest · `, giving
  `@ newest · 1-9/0 restore · a all · j/k move · enter · esc/q · ^d/u`. Line 2 is
  unchanged, including the zero-limit `delete` wording.
- Trash footer: line 1 becomes
  `enter restore to Stash · 1-9/0 restore · a all · j/k move · q close · ^d/u`, because
  `Esc` now goes back and the header advertises `t/esc back`. Line 2 is unchanged.

### Reliability: make `@@` deterministic

The global `@` (`action_restore_prompt_stash`) spawns a task that reads the stash
snapshot on a worker thread before pushing the overlay. A fast second `@` can land
before the overlay exists. It then hits the app binding again, starts a **second** open,
and stacks two overlays. Close that race:

- In `PromptBarStashRestoreOverlayMixin.action_restore_prompt_stash`: if
  `self._prompts_stash_open_in_flight` is already set, set
  `self._prompts_stash_pop_newest_pending = True` and return without spawning. Otherwise
  set the in-flight flag **synchronously, before spawning**, clear the pending flag, and
  spawn a wrapper coroutine. The wrapper awaits
  `_open_prompt_stash_panel(auto_restore_single=True, honor_pending_newest=True)` and
  clears both flags in `finally`.
- Thread `honor_pending_newest: bool = False` through `_open_prompt_stash_panel` into
  `_open_prompts_overlay_async`. After the snapshot read, the limit reconcile, and the
  existing single-entry fast path, check:
  `honor_pending_newest and tab is PromptsTab.STASH and overlay.entries and self._prompts_stash_pop_newest_pending`.
  If true, consume the flag and
  `await self._apply_prompts_stash_result(single_restore_result(newest))` **instead of**
  pushing the overlay. Here `newest` is the first entry of
  `newest_first_stash_entries(overlay.entries)`. There must be **no `await` between this
  check and `push_screen`**, so any later `@` is routed to the pushed modal.
- A second `@` during the single-entry fast path is simply absorbed: one restore, no
  second open. Use `getattr(..., False)` defaults, like the mixin's other lazily created
  attributes.
- Other entry points (`,@`, `Ctrl+G p`, chip, empty `Ctrl+S`, History openers) are
  unchanged.

### Shared helpers (no duplicated ordering or pin rules)

In `src/sase/ace/tui/modals/stash_messages.py`, add and re-export from the `stash_pane`
facade:

- `newest_first_stash_entries(entries)`: sorts by `(created_at, pane_index)` in reverse.
  Use it in `_stash_controller_state.py`'s `_init_stash_controller` and
  `apply_lifecycle_snapshot`, replacing the two inline sorts.
- `single_restore_result(entry, *, pinned: bool | None = None) -> StashRestoreResult`:
  returns `keep_ids` when pinned, else `pop_ids`. `pinned` defaults to `entry.pinned`.
  `StashControllerMixin._single_restore_result` delegates to it with
  `pinned=entry.id in self._pinned`.

Add `StashControllerMixin.newest_restore_result() -> StashRestoreResult | None`, which
returns `None` when empty. `PromptsModal.action_restore_newest_stash` calls it and
dismisses **directly** with
`PromptsResult(tab=STASH, origin=self._origin, stash=result)`. It does not post through
the pane, so it works even before the pane mounts.

## Implementation steps

1. **Trash limit 100.**
   - Set `_DEFAULT_ACE_PROMPT_STASH_TRASH_LIMIT = 100` and update the docstring in
     `src/sase/ace/config.py`.
   - Set `trash_limit: 100` in `src/sase/default_config.yml`.
   - Set `"default": 100` in `src/sase/config/sase.schema.json`.
   - Change the constructor defaults from `trash_limit: int = 20` to `100` in
     `PromptsModal`, `StashPane`, `TrashPane`, and
     `StashControllerStateMixin._init_stash_controller`.
   - No sase-core change: every Rust lifecycle call already receives the limit
     explicitly from `get_ace_prompt_stash_trash_limit()`. Raising the limit never
     evicts anything.
2. **Tab bar widget.** Create the public module
   `src/sase/ace/tui/modals/prompts_tab_bar.py`, public so tests need no private-module
   import. It contains:
   - `PromptsTabBarState`: a frozen dataclass with `surface: str`
     (`"stash" | "trash" | "history"`), `stash_count`, `trash_count`, and `trash_limit`.
   - `PromptsTabBarLayout`: a frozen dataclass with `text: Text`, `tier`, and
     `hits: tuple[tuple[int, int, str], ...]`, where each hit is a cell range and its
     surface id.
   - A pure
     `layout_prompts_tab_bar(state, width, *, hover: str | None = None) -> PromptsTabBarLayout`
     implementing every rule above. Measure every fragment with `rich.cells.cell_len`,
     because the 🗑️ emoji is two cells. Define `TRASH_GLYPH = "🗑️"` (U+1F5D1 U+FE0F;
     Rich measures it as 2 cells) in `prompt_stash_row.py` next to `PIN_GLYPH`, and keep
     the `≡`/`↺` glyphs and hex accents as module constants in the tab bar.
   - `PromptsTabBar(Widget)`: `set_state()` refreshes. `render()` calls the layout with
     `self.size.width` and stores the hits. `on_click` posts
     `PromptsTabBar.SurfaceClicked(surface)` for a hit other than the active surface.
     `on_mouse_move` updates hover and the tooltip, and `on_leave` clears both.

   Keep the id `prompts-modal-tabs`. Leave the generic `PanelTabStrip` untouched; its
   other hosts are unaffected.

3. **`PromptsModal`** (`src/sase/ace/tui/modals/prompts_modal.py`):
   - Compose `PromptsTabBar` in place of the label and strip, and set the container's
     `border_title`.
   - Replace `_build_tabs`/`_refresh_tabs`/`set_active_tab` calls with one
     `_tab_bar_state()` pushed from `_sync_chrome`. This is called on activate, on
     `apply_lifecycle_snapshot`, and on `apply_store_failure`.
   - Add `_TOP_LEVEL_TABS`, `_stash_view`, and `_cycle_tab` over top-level tabs. When
     the target is Stash, it goes to `_stash_view`, and the Trash surface counts as the
     Stash position.
   - `_activate(STASH|TRASH)` records `_stash_view`.
   - Add bindings `t` → `toggle_trash_view` and `at` → `restore_newest_stash`, both with
     `show=False`.
   - Handle `PromptsTabBar.SurfaceClicked` → `_activate(tab)` and
     `TrashPane.BackRequested` → `_activate(STASH)`.
   - Drop the `PanelTab`/`PanelTabStrip` imports. Update the module and enum docstrings.
4. **`TrashPane`** (`trash_pane.py`):
   - Build `TRASH_BINDINGS` with `escape` → `back` and keep `q` → `cancel`.
   - Add the `BackRequested` message (namespace `trash_pane`, re-exported on the class
     like its siblings) and `action_back`.
   - Change the footer line 1 wording. Update the module docstring.
5. **`StashPane`**: override `_hint_text` with the `@ newest · ` prefix and update the
   empty-state pointer copy. Add the shared helpers (see "Shared helpers") in
   `stash_messages.py`, `stash_controller.py`, `_stash_controller_state.py`, and the
   `stash_pane.py` facade `__all__`.
6. **`@@` race guard** in
   `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash_restore_overlay.py`, as
   described under "Reliability". Update the `action_restore_prompt_stash` docstring.
7. **CSS** in `src/sase/ace/tui/styles.tcss`, `PromptsModal` block:
   - Add the container border-title rules and remove `#prompts-modal-title`.
   - Restyle `#prompts-modal-tabs` for the new bar (`height: 2`, `border-bottom`,
     `margin-bottom: 1`).
   - Apply the fill-height rules for the Stash and Trash panels and lists.
   - Refresh the section comment to say that Trash is the Stash tab's view.
8. **Docs.**
   - `docs/configuration.md`: the example shows `trash_limit: 100`, the table default is
     `100`, and the fallback text reads "fall back to `100`".
   - `docs/ace.md`, "Prompts Overlay" section (~line 7859): describe the two tabs, the
     tab bar, the Trash view (🗑️ chip, `t`, `t`/`Esc` back, `q` closes), and `@`/`@@`.
     Keep the per-tab state sentence and add the remembered Stash view.
   - `docs/ace.md`, Stash/Trash prose (~lines 6755–6825):
     - Replace the Stash-then-History-then-Trash tab-order text and the "switch with
       `]`" pointer.
     - Replace `Trash tab` with `Trash view` and `Trash M/N` tab wording with the 🗑️
       chip and `Trash N/LIMIT` label.
     - Change "default 20" to "default 100".
     - Rewrite the compact demo: step 1 toast plus a chip showing `🗑️ 1`, step 2 press
       `t`, step 3 `Enter` then `t`/`Esc` back to Stash.
     - Say that `@` on the Stash tab restores the newest draft, so `@@` pops the latest
       stash when several are stashed.
   - Update the `@` row in the global-keys table (~line 3502) to mention `@@`.
9. **Glossary.** The user asked for this in the prompt. Edit the strands, then run
   `sase memory init`. Keep the frontmatter and the `[[glossary:…]]` links.
   - `sase/memory/glossary/prompt-stash.md`: after the entry-point list, add that on the
     Stash tab `@` restores the newest draft, so `@@` pops the most recently stashed
     prompt. Keep the Trash sentence.
   - `sase/memory/glossary/stash-trash.md`: change "It surfaces as the Trash tab of the
     Prompts overlay" to "It surfaces as the Stash tab's Trash view in the Prompts
     overlay (the 🗑️ chip on the Stash tab, or `t`; `t`/`Esc` return to Stash)". Change
     "default 20" to "default 100". Leave the rest unchanged.

## Tests

- **New** `tests/ace/tui/modals/test_prompts_tab_bar.py`, covering the pure layout:
  - Per-surface fills and styles: which segment has the pill background.
  - Trash badge states: off, empty, count, and full (red).
  - The active Trash label `Trash N/LIMIT` and `Trash off`.
  - Hints per surface.
  - Tier ladder by width (full → compact → micro), plus `full` at width 0.
  - Hit ranges are cell-accurate around the two-cell emoji; the History range starts
    after the emoji's two cells.
  - Hover brightening.
- **New** `tests/ace/tui/modals/test_prompts_modal_trash_view.py`, a Pilot suite (reuse
  the `TrashFlowHost` pattern and `make_record` helpers):
  - `t` opens Trash and `t` returns to the list.
  - `Esc` in Trash returns to the list with the overlay still open; `q` in Trash closes.
  - Clicking the 🗑️ chip and the Stash label (offsets taken from the bar's recorded
    hits).
  - `]`/`[` toggle two tabs and restore the remembered Stash view.
  - `@` restores the newest entry (`pop_ids`), keeps a pinned newest (`keep_ids`),
    ignores staged marks, is a no-op on empty Stash, and is a no-op on the Trash and
    History surfaces.
  - `@` typed into the focused History filter lands in the filter.
  - The bar state follows `apply_lifecycle_snapshot` counts.
- **Update** existing tests that assert the old strip:
  - `tests/ace/tui/modals/test_prompts_modal.py`: the `PanelTabStrip` label and click
    tests move to the new bar and hits.
  - `tests/ace/tui/modals/test_prompts_modal_trash.py`: `Trash 2/20` labels, three-tab
    bracket cycling, and the empty-stash pointer copy.
  - `tests/ace/tui/modals/test_trash_pane.py`: footer text, and `escape` now posts
    `BackRequested`.
- **Update** the default-limit tests:
  - `tests/test_config_schema_ace.py`: rename `..._is_20` to `..._is_100` and
    assert 100.
  - `tests/ace/test_ace_prompt_stash_trash_limit.py`: bundled default and fallbacks
    are 100.
- **New** race tests in `tests/ace/tui/actions/test_prompt_stash_restore_open.py`, with
  the snapshot read blocked on a `threading.Event` so the open stays in flight:
  - Multiple entries plus two `@` presses restore the newest exactly once, push no
    `PromptsModal`, and start the snapshot read once.
  - One entry plus `@@` gives exactly one restore.
  - A failed snapshot read clears both flags, so the next `@` opens normally.
  - A single `@` still pushes the overlay.
- **PNG goldens.** Update the sentinel in
  `tests/ace/tui/visual/test_ace_png_snapshots_prompts_overlay.py` (it waits for
  `Trash 3/20`) to text from the new bar. Keep the Trash snapshots opening on
  `PromptsTab.TRASH`. Rebaseline the prompts-overlay goldens:
  - `prompts_overlay_stash_120x40`
  - `prompts_overlay_stash_narrow_100x40`
  - `prompts_overlay_trash_120x40`
  - `prompts_overlay_trash_empty_120x40`
  - `prompts_overlay_history_120x40`

  Run `just fix-tui-screenshots -- <prompts-overlay selectors>` through `/sase_monitor`,
  then inspect every changed golden. In particular, check that the 🗑️ glyph renders
  (Noto Emoji) and that pills and hints line up.

## Verification

1. `just check` (lint gates plus the scoped tests).
2. Targeted `just fix-tui-screenshots` for the prompts-overlay goldens via
   `/sase_monitor`, inspecting the report and each PNG.
3. Take a live `sase screenshot` with at least 3 stashed drafts. Drive `@` (overlay
   opens), `t` (Trash view with the amber pill), `Esc` (back), `]` (History), `[` (back
   to Stash). Inspect each PNG. Restore any real ACE deck state the capture persisted
   (see the `tui_screenshot` memory).
4. In a real tmux session, press `@@` quickly with 3+ drafts stashed. The newest draft
   lands in the prompt bar and no overlay is left open.

## Non-goals and edge notes

- No keymap-config changes: `t`, `@`, and `[`/`]` in the overlay stay hardcoded like
  every other overlay key. The global `@` keeps its config key `restore_prompt_stash`.
- `@@` is the multi-entry gesture. With exactly one draft, a single `@` already restores
  it. A second `@` pressed after that restore has finished lands in the now-focused
  prompt bar, as it does today. The race guard only absorbs a second `@` that arrives
  while the open or restore is still in flight.
- No Rust or sase-core changes. The standalone `StashedPromptsModal` and the generic
  `PanelTabStrip` are unchanged. `CHANGELOG.md` is release-please managed; do not edit
  it.
- Terminal emoji width: `🗑️` uses VS16 so Rich measures 2 cells, matching the existing
  `🖥️`/`🗺️` icons. If a terminal draws it narrower, only the bar's trailing cells shift
  by one, and hit ranges stay within wide segments.
- Environment note: when this plan was written, the host root filesystem was at 100%
  use. Make sure there is free disk space before running `just check` or the golden
  updates.
