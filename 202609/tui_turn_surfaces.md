---
tier: tale
title: TUI turn surfaces
goal: "Agents-tab modules, row kinds, section ids, and visible copy use sase-turn and
  named-proc vocabulary, the PNG goldens match that copy, and j/k navigation stays on
  the existing performance contract.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1ab.4
bead: sase-1ab.4
create_time: 2026-09-26 14:03:30
status: wip
---

- **PARENT:**
  [202609/sase_turn_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/sase_turn_rename.md)
- **BEAD:**
  [sase-1ab.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ab/sase-1ab.4.md)

# Plan: TUI turn surfaces

Implement phase `sase-1ab.4` of epic `sase-1ab` (plan `plan:202609/sase_turn_rename.md`,
section "TUI turn surfaces"). The parent epic already split the rename. This tale is the
whole phase: one agent, sase repo only. Do not open linked repos. Do not create beads.
Do not close `sase-1ab` or any ancestor. Record leftovers with
`sase bead note sase-1ab.4 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`.

Read `tui.md`, `tui_screenshot.md`, `tui_perf.md`, and `lint_and_test.md` through
`sase memory read` before editing. `just check` does not run PNG snapshots. Do not run
`just check-full`.

## Outcome

A reader of the Agents tab sees `SESSION TURNS`, kind headers `AGENT TURN`, `GATE TURN`,
`MONITOR TURN`, and `NAMED PROC`, footer noun `turn`, and help/modal/notification copy
from the vocabulary below. Python identifiers in the TUI match those words. Pixels that
change are the goldens. Navigation and render handlers gain no disk I/O, awaits,
subprocesses, or full list rebuilds.

## Vocabulary

Apply this only to the session-member concept and to stand-alone named procs.

| Current                                                  | Replacement                                                                         |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| agent shell, `AGENT SHELL`                               | agent turn, `AGENT TURN`                                                            |
| gate shell; kind header `GATE` on a gate row             | gate turn, `GATE TURN`                                                              |
| monitor shell; kind header `MONITOR` on a monitor row    | monitor turn, `MONITOR TURN`                                                        |
| stand-alone proc shell, `PROC SHELL`                     | named proc, `NAMED PROC`                                                            |
| session shells, `SESSION SHELLS`, `Shells: `, `N shells` | session turns, `SESSION TURNS`, `Turns: `, `N turns`                                |
| footer `("0-9", "shell")`, jump noun `shell`             | `("0-9", "turn")`, noun `turn` (`No turn N`, `Turn roster changed; jump cancelled`) |

When one string covers both a stand-alone proc and a session monitor, split it. Example:
`_wait_helpers.py` today says "A proc shell can be forked…" for
`is_proc_shell or is_monitor`. After the rename, a named proc and a monitor turn each
get their own sentence.

`GATE` and `MONITOR` change only where they are the row kind header (identity header,
agent display header, Node Finder kinds). Leave `MONITOR_GLYPH`, monitor gear colors,
and the word "monitor" in non-header prose.

## Scope

Edit:

- `src/sase/ace/tui/**`
- `src/sase/ace/testing/**`
- `tests/ace/**`
- `tests/perf/**` only when a comment or name is the concept (today's hits in
  `tests/perf/bench_phase7_e2e.py` and `tests/perf/bench_bead.py` are Unix-shell
  benchmarks; leave them)
- the comment in `src/sase/default_config.yml` that says a running serial session
  "shares one claim across its live shells" (about line 63)

Also update importers of renamed TUI symbols that live outside those trees. Known today:
`tests/test_agent_loader_pending_gate_turn.py` calls `shell_lane_counts`. After the
renames, search again for the old module paths and symbol names.

Leave untouched:

- `docs/`, `README.md`, the blog, `sase/memory/**`, glossary strands, generated
  `AGENTS.md` copies, and decision records. `docs-memory` owns those. `docs/ace.md`
  still says `SESSION SHELLS` on purpose.
- Generated `CHANGELOG.md`.
- `src/sase/procs/models/proc.py` property aliases `shell_name` and `shell_kind`. Stop
  calling them from the TUI; the aliases stay until a later phase.
- `capacity_session_keys_for_core` legacy kwargs `shell_kind`, `shell_id`, `shell_state`
  in `src/sase/core/runner_slots/_admission_capacity_records.py`. Switch the TUI caller
  to `turn_kind`, `turn_id`, and `turn_state`.
- `AgentType._missing_` accepting the value `"proc-shell"`. Tests must construct
  `AgentType.NAMED_PROC`. `AgentType.PROC_SHELL` is already gone, so `tests/ace`
  references to it fail until this phase updates them.
- Fleet wire fallbacks that already read both spellings: `historical_shell` beside
  `historical_turn` in `_fleet_agents_nodes.py` and `_fleet_agents_identity.py`, and
  `exact_locator.get("shell_id")` beside `turn_id` in `_fleet_agents_nodes.py`. Core
  still emits the legacy spelling until `contract-flip`.
- Unix shells, shell completion, the TUI `!` command and "Enter shell command…",
  custom-mode `shell:` keys (`keymaps/registry.py`, `actions/custom_modes.py`),
  `util/code_injection.py` language id `"shell": "bash"`, and sudo `"shell"` risk
  badges.
- Artifacts-pane chrome: `widgets/artifacts/shell.py`, `PaneCapability.SHELL`,
  `_artifact_tab_model.py` `SHELL = "shell"`, `tests/ace/tui/test_artifacts_shell.py`,
  `.artifacts-shell-*`, `.gate-review-shell`, and "loading shell" panel docstrings.
- Command Line "panel shell" tests: `tests/ace/tui/command_line/test_panel_shell.py`.
- Key bindings. `next_card_block` and `prev_card_block` in
  `commands/_app_metadata_nav.py` bind the key `shell` next to `block`, `reply`, `card`,
  and the bracket keys. That token is a binding. Leave it, and leave
  `default_config.yml` keymaps and `sase.schema.json` alone. The schema's `shell`
  descriptions are Unix-shell text.
- Provider and conversation turn names: `turn_nonce`, `num_turns`, `parse_chat_turns`,
  `extract_previous_conversation_turns`.
- `__pycache__` directories.

Do not substring-replace `shell` with `turn`. `turn` is inside `return`. Classify every
remaining `\bshell` hit.

## File renames

Use `git mv` for these modules and their tests. Update `__init__` re-exports in the same
edit (`modals/__init__.py`, `modals/_export_table.py`, and any `models` or `widgets`
barrel).

| From                                                                 | To                                              |
| -------------------------------------------------------------------- | ----------------------------------------------- |
| `src/sase/ace/tui/actions/agents/_proc_shell_dismiss.py`             | `_named_proc_dismiss.py`                        |
| `src/sase/ace/tui/models/agent_proc_shells.py`                       | `agent_named_procs.py`                          |
| `src/sase/ace/tui/models/_agent_session_shell_membership.py`         | `_agent_session_turn_membership.py`             |
| `src/sase/ace/tui/widgets/prompt_panel/_agent_proc_shell_section.py` | `_agent_named_proc_section.py`                  |
| `src/sase/ace/tui/widgets/prompt_panel/_agent_shell_section.py`      | `_agent_turn_section.py`                        |
| `tests/ace/tui/models/test_agent_proc_shells.py`                     | `test_agent_named_procs.py`                     |
| `tests/ace/tui/test_proc_shell_dismissal.py`                         | `test_named_proc_dismissal.py`                  |
| `tests/ace/tui/test_proc_shell_selection_survives_refresh.py`        | `test_named_proc_selection_survives_refresh.py` |
| `tests/ace/tui/widgets/test_agent_proc_shell_section.py`             | `test_agent_named_proc_section.py`              |
| `tests/ace/tui/widgets/test_agent_shell_section.py`                  | `test_agent_turn_section.py`                    |
| `tests/ace/tui/visual/_ace_agents_proc_shell_png_fixtures.py`        | `_ace_agents_named_proc_png_fixtures.py`        |
| `tests/ace/tui/visual/test_ace_png_snapshots_agents_proc_shells.py`  | `test_ace_png_snapshots_agents_named_procs.py`  |

Do not rename `widgets/artifacts/shell.py`, `test_artifacts_shell.py`, or
`test_panel_shell.py`.

## Identifier renames

Rename the concept in comments, docstrings, log and error messages, test names, and
fixtures together with the code. Do not keep alias modules or deprecated wrappers to
shrink the diff.

In `models/agent_session_members.py`:

- `ShellLaneCounts` → `TurnLaneCounts`
- `NO_SHELL_LANES` → `NO_TURN_LANES`
- `_ShellLaneTally` → `_TurnLaneTally`
- `row_is_agent_session_shell` → `row_is_agent_session_turn`
- `shell_lane_counts` → `turn_lane_counts`
- `panel_shell_lane_counts` → `panel_turn_lane_counts`
- `concrete_agent_session_shell_rows` → `concrete_agent_session_turn_rows`
- `current_agent_session_shell_row` → `current_agent_session_turn_row`
- `_AgentSessionShellAnchors` and `_agent_session_shell_anchors` → the `Turn` forms
- `shell_counts` → `turn_counts`

In `widgets/prompt_panel/_agent_turn_section.py` (after the move):

- `SHELL_SECTION_ID = "shells"` → `TURN_SECTION_ID = "turns"`
- `SHELL_FIELD_LABEL = "Shells: "` → `TURN_FIELD_LABEL = "Turns: "`
- `SHELL_LANE_LIMIT` and the `_SHELL_*` style constants → `TURN_*` / `_TURN_*`
- `_AgentShellLane`, `_MonitorShellLane`, `_GateShellLane`, `ShellLane` →
  `_AgentTurnLane`, `_MonitorTurnLane`, `_GateTurnLane`, `TurnLane`
- `ResponsiveShellSection` → `ResponsiveTurnSection`
- `build_agent_session_shell_lanes` → `build_agent_session_turn_lanes`

In `_agent_named_proc_section.py`:

- `PROC_SHELL_SECTION_ID = "proc-shell"` → `NAMED_PROC_SECTION_ID = "named-proc"`
- `_proc_output_source_id` returns `named-proc:{proc_id or identity}` (today
  `proc-shell:…`, passed only into `render_axe_output`)

In `models/agent.py`:

- `is_proc_shell` → `is_named_proc`
- display fallback `"proc shell"` → `"named proc"`
- display fallback `"gate shell"` → `"gate turn"`
- docstring on `is_gate` that says "gate-shell member" → "gate-turn member"

Style constants in `widgets/_agent_list_styling.py`: `_PROC_SHELL_GLYPH`,
`_PROC_SHELL_GLYPH_STYLE`, `_PROC_SHELL_ROW_STYLE`, `_PROC_SHELL_ID_STYLE` →
`_NAMED_PROC_*`. Update every importer (`_identity_header.py`,
`_agent_display_header.py`, `_identity_header_compact.py`,
`_agent_display_header_metadata.py`, `_agent_list_render_agent.py`,
`_agent_list_render_agent_prefix.py`, `_metadata_pager_document.py`,
`models/node_finder.py`). `agent_info_panel.py` `_PROC_SHELL_BADGE_STYLE` and
`_proc_shell_count` → `_NAMED_PROC_BADGE_STYLE` and `_named_proc_count`. The visible
glyph stays `⚙`; the badge does not gain the words "named proc".

Other concept names already in tree:

- `aggregates_agent_session_shells` and `include_monitor_shells` in
  `models/_agent_time_aggregate.py` and `models/agent_time.py` →
  `aggregates_agent_session_turns`, `include_monitor_turns`
- `is_shell` keyword and `_shell_pair_status` in `models/_fleet_agents_rows.py` →
  `is_turn`, `_turn_pair_status`. The flag means "this remote row is a monitor or gate
  member."
- `AgentSessionShellFacts`, `agent_session_shell_facts` in
  `_agent_display_agent_session.py` → `AgentSessionTurnFacts`,
  `agent_session_turn_facts`
- `block_meta_for_session_shell` in `_agent_session_reply_blocks.py` →
  `block_meta_for_session_turn`. Leave the `shell` key binding alone.
- `_is_untitled_sase_shell` in `models/_agent_tree.py` → `_is_untitled_sase_turn`
- `_is_agent_shell` in `modals/node_finder_preview.py` and
  `node_finder_preview_loader.py` → `_is_agent_turn`
- `ConfirmKillProcShellModal` → `ConfirmKillNamedProcModal`, title `Kill Named Proc`,
  body "Kill this named proc? Running command will stop immediately."
- `proc_shell_count_phrase` → `named_proc_count_phrase`, nouns `named proc` and
  `running named proc`
- `partition_proc_shells`, `ProcShellDismissMixin`, `_dismiss_proc_shell_rows` →
  named-proc forms
- `_do_kill_proc_shell` and the producer-site strings in
  `_proc_producer_sites_actions.py` (`"_do_kill_proc_shell"`,
  `"MonitorStopActionFlowMixin._do_kill_proc_shell"`) → `_do_kill_named_proc` and the
  matching qualified string
- worker group `"proc-shell-dismiss"` → `"named-proc-dismiss"`
- in-memory set `_dismissed_proc_shells` → `_dismissed_procs`, and
  `_schedule_dismissed_proc_shells_startup_prune` /
  `_run_dismissed_proc_shells_startup_prune` → the `dismissed_procs` names.
  `src/sase/ace/testing/_startup.py` patches the schedule method by name; update that
  patch. The on-disk migration to `~/.sase/dismissed_procs.json` already lives in
  `src/sase/ace/dismissed_procs.py`. Do not rewrite it.
- `ObservedProc.shell_name` / `shell_kind` in `_proc_observer_models.py`, the
  assignments in `_proc_observer_store.py`, and the reads in `agent_named_procs.py` →
  `proc_name` / `proc_role`, fed from `Proc.proc_name` / `Proc.proc_role`
- `is_monitor_shell_row` → `is_monitor_turn_row`
- `_SHELL_STATUS_PRESENTATION_FIELDS` in `_agent_clan.py` →
  `_TURN_STATUS_PRESENTATION_FIELDS`
- `SETTLED_AGENT_SESSION_SHELL_DONE_OUTCOMES` in `_loaders/_workflow_loaders.py` →
  `SETTLED_AGENT_SESSION_TURN_DONE_OUTCOMES`
- `_agent_session_shell_kind` / `_id` / `_state` in `_agent_runner_slot_capacity.py` →
  `_agent_session_turn_kind` / `_id` / `_state`, passed as `turn_kind`, `turn_id`,
  `turn_state`
- `_SHELL_ROLES` in the membership module → `_TURN_ROLES`. Keep the role tokens `plan`,
  `code`, `gate`, `monitor`, `proc`, and `member`. `proc` here is a legacy member-role
  token that still means a monitor member. Read `agent_session_role_for_suffix` before
  changing any token. Do not rename the token `proc` to `monitor` inside that frozenset.

Attach every renamed public name to the module `__all__` that already exported the old
one.

## Visible copy

Set these strings. Update the tests that assert them (`test_identity_header.py`,
`test_identity_header_compact.py`, `test_agent_jump_panel_numbers.py`,
`test_agent_jump_panel_visibility.py`, the renamed section tests, node-finder tests, and
any other assertion the suite reports).

- `_agent_display_agent_session.py`: `_AGENT_SESSION_ROSTER_TITLE = "SESSION TURNS"`,
  `hidden_tail_label="turns"`
- `_agent_display_neighbors.py`: `… +N also listed under SESSION TURNS`
- `_agent_turn_section.py`: `… +N more turns (see SESSION TURNS)`
- `_identity_header_compact.py`: `f"{total} turns"`
- `_identity_header.py` and `_agent_display_header.py`: kind headers `NAMED PROC` and
  `AGENT TURN` (today `PROC SHELL` and `AGENT SHELL`)
- `models/node_finder.py` (both kind-label sites): `NAMED PROC`, `AGENT TURN`,
  `GATE TURN`, `MONITOR TURN`
- `modals/node_finder_preview.py`: section `TURNS`, empty text `No loaded turns.`
- `widgets/_keybinding_bindings_agents.py`: the three `("0-9", "shell")` footer entries
  → `("0-9", "turn")`
- `actions/navigation/_member_jump.py`: `_roster_jump_noun` returns `"turn"` for an
  agent-session roster. That one return produces `No turn N` and
  `Turn roster changed; jump cancelled`. Clan and neighbor nouns stay `member` and
  `neighbor`.
- `modals/help_modal/agents_bindings.py`: `Monitor turn, running`,
  `Monitor turn, finished (grey)`, `Gate turn, pending/running (cyan)`,
  `Gate turn, settled (grey)`, `Gate turn, failed (red)`. The `N running monitors` lines
  stay; they already say monitors.
- `modals/confirm_kill_modal.py`: named-proc title and body above.
  `ConfirmCancelGateModal` visible title stays `Cancel Gate`. Its fallback description
  `"gate shell"` in `_monitor_stop_flow.py` becomes `"gate turn"`. Notifications
  `Cannot resolve proc shell id` and `Proc shell has already finished` become
  `Cannot resolve named proc id` and `Named proc has already finished` on the
  stand-alone proc path.
- Dismiss copy in the named-proc dismiss module: `Dismissed N named procs`,
  `No finished named procs to dismiss`, and the save-failure sentence names dismissed
  named procs.
- `_artifact_tab_descriptions.py` agents pane: "sessions and their turns".
- `modals/statistics_pane_legends.py` Runner legend: "the live turn holding a sase
  agent's slot".
- `modals/statistics_help_modal.py`: "as long as any of its turns is live".
- `default_config.yml` capacity comment: "its live turns".

`ConfirmStopMonitorModal` already says "Stop Monitor". Leave that title.

## Legacy ids

`SHELL_SECTION_ID` and `PROC_SHELL_SECTION_ID` are keys in the in-memory
`SectionFoldStateManager` (`models/fold_state.py`). `ace_agents_fold_state.json`
persists Agents-tab group folds (`models/agent_fold_persistence.py`), and those keys are
grouping paths, not `"shells"` or `"proc-shell"`. The `proc-shell:` prefix is a
render-time axe source id.

Before adding a legacy reader, search `src/sase/ace` for writers of the literals
`"shells"`, `"proc-shell"`, and `"proc-shell:"`. Add a reader only where one of those
values is actually stored on disk. Do not add a startup migration for an in-memory fold
key.

## Performance

Renames only. Read `tui_perf.md` and keep navigation and render handlers free of
filesystem work, awaits, subprocesses, and full agent-list rebuilds. Do not insert a
compatibility wrapper on a keypress path.

After the code rename and before closing, run the existing j/k bench:

```bash
pytest -s -m slow tests/ace/tui/bench_tui_jk.py
```

Use `sase monitor start -p verify` when that bench will outlast the turn. The product
target in `tui_perf.md` is p95 under 16 ms on every tab. If the bench fails the same way
on the clean base tree, note a `PROPOSED FOLLOW-UP:` citing any bead that already tracks
it, and continue. A regression introduced by this rename keeps the bead open until the
extra work is gone.

## PNG goldens

`git mv` these seven files in the same change as the `snapshot_name` strings that load
them, so the harness treats them as the same scenes:

| From                                                               | To                                                     |
| ------------------------------------------------------------------ | ------------------------------------------------------ |
| `tests/ace/tui/visual/snapshots/png/agents_proc_shells_120x40.png` | `agents_named_procs_120x40.png`                        |
| `agents_proc_shells_90x30.png`                                     | `agents_named_procs_90x30.png`                         |
| `agents_proc_shell_detail_120x40.png`                              | `agents_named_proc_detail_120x40.png`                  |
| `agents_session_panel_shells_gate_120x40.png`                      | `agents_session_panel_turns_gate_120x40.png`           |
| `agents_session_panel_shells_gate_90x40.png`                       | `agents_session_panel_turns_gate_90x40.png`            |
| `agents_session_panel_shells_monitor_120x40.png`                   | `agents_session_panel_turns_monitor_120x40.png`        |
| `agents_session_panel_shells_monitor_roster_120x40.png`            | `agents_session_panel_turns_monitor_roster_120x40.png` |

The gate names are referenced from
`test_ace_png_snapshots_agents_agent_session_panel_gate.py`. The monitor names are
referenced from `test_ace_png_snapshots_agents_agent_session_panel_monitor.py`. Update
those strings and the window titles that name the concept, including:

- `ACE agents stand-alone proc shells` → `ACE agents stand-alone named procs`
- `ACE agents proc-shell detail` → `ACE agents named-proc detail`
- `ACE session panel shell metadata with gate rows`
- `ACE session panel gate shells narrow`
- `ACE selected gate shell with long output`
- `ACE session panel shell metadata with monitor`
- `ACE session panel SESSION SHELLS roster with monitor`
- `ACE session panel pending shell digit` in
  `test_ace_png_snapshots_agents_agent_session_panel.py`

Leave the `ACE` prefix on those titles. Rewriting every golden title from ACE to TUI
would churn snapshots whose concept copy did not change.

Then re-baseline. Run `just fix` first. Capture with the monitor skill, profile
`verify`, status pair `TESTING` / `TESTED`:

```bash
sase monitor start -p verify --timeout 90m \
  --reason 'Re-baseline TUI goldens after the turn-surface rename' \
  --next 'Inspect the visual report and finish sase-1ab.4.' \
  -- just fix-tui-screenshots
```

Expect on the order of 150–200 Agents-tab PNG updates because `SESSION TURNS`, the kind
headers, the footer, and the shell-count chip change pixels. Pager goldens under
`tests/pager/visual/snapshots/png/` should stay put unless a shared string rendered
there.

Read `.pytest_cache/sase-visual/latest-report.json` and the WARNING block. Status
`partial` means skipped goldens are not current; look at `skipped` and
`pruning_skipped_reason`, then rerun until the required Agents-tab scenes are applied.
Inspect every creation, every removal, and each update group. Expand a group whose diff
is something other than the renamed copy (layout shift, missing row, color change,
clipped text). Delete a stale golden only after one full run, and only when the report
shows it is unreferenced. Generation is not approval.

A check or screenshot failure that reproduces on the clean base tree does not keep this
bead open. Record it as `PROPOSED FOLLOW-UP:` and close anyway.

## Exit

1. `rg -n -w shell src/sase/ace/tui src/sase/ace/testing tests/ace` — every remaining
   hit is artifacts-pane chrome, a modal frame, a Unix shell, a `!` command, completion,
   a key binding, a language id, or a named legacy reader (`historical_shell`, locator
   `shell_id`, the role token `proc` inside `_TURN_ROLES`).
2. `rg -n -w 'turns?' src/sase/ace/tui src/sase/ace/testing tests/ace` — each new "turn"
   is a sase turn or an existing provider/conversation turn. No surface shows
   `num_turns` as turns.
3. The j/k bench shows no regression from this rename.
4. The screenshot report was inspected, and required goldens match the new copy.
5. `sase tool run check` passes. Run `just install` first if this workspace's virtualenv
   is stale. A pre-existing failure that matches the clean base is a
   `PROPOSED FOLLOW-UP:`, not an open bead.
6. `sase bead epic-symbols sase-1ab.4`. The Justfile symvision entries today are
   `sase-18i(CoderPlacement)`, `sase-18i(RetiredGate)`, and
   `sase-19i.7.3.3.2(describe_node_finder_row_from_facts)`. They are not this phase's
   symbols. If this rename leaves a new symvision allowlist need, fix the symbol. Re-key
   a Justfile `--epic-symbol` line to a still-open bead (parent `sase-1ab` or a later
   phase such as `sase-1ab.7`) only when the symbol truly belongs to later work.
   `sase bead close` refuses while a symbol still names `sase-1ab.4`.
7. Close only this bead: `sase bead close sase-1ab.4 --note "<what you verified>"`. The
   note names the check command, the screenshot report status, and the j/k bench result.

The host commit message is a `feat` for the TUI vocabulary. Use `feat!:` with a
`BREAKING CHANGE:` footer only if a keymap or config value actually changes. None is
expected.
