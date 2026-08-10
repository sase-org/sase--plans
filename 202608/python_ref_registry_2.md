---
tier: tale
title: Build the Python ref registry and sidecar configuration
goal:
  Expose validated contextual ref xprompts and one shared sidecar filter policy across
  Python catalog and artifact contexts.
size: medium
proposed_by: bbugyi200.athena.sase-ho.2
bead: sase-ho.2
create_time: 2026-08-08 15:50:44
status: wip
---

- **PARENT:**
  [202608/artifact_reference_xprompts.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_reference_xprompts.md)
- **BEAD:**
  [sase-ho.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ho/sase-ho.2.md)

# Build the Python ref registry and sidecar configuration

## Goal

Implement phase `python-ref-registry` of the Artifact reference xprompts epic for bead
`sase-ho.2`. Consume the released `sase-core-rs 0.21` reference contract, make enabled
sidecars produce a validated effective `ref/<kind>` renderer catalog, and carry the
configured document filter policy into every Rust artifact-reference context without
duplicating Rust's path-matching behavior.

The result must expose contextual ref xprompts and their canonical inputs/provenance to
Python catalog consumers, ship the built-in renderer definitions needed by the later
rendering phase, and keep ordinary xprompts, skills, memories, artifact resolution, and
completion behavior unchanged except for the newly configured document filters.

## Current ground truth

- The repository already pins the published `sase-core-rs>=0.21.0,<0.22.0` release.
  Python already accepts artifact-reference parse schema 4, context schema 1, list
  schema 2, the `filtered` resolution status, and `path_globs` on document roots.
- The Python content-layout adapter does not yet rehydrate the core's `refs` layout
  paths or ordered `ref_sources`, and the Python xprompt loader has no ref registry.
- `artifact_ref_context()` currently creates unfiltered document roots from the SDD
  store. Sidecar config currently has no `ref` schema or typed normalization.
- The native catalog contract already represents ref entries through `kind: ref`,
  `ref_kind`, and a registry-owned canonical input; Python catalog and helper-bridge
  models do not yet carry the corresponding field.
- Rendering `@kind:` and rewriting explicit `#ref/kind` invocations belong to the later
  `artifact-rendering` phase. Shared TUI/LSP payload completion and watcher behavior
  belong to `reference-completion`; this phase supplies the registry metadata and
  filters those phases consume.

## Contract to preserve

- Physical renderer definitions live only in `sase/refs/`, home/project-specific home
  ref roots, plugin `refs/`, or packaged `src/sase/xprompts/refs/`. Their public name is
  contextual and singular: `ref/<kind>`.
- A ref file must declare truthy `ref: true`; `ref: true` is rejected outside a ref
  source, and ordinary definitions cannot claim `ref/`. The filename stem owns the
  artifact kind and renderer frontmatter cannot redefine the registry-owned input.
- Known kinds are the six Rust built-ins plus enabled path-backed sidecar roles.
  Built-in `beads` and `agents` sidecars customize singular `bead` and `agent`; they
  never create plural aliases. Disabled sidecars contribute no generated entry.
- Renderer precedence is project file, explicit sidecar `ref.xprompt`, home and
  project-specific-home files in core order, plugin files, packaged built-ins, then the
  generated document-sidecar default. Filters are independent of renderer precedence and
  always come from effective sidecar config.
- A path-backed role defaults to `path_globs: ["**/*.md"]` when omitted. An explicitly
  empty list remains empty and therefore allows nothing. `bead` and `agent` reject
  filters instead of silently ignoring them.
- Python only transports filter values and calls released core APIs. Rust remains the
  sole authority for glob compilation, path validity, filtering, resolution,
  canonicalization, and native inventory behavior.
- Ordinary xprompt names/source ordering, memory and skill namespaces, sidecar identity,
  and existing artifact output/staging/consumption behavior remain unchanged.

## Implementation

1. Complete the Python adapter for the released core layout and filter contracts.
   Rehydrate `refs` paths on project/home/chezmoi layouts and ordered `RefSource`
   records from `sase_content_layout`, expose filesystem ref sources and canonical
   `ref/<kind>` helpers through `content_layout.py`, and validate the released
   artifact-context/path-filter schema probes with clear stale-extension failures. Keep
   the adapter thin and test its records against the native payload rather than
   recreating core ordering.

2. Extend sidecar configuration with a typed effective ref policy. Add JSON Schema for
   `ref.xprompt` and `ref.filters.path_globs`, including nonempty string patterns while
   allowing an explicitly empty list. Normalize merged per-role config without losing
   omitted-versus-empty semantics, provide the exact default document renderer and
   Markdown filter in one place, map built-in sidecars to singular kinds, and retain
   source/config-location metadata for diagnostics and definition navigation. Extend
   `config.repos` doctor checks for shapes or policies that the open-map schema cannot
   express, especially filters on `beads`/`agents`.

3. Attach effective filters to artifact document roots. Build the role-to-policy map
   from the same merged project config and sidecar inventory used for repository
   resolution, and pass each path-backed root's `path_globs` into
   `ArtifactRefDocumentRoot`. Cover implicit plans/fallback roots, custom sidecars,
   duplicate roots, unset Markdown defaults, explicit empty filters, disabled roles, and
   the absence of document filters on bead/agent resolution. Add a small typed facade
   over the core batch filter API for bounded Python callers, validating its wire schema
   without reimplementing glob matching.

4. Add a dedicated Python ref loader/registry. Load Markdown definitions from the
   core-provided project, home, project-home, plugin, and package source records; apply
   the two-way placement/reserved-namespace rules with source-aware load issues; reject
   invalid or unknown filename-derived kinds; and resolve first-wins precedence while
   retaining shadowed-source metadata. Insert an explicit sidecar-config renderer at its
   required precedence and synthesize fallback entries for every enabled path-backed
   sidecar. Generate the one canonical input from the registry (`commit`, `file_path`,
   `bug`, `artifact_id`, `bead_id`, or `agent_name`) regardless of file frontmatter.

5. Integrate effective ref entries into every Python catalog read surface. Add
   `ref_kind` and any necessary ref provenance/filter summary fields to `XPrompt`, its
   workflow projection, structured catalog records, CLI `xprompt list/show` records,
   editor/mobile helper JSON, and argument-assist snapshots. Report `kind: ref`, the
   synthesized input, winning source/definition path, sidecar role, and shadowed sources
   without treating ref definitions as skills, memories, or project-prefixed xprompts.
   Keep `#ref/` entries non-rendering in this phase so later work can install the
   explicit rewrite safely.

6. Ship packaged `ref: true` renderer files for `commit`, `chat`, `bug`, `file`, `bead`,
   and `agent` under `src/sase/xprompts/refs/`. Define their bodies against the stable
   primitive renderer context planned for the next phase and lock fixtures that show
   they can reproduce the existing hard-coded substitutions byte-for-byte. Add package
   and plugin ref resource directories to the Python-to-native LSP environment so the
   native catalog sees the same physical defaults and plugin sources.

7. Make refresh and validation behavior deterministic. Include ref directories and
   effective sidecar ref config in any registry/context cache key, clear the relevant
   config caches through existing invalidation paths, and ensure a fresh catalog drops a
   disabled sidecar or observes a changed renderer/filter. Route placement, unknown
   kind, malformed frontmatter, and config-policy failures through existing xprompt
   load-issue/doctor reporting with actionable source paths.

## Tests and verification

- Extend content-layout tests for schema 5 ref paths, ordered source records, contextual
  names, and project/home/project-specific source behavior while preserving existing
  memory and skill cases.
- Add schema/config/doctor tests for default renderer/filter values, explicit custom
  wording, custom positive/negative patterns, explicit empty lists, malformed filter
  values, built-in singular mapping, forbidden bead/agent filters, disabled sidecars,
  and merge/cache behavior.
- Add artifact-context and batch-filter facade tests proving every document root carries
  the effective policy and that Python delegates allowed/filtered decisions to core.
- Add loader/registry tests for every source class, exact precedence, shadow reporting,
  `ref: true` placement in both directions, reserved `ref/`, invalid and unknown kinds,
  ignored renderer-declared inputs, generated defaults, implicit sidecars, and
  config-generated provenance.
- Add catalog/helper/LSP-environment tests for `kind: ref`, `ref_kind`, canonical input,
  source and definition metadata, packaged renderer discovery, and parity with the
  released native catalog representation. Assert ordinary catalog snapshots remain
  unchanged apart from additive null/default ref fields.
- Run `just install` before verification, then focused pytest lanes for content layout,
  sidecar config, artifact refs, xprompt loaders/catalogs, doctor, and LSP launch
  environment. Run `tools/validate_sase_core_rs` to prove the installed extension
  exposes the required v0.21 contracts. Finish with `just check`; if scoped selection
  escalates or reports an unusual selection, run `just check-full`.

## Completion and handoff

Summarize the effective registry API, canonical input mapping, source precedence,
sidecar defaults, and filter transport for beads `sase-ho.3` and `sase-ho.4`. Record
genuinely out-of-scope discoveries only as `PROPOSED FOLLOW-UP:` notes on `sase-ho.2`.
Close only `sase-ho.2` with the concrete focused checks and repository verification that
passed; leave epic `sase-ho` open.
