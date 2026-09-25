---
tier: epic
title: Finish ACE agent session surfaces (ace-cutover landing gaps)
goal: 'The ACE agent-session cutover is complete: every non-visual test in tests/ace
  and tests/perf passes, no visible "family" copy for the agent-session concept remains
  in ACE, family-concept test identifiers in tests/ace use agent-session naming, new-shape
  fleet keys have fixture coverage, docs name the renamed perf scenarios and copy,
  the j/k benches show no regression, and `sase tool run check` passes.

  '
parent_bead: sase-17m.5.1
phases:
- id: copy-stragglers
  title: Visible-copy and comment stragglers plus the 15 failing tests
  depends_on: []
  size: medium
  description: 'copy-stragglers: fix the 15 tests/ace widget tests that fail on master
    because they call renamed panel methods or expect retired FAMILY/family copy.
    Rename the remaining visible agent-family copy in src/sase/ace (tribe Composition
    "N families", Artifacts Agents pane description, statistics help eligibility text,
    the fold notify message, revert preview error, confirm-revert "scope family")
    and the agent-family comments and docstrings listed in the plan. Re-baseline,
    through /sase_monitor, only the PNG goldens whose pixels change because of this
    copy.'
- id: widget-tests
  title: Agent-session test identifiers in widgets, modals, actions, and visual tests
  depends_on:
  - copy-stragglers
  size: medium
  description: 'widget-tests: rename the family-concept test functions, helpers, locals,
    fixture kwargs, and docstrings in tests/ace/tui/widgets, tests/ace/tui/modals,
    tests/ace/tui/actions, and tests/ace/tui/visual to agent-session naming. Keep
    unrelated meanings, marked core-emitted legacy wire fixtures, and opaque rendered
    test data. Update the renamed node ID in tests/reproducible_flake_baseline.txt.
    No pixels change.'
- id: tui-tests
  title: Agent-session test identifiers in top-level TUI, models, and contract tests
    plus new-shape fleet fixtures
  depends_on:
  - widget-tests
  size: medium
  description: 'tui-tests: rename the family-concept test functions, helpers, locals,
    fixture kwargs, and docstrings in tests/ace/tui/*.py, tests/ace/tui/models, tests/ace/tui/artifacts_contract,
    and tests/ace/*.py (including _family() in _agent_enter_targets_helpers.py and
    owner_roster_fixture family= kwargs). Name the fleet summary and locator fixtures
    as legacy wire fixtures and add new-shape fleet fixture coverage that proves the
    new agent-session keys are read first.'
- id: docs-verify
  title: Docs integration, perf re-run, classification, and full verification
  depends_on:
  - tui-tests
  size: medium
  description: 'docs-verify: update docs that still name the renamed perf scenarios
    or the retired agent-family copy (docs/perf_runbook.md, docs/ace.md, docs/pager.md,
    the orchestration blog post, and any doc describing copy changed in copy-stragglers).
    Run the full non-visual tests/ace and tests/perf suites, a visual check over the
    session/tribe/clan/fleet goldens, the j/k navigation bench on a quiet host plus
    the view-hints regression check, the final famil classification sweep, and `sase
    tool run check`.'
proposed_by: bbugyi200.athena.sase-17m.5.1.land
create_time: 2026-09-25 04:39:06
status: wip
bead_id: sase-17m.5.1.6
---

- **PROMPT:** [prompts/202609/agent_session_ace_cutover_finish.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_session_ace_cutover_finish.md)
- **PARENT:** [202609/agent_session_ace_cutover.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_session_ace_cutover.md)
- **BEAD:** [sase-17m.5.1.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17m/sase-17m.5.1.6.md)

# Plan: Finish ACE agent session surfaces (ace-cutover landing gaps)

## Context

This epic finishes epic `sase-17m.5.1` ("ACE agent session surfaces (ace-cutover)",
`plan:202609/agent_session_ace_cutover.md`), which is itself the child plan of phase
`sase-17m.5` of the rename epic `sase-17m` (`plan:202609/agent_session_rename.md`). All
five phases of `sase-17m.5.1` closed, but its land agent found gaps it caused that must
be fixed before it can close. The land agent's findings are recorded as note #1 on bead
`sase-17m.5.1`; read it first.

The parent plans stay authoritative. Before any change, every phase worker must read, in
`plan:202609/agent_session_ace_cutover.md`, the sections **State of the world at
planning time**, **Scope boundaries with sibling phases**, **Meanings of "family" inside
ACE that must not change**, and **Rules that bind every phase**. Also read the
**Vocabulary**, **Identifier rules**, and **Compatibility policy** sections of
`plan:202609/agent_session_rename.md`. In summary:

- Use `agent_session` / `AgentSession` / `AGENT_SESSION` inside longer identifiers and
  file names, and bare `session` only for short tokens, enum/string values, and visible
  copy.
- Keep unrelated meanings of "family": the Patch revert/sibling family
  (`RelationKind.FAMILY`, the Patches grouping keys `("project", "family")` /
  `("status", "family")`, `patch_revert_family`), model/provider families and provider
  usage `family:` scopes, search-operator families, `MarkerFamily`, colour/hue/font
  families, `document_filename_family`, and generic English ("natural family" in
  link-reveal docs).
- Keep every marked core-emitted legacy reader (`# legacy agent-family spelling: ...`)
  and every named durable legacy reader. `core-contract` (`sase-17m.8`) owns flipping
  them. Never drop a legacy input.
- Do not edit `src/sase/agents_sync/` or its goldens (`session-pages`, `sase-17m.9`), or
  sase-core, sase-telegram, or chezmoi.
- No internal aliases just to shrink a diff. This epic renames only; it adds no work to
  navigation, render, or refresh handlers.

Before any change, every phase worker must also read the `tui.md` and `lint_and_test.md`
memory notes with `/sase_memory_read`, and phases that touch goldens or perf must read
`tui_perf.md` and `tui_screenshot.md`. In a fresh workspace, run `just install` first.

### State at planning time

- A full non-visual run of `tests/ace` and `tests/perf`
  (`pytest -n 8 -m 'not visual and not slow' tests/ace tests/perf`) gives 11 failed, 4
  errors, 14459 passed. Every failure comes from the epic:
  - `panel._update_family_display(...)` (now `_update_agent_session_display`) in
    `tests/ace/tui/widgets/_summary_fold_contract_helpers.py:89`,
    `test_identity_header_xprompt.py:129`, and
    `test_prompt_panel_section_navigation_rendering.py:263`.
  - `panel._family_text_with_hints(...)` (now `_agent_session_text_with_hints`) in
    `test_agent_display_xprompt_hints.py:212`.
  - Expected retired copy: `"FAMILY\n"` header
    (`test_prompt_panel_section_navigation_rendering.py:294`),
    `"Members: N agents · 1 family"` (`test_agent_display_clan.py`,
    `test_agent_display_clan_roster.py`), and `" 0  [✓] build · family"`
    (`test_agent_display_tribe.py`, `test_agent_display_tribe_roster.py`, 4 nodes).
  - The 4 `test_summary_fold_contracts.py::...[family]` setup errors and
    `test_family_conversation_bodies_do_not_change_across_scale` share the helper
    failure.
- `git grep -in famil` still shows about 270 hits in `src/sase/ace` (mostly unrelated or
  marked readers) and about 1300 hits in `tests/ace`: about 178 family-named test
  functions and helpers in 88 files, plus family-concept locals, kwargs, and docstrings.
  `tests/perf`, `src/sase/default_config.yml`, and the schema (except the
  `legacy_agent_family_syntax` flag entry) are clean.
- On-disk fixture keys already use the new spelling (for example
  `tests/ace/tui/owner_roster_fixture.py` writes `agent_session` from a `family=`
  kwarg), so the remaining test work is identifier renaming, not a persisted-shape
  change.
- `tests/ace/tui/_fleet_summary_fixture.py` and `_fleet_locator_fixture.py` emit only
  the legacy wire keys (`family_id`, `family_role`, `family_label`), and no test in
  `tests/ace` feeds the new keys (`agent_session_id`, `session_label`,
  `agent_session_role`, `agent_session`) that
  `models/_fleet_agents_{nodes,rows,identity,promotion}.py` read first.
- `docs/perf_runbook.md` (~line 1097) still names `family_container_press` and
  `family_container_unfolded_press`, which the epic renamed to `session_container_press`
  and `session_container_unfolded_press`. `docs-memory` (`sase-17m.6`) closed before
  those renames landed.
- `tests/reproducible_flake_baseline.txt` names
  `tests/ace/tui/widgets/test_prompt_panel_header.py::test_family_header_renders_followup_role_attribution`.
  `tests/shard_timings.json` is keyed by file and has no family entries.
- No `--epic-symbol` entries exist for `sase-17m.5.1`.

### Known unrelated visual failures (do not absorb into goldens)

- `sase-18y`: `test_agent_session_panel_fold_levels_and_member_override_png_snapshots`
  times out on the `agent-xprompt` anchor (caused by `sase-18g.2`). Its goldens
  `agents_session_panel_level_2`, `agents_session_conversation_level_1/2` stay
  untouched.
- `sase-18o` (command_line PNGs), `sase-18n` (narrow provider-usage top-bar PNGs), and
  `sase-x5` (includes `test_swarm_clan_panel_png_snapshots`) are convergence or drift
  failures on master.

Exclude these from any `just fix-tui-screenshots` selector, or leave their results
unaccepted. Never accept a golden update that is not explained by this epic's copy.

### Rules that bind every phase

- Run `just fix` and then `sase tool run check` before closing a phase. Never run
  `just check-full`. Hand long commands (visual runs, benches, full suites) to
  `/sase_monitor`.
- Pixels: `just check` does not run PNG snapshots. A phase whose change alters rendered
  pixels re-baselines those goldens in the same phase with a targeted
  `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`, then inspects every
  creation, removal, and update group before finalizing. Identifier-only renames must
  not change pixels; if one does, find out why before accepting.
- Opaque rendered test data (for example agent names such as `family-test`,
  `visual-family`, `remote-family`, `done-family`) stays as it is when changing it would
  move pixels in a golden. Rename it only in non-visual tests where it is a plain
  identifier and no golden depends on it.
- Tests that deliberately prove a legacy input still loads keep their legacy data, are
  named as legacy, and get a new-shape sibling.
- When you rename a test function whose node ID appears in
  `tests/reproducible_flake_baseline.txt`, `tests/shard_timings.json`, or a test-cost
  budget file, update that entry in the same phase.
- Classification: at the end of each phase, `git grep -in famil` over the files the
  phase touched may show only an unrelated meaning, a marked core-emitted legacy reader,
  a named durable legacy reader or fixture, opaque rendered test data, or something
  owned by a later phase here or a sibling phase. Record anything else as a
  `PROPOSED FOLLOW-UP:` note on the phase bead.

## Phase details

### copy-stragglers — Visible-copy and comment stragglers plus the 15 failing tests

- Fix the 15 failing tests listed above so they use the renamed methods and the session
  copy (`SESSION` header, `1 session`, `build · session`). Rename their family-concept
  test function names in the same edit, and update the flake-baseline node ID if one of
  them is listed there.
- Visible copy in `src/sase/ace` (then update every test that pins the old string):
  - `widgets/prompt_panel/_agent_display_tribe_header.py` (~line 67) and
    `widgets/prompt_panel/_identity_header_compact.py` (~line 382): the Composition chip
    `f"famil{'ies' if ... else 'y'}"` becomes `session` / `sessions`.
  - `_artifact_tab_descriptions.py` (~line 28): "Rows are agent runs, families and their
    shells" becomes "agent runs, sessions and their shells".
  - `modals/statistics_help_modal.py` (~lines 276-277): "a live parallel family member"
    and "a serial family" become session wording.
  - `actions/navigation/_fold.py` (~line 323): "Fold levels shape clan, family,
    neighbor, and slow-call summaries" becomes "clan, session, neighbor, ...".
  - `revert_agent_preview.py` (~line 58): the error label `f"family '{base}'"` becomes
    `f"session '{base}'"`.
  - `modals/confirm_revert_agent_modal.py` (~line 148): the subtitle's scope word for
    `preview.scope in ("session", "family")` becomes `"session"`. Keep accepting the
    legacy `"family"` scope value, with its marker comment.
- Comments and docstrings (agent-family concept → agent session): `artifact_reads.py`,
  `bead_touches.py`, `glossary_reads.py`, `memory_reads.py`, `skill_uses.py`,
  `opened_workspaces.py` ("agent-family context", "family rows", "family role label"),
  `modals/agent_workspace_tmux_modal.py` (~line 134), `revert_agent_discovery.py`
  (~lines 27, 31, 67), `widgets/_agent_list_render_agent_prefix.py` (~lines 58-59), and
  the `models/_agent_session_shell_membership.py` module docstring. For that docstring,
  trace whether owner records still carry `family_id` from core. If they do, keep the
  key name and mark it as core-emitted legacy spelling; if not, name the new key.
- Before you change each string, trace it and confirm it is the agent-session concept.
  If it is ambiguous (for example `llm_calls/reader.py`
  `_same_workflow_artifact_family`), trace the value to its source, and leave it with a
  one-line justification in the phase note if it is an unrelated meaning.
- Pixels: the tribe composition chip, the Artifacts Agents description, the statistics
  help text, and the revert modal subtitle or error may appear in goldens. Likely
  selectors include `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py`,
  `..._agents_tribe_prompts.py`, `..._agents_tribe_clan_summaries.py`,
  `..._artifacts_agents.py`, `..._config_center_statistics.py`, `..._revert.py`, and any
  identity-header compact goldens (grep the visual tests for the fixtures that render
  them). Run a targeted re-baseline through `/sase_monitor` and accept only diffs
  confined to the changed text.
- Exit: the 15 tests pass; no visible agent-family copy remains in `src/sase/ace`; the
  touched files classify cleanly; the goldens match; and `sase tool run check` passes.

### widget-tests — Agent-session test identifiers in widgets, modals, actions, and visual tests

- Scope: `tests/ace/tui/widgets/**` (about 74 files and 480 hits),
  `tests/ace/tui/modals/**`, `tests/ace/tui/actions/**`, and `tests/ace/tui/visual/**`
  (about 27 files and 220 hits).
- Rename family-concept test functions (`test_*family*`), helpers (for example
  `_family_with_monitor`, `_family_container_with_running_monitor`, `make_large_family`,
  `render_family`, `_family_case`, `_family_lane_case`), locals (`family_agent`,
  `family_root`, `family_text`, `family_name`, `family_header`, `family_child`, ...),
  constants (`_FAMILY_NAME`), fixture kwargs, docstrings, and comments to agent-session
  naming.
- Keep `MarkerFamily`, `search_operator_family`, `model_family`, provider-usage
  `family:` scopes, Patch revert-family tests, marked legacy wire fixtures, and opaque
  rendered test data (see the rules above). In visual tests, rename identifiers only;
  snapshot ids, golden file names, and rendered data were settled by `sase-17m.5.1.5`
  and must stay pixel-identical.
- Update `tests/reproducible_flake_baseline.txt` (the
  `test_prompt_panel_header.py::test_family_header_renders_followup_role_attribution`
  entry) if you rename that test, and any other node-ID or file entry you rename.
- Verify with the touched test files and a run of the affected visual tests in check
  mode through `/sase_monitor`. Expect zero pixel changes.
- Exit: the scoped directories classify cleanly; the touched tests pass; no golden
  changes; and `sase tool run check` passes.

### tui-tests — Agent-session test identifiers in top-level TUI, models, and contract tests plus new-shape fleet fixtures

- Scope: `tests/ace/tui/*.py` (about 59 files and 470 hits), `tests/ace/tui/models/**`
  (about 19 files and 100 hits), `tests/ace/tui/artifacts_contract/**`, and
  `tests/ace/*.py` (the revert tests).
- Rename the family-concept test functions, helpers, locals, constants, kwargs,
  docstrings, and comments as in `widget-tests`. Specifically:
  - `_family()` in `tests/ace/tui/_agent_enter_targets_helpers.py` and its callers
  - `owner_roster_fixture.py`'s `family=` kwarg (it already writes the `agent_session`
    on-disk key), `FACT_FAMILY_OBSERVATIONS`, and `_write_fact_families`, plus their
    importers (`test_owner_facts_oracle.py`, `test_owner_roster_oracle.py`, and
    `visual/test_ace_png_snapshots_agents_fleet.py`, identifier-only)
  - the family-named tests in `test_agent_ordering*`/sort-and-reorder,
    `test_workflow_step_loader*`, `test_agent_enter_targets*`, and the models tests
- Fleet fixtures:
  - Rename the kwargs of `_fleet_summary_fixture.py` and `_fleet_locator_fixture.py`
    that emit core-emitted legacy keys (for example `family_id` → `legacy_family_id`, or
    add a docstring and marker), so they read as legacy wire fixtures
    (`# legacy agent-family spelling: core emits "<key>" until core-contract`). Keep
    them emitting the legacy keys, because core still does.
  - Add new-shape sibling fixtures or a `new_shape=True` mode that emits
    `agent_session_id`, `session_label`, `agent_session_role`, and `agent_session`.
  - Add focused tests proving that `models/_fleet_agents_nodes.py`,
    `_fleet_agents_rows.py`, `_fleet_agents_identity.py`, and
    `_fleet_agents_promotion.py` read the new keys first and still accept the legacy
    keys.
- Exit: the scoped files classify cleanly; the new fleet coverage passes; all touched
  tests pass; and `sase tool run check` passes.

### docs-verify — Docs integration, perf re-run, classification, and full verification

- Docs (`docs-memory` closed before these renames landed, so this epic owns them):
  - `docs/perf_runbook.md`: rename the `family_container_press` and
    `family_container_unfolded_press` scenario rows to `session_container_press` and
    `session_container_unfolded_press`. Check the file for any other renamed span, task,
    or scenario name (`agents.agent_session_plan_preview_warmup`,
    `sase-agents-session-previews`).
  - `docs/ace.md`: "Sequential plan-family workflows" (~line 5051) and "New plan-family
    metadata" (~line 5066) use session wording. Grep `docs/ace.md` for the copy changed
    in `copy-stragglers` (tribe composition, statistics eligibility, revert scope) and
    match it.
  - `docs/pager.md` (~line 229): "Family and clan conversations" becomes "Session and
    clan conversations".
  - `docs/blog/posts/why-coding-agents-need-orchestration.md` (~line 280): "clan,
    family, or tribe identity" becomes "clan, session, or tribe identity".
  - Leave `docs/agents_sidecar.md`, the `families/` sidecar URLs, and
    `docs/agent_sessions.md`'s historical "formerly agent families" lines for
    `session-pages` or as intentional history.
- Verification:
  - Run `just fix`, then the full non-visual suite
    `pytest -n 8 -m 'not visual and not slow' tests/ace tests/perf` through
    `/sase_monitor`. It must be green.
  - Run a check-mode visual pass over the agent session, tribe, clan, fleet, Artifacts
    Agents, revert, and statistics PNG tests through `/sase_monitor`. The known
    unrelated failures above are the only permitted failures.
  - Perf: wait (through `/sase_monitor` sleeping) until the host's 1-minute load average
    is low (under about 2), then run `pytest -s -m slow tests/ace/tui/bench_tui_jk.py`
    and the view-hints regression check (`tests/perf/check_view_hints_regression.py`).
    Record the p50/p95 numbers in the close note. If any budget misses, run the same
    bench in the same load window on a `git worktree` of the pre-epic commit `02c4b029a`
    to show whether the miss predates the epic, and record both sets.
  - Classification sweep: run case-sensitive and case-insensitive `git grep famil` over
    `src/sase/ace`, `tests/ace`, `tests/perf`, `src/sase/default_config.yml`, and
    `src/sase/config/sase.schema.json`. Classify every hit as an unrelated meaning, a
    marked core-emitted legacy reader, a named durable legacy reader or fixture, opaque
    rendered test data, or the flag. Fix small stragglers in place, and record a
    per-category summary in the phase close note.
  - Run `sase bead epic-symbols` for this epic and each of its phases, resolve any
    entries, and run `sase tool run check`.
- Exit: docs match the landed names and copy; the non-visual suites are green; the
  visual pass shows only the known unrelated failures; perf shows no regression
  attributable to the epic; every `famil` hit in ACE scope is classified; and
  `sase tool run check` passes.
