---
tier: epic
title: Fast inspect-first video artifacts in the tmux viewer pane
goal: 'Opening a video artifact from the Agents tab inside tmux shows a useful frame
  in about half a second (always under 1 s) over SSH. It never garbles pane titles
  or lags tmux or the SASE TUI. The user can scrub frames, play motion where the topology
  supports it, or hand off to a native player. The fixed per-open launch tax is gone
  for every artifact kind.

  '
phases:
- id: tmux-still-hotfix
  title: Stop launching mpv in tmux; show a still frame plus metadata
  depends_on: []
  size: medium
  description: 'tmux-still-hotfix: inside tmux with a kitty VO, render an ffmpeg still
    plus ffprobe metadata through kitten icat instead of mpv. Make missing tools non-fatal,
    add ffmpeg/ffprobe doctor rows, fix the Video Preview docs, and ship the pty+tmux
    regression probe.'
- id: viewer-child-startup
  title: Trim the viewer child's startup tax
  depends_on: []
  size: small
  description: 'viewer-child-startup: pass the parent''s resolved video config to
    the pane child through an env snapshot, lazy-import markdown_pdf, replace the
    clear subprocess with ANSI, add the child-side trace helper, and measure and fix
    kitten icat detection cost inside tmux.'
- id: async-pane-launch
  title: Move tmux split and decoration off the Textual UI thread
  depends_on:
  - viewer-child-startup
  size: medium
  description: 'async-pane-launch: launch the tmux split and pane decoration in a
    pump-free worker with an explicit launch context and bounded tmux timeouts. Show
    an opening state, show a VCS-materialization progress toast, and emit parent-side
    open spans correlated with the child.'
- id: video-frame-cache
  title: Persistent frame, scrub-track, and storyboard cache
  depends_on:
  - tmux-still-hotfix
  size: medium
  description: 'video-frame-cache: add the ffmpeg-backed frame service. It does exact-frame
    extraction, a low-priority background scrub track, a decile storyboard composed
    with Pillow, and a content-digest-keyed bounded cache with atomic writes and cancellation.'
- id: inspect-scrubber
  title: Inspect-first video view with scrubbing keys
  depends_on:
  - video-frame-cache
  - viewer-child-startup
  size: medium
  description: 'inspect-scrubber: replace the video branch with an inspect view. It
    shows the frame, a storyboard strip, and a status line. Keys: h/l and arrows to
    step, '','' and ''.'' for single frames, 0-9 for deciles, s for storyboard, space
    to play. Add a nonblocking escape-aware key reader, step coalescing, stable kitty
    image ids, spans, docs, and a probe extension.'
- id: motion-handoff
  title: Motion backends, native handoff, and audio semantics
  depends_on:
  - inspect-scrubber
  size: medium
  description: 'motion-handoff: add the backend and open_command config keys. mpv
    runs in-pane only where it can render, with sw-fast, a probed shm flag, and --audio=no.
    The o key hands off to a native or configured player. Add degraded banners and
    a motion_start span, update docs and config reference, and record the deferred
    follow-ups.'
- id: non-kitty-fallback
  title: Cell-preview fallback for terminals without kitty graphics
  depends_on:
  - motion-handoff
  size: small
  description: 'non-kitty-fallback: detect kitty-graphics capability from the terminal
    name, using tmux''s client_termname inside tmux. Otherwise render the current
    frame with the existing Unicode cell renderer plus metadata, and fall back the
    same way when kitten icat fails.'
proposed_by: bbugyi200.athena.1di
proposed_with: claude/opus
create_time: 2026-11-17 11:16:30
status: done
bead_id: sase-1tb
---

- **PROMPT:** [prompts/202611/fast_video_artifact_pane.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/fast_video_artifact_pane.md)
- **BEAD:** [sase-1tb](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tb/README.md)

# Plan: Fast inspect-first video artifacts in the tmux viewer pane

## Context

Selecting a video after `a` on the Agents tab is described as "incredibly slow". It is
actually broken under tmux. Read the research report before starting any phase:

```bash
sase artifact read research:202611/fast_video_artifact_pane/fast_video_artifact_pane.md "<why>"
```

The short version:

- **Nothing renders.** `display_video_artifact()` in
  `src/sase/ace/tui/graphics/_viewer_loop_media.py` runs `mpv --no-config --vo=kitty …`.
  mpv 0.40 emits unwrapped kitty APC escapes, and tmux consumes each one as a pane-title
  change. Zero pixels reach the terminal. SASE's own `#{pane_title}` border format then
  turns every 4 KB chunk into a border redraw over SSH: a 6 s clip took 47 s to "play"
  invisibly. mpv is also absent on apollo and the Mac, where the viewer hard-fails with
  `missing_mpv`.
- **mpv's kitty VO is the wrong transport for remote tmux, even when fixed.** It sends
  uncompressed base64 RGB frames (~1,400× the H.264 bytes). In mpv ≥ 0.41 passthrough
  mode it also places frames in window coordinates, so they land on the wrong split.
- **Every open pays a fixed launch tax.** The pane child reloads merged config, imports
  eagerly, and shells out to `clear`. About 13 synchronous `tmux` calls run on the
  Textual UI thread.

Measured on athena while planning (this repo at `df06821acc`):

| Item                                                                 | Value                                                             |
| -------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Child merged-config reload (`load_artifact_video_playback_config()`) | **0.42 s**                                                        |
| Cold `import sase.ace.tui.graphics.viewer`                           | **0.15 s**, of which 0.095 s is the `sase.ace` package `__init__` |
| Tool versions                                                        | `kitten` 0.41.1, mpv 0.40, tmux 3.5a, ffmpeg/ffprobe 7.1          |
| `kitten icat --image-id` / `--transfer-mode`                         | Supported                                                         |
| `--detection-timeout`                                                | Default 10 s                                                      |
| `mpv --no-config --profile=sw-fast` / `--vo-kitty-use-shm`           | Both available                                                    |

The user accepted all of the report's requirement adjustments, R1–R11. Each is mapped to
a phase here:

| Req | Summary                                                                                         | Phase(s)                                                                               |
| --- | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| R1  | Inspect first, motion on demand                                                                 | inspect-scrubber, motion-handoff                                                       |
| R2  | Testable SLOs                                                                                   | Acceptance section; probe in tmux-still-hotfix and inspect-scrubber                    |
| R3  | No collateral damage (titles, pacing, TUI lag)                                                  | tmux-still-hotfix probe, async-pane-launch, video-frame-cache (nice'd background work) |
| R4  | mpv optional; ffmpeg + kitten are the video-inspect dependencies; degraded banners; doctor rows | tmux-still-hotfix, motion-handoff                                                      |
| R5  | kitty is the supported graphics terminal; others get a poster and metadata                      | non-kitty-fallback                                                                     |
| R6  | mpv as an opt-in local/non-tmux backend with sw-fast and probed shm                             | motion-handoff                                                                         |
| R7  | One-key native handoff (`o`), never automatic                                                   | motion-handoff                                                                         |
| R8  | Backend choice is a config field, not a feature flag; existing keys are not renamed             | motion-handoff                                                                         |
| R9  | Stays in the Python TUI (`src/sase/ace/tui/graphics/`); no sase-core work                       | All                                                                                    |
| R10 | Per-stage timing telemetry; VCS materialization progress                                        | viewer-child-startup, async-pane-launch, inspect-scrubber, motion-handoff              |
| R11 | Explicit audio semantics (`--audio=no` only for the silent preview)                             | motion-handoff                                                                         |

## Decisions made while planning

These are final; phases must not re-litigate them.

- **Inspect-first everywhere.** Under tmux and outside it, a video opens on the inspect
  view (frame, storyboard, metadata). Motion starts only on `space`.
- **`p` stays "previous artifact".** R1 mentions `space`/`p` for play, but `p` is
  already pinned (by tests, docs, and the footer) to previous-artifact navigation in
  multi-artifact sequences. Only `space` starts motion.
- **`z` stays tmux pane zoom.** Frames are extracted at the pane's pixel size, so
  zooming re-renders a full-sharpness frame. That is the research's "zoom", with no key
  conflict.
- **mpv never renders through tmux with a kitty VO.** This holds even for mpv ≥ 0.41
  with passthrough, because of wrong-pane placement and bandwidth. Under tmux, mpv may
  run only with a non-kitty `vo` (for example `tct`). Today's config already allows
  that, so the guard keys on `vo`, not only on `$TMUX`.
- **Backend values are `auto | poster | mpv`.** The default is `auto`. The research's
  `kitty` value (streamed animation) is reserved for the deferred streamer; do not
  accept it until it exists.
- **The storyboard uses deciles.** Ten cells sit at 0 %, 10 %, …, 90 % of the duration,
  matching the `0`–`9` jump keys. This is simpler and more predictable than scene
  detection.
- **No feature flag.** The tmux change fixes a broken path, and the remaining behavior
  choices are config (per `sase_flags.md`).
- **`sase artifact open` is unchanged.** It reuses `artifact_video_player_command()`,
  `kitten_icat_command()`, and `load_artifact_video_playback_config()`. Their default
  outputs must stay byte-identical. Add keyword arguments or new builders instead of
  changing defaults.
- **Deferred.** These items are not built here. motion-handoff records them as
  `PROPOSED FOLLOW-UP:` notes:
  - the streamed kitty-animation player (research Phase 3), gated on a real-kitty
    rendering spike and demonstrated demand;
  - the in-TUI file-panel poster rider;
  - a persistent viewer pane or mpv IPC;
  - a gateway byte-range media route.

## Shared contracts

Parallel phases must agree on these names.

- **Env snapshot (viewer-child-startup).** `SASE_ARTIFACT_VIEWER_VIDEO_CONFIG` holds a
  JSON object `{"version": 1, …fields of ArtifactVideoPlaybackConfig}`. It is serialized
  generically from the dataclass, so fields added later flow without touching the codec.
  The parent resolves it through the normal trust-evaluated merged config, never by
  reading project YAML. The child falls back to `load_merged_config()` when the variable
  is absent, malformed, or has an unknown version.
- **Correlation env (async-pane-launch).** `SASE_ARTIFACT_VIEWER_OPEN_ID` is an opaque
  id. `SASE_ARTIFACT_VIEWER_OPEN_T0` is the wall-clock epoch seconds at selection. The
  parent passes `SASE_TUI_TRACE` and `SASE_TUI_TRACE_PATH` through `split-window -e`
  when set, because a tmux pane inherits the tmux session environment, not the TUI's.
- **Span names.**
  - Parent: `artifact_viewer.open.select`, `.materialize`, `.split`, `.decorate`.
  - Child: `artifact_viewer.child_ready`, `.first_visual`, `.storyboard_ready`,
    `.key_to_frame`, `.motion_start`.
  - Every child span carries `open_id` (when known), `mode`, `elapsed_ms` since child
    start, and `since_select_ms` when `OPEN_T0` is known.
- **Module homes.** All live in `src/sase/ace/tui/graphics/`:
  - `_viewer_video_media.py`: ffprobe metadata and the single-still extraction
    (tmux-still-hotfix);
  - `_viewer_trace.py`: child-side spans (viewer-child-startup);
  - `_viewer_video_frames.py`: frame cache, scrub track, and storyboard
    (video-frame-cache);
  - `_viewer_loop_video.py`: the inspect view loop (inspect-scrubber).
- **Test hygiene.** SASE agents run inside tmux. Every viewer test that depends on
  topology must set or delete `TMUX`/`TMUX_PANE` explicitly with `monkeypatch`. Existing
  video tests in `tests/ace/tui/artifact_file_viewer/test_sequence_loop.py` currently
  inherit them; fix that in the first phase that touches them.

## For every phase

- Keep test seams: the injected `read_key`, `run_command`, and `select_pane` callables
  in `run_artifact_sequence_loop()`, and the `_viewer_loop.py` compatibility facade.
- New public helpers exported from `graphics/__init__.py` must be used. Symvision flags
  unused symbols.
- Read the `lint_and_test.md` memory before finishing and run the verification it
  prescribes. Read `docs_writing.md` before editing `docs/`, and `tui_perf.md` before
  touching TUI-thread code.
- Record measured before/after numbers in your bead notes whenever the phase claims a
  speedup.

## Phase: tmux-still-hotfix

**Outcome.** Inside tmux with a kitty `vo`, a selected video shows a still frame and
metadata within about a second, and mpv is never spawned.

### Behavior

1. **Guard.** Add `video_still_preview_required(config)`, true when `is_tmux_session()`
   and `video_vo_uses_kitty_placement(config.vo)`. When true, `display_video_artifact()`
   must not build or run any mpv command. Outside tmux, or with a non-kitty `vo`, the
   current mpv behavior is unchanged in this phase.
2. **Still path** (`_viewer_video_media.py`, called from `display_video_artifact`):
   - Probe metadata with
     `ffprobe -v error -print_format json -show_format -show_streams -select_streams v:0`
     and a 5 s timeout. Parse duration, width, height, fps (rational strings such as
     `30000/1001`), codec, and file size, with every field optional.
   - Extract the first frame with
     `ffmpeg -nostdin -loglevel error -ss 0 -i <path> -frames:v 1 -vf scale=…` and a 10
     s timeout. Write a PNG into the viewer's existing `cache_root`; video-frame-cache
     makes it persistent. Scale down to at most the pane's pixel width, read via
     `TIOCGWINSZ` with a 1600 px fallback. Never upscale.
   - Clear the screen and print the header. The media label includes the metadata
     summary, for example `▶ Video 0:07 · 2560×1548 · 30 fps · h264 · 1.2 MB`.
   - Display the PNG with the existing `kitten_icat_command()` inside the image area.
   - Prefix the footer with `still frame · in-pane playback is unavailable inside tmux`.
   - `r` re-renders the still.
3. **Non-fatal dependencies.**
   - Under the guard, `mpv` is not required. Update
     `validate_artifact_file_viewer_dependencies("video", …)` and the video branch of
     `render_artifact_file_pages()`; the call shape is the worker's choice, but every
     video caller must agree.
   - A missing `ffprobe`, `ffmpeg`, or `kitten`, or a failed or timed-out extraction,
     prints a yellow notice naming the tool and keeps the viewer alive. The footer,
     `n`/`p`, `<tab>`, and `q` keep working.
   - Today any warning returned in `VideoDisplayResult.warnings` makes the sequence loop
     exit with returncode 1. Add a non-fatal notices channel, or print directly, so this
     path never exits on a missing tool.
4. **Doctor** (`src/sase/doctor/checks_tools.py`):
   - Add optional rows `ffmpeg` and `ffprobe` with the feature "terminal video artifact
     stills and metadata".
   - Reword the mpv row to "terminal video playback outside tmux (optional)".
5. **Docs.**
   - Update `docs/reference/agent-attachments.md`, both the artifact-viewer paragraph
     and `#video-preview`, so they state what actually happens inside tmux and the new
     dependency list.
   - Update the `ace.artifact_file_viewer` section of
     `docs/reference/config/ace-behavior.md` to say that `vo: kitty` is not used inside
     tmux.
   - If you change the schema description for `vo`, regenerate with `just fix`.

### Tests

- **Unit** (`tests/ace/tui/artifact_file_viewer/`):
  - dispatch matrix: TMUX set with kitty `vo` runs ffprobe, ffmpeg, and icat with no
    mpv; TMUX set with `vo: tct` runs mpv; TMUX unset runs mpv;
  - ffmpeg command golden;
  - ffprobe JSON parsing from fixtures, including missing streams and rational fps;
  - each missing tool produces a notice, not a loop exit;
  - `r` re-renders;
  - existing mpv tests explicitly unset `TMUX`/`TMUX_PANE`.
- **Regression probe.** New file
  `tests/ace/tui/terminal_smoke/test_video_pane_tmux_probe.py`, marked
  `pytest.mark.slow` and `pytest.mark.terminal_smoke`, and run by
  `just test-terminal-smoke`.
  - Skip unless `tmux`, `ffmpeg`, and `kitten` are on PATH.
  - Generate a 3 s 640×360 `testsrc` mp4 in `tmp_path`.
  - Start an isolated server: `tmux -L sase-video-probe-<pid> -f /dev/null`. Set
    `allow-passthrough all`, `pane-border-status top`, and a `#{pane_title}`
    pane-border-format, which reproduces SASE's decoration.
  - Attach a client inside a pty with pixel geometry set via `TIOCSWINSZ` (for example
    120×40 cells at 1200×800 px). Drain everything the client receives.
  - Set a sentinel pane title, then run
    `python -m sase.ace.tui.graphics.viewer --kind file <clip>`. Put a fake `mpv` shim
    first on PATH that touches a marker file.
  - Assert:
    - at least one kitty APC (`ESC _G`) reaches the client within 1.0 s after the header
      text `Viewing artifact` appears;
    - the pane title still equals the sentinel at the end;
    - total client bytes stay under 4 MB;
    - the mpv marker was never created.
  - Always kill the isolated server in `finally`.

## Phase: viewer-child-startup

**Outcome.** The pane child stops paying ~0.4 s of config reload, plus the eager import
and `clear` costs, on every open. This benefits every artifact kind.

1. **Config snapshot.**
   - Implement the env contract above with `serialize_video_playback_config()` /
     `parse_video_playback_config_env()` beside `load_artifact_video_playback_config()`.
     That loader checks the env first, then falls back.
   - `view_artifact_files_in_tmux_pane()` appends
     `-e SASE_ARTIFACT_VIEWER_VIDEO_CONFIG=<json>` to `split-window`. Compute the value
     in the parent with a merged-config-only resolver that ignores the env var, so a
     nested or stale value never self-propagates. Its argv is a list, so no shell
     quoting is involved.
2. **Lazy imports.**
   - Move `sase.attachments.markdown_pdf` imports in `viewer.py` and `_viewer_render.py`
     to function scope. Keep the existing seam that tests monkeypatch
     (`_viewer_render.render_markdown_pdf`) working, or update those tests.
   - Measure with `python -X importtime -c "import sase.ace.tui.graphics.viewer"`.
   - Trimming the `sase.ace` package-`__init__` cost (~0.095 s) is optional. Do it only
     if a local, low-risk change achieves it, such as lazy attribute exports.
3. **ANSI clear.** `clear_terminal()` writes `\x1b[H\x1b[2J\x1b[3J` (what `clear` emits
   under kitty/tmux terminfo) and flushes. It spawns no subprocess. Update callers and
   tests that asserted a `["clear"]` command.
4. **Child trace helper** (`_viewer_trace.py`).
   - When `SASE_TUI_TRACE` is not `1`, it is a near-zero-cost no-op that imports nothing
     heavy.
   - When enabled, it lazily writes JSONL through the existing trace sink
     (`sase.ace.tui.util.trace` or the bounded JSONL appender it uses) at
     `SASE_TUI_TRACE_PATH`.
   - Emit `child_ready` after imports and config resolution. Emit `first_visual` after
     the first successful `kitten icat` in the sequence and page loops.
   - Expose `mark_first_visual(mode)` so later phases can call it for video.
   - The parent passes `SASE_TUI_TRACE` and `SASE_TUI_TRACE_PATH` through `-e` when they
     are set. async-pane-launch adds the open id and t0.
5. **icat detection cost.**
   - Measure `kitten icat` wall time inside tmux. `--transfer-mode` defaults to
     `detect`, and `--detection-timeout` defaults to 10 s.
   - If detection adds measurable latency inside tmux, pass `--transfer-mode=stream`,
     and lower `--detection-timeout` if needed. Do this only when the viewer runs inside
     tmux, via a new keyword on `kitten_icat_command()` whose default output is
     unchanged.
   - Record the numbers either way.

**Tests:**

- snapshot round trip, plus fallback on absent, malformed, or unknown-version values;
- `split-window` argv contains the `-e` snapshot and trace passthrough;
- importing the viewer in a subprocess leaves `sase.attachments.markdown_pdf` out of
  `sys.modules`;
- `clear_terminal` spawns no subprocess;
- the trace helper writes nothing when disabled and well-formed rows when enabled.

## Phase: async-pane-launch

**Outcome.** Selecting an artifact never blocks the Textual UI thread on `tmux`
subprocesses (`tui_perf.md` rules 1, 2, and 4). The open is visible immediately and
measurable end to end.

1. **Explicit launch context.**
   - Add a small `ArtifactViewerLaunchContext` (`notify_pid`, `return_pane_id`,
     `extra_env`, `open_id`, `open_t0`).
   - Accept it in `view_artifact_files_in_tmux_pane()` and its `registered_*` wrappers,
     and pass `origin_pane_id` explicitly to `decorate_artifact_file_tmux_panes()`.
   - Env-reading defaults remain for other callers.
   - The TUI open path must stop mutating `os.environ`; today
     `_with_artifact_file_viewer_notify_pid()` does.
2. **Off-thread open** (`actions/agents/_panel_artifact_open.py`,
   `_panel_artifact_pane.py`).
   - On the UI thread: install the SIGUSR1 handler (it must stay on the main thread) and
     build the context. Show an immediate opening state, for example a notify "Opening
     artifact viewer…". Set an in-flight guard: a second `a` while an open is in flight
     shows a toast and does nothing.
   - Then `spawn_pump_free_task()` →
     `asyncio.to_thread(split + optional zoom + decorate)`.
   - Completion runs on the UI thread. Re-read the current state, then track the pane id
     and decoration state and collapse the layout. Skip the extra
     `artifact_file_tmux_pane_exists()` round trip; the split just returned the id.
     Notify warnings and clear the guard.
   - If no event loop is running, keep a synchronous fallback, mirroring
     `_open_artifact_files_materializing()`. Teardown cancels in-flight tasks.
3. **Bounded tmux calls.**
   - Every tmux subprocess on the launch, decorate, and restore paths gets a timeout (2
     s). `TimeoutExpired` becomes a warning, never a hang.
   - Optional: batch decoration's snapshot reads and option writes into ≤ 3 tmux
     invocations with `\;` command sequences, if snapshot and restore semantics stay
     identical.
   - The close path (`a` again) may stay synchronous in this phase.
4. **Materialization progress.** When any selected row is byte-free and needs VCS
   materialization, show "Fetching N artifact file(s)…" immediately, before the
   off-thread materialize in `_open_artifact_files_materializing()`.
5. **Telemetry.**
   - Emit the parent spans `open.select` (selection callback to open start),
     `open.materialize` (row count), `open.split`, and `open.decorate` with the existing
     `tui_trace`/`trace_event` helpers.
   - Pass `SASE_ARTIFACT_VIEWER_OPEN_ID` and `SASE_ARTIFACT_VIEWER_OPEN_T0` in
     `extra_env` so child spans report `since_select_ms`.
   - If `docs/contribute/perf-runbook.md` lists trace spans, add these.

**Tests:**

- During an open, record `threading.current_thread()` in a patched `subprocess.run` and
  assert no tmux call happens on the main thread.
- The in-flight guard works.
- Completion tracks the pane and decoration state.
- A tmux timeout surfaces as a warning.
- `os.environ` is unchanged after the open.
- The non-tmux suspend path is unchanged.
- Update the existing action tests (`tests/ace/tui/actions/test_artifact_file_*`,
  `test_artifact_viewer_handoff.py`, `tests/ace/tui/test_artifacts_files_open.py`,
  `tests/ace/tui/artifact_file_viewer/test_tmux.py`) for the async shape.

## Phase: video-frame-cache

**Outcome.** A UI-free service that turns a video into cached stills fast enough for
<100 ms key response on cached frames.

New module `_viewer_video_frames.py`. It reuses the ffprobe metadata from
`_viewer_video_media.py`.

- **Cache.**
  - Use a persistent SASE-managed directory; use the established path helper in
    `sase.core.paths`, for example a `cache/artifact_video` subdir.
  - Key by content digest (streamed sha256 of the file, once per open) plus the derived
    parameters (target pixel size, scrub fps).
  - Write to a temp file plus an atomic rename, so partial outputs are never served.
  - Cap the total size (default 256 MB) with LRU eviction by mtime on write.
- **`extract_frame(path, t, max_px)`.**
  - Use input-seek `ffmpeg -nostdin -loglevel error -ss <t> -i <path> -frames:v 1`,
    scaled to the target size.
  - Choose PNG, or another format kitty accepts, by measuring text legibility against
    bytes. Target ≤ ~350 KB per frame for a half-screen pane.
- **`ScrubTrack`.**
  - One background ffmpeg pass at `fps = min(2, 240 / duration)` (at most 240 frames) at
    the frame-region pixel size. Write numbered frames plus an index.
  - API: `nearest(t) -> Path | None`, `progress`, `done`, `cancel()`.
  - Run at reduced priority (niceness 10, `-threads 2`) so it cannot lag the TUI (R3). A
    cancel kills the process. A completed track is reused from the cache.
- **`storyboard(…)`.**
  - Ten cells at `i × duration / 10`, composed with Pillow into a grid shaped to the
    strip's pixel aspect (for example 10×1 or 5×2).
  - Draw timestamp labels with Pillow's default font (no ffmpeg `drawtext` or fontconfig
    dependency).
  - Build from scrub-track frames when available. Otherwise use ten input-seek
    extractions with concurrency ≤ 4, whichever completes first. Cache the result.
- **Error handling.** Every subprocess has a timeout. Errors come back as typed results
  and never raise into the viewer loop.

**Tests:**

- command goldens;
- fps and frame-budget math;
- cache keys, eviction, and atomic writes;
- storyboard grid and label layout using synthetic Pillow frames (no ffmpeg);
- cancellation kills the process (fake `Popen`);
- one `slow`-marked real-ffmpeg test on a generated `testsrc` clip, skipped without
  ffmpeg.

## Phase: inspect-scrubber

**Outcome.** R1's landing state. Opening a video shows the frame quickly, then a
storyboard, and the keys answer "what happens around 0:22?" without motion.

1. **Loop.**
   - `_viewer_loop_video.py` exposes `run_video_inspect(…) -> VideoInspectOutcome`
     (next, previous, quit, or return-pane), called from the sequence loop's video
     branch. It replaces the tmux-still-hotfix still path in every topology.
   - Pane zoom (`z`) is handled inside the inspect loop and re-renders; it is not an
     outcome.
2. **Layout** (rows, top to bottom):
   - the existing header panel, whose media label carries the metadata;
   - the frame region, about two-thirds of the remaining rows;
   - the storyboard strip, about one-third with a minimum of 5 rows, hidden when the
     pane has fewer than 20 rows or when toggled off;
   - the status line: `00:02.0 / 00:07.0` plus a position bar and scrub-track progress
     while it builds;
   - the footer with the available keys, abbreviated to fit.
3. **First paint.**
   - Show the frame at t=0 right away (`mark_first_visual("video")`).
   - Start the scrub track in the background and draw the storyboard when ready
     (`storyboard_ready` span) without blocking keys.
4. **Keys.**

   | Key                    | Action                                                                |
   | ---------------------- | --------------------------------------------------------------------- |
   | `h` / `←`              | −1 s, snapped to scrub-track frames when ready, else exact extraction |
   | `l` / `→`              | +1 s, same snapping                                                   |
   | `,` / `.`              | ∓ one source frame (exact extraction)                                 |
   | `0`–`9`                | Jump to that decile                                                   |
   | `s`                    | Toggle the storyboard strip                                           |
   | `space`                | Motion (see 7)                                                        |
   | `r`                    | Re-render and retry failed extractions                                |
   | `z`                    | tmux zoom; re-render at the new size                                  |
   | `n`, `p`, `<tab>`, `q` | Unchanged                                                             |

   `j` and `k` stay unavailable for video, as today.

5. **Input.**
   - Add a nonblocking, escape-aware reader: `select` on stdin in raw mode, parsing
     `\x1b[C`/`\x1b[D` and `\x1bOC`/`\x1bOD`, with a ~30 ms ESC disambiguation window.
     Return `None` on timeout so the loop can poll background work.
   - Keep an injectable reader for tests.
   - Coalesce bursts: apply all queued step keys, update the status line immediately (<
     100 ms), then render one frame (`key_to_frame` span).
6. **Kitty image hygiene.**
   - Use a stable kitty image id per region via `kitten icat --image-id`, so each step
     replaces the image instead of accumulating images in kitty memory.
   - Delete the region images on `q`, `n`, `p`, and the SIGHUP/SIGTERM cleanup path
     (`kitten icat --clear` or an `a=d` delete through tmux passthrough). Cancel the
     background extraction on the same paths.
7. **`space`, interim.**
   - When `video_still_preview_required()` is false and mpv is on PATH, run today's
     `artifact_video_player_command()`, then return to the inspect view.
   - Otherwise show a one-line notice: "in-pane playback is unavailable here — scrub
     with h/l or 0–9".
   - This keeps outside-tmux users from losing motion before motion-handoff lands.
8. **Docs.** Replace the Video Preview key description in
   `docs/reference/agent-attachments.md` with the inspect-view keys.
9. **Probe extension** (the tmux-still-hotfix test file).
   - After the view settles, assert:
     - at least two images were transmitted (frame and storyboard);
     - the pane title is unchanged;
     - total client bytes stay under 4 MB for the open plus ten `l` presses.
   - Assert that a new APC follows an `l` press within 300 ms once the scrub track is
     done.

**Tests:**

- key parsing, including arrow sequences split across reads;
- step and decile math with clamping at both ends;
- coalescing: five queued `l` presses produce one render at +5 s;
- the storyboard toggle, and auto-hide on small panes;
- the `space` dispatch matrix;
- outcomes for `n`, `p`, and `q`;
- resize re-renders;
- background work is cancelled on exit.

## Phase: motion-handoff

**Outcome.** R4, R6, R7, R8, and R11 are complete. Motion runs where it can render,
native playback is one key away, and nothing hard-fails on a missing tool.

1. **Config.**
   - Add `ace.artifact_file_viewer.video.backend` (`auto | poster | mpv`, default
     `auto`).
   - Add `open_command`: an argv list or a shlex string, default `[]`. A `{path}` token
     is substituted; if the token is absent, the path is appended.
   - Update `src/sase/default_config.yml`, `src/sase/config/sase.schema.json` (enum and
     descriptions), `ArtifactVideoPlaybackConfig`, and its coercion. The
     viewer-child-startup snapshot carries the new fields automatically; verify that.
   - Keep `audio`, `loop`, `vo`, and `extra_mpv_args`; they now apply to the mpv backend
     only.
2. **Motion dispatch (`space`).**
   - `mpv` and `auto` run mpv when it is installed and the topology allows it: outside
     tmux, or a non-kitty `vo`.
   - `auto` without mpv behaves like `poster`.
   - `poster`, a forbidden topology, or `mpv` with mpv missing shows a notice that names
     the reason. The explicit `mpv` case also shows a banner that mpv is not installed.
   - Return to the inspect view at the same timestamp afterwards.
3. **Pane mpv command.**
   - Add a new builder, for example `artifact_video_motion_command()`. Do not change
     `artifact_video_player_command()`, which `sase artifact open` uses.
   - Flags:
     - `--no-config --profile=sw-fast`;
     - `--start=<scrub t>`;
     - `--audio=no` when `audio: false` (R11), replacing `--mute=yes` for the silent
       preview;
     - `--loop-file=inf` when `loop: true`;
     - kitty placement flags when the VO is kitty;
     - `--vo-kitty-use-shm=yes` only when the capability probe below succeeds;
     - `extra_mpv_args` last.
4. **Shared-memory capability probe** (R6: probe, never env guessing).
   - Runs only outside tmux with a kitty VO.
   - Create a tiny POSIX shm object and send a kitty query
     `ESC _G i=<id>,a=q,t=s,s=1,v=1,f=24;<b64 name> ESC \`.
   - Read the reply in raw mode with a ~150 ms timeout. `OK` means the terminal is
     local.
   - Unlink the shm object if kitty did not. Cache the result for the viewer process.
5. **Native handoff (`o`, R7).**
   - With `open_command` configured: spawn it detached (`start_new_session=True`, stdio
     to DEVNULL) and print the outcome.
   - Otherwise:
     - on macOS, use `open`;
     - on Linux, use `xdg-open` only when `DISPLAY` or `WAYLAND_DISPLAY` is set and the
       session is not SSH;
     - in any other case, show a notice pointing to `open_command`.
   - Never open automatically. Native playback keeps audio.
6. **Degraded mode (R4).**
   - Without ffmpeg/ffprobe: show a metadata-less banner naming the tool. `o` and
     `n`/`p`/`q` still work.
   - No `missing_mpv` hard failure remains anywhere on the pane path.
   - Re-check the doctor row wording against the final behavior.
7. **Telemetry.** Emit `motion_start` when mpv's first output is written, or when the
   process starts if that cannot be observed. Include the backend.
8. **Docs.**
   - Rewrite `#video-preview` in `docs/reference/agent-attachments.md`: the inspect
     view, `space` per backend and topology, `o`, audio semantics, and dependencies.
   - Update `docs/reference/config/ace-behavior.md`, then run `just fix` for the
     generated config reference.
9. **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes on your bead for each item in
   "Deferred" above. For the streamer, include its two gates and point to the research
   report's wire-format rules.

**Tests:**

- the config coercion and schema enum;
- the `space` dispatch matrix across backend × topology × mpv presence;
- motion-command goldens (sw-fast, start, audio, shm present and absent);
- `artifact_video_player_command()` output is unchanged;
- the shm probe with a fake tty for OK, timeout, and error replies, including unlink;
- `o` with a template, an appended path, platform defaults, and the not-available
  notice, with `Popen` faked;
- degraded banners.

## Phase: non-kitty-fallback

**Outcome.** R5. Terminals without kitty graphics get a usable frame and metadata
instead of a blank region.

- **Capability.** Add `kitty_graphics_capable()`.
  - Outside tmux: `TERM` contains `kitty` or `ghostty`, or `KITTY_WINDOW_ID` is set.
  - Inside tmux: one bounded `tmux display -p '#{client_termname}'` (1 s timeout),
    matched against the same names.
  - Cache the result per viewer process.
- **Fallback.**
  - When not capable, or when `kitten icat` exits non-zero, render the current frame in
    the frame region with the existing `CellImageRenderable` (`graphics/cell.py`)
    through Rich.
  - Hide the storyboard strip, keep the status line and footer, and show a one-line
    banner: "terminal without kitty graphics: cell preview".
  - Stepping keys still work; they re-render cells.
- **Scope.** This applies only to the video inspect view. The image, PDF, and Markdown
  paths are unchanged.
- **Docs.** Add one sentence to `#video-preview`.

**Tests:**

- the capability matrix (env values and faked `client_termname`);
- an icat failure triggers the fallback;
- the cell render fits the region;
- stepping in fallback mode.

## Acceptance (R2 SLOs)

Each phase's probes and unit tests verify these where noted. The last phase to touch the
probe reports the measured values in its bead notes.

- **First useful pixel.** A frame within 1 s of the viewer pane starting (target 0.5 s),
  measured by the pty+tmux probe on athena.
- **Input response.** A step key updates the status line in < 100 ms. A cached frame is
  on screen in < 300 ms over localhost tmux.
- **No collateral damage.** The pane title never changes during video viewing. No mpv
  process runs under tmux with a kitty VO. Background extraction runs nice'd.
- **Wire bytes.** Under 4 MB for the probe fixture's open plus ten steps.
- **Launch tax.** Selection → `child_ready` improves by ≥ 0.4 s against a pre-epic
  baseline captured with `SASE_TUI_TRACE=1`.

## Manual verification (for the user, after landing)

On the Mac's kitty over SSH to athena:

1. Open a screen recording and a TUI demo from the Agents tab.
2. Check the time to first frame, storyboard legibility, and the feel of
   `h`/`l`/`,`/`.`/`0`–`9`.
3. Check that `z` re-renders sharper, and that `o` works with a configured
   `open_command`.
4. Confirm the SASE TUI and other panes stay responsive throughout.
5. Repeat one open on apollo, which has no mpv, to confirm the degraded path.

## Non-goals

- No new decoder or streaming service, and no record-time thumbnails.
- No sixel/tct default.
- No changes to `sase artifact open`.
- No sase-core or gateway work.
- No persistent-pane or mpv-IPC work.
- No streamed kitty animation in this epic.
