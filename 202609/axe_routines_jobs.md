---
tier: epic
title: Rename public AXE lumberjacks and chops to routines and jobs
goal:
  Make routines and jobs the consistent public AXE vocabulary while preserving
  scheduling behavior and existing runtime identities.
phases:
  - id: config_contract
    title: Shared configuration names and compatibility contract
    depends_on: []
    description:
      "config_contract: implement Rust-owned public configuration name translation,
      exact source provenance, and the contract rollout flag."
    size: medium
  - id: job_authoring
    title: Public job scripts and SDK
    depends_on:
      - config_contract
    description:
      "job_authoring: add canonical executable, SDK, context, and environment access for
      job authors without replacing the existing execution engine."
    size: medium
  - id: cli_contract
    title: Commands, structured output, and reference presentation
    depends_on:
      - job_authoring
    description:
      "cli_contract: expose axe routine and axe job commands, public JSON projections,
      diagnostics, and job reference spelling while preserving stored identities."
    size: medium
  - id: config_tui
    title: Canonical configuration and AXE presentation
    depends_on:
      - cli_contract
    description:
      "config_tui: connect public configuration views and editors, update defaults and
      schema, and rename visible AXE and automation-tribe text."
    size: medium
  - id: integrations
    title: Telegram scripts and maintained operator configuration
    depends_on:
      - config_tui
    description:
      "integrations: update Telegram public entrypoints and documentation, maintained
      chezmoi configuration, and generated shell completions."
    size: medium
  - id: documentation
    title: Current documentation, glossary, and visual examples
    depends_on:
      - integrations
    description:
      "documentation: update maintained guides, executable examples, glossary
      definitions, and current visual assets to routines and jobs."
    size: medium
  - id: acceptance
    title: Combined contract and upgrade verification
    depends_on:
      - documentation
    description:
      "acceptance: verify the combined rename, legacy-input compatibility, unchanged AXE
      behavior, generated outputs, and repository checks."
    size: medium
proposed_by: bbugyi200.athena.0l8.r0
create_time: 2026-09-15 15:18:39
status: wip
---

- **PROMPT:**
  [prompts/202609/axe_routines_jobs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/axe_routines_jobs.md)

# Rename public AXE lumberjacks and chops to routines and jobs

## Outcome, tier, and boundaries

AXE runs **jobs** in independently supervised **routines**. One execution is a **job
run**. A routine retains its interval, concurrent execution, guards, timeouts, process
supervision, restart policy, and metrics. This is a terminology and public-interface
change; it does not redesign scheduling or agent admission.

The **AXE** tab, `sase axe` command root, `axe` config root, and existing AXE branding
remain. Keep the preceding `sase tui` rename, including its startup options and `axe`
tab selector. Do not revive the canceled proposal's `schedule` command, Schedule tab,
`tui` config root, or unrelated ACE internal renames. Do not reopen or attach this work
as a child of canceled bead `sase-113`.

Use an epic because the requested public scope includes authored configuration,
extension APIs, and structured output as well as terminal labels. Rust must own shared
translation, and plugins and generated interfaces need coordinated verification. Each
phase is bounded, direct implementation work of size medium. Authoring this plan is
xlarge work. Dependencies are intentionally serial to keep the contract changes
reviewable and avoid overlapping edits across these closely connected surfaces.

Background: canceled `sase-113` and its archived plan
`plan:202609/schedule_tui_interfaces.md` were inspected through audited commands. Its
broader scope is not adopted. The preceding `0l8` transcript and this checkout show the
public `sase tui` rename already present. Recheck the current tree before implementing
so work from other rename agents is preserved.

All paths below are relative to their named repository. Open non-primary repositories
with `/sase_repo`; use only the returned paths. Main implementation repositories are
`sase` and `sase-core`; actual additional public matches were found in `sase-telegram`
and `chezmoi`. The inspected `sase-github`, `sase-nvim`, and `sase-research-artifacts`
sources currently have no matching maintained references; recheck them at integration
time and edit only if new relevant matches exist.

## Public contract

| Surface                       | Canonical spelling                                                                                                                                                  | Boundary treatment                                                                     |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Command groups                | `sase axe routine`, `sase axe job`                                                                                                                                  | Keep existing handlers and private argparse attributes where useful.                   |
| Routine children              | `list`, `run`, `status`                                                                                                                                             | Bare group delegates to `list`; `run` remains a foreground routine process.            |
| Job children                  | `doctor`, `list`, `run`                                                                                                                                             | Bare group delegates to `list`; `run` executes one job once.                           |
| Job options                   | `-L/--routine`, `-V/--job-verbose`                                                                                                                                  | Preserve existing short options and all other option semantics.                        |
| Authored AXE configuration    | `axe.routines`, nested `jobs`                                                                                                                                       | Translate structural keys to the existing internal model per source layer.             |
| Related configuration         | `job_timeout`, `job_script_dirs`, `routine_log_max_bytes`, `routine_log_temp_max_age_seconds`, `routine_restart_backoff_max_seconds`, `verbose_routine_diagnostics` | Rename only fields in their defined structural contexts.                               |
| Script authoring              | `sase_job_*`, `sase.jobs`, `SASE_JOB_*`                                                                                                                             | Thin public access to existing implementations and invocation protocol.                |
| Script routine identity       | `SASE_JOB_ROUTINE`, context `routine_name`                                                                                                                          | Preserve equivalent legacy input/access for installed scripts.                         |
| Diagnostics                   | `sase doctor -C axe.jobs`                                                                                                                                           | Accept the old filter as a compatibility alias; normal listings use the new filter.    |
| Automation tribe              | `job`, displayed `@job`                                                                                                                                             | Resolve to the existing built-in automation identity; preserve historical assignments. |
| Virtual automation references | `job:<routine>/<job>`                                                                                                                                               | Public spelling for the same stored `chop:` link identity.                             |
| Human presentation            | routine, job, job run                                                                                                                                               | Includes help, errors, logs, notifications, editors, clipboard output, and docs.       |
| Public JSON                   | routine/job field names on canonical output                                                                                                                         | Version public projections separately from internal wire and persisted records.        |

User-editable keys, documented SDK names, executable names, environment variables,
diagnostic selectors, and public JSON are included because users author or consume them.
A filename, raw path, stored identity, or arbitrary user text is not changed just
because it contains an old word.

Keep `src/sase/ace`, `src/sase/axe`, `src/sase/chops`, existing Python classes and
functions, Rust modules/binding names, widget IDs, CSS selectors, action IDs, operation
IDs, state-directory names, PID/lock paths, schema-v1 internal wire shapes, run IDs, and
recorded agent names. Do not rewrite `.chop.` segments in existing agent names, stored
tribe assignments, historical output, or stored artifact/link keys. Preserve
user-selected routine/job names, arbitrary map keys, executable paths, and free text.
Unrelated CI jobs, shell jobs, and Python coroutines are outside scope.

## Compatibility and rollout decisions

1. Canonical help, completion, new examples, and ordinary human output use routines and
   jobs. Keep `lumberjack`/`chop` command groups and old long options as hidden input
   aliases for installed callers. Capture invoked spelling before normalizing dispatch.
   Old aliases must not leak through argparse metavars or completion choices.
2. Accept old and new configuration names in every layer. Existing user config must
   continue overriding renamed defaults with identical semantics. Do not auto-rewrite
   live files, add a general migration command, or require a coordinated machine-wide
   cutover. Update tracked operator configuration explicitly in phase 5.
3. Add canonical script entrypoints and SDK/env/context access while retaining the old
   entrypoints and protocol. This is additive boundary work, not a module move.
4. Read `sase_flags.md` and use `sase flag new` after approval for one sunset flag,
   provisionally `axe_routine_job_contract`; first check for an equivalent existing
   flag. The enabled/default branch selects canonical effective config projections and
   version-2 public JSON. The disabled branch retains the previous config projection and
   JSON contract for consumers still migrating. Both accept new and old inputs; human
   nomenclature remains canonical. Record these exact branches and both-state checks in
   the generated flag dossier. Flag resolution must not depend on a config normalization
   step that itself needs the flag.
5. Hidden legacy `chop` CLI invocations retain their schema-v1 JSON response. Canonical
   `job` invocations use v2 when the flag is enabled and v1 when disabled. The unchanged
   `sase axe status --json` uses v2 when enabled and v1 when disabled. Configuration
   source/raw views always show actual source text, regardless of the flag.
6. The sunset flag's removal condition is verified adoption of canonical config/output
   by maintained consumers and completion of both-state upgrade tests. Keep it through
   this rollout. Its later removal deletes the Off projection branch; retiring accepted
   historical command/config/script inputs is a separate explicitly approved change.
7. Do not land an unfinished public surface independently. Stage the phases together for
   combined acceptance. If host workflow requires early user-visible phase landing,
   follow `sase_flags.md` for temporary beta scaffolding and remove that scaffold before
   the complete feature lands. Never deploy generated skills from an unlanded tree.

## Phase 1: Shared configuration names and compatibility contract

Primary owners: `sase-core/crates/sase_core/src/config/axe.rs`, the surrounding config
composition/validation modules, `axe_chop/config.rs`, and `sase_core_py` bindings;
`sase/src/sase/feature_flags/` for the rollout declaration and thin host wiring.

- Implement a single structural normalization/public-projection mapping in Rust. Keep
  `axe` and `ace` roots intact. Cover every field in the table and any other actual
  public field whose name contains lumberjack/chop. Do not substring-replace arbitrary
  keys, `vars`, environment payloads, script values, descriptions, or entity names.
- Normalize each layer before composition, including bundled defaults, plugin layers,
  user base, machine overlays, and project configuration. Preserve order, keyed merge,
  legacy list semantics, sparse overrides, disable/delete behavior, target expansion,
  and exact segment keys such as `checks.release`.
- Across layers, existing precedence wins. Within one layer, disjoint synonymous
  contributions merge and identical duplicate leaves may collapse. Conflicting leaves
  fail with both authored paths and the source layer. Unequal synonym lists for the same
  collection fail instead of silently concatenating or choosing one spelling.
- Preserve both original source segments and normalized identity. Public projection must
  not destroy the path needed to edit/reset/delete the selected source entry. New
  entries use canonical paths; edits to an existing old entry target its real subtree
  and never create a second alias subtree.
- Keep existing runtime dictionaries and wire models usable. Add/version binding APIs as
  needed; Python may marshal, perform file I/O, and apply returned edit plans but must
  not reproduce normalization or conflict rules. Expose projection support to general
  config views as well as the AXE-specific editor.

Acceptance: Rust tests for old/new/mixed-layer parity, alias collisions, list/map forms,
dotted names, expanded targets, source provenance, and add/set/reset/delete round trips;
binding tests for the new API and both flag states. Run the Rust repository's required
`just check` with Python >= 3.12, including PyO3. A core-only cargo test is
insufficient.

## Phase 2: Public job scripts and SDK

Primary owners: `sase/pyproject.toml`, `src/sase/chops/{__init__,sdk,report}.py`, a new
thin `src/sase/jobs/` public facade, and
`src/sase/axe/{chop_script_runner,chop_inventory,chop_script_context,chop_env}.py` plus
the actual env assembly/scrubbing callers.

- Add `sase_job_*` console entrypoints pointing to the same implementation functions as
  each built-in `sase_chop_*` entrypoint. Keep private script-module filenames.
- Expose job-named equivalents of the existing public exports: `JobArguments`,
  `JobInvocation`, `JobLogger`, `JobReport`, `JobResultBuilder`, `JobResultStatus`,
  `JobSummary`, schema constants, and invocation/result/report helpers. Re-export or
  adapt existing objects; keep result validation and scheduling in their current owner.
- Provide canonical `SASE_JOB_*` equivalents of documented invocation controls,
  including `SASE_JOB_ROUTINE` for `SASE_CHOP_LUMBERJACK`. Supply equal values to both
  env families. Accept either input family; conflicting supplied values produce an
  actionable error instead of accidentally changing ownership or result paths. Preserve
  both families for agents deliberately linked to this job run, including split-agent
  linkage, and scrub both families from unrelated child launches. Keep the durable
  `agent_chops.json` linkage format and its identity semantics.
- Expose `routine_name` and `verbose_routine_diagnostics` in job context access and
  additive context JSON fields; retain legacy readers and validate conflicting aliases.
  Do not rename required fields under an unchanged strict wire schema. Structured result
  statuses, proposals, guards, and deduplication keep their current semantics.
- Exact configured executable names still resolve exactly. Discover both built-in
  prefixes in PATH and configured directories, show the canonical alias once when two
  entries demonstrably identify the same packaged entrypoint, and retain old-only plugin
  discovery. Never infer executable equivalence from a prefix alone.
- Update public script help and SDK errors. Scripts currently use a runner-internal
  `--context` argument; this rename does not redesign that invocation protocol.

Acceptance: installed console-entrypoint help, new/old SDK result parity, old-only
script execution, canonical discovery without duplicate built-ins, conflicting env and
context aliases, env scrubbing, target expansion, timeout/history behavior, and
identical launch/dedupe outcomes. Use isolated fixture scripts and mocked launches.

## Phase 3: Commands, structured output, and reference presentation

Primary owners: `src/sase/main/{parser_ace,axe_handler,parser}.py`,
`src/sase/axe/{cli,chop_render,chop_doctor,status_render,restart_render}.py`, diagnostic
registration, completion metadata, and callers that construct routine subprocess argv.
Shared semantics and user-visible diagnostics also come from Rust `axe_status`,
`axe_chop`, and artifact-link modules; change those owners and bindings as needed.

- Register sorted public AXE children
  `{ensure,job,maintenance,restart,routine,start,status,stop}`. Preserve the internal
  `bgcmd-launch` command and the existing bare-root usage/exit behavior.
- Rename group help, positional display to JOB/ROUTINE, run options, usage errors,
  ambiguity messages, default-list notices, doctor remediation commands, and all
  routine/job run output. Keep private destinations such as `chop_name` if convenient.
  Use the central `_default_list_subcommands()` convention; do not duplicate it.
- Update orchestrator spawn/restart argv and other CLI emitters to the canonical nested
  command. Retain the same AXE process, lock, watchdog, and service identities. Audit
  command recognition in process discovery so existing and new worker argv both identify
  the same routine and cannot produce duplicate daemons.
- Add explicit public JSON projections: `chops` to `jobs`, `lumberjacks` to `routines`,
  `lumberjack`/`lumberjack_name` to `routine`/`routine_name`, `configured_chops` to
  `configured_jobs`, and other contract-owned fields using these terms. Inventory the
  actual list/doctor/status keys. Keep arbitrary job payloads, IDs, and recorded paths
  verbatim. Publish schema_version 2 for renamed envelopes, retain the v1 internal
  snapshot, and implement the compatibility selection above. Keep stdout valid JSON.
- Rename public diagnostic IDs/filters such as `axe.chops` to `axe.jobs`, accepting
  legacy selectors at the boundary. Do not needlessly rename internal diagnostic codes
  that are only embedded in historical/internal records.
- Render and accept `job:<routine>/<job>` wherever existing virtual `chop:` references
  are supported: link CLI operations, projected link tables, clipboard references, TUI
  link rail/modal/navigation, and applicable completion. Normalize before graph lookup,
  insertion, inverse projection, deduplication, and read tracking so there is still one
  stored artifact. Preserve `chop:` input and existing projection rule IDs. This virtual
  type need not become a new file-backed artifact provider.
- Keep actual existing agent names, including `.chop.` segments, intact. Labels around
  those names and projected reference explanations use job terminology.

Acceptance: parser and dispatch behavior for every renamed child, hidden aliases,
short/long options, lazy/full help, completion, default-list notices, invalid input,
subprocess argv, old/new process recognition, meaningful human diagnostics, v1/v2 JSON
in both flag states, and reference round trips without duplicate graph edges. Never
start/stop the user's live AXE daemon during tests.

## Phase 4: Canonical configuration and AXE presentation

Primary owners: `src/sase/config/`,
`src/sase/axe/{config,config_backend,_config_layers}.py`, `src/sase/default_config.yml`,
`src/sase/config/sase.schema.json`, and `src/sase/ace/tui/`.

- Wire phase 1 into config load, effective show/catalog/completion, validation, doctor,
  and edit previews. Internal consumers of `load_merged_config()` retain their model.
  Raw layer displays show the real authored paths and may add the canonical equivalent.
  Exercise both generic config editing and AXE entry editing.
- Update defaults and the public schema together, including canonical job script values
  now provided by phase 2. Keep old input paths accepted. Preserve every value, cadence,
  trigger, timeout, keybinding chord, routine/job identity, and enabled state. Rename
  only public keymap keys actually containing chop/lumberjack, if present, and map them
  to unchanged action methods. Keep other `ace.*` and `axe.*` identifiers.
- Update sidebar/dashboard headers and counters, onboarding, descriptions, add kind
  pickers, parent selectors, property editor fields/validation, run history, overrun
  messages, toast/notification text, command palette, footer/help, clipboard/export,
  log-source labels, and link panels. Start with `axe_add_modals.py`,
  `axe_entry_editor_*`, `axe_onboarding.py`, `_axe_dashboard_*`, `bgcmd_list.py`,
  `actions/axe*`, and `modals/help_modal/axe_bindings.py`.
- Some pickers display their stored kind directly. Add a display label rather than
  changing the value consumed by action dispatch. Do not run word substitution over
  rendered user descriptions, raw YAML, recorded output, or file paths.
- Expose the built-in automation tribe as `job`/`@job` in configuration, CLI/TUI lists,
  filtering, completion, and assignment inputs while mapping historical built-in `chop`
  assignments to the same identity. Put shared resolution in Rust. Preserve its glyph,
  color, collapse default, and grouping behavior. Detect an independently configured
  `job` tribe collision and report it; never silently merge user tribes.
- Read `tui_perf.md` and the subtree `AGENTS.md`. Keep the existing cached/worker-based
  rendering and selection paths, help-box widths, and conditional footer conventions.

Acceptance: mixed legacy/canonical config view and edit tests, schema/default parity,
existing YAML source preservation, routine/job add/edit/run tests, selection stability,
tribe filtering and collision tests, and affected narrow/wide AXE visual snapshots.
Inspect actual/expected/diff images before updating intentional goldens.

## Phase 5: Integrations and maintained operator configuration

- In `sase-telegram`, add `sase_job_tg_inbound` and `sase_job_tg_outbound` to
  `pyproject.toml`, keep existing executable compatibility, and update package summary,
  README, `docs/{inbound,outbound,architecture}.md`, script help, public messages,
  executable tests, and `.github/workflows/publish.yml` smoke checks. Retain internal
  module names, lock files, receiver ownership, and notification semantics.
- Update public Telegram prerequisite checks in SASE to recognize both entrypoint
  spellings and recommend the canonical one. Verify an installed old-only plugin and a
  new plugin, including discovery without duplicate rows.
- In the opened `chezmoi` checkout, update structural AXE keys and canonical executable
  values in `home/dot_config/sase/{sase,sase_athena}.yml`, and current prose in
  `actstat-ci-watch.yml`. Preserve unrelated configuration, secret references, and
  user-chosen names. Validate copied fixtures against both old and new representations.
- Regenerate bash, fish, and zsh completion sources from the canonical parser into the
  corresponding tracked chezmoi files using the established generation flow. Do not
  hand-edit generated tables. Verify no public old group/option is suggested and AXE
  remains present. Keep any concurrent `sase tui` completion changes.
- Audit the other inspected integration repositories again; no speculative refactor is
  authorized where there is no matching public surface. Leave released changelogs and
  historical plans untouched.
- Run each changed repository's prescribed checks. Coordinate packaged entrypoint and
  Rust-binding availability before host deployment; do not activate new operator
  scripts/configuration against an old installed runtime during phase work.

Acceptance: local integration tests with Telegram/network calls mocked, canonical and
legacy installed-entrypoint smoke tests, combined operator-config parity, and shell
completion generation/syntax checks. Do not send Telegram messages or run the live
automation jobs as a rename smoke test.

## Phase 6: Current documentation, glossary, and visual examples

- Update current guides and cross-links: `docs/{axe,cli,configuration,plugins,ace}.md`
  are the main clusters; also audit architecture, artifact links, notifications, beads,
  SDD, project/workspace, telemetry, LLM, xprompt, performance, development, and Rust
  backend guides, README/site landing content, current demos, and published help text.
  Keep AXE/axe and the already-selected TUI vocabulary.
- Replace runnable examples with canonical groups, options, config keys, entrypoints,
  SDK imports, and environment names. Repair heading anchors such as
  `#default-lumberjacks` and incoming links together. Include a concise old-to-new
  compatibility table explaining the JSON version transition and actual unchanged state
  paths; do not obscure literal internal import/path names in technical docs.
- Under `/sase_memory_write`, update the existing glossary strand files
  `sase/memory/glossary/lumberjack.md` and `sase/memory/glossary/chop.md`: titles and
  canonical keywords become Routine and Job, and their definitions use the new words.
  Preserve filename identity, retain the old words as lookup aliases, and author any
  required links. This is part of the user's request to update the public vocabulary.
  Run `sase memory init` to regenerate the glossary roster, README, and instruction
  shims; never hand-edit those generated files. Do not rewrite immutable decisions.
- Re-audit generated skill/xprompt sources for public references introduced since this
  plan. Modify a matching source template, not installed SKILL.md copies. No matching
  bundled skill prose was found during planning. If sources change, use
  `sase skill init --diff` or `--dry-run`; publish only from the clean landed revision
  as required by `generated_skills.md`.
- Inspect images actually embedded in current docs, especially
  `docs/images/sase-component-communication.png` and
  `docs/images/rust-backend-boundary-infographic.png`, for visible old terminology.
  Update affected current visuals through their source/generation workflow and the
  applicable image skill. Update current demo fixtures/recordings where the old terms
  are visibly taught. Preserve historical blog posts, retired image-generation records,
  archived plans/research/prompts, and old logs. Avoid media work on assets with no
  visible matching terminology.

Acceptance: canonical examples parse or execute against isolated fixtures, links and
anchors resolve, glossary Job/Routine and legacy alias reads work, generated output is
consistent, and current published visuals have been inspected. Read required memory
through `sase memory read`, not direct file reads.

## Phase 7: Combined acceptance and rollout verification

1. Audit residual `lumberjack`, `lumberjacks`, `chop`, `chops`, `sase_chop_`, and
   `SASE_CHOP_` matches across changed repos. Classify each retained class as private
   implementation, immutable history, actual raw identity/path, or explicitly tested
   compatibility. Review strings in private modules too; directory location alone is not
   an exemption for displayed text. Check for stale embedded commands and indirect
   kind-to-label leaks. Do not add a blanket zero-match test or a giant allowlist.
2. Run one end-to-end isolated fixture through legacy input config and canonical input
   config: list routines/jobs, doctor, edit one exact override, manually run a script,
   inspect its history and public JSON, and navigate its job/agent links in the TUI.
   Assert equivalent scheduling/launch outcomes, shared identities, no duplicated
   routines/jobs/links, and unchanged AXE tab/root behavior. Test both rollout states.
3. Include an old-only plugin, equal and conflicting aliases, duplicate job names across
   routines, generated target names, actual legacy state paths, and a custom `job` tribe
   collision. Confirm no secrets appear in new diagnostics.
4. Run focused existing parser/default-list/completion, config composition/edit, script
   SDK/inventory/env, doctor/status, reference-link, tribe, and TUI suites. Extend tests
   for changed contracts and meaningful regressions, not one assertion per mechanically
   replaced string. Inspect and refresh affected visual snapshots.
5. Run `just check` in SASE after file changes, and the complete required Rust check
   including binding tests plus each changed integration's checks. Before landing the
   combined epic, run `just check-full` through `/sase_monitor` with TESTING/TESTED
   status as prescribed by `lint_and_test.md`. Use the monitor for other long checks
   too; do not promise to resume without a mechanical handoff.
6. Validate generated schema/defaults, completion artifacts, memory output, and current
   docs/media. If Rust binding changes require a dependency floor, use the existing
   core-release/floor workflow and host-owned finalizers; do not manually bump Rust
   release versions or claim that a locally rebuilt binding is already published.
7. Report the compatibility contract, residual categories, checks, and any actual
   rollout prerequisite. No implementation agent manually creates commits, branches, or
   PRs; submit the required final declaration for all changed repositories and let
   host-owned finalizers land the reviewed result. Preserve the canceled epic's state.

Completion means users encounter routine/job terminology throughout maintained default
interfaces and can use canonical commands/config/scripts, while existing accepted inputs
and stored automation history retain their behavior and identity. The AXE tab and
`sase axe` remain exactly the requested product entrypoints.
