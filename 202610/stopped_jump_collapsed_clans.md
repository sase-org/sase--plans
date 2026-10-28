---
tier: tale
title: Make ,J reach stopped agents inside collapsed agent clans
goal:
  Pressing ,J on the Agents tab expands a collapsed clan and selects its stopped member
  instead of toasting "No stopped agents".
size: small
proposed_by: bbugyi200.athena.15b
proposed_with: claude/opus
create_time: 2026-10-28 14:48:52
status: wip
---

# `,J` reaches stopped agents folded inside collapsed agent clans

## Problem

On the Agents tab, `,J` (`jump_to_next_stopped_agent`) toasts `No stopped agents` when
the only stopped agent sits inside a collapsed agent clan, even though the header reads
`1 stopped`. The reported case is the `@epic` panel's `Stopped` group: a collapsed clan
row `sase (QUESTION) ×8 [S1 W2 D5]`. Its stopped member is a direct clan member hidden
by the clan's collapsed outer fold. `,J` works for stopped rows that are not inside a
collapsed clan, and `,j` (next unread done) already reaches members of collapsed clans.

## Root cause

`,J` has a discovery gap and a reveal guard.

1. **Discovery.** `_jump_to_next_stopped_agent` in
   `src/sase/ace/tui/actions/agents/_unread_navigation.py` calls
   `_jump_to_next_matching_agent_by_time` without `include_collapsed_clan_members=True`.
   In `src/sase/ace/tui/actions/agents/_unread_jump_candidates.py`,
   `_timed_agent_jump_candidates` skips clan container rows (`agent.is_clan_container`).
   It adds members hidden by a collapsed clan (`_collapsed_clan_jump_candidates`) only
   when that flag is set. `_off_tab_jump_candidates` skips rows on the active tab. On
   the active tab, a stopped member of a collapsed clan therefore yields no candidate.
   The leader handler (`src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`,
   `jump_to_next_stopped_agent` branch) then toasts `No stopped agents`. Commit
   `fd31c86915` ("navigate unread agents in collapsed clans") gave `,j` this flag, but
   `,J` never received it.
2. **Reveal guard.** `_reveal_and_select_timed_jump_candidate` (same file as item 1)
   expands the target clan and refilters, then re-validates the target with an
   unread-only check:
   `if target_agent.identity not in unread_ids or not predicate(target_agent): ... return False`.
   With only item 1 fixed, a stopped member (which is not unread) would still be
   rejected after its clan was expanded.

The footer hint and palette availability need no change. `_has_stopped_agent` and
`commands/context.py::_stopped_agent_count` scan `_agents`, which includes the clan
container. `aggregate_agent_group_status` (`src/sase/agent/status_buckets.py`) ranks
QUESTION / WAITING INPUT and pending-plan-review statuses above every other status. So a
clan with any stopped member always shows a stopped status, and the hint already shows
up in this scenario, as the screenshot confirms.

## Changes

### 1. Let `,J` discover clan-hidden stopped members

In `_unread_navigation.py`, `_jump_to_next_stopped_agent` passes
`include_collapsed_clan_members=True` to `_jump_to_next_matching_agent_by_time`. Keep
`after_select=None`, so `,J` still never acknowledges unread state or dismisses
notifications, and keep `time_for_agent=_stopped_agent_jump_time`. Update the docstring:
the jump now covers visible stopped rows, off-tab stopped rows, and direct rows hidden
only by a collapsed clan fold.

### 2. Make the clan reveal predicate-driven

In `_reveal_and_select_timed_jump_candidate`, replace the post-refilter guard with
`if not predicate(target_agent):` and drop the local `unread_ids` lookup. Every
unread-specific condition then lives in the caller's predicate. Update the method
docstring if it implies unread.

`,j` must stay exactly as strict after a refilter. Today its predicate closes over the
`unread_ids` set object captured before the jump. But `_unread_completed_agent_ids` is
reassigned (not only mutated) in `_unread_state.py`, `_loading_finalize.py`, and
`_notification_unread_projection.py`. So in `_jump_to_next_unread_done_agent`, change
`predicate` to read `getattr(self, "_unread_completed_agent_ids", set())` when it runs,
instead of using the captured local. Leave the early `if not unread_ids: return False`
probe and the cached candidate list (`_unread_timed_jump_candidates`) unchanged.

### 3. Skip the clan projection when it cannot produce a target

`_collapsed_clan_jump_candidates` runs `prospective_clan_members`. That call does
`filter_agents_by_fold_state` plus one `build_agent_tree` per panel, all in memory, but
it is O(roster). `,J` has no candidate cache. Without a guard, every `,J` press on a tab
with any collapsed clan would pay for the projection even when no hidden row is stopped.
Add a cheap pre-check before the projection: return `[]` unless some non-container row
in `complete` satisfies `predicate` and its identity is not in `seen`. Both predicates
are O(1) per row: a status-bucket lookup for stopped, a set lookup for unread. `,j`'s
cached path is unaffected apart from skipping a useless projection.

Do not add a stopped-candidate cache. Stopped status changes have no dedicated
generation counter, and a stale cache would serve wrong targets.

### 4. Tests

Move `_CollapsedClanUnreadJumpApp` from
`tests/ace/tui/test_agent_unread_done_navigation_folds.py` into
`tests/ace/tui/_agent_unread_navigation_helpers.py` as a public `CollapsedClanJumpApp`
(behavior unchanged). Import it from the folds test, and keep that file's tests passing
unchanged. Add `tests/ace/tui/test_agent_stopped_navigation_clans.py`
(`test_agent_stopped_navigation.py` is already ~390 lines). Build rosters with
`project_clan_tree` and `make_agent`, following the existing folds tests. The new file
covers:

- **Reveal from a collapsed clan.** Use a RUNNING origin plus a QUESTION member (with
  `questions_times`) of a collapsed clan. `_jump_to_next_stopped_agent()` returns True,
  the clan's fold is `EXPANDED`, and the selected row is the member. The unread set is
  unchanged, `sase.notifications.dismiss_notifications_matching_agents` is not called,
  and the jump anchor stack holds the origin.
- **Only the target clan expands.** Two collapsed clans each hold a stopped member (PLAN
  with `plan_times` versus QUESTION with `questions_times`). Only the clan of the more
  recent member expands.
- **Recency across visible and clan-hidden rows, with wrap.** A visible stopped row is
  older than a stopped member hidden in a clan. The first `,J` reveals the member. The
  next press moves to the visible row. The third press wraps back to the now-visible
  member without another refilter.
- **Collapsed tribe panel.** The clan lives in a collapsed tribe panel (for example
  tribe `epic`). The jump expands both the clan and the panel, and focuses that panel.
- **Stale target.** Use the helper's `stale_identity` so the member disappears on
  refilter. The jump returns False and does not change the selection.
- **No-op projection guard.** With collapsed clans whose hidden members are not stopped,
  `_jump_to_next_stopped_agent()` returns False without calling
  `_prospective_clan_member_panels`. Wrap it in `Mock(wraps=...)` as
  `test_unread_footer_and_jump_share_cached_clan_projection` does.

Add one `,j` regression to the folds test file. The refilter **replaces**
`_unread_completed_agent_ids` with a new set that lacks the target. The jump returns
False, and `sase.notifications.ack_agent_completions` is not called. This guards the
lazy predicate from Change 2.

### 5. Docs

`docs/reference/agents/agent-clans.md` documents navigation into clans. After "Press `l`
once on a collapsed clan to reveal its direct members.", add one sentence in the same
voice. It should say that `,j` (next unread done agent) and `,J` (next stopped agent)
also reach a member row hidden only by a collapsed clan: the jump expands just that clan
and selects the member. Keymap reference rows and `default_config.yml` need no change,
because the key and its label are unchanged.

## Non-goals

- Members hidden by an inner sequential-session or workflow fold under the clan. `,j`
  does not reveal them either (see
  `test_unread_jump_does_not_reveal_member_hidden_by_inner_agent_session_fold`), and the
  shared projection keeps inner folds as they are.
- Members inside a collapsed outer clan nested around a collapsed inner clan. The shared
  projection relaxes one clan fold, the same limit `,j` has.
- Changing which statuses count as stopped, or how `,J` orders targets.
- Moving this logic to `sase-core`. It is presentation-only Textual navigation state,
  which stays in this repo under the Rust boundary rule.

## Verification

- Run targeted tests first: `tests/ace/tui/test_agent_stopped_navigation_clans.py`,
  `tests/ace/tui/test_agent_stopped_navigation.py`,
  `tests/ace/tui/test_agent_unread_done_navigation_folds.py`,
  `tests/ace/tui/test_agent_unread_done_navigation.py`,
  `tests/ace/tui/test_agent_unread_done_navigation_panels.py`,
  `tests/ace/tui/test_agent_unread_jump_fast_path.py`,
  `tests/ace/tui/test_leader_keymap_dispatch.py`, and
  `tests/test_command_availability_agents_navigation.py`.
- Then run the repo's standard check as the `lint_and_test` memory note instructs
  (`just check` through `sase tool run`).
- Optional manual check: in `sase tui`, press `,J` with a stopped member inside a
  collapsed `@epic` clan. Focus should land on the member with the clan expanded, and
  `Ctrl+O` should return to the origin row.
