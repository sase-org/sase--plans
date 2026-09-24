---
tier: tale
title: Let ACE fork finished EPIC CREATED and PLAN COMMITTED planner rows
goal: Pressing F on a finished EPIC CREATED or PLAN COMMITTED agent row opens a
size: small
proposed_by: bbugyi200.athena.0ri
create_time: 2026-09-24 17:19:12
status: wip
---

# Plan: Let ACE Fork Finished `EPIC CREATED` / `PLAN COMMITTED` Planner Rows

## Problem

In the Agents tab, selecting the family root `0re` (row status `EPIC CREATED ✓`, in the
**Done** bucket) and pressing `F` (fork) shows the warning toast **"Agent not finished
yet"**, and no `#fork:0re` prompt opens. The footer for that row is wrong in the same
way: it shows `x kill` and `W new w/ wait` (the active-agent hints) and no `F fork` or
`e edit chat`, even though the agent finished long ago.

## Root Cause

This is a TUI-only status-classification gap. Nothing downstream is broken.

`resolve_agent_prompt_target_scope(..., action="fork")` in
`src/sase/ace/tui/actions/agents/_wait_helpers.py` sorts a row into one of these
branches:

1. The status is **not** in `DISMISSABLE_STATUSES` (an active row) → fork by name.
2. The status is `PLAN DONE` / `TALE DONE` on a non-root row → fork the coder follow-up.
3. The status is failed → fork by name.
4. Otherwise `is_resumable_done_status(status)` must be true, or the helper returns
   `"Agent not finished yet"`.

`RESUMABLE_DONE_STATUSES` in `src/sase/ace/tui/models/agent_status.py` is only
`{"DONE", "PLAN DONE", "TALE DONE"}`. It was added in May for the old `r` resume action,
and that plan deliberately left out `EPIC CREATED`, `PLAN COMMITTED`, and
`PLAN REJECTED` as "terminal for cleanup but not necessarily resumable chat
continuations". The `r` resume action no longer exists. Today this predicate only gates
**fork** and **edit chat**, but it still has the old exclusions. So `EPIC CREATED` and
`PLAN COMMITTED` fall into a gap: they are in `DISMISSABLE_STATUSES`, so branch 1 skips
them, but they are not "resumable", so branch 4 rejects them with a misleading message.
Four places share the same predicate and therefore the same gap:

- the fork action (`_wait_helpers.py`), which produces the toast in the screenshot;
- the footer (`src/sase/ace/tui/widgets/_keybinding_bindings_agents.py`). The row drops
  into the "RUNNING or other active statuses" `else` branch, which gives `x kill`,
  `W new w/ wait`, and no fork;
- command-palette availability for `app.edit_hooks` (fork) and `app.edit_spec`
  (`src/sase/ace/tui/commands/_availability_agents.py`);
- edit chat (`src/sase/ace/tui/actions/agents/_panel_detail.py`), which shows the same
  "Agent not finished yet" toast.

### Evidence that the rest of the pipeline already supports this fork

These checks ran against the real `0re` artifacts:

- `python -m sase.scripts.agent_chat_from_name 0re` resolves a `family` fork source with
  members `0re--plan` (completed, has a chat transcript), `0re--gate` (answered), and
  `0re--mon` (the completed epic-launch monitor proc). So `#fork:0re` builds context
  correctly.
- The runner's wait index (`build_wait_dependency_index(...)` +
  `dependency_resolution_status(idx, ["0re"], [fork_wait_dependency("0re")])`) returns
  `resolved`. So the implied `%wait:0re` from `#fork:0re` releases right away.
  (`EPIC CREATED` maps to outcome `epic_approved`, and `PLAN COMMITTED` maps to
  `plan_committed`. Both are in `WAIT_SUCCESS_OUTCOMES`.)
- The row has a `response_path` (the planner's chat), so the footer and palette
  conditions that need a chat file are also met.

### Why `PLAN REJECTED` and `STOPPED` stay excluded

- `STOPPED` is a repeat slot that never ran, so there is no chat to fork. The existing
  test already pins that it is rejected.
- `PLAN REJECTED` maps to outcome `plan_rejected`, which is neither a wait success nor a
  failure outcome. A check against a real rejected family (`0br`) showed that
  `fork_source_status` for its implied fork wait stays `waiting`. A `#fork:<rejected>`
  child would therefore block forever. Making it forkable needs a change to the
  fork-wait semantics in the wait index. That is out of scope here; see "Out of Scope".

## Design

Keep one shared predicate, and widen it to "finished successfully with a chat that can
be opened or forked". Then fork, edit chat, footer, and palette all move together and
cannot drift apart again.

## Implementation Steps

1. **`src/sase/ace/tui/models/agent_status.py`**
   - Add `"EPIC CREATED"` and `"PLAN COMMITTED"` to `RESUMABLE_DONE_STATUSES`. Keep the
     name so the change stays small, since `#fork` is now the way a finished chat is
     continued.
   - Update the `is_resumable_done_status` docstring and add a short comment above the
     set. It should say the set means "terminal rows whose finished chat can be opened
     or forked". It should also say why `PLAN REJECTED` (its outcome never satisfies the
     implied `#fork` wait) and `STOPPED` (never ran, no chat) are excluded.

2. **`src/sase/ace/tui/actions/agents/_wait_helpers.py`**
   - No branch change is needed for the fix itself. An `EPIC CREATED` / `PLAN COMMITTED`
     row now reaches the final branch and forks by `action_agent_prompt_name(agent)`.
     For a family root that is the family name (`#fork:0re`).
   - Make the remaining rejection accurate. When the row's status is in
     `DISMISSABLE_STATUSES` (terminal) but not resumable-done, return
     `f"Cannot fork a {agent.status} agent"`. Keep `"Agent not finished yet"` only for
     statuses that really are not terminal (in practice, statuses outside all the
     branches above).

3. **Footer, palette, and edit chat.** No code edits are expected in
   `_keybinding_bindings_agents.py`, `_availability_agents.py`, or `_panel_detail.py`.
   They already call `is_resumable_done_status`, so they pick up the wider set
   automatically. Check that an `EPIC CREATED` row with a `response_path` now gets
   `x dismiss`, `e edit chat`, and `F fork` in the footer (and not `x kill` or
   `W new w/ wait`), and that `app.edit_hooks` / `app.edit_spec` become available.

4. **Tests**
   - Predicate: `is_resumable_done_status` is True for `EPIC CREATED` and
     `PLAN COMMITTED`, and False for `PLAN REJECTED` and `STOPPED`. Extend
     `tests/ace/tui/models/test_agent_status_stopped.py` or add a sibling predicate
     test.
   - Fork action, in `tests/ace/tui/test_agent_marking_wait_fork.py` using
     `_FakeWaitApp` / `_make_agent`:
     - an `EPIC CREATED` family root (`agent_session="alice"`,
       `agent_session_role="root"`, `plan_chain_root=True`) prefills `#fork:alice ` with
       display name `fork(alice)`;
     - a `PLAN COMMITTED` named agent prefills `#fork:<name> `;
     - a `PLAN REJECTED` named agent opens no prompt and warns
       `"Cannot fork a PLAN REJECTED agent"`;
     - update `test_fork_stopped_agent_still_warns_not_finished`. Rename it to reflect
       the new message and assert `"Cannot fork a STOPPED agent"`.
   - Footer, in `tests/test_keybinding_footer_agent.py`: an `EPIC CREATED` agent with a
     `response_path` includes the fork key (`_edit_hooks_key`), `("x", "dismiss")`, and
     edit chat, and does not include `("x", "kill")`.
   - Palette, in `tests/test_command_availability_agents_actions.py`, following the
     style of the existing `TALE DONE` test: `app.edit_hooks` is available for
     `EPIC CREATED` with a `response_path` and not available without one.
     `PLAN REJECTED` stays unavailable.

5. **Docs.** In `docs/ace.md` under "Forking Agents and Groups", add one sentence:
   finished planner rows (`EPIC CREATED`, `PLAN COMMITTED`) are fork targets, and a
   family root forks the whole family (planner transcript plus gate/monitor shells).
   Rejected-plan and `STOPPED` rows are not fork targets.

6. **Verify.** Run the targeted tests above, then the repo's standard verification per
   the `lint_and_test` memory (`just check` through the guarded-recipe flow).

## Out of Scope

- Allowing `#fork` of `PLAN REJECTED` rows. This needs the fork-source wait semantics
  (`WaitDependencyForkQueries.fork_source_status` and the session aggregate in
  `src/sase/core/wait_dependency_resolution/`) to treat a `plan_rejected` parent as
  terminal for fork waits, the same way failed parents are handled. If that is wanted,
  file it as a separate task.
- The footer / palette `n name` affordance for other terminal statuses, and the `x kill`
  label for other dismissable statuses that still fall into the active-branch footer
  (for example `PLAN REJECTED` with a stale pid). This plan does not change them.

## Expected Outcome

Selecting `0re` (`EPIC CREATED ✓`) and pressing `F` opens the prompt bar prefilled with
`#fork:0re ` (plus any smart VCS tag). The launched child gets the planner transcript
and the gate/monitor shell context, and its implied `%wait:0re` resolves right away. The
footer for such rows shows `x dismiss · e edit chat · F fork` instead of `x kill`.
