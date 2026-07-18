---
tier: tale
title: Bound the plain-text render fallback so huge commit diffs cannot freeze the
  TUI
goal: 'Rendering any commit diff (or other over-cap content) in sase ace never blocks
  the UI thread for more than a perceptible instant: the lazy-syntax plain-text fallback
  is always bounded by both bytes and lines, and the Artifacts Commits detail pane
  and commit view modal render multi-megabyte diffs instantly with a clear truncation
  notice.

  '
create_time: 2026-07-18 06:49:02
status: done
prompt: 202607/prompts/ace_commit_diff_render_freeze.md
---

# Plan: Bound the plain-text render fallback so huge commit diffs cannot freeze the TUI

## Problem

Navigating the Commits sub-tab of the Artifacts tab froze `sase ace` hard (2026-07-18 06:33,
`~/.sase/logs/tui_stalls.jsonl` pid 1109452). The stall watchdog captured the UI thread stuck inside Textual's
compositor at `rich/segment.py:337 split_and_crop_lines → text.partition("\n")` across three consecutive samples with
**no recovery event** — the frame on screen stayed at "Loading diff…" because the freeze happened while compositing the
_new_ frame containing the just-loaded diff.

The selected commit was `4be1c25` in the `plans` sidecar ("chore(beads): close sase-6q.2"). Its diff is **3.26 MB across
only 4,296 lines** (beads event-stream JSONL rewrites; lines up to ~7 KB — byte-heavy, not line-heavy).

## Root cause (verified empirically)

1. The diff loads off-thread correctly, then `build_commit_detail`
   (`src/sase/ace/tui/widgets/artifacts/commits_rendering.py`, the `lazy_renderable(...)` call in the diff branch)
   renders it **without `max_render_lines`**.
2. In `src/sase/ace/tui/util/lazy_syntax.py`, content over the highlight cap (64 KB / 1,500 lines) falls back to a plain
   `Text` of the **entire** content. The `max_render_lines` safety valve is opt-in, and only the file panel opts in.
3. Even opting in would not have helped here: `FILE_PANEL_MAX_RENDER_LINES = 5_000` is a **line** cap and this diff has
   only 4,296 lines. There is no byte cap on the plain path.
4. A span-less Rich `Text` renders as a **single giant segment** (verified: one 3.29 MB segment).
   `Segment.split_and_crop_lines` then partitions the remainder per output line — **quadratic** in content size.
5. Measured on the real diff through the real pipeline (`console.render` + `split_and_crop_lines` at width 110): **26.8
   s of hard event-loop block for a single render pass** (26.4 s in the split alone), and re-renders (e.g. width change
   when the scrollbar appears) repeat it. Scaling check: split time quadruples per content doubling (0.02 → 0.07 → 0.26
   → 1.04 s for 0.38 → 3.0 MB synthetic content).

The same unguarded pattern exists in `CommitViewModal._build_content` (`src/sase/ace/tui/modals/commit_view_modal.py`),
and every other `lazy_renderable` caller except the file panel passes no cap at all (`_agent_display_render.py`,
`_agent_display_attempts.py`, `_agent_display_content.py`, `_workflow_render.py`, `preview_panel_modal.py`). The file
panel itself is still vulnerable to byte-heavy content (long lines under the 5,000-line cap).

Related but non-causal: `parse_unified_diff_deltas` on the 3.26 MB diff takes ~8 ms on the UI thread — acceptable; leave
it as is.

This is presentation-only Textual/Rich work, so it stays in this repo (no Rust core changes).

## Design

Make the plain-text fallback in `lazy_renderable` **unconditionally bounded**, instead of relying on callers to opt in.
The fallback exists to keep rendering fast; an unbounded fallback defeats its purpose.

### 1. Always-on byte and line caps in `lazy_syntax.py`

- Add a `PLAIN_RENDER_MAX_BYTES` constant for the plain fallback. Choose the value by benchmark so the worst-case plain
  render (wrap + segment split) stays within the module's stated sub-100 ms budget; ~128 KB is the expected ballpark
  (quadratic split cost at 128 KB measures ~0.04 s; validate on the target host with the bench below).
- In `build_plain`, apply **both** caps head-biased (diffs are head-shaped: file headers first):
  - line cap: use the caller's `max_render_lines` when given, otherwise default to `FILE_PANEL_MAX_RENDER_LINES`
    (consider renaming or introducing `PLAIN_RENDER_MAX_LINES` with the file panel keeping its existing constant as an
    alias) — `None` must no longer mean "unbounded";
  - byte cap: truncate the rendered content to `PLAIN_RENDER_MAX_BYTES`, cutting at a line boundary where practical.
  - The truncation notice must report what was elided (remaining lines and/or approximate size), whichever cap tripped.
- Parameterize the trailing truncation hint. The current hard-coded "press E to open in editor" is file-panel-specific;
  the default should be a generic "truncated for display" style message, with the file panel passing its existing
  wording so its UX is unchanged. A surface must never advertise a keybinding it does not have.
- Extend `_SyntaxRenderableKey` with any new parameters that alter output (byte cap if it becomes a parameter, hint
  text) so cached renders cannot be served for the wrong configuration (over-broad cache keys serve stale rows).

### 2. Defense-in-depth: never emit one giant segment

In the plain fallback, build the capped body so it renders as per-line segments (e.g. assemble the `Text` line-by-line
or use a per-line construction) rather than a single span-less blob. With the caps in place this is belt-and-suspenders,
but it converts any future cap-tuning mistake from quadratic to linear. Keep it only if it stays simple and does not
change rendered output.

### 3. Wire the commit-diff call sites

- `build_commit_detail` in `src/sase/ace/tui/widgets/artifacts/commits_rendering.py` and
  `CommitViewModal._build_content` in `src/sase/ace/tui/modals/commit_view_modal.py`: pass an explicit line cap (the
  shared default is fine) and a context-appropriate truncation hint (e.g. suggesting `git show <sha>` / opening the
  repo; both surfaces keep their existing keymaps, so do not reference keys they lack).
- The remaining unguarded callers get the new default caps automatically; no per-site changes needed beyond confirming
  their tests still pass.
- Keep the diff loader unchanged — loading a multi-MB string off-thread is cheap; the render layer is the single right
  place to bound display cost.

## Testing

Extend `tests/ace/tui/util/test_lazy_syntax.py`:

- **Byte-heavy over-cap content** (few lines × ~7 KB lines, mimicking the beads-diff shape that caused the freeze):
  plain fallback's flattened content is ≤ the byte cap plus notices, even when the line count is under the line cap.
- **Default cap with `max_render_lines=None`**: content is bounded (no unbounded path remains).
- **Hint parameterization**: default generic message; explicit hint respected. Update the existing assertions that pin
  the old hard-coded hint (`tests/ace/tui/util/test_lazy_syntax.py`, `tests/test_file_panel.py` — the file panel should
  keep its exact current message).
- **Cache-key correctness**: differing hint/caps do not reuse each other's cached renderables.
- If the per-line construction from Design §2 is kept: assert no rendered segment of the plain fallback contains more
  than one newline (structural guard against the quadratic input shape — prefer structural asserts over timing asserts,
  which are flaky).

Extend `tests/ace/tui/test_commits_pane.py` and `tests/ace/tui/modals/test_commit_view_modal.py`: a multi-megabyte diff
yields a detail renderable whose flattened plain content is bounded and includes the truncation notice.

Run the PNG visual snapshot suite (`just test-visual`); snapshots use small fixtures, so no golden updates are expected
— investigate rather than blindly accept if any diff appears.

## Verification

- Micro-bench the capped worst case through the real pipeline (`console.render` + `Segment.split_and_crop_lines` at
  width ~110) to confirm the sub-100 ms budget at the chosen byte cap.
- Manual repro: `sase ace` → Artifacts → Commits, select a beads-maintenance commit in the `plans` sidecar with a
  multi-MB diff. With lowered watchdog thresholds (`SASE_TUI_STALL_*` / `SASE_TUI_PUMP_STALL_*`), confirm no new rows in
  `~/.sase/logs/tui_stalls.jsonl`; with `SASE_TUI_PERF=1`, confirm j/k key-to-paint p95 stays < 16 ms while navigating
  across the huge-diff commit. Also open the same commit in the commit view modal (enter) and scroll.

## Risks

- **Truncation hides diff content.** Mitigated by the notice stating what was elided and how to see the full diff; 128
  KB of head-biased diff covers the overwhelming majority of real commits (the highlight cap already ensures anything
  over 64 KB was plain text).
- **Behavior change for other `lazy_renderable` callers** (prompt panel markdown/json/pytb, workflow render, preview
  panel): content over the highlight caps now truncates at the default line/byte caps instead of rendering fully. This
  is the intended protection — those surfaces were equally freezable — but their tests must be swept for pinned
  full-content assertions.
- **Hint text changes** break the two existing pinned assertions; update them in the same change.
