---
tier: epic
title: Canonical SASE directories for project and home content
goal: 'Project-local configuration, xprompts, and memory, plus home-managed xprompts
  and memory, live under one canonical sase/ tree without breaking unmigrated projects,
  and every enabled project, integration, document, and infographic reflects the new
  layout.

  '
phases:
- id: layout-contract
  title: Canonical layout contract and compatibility policy
  depends_on: []
- id: config-xprompt-runtime
  title: Project config and xprompt runtime migration
  depends_on:
  - layout-contract
- id: memory-amd-runtime
  title: Memory and agent-document runtime migration
  depends_on:
  - layout-contract
- id: editor-and-core-integrations
  title: Rust catalog and editor integration alignment
  depends_on:
  - config-xprompt-runtime
  - memory-amd-runtime
- id: sase-project-migration
  title: SASE project-local content migration
  depends_on:
  - config-xprompt-runtime
  - memory-amd-runtime
- id: enabled-project-home-migration
  title: Enabled projects and chezmoi home migration
  depends_on:
  - config-xprompt-runtime
  - memory-amd-runtime
- id: documentation-refresh
  title: Documentation and migration guidance refresh
  depends_on:
  - editor-and-core-integrations
  - sase-project-migration
  - enabled-project-home-migration
- id: gpt-image-refresh
  title: GPT Image infographic regeneration and review
  depends_on:
  - documentation-refresh
- id: end-to-end-validation
  title: Cross-project migration and regression validation
  depends_on:
  - gpt-image-refresh
create_time: 2026-07-16 12:37:10
status: done
bead_id: sase-6d
---

# Plan: Canonical SASE directories for project and home content

## Context and outcome

SASE already reserves `<root>/sase/repos/` for workspace-scoped linked, sidecar, and external repository checkouts. The
rest of SASE's project-owned material is currently scattered across root-level `sase.yml`, `xprompts/` or `.xprompts/`,
and `memory/` paths. Home-managed material is similarly split between `~/.xprompts/`, `~/xprompts/`, `~/memory/`, and
chezmoi source equivalents. Consolidate these into one visible `sase/` namespace while keeping unrelated SASE state and
package resources stable.

The canonical layout is:

| Scope                      | Canonical path             | Notes                                                                                        |
| -------------------------- | -------------------------- | -------------------------------------------------------------------------------------------- |
| Project config             | `<project>/sase/sase.yml`  | Replaces only project-local `./sase.yml`.                                                    |
| Project xprompts/workflows | `<project>/sase/xprompts/` | Includes Markdown, YAML workflows, and `steps/`.                                             |
| Project memory             | `<project>/sase/memory/`   | Includes flat notes, generated README, and assets.                                           |
| Project repo checkouts     | `<project>/sase/repos/`    | Already canonical and remains runtime/ignore-managed.                                        |
| Home xprompts/workflows    | `~/sase/xprompts/`         | Chezmoi source is `home/sase/xprompts/`; project-specific children may live under this tree. |
| Home memory                | `~/sase/memory/`           | Chezmoi source is `home/sase/memory/`.                                                       |

The global user configuration remains `~/.config/sase/sase.yml` (and `sase_*.yml` overlays). Package-owned
`src/sase/xprompts/`, `src/sase/default_xprompts/`, plugin `xprompts/` resources, `~/.sase/` state, and SDD/repository
checkout semantics are not moved. Keeping those distinctions explicit prevents mechanical replacements from corrupting
package resources, URLs, API routes, historical changelog text, or plugin contracts.

The enabled-project inventory at planning time is `actstat`, `bob-cli`, and `sase`; all three are launchable. The hidden
system-managed home project is also in scope through the audited `chezmoi` linked repository. Maintained linked-repo
inspection found implementation changes in `sase-core` and path/schema changes in `sase-nvim`; `sase-github` only uses
the unchanged global config path, and `sase-telegram` has no affected references.

## Compatibility and migration policy

- Centralize the layout in a shared path contract instead of duplicating `root / "sase" / ...` throughout Python, Rust,
  TUI, and integration code. Use the Rust core for shared catalog/domain behavior and thin Python adapters for
  filesystem and chezmoi concerns.
- New reads prefer the canonical path. During a documented compatibility window, legacy project/home paths remain read
  fallbacks so disabled or third-party projects do not fail immediately. If canonical and legacy config or memory trees
  coexist, report a clear collision instead of merging split-brain state. Xprompt collisions follow one documented
  first-wins order with canonical locations ahead of the corresponding legacy locations.
- All creation, save, edit, init, proposal-approval, and generated-file flows write canonical paths only. Existing
  legacy sources may be displayed as legacy/read-compatible, but normal UI and CLI destinations must not create new
  legacy content.
- Preserve the public `sase memory read <note-relative-path>` argument contract: callers still pass a path relative to
  the selected memory root. User-facing inventories, generated `AGENTS.md`, parent references, and diagnostics render
  canonical `sase/memory/...` or `~/sase/memory/...` paths after migration.
- Make migration reviewable through existing init planning/check flows where practical: identify moves separately from
  generated rewrites, create parent directories safely, preserve file contents and history, and make repeat runs
  idempotent. Do not silently delete a non-identical legacy tree.

## Phase 1: Canonical layout contract and compatibility policy

Introduce named project/home layout resolvers and a single precedence/collision policy used by config, xprompt,
workflow, memory, AMD, init, and catalog callers. Define canonical, legacy, source-controlled, generated, and
runtime-only paths separately so `sase/repos/` remains ignored while sibling `sase/sase.yml`, `sase/xprompts/`, and
`sase/memory/` remain trackable.

Specify and test the transition rules, including missing roots, commands launched below a repository root, missing or
deleted CWDs, chezmoi source remapping, path shortening/display, symlinks, both-path collisions, and legacy-only
projects. The xprompt priority specification must cover Markdown, YAML workflows, shared `steps/`, global home entries,
project-specific home entries, config-backed prompts, plugins, and package defaults without changing package/plugin
contracts accidentally.

## Phase 2: Project config and xprompt runtime migration

Route every project-local config read through the canonical resolver: merged-config cache tokens/layers, xprompt source
attribution, cross-project catalog loading, mentor scoping, provider/VCS selection, linked-repo inventory, SDD
management authorization, repo/memory init opt-in, doctor checks, and any direct registry or helper reads. Keep the
global config and overlays at their current XDG paths. Update config editing, snippets, Config Center, xprompt browser
source labels, source-to-file resolution, and save destinations so project-local operations target `sase/sase.yml`.

Update file-backed xprompt and workflow discovery to use project `sase/xprompts/` and home `~/sase/xprompts/`, including
project-specific home children and `steps/` imports. Apply the same ordered sources to CLI expansion, ACE completion and
browser catalogs, workflow validation, prompt export/save, source classification, jump/preview behavior, and the xprompt
LSP environment. Retain explicit legacy read coverage and collision tests, but make all write and copy-on-edit defaults
canonical.

## Phase 3: Memory and agent-document runtime migration

Move the memory model from `<root>/memory/` to `<root>/sase/memory/` across note discovery, flat-note validation,
frontmatter parents, inventory reachability, audited reads/logs, proposal review/approval, display paths, README and
asset generation, dirty-path classification, commit ownership, and `sase memory init`. Update AMD parsing/rendering so
managed root instruction files and provider shims refer to `sase/memory/...`, while legacy root documents can still be
inspected during the compatibility window.

Teach home and chezmoi planning to write `~/sase/memory/` and `home/sase/memory/`. Ensure generated project and home
`AGENTS.md` files remain at their provider-discovered roots, only their memory references change. Cover both managed and
custom agent documents, nested long-memory parents, root and home inventories, identical generated assets, opt-in
creation of `sase/sase.yml`, `--check`/preview behavior, and safe migration when old and new trees coexist.

## Phase 4: Rust catalog and editor integration alignment

Update `sase-core`'s shared xprompt catalog and editor metadata to resolve project config and workspace xprompts through
the canonical layout and to display canonical memory paths. Preserve global config and package resource paths. Update
wire fixtures, Rust parity/catalog tests, gateway metadata fixtures, definition/hover/completion paths, and bindings so
Python, gateway, and future frontends agree.

Update `sase-nvim` YAML schema globs, xprompt Markdown/YAML detection, alternate-edit support, LSP smoke fixtures, and
README examples for `**/sase/sase.yml` and `**/sase/xprompts/**`, retaining intentionally documented legacy matches for
the compatibility window. Confirm that `sase-github` and `sase-telegram` need no edits beyond recording the audit
result; do not rewrite their valid references to global config or plugin package resources.

## Phase 5: SASE project-local content migration

Use history-preserving moves in the main SASE repository:

- `sase.yml` to `sase/sase.yml`;
- repository-maintenance `xprompts/` to `sase/xprompts/`;
- project memory and its copied directory-map asset to `sase/memory/`.

Update path-bearing workflow bodies, tests, fixtures, lint/symvision pragmas, config mentor globs/text, docs build
links, and generated root instruction/provider files. Do not move `src/sase/xprompts/`, `src/sase/default_xprompts/`,
`src/sase/memory/`, or `sase/repos/`. Ensure `.gitignore`, packaging, schemas, CI path filters, and repository-init
logic allow the three tracked canonical children while continuing to exclude only runtime repo checkouts.

## Phase 6: Enabled projects and chezmoi home migration

Migrate every currently enabled project, using audited repo access for non-current repositories:

- `actstat`: move project `sase.yml` and `memory/` into `sase/`, then regenerate managed instructions/assets.
- `bob-cli`: move project `sase.yml` and `memory/` into `sase/`, update the `cli_rules.md` references in generated root
  instructions, then regenerate managed instructions/assets.
- `sase`: consume the Phase 5 migration and verify its lifecycle workspace resolves the canonical files.
- `chezmoi` as a SASE project: move its own root `sase.yml` and project memory into `sase/`.
- Chezmoi-managed home: move `home/dot_xprompts/` to `home/sase/xprompts/` and `home/memory/` to `home/sase/memory/`;
  keep `home/dot_config/sase/sase.yml` and overlays unchanged. Regenerate/apply home instruction files and provider
  skills from their authoritative sources rather than hand-editing generated outputs.

Run each repository's relevant generation/check workflow after its moves and verify that no enabled project is relying
on the legacy fallback. Preserve unrelated user changes and keep cross-repository commits/rollouts independently
reviewable.

## Phase 7: Documentation and migration guidance refresh

Update the authoritative path and precedence descriptions in README/docs, especially configuration, xprompt,
workflow-spec, memory, init, development, architecture, ACE save/browser behavior, prompt export/save, getting-started,
agent-family, SDD storage, and relevant blog examples. Clearly distinguish canonical project/home paths from unchanged
global config, plugin/package resources, `.sase` state, and legacy compatibility. Add a focused migration note with
before/after examples, collision behavior, write policy, and the compatibility/removal timeline; update CLI help,
diagnostics, templates, long-memory notes, generated memory READMEs, and provider instructions from source.

Audit all tracked references rather than blind-replacing strings: historical changelog entries, API `/xprompts` routes,
plugin resource paths, `src/sase/...` package paths, and third-party examples may remain valid. Refresh `sase-nvim`
documentation alongside the main docs. Run documentation link/build checks before image work so image labels are derived
from settled, authoritative prose.

## Phase 8: GPT Image infographic regeneration and review

Use the `imagegen` skill and GPT Image to regenerate the text-free structural bases for both affected documentation
assets, then apply deterministic local labels so all technical paths are exact:

- `docs/images/xprompt-resolution-infographic.png`: replace the stale discovery stack with the final canonical and
  compatibility ordering, remove already-obsolete dynamic-memory/keyword material, keep workflow and fan-out semantics
  aligned with `docs/xprompt.md`, update its prompt/critique sidecars, and embed the reviewed asset where the docs
  specify.
- `src/sase/memory/assets/memory-directory-map.png`: relabel sources and generated README paths as `sase/memory/...`,
  update the prompt/post-processing record, and propagate the byte-identical generated copy into the canonical memory
  asset path for the SASE project, the other enabled projects, and chezmoi-managed home output.

Inspect generated images at full resolution for legibility, cropping, arrow correctness, label contrast, and absence of
model-generated pseudo-text. Compare every label against the final docs/runtime order, keep prompt and critique
provenance beside the assets, and use exact-byte or documented deterministic comparisons for copied memory assets.

## Phase 9: Cross-project migration and regression validation

Validate the complete system from canonical-only fixtures, legacy-only fixtures, and deliberate collision fixtures.
Exercise config merge/provenance/cache behavior, xprompt/workflow resolution and `steps/`, ACE browse/save/edit paths,
cross-project prompt catalogs, LSP/editor definitions, memory init/list/read/write/review, AMD/provider generation,
repo/SDD authorization, doctor output, chezmoi remapping/apply, and Rust gateway/catalog parity.

Run `just install` and `just check` in the main SASE repository, the appropriate Rust checks in `sase-core`, the
documented checks in `sase-nvim`, and relevant checks in each changed enabled project/chezmoi repository. Run visual and
documentation checks in proportion to changed snapshots/assets. Finally:

- confirm `sase project list --json` still reports `actstat`, `bob-cli`, and `sase` enabled and launchable;
- run init/check and representative config, xprompt, workflow, and memory commands from each enabled project root;
- verify home behavior after `chezmoi apply` without modifying the unchanged global config location;
- search every changed repository for stale operational references, allowing only explicit compatibility tests,
  migration guidance, historical text, and unchanged package/plugin/global paths;
- verify repeat init/check runs are clean and no canonical/legacy duplicate tree remains.

Record the compatibility policy and any intentional residual legacy matches in the final handoff so a later removal can
be planned without rediscovering the migration boundary.
