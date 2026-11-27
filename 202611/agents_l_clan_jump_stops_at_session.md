---
tier: tale
title: Agents-tab l on a clan stops at the member session
goal:
  Pressing l on an agent clan row selects its lone running (or failed) member session
  without opening it. A second l on that session opens it and selects its lone running
  monitor (or failed turn).
size: small
proposed_by: bbugyi200.athena.1h3
proposed_with: claude/opus
create_time: 2026-11-27 09:49:11
status: wip
---

# Plan: Agents-tab `l` on a clan stops at the member session, never inside it

## Problem

In the Agents tab, the `bob-cli-qj` clan row was selected while collapsed. Its only
running member was the session `bob-cli-qj.land`, and that session's only in-flight turn
was the running monitor `bob-cli-qj.land--mon`. Pressing `l` jumped straight to
`bob-cli-qj.land--mon` and opened both the clan fold and the `bob-cli-qj.land` session
fold. The footer chip read `l running monitor`.

Expected behavior:

1. `l` on the clan row selects the `bob-cli-qj.land` session row. Only the clan fold
   opens; the session stays as it was (collapsed).
2. A second `l`, now on the `bob-cli-qj.land` session row, opens that session and
   selects `bob-cli-qj.land--mon`. That second step already works today.

The rule is general: `l` on an agent clan never opens a member session to land on one of
its children. That covers the failed-member path as well as the running-member path.

## Root cause

The clan branch of `AgentPanelClanFoldingMixin._resolve_member_jump_target` in
`src/sase/ace/tui/actions/agents/_folding_clans.py` drills through the clan member:

- **Running:** after `clan_lone_running_member(selected)` returns the member, it calls
  `agent_session_lone_running_monitor_row(member)`. If that finds a monitor, it returns
  the monitor with label `"running monitor"`.
- **Failed:** after `clan_lone_failed_member(selected)`, it calls
  `agent_session_failed_turn_row(failed)` and returns the failed turn with label
  `"failed turn"`.

`_expand_fold` in `_folding_agent_groups.py` and the footer chip in
`_display_detail_footer.py` both consume this resolver. Fixing the resolver fixes the
key and the chip together.

The non-clan branch, for a selected session row, already does the second step correctly.
`agent_session_lone_running_monitor_row(selected)` or
`agent_session_failed_turn_row(selected)` returns the child turn, and
`_jump_to_member_row` reveals it, opening the session fold.

A read-only probe of the existing test harnesses confirmed the target behavior. Calling
`_jump_to_member_row(<session row>)` from a collapsed clan:

- selects the session;
- leaves the clan fold `EXPANDED` and the session fold `COLLAPSED`;
- pushes one back-anchor.

A following `action_expand_or_layout()` then selects the monitor, moves the session fold
out of `COLLAPSED`, and pushes a second anchor. The failed-session fixture behaves the
same way and ends on `alpha--code`. No reveal, fold, or navigation code needs to change.
Only the resolver's clan branch does.

## Changes

### 1. Resolver: the clan branch only targets direct clan members

In `src/sase/ace/tui/actions/agents/_folding_clans.py`, change
`_resolve_member_jump_target` so a selected clan container resolves to the direct member
itself:

```python
if selected.is_clan_container:
    from ...models._agent_clan import (
        clan_lone_failed_member,
        clan_lone_running_member,
    )

    member = clan_lone_running_member(selected)
    if member is not None:
        if member.identity != selected.identity:
            return _MemberJumpTarget(row=member, label="running member")
        return None
    failed = clan_lone_failed_member(selected)
    if failed is None or failed.identity == selected.identity:
        return None
    return _MemberJumpTarget(row=failed, label="failed member")
```

- Leave the non-clan (session row) branch unchanged. It keeps the `"running monitor"`
  and `"failed turn"` targets. `agent_session_lone_running_monitor_row` and
  `agent_session_failed_turn_row` are still used there, so keep the import. You can move
  it below the clan branch if that reads better.
- Keep the outer `try/except Exception: return None` and the early tab, panel-focus, and
  group-banner guards exactly as they are.
- Rewrite the method docstring:
  - A clan row with exactly one running member targets that member, even when the member
    is a session whose only in-flight turn is a monitor.
  - With no running member, a clan with exactly one failed member targets that member.
  - A clan jump never opens a member session. `l` on the session row then reaches the
    monitor or failed turn.

### 2. Docstring and config comment

- `src/sase/ace/tui/actions/agents/_folding_agent_groups.py`, `_expand_fold` docstring:
  - Drop "(or through it to a session's lone running monitor)" and "(or through it to
    the session's failed turn)" from the clan sentence.
  - Keep the sequential-session sentence. On a session row, `l` jumps to its lone
    running monitor or failed turn.
- `src/sase/default_config.yml`, the `# l expands or enters a selected panel; ...`
  comment block above `hooks_or_collapse`, around lines 1031–1038:
  - Remove "drilling through to a session's lone running monitor when that is the clan's
    only live work" and ", or to that session's failed turn".
  - Add that a clan jump stops at the member session row, and a second `l` on that
    session reaches its lone running monitor or failed turn.
  - The binding values do not change.

### 3. Reference docs (behavior of an existing feature → Reference pages only)

- `docs/reference/agents/agent-clans.md`, the paragraph starting "Press `l` once on a
  collapsed clan…":
  - Replace "If that member is a session whose only in-flight turn is a running monitor,
    `l` selects the monitor instead, expanding the clan and session folds on its path."
    with this rule: the clan jump stops at that member even when it is a session, and
    only the clan fold opens. Pressing `l` again on the session selects its lone running
    monitor or failed turn.
  - Change the failed sentence to "`l` selects that member".
- `docs/reference/tui/agents-tab-grouping.md`, the sentence ending "…in which case `l`
  jumps to that member, or through it to the session's running monitor…":
  - Remove both "or through it to the session's running monitor…" and "or its session's
    failed turn".
  - Keep the cross-link to Agent clans.
- `docs/reference/tui/agents-tab-panels.md`, the "The running-member/monitor jump is the
  exception: …" passage:
  - The session-row behavior stays. `l` on a session whose only in-flight turn is a
    running monitor selects that monitor, and `l` on a failed session with nothing
    running selects its failed turn.
  - `l` on a clan with one running member, or with no running member and one failed
    member, selects that member and never opens its session.
- `docs/reference/tui/agent-row-glyphs.md` describes only the session-row `l`, so it
  stays correct. Leave it.
- The help text `"Clan/session: jump to lone running/failed node"` in
  `src/sase/ace/tui/modals/help_modal/agents_mode_sections.py` stays accurate. Leave it.
- Changelog: release-please generates `CHANGELOG.md`, so do not edit it by hand.

### 4. Tests

In `tests/ace/tui/test_agent_session_monitor_l_jump.py`, replace
`test_clan_with_monitor_only_session_jumps_to_monitor` with
`test_clan_with_monitor_only_session_jumps_to_session_then_monitor`. Use the same
fixture:
`_make_session_with_lone_running_monitor(clan="research", extra_members=[done])`.

- **Before any press:** `app._resolve_member_jump_target()` returns
  `label == "running member"`, and its `row.identity` is the session's.
- **First `action_expand_or_layout()`:**
  - The selected row is the session.
  - The clan fold is not `COLLAPSED`.
  - The session fold (`agent_fold_key(session)`) is still `FoldLevel.COLLAPSED`.
  - The monitor is not in `app._agents`.
  - The anchor stack grew by 1.
  - `app.notifications == []`.
  - The resolver now returns `label == "running monitor"`.
- **Second `action_expand_or_layout()`:**
  - The selected row is the monitor.
  - The session fold is not `COLLAPSED`.
  - The anchor stack grew by 2 in total.
  - There are no notifications.

Add `test_clan_jump_with_expanded_monitor_only_session_still_selects_session` to the
same file:

1. Expand the session fold first and call `app._refilter_agents()`.
2. Keep the clan collapsed and the clan row selected.
3. Press `l` once.
4. Assert the session row is selected, not the monitor, even though the monitor becomes
   visible.

In `tests/ace/tui/test_agent_clan_failed_member_l_jump.py`, replace
`test_clan_with_failed_session_drills_to_failed_turn` with
`test_clan_with_failed_session_jumps_to_session_then_failed_turn`:

- **Before any press:** the resolver returns `label == "failed member"` on the session
  row.
- **First `l`:**
  - The `alpha` session row is selected.
  - The clan fold is not `COLLAPSED`.
  - The session fold is `COLLAPSED`.
  - `alpha--code` is not in `app._agents`.
  - The anchor stack grew by 1.
- **Second `l`:**
  - `alpha--code` is selected.
  - The session fold is not `COLLAPSED`.
  - The anchor stack grew by 2 in total.
  - There are no notifications.

The other `l`-jump tests need no edits but must still pass:

- `test_agent_clan_running_member_l_jump.py`, which covers plain members.
- The session-row tests in both files above.
- `test_agent_display_defer_detail.py` and
  `widgets/test_keybinding_footer_llm_calls_detail.py`, which stub the resolver with
  fixed labels.

### 5. PNG goldens

`tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py::test_failed_clan_incident_by_status_collapsed_png_snapshot`
renders the failed clan row selected. The probe confirms that the clan's lone failed
member `research.8` currently resolves through to the failed turn `research.8--1`. Its
footer chip should change from `l failed turn` to `l failed member`.

1. Through `/sase_monitor`, using the `TESTING`/`TESTED` status pair, run:
   `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py tests/ace/tui/visual/test_ace_png_snapshots_agents_agent_sessions.py`.
2. Inspect the report and the golden diff. The only expected change is that chip text in
   `agents_failed_clan_incident_by_status_collapsed_{120x40,80x40}.png`. The session-row
   snapshots should be unchanged.
3. Treat any other drift as a bug to explain before landing.

## Verification

1. Run
   `.venv/bin/python -m pytest -q tests/ace/tui/test_agent_session_monitor_l_jump.py tests/ace/tui/test_agent_clan_failed_member_l_jump.py tests/ace/tui/test_agent_clan_running_member_l_jump.py tests/ace/tui/test_agent_display_defer_detail.py tests/ace/tui/widgets/test_keybinding_footer_llm_calls_detail.py tests/test_keymaps_display_help_agents.py tests/test_agent_clan.py`.
2. Run `just fix`, then `sase tool run check`.
3. Run the targeted PNG refresh from step 5 and inspect it.
4. Optional manual check (`sase ace`, Agents tab):
   1. Select a collapsed clan whose only running member is a session with only a running
      monitor in flight.
   2. The footer chip should read `l running member`.
   3. `l` should land on the session row with the session still collapsed.
   4. The chip should then read `l running monitor`.
   5. A second `l` should open the session and select the `--mon` row.

## Scope notes

- No decision record governs this keymap. The change stays in this repo because it is
  presentation-only Agents-tab navigation (Textual selection and fold state). Shared
  backend behavior belongs in `sase-core` per `decisions:rust-core-required`, and this
  is not shared backend behavior.
- This is a bug fix to an existing binding, not a deprecation or a beta. No feature flag
  is needed.
- No SASE memory note describes the old drill-through, so no memory update is needed.
- Out of scope:
  - numeric `CLAN MEMBERS` jumps;
  - the `l` behavior on session rows;
  - Archive and Tools-card `l` routes;
  - the session-row resolver branches.
