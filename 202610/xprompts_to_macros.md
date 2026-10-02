---
tier: epic
title: Rename xprompts to macros
goal: 'SASE calls its reusable `#name` prompt definitions "macros" in code, CLI, config,
  directories, TUI, LSP, plugins, docs, memory, skills, and chezmoi. Agent artifacts,
  proc rows, state files, and stored prompts written before the rename still load.
  Retired xprompt spellings keep working behind one sunset flag until callers migrate.

  '
phases:
  - id: core-expand
    title: sase-core additive macro rename
    depends_on: []
    size: large
    description:
      "core-expand: rename the concept inside sase-core while serialized output stays
      byte-identical. Accept every new input spelling, register new binding names next
      to the legacy ones, and add a sase-macro-lsp binary to the existing LSP crate."
  - id: sase-durable
    title: Durable data, core wires, and LSP build tooling
    depends_on:
      - core-expand
    size: medium
    description:
      "sase-durable: pin core-expand and call the new bindings. Route every durable
      sase-written name through permanent legacy readers that have legacy-input tests.
      Make the LSP build and launch tooling work with either crate name."
  - id: sase-modules
    title: Module, package, and identifier rename outside the TUI
    depends_on:
      - sase-durable
    size: large
    description:
      "sase-modules: rename query-language macros to shorthands first. Then move the
      sase.xprompt package and every other xprompt-named module, data dir, and test
      path. Rewrite identifiers with a token-aware codemod, add a temporary plugin
      import shim, and start the terminology guard."
  - id: sase-syntax
    title: User syntax, CLI, config, discovery, and the sunset flag
    depends_on:
      - sase-modules
    size: large
    description:
      "sase-syntax: create the legacy_xprompt_syntax sunset flag. Switch every
      user-facing string contract outside the TUI to macro spellings with flag-gated
      aliases: CLI, config and frontmatter keys, directories, plugin groups, env vars,
      doctor ids, and skill sources."
  - id: tui
    title: TUI macro surfaces and goldens
    depends_on:
      - sase-syntax
    size: large
    description:
      "tui: rename TUI modules, identifiers, CSS, copy, keymap actions, and Admin Center
      ids. Apply the raw-prompt wording for the agent prompt tab and headings, and
      re-baseline the PNG goldens whose pixels change."
  - id: docs-memory
    title: Documentation, site redirect, memory, and first skill redeploy
    depends_on:
      - sase-syntax
    size: medium
    description:
      "docs-memory: redeploy the generated skills and rewrite docs, README, and blog.
      Move the docs page with a redirect from the old URL. Rename the xprompts memory
      note and the five glossary strands, then republish memory."
  - id: telegram
    title: sase-telegram cutover
    depends_on:
      - sase-syntax
    size: small
    description:
      "telegram: switch sase-telegram to a new-first compatibility import helper and the
      raw_prompt.md artifact. Rename the bot command to /macros, keeping a flag-gated
      /xprompts alias, and rename its internals and docs."
  - id: plugins
    title: sase-github, sase-research-artifacts, and bugyi-chops cutover
    depends_on:
      - sase-syntax
    size: small
    description:
      "plugins: register the sase_macros entry-point group next to the legacy group,
      keep the packaged xprompts/ directories for now, and use new-first imports in
      tests. Rename docs and internals in all three plugins."
  - id: nvim
    title: sase-nvim cutover
    depends_on:
      - sase-syntax
    size: medium
    description:
      "nvim: rename sase-nvim's Lua modules, setup keys, commands, highlight groups,
      Telescope extension, and LSP client name, with deprecation shims. Talk to sase
      macro and sase-macro-lsp first, falling back to the legacy CLI and binary names."
  - id: core-flip
    title: sase-core contract flip with same-turn pin bump
    depends_on:
      - tui
      - telegram
      - plugins
      - nvim
    size: large
    description:
      "core-flip: make sase-core emit only macro spellings and remove the legacy binding
      names. Rename the LSP crate, bump the index and wire schemas, and add the mobile
      macros route. In the same declared turn, update sase mirrors and pin, and the
      chezmoi LSP install script."
  - id: audit-deploy
    title: Cross-repo audit, guardrail, chezmoi, and machine migration
    depends_on:
      - docs-memory
      - core-flip
    size: medium
    description:
      "audit-deploy: remove the temporary import shim and widen the guard to the whole
      repo. Sweep every repo, migrate the chezmoi sources and the live athena state,
      redeploy skills, annotate open beads, and record deferred follow-ups."
proposed_by: bbugyi200.athena.0v4
create_time: 2026-10-02 06:51:18
status: wip
---

# Plan: Rename xprompts to macros

## Context

An **xprompt** is SASE's reusable prompt definition. It is a `.md` prompt part or a
`.yml` workflow, invoked as `#name`, `#name(args)`, or `#name:arg`. Definitions come
from `sase/xprompts/` directories, `xprompts:` config maps, plugins, and package
built-ins. The user wants the concept renamed to **macro** in this repo, in every linked
repo, in every enabled SASE project on this machine, and in chezmoi.

Scale (case-insensitive `xprompt`, tracked files):

- **sase:** about 22,500 occurrences in about 1,790 files.
  - `src/`: 8,775 lines, of which 4,411 are in `src/sase/ace` and 2,164 in the 134-file
    `sase.xprompt` package.
  - Tests: 9,419 lines in 906 files.
  - Docs, README, and CHANGELOG: about 1,344 lines.
  - 465 paths contain the word, including 34 PNG goldens. 337 source and 433 test files
    import `sase.xprompt`, and about 707 strings are `"sase.xprompt.…"` patch targets.
- **sase-core:** about 3,060 occurrences in 174 files. The editor has 836, the
  `xprompt_catalog` module 613, the `sase_xprompt_lsp` crate 449, and the gateway 92. 55
  paths contain the word.
- **sase-nvim:** 526 occurrences, including public setup keys, commands, highlight
  groups, and the LSP client name.
- **sase-telegram:** 155. Its `src/` imports `sase.xprompt.*` at runtime, and it has a
  `/xprompts` bot command.
- **sase-research-artifacts** (93) and **sase-github** (35): the `sase_xprompts`
  entry-point group and packaged `xprompts/` directories.
- **chezmoi:** 115 occurrences, including:
  - `home/sase/xprompts/`, which deploys to `~/sase/xprompts/`
  - the `xprompts:` key in `home/dot_config/sase/sase.yml`
  - the LSP install script
  - 28 generated skill copies
- **bugyi-chops**, an editable sase plugin installed in the uv tool env: one test import
  of `sase.xprompt.directives` and one README line.
- **bob-cli:** 21 occurrences, all in one test-fixture document name.
- **actstat:** none.

Precedents. This plan reuses the expand/contract, legacy-reader, and sunset-flag
machinery of `plan:202609/sase_turn_rename.md` (epic sase-1ab) and
`plan:202609/agent_session_rename.md` (sase-17m).
`plan:202609/sase_turn_rename_finish.md` records what went wrong in sase-1ab, and this
plan guards against each failure:

1. **The contract flip was verified but never committed.** Here, `core-flip` is one
   cross-repo turn whose declaration covers sase-core, sase, and chezmoi, so the host
   commits core and moves `sase-core-revision.txt` in the same turn.
2. **Mechanical renames corrupted legacy readers.** For example,
   `"proc-shell" → "named-proc"` became `"named-proc" → "named-proc"`. Here,
   `sase-durable` lands the legacy readers and their legacy-input tests first, in named
   homes that later codemods skip.
3. **Stale assertions and prose stragglers survived.** Here, each cutover phase widens
   one terminology guard test over its own scope. Stragglers, and concurrent commits
   that reintroduce the word, then fail CI.

## Shared Policy (binds every phase)

### Vocabulary

**Rule 1: the reusable definition.** This is the default for every hit.

| Old                                                                                                                | New                                                                                             |
| ------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| xprompt / xprompts, XPrompt / XPrompts / Xprompt, XPROMPT                                                          | macro / macros, Macro / Macros, MACRO                                                           |
| an xprompt / An xprompt                                                                                            | a macro / A macro                                                                               |
| xprompt part, swarm, workflow, memory, skill; local / project-local / VCS xprompt                                  | macro part, swarm, workflow, memory, skill; local / project-local / VCS macro                   |
| mini-xprompt pane; xprompt catalog, `xprompt-catalog`; xprompt LSP                                                 | mini-macro pane; macro catalog, `macro-catalog`; macro LSP                                      |
| `sase xprompt <sub>`; `sase path xprompts-dir`, `xprompts-schema`, `xprompts-collection-schema`                    | `sase macro <sub>`; `sase path macros-dir`, `macros-schema`, `macros-collection-schema`         |
| `%xprompts_enabled`                                                                                                | `%macros_enabled`                                                                               |
| `sase/xprompts/`, `~/sase/xprompts/`, `~/sase/xprompts/<project>/`                                                 | `sase/macros/`, `~/sase/macros/`, `~/sase/macros/<project>/`                                    |
| package `sase.xprompt`, data dirs `src/sase/xprompts/` and `src/sase/default_xprompts/`                            | `sase.macro`, `src/sase/macros/`, `src/sase/default_macros/`                                    |
| config `xprompts:`, `xprompt_aliases:`, `auto_xprompt_menu`, `xprompt_placeholder_args`, `mentors[].xprompt`       | `macros:`, `macro_aliases:`, `auto_macro_menu`, `macro_placeholder_args`, `mentors[].macro`     |
| frontmatter `xprompts:` (local helpers)                                                                            | `macros:`                                                                                       |
| keymap actions `focus_xprompt`, `clear_xprompt_focus`, `start_last_vcs_xprompt_in_editor`                          | `focus_macro`, `clear_macro_focus`, `start_last_vcs_macro_in_editor`                            |
| entry-point group `sase_xprompts`; env `SASE_XPROMPT_*`, `SASE_*_XPROMPTS`                                         | `sase_macros`; `SASE_MACRO_*`, `SASE_*_MACROS`                                                  |
| `sase-xprompt-lsp`, crate `sase_xprompt_lsp`, LSP commands `sase.xpromptLsp.*`                                     | `sase-macro-lsp`, `sase_macro_lsp`, `sase.macroLsp.*`                                           |
| `~/.sase/vcs_xprompt_mru.json`, `~/.sase/xprompt_save_state.json` (key `xprompt`), `~/.sase/xprompt_lsp/`          | `vcs_macro_mru.json`, `macro_save_state.json` (key `macro`), `macro_lsp/`                       |
| agent artifacts `xprompts.json`, `xprompts_<step>.json`                                                            | `macros.json`, `macros_<step>.json`                                                             |
| doctor ids `config.model_xprompts`, `config.xprompt_definitions`, `config.xprompt_directives`, `tools.xprompt_lsp` | `config.model_macros`, `config.macro_definitions`, `config.macro_directives`, `tools.macro_lsp` |
| telegram `/xprompts`; mobile route `/api/v1/xprompts/catalog`                                                      | `/macros`; `/api/v1/macros/catalog`                                                             |
| glossary Xprompt, Xprompt Part / Swarm / Workflow / Memory; alias "xprompt project tag"                            | Macro, Macro Part / Swarm / Workflow / Memory; "macro project tag"                              |

**Rule 2: the authored prompt as a whole.** A few names use "xprompt" for the whole
prompt as the user typed it, before macro expansion, rather than for a definition. These
become "prompt", never "macro":

- Artifact files: `raw_xprompt.md` → `raw_prompt.md`, and `submitted_xprompt.md` →
  `submitted_prompt.md`.
- Attributes: `submitted_xprompt` → `submitted_prompt`, and `get_raw_xprompt_content` →
  `get_raw_prompt_content`.
- TUI copy: the agent preview tab `XPROMPT` and the `AGENT XPROMPT` heading become
  `RAW PROMPT` and `AGENT RAW PROMPT`. The existing `PROMPT` heading already shows the
  expanded prompt.
- The `%proc` origin family: `xprompt-proc` → `prompt-proc`, `xprompt_proc` →
  `prompt_proc`, `XpromptProcMetaWire` → `PromptProcMetaWire`, `XPROMPT_PROC_ORIGIN` →
  `PROMPT_PROC_ORIGIN`, and the binding `xprompt_proc_origin` → `prompt_proc_origin`. A
  `%proc` unit is a directive in a prompt, not a macro.

Apply Rule 2 only to these names and to unmistakable occurrences of the same meaning.
Before renaming any other name under Rule 2, check it for collisions and record it on
your phase bead.

### Existing "macro" meanings

- **Query-language "status macros"** are the `%w`/`%d`/`%y`/`%m`/`%s`/`%r` Patch-filter
  shorthands: `QueryMacroSpec`, `HOST_MACRO_TRIGGERS`, the profile `macros` key, and
  completion kind `macro`. They collide with the new concept on both the word and the
  `%` sigil, so they become **shorthands** (`QueryShorthandSpec`, profile key
  `shorthands`, completion kind `shorthand`, "Status shorthands"). That rename happens
  before any xprompt rename in the same scope.
- **Jinja macros** stay unchanged in name. Macro bodies are Jinja templates, so
  `{% macro %}`/`endmacro` keyword lists,
  `JinjaLocalKind::{MacroName, MacroParam, MacroSpecial}`, and Jinja scope handling keep
  their names. User-facing copy says "Jinja macro" wherever it means the Jinja feature,
  for example "Define a reusable Jinja macro."
- **The LSP-standard semantic token type `macro`** (`MACRO_TOKEN_TYPE`) keeps coloring
  `%directive` names, and macro references keep `function`. These are LSP protocol
  vocabulary that editor themes depend on, so do not reorder the legend.
- **Rust and tooling uses are unrelated:** `macro_rules!`, `proc-macro2`, tokio's
  `macros` feature, and font binaries.

### Identifier and Mechanical-Rename Rules

- Replace case-aware and in this order:
  1. the article forms (`an xprompt` → `a macro`)
  2. `XPROMPT` → `MACRO`
  3. `XPrompt` → `Macro`
  4. `Xprompt` → `Macro`
  5. `xprompt` → `macro`

  Plurals and camelCase follow from this order: `xprompts` → `macros`, `xpromptLsp` →
  `macroLsp`, `nxprompts` → `nmacros`. Then fix any prose the replacement left
  ungrammatical.

- **`macro` is a reserved keyword in Rust.** Never introduce a bare `macro` identifier,
  field, variable, or module there. Use a descriptive name such as `macro_def`,
  `macro_ref`, or `catalog_macro`. A JSON field that must serialize as `macro` uses
  `#[serde(rename = "macro")]` on a differently named field. `macros` is legal.
- **`macro` is a Jinja statement keyword.** Do not expose a template variable named
  `macro`. For example, `entry.xprompt` in `catalog_template.html.j2` becomes
  `entry.macro_def`.
- **`macro` sorts differently than `xprompt`.** Re-sort CLI subcommand and option lists
  (`cli_rules.md`), `keep-sorted` blocks, `__all__` lists, and schema properties. Run
  each repo's formatter, since import order and Markdown line wraps change.
- Move paths with `git mv`. Rename PNG goldens by regenerating them, never by blind
  replacement of file contents.
- Codemods must skip:
  - the legacy homes: `src/sase/legacy_xprompt_names.py`,
    `src/sase/legacy_xprompt_syntax.py`, and Rust constants commented
    `// legacy xprompt spelling`
  - every `LEGACY_*` and `legacy_*` identifier
  - fixtures named `*legacy_xprompt*`
  - the history listed below
- After each codemod, review the diff of every legacy constant and run the legacy-input
  tests.
- "xprompt" has no unrelated meaning. When a phase finishes, every surviving
  case-insensitive hit in its scope must be allowlisted and classified.
- Concurrent work lands on sase master constantly (about 30 claims). Keep each codemod
  as a re-runnable scratch script, and run it right after syncing to the latest master.
  If landing conflicts, re-sync and re-run the script instead of hand-merging.

### Deliberately Unchanged History

- **Generated CHANGELOGs** in every repo (release-please in sase, release-plz in
  sase-core, and the plugin CHANGELOGs). Breaking-change footers produce the new
  entries.
- **Accepted decision records** under `sase/memory/decisions/`. The one exception:
  `hold-pull-fail-open.md` gets its link target `[[xprompts.md]]` → `[[macros.md]]` and
  its path `docs/xprompt.md` → `docs/macros.md`. This is link maintenance, not a change
  of course. Its claim and body stay as they are.
- **Archives and history:** sidecar archives (plans, beads, research, agent prompts and
  pages), `sdd/` archived tales, linked repos' `.beads/issues.jsonl`, and git history.
- **Machine-local history:** `~/.sase/glossary_read_reports/`,
  `~/.sase/file_reference_history.json`, and existing agent artifact directories. Legacy
  readers read the artifact directories, and nothing rewrites them.

### Compatibility Policy

#### 1. Durable data

Durable data gets permanent, unconditional readers that are never flag-gated:

- Readers prefer the new name and fall back to the legacy one. Writers emit only the new
  name.
- Python legacy names live only in `src/sase/legacy_xprompt_names.py`. Rust legacy names
  are constants commented `// legacy xprompt spelling`, beside their domain, matching
  the existing `shell_name` precedent.

Durable surfaces:

- **Agent artifacts:** `raw_prompt.md`, `submitted_prompt.md`, `macros.json`, and
  `macros_<step>.json`. Readers include the Rust scanner and alias history, revival,
  restart recovery, chat-from-name, the mobile agent state, name migration, agents_sync
  publication, and telegram's retry fallback.
- **`~/.sase/vcs_macro_mru.json` and `~/.sase/macro_save_state.json` (key `macro`):**
  read the new file, fall back to the legacy file, write only the new file, and delete
  the legacy file after the first successful write.
- **`.sase-skills-manifest.json`:** the key `macro_set_sha256` (legacy
  `xprompt_set_sha256`). A legacy manifest must still satisfy the deploy provenance
  guard.
- **`procs.jsonl`:** the `prompt_proc` field and origin `prompt-proc`. The legacy
  `xprompt_proc` and `xprompt-proc` read as equal.
- **`%xprompts_enabled:false … :true` regions** are stored in agent prompts, chat-fork
  prompts, follow-up records, and SDD files. `%xprompts_enabled` therefore stays a
  permanent, unconditional alias of `%macros_enabled`. Writers and docs use only the new
  spelling.
- **TUI resume state:** Admin Center sub-tab and Statistics view ids `"xprompts"` read
  as `"macros"`.
- **Rebuildable state:**
  - The agent-scan index column `xprompts_sig` and `record_json.used_xprompts` are
    renamed in `core-flip`. The schema bump goes through the existing migration path,
    and never moves an archive-sized rebuild onto TUI startup or the UI thread.
  - `~/.sase/xprompt_lsp/` and the managed tmp `xprompts_catalog` directory get new
    names. The old names are disposable residue.
- **Verify, don't assume:** the artifact-ref target kind `xprompt_skill`,
  `WorkflowType.SIMPLE_XPROMPT` (`"simple_xprompt"`), `submitted_xprompt` keys, and any
  other value found to be stored. Add a legacy reader only where a value is actually
  persisted.

#### 2. User-authored and client-facing spellings

One `sunset` flag, `legacy_xprompt_syntax`, keeps the old spellings as silent aliases
while callers migrate.

- All Python aliases live in `src/sase/legacy_xprompt_syntax.py`, modeled on
  `src/sase/agent/legacy_sase_shell_syntax.py`. Errors read "`<old>` is retired; use
  `<new>`".
- An input that supplies both the old and new spelling is an error.
- Help, completion, examples, and output never show a legacy spelling.
- Rust consumers receive the flag as an `accept_legacy_xprompt_names` option (default
  `true`) on catalog-loading requests. The LSP receives it as an initializationOption,
  the way `typed_launch_units` does.
- A new `sase doctor` check, `config.retired_xprompt_names`, lists every legacy spelling
  it finds, with its macro replacement, in either flag state. It covers directories,
  config and frontmatter keys, keymap actions, env vars, and plugin groups and
  directories.

Create the flag only with
`sase flag new legacy_xprompt_syntax -k sunset --when-enabled … --when-disabled … --remove-when …`,
using these sentences (`@<path>` files are fine):

- **When enabled:** SASE silently accepts the retired xprompt spellings as aliases of
  their macro replacements:
  - the `sase xprompt` command, the `sase path xprompts-*` targets, and the
    `xprompt-catalog` helper bridges
  - the `xprompts`, `xprompt_aliases`, `auto_xprompt_menu`, `xprompt_placeholder_args`,
    and `mentors[].xprompt` config keys
  - the `focus_xprompt`, `clear_xprompt_focus`, and `start_last_vcs_xprompt_in_editor`
    keymap actions
  - the `xprompts:` frontmatter key
  - xprompt-named definition directories: `sase/xprompts/`, `.xprompts/`, `xprompts/`,
    `~/sase/xprompts/`, `~/.xprompts/`, `~/xprompts/`, and
    `~/.config/sase/xprompts/<project>/`
  - the `sase_xprompts` plugin entry-point group and packaged `xprompts/` plugin
    directories
  - the `SASE_XPROMPT_LSP_CMD` and `SASE_DISABLE_PLUGIN_XPROMPTS` environment variables
  - a `sase-xprompt-lsp` binary on PATH
- **When disabled:** SASE rejects those spellings with an error naming the macro
  replacement. It skips xprompt-named directories and plugin directories, which
  `sase doctor` reports. It still loads agent artifacts, proc rows, MRU and save-state
  files, skills manifests, and `%xprompts_enabled` regions written before the rename.
- **Remove when:** no maintained prompt, macro, skill, config, script, or plugin in the
  sase-org repos, bugyi-chops, or chezmoi uses a retired xprompt spelling; every
  maintained plugin ships a packaged `macros/` directory under `sase_macros` with a sase
  floor that reads it; and a sase release with macro syntax has shipped.

#### 3. Two-step Rust contract (expand/contract)

Every sase workspace builds `sase_core_rs` and the LSP from the linked sase-core
checkout, so one breaking core commit would break every concurrent sase workspace.

- `core-expand` is additive and keeps serialized output byte-identical.
- `core-flip` flips the output only after every sase, telegram, plugin, and nvim reader
  accepts both spellings, and only after current sase master passes against the new
  core.
- No Python code may hard-code a core schema literal. Compare against the imported
  mirror constants.

#### 4. Mobile API v1

`/api/v1/macros/catalog` becomes the canonical route. `/api/v1/xprompts/catalog` stays
inside v1 as a deprecated alias that serves the same payload, because mobile clients are
external. Remove it only in a future API version. The API version governs this alias,
not the flag.

#### 5. Packaged plugin directories

Plugins register `sase_macros` next to `sase_xprompts`, but they keep shipping their
packaged `xprompts/` directory. Their sase floors (`sase>=0.17.0`) and the
published-floor smoke jobs cannot require a sase release that does not exist yet.

Moving each directory to `macros/`, dropping the legacy group, and raising the floor
waits for a published sase release that reads `sase_macros` and `macros/`. That step is
part of the flag's removal gate, and `audit-deploy` records it as a follow-up.

#### 6. Breaking surfaces and versions

Phases that change CLI JSON values, flags, config keys, env vars, keymap actions,
diagnostic codes, the entry-point group, binary names, or wire spellings use a `feat!:`
subject with a `BREAKING CHANGE:` footer that lists them.

`core-flip` bumps every versioned wire or persisted schema whose emitted shape changes.
Never edit a `version` field or CHANGELOG by hand.

#### 7. Process

- Open every repo other than your workspace with `/sase_repo` (`sase repo open <name>`).
  bugyi-chops is `gh:bbugyi200/bugyi-chops`, and bob-cli is the `bob-cli` project. Read
  each repo's `AGENTS.md` (or `CLAUDE.md`) before editing.
- Verify with `sase tool run check` in every repo you changed, or that repo's documented
  check. Run `just install` first in a fresh sase workspace. Never run
  `just check-full`.
- Read these notes through `/sase_memory_read` before the matching work:
  - `lint_and_test.md` before finishing any sase change
  - `tui.md` before TUI work
  - `generated_skills.md` before skill work
  - `cli_rules.md` before CLI changes
  - `sase_flags.md` before creating the flag
  - `symvision.md` for Symvision failures
  - `xprompts.md` (renamed `macros.md` by `docs-memory`)
- Phase workers record `PROPOSED FOLLOW-UP:` notes on their own phase bead instead of
  creating beads.
- A phase's final declaration covers every repo that phase changed. A linked-repo change
  that was verified but not declared counts as a failed phase.

## sase-core additive macro rename

Repo: sase-core. This is a non-breaking `feat:` change. After it lands, a sase tree on
the previous pin must still pass.

**Rename internals** per the identifier rules:

- Modules:
  - `xprompt_catalog/` → `macro_catalog/`
  - `xprompt_text_block.rs` → `macro_text_block.rs`
  - `editor/xprompt_args.rs` → `editor/macro_args.rs`
  - `agent_stats/run/xprompts.rs` and `run/tests/xprompts.rs` → `macros.rs`
- Types and functions, for example `CatalogXprompt`, `XpromptSourceWire`,
  `MemoryXprompt*Wire`, `UsedXPromptWire`, `XpromptArgumentSource`, `XpromptLspServer`,
  and `XpromptProcMetaWire` (→ `PromptProcMetaWire`, Rule 2).
- **Exception:** types whose names appear in a golden contract under
  `crates/sase_gateway/contracts/` wait for `core-flip`.
- Remove the xprompt-named entries from the root `pub use` list in
  `crates/sase_core/src/lib.rs` and the `core_*` aliases in
  `crates/sase_core_py/src/prelude.rs`. Import those items by module path instead, as
  sase-core's `AGENTS.md` requires. Do not add new names to either list.

**Serialized output stays byte-identical.**

- Every renamed serialized field or variant gets
  `#[serde(rename = "<legacy>", alias = "<new>")]`.
- `deny_unknown_fields` request wires need the alias so Python can send new keys. This
  covers binding options, stats requests, completion requests, and launch requests.
- Hand-read JSON reads the new key first, then the legacy key.

**Accept new inputs.** In this phase the new spellings are accepted unconditionally, and
emitted values stay legacy:

- **Env vars:** read `SASE_MACRO_*` first, then `SASE_XPROMPT_*`. That covers
  `PACKAGE_DIR`, `BUILTIN_DIR`, `DEFAULT_DIR`, `PLUGIN_DIRS_JSON`,
  `PLUGIN_CONFIG_PATHS_JSON`, `VCS_PROJECT_CATALOG`, `MODEL_CATALOG`, `MACHINE_CATALOG`,
  `ARTIFACT_REF_CATALOG`, and `GLOSSARY_CATALOG`. Set both `SASE_AGENT_LOCAL_MACROS` and
  `SASE_AGENT_LOCAL_XPROMPTS` for child agents.
- **Binding option keys:** `package_macros_dir`, `default_macros_dir`, and
  `plugin_macro_dirs`.
- **YAML keys:** `macros:` in `sase.yml`, project config, local definition files, and
  frontmatter (`editor/frontmatter.rs`, `editor/diagnostics.rs`). Both keys present is
  an error or diagnostic.
- **Directive:** `%macros_enabled` in `agent_launch/directive_scan.rs`,
  `editor/directive/metadata.rs`, and `editor/wire.rs`. `%xprompts_enabled` stays
  accepted permanently.
- **Content layout** (`content_layout.rs`):
  - Add the canonical `sase/macros` (project), `~/sase/macros` (home), and
    `~/sase/macros/<project>` sources ahead of every xprompt-named directory.
  - The xprompt-named directories become `legacy` sources, tagged so callers can tell
    they carry a retired xprompt name.
  - Expose the result in new additive `macros` and `macro_sources` keys, next to the
    unchanged `xprompts` and `xprompt_sources` keys. Include the
    `entrypoint:sase_macros/…`, `package:macros`, `package:macros/steps`,
    `package:macros/skills`, and `package:default_macros` locators, and the macro
    equivalent of the chezmoi `dot_xprompts` entry.
  - Confirm that the current sase Python mirrors
    (`src/sase/core/content_layout_wire.py`) tolerate the extra keys.
- **Package and plugin directory probing** (`macro_catalog/loader.rs`,
  `loader_sources.rs`): try `macros/` before `xprompts/`, `default_macros/` before
  `default_xprompts/`, and `macros/skills` before `xprompts/skills`.
- **LSP path matching** (`server/actions.rs`, `server/jinja.rs`): recognize `macros`,
  `default_macros`, and `macros.yml|yaml` alongside the legacy names.
- **Agent artifacts:** the scanner and alias history read `macros.json` and
  `raw_prompt.md` first, then the legacy names.
- **Home zones** (`note_attachment/zones.rs`): list `vcs_macro_mru.json` and
  `macro_save_state.json` alongside the legacy names.
- **Procs:** accept the `prompt_proc` field alias and origin `prompt-proc` as equal to
  the legacy spellings.
- **Query profiles:** rename the query-language macro concept to shorthand internally
  (`QueryShorthandSpec`, `HOST_SHORTHAND_TRIGGERS`, `shorthand_target`,
  `shorthands_for_trigger`, `validate_shorthands`). The profile wire accepts
  `shorthands` and keeps emitting `macros`.
- **Legacy-name option:** add `accept_legacy_xprompt_names` (default `true`) to the
  catalog-loading requests and as an LSP initializationOption. When it is `false`, skip
  legacy-tagged sources and report legacy keys as retirement diagnostics that name the
  macro spelling.

**Bindings.** Register the new names, and keep the legacy names registered for the same
functions until `core-flip`:

| New name                                     | Legacy name kept until `core-flip`             |
| -------------------------------------------- | ---------------------------------------------- |
| `resolve_macro_skill_definition`             | `resolve_xprompt_skill_definition`             |
| `macro_skill_definition_wire_schema_version` | `xprompt_skill_definition_wire_schema_version` |
| `macro_argument_spans`                       | `xprompt_argument_spans`                       |
| `prompt_proc_origin`                         | `xprompt_proc_origin`                          |

**LSP crate.**

- Keep the package name `sase_xprompt_lsp` in this phase. Renaming it now would break
  `cargo build -p sase_xprompt_lsp` in every sase workspace still on the old Justfile.
- Add a second `[[bin]] name = "sase-macro-lsp"` built from the same `main.rs`.
- Register `sase.macroLsp.refreshCatalog` and `sase.macroLsp.openSource` next to the
  legacy command ids.

**Do not change** any of these in this phase:

- any `*_SCHEMA_VERSION` or SQLite column
- the golden contracts (`sase_gateway/contracts/`, `python_wire_parity.rs`)
- diagnostic codes, messages, hover or code-action text, or `serverInfo.name`
- fixture files that sase mirrors, such as `tests/fixtures/xprompt_args_corpus.json`
- any CHANGELOG

**Tests:**

- every new input accepts both spellings
- serialized output is unchanged
- both binding names round-trip
- the second LSP binary builds
- `accept_legacy_xprompt_names=false` skips legacy sources

**Exit:** `sase tool run check` passes in sase-core. Current sase master, built against
this core with no sase change, also passes `sase tool run check`.

## Durable data, core wires, and LSP build tooling

Repo: sase.

- **Pin.** Run `just ratchet-core-revision` to move the pin to the landed `core-expand`
  commit, then `just install`. Do not touch the published `sase-core-rs` window in
  `pyproject.toml`.
- **Legacy-names module.** Create `src/sase/legacy_xprompt_names.py`. It is the only
  home of `LEGACY_*` constants and the dual-read helpers for durable data. It is
  permanent and unconditional, and every durable reader goes through it.
- **Bindings.** Call the four new binding names with the new option keys. Update
  `tools/check_sase_core_rs_bindings` expectations and `tools/validate_sase_core_rs`.
- **Durable surfaces.** Each follows the durable-data policy: writers emit only the new
  name, and readers try the new name, then the legacy one.
  - **Agent artifacts:**
    - Writers: `xprompt/used_xprompts.py`, `agent/_restart_recovery.py`, prompt archive,
      `core/revival_inputs.py`. For the TUI's
      `actions/agents/_directive_persistence.py`, change only the string constant.
    - Readers: `scripts/_agent_chat_from_name_common.py`,
      `integrations/_mobile_agent_state.py`, `integrations/_mobile_agent_lifecycle.py`,
      `agent/names/_migration.py`, and agents_sync publication.
    - A revived or restarted pre-rename agent must find its raw prompt.
  - **MRU and save state:** `history/vcs_xprompt_mru.py`, `current_project.py`, and
    `xprompt/save_state.py`, migrating the files as the policy says.
  - **Skills manifest:** `_init_skills_manifest.py`.
  - **Directive regions:** the Python region parser (`xprompt/_disabled_regions.py`)
    accepts both spellings. Directive writers switch in `sase-syntax`.
  - **Procs:**
    - Compare origins through a helper that accepts both spellings.
    - `Proc.from_dict` reads `prompt_proc`, then `xprompt_proc`.
    - Payloads sent to core use the new keys, which `core-expand` accepts.
  - **Caches:** `~/.sase/macro_lsp/` and the managed tmp `macros_catalog` directory. The
    reaper treats the legacy names as removable residue.
- **Wire mirrors.** These mirrors hydrate from either spelling:
  - content layout (consume `macros` and `macro_sources` when present)
  - agent scan (`UsedXPromptWire`, `agent_scan_wire_conversion.py`,
    `agent_alias_history_wire.py`)
  - agent stats (`stats/query.py`)
  - procs
- **Env transport.**
  - The LSP launcher (`integrations/xprompt_lsp.py`) sets `SASE_MACRO_*`.
  - `SASE_LAUNCH_SWARM_XPROMPTS` becomes `SASE_LAUNCH_SWARM_MACROS` (Python-only and
    transient).
  - `SASE_AGENT_LOCAL_MACROS` is read first, with a legacy fallback that `core-flip`
    removes.
  - `SASE_XPROMPT_LSP_CMD` and `SASE_DISABLE_PLUGIN_XPROMPTS` wait for `sase-syntax`.
- **LSP command.** Send `sase.macroLsp.refreshCatalog`.
- **LSP build tooling.**
  - Covers `Justfile` (`rust-lsp-install`), `tools/sase_core_wheel_cache`,
    `dev_update/prebuild_producer.py`, `.github/workflows/build-core.yml`,
    `.github/workflows/master-gate.yml`, and `.github/actions/setup-sase/action.yml`.
  - Each builds whichever LSP package the core checkout has: `sase_macro_lsp` when
    `crates/sase_macro_lsp/Cargo.toml` exists, else `sase_xprompt_lsp`.
  - Each installs the `sase-macro-lsp` binary.
  - `sase lsp` prefers `sase-macro-lsp` and falls back to `sase-xprompt-lsp` on PATH.
    `sase-syntax` moves that fallback into the flag module.
- **Contract-flip readiness.** Audit every Python comparison against a core schema
  version or emitted spelling, including `src/sase/core/health.py` and the
  `xprompts.json` probe in `tools/validate_sase_core_rs`. Record anything that would
  hard-fail on the flip as a note on this phase bead for `core-flip`.
- **Tests.**
  - Legacy-input tests use realistic pre-rename fixtures named `*legacy_xprompt*`:
    - an agent dir with only `raw_xprompt.md` and `xprompts.json`
    - legacy MRU and save-state files
    - a legacy skills manifest
    - legacy proc rows
    - prompts with `%xprompts_enabled` regions
  - Writer tests prove that no legacy name is written.
- **Exit:** `sase tool run check`.

## Module, package, and identifier rename outside the TUI

Repo: sase.

**Scope:** everything outside `src/sase/ace/tui/` and `tests/ace/`. Touch TUI files only
to follow renamed imports and names, and for step 0.

**Step 0: shorthands first.** Before any xprompt rename, rename the query-language macro
concept to shorthand:

- Python identifiers in `ace/query_profile/` (`QueryShorthandSpec`,
  `HOST_SHORTHAND_TRIGGERS`, `profile.shorthands`)
- the profile key sent to core, which becomes `shorthands` (accepted by `core-expand`)
- `query/profile_reference_boolean.py` and `profile_highlighting.py`
- completion kind `shorthand` in `query/completion.py` and
  `widgets/artifacts/patch_filter_bar.py`
- the "Status shorthands" help copy in `modals/help_modal/patches_artifact_bindings.py`
- the matching tests: `tests/test_query_profile_*.py`,
  `tests/test_profile_highlighting.py`, and `tests/ace/tui/test_patch_filter_bar.py`

Afterwards, no `macro` identifier in sase means anything but a Jinja macro.

**Paths** (`git mv`):

- `src/sase/xprompt/` → `src/sase/macro/`. This includes `catalog_template.html.j2` and
  `catalog_style.css`.
- `src/sase/xprompts/` → `src/sase/macros/`. This holds the built-in definitions,
  `workflow.schema.json`, `steps/`, and `skills/`.
- `src/sase/default_xprompts/` → `src/sase/default_macros/`.
- Update everything that hard-codes those paths:
  - any `pyproject.toml` package-data globs
  - the skill source path in `macro/loader_skills.py`,
    `completion/candidates/catalog_prompts.py`, `patch_stitch_audit.py`, and
    `tools/validate_sase_core_rs`
  - the `sase skill init` deploy guard that refuses a dirty skills source tree
- Other modules:
  - `sase/xprompt_links.py`
  - `config/xprompt_sources.py`
  - `core/xprompt_skill_definition_facade.py`
  - `integrations/xprompt_lsp.py`
  - `history/vcs_xprompt_mru.py`
  - `bead/xprompts.py`
  - `agent/xprompt_swarm.py`, `_xprompt_swarm_parsing.py`,
    `_xprompt_swarm_rendering.py`, `multi_agent_xprompt.py`, and
    `multi_prompt_xprompts.py`
  - `doctor/checks_config_xprompts.py` and `checks_deep_xprompt_lsp.py`
  - `main/parser_xprompt.py` and `xprompt_handler.py`
- Tests: `tests/xprompt/` → `tests/macro/`, `tests/_xprompt_*_helpers.py`,
  `tests/test_xprompt_*.py`, and every other test path outside `tests/ace/` containing
  the word.

**Identifiers.**

- Use a token-aware codemod (Python `tokenize`/`ast` or LibCST, not raw `sed`). It
  rewrites:
  - NAME tokens and import statements
  - comments and docstrings
  - test names
  - dotted-module-path strings: `mock.patch("sase.xprompt.…")`, `importlib` targets, and
    string-built module paths
- It must not change any other string-literal value. CLI names, config and frontmatter
  keys, env-var names, directory names, JSON keys and values, help text, and log and
  error messages belong to `sase-syntax`.
- Apply the Rule 2 names: `get_raw_prompt_content`, `submitted_prompt`, and
  `PROMPT_PROC_ORIGIN`. Check first that `submitted_xprompt` is not a persisted key, or
  handle it as durable data.
- Rename `__init__.py` re-exports (about 100 names) and `__all__` lists. Run `just fmt`.
- Refresh test ids in `tests/shard_timings.json` and
  `tests/reproducible_flake_baseline.txt`, and any Symvision whitelist entries that name
  renamed symbols.

**Temporary import shim.** The editable sase-telegram, sase-research-artifacts, and
bugyi-chops installs on this machine import `sase.xprompt.*` at runtime and in tests. A
live telegram bot must not break between this phase and the plugin phases.

- Add `src/sase/xprompt/__init__.py`. It aliases the submodule paths those repos import
  today to their `sase.macro` successors: `models`, the package root, `directives`,
  `workflow_validator_extract`, `catalog`, `loader_sources`, `processor`,
  `loader_parsing`, `runtime_context`, and `workflow_executor_utils`. It also aliases
  `sase.agent.xprompt_swarm`.
- Keep old-name aliases for the specific names those repos import:
  - `InputType`, `XPromptValidationError`
  - `replace_ref_in_vcs_tag`, `extract_vcs_workflow_tag`
  - `extract_xprompt_calls`, `process_xprompt_references`
  - `extract_prompt_directives`, `plan_prompt_fanout_variants`
  - `NoXpromptsFound`, `PdfEngineUnavailable`, `build_xprompts_catalog`
  - `CatalogArtifact`, `CatalogStats`
  - `load_xprompts_from_plugins`, `expand_single_xprompt`
  - `parse_yaml_front_matter`, `UNSET`
  - `bind_runtime_template_vars`, `render_template`
  - `expand_xprompt_swarms_with_metadata`, `list_patch_xprompt_tags`
- Mark each alias `# TEMP(xprompt->macro shim): removed in audit-deploy`. The shim is
  epic scaffolding, so it needs no flag. Add no other alias modules.

**Guard.** Add `tests/test_macro_terminology.py`, modeled on
`tests/test_sase_turn_terminology.py` and using the `contract` marker.

- It fails on any case-insensitive `xprompt` in Python identifiers, import paths, and
  module, package, or test paths under `src/` and `tests/` outside the TUI scope.
- A commented allowlist names the legacy homes and the shim.
- Later phases widen it.

**Exit:** `sase tool run check`.

## User syntax, CLI, config, discovery, and the sunset flag

Repo: sase. Scope: every string-literal contract and user-visible string outside the
TUI.

**Flag.** Create `legacy_xprompt_syntax` first, with the command and sentences from the
compatibility policy. Paste the registry entry it prints, and sync the flag schema
(`tools/sync_feature_flags_schema`). Put every alias in
`src/sase/legacy_xprompt_syntax.py`.

**CLI** (read `cli_rules.md` first):

- **`sase macro`:** `sase macro {catalog,expand,explain,graph,list,show}` keeps today's
  options. Update `main/parser_registry.py`, the `main/entry.py` dispatch, and
  `dest="macro_subcommand"`. `sase xprompt` becomes a hidden, flag-gated alias. Re-sort
  the subcommand lists.
- **`sase path` targets:** `macros-dir`, `macros-schema`, and
  `macros-collection-schema`, with hidden legacy targets. Today
  `xprompts-collection-schema` points at an untracked `xprompts.schema.json`. Verify
  that, then repair the target or drop it.
- **Helper bridges:** the hidden `sase editor helper-bridge macro-catalog` and
  `sase mobile helper-bridge macro-catalog`. The legacy `xprompt-catalog` stays accepted
  through the flag, because pre-flip gateways call it.
- **Help text:**
  - `sase lsp`
  - `sase prompt save` (`--global` → `~/sase/macros/`, `--project` →
    `~/sase/macros/PROJECT/`)
  - `sase project current|set` ("VCS macro MRU")
  - `sase run`, the root help, and snippet help
- **Completion:** `ValueKind.MACRO = "macro"`. Refresh
  `tests/completion/snapshots/cli_spec.json` with `just sync-completion-spec`.
- **JSON output:** `sase macro list -j` emits `type` and `kind` values `macro`. Rename
  every other `xprompt` key or value in CLI JSON. Both are breaking.
- **Doctor:** rename the check ids per the vocabulary, and add
  `config.retired_xprompt_names`.

**Config** (`src/sase/default_config.yml`, `src/sase/config/sase.schema.json`, the
settings loaders, and the successor of `config/xprompt_sources.py`):

- Keys: `macros:`, `macro_aliases:`, `auto_macro_menu`, `macro_placeholder_args`,
  `mentors[].macro`, and schema `macroInputDefinitions`.
- Legacy keys are accepted while the flag is on. Supplying both is an error. With the
  flag off, legacy keys are rejected like other unknown keys, naming the replacement.
- The same rules apply to plugin default configs and overlays.
- Keymap actions belong to the `tui` phase.

**Frontmatter:** `macros:` for local helpers, in `macro/loader_sources.py`,
`prompt_frontmatter.py`, `agent/multi_prompt.py`, and `macros/workflow.schema.json`,
with the legacy key behind the flag. Update the nine tracked files that use it, for
example `sase/xprompts/reads.md`.

**Directive writers** emit `%macros_enabled`: `qa_prompt.py`,
`monitor/followup_persistence.py`, `sdd/_write.py`, the built-ins `fork_by_chat.yml` and
`make_mentor_changes.yml`, and the `sase_run` skill source.

**Discovery and writers:**

- Consume the content-layout `macro_sources`. The canonical write targets become
  `sase/macros/`, `~/sase/macros/`, and `~/sase/macros/<project>/`.
- Replace the duplicated Python directory literals with the content-layout contract. In
  TUI files, change only the literal:
  - the `macro/loader_sources.py` cwd fallback
  - `ace/tui/prompt_catalog.py`, `xprompt_browser_helpers.py`, `add_xprompt_modal.py`,
    and `xprompt_location_modal.py`
  - `completion/candidates/catalog_prompts.py` and `catalog_snippets.py`
  - `workflow_loader_sources.py`
- Skip legacy-tagged sources when the flag is off.
- `skill_destination_for_xprompt_dir` (renamed) maps `<scope>/macros`, and legacy dirs,
  to the sibling `skills/`.
- Move this repo's `sase/xprompts/reads.md` and `sase/xprompts/sync.md` to
  `sase/macros/`.

**Plugins:**

- Read the `sase_macros` group, plus `sase_xprompts` while the flag is on. A
  distribution registered under both loads once, and the new group wins.
- Probe packaged `macros/` first, then `xprompts/` (flag), at every Python site,
  including `main/plugin_discovery.py`.
- `SASE_DISABLE_PLUGIN_MACROS` replaces `SASE_DISABLE_PLUGIN_XPROMPTS`, which stays
  readable behind the flag.

**Env and LSP:**

- `SASE_MACRO_LSP_CMD`, with the legacy name behind the flag.
- The `sase-xprompt-lsp` binary fallback moves into the flag module.
- Pass `accept_legacy_xprompt_names` to core requests and to the LSP
  initializationOptions.

**Strings:**

- Rename help, messages, logs, errors, and prose strings outside the TUI.
- Update the skill sources in `src/sase/macros/skills/`, at least `sase_run.md`,
  `sase_project.md`, `sase_chats.md`, and `sase_agents_status.md`, with the new
  commands, directive, headings, and `raw_prompt.md`. Do not deploy them.
- Update `smoke/pypi/smoke_check.sh` and `demos/scripts/seed_sase_ace_demo`.

**Tests:**

- both flag states for every alias
- legacy directories ignored when the flag is off
- both-present errors
- the doctor check

**Guard.** Widen it to every `xprompt` hit under `src/` and `tests/` outside the TUI
scope, plus the skill sources. Allowlist:

- the legacy homes and the flag module
- the legacy-input tests
- the doctor check's retired-name table
- the shim

**Commit** with a `feat!:` subject and a `BREAKING CHANGE:` footer listing:

- `sase xprompt` → `sase macro`
- the `sase path` targets
- the config keys and the frontmatter key
- the env vars and the entry-point group
- the doctor ids
- the CLI JSON values
- the discovery directories

**Exit:** `sase tool run check`, with every remaining non-TUI hit allowlisted.

## TUI macro surfaces and goldens

Repo: sase.

**Scope:**

- `src/sase/ace/tui/**`, `src/sase/ace/testing/**`, `tests/ace/**`, and `tests/perf/**`
- the TUI parts of `src/sase/default_config.yml` and `src/sase/config/sase.schema.json`

Read `tui.md` and the notes it links first.

**Modules:**

- the `modals/xprompt_*` modules
- `unified_xprompt_save_*`
- `mini_xprompt_*`
- `widgets/_xprompt_arg_assist_*` and `widgets/xprompt_arg_assist.py`
- `xprompt_completion`
- `prompt_panel/_agent_xprompts`
- `statistics_pane_xprompts`
- `util/xprompt_syntax`
- `xprompt_browser_helpers`
- `actions/agent_workflow/*xprompt*`
- the matching tests and PNG fixture modules

Rename identifiers and the CSS ids and classes (`styles.tcss`, about 165 lines) together
with their Python selectors.

**Copy:**

- **Admin Center:** the "XPrompts"/"XP" config sub-tab becomes "Macros", with a 2-letter
  short label that no sibling tab uses.
- **Statistics:** "Macros", "Macro focus", and "Macro statistics unavailable".
- **Browser and pickers:** "Macro Browser [n macros]", "Select Macro", and "Save Pane as
  Local Macro".
- **Other surfaces:**
  - the mini-macro pane copy
  - the help modal (`help_modal/binding_common.py`, plus the agents, axe, and patches
    help files)
  - keymap descriptions (`keymaps/metadata.py`)
  - command-palette keywords (`_app_metadata_display.py`)
  - `prompt_submit_choice_modal.py`
- **Rule 2:**
  - `PREVIEW_TAB_LABEL` becomes `RAW PROMPT` (or `RAW` if the tab strip cannot fit it).
  - The `AGENT XPROMPT` heading becomes `AGENT RAW PROMPT` at its roughly ten sites,
    including `agent_run_log_modal.py`, `_identity_header.py`, and
    `_agent_display_hint_body.py`.

**Keymap actions:**

- The actions become `focus_macro`, `clear_macro_focus`, and
  `start_last_vcs_macro_in_editor`.
- The legacy names go through `legacy_xprompt_syntax.py`.
- Update `default_config.yml` keymaps and the schema (the default-keymap gotcha).

**Admin Center resume ids:** `"macros"`, with a legacy `"xprompts"` reader for resume
state that is unconditional and durable. Keep it distinct from the existing top-level
`"xprompts"→"config"` migration in `config_center_catalog.py`, which works at a
different level.

**Stats requests:** send `macro_top_n` and `macro_focus`. `core-expand` accepts them as
aliases.

**Performance:**

- Renames only. Add no filesystem work, awaits, subprocesses, or full rebuilds to
  navigation or render handlers.
- Run the existing j/k navigation benchmark to confirm no regression.

**PNG goldens:**

- Rename the 34 xprompt-named goldens and their tests, including window titles.
- Re-baseline only the goldens whose pixels change, such as the agents, prompt, mini,
  Admin Center, Statistics, and help goldens.
- Run `just fix-tui-screenshots` through `/sase_monitor`.
- Inspect every creation, removal, and update group in the report.
- Remove stale goldens only after a full run.

**Guard:** widen it to the TUI scope.

**Commit** with a `feat!:` subject and a footer listing the keymap actions.

**Exit:** `sase tool run check`.

## Documentation, site redirect, memory, and first skill redeploy

Repos: sase, plus a chezmoi skill redeploy. The user's request ("catch all references in
this repo") authorizes the memory edits. Follow `/sase_memory_write`.

**First, redeploy skills.** The `sase-syntax` skill sources have landed. From the clean
landed tree, run `sase skill init --force`, then `chezmoi apply` if it was skipped, per
`generated_skills.md`. Declare the chezmoi changes.

**Docs:**

- **Page move:** `git mv docs/xprompt.md docs/macros.md`, and change the mkdocs nav
  entry to `Macros: macros.md`.
- **Redirect:** add `mkdocs-redirects` with `redirect_maps: {xprompt.md: macros.md}`, so
  `sase.sh/xprompt/` keeps working. The plugin goes in the `pyproject.toml` docs
  dependency lists and the docs recipe's install line in the `Justfile`.
- **Links:** fix every `xprompt.md#…` link (about 337) and every anchor whose heading
  changed. Run the strict docs build recipe.
- **Rewrite the concept** in every file:
  - heavy files: `ace.md`, `configuration.md`, `editor.md`, `cli.md`,
    `content_layout.md` (the new discovery order), `llms.md`, `workflow_spec.md`,
    `plugins.md`, `mobile_gateway.md`, and `architecture.md`
  - `plugins.md` covers the `sase_macros` group, and notes that plugins may still ship
    `xprompts/` during the transition
  - `mobile_gateway.md` covers the route rename
  - `query_language.md` and `ace.md` say "status shorthands"
  - anything else the sweep finds
- **Renamed-from section:** `docs/macros.md` gets one short section, "Renamed from
  xprompts". It lists every retired spelling and its replacement, the
  `legacy_xprompt_syntax` flag, and that `%xprompts_enabled` stays accepted. This
  section is the docs allowlist for the guard.
- **Blog:**
  - `docs/blog/posts/xprompts-in-depth.md` (a draft) → `macros-in-depth.md`, and update
    its slug in the `Justfile` draft list.
  - Rewrite the terminology in the published posts, and add a one-line note at the top
    of each changed post saying xprompts were renamed to macros.
- **Images:** rename
  `docs/images/xprompt-resolution-infographic.{png,prompt.md,critique.md}`. If the PNG
  renders the old word, regenerate it through the pipeline its `.prompt.md` describes.
  If you cannot, record a follow-up.
- **README and onboarding:** `README.md` links to `sase.sh/macros/`, and so does the
  onboarding URL in `agent_onboarding.py`.

**Memory:**

- `sase/memory/xprompts.md` → `sase/memory/macros.md`, titled "Macros, Directives, and
  Launch VCS", with its description line updated.
- Glossary strands: `glossary/xprompt.md`, `xprompt-part.md`, `xprompt-swarm.md`,
  `xprompt-workflow.md`, and `xprompt-memory.md` become `macro.md`, `macro-part.md`,
  `macro-swarm.md`, `macro-workflow.md`, and `macro-memory.md`.
  - The keywords become Macro, Macro Part, Macro Swarm, Macro Workflow, and Macro
    Memory.
  - The Macro strand gets a "formerly called an xprompt" clause and keeps `xprompt` as
    an alias, so old `glossary:xprompt` reads resolve.
  - Definitions name `sase/macros/` and the `macros` config field.
- Project Tag alias: "xprompt project tag" → "macro project tag".
- Other notes and strands that mention the concept: `generated_skills.md`
  (`src/sase/macros/skills/`, "macro skills"), `dispatch.md`, and whatever else the
  sweep finds. A generated note such as `sase.md` changes in its generator template, not
  by hand.
- The decision record `hold-pull-fail-open.md` gets the link-target change only.
- Run `sase memory init`. Review the regenerated `glossary.md` roster, `AGENTS.md`,
  `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, `tools/*.md`, and the memory
  README. Confirm that `sase memory read macros.md` and `glossary:macro` resolve.
- Update `tools/smoke_sase_core_rs_glossary_line_break`, whose fixture uses the "Xprompt
  Memory" term.

**Guard.** Widen it to `docs/`, `README.md`, `mkdocs.yml` (allowing the redirect line),
and `sase/memory/` (allowing the decision records and the glossary "formerly" clause and
alias).

**Exit:** `sase tool run check`, including the docs lints, and the strict docs build.

## sase-telegram cutover

Repo: sase-telegram (read its `AGENTS.md`).

- **Compatibility imports.** Add one helper module that imports the renamed sase APIs
  new-first with a legacy fallback. It must work against telegram's installed `sase>=`
  floor and against sase master.
  - It covers `InputType`, `MacroValidationError`, `replace_ref_in_vcs_tag`,
    `extract_macro_calls`, `extract_prompt_directives`, `plan_prompt_fanout_variants`,
    `process_macro_references`, `list_patch_macro_tags`, `NoMacrosFound`,
    `PdfEngineUnavailable`, and `build_macros_catalog`.
  - Route `formatting.py`, `inbound_handlers/gate_input_steps.py`,
    `inbound_handlers/agent_launch.py`, and the commands module through it.
  - Point test `patch()` targets at the module the helper actually resolved.
- **Module and identifiers.** Rename `inbound_handlers/xprompt_commands.py` →
  `macro_commands.py`, and the identifiers (`normalize_launch_macro_at_refs`,
  `_handle_macros_command`, and so on).
- **Strings:** "📚 Building your macros catalog…", "📚 <b>Macros Catalog</b>", "N macros
  across M projects", and the related messages.
- **Bot command.**
  - Register `/macros` in `_SLASH_COMMANDS` as "Export the macros catalog as a PDF".
    `RESERVED_COMMAND_NAMES` keeps both names.
  - `/xprompts` stays an unlisted alias while sase's `legacy_xprompt_syntax` flag is
    enabled, following the `bead`/`beads` alias pattern. Treat a missing flag (an older
    sase) as enabled.
- **Retry fallback** (`agent_launch.py`): read `raw_prompt.md` first, then
  `raw_xprompt.md`.
- **Docs and tests:** update `README.md`, `docs/inbound.md`, and the tests
  (`tests/test_inbound.py` classes and patch targets). Leave the CHANGELOG and `sdd/`
  unchanged.
- **Exit:** `sase tool run check` in sase-telegram.

## sase-github, sase-research-artifacts, and bugyi-chops cutover

Repos: sase-github, sase-research-artifacts, and bugyi-chops
(`sase repo open gh:bbugyi200/bugyi-chops`). Read each repo's guide first.

**sase-github:**

- Add `[project.entry-points."sase_macros"] sase_github = "sase_github"` next to the
  legacy `sase_xprompts` entry.
- Keep `src/sase_github/xprompts/`, per the plugin-directory policy.
- Drop the empty `xprompts: {}` from `src/sase_github/default_config.yml`, keeping a
  valid empty config if the `sase_config` entry point needs one.
- The `.github/workflows/publish.yml` smoke check expects both groups.
- Docs:
  - `README.md` "Macros" table
  - `docs/xprompts.md` → `docs/macros.md`
  - `docs/architecture.md`
  - `docs/configuration.md`, also fixing its stale `xprompts.pr_diff` claim
  - `CLAUDE.md`, or its memory source if it is generated
  - script and test docstrings

**sase-research-artifacts:**

- Register the same pair of groups, and keep `src/sase_research_artifacts/xprompts/`.
- Tests that run against sase master import through a new-first fallback helper
  (`load_macros_from_plugins`/`load_xprompts_from_plugins`, `expand_single_macro`, and
  so on).
- Keep the legacy import names in the `publish.yml` published-floors smoke, which pins
  an old sase, and in its contract test `tests/test_ci_install_contract.py`.
- `tests/test_wheel_contract.py` asserts both groups.
- Rename `tests/test_xprompt_loading.py` → `test_macro_loading.py`.
- Update `docs/xprompts.md` → `docs/macros.md`, `README.md`, `AGENTS.md`,
  `docs/architecture.md`, and the agent-facing prose in `research_swarm.md`.

**bugyi-chops:** `tests/test_toobig_split.py` imports `extract_prompt_directives`
new-first with a fallback, and `README.md` line 68 says macros.

**Exit:** each repo's check passes.

## sase-nvim cutover

Repo: sase-nvim. It has no `AGENTS.md`, CHANGELOG, or Justfile, and its tests are
headless `tests/*.lua`.

**Public API, with deprecation shims:**

- **Setup keys:** `macro_highlight` and `macro_spacer`. The legacy `xprompt_*` keys
  still map, with a one-time deprecation notice.
- **Highlight groups:** `SaseMacroArgKey` and `SaseMacroArgOperator`. A user override of
  either the new or the legacy name must take effect.
- **Commands:** `:SaseMacros` and `:SaseMacrosRefresh`, with the legacy commands kept as
  deprecated aliases.
- **Telescope:** the extension export `macros`, with a legacy `xprompts` export.
- **LSP client name:** `sase-macro-lsp`, at every `CLIENT_NAME` site.
- **Other names:** `vim.g.loaded_sase_macro` and renamed augroups.

**sase contracts, new first:**

- **CLI:** `sase macro list`, falling back to `sase xprompt list` when the installed
  sase predates the rename. Accept the `type`/`kind` values `macro` and legacy
  `xprompt`.
- **Binary:** `sase-macro-lsp`, then `sase-xprompt-lsp`.
- **Env var:** `SASE_MACRO_LSP_CMD`, then `SASE_XPROMPT_LSP_CMD`.
- **yamlls schema:** `sase path macros-schema`, falling back to the legacy target, with
  globs for `**/sase/macros/**` plus the legacy `**/sase/xprompts/**`.
- **Path detection** (`lsp.lua`): `sase/macros` and `default_macros`, plus the legacy
  names.
- **Smoke tests:** detect the crate name (`cargo run -p sase_macro_lsp` or
  `sase_xprompt_lsp`), and use the `SASE_MACRO_*` env vars.

Mark each fallback `-- legacy xprompt spelling; remove with legacy_xprompt_syntax`.

**Modules and docs:**

- Lua modules: `sase.macro`, `sase.complete.macro`, `sase.macro_semantic_highlight`,
  `sase.macro_spacer`, and `plugin/sase_macro.lua`.
- Rename the fixtures and tests, update the user-facing labels, and add a migration note
  to the README.
- Confirm that chezmoi's `home/dot_config/nvim/lua/plugins/sase_nvim.lua` still needs no
  change.

**Exit:** the headless test suite passes.

## sase-core contract flip with same-turn pin bump

Repos: sase-core, sase, and chezmoi, all in **one declared turn**. sase-core takes a
breaking `feat!:` change with a `BREAKING CHANGE:` footer. Start only after every
earlier phase has landed.

**sase-core.**

- **Serialized output:**
  - Remove the legacy `rename` pins.
  - Keep `alias = "<legacy>"`, commented `// legacy xprompt spelling`, only where
    durable data can still carry the old spelling: proc rows, artifact names,
    `%xprompts_enabled`, and any persisted value the earlier phases verified.
  - Drop the aliases on in-process request wires.
  - The content layout loses its `xprompts`/`xprompt_sources` keys, and `sase/macros` is
    canonical.
- **Bindings:** remove the four legacy binding names and their aliases, and assert them
  absent.
- **Env vars:** stop reading the `SASE_XPROMPT_*` transport variables, and set only
  `SASE_AGENT_LOCAL_MACROS`.
- **Emitted values:**
  - origin `prompt-proc` and field `prompt_proc`
  - `WorkflowKind` value `macro`
  - completion context kinds `macro` and `macro_argument_*`, plus `active_macro`
  - argument source `macro`
  - Jinja scope kind `macro`, `macro_only`, and `macro_skill_only`
  - artifact-ref target kind `macro_skill`, which keeps parsing legacy if it is
    persisted
  - agent-stats keys (`macros`, `runs_with_macros`, `runs_without_macros`,
    `distinct_macros`, `macro_top_n`, `macro_breakdown_top_n`, `macro_focus`)
  - `local_macros_file`
  - the `axe_chop` migration key
  - `MACRO_SKILL_DEFINITION_WIRE_SCHEMA_VERSION`
  - query profiles emit `shorthands` only
- **Diagnostics:** about 50 codes become `*macro*`, for example `unknown_macro`,
  `invalid_macro_arg_type`, and `missing_macro_memory_tag`. Also change:
  - the messages, for example "Unknown macro `{name}`"
  - the source, which becomes `sase-macro`
  - the code-action titles, "Open macro source" and "Refresh macro catalog"
  - hover text, including the "Jinja macro" disambiguation
- **LSP crate:**
  - `git mv crates/sase_xprompt_lsp crates/sase_macro_lsp`. The package becomes
    `sase_macro_lsp`, and `sase-macro-lsp` is its only binary.
  - Update the description, `serverInfo.name`, `--version`, the log filter, and
    `MacroLspServer`. Only the `sase.macroLsp.*` command ids remain.
  - Update the root `Cargo.toml` members and `Cargo.lock`.
  - In `release-plz.toml`, add `[[package]] name = "sase_macro_lsp"` with
    `release = false` **and its own** `git_tag_name = "sase_macro_lsp-v{{ version }}"`.
    Without it, git_only release-pr cannot find the renamed crate at the last `v*` tag,
    and every release fails.
  - Run `cargo hakari generate && cargo hakari manage-deps` if the features check fails.
- **Agent-scan index:** `xprompts_sig` → `macros_sig`, `used_xprompts` → `used_macros`,
  and `UsedMacroWire`. Bump the index schema (v19 today) through the existing migration
  path.
- **Schema versions:** bump every one whose emitted shape changed, for example the proc
  wire (keeping the older versions supported), proc dispatch, agent scan, agent stats,
  and content layout. Bump the fleet protocol only if a fleet wire changed.
- **Gateway:**
  - Rename `MobileXprompt*Wire` → `MobileMacro*Wire`.
  - Add `/api/v1/macros/catalog`, and keep the legacy route as a deprecated alias.
  - The host bridge calls `macro-catalog`.
  - Regenerate `crates/sase_gateway/contracts/api_v1/mobile_api_v1.json` with
    `UPDATE_MOBILE_CONTRACT=1`, and update the gateway README.
- **Fixtures:**
  - `tests/fixtures/xprompt_args_corpus.json` → `macro_args_corpus.json`, in both repos
    in this turn.
  - Regenerate `tests/fixtures/command_line/sase_spec.json` from the landed sase CLI,
    per that directory's README.
  - Update `python_wire_parity.rs` and the prose in
    `tests/fixtures/note_attachment/at_bearing_notes.jsonl`.
- **Docs:** `README.md`, the `AGENTS.md` crate-table row,
  `crates/sase_core_py/PYPI_README.md`, and `docs/`.

**sase, same turn.** The host commits sase-core first and writes the pushed SHA into
`sase-core-revision.txt` (`repos.linked[].revision_pin`).

- Drop the LSP package-name detection and always use `sase_macro_lsp`, in the Justfile,
  tools, and CI.
- Update the Python mirrors and the schema-version mirrors listed by `sase-durable`,
  plus `src/sase/core/health.py` and `tools/validate_sase_core_rs`.
- Update the fixtures and goldens that capture core output keys.
- Remove the `SASE_AGENT_LOCAL_XPROMPTS` fallback, and update doc lines that name the
  crate path.
- Keep every durable legacy reader and its tests.

**chezmoi, same turn.**

- In `home/bin/executable_install_sase_github`:
  - rename `install_xprompt_lsp` → `install_macro_lsp`
  - probe with `command -v sase-macro-lsp`
  - install with
    `cargo install --path "$SASE_CORE_DIR/crates/sase_macro_lsp" --bin sase-macro-lsp --locked --force`
  - run `cargo uninstall sase_xprompt_lsp` when a stale `sase-xprompt-lsp` is present
- Update `tests/bash/install_sase_github_test.sh` to match.
- Run chezmoi's check per its `AGENTS.md`.

**Before declaring:**

- Build this core locally, then run sase on the new pin with `sase tool run check`.
- Confirm `sase core health` is green.
- Declare all three repos.

**Exit:** each repo's check passes, and `sase core health` is green.

## Cross-repo audit, guardrail, chezmoi, and machine migration

Repos: sase, sase-core, sase-telegram, sase-github, sase-research-artifacts, sase-nvim,
chezmoi, bugyi-chops, and bob-cli. actstat was verified to have no hits.

- **Remove the shim.** Confirm that telegram, both plugins, and bugyi-chops import
  new-first. Then delete `src/sase/xprompt/` and every `TEMP(xprompt->macro shim)`
  alias.
- **Final guard.** `tests/test_macro_terminology.py` now scans every tracked file in
  sase (`Justfile`, `tools/`, `.github/`, `smoke/`, `demos/`, and the rest). Its
  commented allowlist contains only:
  - the legacy homes and the flag module
  - legacy-input tests and fixtures
  - the doctor retired-name table
  - the "Renamed from xprompts" docs section and the mkdocs redirect line
  - the glossary "formerly" clause and alias
  - the decision records, CHANGELOG, and `sdd/`
- **Sweep.**
  - Run `git grep -i xprompt` in every listed repo, and classify each hit as allowlisted
    legacy, history, or the deferred plugin directory. Fix stragglers in place.
  - Sweep commits that landed on sase, sase-core, and the plugins since this epic began,
    to catch wording added by concurrent work.
  - Annotate each still-open bead whose title or description names a renamed identifier
    with `sase bead note`, mapping old names to new.
- **bob-cli.** Rename the fixture across `docs/highlights-ref-sync.md`,
  `src/native/highlights_ref/create.rs`, and `tests/cli/highlights/create.rs`:
  `xprompt_role_binding` becomes `macro_role_binding`, including the derived id, path,
  title, and PDF name, and the non-UTF-8 `xprompt_\xff.md` becomes `macro_\xff.md`. Keep
  the assertions consistent, and run bob-cli's check.
- **chezmoi.**
  - `git mv home/sase/xprompts home/sase/macros`.
  - Add a `.chezmoiremove` entry for `sase/xprompts` at the source root, so every
    machine drops the stale target directory. Verify the result with
    `chezmoi apply --dry-run`.
  - In `home/dot_config/sase/sase.yml`: rename `xprompts:` → `macros:`, change the
    snippet expansions (`x`, `tx`, `xm`, `xp`, `xs`, `xsw`, `xw`) to say macro, macro
    memory, macro skill, macro swarm, and macro workflow, and update the tribes comment.
    The trigger keys stay unchanged, and the `keep-sorted` order holds.
  - Change the expansions in `home/dot_config/nvim/lua/bb_utils/_snip_utils.lua`.
  - In `home/dot_config/aliases.sh`, `sase -p xprompt` → `sase -p macro`. The alias name
    stays `sax`.
  - Update the comments in `home/bin/executable_sshot_fetch` and
    `home/bin/executable_macscrot`.
  - Drop `.xprompts/*` from `home/dot_gitignore_global` once no `.xprompts/` directory
    exists in any checkout on this machine. If one does exist, record a follow-up
    instead of moving private files.
  - Leave chezmoi's `.beads/` and `sdd/` unchanged.
- **Other machines.** The `sase.yml` key rename and the `.chezmoiremove` also reach
  every machine that applies this chezmoi source. Before landing them, read the
  `tailnet.md` and `dispatch.md` reference memory, and confirm that each such machine
  and each enrolled dispatch machine runs a sase that reads `macros:` and
  `~/sase/macros/`. Use `sase update` there where needed. If a machine cannot be updated
  now, keep the chezmoi config on the legacy spellings and record a follow-up.
- **Live athena migration.**
  - The installed `sase` is an editable uv tool install of the primary checkout, with
    the plugins injected. Preview with `sase update -n`, then run `sase update` so it
    runs landed master. If a checkout is dirty, stop and record a follow-up instead of
    forcing.
  - Install `sase-macro-lsp`, remove `~/.cargo/bin/sase-xprompt-lsp`, and confirm the
    `tools.macro_lsp` doctor check is green.
  - Confirm that `~/.sase/vcs_xprompt_mru.json` and `~/.sase/xprompt_save_state.json`
    migrated on first use, and delete the disposable `~/.sase/xprompt_lsp/`.
  - Run `chezmoi apply`, and confirm that `~/sase/macros/` holds `pick_plan.md` and
    `sshot.yml` and that `~/sase/xprompts/` is gone.
  - Confirm that `sase doctor` reports no retired xprompt names for the home config and
    for each enabled project (sase, actstat, bob-cli).
- **Skills.** From the landed tree, run `sase skill init --force` (then `chezmoi apply`
  if it was skipped). Confirm that `sase skill init --check` is clean and that the
  manifest carries `macro_set_sha256`.
- **Confirm the flag.** The `legacy_xprompt_syntax` flag bead and its registry entry
  exist, with both-state tests.
- **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes for:
  - moving the plugins to packaged `macros/`, dropping `sase_xprompts`, and raising
    their sase floors once a sase release with macro support is published
  - removing the mobile v1 route alias at the next API version
  - any image or machine step that was deferred
- **The epic is done when** all of the following hold:
  - A fresh agent can define `sase/macros/foo.md`, invoke `#foo`, inspect it with
    `sase macro list|show|explain`, browse it in the TUI's Macros surfaces, and edit it
    in nvim through `sase-macro-lsp`, using only macro spellings.
  - Pre-rename agent artifacts, proc rows, MRU and save-state files, skills manifests,
    stored `%xprompts_enabled` prompts, and index databases still load.
  - `git grep -i xprompt` in every listed repo returns only classified, allowlisted
    lines.
  - Rust and Python agree, and `sase core health` is green.
  - `sase tool run check`, or the repo's documented check, passes in every changed repo.
