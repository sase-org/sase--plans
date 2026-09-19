---
tier: tale
title: Bulk-select tmux workspaces from the Agents chooser
goal:
  Let users mark multiple current or linked workspace targets in the Agents-tab tmux
  chooser and open the marked set without regressing single-target shortcuts, feedback,
  or TUI responsiveness.
size: medium
proposed_by: bbugyi200.athena.0nk
create_time: 2026-09-18 22:08:09
status: wip
---

# Plan: Bulk-select tmux workspaces from the Agents chooser

## Outcome

Pressing `t` on an Agents-tab row with cached opened-workspace context continues to show
the `Tmux Workspace` chooser. Inside that chooser, `m` marks or unmarks the highlighted
target and advances to the next row. `Enter` opens every marked target in the chooser's
displayed order, or opens only the highlighted target when nothing is marked. The
existing single-key selectors remain an immediate single-target shortcut, and `q` /
`Esc` still cancel without opening anything.

This is a `medium` tale: one implementation agent can make and verify the bounded TUI
change, but it crosses modal interaction state, tmux action dispatch, help/docs, unit
coverage, and an existing visual golden.

## Interaction contract

- Reserve lowercase `m` from the chooser's generated selector pool. Retain enough unique
  lower-case, digit, and upper-case selectors for the current maximum of 50 opened
  linked workspaces plus the always-present `CURRENT` row.
- Render a compact `[x]` marker on marked rows and no empty checkbox on unmarked rows,
  following the existing artifact-file chooser convention. Keep `CURRENT` first and
  preserve the existing `CURRENT` / `LINKED`, label, path, role, and reason content.
- `m` toggles the highlighted row, refreshes only that option plus the hint, and moves
  the highlight to the next row with wraparound. Re-marking an already marked row
  removes it.
- The hint advertises `m` and appends `marked: N` only while marks exist.
- `Enter` and mouse/OptionList activation open the marked set when non-empty; otherwise
  they open the highlighted row. Marked targets are returned in display order rather
  than mark order so opening is deterministic.
- A displayed selector still immediately opens just its associated row, preserving the
  chooser's fast single-target path even if marks already exist.
- Cancellation returns no selection and performs no tmux work.

## Implementation

### 1. Add mark state and an explicit multi-selection result to the modal

Update `src/sase/ace/tui/modals/agent_workspace_tmux_modal.py`:

- Add `m -> toggle_mark` to the modal-local bindings and exclude `m` from generated
  selectors.
- Store marked row indexes in a set, but expose selected indexes or choices in original
  row order through one explicit modal result shape. Normalize the single-highlight,
  marked-set, selector, mouse, and cancel paths at this boundary so the caller does not
  have to infer ambiguous `int` versus `list` semantics.
- Extend `_agent_workspace_tmux_option_text()` with a `marked` input and use targeted
  `OptionList.replace_option_prompt_at_index()` updates. Do not rebuild/remount the
  chooser on each mark.
- Update the hint in place after every toggle and preserve keyboard focus and existing
  `j`/`k`, arrow, and Ctrl navigation behavior.

### 2. Open selected targets as one ordered, non-blocking operation

Update `src/sase/ace/tui/actions/agents/_panel_tmux.py`:

- Change the chooser callback to accept the explicit selection result, reject cancel or
  empty results, and map selections back to the immutable choices captured when the
  modal opened.
- Preserve current-target resolution and linked-target semantics: `CURRENT` uses the
  selected agent's existing current-workspace target, while each `LINKED` row uses its
  recorded directory and window name. Expand `~` and validate linked directories; a
  stale linked target is reported and skipped without preventing other marked targets
  from opening.
- Process valid targets sequentially in chooser order, using the existing rule of
  selecting an exact-name tmux window when it exists and otherwise creating it in the
  target directory. The last selected target therefore remains the active tmux window.
- Move directory checks and all tmux subprocess calls for the selection onto one
  thread-backed Textual worker. Marshal completion feedback to the UI thread. Route
  existing single-target chooser/direct entry points through the same dispatch helper
  where practical so bulk support does not multiply blocking subprocess paths.
- Preserve the existing single-target `Opened tmux window`, `Switched to tmux window`,
  missing-directory, and missing-command feedback. For multiple selections, report one
  concise opened/switched/skipped summary and identify skipped targets, while continuing
  past a stale linked directory. Do not claim success for a target whose tmux command
  failed.

The worker boundary is important because a marked set can expand one keypress into many
filesystem checks and subprocess calls; none of that work should run on Textual's event
loop or serial message pump.

### 3. Keep user-facing guidance synchronized

- In `src/sase/ace/tui/modals/help_modal/agents_bindings.py`, revise the Agents `t` help
  description within the existing 32-character description limit so it mentions that the
  chooser supports marking multiple targets.
- In `docs/ace.md`, document the chooser controls and explicitly distinguish quick
  single-key opening from `m` plus `Enter` bulk opening.
- Do not add an app-level keymap field or change `src/sase/default_config.yml`: `m` is a
  modal-local control, matching other selection modals. Do not change the Agents-tab `m`
  behavior outside the chooser.
- No Rust-core change is needed; selection rendering and tmux window launching are
  TUI-only behavior.

## Tests and visual verification

Extend `tests/ace/tui/modals/test_agent_workspace_tmux_modal.py` to cover:

- selector generation excludes `m` as well as navigation/cancel keys and still covers 51
  rows uniquely;
- marked and unmarked row rendering;
- toggling, unmarking, auto-advance, wraparound, and live marked-count hints;
- `Enter` and OptionList activation return all marked rows in display order, while the
  unmarked and quick-selector paths still return exactly one row;
- cancel with marks returns no selection.

Extend `tests/ace/tui/actions/test_agent_panel_tmux.py` to cover:

- a mixed `CURRENT` plus `LINKED` marked selection opens every valid target in chooser
  order with the expected directories/window names;
- existing-window and new-window outcomes remain correct, with the final selected target
  active;
- a missing linked directory is skipped while remaining marked targets still open;
- cancel/empty selection launches nothing;
- dispatch uses a thread-backed worker and reports single versus bulk outcomes without
  issuing tmux subprocesses on the UI thread.

Update `tests/test_keymaps_display_help_agents.py` for the revised help copy. Update
`tests/ace/tui/visual/test_ace_png_snapshots_agents_modals.py` so the existing tmux
workspace modal snapshot marks at least one row and asserts the visible `[x]`, `m` hint,
and marked count. Regenerate
`tests/ace/tui/visual/snapshots/png/agent_workspace_tmux_modal_100x28.png` and inspect
the retained visual report and changed PNG for alignment, clipping, focus, marker
contrast, and readable hints.

Verification sequence:

1. Run the focused modal, action, and help tests.
2. Run the targeted `just fix-tui-screenshots -- <pytest selector>` flow for
   `test_agent_workspace_tmux_modal_png_snapshot`, inspect the report and golden, then
   run its check-only equivalent if needed.
3. Run `just fix`, followed by the repository-default `just check`. Escalate to the
   monitored `just check-full` only if the scoped gate broadens or reports an unusual
   selection, leaving the normal exhaustive landing gate to the host otherwise.

## Acceptance criteria

- A user can press `m` on two or more chooser rows, press `Enter`, and get one tmux
  window per valid marked workspace in visible order.
- Marking is visually obvious, auto-advances efficiently, wraps, and can be undone.
- Existing `t` direct-open behavior without cached workspace context, `T` primary-open
  behavior, one-key selector opening, navigation, mouse selection, and cancellation do
  not regress.
- Missing linked directories or an individual tmux failure do not prevent other marked
  workspaces from being attempted, and feedback distinguishes successes from skips or
  failures.
- Bulk opening introduces no synchronous filesystem or subprocess work on the Textual
  event loop.
- Help, docs, focused tests, the reviewed visual golden, and `just check` all agree with
  the shipped behavior.
