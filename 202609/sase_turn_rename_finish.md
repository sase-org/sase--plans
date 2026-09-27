---
tier: epic
title: Finish the sase turn rename
goal: The sase-core contract flip is on sase-core master and pinned in sase, so core
  emits turn and named-proc spellings. Every durable legacy reader and sunset-flag
  alias that the rename corrupted works again and is proven by tests. The stale rename-era
  tests pass, the remaining shell-concept wording in source is gone, and the parent
  epic sase-1ab can land against a verified tree.
phases:
- id: reader-repair
  title: Legacy reader and sunset-flag repair
  depends_on: []
  size: medium
  description: 'reader-repair: fix the durable readers the runtime cutover corrupted
    (the agent_session_turn stripper, the continuation_mode and proc origin no-ops),
    route authored gate specs and the gate.shell config key through the flag-gated
    normalizers, fix the undefined LEGACY_NAMED_PROC_SECTION_ID, and audit every rename
    commit for more corruption, with a legacy-input test for each fix.'
- id: test-repair
  title: Rename-stale tests and CLI contracts
  depends_on: []
  size: medium
  description: 'test-repair: bring the 24 deterministic rename-stale test nodes to
    the turn and named-proc contracts, refresh the completion spec, caption the turn
    completion slots, and make the proc_wire_schema_version lookup optional so the
    pinned bindings check passes against the published core.'
- id: core-flip
  title: Land the sase-core contract flip
  depends_on:
  - reader-repair
  - test-repair
  size: medium
  description: 'core-flip: re-apply the orphaned contract-flip diff onto current sase-core
    master, give the gate_turn_id column migration artifact-index schema 35, prove
    current sase master against it, and land the feat! commit on sase-core origin/master.'
- id: pin-bump
  title: Core pin bump and mirrors
  depends_on:
  - core-flip
  size: medium
  description: 'pin-bump: move sase-core-revision.txt to the landed flip commit, bump
    the Python schema mirrors and probes, and flip the fleet fixtures while staying
    compatible with the published pre-flip core floor.'
- id: vocab-sweep
  title: Finish turn vocabulary in source
  depends_on:
  - reader-repair
  - test-repair
  size: medium
  description: 'vocab-sweep: rename the deferred shell-followup/shell-member cluster,
    rewrite the remaining gate/agent/monitor/proc/session shell prose and visible
    messages in src, and extend the terminology guard to keep it from coming back.'
- id: acceptance
  title: Acceptance audit and cross-repo closeout
  depends_on:
  - pin-bump
  - vocab-sweep
  size: small
  description: 'acceptance: prove the parent epic''s done criteria end to end, sweep
    commits that landed during the rename for new shell wording, close sase-16v, and
    confirm deployed skills and linked repos are current.'
proposed_by: bbugyi200.athena.sase-1ab.land
parent_bead: sase-1ab
create_time: 2026-09-27 08:44:22
status: wip
bead_id: sase-1ab.10
---

- **PROMPT:** [prompts/202609/sase_turn_rename_finish.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sase_turn_rename_finish.md)
- **PARENT:** [202609/sase_turn_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/sase_turn_rename.md)
- **BEAD:** [sase-1ab.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ab/sase-1ab.10.md)

# Plan: Finish the sase turn rename

## Context

This child epic holds the remaining work found while landing epic `sase-1ab` ("Rename
sase shell to sase turn", `plan:202609/sase_turn_rename.md`). All nine `sase-1ab` phases
are closed, but the landing audit on sase master `7b20f4c1c` found the epic incomplete.
Read the parent plan first. Its vocabulary table, identifier rules, unchanged meanings
of "shell", and compatibility policy all bind this plan. The work left:

1. **The contract flip never landed.** Phase `sase-1ab.7` verified its breaking
   sase-core change in a linked checkout, but never committed it. sase-core
   `origin/master` (`912331c`) still emits `AGENT_SCAN_WIRE_SCHEMA_VERSION` 10,
   `PROC_WIRE_SCHEMA_VERSION` 3, and `FLEET_PROTOCOL_VERSION` 2. It still registers the
   legacy `find_gate_shell_by_gate_id` and `validate_standalone_proc_shell_name`
   bindings, and it has no `proc_wire_schema_version` binding. Phase `sase-1ab.8` then
   deferred the pin move because no flip commit existed upstream. The uncommitted
   37-file diff is preserved as artifact `file:explicit:b639458b308ef1c5aa57f797`, based
   on sase-core `b2e4ea6`. Read it with `sase artifact read`.
2. **The runtime cutover (`d5fc75864`, `sase-1ab.3`) corrupted legacy readers with
   mechanical renames.** `sase-1ab.8` already fixed three such proc readers in
   `25a7bd24f`. The landing audit found more; see the reader-repair section.
3. **24 deterministic test nodes still assert pre-rename spellings.** Two lint failures
   also belong to the rename: a mypy `name-defined` error, and four unused sunset-flag
   normalizers that Symvision masks behind another error today.
4. **Shell-concept prose remains in `src/`.** About 188 `gate shell` hits (some are
   user-visible output), about 57 other `<kind> shell` phrases, and the deferred
   follow-up identifier cluster from `sase-1ab.9` note #1.

Linked repos: the `sase-telegram`, `sase-github`, and `sase-research-artifacts` rename
commits are already on their `origin/master`. Chezmoi skills are current
(`sase skill init --check` is clean).

### Known foreign failures (do not chase)

The landing audit ran the full fast suite (`-n 16`) at `7b20f4c1c`: 39 failed, 48,428
passed. The failures below are **not** rename work and are already routed. Treat them as
KNOWN. Do not fix them here unless the fix is a one-line collateral of your own change.

- Finalizer epic `sase-1b2` (DISCOVERED ISSUE note filed):
  - mypy error at `ace/tui/models/agent_bundle.py:116`
  - Symvision private import `_root_represents_member`
  - two history-wire pins asserting index schema 33
    (`tests/core/test_agent_alias_history_wire.py`,
    `tests/core/test_agent_output_variable_history_wire.py`)
  - `tests/test_timezone_display_guard.py`, flagging `_agent_finalizer_receipt.py:45`
- Receipts epic `sase-1ah` (note filed):
  - `tests/test_config_schema_tools.py::test_project_sase_yml_matches_public_schema`
    (`receipt` key)
  - the `tool/receipt` and `tool/receipts` slots in
    `tests/completion/test_kind_coverage.py`
- Node Finder epic `sase-19i.7.3.3.3.3` (already noted there): the three mypy errors in
  `ace/tui/models/agent_groups/_tree.py` (lines 622-629).
- Task beads:
  - `sase-1as` (getting-started Grok wording)
  - `sase-13p` (TUI import budget, 3402 vs 3400)
  - `sase-1b8` (`test_expanded_overflowing_header_claims_half_page_scroll`)
  - `sase-1b7` (`bench_tui_jk.py` import)
  - `sase-1al`, `sase-18n`, `sase-16o`, `sase-1ba`, `sase-1bb` (visual and flaky nodes)
  - `sase-1b9` (telegram `%id` test)
  - `sase-1ay` (Symvision unused-public backlog). Its four `legacy_sase_shell_syntax`
    symbols **are** this epic's; reader-repair resolves them.
- A raw `pytest` run outside `tools/run_pytest` also fails these nodes, because the
  agent temp path is long and no home isolation applies:
  `test_suite_gate_scaled_integration` (AF_UNIX path too long),
  `test_dispatch_federation` IPC, the snippet CLI rich-path tests,
  `doctor/test_checks_beads`, `completion/test_candidates_project_providers`, and
  `test_vcs_xprompt_mru_pruning`. Verify through `sase tool run check`, not raw pytest.

### Process rules

- Read `sase/memory/lint_and_test.md` and, for Symvision work, `symvision.md` through
  `/sase_memory_read`. Run `just install` first in a fresh workspace. Verify with
  `sase tool run check` in every repo you change. Never run `just check-full`.
- Open linked repos only through `/sase_repo`, and read their `AGENTS.md` first.
- Phase workers record `PROPOSED FOLLOW-UP:` notes on their own phase bead instead of
  creating beads.
- Each repair keeps the parent plan's compatibility policy. Durable readers are
  permanent, unconditional, and live in named helpers or `LEGACY_*` constants. Authored
  syntax follows the `legacy_sase_shell_syntax` sunset flag. Test both flag states.

## Legacy reader and sunset-flag repair

Repo: sase.

Fix these confirmed defects, each with a regression test that loads a realistic
pre-rename input or exercises the writer:

- `src/sase/plan_chain.py`:
  - `LEGACY_AGENT_SESSION_TURN_KEY = "agent_session_turn"` is a bogus legacy constant
    holding the canonical key, and `AGENT_SESSION_TURN_KEY` aliases it.
    `_strip_legacy_shell_keys` pops it, so `set_agent_session_fields` drops the turn
    object on every call. `set_agent_session_fields({}, turn={...})` returns `{}` today.
    It also strips an existing `agent_session_turn` from metadata passed through
    `agent/_agent_session_promotion.py`, `agents_sync/inventory.py`,
    `axe/run_agent_directive_metadata.py`, and `axe/run_agent_helpers_artifacts.py`.
  - Delete the bogus constant, restore `AGENT_SESSION_TURN_KEY = "agent_session_turn"`,
    and keep the strip tuple to real legacy keys (`agent_session_shell`, `shell_kind`,
    then the agent-family keys).
  - Check whether `core/wire.py` imports the bogus name, and fix it there too.
  - Test that `set_agent_session_fields` writes and preserves `agent_session_turn` and
    drops only legacy keys.
- `src/sase/notification_gates/model_request.py` (about line 283): the durable
  normalization now reads `if continuation == "gate_turn": continuation = "gate_turn"`,
  a no-op. Before the cutover it mapped `"gate_shell"`. Restore it through
  `normalize_persisted_continuation_mode` for stored bundles, and route authored
  requests through the flag-gated `normalize_continuation_mode`. Keep the stored-bundle
  hash rule: never rewrite a stored envelope.
- `src/sase/procs/models/operations.py` (`ProcReserve.from_dict`, about line 74): the
  legacy origin `proc-shell` → `named-proc` mapping became `named-proc` → `named-proc`.
  Restore it and add a legacy-row test.
- `src/sase/main/gate_handler.py`: authored gate specs pass through
  `normalize_persisted_gate_spec_block`, which accepts a `"shell"` block
  unconditionally. Authored input must use the flag-gated `normalize_gate_spec_block`,
  which rejects with the flag off and names `"turn"`. Durable bundle readers keep the
  persisted variant.
- `src/sase/config/_settings_system.py`: with the flag off,
  `gate.shell.reclaim_grace_seconds` is silently ignored. The parent policy requires
  reporting it the way other unknown config keys are reported, naming
  `gate.turn.reclaim_grace_seconds`. Wire `normalize_reclaim_config` (or the config
  validation path's equivalent) so both flag states behave as the flag's registry
  sentences say.
- After these fixes, `normalize_continuation_mode`, `normalize_gate_spec_block`,
  `normalize_persisted_continuation_mode`, and `normalize_reclaim_config` have non-test
  consumers. Confirm `just symvision` no longer lists them. Private-import errors from
  other epics may mask that list; see the known foreign failures. Note on `sase-1ay`
  that this epic's four symbols are resolved.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_display_hint_sections.py:74` uses
  `LEGACY_NAMED_PROC_SECTION_ID` without importing it, a mypy `name-defined` error. The
  constant lives in `_agent_named_proc_section.py`. `render_named_proc_hint_document`
  raises `NameError` whenever `lane_fold_overrides` is a Mapping. Import it and add a
  test that reads a persisted legacy `proc-shell` fold id.

Then audit systematically. The corruption pattern is a mechanical rename applied to a
legacy literal, which turns a reader into a no-op or a writer into a stripper. Review
every hunk that touched a legacy spelling in the epic's sase commits:

- `c051b9a31`
- `4ef716648`
- `d5fc75864`
- `63d2bdcea` (docs only; skim)
- `d4c7b5ca9`
- `55e9e96de`
- `25a7bd24f`
- `eac55e929`

The legacy spellings to look for are `proc-shell`, `shell:`, `gate_shell*`,
`shell_name`, `shell_kind`, `agent_session_shell`, `family_shell`, `shell` blocks and
forks, `plan_shell_*` keys and files, `invalid_shell`, `missing_gate_shell_row`,
`create_gate_shell`, `dismissed_proc_shells.json`, and the `shells` and `proc-shell`
fold ids. Flag any of these patterns:

- `x == NEW` followed by `x = NEW`
- a `LEGACY_*` constant whose value is a new spelling
- a reader tuple that lists the same key twice or omits the legacy key
- a legacy-input test whose fixture was renamed to the new spelling, so it no longer
  tests the legacy path

For each durable surface in the parent policy, confirm a test loads the real legacy
shape:

- agent meta and `done.json`
- pending plan gate files and meta
- gate bundle envelope, cancel sources, and error codes
- `procs.jsonl` rows, including a live `shell:` concurrency conflict
- the dismissed-procs file migration
- fold ids
- runner-slot records

Record anything too large to fix here as a `PROPOSED FOLLOW-UP:`.

Exit: `sase tool run check` passes except the known foreign failures, and mypy reports
no rename-owned error.

## Rename-stale tests and CLI contracts

Repo: sase. For each node, decide whether the test or the product is wrong. The rename
made every node below stale (all fail on a clean tree at `7b20f4c1c`):

- `tests/agent/test_legacy_agent_family_syntax.py::test_gate_help_lists_only_canonical_next_fork_value`
  (expects `{session,shell,none}`)
- `tests/main/test_parser_proc.py`: `test_proc_run_help_documents_command_and_examples`,
  `test_proc_kill_help_documents_prefix_and_json`,
  `test_proc_show_help_documents_log_and_follow_options`,
  `test_proc_list_help_documents_every_filter_and_examples`, and
  `test_proc_run_and_list_parse_named_named_proc` (they expect `-N/--shell`, `.shell`,
  and "named proc shell")
- `tests/test_keybinding_footer_agent.py`: the three `*_advertises_shell_digits` nodes.
  Rename them for turn digits and assert `("0-9", "turn")`.
- `tests/test_gate_cli_show.py::test_show_rejects_neither_ref_nor_id_and_kind`
  (gate-turn reference wording)
- `tests/test_dynamic_agent_session_root_zero_suffix.py::test_generic_root_presents_session_container_name`
  (`AGENT SHELL` header)
- `tests/test_agent_session_wire_mirrors.py::test_scan_wire_json_emits_only_new_spellings`.
  It lists `agent_session_turn` as a legacy key. Replace it with `agent_session_shell`
  and `shell_kind`.
- `tests/test_procs_models_surface.py::test_models_all_matches_legacy_surface`. Decide
  the intended `procs.models` `__all__`; a `LEGACY_*` constant must hold a legacy value.
- `tests/telemetry/test_catalog.py::test_get_subsystems_order_matches_constant`
  (unexpected subsystem `Gate Turn`). Update the ordering constant or catalog, not
  metric names.
- `tests/main/test_init_skills_sources.py`: the `sase_gate` and `sase_questions`
  parametrizations expect `sase gate create --shell` and "question gate shell". Update
  the expected phrases to the turn wording that the skill sources now teach.
- `tests/test_agent_artifact_marker_mutation_audit.py` (three nodes) and
  `tests/test_agent_artifact_marker_path_passing_audit.py::test_tracked_marker_path_passing_sites_are_reviewed`:
  the reviewed-site tables still name `src/sase/shells/*` and other pre-rename paths and
  functions. Re-review each site at its new path; do not rubber-stamp.
- `tests/completion/test_snapshot.py` (two nodes): refresh
  `tests/completion/snapshots/cli_spec.json` with `tools/sync_completion_spec`, and
  inspect the diff.
- `tests/completion/test_kind_coverage.py`: caption `gate/create:turn_status` and
  `gate/create:turn_stop_status` in `src/sase/completion/kinds.py`, matching how the old
  shell status slots were captioned. The `tool/receipt*` slots in the same test belong
  to `sase-1ah`.
- `tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name`
  and `tools/check_sase_core_rs_bindings`. `src/sase/procs/store.py`
  `_reserve_request_schema_version` calls
  `require_rust_binding("proc_wire_schema_version")`, which no released or pinned core
  has, so the CI "Check pinned core bindings" step and the release-core-floor smoke
  fail. Make it an optional lookup that the collector does not count as required,
  following an existing optional-binding convention if `src/sase/core/rust.py` has one.
  It keeps the `PROC_WIRE_SCHEMA_VERSION` fallback.

Rerun `tests/test_contract_manifest.py`, `tests/test_sase_turn_terminology.py`, and
`tests/test_agent_session_terminology.py` after the changes.

Exit: every listed node passes, and `sase tool run check` shows no rename-owned failure.

## Land the sase-core contract flip

Repo: sase-core, a breaking `feat!:` change with a `BREAKING CHANGE:` footer. The parent
plan's "sase-core contract flip" section is the spec. The artifact diff is an
accelerator, not an authority: re-check every hunk against that spec.

- Re-apply the diff. A 3-way apply of the artifact onto `912331c` conflicts only in
  `crates/sase_core/src/agent_scan/index/storage.rs`. Master's finalizer work
  (`f4f2e96`) already took artifact-index schema 34 with
  `migrate_record_json_refresh_v34`. Keep that step. Move the `gate_shell_id` →
  `gate_turn_id` column rename to its own `prior_version < 35` step (rename the function
  to `..._v35` along with its test), and set `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION`
  to 35. The 3-way merge leaves the constant at 34. The migration must never move an
  archive-sized rebuild onto TUI startup.
- Confirm the emitted shapes and versions:
  - scan wire 11
  - index 35
  - proc wire 4, keeping older versions supported
  - fleet contract 7
  - runner capacity 7
  - hold 3
  - launch plan 3
  - proc dispatch 2
  - `FLEET_PROTOCOL_VERSION` 3 in both `fleet_contract/error.rs` and
    `sase_gateway/src/wire.rs`
  - the new `proc_wire_schema_version` binding
  - the two legacy binding names removed and asserted absent
- Also check the shapes master added since `b2e4ea6` (for example the `finalizer_status`
  scan field) for any new shell-named field.
- Regenerate `fleet_api_v1.json` (`UPDATE_FLEET_CONTRACT=1`) and
  `crates/sase_core/tests/fixtures/command_line/sase_spec.json` from the current sase
  CLI per that directory's `README.md`. Update `tests/python_wire_parity.rs` and the
  triage path fixture.
- Build this core and run current sase master against it with `sase tool run check` from
  a sase checkout whose extension was built from this core. Readers must accept the new
  spellings, and version mismatches must degrade, not crash. Fix any sase breakage in
  sase so that it works against both cores. Record every mirror constant that pin-bump
  must move as a note on this phase bead.
- **Landing is the exit criterion.** The previous attempt verified and then left the
  diff uncommitted. Declare a `commit` decision for the linked sase-core repo in your
  final declaration, and confirm with `git log origin/master` in sase-core that the flip
  commit is there before you declare the phase done. Put the commit SHA in the phase
  close note.

Exit: `sase tool run check` passes in sase-core, the sase-against-new-core run is green
apart from known foreign failures, and the flip commit is on sase-core `origin/master`.

## Core pin bump and mirrors

Repo: sase. This is the parent plan's "Core pin bump and mirrors" section, with one
constraint: `pyproject.toml` keeps `sase-core-rs>=0.35.0,<0.36.0` (the release lane owns
it). The published 0.35.0 core is pre-flip, so sase must keep working against both cores
until a later ratchet.

- Run `just ratchet-core-revision` to the landed flip commit, then `just install`.
- Move the mirrors core-flip listed:
  - `core/agent_scan_wire_records.py`: `AGENT_SCAN` 10→11, `INDEX` 34→35
  - `procs/models/common.py`: `PROC` 3→4
  - `dispatch/models.py`: `FLEET_PROTOCOL` 2→3
  - `core/agent_launch_wire_records.py`: `LAUNCH_PLAN` 2→3
  - any fleet-contract, runner, or hold mirrors
- Update `src/sase/core/health.py` and the `tools/validate_sase_core_rs` probes.
- Keep the dual-core `SUPPORTED_*` version sets and binding fallbacks that `55e9e96de`
  added, so the release-core-floor smoke still passes on 0.35.0. Keep every durable
  legacy reader and its legacy-input tests.
- Flip the fleet parity and projection fixtures to `agent_turn` / `historical_turn`:
  - `tests/ace/tui/test_fleet_agents_display_parity.py`
  - `tests/ace/tui/test_fleet_agents_projection_agent_session_tree.py`
  - any other fixture capturing core output keys (`row_kind`, `turn_id`, `turn_kind`,
    `proc_name`, lifecycle `named-proc`)
- Remove the pre-cutover `LEGACY_AGENT_SESSION_SHELL_KEY` reader bridge comment's
  "pinned pre-cutover core" rationale, but keep the reader itself. Pre-rename files on
  disk still carry the key.
- Record a `PROPOSED FOLLOW-UP:` for the post-release cleanup: once the published window
  moves past the flip release, drop the dual-core fallbacks and the
  `proc_wire_schema_version` optional lookup's fallback.

Exit: `sase tool run check` passes except known foreign failures, `sase core health` is
green, and `tools/check_sase_core_rs_bindings` passes against both the pinned and the
published core.

## Finish turn vocabulary in source

Repo: sase. This phase changes wording and names only; keep behavior identical.

- Rename the deferred follow-up cluster from `sase-1ab.9` note #1, with its fields and
  tests:
  - `ShellFollowupWorkspace`
  - `launch_shell_followup`
  - `resolve_shell_next_action`
  - `SUCCESSFUL_SHELL_FOLLOWUP_OUTCOMES`
  - the `shell_member_kind` / `shell_followup_agent` fields
  - the dead `_fork_target` `fork == "shell"` branch

  They live in `turns/followup.py`, `gate_turn/followup*.py`,
  `gate_turn/kind_next_action.py`, `monitor/followup.py`,
  `core/wait_dependency_resolution/*`, and the `monitor_state.py` mirror comment. Some
  of these fields are persisted or appear on the wire. For those, keep a named legacy
  reader and write only the new spelling. Remove the cluster's entries from the
  `tests/test_sase_turn_terminology.py` allowlist.

- Rewrite the concept prose in `src/`:
  - `git grep -niE 'gate[ -]shells?\b' -- src` (about 188 hits)
  - `git grep -niE '\b(agent|monitor|proc|session|sase) shells?\b' -- src` (about 57)
  - docstrings, comments, log lines, and error messages
  - user-visible output, for example `axe/run_agent_exec_gate.py` ("Gate shell: …",
    "handed the remaining decision to a gate shell")

  Change the chop-SDK log contracts (`tests/test_chop_sdk.py`) and any chat-history
  assertions together with their source strings. Keep every unrelated meaning listed in
  the parent plan. Where "agent shell" means a process environment, write "SASE agent
  process".

- Extend `tests/test_sase_turn_terminology.py` to fail on those concept phrases in
  `src/`, with a commented allowlist limited to named legacy readers and the sunset-flag
  module.
- If visible TUI copy changes, re-baseline only the affected PNG goldens through
  `/sase_monitor` (`just fix-tui-screenshots -- <selectors>`), and inspect each update.

Exit: the extended guard passes, `sase tool run check` passes except known foreign
failures, and every remaining `shell` hit in `src/` is an unrelated meaning, a named
legacy reader, or the flag branch.

## Acceptance audit and cross-repo closeout

Repos: sase, plus sase-telegram and chezmoi only if drift is found.

- Prove the parent epic's done criteria with targeted tests or fakey end-to-end
  exercises, and cite them in the phase note:
  - `sase gate create --turn --next-fork turn`
  - a monitor turn
  - `sase proc run -N <name>`, with JSON keys `proc_name` / `proc_role` and lifecycle
    `named-proc`
  - the TUI `SESSION TURNS` lane with the `AGENT TURN` / `GATE TURN` / `MONITOR TURN` /
    `NAMED PROC` headers
  - pre-rename agent meta, pending plan gates, gate bundles, proc rows, dismissed procs,
    and an index database at schema 33 or 34 all still load
- Sweep for new shell-concept wording (parent-plan vocabulary): check every commit on
  sase, sase-core, and sase-telegram since `2026-09-26 00:15` that is not this epic's or
  the parent's, and fix what you find.
- In sase-telegram, confirm `tests/test_gate_turn_settlement.py` passes against sase
  master. Then close task bead `sase-16v` with a note (`sase-1ab.6` note #2 says the
  rewrite resolved it). Leave `sase-1b9` alone.
- Run `sase skill init --check`, and redeploy per `generated_skills.md` if it reports
  drift.
- Run `sase core health`.

Exit: `sase tool run check` passes except known foreign failures in every repo this
phase changed, and the phase note lists the acceptance evidence.
