---
tier: tale
title: Build the Python artifact-reference renderer registry
goal:
  SASE loads validated contextual ref renderers and sidecar filters into artifact
  resolution and catalog metadata using the released shared core contract.
size: medium
proposed_by: bbugyi200.athena.sase-ho.2
bead: sase-ho.2
create_time: 2026-08-08 14:44:52
status: wip
---

- **PARENT:**
  [202608/artifact_reference_xprompts.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_reference_xprompts.md)
- **BEAD:**
  [sase-ho.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ho/sase-ho.2.md)

# Plan: Build the Python artifact-reference renderer registry

## Context

Phase `sase-ho.2` supplies the Python assembly layer for the artifact-reference xprompt
design. The completed core phase added content-layout schema 5, artifact
parse/resolution schema 4, artifact-context and path-filter schema 1, `refs` source
records, ref placement helpers, `ref_kind` catalog metadata, canonical ref inputs, the
shared POSIX path filter, and the `filtered` resolution status. The current SASE tree
still models artifact wire schema 3, has no `refs` mirror in its typed content layout,
sends document roots without filter policy, and loads only ordinary xprompts, skills,
and memories.

The published `sase-core-rs 0.20.1` wheel was independently probed and does not contain
the completed core contract even though the linked core repository does. Before updating
Python callers, identify or publish the first release containing the core phase commit,
update `pyproject.toml` and `uv.lock` to that release, and verify the installed
extension reports the exact schema/API contract. Do not point CI at an unreleased
version or weaken compatibility checks to accommodate the old wheel.

This remains one `tale`: configuration, registry loading, catalog projection, and
artifact-context assembly are one cohesive Python feature and should land together so no
intermediate tree accepts ref configuration that it cannot represent.

## Implementation

### 1. Adopt and guard the released core contract

- Raise the `sase-core-rs` lower/upper bounds and refresh the lockfile only after a
  published artifact containing the core ref/filter commit is available.
- Update the Python artifact wire constants and status literals for schema 4 and
  `filtered`. Add explicit startup/operation checks for the parse/resolution,
  artifact-context, path-filter, and content-layout versions so a stale extension fails
  with the expected-versus-actual contract instead of a missing-attribute or
  malformed-record error.
- Extend `core/content_layout_wire.py` and `content_layout.py` with typed `refs` paths,
  ordered `RefSource` records, `resolve_ref_file_sources`, contextual
  `ref_reference_name`, reserved-namespace checks, and source-aware placement issue
  conversion, all delegated to the Rust bindings.
- Extend `ArtifactRefDocumentRoot` with `path_globs: tuple[str, ...] | None`, retain the
  distinction between omitted and explicitly empty policies in `to_wire`, and expose the
  Rust batch-filter adapter for bounded Python candidate lists without reimplementing
  glob semantics.

### 2. Parse and validate effective sidecar ref configuration

- Add typed immutable models/helpers for `repos.sidecar.*.<role>.ref`, with `xprompt` as
  inline Markdown/Jinja text and `filters.path_globs` as an optional ordered tuple.
  Preserve three states: missing filters becomes the Markdown default `("**/*.md",)`, an
  explicit empty list stays empty, and configured patterns stay byte-for-byte ordered.
- Extend `config/sase.schema.json` and `default_config.yml` examples/comments. Reject
  non-string xprompt bodies, non-string glob entries, malformed `ref` or `filters`
  mappings, and filters on `beads` or `agents`. Keep runtime parsing defensive while
  making `sase doctor` surface source/role-aware errors.
- Carry the normalized ref configuration through sidecar entry merging without losing
  inherited nested values. Resolve effective roles from configured, implicit
  managed-project, and materialized-store sidecars; disabled roles must suppress both
  generated catalog entries and artifact roots.
- Attach the role-owned filter policy to every document root, including the plans state
  fallback root. Never attach document filters to bead or agent entity roots. Use
  `("**/*.md",)` for path-backed roles with no explicit policy.

### 3. Implement contextual ref discovery and synthesis

- Add a dedicated ref-loader/registry module rather than folding ref semantics into the
  ordinary xprompt loader. Discover project, home, project-specific home, plugin
  `refs/`, and packaged `xprompts/refs/` definitions in the Rust-declared order; refs
  remain contextual `ref/<kind>` names and are never project-prefixed.
- Parse Markdown with existing frontmatter/body helpers, but derive the kind from the
  filename. Require `ref: true`, reject invalid/unknown kinds, ignore renderer input
  declarations in favor of the registry's canonical single input, and reject local
  helpers/skill/memory semantics on ref entries. Enforce the inverse rule in ordinary
  xprompt/config/plugin/skill loaders so `ref: true` and the reserved `ref/` namespace
  produce recorded source-aware load issues outside a canonical ref source.
- Build one effective renderer for every built-in artifact kind and every enabled
  path-backed sidecar. Map the `beads` and `agents` sidecars to singular `bead` and
  `agent`; allow their inline `ref.xprompt` to override wording but never synthesize
  plural aliases. Apply the required precedence: project file, explicit sidecar inline
  renderer, home/project-home files, deterministic plugins, packaged built-ins, then the
  generated path-sidecar default.
- Track the winning renderer plus deterministic shadowed candidates, source scope,
  definition/config provenance, sidecar role, and effective filter summary. Inline
  config renderers navigate to the owning SASE config; generated defaults retain their
  owning role/config provenance without pretending to have a renderer file.
- Add ref definitions to `get_all_xprompts`/workflow conversion after ordinary sources
  so the reserved namespace cannot collide, and include ref directories, plugin
  resources, effective project identity, sidecar configuration, and enabled roles in
  relevant cache/source tokens. A refresh after disabling a sidecar must remove its
  entry.

### 4. Extend catalog and show metadata without changing ordinary entries

- Extend `XPrompt`, converted `Workflow`, structured catalog sources/entries, CLI show
  records, and editor-helper serialization with authoritative `ref_kind` and ref
  provenance metadata. `ref_kind != None` makes the user-facing `kind` equal `ref`; all
  ordinary, skill, and memory records retain their current values.
- Serialize each entry with the registry-supplied canonical input, source display,
  definition/config target, sidecar, filter summary, and shadow information. Ensure
  project-context catalog gathering includes contextual project refs without duplicating
  them under project-prefixed names.
- Compare Python catalog projections with the native core catalog for common
  file-backed/package entries, asserting parity for names, kinds, canonical inputs, and
  source metadata. Cover Python-only config-generated sidecar metadata with focused
  snapshots.

### 5. Ship compatibility-preserving built-in renderers

- Add packaged `src/sase/xprompts/refs/{commit,chat,bug,file,bead,agent}.md` definitions
  with `ref: true` and bodies matching the current hard-coded replacement text
  byte-for-byte for the documented canonical variable of each kind. Do not move
  rendering into this phase; these files become registry/catalog inputs for the
  artifact-rendering phase.
- Use the exact generated default body
  `the {{ file_path }} file in the {{ sidecar }} sidecar repo` for plans and custom
  path-backed sidecars. Renderer wording overrides must not alter effective filters.

## Regression coverage

- Add wire-adapter tests for every expected schema version, stale-extension error,
  filtered status, ref source record, ref placement diagnostic, omitted/empty/custom
  `path_globs`, and Rust batch-filter delegation.
- Add configuration/schema/doctor tests for valid inline bodies, Markdown defaults,
  custom positive/negative filters, explicit empty filters, nested layer merging,
  invalid shapes, and forbidden filters on beads/agents.
- Add registry tests for every source class, full precedence and shadow reporting,
  contextual project switching, unknown/invalid filename kinds, missing `ref: true`,
  wrong placement in ordinary sources/config/skills, plugin ordering, generated
  defaults, disabled roles, implicit plans/beads/agents, custom sidecars, and the
  singular built-in mapping.
- Add catalog/show/editor-helper tests proving `kind: ref`, `ref_kind`, canonical input
  types/names, provenance/filter/shadow metadata, definition navigation, and
  Python/native parity. Lock the six packaged renderer bodies to existing output
  fixtures and the path-backed renderer to the exact required sentence.

## Verification

1. Before tests, inspect the repository verification configuration, then run
   `just install` so the workspace extension and dependencies match the new pin.
2. Run focused artifact-ref, content-layout, sidecar-config, xprompt-loader, catalog,
   CLI-show, editor-helper, and doctor tests while iterating.
3. Run `just check`; if its scoped selector escalates, reports unusual selection, or the
   dependency/schema changes touch the broadening set, run `just check-full`.
4. Probe the installed `sase_core_rs` version functions and load a representative
   default/custom project catalog to confirm built-ins, `ref/plans`, and `ref/research`
   carry the expected inputs, filters, and provenance while a disabled sidecar is
   absent.

## Boundaries and risks

- Rust remains the sole authority for path grammar, glob matching, filtering,
  canonicalization, and native completion; Python only validates/assembles config,
  renderer sources, and catalog metadata.
- Renderer files customize wording only. They cannot register artifact kinds, change
  canonical inputs, alter filters, or trigger recursive artifact expansion.
- This phase does not implement `#ref/` preprocessing or late artifact rendering, and
  does not change TUI/LSP completion beyond exposing the catalog/context data required
  by their downstream phases.
- The core release is load-bearing. If no published artifact contains the completed core
  contract, stop before code changes and record a `PROPOSED FOLLOW-UP:` on `sase-ho.2`
  for the epic land agent rather than accepting a non-reproducible pin.
