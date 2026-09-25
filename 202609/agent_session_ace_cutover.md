---
tier: epic
title: ACE agent session surfaces (ace-cutover)
goal: 'Inside ACE (src/sase/ace/**, tests/ace/**, tests/perf/**, default_config.yml, and
  sase.schema.json), the former agent-family concept is named "agent session" in every
  module, class, identifier, row kind, relation, grouping mode, trace name, perf
  scenario, comment, test, and golden. The visible copy reads SESSION SHELLS, SESSION,
  "Session" grouping, and "collapse session". Core-emitted legacy spellings stay as
  marked readers. Unrelated meanings of "family" are unchanged. The performance contract
  does not change, and `sase tool run check` passes.

  '
phases:
  - id: models
    title: ACE model modules and Agent identifiers
    depends_on: []
    size: medium
    description:
      "models: rename the family-named modules in src/sase/ace/tui/models/
      (_agent_imported_family, _agent_parallel_family,
      _agent_status_family{,_core,_planner,_policy}, _family_shell_membership,
      agent_family_members, agent_family_preview_cache). Also rename their classes and
      functions, the family-concept Agent methods and properties
      (is_family_member_child, family_reference_name, presented_family_reference_name),
      AgentChildLinkage.FAMILY_MEMBER, and the family identifiers in agent_groups,
      agent_tribe_summary, agent_nodes, agent_bundle, clan, loaders, and fleet-agents
      models. Core-emitted legacy keys stay as marked readers. Update every importer, in
      or outside ACE, and rename the matching tests/ace/tui/models tests and helpers."
  - id: actions
    title: Agents actions, folding, navigation, and preview warmup
    depends_on:
      - models
    size: medium
    description:
      'actions: rename actions/agents/_loading_family_previews.py and its mixin methods.
      Rename the trace span agents.family_plan_preview_warmup to
      agents.agent_session_plan_preview_warmup and the task sase-agents-family-previews
      to sase-agents-session-previews. Change the fold and navigation kind value
      "family" to "session" and rename the family identifiers in actions/agents,
      actions/navigation, actions/agent_workflow, and the other ACE action and app
      modules. Rename the perf scenarios family_container_press and
      family_container_unfolded_press to session_container_*, along with their baselines
      and bench assertions. Update the tests for all of these.'
  - id: contract-completion
    title: Artifacts-pane contract, row kinds, and completion kinds
    depends_on:
      - actions
    size: medium
    description:
      'contract-completion: rename the Agents-pane relation
      family/agent_family_container to session/agent_session_container, the grouping
      mode by_family (label Family, keys family) to by_session (label Session), and
      _artifact_tab_model FAMILY. Keep the Patch RelationKind.FAMILY. Rename the row
      kinds and identifiers in widgets/artifacts (agents_list, agents_navigation,
      agents_revival, query_rows) and relations/agents.py. Change the agent completion
      candidate kind "family" to "session" across the completion models, directive
      completion, and the prompt-bar completion rows. Update the artifacts contract
      goldens and the completion parity tests.'
  - id: copy
    title: Prompt-panel widgets, visible copy, keymaps, and config
    depends_on:
      - contract-completion
    size: medium
    description:
      'copy: rename widgets/prompt_panel/_agent_display_family{,_render} and every
      family identifier, widget id, and CSS id in the ACE widgets. Change the visible
      copy from FAMILY SHELLS to SESSION SHELLS (including the "also listed under" and
      "see ... SHELLS" tails) and the identity header from FAMILY to SESSION, and rename
      FAMILY_IDENTITY_COLOR. Update the bindings.py and keymaps/metadata.py labels, the
      help_modal text, the "collapse family" command-palette alias, and the clipboard
      copy. Update the family wording in the default_config.yml comments and the
      sase.schema.json descriptions. Re-baseline, through /sase_monitor, only the PNG
      goldens whose pixels change because of the copy.'
  - id: snapshots-sweep
    title: Snapshot renames, perf check, and classification sweep
    depends_on:
      - copy
    size: medium
    description:
      "snapshots-sweep: rename the family-named PNG snapshot tests, fixture modules, and
      goldens, and every remaining family-named tests/ace or tests/perf file. Update the
      shard-timing and flake baselines. Run a full just fix-tui-screenshots through
      /sase_monitor, inspect the report, and remove stale goldens only after the full
      run. Run the j/k navigation benchmark to confirm no regression. Classify every
      remaining famil hit in ACE scope, record hand-offs on sase-17m.5, and run sase
      tool run check."
proposed_by: bbugyi200.athena.sase-17m.5
parent_bead: sase-17m.5
create_time: 2026-09-25 00:05:57
status: wip
---

- **PROMPT:**
  [prompts/202609/agent_session_ace_cutover.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_session_ace_cutover.md)
- **PARENT:**
  [202609/agent_session_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_session_rename.md)

# Plan: ACE agent session surfaces (ace-cutover)

## Context

This epic implements the `ace-cutover` phase (bead `sase-17m.5`) of the parent epic
"Rename agent family to sase agent session" (`plan:202609/agent_session_rename.md`, epic
`sase-17m`). The parent plan has the final say on vocabulary, identifier rules, the
meanings of "family" that must not change, and compatibility policy. Before you start
any phase here, read these parent sections: **Vocabulary**, **Identifier rules**,
**Meanings of "family" that must not change**, **Compatibility policy**, and **ACE agent
session surfaces**. Also read the sibling child plan
`plan:202609/agent_session_runtime_cutover.md`, whose conventions this epic reuses.

Repo: **sase** only. Do not edit sase-core, sase-telegram, or chezmoi.

Before any change, every phase worker must read the `tui.md`, `tui_perf.md`,
`tui_screenshot.md`, and `lint_and_test.md` memory notes with `/sase_memory_read`. The
`copy` phase must also read `gotchas` guidance about `default_config.yml`, which is
already inlined in the core memory. In a fresh workspace, run `just install` first.

### State of the world at planning time

- `wire-cutover` (`sase-17m.3`) and `runtime-cutover` (`sase-17m.4`, child epic
  `sase-17m.4.1`) have landed:
  - The `Agent` dataclass fields are `agent_session`, `agent_session_role`,
    `agent_session_parallel`, and `agent_session_shell`.
  - `plan_chain.py` owns the `AGENT_SESSION_*` constants and the `LEGACY_AGENT_FAMILY_*`
    readers. The deprecated `AGENT_FAMILY_*` aliases are gone.
  - The canonical syntax is `session=`, `session:`/`kind:session`,
    `--next-fork session`, and `SASE_AGENT_SESSION_ATTACH`, with retired spellings gated
    by the `legacy_agent_family_syntax` flag
    (`src/sase/agent/legacy_agent_family_syntax.py`). ACE already uses the shared agent
    query dialect files (`query_profile/profiles/_agents_shared.py`,
    `models/agent_live_query*.py`), which `sase-17m.4.1` owns and already cut over.
- sase-core still **emits legacy spellings** until `core-contract` (`sase-17m.8`).
  Examples in ACE: fleet summaries and locators carry `family_id`, `family_label`, and
  `family_role`, and core container kinds are `"family"`. Python code that reads a
  **core-emitted** value keeps accepting it. Rename the Python identifier around it,
  read the new key first when one exists, and mark each spot with
  `# legacy agent-family spelling: core emits "<value>" until core-contract`.
- Durable data that already has named legacy readers stays as it is:
  - `src/sase/ace/revert_agent_models.py` (`LEGACY_REVERT_SCOPE`,
    `LEGACY_REVERT_SESSION_BASE_KEY`) and its use in `confirm_revert_agent_modal.py`
  - notification `action_data` keys in `actions/agents/_notification_matching.py`
  - `models/agent_bundle.py`'s `family_container` → `agent_session_container` legacy
    bundle-key map
  - `models/_fleet_agents_promotion.py`'s `agent_session_id` / `family_id` fallback

  You may rename the identifiers around these readers, but never drop the legacy input.

- No `--epic-symbol sase-17m(...)` entries remain in the Justfile. If a phase leaves a
  public symbol with no consumer, re-key it to `sase-17m.5` or a later open phase bead,
  and make sure the next phase resolves it.
- Scale: `src/sase/ace` has about 1.4k case-insensitive `famil` hits in about 213 files.
  `tests/ace` has about 3.0k hits in about 246 files, and `tests/perf` has about 80 hits
  in 10 files. Many hits are unrelated meanings that must stay (see below).
- The `sase-17m.4` bead note #4 lists hand-offs from `runtime-cutover` to this phase.
  The phases below cover each one:
  - the ACE model modules and their non-ACE importers, including
    `src/sase/agent_session_plan_preview.py` and its line ~59 comment,
    `tests/test_agent_loader_*` files, and `tests/shard_timings.json`
  - `Agent.is_family_member_child`, `AgentChildLinkage.FAMILY_MEMBER`, and
    `family_reference_name()`
  - `AgentCompletionCandidate(kind="family")` and its detail, including the
    `tests/_xprompt_directive_completion_parity_*` tests
  - the `FAMILY` header and the "Collapse selected workflow/family" keymap and help
    labels, including `tests/test_command_catalog.py` and
    `tests/test_keymaps_display_help_agents.py`
  - the `clipboard/_agents.py` ~line 101 "selected family container" copy
  - the `default_config.yml` and `config/sase.schema.json` family wording

### Scope boundaries with sibling phases

- **`docs-memory` (`sase-17m.6`)** is running concurrently and owns `docs/` and memory.
  Do not edit `docs/`. `snapshots-sweep` compares `docs/ace.md` against the landed copy:
  `SESSION SHELLS`, the `SESSION` header, **Session** grouping, and the fold-scope
  labels. It records every mismatch as a hand-off note on `sase-17m.5` for the parent
  land agent.
- **`session-pages` (`sase-17m.9`)** owns `src/sase/agents_sync/` and the sidecar
  `families/` URLs. Touch those only to follow a renamed import.
- **`core-contract` (`sase-17m.8`)** flips core-emitted values. Leave the marked legacy
  readers for it.
- Every phase here may edit a non-ACE file **only to follow a renamed ACE import,
  symbol, or visible string**. The tests listed in the hand-offs above are examples.

### Meanings of "family" inside ACE that must not change

- The Patch revert/sibling family: `RelationKind.FAMILY`, `RelationRole.FAMILY`, the
  `retry_chain` relation's `kind=RelationKind.FAMILY`, `ace/query/matchers.py`, and the
  `patches_artifact_bindings.py` "Revert-family siblings" text.
- Model and provider families: `model_family`, `_PROVIDER_FAMILY_COLORS`, and the usage
  indicator (`widgets/_usage_indicator_format.py`), provider-usage `family:` scopes, and
  `vcs_family`.
- Search operator families in `widgets/_prompt_input_bar_search.py`
  (`getattr(operator, "family", "transform")`), and generic "family" wording for
  colours, palettes, markers, and fonts (`font-family`).
- Unrelated test names such as `test_check_cost_budgets_classifies_severity_by_family`.

If you are unsure, trace the value to its source. The concept here is always the
`<session>--<suffix>` sequence, its container, members, roles, and shells.

### Rules that bind every phase

- **Identifier rules:** follow the parent plan. Use `agent_session` / `AgentSession` /
  `AGENT_SESSION` inside longer identifiers, module names, and file names; never bare
  `session` there. Use bare `session` only for enum and string values and short tokens,
  for example kinds, row kinds, relation names, fold kinds, grouping mode ids, and
  visible copy.
- **Durable values:** before you rename any string value, check whether it is persisted
  anywhere: fold state, saved queries or selections, the grouping-mode file, pane
  persistence goldens, notification data, or dismissed bundles. At planning time none of
  the ACE-owned values listed in this plan were found to be persisted. The grouping-mode
  file stores `GroupingMode` members, which have no family value. If a phase finds a
  persisted value, it must add an explicitly named legacy reader (for example a
  `LEGACY_*` constant) that loads the old value, emit only the new value, and add a test
  that loads the legacy value.
- **No internal aliases** just to shrink a diff. Update every importer in the same
  phase.
- **Tests:** rename the family-named tests, helpers, and fixtures that cover each
  phase's modules in that phase. When you rename or move a test file, also update its
  test IDs in `tests/shard_timings.json` and in the reproducible-flake baseline files.
  Test fixtures that build agent metadata use the new keys, except fixtures that
  deliberately prove a legacy input still loads. Keep those, name them as legacy, and
  add a new-shape fixture next to them.
- **Performance contract:** this epic only renames. Add no filesystem work, awaits,
  subprocesses, timers, or full rebuilds to navigation, render, or refresh handlers.
  Keep every existing coalescing guard and cache key shape (only its spelling may
  change).
- **Pixels:** `just check` does not run PNG snapshots, and CI's visual job would go red
  if a phase lands copy changes without their goldens. Any phase whose change alters
  rendered pixels must re-baseline the affected goldens in the same phase. Run a
  targeted `just fix-tui-screenshots -- <selectors>` through `/sase_monitor`, then
  inspect every creation, removal, and update group before finalizing. Pure renames of
  identifiers must not change pixels. If one does, find out why before you accept the
  golden.
- **Verification:** run `just fix` and then `sase tool run check` before closing a
  phase. Never run `just check-full`.
- **Classification:** at the end of each phase, `git grep -in famil` over the files the
  phase touched may show only one of these: an unrelated meaning, a marked core-emitted
  legacy reader, a named durable legacy reader or fixture, or something owned by a later
  phase of this epic or a sibling phase. Record anything else as a `PROPOSED FOLLOW-UP:`
  note on the phase bead.

## Phase details

### models — ACE model modules and Agent identifiers

- Rename these modules under `src/sase/ace/tui/models/`, with their classes, functions,
  constants, and locals:
  - `_agent_imported_family.py` → `_agent_imported_agent_session.py`
  - `_agent_parallel_family.py` → `_agent_parallel_agent_session.py`
    (`aggregate_parallel_family_status` → `aggregate_parallel_agent_session_status`)
  - `_agent_status_family{,_core,_planner,_policy}.py` →
    `_agent_status_agent_session{,_core,_planner,_policy}.py`
  - `_family_shell_membership.py` → `_agent_session_shell_membership.py`
  - `agent_family_members.py` → `agent_session_members.py` (`row_is_family_shell`,
    `is_family_container_row`, `is_family_root_entry`, `is_sequential_family_container`,
    and the `__all__`/lazy-export names such as `family_member_status_buckets` and
    `family_roster_container`)
  - `agent_family_preview_cache.py` → `agent_session_preview_cache.py`
- Rename the family-concept names on `Agent` and its models:
  - `is_family_member_child`, `family_reference_name()`, and
    `presented_family_reference_name()`
  - `AgentChildLinkage.FAMILY_MEMBER = "family_member"` →
    `AGENT_SESSION_MEMBER = "session_member"`, after you check that the value is never
    persisted
  - the family comments in `_agent_state.py`, for example the "Family container row
    whose FAMILY SHELLS roster lists this row" comment
- Rename the family identifiers in the other model modules. This covers `agent.py`,
  `agent_nodes.py`, `agent_bundle.py` (keep its legacy bundle-key map), `_agent_clan*`,
  `_agent_ordering.py`, `_agent_status_apply.py`, `_loaders/`, `agent_groups/`
  (`_agent_family_base_from_row`, `roots_by_family`), and `agent_tribe_summary.py`.
  - In `agent_tribe_summary.py`, the unit kind `"family"` becomes `"session"`.
- Fleet-agents models (`_fleet_agents_nodes.py`, `_fleet_agents_rows.py`,
  `_fleet_agents_identity.py`, `_fleet_agents_promotion.py`):
  - Rename the locals and helpers, such as `roots_by_family` and `family_key`.
  - Read `agent_session_*` keys first and keep the `family_id`, `family_label`, and
    `family_role` reads as marked core-emitted legacy readers.
- Update every importer: other ACE packages, `src/sase/agent_session_plan_preview.py`
  (including its comment), `tests/test_agent_loader_*`, and any other non-ACE tests.
- Rename the matching tests and helpers under `tests/ace/tui/models/` (for example
  `test_agent_family_members.py`, `_agent_family_members_helpers.py`,
  `test_agent_family_preview_cache.py`, `test_imported_family_tree.py`,
  `test_agent_family_*_lanes.py`, `test_agent_neighbor_family_lanes.py`,
  `test_monitor_family_root_projection.py`, and `test_agent_family_member_statuses.py`).
  Also rename the family-concept test function names in these files.
- Exit: no family-concept identifier remains in `src/sase/ace/tui/models/` or
  `tests/ace/tui/models/`, except marked readers. No pixels change, and
  `sase tool run check` passes.

### actions — Agents actions, folding, navigation, and preview warmup

- Rename `actions/agents/_loading_family_previews.py` →
  `_loading_agent_session_previews.py` along with its mixin and methods
  (`_schedule_family_plan_preview_warmup`, `_spawn_family_plan_preview_warmup_task`,
  `_run_family_plan_preview_warmup`). Also update the string-named hooks in
  `_startup_loads_maintenance.py`, `_loading_apply.py`, and `_wait_actions.py`
  (`getattr(self, "_schedule_family_plan_preview_warmup", None)`), and the refresh
  source `"family_preview_followup"` → `"session_preview_followup"`.
- Trace and task names: `agents.family_plan_preview_warmup` →
  `agents.agent_session_plan_preview_warmup`, and `sase-agents-family-previews` →
  `sase-agents-session-previews`. Update every consumer: trace readers, perf tooling,
  and tests. If a `docs/` file such as `docs/perf_runbook.md` names the old span or
  task, leave it to `docs-memory` and record a hand-off note on `sase-17m.5`.
- Fold and navigation kinds: change `Literal[..., "family", ...]` and every `"family"`
  fold/navigation kind value to `"session"` in `actions/agents/_folding*.py`,
  `_folding_agent_tree.py`, `_folding.py`, `_display_detail_footer.py`,
  `_confirmation_sase_agents.py`, and `_wait_helpers.py`. Do the same for the consumers
  in `widgets/_keybinding_bindings_agents.py` (`left_navigation_kind` and
  `structural_collapse_kind` sets). Keep the kind strings in the same order and the same
  branching.
- Rename the family identifiers, comments, and messages in the rest of
  `actions/agents/`, `actions/navigation/`, `actions/agent_workflow/`,
  `actions/_link_follow_helpers.py` (which follows `family_reference_name`), the ACE
  app/state modules, and `agent_context_members.py`. Rename
  `_loading_helpers._question_override_answered_by_family` too.
- Perf scenarios: `family_container_press` → `session_container_press` and
  `family_container_unfolded_press` → `session_container_unfolded_press`. Change these
  in `tests/perf/tui_trace/view_hints.py`,
  `tests/perf/baselines/view_hints_baseline.json` (keys only; keep the numbers),
  `tests/perf/bench_tui_trace.py`, and `tests/perf/check_view_hints_regression.py`,
  including its "unfolded family press" message. Move the `tests/perf` fixtures to the
  new metadata keys (`agent_session_shell` rather than `family_shell`), and rename
  family-concept locals and scenario names such as `mixed_family_retry`. Keep
  `container_kind`/`reservation_kind` `"family"` in `bench_agent_catalog.py` only if it
  deliberately mirrors core-emitted or legacy data, and mark it if so.
- Rename the matching tests (for example
  `tests/ace/tui/test_agent_enter_targets_family.py`,
  `test_agent_family_status_convergence_repro.py`,
  `test_agent_runner_slots_families.py`, `test_family_member_relaunch.py`,
  `actions/test_agent_retry_family_projection.py`, `_retry_family_loader_fixture.py`,
  and the fold-transition tests).
- Exit: no family-concept identifier or kind value remains in `src/sase/ace/tui/actions`
  or `tests/perf`, except marked readers. No pixels change, and `sase tool run check`
  passes.

### contract-completion — Artifacts-pane contract, row kinds, and completion kinds

- In `src/sase/ace/tui/_artifact_tab_contract_adapters.py`:
  - The Agents-pane relation `name="family"`, `label="Family"`,
    `source="agent_family_container"` becomes `name="session"`, `label="Session"`,
    `source="agent_session_container"`. Update the source resolver and
    `relations/agents.py` (`self._decls.get("family")`).
  - The grouping mode `by_family` / `"Family"` becomes `by_session` / `"Session"`.
    `default_mode` becomes `by_session`, and the other panes' grouping `keys` entries
    (`("project", "family")`, `("status", "family")`) change to `"session"`. Confirm
    that the key resolver reads the renamed key.
  - Keep `retry_chain`'s `RelationKind.FAMILY` (the Patch kind).
- `_artifact_tab_model.py`: `FAMILY = "family"` becomes `SESSION = "session"` if it is
  the agent-session concept. Leave it alone if it is the Patch kind.
- Rename the family row kinds and identifiers in `widgets/artifacts/` (`agents_list.py`
  `mode_id == "by_family"`, `agents_navigation.py`, `agents_revival.py`,
  `query_rows.py`), and the family identifiers in the rest of `widgets/artifacts/`.
- Completion: in `_agent_completion_models.py` and `_agent_completion_candidates.py`,
  change the `AgentCompletionCandidate` kind `"family"` to `"session"` and update its
  detail text (runtime-cutover's editor bridge wording was "session · N members").
  Update every consumer: `widgets/_directive_completion_agents.py` (`IDENTITY_ROLES`,
  `_TARGET_KIND_ORDER`), `widgets/_prompt_input_bar_completion_rows_agents.py` (glyph
  map, `_FAMILY_NAME_STYLE`, and the kind check), and
  `widgets/_prompt_input_bar_completion_panel_labels.py`.
  - Change the completion glyph `"F"` to `"S"` unless another kind in that map already
    uses `"S"`.
- Update the artifacts-contract goldens under
  `tests/ace/tui/artifacts_contract/goldens/` (`relations/cases.json` and any grouping
  or relation cases), plus `tests/ace/tui/test_agents_pane_detail_relations.py`,
  `tests/ace/tui/models/test_agent_nodes.py`, and the non-ACE
  `tests/_xprompt_directive_completion_parity_*` tests.
- Pixels: the grouping-mode label and the completion glyph and detail are visible.
  Re-baseline the affected goldens with a targeted run, as the rules above require.
- Exit: the pane contract, row kinds, and completion kinds use `session`. The Patch kind
  is unchanged, and `sase tool run check` passes.

### copy — Prompt-panel widgets, visible copy, keymaps, and config

- Rename `widgets/prompt_panel/_agent_display_family.py` →
  `_agent_display_agent_session.py` and `_agent_display_family_render.py` →
  `_agent_display_agent_session_render.py`, with their classes and functions. Also
  rename the family identifiers, widget ids, and CSS ids across `widgets/`
  (`prompt_panel/`, `_prompt_list_markers.py`, and the rest). Leave the usage
  indicator's model-family code alone.
- Visible copy:
  - `_FAMILY_ROSTER_TITLE = "FAMILY SHELLS"` →
    `_AGENT_SESSION_ROSTER_TITLE = "SESSION SHELLS"`
  - `… +N also listed under FAMILY SHELLS` → `… also listed under SESSION SHELLS` in
    `_agent_display_neighbors.py`
  - `… +N more shells (see FAMILY SHELLS)` → `(see SESSION SHELLS)` in
    `_agent_shell_section.py`
  - the identity header `"FAMILY"` → `"SESSION"` in `_identity_header.py` and
    `_agent_display_header.py`. Rename `FAMILY_IDENTITY_COLOR` →
    `SESSION_IDENTITY_COLOR` and keep its colour.
  - `bindings.py` and `keymaps/metadata.py`: "Collapse Selected Workflow/Family / …" →
    "Collapse Selected Workflow/Session / …"
  - `commands/_app_metadata_actions.py`: the "Collapse selected workflow/family, …"
    description and the alias "collapse family" → "collapse session"
  - `modals/help_modal/agents_bindings.py`: "Up: workflow/family/clan/tribe", "Collapse
    selected workflow/family one level", "Set family level 1-2", and "steps or family
    members" use session wording. Leave the Patch help in `patches_artifact_bindings.py`
    alone.
  - `clipboard/_agents.py` ~line 101: "selected family container" → "selected session
    container"
  - any other family copy in ACE widgets, modals, notifications, and toasts
- Config:
  - Change the family wording in the `src/sase/default_config.yml` comments (runner
    capacity ~62/64, pipe depth ~69, the fold/collapse keymap comments ~764/768, and the
    gate-shell sweep description ~1470) to session wording.
  - Change the matching `src/sase/config/sase.schema.json` descriptions (gate-shell
    family members, runner-capacity serial and parallel families, pipe chain).
  - Check whether a generator owns any of these strings, for example
    `tools/sync_feature_flags_schema` for the flags block. If one does, regenerate
    instead of hand-editing. Leave the `legacy_agent_family_syntax` flag entry alone.
  - No keymap keys or action ids contain "family". If one turns up, rename it and keep
    `default_config.yml` in sync.
- Update the non-ACE tests that pin this copy: `tests/test_command_catalog.py`,
  `tests/test_keymaps_display_help_agents.py`, and any config/schema tests. Rename the
  widget tests (`tests/ace/tui/widgets/test_agent_display_family_*.py`,
  `_agent_display_family_helpers.py`, and `test_agent_parallel_family_count_chips.py`).
- Pixels: re-baseline every golden whose pixels change because of the copy, through
  `/sase_monitor`, and inspect every group. Do not rename golden files in this phase;
  `snapshots-sweep` does that.
- Exit: no visible "family" copy for this concept remains in ACE. Config and schema
  wording is updated, goldens match, and `sase tool run check` passes.

### snapshots-sweep — Snapshot renames, perf check, and classification sweep

- Rename the family-named PNG snapshot tests and fixture modules:
  - `tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py` →
    `..._agents_sessions.py`
  - `test_ace_png_snapshots_agents_family_panel{,_gate,_monitor}.py` →
    `..._agents_session_panel{,_gate,_monitor}.py`
  - `_ace_agents_png_snapshot_family_fixtures.py` and
    `_ace_agents_png_snapshot_family_panel_fixtures.py` → `..._session_...`
  - the family-concept snapshot ids and their 26 goldens under
    `tests/ace/tui/visual/snapshots/png/`, for example
    `agents_family_panel_level_1_120x40.png` → `agents_session_panel_level_1_120x40.png`
    and `agents_fleet_production_families_120x40.png` →
    `agents_fleet_production_sessions_120x40.png`
  - Use `git mv` for goldens whose pixels do not change, so each rename stays
    pixel-identical.
- Rename every remaining family-named file in `tests/ace/**` and `tests/perf/**`. Update
  the test IDs in `tests/shard_timings.json` and the flake baselines, plus any test-cost
  budget entries that name them.
- Run a full `just fix-tui-screenshots` through `/sase_monitor` with a generous timeout.
  The follow-up must inspect `latest-report.json` and every creation, removal, and
  update group. Expect renames to show as a creation plus a removal with identical
  pixels. Remove stale family-named goldens only after this full run.
- Run the existing j/k navigation benchmark
  (`pytest -s -m slow tests/ace/tui/bench_tui_jk.py`) and the view-hints regression
  check. Compare against the recorded baselines and confirm there is no regression.
  Record the p50/p95 numbers in the close note.
- Classification sweep:
  - Run case-sensitive and case-insensitive `git grep famil` over `src/sase/ace`,
    `tests/ace`, `tests/perf`, `src/sase/default_config.yml`, and
    `src/sase/config/sase.schema.json`.
  - Classify every hit as one of: an unrelated meaning, a marked core-emitted legacy
    reader, a named durable legacy reader or fixture, or the flag.
  - Fix small stragglers in place.
  - Record, as notes on `sase-17m.5`:
    - a hand-off list of the marked core-emitted readers for `core-contract`
    - any `docs/ace.md` copy mismatches for `docs-memory` or the parent land agent
    - any leftovers, as `PROPOSED FOLLOW-UP:` entries
- Run `sase bead epic-symbols sase-17m.5` and resolve any entries.
- Exit: every remaining `famil` hit in ACE scope is classified, goldens match, perf
  shows no regression, and `sase tool run check` passes.
