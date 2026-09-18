---
tier: tale
title: Default the Agents detail view to metadata only
goal: "Make a fresh Agents tab devote the full detail area to agent metadata while
  keeping File and LLM Calls content available only after the user explicitly chooses a
  visible secondary layout through the p-key Agent view picker.

  "
size: small
proposed_by: bbugyi200.apollo.0e.f0.f0.w0
create_time: 2026-09-18 17:29:43
status: wip
---

# Default the Agents Detail View to Metadata Only

## Problem and current behavior

`AgentDetail` currently initializes its session-local layout to
`DetailLayoutMode.SECONDARY_LARGER` and its secondary source to `DetailPanelMode.AUTO`.
As soon as the selected agent's file content becomes available, the Agents tab therefore
gives the File panel 70% of the detail area without an explicit user choice. LLM Calls
can likewise occupy the secondary area after its source is selected.

The new `p`-key Agent view picker already owns the explicit controls needed for this
behavior: `[` selects metadata-only, `1` / `=` / `2` select visible split layouts, `]`
selects secondary-only, and `f` / `t` select the File or LLM Calls secondary source.
This change should use that existing model rather than add another keymap, configuration
setting, persisted preference, or backend behavior.

## Implementation

1. In `src/sase/ace/tui/widgets/agent_detail.py`, initialize a fresh `AgentDetail` with
   `DetailLayoutMode.METADATA_ONLY` instead of `SECONDARY_LARGER`. Make the widget's
   composed first-frame classes agree with that state: metadata starts expanded and the
   File and LLM Calls scrolls start hidden, so startup cannot briefly paint the old
   split while asynchronous detail content is resolving.
2. Keep `DetailPanelMode.AUTO` as the initial secondary source and preserve the existing
   session-local interaction semantics in
   `src/sase/ace/tui/widgets/_agent_detail_panels.py` and
   `src/sase/ace/tui/actions/agents/_agent_view_picker.py`:
   - file and LLM Calls availability may continue loading in the background so picker
     choices become enabled without blocking the Textual event loop;
   - navigating among agents retains a layout that the user explicitly selected during
     the session;
   - choosing `f` or `t` changes the selected secondary source without silently changing
     the saved layout;
   - choosing `1`, `=`, `2`, or `]` through the `p` picker reveals the selected
     secondary content, while `[` returns to metadata-only. Do not introduce a second
     refresh path, synchronous I/O, a durable preference, or a default-config/keymap
     change.
3. Update the Agents detail-view documentation in `docs/ace.md` to say that
   metadata-only is the fresh-session default and that File/LLM Calls remain hidden
   until the user selects a visible layout in the `p` picker. Remove the existing claim
   that the default is the secondary-larger split, while retaining the documented
   split-cycle order and the independence of source and layout. Keep
   `docs/configuration.md` unchanged unless wording is needed to make the existing
   `choose_agent_view` action accurately describe the opt-in path; the `p` binding
   itself does not change.

## Regression coverage

1. Extend `tests/ace/tui/test_agent_view_picker.py` with a mounted-app regression using
   an agent that has real file content. After content availability settles, assert that
   a fresh tab still reports `METADATA_ONLY`, shows metadata, and hides both secondary
   scrolls. Then drive only the public picker route to select a visible split and assert
   that the File panel appears; select LLM Calls through the picker with seeded
   availability and assert that the same explicit layout can display it. This proves
   both halves of the contract: hidden by default, available by deliberate `p`-picker
   action.
2. Adjust existing picker tests that currently wait for the File panel to be visible at
   startup. Wait for `_has_file_content` (availability) instead, then choose or seed the
   layout required by that test before asserting cycle, fullscreen, mode-switch, or
   persistence behavior. Retain coverage that bare brackets outside the picker are inert
   and that an explicit layout survives row navigation and File/LLM Calls source
   changes.
3. Update `tests/ace/tui/visual/test_ace_png_snapshots_agents_panel_layout.py` so layout
   scenarios wait for hidden File content to become available before opening the picker.
   Regenerate only the intentionally changed PNG goldens under
   `tests/ace/tui/visual/snapshots/png/`, inspect the actual/expected/diff artifacts,
   and confirm that the default/picker frame visibly identifies metadata-only as
   current. Leave snapshots unrelated to the changed default untouched.

## Verification

1. Run the focused mounted behavior suite:
   `pytest -q tests/ace/tui/test_agent_view_picker.py`.
2. Run the focused Agents panel-layout visual tests with `just test-visual` and inspect
   the rendered PNG diffs before accepting intentional golden updates. Re-run the same
   focused visual tests without the update flag to prove exact convergence.
3. Use `sase screenshot` for a live Agents-tab smoke check when the local fixture has an
   agent with secondary content: verify the initial frame is metadata-only, then drive
   `p` and a visible layout choice and inspect that the File/LLM Calls pane appears.
4. Run `just fix`, followed by the repository-default `just check`. Escalate to
   `just check-full` only if the scoped selector broadens or reports unusual coverage,
   using the required SASE monitor workflow for that long command.

## Acceptance criteria

- On every fresh TUI session, an ordinary selected agent gets the full metadata detail
  area even when File or LLM Calls content exists.
- No asynchronous File/LLM Calls availability event can override the metadata-only
  default or cause a transient/default split.
- The existing `p` Agent view picker is the only route needed to opt into a visible File
  or LLM Calls layout, and explicit layout/source choices continue to persist for the
  remainder of the session and across agent navigation.
- Keybindings, default configuration, backend/domain logic, and durable user settings
  are unchanged.
