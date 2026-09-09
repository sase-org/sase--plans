---
tier: tale
title: Stabilize flaky CI tests breaking master
goal: "Master CI stops failing intermittently: PNG visual snapshots of focused prompt
  inputs no longer depend on Textual's wall-clock cursor blink phase, the
  pending-question marker test no longer races the questions flow, and matrix-job visual
  failures upload inspectable artifacts.

  "
create_time: 2026-09-09 19:53:11
status: wip
---

# Plan: Stabilize flaky CI tests breaking master

## Context

GitHub Actions CI for `sase-org/sase` master is failing intermittently. Recent history
(all on 2026-07-17):

- Run 29547667755 (`0c438540c`): `test (3.12)` failed —
  `tests/ace/tui/visual/test_ace_png_snapshots_vcs_repo_completion.py::test_vcs_repo_error_panel_png_snapshot`
  with a PNG mismatch of 312/1520532 pixels (all "material", alpha-aware color
  distance > 8).
- Run 29545557760 (`560177340`): `test (3.12)` failed — BOTH
  `test_vcs_repo_completion_panel_png_snapshot` and
  `test_vcs_repo_error_panel_png_snapshot` in the same file.
- Run 29544111582 (`a0a81e445`): `test (3.14)` failed —
  `tests/test_axe_run_agent_helpers_questions.py::test_pending_question_marker_deleted_on_kill`
  with `assert None is not None`.
- In every one of those runs the dedicated `visual-test` job (focused
  `just test-visual`) PASSED; only the full-suite `test` matrix job (3.5h+ wall clock
  under parallel load) flaked. An adjacent run (`d39577633`) passed entirely. These are
  load-dependent flakes, not regressions from the commits they landed on.

## Root causes

### 1. Cursor blink phase in PNG visual snapshots

The failing snapshot tests mount a focused `PromptInputBar` and capture the screen.
Textual's `TextArea`/`Input` cursor blink is a recurring 0.5s wall-clock timer;
`wait_for_visual_idle()` in `tests/ace/tui/visual/_ace_png_snapshot_waits.py`
deliberately ignores recurring timers, and its 3-stable-frame convergence window (~30ms)
is far shorter than a blink half-period, so it converges in whichever blink phase the
loaded runner happens to be in. The 312-pixel diff is exactly one character cell
(1520532 px / 4800 cells ≈ 317 px/cell): the cursor block.

Three test files were already patched for this exact flake by setting
`cursor_blink = False` after focus (`test_ace_png_snapshots_placeholder_completion.py`,
`test_ace_png_snapshots_prompt_stack.py`, and — via commit `50809bdb8` —
`test_ace_png_snapshots_xprompt_save.py`). The files that still mount focused prompt
bars WITHOUT disabling blink are:

- `tests/ace/tui/visual/test_ace_png_snapshots_vcs_repo_completion.py` (failing in CI)
- `tests/ace/tui/visual/test_ace_png_snapshots_vcs_project_completion.py`
- `tests/ace/tui/visual/test_ace_png_snapshots_vcs_ref_completion.py`
- `tests/ace/tui/visual/test_ace_png_snapshots_frontmatter_panel.py` (focused cell/list
  editors in its edit-mode tests)

Textual semantics confirm the fix is golden-neutral: setting `cursor_blink = False` on a
mounted focused widget pauses the blink timer and pins the cursor VISIBLE
(`TextArea._watch_cursor_blink` → `_pause_blink(visible=self.has_focus)`;
`Input._watch_cursor_blink` → `_cursor_visible = True`), which is the phase fast local
capture runs (and therefore the committed goldens) encode.

### 2. Fixed-delay race in the pending-question marker tests

`_run_questions_flow()` in `tests/test_axe_run_agent_helpers_questions.py` starts a
helper thread that does `time.sleep(0.05)` and then snapshots `pending_question.json`
and responds/kills. The product code (`handle_questions_flow` in
`src/sase/axe/run_agent_helpers_questions.py`) creates the question gate, sends
notifications, and only then writes the marker before entering its poll loop. Under CI
load the main thread can take longer than 50ms to reach the marker write; the helper
then snapshots nothing, triggers the kill, the poll exits immediately, and
`marker_payload` is `None`. The same fixed delay also underpins the
`send_response_after` tests (they read `session_id`/`request_path` from the marker
snapshot), so they share the race.

## Changes

### A. Pin cursor blink centrally in the visual idle wait

In `tests/ace/tui/visual/_ace_png_snapshot_waits.py`, add a
`_disable_cursor_blink(page)` helper beside `_clear_transient_button_state` that walks
every screen in `page.app.screen_stack` (children via `walk_children`) and sets
`cursor_blink = False` on every `textual.widgets.Input` and `textual.widgets.TextArea`
instance. Call it each iteration of `wait_for_visual_idle()` next to
`_clear_transient_button_state(page)`. Reactive no-op semantics make repeat assignment
free, and once disabled the blink timer never flips the frame again — so any capture
that follows a `wait_for_visual_idle()` call (the established pattern in every PNG test)
is blink-deterministic.

Then remove the now-redundant per-file blink disables so there is exactly one mechanism:

- `test_ace_png_snapshots_placeholder_completion.py` (the
  `text_area.cursor_blink = False` loop in `_mount_prompt_bar`)
- `test_ace_png_snapshots_prompt_stack.py` (same pattern + its comment)
- `test_ace_png_snapshots_xprompt_save.py` (the `name.cursor_blink = False` line + its
  comment)

Keep the removal minimal — do not otherwise restructure those helpers.

### B. Replace the fixed sleep with a bounded marker wait

In `tests/test_axe_run_agent_helpers_questions.py`, change `_run_questions_flow`'s
helper thread to wait for the marker file instead of sleeping a fixed 50ms: poll
`os.path.exists(marker_path)` every ~10ms with a generous deadline (~10s). On success,
snapshot the marker as today; if the deadline expires, proceed anyway (the flow blocks
until the helper responds/kills, so the helper must always eventually act — the test
then fails with its existing clear assertion rather than hanging). All
`_run_questions_flow` callers go through the marker-writing path (the auto-approve test
calls `handle_questions_flow` directly and is unaffected), so the wait is safe for every
call site and also fixes the same latent race in the `send_response_after` tests.

### C. Upload visual failure artifacts from the test matrix job

The `test` matrix job runs the visual suite on its 3.12 leg but has no artifact upload,
which is why the failing runs left nothing to inspect (the upload steps exist only in
the `visual-test` job). In `.github/workflows/ci.yml`, add one step to the `test` job
after "Run tests": upload `.pytest_cache/sase-visual` with `actions/upload-artifact@v4`,
`if: failure() && matrix.python-version == '3.12'`, a distinct artifact name (e.g.
`ace-visual-artifacts-test-matrix`), and `if-no-files-found: ignore`. Do not add the
HTML report steps — the raw actual/expected/diff/summary artifacts are what diagnosis
needs.

## Validation

- `just test-visual` — full PNG suite must pass unchanged (proves the central blink pin
  is golden-neutral). If any golden shifts because it was recorded in the hidden-cursor
  phase, inspect the diff artifact to confirm it is only a cursor cell and re-record
  that golden with `--sase-update-visual-snapshots`, noting it in the commit message.
- Re-run the previously flaky visual tests several times in a loop (e.g.
  `pytest tests/ace/tui/visual/test_ace_png_snapshots_vcs_repo_completion.py` repeated
  ~5x) to shake out residual nondeterminism.
- Run `pytest tests/test_axe_run_agent_helpers_questions.py` repeatedly (~10x) — the
  marker-wait must hold under repetition.
- `just check` for the full lint + type + test gate.
- CI yaml change is validated by review only (it is failure-path observability; it
  cannot break a green run), plus `just check`'s normal gates.

## Risks

- Forcing the cursor visible could deterministically break a golden that was recorded in
  the hidden phase. Mitigation: full local visual run before landing; re-record only
  confirmed cursor-cell diffs.
- The screen walk in `wait_for_visual_idle` runs every iteration; it is bounded by
  widget count and trivially cheap next to `export_svg`, so no measurable slowdown is
  expected in the visual suite.
- The 10s marker deadline slightly delays only the already-broken failure path; passing
  paths proceed as soon as the marker lands.
