---
tier: tale
title: Enter acts on a clan's lone stopped member
goal:
  Pressing Enter on an agent clan node with exactly one stopped member acts on that
  member as if its row were selected.
size: small
decisions:
  siblings:
    ask:
      Should Enter still act on the lone stopped member while other clan members are
      running or failed?
    choices:
      any_siblings:
        Act whenever exactly one member is stopped (matches the S1 chip), whatever the
        others do
      mirrored_only:
        Act only when the clan row mirrors that member (no running or failed sibling);
        else toast
    default: any_siblings
    why:
      The S1 chip already promises one member needs you; running siblings never own a
      pending gate
    answer: any_siblings
proposed_by: bbugyi200.athena.11c
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-18 10:56:27
status: wip
---

# Make `Enter` act on a clan's lone stopped member

## Problem

On the Agents tab, pressing `Enter` (`act_on_agent`) on an agent clan node always toasts
`Select an agent inside this clan`. The screenshot shows an `@epic` clan row
`bob-cli (SUDO) ×3 [S1 D2]` (clan `bob-cli-8v`): one member (`.land`) is stopped on a
pending SUDO gate and the other two are done. The row already shows the member's `SUDO`
status, the clan sits in the `Stopped` group (`1 awaiting`), and the footer lists no
`Enter` action, yet `Enter` refuses to act. The user has to expand the clan or press a
member jump key first. When exactly one member is waiting on the user, there is only one
reasonable thing for `Enter` to do.

Root cause: `resolve_agent_enter_targets()` in
`src/sase/ace/tui/actions/agents/_agent_enter_resolver.py` returns an empty resolution
with `CLAN_EMPTY_MESSAGE` as soon as `classify_scope()` returns `"clan"`. It never looks
at the clan's members. The footer hides the hint for the same reason, and
`_compute_agent_bindings()` in `src/sase/ace/tui/widgets/_keybinding_bindings_agents.py`
returns early in its clan branch without ever reaching the `act_on_agent` hint.

## Behavior after this change

A clan container with **exactly one stopped direct member** acts on that member as if
the member row were selected:

- `Enter` resolves the member's targets: its pending gate(s) for a session member, the
  question or plan-review notification, the question-marker or workflow-HITL fallback,
  remote attention, and the member's Patch last. It runs or offers them exactly as
  `Enter` on the member row would. In the screenshot this opens the `.land` member's
  SUDO gate.
- The chooser, when several targets apply, is titled `Act on <member>` (the member's
  scope title), so it is clear which agent is being acted on.
- The footer shows the same `Enter` hint the member row would show (for example
  `review sudo request`, or `choose action`), and the command palette's "Act on agent"
  becomes available.
- Every follow-up step uses the member's identity, never the clan's: the
  question-marker, HITL, and remote-attention executors; chooser revalidation; and the
  off-thread gate-detail revalidation.

Everything else stays the same. A clan with zero stopped members, or two or more, still
toasts `Select an agent inside this clan`. So does a clan whose lone stopped member
resolves no targets, because "No pending gate or Patch for this agent" would misdescribe
a clan row.

"Stopped" means the same "needs you" rule that the header's `stopped` count and clan
waves already use:
`agent_status_bucket(member) == "Stopped" or agent_is_asking(member.status)`. It covers
a pending gate (bucket override), a plan waiting for review, `QUESTION`, and
`WAITING INPUT`. Members come from `clan_members()`, deduplicated by `identity` the way
`clan_member_counts()` does it. Nested clan containers are never members, so delegation
cannot recurse.

> [!decision] siblings = mirrored_only Also return `None` from the helper below when any
> other unique member's `agent_status_bucket` is `Running`, `Starting`, or `Failed`, so
> delegation happens only when the clan row mirrors the stopped member
> (`clan_status_source_index` semantics). Add a resolver test showing that a stopped
> member plus a running sibling keeps the toast, and drop the "running sibling still
> delegates" test.

## Implementation

1. **Clan helper**: `src/sase/ace/tui/models/_agent_clan.py`.
   - Add `clan_lone_stopped_member(agent: Agent) -> Agent | None`, next to
     `clan_running_lane_rows`. Return `None` for non-clan rows. Otherwise walk
     `clan_members(agent)`, deduplicate by `identity`, keep the members matching the
     "needs you" rule above, and return the single match or `None`. Both
     `agent_is_asking` and `agent_status_bucket` are already imported there. Add the
     helper to `__all__`. Its docstring should name the rule and say why it matches the
     header's stopped count.
   - Keep it pure and in-memory. It runs on the footer and palette paths every time the
     selection changes (`sase/memory/tui_perf.md` rules 1 and 11): no I/O and no
     resolvers with side effects.

2. **Resolution subject**: `src/sase/ace/tui/actions/agents/_agent_enter_models.py`.
   - Add `subject_identity: tuple[object, ...] | None = None` to `AgentEnterResolution`,
     documented as "Identity of the row the targets act on when it is not the selected
     row (a clan's lone stopped member); `None` means the selected row."

3. **Resolver delegation**: `src/sase/ace/tui/actions/agents/_agent_enter_resolver.py`.
   - In the `scope == "clan"` branch, call `clan_lone_stopped_member(agent)`. When it
     returns a member, recursively call `resolve_agent_enter_targets(member, ...)` with
     the same `gate_notifications`, `patch_name_for`, and `patch_lookup`. If that
     resolution has targets, return
     `dataclasses.replace(delegated, subject_identity=identity_of(member))`, which keeps
     the member's `scope_title`. Otherwise fall through to the existing
     `CLAN_EMPTY_MESSAGE` resolution.
   - Update the function docstring's scope summary to mention the clan delegation.

4. **Dispatch with the subject's identity**:
   `src/sase/ace/tui/actions/agents/_agent_enter_action.py`.
   - In `_dispatch_agent_enter_resolution`, prefer `resolution.subject_identity` over
     `agent.identity` when choosing `identity`. Keep the existing `row_identity`
     fallback. Nothing else needs to change. `_run_agent_enter_target`, the chooser's
     `_on_choice` (which re-captures `current` by that identity and re-resolves it), and
     `_finish_agent_enter_gate_load` all then work on the member. `_agent_by_identity`
     already searches `_agents_with_children`, so folded clan members still resolve.

5. **Footer hint for clans**: `src/sase/ace/tui/widgets/_keybinding_bindings_agents.py`.
   - In the `if agent.is_clan_container:` branch of `_compute_agent_bindings` (the one
     that adds `kill/dismiss clan` and returns early), append
     `(self._kd("act_on_agent"), enter_action_label)` when `enter_action_label` is set.
     This matches the non-clan path. `_display_detail_footer.py` already computes
     `enter_action_label` for clan rows through `_agent_enter_resolution`, so no change
     is needed there.

6. **Docs**: `docs/reference/tui/agents-tab.md`, section "Enter: act on an agent".
   - Replace the clan half of the bullet "A **clan container** toasts
     `Select an agent inside this clan`…" with the new rule. A clan with exactly one
     stopped member (a pending gate, a plan waiting for review, or a question) acts on
     that member: the footer hint, the targets, and the `Act on <member>` chooser are
     the member's. Any other clan, or a lone stopped member with no target, toasts
     `Select an agent inside this clan`. Keep the monitors, named procs, and grouping
     banner sentence. Follow `sase/memory/docs_writing.md` (read it with
     `/sase_memory_read`): this is a Reference-page behavior update, with no page moves
     and no new guide prose.

## Tests

Put the resolver and executor coverage in a new
`tests/ace/tui/test_agent_enter_targets_clan.py`, built on
`tests/ace/tui/_agent_enter_targets_helpers.py` (`_resolve`, `_gate_row`,
`_agent_session`, `_notification`, `_matching_action_data`, `_EnterApp`). Build a clan
container with
`replace(make_agent(...), is_clan_container=True, agent_clan=..., agent_clan_generation=...)`
and set `runtime_children` to members that carry the same `agent_clan` and
`agent_clan_generation` (see `tests/test_agent_clan.py`). Keep the existing memberless
`test_clan_container_points_inside` and the wire toast test unchanged.

- **Resolver.**
  - A clan whose members are one session (via `_agent_session`, with a pending SUDO
    `_gate_row` member and the session row's `status_bucket="Stopped"`) plus two `DONE`
    members resolves exactly the session's targets. Its `scope_title` is the session's,
    and its `subject_identity` equals the session row's `identity`.
  - A lone `QUESTION` member with no notification yields its `question_marker` target.
  - Zero stopped members (running plus done) keeps `CLAN_EMPTY_MESSAGE`, with no targets
    and `subject_identity is None`.
  - Two stopped members keep `CLAN_EMPTY_MESSAGE`.
  - The same member present twice in `runtime_children` (same identity) still counts as
    one and delegates.
  - A lone stopped member that resolves no targets keeps `CLAN_EMPTY_MESSAGE`. For
    example, use a `PLAN` member with no matching notification whose `cl_name` is the
    Patch sentinel `~`: `PLAN` gets neither the question-marker nor the HITL fallback.
  - A stopped member plus a `RUNNING` sibling still delegates (default `siblings`
    branch).
  - A stopped outsider from another clan or generation is ignored.
- **Executor and dispatch.** Use `_EnterApp(agents=[clan, member, ...])` and call
  `app._dispatch_agent_enter_resolution(clan, app._agent_enter_resolution(clan))`:
  - For a lone `QUESTION` member, `app.marker_calls == [member]` (the member, not the
    clan).
  - For a member with two targets (a pending gate plus a Patch), stub `push_screen` on
    the instance to capture the chooser and callback. Assert the chooser title is
    `Act on <member title>`. Invoke the callback with the gate target's key and assert
    it re-resolves through the member and dispatches, for example by monkeypatching
    `..._notification_dispatch.open_notification_action` as the existing executor tests
    do.
- **Clan helper unit tests** in `tests/test_agent_clan.py` for
  `clan_lone_stopped_member`:
  - Non-clan row returns `None`.
  - A `status_bucket="Stopped"` override counts.
  - `WAITING INPUT` counts through `agent_is_asking`.
  - Two stopped members return `None`.
  - Identity duplicates are deduplicated.
- **Footer.** In `tests/ace/tui/test_agent_enter_wire.py`, next to
  `test_footer_enter_hint_primary_and_secondary`, use a clan container row
  (`is_clan_container=True`, `agent_clan` set). Assert that
  `footer._compute_agent_bindings(clan, enter_action_label="review sudo request")`
  contains `(footer._kd("act_on_agent"), "review sudo request")` alongside
  `kill/dismiss clan`, and that no `act_on_agent` key appears when
  `enter_action_label=None`.
- **Wire, optional but preferred.** In `tests/ace/tui/test_agent_enter_wire.py`, an
  `async` test where `_WireApp` selects a clan container whose lone stopped member
  (`PLAN` status) has a matching `PlanApproval` notification. Add the member to
  `_agents_with_children`. Pressing `enter` once dispatches that notification id;
  monkeypatch `open_notification_action` the way `test_enter_sudo_gate_dispatches` does.

## Verification

- Run `just fmt`, then `sase tool run check` (the wrapped `just check`). Do not run
  `just check-full`.
- No PNG visual golden should change, because no golden selects a clan with a lone
  stopped member. If `just check` reports one anyway, follow `sase/memory/tui.md` and
  its `tui_screenshot.md` child before regenerating anything.

## Non-goals

- **Clans with two or more stopped members.** They keep the toast. Combining several
  members' targets into one chooser would need a subject identity per target and is a
  separate feature.
- **Other surfaces.** No change to member jump keys (`0-9`), clan kill or dismiss,
  `can_jump` / `jump_to_agent_patch` on clans, the clan status projection, or the
  `S`/`D` count chip.
- **Rust core.** No sase-core change. Enter-target resolution is TUI interaction logic
  that already lives in this repo, and member buckets already come through the existing
  `status_bucket_for_status` binding.
