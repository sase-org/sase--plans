---
tier: tale
size: medium
title: Additive macro rename in sase-core for sase-1eq.1
goal:
  Complete sase-1eq.1 by accepting macro inputs and bindings in sase-core while
  preserving existing wire output and unchanged sase compatibility.
proposed_by: bbugyi200.athena.sase-1eq.1
bead: sase-1eq.1
create_time: 2026-10-02 07:03:21
status: wip
---

- **PARENT:**
  [202610/xprompts_to_macros.md](https://github.com/sase-org/sase--plans/blob/main/202610/xprompts_to_macros.md)
- **BEAD:** sase-1eq.1

# Additive macro rename in sase-core

Implement reserved phase **sase-1eq.1**, `core-expand`, from
`plan:202610/xprompts_to_macros.md`. This is one bounded implementation turn in the
linked **sase-core** repository. The goal is to let later phases adopt macro inputs and
Rust names while the existing sase Python tree continues to work against this core. The
output contract changes happen in the later `core-flip` phase.

Use `sase repo open sase-core -r "Implement additive macro rename for sase-1eq.1"` and
read the printed checkout's `AGENTS.md` before editing. All paths below beginning with
`crates/` are relative to that checkout. The primary sase checkout supplies the
unchanged Python compatibility check. Do not modify another phase's Python callers,
configuration, feature-flag registry, deployment scripts, or memory files.

## Contract and scope

- Keep every existing serialized field, enum value, schema version, diagnostic code,
  diagnostic message, hover string, code-action title, server identity, and SQLite
  column unchanged. For an internally renamed serialized item, use
  `#[serde(rename = "<legacy>", alias = "<new>")]`. Preserve omission/default rules and
  object ordering. Apply equivalent normalization to manually parsed JSON.
- The deliberate additive exceptions are the new content-layout `macros` and
  `macro_sources` fields, accepted request/options aliases, the legacy-input option, new
  binding exports, and two additional advertised LSP command ids. Existing
  content-layout keys retain their old values and source lists. Existing capability
  values remain unchanged except for advertising the new commands.
- New names are accepted unconditionally in this phase. The
  `accept_legacy_xprompt_names` loading option defaults to **true**; false filters
  retired definition sources and reports retired authored keys with their macro
  replacement. Durable artifacts, proc rows, and stored `%xprompts_enabled` regions
  continue to load regardless of this option.
- Supplying both old and new keys for the same logical input is an error or an editor
  diagnostic, including config, frontmatter, request/options, and profile keys.
  Environment variables and artifact filenames instead use new-first precedence with a
  legacy fallback.
- Leave `crates/sase_gateway/contracts/`, the types named in those contracts,
  `crates/sase_core/tests/python_wire_parity.rs`, mirrored fixture files such as
  `tests/fixtures/xprompt_args_corpus.json`, CHANGELOGs, package versions, and the
  `sase_xprompt_lsp` package/path unchanged. Schema/index migrations, gateway routes,
  emitted terminology changes, and removal of legacy exports belong to core-flip.

## 1. Establish the compatibility inventory

Record the clean starting revisions and inspect all tracked case-insensitive `xprompt`
hits in sase-core. Inventory serializable fields/variants, string-returning methods,
hand-read JSON/YAML, environment variables, source locators, and binding registrations
before applying a rename. Read the whole parent design through `sase artifact read` if
any shared policy is unclear.

Preserve the existing parity corpus as the contract witness. Add focused tests beside
the affected domains to compare old-input and new-input results, including serialized
JSON strings where byte equality matters. New tests may retain explicitly identified
legacy spellings; do not regenerate the protected corpus to make a failure pass.

Treat the query-language status macro concept first: its new internal name is
**shorthand**, distinct from reusable prompt macros. Keep Jinja's own macro keyword,
`JinjaLocalKind::{MacroName, MacroParam, MacroSpecial}`, Rust tooling terminology, and
the LSP semantic-token legend unchanged. In particular, directive tokens retain the
standard `macro` token type and definition references retain `function`.

## 2. Rename Rust internals and preserve their wires

Make a re-runnable scratch codemod outside tracked product sources, with explicit
exclusions for legacy constants, protected contracts/fixtures, and history. Apply the
parent's case-aware/article replacement rules only to approved internal identifiers and
prose. Use `git mv` for:

- `crates/sase_core/src/xprompt_catalog/` to `macro_catalog/`.
- `crates/sase_core/src/xprompt_text_block.rs` to `macro_text_block.rs`.
- `crates/sase_core/src/editor/xprompt_args.rs` to `editor/macro_args.rs`.
- `crates/sase_core/src/agent_stats/run/xprompts.rs` and its `run/tests/xprompts.rs`
  counterpart to `macros.rs`.

Update module declarations, imports, internal tests, and downstream Rust callers across
core, PyO3, LSP, and gateway compilation units. Rename types/functions such as
`CatalogXprompt`, `XpromptSourceWire`, `MemoryXprompt*Wire`, `UsedXPromptWire`,
`XpromptArgumentSource`, and `XpromptLspServer`. Use the parent Rule 2 for authored
prompts: `XpromptProcMetaWire` becomes `PromptProcMetaWire`, the proc metadata field
becomes `prompt_proc`, and `local_xprompts_file` becomes `local_macros_file`. Do not
apply that rule to additional ambiguous identifiers without checking for collisions and
recording the mapping on the phase bead.

Rust reserves bare `macro`; use `macro_def`, `macro_ref`, or another descriptive
identifier, and serde attributes for JSON keys that must be `macro`. Rename serialized
members in editor/completion/Jinja wires, scan/index records, stats requests/results,
launch requests, proc wires, and artifact-ref kinds with their legacy emission pins.
Review every renamed enum that relies on `rename_all`, every `as_str`/manual map, and
every strict request parser. Wire-schema constant values remain unchanged, and any
renamed constant needed by an existing Python lookup retains that lookup's export.

Remove xprompt-named root re-exports from `crates/sase_core/src/lib.rs` and their
`core_*` prelude aliases from `crates/sase_core_py/src/prelude.rs`. Import renamed items
from their owning module paths everywhere instead of adding replacement root exports or
prelude aliases. Keep the public Python legacy bindings available.

In `query/profile.rs` and its callers, use `QueryShorthandSpec`,
`HOST_SHORTHAND_TRIGGERS`, `shorthand_target`, `shorthands_for_trigger`, and
`validate_shorthands`. The profile accepts `shorthands` as an alias while emitting
`macros`; preserve digest construction, error text, query evaluation, and completion
wire values. Compare both profile spellings with the existing query fixtures.

## 3. Add canonical sources and the legacy-loading policy

In `content_layout.rs`, retain the old `xprompts` layouts and `xprompt_sources` list
unchanged while exposing a separate canonical macro layout and ordered macro source
list. Add `macros` in the project, home, and chezmoi layouts and `macro_sources` in the
aggregate result. The macro layouts write to `sase/macros` and expose retired paths as
legacy candidates, including the chezmoi `dot_macros` counterpart.

In the new source list, canonical project `sase/macros`, home `~/sase/macros`, and
home-project `~/sase/macros/<project>` directories precede retired definition
directories, with those directories explicitly marked legacy. Preserve the existing
scope precedence, first-wins behavior, config-collision rules, namespacing, and
skill/memory placement rules. Include `entrypoint:sase_macros/...`, `package:macros`,
`package:macros/steps`, `package:macros/skills`, and `package:default_macros` locators.
Add new package/plugin skill source locators without changing the old serialized source
list.

Teach `macro_catalog/{types,loader,loader_sources,parsing}.rs` to consume this canonical
list. Resolve packaged `macros/` before `xprompts/`, `default_macros/` before
`default_xprompts/`, and `macros/skills` before `xprompts/skills`; keep dedicated
`sase/skills` and plugin `skills/` placement behavior. Preserve existing explicit
resource-option precedence.

Read these transport variables new-first with corresponding `SASE_XPROMPT_*` fallbacks:
`SASE_MACRO_PACKAGE_DIR`, `BUILTIN_DIR`, `DEFAULT_DIR`, `PLUGIN_DIRS_JSON`,
`PLUGIN_CONFIG_PATHS_JSON`, `VCS_PROJECT_CATALOG`, `MODEL_CATALOG`, `MACHINE_CATALOG`,
`ARTIFACT_REF_CATALOG`, and `GLOSSARY_CATALOG` (each latter suffix also has the
`SASE_MACRO_` prefix). Recognize new plugin metadata variables in LSP detection as well
as actual loading. Accept the Python option keys `package_macros_dir`,
`default_macros_dir`, and `plugin_macro_dirs` alongside their legacy equivalents in
`editor_completion/mod.rs`.

Thread `accept_legacy_xprompt_names`, default true, through all relevant catalog loading
requests/options and their entry points, including snippet/skill loading, editor
services, and LSP refreshes. False skips all xprompt-named directory/resource sources,
rejects or diagnoses old authored definition keys, and names the macro replacement. Mark
durable legacy string constants with `// legacy xprompt spelling`; keep gateable
authored-name constants equally explicit. Do not create the Python sunset feature flag
here; the later phase owns it.

Add tests for canonical precedence when both directory families exist, old-only
installations, explicit resources, package/plugin skills, both plugin metadata variable
families, and the false option. The existing sase mirror in
`src/sase/core/content_layout_wire.py` reads selected mapping keys and tolerates the
additions; verify this against the installed new core without editing that mirror.

## 4. Accept macro keys and the permanent directive alias

Handle `macros:` in global/project `sase.yml`, project-local definition files, YAML
workflows, and Markdown frontmatter. Cover each path through catalog parsing,
`editor/frontmatter.rs`, and `editor/diagnostics.rs`, including local helper discovery
and argument validation. Reject two spellings together even when one map is empty.
Retain existing diagnostics and presentation for existing inputs; only conflict and
retired-name diagnostics required by the new policy are added.

Recognize `%macros_enabled` in `agent_launch/directive_scan.rs`,
`editor/directive/metadata.rs`, `editor/wire.rs`, and every consumer of the shared
literal-zone scanner. Both markers disable/enable expansion and directive scanning,
including mixed-family opening/closing markers. Keep `%xprompts_enabled` as a permanent
alias even when the legacy catalog option is false. Until core-flip, normalize aliases
to the legacy emitted metadata rather than renaming the existing directive list
wholesale.

In launch preparation, accept the new local-definition request key and set **both**
`SASE_AGENT_LOCAL_MACROS` and `SASE_AGENT_LOCAL_XPROMPTS` to the same file for children.
Exercise serialized launch results as well as scanning behavior; the extra environment
entry is the specifically required additive child transport.

## 5. Read new durable filenames without an index migration

In `agent_scan/scanner.rs`, prefer `macros.json` and `raw_prompt.md`, falling back to
`xprompts.json` and `raw_xprompt.md`. Apply the same raw-prompt resolution in
`agent_scan/index/alias_history.rs`. Do not merge a stale legacy file over a present
canonical file. Keep existing malformed-file handling and capacity-only fast paths.

Update `agent_scan/index/record_summary.rs` so the existing signature slot observes the
selected new or legacy macro artifact and a canonical file's later appearance,
replacement, or deletion invalidates cached rows. Keep `xprompts_sig`, stored
`record_json.used_xprompts`, and the v19 migration behavior unchanged; internal Rust
names can change with serialization pins. Test cold scans, indexed refreshes, and
alias-history snippets for new-only, old-only, both-present, and late-write cases. Avoid
an archive rebuild or new work on a frontend/UI thread.

In `note_attachment/zones.rs`, recognize `vcs_macro_mru.json` and
`macro_save_state.json` alongside the old names. For procs, accept `prompt_proc`
metadata and normalize `prompt-proc` to the existing `xprompt-proc` output value. Cover
reservations, updates, existing stored rows, and classification/equality so either
origin spelling denotes the same proc family. The renamed `prompt_proc_origin` binding
still returns the legacy value during this expansion phase.

## 6. Expose additive bindings and LSP entry points

Register four new Python names on the same underlying implementations, retaining the
legacy registrations until core-flip:

| New binding                                  | Legacy binding                                 |
| -------------------------------------------- | ---------------------------------------------- |
| `resolve_macro_skill_definition`             | `resolve_xprompt_skill_definition`             |
| `macro_skill_definition_wire_schema_version` | `xprompt_skill_definition_wire_schema_version` |
| `macro_argument_spans`                       | `xprompt_argument_spans`                       |
| `prompt_proc_origin`                         | `xprompt_proc_origin`                          |

Update `editor_completion` and `agent_launch` registration functions and their PyO3
tests. Assert that both names are present and old/new invocations return identical wire
values and exceptions. Do not duplicate backend logic in an alias wrapper.

Keep `crates/sase_xprompt_lsp` and its Cargo package name. Add a second `[[bin]]` named
`sase-macro-lsp` using the existing `src/main.rs`, retaining `sase-xprompt-lsp`.
Advertise and dispatch `sase.macroLsp.refreshCatalog` and `sase.macroLsp.openSource`
alongside the old command ids. Keep existing command payloads, action labels, log/user
text, serverInfo, version behavior, and protocol semantic-token ordering compatible.

In `server/{actions,jinja}.rs` and related watchers/document classifiers, recognize
`macros`, `default_macros`, and `macros.yml|yaml` alongside old paths. Add the
default-true initializationOption to `server/{initialize,state}.rs` and carry it into
`catalog_cache.rs` refresh and cache identity/invalidation. A helper-merge fallback or
cached catalog loaded under true must never reintroduce a retired source after false is
selected. Preserve the existing bridge command/wire names until core-flip and test both
refresh command families, source actions, option states, and document paths.

## 7. Verify and review the additive contract

Use the linked repository's `just fmt`, `just fast`, and meaningful targeted
`just test -p <crate> <filter>` runs while iterating. These invoke `scripts/check.sh`,
which supplies Python >=3.12 and the hermetic test environment. Never use bare cargo.
`just test -p sase_xprompt_lsp --bin sase-macro-lsp` verifies the new binary target
compiles; the sase Rust dev-install recipe builds the unchanged LSP package's normal
binaries, including both names. Inspect the resulting new executable and smoke its
supported version invocation.

Run `sase tool run check` inside sase-core. This is the full linked-repo gate: format,
feature unification, clippy, all workspace tests including PyO3 and LSP, and script
tests. Do not replace it with core-only targeted tests. Repair features through the
repository's documented hakari procedure only if the feature gate requires it.

In the primary sase checkout, build/install this changed linked core into the local
workspace virtualenv using the existing `just rust-dev-install` recipe, then run
`sase core health` and `sase tool run check` with **no tracked sase source change**.
Record which extension was imported and the core revision/dirty state tested. Run the
existing relevant Python core wire/layout/editor/launch/statistics tests explicitly when
the unchanged tree's diff-scoped gate would otherwise omit them. Prove that old Python
bindings remain registered, the old layout mirror parses the additive layout, and
protected wire parity tests pass without changing their expected data.

Long builds and verification commands use `/sase_monitor`; wait for each monitor-start
handoff itself to exit, and ensure its continuation preserves this assigned phase and
completes all remaining verification/closure work. Run formatting before the final
verification monitor. Never start `just check-full`: it is outside this phase's
requested gate. Do not cancel a running check merely to move it to a monitor.

Review `git diff` for every emission pin, legacy constant, root/prelude export, gateway
contract, schema literal, and protected fixture. Classify every surviving
case-insensitive xprompt hit in this phase's scope as an emission pin, input alias,
legacy-reader test, unchanged LSP package/binary/command, protected contract/fixture, or
history. Record the inventory and verification evidence on sase-1eq.1; unexplained
internal identifiers are remaining work.

## 8. Close only the assigned phase and declare its work

Do not change bead status by hand. Do not create beads. Record discovered work with
`sase bead note sase-1eq.1 'PROPOSED FOLLOW-UP: <summary — evidence and detail>'`. If a
check failure reproduces identically on the clean base, record that witness and any
existing tracking bead in such a note; that failure does not prevent phase closure. New
failures caused by this work must be fixed.

Immediately before closing, run `sase bead epic-symbols sase-1eq.1`. The planning
inspection found no entries, but recheck after implementation. Resolve any new entries
or re-key the corresponding Justfile line to the still-open parent epic or appropriate
later phase before closing, without changing an ancestor's status.

Once the whole scope is implemented and verified, run
`sase bead close sase-1eq.1 --note "<core gate, unchanged sase compatibility gate, binding/binary and legacy-input evidence; any clean-base failures>"`.
**Do not close sase-1eq or any ancestor plan bead.** Then use `/sase_final` for the
host-owned commit declaration covering each repository actually changed. Use an additive
`feat:` Conventional Commit for core. Do not create commits, branches, or PRs manually.
This phase does not hand-edit the CI pin because it introduces no new binding calls in
sase; later caller/contract phases coordinate their required pin updates with host
finalization.

Completion means all accepted input families and both bindings work, legacy outputs and
callers pass the compatibility witnesses, both LSP binaries build, the false
legacy-loading policy holds without blocking durable readers, the required gates pass or
carry identical clean-base failure evidence, and only sase-1eq.1 is closed.
