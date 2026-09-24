---
tier: epic
title: Runtime, syntax, and CLI cutover to agent session (runtime-cutover)
goal: 'Outside src/sase/ace, sase names the former agent-family concept "agent session"
  in every module, identifier, comment, message, and test. %id(..., session=), agent
  queries session:/kind:session, --next-fork session, gate spec "fork": "session",
  and SASE_AGENT_SESSION_ATTACH are the canonical user syntax. The retired spellings
  keep working only behind the legacy_agent_family_syntax sunset flag. CLI help, JSON
  output, the editor bridge, and the skill sources use the new vocabulary, and `sase
  tool run check` passes.

  '
phases:
- id: attach-modules
  title: Agent-session attach and promotion modules
  depends_on: []
  size: medium
  description: 'attach-modules: rename agent/_family_attach_{candidates,directives,launch,resolution,types}.py,
    _family_promotion.py, and family_attach.py to their _agent_session_* / agent_session_attach
    names. Rename their types, functions, and locals, the attach env-payload JSON
    keys (keeping named legacy readers), and the xprompt directive fields family_attach_parent/suffix
    and name_family_args. Update every importer, including ACE imports only, and rename
    the matching tests.'
- id: names-preview
  title: Name lookup, plan_chain, and plan preview
  depends_on:
  - attach-modules
  size: medium
  description: 'names-preview: rename the family-concept identifiers in agent/names/
    (find_agent_family, agent_family_base, reserved-name and forced-reuse helpers)
    and plan_chain.py, and delete the deprecated AGENT_FAMILY_* aliases. Rename agent_family_plan_preview.py
    to agent_session_plan_preview.py along with its types. Update every importer,
    including ACE imports only, and the tests.'
- id: agent-runtime
  title: Remaining agent package runtime identifiers
  depends_on:
  - names-preview
  size: medium
  description: 'agent-runtime: rename the family-concept identifiers, comments, and
    messages left in src/sase/agent/ after the attach and names phases. That covers
    launch_executor, launch_validation, detached_child, multi_prompt_launch_execution,
    launch_request_*, launch_hold_preview, wait_watch internals, and relaunch_prompt
    internals. User-facing syntax and wait -j output values are left to later phases.'
- id: lanes
  title: Axe, monitor, gate, shell, and bead lanes
  depends_on:
  - agent-runtime
  size: medium
  description: 'lanes: rename family-concept identifiers, constants (GATE_FAMILY_ROLE,
    MONITOR_FAMILY_ROLE, spawn_family_successor), comments, and messages in axe, monitor,
    monitor_state.py, gate_shell, plan_shell, question_shell, notification_gates,
    sudo, runner_slots (both trees), dispatch, shells, and bead, plus their tests.'
- id: core-history
  title: Core mirrors, chat fork, scripts, and remaining non-ACE packages
  depends_on:
  - lanes
  size: medium
  description: 'core-history: rename history/chat_fork/family.py and scripts/_agent_chat_from_name_family.py
    and the family-concept identifiers in src/sase/core (keeping core-emitted legacy
    values as named mirrors), history, scripts, stats, ops, sdd (except sidecar paths),
    llm_provider, workspace_provider, config/_settings_runner.py, sase_agent.py (except
    the families/ URL), main internals, xprompt internals, and the remaining top-level
    modules.'
- id: syntax-flag
  title: Canonical session syntax and the legacy_agent_family_syntax flag
  depends_on:
  - core-history
  size: medium
  description: 'syntax-flag: create the legacy_agent_family_syntax sunset flag with
    sase flag new. Make %id(<suffix>, session=<parent>), --next-fork session, gate
    spec "fork": "session", and SASE_AGENT_SESSION_ATTACH canonical. Route the retired
    spellings through one module that accepts them when the flag is on and rejects
    them with a replacement-naming error when it is off. Test both flag states.'
- id: query-cli-json
  title: Agent query dialect, CLI help, JSON output, and editor bridge
  depends_on:
  - syntax-flag
  size: medium
  description: 'query-cli-json: make agent query session:/kind:session canonical,
    with flag-gated family:/kind:family aliases. Update the CLI help text and flip
    the JSON output of sase agent list/search/index/wait -j and the sase editor bridge
    to agent_session keys and session kinds. Refresh the cli_spec snapshot, confirm
    sase-nvim does not branch on the old editor kinds, and declare a feat! breaking
    change.'
- id: skills-sweep
  title: Skill sources, leftover tests, and classification sweep
  depends_on:
  - query-cli-json
  size: medium
  description: 'skills-sweep: update the sase_run, sase_gate, sase_pipe, sase_questions,
    sase_monitor, and sase_agents_status skill sources and with_feedback.yml. Rename
    any family-named test files still in scope and update the shard-timing and flake
    baselines. Classify every remaining non-ACE famil hit, record hand-offs on sase-17m.4,
    and run sase tool run check.'
proposed_by: bbugyi200.athena.sase-17m.4
parent_bead: sase-17m.4
create_time: 2026-09-24 13:32:24
status: wip
bead_id: sase-17m.4.1
---

- **PROMPT:** [prompts/202609/agent_session_runtime_cutover.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_session_runtime_cutover.md)
- **PARENT:** [202609/agent_session_rename.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_session_rename.md)
- **BEAD:** [sase-17m.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17m/sase-17m.4.1.md)

# Plan: Runtime, syntax, and CLI cutover to agent session (runtime-cutover)

## Context

This epic implements the `runtime-cutover` phase (bead `sase-17m.4`) of the parent epic
"Rename agent family to sase agent session" (`plan:202609/agent_session_rename.md`, epic
`sase-17m`). The parent plan has the final say on vocabulary, identifier rules, the
meanings of "family" that must not change, and compatibility policy. Before you start
any phase here, read these parent sections: **Vocabulary**, **Identifier rules**,
**Meanings of "family" that must not change**, **Compatibility policy**, and **Runtime,
syntax, and CLI cutover**.

Repo: **sase** only. Do not edit sase-core, sase-telegram, or chezmoi. The one exception
is a read-only check of sase-nvim (see `query-cli-json`), done through `/sase_repo`.

### State of the world at planning time

- `wire-cutover` (`sase-17m.3`, child epic `sase-17m.3.1`) has landed. What it did:
  - It moved canonical metadata keys to `src/sase/plan_chain.py`: `AGENT_SESSION_KEY`,
    `AGENT_SESSION_ROLE_KEY`, `AGENT_SESSION_PARALLEL_KEY`, `AGENT_SESSION_SHELL_KEY`,
    `AGENT_SESSION_SEPARATOR`, and the `LEGACY_AGENT_FAMILY_*` constants with their
    accessors.
  - It renamed the Python wire mirrors, the `Agent` dataclass fields (`agent_session*`),
    the durable Python-owned JSON, and the name registry (schema v3).
  - `plan_chain.py` still keeps deprecated aliases (`AGENT_FAMILY_FIELD`,
    `AGENT_FAMILY_ROLE_FIELD`, `AGENT_FAMILY_PARALLEL_FIELD`, `AGENT_FAMILY_SEPARATOR`),
    and ACE still imports some of them. `names-preview` deletes them.
- sase-core still **emits legacy spellings** and accepts both until `core-contract`
  (`sase-17m.8`). Examples: the container kind `"family"` in fleet and agents-sync
  snapshots, `AgentSessionNameKind` values, relationship/stats/editor kinds from core,
  and `agent_family` columns in the artifact index. Python code that mirrors a
  **core-emitted** value keeps accepting that legacy value. Rename the Python identifier
  around it, but not the value core sends. Mark each such spot with
  `# legacy agent-family spelling: core emits "<value>" until core-contract`.
  Python-owned output (CLI JSON, editor bridge, Python-written durable JSON) uses only
  the new spelling.
- Scale: outside `src/sase/ace/`, about 2.2k case-insensitive `famil` hits in about 290
  src files, and about 2.5k hits in about 315 test files outside `tests/ace` and
  `tests/perf`. Many of these are unrelated meanings that must stay (`vcs_family`,
  `model_family`, the Patch revert family, `font-family`, and so on).
- The `sase-17m.4` bead note lists concrete hand-offs from wire-cutover. The phases
  below cover each one:
  1. the attach env payload and `FamilyAttachLaunchPlan` JSON
  2. `spawn_family_successor` and the family locals in agent/axe/monitor/gate_shell
  3. the xprompt directive fields `family_attach_parent`/`family_attach_suffix`
  4. the `sase agent list -j` keys
  5. `GATE_FAMILY_ROLE` and `MONITOR_FAMILY_ROLE`

  The launch-request `agent_meta.agent_family*` dotted-path context reads are already
  named legacy readers. Leave them as they are.

### Scope boundary with sibling phases of the parent epic

- `ace-cutover` (`sase-17m.5`) owns everything under `src/sase/ace/**` and
  `tests/ace/**`, `tests/perf/**`, `default_config.yml`, and `sase.schema.json`. That
  includes ACE module names, classes, row kinds, labels, and PNG goldens. Two exceptions
  here:
  - Every phase here may edit ACE files **only to follow a renamed import or symbol**.
  - `query-cli-json` also owns the shared agent query dialect files that the parent plan
    assigns to this phase: `src/sase/ace/query_profile/profiles/_agents_shared.py` and
    `src/sase/ace/tui/models/agent_live_query*.py`.
- `session-pages` (`sase-17m.9`) owns all of `src/sase/agents_sync/`, including
  `rendering_family_page.py`, the sidecar `families/` URLs in `sase_agent.py` and
  `sdd/hosted_links.py`, `inventory_history.py` path parsing, the sidecar templates, and
  the `tests/agents_sync` goldens. Here, touch agents_sync only to follow renamed
  imports.
- `telegram` (`sase-17m.7`) fixes sase-telegram's `find_agent_family` import. Do not
  keep a sase-side alias for it.
- `docs-memory` (`sase-17m.6`) owns `docs/` and `sase/memory/`. Do not edit either here.
- `core-contract` owns flipping the core-emitted values and the Rust schema versions.

### Rules for every phase

- Follow the parent's identifier rules exactly:
  - Use `agent_session`/`AgentSession`/`AGENT_SESSION` inside identifiers, JSON keys,
    and module names. Never use a bare `session` there, because that word already means
    provider transcripts, TUI sessions, question sessions, and so on.
  - Use a bare `session` for enum/string values and for the short tokens users type.
- **No internal aliases** to shrink the diff. When you rename a module or public symbol,
  update every importer in the same phase: src, tests, ACE (imports only), tools, and
  demos. Check with `git grep` for the old name.
- Rename comments, docstrings, log messages, error messages, and test names along with
  the code. Test functions and helpers that exist only to prove legacy input still loads
  keep a `legacy` word in their names.
- Durable-data legacy readers from wire-cutover are permanent. Do not remove or rename
  their `LEGACY_*` constants or `legacy_*` helpers. You may rename a caller's local
  variable around them.
- If you rename a test file, update its test IDs in `tests/shard_timings.json` and
  `tests/reproducible_flake_baseline.txt`.
- Never change an unrelated meaning of "family" (see the parent plan's list).
- Changes outside a phase's package list are limited to following imports. If you find a
  concept hit outside your list that no later phase here owns, fix it when it is small.
  Otherwise, leave it for `skills-sweep`.
- Symvision: a rename must not leave a public symbol unused. If a renamed public helper
  loses its last consumer, delete it instead of adding an `--epic-symbol` entry. Run
  `sase bead epic-symbols <your phase bead>` before closing.
- Verification:
  - Run `just install` first in a fresh workspace.
  - Then run `just fix`, then `sase tool run check`. If it may outlast your turn, run it
    through `/sase_monitor`.
  - Never run `just check-full` unless someone explicitly tells you to.
  - Read `sase/memory/lint_and_test.md` before you finish.
- Commit subjects are `refactor(agent-session): … (<phase bead id>)`. The only exception
  is `query-cli-json`, which uses `feat(agent-session)!:` (see below).
- Record out-of-scope discoveries as `PROPOSED FOLLOW-UP:` notes on your own phase bead.
  Do not create beads. The single exception is the flag bead that `sase flag new`
  creates in `syntax-flag`. The parent plan and the flag rules require that bead.

## Phase: attach-modules — Agent-session attach and promotion modules

Packages: `src/sase/agent/_family_attach_*.py`, `_family_promotion.py`,
`family_attach.py`, plus the xprompt directive plumbing that carries attach fields.

1. Rename the modules with `git mv`:
   - `agent/_family_attach_{candidates,directives,launch,resolution,types}.py` →
     `agent/_agent_session_attach_{…}.py`
   - `agent/_family_promotion.py` → `agent/_agent_session_promotion.py`
   - `agent/family_attach.py` → `agent/agent_session_attach.py`
2. Rename their family-concept symbols. Examples:
   - `FamilyAttachError`, `FamilyAttachLaunchPlan`, `FamilyAttachSibling`,
     `FamilyAttachDirective`, `FamilyCandidate`, `_FamilyAttachPlanResolver` →
     `AgentSessionAttach*`, `AgentSessionCandidate`, …
   - `resolve_family_attach_plan`, `load_family_attach_plan_from_env`,
     `prepare_family_attach_launch`, `extract_family_attach_directive`,
     `build_family_attach_sibling_from_spawn`, `promote_agent_to_family`,
     `convert_registered_agent_to_family`, `promote_family_parent_for_attach`,
     `normalize_family_suffix_arg`, `default_with_feedback_parent_from_family_attach` →
     their `agent_session` equivalents
   - locals such as `family_attach_plan`, `pending_family_parents`, `family_parent`,
     `family_suffix`, `family_candidate`
   - `ParsedNameDirective.family_parent`/`family_suffix`
3. Attach env payload: the JSON that `FamilyAttachLaunchPlan` writes into the attach
   environment variable must use `agent_session_role`,
   `parent_agent_session_member_name`, and `parent_agent_session_role_suffix`. The
   loader reads the new key first, then the legacy key, through one named helper marked
   `# legacy agent-family spelling`. Keep the **variable name**
   `SASE_AGENT_FAMILY_ATTACH` in this phase, but name the Python constant for what it is
   (for example `LEGACY_AGENT_FAMILY_ATTACH_ENV`). `syntax-flag` introduces
   `SASE_AGENT_SESSION_ATTACH`.
4. xprompt directive fields: in `xprompt/_directive_types.py`, `_directive_extract.py`,
   and `_directive_collect.py`, rename
   - `family_attach_parent`/`family_attach_suffix` → `agent_session_attach_parent`/
     `agent_session_attach_suffix`
   - `name_family_args` → `name_agent_session_args`

   Update their readers: `agent/launch_validation.py`, `agent/relaunch_prompt.py`,
   `agent/launch_hold_preview.py`, and the axe/ACE callers. Where these fields cross
   into the launch wire mirrors under `src/sase/core`, map at that boundary. Do not
   change any wire key that wire-cutover already settled.

5. Do **not** change the `family=` keyword that `parse_name_directive_args` accepts, or
   the user-facing error text about it. That is `syntax-flag`'s job. Internal wording in
   those messages that is not about the keyword may change.
6. Tests: rename
   - `tests/_dynamic_agent_family_attach_helpers.py`
   - `tests/test_dynamic_agent_family_attach_{directives,executor,inbatch_launch,metadata,resolution}.py`
   - `tests/test_dynamic_agent_family_root_zero_suffix.py`
   - `tests/test_parallel_agent_family_{launch,metadata}.py`

   Use `agent_session` names, update the baselines, and keep or add a legacy env-payload
   test that loads a pre-rename payload.

## Phase: names-preview — Name lookup, plan_chain, and plan preview

Packages: `src/sase/agent/names/`, `src/sase/plan_chain.py`,
`src/sase/agent_family_plan_preview.py`.

1. `plan_chain.py`:
   - Rename the remaining family-concept helpers: `agent_family_base`,
     `agent_family_role_for_suffix`, `agent_family_suffix_token`,
     `allocate_agent_family_child_suffix`, `is_agent_family_member`,
     `agent_family_phase_name`, and the private `_…family…` helpers.
   - Delete the deprecated `AGENT_FAMILY_FIELD`, `AGENT_FAMILY_ROLE_FIELD`,
     `AGENT_FAMILY_PARALLEL_FIELD`, and `AGENT_FAMILY_SEPARATOR` aliases. Move every
     user to the `AGENT_SESSION_*` or `LEGACY_AGENT_FAMILY_*` constant it actually
     means.
   - Keep `LEGACY_AGENT_FAMILY_*` and `strip_legacy_agent_family_keys`.
2. `agent/names/`:
   - Rename `find_agent_family` → `find_agent_session`, plus `AgentFamilyMember`,
     `AgentFamily`, `_AgentFamilySnapshot`, `allow_reserved_family_separator_names`,
     `get_reserved_family_names(_for_display)`, `_AgentNameFamilyCollisionError`, the
     forced-reuse `_wipe_families_for_forced_reuse`, and the lookup-group, resolution,
     registry-scan, and group-mutation locals.
   - Keep `LEGACY_AGENT_FAMILY_CONTAINER_KIND` and the other named legacy readers.
   - The user-facing `%i(suffix, family=parent)` hint in `names/_registry.py` stays
     until `syntax-flag`.
3. Rename `agent_family_plan_preview.py` → `agent_session_plan_preview.py` and its
   symbols: `AgentFamilyPlanPreview`, `AgentFamilyPlanPreviewKind`,
   `FamilyPlanPreviewResult`, `agent_family_plan_preview_*`, and `_FamilyPlanMember`.
   Update its importers in ACE (imports only),
   `integrations/_editor_helper_agent_plans.py` (internal names only; the editor output
   keys belong to `query-cli-json`), and `agents/_restart_render.py`. The visible
   "Family" row label there is CLI copy for `query-cli-json`.
4. Tests: rename `tests/test_agent_family_plan_preview.py` →
   `tests/test_agent_session_plan_preview.py`, update the name-lookup and plan_chain
   tests, and update the baselines.

## Phase: agent-runtime — Remaining agent package runtime identifiers

Packages: everything left under `src/sase/agent/` after the two phases above. Examples:

- `launch_executor.py`, `launch_validation.py`, `detached_child.py`,
  `multi_prompt_launch_execution.py`
- `launch_request*.py`, `launch_hold_preview.py`
- `wait_watch/` internals, and `relaunch_prompt.py` internals such as `facing_family`
  and `_rewrite_family_member_or_preserve_clan`

1. Rename the family-concept identifiers, comments, and messages.
2. Leave these for `syntax-flag`:
   - `relaunch_prompt.py`'s emitted `family=` rewrite text and its user-facing errors
   - the `%id` keyword parsing
3. Leave these for `query-cli-json`:
   - `wait_watch/_types.py`'s `FAMILY = "family"` target kind (its value is
     `sase agent wait -j` output)
   - the other `wait -j` output
4. Tests: update the tests that cover these modules, and rename any family-named test
   file whose subject is this code.

## Phase: lanes — Axe, monitor, gate, shell, and bead lanes

Packages:

- `src/sase/axe/`, `src/sase/monitor/`, `src/sase/monitor_state.py`
- `src/sase/gate_shell/`, `src/sase/plan_shell/`, `src/sase/question_shell/`
- `src/sase/notification_gates/`, `src/sase/sudo/`
- `src/sase/runner_slots/` (if present) and `src/sase/core/runner_slots/`
- `src/sase/dispatch/`, `src/sase/shells/`, `src/sase/bead/`

1. Rename the constants:
   - `GATE_FAMILY_ROLE` → `GATE_AGENT_SESSION_ROLE` (in both `gate_shell/state.py` and
     `core/runner_slots/_admission_predicates.py`)
   - `MONITOR_FAMILY_ROLE` → `MONITOR_AGENT_SESSION_ROLE`
   - `spawn_family_successor`/`spawn_shell_family_successor`,
     `create_family_shell_member`, `read_family_monitor_marker`, `family_lanes`,
     `_family_handoff_state`, and the other family-concept identifiers, comments, and
     messages

   The role _values_ (`"gate"`, `"monitor"`) do not change.

2. Keep `notification_gates/model_shell.py`'s `LEGACY_GATE_SHELL_NEXT_FORK` durable
   reader. A stored gate descriptor with `"family"` must always load, whatever the flag
   says. The CLI `--next-fork` choices and user-authored gate specs belong to
   `syntax-flag`.
3. `bead/cli_work_cleanup_targets.py`:
   - Rename the family-concept identifiers.
   - A `membership == "family"` comparison against a core-emitted or registry value
     keeps accepting the legacy value through a named helper.
4. Tests: update the covering tests. Rename these and update the baselines:
   - `tests/test_axe_chop_wait_checks_plan_families*.py`
   - `tests/test_run_agent_runner_slot_capacity_family.py`

## Phase: core-history — Core mirrors, chat fork, scripts, and remaining non-ACE packages

Packages:

- `src/sase/core/` (wait-dependency resolution, artifact relations/layout,
  artifact-index lifecycle, `wire.py`, `agent_identity_facade.py`, cleanup/archive/hold
  mirrors)
- `src/sase/history/`, `src/sase/scripts/`, `src/sase/stats/`, `src/sase/ops/`
- `src/sase/sdd/` (except sidecar path code), `src/sase/llm_provider/`,
  `src/sase/workspace_provider/`, `src/sase/config/_settings_runner.py`
- `src/sase/sase_agent.py` (except the `families/` URL)
- `src/sase/main/` internals (not parser help text), `src/sase/xprompt/` internals (not
  directive syntax)
- `src/sase/continuation_baseline.py` and any remaining top-level module

1. Rename the modules:
   - `history/chat_fork/family.py` → `history/chat_fork/agent_session.py`
   - `scripts/_agent_chat_from_name_family.py` →
     `scripts/_agent_chat_from_name_agent_session.py`

   Also rename their symbols, for example `ForkFamilyMemberSource`,
   `ForkExcludedFamilyMember`, and `format_family_fork_source`.

2. `src/sase/core` mirrors of core-emitted values keep the legacy value and accept both.
   Rename the Python member or identifier only where you can do it without changing the
   value that is compared or sent. Examples: `AgentSessionNameKind.FAMILY`,
   `artifact_relation_layout` kinds, and index column names. Mark each with the
   core-contract comment from **Context**. Record the list in a `PROPOSED FOLLOW-UP:`
   note for `core-contract`, so its flip also renames these members.
3. `stats/`: keep the `LEGACY_RUNTIME_GROUP_BY` reader, and rename the remaining
   identifiers.
4. `llm_provider/` and `workspace_provider/`: most hits are probably `model_family` or
   `vcs_family`. Change only concept hits.
5. Tests:
   - Rename `tests/test_agent_chat_from_name_family{,_gate,_monitor}.py`.
   - Update the chat-fork, wait-dependency, stats, and ops tests.
   - Update the baselines.

## Phase: syntax-flag — Canonical session syntax and the legacy_agent_family_syntax flag

1. Read `sase/memory/sase_flags.md`, then create the flag. The three sentences are
   verbatim from the parent plan:

   ```
   sase flag new legacy_agent_family_syntax -k sunset \
     --when-enabled "SASE silently accepts the retired agent-family spellings as aliases of their agent-session replacements: %id(..., family=...), the family:/kind:family agent queries, --next-fork family, gate \"fork\": \"family\", and SASE_AGENT_FAMILY_ATTACH." \
     --when-disabled "SASE rejects those spellings with an error naming the agent-session replacement and ignores the legacy environment variable." \
     --remove-when "No maintained prompt, xprompt, skill, saved query, gate spec, or config in the sase-org repos or chezmoi still uses a retired agent-family spelling, and a sase release with agent-session syntax has shipped."
   ```

   - Paste the printed registry entry into `src/sase/feature_flags/registry.py`.
   - Its description must also say: "Removing this flag also removes sase-core's hidden
     `family` `%id` keyword alias."

2. Put every flag branch in **one** module, for example
   `src/sase/agent/legacy_agent_family_syntax.py`. That way, removing the flag means
   deleting this module's Off branch and the aliases. It owns:
   - `legacy_agent_family_syntax_enabled()`, which resolves through the feature-flag
     resolver
   - one error helper that builds `"<legacy> is retired; use <replacement>"`-style
     messages
   - the normalizers used below (`%id` keyword, `--next-fork`/gate-spec fork value, env
     var, and later the query aliases)
3. `%id(<suffix>, session=<parent>)`:
   - In `_agent_session_attach_directives.py`, accept `session` as the canonical
     keyword. Accept `family` only through the flag module: normalize it to `session`
     when enabled, and raise the replacement-naming error when disabled.
   - `clan=`/`session=`/`tribe=` stay mutually exclusive, and supplying both `session=`
     and `family=` is an error.
   - Update every user-facing message and generated text to say `session=`:
     - `names/_registry.py`
     - `relaunch_prompt.py`: rewrite emission and errors
     - `xprompt/_directive_collect.py`
     - `xprompt/_directive_edit_identity.py`: must emit `session=`
     - `xprompt/workflow_loader_definition.py`
     - any Python completion or snippet token source (grep `src/sase/completion` and
       `src/sase/xprompt` for `family=`)
   - The Rust `%id` parser from core-expand already accepts both keywords. Python
     enforces the flag after parsing.
4. `sase gate create -f/--next-fork {session,shell,none}` in `main/parser_gate.py`:
   - Help and choices show only `session`.
   - Accept `family` through the flag module. Do not list it in `choices`: use a custom
     type/normalizer so help and completion never show it.
   - User-authored gate specs with `"fork": "family"` go through the same normalizer at
     the spec-parse boundary. Find it by grepping `notification_gates`, `gate_shell`,
     and `main/gate*` for the `next`/`fork` spec parsing.
   - The durable descriptor reader (`LEGACY_GATE_SHELL_NEXT_FORK`) stays unconditional.
5. `SASE_AGENT_SESSION_ATTACH`:
   - Writers set only `SASE_AGENT_SESSION_ATTACH`.
   - The loader reads it first. When it is absent and the flag is enabled, it reads
     `SASE_AGENT_FAMILY_ATTACH`, so a parent launched before an upgrade still hands off.
     When the flag is disabled, it ignores the legacy variable.
   - Also scrub or propagate the new name wherever the old one is cleared or copied
     (grep for the old name in `src/`).
6. Tests, for every surface above, in **both** flag states:
   - With the flag enabled, the legacy spelling works and behaves identically to the new
     one.
   - With the flag disabled, the legacy spelling is rejected with a message naming the
     replacement, and the legacy env var is ignored.
   - The canonical spellings work in both states.
   - Rename `tests/test_directives_family.py` and update the `%id` directive tests to
     use `session=` as the primary form.
   - Use the existing feature-flag test fixtures. Grep `tests/` for `FeatureFlag.` usage
     to find them.
   - Run `tools/check_feature_flags` (it is part of `just check`) so the registry and
     the bead agree.

## Phase: query-cli-json — Agent query dialect, CLI help, JSON output, and editor bridge

Read `sase/memory/cli_rules.md` first.

1. Agent query dialect:
   - Rename the field `family` → `session`, and the kind value `family` → `session`, in:
     - `src/sase/ace/query_profile/profiles/_agents_shared.py` (`AGENT_KIND_VALUES`,
       field spec, hints)
     - `src/sase/agents/catalog/_query.py`, `_derive.py`, and `_models.py` (rename
       `AgentCatalogRow.family` → `agent_session`)
     - `agents/catalog/_family.py` → `agents/catalog/_agent_session.py`
       (`family_and_role` → `agent_session_and_role`)
     - `enrich_catalog_families`
     - `src/sase/ace/tui/models/agent_live_query.py` and `agent_live_query_pushdown.py`
   - `agents/catalog/_derive.py` currently re-emits `"family"` for registry or core
     container kinds. Emit `"session"`, and keep accepting both inputs.
   - Before flipping the catalog kind, check whether `agents_sync` consumes catalog
     kinds. If it does, map back to `"family"` at the agents_sync boundary for
     `session-pages` to remove, and record that in a follow-up note.
   - Legacy aliases:
     - Add one normalizer in the flag module that rewrites a parsed query's `family`
       field to `session` and `kind:family` to `kind:session`. It must work on the
       parsed query AST or tokens (`src/sase/ace/query/parser.py` / `tokenizer.py`), not
       with a raw-text regex.
     - Apply it at every agent-query entry point: `sase agent search`
       (`agents/cli_search.py`), the ACE Agents live query, and any saved-query load
       path that feeds the agents profile.
     - With the flag off, return a query error naming `session:`/`kind:session`.
     - Completion and hints show only `session`.
   - Update the `parser_agent_search.py` example to `session:"research.12"`.
2. CLI help text:
   - Update the family wording in `main/parser_agent_lifecycle.py`,
     `parser_agent_hold.py`, `parser_monitor.py`, `parser_gate.py`, `parser_pipe.py`,
     `pipe_handler.py`, `plan_show_render.py`, and `init_project_scope.py`.
   - Rename the "Family" row in `agents/_restart_render.py` to "Session".
   - Keep options sorted, and keep short aliases.
3. JSON output (Python-owned, new spellings only):
   - `sase agent list -j`: `agent_family`/`agent_family_role` → `agent_session`/
     `agent_session_role` (`agents/cli_list.py`)
   - `sase agent search -j`: `family` → `agent_session`, kind value `session`
     (`agents/cli_search.py`)
   - `sase agent index`: `dismissal_family_rows_*`/`dismissed_family_rows_*`/
     `dismissed_family_candidate_rows` → `dismissed_agent_session_*`
     (`agents/cli_index.py`)
   - `sase agent wait -j`: target kind `session` (`agent/wait_watch/_types.py`)
   - `sase editor` bridge: `kind: "session"`, `agent_session*` keys, and the
     `session · N members` detail text (`integrations/_editor_helper_agents.py`,
     `_editor_helper_agent_plans.py`)
   - Rename `tests/test_editor_helper_family_catalog.py`.
4. Refresh `tests/completion/snapshots/cli_spec.json` with its documented regeneration
   command, and review the diff.
5. sase-nvim: open it with `/sase_repo` (read-only) and grep for the old editor kinds
   and keys (`"family"`, `agent_family`).
   - If it does not branch on them, say so in your close note.
   - If it does, record a `PROPOSED FOLLOW-UP:` note. Do not edit sase-nvim here.
6. The commit is breaking. In your `/sase_final` declaration:
   - Use the subject
     `feat(agent-session)!: agent-session query syntax, CLI help, and JSON output (<phase bead id>)`.
   - Add a `BREAKING CHANGE:` footer that lists the changed JSON keys and values for
     `agent list -j`, `agent search -j`, `agent index`, `agent wait -j`, and the editor
     bridge, plus the new query field and kind.
7. Tests:
   - query tests for `session:`/`kind:session`
   - both flag states for `family:`/`kind:family`
   - JSON-output tests that assert the new keys and that no `family`/`agent_family` key
     is emitted

## Phase: skills-sweep — Skill sources, leftover tests, and classification sweep

1. Read `sase/memory/generated_skills.md`. Then update these sources in
   `src/sase/xprompts/skills/` to the new vocabulary and syntax: `sase_run.md`,
   `sase_gate.md` (including `"fork": "session"` and `--next-fork session`),
   `sase_pipe.md` (including its frontmatter description), `sase_questions.md`,
   `sase_monitor.md`, and `sase_agents_status.md`. Also update
   `src/sase/xprompts/with_feedback.yml`.
   - Leave the Patch meaning in `sase_patches.md`.
   - Preview with `sase skill init --diff`. Do **not** deploy to chezmoi.
   - Fix skill-rendering or golden tests that assert the old text.
2. Rename every family-named test file that is still in scope and that earlier phases
   did not rename. Examples:
   - `tests/test_agent_loader_status_override_{gate_shell_family,monitor_family,parallel_family,promoted_plan_family,question_families}.py`
   - `tests/test_agent_loader_dedup_pid_families.py`

   Update the family-concept test names inside them, and update the baselines.
   `tests/agents_sync/goldens/*family*` stay for `session-pages`.

3. Classification sweep. Run:

   ```
   git grep -n -i famil -- src tests tools demos ':!src/sase/ace' ':!tests/ace' ':!tests/perf'
   ```

   Every hit must fall into one of these groups:
   - (a) an unrelated meaning from the parent list
   - (b) a named legacy reader, constant, fixture, or test, including the core-emitted
     mirrors marked for `core-contract`
   - (c) the `legacy_agent_family_syntax` flag module, its callers' alias handling, and
     its tests
   - (d) the agents-sidecar path and publication code left for `session-pages`
     (`agents_sync/`, the `families/` URLs, and their goldens)

   Fix small stragglers in place. Put everything else in a `PROPOSED FOLLOW-UP:` note on
   your phase bead. Also confirm with `git grep -n -iE 'agent[ _-]session'` that every
   hit means the new concept.

4. Record hand-offs as notes on the parent phase bead `sase-17m.4`, using
   `sase bead note sase-17m.4 'HAND-OFF: …'`:
   - for `core-contract`: the core-emitted mirror members to rename after the flip
   - for `ace-cutover`: any ACE-side identifiers still importing old names, if there are
     any
   - for `telegram`: the renamed `find_agent_session` and attribute names
   - for `docs-memory`: the final syntax, JSON keys, and flag name
5. Run `sase tool run check`. Confirm that `sase bead epic-symbols` reports no leftover
   entries for this epic's phases.
