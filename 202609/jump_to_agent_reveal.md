---
tier: tale
title: Notification jumps reveal collapsed and hidden Agents rows
goal:
  Activating a JumpToAgent notification (or a Runners-modal agent jump) always lands on
  and selects the target Agents-tab row, peeling back collapsed folds, grouping banners,
  and panels through the same reveal ladder the Node Finder uses, instead of reporting
  "not found" or selecting an invisible row.
size: medium
proposed_by: bbugyi200.athena.0tb
create_time: 2026-09-27 17:49:45
status: wip
---

# Plan: JumpToAgent reveals collapsed rows

This is the pre-task of the SASE Goals program (research note
`202609/sase_goals_epic_roadmap/sase_goals_epic_roadmap.md`, "Pre-task: `JumpToAgent`
reveals collapsed rows"). It must land before the first Goals epic (G1) is launched,
because every later goal → agent jump reuses this path. Its check-it item is: _a `done`
notification for an agent in a collapsed group opens and selects that row._

## 1. The bug

`handle_jump_to_agent` (`src/sase/ace/tui/actions/agents/_notification_handlers.py`)
sets `app.current_tab = "agents"`, linearly scans `app._agents`, and assigns
`app.current_idx` on the first row whose `cl_name` / `agent_type` / `raw_suffix`
matches. It never goes through the shared reveal contract
(`prepare_agent_navigation_target` / `reveal_agent_navigation_target` in
`src/sase/ace/tui/actions/navigation/_agent_reveal.py`). `app._agents` is the folded and
filtered projection, not the loaded roster (`_agents_with_children`). The results:

1. **Collapsed fold (clan, agent session, workflow parent):** the target is not in
   `_agents` at all, so the user gets `Agent '<name>' not found` for an agent that is
   loaded.
2. **Collapsed grouping banner or collapsed or isolated tribe panel:** the target is in
   `_agents`, so `current_idx` points at a row that is not rendered. Panel focus never
   moves, and the highlight lands on nothing visible.
3. **Other problems:**
   - Hidden by the Agents query or by `I`: the jump dead-ends with "not found".
   - Arriving from another top-level tab: assigning `current_tab` directly skips
     `_save_current_tab_position()` and never restores the Agents index first.
   - The programmatic `current_idx` assignment skips what every other agent-selecting
     path does: acknowledging the target's unread state and saving a `'` / `Ctrl+O`
     jump-back anchor.

`navigate_to_agent_tab` (`src/sase/ace/tui/actions/agents/_notification_navigation.py`,
used by the Runners modal's agents jump in `actions/axe.py::action_show_runners`) has
the same defect (PID match first, then `cl_name`, both over `app._agents` only). Fix
both through one helper.

## 2. The established path to reuse (do not build a new reveal)

- `NodeJumpNavigationMixin._jump_to_node_identity(identity, *, name)`
  (`src/sase/ace/tui/actions/navigation/_node_jump.py`) is the canonical "reach any
  loaded agent by identity" ladder:
  1. It runs the artifact-file-viewer navigation guard.
  2. It calls `_try_reveal_agent_row`.
  3. On `TARGET_MISSING` for an `I`-hidden row, it turns `I` off and retries after the
     reload.
  4. On `TARGET_FILTERED` with an active query, it clears the Agents query once
     (recorded in query history, announced with a restore-hint toast) and retries.
  5. Otherwise it reports through `_notify_member_reveal_failure`.
- `MemberJumpNavigationMixin._try_reveal_agent_row`
  (`src/sase/ace/tui/actions/navigation/_member_jump.py`) does the rest:
  - applies prepare → reveal (ancestor folds, grouping banners, panel expansion);
  - saves a jump-back anchor (restored on failure);
  - moves panel focus and clears `_current_group_key` / `_expanded_panel_focus`;
  - sets `current_idx`;
  - acknowledges the target's unread state;
  - does one structural `_refresh_agents_display(list_changed=True, defer_detail=True)`
    only when needed.
- Both mixins are on `AceApp` via `actions/navigation/_advanced.py`. The Procs-pane
  monitor jump (`modals/procs_pane_agent_jump.py`) and member-digit jumps already use
  this path.

**Decision:** route notification and runner jumps through `_jump_to_node_identity`, the
same ladder the Node Finder uses. As a result, a notification jump to an agent hidden by
the Agents query clears the query with the Node Finder's toast (restorable from query
history), and an `I`-hidden target turns `I` off. Both are existing, documented,
reversible behaviors. The roadmap wants a single path that later goal → agent jumps
reuse, and a second, partial reveal path would drift.

## 3. Step 0: sequence after `sase-1bc.6.1.4` (do this before editing)

`sase-1bc.6.1.4` ("Switch-then-reveal for every cross-tab jump", agent-tabs epic
`sase-1bc.6.1`) was in progress when this plan was written. It edits the same two
functions. It adds a new module `src/sase/ace/tui/actions/agents/_agent_tab_jump.py`
(`ensure_agent_tab_for_identity`, `restore_agent_tab`, and an owner-level
`_ensure_agent_tab_for`), and calls it:

- inside `_try_reveal_agent_row`, link follow, and the last-launch reveal;
- directly from `handle_jump_to_agent` and `navigate_to_agent_tab`.

Its draft still scans `app._agents` and never reveals, so it does **not** fix this bug.
The landed shape may differ from the draft.

1. Check whether it has landed:
   - `git fetch origin && git log --oneline origin/master --grep='sase-1bc.6.1.4'`
   - `sase bead read sase-1bc.6.1.4 -r "<why>"`
2. **If it has landed:** make sure the workspace contains that commit. If it doesn't,
   fast-forward the clean tree with `git merge --ff-only origin/master`. Then build on
   it.
3. **If it has not landed:** do not edit `_notification_handlers.py`,
   `_notification_navigation.py`, or `_node_jump.py` yet.
   - Use your `/sase_monitor` skill to wait for the commit. Poll `origin/master` every
     ~2 minutes, capped at ~4 hours.
   - Continue from step 2 in the follow-up turn.
   - If the cap expires, or the bead shows the phase was abandoned, proceed on current
     master. Say in your final response that `sase-1bc.6.1.4` must rebase over this
     change.
4. **Exactly one agent-tab switch per jump:**
   - If the landed `_try_reveal_agent_row` already switches tabs and restores the
     previous tab on failure, the ladder covers it. Drop the handlers' own direct
     `ensure_agent_tab_for_identity` / `_ensure_agent_tab_for` calls.
   - Otherwise keep a single switch immediately before the ladder, and restore on
     failure.
   - Keep `sase-1bc.6.1.4`'s cross-tab tests green. Only adjust a test if it asserts the
     removed direct call. The behavior it checks (the tab switched and the row is
     selected) must still hold.
5. If the landed code already routes these callers through the reveal, keep it. Add only
   what is missing from §4 and §5.

## 4. Implementation

All changes are Python and presentation-level TUI navigation. No Rust or `sase-core`
change is needed: this is Agents-tab selection state, not shared backend logic. No
rendered output changes, so there is no golden churn.

### 4.1 `_jump_to_node_identity` takes a subject

In `src/sase/ace/tui/actions/navigation/_node_jump.py`, add a keyword-only
`subject: str = "Node"` to `_jump_to_node_identity`. Thread it into:

- the recursive retry inside `_after_hidden_agents_reload`;
- the final `notify(failure, subject=subject)`.

The default keeps every existing Node Finder message byte-identical.

### 4.2 Shared resolve-and-jump helpers

Add these to `src/sase/ace/tui/actions/agents/_notification_navigation.py`, next to
`navigate_to_agent_tab`. Keep them small and `getattr`-tolerant, like the module's other
helpers.

- **`resolve_loaded_agent(app, predicate) -> Agent | None`.** Returns the first agent
  row satisfying `predicate`, searching these tiers in order:
  1. `app._agents`, the visible projection. Checking it first preserves today's pick
     when several rows match, for example a legacy notification with no `raw_suffix`.
  2. `loaded_real_agent_roster(app)`, the complete loaded roster without clan
     containers.
  3. `getattr(app, "_hideable_agents", ())`, rows hidden by `I`.

  Skip clan containers (`is_clan_container`) in every tier. The function is pure, with
  no I/O.

- **`enter_agents_tab(app) -> None`.** If `app.current_tab != "agents"`, call
  `app._save_current_tab_position()` and then `app._switch_to_tab("agents")`. That is
  the same pair `action_next_tab` uses, and it restores the Agents index so the
  jump-back anchor and the unread "departure" bookkeeping in `_try_reveal_agent_row` see
  the user's last Agents row, not an Artifacts or Services index. Fall back to
  `app.current_tab = "agents"` when `_switch_to_tab` is absent (test doubles).
- **`jump_to_loaded_agent(app, target: Agent) -> bool`.**
  - If `app._jump_to_node_identity` is callable, return
    `bool(jump(target.identity, name=target.display_name, subject="Agent"))`.
  - Otherwise, the legacy harness fallback selects a directly visible `app._agents` row
    with the same identity and returns True.
  - If neither works, return False. The caller warns.

### 4.3 `handle_jump_to_agent`

Keep the signature, the missing-`cl_name` warning, and the matching semantics: exact
`cl_name`; `agent_type` only when present; exact `raw_suffix` only when present. Then:

1. Call `enter_agents_tab(app)`. Today the handler always switches to the Agents tab,
   even on a miss.
2. Call `target = resolve_loaded_agent(app, _matches)`.
3. If `target is None`, keep today's `Agent '<humanized cl_name>' not found` warning and
   return False.
4. Otherwise return `jump_to_loaded_agent(app, target)`. The ladder owns failure toasts,
   worded with the subject `Agent`.

Apply §3.4 for the agent-tab switch.

### 4.4 `navigate_to_agent_tab`

Same shape:

1. Call `enter_agents_tab(app)`.
2. Resolve with the PID predicate across all tiers first, then with the `cl_name`
   predicate. This preserves "PID first" precedence.
3. Call `jump_to_loaded_agent`.
4. Keep the existing not-found warning.

### 4.5 Docs

In `docs/ace.md` § Notification Actions, change the `JumpToAgent` row's behavior to say
that it jumps to the matching Agents-tab row and reveals it first, like the Node Finder:

- expands collapsed folds, grouping banners, and panels;
- shows `I`-hidden rows;
- clears a hiding Agents query with a toast.

Keep the table formatted (`just fmt`).

## 5. Tests

Add `tests/ace/tui/test_notification_jump_to_agent_reveal.py`. Use the production-shaped
harness from `tests/ace/tui/_member_jump_navigation_helpers.py` composed with the
ladder: `class _Harness(NodeJumpNavigationMixin, JumpHarness)`, as in
`tests/ace/tui/test_node_jump_ladder.py`. Stub `_save_current_tab_position` /
`_switch_to_tab` where a case needs them. Build notifications as
`Notification(id=..., timestamp=..., sender="user-agent", action="JumpToAgent", action_data={...})`.
`make_agent` gives every row `cl_name="jump-test"` with a distinct `raw_suffix`, so the
cases also exercise suffix matching.

1. **Collapsed clan member.** Use `make_clan(2)` and target
   `container.runtime_children[1]`, and assert the target is absent from `_agents`
   first. `handle_jump_to_agent` returns True and the target row is selected. No warning
   is emitted, and a jump-back anchor was pushed.
2. **Every layer at once.** Reuse the setup from
   `test_member_jump_reveal_layers.py::test_jump_expands_target_group_and_different_collapsed_panel`:
   collapsed clan fold, collapsed grouping banners, and a collapsed panel. The
   notification jump expands all three and selects the target, and the recorded
   group-fold and panel-fold changes match.
3. **From another tab.** Start with `current_tab = "artifacts"`. After the jump,
   `current_tab == "agents"`, the target is selected, and the saved anchor refers to the
   pre-jump Agents row, not the Artifacts index.
4. **Ambiguous legacy payload.** Two rows share `cl_name`: one visible, one inside a
   collapsed fold. A notification without `raw_suffix` selects the visible one, which is
   today's pick.
5. **Unloaded agent.** An unknown `cl_name` returns False with
   `Agent '<name>' not found`.
6. **Routing.** The handler calls
   `_jump_to_node_identity(target.identity, name=target.display_name, subject="Agent")`.
   A failed reveal (`TARGET_NOT_VISIBLE`) toasts `Agent is no longer visible`.
7. **`navigate_to_agent_tab`.** A PID match on a collapsed clan member reveals and
   selects it, and PID beats a `cl_name` match on a different row.
8. **Node Finder unchanged.** The existing `test_node_jump_ladder.py` cases stay green
   with the default subject `Node`.
9. **Real app, end to end.** Follow
   `test_node_jump_ladder.py::test_real_ace_page_query_clear_retry_records_the_live_transition`:
   `patch_startup_loaders(monkeypatch, agents=[...two clan members...])` plus
   `AcePage(...)` and `wait_for_startup`.
   - Collapse the clan fold, then move to another top-level tab.
   - Dispatch through the real router, `open_notification_action(app, notification)`
     from `actions/agents/_notification_dispatch.py`.
   - Assert that `current_tab == "agents"`, the selected agent's identity is the
     target's, and the clan fold is expanded.

   This is the roadmap's check-it scenario.

`tests/ace/tui/test_notification_dispatch.py` must stay green: the handler name and
routing are unchanged.

## 6. Out of scope

- **Agents not in the loaded roster** (archived, or beyond the bounded history window)
  still report "not found". The Goals G5 card owns the archived-agent fallback.
- **Cross-agent-tab switching** is owned by `sase-1bc.6.1.4`. Only reconcile with it as
  §3 describes.
- **No PNG goldens:** rendering is unchanged, so do not run `just fix-tui-screenshots`.
  Do not run `just check-full`.
- **Other `_agents` scanners** listed in the agent-tabs plan (run-log modal, Files open
  agent, revive select, link-trail restore) belong to `sase-1bc.6.1.4`. File any
  leftover defect you notice there as a `bug` task bead with `/sase_new_task`; do not
  widen this tale.

## 7. Verification

1. Run the new module plus the neighbors first:
   `pytest tests/ace/tui/test_notification_jump_to_agent_reveal.py tests/ace/tui/test_node_jump_ladder.py tests/ace/tui/test_member_jump_reveal_layers.py tests/ace/tui/test_notification_dispatch.py tests/test_notification_agent_targeting.py`.
   Include `tests/ace/tui/test_agent_tab_cross_nav.py` if `sase-1bc.6.1.4` landed it.
2. Run `just fmt`, then `sase tool run check`. If `sase tool` is unavailable, run
   `SASE_TOOL_BYPASS='<why>' just check`. It must pass with no new failures.

## 8. Done when

- A `JumpToAgent` notification for an agent inside a collapsed clan or session fold, a
  collapsed grouping banner, or a collapsed panel opens the Agents tab and selects that
  row, from any top-level tab. The flag-off default is covered by tests 1–3 and 9.
- `'` / `Ctrl+O` returns to the previously selected Agents row, and the jumped-to row's
  unread state is acknowledged, like every other agent-selecting jump.
- The Runners-modal agent jump behaves the same way.
- Query-hidden and `I`-hidden targets go through the Node Finder ladder with its
  existing toasts. An agent that isn't loaded still warns "not found".
- The `JumpToAgent` docs row is updated, `sase tool run check` passes, and there are no
  golden changes.
