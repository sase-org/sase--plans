---
tier: tale
title: "`l` jumps to a session's or clan's lone running monitor"
goal:
  On the Agents tab, `l` on a session or clan row whose only running node is a monitor
  selects that monitor.
size: medium
decisions:
  clan_drill:
    ask:
      Should `l` on a clan whose only running node is a session's monitor jump straight
      to that monitor?
    choices:
      monitor:
        One `l` from the clan lands on the monitor; `h` walks back via the session
      session: First `l` selects the session row; a second `l` reaches the monitor
    default: monitor
    why: Matches 'only node running in an agent clan' literally; Ctrl+O still jumps back
    answer: monitor
proposed_by: bbugyi200.athena.1fp
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-24 07:23:14
status: wip
---

# `l` jumps to a session's or clan's lone running monitor

## Problem

In the screencast, the Agents tab cursor sits on the collapsed agent-session row
`research.65.linker.w0.r0` (project `love`, status `WAITING SPOTIFY`, `⚙1`). Its only
live work is the monitor turn `research.65.linker.w0.r0--mon`. Pressing `l` only expands
the session: the cursor stays on the session row, and the user has to press `jj` to
reach the monitor.

## Root cause

`l` → `action_expand_or_layout` → `_expand_fold`
(`src/sase/ace/tui/actions/agents/_folding_agent_groups.py`). The "jump to the lone
running member" rule from commit `b8145cb51a` only runs inside
`if agent.is_clan_container:`. It uses `_resolve_clan_running_member_jump_target`
(`src/sase/ace/tui/actions/agents/_folding_clans.py`), which calls
`clan_lone_running_member` (`src/sase/ace/tui/models/_agent_clan.py`). The selected row
is not a clan. It is a sequential agent session (the live data shows `agent_clan: null`,
`is_sequential_agent_session_container == True`), so `_expand_fold` falls through to
`_fold_manager.expand(wf_key)`, which only moves the fold one step down (COLLAPSED →
EXPANDED → FULLY_EXPANDED).

The model already exposes what we need. Run against the live session,
`agent_session_running_monitor_row(session)` in
`src/sase/ace/tui/models/agent_session_members.py` returns the `--mon` row. Among the
session's `concrete_agent_session_turn_rows` (`love` container proxy, `--mon`), only the
monitor is in flight per `agent_row_is_in_flight`. The container proxy has a `stop_time`
and is not in flight.

Clans are already close. A session whose current turn is a running monitor mirrors the
monitor's status onto its container row with a `Running` bucket
(`_mirror_root_from_child` in `src/sase/ace/tui/models/_agent_status_apply.py`). So
`clan_lone_running_member` already returns that session when it is the clan's only
running member. Today, `l` on the clan therefore lands on the session row, never on the
monitor.

## Behavior after the fix

- **Session row.** Pressing `l` on a selected sequential-session container row jumps to
  the session's monitor turn, expanding the session first if needed. This applies only
  when that monitor is running and is the session's only in-flight turn. The jump works
  like the clan jump: `_try_reveal_agent_row`, one jump-anchor push (Ctrl+O returns), no
  toast. This holds at every fold level of the session, so an EXPANDED session no longer
  steps to FULLY_EXPANDED in this state. That step is still available by pressing `l` on
  the monitor row, which already advances its governing session fold.
- **Otherwise unchanged.** If the session has no running monitor, or more than one turn
  in flight (for example the starting agent turn is still in flight next to a freshly
  started monitor), `l` keeps today's fold ladder exactly.
- **Agent and gate turns.** These are deliberately not jump targets. For a running agent
  turn, the session row already is the natural view, and taking over `l` there would
  remove the session's FULLY_EXPANDED step during ordinary runs. The user asked only for
  monitors.
- **Clan row** (`clan_drill`). Pressing `l` on a clan whose `clan_lone_running_member`
  is a session that passes the session rule above:

  > [!decision] clan_drill = monitor The jump target is the session's monitor row. One
  > jump (one anchor push) reveals the clan and session folds on the monitor's path.

  > [!decision] clan_drill = session The clan jump keeps returning the session row. The
  > new session rule then makes a second `l` reach the monitor.

  All other clan cases (zero or several running members, or a lone running member that
  is not a monitor-only session) behave exactly as today.

- **Footer hint.** The footer chip says `l running monitor` when the resolved target is
  a monitor row, and keeps `l running member` for other clan-member targets. The chip
  and the key must keep sharing one resolver so they never disagree.
- **Unchanged guards.** Whole-panel focus, grouping-banner focus, Archive routing, and
  the Tools-card (LLM Calls / Runs) routing keep their current precedence and stay ahead
  of the jump.

## Implementation

1. **Model helper** (`src/sase/ace/tui/models/agent_session_members.py`, add to
   `__all__`): `agent_session_lone_running_monitor_row(agent: Agent) -> Agent | None`.
   - Return `None` for clan containers and for anything that is not
     `is_sequential_agent_session_container`.
   - Collect `concrete_agent_session_turn_rows(agent)`, deduped by `row.identity`, that
     satisfy `agent_row_is_in_flight`.
   - Return the row only when exactly one is in flight and it `is_monitor`. A held
     (not-adopted) monitor is not in flight because it requires
     `monitor_state == "running"`, so it never qualifies.
   - Keep it pure and in-memory, bounded to the session subtree, with no projection-wide
     scan. The footer evaluates it on every selection change (see the `tui_perf` memory
     rules).
2. **Unified resolver** (`src/sase/ace/tui/actions/agents/_folding_clans.py`, or a small
   new `_folding_running_member.py` mixin placed in the same MRO if that reads cleaner).
   Rename `_resolve_clan_running_member_jump_target` to
   `_resolve_running_member_jump_target` and `_jump_to_clan_running_member` to
   `_jump_to_running_member`, then update every reference, including the three test
   stubs in `tests/ace/tui/test_agent_display_defer_detail.py`. Keep all existing guards
   (Agents tab only, no panel focus, no `_current_group_key`, a selected row exists).
   Then:
   - Clan container: `member = clan_lone_running_member(selected)`. Under
     `clan_drill = monitor`, return
     `agent_session_lone_running_monitor_row(member) or member` when `member` is not
     `None`. Under `clan_drill = session`, return `member`.
   - Any other selected row: return `agent_session_lone_running_monitor_row(selected)`.
   - Keep the existing `try/except` → `None` defensive shape.
3. **`_expand_fold`** (`_folding_agent_groups.py`). After resolving `agent` and
   `wf_key`, try the unified resolver for every selected agent row, not only clan
   containers. If it returns a target and `_jump_to_running_member(target)` succeeds,
   return. Keep the clan-only "already expanded with no jump target → no-op" branch, and
   let non-clan rows fall through to `_fold_manager.expand(wf_key)` exactly as today.
   Update the docstring.
4. **Footer** (`src/sase/ace/tui/actions/agents/_display_detail_footer.py`). Call the
   renamed resolver once. Keep `running_member_jump_available`, and add a sibling
   keyword (for example `running_member_jump_label: str = "running member"`, set to
   `"running monitor"` when `target.is_monitor`). Thread it through both signatures and
   the forwarding call in `src/sase/ace/tui/widgets/_keybinding_modes.py`, and through
   `src/sase/ace/tui/widgets/_keybinding_bindings_agents.py`, where the chip text comes
   from the label.
5. **Help and config text.**
   - In `src/sase/ace/tui/modals/help_modal/agents_mode_sections.py`, change the line
     `"Clan: jump to its lone running member"` to cover both cases, for example
     `"Clan/session: jump to lone running member/monitor"`. Update
     `tests/test_keymaps_display_help_agents.py` if it asserts the text.
   - Update the `expand_or_layout` comment block in `src/sase/default_config.yml`
     (around the `l expands or enters a selected panel` sentence) to mention the session
     monitor jump.
6. **Docs.** Read the `docs_writing` memory first. These are behavior updates to
   existing Reference pages, so keep each to a sentence or two:
   - `docs/reference/tui/agent-row-glyphs.md`: the paragraph saying "A single `l` on the
     session container row reveals every member and monitor…" should add that when the
     session's only in-flight turn is a running monitor, `l` selects that monitor
     (expanding first if needed).
   - `docs/reference/agents/agent-clans.md` (the "When exactly one direct member is
     running (`R1`)…" sentence) and `docs/reference/tui/agents-tab-grouping.md` (the
     "unless the clan has exactly one running member…" sentence): describe the clan →
     monitor drill-through under `clan_drill = monitor`, or the two-press path under
     `session`.
   - `docs/reference/tui/agents-tab-panels.md` ("Lowercase `l` only advances a real
     fold…"): add a short pointer to the running-member/monitor jump exception if the
     sentence would otherwise be false.
   - Run `just fix` so any generated keybinding reference picks up the help-text change.

## Tests

- **Model** (next to the existing `agent_session_members` tests; reuse
  `make_monitor_member` from `tests/ace/tui/models/_agent_session_members_helpers.py`).
  `agent_session_lone_running_monitor_row`:
  - returns the monitor for a session whose first turn is DONE (has `stop_time`) and
    whose `--mon` member has `monitor_state="running"` and no `stop_time`;
  - returns `None` when the current turn is a running agent turn, when the monitor has
    settled (terminal state or `stop_time`), when the monitor is held, when an agent
    turn is still in flight beside a running monitor, and for clan containers and
    non-session rows.
- **Action.** Add `tests/ace/tui/test_agent_session_monitor_l_jump.py`, or extend
  `tests/ace/tui/test_agent_clan_running_member_l_jump.py`, using the production-shaped
  `ClanJumpHarness` / `ClanLJumpHarness` pattern:
  - a collapsed session with a lone running monitor: `action_expand_or_layout` selects
    the monitor, the session fold is no longer COLLAPSED, the anchor stack grows by
    exactly 1, and there are no notifications;
  - an expanded session: selects the monitor with fold states unchanged (no
    FULLY_EXPANDED step);
  - a session running an agent turn: the old ladder (COLLAPSED → EXPANDED →
    FULLY_EXPANDED) holds and the selection stays on the session;
  - a clan whose only running member is a monitor-only session: lands on the monitor
    (`clan_drill = monitor`) or on the session row (`session`);
  - all existing clan tests still pass unchanged.
- **Footer.**
  - Update the `test_agent_display_defer_detail.py` stubs to the renamed resolver, and
    assert that the label is `running monitor` for a monitor target and `running member`
    otherwise.
  - Extend `tests/ace/tui/test_keybinding_footer_llm_calls_detail.py` (or the nearest
    footer-binding test) so the chip renders the label.
- **Visual snapshots.** If an existing PNG snapshot selects a session row with a running
  monitor, the new footer chip changes it. Find such scenarios under `tests/visual/` and
  refresh only those goldens with the targeted screenshot recipe from the `tui` /
  `tui_screenshot` memory. Do not run `just check-full`.

## Verification

- Run `just check`, which must pass.
- Optional manual check: in `sase tui`, select a collapsed session whose only live work
  is a running monitor. The footer shows `l running monitor`, one `l` selects the
  `--mon` row (whose footer offers `x stop monitor`), `h` returns to the session, and
  Ctrl+O returns to the original row.

## Out of scope

- Jumping to lone running agent or gate turns of a session.
- Any change to how clans count running members, status mirroring, or Rust core. This is
  TUI navigation only; no `sase-core` change is needed.
