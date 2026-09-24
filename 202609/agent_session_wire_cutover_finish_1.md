---
tier: tale
size: medium
title:
  Finish the agent-session wire cutover (regressions, plan_chain aliases, core mirror
  names)
goal:
  The regressions the sase-17m.3.1 epic introduced are fixed. The Rust cleanup planner
  sees parallel session membership again, and the legacy fixtures and stale test
  expectations are repaired. No caller uses the deprecated plan_chain agent_family_*
  aliases, and those aliases are deleted. src/sase/core no longer carries family-concept
  field or type names outside named legacy readers. sase tool run check shows no failure
  beyond the pre-existing, separately tracked ones.
proposed_by: bbugyi200.athena.sase-17m.3.1.land
bead: sase-17m.3.1
create_time: 2026-09-24 12:05:08
status: wip
---

- **BEAD:**
  [sase-17m.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17m/sase-17m.3.1.md)

# Plan: Finish the agent-session wire cutover

## Context

Epic `sase-17m.3.1` ("Python persistence and wire cutover to agent session",
`plan:202609/agent_session_wire_cutover.md`) closed all seven phases. Its land agent
then found work that is still unfinished or regressed. This plan covers only that
remaining work. Close-out of the epic itself is not part of this plan; the epic's land
agent resumes after this plan lands.

Read these first:

- `plan:202609/agent_session_wire_cutover.md`, especially its **Rules for every phase**
  section and the `canonical-keys` and `wire-mirrors` phases
- the **Identifier rules** and **Meanings of "family" that must not change** sections of
  the parent plan `plan:202609/agent_session_rename.md`

Repo: **sase** only. Do not edit sase-core. The core pin (`sase-core-revision.txt`,
`9956773`, which includes core-expand) serializes legacy spellings and accepts most new
spellings as aliases. The exception is `agent_session_parallel`: core declares it only
as `rename = "agent_family_parallel"`, with no alias, in `runner_capacity/wire.rs`,
`agent_cleanup/wire.rs`, `agent_scan/wire.rs` (`AgentMetaWire`), and
`fleet_owner_facts.rs`. Anything Python sends to those structs must use the legacy key,
through one named boundary helper. Adding that alias in core belongs to `core-contract`
(`sase-17m.8`), which already has a note about it.

Rules that carry over from the epic:

- Put every legacy fallback in an explicitly named helper or constant.
- Mark each legacy fallback with a `# legacy agent-family spelling` comment.
- Writers emit only new spellings.
- Leave unrelated meanings of "family" alone.
- Do not rename runtime modules, user syntax, or CLI output keys. `runtime-cutover`
  (`sase-17m.4`) owns those, and `ace-cutover` (`sase-17m.5`) owns ACE names.

## Evidence at planning time

At master `db4266e1c` with the pinned core `9956773` installed, the six regression tests
listed in step 1 still fail. A focused run of their five test files gave 6 failed and 63
passed. None of the four commits that landed after `77e0cfb6c` touches this concept.

An earlier full `sase tool run test` lane at master `77e0cfb6c` had 19 failures:

- Six are regressions introduced by this epic. They are listed in step 1.
- Twelve are pre-existing and unrelated. They also fail on the pre-epic tree and are
  already routed elsewhere:
  - 8 prompt-panel `CardPart` tests and the timezone LLM-calls test → `sase-17d`
  - `test_no_ref_prefix_dispatch` → `sase-174`
  - `test_same_body_in_different_projects_does_not_coalesce` → `sase-175`
  - `test_app_import_budget` → `sase-13p`
- One is the flaky zsh `sbd` completion smoke test.

At `77e0cfb6c`, `sase tool run check` stopped at `lint (symvision)` on 73 pre-existing
private-import violations. They are in `llm_provider/usage` and the `plugins_browser`
install modules, tracked by `sase-17l`, `sase-17j`, and `sase-17c`. Do not fix those
here. If symvision is still red, prove that your diff adds no symvision finding: the
symvision output must be identical with and without your diff. The pin move to `9956773`
added the `tool_run_claim` / `tool_run_request_stop` bindings, so the binding-validation
failure that `sase-17m.3.1.7` reported should be gone.

## Step 1 — Fix the regressions this epic introduced

1. **Cleanup planner loses parallel membership (src bug).**
   - `sase.core.agent_cleanup_facade.plan_agent_cleanup` sends
     `agent_cleanup_wire_to_json_dict(wire_targets)` to the Rust `plan_agent_cleanup`
     binding. Each target dict now carries `agent_session_parallel`, which the pinned
     core `AgentCleanupTargetWire` silently ignores. The Rust plan therefore treats
     parallel family roots as sequential, which changes the kill, dismiss, and skip
     items.
   - Add one named core-boundary projection for the cleanup targets, for example
     `cleanup_targets_for_core(...)` in `src/sase/core/agent_cleanup_wire.py`. It emits
     `agent_family_parallel` in place of `agent_session_parallel`. Model its docstring
     and legacy comment on
     `sase.core.runner_slots._admission_capacity_records.capacity_session_keys_for_core`.
   - Use the projection only on the Rust binding path. Leave
     `agent_cleanup_wire_to_json_dict` unchanged for its ACE callers.
   - Then audit every other Python → `sase_core_rs` payload that can carry
     `agent_session_parallel`, and route any that reach one of the four no-alias structs
     through a legacy-key boundary helper. Search with
     `git grep -n "agent_session_parallel" -- src` plus every `asdict` or
     `*_to_json_dict` handed to a `require_rust_binding(...)` call.
   - Add a test proving that the payload the binding receives carries
     `agent_family_parallel: true` and no `agent_session_parallel` key.
   - Exit: `tests/test_core_facade/test_agent_cleanup_facade.py` passes, including
     `test_rust_cleanup_planner_matches_python_reference[parallel-family-root]` and
     `[clan-scope-active-parallel-family]`.
2. **Legacy migration fixture was renamed mechanically.**
   - In `tests/agent_scan_golden/fixture_builder.py::_build_ace_run_running`, the
     `legacy_clan` agent meta is a pre-rename on-disk fixture. Core derives a legacy
     `agent_clan` from it, but only from `agent_family_parallel`.
   - Restore its keys to `agent_family`, `agent_family_role`, and
     `agent_family_parallel`, and add a comment saying it is a legacy-shape fixture.
   - Then review every test fixture that the epic's commits changed from
     `"agent_family_parallel"` to `"agent_session_parallel"`. Find them with
     `git log -p bb81b993a^..HEAD -- tests | grep -n 'agent_family_parallel'`.
   - Restore the legacy spelling wherever the fixture represents pre-rename on-disk data
     whose legacy semantics (legacy clan derivation, legacy parallel marker) are under
     test. Keep or add a new-shape fixture next to each one.
   - Exit: `tests/test_core_agent_scan_records_running.py` and the other
     `agent_scan_golden` consumers pass.
3. **Stale test expectations.** Update each test to the new spelling, and add a
   legacy-input case where one is missing:
   - `tests/test_agent_name_wipe.py::test_wipe_container_name_preserves_member_artifacts_and_registry[family]`:
     the registry now stores `container_kind == "session"`. Rename the param id to
     `session` and expect `"session"`. Add a legacy param whose meta uses `agent_family`
     / `agent_family_parallel` and still resolves to `"session"`.
   - `tests/test_axe_run_agent_helpers_artifacts.py::test_promote_to_workflow_ignores_preexisting_hood_neighbor_prefix`:
     expect `container_kind == "session"`.
   - `tests/ace/tui/test_agent_family_status_convergence_repro.py::_load_production_completion`:
     expect `action_data["agent_session_root_suffix"]`, and assert that
     `family_root_suffix` is absent.

## Step 2 — Finish `canonical-keys`: drop the deprecated plan_chain aliases

The `canonical-keys` phase added the `agent_session_*` helpers to
`src/sase/plan_chain.py` but left deprecated wrappers and constants in place, and
roughly 200 call sites still use them. Both the epic plan ("update all callers in src
and tests") and the parent plan ("Do not keep internal aliases just to shrink the diff")
require the migration.

1. Replace every use in `src` and `tests`:

   | Deprecated name                      | Replacement                           |
   | ------------------------------------ | ------------------------------------- |
   | `agent_family_phase_name`            | `agent_session_phase_name`            |
   | `agent_family_base`                  | `agent_session_base`                  |
   | `agent_family_suffix_token`          | `agent_session_suffix_token`          |
   | `is_agent_family_member`             | `is_agent_session_member`             |
   | `agent_family_role_for_suffix`       | `agent_session_role_for_suffix`       |
   | `allocate_agent_family_child_suffix` | `allocate_agent_session_child_suffix` |
   | `AGENT_FAMILY_SEPARATOR`             | `AGENT_SESSION_SEPARATOR`             |
   - In tests, replace `AGENT_FAMILY_FIELD` / `AGENT_FAMILY_ROLE_FIELD` /
     `AGENT_FAMILY_PARALLEL_FIELD` with the matching `LEGACY_AGENT_FAMILY_*` constant
     when the test builds a legacy fixture. Use the `AGENT_SESSION_*` constant
     otherwise.
   - Also update:
     - `monkeypatch` / `patch` target strings that name the old attributes
     - re-exports and `__all__` lists
     - `from sase.plan_chain import ...` lines inside function bodies
   - The linked plugin repos (sase-telegram, sase-github, sase-nvim,
     sase-research-artifacts) do not import these names.

2. Delete the deprecated block from `src/sase/plan_chain.py`:
   - the `AGENT_FAMILY_*` alias constants
   - `_EXPLICIT_FAMILY_ROLES`
   - the "Deprecated wrappers for the renamed agent-family helpers" section, including
     the private `_stored_family_role`, `_split_agent_family_name`,
     `_agent_family_suffix`, `_reserved_agent_family_names`, and
     `_allocate_agent_family_child_name` wrappers
3. Local variables that exist only to carry the renamed helper's result may be renamed
   in files you already touch, such as `family_base` → `session_base`. Do not start
   module renames. The `_family_attach_*` modules and ACE module names stay.
4. Exit:
   `git grep -nwE 'agent_family_(phase_name|base|suffix_token|role_for_suffix)|is_agent_family_member|allocate_agent_family_child_suffix|AGENT_FAMILY_(SEPARATOR|FIELD|ROLE_FIELD|PARALLEL_FIELD)|_EXPLICIT_FAMILY_ROLES' -- src tests tools demos`
   finds nothing.

## Step 3 — Finish `wire-mirrors`: family-concept names left in `src/sase/core/`

The `wire-mirrors` exit criterion says no family-concept field or type name may remain
in `src/sase/core/` except named legacy readers and `AgentSessionNameKind.FAMILY`. Close
the remaining gaps:

1. **Wait-dependency index vocabulary.** These names live in
   `src/sase/core/wait_dependency_resolution/` (`_types.py`, `_index.py`,
   `_index_entities.py`, `_index_queries.py`, `_index_fork_queries.py`, and
   `_index_identity_queries.py`). They are in-process only; the index is rebuilt from
   scans and never persisted.

   | Current name                           | Rename to                                     |
   | -------------------------------------- | --------------------------------------------- |
   | `FamilyCandidate`                      | `AgentSessionCandidate`                       |
   | `ArtifactCandidate.family_name`        | `agent_session_name`                          |
   | the index's `families` dict            | `agent_sessions`                              |
   | `family_candidate`                     | `agent_session_candidate`                     |
   | `family_candidate_for_root`            | `agent_session_candidate_for_root`            |
   | `_family_entity`                       | `_agent_session_entity`                       |
   | `_family_generation`                   | a name such as `_agent_session_chain`         |
   | `_family_members_after_shell_handoffs` | `_agent_session_members_after_shell_handoffs` |
   | `_family_handoff_state`                | `_agent_session_handoff_state`                |
   | `_fork_family_name_status`             | `_fork_agent_session_name_status`             |
   - Do not reuse the name `agent_session_generation` for `_family_generation`; that
     name already means an ownership wire key.
   - Update the callers `src/sase/core/agent_hold_liveness.py`
     (`family_candidate_for_root`) and `src/sase/core/agent_tribe_evidence.py`
     (`families={}`), plus the tests that call `index.family_candidate(...)`:
     `tests/test_gate_wait_dependency*.py`, `tests/test_monitor_wait_dependency*.py`,
     and so on.
   - Keep the stored-fork-source reader `kind in ("session", "family")` as the named
     legacy branch it already is.

2. **Launch wire attach fields.**
   - Rename `family_attach_parent` / `family_attach_suffix` to
     `agent_session_attach_parent` / `agent_session_attach_suffix` in:
     - `src/sase/core/agent_launch_wire_records.py`
     - `agent_launch_wire_from_dict.py`
     - `agent_launch_wire_conversion.py`
   - `from_dict` reads the new key, then the legacy key, through a named legacy reader.
   - Payloads sent to core emit only the new keys. Pinned core accepts
     `agent_session_attach_parent` / `agent_session_attach_suffix` as aliases
     (`agent_launch/wires.rs`). Values core returns still use the legacy keys, so
     readers such as `agent/launch_hold_preview.py` must accept either spelling.
   - Map from the xprompt directive fields (`xprompt/_directive_types.py`
     `family_attach_*`) at the construction sites, such as `agent/launch_validation.py`.
     The directive fields stay for `runtime-cutover`.
   - If any launch wire dict is persisted to disk, add a legacy-input test for it.
   - Add:
     - an either-spelling hydration test
     - a no-legacy-emitted test
     - a real `sase_core_rs` round trip proving that core accepts the new keys
3. **Dismissed-projection report fields.**
   - In `src/sase/core/agent_artifact_index_lifecycle.py`, rename
     `DismissedProjectionSyncReport.dismissal_family_rows_backfilled` and
     `dismissal_family_rows_skipped_live_or_unknown` to
     `dismissal_agent_session_rows_*`. These are Python-only fields.
   - Update `tests/agent_artifact_index_lifecycle/test_projection_sync.py`.
   - Reword the "dismissed-family reconciliation" log and docstring text, here and in
     `agent_scan_wire_records.py`.
4. **Small leftovers.**
   - Rename the `live_family_members` local in `core/agent_cleanup_python.py`.
   - Keep the skip-reason text `"parallel family still active"` byte-identical. It must
     match the Rust planner's text for the parity tests until `core-contract`.
   - Update docstrings and comments that name the agent-family concept in the core files
     you touch, for example `agent_runtime_wire.py`, `agent_runtime_facade.py`,
     `agent_cleanup_targets.py`, and `dismissed_agent_completion.py`.
5. Leave these alone:
   - `AgentSessionNameKind.FAMILY` and the `"family_name"` legacy payload reads in
     `agent_identity_facade.py`
   - the artifact-pane relation primitive `RelationKind.FAMILY` / `RelationRole.FAMILY`
     and `RelationEdges.family` in `artifact_relation_layout.py` /
     `artifact_relations.py`
   - `GATE_FAMILY_ROLE` in `core/runner_slots/_admission_predicates.py`, which belongs
     to `runtime-cutover`
   - the capacity boundary helper's `agent_family_parallel` key
   - unrelated meanings such as "workflow family" and the merge-subject family in
     `vcs_log_facade.py`
6. Exit: every hit of `git grep -nwiE '[a-z_]*famil(y|ies)[a-z_]*' -- src/sase/core`
   falls into one of the step 5 categories or is a named legacy reader with its comment.

## Step 4 — Verify

1. Run `just install` in a fresh workspace, then run `just fix`.
2. Run the focused suites:
   - `tests/test_core_facade/`
   - `tests/agent_artifact_index_lifecycle/`
   - the wait-dependency, monitor, and gate wait-dependency tests
   - `tests/test_agent_name_wipe.py`
   - `tests/test_axe_run_agent_helpers_artifacts.py`
   - `tests/ace/tui/test_agent_family_status_convergence_repro.py`
   - `tests/test_core_agent_scan_records_running.py`
   - `tests/test_plan_chain_agent_session_keys.py`
   - `tests/test_agent_session_durable_json.py`
   - `tests/test_agent_session_wire_mirrors.py`
   - the launch wire tests
3. Run `sase tool run check`. If `lint (symvision)` fails, diff its finding list against
   a run without your diff (for example `git stash`). It must be identical, apart from
   findings your diff removes. The test lane must not show any failure beyond the
   pre-existing set listed under **Evidence at planning time**.
4. Do not run `just check-full`.
5. Record out-of-scope discoveries as `PROPOSED FOLLOW-UP:` notes on your bead.
