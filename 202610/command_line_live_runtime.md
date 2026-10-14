---
tier: tale
title: Live-ticking runtime for running Command Line blocks
goal:
  The Command Line panel's running-block header counts up live (running 0s, 1s, 2s, ...)
  until the command exits.
size: small
decisions:
  long_runs:
    ask: Should running blocks switch to a minutes format once they pass 60 seconds?
    choices:
      seconds:
        Keep plain whole seconds ("running 547s"), matching the finished-block "547.3s"
        style
      minutes:
        Show "running 9m07s" past 60 s and "running 1h02m" past an hour; finished blocks
        keep "547.3s"
    default: seconds
    why: Seconds is today's running format and stays consistent with settled blocks
    answer: seconds
proposed_by: bbugyi200.athena.10a
decided_by: auto
create_time: 2026-10-14 06:27:31
status: wip
---

# Command Line panel: live-ticking runtime for running blocks

## Problem

In the TUI's `:` Command Line panel, a running block's header reads
`⠹ agent archive rebuild-index  running 0s · proc bh27g7` and stays at `0s` until the
command exits. Only then does the header switch to the settled
`exit 0 · 12.3s · 06:20 · proc …` form. The user wants the running header to count up
live while the command runs.

## Root cause

- `block_header_right()` (`src/sase/ace/tui/command_line/block_render.py`) renders the
  running header from its `elapsed` argument:
  `running_for = f"{elapsed:.0f}s" if elapsed else "0s"`.
- `_CommandLineBlockWidget.render_block()`
  (`src/sase/ace/tui/command_line/transcript.py`) passes `elapsed=block.elapsed`.
- `CommandLineBlock.elapsed` (`src/sase/ace/tui/command_line/session.py`) is `None`
  until the block settles. It is assigned only in `apply_exit_completion()` /
  `apply_local_block()` (`submit.py`) and in the terminal branch of `block_from_proc()`
  (`restore.py`). `None` renders as `0s`.
- The pump-free tail loop (`CommandLineTranscript._poll_once`, every
  `TAIL_POLL_SECONDS = 0.2`) already repaints every visible running block on each tick
  and advances its spinner. It never refreshes `block.elapsed`, so the repaint keeps
  drawing the same `0s`.

A second, latent bug: when a block is restored for a still-active proc (after a TUI
restart, or a Procs-pane `⏎` jump through `append_proc_block`), `block_from_proc()`
leaves `started_at` at its dataclass default `time.time()`. A live timer would then
restart from 0 at restore time instead of counting from the proc's real start.

## Design

Refresh `block.elapsed` on the existing 200 ms tail tick, right before the repaint that
already happens there. `render_block` / `block_header_right` stay pure: they still
render whatever `block.elapsed` holds. That gives us:

- no new timer, interval, or refresh path (TUI perf rules 2 and 5: the tick is already a
  pump-free task, and the update is a float subtraction);
- only visible running blocks pay for it, because `_poll_once` already gates on
  `_tail_target_visible`;
- deterministic visual snapshots: the PNG helpers stop the tail task and seed `elapsed`
  directly (`test_command_line_running_png_snapshot` seeds `elapsed=42.0` and asserts
  `running 42s`). That snapshot keeps passing unchanged.

When a hidden panel is reopened, the first paint may show the last ticked value for up
to one tick before the tail task catches up. That is acceptable and matches how the
spinner already behaves.

This is presentation-only Textual state, so it stays in this repo; nothing crosses the
`sase_core` boundary.

## Changes

### 1. `src/sase/ace/tui/command_line/transcript.py` — tick the clock in `_poll_once`

Restructure the per-widget loop body in `CommandLineTranscript._poll_once`:

1. Skip blocks that are not `block.running`, then skip blocks that fail
   `_tail_target_visible(widget)`. This is the same order as today, except that a
   missing `proc_id` no longer skips the block outright.
2. Only when `block.proc_id is not None`, run the existing cursor lookup/creation, the
   `await asyncio.to_thread(cursor.read_new)` read, and the `tail_text` / `lost_bytes`
   merge. Keep the existing `except Exception: continue` around the read.
3. After the read, **re-check `block.running`** (TUI perf rule 4: an exit completion can
   settle the block on the UI thread while the read is awaited). If the block is still
   running, set `block.elapsed = block.elapsed_seconds()`. If it settled, leave the
   final `elapsed` from `apply_exit_completion` alone. (`elapsed_seconds()` would return
   the same value because `finished_at` is set, but the guard makes the intent explicit
   and keeps the code from overwriting settled state.)
4. Advance `widget._spinner_index` and `widget.repaint()` exactly as today.

As a result, a `submitting` block (no `proc_id` yet) also ticks its spinner and elapsed
instead of freezing on the first frame and `0s` while the submit worker runs. No log
read happens for it.

Update the module docstring: it should say each tick refreshes the running block's
elapsed time as well as polling its log tail.

### 2. `src/sase/ace/tui/command_line/block_render.py` — stopwatch formatting

In `block_header_right`'s `("submitting", "running")` branch, floor the value instead of
rounding it, so the counter behaves like a stopwatch (`0s` for [0, 1), `1s` for [1, 2),
…). Today `f"{elapsed:.0f}s"` uses round-half-even and would flip to `1s` at roughly 0.5
s. Use `int(elapsed)` and keep the `None`/zero → `0s` fallback. Do not change the
settled-block `.1f` format.

> [!decision] long_runs = minutes Add a small private helper, for example
> `_running_duration(seconds: float) -> str`, used only by the running branch. It
> returns `"{s}s"` under 60 s, `"{m}m{s:02d}s"` under an hour (`9m07s`), and
> `"{h}h{m:02d}m"` from an hour up (`1h02m`). Floor at every step. Leave the settled
> `.1f` format and the completion-toast format in `exits.py` unchanged. Add unit cases
> for 59.9 → `59s`, 60.0 → `1m00s`, 547.3 → `9m07s`, and 3725 → `1h02m`.

### 3. `src/sase/ace/tui/command_line/restore.py` — real start time for active restored blocks

In `block_from_proc`'s non-terminal (`else: block.status = "running"`) branch, set
`block.started_at` from `_parse_proc_time(proc.started_at)`, falling back to
`_parse_proc_time(proc.created_at)` when `started_at` is missing or unparsable. Leave
the default only when both are unusable. The live timer then continues from the proc's
true start after a TUI restart or a Procs-pane jump. Leave the terminal branch alone.

### 4. `src/sase/ace/tui/command_line/session.py` — document the field

Add a short field comment on `CommandLineBlock.elapsed` that matches the existing
`started_at` comment style. While the block is running, the tail tick refreshes
`elapsed` with live seconds since submit. Once the block settles, it holds the final
runtime.

## Tests

Put the new tests next to the existing ones and keep each file's `__all__` list current
(these test modules export their test names, and `test_transcript_blocks.py` re-exports
from the split modules).

1. **Header formatting**: in `tests/ace/tui/command_line/test_panel_shell_submit.py`,
   extend `test_gutter_and_header_metadata` or add a sibling test asserting
   `block_header_right("running", elapsed=None) == "running 0s"`,
   `elapsed=0.7 → "running 0s"` (floor, not round), `elapsed=1.6 → "running 1s"`, and
   `elapsed=42.0 → "running 42s"`. With `proc_id="bh27g7xx"`, also assert the
   ` · proc bh27g7` suffix is still present.
2. **Live tick (Pilot)**: in `tests/ace/tui/command_line/test_policy_io.py`, model the
   test on `test_hidden_running_blocks_are_not_tailed`: monkeypatch
   `ProcLogCursor.read_new` with a spy, open the panel, call
   `screen.transcript.stop_tail_task()` so only manual polls run, and seed a running
   block with a `proc_id` and `started_at = time.time() - 5.0`. Then:
   - before polling, assert `block.elapsed is None` and the widget renders `running 0s`;
   - `await screen.transcript._poll_once()`, then assert `5.0 <= block.elapsed < 30.0`
     and that the widget's rendered text matches `running \d+s` with a value ≥ 5. Read
     it via `str(_CommandLineBlockWidget.render_block(block))` or the widget's
     renderable. Use bounded ranges, not exact values, because wall-clock time keeps
     moving on a loaded host.
3. **Submitting block ticks without a log read**: in the same file, a `submitting` block
   with `proc_id=None` and a backdated `started_at` gets `elapsed ≥ 3.0` after
   `_poll_once()`, its widget's `_spinner_index` advanced, and the `read_new` spy
   recorded no read for it.
4. **Settle-during-await guard**: the `read_new` spy settles the block mid-read by
   setting `status = "success"`, `finished_at`, and `elapsed = 1.25` for a specific proc
   id. After `_poll_once()`, assert `block.elapsed == 1.25`.
5. **Restore start time**: in
   `tests/ace/tui/command_line/test_transcript_blocks_state.py`, extend
   `test_block_from_proc_maps_terminal_and_running` or add a sibling. An active row with
   `started_at="2026-09-24T11:00:00Z"` restores with `block.started_at` equal to that
   epoch. An active row with `started_at=None` falls back to `created_at`. Both restore
   with `elapsed is None` until a tick.
6. Leave `test_command_line_running_png_snapshot` untouched. It must still pass, because
   the snapshot helper stops the tail task and seeds `elapsed=42.0`. No PNG goldens
   should change for the `seconds` default, and none for `minutes` either, because 42 s
   is under a minute.

## Verification

- Run the targeted tests first:
  `pytest tests/ace/tui/command_line/test_panel_shell_submit.py tests/ace/tui/command_line/test_policy_io.py tests/ace/tui/command_line/test_transcript_blocks_state.py tests/ace/tui/command_line/test_transcript_blocks.py`.
- Then run `just check`, following the lint-and-test memory note. Do not run
  `just check-full`.
- Optional manual check: in `sase tui`, open `:` and run a slow command (for example
  `agent wait … --timeout 1m`). Confirm the header counts `running 0s → 1s → 2s …` and
  then settles to `exit N · X.Xs · HH:MM · proc …`. Close and reopen the panel mid-run
  and confirm the timer resumes from the true elapsed time within a tick.

## Non-goals

- No change to the Procs pane's `m:ss` elapsed column, the completion toast text, or the
  settled-block header format.
- No new timers or intervals. The existing 200 ms pump-free tail tick is the only clock.
- No keymap or `default_config.yml` changes, and no help-modal changes. No keybinding or
  option changes.
