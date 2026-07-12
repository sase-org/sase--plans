---
create_time: 2026-07-09 13:36:36
status: done
prompt: 202607/prompts/agents_view_hints_disappear.md
tier: tale
---
# Fix: Agents-tab `v` (view) hints disappear before the user submits input

## Problem / Symptom

On the **Agents** tab of the `sase ace` TUI, pressing `v` (the "view files" keymap) is supposed to:

1. Re-render the agent detail panel with numbered `[N]` markers next to every file path / commit (the "view-mode
   hints"), and
2. Mount a `HintInputBar` below the panel so the user can type hint numbers (e.g. `3`, `1-5`, `3@`, `3%`) and press
   Enter.

The reported bug: **sometimes the `[N]` hint markers vanish from the detail panel before the user finishes typing and
submits**, leaving an input bar that references hints the user can no longer see. The behavior is intermittent
("sometimes"), which points to a timing/race condition rather than a deterministic logic error.

Note: this is about the **view-mode `[N]` file/commit hints** rendered inside the detail panel, NOT the per-row "live
file-change pencil" badge (a separate feature covered by `tests/ace/tui/test_agents_live_hint_refresh.py`). The two
share the word "hint" but are unrelated.

## Root Cause

When `v` is pressed on the Agents tab, `FileViewingMixin._view_agent_files` (`src/sase/ace/tui/actions/hints/_files.py`)
renders the panel **with hints** via `AgentDetail.update_display_with_hints(agent)`, stores the resulting hint state
(`_hint_mappings`, `_hint_commit_views`, `_hint_tool_call_reports`), sets `_hint_mode_active = True`, and mounts the
`HintInputBar` into `#agent-detail-container`.

The hint markers live in the **body of the detail panel** (`#agent-detail-panel` / `#agent-prompt-panel`). The input bar
is a **separate sibling widget**. So anything that repaints the detail panel body — without re-applying hints — silently
erases the visible hint numbers while leaving the input bar in place. That is exactly the observed symptom.

Every agents-tab detail repaint funnels through three helpers in `src/sase/ace/tui/actions/agents/_display_detail.py`:

- `_apply_agent_detail_update(...)` → `agent_detail.update_display(...)` (full repaint)
- `_apply_agent_detail_immediate(...)` → `agent_detail.update_display_immediate(...)` (header-only repaint)
- `_fire_debounced_detail_update(...)` → calls `_apply_agent_detail_update(...)`

**None of these three check `_hint_mode_active`.** They unconditionally call the plain (hint-free) `update_display` /
`update_display_immediate`, which repaint the panel body and drop the `[N]` markers.

This is a clear asymmetry — the two sibling code paths that _should_ mirror this one both already guard against it:

- The **auto-refresh tick** short-circuits during hint mode: `src/sase/ace/tui/actions/event_refresh/_auto_refresh.py`
  returns early when `_hint_mode_active` is set (around line 128), so _watcher-driven periodic_ refreshes do not wipe
  hints.
- The **ChangeSpecs-tab** detail path is hint-aware: `_apply_detail_panel_update`
  (`src/sase/ace/tui/actions/changespec/_display.py`, around line 214) branches to `update_display_with_hints(...)` when
  `_hint_mode_active` is true and re-stores the hint mappings. That is precisely why the same bug is _not_ observed on
  the ChangeSpecs tab.

The Agents-tab detail path never received the equivalent guard, so any repaint that is NOT the auto-refresh tick will
wipe the hints.

### Why it is intermittent ("sometimes")

The auto-refresh tick is guarded, but several **other** repaint paths are not, and they fire only when a background
event happens to land during the (usually brief) window between pressing `v` and submitting:

1. **In-flight async agent load completes during the hint window.** The auto-refresh guard prevents _new_ loads from
   starting during hint mode, but a load already in progress when `v` was pressed still finishes and runs its finalize
   path (`src/sase/ace/tui/actions/agents/_loading_finalize.py` →
   `_refresh_agents_display_after_finalize(..., defer_detail=True)`), which schedules `_fire_debounced_detail_update` →
   `_apply_agent_detail_update`. This is the most common trigger when the user is watching a running agent (running
   agents emit output continuously, so loads are frequently in flight).
2. **Notification unread projection.** `_patch_unread_completed_agent_changes`
   (`src/sase/ace/tui/actions/agents/_notification_unread_projection.py`, around line 41) calls
   `_refresh_agents_display(list_changed=True, defer_detail=True)` when a completed-agent row can't be patched in place.
   This runs inside completion polling, which executes _before_ the `_hint_mode_active` guard in the auto-refresh tick —
   so the guard does not cover it.
3. Assorted direct action handlers (marking, tagging, revive, etc.) also call `_refresh_agents_display`, but those are
   user-initiated and unlikely mid-hint-entry.

All of these ultimately repaint through the same three unguarded helpers, so the intermittency is fully explained and a
single centralized fix addresses every path.

## Fix Design

Make the Agents-tab detail-render helpers **hint-aware**, mirroring the proven ChangeSpecs-tab pattern. When
`_hint_mode_active` is true and the Agents tab is active, a repaint must re-render **with** hints (and refresh the
stored hint mappings) instead of wiping them.

### Approach: re-render with hints (mirror the ChangeSpecs pattern)

In `src/sase/ace/tui/actions/agents/_display_detail.py`, guard the detail repaint so that during Agents-tab hint mode
the panel is rendered via `AgentDetail.update_display_with_hints(current_agent)` and the returned `AgentHintRender` is
re-stored into the same state fields that `_view_agent_files` populates:

- `_hint_mappings` ← `render.file_hints`
- `_hint_commit_views` ← `render.commit_views`
- `_hint_tool_call_reports` ← `render.tool_call_reports`

Concretely:

- Add a small helper (e.g. `_render_agent_detail_with_hints(agent_detail, current_agent)`) that calls
  `update_display_with_hints` and re-stores the three hint-state fields.
- In `_apply_agent_detail_update`, when `_hint_mode_active` and `current_tab == "agents"` and a current agent exists,
  route through the helper instead of `agent_detail.update_display(...)`. Keep the existing footer/
  onboarding/empty-state handling unchanged (they already run after the render branch).
- Apply the same guard to `_apply_agent_detail_immediate` (the header-only path used during j/k). Since there is no
  header-only "with hints" variant and this path is not hot during hint mode, routing it through the same hint-aware
  full render is acceptable and keeps behavior consistent.
- `_fire_debounced_detail_update` needs no separate change because it delegates to `_apply_agent_detail_update`.

Rationale for re-rendering _with_ hints (rather than simply skipping the repaint):

- It is the exact pattern already used and proven on the ChangeSpecs tab, minimizing behavioral surprise and regression
  risk.
- It keeps the detail content fresh while preserving hints. Because view-mode hint numbering is assigned top-to-bottom
  and new agent output appends at the bottom, existing hint numbers stay stable; any newly appeared paths simply get
  appended numbers, and `_hint_mappings` is updated to stay consistent with what is on screen (so `parse_view_input`
  still resolves correctly).
- Focus is preserved: the `HintInputBar` is a separate mounted widget, so repainting the panel body does not remount it
  or steal focus.

### Scope / guard correctness

- On the Agents tab, the only hint bar is the view-files mode (hooks/mentors/accept/rewind hint bars are
  ChangeSpecs-only, mounted into `#detail-panel`). Gating on `_hint_mode_active and current_tab == "agents"` in these
  agents-only helpers is therefore sufficient and safe.
- The teardown path is already correct and needs no change: `_remove_hint_input_bar`
  (`src/sase/ace/tui/actions/hints/_processing.py`) sets `_hint_mode_active = False` **before** calling
  `_refresh_agents_display()`, so after submit/cancel the final repaint renders normally (no leftover hints).

## Files to Change

- `src/sase/ace/tui/actions/agents/_display_detail.py` — add the hint-aware guard + helper; route
  `_apply_agent_detail_update` and `_apply_agent_detail_immediate` through hinted rendering when Agents-tab hint mode is
  active.

No changes needed to the auto-refresh guard, the ChangeSpecs path, the hint teardown path, or the hint rendering widget
— those already behave correctly.

## Testing Plan

- **Regression unit test** (new file under `tests/ace/tui/`, e.g. `test_agents_view_hint_survives_refresh.py`):
  construct the app state with `_hint_mode_active = True`, `current_tab == "agents"`, and a selected agent; drive
  `_apply_agent_detail_update` (and `_fire_debounced_detail_update`) and assert it renders **with** hints (e.g.
  `update_display_with_hints` is invoked and the hint-state fields are re-stored / preserved) rather than the plain
  `update_display`. Add a companion assertion for `_apply_agent_detail_immediate`.
- **Path-level test**: simulate the finalize race — hint mode active, then invoke the finalize/deferred refresh path —
  and assert the hints are preserved (mappings unchanged / still populated).
- Confirm the existing ChangeSpecs-tab behavior is unaffected (its guard is untouched).
- Consider whether a PNG visual snapshot adds value; the unit tests above are the primary guard and the behavior is
  easier to assert at the render-dispatch level than pixel level.

## Validation

- `just install` (ephemeral workspace may have stale deps).
- `just check` (ruff + mypy + tests).
- Manual smoke: on the Agents tab, select a running agent, press `v`, wait for a background refresh to land, and confirm
  the `[N]` hints remain visible until submit/cancel.

## Out of Scope

- The per-row live file-change "pencil" badge refresh (`test_agents_live_hint_refresh.py`) — unrelated.
- Reworking the auto-refresh guard or the ChangeSpecs detail path — both already correct.
- Any change to hint parsing, the `HintInputBar` widget, or the commit-view modal.
