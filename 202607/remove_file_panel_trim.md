---
create_time: 2026-07-12 12:43:38
status: wip
prompt: 202607/prompts/remove_file_panel_trim.md
tier: tale
---
# Plan: Stop Buffering File Panel Lines (Remove Trim System)

## Goal

The agents-tab file panel (and its zoom-modal variant) currently buffers file/diff content to the viewport height,
appending a "▾ N more lines below" indicator and requiring the user to expand content interactively (`=` show-all, `-`
reset, `ctrl+d`-at-bottom auto-expand). Remove this buffering entirely: the panel should always display the full
content, natively scrollable in its existing `VerticalScroll` container, **with no performance regression**. Remove all
trim-related keymaps (default keys are `-` = `reset_file_trim` and `=` = `show_all_file_lines`).

## Background: why trim exists and why it is now safe to remove

Trim was added in `feat: Add automatic content trimming to file panel` (sase-614.1) to bound render cost, _before_ the
Phase 6 lazy-syntax caps existed. The performance landscape has changed:

1. `src/sase/ace/tui/util/lazy_syntax.py` already caps syntax highlighting at 64KB / 1,500 lines (24KB / 600 for
   markdown); larger content falls back to plain `Text` with a notice.
2. Rich `Syntax` lexes the **entire** content even when `line_range` trims the output — today's trimmed render already
   pays full lexing cost. `line_range` only bounds wrapping/segment generation. So rendering full content for under-cap
   files adds only the (bounded) wrap cost of below-viewport lines.
3. Textual `Static` widgets render content into a cached strip list once per content/size change; scrolling replays
   cached strips. Full render is a one-time-per-update cost, not per-frame.

The only genuinely unbounded cost of untrimmed rendering is laying out a plain `Text` of a pathologically huge file
(e.g. a 100k-line lockfile diff), which is O(total lines) on the UI thread. That case gets a large safety cap (see
Design §2b) rather than the current viewport-height buffering.

## Current implementation map

**Core trim machinery** (all in `src/sase/ace/tui/widgets/file_panel/`):

- `_trim.py` — `FilePanelTrimMixin`: `_compute_trim_size` (viewport height − 4), `_apply_deferred_trim` (retry after
  layout when container was hidden), `_render_trimmed_content`, `_check_trim_overflow` (iterative word-wrap overflow
  correction via `call_after_refresh` loops), `expand_by_page`, `collapse_by_page` (already dead code — no callers),
  `reset_trim`, `show_all_lines`, `is_trimmed`, scroll save/restore, `_update_timestamp_header` (in-place header string
  surgery), `_render_full_content`.
- `_display.py` — four display paths each duplicate the trim-vs-full branch and the "▾ N more lines below" indicator:
  `_display_file_with_timestamp` (live diff), `display_linked_diff`, `_render_static_file_result`,
  `_render_static_diff_result`.
- `_fetch.py` — refresh paths branch on `_is_trimmed` to pick trimmed/full re-render.
- `_panel.py` — trim state fields (`_total_line_count`, `_visible_line_count`, `_base_trim_size`, `_is_trimmed`).
- `_messages.py` — `FileTrimChanged(visible_lines, total_lines, is_trimmed)`.

**Consumers / keymaps / UI:**

- `src/sase/ace/tui/bindings.py:122-123` — `minus` / `equals_sign` bindings.
- `src/sase/default_config.yml:150-152` — `# File trim` keymap entries.
- `src/sase/ace/tui/keymaps/types.py` — `reset_file_trim` / `show_all_file_lines` fields + display-label entries (~lines
  107-108, 327-328).
- `src/sase/ace/tui/commands/_app_metadata.py:257-258` — command palette entries.
- `src/sase/ace/tui/actions/agents/_panel_detail.py` — `action_reset_file_trim`, `action_show_all_file_lines`.
- `src/sase/ace/tui/actions/navigation/_basic.py` — `action_scroll_detail_down` auto-expands a page when
  at-bottom-and-trimmed; `action_scroll_to_bottom` calls `show_all_file_lines()` before scrolling.
- `src/sase/ace/tui/widgets/agent_detail.py` — `expand_file_trim`, `reset_file_trim`, `show_all_file_lines`,
  `is_file_trimmed` wrappers; `toggle_layout` re-trims after layout.
- `src/sase/ace/tui/widgets/_agent_detail_panels.py` — `on_file_trim_changed` → border subtitle ("Lines 1-N of M" vs "M
  lines"); `reset_trim` hooks in `_apply_panel_mode` and `on_file_visibility_changed`.
- Zoom modal: `zoom_panel_modal.py` (bindings + action shims), `zoom_panel_navigation.py` (`action_show_all_file_lines`
  / `action_reset_file_trim`), `zoom_panel_events.py` (`on_file_trim_changed` subtitle), `zoom_panel_content.py`
  (reset-trim retry loop for hidden modal layout).
- Help modal: `src/sase/ace/tui/modals/help_modal/agents_bindings.py:92-93`.
- Docs: `docs/ace.md` keymap table rows for `-` and `=`.
- Footer (`keybinding_footer.py`): no trim entries exist — nothing to do there.

## Design

### 1. Always render full content; delete the trim state machine

- All four display paths in `_display.py` render the complete content through `lazy_renderable` (no `line_range`), with
  the existing header/banner/cleanup composition. Remove the indicator `Text` everywhere.
- Delete `FilePanelTrimMixin`'s state machine: `_compute_trim_size`, `_apply_deferred_trim`, `_render_trimmed_content`,
  `_check_trim_overflow`, `expand_by_page`, `collapse_by_page`, `reset_trim`, `show_all_lines`, `is_trimmed`. Keep
  (relocated/renamed as appropriate): `_count_lines`, the single full-content render helper, scroll save/restore,
  `_get_scroll_container` (still used for image preview sizing and scroll restore), and a renamed `_reset_content_state`
  (was `_reset_trim_state` — still needed by `_state.py` / `_file_list.py`). With the mixin reduced this may fold into
  `_display.py`; keep whatever split reads best.
- `_full_content` and `_total_line_count` **stay** — they back the zoom-modal search corpus, `get_current_content()`
  (editor export via `E`), refresh-time re-renders, and the "N lines" border subtitle.
- A hidden/not-yet-laid-out container no longer matters: rendering no longer depends on viewport height, so all the
  deferred-trim / re-trim-after-layout hooks disappear (`toggle_layout`, `_apply_panel_mode`,
  `on_file_visibility_changed`, `zoom_panel_content.py`'s retry loop, `zoom_panel_content.py` freeze comment). This
  deletes a whole class of `call_after_refresh` re-render churn — a small perf _win_ on panel-mode and layout changes.

### 2. Performance guardrails (the "no performance cost" contract)

a. **Existing highlight caps stay as-is.** `lazy_renderable` now measures the full content (no `line_range`),
so >1,500-line / >64KB files take the plain-`Text` path with the existing "Large output rendered without syntax
highlighting" notice. Under-cap files pay the same lexing cost they already pay today.

b. **New pathological-size render cap (safety valve, not paging).** Rendering a plain `Text` of unbounded size on the UI
thread violates the never-block-the-event-loop rule. Add a single large constant in `lazy_syntax.py` (e.g.
`FILE_PANEL_MAX_RENDER_LINES`, order of 10k–20k lines — final value chosen by profiling, see §6). Content above it
renders the first N lines plus one tail notice: `"… M more lines — press E to open in editor"`. There are **no keymaps**
to grow it; `E` (edit panel) is the escape hatch. In practice essentially all real diffs/files render fully; this only
guards lockfile-scale monsters that today would also require several `ctrl+d` expansions to view.

c. **Keep steady-state refreshes cheap.** The live-diff path re-renders every ~10s auto-refresh tick even when only the
`# Last fetched: HH:MM:SS` header changed (the header is currently string-spliced into `_full_content`, defeating any
content-keyed caching). Restructure:

- Render the timestamp/refresh-indicator header as a separate small `Text` element in the `Group` instead of prepending
  it into the content string (deletes `_update_timestamp_header`'s string surgery).
- Give the file panel a per-panel render cache for the content body, reusing the existing `LazySyntaxRenderCache`
  machinery (extended or wrapped so the over-cap plain-`Text` body is cached the same way — a 1–2 entry cache is
  enough). Timestamp-only ticks and unchanged-content refreshes then replay cached segments instead of
  re-lexing/re-wrapping.
- Keep `_last_file_content` change detection exactly as-is (it compares the raw diff, which no longer embeds the
  timestamp — verify the first-load `None`/visibility edge cases still hold).

d. **No behavior change to detail-panel debouncing or off-thread reads** — file reads/diff fetches stay in thread
workers; rendering stays on the UI thread post-debounce, same as today.

### 3. Message and subtitle simplification

- Replace `FileTrimChanged(visible_lines, total_lines, is_trimmed)` with
  `FileLineCountChanged(visible_lines, total_lines, capped)` (exported from `widgets/file_panel/__init__.py` and
  `widgets/__init__.py`; old name removed).
- `_agent_detail_panels.py::_update_file_scroll_subtitle` and `zoom_panel_events.py::on_file_trim_changed` (renamed
  accordingly): subtitle shows `"M lines"` normally; when the §2b safety cap engaged, show
  `"1-N of M lines (E: editor)"` in the existing dim style so truncation is never silent.
- The screenshot's `Lines 1-18 of 52` subtitle state disappears for all normal files.

### 4. Keymap / action removal

Remove `reset_file_trim` and `show_all_file_lines` end to end:

- `src/sase/default_config.yml` — delete the `# File trim` block (this is the required default-keymap-config update).
- `src/sase/ace/tui/bindings.py` — delete both `Binding`s.
- `src/sase/ace/tui/keymaps/types.py` — delete the two dataclass fields and the two display-label tuples. The loader
  already warns-and-ignores unknown keymap actions in user config (`keymaps/loader.py`), so users with these keys in
  `sase.yml` degrade gracefully to a log warning — no migration shim needed.
- `src/sase/ace/tui/commands/_app_metadata.py` — delete both command-palette entries.
- `src/sase/ace/tui/actions/agents/_panel_detail.py` — delete both `action_*` methods.
- `src/sase/ace/tui/widgets/agent_detail.py` — delete `expand_file_trim`, `reset_file_trim`, `show_all_file_lines`,
  `is_file_trimmed`.
- Zoom modal — delete the `equals_sign`/`minus` `Binding`s and action shims in `zoom_panel_modal.py`, and the two
  actions in `zoom_panel_navigation.py`.
- `src/sase/ace/tui/actions/navigation/_basic.py` — `action_scroll_detail_down` loses the at-bottom auto-expand branch
  (plain half-page scroll everywhere); `action_scroll_to_bottom` loses the `show_all_file_lines()` branch (plain
  `scroll_end`).
- Help modal (`help_modal/agents_bindings.py`) — remove both rows (keeps the help popup in sync per the ace guidelines;
  box widths unaffected by row removal).
- `docs/ace.md` — remove the `-` and `=` rows from the agents keymap table.
- Net effect: the `-` and `=` keys become free on the agents tab and in the zoom modal.

### 5. Test plan

- **Update existing tests** that exercise or stub trim internals: `tests/test_file_panel.py` (e.g.
  `test_linked_diff_trim_rerender_keeps_banner` becomes a full-render banner test; fixture stubs of
  `_base_trim_size`/`_is_trimmed`/`_compute_trim_size` removed), `tests/ace/tui/test_agents_zoom_panel_files.py` (trim
  assertions → full-content assertions), `tests/ace/tui/test_agents_zoom_panel_search.py` (the "search sees below-trim
  content" test simplifies: rendered content now equals the corpus),
  `tests/ace/tui/test_file_panel_selection_preserved.py`, `tests/ace/tui/widgets/file_panel/test_async_static_reads.py`,
  `tests/ace/tui/widgets/file_panel/test_image_preview.py`.
- **New coverage:**
  - Each display mode (live diff, linked diff, static file, static diff) renders all lines with no "more lines below"
    indicator and posts the new line-count message.
  - Safety cap: over-cap content renders exactly N lines + tail notice; subtitle reflects the capped range.
  - Timestamp-only auto-refresh reuses the cached body renderable (assert no re-highlight, e.g. via cache hit count) and
    preserves scroll position.
  - `G` / `ctrl+d` perform plain scrolls (no trim expansion side effects).
  - Bindings/help/palette/keymap-schema no longer reference the removed actions.
- **Visual snapshots:** run `just test-visual`; file-panel and zoom-modal scenes (e.g. `agents_file_zoom_modal`,
  `agents_linked_repo_diff_file_panel`) will change where the indicator/subtitle appeared — inspect diffs in
  `.pytest_cache/sase-visual/` and accept intentional changes with `--sase-update-visual-snapshots`.

### 6. Performance validation (required before landing)

Per the TUI-perf playbook: measure, don't guess.

- Choose the §2b cap constant by profiling full plain-`Text` renders at several sizes (`sase ace --profile`, `tui_trace`
  spans around `widget.file_panel.update_display`) so a cap-sized render stays within a normal detail-panel paint
  budget.
- Before/after: `SASE_TUI_PERF=1` j/k sweeps over agents with large diffs (p95 < 16 ms must hold) and
  `pytest -s -m slow tests/ace/tui/bench_tui_jk.py`.
- Verify the 10s auto-refresh tick on a large visible diff does not regress (this is where §2c earns its keep).

## Out of scope

- Tools-panel detail levels (`h`/`l`/`H`/`L`) and prompt-panel stash/rendering — unrelated progressive-disclosure
  systems, untouched.
- Axe output tail-capping (`cap_ansi_output`) — different surface, unchanged.
- No Rust core (`sase-core`) changes: this is presentation-only Textual rendering.

## Risks / notes

- Biggest risk is the over-cap plain-text path: the cap constant must be validated empirically, not assumed. If
  profiling shows even ~10k-line plain renders are too slow on target hardware, lower the cap — the escape hatch (`E`)
  keeps full content reachable.
- The timestamp-header separation (§2c) touches change-detection and zoom-search corpus assembly; verify search still
  indexes full content and "content unchanged" refreshes still skip visibility-message churn.
- Users with `reset_file_trim`/`show_all_file_lines` in personal keymap config get a benign "unknown keymap action
  (ignored)" log warning after upgrade.
