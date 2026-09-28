---
tier: tale
title: "Land epic sase-1bc.6.1.6: close the agent-tabs repair gaps and close the epic"
goal:
  Every defect the sase-1bc.6.1.6 landing review found in the agent-tabs repairs is
  fixed or proven absent, and every cross-tab entry point has flag-on and flag-off
  tests. The oversized cross-nav test module is split under the toobig limits. Epic
  sase-1bc.6.1.6 is closed with a verification note, symvision is clean, and its plan
  file is marked done. With `agent_tabs` off, the TUI behaves as it did before.
size: medium
proposed_by: bbugyi200.athena.0tf
create_time: 2026-09-28 06:18:52
status: wip
---

# Land epic sase-1bc.6.1.6: finish the agent-tabs repair gaps, then close it

## Context

Epic `sase-1bc.6.1.6` (plan `plan:202609/agent_tabs_scope_repairs.md`) repaired the
flag-gated `agent_tabs` feature. All three phases are closed:

- `sase-1bc.6.1.6.1` (commit `dae0f6efad`)
- `sase-1bc.6.1.6.2` (commit `3ba7f3b22f`)
- `sase-1bc.6.1.6.3` (commit `d094fe70ee`)

The epic's land agent (`sase-1bc.6.1.6.land`) is still stuck in WAITING, so this landing
was done by hand. The landing review has already confirmed the following:

- **Verified in source.** Every phase-1 item and every phase-3 item is in the tree, and
  every phase-2 code change is too.
- **Symvision.** It reports every symbol used on master.
- **Epic symbols.** `sase bead epic-symbols sase-1bc.6.1.6` lists no entries, and the
  Justfile has no `sase-1bc` `--epic-symbol` line.
- **Follow-ups already handled.** Every `PROPOSED FOLLOW-UP:` was triaged and recorded
  in a note on `sase-1bc.6.1.6`. Two defects from active epic `sase-1bt` were recorded
  there as `DISCOVERED ISSUE` notes:
  - the Tools pane `Open Agent` name/identity mismatch;
  - full tool-run chip rebuilds.

  **Do not fix those two here.** `sase-1bt` phases are editing those files right now.

Only the epic-caused gaps below remain. Items 1–5 are small code fixes. Item 6 is the
test coverage that phase 2's "Done when" required but did not fully deliver. Item 7
closes the epic.

Rules:

- **Flag gating.** With `agent_tabs` off, behavior must match the pre-epic TUI. Keep Off
  branches explicit.
- **Tests.** Cover both flag states, and prefer the real mixins over stubbed reveal
  helpers.
- **Flag in the agent shell.** The agent shell may export `SASE_FEATURE_FLAGS` with
  `"agent_tabs":false`, which beats `override_flags`. Tests must not rely on the shell's
  value.
- **Concurrent split.** A concurrent toobig split agent may move the code in
  `src/sase/ace/tui/actions/agents/_agent_tabs.py` into sibling modules before you
  start. Locate functions by name (`grep -rn "def _restore_tab_memory" src/sase`), not
  by file or line.

Paths are repo-relative. Line numbers are approximate.

## Changes

### 1. Link-trail restore must not strand the user on the Agents tab

- **Bug.** In `src/sase/ace/tui/actions/link_trail.py`, `_restore_agents_link_trail_hop`
  now switches `current_tab` to `"agents"` before calling `_try_reveal_agent_row`. When
  the reveal fails, it returns False and leaves the user on the Agents tab.
- **Regression.** Before `3ba7f3b22f`, the top-level tab switched only after a match was
  found, so this changes flag-off behavior too.
- **Fix.** Remember the top-level tab before switching. If the reveal is unavailable or
  returns a failure, set `current_tab` back to that tab before returning False, but only
  when this method switched it.
- **Tests.** Cover both flag states: a failed reveal leaves `current_tab` unchanged, and
  a successful reveal lands on `"agents"` with the row selected.

### 2. First visit to a tab must not inherit the source tab's panel focus

`_restore_tab_memory` restores a saved panel only when the target tab has memory for it.
On a first visit, or when the saved panel no longer exists, it leaves two things
untouched:

- `_panel_group.focused_idx`
- `_expanded_panel_focus`

`_sync_panel_group` (`src/sase/ace/tui/actions/agents/_display_panel_collection.py`)
keeps the previous focused key whenever the new tab also has that key. So tribe-panel
focus from tab A can carry into a first visit of tab B, even though the plan requires a
first visit to select row 0.

Do these steps in order:

1. Write a real-mixin test first. Tab A has two tribe panels, and focus is on the second
   one with expanded panel focus on. Switch to a never-visited tab B that has the same
   two tribe panels. Assert that row 0 is selected, the focused panel is the one that
   contains row 0, and `_expanded_panel_focus` is False.
2. If the test fails, fix `_restore_tab_memory` for the no-memory and panel-gone cases:
   - focus the panel that holds the restored row;
   - clear `_expanded_panel_focus`.

   Keep the existing restore path for a saved panel that still exists.

3. If the test passes on the current code, keep it as a regression test and change
   nothing.

### 3. Tab-index memo snapshot must not trust recycled ids

- **Bug.** `cached_agent_tab_index` (`src/sase/ace/tui/models/agent_tab_index.py`)
  snapshots membership as `tuple(id(row) for row in roster)`. The cache entry keeps the
  roster list alive, but not the rows that were in it at build time. If a row is removed
  in place and a new row reuses its id, the snapshot can match and return a stale index.
- **Fix.** Store the snapshot as a tuple of the row objects (strong references) and
  compare it element-wise with `is`. Keep these unchanged:
  - the two-entry cap;
  - the `hit[0] is roster` check;
  - the docstring's retention guarantee (update its wording to match).
- **Tests.** The existing tests in `tests/ace/tui/models/test_agent_tab_index.py` must
  still pass. Add one test that pops a row, inserts a different row object at the same
  position, and asserts the memo misses.

### 4. Honest legacy bracket-yield warning

In `src/sase/ace/tui/keymaps/registry.py` (the `legacy_card_block_brackets` block), the
`Card-block stepping moved from [ / ]` warning always claims the bracket overrides "are
honored" and that "agent tab cycling yields the brackets".

That is false when the colliding tab action is explicitly configured (for example
`next_agents_tab: "]"`). In that case the tab keeps the bracket, and the later
duplicate-binding pass reverts the card-block override.

**Fix.** Emit the warning only after the collision and duplicate resolution:

- name only the card-block actions that still hold a bracket;
- mention the tab yield only for tab actions that were actually unbound;
- log the revert when a card-block override is reverted.

Keep the key-binding outcomes exactly as they are. Extend
`tests/ace/tui/widgets/decks/test_deck_card_block_keys.py` to assert the warning text
(use `caplog`) for two cases:

- the plain legacy override;
- an explicit default-valued tab binding.

### 5. Pyright narrowing nit (folded-in follow-up from sase-1bc.6.1.6.2)

In `tests/test_agent_revive.py`, around lines 313–317 and 402–406,
`assert delta is not False` does not narrow `AgentReviveDelta | bool`. Replace it with
`assert isinstance(delta, AgentReviveDelta)`, importing the class if needed. This is a
test-only change.

### 6. Missing cross-tab entry-point tests, and splitting the oversized test module

This epic grew `tests/ace/tui/test_agent_tab_cross_nav.py` to 1009 lines, which is over
the toobig limit of 1000.

**Split first.**

- Move shared fixtures into a new helper module,
  `tests/ace/tui/_agent_tab_cross_nav_helpers.py`. These are `_view`, `_row`,
  `_TabOwner`, `_two_tab_owner`, `_cross_tab_harness_class`, `_AnchorOwner`, and
  `_finder_row`, each given a non-underscore name so it can be imported.
- Move the existing entry-point tests into one or more new modules. These are the
  revive, Files, notification, run-log, last-launch, and Procs tests. Suggested names
  are `test_agent_tab_cross_nav_entry_points.py` and
  `test_agent_tab_cross_nav_modals.py`.
- Every touched or new test file must end under 850 lines, the toobig warning threshold.
- Check with `.venv/bin/toobig tests 1000 850 700 | grep agent_tab`, which must print no
  WARNING or VIOLATION lines for these files.

**Then add these tests.** Each drives the real mixins, with the target row on a
non-active agent tab. Flag on: assert the agent tab switched and the row is selected.
Flag off: assert the pre-epic outcome, driving the real `_try_reveal_agent_row` rather
than a stub.

| Entry point                                                                                                                     | What is missing today                                                                                                                                                                                          |
| ------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Link follow, `_reveal_loaded_agent` (`src/sase/ace/tui/actions/_link_follow_targets.py`)                                        | Flag-on and flag-off tests.                                                                                                                                                                                    |
| Link-trail restore, `_restore_agents_link_trail_hop`                                                                            | Flag-on success test. The flag-off test must use the real reveal. Item 1 adds the failure cases.                                                                                                               |
| Relation-jump activation, `_activate_member_jump_target` (`src/sase/ace/tui/actions/navigation/_member_jump.py`)                | Flag-on cross-tab test and flag-off test.                                                                                                                                                                      |
| Procs monitor jump, `action_open_monitor_agent` → `_jump_to_monitor_agent` (`src/sase/ace/tui/modals/procs_pane_agent_jump.py`) | End-to-end jump in both flag states, not only the `_monitor_jump_agent` lookup.                                                                                                                                |
| `_reveal_last_launch_target` (`src/sase/ace/tui/actions/agent_workflow/_kill_last_launch.py`)                                   | Successful cross-tab reveal; reveal-returned-failure restoring the tab (only the raise path is tested today); and a flag-off test.                                                                             |
| Files open agent (`_select_file_agent`, `src/sase/ace/tui/actions/artifacts_files.py`)                                          | Flag-off test through the real reveal.                                                                                                                                                                         |
| `,j` unread (`src/sase/ace/tui/actions/agents/_unread_navigation.py`)                                                           | Action-level off-tab jump. Also a test that `_unread_timed_jump_candidates` (`_unread_jump_candidates.py`) recomputes after an agent-tab switch, since the cache key includes `current_agent_tab_scope_token`. |
| Failed `_try_reveal_agent_row`                                                                                                  | Assert that both anchor stacks (`_entry_jump_agents_anchor_stack` and `_entry_jump_agents_forward_anchor_stack`) are restored as well as the tab.                                                              |
| Node Finder off-tab chip                                                                                                        | Rendering test: a row with `tab_label` renders `[<label>]` through `src/sase/ace/tui/modals/node_finder_rendering.py`, and a row without one does not.                                                         |

## Verification

1. Run the agent-tab suites and the touched suites directly:

   ```bash
   .venv/bin/python -m pytest -q \
     tests/ace/tui/test_agent_tab_*.py \
     tests/ace/tui/models/test_agent_tab_index.py \
     tests/ace/tui/widgets/decks/test_deck_card_block_keys.py \
     tests/ace/tui/test_link_trail.py \
     tests/test_agent_revive.py
   ```

   Also run the new cross-nav modules. If this workspace's venv is stale, run
   `just install` first through `/sase_monitor`, because it rebuilds the Rust binding.

2. Run `sase tool run check`, never `check-full`. These failures are known,
   pre-existing, and tracked elsewhere. Do not fix them:
   - `test_no_system_clock_display_sites` (`sase-1bp`);
   - `tests/test_axe_run_agent_exec_repeat_env.py` (`sase-1bc` note #3);
   - `tests/completion/test_snapshot.py` (`sase-18s`);
   - `tests/ace/tui/test_agent_completion.py` and the two directive-completion absence
     tests (`sase-1bc` notes #1/#2);
   - `test_expanded_overflowing_header_claims_half_page_scroll` (`sase-1b8`).

   Treat any other new or unknown failure as yours.

3. No rendered TUI output changes, so no PNG golden run is needed. If you touch
   rendering anyway, run a targeted `just fix-tui-screenshots -- <selector>` through
   `/sase_monitor` and inspect every change. Flag-off goldens must not change.

## Closeout (final step)

1. Run `sase bead epic-symbols sase-1bc.6.1.6`. For every listed entry, either resolve
   the symbol or re-key it to a still-open bead. None are expected.
2. Close the epic:

   ```bash
   sase bead close sase-1bc.6.1.6 --note "<verification>"
   ```

   The note should say:
   - all three phases were verified in source (`dae0f6efad`, `3ba7f3b22f`,
     `d094fe70ee`);
   - which of items 1–6 landed, including whether item 2 needed a fix;
   - the symvision result;
   - the `sase tool run check` run id and its known-only failures;
   - that follow-up triage is recorded in the epic's LANDING TRIAGE note;
   - that phase 2 deliberately routes run-log, revive, Files, and link-trail through
     `_try_reveal_agent_row` in both flag states, per the epic plan's "no worse than
     before" rule.

   Never use `--force`.

3. Run `just _lint-symvision` (or `just symvision`) and confirm it prints
   `All public/private classes/functions are used properly!`.
4. In the plans sidecar (the `PLAN` path printed by
   `sase bead read sase-1bc.6.1.6 -r "<why>"`), change `status: wip` to `status: done`
   in the frontmatter of `202609/agent_tabs_scope_repairs.md`.
5. The epic's parent is `sase-1bc.6.1`, a plan bead whose landing this repair epic
   interrupted.
   - **Do not close `sase-1bc.6.1`.** The request covered only `sase-1bc.6.1.6`, and the
     parent landing needs its own review.
   - Append a note on the parent:
     `sase bead note sase-1bc.6.1 "Child repair epic sase-1bc.6.1.6 landed and closed (<commit>); the sase-1bc.6.1 landing can resume."`
   - Say this in the final response.

## Out of scope

- The two `sase-1bt` defects recorded on that epic (Tools pane `Open Agent`, and chip
  rebuilds in `tool_runs/loader.py`).
- The pre-existing failures listed under Verification.
- Closing `sase-1bc.6.1` or any other ancestor.
- Killing or relaunching the stuck `sase-1bc.6.1.6.land` agent.
