---
tier: tale
title: "`l` jumps to a clan's lone failed node when nothing in the clan is running"
goal:
  On the Agents tab, `l` on a clan with no running member and exactly one failed member
  selects that failed node, drilling into a failed session's failed turn, with a
  matching footer chip.
size: medium
decisions:
  session_rows:
    ask: Should `l` on a failed session row also jump to that session's failed turn?
    choices:
      jump:
        A failed session's `l` selects its failed turn, like the running-monitor rule
      clan_only: Only clan rows jump; a failed session's `l` keeps today's fold ladder
    default: jump
    why:
      Session rows already take the monitor jump; your last "agent clan" row was a
      session
    answer: jump
proposed_by: bbugyi200.athena.1fp.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-24 09:45:58
status: wip
---

# `l` jumps to a clan's lone failed node when nothing in the clan is running

## Problem

On the Agents tab, `l` on a clan row already jumps to the clan's only running member, or
through that member to a session's lone running monitor (commit `8ea12e5855`). When
nothing in the clan is running but exactly one member has failed, `l` only reveals the
members. The cursor stays on the clan row, and the user has to scan the members (and
then open that member's session) to find what failed.

Requested behavior: when exactly one node in a clan has failed and no other node in the
clan is running, `l` jumps to the failed node.

## Root cause

`l` → `action_expand_or_layout` → `_expand_fold`
(`src/sase/ace/tui/actions/agents/_folding_agent_groups.py`) asks
`_resolve_running_member_jump_target`
(`src/sase/ace/tui/actions/agents/_folding_clans.py`) for a target. On a clan row, that
resolver only consults `clan_lone_running_member`
(`src/sase/ace/tui/models/_agent_clan.py`), and on any other row it only consults
`agent_session_lone_running_monitor_row`
(`src/sase/ace/tui/models/agent_session_members.py`). Nothing considers failed members.
So with zero running members, `_expand_fold` falls through to the clan's binary fold
(expand once, then a no-op). The footer chip uses the same resolver, so it also never
offers a failed-node jump.

The model already has everything needed:

- `clan_member_counts` counts a clan's direct members by `agent_status_bucket`; that
  count is the clan row's `F` chip (`[F1 …]`). A failed bucket is the `Failed` status
  bucket (`FAILED`, `PLAN FAILED`, `EPIC FAILED`, or a `status_bucket="Failed"`
  override).
- `clan_running_lane_rows` finds members whose own row or session turn is still in
  flight, even when a member's mirrored bucket has not caught up.
- A sequential session's failure frontier is already defined by
  `failure_focus_for_agent` (`src/sase/ace/tui/models/failure_focus.py`): the newest
  `Failed` descendant that was not retried and is not a held (not-adopted) monitor. The
  clan `NEEDS ATTENTION` callout and the clan row's failure focus already name that row,
  for example `research.8--1` in the failed-clan incident fixture.

## Behavior after the fix

- **Clan row.** Pressing `l` on a selected clan row with **no running member and exactly
  one failed member** jumps to that member. "Running" means any member in the
  `Running`/`Starting` bucket (the `R` chip) or any member with an in-flight row or
  session turn (`clan_running_lane_rows`). Waiting, queued, asking/plan-review, and done
  members do not block the jump, because none of them are running.
  - If the failed member is a sequential session whose failure frontier is one of its
    concrete turns, the target is that turn (for example `foo.bar--code`). This drills
    through the same way as the approved `clan_drill = monitor` precedent. Otherwise the
    target is the member row itself (a plain agent, a workflow, or a session whose only
    failed row is the container).
  - The jump works like every other member jump: `_try_reveal_agent_row` expands only
    the folds on the target's path, pushes one jump anchor (Ctrl+O returns), and shows
    no toast. `h` walks back up as usual.
  - Zero failed members, two or more failed members, or any running member: `l` behaves
    exactly as today. The lone-running-member rule needs exactly one running member and
    the failed rule needs zero, so the two rules can never compete.
- **Session row** (`session_rows`). For a selected sequential-session row whose own
  bucket is `Failed` and that has no in-flight turn:

  > [!decision] session_rows = jump `l` selects the session's failure-frontier turn,
  > expanding the session first if needed, at every fold level. This is the same
  > trade-off as the running-monitor rule: while the session stays failed, an EXPANDED
  > session no longer steps to FULLY_EXPANDED from its own row. That step stays
  > available by pressing `l` on any of its turn rows, which advances the governing
  > session fold.

  > [!decision] session_rows = clan_only The session-row branch is not added. `l` on a
  > failed session row keeps today's fold ladder. The session helper is still used for
  > the clan drill-through.

  A session with a running monitor keeps the existing running-monitor jump. The failed
  rule requires no in-flight turn, so the two rules can never compete.

- **Footer chip.** The chip text names what `l` will land on: `running member`,
  `running monitor`, `failed member`, or `failed turn`. The chip and the key must keep
  sharing one resolver so they never disagree.
- **Unchanged guards.** Whole-panel focus, grouping-banner focus, Archive routing, and
  the Tools-card (LLM Calls / Runs) routing keep their current precedence ahead of the
  jump. A reveal failure (for example, the target is filtered out) silently falls back
  to today's fold behavior.

## Implementation

1. **Clan model helper** (`src/sase/ace/tui/models/_agent_clan.py`, next to
   `clan_lone_running_member`, added to `__all__`):
   `clan_lone_failed_member(agent: Agent) -> Agent | None`.
   - Return `None` for a non-clan row.
   - Make one pass over `clan_members(agent)`, deduped by `identity` like the sibling
     helpers. Return `None` as soon as a member is running: its `agent_status_bucket` is
     in `_CLAN_RUNNING_BUCKETS`, or `agent_row_is_in_flight(member)`, or
     `current_agent_session_turn_row(member) is not None` (the same test
     `clan_running_lane_rows` uses). Collect members whose bucket is `"Failed"`, and
     return `None` on the second one.
   - Return the single failed member, or `None` when there is none.
   - Keep it pure and bounded to the clan's loaded subtree. The footer evaluates it on
     every selection change of a clan row (see the `tui_perf` memory).
2. **Session model helper** (`src/sase/ace/tui/models/agent_session_members.py`, next to
   `agent_session_lone_running_monitor_row`, added to `__all__`):
   `agent_session_failed_turn_row(agent: Agent) -> Agent | None`.
   - Return `None` for clan containers, for anything that is not
     `is_sequential_agent_session_container`, and unless
     `agent_status_bucket(agent) == "Failed"`.
   - Walk `concrete_agent_session_turn_rows(agent)` once, deduped by `identity`. Return
     `None` if any row is `agent_row_is_in_flight`.
   - The candidates are rows that are not the container itself
     (`row.identity != agent.identity`, because a container that represents its own
     first turn mirrors the session's status), have
     `monitor_row_lane_bucket(row) == "Failed"` (a failed check whose continuation
     launched does not count), have `row.retried_as_timestamp is None`, and are not
     `row_is_not_adopted_handoff(row)`. These are the same exclusions
     `failure_focus_for_agent` uses.
   - Return the last candidate in the walk's causal order, or `None`.
3. **Unified resolver** (`src/sase/ace/tui/actions/agents/_folding_clans.py`).
   - Add a frozen, slotted `MemberJumpTarget` dataclass with `row: Agent` and
     `label: str`.
   - Rename `_resolve_running_member_jump_target` to `_resolve_member_jump_target`,
     which returns `MemberJumpTarget | None`. Rename `_jump_to_running_member(member)`
     to `_jump_to_member_row(row)`. Update every reference, including the test stubs in
     `tests/ace/tui/test_agent_display_defer_detail.py`.
   - Keep all existing guards and the `try/except` → `None` defensive shape.
   - **Clan container:** first, the existing lone-running rule. If the member's
     `agent_session_lone_running_monitor_row` exists, the target is
     `(monitor, "running monitor")`; otherwise it is `(member, "running member")`.
     Otherwise, if `clan_lone_failed_member` returns a member, the target is
     `(agent_session_failed_turn_row(member), "failed turn")` when that returns a row,
     else `(member, "failed member")`.
   - **Any other row:** first,
     `(agent_session_lone_running_monitor_row(selected), "running monitor")`. Under
     `session_rows = jump`, fall back to
     `(agent_session_failed_turn_row(selected), "failed turn")`.
   - Never return a target whose `row.identity` equals the selected row's identity.
   - Update the docstring.
4. **`_expand_fold`** (`_folding_agent_groups.py`): call the renamed resolver and pass
   `target.row` to `_jump_to_member_row`. Keep the clan-only "already expanded with no
   target → no-op" branch, and the fall-through to `_fold_manager.expand(wf_key)` for
   other rows. Update the docstring bullets.
5. **Footer.**
   - In `src/sase/ace/tui/actions/agents/_display_detail_footer.py`, call the resolver
     once and pass `member_jump_label = target.label if target else None`.
   - Replace the two keywords `running_member_jump_available` and
     `running_member_jump_label` with one keyword,
     `member_jump_label: str | None = None`. Update both signatures and the forwarding
     call in `src/sase/ace/tui/widgets/_keybinding_modes.py`, and
     `src/sase/ace/tui/widgets/_keybinding_bindings_agents.py`. There, the chip
     `(expand_or_layout key, member_jump_label)` is appended when the label is not
     `None` and the panel is not focused.
6. **Help and config text.**
   - In `src/sase/ace/tui/modals/help_modal/agents_mode_sections.py`, change
     `"Clan/session: jump to lone running member/monitor"` to
     `"Clan/session: jump to lone running/failed node"`, and update
     `tests/test_keymaps_display_help_agents.py`.
   - In `src/sase/default_config.yml`, extend the `expand_or_layout` comment block (the
     `l expands or enters a selected panel` sentence). It should say that with no
     running member, `l` on a clan with one failed member jumps to it, or to that
     session's failed turn. Under `jump`, add that `l` on a failed session row jumps to
     its failed turn.
7. **Docs.** Read the `docs_writing` memory first. These are behavior updates to
   existing Reference pages, so keep each to a sentence or two:
   - `docs/reference/agents/agent-clans.md` (the "When exactly one direct member is
     running (the row shows `R1`)…" passage): add the failed rule (`F1` and no running
     member → the failed member, or its session's failed turn). Replace "With zero or
     several running members, `l` only reveals the members" with "Otherwise, `l` only
     reveals the members".
   - `docs/reference/tui/agents-tab-grouping.md` (the "unless the clan has exactly one
     running member…" sentence): add the failed-member case.
   - `docs/reference/tui/agents-tab-panels.md` (the "The running-member/monitor jump is
     the exception…" sentence): add the failed clan case, plus the failed session case
     under `jump`.
   - `docs/reference/tui/agent-row-glyphs.md` (the session `l` paragraph that mentions
     the running monitor): under `jump`, add that a failed session with nothing running
     selects its newest failed turn.
   - Run `just fix` so any generated keybinding reference picks up the help-text change.

No `sase-core` change is needed. This is TUI navigation over already-projected rows, and
it does not change how statuses, counts, or failure focus are computed.

## Tests

- **Clan model** (`tests/test_agent_clan.py`, next to the `clan_lone_running_member`
  tests). `clan_lone_failed_member`:
  - returns the member for `[FAILED, DONE, WAITING]`, `[FAILED, QUESTION]`, and for a
    `status_bucket="Failed"` override;
  - returns `None` for two failed members, for a failed member next to a
    `RUNNING`/`STARTING` member, and for a failed member next to a member that is in
    flight but bucketed `Done` (`status="RUNNING"`, `status_bucket="Done"`, no
    `stop_time`);
  - returns `None` for a non-clan row and for zero failed members;
  - dedupes repeated identities.
- **Session model** (`tests/ace/tui/models/test_agent_session_members_current.py`; reuse
  `tests/ace/tui/models/_agent_session_members_helpers.py`).
  `agent_session_failed_turn_row`:
  - returns the failed last agent turn of a `Failed` session;
  - still returns that agent turn when a newer held (not-adopted) monitor exists (the
    incident shape);
  - skips a failed turn with `retried_as_timestamp` set;
  - returns `None` when the session bucket is not `Failed`, when any turn is in flight,
    when the only failed row is the container itself, and for clan containers and
    non-session rows.
- **Action.** Add `tests/ace/tui/test_agent_clan_failed_member_l_jump.py`, using the
  `ClanLJumpHarness` pattern from
  `tests/ace/tui/test_agent_clan_running_member_l_jump.py` and
  `tests/ace/tui/test_agent_session_monitor_l_jump.py`:
  - collapsed clan `[FAILED, DONE, WAITING]`: `action_expand_or_layout` selects the
    failed member, the clan fold is no longer COLLAPSED, the anchor stack grows by
    exactly 1, and there are no notifications;
  - expanded clan: selects the failed member with fold states unchanged;
  - `[FAILED, RUNNING, DONE]`: lands on the running member (existing rule wins);
  - `[FAILED, FAILED, DONE]`: today's behavior (expand, then no-op), selection stays on
    the clan;
  - a clan whose lone failed member is a session with a failed `--code` turn: lands on
    that turn, with the clan and session folds on its path opened;
  - under `jump`, a collapsed failed session row selects its failed turn, and an
    expanded one selects it without a fold change. Under `clan_only`, the failed session
    keeps the COLLAPSED → EXPANDED → FULLY_EXPANDED ladder.
- **Existing tests.**
  - In `test_agent_clan_running_member_l_jump.py`,
    `test_zero_running_keeps_today_behavior` uses `["DONE", "WAITING", "FAILED"]`, which
    now jumps. Change its fixture to have no failed member (for example
    `["DONE", "WAITING", "QUESTION"]`) so it keeps testing the fallback.
  - Grep `tests/` for other `l` / `action_expand_or_layout` tests on failed clans or
    failed sessions, and adjust any whose expectation changes.
- **Footer.**
  - Update the `test_agent_display_defer_detail.py` stubs to return `MemberJumpTarget`,
    and assert that `member_jump_label` passes through for each label and is `None` when
    the resolver returns `None`.
  - Update `tests/ace/tui/widgets/test_keybinding_footer_llm_calls_detail.py` so the
    chip renders the given label (including `failed member`) and is absent for `None`
    and under panel focus.
- **Visual snapshots.** The new chip changes any golden that shows a selected failed
  clan row (for example `test_failed_clan_incident_by_status_collapsed_png_snapshot`)
  or, under `jump`, a selected failed session row.
  - Under `jump`, `test_failed_session_incident_by_status_expanded_png_snapshot` presses
    `l` on the failed session, which now selects `--1`. Add an `h` after that `l` so the
    session row stays the subject, because the snapshot is about the session's NEEDS
    ATTENTION panel.
  - Refresh only the affected goldens, following the `tui_screenshot` memory's Golden
    Maintenance rules (targeted `just fix-tui-screenshots`, then inspect its report and
    every golden change). Do not run `just check-full`.

## Verification

- Run `just check`, which must pass.
- Optional manual check: in `sase tui`, select a clan whose row shows `F1` and no `R`
  count. The footer shows `l failed member` or `l failed turn`, and one `l` selects the
  failed node. `h` walks back up, and Ctrl+O returns to the clan row. Under `jump`, do
  the same from a failed session row.

## Out of scope

- Jumping when two or more members failed (there is no single target), and choosing a
  failed member over a member that is waiting for input. Enter on a clan keeps
  delegating to its lone stopped member.
- Drilling into a failed workflow step of a non-session workflow member. The member row
  is the target.
- Any change to status mirroring, clan counts, failure focus, or Rust core.
