---
tier: epic
title: Complete the non-TUI macro syntax cutover
goal: Make macro spellings canonical across non-TUI SASE surfaces while preserving
  flag-gated authored aliases and unconditional durable readers.
parent_bead: sase-1eq.4
phases:
- id: compatibility
  title: Sunset flag and shared compatibility contracts
  size: medium
  depends_on: []
  description: 'compatibility: create legacy_xprompt_syntax through sase flag new,
    add the shared Rust config normalization contract and thin Python compatibility
    facade, and verify both flag states without flipping existing Rust output contracts.'
- id: config-frontmatter
  title: Canonical config and local macro frontmatter
  size: medium
  depends_on:
  - compatibility
  description: 'config-frontmatter: normalize each authored layer before merging,
    migrate schemas and consumers to macro keys, update local helper parsing and canonical
    writers, and test collisions and flag-off errors.'
- id: discovery
  title: Macro directory, plugin, and LSP discovery
  size: medium
  depends_on:
  - config-frontmatter
  description: 'discovery: use the content-layout macro source order and write paths,
    share deduplicated plugin discovery, gate legacy directories and public environment
    aliases, and propagate the policy to Rust and LSP.'
- id: cli-doctor
  title: Macro CLI, completion, and retirement diagnostics
  size: medium
  depends_on:
  - discovery
  description: 'cli-doctor: publish canonical macro commands and JSON, hide flag-gated
    old aliases, repair schema path targets, refresh shell completion, and report
    every retired authored surface in either flag state.'
- id: strings-guard
  title: Remaining strings, skill sources, and terminology guard
  size: medium
  depends_on:
  - cli-doctor
  description: 'strings-guard: finish non-TUI strings and directive writers, update
    maintained skill sources and smoke/demo scripts, tighten the guard with classified
    exceptions, and record integration evidence for the assigned parent phase without
    closing any ancestor.'
proposed_by: bbugyi200.athena.sase-1eq.4
create_time: 2026-10-03 05:59:52
status: done
bead_id: sase-1eq.4.1
---

- **PROMPT:** [prompts/202610/macro_syntax_cutover.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/macro_syntax_cutover.md)
- **PARENT:** [202610/xprompts_to_macros.md](https://github.com/sase-org/sase--plans/blob/main/202610/xprompts_to_macros.md)
- **BEAD:** [sase-1eq.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eq/sase-1eq.4.1.md)

# Complete the non-TUI macro syntax cutover

## Scope and evidence

This implements the entire `sase-syntax` scope of bead `sase-1eq.4`, defined in
`plan:202610/xprompts_to_macros.md`. Read that artifact through `sase artifact read` and
the assigned bead through `sase bead read` before implementing. Its Shared Policy binds
this plan. The prerequisite phases `sase-1eq.2` and `sase-1eq.3` are closed.

The renamed Python package, built-in directories, and import shim already exist:
`sase.macro`, `src/sase/macros/`, `src/sase/default_macros/`, and
`src/sase/xprompt/__init__.py`. Do not redo the module rename or remove the shim. The
current guard covers identifiers/imports/paths, not ordinary string literals.
Exploration found retired strings in about 416 non-TUI source/test files. This is an
epic because config semantics, filesystem/plugin discovery, and CLI publication need
separate, testable changes. Each phase is bounded medium implementation work; explicit
dependencies serialize changes to the shared compatibility facade and schemas.

Concrete gaps found in the current tree:

- `main/parser_macro.py`, parser registration, and dispatch still expose `xprompt`;
  `completion/kinds.py` still gives `ValueKind.MACRO` the value `xprompt`.
- `config/loading.py` performs runtime merging while Rust `config` owns inventory,
  validation, and edit plans. Neither has a complete macro config alias projection.
- `config/macro_sources.py`, Markdown/YAML helper loaders, and surgical config writers
  still read/write `xprompts`. `agent/multi_prompt.py` pops only that key.
- Several fallback readers still hard-code old directories. The Rust layout already
  provides `macros`, `macro_sources`, ordered entries, and legacy roles.
- Plugin entry points and resource probes still use `sase_xprompts`/`xprompts` at
  multiple independent sites, including skills and catalog export.
- `integrations/macro_lsp.py` still reads only `SASE_XPROMPT_LSP_CMD`, and its binary
  fallback is unconditional. `core/macro_skill_definition_facade.py` uses canonical
  option names but supplies old package resource paths.
- `%macros_enabled` is already readable, including the permanent old alias; writers
  still emit `%xprompts_enabled`.
- `macros-collection-schema` cannot simply be a rename: the current collection target
  resolves a missing `xprompts.schema.json` file.

Primary implementation is in sase. Open the linked `sase-core` checkout with
`sase repo open sase-core -r "Implement shared macro syntax compatibility"` before
reading or changing it, and follow its `AGENTS.md`. The Rust work below is additive
support required by the project backend boundary, not the later `core-flip` phase. No
plugin repository, chezmoi, durable archive, memory file, site, or TUI rendering changes
belong to this plan. TUI changes are limited to path literals/adapters and the minimal
config-key lookup follow-through described below. Read `tui.md` and the TUI's local
`AGENTS.md` before touching those files. The original epic's TUI phase owns keymap
action migration, widget terminology, view ids, highlighting presentation, and PNG
goldens.

## Shared implementation rules

Use `/sase_memory_read` to read `sase_flags.md`, `cli_rules.md`, `generated_skills.md`,
`lint_and_test.md`, and `xprompts.md` (or its renamed `macros.md` successor) when
working in their domains. Read `symvision.md` before repairing its failures.

Canonical output and authored examples use macro vocabulary. `#name`, `#!name`, and
their argument grammar retain their behavior. Whole authored prompts use `raw_prompt`
and `submitted_prompt`, rather than misleading macro names. Jinja syntax, its local
scope identifiers, Rust language macros, query status shorthands, and LSP standard
semantic token names keep their distinct meanings.

All temporary Python alias spellings and policy live in
`src/sase/legacy_xprompt_syntax.py`, modeled on
`src/sase/agent/legacy_sase_shell_syntax.py`. Keep it a host adapter for shared Rust
semantics, with CLI/environment/plugin integration as appropriate. Errors say
`<old> is retired; use <new>`. No silent fallback may swallow this policy error. Detect
duplicate keys by presence, including null, empty mapping, false, and empty string.
Supplying both spellings in the same authored mapping is an error in both flag states.
Normalize each layer independently: canonical bundled defaults plus an old user override
are valid while the flag is enabled and must follow existing layer precedence, rather
than accidentally become a both-present error after merging.

Distribution registration under both plugin groups and coexistence of definition
directories are deliberate exceptions to duplicate authored-key rejection. New plugin
registration/resources win and each distribution loads once; macro definitions keep the
core's first-wins source ordering.

`legacy_xprompt_names.py` owns permanent durable readers. Do not flag, rewrite, or
delete those readers, the old `%xprompts_enabled` parser, or realistic legacy fixtures.
Keep existing expand/contract support for the old core until `sase-1eq.10` flips it. No
released version, CHANGELOG, core emitted key, or wire schema is mechanically renamed in
this phase. A new optional request field/binding is additive; preserve existing request
defaults and mirror the owning domain's schema conventions.

Keep mechanical replacements in a repeatable scratch script, apply case-aware article
and identifier rules from the parent plan, and skip legacy homes, `LEGACY_*`/`legacy_*`
names, the shim, legacy fixture families, and history. Inspect every legacy diff. Avoid
replacing arbitrary user-authored bodies or template variables named `macro`.

## Compatibility

### Flag creation

First inspect the live registry and flag state to avoid duplicate creation after a
retry. If absent, use only `sase flag new legacy_xprompt_syntax -k sunset` with the
three authored prose arguments below, paste the printed registry scaffold into
`src/sase/feature_flags/registry.py`, then run `tools/sync_feature_flags_schema` in its
rewrite mode and check its result. Keep the default on, derived from `sunset`. This
dedicated flag operation is required implementation, not a discovered task; never
manually create a task bead or hand-invent a registry/bead id.

Use the parent design's full prose, including every surface (prose arguments accept
`@<scratch_file>` when these sentences are too long for an inline command):

- **When enabled:** SASE silently accepts the retired xprompt spellings as aliases of
  their macro replacements: `sase xprompt`, `sase path xprompts-*`, `xprompt-catalog`
  helper bridges; `xprompts`, `xprompt_aliases`, `auto_xprompt_menu`,
  `xprompt_placeholder_args`, `mentors[].xprompt`, `focus_xprompt`,
  `clear_xprompt_focus`, and `start_last_vcs_xprompt_in_editor`; `xprompts:`
  frontmatter; `sase/xprompts/`, `.xprompts/`, `xprompts/`, `~/sase/xprompts/`,
  `~/.xprompts/`, `~/xprompts/`, and `~/.config/sase/xprompts/<project>/`;
  `sase_xprompts` and packaged `xprompts/`; `SASE_XPROMPT_LSP_CMD`,
  `SASE_DISABLE_PLUGIN_XPROMPTS`, and `sase-xprompt-lsp` on PATH.
- **When disabled:** SASE rejects those spellings with an error naming the macro
  replacement, skips xprompt-named definition and plugin directories that doctor
  reports, and still loads pre-rename agent artifacts, proc rows, MRU/save-state files,
  skills manifests, and `%xprompts_enabled` regions.
- **Remove when:** no maintained prompt, macro, skill, config, script, or plugin in
  sase-org, bugyi-chops, or chezmoi uses retired xprompt spellings; every maintained
  plugin ships `macros/` under `sase_macros` with a sase floor that reads it; and a
  release with macro syntax has shipped.

The TUI phase implements the keymap aliases described by that removal contract. Provide
reusable alias metadata for its doctor reporting now; do not switch keymaps or TUI
consumers prematurely.

### Shared config semantics

Extend the owning `sase_core::config` domain with a small deterministic normalization
API for authored macro config. Its input is an already-decoded mapping plus an explicit
`accept_legacy_xprompt_names` boolean; its output contains the canonical mapping and
source-qualified retirement/collision diagnostics. Cover top-level `xprompts`/`macros`,
`xprompt_aliases`/`macro_aliases`,
`ace.prompt_completion.auto_xprompt_menu`/`auto_macro_menu`,
`ace.prompt_inputs.xprompt_placeholder_args`/`macro_placeholder_args`, and
`mentor_profiles[].mentors[].xprompt`/`macro`. Treat entry names and template bodies as
data, not arbitrary recursive keys to rename. Share local-definition key selection with
the existing macro catalog policy where practical; preserve nested `_helper` semantics.
Keep legacy Rust constants explicitly commented as legacy xprompt spelling.

Use this normalization in Rust config inventory/effective views and edit planning, with
source identity retained. A config edit writes canonical key paths and must
replace/remove the corresponding old key in its authored map rather than leave a
both-present file. Canonical editing of a legacy layer must produce a valid candidate,
preserve unrelated values/comments through Python's YAML writer, and match runtime
effective config. Avoid broad refactors of AXE config merge or unrelated aliases.

Expose and register any new binding in the existing `sase_core_py` config domain; add
Rust and binding round-trip tests. Python compatibility wrappers call this binding
instead of duplicating backend normalization. Ship the minimum Python adapter tests and
update binding checks/health probes for any new required binding.

Feature-flag resolution reads raw layers via `load_config_layers()`. Preserve that raw
path so normalization calling `current_flags()` cannot recursively bootstrap the flag
snapshot. Explicitly test loading before snapshot installation. Include the policy in
caches of normalized config so a changed/overridden flag cannot reuse a previously
accepted normalized value.

If core changes, verify and declare both checkouts together. The configured linked repo
already has `revision_pin: sase-core-revision.txt`; host finalization must commit core
and move that pin past the new binding commit before the Python caller lands. Use the
documented pin workflow and rebuild the extension for verification; never invent a
future SHA or widen the published dependency window. Leave later core-flip output/schema
changes for their assigned phase.

**Exit:** flag registry/schema integrity; presence-based alias and collision tests in
both states; raw flag bootstrap and Rust binding round trips; each changed repository's
wrapped check. Any temporary unused export is keyed to a still-open child epic or its
actual later consumer, not this phase when it closes.

## Config and frontmatter

Apply the shared normalization before runtime config merge, plugin defaults, user base,
machine overlays, and local config. Update `config/loading.py`,
`config/macro_sources.py`, layer inventory and edit adapters, and all non-TUI consumers.
Raw layer metadata remains available to doctor, while effective values use canonical
keys. Policy errors must propagate as actionable errors, including from loader paths
that currently catch every exception and log at debug level.

Rename default config keys and `config/sase.schema.json` fields to `macros`,
`macro_aliases`, `auto_macro_menu`, `macro_placeholder_args`, and mentor `macro`. Rename
`xpromptInputDefinitions` to `macroInputDefinitions` and update every `$ref`. The
canonical schema/help/completion surface shows new keys. Validate normalized old input
through the policy adapter rather than silently advertising two supported keys. Test the
editor/runtime/schema contract together, including same-layer collision,
legacy-to-canonical edit, and the existing concatenate/replace layer strategies.

Update local helper selection in `macro/loader_sources.py`, `loader_parsing.py`,
`workflow_loader_definition.py`, `prompt_frontmatter.py`, `agent/multi_prompt.py`, and
any inline/nested config-definition reader. Markdown frontmatter, workflow local
helpers, and user-prompt helpers must all accept `macros`, gate `xprompts`, and reject
both. Reuse the existing Rust catalog behavior when it owns the operation; keep Python
glue's behavior in parity. A retired-name failure cannot be reduced to empty helpers or
literal pass-through merely because a caller has a broad catch.

Update `src/sase/macros/workflow.schema.json`, tracked built-ins, and project macro
frontmatter. Discover the actual files instead of trusting the parent plan's old count
or old resource paths. Writers in `macro/config_yaml.py`, `save.py`, `save_index.py`,
`prompt_frontmatter.py`, and source-location display emit canonical maps/provenance and
preserve existing surgical YAML/comment handling. Legacy edits must not leave two
section keys behind. Update persisted save-state adapters through the permanent reader,
not the sunset gate.

The schema/default keymap actions remain owned by `sase-1eq.5`; distinguish these
precisely from the prompt settings above. Add narrow interim guard exceptions only where
their owner and removal condition are explicit.

Follow the two settings lookup literals in `ace/tui/widgets/prompt_completion.py` and
`_local_xprompt_conversion.py` to the canonical keys so a user's false value still works
after normalization. Keep their current Python field/function names for the TUI phase.
This is necessary config glue, not a widget rename. Similarly retain any internal
compatibility projection consumed by the current TUI (for example the save-state
adapter's old lookup key) through a named legacy constant until the TUI phase follows
it; canonical persisted writers already use the new key. A default-value fallback must
not hide a dropped setting.

**Exit:** parameterized config/frontmatter tests for enabled, disabled, duplicate,
empty/null, nested helpers, malformed content, independent layer precedence, config
edits, and canonical serialization; update existing schema/mentor/save tests; wrapped
check in each changed checkout.

## Discovery

Use `content_layout.resolve_macro_file_sources()` and the Rust `macro_sources` contract
for definition reading, including project namespace discovery and existing order.
Writers use `resolve_project_layout(...).macros.write_path`, home `macros.write_path`,
and project-specific home macro destinations. Test the no-project/cwd fallback,
subdirectory launches, project namespace, and `use_chezmoi` remapping.

Gate only retired macro definition locations/resources. Do not blindly filter every
`role == legacy` record: a project's legacy `sase.yml` location is a separate content
layout compatibility surface and can contain valid canonical `macros`. With the flag
off, old definition directories are invisible to expansion, workflow loading, shell
completion, PDF catalogs, save choices, and Rust-backed skills/catalog operations. Keep
normal first-wins shadowing when both directory families are present.

Remove duplicated definition-directory lists in `macro/loader_sources.py`,
`workflow_loader_sources.py`, completion's `catalog_prompts.py`/`catalog_snippets.py`,
and their follow-through consumers. In `ace/tui/prompt_catalog.py`,
`xprompt_browser_helpers.py`, `add_xprompt_modal.py`, and `xprompt_location_modal.py`,
change only the path literal or shared layout adapter needed for this contract. Do not
rename those TUI modules or change rendered copy here.

`skill_destination_for_macro_dir` maps canonical and accepted old definition roots to
the sibling `skills/` directory. Update skill placement migration hints with canonical
destinations while preserving its current restrictions. Move tracked
`sase/xprompts/reads.md` and `sync.md` to `sase/macros/` with `git mv`; their
frontmatter is already canonical from the preceding phase. Do not migrate private home
content.

### Plugins

Consolidate macro plugin/resource discovery around `main/plugin_discovery.py` so
`macro/loader_sources.py`, `workflow_loader_sources.py`, `_catalog_sources.py`,
`macro_sources.py`, `loader_skills.py`, `skill_locations.py`, LSP transport, and
`core/macro_skill_definition_facade.py` agree. Read `sase_macros` first, then the
retired group only when enabled. Deduplicate by distribution/resource identity rather
than merely entry-point name; a dual registration loads once and canonical wins.

For each accepted plugin module, probe `macros/` first and packaged `xprompts/` only
when enabled, independently of the entry-point spelling. A newly dual-registered plugin
may still ship only `xprompts/` because the release floor cannot yet require new sase.
Support this transition when enabled; skip and report it when disabled. Keep plugin
skills/default configs working through their appropriate groups and normalization. Never
move a plugin's packaged directory or raise its floor in this phase.

Use `SASE_DISABLE_PLUGIN_MACROS`; accept the retired disable variable only through the
compatibility facade. Supplying both public env names is an actionable error, including
empty values, and an old env name with the flag off names its replacement. Preserve
`SASE_DISABLE_PLUGINS` and unrelated group controls.

### LSP and Rust catalog requests

Use `SASE_MACRO_LSP_CMD` first with the public legacy name handled in the compatibility
home. Move legacy binary spellings/fallback enumeration there too. Prefer
`sase-macro-lsp` in the current venv, PATH, and built targets; accept `sase-xprompt-lsp`
only when the flag allows it. Explicit overrides keep current quoting/path behavior.
Pre-flip Cargo crate detection remains available as build tooling, not an advertised
legacy user command. Do not accidentally choose a newer legacy binary over a usable
canonical one.

Pass `accept_legacy_xprompt_names` explicitly at every existing catalog-loading binding,
including skill definition resolution and editor snippet catalogs. Supply actual
`macros`/`default_macros` package resources to their option keys. Propagate the policy
to the LSP as its existing initialization option. Where `sase lsp` execs the server and
does not own the client's initialize message, set the server's existing
`SASE_ACCEPT_LEGACY_XPROMPT_NAMES` policy transport consistently; do not add a JSON-RPC
proxy solely to inject an option. Verify both initialize-option and exec-wrapper paths,
modeled on the existing typed-launch-units transport.

Retain the expand-phase dual-write of internal `SASE_MACRO_*` transport variables and
the agent-local/swarm env fallbacks that `sase-1eq.10` owns. Separate this private
inter-process handshake from the two public env aliases above. Centralize inevitable
legacy transport constants in an appropriate legacy home, with exact ownership/tests.

**Exit:** paired old/new temporary directory and fake-distribution fixtures across
expansion, workflows, completion, export, skills, and Rust catalogs; both flag states;
new-first dedup and legacy-only plugin tests; public env collision tests; LSP command,
policy, resource-path, and pre-flip transport tests; wrapped check.

## CLI and doctor

Publish `sase macro {catalog,expand,explain,graph,list,show}` with existing options and
`dest="macro_subcommand"`. Update the registry, full registrar table, narrowed parser
hint, root help, default-to-list delegation, main dispatch, and completion contracts.
Normalize the legacy command at the correct root-command position in the compatibility
home so it can remain hidden yet produce an explicit retirement error when disabled.
Never rewrite an `xprompt` word inside arbitrary prompt arguments. Both full and
narrowed parser construction must work; legacy invocation help uses canonical usage.

Publish `sase path macros-dir`, `macros-schema`, and `macros-collection-schema` with
hidden gated aliases. Verify all printed paths exist. Repair the missing collection
schema target by adding a canonical collection schema appropriate to the existing
definition mapping and package it, or remove the unusable target if no real collection
contract exists; record the evidence and resulting supported targets. No target should
continue returning a nonexistent file. Update option choices without advertising legacy
values.

Canonical hidden editor/mobile helper operation is `macro-catalog`; accept the old
operation through the flag because the pre-flip gateway still calls it. Update parser
choices and handler routing together. Keep gateway/mobile API version compatibility and
the later core output flip owned by their phase; this is a helper selector rename.

Change CLI `type`/`kind` values and definition keys to `macro`, including list/show,
catalog metadata, explain, graph, source provenance, and save output where applicable.
Map any pre-flip core output at the thin boundary; do not make a CLI assert a shape the
pinned core cannot emit. Validate no new emitted retired names in canonical CLI JSON.
`ValueKind.MACRO` becomes `"macro"`; update command-path annotations, dynamic
candidates, serializers, and generated shell completion. Regenerate
`tests/completion/snapshots/cli_spec.json` using `just sync-completion-spec` and inspect
the diff. Legacy commands/targets do not appear in help or completion in either state.
Keep public long options paired with short aliases and inventory order alphabetical.

Update `sase lsp`, `sase prompt save`, `sase project current|set`, `sase run`, root
help, and snippet help. Examples and paths use canonical macro spellings and MRU names.

Rename doctor checks to `config.model_macros`, `config.macro_definitions`,
`config.macro_directives`, and `tools.macro_lsp`; update registration, results, hints,
filters, JSON, and existing tests together. Add `config.retired_xprompt_names` (this
intentional legacy id is required by the parent design).

The new check inspects raw authored inputs independently of the loader's acceptance
policy, so it works even when ordinary loading would reject/skip them. Report each
location/key/group and its macro replacement in both states: project/home definition
directories, base/overlay/local/plugin default config keys, mentor fields, Markdown and
workflow frontmatter/nested helpers, three keymap actions, ambient public env variables,
plugin entry-point groups and package directories. Reuse the compatibility metadata and
core diagnostics; keep filesystem/entry-point inspection in Python. Retain raw metadata
even if a load failed; avoid expanding templates or exposing env values. Keep the
disabled-state check accessible on a tree containing legacy input. Durable files and
permanent `%xprompts_enabled` regions are not retirement findings.

**Exit:** full/narrow parser and root/default delegation regressions; hidden alias and
canonical help/completion checks in both states; verified packaged schema targets;
canonical CLI JSON; both helper selectors; stable doctor ids and complete retirement
fixtures, including disabled-state detection; wrapped check.

## Strings and guard

Finish user-visible strings and non-durable literal contracts throughout non-TUI `src/`
and `tests/`, including errors, warnings, logs, PDF/catalog/template labels,
graph/explain/show headings, config descriptions, and script output. Fix grammar after
case-aware replacements. Keep Jinja's statement/scoping keywords and LSP semantic token
vocabulary unchanged; any old core-emitted highlight/context value is adapted through
named pre-flip compatibility constants until its owning phase flips it.

Directive writers emit `%macros_enabled` in `macro/_disabled_regions.py`,
`main/qa_prompt.py`, `monitor/followup_persistence.py`, `sdd/_write.py`, and the
`fork_by_chat.yml`/`make_mentor_changes.yml` built-ins. Verify every writer discovered
by the sweep, and prove old stored disabled regions still parse with the flag off.

Update maintained sources under `src/sase/macros/skills/`, especially `sase_run.md`,
`sase_project.md`, `sase_chats.md`, and `sase_agents_status.md`, with canonical command
names, directive regions, paths, headings, and `raw_prompt.md`. Preview through
`sase skill init --diff` or `--dry-run` if needed; never deploy from this unlanded tree
and never hand-edit generated provider skill files. Skill redeploy belongs to the
original docs/audit phases. Update `smoke/pypi/smoke_check.sh` and
`demos/scripts/seed_sase_ace_demo` and validate their syntax and meaningful smoke
contract coverage without starting persistent demos.

Widen `tests/test_macro_terminology.py` to inspect all tracked non-TUI source/test
content, including string literals, non-Python resources, comments/prose, and skill
sources. Avoid untracked caches and generated bytecode. Keep the contract marker. Remove
broad prior allowances such as every `_ENV` identifier and syntax-owned singletons now
that this phase implements them. Exceptions must name their reason, specific location or
fixture family, and owning phase when temporary:

- Permanent durable legacy home and its imports/legacy-input tests.
- Temporary syntax home, feature-flag definition/schema text, and its imports/tests.
- The doctor's required legacy id and retired-name table/fixtures.
- The explicitly marked external-plugin import shim and its contract tests.
- Exact TUI-owned keymap/schema references deferred to `sase-1eq.5` and exact internal
  pre-flip wire/transport references deferred to `sase-1eq.10`, only where adaptation
  cannot eliminate them. Record these exceptions on the assigned bead; they are not
  broad file or suffix exemptions and must be removed by those phases.

Do not classify a product straggler as legacy solely to pass the guard. Extend the
existing guard's checks so a newly introduced non-TUI string, resource, path, or
identifier fails, while required legacy fixtures still pass. Update affected
expectations and snapshot selectors intentionally, preserving assertions about behavior
and avoiding blind fixture rewrites of old-input evidence.

**Exit:** terminology guard, legacy-input regressions, both-state compatibility matrix,
smoke/script checks, skill preview as applicable, and wrapped repository checks.

## Verification and completion evidence

Every worker verifies its own changed repositories through `sase tool run check` and
appropriate focused tests. In sase, format/fix before the final wrapped check. A fresh
workspace may need `just install`; long installs/checks must use `/sase_monitor`
according to its skill, and any handoff command itself must be awaited until exit. Do
not run `just check-full`. Core uses its documented `just`/wrapped tool recipes, not raw
cargo. If core was changed, verify sase against that built core and ensure
`sase core health` and the host-owned pin obligation are satisfied.

Integration evidence must include canonical project/global/project-specific macro
definitions invoked as `#name`; CLI list/show/expand/explain output; old-only config,
helpers, directories, plugin resources and env aliases in both states; same-source
collisions; old durable artifacts/manifest/MRU/save-state/disabled-region readers with
legacy syntax disabled; and canonical-only writers. Inspect the final diff and run a
case-insensitive tracked sweep, classifying every surviving hit within phase scope. The
new doctor check must list legacy inputs in both states without a traceback.

If a failure reproduces identically on an unchanged clean base, record precise base SHA,
command, result, and any existing tracking bead in a `PROPOSED FOLLOW-UP:` note on the
assigned phase; that failure does not keep an otherwise completed phase open. Do not
reuse predecessor reports as proof: independently reproduce current failures. The
predecessor noted MRU provider-mismatch pruning and wrapped restart-recovery stderr
failures; investigate only if they recur. Never create discovered-work beads.

Before closing any assigned phase, run `sase bead epic-symbols <assigned-bead>`. Resolve
every listed symbol or re-key its Justfile exception to a still-open parent epic/later
consumer, then recheck and close only that assigned bead with
`sase bead close <assigned-bead> --note "<verification evidence>"`. Do not hand-set
status. Child phase workers do not close `sase-1eq.4`, their child plan, `sase-1eq`, or
any other ancestor. The child epic's land work supplies evidence for the host/authorized
parent-phase completion path, including an `epic-symbols sase-1eq.4` check and a note
that all five sections passed. An ancestor-closing instruction is never authorization
for a child worker.

Declare every changed repository through `/sase_final`; the host owns commits, branches,
and PRs. Sase declarations that change user-reaching contracts use a `feat!:` subject
and a `BREAKING CHANGE:` footer listing command/path targets, config/frontmatter keys,
public env names, plugin groups, discovery directories, doctor ids, and CLI JSON values
changed by that turn. Do not manually commit. The original epic, other phase beads,
compatibility flag removal, core-flip, TUI goldens, docs/memory migration, plugin floor
changes, and machine/skill deployment remain the responsibility of their assigned
owners.
