---
tier: epic
title: Rename sase modules from xprompt to macro
goal: "Outside the TUI, query-language status macros are shorthands, the xprompt package
  and sibling modules live on macro paths, identifiers follow a token-aware rename,
  external plugins still import the old paths through one temporary shim, and a
  terminology guard holds that boundary.

  "
phases:
  - id: shorthands
    title: Query-language shorthands
    depends_on: []
    size: medium
    description:
      "shorthands: Rename the query-language status-macro concept to shorthand,
      including the profile wire key sent to core, before any xprompt rename."
  - id: paths
    title: Package and module paths
    depends_on:
      - shorthands
    size: medium
    description:
      "paths: Move non-TUI xprompt packages and modules onto macro paths, retarget
      imports and package resource paths, and add the temporary import shim."
  - id: identifiers
    title: Token-aware identifier rename
    depends_on:
      - paths
    size: medium
    description:
      "identifiers: Rewrite xprompt identifiers outside the TUI with a token-aware
      codemod, including Rule 2 names, the catalog template attribute, and the shim's
      old-name bindings."
  - id: guard
    title: Terminology guard
    depends_on:
      - identifiers
    size: small
    description:
      "guard: Add the contract test that fails on non-TUI xprompt identifiers and paths,
      allowlisting the legacy homes and the shim."
proposed_by: bbugyi200.athena.sase-1eq.3
parent_bead: sase-1eq.3
create_time: 2026-10-02 19:30:19
status: wip
---

- **PROMPT:**
  [prompts/202610/sase_modules_rename.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/sase_modules_rename.md)
- **PARENT:**
  [202610/xprompts_to_macros.md](https://github.com/sase-org/sase--plans/blob/main/202610/xprompts_to_macros.md)

# Plan: Rename sase modules from xprompt to macro

This is the child epic for phase bead sase-1eq.3 (`sase-modules` in
`plan:202610/xprompts_to_macros.md`). Repo: sase only. Do not edit sase-core, docs,
memory, changelog, chezmoi, or plugin repos. Do not start the sunset flag, CLI
spellings, config keys, TUI module renames, or PNG goldens. Those belong to later phases
of the parent epic.

Each phase worker closes only its own phase bead. Record discovered work as
`PROPOSED FOLLOW-UP:` on that phase bead. Do not create beads. Do not close sase-1eq.3
or sase-1eq. Before closing, run `sase bead epic-symbols` on the phase bead and clear
any Justfile `--epic-symbol` entries it names. A `sase tool run check` failure that
reproduces on the clean base tree is a follow-up note, not a reason to leave the phase
open.

Read `lint_and_test.md` through `sase memory read` before finishing a sase change. Exit
every phase with `sase tool run check`. Never run `just check-full`. Run `just fmt`
after import and `__all__` edits. Keep the codemod as a re-runnable scratch script under
`/tmp`, run it after syncing to current master, and re-sync and re-run it instead of
hand-merging conflicts.

## Shared rules

The parent plan's vocabulary, identifier order, and compatibility policy bind this epic.
The parts that matter here:

- Replace case-aware, in order: `an xprompt` / `An xprompt`, then `XPROMPT`, `XPrompt`,
  `Xprompt`, then `xprompt`. Plurals and camelCase follow (`xprompts` to `macros`,
  `xpromptLsp` to `macroLsp`). Fix prose the replacement left ungrammatical.
- `macro` is a Rust keyword and a Jinja statement keyword. Do not create a bare Python
  or template variable named `macro`. The catalog field `entry.xprompt` becomes
  `entry.macro_def`. Jinja `{% macro %}`, `JinjaLocalKind` macro variants, the LSP
  semantic token `macro`, `macro_rules!`, and existing directive-shorthand modules
  (`xprompt/_directive_shorthand.py`, `xprompt/_parsing_shorthand.py`) stay.
- Scope is everything outside `src/sase/ace/tui/` and `tests/ace/`. Touch TUI files only
  for the shorthand phase's named files, to follow a renamed import or symbol, and for
  the Rule 2 function `get_raw_xprompt_content`. Do not `git mv` TUI paths or PNG
  goldens. Do not rewrite TUI copy, CSS, or keymap action strings.
- The codemod rewrites Python NAME tokens, import statements, comments, docstrings, test
  names, and dotted module-path strings (`mock.patch("sase.xprompt…")`, `importlib`
  targets, string-built module paths). It does not rewrite other string literals. CLI
  names, config and frontmatter keys, env vars, on-disk user directories
  (`sase/xprompts/`, `~/sase/xprompts/`), JSON keys and values, help text, and log and
  error messages belong to `sase-syntax`.
- Package resource paths that move with the directories are in scope. User discovery
  directories and the content-layout locator core still emits are not.
  `tools/validate_sase_core_rs` expects `package:xprompts/skills` because the pinned
  core still emits that locator. Leave that expected locator.
- Skip the codemod on `src/sase/legacy_xprompt_names.py`, on
  `src/sase/legacy_xprompt_syntax.py` if it exists, on every `LEGACY_*` and `legacy_*`
  identifier, on fixtures named `*legacy_xprompt*`, and on generated changelogs and
  `sase/memory/decisions/`. After the codemod, review the diff of
  `legacy_xprompt_names.py` and run its legacy-input tests.
- Do not move `sase/xprompts/reads.md` or `sase/xprompts/sync.md`. That repo directory
  move is `sase-syntax`. Do not move docs or memory pages.
- `pyproject.toml` has no xprompt package-data glob. Hatch ships `src/sase`. Re-check
  after the moves; do not add a glob unless a test proves the moved data files are
  missing from the wheel.

## Query-language shorthands

Rename the Patch-filter status macros (`%w` `%d` `%y` `%m` `%s` `%r`) to shorthands
before any xprompt rename. Afterwards, no `macro` identifier in this subsystem means
that concept. Jinja macros and the content-layout `macros` key in
`src/sase/core/content_layout_wire.py` are a different concept. Do not touch them. Do
not edit `docs/ace.md` or `docs/query_language.md`.

Rename identifiers and the field that holds them:

- `QueryMacroSpec` to `QueryShorthandSpec`
- `HOST_MACRO_TRIGGERS` to `HOST_SHORTHAND_TRIGGERS`
- schema and compiled-profile field `macros` to `shorthands`
- local variables and helpers whose names mean this concept (`macro_map`,
  `macro_tokens`, `macro_keys`, and the same idea in profile modules)
- completion kind `"macro"` to `"shorthand"` in `src/sase/ace/query/completion.py` and
  `src/sase/ace/tui/widgets/artifacts/patch_filter_bar.py`
- help copy `"Status macros"` to `"Status shorthands"` in
  `src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py`
- compiler error text that says "macro" for this concept ("macro trigger", "duplicate
  macro") to "shorthand"

Files: `src/sase/ace/query_profile/` (including profiles, registry, types, compiler, and
`__init__.py` exports), `src/sase/ace/query/profile_reference_boolean.py`,
`src/sase/ace/query/profile_highlighting.py`, the two TUI files above, and the tests
`tests/test_query_profile_*.py`, `tests/test_profile_highlighting.py`, and
`tests/ace/tui/test_patch_filter_bar.py`. Search for `QueryMacroSpec` and
`HOST_MACRO_TRIGGERS` and update every caller, including TUI readers of
`profile.macros`.

Wire key. `CompiledQueryProfile.to_wire()` is what `compile_corpus_with_profile` and
`compile_query_with_profile` receive. Pinned sase-core accepts the key `shorthands` as
an alias (`#[serde(default, alias = "shorthands")]` on the field still named `macros` in
`crates/sase_core/src/query/profile.rs`) and still hashes its canonical payload with the
key `macros`. Rust rejects a digest that does not match that canonical payload.

- Send `shorthands` and do not also send `macros`.
- Keep computing the digest from a payload whose list is under the key `macros`, so the
  digest still matches Rust. Store that same digest on the compiled profile.
- Do not change sase-core. A later core flip owns the canonical key.

Tests must show a compiled profile still round-trips through
`compile_query_with_profile` with the `shorthands` key and the unchanged digest. Update
tests that assert the wire dict's key name. Leave digest fixtures alone when the hashed
bytes did not change.

Do not rename Jinja token kind `"macro"` in `src/sase/xprompt/jinja_inspect.py`,
`workflow_validator_extract.py`, or `tests/xprompt/test_argument_surface_parity.py`.

## Package and module paths

Move paths with `git mv`. Enumerate with `git ls-files '*xprompt*'` and skip
`src/sase/ace/tui/`, `tests/ace/`, `docs/`, `sase/memory/`, `sase/xprompts/`,
changelogs, PNG goldens, and `src/sase/legacy_xprompt_names.py`.

Minimum moves:

- `src/sase/xprompt/` to `src/sase/macro/` (package, `catalog_template.html.j2`,
  `catalog_style.css`, and built-in loaders)
- `src/sase/xprompts/` to `src/sase/macros/` (built-in definitions, schema, `steps/`,
  `skills/`)
- `src/sase/default_xprompts/` to `src/sase/default_macros/`
- `src/sase/xprompt_links.py` if present, else only the modules that exist
- `src/sase/config/xprompt_sources.py`
- `src/sase/core/xprompt_skill_definition_facade.py`
- `src/sase/integrations/xprompt_lsp.py`
- `src/sase/history/vcs_xprompt_mru.py`
- `src/sase/bead/xprompts.py`
- `src/sase/agent/xprompt_swarm.py`, `_xprompt_swarm_parsing.py`,
  `_xprompt_swarm_rendering.py`, `multi_agent_xprompt.py`, `multi_prompt_xprompts.py`
- `src/sase/doctor/checks_config_xprompts.py`, `checks_deep_xprompt_lsp.py`
- `src/sase/main/parser_xprompt.py`, `xprompt_handler.py`
- `tests/xprompt/` to `tests/macro/`
- `tests/_xprompt_*_helpers.py`, `tests/test_xprompt_*.py`, and every other test path
  outside `tests/ace/` whose path contains the word

New filenames follow the case order (`xprompt_swarm.py` to `macro_swarm.py`,
`xprompts.py` to `macros.py`). Do not rename identifiers inside files in this phase.
Classes such as `XPromptValidationError` stay until the next phase.
`legacy_xprompt_names.py` stays put.

Update imports and resource paths to the new modules:

- `from sase.xprompt…` / `import sase.xprompt…` become `sase.macro`
- `sase.agent.xprompt_swarm` becomes the new swarm module name
- string module paths: `mock.patch("sase.xprompt…")`, `importlib` targets, and
  `importlib.resources` paths such as
  `_SASE_PACKAGE_SKILLS_RESOURCE = ("xprompts", "skills")` and the matching
  `default_xprompts` tuples in `loader_sources.py` and `integrations/xprompt_lsp.py`
  (now under its new path)
- filesystem path prefixes in `patch_stitch_audit.py`,
  `completion/candidates/catalog_prompts.py`, and the `sase skill init` dirty-tree guard
  (`main/_init_skills_source_integrity.py` and its tests)
- the warning string that names `importlib.resources('sase/xprompts/skills')`, because
  it is the moved resource path

Leave `_PLUGIN_XPROMPT_DESTINATION` and any prose about a plugin's packaged `xprompts/`
directory. Plugins still ship that directory. Leave the `package:xprompts/skills`
locator assertion in `tools/validate_sase_core_rs`. Leave user-facing log and help
strings that are not resource paths.

TUI files may change only the import module path (`sase.xprompt` to `sase.macro`, and
the swarm module path). Do not rename TUI symbols yet.

In-repo call sites must import the new paths. The shim is only for the editable installs
of sase-telegram, sase-research-artifacts, and bugyi-chops, which keep importing
`sase.xprompt.*` until their own phases. Do not edit those repos. Open `sase-github`
with `sase repo open` and, if it imports `sase.xprompt` or `sase.agent.xprompt_swarm`,
extend the shim to those names. Do not edit sase-github.

Temporary shim, one new package and no other alias modules:

- Add `src/sase/xprompt/__init__.py` after the directory move. Mark every alias
  `# TEMP(xprompt->macro shim): removed in audit-deploy`.
- Register successor submodules in `sys.modules` under the old paths using synthetic
  module objects created in that file, not by aliasing the real module object. Sharing
  the real module would put old names on `sase.macro` and would make
  `mock.patch("sase.xprompt…")` patch the new module. Submodules those repos import
  today: `models`, `directives`, `workflow_validator_extract`, `catalog`,
  `loader_sources`, `processor`, `loader_parsing`, `runtime_context`,
  `workflow_executor_utils`.
- Re-export these names, still under their current spellings in this phase (the
  identifier phase rebinds them to the renamed objects): `InputType`,
  `XPromptValidationError`, `replace_ref_in_vcs_tag`, `extract_vcs_workflow_tag`,
  `extract_xprompt_calls`, `process_xprompt_references`, `extract_prompt_directives`,
  `plan_prompt_fanout_variants`, `NoXpromptsFound`, `PdfEngineUnavailable`,
  `build_xprompts_catalog`, `CatalogArtifact`, `CatalogStats`,
  `load_xprompts_from_plugins`, `expand_single_xprompt`, `parse_yaml_front_matter`,
  `UNSET`, `bind_runtime_template_vars`, `render_template`, `render_toplevel_jinja2`,
  `expand_xprompt_swarms_with_metadata`, `list_patch_xprompt_tags`.
  `render_toplevel_jinja2` is required: sase-research-artifacts imports it from
  `sase.xprompt`. The parent plan's list omitted it.
- `from sase.agent.xprompt_swarm import …` does not import `sase.xprompt` first, and
  `sase.agent.__getattr__` does not serve submodules. From the shim, install a
  `MetaPathFinder` that resolves only `sase.agent.xprompt_swarm` to the renamed swarm
  module, lazily, on `find_spec`. Register that finder from the end of
  `src/sase/agent/__init__.py` by importing the shim, on a line that carries the same
  TEMP marker. Do not import the swarm module while `sase.agent` is still initializing.
  Do not add `sase/agent/xprompt_swarm.py`.

Prove the shim in a fresh interpreter: `from sase.xprompt.models import InputType`,
`from sase.xprompt import render_toplevel_jinja2`,
`from sase.xprompt.catalog import build_xprompts_catalog`, and
`from sase.agent.xprompt_swarm import expand_xprompt_swarms_with_metadata`, without
importing `sase.xprompt` before the swarm import.

Refresh `tests/shard_timings.json` and `tests/reproducible_flake_baseline.txt` for test
paths this phase moved. Leave `tests/ace/` node ids alone.

## Token-aware identifier rename

Rewrite identifiers in the moved tree and in every non-TUI caller. Use Python `tokenize`
or LibCST. Do not use `sed` or a raw substring replace. Keep the script re-runnable.

The script rewrites NAME tokens, imports, comments, docstrings, and test names that
contain `xprompt`, plus dotted module-path strings. It skips the homes and fixtures in
Shared rules. It does not rewrite other string literals. Run it only outside
`src/sase/ace/tui/` and `tests/ace/`, then update TUI files in a separate pass that
changes only:

- imports and attribute or call references to symbols this phase renamed
- the Rule 2 function `get_raw_xprompt_content`, which is defined in
  `src/sase/ace/tui/models/artifact_files.py` and `models/agent.py`

Do not rename TUI-local definitions (`focus_xprompt`, browser modules, CSS ids, copy,
keymap strings).

Rule 2 names, checked against the tree:

- `get_raw_xprompt_content` becomes `get_raw_prompt_content`. The function already reads
  through `resolve_raw_prompt_path`. Rename the function and its docstring. Do not
  change `LEGACY_RAW_XPROMPT_FILENAME` or the legacy reader.
- Attribute `submitted_xprompt` on workflow objects and function parameters becomes
  `submitted_prompt`. There is no persisted JSON key `"submitted_xprompt"`. The filename
  string lives only as `LEGACY_SUBMITTED_XPROMPT_FILENAME`. Do not change that constant.
  Tests that build a legacy artifact file keep the legacy filename.
- `XPROMPT_PROC_ORIGIN` in `src/sase/procs/models/common.py` is the legacy value
  `"xprompt-proc"`. `legacy_xprompt_names.py` already defines
  `PROMPT_PROC_ORIGIN = "prompt-proc"` and
  `LEGACY_XPROMPT_PROC_ORIGIN = "xprompt-proc"`. Do not let the codemod emit
  `PROMPT_PROC_ORIGIN = "xprompt-proc"`. Retarget callers to the legacy-names constants
  and remove the duplicate procs constant when nothing else needs it. Skip
  `legacy_xprompt_names.py` entirely, then review its diff is empty.

Catalog template. The dataclass field `xprompt: XPrompt` in `_catalog_models.py` is what
`entry.xprompt` reads. Rename that field to `macro_def` and update
`catalog_template.html.j2` (`entry.xprompt`, `{% set xp = entry.xprompt %}`). Do not
name the template variable `macro`. Leave the visible strings ("xprompts Catalog", "No
project-local xprompts") and the CSS comments. They are `sase-syntax` prose.

Shim. Rebind each old exported name to the renamed object
(`build_xprompts_catalog = build_macros_catalog`, and the same for the rest of the
list). Old names stay only inside `src/sase/xprompt/__init__.py` and the one marked line
in `sase/agent/__init__.py`.

`__init__.py` re-exports and `__all__` lists in the moved package take the new names.
`just fmt` re-sorts imports. Re-sort `__all__` and `keep-sorted` blocks whose entries
moved. Do not re-sort CLI subcommand lists.

Refresh shard timings, the flake baseline, and Symvision whitelist entries or
`# symvision:` pragmas that name a renamed symbol or path. Read `symvision.md` through
`sase memory read` before editing a whitelist. Do not rename `tests/ace/` timing keys.

Review the diff of every legacy constant. Run the legacy-input tests for
`legacy_xprompt_names`. Then run `sase tool run check`.

## Terminology guard

Add `tests/test_macro_terminology.py`, marked `pytest.mark.contract`, modeled on
`tests/test_sase_turn_terminology.py`.

It fails when any of these, outside `src/sase/ace/tui/` and `tests/ace/`, contains
`xprompt` case-insensitively:

- a path component under `src/` or `tests/`
- a Python NAME token
- an `Import` or `ImportFrom` module string

It does not fail on other string literals. User-facing xprompt strings are still
supposed to exist until `sase-syntax`.

Allowlist, commented with why:

- `src/sase/legacy_xprompt_names.py`
- `src/sase/legacy_xprompt_syntax.py` when that file exists
- `src/sase/xprompt/__init__.py` (the shim)
- fixtures whose paths contain `legacy_xprompt`
- a source line that carries `# TEMP(xprompt->macro shim)` (the agent package
  registration line)

Do not allowlist a whole package to hide a missed rename. Fix in-scope stragglers. If a
hit is a user-facing string the guard correctly ignores, leave it. If the guard cannot
express a real survivor except by allowlisting a file that also contains live concept
names, record a `PROPOSED FOLLOW-UP:` instead of widening the allowlist past the homes
above.

Widen nothing else. Later parent-epic phases extend this test.

Exit with `sase tool run check`.

## Land

The child epic's land agent closes this epic's bead, then closes parent phase
sase-1eq.3. Dependents of sase-1eq.3 stay parked until that phase bead closes. The land
agent does not close sase-1eq. Phase workers do not close either ancestor. Before
closing sase-1eq.3, the land agent runs `sase bead epic-symbols sase-1eq.3` and
`sase tool run check` if the guard phase's check is not already current. Follow-up notes
on the phase beads are triaged by that land agent into task beads; phase workers only
write the notes.
