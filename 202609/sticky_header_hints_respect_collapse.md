---
tier: tale
title: Keep the Agents-tab sticky header collapsed in hint mode
goal: Pressing `v` on the Agents tab never expands the sticky header; header content
  is hinted only when the header was already expanded, and a collapsed header stays
  visually unchanged with no hidden hint numbers.
size: medium
proposed_by: bbugyi200.athena.0uh
status: done
---

# Plan: Keep the Agents-tab sticky header's expand state unchanged by `v` hint mode

## Problem

On the Agents tab, pressing `v` (`view_files`) re-renders the selected node's document
with numbered `[N]` hint markers. The sticky identity header above the deck panels
(`AgentHeaderPanel`, `src/sase/ace/tui/widgets/agent_header_panel.py`) is fed from that
document through the prompt panel's detached-identity sink. Today:

- Hint mode puts markers into header content: the identity's `Timestamps:` line
  (`append_timestamp_fields` in `_agent_display_header_metadata_sections.py`, reached
  through `append_agent_metadata_fields`) and the detached `AGENT XPROMPT`
  (`_agent_display_hint_body.py` and, for session containers,
  `_agent_display_agent_session_render.py`).
- Those builders set `IdentityHeader.has_hints`, and `AgentHeaderPanel.show_identity`
  renders the expanded layout whenever `self._expanded or header.has_hints` (`on_resize`
  and `is_header_scrollable` repeat the same override).
- On the regular agent path, `build_header_text` (`_agent_display_header.py`) computes
  `has_hints` only after the _whole_ header body is built, so a hint anywhere in the
  header portion (output variables, beads, plan, …) also forces expansion. In practice
  nearly every `v` pops the header open.

The expanded header grows, `AgentDetail._on_identity_header` sees a row-count change and
re-applies deck pins, and whatever the user was reading in the deck below — maybe the
very text they wanted to target with a hint — is pushed off screen.

## Desired behavior

- `v` (and every other flow that renders through `update_display_with_hints`) never
  changes the header's expanded/collapsed state. Only the `toggle_agent_header` key
  (`d`) does.
- Header **expanded** before `v`: unchanged from today — header fields and the detached
  `AGENT XPROMPT` carry hint markers, numbered first (`[1]…`), and remain selectable.
- Header **collapsed** before `v`: the header stays collapsed and visually identical to
  its non-hint paint (same chip rows, same un-annotated xprompt preview card, same
  subtitle and row count, so no deck pin re-apply). No hint numbers are allocated for
  header content at all, so there are no invisible-but-selectable targets and no
  numbering gaps: the first body hint is `[1]`.
- Documents whose identity is _not_ detached (prompt panels without an identity sink,
  e.g. zoom/modal panels that inline the identity) keep hinting their inline identity
  exactly as today.
- Non-goal: re-rendering hints if the header is toggled while hint mode is already
  active (the hint bar holds focus, so `d` normally types into it). Existing hint
  mappings stay as they were rendered.

This is presentation-only Textual/Rich state in this repo; no `sase_core` change is
needed.

## Implementation

### 1. Let the prompt panel ask whether the header is expanded

In `src/sase/ace/tui/widgets/prompt_panel/__init__.py` (`AgentPromptPanel`):

- Extend
  `attach_identity_header_sink(sink, *, detach_xprompt=False, header_expanded: Callable[[], bool] | None = None)`
  and store the probe on a new class attribute `_identity_header_expanded_probe`
  (cleared when `sink` is `None`).
- Add a property `identity_header_hints_enabled -> bool`: `True` when
  `detaches_identity_header` is false or no probe is attached; otherwise
  `bool(probe())`, treating a raising probe as `False` (collapsed).

In `src/sase/ace/tui/widgets/agent_detail.py`, `on_mount` passes
`header_expanded=self._header_expanded`, a small new method that returns
`bool(panel.is_expanded)` for `_header_panel_or_none()` and `False` when the panel is
missing. Pull rather than push, so `AgentHeaderPanel._expanded` stays the only owner of
the state.

### 2. Skip hint allocation for header content when the header is collapsed

- `build_header_text` (`_agent_display_header.py`) gains a keyword
  `identity_hints: bool = True`. In the `detach_identity` branch only, pass
  `hint_state=hint_state if identity_hints else None` to `append_agent_metadata_fields`
  for `identity_text`, so the `Timestamps:` line renders through its existing plain
  branch. The non-detached branch and all body sections keep receiving `hint_state`.
- `_agent_display_hint_render.py`, agent path (the `build_header_text(...)` call that
  passes `hint_state=header_hint_state`): add
  `identity_hints=bool(getattr(self, "identity_header_hints_enabled", True))`. The clan
  call site needs nothing — clan identity fields never take hints.
- `_agent_display_xprompt.py`: add a helper `xprompt_hints_enabled(panel) -> bool`
  returning `not detaches_xprompt or identity_header_hints_enabled` (both read with
  `getattr` defaults `False` / `True`), and export it.
- `_agent_display_hint_body.py` (`render_agent_prompt_hint_body`): when
  `xprompt_hints_enabled(panel)` is false, build the xprompt exactly like the regular
  display path (`_agent_display_render.py`: `panel._display_raw_xprompt(agent, raw)`
  then `panel._render_xprompt(agent, raw, humanized, context=highlight_context)`),
  attach it with `attach_xprompt_to_identity`, and leave `hint_counter` untouched.
  Otherwise keep the existing annotated path. Using the regular renderer (and its
  highlight cache) is what makes the collapsed preview byte-identical to the non-hint
  paint, so `AgentHeaderPanel.show_identity`'s digest short-circuits and no repaint or
  pin re-apply happens.
- `_agent_display_agent_session_render.py` (`_update_agent_session_display`): treat the
  xprompt as non-hint (`_render_xprompt` branch, no `_agent_session_text_with_hints`)
  when `hint_state is None or not xprompt_hints_enabled(self)`. Prompt/reply hinting
  below is unchanged.

Body hints continue from whatever counter the header left, which is `1` when header
hints were skipped.

### 3. Key the hint render cache on the new input

In `_agent_display_hint_cache.py`, add `identity_header_hints: bool = True` to
`AgentHintRenderCacheKey` and fill it in `agent_hint_render_cache_key` from
`bool(getattr(widget, "identity_header_hints_enabled", True))`. Otherwise a document
annotated while the header was expanded would be restored from cache after the user
collapses the header and presses `v` again (and vice versa). `hint_document_is_current`
compares this key, so the refresh path in `actions/agents/_display_detail_render.py`
follows automatically.

### 4. Remove the forced expansion and the now-redundant `has_hints` flag

- `AgentHeaderPanel`: `show_identity` uses `shown_expanded = self._expanded`;
  `on_resize` early-returns on `identity is None or self._expanded`;
  `is_header_scrollable` checks only `self._expanded`.
- Delete `IdentityHeader.has_hints` (`prompt_panel/_identity_header.py`) and the
  `has_hints` keyword of `IdentityHeader.with_xprompt` and `attach_xprompt_to_identity`.
  Drop every producer and the bookkeeping that only fed it: `_agent_display_header.py`
  (`has_hints` block and `hint_counter_before`), `_agent_display_clan.py` (`has_hints`,
  `hint_counter_before`, `hint_counter_after_identity`), `_agent_display_hint_body.py`
  (`xprompt_hints_before`), `_agent_display_agent_session_render.py`
  (`xprompt_hints_before`), `_workflow_render.py` and `_agent_display_tribe.py`
  (`has_hints=False`). Afterwards `rg -n has_hints src tests` must return nothing.

### 5. Docs

`docs/ace.md`, Header panel bullet in the Agents-tab main deck section: replace "While
file-hint markers (`[N]`) are visible the panel renders expanded so every hint stays
selectable." with text saying hint mode never changes the panel's state — expanded, its
fields and `AGENT XPROMPT` carry hint markers numbered first; collapsed, it keeps its
normal preview, header content gets no markers, and numbering starts in the deck body.
No keymap or `default_config.yml` change.

## Tests

- `tests/ace/tui/widgets/test_agent_header_panel_basic.py`: replace
  `test_hint_document_forces_expansion` with DetailApp tests that drive
  `detail.update_display_with_hints(agent)` on an agent whose xprompt names an existing
  workspace file (see `artifact_agent` / `show_agent_full` in
  `_agent_header_panel_shared.py`):
  - collapsed: after the hint render, `panel.is_expanded` is false, the header text and
    `rendered_row_count` equal their pre-`v` values, the header text contains no `[1]`,
    and the xprompt file is absent from the returned `file_hints` (any body hint starts
    at `1`);
  - expanded (`detail.toggle_header_expanded()` first): the header shows the xprompt
    with `[1]`, and `file_hints[1]` is that file.
- `tests/ace/tui/widgets/test_agent_header_panel_scroll.py`: rework
  `test_hint_expanded_header_claims_scroll` so the header is expanded with
  `detail.toggle_header_expanded()` (no `has_hints` replace), and assert a collapsed
  header stays non-scrollable after a hint render.
- `tests/ace/tui/widgets/test_identity_header_xprompt.py`: drop the `has_hints`
  assertion from `test_hinted_xprompt_moves_to_identity_and_keeps_its_markers`; add a
  `_DetachedPanel` variant with `identity_header_hints_enabled = False` asserting the
  identity xprompt plain text has no marker, `file_hints` lacks the xprompt path, and a
  prompt-body path is numbered `[1]`.
- `tests/ace/tui/widgets/test_agent_display_agent_session_hints.py`: the same
  collapsed-header assertion for a session container's detached xprompt.
- `build_header_text(..., hint_state=state, detach_identity=True, identity_hints=False)`:
  identity text has no `[N]` and the identity leaves `state.hint_counter` unchanged (use
  a fixture whose Timestamps line is hintable; if none exists, assert the
  counter/mapping invariants only).
- `tests/ace/tui/widgets/test_identity_header.py`: drop the `has_hints` assertion in
  `test_clan_hint_mode_numbers_body_from_one_without_header_hints` (keep the rest).
- `tests/ace/tui/widgets/test_agent_display_hint_cache.py`: flipping
  `identity_header_hints_enabled` yields a different cache key / a cache miss, and
  `hint_document_is_current` becomes false.
- `AgentPromptPanel.identity_header_hints_enabled`: true without a sink, true with a
  sink but no probe, follows the probe, false when the probe raises.

## Verification

- Read the `lint_and_test` memory note and run the verification it prescribes
  (`just check` through `sase tool run`).
- Clan hint-mode PNG goldens (`test_ace_png_snapshots_agents_clan_panel.py`) press `v`,
  but clan identities never carried hints, so they should stay unchanged; confirm with a
  targeted `just fix-tui-screenshots -- <that file>` run and read its report.
- Optional live check: `sase screenshot` on the Agents tab with a collapsed header,
  before and after pressing `v`, showing the header and deck unchanged except for body
  markers.
