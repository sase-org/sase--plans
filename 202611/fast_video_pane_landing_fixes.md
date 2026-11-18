---
tier: tale
size: medium
title: Finish and land the fast inspect-first video artifact epic (sase-1tb)
goal: The inspect-first video view handles real key bursts, resize and zoom, large
  files, idle panes, and kitten-less hosts correctly. The tmux probe and TUI import
  budget pass on master. Epic sase-1tb is closed with its plan marked done.
proposed_by: bbugyi200.athena.sase-1tb.land
proposed_with: claude/opus
bead: sase-1tb
status: done
---

- **PARENT:**
  [202611/fast_video_artifact_pane.md](https://github.com/sase-org/sase--plans/blob/main/202611/fast_video_artifact_pane.md)
- **BEAD:**
  [sase-1tb](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tb/README.md)

# Finish and land the fast inspect-first video artifact epic (sase-1tb)

## Context

Epic **sase-1tb** ("Fast inspect-first video artifacts in the tmux viewer pane", plan
`plan:202611/fast_video_artifact_pane.md`) has all seven phases closed (sase-1tb.1 …
sase-1tb.7). Its land agent verified the phases against the plan and the source on
`origin/master` at `f28da33a13`. It found the defects below, all caused by the epic. You
are the last agent on this epic: fix them, verify, then close the epic yourself. Nothing
resumes the landing after you.

Read the epic first: `sase bead read sase-1tb -r "<why>"`. Its newest note (the land
triage) already records every follow-up decision. Do **not** re-triage the child
`PROPOSED FOLLOW-UP:` notes and do **not** create task beads for them. The epic plan's
"Decisions made while planning" are final. In particular: inspect-first everywhere, `p`
stays previous-artifact, backend values `auto | poster | mpv`, no feature flag,
`sase artifact open` byte-identical (`artifact_video_player_command()`,
`kitten_icat_command()` defaults unchanged), and no sase-core work.

All code lives in `src/sase/ace/tui/graphics/` and the tests in
`tests/ace/tui/artifact_file_viewer/` and `tests/ace/tui/terminal_smoke/`. SASE agents
run inside tmux, so every topology-dependent test must set or delete `TMUX`/`TMUX_PANE`
(and `TERM` where capability matters) explicitly with `monkeypatch`.

Before you edit TUI-thread code, read the `tui.md` / `tui_perf.md` memory. Before you
finish, read `lint_and_test.md` and `symvision.md` with `/sase_memory_read`.

## Defects to fix (all reproduced by the land agent)

### 1. Inspect-view key reader drops queued keys and echoes keys typed during a render

`_read_inspect_key()` in `_viewer_loop_video.py` calls `tty.setraw(fd)`. Its default
`when=termios.TCSAFLUSH` discards every unread input byte. The poll loop also restores
cooked mode (ECHO + ICANON) between 50 ms polls and during every render.

Reproduced two ways:

- In a pty, writing `lllll` and then calling the reader repeatedly returns only `['l']`.
- In a real tmux pane, the first `l` press after open was lost, and a typed `l` was
  echoed onto the status line. A five-key `lllll` burst advanced only +1 s, not +5 s. So
  step coalescing (plan: "five queued `l` presses produce one render at +5 s") never
  works outside the injected-reader unit tests.

Fix:

- The default reader must never flush pending input. Use
  `tty.setraw(fd, termios.TCSANOW)` or an equivalent no-flush mode change.
- While `_InspectSession.run()` is active with the default reader on a tty, keep the
  terminal in a session-scoped "quiet" mode: `ECHO` and `ICANON` off, `VMIN=1`,
  `VTIME=0`, applied with `TCSANOW`.
  - Keep `OPOST` on, because Rich output needs `\n` → `\r\n`. Leave `ISIG` as it is
    today between reads.
  - Restore the saved attributes in `cleanup()` (every exit path: `q`, `n`, `p`,
    `<tab>`, exceptions, and the SIGHUP/SIGTERM `_ArtifactViewerSignalExit` path, which
    already unwinds through `finally`).
  - Also restore them around in-pane mpv motion (`_play_motion` → `self._run(command)`).
    Re-enter quiet mode afterwards.
- When `read_key` is injected (tests) or stdin is not a tty (`termios.error`), skip the
  quiet mode entirely.
- To keep `_viewer_loop_video.py` (900 lines; `toobig` warns at 850 and fails at 1000)
  small, move the key reader (`_decode_inspect_key`, `_completed_inspect_escape`,
  `_assemble_inspect_key`, `_read_inspect_key`) and the new quiet-terminal helper into a
  new private module, for example `_viewer_video_input.py`. Update the test imports.

Tests:

- A pty-based unit test (`pty.fork()` or `os.openpty()` with the slave as stdin) shows
  that bytes queued before a read are all returned by successive reads, not flushed.
- Quiet mode is entered and restored, including around motion and on an exception.
- No quiet mode is applied with an injected reader.

### 2. The production sequence loop locks the inspect view to its initial geometry

`_viewer_loop_sequence.py` passes `image_area=current_area` (always a concrete
`artifact_image_area()` in production) into `run_video_inspect`. `_InspectSession` then
sets `_fixed_area`, so `_poll_resize()` never runs, and `z` (tmux zoom) recomputes the
same stale layout. The plan requires "`z` tmux zoom; re-render at the new size" and
"resize re-renders". The resize unit test passes only because it calls
`run_video_inspect` without `image_area`.

Fix:

- Pass the caller's explicit `image_area` argument through (it is `None` in production)
  instead of `current_area`.
- In the `z` handler, refresh `self._term_size` from `shutil.get_terminal_size()` (when
  not fixed) before recomputing the layout, so a zoom repaints once at the new size.

Test: a sequence-loop-level test with no `image_area` shows that a terminal-size change
reaches the inspect layout (frame/storyboard areas change), and `z` re-lays out at the
new size.

### 3. The full-file sha256 is recomputed on every frame request instead of once per open

`content_digest_for_path()` (`_viewer_video_frames.py`) streams the whole file. It runs
in `_InspectSession._prepare`, again in `metadata_for_frame_service`, again in every
`extract_frame` (every exact step, every `,`/`.`, and every step before the scrub track
finishes), and again in `start_scrub_track` and `build_storyboard`. Three hashes run
before the first frame. Measured: 75 ms per call on a 100 MB file, so a 500 MB recording
spends more than 1 s hashing before first paint. That breaks the 1 s first-pixel SLO and
the plan's "streamed sha256 of the file, once per open".

Fix:

- Memoize the digest per viewer process, keyed by the resolved path plus
  `(st_dev, st_ino, st_size, st_mtime_ns)`. Use a small bounded dict guarded by a
  `threading.Lock`, because the storyboard thread calls it too.
- A changed file (new size, mtime, or inode) must re-hash.
- Keep the public signature unchanged.

Test: one open plus several exact steps hashes the file exactly once (count `open()` or
`hashlib.sha256` calls), and modifying the file invalidates the memo.

### 4. The idle inspect view streams its unchanged status line forever

`_poll_background()` calls `_print_status()` on every 50 ms poll, even when nothing
changed. Measured: about 13.6 KB to the tmux client over 5 s while idle, against 0 B for
an idle shell. That is continuous SSH traffic and tmux redraw work, which violates R3
("no collateral damage").

Fix:

- Cache the last written status text and row.
- Write only when either changes.
- Invalidate the cache whenever the screen is cleared or fully repainted
  (`_paint(full=True)`, layout changes, motion return), so the line always reappears.

Test: N idle polls with an unchanged state produce one status write, and a
scrub-progress change produces another.

### 5. A missing `kitten` binary leaves a blank frame region

`_InspectSession._icat()` sets a notice and returns `False` when `kitten` is not on
PATH, so the frame region stays empty. R5's outcome is that terminals without kitty
graphics get a usable frame, and the non-kitty-fallback phase already falls back to the
cell preview when `kitten icat` fails.

Fix:

- Treat a missing `kitten` exactly like an icat failure: set `cell_fallback = True`,
  keep the notice naming `kitten`, and return `True` so the caller re-lays out and
  repaints as cells.
- `cleanup()` must still skip `kitten icat --clear` when `kitten` is absent.
- The docs sentence "Missing `ffprobe`/`ffmpeg`/`kitten` banners name the tool" stays
  true, so no docs edit is needed. `docs/reference/agent-attachments.md` sits at its
  word budget, so do not add words there.

Test: with `kitten` absent, the inspect view renders cells and shows the cell banner
plus the kitten notice.

### 6. The tmux terminal-smoke probe fails on master since sase-1tb.7

`tests/ace/tui/terminal_smoke/test_video_pane_tmux_probe.py` fails at
`child.expect(b"\x1b_G")`. The test harness pins `TERM=xterm-256color` for every test
(`tests/_conftest_environment.py`), so the probe's tmux client reports
`client_termname=xterm-256color`. `kitty_graphics_capable()` then says no, and the
viewer shows the cell preview. With `TERM=xterm-kitty` the client reports `xterm-kitty`
and images flow again. After that, the "APC within 300 ms of an `l` press" assertion
also fails, because of defect 1.

Fix:

- Set `env["TERM"] = "xterm-kitty"` in the probe env. Skip when `infocmp xterm-kitty`
  fails.
- Add a burst assertion: after the view settles, send `lllll` in one write and assert
  that the pane's status line reaches `00:05.0` (read it with
  `tmux capture-pane -p -t probe`). Use a clip of at least 6 s so +5 s is not clamped.
- Keep every existing assertion: title unchanged, fewer than 4 MB, no mpv marker, at
  least 2 APCs.

Run it with `just test-terminal-smoke -k video_pane` (that recipe is not guarded and
installs pexpect/pyte). Record the measured header→first APC time, `l`→APC time, and
total client bytes.

### 7. The epic's first-paint import growth blows the TUI import-footprint cap on master

`tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
fails on master: 11,929,502 bytes against an 11,924,777 cap.

- The 8fa9c1ad8b rebaseline (11,920,681) was measured on a tree without sase-1tb.6, and
  the tree at 8fa9c1ad8b already measures 11,927,211.
- `tools/tui_import_closure --diff 487825956a~1` attributes the epic's first-paint
  growth mostly to `_viewer_loop_media` (+7,070) and `_viewer_loop_terminal` (+1,729).
  Both are eager, through `graphics/__init__.py` → `_viewer_loop.py`.
- The test's policy forbids raising the cap for deferrable code: "defer the new edges".

Fix: move video-only code that only the (already deferred) inspect view uses out of the
eager modules into deferred modules.

- From `_viewer_loop_media.py` into a new module such as `_viewer_video_motion.py`:
  `artifact_video_motion_command`, `build_native_open_command`, `kitty_shm_available`,
  `reset_kitty_shm_cache`, `_probe_kitty_shm_once`, `_kitty_shm_query_exchange`,
  `motion_playback_block_reason`, `motion_playback_allowed`, `normalize_video_backend`.
  - Keep `load_merged_artifact_video_playback_config` and its `_coerce_*` helpers eager;
    the TUI parent needs them for the env snapshot.
  - Keep `artifact_video_player_command` and `video_still_preview_required` where they
    are.
- From `_viewer_loop_terminal.py` into a deferred module (the same new module or one
  such as `_viewer_kitty_capability.py`): `kitty_graphics_capable`,
  `reset_kitty_graphics_cache`, `_tmux_client_termname_once`,
  `_term_name_advertises_kitty_graphics`, `_KITTY_GRAPHICS_*`.
- Make `_viewer_loop_video.py` import from the new modules.
- Remove these names from the eager facades (`graphics/__init__.py`, `_viewer_loop.py`,
  `viewer.py` `__all__`/imports). Nothing in production imports them from the facades.
  Point the tests at the new modules (`test_video_motion.py`, `test_video_fallback.py`,
  `test_video_inspect.py`, `test_video_still.py`, `_helpers.py`).
- Delete `motion_playback_allowed` if nothing uses it after the move.
- Add the new module names to `_DEFERRED_MODULES` in `test_app_import_budget.py`.
- Do **not** change the baseline or cap.

Verify with `tools/tui_import_closure --diff 487825956a~1` and the budget test:
footprint below 11,924,777. If something unrelated outside this epic still keeps it
above the cap after the deferral, attribute it with `tools/tui_import_closure --diff`
and record the numbers in the close note. Do not raise the cap.

### 8. The legacy still/mpv video display path is unreachable

`run_video_inspect` replaced the video branch in every topology. Production no longer
calls `display_video_artifact()` (`_viewer_loop_media.py`) or
`display_video_still_artifact()`, `_extract_video_still()`, and
`_video_still_extraction_command()` (`_viewer_video_media.py`). Only tests and the
`_display_video_artifact` facade alias reference them, and the legacy outside-tmux
branch still runs mpv without checking for it.

Fix:

- Delete those functions, the `_display_video_artifact` alias in `_viewer_loop.py`, and
  their tests in `test_video_still.py`.
- Also delete `VideoDisplayResult` and `move_cursor_below_image` if nothing else uses
  them; grep `src/` and `tests/` first.
- Keep the metadata helpers the inspect view and frame cache use: `VideoMetadata`,
  `ffprobe_metadata_command`, `parse_ffprobe_metadata`, `probe_video_metadata`,
  `format_video_metadata_summary`, `terminal_pixel_size`.
- Keep `video_still_preview_required` and `video_vo_uses_kitty_placement`; motion
  dispatch uses them.
- Keep the `_viewer_loop.py` facade itself, minus that alias, plus the `read_key`,
  `run_command`, and `select_pane` seams.

## Verification

1. Targeted tests:
   `.venv/bin/python -m pytest tests/ace/tui/artifact_file_viewer tests/ace/tui/test_app_import_budget.py tests/main/test_doctor_tools.py -q`.
2. The probe: `just test-terminal-smoke -k video_pane`. Record the numbers.
3. `just fix`, then `sase tool run check` (hand it to `/sase_monitor` if it will not fit
   your synchronous limit). Do not run `just check-full`. Treat UNKNOWN triage items as
   yours.
4. `just symvision` clean, with no new `--epic-symbol` rows.

## Final step: close epic sase-1tb (do this yourself, in the same turn)

1. Run `sase bead epic-symbols sase-1tb`. The land agent found none. For any entry that
   now appears, resolve the symbol (wire it up, privatize it, add a non-test pragma, or
   delete it per the Symvision epic-whitelist policy in `symvision.md`). Do not leave an
   entry keyed to sase-1tb.
2. Close the epic. Fill in the bracketed parts with your own results:

   ```bash
   sase bead close sase-1tb --note "Land verification: all 7 phases (sase-1tb.1-.7) closed; source at f28da33a13 reviewed against the plan (still/metadata path, env config snapshot, lazy imports, ANSI clear, child spans, off-thread tmux split/decorate with launch context and 2s timeouts, frame/scrub/storyboard cache, inspect view and keys, motion backends/open_command/shm probe/native handoff, kitty capability and cell fallback, doctor rows, docs, config schema). Integration: commits since 487825956a outside the epic (8a6451068e, 8fa9c1ad8b, 71f0060927, cd0b9c5273, 1c626f39e5, be6adc20d0, 81656950c6, efc66a420b, 1e83f12981, 07aedfc8ca) neither overlap nor duplicate the viewer; cd0b9c5273 resolved the sase-1tb.6 MemoryAuditRow follow-up. Follow-up triage: see the land-triage note (sase-uz +1, sase-r2 +1, features sase-1td/1te/1tf/1tg, declines). Landing fixes: [list defects 1-8 fixed, with commit]. Measured: [probe header->first APC ms, l->APC ms, client bytes, burst reaches +5s; child first_visual ms; import footprint vs 11,924,777 cap]. Checks: [sase tool run check id + result], just symvision clean."
   ```

   - If the close is refused because epic-symbol entries remain, clean them up and close
     again.
   - Never use `--force` merely to make the close succeed.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   shown by `sase bead read sase-1tb` (`plan:202611/fast_video_artifact_pane.md`;
   frontmatter currently `status: wip`).
5. sase-1tb has no `parent_bead`, so the landing is complete after step 4.
