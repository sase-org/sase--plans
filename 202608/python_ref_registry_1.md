---
tier: tale
title: Build the Python artifact-reference renderer registry
goal:
  SASE exposes every known artifact kind as a validated contextual ref xprompt with
  sidecar-owned filters, deterministic renderer provenance, and matching catalog
  metadata.
size: medium
proposed_by: bbugyi200.athena.sase-ho.2
bead: sase-ho.2
create_time: 2026-08-08 15:02:23
status: wip
---

- **PARENT:**
  [202608/artifact_reference_xprompts.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_reference_xprompts.md)
- **BEAD:**
  [sase-ho.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ho/sase-ho.2.md)

# Build the Python artifact-reference renderer registry

## Goal

Consume the shared artifact-reference contract from `sase-core`, expose every known
artifact kind as one contextual `ref/<kind>` xprompt, configure document-sidecar
renderers and path filters without duplicating Rust policy, and carry the resulting
metadata consistently through Python catalogs and refresh paths.

This tale implements phase `python-ref-registry` of the approved Artifact reference
xprompts epic. It deliberately stops before rewriting `#ref/` calls into artifact
tokens, rendering resolved artifacts through templates, or unifying the `@` and `#ref/`
payload-completion menus; those belong to the dependent phases.

## Verified starting point and prerequisite gate

- Linked `sase-core` commit `4071bf0` contains content-layout schema 5, artifact wire
  schema 4, artifact context schema 1, document-root `path_globs`, filtered resolution,
  the batch path-filter binding, contextual ref placement helpers, canonical ref catalog
  inputs, and `ref_kind` editor/catalog metadata.
- The highest published `sase-core-rs` version is still `0.20.1`, while commit `4071bf0`
  is after tag `v0.20.1`. The installed `0.20.1` extension lacks the new schema and
  filter bindings, so the phase cannot truthfully raise its dependency floor or pass a
  published-wheel verification yet.
- Recheck the package index immediately before dependency work. Use the linked core
  checkout for local development only. Do not point released metadata at a Git commit,
  claim compatibility with `0.20.1`, publish anything, or close the phase until a
  release containing `4071bf0` is consumable. Once it exists, raise the floor and
  regenerate the lock atomically, then verify both the linked-core development install
  and a clean published-wheel install. If publication is still missing after the code is
  otherwise ready, leave the bead open and retain the already-recorded proposed
  follow-up rather than weakening this gate.

## Contract and implementation decisions

1. Rust remains the authority for content placement, `ref/` namespace validity, artifact
   parsing/resolution, glob compilation and matching, canonical inputs in the native
   catalog, and filtered resolution diagnostics. Python may preserve and pass filter
   values, but must not implement or approximate their semantics.
2. A sidecar ref setting is typed as an optional renderer body plus optional
   `path_globs`. Preserve three distinct filter states: omitted means the document
   default `("**/*.md",)`, an explicit empty list means allow nothing, and a non-empty
   list is passed through unchanged. The effective filter comes from sidecar config even
   when renderer prose is won by a higher-precedence file.
3. Built-in artifact metadata is registry-owned and immutable:

   | kind                  | argument      | type    |
   | --------------------- | ------------- | ------- |
   | `commit`              | `commit`      | `line`  |
   | `chat`                | `file_path`   | `path`  |
   | `bug`                 | `bug`         | `line`  |
   | `file`                | `artifact_id` | `line`  |
   | `bead`                | `bead_id`     | `word`  |
   | `agent`               | `agent_name`  | `agent` |
   | document sidecar role | `file_path`   | `path`  |

   The `beads` and `agents` sidecars customize singular `bead` and `agent`; they never
   add plural kinds and reject filters. `plans` and enabled custom sidecar roles are
   document kinds. Disabled roles contribute neither a generated renderer nor a document
   root.

4. Ref files derive their kind from the filename and must declare `ref: true`.
   Renderer-declared `name`, `input`, skills, memory metadata, snippets, and local
   helpers cannot replace registry metadata. Ordinary Markdown, YAML, config, workflow,
   skill, and memory sources are checked for illegal `ref:` declarations and reserved
   `ref/` names so load order cannot make a misplaced definition valid.
5. Renderer winner order is project ref file, explicit sidecar config renderer, home and
   project-specific-home ref files in the core-provided order, plugin ref files,
   packaged built-in file, then generated document-sidecar default. Load all applicable
   candidates deterministically, retain winner provenance and shadowed-source metadata,
   and report same-precedence duplicates or invalid candidates as source-aware load
   issues. An unknown file-backed kind is rejected against the effective registry; a
   file cannot register a resolver.
6. Every effective entry has `name=ref/<kind>`, `kind=ref`, `ref_kind=<kind>`, exactly
   one synthesized required input, source provenance, optional sidecar role, filter
   summary for document kinds, and shadow metadata. Propagate `ref_kind` and the
   additive provenance fields through `XPrompt`, converted `Workflow`, structured
   catalog records, CLI/helper projections, and ACE assist models. Existing entries and
   payloads that omit the new optional fields remain valid.
7. Package six `src/sase/xprompts/refs/*.md` renderer sources for the existing built-in
   kinds. Lock their bodies to the current replacement text contract with exact-text
   fixtures, while leaving actual late rendering to the next phase. A path-backed
   sidecar without an override synthesizes the exact body
   `the {{ file_path }} file in the {{ sidecar }} sidecar repo`.
8. Catalog discovery must be contextual and read-only. Resolve selected registered
   project roots without cloning sidecars or allocating project state. Add project,
   home, project-home, and plugin ref roots plus effective sidecar config to existing
   off-thread prompt-catalog source tokens and watch paths. Continue to serve the warm
   catalog on keystroke paths; no stat, glob, config load, repository materialization,
   or catalog rebuild may move onto the Textual event loop.

## Implementation steps

### 1. Consume and guard the new core wires

- Raise `sase-core-rs` only after the release gate above succeeds and update `uv.lock`
  in the same change.
- Extend the content-layout Python mirror with `refs` on project/home/chezmoi layouts,
  typed `RefSource` records, and a ref-source resolver. Add thin adapters for
  `ref_reference_name`, ref placement/reserved-namespace checks, the artifact context
  schema version, the path-filter schema version, and batch filtering where later Python
  callers need it.
- Update the artifact models to emit context schema 1, preserve optional-versus-empty
  `path_globs` on document roots, accept the `filtered` status and diagnostic, and pin
  artifact wire schema 4/list schema 2. Error messages must name an old installed
  extension clearly instead of failing later on missing keys.
- Extend `tools/validate_sase_core_rs` and its tests with all newly required binding
  names and schema probes so a stale `0.20.1` extension is rejected during setup.

### 2. Parse and validate effective sidecar ref configuration

- Add `ref.xprompt` and `ref.filters.path_globs` to the public config schema and default
  examples without adding a renderer override by default.
- Introduce focused typed normalization helpers beside linked/sidecar config assembly.
  They must merge layered sidecar mappings normally, preserve an explicit empty glob
  list, apply the Markdown default only to document roles, map `beads`/`agents` to the
  singular kinds, and expose stable renderer/filter provenance to the registry.
- Extend `config.repos` doctor checks with actionable diagnostics for malformed ref
  mappings, invalid renderer/glob types, filters on `beads` or `agents`, and any role
  shape that schema validation cannot explain after merged config is loaded.
- Attach the effective path globs to every enabled document root assembled from split
  sidecar storage, configured custom roles, implicit managed `plans`, and fallback plans
  roots. Never attach document filters to bead or agent namespaces.

### 3. Build contextual ref discovery and synthesis

- Add a dedicated ref loader/registry module modeled on the existing skill and memory
  loaders, consuming only core-provided ref source records and placement decisions.
- Load filesystem and plugin `refs/*.md` definitions, then interleave config-generated,
  packaged, and generated-default candidates at the specified precedence. Synthesize
  canonical inputs and metadata from the registry rather than frontmatter.
- Integrate effective refs into default/current/selected-project xprompt loading and
  direct project catalog collection without project-prefixing `ref/<kind>` or leaking
  refs from unrelated enabled projects.
- Centralize two-way ref placement checks across ordinary Markdown, plugin, config,
  workflow, skill, memory, and canonical ref loaders. Record deterministic load issues
  for missing `ref: true`, declarations outside `refs/`, reserved names, invalid stems,
  unknown kinds, and conflicting metadata.
- Track winning and shadowed candidates separately from filter ownership so wording
  overrides cannot alter the configured path policy.

### 4. Carry ref identity through catalogs and refresh machinery

- Add optional ref metadata to xprompt/workflow/catalog/ACE assist models and conversion
  functions, render catalog kind as `ref`, and retain source/definition navigation for
  physical files or the owning config location for config-generated entries.
- Update structured catalog serialization, stats/classification, CLI list/show models,
  helper bridges, and native-parity fixtures to include additive `ref_kind`, sidecar,
  filter summary, and shadow provenance where those surfaces already expose catalog
  metadata.
- Export packaged/plugin ref directories to the native xprompt LSP environment using the
  binding's `SASE_REF_BUILTIN_DIR` and `SASE_REF_PLUGIN_DIRS_JSON` contract. Keep
  dynamic sidecar names in the project artifact context so subsequent completion work
  can bind ref arguments to the same inventory.
- Extend the ACE prompt catalog's off-thread token and watcher roots with all contextual
  ref directories. Config already participates in the token; add tests that renderer
  file edits and enable/disable/config changes invalidate the correct selected-project
  snapshot without introducing event-loop I/O.

### 5. Tests and compatibility fixtures

- Core-adapter tests: schema rejection, content-layout ref fields/sources, placement
  helpers, context schema, `path_globs` omitted/empty/non-empty, filtered resolution,
  diagnostic parsing, and stale-binding guard behavior.
- Config/context tests: valid schema examples, invalid types, document defaults,
  positive/negative/empty preservation, implicit plans and custom roles, singular
  beads/agents mapping, filters rejected for non-document kinds, disabled roles, and
  every fallback document root carrying the same policy.
- Loader tests: project/config/home/project-home/plugin/package/generated precedence,
  winner and shadow provenance, duplicate/placement/unknown-kind diagnostics, immutable
  canonical inputs, no project prefix, no unrelated-project leakage, and renderer
  customization independent of filters.
- Catalog tests: a default catalog has all six built-ins plus enabled document roles;
  research needs no renderer override and gets the exact Markdown filter/default body;
  `kind`, `ref_kind`, input, sidecar, source, definition target, filter summary, and
  shadow metadata survive XPrompt-to-Workflow, structured helper, CLI, and ACE
  projections; physical-source Python/native snapshots agree on name/input/kind/source.
- Asset tests: each built-in packaged renderer is present in the built wheel and its
  expected output fixture remains byte-identical to the existing hard-coded contract.
- Refresh tests: create/edit/delete in each ref source and sidecar config role
  enable/disable changes update source tokens and watched roots while cached reads stay
  on the existing off-thread/coalesced path.

## Verification

1. Reopen the linked core checkout and confirm it still contains `4071bf0` or its
   released descendant. Recheck the package index; satisfy the release gate before
   changing dependency metadata.
2. Run `just install` so this ephemeral checkout builds against the refreshed linked
   core, then run the focused config, content-layout, artifact-reference, xprompt
   loader/catalog, LSP environment, doctor, and ACE prompt-catalog tests while
   iterating.
3. Run `just check-full`, not only `just check`: this phase changes the Rust/Python
   compatibility boundary, config schema, xprompt discovery broadening set, package
   assets, and stable catalog fixtures.
4. Build wheel and sdist artifacts and inspect them to prove all six ref Markdown files
   and schema/default config assets ship.
5. In a clean temporary environment without the linked-core override, install the built
   SASE package against the newly published core wheel, run the stale-binding validator,
   import the new binding names, and execute the focused registry/catalog smoke tests.
   This is the final proof required before closing `sase-ho.2`.

## Acceptance criteria

- Every effective known artifact kind has exactly one contextual `ref/<kind>` entry with
  registry-owned canonical input metadata, deterministic renderer provenance, and no
  ability for a file to register a new resolver.
- Enabled path-backed sidecars default to Markdown-only filters and the exact relative
  path sentence; explicit empty and custom glob lists survive unchanged into every
  document root, and Rust is the only component that decides whether a path matches.
- `beads` and `agents` customize singular built-in kinds and reject document filters;
  disabled sidecars disappear from the effective catalog and context after refresh.
- Project/home/plugin/package/config/generated precedence, shadow diagnostics, source
  navigation, structured helper metadata, and native physical-source catalog metadata
  agree under tests.
- A stale pre-contract extension fails immediately, the dependency floor names the
  released core containing `4071bf0`, clean published-wheel verification passes, and
  `just check-full` is green.
