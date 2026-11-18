---
tier: tale
title: Prompt K preview support for video files
goal:
  Pressing K on a video path in the prompt opens an inline video preview (metadata,
  still frame, decile skimming, Z handoff to the full viewer) instead of the "Cannot
  preview binary file" toast.
size: medium
decisions:
  jump_video:
    ask:
      Should `Ctrl+]` on a video path open it straight in the artifact viewer, the way
      it already does for images?
    choices:
      viewer:
        Ctrl+] opens videos directly in the artifact viewer's inspect view, matching
        images
      unchanged:
        Ctrl+] keeps offering the tmux/editor action picker, which opens a binary mp4 in
        $EDITOR
    default: viewer
    why:
      Opening an mp4 in an editor is never useful, and the image branch shows the
      intended pattern
    answer: viewer
proposed_by: bbugyi200.athena.1e0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 11:24:23
status: wip
---

# Prompt `K` preview: first-class support for video files

## Problem

In prompt NORMAL mode, `K` previews the macro, skill, or file under the cursor. Image
files already open an inline image reader (`ImagePreviewPanelModal`), but a video path
such as `~/tmp/screencasts/20261118_110822.mp4` falls through to the text reader. The
binary sniff in `_read_text_file()` then rejects it with the toast
`Cannot preview binary file: ~/tmp/screencasts/20261118_110822.mp4`.

The rest of sase's TUI already understands videos. `sase.media_types` defines
`SUPPORTED_VIDEO_EXTENSIONS` (`.mp4 .m4v .mov .webm`). The terminal artifact viewer has
a full video stack under `src/sase/ace/tui/graphics/_viewer_video_*` and
`_viewer_loop_video_*`:

- ffprobe metadata cached by content digest: `metadata_for_frame_service`,
  `format_video_metadata_summary`
- cached exact-frame extraction through ffmpeg: `extract_frame` into
  `artifact_video_cache_dir()`
- an inspect-first view that steps and jumps to deciles (`0`-`9`), shows a storyboard,
  and plays the video

Only the prompt `K` path is missing this.

## Goal

`K` on a supported video path opens a **video preview reader**. It is a sibling of the
image reader and shows:

1. the title (`@ VIDEO  <token>`) and the resolved path, like other file previews;
2. a metadata line, e.g. `4:05 · 2858×1728 · 30 fps · h264 · 12.3 MB`;
3. a still frame of the video rendered inline with the existing half-block cell image
   renderer, sized to the viewport like the image reader;
4. a position line showing where that frame sits, e.g.
   `frame 3/10 · 00:49.0 / 04:05.0  [=====>-------------]`;
5. quick skimming keys: `l`/`→` and `h`/`←` move one tenth forward or back, and `0`-`9`
   jump to that decile. These match the digit jumps in the artifact viewer's inspect
   view;
6. `Z` hands the file to the full artifact viewer (inspect view with hi-res frames,
   storyboard, and playback). This already works through the inherited
   `SourceFileActionsMixin.action_open_in_viewer` → `open_artifact_path`, which routes
   videos to `run_video_inspect`.

Non-goals: inline motion playback; kitty-graphics rendering inside Textual; a storyboard
grid inline (at half-block resolution a 10-cell grid of a screencast is illegible, so
one large frame plus decile stepping is the useful form); prefetching every decile;
opening the viewer at the modal's current timestamp; extending the supported video
extension list.

## Design

### 1. Resolution: `src/sase/ace/tui/widgets/_prompt_preview_target.py`

- Widen `PreviewMedia` to `Literal["text", "image", "video"]`.
- In `_resolve_file_preview()`, right after the image branch (so after the existing
  not-found, directory, and non-file checks), return a payload for supported videos
  **without reading bytes or running ffprobe**:
  `PreviewPayload(kind_label="video", icon="@", title=token.raw, source_path=str(resolved), content="", lexer="text", media="video")`.
  Use `is_supported_video_path` from `sase.ace.tui.graphics.videos`, mirroring the
  `images` import. Resolution must stay cheap; all probing happens in the modal off the
  UI thread.
- Unsupported binaries (e.g. `.mkv`, `.bin`) keep the existing
  `Cannot preview binary file` error.

### 2. Routing: `src/sase/ace/tui/widgets/_prompt_preview.py`

In `_resolve_preview_async`, add a `payload.media == "video"` branch that lazily imports
and pushes `VideoPreviewPanelModal(payload)`. It sits next to the existing image branch.

### 3. Public graphics seam: new `src/sase/ace/tui/graphics/video_preview.py`

The modal must not reach into the `_viewer_video_*` private modules. Add one small
public module that wraps the existing helpers and re-export its names from
`sase/ace/tui/graphics/__init__.py`. Use these responsibilities; exact names are the
implementer's choice:

- **Probe** (`load_video_preview_info(path, *, cache_dir=None)`): returns a frozen
  dataclass with `metadata: VideoMetadata | None`, `summary: str | None` (from
  `format_video_metadata_summary`), `duration_seconds: float | None`, and a user-facing
  `notice: str | None`. Example notices:
  `ffprobe not found: video metadata unavailable`,
  `ffmpeg not found: video frames unavailable`. When ffprobe is missing or fails, still
  report the file size from `stat`. Implement it with
  `metadata_for_frame_service(path, cache_dir=artifact_video_cache_dir(cache_dir=...))`
  so repeat opens reuse the digest-keyed metadata cache that the artifact viewer shares.
- **Frame**
  (`video_preview_frame(path, at_seconds, context, *, columns, rows, cache_dir=None)`):
  runs `extract_frame(path, at_seconds, VIDEO_PREVIEW_FRAME_PX, cache_dir=...)` at a
  fixed pixel width (≈640, so frames cache across resizes and extraction stays fast). On
  success it returns
  `image_preview(frame_path, context, columns=..., rows=..., fallback_hint="Press Z to open it in the artifact viewer")`.
  On failure it returns a textual fallback renderable: `▶ video`, the file name, the
  path, the `FrameExtractResult.error` reason, and the `Z` hint. Reuse or generalize
  `ImageFallbackRenderable` instead of inventing a new style. The function never raises.
- **Position helpers**: `decile -> seconds` delegates to the existing
  `decile_timestamp`. The position line reuses `format_inspect_status` (prefixed with
  `frame N/10 ·`). Duplicating either one's math is not allowed.

Every function here is blocking (sha256 digest, ffprobe, ffmpeg, Pillow) and must only
be called from a worker thread. `content_digest_for_path` is memoized per process by
`(path, dev, ino, size, mtime)`, so after the first open, opens of the same clip skip
hashing.

The video stack is terminal-presentation code that already lives in Python under
`ace/tui/graphics`. Under the Rust core boundary litmus test no other frontend needs to
match it, so no `sase-core` change is needed.

### 4. Modal: new `src/sase/ace/tui/modals/preview_panel_video_modal.py`

Add `VideoPreviewPanelModal`. Prefer subclassing `ImagePreviewPanelModal` so it inherits
max-geometry sizing, viewport-sized rendering, resize re-render, and the stale-request
guard. Refactor the image modal only as much as needed to expose a seam (e.g. a
`_render_renderable(columns, rows)` hook and an `_invalidate_image_render()` that clears
`_image_requested_size` so a decile change re-renders at the same size). Existing
image-modal behavior and tests must stay unchanged.

Behavior:

- **On mount**: show `Loading video…` in `#preview-content` and start one pump-free task
  (`spawn_pump_free_task` → `asyncio.to_thread(load_video_preview_info, ...)`). When it
  lands, after re-checking `is_attached`, render the metadata line and position line.
  Use the `#preview-properties` band slot or the title's second line, whichever fits the
  existing layout without CSS churn. Then request the first frame at decile 0 (t=0,
  matching the inspect viewer's opening playhead). The first frame request may run
  concurrently with the probe at t=0, since ffmpeg does not need the duration.
- **Stepping keys** (`h`/`left`, `l`/`right`, `0`-`9`; add them to `BINDINGS`): the
  handler is synchronous and thin. It clamps the decile to 0-9, updates the position
  line immediately, and schedules a frame render. Handle the overlap with the base
  modal's bindings: `j`/`k`/`g`/`G` scrolling is harmless and can stay. Override any
  inherited binding that would conflict.
- **Coalescing (TUI perf rule 2)**: at most **one** frame extraction is in flight per
  modal. A key press while one is running records only the latest target decile. When
  the running extraction finishes, it renders its result only if it still matches the
  target. Otherwise it starts exactly one follow-up for the latest target. After every
  `await`, re-check `is_attached` and the request id. Mashing `l` must never queue up
  ten ffmpeg processes.
- **Unknown duration** (ffprobe missing or failed): show the t=0 frame. Stepping keys
  show a warning toast (`Video duration unknown; press Z for the artifact viewer`) and
  the footer omits the stepping hints.
- **ffmpeg missing or extraction failed**: the content area shows the fallback
  renderable from §3, including the `Z` hint.
- **Footer**: `h/l ±10% | 0-9 jump | Y path | % copy | Z inspect/play | esc close`. Drop
  the first two segments when stepping is unavailable.
- **Inherited actions**: `Y` (copy path), `%`, `Z` (open the viewer; the modal stays
  open, matching images), `q`/`esc`.
- **Warning overrides**, mirroring the image modal: `/` →
  `Search is not available for video previews`; `y` →
  `Video previews have no text to copy; press Y to copy the path`; `o` →
  `Videos open in the artifact viewer; press Z`. If the warning strings differ only by
  noun, factor them through a class attribute on the image modal.
- **Teardown**: inherited `on_unmount` → `cancel_pump_free_tasks(self)`. ffmpeg threads
  keep their existing bounded timeouts (5 s ffprobe, 15 s frame) and their results are
  dropped once the modal is gone.

### 5. `Ctrl+]` parity (decision `jump_video`)

> [!decision] jump_video = viewer In
> `src/sase/ace/tui/widgets/_prompt_jump.py::_present_jump_actions`, widen the image
> check to `is_supported_image_path(...) or is_supported_video_path(...)`. Generalize
> `_view_jump_image` (name, docstring, and status message such as `Opening video…` /
> `Opening image…`) so videos reuse the same suspend + `view_artifact_file` path. Add a
> test next to `test_ctrl_bracket_on_image_views_it_inside_suspend_without_editor` in
> `tests/ace/tui/widgets/test_prompt_normal_mode_jump.py`.

> [!decision] jump_video = unchanged Leave `_prompt_jump.py` and its tests alone; only
> `K` changes.

### 6. Docs

These are Reference updates for an existing feature's behavior, per the docs routing
rule.

- `docs/reference/tui/prompt-completion.md` (the `K` paragraph near "Image files preview
  inline in the reader"): add that supported video files preview inline as a still frame
  with a duration/resolution/codec line, that `h`/`l` step by tenths and `0`-`9` jump to
  a decile, and that `Z` opens the full artifact viewer for frame-accurate inspection
  and playback. Under `jump_video = viewer`, also say that `Ctrl+]` opens images **and
  videos** directly in the artifact viewer.
- `docs/reference/tui/files-and-preview.md` → **Preview Reader**: add a short "Image and
  video previews" note (or table rows) with the video-only keys and explain that
  ffprobe/ffmpeg are optional. Without them the reader degrades to a placeholder with
  the `Z` hint; `sase doctor` already reports `ffprobe`.
- The help modal entry `("K", "Preview macro/skill/file/bead")` already covers videos;
  leave it unchanged. Run `just fix` if generated docs (`docs/README.md`,
  `docs/llms.txt`) need regeneration.

## Tests

Stub the graphics seam in modal and widget tests (monkeypatch the names the modal module
imports) so they do not depend on ffmpeg. Seam tests patch `shutil.which` and
`subprocess.run` the way `tests/ace/tui/artifact_file_viewer/test_video_still.py` does,
and always pass a `tmp_path` `cache_dir` so nothing writes to `~/.sase`.

1. `tests/ace/tui/widgets/test_prompt_preview_target.py`:
   - `clip.MP4` (written with NUL bytes) resolves to `media == "video"`,
     `kind_label == "video"`, `content == ""`, and the right `source_path`;
   - a missing `.mp4` → `File not found`; a directory named `x.mp4` → the directory
     error;
   - a `.mkv` binary still raises `Cannot preview binary file`.
2. `tests/ace/tui/widgets/test_prompt_normal_mode_preview.py`: `K` on a video path
   pushes `VideoPreviewPanelModal`, not the plain or image reader, using a stubbed seam.
3. New `tests/ace/tui/modals/test_preview_panel_video_modal.py`, modeled on
   `test_preview_panel_image_modal.py`:
   - the metadata summary and position line render;
   - a stubbed frame (generated PNG → `CellImageRenderable`) lands at viewport size;
   - `l` / `h` / `5` request `decile_timestamp(n, duration)` and update the position
     line; clamping at 0 and 9;
   - probe and frame work run off the main thread;
   - coalescing: with a blocking stub, rapid `l` presses leave at most one extraction in
     flight, and the final frame matches the last requested decile;
   - unknown duration: a stepping key emits a warning and the footer omits the stepping
     hints;
   - a failed extraction renders the fallback with the `Z` hint;
   - `Z` calls `open_artifact_path` with the path; `/`, `y`, `o` emit the video warnings
     and never run an editor;
   - the footer and title (`VIDEO`) text match.
4. New `tests/ace/tui/graphics/test_video_preview.py` for the seam:
   - success path → image renderable;
   - missing ffmpeg → fallback with reason;
   - missing ffprobe → info carries size plus a notice and `duration_seconds is None`;
   - optionally one real-ffmpeg test, skipped when `shutil.which("ffmpeg")` is `None`,
     that synthesizes a 1 s `testsrc` clip into `tmp_path` and checks the probed
     duration and that a frame renders.
5. Under `jump_video = viewer`: the `Ctrl+]` video test from §5.
6. Re-run the existing image-modal, prompt-preview, and prompt-jump suites unchanged.

## Verification

- Read `sase/memory/lint_and_test.md` (via `/sase_memory_read`) and run the required
  checks the sanctioned way (`just check`), fixing any lint, Symvision (new public names
  must have real non-test consumers), or type errors.
- Optional live check: open `sase tui`, type a prompt containing a real screencast path,
  press `K` in NORMAL mode, confirm the frame, metadata, `h`/`l`/digit stepping, and `Z`
  handoff. No `Cannot preview binary file` toast should appear.
