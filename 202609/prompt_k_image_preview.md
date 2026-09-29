---
tier: tale
title: Prompt K previews image file paths inline
goal:
  In prompt NORMAL mode, K on a supported image path opens the preview reader with an
  inline cell-rendered image (Z escalates to the artifact viewer). Today it shows a
  "Cannot preview binary file" warning.
size: medium
proposed_by: bbugyi200.apollo.2y
create_time: 2026-09-29 08:37:39
status: wip
---

# Plan: Prompt NORMAL-mode `K` previews image file paths inline

## Goal

In the prompt input widget's NORMAL mode, pressing `K` with the cursor on a supported
image path (`.png`, `.jpg`, `.jpeg`, `.webp`, `.gif`, case-insensitive) should open the
preview reader showing the image itself, rendered inline as a portable Pillow cell
preview. Today it only shows a warning toast.

## Current behavior (root cause)

- `K` dispatches from `src/sase/ace/tui/widgets/_vim_normal.py` to
  `PromptPreviewMixin._preview_token_under_cursor`
  (`src/sase/ace/tui/widgets/_prompt_preview.py`). That method detects the token and
  resolves it off-thread with `resolve_preview_target`
  (`src/sase/ace/tui/widgets/_prompt_preview_target.py`), then pushes
  `PreviewPanelModal(payload)`.
- For file tokens, `_resolve_file_preview` always calls `_read_text_file`. That call
  sniffs for NUL bytes and raises `PreviewError("Cannot preview binary file: …")`, so
  `K` on an image path only toasts a warning.

## How the rest of the TUI previews images (the inspiration)

- **Inline Pillow cell previews.** The agent file panel
  (`widgets/file_panel/_display.py::_display_static_image`) and the notification modal's
  attachment pane (`modals/notification_modal_attachments.py::_attachment_body`) both
  work the same way:
  1. Gate on `is_supported_image_path(path)`.
  2. Size the preview to the visible scroll viewport with
     `image_preview_size_for_viewport(scroll_widget=…, content_widget=…)`.
  3. Render it with `image_preview(path, image_render_context(), columns=…, rows=…)`
     from `sase.ace.tui.graphics`. That call returns a `CellImageRenderable`, or an
     `ImageFallbackRenderable` that never raises.
- **Full-fidelity terminal viewer.** `Ctrl+]` on an image path
  (`_prompt_jump.py::_view_jump_image`) and the preview reader's `Z` key
  (`modals/_source_file_actions.py::action_open_in_viewer` → `open_artifact_path`) hand
  the file to the kitty or tmux artifact viewer.

## Design decision

`K` opens the preview reader with an inline cell-rendered image. `Z` inside the reader
escalates to the full-fidelity artifact viewer, and `Ctrl+]` stays exactly as it is (it
already opens images directly in the viewer).

Rationale:

- Every other `K` target opens an in-TUI reader you dismiss with `Esc`.
- `Ctrl+]` already covers "open in the real viewer", so making `K` do the same would
  duplicate it.
- The inline preview matches the file panel and notification modal, and doesn't suspend
  the TUI.

Implementation shape: a dedicated `ImagePreviewPanelModal` **subclass** of
`PreviewPanelModal`, in a new module. We don't add branches inside the text reader,
because:

- `preview_panel_modal.py` is already about 580 lines, and the `toobig` gate warns
  at 700.
- Search, Markdown rendering, properties, and content-aware text geometry don't apply to
  images.

The subclass inherits all TCSS, because Textual type selectors follow the first DOMNode
base class. It also inherits the title, the path, copy-path and viewer actions, the
copy-mode `%` forwarding, and teardown.

## Changes

### 1. `PreviewPayload` gets a media discriminator

In `src/sase/ace/tui/widgets/_prompt_preview_target.py`:

- Add `PreviewMedia = Literal["text", "image"]`.
- Add a trailing field `media: PreviewMedia = "text"` to the frozen `PreviewPayload`
  dataclass. The default keeps every existing constructor unchanged: the artifacts
  Files, beads, plans, and alias-history callers.

### 2. The resolver returns an image payload instead of reading bytes

In `_resolve_file_preview`, keep the existing checks in their current order: exists,
then `is_dir`, then `is_file`. They keep producing the same "File not found" / directory
/ non-file errors. Then add a new check **before** `_read_text_file`:

- If `is_supported_image_path(resolved)` is true, return a payload with:
  - `kind_label="image"`, `icon="@"`, `title=token.raw`
  - `source_path=str(resolved)`
  - `content=""`, `lexer="text"`, `media="image"`
- Import it from the lightweight `sase.ace.tui.graphics.images` module, not the package
  root, so the resolver doesn't pull in viewer modules.
- The resolver does no image decoding. It already runs inside `asyncio.to_thread`.
- Non-image binaries (for example `blob.bin`) still raise "Cannot preview binary file".

### 3. The image fallback can carry a surface-specific hint

`ImageFallbackRenderable` (`src/sase/ace/tui/graphics/renderable.py`) hard-codes "Open
artifact with A". That key is wrong inside the preview reader.

- Add a trailing dataclass field `hint: str = "Open artifact with A"` and render it in
  place of the literal.
- Add a keyword-only `fallback_hint: str | None = None` to `image_preview(...)`. When it
  is set, every fallback that function builds uses it.
- Existing callers and the existing `test_image_file_previews.py` assertion stay
  unchanged.

### 4. New `ImagePreviewPanelModal`

Create the new module `src/sase/ace/tui/modals/preview_panel_image_modal.py` with
`class ImagePreviewPanelModal(PreviewPanelModal)`, exported in `__all__`.

Import `image_preview`, `image_preview_size_for_viewport`, and `image_render_context` at
module level from `sase.ace.tui.graphics`, the same as the notification modal, so tests
can monkeypatch `preview_panel_image_modal.image_preview`.

State, set in `__init__` after `super().__init__(payload)`:

- `_image_renderable: RenderableType | None = None`
- `_image_request_id: int = 0`
- `_image_requested_size: tuple[int, int] | None = None`

Overrides:

- **`_build_content()`**: returns `Text("Loading image…", style="dim italic")` until the
  first render lands, then the stored renderable.
- **`_build_footer()`**: returns exactly `"Y path | % copy | Z viewer | esc close"`.
  - There are no scroll, search, or `y` hints, because the image is sized to fit the
    viewport.
  - Image payloads always have a `source_path`.
- **`_preview_apply_geometry(self, *, reset_floor: bool = False)`**:
  - Replaces the text-measuring geometry.
  - Reads `self.app.size` (return early if unattached or the size is not positive).
  - Honors `reset_floor` like the base (set `_geometry_floor = None`).
  - Applies `max_geometry(screen_w, screen_h)` through the inherited
    `_preview_set_container` (it keeps the grow-only floor and `_last_geometry_screen`).
  - Then calls `self.call_after_refresh(self._schedule_image_render)` so sizing uses the
    post-layout viewport.
  - This single hook covers both first mount (`PreviewPanelModal.on_mount` calls it) and
    terminal resizes (`PreviewPanelGeometryMixin.on_resize` resets the floor and calls
    it).
  - **Do not define `on_mount` / `on_resize` / `on_unmount` in the subclass.** Textual
    dispatches handlers from every class in the MRO, so redefining them (especially with
    `super()` calls) double-runs base logic. The inherited `on_unmount` already calls
    `cancel_pump_free_tasks(self)`.
- **`_schedule_image_render()`**: must stay thin and synchronous, because it runs as a
  pump callback.
  1. Return if not attached.
  2. Compute `(columns, rows)` with
     `image_preview_size_for_viewport(scroll_widget=<#preview-scroll VerticalScroll>, content_widget=<#preview-content Static>)`.
  3. Return early if that equals `_image_requested_size`.
  4. Otherwise store it, increment `_image_request_id`, and launch
     `self._render_image(request_id, columns, rows)` via
     `spawn_pump_free_task(self, ..., name="sase-preview-render-image", registry_attr="_pump_free_async_tasks")`.
  5. Like the base Markdown render, don't do slow awaits on the pump (TUI perf rules 1
     and 2).
- **`async _render_image(request_id, columns, rows)`**:
  1. Decode off the UI thread with
     `await asyncio.to_thread(image_preview, path, image_render_context(), columns=columns, rows=rows, fallback_hint="Press Z to open it in the artifact viewer")`.
     The decode warms `_cell_image_lines`' LRU cache, so paint-time renders are cache
     hits.
  2. After the await, drop the result if `request_id != self._image_request_id` or the
     modal is no longer attached (perf rule 4).
  3. Otherwise store it and update `#preview-content`.
- **Action overrides**, each a `notify(..., severity="warning")` that performs no other
  side effect:
  - `action_open_search` (`/`): "Search is not available for image previews".
  - `action_copy_contents` (`y`): "Image previews have no text to copy; press Y to copy
    the path".
  - `action_open_in_editor` (`o`): "Images open in the artifact viewer; press Z".
- **Inherited unchanged:**
  - `Z` → `open_artifact_path`
  - `Y` copy path, `%` copy mode, `q` / `Esc` close
  - `R` (already warns "not Markdown", since the lexer is `text`)
  - `p` (already warns no properties)
  - `n` / `N` (no-ops with no matches) and the scroll keys

### 5. Route image payloads to the new modal

In `PromptPreviewMixin._resolve_preview_async`, after the existing staleness check:

- If `payload.media == "image"`, lazily import and push
  `ImagePreviewPanelModal(payload)`.
- Otherwise push `PreviewPanelModal(payload)` exactly as today.
- Keep the lazy-import style that is already used there.

No keymap config changes: `K` is hard-wired in `_vim_normal.py`, not in
`default_config.yml`.

### 6. Docs

Update `docs/ace.md`:

- **"Preview Reader" section:** add a short paragraph.
  - Prompt `K` on a supported image path (list the extensions) opens the reader with an
    inline cell preview sized to the near-full-screen panel. It re-renders on terminal
    resize.
  - The footer shows `Y path | % copy | Z viewer | esc close`, and `Z` opens the
    full-fidelity artifact viewer.
  - `y`, `/`, and `o` warn instead of acting on image previews.
- **Prompt NORMAL-mode paragraph** ("In prompt NORMAL mode, `K` previews the xprompt,
  slash skill, or file under the cursor…"): mention that image files preview inline and
  that `Ctrl+]` still opens images directly in the artifact viewer.

Run `just fmt` so the Markdown formatter keeps the line wrapping consistent.

## Tests

Use shared wait helpers (`page.wait_for` / `sase.ace.testing.wait_for`), never private
polling loops (the `check_test_wait_helpers` lint). Create real images with Pillow
(`Image.new("RGBA", (w, h), color).save(path)`), following the pattern in
`tests/ace/tui/graphics/test_renderable.py`.

1. **Resolver**, in `tests/ace/tui/widgets/test_prompt_preview_target.py`:
   - An image token resolves to `media == "image"`, `kind_label == "image"`, the
     resolved absolute `source_path`, and `content == ""`. Cover an uppercase suffix
     such as `shot.PNG`, and a file whose bytes contain NUL, which proves nothing is
     read or sniffed.
   - A missing `missing.png` still raises "File not found".
   - A directory named `dir.png` still raises the directory error.
   - The existing `blob.bin` binary assertion keeps passing.
2. **Fallback hint**, in `tests/ace/tui/graphics/test_image_file_previews.py`:
   - `image_preview(<undecodable .png>, ctx, fallback_hint="Press Z …")` renders the
     custom hint and not "Open artifact with A".
   - The default still renders "Open artifact with A".
3. **Modal**, in the new `tests/ace/tui/modals/test_preview_panel_image_modal.py`, with
   a tiny `App` that pushes `ImagePreviewPanelModal` (mirror `_PreviewModalTestApp` in
   `preview_panel_modal_test_helpers.py`):
   - After mount at 100×30, `#preview-content` eventually renders a
     `CellImageRenderable`.
     - Its `columns` / `rows` equal `image_preview_size_for_viewport` for the laid-out
       scroll.
     - The container's width and height equal `max_geometry(100, 30)`.
   - The footer string is exactly `Y path | % copy | Z viewer | esc close`, and the
     title contains `IMAGE`.
   - Decoding runs off the main thread. Monkeypatch
     `preview_panel_image_modal.image_preview` with a wrapper that records
     `threading.current_thread() is threading.main_thread()`, and assert that it
     recorded `False`.
   - `await pilot.resize_terminal(140, 40)` produces a second render with the new
     viewport size and a container equal to `max_geometry(140, 40)`.
   - A stale result is dropped. Bump `_image_request_id`, then `await` a `_render_image`
     call carrying the old id, and assert the content renderable is unchanged.
   - An undecodable `.png` renders an `ImageFallbackRenderable` whose hint mentions `Z`.
   - `y`, `/`, and `o` each emit their warning:
     - The search input stays hidden.
     - Patch `subprocess.run` in `sase.ace.tui.modals._source_file_actions` to fail the
       test if called.
   - `Z` calls `open_artifact_path` (monkeypatched in
     `sase.ace.tui.modals._source_file_actions`) with the resolved image path.
4. **End to end**, in `tests/ace/tui/widgets/test_prompt_normal_mode_preview.py`:
   - With the real resolver (no monkeypatch), write a PNG under `tmp_path` and open
     `PromptPage(f"view {image}", cursor=(0, 5), size=(100, 30))`.
   - Pressing `K` pushes `ImagePreviewPanelModal`, and its content becomes a
     `CellImageRenderable`.
   - A text file still opens the plain `PreviewPanelModal`, not the image subclass.
5. Existing `test_preview_panel_modal*.py`, `test_prompt_normal_mode_jump.py` (the
   `Ctrl+]` image viewer tests), and notification/file-panel image tests must keep
   passing unchanged.

## Verification

- `just fmt` (or `just fix`), then `sase tool run check`, which runs the lint gates and
  the diff-scoped tests.
- No PNG visual golden covers this modal, so no screenshot update is needed.

## Out of scope

- Video paths: `K` on `.mp4` and similar still reports a binary file. A placeholder plus
  a `Z` play hint could follow later.
- Other `PreviewPanelModal` callers. The artifacts Files `Enter` action keeps routing
  media to the rich viewer.
- The cell renderer's resampling and aspect handling, and `Ctrl+]` image behavior.
- Showing image metadata such as dimensions or format.
