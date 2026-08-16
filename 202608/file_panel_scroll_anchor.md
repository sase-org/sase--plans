---
tier: tale
title: Stop the ACE file panel from losing the reader's scroll position
goal:
  The ACE agent file panel keeps the reader parked on the same content across background
  refreshes, async static reads, and transient content shrinks, and remembers a per-page
  reading position instead of carrying a raw row offset between pages.
size: medium
proposed_by: bbugyi200.athena.03x
create_time: 2026-08-16 12:52:51
status: wip
---

- **PROMPT:**
  [prompts/202608/file_panel_scroll_anchor.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/file_panel_scroll_anchor.md)

# Plan

## 1. Problem

The user reports that the file panel's scroll position "frequently jumps" while they are
reading a fixed location. The suspicion is **confirmed**. It is not one bug: three
independent defects in the panel's scroll-preservation code each produce the same
symptom, and the existing `_save_scroll_position` / `_restore_scroll_position` pair
fails to prevent any of them in the cases that matter.

All findings below were reproduced against the real widget in a Textual `run_test`
harness (textual 8.0.1) driving `AgentFilePanel` inside a
`VerticalScroll#agent-file-scroll`.

### 1.1 Root cause A — the restore runs _before_ the content it is supposed to protect

`FilePanelFileListMixin._reconcile_file_list`
(`src/sase/ace/tui/widgets/file_panel/_file_list.py:291-293` and `:330-333`) and
`FilePanelFetchMixin.on_worker_state_changed`
(`src/sase/ace/tui/widgets/file_panel/_fetch.py:203-213`) all use the same shape:

```python
scroll_pos = self._save_scroll_position()
self._display_file_at_current_index()
self._restore_scroll_position(scroll_pos)
```

For static-file and static-diff (commit-diff) pages, `_display_file_at_current_index()`
does **not** render. It calls `display_static_file` / `display_static_diff`, which only
_schedule_ a thread worker (`FilePanelDisplayMixin._schedule_static_read`,
`src/sase/ace/tui/widgets/file_panel/_display.py:227`). The actual `self.update(...)`
happens later on the UI thread in `_render_static_file_result` /
`_render_static_diff_result` (`_display.py:283` and `:315`), and **neither of those
touches the scroll position at all**.

Observed ordering, instrumented on the real widget:

```
['restore-scheduled(300)', 'restore-ran(300)', 'static-render(ok)']
```

The restore fires one frame after the _old_ content is still on screen — it restores 300
to 300, a no-op — and the render that actually replaces the content is entirely
unguarded. Every plan file, artifact file, `extra_files` page and persisted commit diff
is affected.

### 1.2 Root cause B — clamp-then-forget

Textual clamps `scroll_y` to `max_scroll_y` whenever the scrollable virtual size shrinks
(`Widget._scroll_update`, `textual/widget.py:4107`). The panel renders short placeholder
bodies on ordinary transitions: `"No changes detected."` (`_display.py:93-103`),
`"Could not read file."` / `"File is empty."` (`_display.py:287-299`), `_show_loading()`
(`_state.py:12-19`), and any refresh where the live diff or the file on disk is
temporarily shorter.

Once clamped, the reader's position is gone permanently — the saved value is never
re-applied when the content grows back. Reproduced:

```
B user at: 300
B after shrink: 37        # live diff shrank
B after regrow: 37        # full content back, position NOT recovered
```

and via the static-page path, where the clamp goes all the way to the top:

```
after drastic shrink: 0
after regrow: 0 (300 = recovered)
```

### 1.3 Root cause C — the "preserved" offset is not anchored to content

`_save_scroll_position` stores a raw row offset. For a running agent the live diff is
refetched every `refresh_interval` (default 10 s, `app.py:262`) and its content changes
constantly — `git diff` orders files by path, so a newly touched file inserts a whole
block _above_ whatever the reader is looking at. Restoring the same absolute row then
slides the text under the reader by exactly the number of inserted rows. Reproduced with
40 lines prepended:

```
ANCHOR: scroll_y 150 (unchanged)
        top line before: '+line 0148 ...'
        top line after:  '+line 0108 ...'
```

`scroll_y` was "preserved" and the reader still lost their place by 40 lines. This is
the most frequent manifestation for an actively running agent.

### 1.4 Secondary defect — page switches inherit a stale row offset

Nothing resets the scroll when the visible page changes. `next_file` / `prev_file`
(`ctrl+n` / `ctrl+p`, `_file_list.py:96-120`) and the agent-switch reset path in
`_update_display_body` (`_panel.py:118-155`) leave `scroll_y` where it was, so a freshly
opened page lands mid-document at an arbitrary offset. Reproduced:

```
H4 after next_file to b.txt: 150 (0 = reset to top)
```

This is the same underlying modelling error as A–C: the scroll position is treated as a
property of the _container_ rather than of the _page being read_.

### 1.5 Ruled out

- Hiding and re-showing `#agent-file-scroll` (`display: none` via the `hidden` class in
  `_agent_detail_panels.py`) does **not** reset the offset — verified.
- Nothing anchors the file scroll to the bottom; there is no auto-scroll-to-end behavior
  on this panel.
- `ZoomFilePanel` already overrides `_get_scroll_container` to target
  `#zoom-file-scroll` (`modals/zoom_panel_widgets.py:53-57`), so the zoom modal is not
  resolving the wrong container — but it inherits every defect above.

## 2. Approach

Replace the ad-hoc save/restore call sites with a single scroll-anchor controller owned
by the file panel, built on three ideas:

1. **Restore where the content lands, not where the render is requested.** Every
   `self.update(...)` in the panel goes through one funnel that captures before and
   re-applies after. Async static reads are then covered automatically.
2. **Keep the reader's _intent_ separate from the container's current value.** The
   container's `scroll_y` may be clamped by Textual at any moment; the desired anchor is
   remembered independently and re-applied on the next render, which fixes
   clamp-then-forget without any extra bookkeeping.
3. **Anchor on rendered content, not on a row number.** Use `Widget.render_line(y).text`
   — Textual's per-width rendered-line cache — to fingerprint the row the reader is
   parked on, and relocate that fingerprint in the newly rendered body. This is exact,
   wrap-agnostic, and needs no reimplementation of Rich's wrapping.

The anchor is stored **per page slot**, which also fixes §1.4: a page the reader has
never opened starts at the top, and a page they have read before returns to where they
left off.

`Widget.render_line(y)` was verified to work on the real panel and to be served from
`Widget._render_cache` (populated once per width by `_render_content`), so the anchor
search is list indexing plus string normalization, not re-rendering. Verified relocation
across a 40-line insertion:

```
anchor at row 150 -> '+line 0148 content here'
after +40 lines above: anchor now at row 190 (expect 190)
restored top row: '+line 0148 content here'
```

### 2.1 Boundary note (Rust core)

Per `sase/memory/rust_core_backend_boundary.md` this stays in the Python TUI: scroll
anchoring is presentation state tied to Textual's rendered-line cache and
`VerticalScroll`, and no other frontend needs it to match. The one part that is arguably
content logic — fingerprint normalization and relocation — is kept in a standalone pure
module with no Textual imports so it can move later without a rewrite.

## 3. Implementation

### 3.1 New module: `src/sase/ace/tui/widgets/file_panel/_scroll_anchor.py`

Pure helpers, **no Textual imports**, so they are unit-testable in isolation:

- `GUTTER_RE` — matches the `Syntax(line_numbers=True)` gutter prefix (`^\s*\d*\s?`).
  Wrapped continuation rows render a blank gutter, so the same expression normalizes
  both.
- `normalize_row(text: str) -> str` — strip the gutter prefix and trailing whitespace.
- `@dataclass(frozen=True) ScrollAnchor` with fields:
  - `row: int` — the row offset as last observed.
  - `line_start_row_text: str | None` — normalized text of the nearest row at or above
    `row` that begins a source line (non-blank gutter).
  - `sub_offset: int` — `row - <row of that line start>`, so a reader parked on a
    wrapped continuation row lands back on the same continuation row.
  - `content_digest: str | None` — digest of the body content the anchor was taken from,
    used to take the exact fast path when the content did not change.
- `capture_anchor(row, row_texts, gutter_present, content_digest) -> ScrollAnchor` —
  `row_texts` is a `Sequence[str]` of raw rendered row texts; walk up from `row` to the
  nearest source-line start (or, when no gutter is present, use `row` itself with
  `sub_offset = 0`).
- `resolve_anchor(anchor, row_texts, gutter_present, content_digest, search_window=512) -> int`
  — if `content_digest` matches the anchor's, return `anchor.row` unchanged (exact fast
  path). Otherwise scan outward from `anchor.row` in nearest-first order, bounded by
  `search_window`, for a row whose normalized text equals `line_start_row_text` **and**
  (when a gutter is present) which is itself a source-line start; return that row plus
  `sub_offset`. Fall back to `anchor.row` when nothing matches.
- `AnchorStore` — a bounded LRU (`OrderedDict`, `max_entries=64`) keyed by the page-slot
  key described in §3.2.

### 3.2 Panel state (`_panel.py`, `_content.py`)

Add to `AgentFilePanel.__init__` and declare on `FilePanelContentMixin`:

- `_scroll_anchors: AnchorStore`
- `_anchor_slot_key: tuple[object, str] | None` — `(agent_identity_or_None, slot)` for
  the page currently rendered. Track the identity in a dedicated
  `_anchor_agent_identity` field that is updated by `update_display` / `refresh_file` /
  `_reconcile_file_list` but is **not** cleared by `set_file_list` (which deliberately
  nulls `_current_agent`, `_file_list.py:66`), so a static file list keeps a stable key.
- `_last_applied_scroll_row: int | None` — the post-clamp value the controller itself
  last wrote, used to distinguish "the reader moved" from "Textual clamped us".
- `_anchor_restore_pending: bool` — coalesces restores so one render never schedules two
  `call_after_refresh` callbacks.

### 3.3 The render funnel (`_content.py`)

Add
`FilePanelContentMixin._update_body(self, renderable, *, anchor: bool = True) -> None`:

```
def _update_body(self, renderable, *, anchor=True):
    if anchor:
        self._capture_scroll_anchor()
    self.update(renderable)
    if anchor:
        self._schedule_scroll_anchor_restore()
```

`_capture_scroll_anchor()`:

- resolve the container; return if `None` or if `_anchor_slot_key is None`;
- `observed = int(container.scroll_y)`;
- if `_last_applied_scroll_row is None or observed != _last_applied_scroll_row`, the
  reader moved: build a fresh `ScrollAnchor` from the **currently rendered** rows
  (`self.render_line(y).text` for the window needed by `capture_anchor`) and store it
  under `_anchor_slot_key`;
- otherwise keep the stored anchor untouched — this is what survives a clamp.

`_schedule_scroll_anchor_restore()` sets `_anchor_restore_pending` and calls
`self.call_after_refresh(self._apply_scroll_anchor)`.

`_apply_scroll_anchor()` (runs post-layout, so the new virtual size and the new
rendered-line cache are both valid):

- clear `_anchor_restore_pending`;
- look up the anchor for `_anchor_slot_key`; when absent, target row `0`;
- gather `row_texts` for the bounded search window via `self.render_line`, resolve the
  target row, `container.scroll_to(y=row, animate=False)`;
- set `_last_applied_scroll_row = int(container.scroll_y)` (the post-clamp actual, which
  is what makes the clamp-vs-reader test in `_capture_scroll_anchor` correct).

Record the content digest alongside the anchor whenever `_full_content` is set, so
`resolve_anchor` can take its exact fast path when content is unchanged.

Add `_note_slot_change(new_key)`: capture the current position into the _outgoing_ key,
set `_anchor_slot_key = new_key`, and reset `_last_applied_scroll_row = None` so the
next restore uses the incoming page's remembered anchor (or the top).

### 3.4 Route every render through the funnel

Replace all 16 `self.update(...)` call sites in the file panel package:

- `_content.py:90,98,101,103` (`_render_full_content`, all four content modes)
- `_display.py:102` (no-changes placeholder), `:189`
  (`display_linked_diff_unavailable`), `:289,296` (static file missing/empty),
  `:321,328` (static diff missing/empty), `:369` (`_display_static_image`), `:388`
  (`_display_static_video`)
- `_file_list.py:262` (`_display_commit_diff_unavailable`)
- `_fetch.py:219` (error state)
- `_state.py:19` (`_show_loading`), `:29` (`show_empty`)

`show_empty()` clears `_anchor_slot_key` (no agent selected) and passes `anchor=False`.
Every other site goes through `_update_body` with anchoring on, including the short
placeholders — they clamp to the top, but the desired anchor survives and is re-applied
on the next real render, which is precisely the §1.2 fix.

### 3.5 Delete the old mechanism

Remove `_save_scroll_position` / `_restore_scroll_position` from `_content.py` and the
four call-site pairs in `_file_list.py:291-293`, `_file_list.py:330-333` and
`_fetch.py:203-213`. Keep `_get_scroll_container` — the funnel and `_image_preview_size`
(`_display.py:399`) both need it, and `ZoomFilePanel` overrides it.

### 3.6 Wire slot changes

Call `_note_slot_change(...)` from:

- `next_file` / `prev_file` (`_file_list.py:96,109`) — before
  `_display_file_at_current_index()`.
- `_reconcile_file_list` (`_file_list.py:276`) — when the resolved current slot value
  changes.
- `_update_display_body` (`_panel.py:81`) — on the different-agent / full-reset path and
  on the same-agent path where the preserved slot is re-resolved.
- `set_file_list` (`_file_list.py:51`) — after the file list is replaced.

`ZoomFilePanel` inherits all of this; confirm no override in
`modals/zoom_panel_widgets.py` bypasses the funnel, and that its `#zoom-file-scroll`
container is used for both capture and restore.

## 4. Tests

New file `tests/ace/tui/widgets/file_panel/test_scroll_anchor.py` (Textual `run_test`
harness mirroring the reproductions above; a `VerticalScroll#agent-file-scroll` hosting
an `AgentFilePanel`, size `(100, 30)`):

1. **Async static render preserves position** — scroll a static page, trigger a
   re-render whose content is unchanged, assert the position holds _after the worker
   result lands_ (this fails on `master`; the restore currently runs before the render).
2. **Transient shrink then regrow recovers** — park at row 300, render a much shorter
   body, render the full body again, assert the position returns to 300. Cover both the
   live-diff path and the static-file path.
3. **Content inserted above keeps the reader on the same text** — the §1.3 scenario:
   assert the normalized top row text is identical before and after 40 lines are
   prepended, and that `scroll_y` moved by 40.
4. **Reader movement wins over a stale anchor** — scroll after a render has been
   requested; assert the controller adopts the new position instead of yanking back.
5. **Per-page anchors** — read page A at row 150, `next_file` to a never-opened page B,
   assert B starts at row 0; `prev_file` back to A, assert row 150.
6. **Placeholders do not destroy the anchor** — render `"No changes detected."` then the
   full diff again, assert recovery.
7. **Zoom modal** — extend the existing `tests/ace/tui/test_agents_zoom_panel_files.py`
   coverage with one case proving `ZoomFilePanel` anchors against `#zoom-file-scroll`.

New file `tests/ace/tui/widgets/file_panel/test_scroll_anchor_helpers.py` — pure unit
tests for `_scroll_anchor.py`: gutter normalization (numbered rows, blank continuation
rows, no-gutter plain path), `capture_anchor` walking up to a line start,
`resolve_anchor` exact fast path on an unchanged digest, nearest-first relocation,
`sub_offset` round-trip on wrapped rows, and the bounded-window fallback returning
`anchor.row` when nothing matches.

**Acceptance bar for wrapped content:** exact restoration is required for unwrapped
bodies (diffs, code and markdown at normal panel widths). For wrapped bodies with
repetitive continuation rows the anchor may land on a nearby row; the test asserts only
that the result is no worse than the raw row offset the current code would produce.

## 5. Verification

- `just install` first (ephemeral workspace).
- `just check` during development.
- `just check-full` before landing, run through `/sase_monitor` with a `--next` action —
  this change touches a widely imported TUI widget package.
- `just test-visual` if any PNG snapshot covering the agents detail pane shifts; goldens
  live in `tests/ace/tui/visual/snapshots/png/`.
- Review `sase/memory/tui_perf.md` before finalizing: the anchor search runs inside
  `call_after_refresh`, is bounded to a 512-row window, and is skipped entirely when the
  content digest is unchanged, but the implementer should confirm no regression in the
  j/k navigation benchmark (`tests/ace/tui/bench_tui_jk.py`).

## 6. Out of scope

- The prompt panel (`#agent-prompt-scroll`) and tools panel (`#agent-tools-scroll`) have
  the same class of problem but are not touched here; file a follow-up task bead via
  `/sase_new_task` if the reader wants them aligned.
- `_render_static_file_result` never sets `_last_file_content` (`_display.py:283-313`),
  so `get_current_content()` returns stale text for static pages. Real, but unrelated to
  scrolling — file a separate task bead rather than folding it in.
