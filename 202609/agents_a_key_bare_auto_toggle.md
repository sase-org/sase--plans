---
tier: tale
title: Agents-tab A toggles bare %auto instead of opening the Auto-Approve menu
goal:
  Pressing A on an active local agent toggles bare %auto (approving whatever plan tier
  the agent proposes) with no panel, and the AutoApproveModal is deleted.
size: small
proposed_by: bbugyi200.athena.0ov
create_time: 2026-09-21 17:09:14
status: wip
---

# Agents-tab `A`: toggle bare `%auto` instead of opening the Auto-Approve menu

## Goal

On the Agents tab, pressing `A` on an auto-approve-eligible local agent should no longer
open the `AutoApproveModal` (the Plan / Tale / Epic / Disable panel). Instead it
**toggles bare `%auto`** on the selected agent:

- auto-approval **off** → enable bare `%auto` (no argument);
- auto-approval **on** (any form: `%auto`, `%auto:plan`, `%auto:tale`, `%auto:epic`,
  whether from launch or from a previous toggle) → disable it.

The Auto-Approve modal, its styling, its exports, its unit tests, and its PNG golden are
deleted.

## Why bare `%auto` is enough now

Plans now declare their own tier in frontmatter (`tier: tale` / `tier: epic`), and a
bare `%auto` approves whatever tier the agent proposes. No backend changes are needed;
this is TUI/presentation work plus docs:

- `sase.main.plan_approve_handler.get_auto_plan_approval_action()` returns `"approve"`
  when agent meta has `approve: true` and no `auto_approve_argument` /
  `auto_approve_plan_action`. `get_auto_plan_approval_argument()` then returns `None`.
- `sase.plan_shell.create.create_plan_gate_shell()` passes `auto_enabled=True`,
  `auto_argument=None` into `build_plan_approval_gate_spec()`.
- `sase._plan_gate_metadata.validate_plan_auto_argument()` accepts `None` for both
  tiers, and `sase.notification_gates.adapters` resolves the auto selection to the
  tier's `primary_branch`: `approve + commit` (a tale) for `tier: tale`, `approve` (an
  epic) for `tier: epic`.
- A pinned `%auto:tale` against an epic plan (or `%auto:epic` against a tale) is
  rejected by `validate_plan_auto_argument`. That is why the menu's Tale/Epic pins are
  no longer worth having.
- `is_auto_approve_active()` also reads meta `approve`, so the toggle has the same
  question-gate auto-answer effect as launching with bare `%auto`. That is intended: the
  key is exactly "bare `%auto`".

The persistence wire this uses already exists. It is the payload the menu's current
**Plan** choice (enable) and **Disable** choice send through `submit_agent_directive`
(`sase agent persist-directive`):

- enable: `meta_remove: [approve, auto_approve_plan_action, auto_approve_argument]`,
  `meta_set: {approve: true}`, `prompt: {kind: set_auto_mode, mode: "plan"}`.
  `set_prompt_auto_mode(prompt, "plan")` writes the canonical bare `%auto` line.
- disable: same `meta_remove`, `meta_set: {}`,
  `prompt: {kind: set_auto_mode, mode: null}`, which strips the `%auto` directive.

The Rust core is not involved. The toggle decision is TUI glue over an existing Python
ops wire, so nothing changes in `sase-core`. This is also not a deprecation that must
keep the old branch reachable, so no feature flag is needed.

## Changes

### 1. `src/sase/ace/tui/actions/agents/_approve.py`: menu becomes a toggle

- Delete `_auto_approval_choice_for_agent`, `_auto_approval_state_for_choice`, and the
  `AutoApproveChoice` `TYPE_CHECKING` import.
- Add a module helper `_auto_approve_active(agent) -> bool` that returns
  `bool(agent.approve or agent.auto_approve_plan_action)`. The in-memory `approve` stays
  `True` for tale/epic agents, but check both fields to be safe.
- Replace `action_open_auto_approve_menu` with `action_toggle_auto_approve(self)`. Keep
  the same guards and warnings: no-op off the agents tab, `"No agent selected"` warning,
  and `"Agent not in an active status"` warning for statuses outside
  `AUTO_APPROVE_ELIGIBLE_STATUSES`. Then call
  `self._set_auto_approve(agent, enabled=not _auto_approve_active(agent))`. No
  `push_screen`.
- Replace `_apply_auto_approve_choice(agent, choice)` with
  `_set_auto_approve(self, agent, *, enabled: bool)`. Keep today's machinery unchanged:
  the artifacts-dir warning, the `_directive_generation` guard, optimistic in-memory
  patch, `_try_patch_agent_row` with the `_refresh_agents_display(list_changed=True)`
  fallback, rollback plus an error toast on persist failure, and the
  `Persist auto: <name>` proc display name. Only the state mapping collapses to two
  cases:
  - enabled: `agent.approve = True`, `agent.auto_approve_plan_action = None`, meta and
    prompt payload as in "enable" above. Toast: `Auto-approve enabled (%auto)`.
  - disabled: `agent.approve = False`, `agent.auto_approve_plan_action = None`, meta and
    prompt payload as in "disable" above. Toast: `Auto-approve disabled`.
- Update the module and class docstrings to describe the bare-`%auto` toggle.

### 2. `src/sase/ace/tui/actions/proposal_rebase.py`

In `action_accept_proposal`, the agents-tab branch for
`agent.status in AUTO_APPROVE_ELIGIBLE_STATUSES` now calls
`self.action_toggle_auto_approve()` instead of `self.action_open_auto_approve_menu()`.
Remote rows (`action_answer_remote_attention`) and `WAITING INPUT` (HITL answer) routing
stay unchanged.

### 3. Delete the modal and its wiring

- Delete `src/sase/ace/tui/modals/auto_approve_modal.py`.
- Remove the `AutoApproveChoice` and `AutoApproveModal` entries from
  `src/sase/ace/tui/modals/__init__.py` (`__all__`),
  `src/sase/ace/tui/modals/__init__.pyi`, and
  `src/sase/ace/tui/modals/_export_table.py`.
- Remove the "Auto-Approve quick-action menu styling" block (`AutoApproveModal`,
  `#auto-approve-container`, `#auto-approve-title`, `.auto-approve-choice-row`,
  `.auto-approve-choice-row.selected`, `#auto-approve-agent`, `#auto-approve-footer`)
  from `src/sase/ace/tui/styles.tcss`.
- Delete `tests/ace/tui/modals/test_auto_approve_modal.py`.
- Delete `test_auto_approve_modal_png_snapshot` from
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_modals.py`, prune any imports it
  leaves unused, and `git rm`
  `tests/ace/tui/visual/snapshots/png/auto_approve_modal_60x30.png`.
- Afterwards,
  `git grep -n "AutoApprove\|auto_approve_modal\|open_auto_approve_menu\|_apply_auto_approve_choice"`
  (excluding `CHANGELOG.md`) must return nothing.

### 4. Footer label: `src/sase/ace/tui/widgets/_keybinding_bindings.py`

In the agent-bindings branch that appends the `accept_proposal` label for
`AUTO_APPROVE_ELIGIBLE_STATUSES`, make the label state-aware so it names what the key
will do: `"auto-approve"` when auto-approval is off, `"unapprove"` when it is on (use
the same `approve or auto_approve_plan_action` test). Replace the stale comment about
the key "always opening the Auto-Approve menu".

### 5. Help text: `src/sase/ace/tui/modals/help_modal/agents_bindings.py`

Change the Agent Actions row for `d(a.accept_proposal)` from
`"Auto-approve / answer local or remote attention"` to
`"Toggle %auto / answer local or remote attention"`. Leave the `⚡` / `⚡T` / `⚡E`
glyph legend alone: agents launched with `%auto:tale` / `%auto:epic` still render those
icons.

### 6. Comment touch-up: `src/sase/ace/tui/models/_agent_state.py`

The `approve` field comment mentions "the Auto-Approve menu". Change it to "the
Agents-tab `A` toggle".

### 7. Docs: `docs/ace.md`

- Agent Actions key table (`| \`A\` | Open auto-approve menu / answer HITL
  |`): change to `Toggle bare \`%auto\` plan auto-approval / answer
  HITL`. Keep the table aligned; run `just fmt`.
- Replace the "For active Agents-tab rows, `A` opens the **Auto-Approve menu** …"
  paragraph (near the Plan Approval section) with a description of the toggle. It should
  say:
  - On an active agent, `A` toggles bare `%auto`. If auto-approval is off, it turns it
    on exactly as if the agent had been launched with `%auto`. If any auto-approval is
    on (including a launch-time `%auto:tale` / `%auto:epic`), it turns it off.
  - The agent's next submitted plan is approved at its authored tier. A `tier: tale`
    plan is approved and committed as a tale; a `tier: epic` plan is approved as an epic
    and follows the epic follow-up path.
  - The row shows `⚡` while enabled, and the footer label switches between
    `auto-approve` and `unapprove`.
  - Like bare `%auto`, it also auto-settles question gates. Delete the old "unlike bare
    `%auto`, it does not automatically answer questions" sentence: it is inaccurate for
    the meta `approve` flag.
- Leave `docs/blog/posts/structured-agentic-software-engineering.md` unchanged (a dated
  publication), and leave `CHANGELOG.md` alone (release tooling owns it).

## Tests

- Rewrite `tests/ace/tui/test_agent_toggle_approve.py` for the toggle. Keep its
  `FakeApproveApp` harness; `push_screen` can go. Cover:
  - off → on: in-memory `approve is True`, `auto_approve_plan_action is None`, one
    optimistic refresh, the write is scheduled rather than done inline, and the
    `Auto-approve enabled` toast. After running the scheduled proc, `agent_meta.json` is
    exactly `{"approve": true}`.
  - off → on with preexisting meta keys: other keys are preserved, and stale
    `auto_approve_plan_action` / `auto_approve_argument` keys are removed.
  - plain on → off: meta keeps only unrelated keys, and the `Auto-approve disabled`
    toast appears.
  - launch-time tale/epic on → off: an agent with `approve=True`,
    `auto_approve_plan_action="epic"` and meta containing `auto_approve_plan_action` and
    `auto_approve_argument` ends fully disabled, with all three keys removed.
  - The artifact-index refresh still fires, as in the existing
    `update_agent_artifact_index_for_marker_mutation` test.
  - rollback on persist failure: the optimistic flip is reverted, an error toast
    appears, and refresh fires twice.
  - guards: off the agents tab it is a no-op; an ineligible status and no selected agent
    each warn and schedule nothing; a missing artifacts dir warns and schedules nothing.
  - optional: the persisted prompt gains or loses the bare `%auto` line, if the fake's
    `_persist_directive_from_payload` path writes a prompt file that is easy to assert
    on.
- `tests/test_keybinding_footer_agent.py`: replace
  `test_keybinding_footer_approve_eligible_shows_auto_approve_label` with a state-aware
  assertion. `(False, None)` shows `"auto-approve"`. `(True, None)`, `(True, "tale")`,
  and `(True, "epic")` show `"unapprove"`. Keep the neighbouring test's expectation that
  ineligible agents show neither label.
- `tests/test_keymaps_display_help_panels.py`: update the
  `("A", "Auto-approve / answer local or remote attention")` expectation to the new help
  text.

## PNG goldens

- Delete `auto_approve_modal_60x30.png` (step 3).
- The footer label change can alter goldens from
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_auto_approve.py`. It is the only
  visual test module with `approve=True` agents. Run a targeted capture:
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_auto_approve.py tests/ace/tui/visual/test_ace_png_snapshots_agents_modals.py tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py`
  (through `/sase_monitor` with `TESTING`/`TESTED` if it runs long). Inspect the report
  under `.pytest_cache/sase-visual/`. Only footer-label differences (`auto-approve` →
  `unapprove` on selected `⚡` rows) and the removed modal golden are expected. Anything
  else is a regression to investigate, not approve.

## Verification

1. `just install` if the workspace venv is stale, then `just fmt`.
2. `sase tool run check` (or `just check`). The lint gates include symvision, so removed
   symbols must leave no dangling references or whitelist entries. Do **not** run
   `just check-full`.
3. The targeted PNG run above.
4. Optional manual smoke test, if the implementer launches the TUI: on the Agents tab,
   select a RUNNING agent without `⚡` and press `A`. No panel opens, the toast says
   `Auto-approve enabled (%auto)`, the row shows `⚡`, the footer shows `A unapprove`,
   and the agent's `agent_meta.json` has `"approve": true`. Press `A` again: the row
   loses `⚡` and the toast says `Auto-approve disabled`.

## Out of scope / known pre-existing limitation

An agent launched with bare `%auto` also has `SASE_AGENT_AUTO_APPROVE=1` in its runner
env (`src/sase/axe/run_agent_runner_launch.py`). `get_auto_plan_approval_action()` and
`is_auto_approve_active()` consult that env var before agent meta. So toggling such an
agent **off** in the TUI clears meta and the prompt, but the already-running process may
still auto-approve its next plan. The deleted menu's Disable choice had the same
limitation; this change does not fix it. If the implementer confirms it, capture it as a
`bug` task bead through `/sase_new_task` rather than widening this change.
