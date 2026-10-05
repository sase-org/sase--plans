---
tier: tale
title: Complete plugin input types for phase sase-1g4.5
goal:
  Plugin shared enums resolve consistently in the runtime and LSP, with discoverable CLI
  output and actionable doctor findings.
size: medium
proposed_by: bbugyi200.athena.sase-1g4.5
bead: sase-1g4.5
create_time: 2026-10-05 13:03:03
status: wip
---

- **PARENT:**
  [202610/macro_named_input_types.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_named_input_types.md)
- **BEAD:**
  [sase-1g4.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g4/sase-1g4.5.md)

# Plugin input types for sase-1g4.5

## Assignment and scope

Implement the `plugin-types` phase of epic `sase-1g4`, tracked by the already assigned,
in-progress bead `sase-1g4.5`. This is one bounded follow-up implementation: the earlier
phases have already supplied the type catalog, enum rules, hint wires, completion,
diagnostics, and builtin `model`/`effort` behavior. Extend those paths rather than
rebuilding them. Do not set bead status manually.

Read the accepted design and its research with audited commands:

```bash
sase bead read sase-1g4.5 -r "Need the phase scope and design file"
sase artifact read plan:202610/macro_named_input_types.md "Need the accepted plugin-types design"
sase artifact read research:202610/macro_enum_inputs_named_types/macro_enum_inputs_named_types.md "Need the research context required by the epic"
```

During planning, the first bead read failed with
`bead id segment must contain only letters, digits, '-' and '_'` while expanding
artifact links. The audited retry with `--no-links` succeeded and the design was read
separately. Use that workaround if still necessary; repairing artifact identity is
outside this phase. Record a `PROPOSED FOLLOW-UP:` note on this phase if the problem
persists. The research sidecar initially lacked the report;
`sase repo open sase--research` refreshed it and the audited artifact read then
succeeded.

Read `cli_rules.md`, `macros.md`, `sase_beads.md`, `sase_artifacts.md`, and
`lint_and_test.md` through `sase memory read`. Before editing another repository, use
`sase repo open sase-core -r "Implement plugin macro input types"` and read its
`AGENTS.md`. All Rust paths below are relative to that printed checkout; all Python
paths are relative to the sase checkout. Reinspect the current tree before editing:
planning inspected sase `69b492c278` and sase-core `c57b662c`, and sibling phases may
have landed meanwhile.

Do not create beads or close any parent/ancestor. Capture discovered work only with
`sase bead note sase-1g4.5 'PROPOSED FOLLOW-UP: <summary — detail>'`. The
research-plugin dogfood, broad documentation rewrite, memory update, and TUI model-menu
work belong to other phases. This phase includes its own plugin/CLI documentation and
thin authoring adapters needed to expose the registry. No new config-defined types, live
callbacks, plugin bare-name shorthand, narrowing, gate declaration rules, or feature
flags.

## Contract to preserve

Plugins ship `input_types.yml` at the macro-plugin package root:

```yaml
schema_version: 1
types:
  audio_edition:
    description: Narration length for guide-backed audio editions.
    choices:
      - { value: brief, description: About 4 minutes }
      - { value: full, label: Full edition, description: About 16 minutes }
```

The only supported type is a static closed enum. IDs match `[a-z0-9][a-z0-9_-]*`. The
file envelope accepts exactly `schema_version` and `types`; each type accepts exactly
required `description` and `choices`. Descriptions must be strings and choices nonempty.
Choice objects accept `value`, optional `label`, and optional `description`. Reuse the
existing Rust choice validator and PyYAML scalar parity helper: values must be strings,
nonempty words without Unicode whitespace, distinct, and different from literal `null`.
Matching is exact and case-sensitive. Keep shorthand quoting warnings and choice order.
A type with any error is wholly skipped while valid sibling types remain usable;
malformed envelopes/files produce file diagnostics without breaking unrelated plugins or
macros.

Only `<distribution>@<id>` accesses a plugin type. Normalize distributions using the
existing Rust PEP 503 helper. Preserve case-insensitive bare builtins and
`builtin@<name>` aliases. Resolve a plugin type to base `enum`, its canonical
`named_type`, complete choices, and plugin provenance. Missing-plugin errors name the
distribution and install command; installed plugins with unknown IDs suggest known IDs.
Unknown-type guidance also points to `sase macro types`.

Wire additions must be optional/defaulted. Existing calls without a registry continue to
work. Runtime unresolved types skip only their macro and become load issues; editor
catalogs keep unresolved inputs visible as `line` while the authored source carries a
diagnostic. Preserve `strict_macro_input_types` compatibility behavior, builtin
model/effort semantics, and resolved-wire versus authored-YAML separation: serializers
write a named type without copying its resolved choices back into the declaration.

## 1. Load and expose the registry in sase-core

Extend `crates/sase_core/src/macro_input_types/registry.rs`, `resolve.rs`, and the
module facade. The existing registry has builtin entries plus only a distribution-to-ID
fixture map; its qualified resolver currently returns an enum with empty choices.
Replace that incomplete production path with actual catalog entries loaded from
discovery records shaped `{distribution, module, path}`.

Define serde request/result wires for loading the records and transferring a registry
snapshot to bindings. Include builtin and valid plugin entries, known distribution
identity, and structured diagnostics with severity, distribution, path, optional type
ID, and line/location. Keep file I/O and all schema/choice validation in Rust. Preserve
existing fixture-map compatibility where callers/tests rely on it, but production
resolution must never treat an ID-only fixture as a valid empty closed set.

Read each manifest once per registry load and keep its source text for line diagnostics
and unquoted YAML 1.1-sensitive values. Reuse `validate_enum_choices_yaml` and
`pyyaml_plain_scalar_is_non_string`, including for plain scalar choice objects. Do not
write another set of scalar resolver regexes or a new whole-frontmatter parser. Detect
duplicate IDs/qualified definitions and emit deterministic diagnostics rather than
silently overwriting; do not let entry-point aliases load the same file twice. A file
read/parse failure cannot discard valid files. Preserve known macro-plugin distribution
identity even when it declares no valid types so diagnostics distinguish an unavailable
plugin from an installed provider with no such ID.

Expose and register a `load_macro_input_type_registry` binding in
`crates/sase_core_py/src/macro_input_types/`, returning the Rust snapshot and findings.
Let `resolve_input_type` accept that snapshot in its existing request, and make
`macro_input_type_catalog` optionally project it. Preserve their old no-registry calls.
Add binding round-trip/registration coverage. Frontmatter schema, validation, and
field-validation bindings in `crates/sase_core_py/src/editor_completion/` gain optional
registry arguments while retaining existing signatures for old callers.

## 2. Discover files and connect the Python runtime

Add `discover_macro_plugin_input_type_files()` in or beside
`src/sase/main/plugin_discovery.py`. Reuse the existing canonical-first, deduplicated
`sase_macros` entry-point enumeration, admitting `sase_xprompts` only when
`legacy_xprompt_syntax` permits it. Retain `ep.dist` metadata instead of guessing the
distribution from the module or entry-point name. Locate the manifest at the package
root with `importlib.resources`; respect global/macro-plugin disable controls and log
failed entry-point imports without crashing the catalog. Follow existing concrete-path
resource conventions, and keep known distribution inventory available independently of
whether a manifest exists.

Implement a thin registry adapter under `src/sase/macro/` with process-local caching
keyed by discovered distribution/module/path identity and file stat signatures (mtime,
with size as appropriate). Add a cache-clear hook for tests/explicit refresh. File
edits, deletion/recreation, newly discovered entries, and changed legacy/disable policy
must invalidate the snapshot. Retain registry diagnostics in the cache but re-emit them
into every active `collect_macro_load_issues()` context: a prior catalog load must not
make the doctor blind to cached failures. Obtain the complete Rust registry without
recursively calling macro/config discovery.

Pass the snapshot from `_loader_parsing_inputs.py` through the resolver for shortform
and longform declarations. Existing file/config/local/plugin/skill/workflow paths share
this parser and must retain their per-macro catches. Ensure named enum defaults are
checked against resolved choices through the Rust rules and named types reject an
authored `choices` override with the explanation that the type already defines its
values. The current `parse_input_definition` replaces resolved choices when
`choices_raw` is present, so close that path for plugin types. Keep shared validation in
Rust and Python limited to argument conversion, discovery, caching, and rendering.

Use the registry in `src/sase/macro/frontmatter_schema.py` so the existing authoring
schema/validation callers see installed types through a thin adapter. Static committed
JSON schemas must remain generated from the builtin catalog plus the existing
qualified-type pattern; do not bake the machine's installed plugin types into them.

## 3. Give the LSP the same files and registry

In `src/sase/integrations/macro_lsp.py`, export discovery records directly as
`SASE_MACRO_PLUGIN_INPUT_TYPES_JSON`. Honor explicit caller environment values and the
existing canonical/legacy environment policy. This new variable is canonical; there was
no earlier input-types variable to invent a required legacy alias for. Do not
materialize choices into another on-disk JSON catalog.

In sase-core, connect registry loading to `macro_catalog` resource/options loading and
the actual catalog cache load/refresh path used by `crates/sase_macro_lsp`. Use one
registry snapshot for a catalog load, declarations, frontmatter assistance, and
diagnostics; refresh must reread manifests and replace the snapshot consistently. Thread
the registry through `macro_catalog/parsing.rs` (`resolve_rich` currently uses
`InputTypeRegistry::builtin()`), including markdown, workflow, config, and nested local
macro inputs. Preserve the editor's unresolved-input fallback.

Add registry-aware frontmatter schema/validation/hover functions in
`crates/sase_core/src/editor/frontmatter.rs`, retaining builtin-only wrapper APIs.
Replace builtin-only LSP frontmatter type completion in
`server/completion_items.rs`/`completion.rs` with the current registry projection.
Connect document diagnostics, hover, and refresh in the server rather than relying on
process-global mutable state. Existing enum argument candidates and diagnostic-driven
fixes must work automatically from the resolved hints. Invocation hover already derives
`plugin <distribution>` from `named_type`; ensure declaration hover also shows correct
plugin provenance and source, descriptions, and choices. Audit remaining direct
builtin-registry calls in these paths and retain only intentionally builtin defaults.

## 4. Expose the catalog and required-plugin findings

Add alphabetically placed `types` parsing to `src/sase/main/parser_macro.py`, dispatch
it from `macro_handler.py`, and keep rendering in a focused module such as
`src/sase/macro/cli_types.py`. Support exactly:

```text
sase macro types [NAME] [-j/--json]
```

Use clear help, the public option's short alias, and the existing Rich rendering
conventions. With no NAME, show grouped Scalar / Builtin / Plugin rows containing name,
kind, values or rule, and source. With NAME, resolve aliases canonically and show a
detail card with descriptions, rule, choices/labels/descriptions, and source file. `-j`
prints only the Rust catalog projection (or the requested canonical entry), with
errors/findings on stderr and a nonzero exit for an invalid requested name. Preserve the
existing `macro list` default behavior and add `types` to usage strings and relevant
completion/help expectations.

Extend `check_config_macro_input_types` in `src/sase/doctor/checks_config_macros.py`:

- Report registry file/type errors and warnings, even if no loaded macro uses the bad
  type, and retain path/line data in JSON. Do not duplicate findings per macro load.
- Retain missing-plugin/unknown-ID load issues, including skipped macros.
- For project-authored macro/workflow declarations in project `sase/macros/` and project
  config, warn when a referenced distribution is missing from that project's
  `plugins.required`; name the exact requirement to add and the declaration source.
  Cover canonical legacy project sources where existing discovery allows them.
- Use existing PEP 508 parsing and normalized distribution conventions from
  `src/sase/plugins/required.py` without broadening its existing fail-closed `use:`
  policy. An undeclared input-type dependency is this doctor's WARN, not a global
  catalog failure. Do not warn about home, bundled, or a plugin's own macros. Compare
  each project's declarations with its own config rather than an aggregate merged
  config; include references whose macros were skipped, not just successfully loaded
  inputs. Keep any new shared requirement-classification logic in Rust with thin Python
  inventory/rendering glue.

Add “Shipping input types” to `docs/plugins.md` with the manifest, package-root
location, qualified spelling, validation/isolation, and project requirement guidance.
Document `sase macro types` in `docs/cli.md` and the named plugin syntax/link in
`docs/macros.md`. Coordinate narrowly with the dogfood phase rather than rewriting its
broader material.

## 5. Verification and phase completion

Use temporary plugin packages, fake distribution-backed entry points, and deterministic
manifest fixtures; tests must not require live installed research plugins. Cover:

1. Rust loader schema version/unknown keys/malformed files, missing fields, duplicate
   IDs, distribution normalization, quote-it parity for unquoted `yes`, choice-object
   labels/descriptions, invalid word/null/duplicate values, and shorthand warnings. A
   bad type must not partially resolve, while a valid sibling/other plugin resolves.
2. Runtime missing plugin skips only the affected macro, unknown ID suggestions,
   shortform/longform file/config/workflow inputs, defaults, repeatables, authored
   choices override rejection, and identical binding from two macros sharing a type.
   Preserve builtin aliases and strict-flag compatibility regression coverage.
3. Discovery canonical versus legacy groups, real distribution versus module names, dual
   registration deduplication, disabled plugins, failed imports, and absent files.
   Exercise registry refresh after editing/removing a manifest and after discovery
   changes, including cached diagnostics appearing in a later doctor collection.
4. LSP JSON-RPC or existing server integration tests for plugin `type:` completion,
   declaration diagnostics/defaults, value completion text edits, invocation errors and
   suggested fixes, hover provenance, and refresh after manifest changes. Assert the
   resulting completion edit binds in the runtime and needs no materialized registry
   file. An unresolved type remains visible as a line input in the editor catalog.
5. CLI parser/help/table/detail and JSON modes, including `sase macro types`,
   `sase macro types effort`, a plugin name, builtin aliases, and unknown names. Parse
   JSON rather than checking formatting only. Doctor tests for malformed unused
   manifests, missing installation, undeclared project dependencies, declaration with
   PEP 508 spelling variations, and absence of warnings for home/plugin-owned sources.

Run targeted Rust tests through `just test -p <crate> <filter>` (never bare cargo) and
focused Python tests while implementing. Install the changed extension/LSP into this
workspace using the repository recipes before Python integration tests, so they test the
new bindings. Read each repository's current check instructions. Apply formatting/fix
recipes and run `sase tool run check` from both modified checkouts; targeted tests do
not replace either gate. Do not run `just check-full`, which has not been requested.

Use `/sase_monitor` before starting commands that need a detached long run, including
known-long installation and final checks. Prepare the follow-up to inspect results,
finish remaining verification, and close this same phase; wait until each handoff
command itself exits. Never end with an unmonitored command still running. If checks
fail, fix phase-caused failures. A failure reproduced identically on the clean base does
not keep this phase open: record evidence and any existing tracking task as a
`PROPOSED FOLLOW-UP:` note, then continue phase completion. No manual commits, branches,
PRs, version bumps, or release changelog edits.

Before closing, run `sase bead epic-symbols sase-1g4.5`. Planning found no entries, but
recheck the final tree. Resolve each remaining symbol or re-key its Justfile exemption
to the still-open parent/later phase; read `symvision.md` first if needed. Then close
only this assigned bead:

```bash
sase bead close sase-1g4.5 --note "<actual runtime/LSP/CLI/doctor coverage and both check results, including any clean-base failures>"
```

Do not close any ancestor, including a child plan's parent bead. Use `/sase_final` as
the last action for a normal completion, declaring the changes in both sase-core and
sase for host-owned commits. The host commits/pushes the linked core first and updates
`sase-core-revision.txt` before committing sase; this pin ordering is required for new
bindings and is documented in `docs/rust_backend.md`. Manually ratchet the pin only if
the needed core commit is already landed. Final completion evidence must identify the
binding tests, runtime/LSP parity checks, CLI/doctor results, and any proposed
follow-up.
