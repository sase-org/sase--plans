---
tier: epic
title: Finish the additive Rust macro rename and close sase-1eq.1
goal: Complete all remaining core-expand contracts, prove compatibility with unchanged
  sase, and close only the original phase after its child epic lands.
parent_bead: sase-1eq.1
phases:
- id: catalog-editor-names
  title: Rename catalog and editor internals with pinned legacy output
  size: medium
  depends_on: []
  description: 'catalog-editor-names: Complete catalog/editor/content-layout internal
    identifier renames, update Rust consumers through owning-module imports, preserve
    all existing serialized values and diagnostics, and register the three macro editor/skill
    binding aliases. Follow the catalog-editor-names section and shared compatibility
    rules.'
- id: runtime-wire-names
  title: Rename runtime wires and normalize prompt proc aliases
  size: medium
  depends_on:
  - catalog-editor-names
  description: 'runtime-wire-names: Rename scan/statistics/launch/proc and remaining
    runtime internals with legacy serialization pins and new request aliases. Normalize
    prompt-proc inputs, add prompt_proc_origin, and complete root/prelude cleanup.
    Preserve index schemas, contracts, parity expectations, and emitted diagnostics.'
- id: catalog-sources
  title: Add canonical macro sources and legacy loading policy
  size: medium
  depends_on:
  - runtime-wire-names
  description: 'catalog-sources: Add canonical macro layouts and source lists beside
    untouched legacy layouts; load new-first package/plugin/home/project paths and
    transport variables. Thread default-true accept_legacy_xprompt_names through core
    and Python catalog entry points, rejecting retired definition sources when false
    while preserving skills and memory placement. Cover explicit resource precedence
    and option aliases.'
- id: authored-inputs
  title: Accept macro definition keys and permanent directive aliases
  size: medium
  depends_on:
  - catalog-sources
  description: 'authored-inputs: Accept macros keys in all YAML/frontmatter/config
    loading and editor paths, diagnose duplicate spellings and retired authored keys,
    support both literal-zone directive families including mixed markers, and emit
    both local-definition child environment variables. Keep legacy presentation and
    output unchanged.'
- id: durable-readers
  title: Read new artifact filenames with permanent legacy fallbacks
  size: medium
  depends_on:
  - authored-inputs
  description: 'durable-readers: Implement new-first macros.json/raw_prompt.md selection
    in scans and alias history, preserve indexed storage while invalidating signatures
    on selection changes, and recognize renamed home state files. Verify cold/indexed
    scans, capacity-only behavior, and durable compatibility independent of the legacy
    option.'
- id: lsp-inputs
  title: Expose the macro LSP binary, commands, and policy-aware catalogs
  size: medium
  depends_on:
  - durable-readers
  description: 'lsp-inputs: Add sase-macro-lsp to the existing package, new command
    aliases and document paths, new-first metadata environment variables, and the
    legacy initialization option throughout refresh/cache/helper behavior. Prevent
    stale or helper catalogs from restoring retired definitions when false; preserve
    old protocol output.'
- id: compatibility-audit
  title: Verify the combined additive contract against unchanged sase
  size: medium
  depends_on:
  - lsp-inputs
  description: 'compatibility-audit: Audit all residual terminology and protected
    output against the starting core, fix omissions within core-expand, run the complete
    core gate, install the combined core into the unchanged sase workspace, and run
    core health, focused Python compatibility tests, and the sase gate. Record evidence
    on this phase and sase-1eq.1. Prepare closure evidence; leave ancestor closure
    to the child land agent.'
proposed_by: bbugyi200.athena.sase-1eq.1.f0
create_time: 2026-10-02 07:55:43
status: wip
bead_id: sase-1eq.1.1
---

- **PROMPT:** [prompts/202610/finish_core_macro_expand.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_core_macro_expand.md)
- **BEAD:** [sase-1eq.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eq/sase-1eq.1.1.md)

# Finish core-expand without changing its output contract

This is the completion plan for **sase-1eq.1**, the `core-expand` phase of
`plan:202610/xprompts_to_macros.md`. It replaces the unimplemented portions of
`plan:202610/core_macro_expand.md`; it does not restart the work already landed. Read
both artifacts with `sase artifact read` and read the current phase with
`sase bead read sase-1eq.1 -r "Need the remaining additive macro scope"`.

The inspected sase-core HEAD was `015ce7f6ad1cc5ade253dc6d174ce55ae9dc30d3`, with a
clean working tree. It already contains the module moves to `macro_catalog/`,
`macro_text_block.rs`, `editor/macro_args.rs`, and `agent_stats/run/macros.rs`
(including tests), plus the query-shorthand internal rename and the `shorthands` profile
alias. Its note reports passing query tests and a core gate. Those results do not verify
the remaining work. Preserve this commit, avoid repeating its moves, and recheck the
query compatibility contract during integration.

The inspected remainder includes roughly 800 editor and 600 catalog occurrences of the
old terminology, runtime wire changes, three kinds of source discovery, durable cache
invalidation, and LSP helper/cache policy. It exceeds a single direct implementation
tale. The seven medium phases above run sequentially because they share types, imports,
loaders, and bindings. Every intermediate commit must remain compatible with existing
Python callers. No worker needs to author another plan for the bounded phase described
here.

`parent_bead: sase-1eq.1` deliberately attaches this completion epic to the original
phase, including when proposed from a fork without a bead environment variable. Approval
owns creating and launching its child phases. Do not manually create beads. This plan's
land agent closes the child epic and then the original phase using the nested-landing
procedure below. Child phase workers never close ancestors.

## Repository and compatibility rules for every phase

Open **sase-core** using
`sase repo open sase-core -r "Implement <phase> for the sase-1eq.1 completion plan"`
from the assigned sase workspace. Read the printed checkout's `AGENTS.md`; all `crates/`
paths below refer to that checkout. Record the starting revisions and local changes. Use
only the printed checkout and the assigned primary workspace, never another agent's
checkout or a hard-coded workspace path.

Implementation belongs in Rust. Keep the primary sase tree unchanged: Python caller
migration, its CI pin, sunset flag creation, deployment tooling, docs/memory migration,
gateway route migration, and the output contract flip belong to the original epic's
later phases. Do not edit package versions, CHANGELOGs, SQLite columns/migrations,
schema version values, gateway contract snapshots, mirrored fixtures, or the LSP
crate/package name. No new binding calls are introduced in sase by this plan, so its pin
is not moved here. Declare every repository actually changed through `/sase_final`; use
additive `feat:` commits and let the host create commits/branches/PRs.

Existing serialized fields, enum values, omitted/default fields, ordering, error
messages, diagnostic codes, hover/code-action text, and server identity stay unchanged.
Rename Rust members with `#[serde(rename = "<legacy>", alias = "<new>")]` where
appropriate; explicitly pin variants governed by `rename_all`. Manual JSON/YAML parsers
and string-returning methods need equivalent handling. Duplicate old/new keys are errors
even when one value is empty or null. Environment variables and artifact filenames
instead use new-first precedence with a legacy fallback.

The permitted additive output changes are the macro layout/source keys, accepted
request/options keys, the legacy-loading option, four binding exports, two LSP command
IDs, and `SASE_AGENT_LOCAL_MACROS` in the launch environment. Do not opportunistically
modernize existing output strings. Keep exported legacy Python schema lookups valid.

Reserve bare `macro` in Rust: use `macro_def`, `macro_ref`, or another descriptive
identifier. Retain Jinja's own macro keywords and `JinjaLocalKind` variants, unrelated
Rust/tooling terminology, and the LSP semantic-token legend (directives use `macro`,
definition references use `function`). Query status shortcuts are **shorthands**; their
wire still emits `macros`, including its digest and existing errors.

Keep an explicit inventory of remaining case-insensitive `xprompt` hits per phase.
Classify every retained hit as an emission pin/presentation contract, input alias,
legacy-reader test, unchanged LSP package/binary/command, protected contract/fixture, or
history. Annotate durable and gateable legacy constants with
`// legacy xprompt spelling`. Use a repeatable scratch codemod if useful, outside
tracked product sources, with exclusions for these constants, `LEGACY_*`/`legacy_*`,
protected fixtures/contracts, and history. Review the resulting diff.

**One precise correction to the earlier protected-file rule:**
`crates/sase_core/tests/python_wire_parity.rs` directly constructs a `ProcWire` using
`xprompt_proc: None` (line 392 in the inspected tree). Renaming that Rust field requires
changing this one initializer to `prompt_proc: None`. This plan permits that mechanical
identifier edit only; all expected JSON, schema numbers, and parity assertions stay
byte-for-byte unchanged. This resolves the earlier plan's conflicting requirements
without adding a duplicate field or weakening the compatibility witness.

Gateway contract-named types, such as `MobileXprompt*Wire`, remain named as before.
Remove their obsolete root/prelude aliases only by importing from their existing owning
modules; do not rename the protected types. A remaining protected type name is a
classified exception, not an invitation to regenerate a snapshot.

## Phase catalog-editor-names

Rename internal catalog and editor terminology in `macro_catalog/`, `content_layout.rs`,
`editor/`, related completion/Jinja wires, and their Rust consumers. This phase changes
names and accepts wire aliases; it does not yet change source selection or parsing
policy. Examples include `CatalogXprompt` to `CatalogMacro`, `XpromptSourceWire` to
`MacroSourceWire`, `MemoryXprompt*Wire` to `MemoryMacro*Wire`, `XpromptArgumentSource`
to `MacroArgumentSource`, catalog load/options/resource types, and skill-resolution
types/functions. Preserve `WorkflowKind` and `DefinitionSection` string results, catalog
diagnostics, and source locator strings until their behavioral phase explicitly adds new
inputs. Do not rename contract-named host-bridge types.

Update consumers in core, PyO3, gateway, and LSP in the same change. Remove the affected
xprompt root re-exports from `crates/sase_core/src/lib.rs` and the affected `core_*`
aliases in `crates/sase_core_py/src/prelude.rs`; import directly from owning modules. Do
not add replacement root exports or prelude aliases.

Register aliases on the same implementation in the PyO3 editor-completion domain:

| New Python export                            | Existing export retained                       |
| -------------------------------------------- | ---------------------------------------------- |
| `resolve_macro_skill_definition`             | `resolve_xprompt_skill_definition`             |
| `macro_skill_definition_wire_schema_version` | `xprompt_skill_definition_wire_schema_version` |
| `macro_argument_spans`                       | `xprompt_argument_spans`                       |

Use module registration aliases or the repository's equivalent single-implementation
pattern. Keep existing function signatures and exception text. Tests must look up both
names on an initialized module and compare representative results and failures; compile
success alone does not prove registration.

Verify old/new serde inputs serialize identically for renamed editor/Jinja/catalog
wires, including enum values and `as_str` implementations. Preserve existing wire and
argument corpus expectations. Run relevant catalog/editor/PyO3 tests and the shared
phase gate below.

## Phase runtime-wire-names

Complete the runtime identifier sweep in `agent_scan/`, `agent_stats/`, `agent_launch/`,
`procs/`, dispatch, artifact-reference kinds, and their binding/Rust consumers. Examples
include `UsedXPromptWire` to `UsedMacroWire`, statistics structures and request fields,
`used_xprompts` to `used_macros` internally, and the local-definition launch field to
`local_macros_file`. Keep serialized keys such as `used_xprompts`, `xprompts`,
`runs_with_xprompts`, `runs_without_xprompts`, `distinct_xprompts`, `xprompt_top_n`,
`xprompt_breakdown_top_n`, `xprompt_focus`, and `local_xprompts_file` pinned to their
actual legacy spelling. Inventory the exact current fields instead of trusting an
example list. Accept corresponding macro request aliases and reject both together.

The whole authored prompt uses **prompt**, rather than macro: rename
`XpromptProcMetaWire` to `PromptProcMetaWire`, metadata members to `prompt_proc`, and
the internal origin helper/constant accordingly. New `prompt-proc` input normalizes to
the current `xprompt-proc` emitted value. All stored rows, reservations, updates,
classification, and equality paths treat either spelling as the same proc family.
Register `prompt_proc_origin` alongside `xprompt_proc_origin`, returning the same legacy
value. Preserve exception text and all existing schema versions.

Apply the permitted single parity-test initializer change described above. Keep index
columns and serialized record JSON unchanged. Move affected consumers away from root
re-exports/prelude aliases and finish any nonprotected internal identifier omissions
from the previous phase. Any additional whole-prompt mapping must be checked for name
collisions and documented on the phase bead before use.

Test old/new request equality and duplicate-key rejection, byte-identical output, all
proc mutation/read paths, and old stored records. Run the protected parity test, the
query profile/digest suite, relevant stats/scan/launch/proc/PyO3 tests, and the shared
phase gate.

## Phase catalog-sources

In `content_layout.rs`, add `macros` to project/home/chezmoi layouts and `macro_sources`
to the aggregate layout. Keep every existing `xprompts`, `xprompt_sources`, and
`skill_sources` value unchanged; construct separate canonical macro metadata rather than
changing the old lists in place. Canonical writers resolve to `sase/macros`, with the
corresponding chezmoi macro path and retired `dot_xprompts` candidate.

Order canonical project `sase/macros`, home `~/sase/macros`, and project-specific home
`~/sase/macros/<project>` directory sources before retired xprompt-named directories.
Preserve the established scope ordering within each family, first-wins behavior, project
namespaces, and the old-only installation's precedence. Keep config collision rules and
skill/memory placement rules. Include `entrypoint:sase_macros/...`, `package:macros`,
`package:macros/steps`, `package:macros/skills`, and `package:default_macros` locators.
Expose canonical skill locator additions without mutating the legacy serialized lists;
do not move dedicated `sase/skills` or plugin `skills/` sources.

Teach `macro_catalog/{types,loader,loader_sources}.rs` to consume the canonical list.
Probe `macros/` before `xprompts/`, `default_macros/` before `default_xprompts/`, and
`macros/skills` before `xprompts/skills`. Explicit resource options retain their
precedence over inferred environment/package paths. Distinguish a retired definition
source from unrelated legacy config/memory paths; do not filter every generic
`LayoutPathRoleWire::Legacy` indiscriminately.

Resolve these env suffixes with `SASE_MACRO_` first and `SASE_XPROMPT_` fallback:
`PACKAGE_DIR`, `BUILTIN_DIR`, `DEFAULT_DIR`, `PLUGIN_DIRS_JSON`, and
`PLUGIN_CONFIG_PATHS_JSON`. Test both-present precedence, not merely one spelling at a
time. The LSP phase owns the remaining metadata catalog suffixes.

Add `accept_legacy_xprompt_names`, default **true**, to all relevant catalog options and
entry points. Ensure every default/constructor path yields true (a derived bool default
would silently yield false). Thread it through editor, snippet and skill loading, PyO3
option parsing, and any Rust callers that instantiate option structs. Preserve existing
Python positional arguments; new policy arguments must be optional. Accept
`package_macros_dir`, `default_macros_dir`, and `plugin_macro_dirs` alongside their old
keys, with duplicate rejection even for empty maps. Existing option keys and transport
env names remain accepted during this expansion phase.

When false, skip retired directory/resource sources even when passed through an explicit
option or plugin metadata; leave durable readers and old literal-zone markers
unconditional. Authored-key errors are completed by the following phase, and LSP
cache/helper propagation by the LSP phase; do not claim those paths complete here.

Test canonical-only, old-only, both families with conflicting definitions, explicit
resources, plugin/package skill placement, config collision behavior, option defaults,
and both policy states. Compare the old layout keys against their previous values. Run
the relevant layout/catalog/PyO3 tests and shared phase gate.

## Phase authored-inputs

Accept `macros:` in global/project `sase.yml`, local definition files, YAML workflows,
Markdown frontmatter, and nested local helpers. Cover catalog parsing as well as
`editor/frontmatter.rs` and `editor/diagnostics.rs`, including helper discovery and
argument validation. Detect both old and new keys by presence before attempting to parse
their values; empty or null input is still a conflict. When the loading policy is false,
retired authored keys produce an error or editor diagnostic naming `macros`. Do not
silently drop malformed or forbidden definitions in a parser returning `Option`. Keep
all preexisting diagnostics for valid legacy-policy inputs unchanged.

Make `%macros_enabled` and `%xprompts_enabled` equivalent in
`agent_launch/directive_scan.rs`, `editor/directive/metadata.rs`, `editor/wire.rs`, and
every consumer of the shared literal-zone scanning logic. Test mixed-family
opening/closing markers, fences, directive stripping, and argument/reference scanning
inside disabled regions. The legacy directive is permanent and works with the catalog
policy set to false. Existing emitted directive metadata remains canonicalized to the
legacy spelling; accepting the new input must not change the old metadata contract.

In launch preparation, verify the new local-definition request alias added by
runtime-wire-names, then set both `SASE_AGENT_LOCAL_MACROS` and
`SASE_AGENT_LOCAL_XPROMPTS` to exactly the same file. Test serialized launch results,
including absence behavior when no local definitions are supplied.

Test old-only, new-only, both-key, and false-policy cases at each parser boundary, not
solely at the top-level catalog. Run relevant editor/catalog/launch tests and the shared
phase gate.

## Phase durable-readers

In `agent_scan/scanner.rs`, prefer `macros.json` and `raw_prompt.md`, falling back to
`xprompts.json` and `raw_xprompt.md` only when the canonical file is absent. Apply the
same raw-prompt selection to `agent_scan/index/alias_history.rs`. A present malformed
canonical file follows existing malformed-file behavior; a stale legacy file must not
override it. Factor the shared selection rule within the owning Rust domain to keep cold
scans and indexed reads consistent.

Update `agent_scan/index/record_summary.rs` signatures/marker detection to observe the
selected artifact and its identity, including late canonical creation, replacement, and
deletion back to the legacy file. A canonical and legacy file with equal size and mtime
must not disguise a source switch. Preserve `xprompts_sig`, schema v19 behavior, and
stored `record_json.used_xprompts`. Ensure raw-prompt selection also refreshes cached
alias-history snippets through its existing invalidation route. Keep capacity-only fast
paths free of unnecessary file reads; no archive rebuild, new index schema, or frontend
work is needed.

Add `vcs_macro_mru.json` and `macro_save_state.json` to the recognized home-zone names
in `note_attachment/zones.rs`, retaining the old files. Reuse the earlier proc tests as
durable evidence; the legacy catalog option must not gate any of these readers.

Test new-only, old-only, both-present, malformed canonical, late write, source removal,
and indexed refresh for both cold scan results and alias snippets. Run relevant
scan/index/note-attachment tests and the shared phase gate.

## Phase lsp-inputs

Keep `crates/sase_xprompt_lsp` and package `sase_xprompt_lsp`. Add a second `[[bin]]`
named `sase-macro-lsp` pointing at `src/main.rs`. Retain `sase-xprompt-lsp`, legacy
`serverInfo.name`, version output, log/user text, code-action labels, diagnostics, and
semantic-token ordering. Rename internal `XpromptLspServer` and remaining unprotected
LSP identifiers with the normal mapping.

Advertise and dispatch `sase.macroLsp.refreshCatalog` and `sase.macroLsp.openSource`
alongside the old IDs with identical payload handling. Recognize `macros`,
`default_macros`, and `macros.yml|yaml` in `server/{actions,jinja}.rs` and all
corresponding document classifiers/watchers.

Read `VCS_PROJECT_CATALOG`, `MODEL_CATALOG`, `MACHINE_CATALOG`, `ARTIFACT_REF_CATALOG`,
and `GLOSSARY_CATALOG` with the new `SASE_MACRO_` prefix first and the old prefix as
fallback. Plugin metadata detection and loading must both recognize the new variable
family added by catalog-sources.

Add the default-true `accept_legacy_xprompt_names` initializationOption to
`server/{initialize,state}.rs`, carry it into every catalog/snippet/skill refresh, and
include it in `catalog_cache.rs` cache identity/invalidation. A catalog loaded under
true may not be reused under false. Review stale-on-error and helper-merge paths as well
as fresh Rust loads. An old helper that cannot prove its entries comply with false must
not restore retired definitions; use the policy-aware Rust result or return an existing
appropriate failure rather than merging unverified helper data. Keep bridge command
names and their wire contracts unchanged. No Python helper change is in scope.

Add JSON-RPC tests for both command families, capabilities, source actions, new paths,
both option states, true-to-false cache isolation, helper fallback, metadata env
precedence, and unchanged semantic-token/diagnostic output. Run
`just test -p sase_xprompt_lsp --bin sase-macro-lsp` as well as the relevant legacy
target tests and the full shared phase gate. Test the built new executable's supported
version invocation directly. Current sase `rust-dev-install` builds package binaries but
copies only the old binary into its venv; changing that installer belongs to the next
original phase, so do not assert that the new executable is on PATH here.

## Shared phase verification and work accounting

Use sase-core's `just fmt`, `just fast`, and targeted `just test -p <crate> <filter>`
while iterating. Never run bare cargo: these recipes provide Python >=3.12 and the
hermetic environment. The complete per-change gate is `sase tool run check` from the
linked checkout, including format, feature unification, clippy, workspace tests,
PyO3/LSP tests, and script tests. Targeted tests do not replace it. Follow the
documented feature-unification repair only if that gate requires it.

Long builds/gates use `/sase_monitor` before starting. Monitor continuations must name
the assigned phase, outstanding checks, and its own closure/final-declaration steps.
Wait until each monitor-start command exits; do not finish a turn with an unhanded
background command, and do not cancel an existing check to move it to a monitor. Run
formatting before a final verification handoff. Never run `just check-full` for this
plan.

Record implementation, residual-hit classification, and command/evidence IDs on the
assigned child phase bead. Record discovered out-of-scope work as `PROPOSED FOLLOW-UP:`
notes; child workers do not create task beads. New failures caused by the phase must be
fixed. A failure reproduced identically on the clean starting tree is documented with
its independent witness and any existing tracking bead, rather than hiding the failure
or leaving a finished phase open indefinitely. Keep any baseline reproduction in an
isolated scratch checkout within the assigned workspace and do not overwrite active
work.

Before closing a child phase, run `sase bead epic-symbols <assigned-phase>`, resolve or
re-key each entry to a still-open appropriate bead, then close only that assigned phase
with its verification note. Submit the host-owned final declaration. A phase's passing
check proves its bounded deliverable; only the final audit proves the complete original
phase contract.

## Phase compatibility-audit

Read the accumulated phase notes and recheck the two original plan artifacts. Audit the
aggregate diff from `015ce7f6ad1cc5ade253dc6d174ce55ae9dc30d3`, allowing unrelated
upstream drift only when independently identified. Confirm every original requirement
has an implementation owner and test. Fix omissions introduced by these phases before
claiming completion; do not reclassify unfinished core-expand work as a follow-up.

Required evidence:

| Contract                        | Evidence                                                                                                                                                |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Old output compatibility        | Paired old/new inputs serialize identically; protected JSON/fixtures/schema versions and existing presentation strings preserved                        |
| Layout/discovery                | Legacy layout lists unchanged; canonical sources precede retired sources; scope, config collision, explicit resources and skill/memory behavior covered |
| Legacy policy                   | Defaults true; false blocks retired sources/keys across core, PyO3 and LSP, including cache/helper fallback                                             |
| Permanent durable compatibility | New-first filename/proc reads, legacy fallback, indexed invalidation and both directive families independent of policy                                  |
| Additive public API             | All four old/new binding pairs resolve and agree; both LSP binaries build; both command families work                                                   |
| Query shorthand                 | Existing digest/errors/output remain unchanged; old/new profile keys agree and conflict when both supplied                                              |
| Residual terminology            | Every tracked hit classified; no unexplained internal leftovers, new root exports, or prelude aliases                                                   |

Run the complete sase-core `sase tool run check` on the combined tree unless a prior run
already proves that exact combined content. In the assigned primary sase checkout, read
`lint_and_test.md` with `/sase_memory_read`; prepare a fresh venv with `just install` if
needed and build this linked core using `just rust-dev-install`. Use `/sase_monitor` for
both known-long commands. Inspect the recipe before executing so its refresh step does
not replace the intended linked revision or changes. Record the core revision/dirty
state before and after, imported `sase_core_rs` path, and the primary sase revision. No
tracked sase source change is allowed to make this gate pass.

Run `sase core health` through the workspace environment and `sase tool run check` in
sase. Because that check is diff-scoped, also run the existing relevant Python tests
explicitly with `just test <test paths>`. Include `tests/test_content_layout.py`,
`tests/test_core_health.py`, `tests/test_editor_helper_xprompt_catalog.py`,
`tests/test_editor_helper_snippet_catalog.py`, the argument corpus users,
`tests/core/test_agent_launch_wire_contract.py`,
`tests/core/test_agent_launch_prepare_spawn.py`,
`tests/test_core_agent_scan_wire_xprompts.py`, proc wire consumers, and statistics
consumers found by searching for the renamed binding/wire fields. Adjust paths only for
genuine upstream moves and record the actual selected tests. Explicitly exercise legacy
Python binding registrations and the unchanged Python layout mirror against the rebuilt
extension. Do not regenerate old expected data to pass.

Record aggregate evidence and any identical clean-base failures on **sase-1eq.1** as
well as the assigned audit phase. Close only the audit phase after its symbol check.
Leave the original phase open for the child epic's land agent.

## Child land agent: close sase-1eq.1 after the complete proof

This section is for the child epic's land agent, never a child phase worker. Reconcile
all descendants and evidence, review post-phase drift, and apply the standard child epic
landing protocol. Do not rerun unchanged gates without a reason; changed content or
unresolved concerns need fresh appropriate verification. Resolve or intentionally triage
proposed follow-ups under the normal landing rules, and fix any remaining core-expand
defects before closing.

1. Check symbols and close this child epic normally only after all its child phases are
   closed and the compatibility-audit proof is complete. Mark this child plan done
   through the standard plan artifact workflow; never force-close descendants.
2. Read the parent link and original phase again. Verify the child completed **all** of
   sase-1eq.1's additive contract, including unchanged-sase verification, rather than
   merely the mechanical renames. Check `sase bead epic-symbols sase-1eq.1` again; the
   planning inspection found no entries, but later phases may add them. Resolve each
   entry or re-key it to the still-open original epic or a genuinely appropriate later
   phase before closing.
3. Run
   `sase bead close sase-1eq.1 --note "<combined core gate; unchanged sase revision, rebuilt core identity, health and compatibility tests; both bindings and binaries; policy/durable coverage; any independently reproduced clean-base failures>"`.
   Do not close **sase-1eq** or any other ancestor. The containing epic already has its
   own land agent and downstream work.
4. Use `/sase_final` as the final action if this is a normal turn completion. Declare
   all actual code changes and truthful bead completion. No separate manual git commit
   and no wait for this turn's future host-created commit or CI is required for closure.

The requested work is complete only when every remaining additive requirement has
evidence, unchanged Python callers still pass, durable legacy state still reads, and
**sase-1eq.1 is closed**. Merely submitting this plan or finishing its internal rename
phases does not satisfy that completion condition.
