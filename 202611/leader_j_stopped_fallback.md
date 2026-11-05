---
tier: tale
title: Agents-tab ,j falls back to the ,J stopped-agent jump before toasting
goal:
  On the Agents tab, ,j jumps to the next unread completed agent. If there is none, it
  runs the ,J jump to the next stopped agent, and it shows an error toast only when both
  miss.
size: small
proposed_by: bbugyi200.athena.18p
proposed_with: claude/opus
create_time: 2026-11-05 13:49:55
status: wip
---

# Plan: `,j` Falls Back to `,J` Before Toasting

## Goal

On the Agents tab, `,j` (leader `jump_to_next_unread_done_agent`) currently toasts
`No unread completed agents` whenever the unread jump misses. Change it so a miss first
runs the `,J` behavior (`jump_to_next_stopped_agent`), and only when that also misses
show a single error toast. `,J` itself is unchanged.

The palette entry `leader.jump_to_next_unread_done_agent` uses executor
`leader_mode_key` with subkey `j`, so it goes through the same leader handler and needs
no separate dispatch change. `,,` (repeat last) remembers the raw `j`, so a repeat also
gets the fallback.

## Current Behavior (for reference)

- Dispatch: `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`, in the
  `jump_to_next_unread_done_agent` branch (around line 271). It calls
  `self._jump_to_next_unread_done_agent()` inside
  `LeaderModeMixin._leader_unread_span(self, ",j")` and calls
  `self.notify("No unread completed agents")` on a miss. The `,J` branch right below
  does the same with `_jump_to_next_stopped_agent()` and `No stopped agents`.
- Jump implementations: `src/sase/ace/tui/actions/agents/_unread_navigation.py`.
  `_jump_to_next_stopped_agent()` already handles focus, clan/panel reveal, off-tab
  candidates, jump-anchor saving, and `land_stopped_jump_on_main_context`. The fallback
  must call it as-is, not reimplement it.
- Palette availability: `src/sase/ace/tui/commands/_availability_agents.py` gates
  `leader.jump_to_next_unread_done_agent` on `ctx.unread_completed_agent_count > 0`.

## Implementation

### 1. Leader dispatch fallback (`_leader_mode.py`)

Rewrite the `jump_to_next_unread_done_agent` branch so it looks like this (keep
`_remember_leader_key`, `_begin_leader_perf(self, ",j")`, and
`_finish_leader_perf(self)` as they are):

```python
if self.current_tab == "agents":
    with LeaderModeMixin._leader_unread_span(self, ",j") as trace:
        hit = self._jump_to_next_unread_done_agent()
        fell_back = not hit
        if fell_back:
            # ,j falls back to ,J so it always lands somewhere useful;
            # only a double miss toasts.
            hit = self._jump_to_next_stopped_agent()
        trace["fallback"] = fell_back
    if not hit:
        self.notify("No unread completed or stopped agents")
```

Notes:

- `_leader_unread_span` returns `tui_trace(...)`, which yields a mutable `extra` dict
  (an empty, write-discarding dict when tracing is off). Writing `fallback` keeps
  exactly one `leader.unread_jump` span per `,j` press, so
  `tests/ace/tui/test_unread_trace_spans.py::_one_span` still holds, and a perf reader
  can tell a slower fallback `,j` from a plain unread jump.
- Keep the fast-path contract: no trailing `_refresh_current_tab()` on a hit, a fallback
  hit, or a double miss. Update the existing branch comment to say the stopped fallback
  paints its own highlight too.
- Fall back on any unread miss, including the rare case where unread IDs exist but the
  unread jump cannot reveal them (for example, a failed off-tab reveal). Both jump
  helpers already roll back their own tab and anchor state when they fail.
- The fallback is silent: no informational toast when it succeeds.
- The `,J` branch and its `No stopped agents` toast stay unchanged. The `,u` branch's
  `No unread completed agents` toast stays unchanged.
- Do not rename the config key `jump_to_next_unread_done_agent`. Users' keymap overrides
  depend on it.

### 2. Palette availability (`_availability_agents.py`)

Change the `leader.jump_to_next_unread_done_agent` rule to:

```python
return ctx.unread_completed_agent_count > 0 or ctx.stopped_agent_count > 0
```

Tab scoping (`AGENTS_ONLY` in `_mode_commands.py`) is unchanged.

### 3. Labels

- `src/sase/ace/tui/commands/_mode_commands.py` `_LEADER_LABELS`:
  `"jump_to_next_unread_done_agent": "Jump to next unread completed agent (else next stopped)"`.
- `src/sase/ace/tui/modals/help_modal/agents_mode_sections.py`: change the `,j` help row
  text to `"Jump to next unread done agent (else next stopped)"`.
- Leave the footer hint in `src/sase/ace/tui/widgets/_keybinding_modes.py` unchanged
  (`j next unread done`, shown only when unread completed agents exist). When there are
  no unread agents, the footer already shows `J next stopped` for that destination, and
  a second identical hint would only add noise.
- `src/sase/default_config.yml`: add a short trailing comment on the
  `jump_to_next_unread_done_agent: "j"` line under `ace.keymaps.modes.leader_mode.keys`,
  for example `# falls back to jump_to_next_stopped_agent before toasting`. Key values
  do not change.

The `_LEADER_LABELS` change feeds the checked-in Footprint catalog
(`src/sase/footprint/catalog.jsonl`, entry `tui:leader.jump_to_next_unread_done_agent`).
Regenerate it with `just fix`, which runs `sync-footprint-catalog`, `docs-gen`, and the
formatters. Do not hand-edit `catalog.jsonl` or the generated
`docs/reference/tui/keymaps.md`. Its `,j` row label comes from the config key name and
should not change.

### 4. Reference docs

Follow the docs routing rule: behavior changes go to the feature's Reference page, and
generated pages are regenerated rather than edited.

- `docs/reference/tui/agents-tab-modes.md`, Leader Mode section (the paragraph ending
  "`,j` can reveal a direct member hidden by a collapsed clan."): add one sentence
  saying that when no unread completed agent can be reached, `,j` falls back to the `,J`
  stopped-agent jump, and toasts only when neither exists.
- `docs/contribute/perf-runbook.md`, in the unread-span capture section (near the `,u` /
  `,j` / `,J` jq example): add one sentence saying a `,j` that falls back records
  `fallback: true` on its single `leader.unread_jump` span, and the span duration then
  includes the stopped jump.
- Leave `docs/reference/tui/agent-statuses.md` and `docs/reference/tui/agents-tab.md`
  alone. Their `,j` statements (acknowledging the unread target, cross-tab reach) stay
  true.

No memory note becomes false. The glossary strand `agent-relation-jump-target` mentions
"`,j`/`,J` unread and stopped-agent jumps" only to disambiguate, and that stays
accurate.

## Tests

Update tests that encode the old miss-toasts-immediately behavior, and add fallback
coverage:

1. `tests/ace/tui/test_leader_keymap_dispatch.py` (uses `_FakeApp` from
   `tests/ace/tui/_leader_keymap_helpers.py`, which stubs both jumps with
   `jump_unread_result` / `jump_stopped_result` and counters):
   - Rename and update `test_leader_j_notifies_when_no_unread_done_agent`: set both
     results to `False` and assert `jump_unread_count == 1`, `jump_stopped_count == 1`,
     `notifications == ["No unread completed or stopped agents"]`, and
     `refresh_count == 0`.
   - Add `test_leader_j_falls_back_to_next_stopped_agent`: unread `False`, stopped
     `True`. Assert both counters are 1, there are no notifications, and
     `refresh_count == 0`.
   - Add (or extend an existing hit test to cover) the unread-hit case. Assert
     `jump_stopped_count == 0` so the fallback never runs after an unread hit.
   - `test_leader_j_noops_on_non_agents_tabs`: also assert `jump_stopped_count == 0`.
2. `tests/ace/tui/test_agent_unread_done_navigation.py`,
   `test_repeated_leader_j_walks_unread_done_agents_by_recency`: the final `,,` after
   the unread set empties now falls back to the stopped jump, because the DONE, FAILED,
   and PLAN DONE rows are stopped. Update the tail assertions to the real fallback
   result: focus moves per `,J` recency order, no toast, and the anchor stack, refresh
   calls, and ack and notification counters match what the run actually does. Keep
   asserting that `_last_leader_key == "j"`, that the unread set is empty, and that the
   fallback jump does not acknowledge or patch anything new. Derive the exact values by
   running the test; do not guess them.
3. `tests/ace/tui/test_agent_unread_jump_fast_path.py`,
   `test_leader_j_hit_and_miss_skip_trailing_tab_refresh`: the second `,,` now hits via
   fallback on the already-read DONE row. Assert it does not toast and still makes no
   trailing tab refresh. Add a true double-miss case with an app whose loaded agents are
   all non-stopped (for example, only `RUNNING`), and assert the
   `No unread completed or stopped agents` toast and `current_tab_refresh_calls == 0`.
4. `tests/test_command_availability_agents_navigation.py`: update
   `test_jump_to_next_unread_done_agent_requires_unread_completed_agent` (rename it to
   something like `..._requires_unread_or_stopped_agent`). It should be unavailable when
   both counts are 0, available with only an unread count, available with only a stopped
   count, and still unavailable on non-Agents tabs.
5. `tests/test_command_catalog_build.py::test_jump_to_next_unread_done_agent_leader_command_is_agents_only`:
   update the expected label.
6. `tests/test_keymaps_display_help_panels.py` (around line 228): update the expected
   `(",j", ...)` help pair.
7. `tests/ace/tui/test_unread_trace_spans.py::test_leader_jump_span`: also assert
   `span["fallback"] is False` for a hit. Add a sibling case with
   `jump_unread_result = False` that asserts one span with `fallback is True` and
   `jump_stopped_count == 1`.

Then run `grep -rn "No unread completed agents" tests/` and make sure every remaining
hit belongs to `,u` (bulk ack), not `,j`.

## Verification

1. `just fix` (formatters, `docs-gen`, `sync-footprint-catalog`). Review the regenerated
   `catalog.jsonl` diff: only the one `tui:leader.jump_to_next_unread_done_agent` label
   line should change, plus any docs-gen change from the config comment.
2. `sase tool run check`. Run `just install-venv` first if the workspace venv is stale.
3. Footer hints are unchanged. The help-panel PNG goldens
   (`tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py`) capture the Patches
   keymaps page and the Agents onboarding guide, not the Agents leader rows, so no
   visual snapshot run is expected. If a golden does drift, run
   `just fix-tui-screenshots -- <selector>` for that test only and inspect the report
   and golden diff before accepting it.

## Out of Scope

- Changing `,J`, `,u`, the footer hint gating, or any default key binding.
- Moving jump logic into `sase-core`. This is TUI-only navigation and keybinding glue,
  which stays in this repo under the Rust core boundary rule.
