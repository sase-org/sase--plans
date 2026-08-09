---
tier: epic
title: Move glossary and amd_h1_title under a new memory config section
goal: '`memory.glossary` and `memory.h1_title` are the canonical config paths across
  the schema, the readers, the Rust layer diagnostics, the docs, and every real config
  file SASE owns, while the legacy top-level `glossary` and `amd_h1_title` keys keep
  working as deprecated aliases so no repository silently loses its glossary or its
  generated AGENTS.md title during the migration.

  '
phases:
- id: core-scope
  title: Nested glossary scope diagnostic in sase-core
  depends_on: []
  size: small
  description: 'core-scope: teach the Rust config provenance pass to diagnose a non-local
    `memory.glossary` in addition to the legacy top-level `glossary`, and extend the
    config parity test to cover both paths.'
- id: config-surface
  title: Config schema, deprecation registry, and packaged defaults
  depends_on:
  - core-scope
  size: small
  description: 'config-surface: add the `memory` object with `h1_title` and `glossary`
    to sase.schema.json, mark the two legacy top-level keys deprecated, register them
    in DEPRECATED_TOP_LEVEL_KEYS, restructure default_config.yml, and update the schema
    and inventory tests.'
- id: read-sites
  title: Nested reads with legacy fallback
  depends_on: []
  size: medium
  description: 'read-sites: add one shared glossary-location resolver, read `memory.h1_title`
    and `memory.glossary` first with a legacy fallback in the AMD title loader, the
    memory init glossary loader, and the editor glossary catalog, and update every
    affected test fixture.'
- id: self-migration
  title: Migrate sase's own config and documentation
  depends_on:
  - config-surface
  - read-sites
  size: medium
  description: 'self-migration: move this repository''s own `sase/sase.yml` keys under
    `memory:`, regenerate memory, and rewrite the configuration/init/memory/xprompt/ace
    docs to document the canonical paths and the deprecated aliases.'
- id: downstream-repos
  title: Migrate downstream repository configs
  depends_on:
  - config-surface
  - read-sites
  size: small
  description: 'downstream-repos: migrate the bob-cli project config and the chezmoi
    user overlay to the nested form, correct the sase-nvim README reference, and confirm
    actstat needs no change.'
proposed_by: bbugyi200.athena.we.f0.w1
create_time: 2026-08-09 10:21:54
status: done
bead_id: sase-ia
---

- **PROMPT:** [prompts/202608/memory_config_section.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/memory_config_section.md)
- **BEAD:** [sase-ia](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ia/README.md)

# Plan: Move glossary and amd_h1_title under a new memory config section

## Goal

Introduce a `memory:` config section and relocate two existing top-level keys into it:

| Legacy top-level key | Canonical path    |
| -------------------- | ----------------- |
| `glossary`           | `memory.glossary` |
| `amd_h1_title`       | `memory.h1_title` |

Both legacy keys remain readable as deprecated aliases. Every config file SASE actually
owns is migrated to the canonical form as part of this epic.

## Current state

`glossary` and `amd_h1_title` are top-level keys in `src/sase/config/sase.schema.json`
(`amd_h1_title` at the `properties` head, `glossary` just below the template keys, using
the `glossaryEntry` definition). They are read in three places, all of which parse a
single YAML file directly rather than the merged config:

- `src/sase/amd/_config.py` — `_load_project_amd_h1_title` reads the project config and
  `_load_user_amd_h1_title` folds `~/.config/sase/sase.yml` plus its `sase_*.yml`
  overlays, keeping the last file that declares the key.
- `src/sase/main/init_memory/glossary.py` — `load_project_glossary_memory` reads
  `loaded.config["glossary"]` and renders `sase/memory/glossary.md`.
- `src/sase/xprompt/glossary_catalog.py` — `_load_editor_glossary_catalog` reads the
  same key from a ruamel round-trip mapping so it can attach YAML line/column ranges,
  and stamps `config_key_path=("glossary", term)` into each `GlossarySource`.

Two more surfaces know the key by name:

- `crates/sase_core/src/config/provenance.rs` in the `sase-core` repo emits the
  `glossary_scope` error when a non-`local` layer declares `glossary`; the invariant is
  that a glossary may only come from a project's own `sase/sase.yml`.
- `src/sase/default_config.yml` ships `amd_h1_title: null` and a commented glossary
  example.

Unknown top-level config keys are silently ignored at runtime —
`UNSUPPORTED_TOP_LEVEL_KEYS` in `src/sase/config/layers.py` only powers reporting. That
is why this epic keeps compatibility aliases; see the decision below.

## Key decisions

**1. Keep the legacy keys working as deprecated aliases.** A hard cut would be silent,
not loud: `sase memory init` runs as a post-commit hook in every managed repo, so the
first commit in a not-yet-migrated repo would quietly regenerate its `AGENTS.md` with
the glossary section deleted and no error anywhere. The repo already has the exact
precedent for this migration shape — `linked_repos` → `repos.linked` and `machine_name`
→ `id.machine_name` both live in `DEPRECATED_TOP_LEVEL_KEYS`, keep working, and surface
a non-fatal `deprecated_key` diagnostic through `sase config layers` and `sase doctor`.
Adding the two new entries reuses that whole rail for free.

**2. Canonical wins over legacy within one file.** When a single config file declares
both `memory.h1_title` and `amd_h1_title` (or both glossary locations), the `memory.*`
value is used. Layer precedence between files is unchanged.

**3. The glossary loaders stay in Python.** Per the Rust core backend boundary, the Rust
core owns glossary _entry_ validation and matching plus config-_layer_ diagnostics, and
already does. YAML-shape reading for these two keys is already Python-side in both
readers; this epic keeps that split and only adds a shared resolver. Porting the loaders
to Rust is out of scope.

**4. Do not put `glossary` in `default_config.yml`.** The packaged default layer may
declare `memory.h1_title: null` but must never declare `memory.glossary`, or every
project would inherit a non-local glossary and trip the scope diagnostic.

## Repository inventory

Verified while planning — these are the only config files that carry either key:

| Repo    | File                                   | Keys present   |
| ------- | -------------------------------------- | -------------- |
| sase    | `sase/sase.yml`                        | both           |
| sase    | `src/sase/default_config.yml`          | `amd_h1_title` |
| bob-cli | `sase/sase.yml`                        | `glossary`     |
| chezmoi | `home/dot_config/sase/sase_athena.yml` | `amd_h1_title` |
| actstat | `sase/sase.yml`                        | neither        |

`actstat` is the third enabled project and needs no change. The `sase-github`,
`sase-telegram`, and `sase-core` repos have no `sase/sase.yml` at all.
`home/dot_config/sase/sase.yml` and `sase_kellys_mbp.yml` in chezmoi declare neither
key.

Every phase that touches a repo other than the sase checkout MUST open it with the
`/sase_repo` skill first and use only the path that command prints.

## Nested glossary scope diagnostic in sase-core

Open `sase-core` with `/sase_repo`.

In `crates/sase_core/src/config/provenance.rs`, `build_sources` currently checks
`keys.iter().any(|key| key == "glossary")`. `layer.value` is the full JSON value for the
layer, so replace that single check with one that considers both locations:

- legacy: a top-level `glossary` key;
- canonical: a `memory` object containing a `glossary` key.

Emit one `glossary_scope` error per offending location for any layer whose `kind` is not
`local`, with `path` set to the location that was found (`glossary` or
`memory.glossary`) and the message naming that same path, e.g.
``"`memory.glossary` is only valid in project-local sase.yml"``. A layer that declares
`memory` without `glossary` — which the packaged default layer will do once
`config-surface` lands — must not be diagnosed.

Extend `crates/sase_core/tests/config_parity.rs`'s
`inventory_diagnoses_glossary_outside_local_layer` (or add a sibling test) so it covers:
a non-local `memory.glossary` is diagnosed; a local `memory.glossary` is not; the legacy
top-level `glossary` behavior is unchanged; and a non-local `memory` without `glossary`
produces no `glossary_scope` diagnostic.

This phase lands first because sase's CI builds `sase_core_rs` from `sase-org/sase-core`
HEAD (`.github/workflows/ci.yml`, `build-core` job), and `just install` in a workspace
rebuilds it from the linked checkout. The `config-surface` phase's inventory test cannot
pass until this is on `sase-core` master.

## Config schema, deprecation registry, and packaged defaults

All work in the sase repo.

**`src/sase/config/sase.schema.json`**

- Add a top-level `memory` object property: `"type": "object"`,
  `"additionalProperties": false`, with a description covering memory and
  agent-instruction generation. Give it two properties:
  - `h1_title` — `["string", "null"]`, default `null`, carrying the wording currently on
    `amd_h1_title`.
  - `glossary` — the current `glossary` block verbatim (its `propertyNames` pattern and
    `additionalProperties` `$ref` to `#/definitions/glossaryEntry`).
- Leave the `glossaryEntry` definition where it is.
- Keep the top-level `amd_h1_title` and `glossary` properties, add `"deprecated": true`
  to each, and reword each description to point at its `memory.*` replacement — follow
  the existing top-level `machine_name` entry as the model.

**`src/sase/config/layers.py`**

Add to `DEPRECATED_TOP_LEVEL_KEYS`, keeping the mapping alphabetically ordered:

```python
"amd_h1_title": "memory.h1_title",
"glossary": "memory.glossary",
```

No other change is needed there: `_collect_deprecated_keys`, `sase config layers`, the
`sase doctor` config-layers check, and the Rust `deprecated_key` diagnostic all read
that mapping.

**`src/sase/default_config.yml`**

Replace the leading `amd_h1_title: null` line and the commented glossary example with a
single `memory:` block holding `h1_title: null` and the glossary example, re-indented,
still fully commented out. Leave `amd_agents_template`, `amd_agents_minimal_template`,
`memory_sase_template`, and `memory_readme_template` where they are.

**Tests**

- `tests/test_config_schema.py` — `test_config_schema_validates_project_glossary_shape`
  should validate the canonical `{"memory": {"glossary": ...}}` shape and reject the
  same invalid entries under it; keep a case proving the legacy top-level form still
  validates; assert `schema()["properties"]["glossary"]["deprecated"] is True`.
- `tests/test_config_schema_agent_experience.py` —
  `test_config_schema_accepts_amd_h1_title_string_or_null` should cover
  `{"memory": {"h1_title": ...}}` and keep the legacy key accepted; assert the legacy
  key is marked deprecated. Add a case proving `memory` rejects an unknown property.
- `tests/test_config_inventory.py` — the `glossary_scope` assertions around line 353
  should exercise the nested path and keep a legacy case.

`tests/test_config_schema.py` is listed in `tests/contract_manifest.txt`, so this phase
must verify with `just check-full`, not just `just check`.

## Nested reads with legacy fallback

All work in the sase repo. Run `just install` first — this workspace's virtualenv may be
stale, and the phase depends on nothing from `config-surface` at runtime.

**New shared resolver.** Both glossary readers need identical location logic, including
the resolved key path for diagnostics and for `GlossarySource.config_key_path`. Add one
small module (`src/sase/glossary_config.py` is a reasonable home) exporting the key
constants and a resolver that takes a loaded mapping and returns the glossary node, the
resolved key path as a tuple (`("memory", "glossary")` or `("glossary",)`), the dotted
display path for messages, and an error string when `memory` is present but is not a
mapping. Returning the node itself matters: a ruamel round-trip nested lookup still
carries `.lc` data, which is what `_node_key_location` needs.

**`src/sase/amd/_config.py`**

Add a helper that extracts the title from one loaded mapping and reports which path it
came from, then use it in both `_load_project_amd_h1_title` and
`_load_user_amd_h1_title`:

- prefer `memory.h1_title` when `memory` is a mapping that contains `h1_title`;
- otherwise fall back to the top-level `amd_h1_title`;
- error when `memory` is present but is not a mapping;
- keep `_validate_amd_h1_title`'s existing rules (null allowed, non-string rejected,
  blank rejected) but make the message name the path actually read, so a nested value
  reports `memory.h1_title must be a string or null`.

Preserve `_load_user_amd_h1_title`'s existing semantics exactly: it walks `sase.yml`
then sorted `sase_*.yml` overlays and keeps the last file that _declares_ the key, so
the helper must distinguish "declared" from "declared as null".

**`src/sase/main/init_memory/glossary.py`**

Use the shared resolver instead of `loaded.config.get(GLOSSARY_CONFIG_KEY)`. Thread the
resolved display path through `_glossary_entries`' `prefix`, the per-term `path`
strings, and the `diagnostic.path or ...` fallback so every error message names the
location the user actually wrote. Surface the resolver's error as a blocker.

**`src/sase/xprompt/glossary_catalog.py`**

Same resolver against the round-trip mapping. Thread the resolved key path into
`_glossary_entries` and `_glossary_source` so `config_key_path` becomes
`("memory", "glossary", term)` for the canonical form and stays `("glossary", term)` for
the legacy form. `_key_range` keeps operating on the glossary mapping node itself, so
term ranges are unaffected; `_value_range` for `definition`/`aliases` is likewise
unchanged. Update `GLOSSARY_CONFIG_KEY`'s use in `_format_diagnostic` and reconcile the
module's `__all__` with whatever constants survive.

The `config_key_path` value flows out through the LSP catalog payload and is joined with
`.` for display in `sase-core`'s `glossary.rs`; no Rust change is needed for that, and
no consumer parses the tuple's length.

**Tests to update**

- `tests/xprompt/test_glossary_catalog.py` — nest every `glossary:` fixture under
  `memory:`; the `config_key_path` expectation near line 136 becomes
  `["memory", "glossary", "Agent Clan"]`; add a legacy-form test asserting the entry
  still loads with `["glossary", "Agent Clan"]`, and a precedence test where one file
  declares both.
- `tests/main/test_init_memory_glossary.py` — nest all seven `glossary:` fixtures; add a
  legacy-form case and a both-declared precedence case.
- `tests/main/test_init_memory_agents_templates.py`,
  `tests/main/test_init_memory_validation.py`, `tests/main/test_init_memory_opt_in.py`,
  `tests/main/test_init_memory_managed_agents.py`,
  `tests/main/test_init_memory_markdown_templates.py`,
  `tests/main/test_init_memory_agent_docs.py`,
  `tests/main/test_init_memory_bead_note.py` — migrate `amd_h1_title:` fixtures to
  `memory:`/`h1_title:`. Keep at least one legacy fixture
  (`test_init_memory_opt_in.py`'s "legacy title is not an opt-in" case is a natural one)
  and update the `"amd_h1_title must be a string"` blocker assertion in
  `test_init_memory_validation.py` to the nested message.
- `tests/test_core_glossary_facade.py` — update the hand-built
  `config_key_path=("glossary", ...)` source to the canonical tuple.

Verify with `just check`; run `just check-full` if the scoped selection reports anything
unusual.

## Migrate sase's own config and documentation

All work in the sase repo, after `config-surface` and `read-sites`.

**`sase/sase.yml`** — move `amd_h1_title` to `memory.h1_title` and the whole `glossary:`
block to `memory.glossary`, preserving every definition and alias byte-for-byte and
keeping the block's position relative to `is_sase_managed`. Then run `sase memory init`
(the repo's own post-commit hook does this too) and confirm the regenerated `AGENTS.md`,
`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, `sase/memory/glossary.md`, and
`sase/memory/README.md` are **unchanged** — the values did not change, only their
location, so any diff in generated output is a bug in `read-sites`.

**Docs.** `docs/configuration.md`:

- Rename `### amd_h1_title` to `### memory.h1_title` and `### glossary` to
  `### memory.glossary`, update their YAML samples and field tables to the nested form,
  and add a short note in each that the legacy top-level key is deprecated but still
  read, with the canonical value winning when both appear in one file.
- Fix the Table of Contents: add entries for both sections. Note that the current TOC is
  missing a `glossary` entry entirely — this is a pre-existing gap to close here.
- Update the prose references near lines 3446, 3505, and 3521.

Then update the cross-references and anchors in `docs/init.md` (the managed-project
glossary paragraph and the `amd_h1_title` opt-in wording), `docs/memory.md` (the
`amd_h1_title` sentence), `docs/xprompt.md`, and `docs/ace.md`. Anchors currently
written as `configuration.md#glossary` must move to the new heading's slug;
`mkdocs build --strict` fails on a broken anchor, so verify with `just docs-check` in
addition to `just check-full`.

`CHANGELOG.md` is generated by release-please — do not edit it; describe the move in the
commit subject and body instead.

## Migrate downstream repository configs

Each repo below must be opened with `/sase_repo` first.

**bob-cli** — in `sase/sase.yml`, nest the existing `glossary:` block (Pomodoro,
Schedule Log, Task Link) under `memory:`, preserving definitions and aliases exactly.
Run `sase memory init` in that repo and confirm its generated instruction files and
`sase/memory/glossary.md` are unchanged.

**chezmoi** — in `home/dot_config/sase/sase_athena.yml`, replace
`amd_h1_title: "athena - Bryan Bugyi's Home Server"` with the nested
`memory:`/`h1_title:` form. This is the chezmoi _source_ file; deploying it to
`~/.config/sase/` requires a `chezmoi apply` that the user performs — call that out in
the phase's completion notes rather than running it. Until it is applied, the deployed
legacy key keeps working through the compatibility alias, which is exactly why the alias
exists.

**sase-nvim** — `README.md` around line 266 tells the reader to start "from a SASE
project with a `glossary` entry in `sase/sase.yml`"; update that to `memory.glossary`.
No Lua or syntax change is needed — the plugin consumes LSP semantic tokens, not config
keys.

**actstat** — verified during planning to declare neither key. Re-confirm and record
that no change was needed.

## Verification

- `just install` before anything else in each sase workspace; the workspace venv may be
  stale and `just install` is what rebuilds `sase_core_rs` from the linked `sase-core`
  checkout.
- `just check-full` for the `config-surface` and `self-migration` phases, because
  `tests/test_config_schema.py` is in `tests/contract_manifest.txt`; `just check` is
  enough for `read-sites` unless its scoped selection escalates.
- `just docs-check` for `self-migration`.
- `cargo test` in `sase-core` for `core-scope`.
- After the full epic, `sase config layers` should report `amd_h1_title` and `glossary`
  under "deprecated key" only if some config still uses them — after this epic, none
  should, except the deployed chezmoi copy until the user runs `chezmoi apply`.

## Out of scope

- `amd_agents_template`, `amd_agents_minimal_template`, `memory_sase_template`, and
  `memory_readme_template` stay top-level. Folding them into the new `memory:` section
  is a plausible follow-up — worth a task bead once this lands — but the user asked for
  two specific keys and moving four more would enlarge the blast radius.
- Removing the deprecated aliases. That should become a follow-up task bead filed after
  the chezmoi change is applied on every machine, so the aliases have actually gone
  unused.
- Porting the glossary YAML loaders into the Rust core.
