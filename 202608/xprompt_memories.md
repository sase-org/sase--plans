---
tier: epic
title: Xprompt memories and memory namespace invocation
goal: 'Valid SASE memory notes are exposed as an explicit xprompt-memory type under
  the required memory/ namespace, so the active context''s glossary note expands with
  #memory/glossary while bare #glossary remains unresolved.'
phases:
- id: shared-memory-contract
  title: Shared xprompt-memory layout and catalog contract
  depends_on: []
  size: medium
  description: 'shared-memory-contract: define memory source precedence, reference
    naming, type metadata, and native editor/catalog behavior in the Rust core.'
- id: python-memory-runtime
  title: Python discovery and expansion integration
  depends_on:
  - shared-memory-contract
  size: medium
  description: 'python-memory-runtime: consume the shared contract, load valid memory
    notes as contextual xprompts, and enforce memory namespace semantics.'
- id: memory-user-surfaces
  title: CLI, ACE, helper, and editor presentation
  depends_on:
  - python-memory-runtime
  size: medium
  description: 'memory-user-surfaces: expose xprompt-memory identity and refresh behavior
    consistently across every xprompt discovery and authoring surface.'
- id: memory-docs
  title: Memory documentation and glossary regeneration
  depends_on:
  - python-memory-runtime
  size: small
  description: 'memory-docs: document explicit xprompt-memory inclusion, add the glossary
    term, and regenerate managed memory outputs through sase memory init.'
- id: xprompt-memory-verification
  title: Cross-runtime verification
  depends_on:
  - memory-user-surfaces
  - memory-docs
  size: small
  description: 'xprompt-memory-verification: prove contextual precedence, namespaced
    expansion, catalog parity, memory regeneration, and absence of dynamic matching
    end to end.'
proposed_by: bbugyi200.athena.vh.f3
create_time: 2026-08-08 08:49:43
status: wip
bead_id: sase-hf
---

- **PROMPT:** [prompts/202608/xprompt_memories.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/xprompt_memories.md)
- **BEAD:** [sase-hf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-hf/README.md)

# Plan: Xprompt memories and memory namespace invocation

## Context and verified current behavior

SASE memory notes are flat Markdown files selected from the active project's
`sase/memory/` root and then the home `~/sase/memory/` root. Each non-README note
declares `type: short` or `type: long`; the existing memory parser strips frontmatter,
carries `description` and `parent`, excludes nested assets, and treats canonical
`sase/memory/` plus legacy `memory/` as exclusive alternatives within a scope. Project
memory therefore already has the right contextual precedence for a bare `memory/`
xprompt namespace, but neither the Python xprompt loader nor the native Rust catalog
currently includes these notes.

The recently landed xprompt-skill work provides the architectural precedent: Rust owns
special-source layout and canonical reference naming, Python consumes that wire, and
both native and helper catalogs carry explicit type metadata. Skills use `skills/<name>`
for `#` expansion while retaining a separate slash name. Memories need only one callable
name, derived from their canonical flat filename: `sase/memory/glossary.md` becomes
`memory/glossary` and is invoked as `#memory/glossary`.

This is not a restoration of the retired dynamic-memory runtime. Historical SASE code
discovered keyworded `memory/long/*.md` files and automatically injected matched
context. That code was intentionally removed. Xprompt memories are explicit prompt
composition only: no `keywords` field, prompt scanning, automatic matching,
`.sase/memory/` cache, or `### DYNAMIC MEMORY` section returns.

## Contract and compatibility decisions

1. Every valid, flat, non-README SASE memory note with `type: short` or `type: long` is
   an xprompt memory automatically. No new opt-in frontmatter is required, and nested
   files such as `sase/memory/assets/**` are not catalog entries.
2. The xprompt reference is always `memory/<filename-stem>`. A memory note's filename
   remains its identity; xprompt-specific `name`, `skill`, `snippet`, input, workflow,
   and local-helper metadata do not redefine it. Existing xprompt-reference grammar
   still applies, and an invalid or ambiguous memory filename must produce an actionable
   validation/load diagnostic rather than a silently unreachable entry.
3. The `memory/` prefix is mandatory. There is no `#foo` alias for `#memory/foo`, no
   `#memory/long/foo` compatibility form, and no automatic rewrite of old references.
   Reserve the top-level `memory/` reference namespace for this source type so an
   ordinary xprompt, workflow, config entry, plugin, or skill cannot masquerade as an
   xprompt memory; inventory existing definitions before enforcing the diagnostic.
4. Resolution is contextual and first-wins across scopes: the selected active project's
   note shadows the same-named home note. Canonical-versus-legacy coexistence inside
   either scope continues to use memory's existing exclusive collision policy and must
   fail with the established migration diagnostic. Selecting a registered project
   changes which project root supplies `#memory/foo`; it does not prepend the project
   name and does not aggregate unrelated enabled projects into one prompt catalog.
5. Xprompt memory bodies use the ordinary simple-Markdown-xprompt rendering path after
   memory frontmatter is stripped. They take no xprompt arguments and do not synthesize
   the `## Children` section used by `sase memory read`; the explicit reference expands
   the selected file body. Recursive xprompt references already authored in that body
   retain ordinary xprompt behavior.
6. Carry an optional `memory_type` (`short` or `long`) through `XPrompt`, converted
   workflows, structured catalogs, editor/mobile wires, and show/list models. A non-null
   value is the authoritative special-type marker; user-facing `kind` renders as
   `memory`, while `source_bucket` remains available for provenance. Older helper
   payloads that omit the additive field continue to deserialize.
7. `#memory/foo` is a launch-time, explicitly authored inclusion, not an agent-initiated
   audited lookup. Catalog discovery, previews, and expansion do not append
   `sase memory read` events or require a reason. The existing `/sase_memory_read` flow
   remains the required audited path when an already running agent decides to consult
   long-term memory, and continues to reject short notes and append child references as
   it does today.

## Phase 1: Shared xprompt-memory layout and catalog contract

Work in the linked `sase-core` repository, opened through `/sase_repo`.

- Extend the Rust-owned content-layout wire with an ordered memory-xprompt source
  contract derived from the existing project and home compatible memory layouts.
  Preserve project-before-home order, canonical/legacy collision behavior,
  source-control tracking, flat Markdown format, and the lack of plugin/package memory
  sources. Bump the layout schema deliberately and update every binding and
  schema-version assertion together.
- Add one canonical reference helper/constant for `memory/<stem>` and validation for the
  reserved namespace and invokable stem. Keep the selected project contextual rather
  than embedding it into the reference name.
- Teach the native xprompt catalog to parse valid memory-note frontmatter, strip it from
  the prompt body, exclude `README.md` and nested files, preserve description and tier,
  and load memory sources with the same winner/collision behavior the memory subsystem
  already exposes. Invalid note type, split canonical/legacy state, invalid stem, and
  reserved-namespace misuse must become deterministic diagnostics.
- Add backward-compatible `memory_type` metadata to the native catalog model,
  `XpromptAssistEntry`, mobile/editor entry wire, gateway contract, and any stats
  projection that counts special xprompt types. `kind` should be `memory`; memory
  entries must never participate in `/` skill completion.
- Update native completion, hover, definition lookup, diagnostics, source display,
  document eligibility, and catalog invalidation so changes under project or home memory
  roots are visible immediately. Definition navigation should land on the memory note,
  while argument assistance correctly reports no inputs.
- Add Rust tests for both tiers, project-over-home precedence, explicit project
  selection, canonical/legacy collision, README/nested exclusion, invalid stems,
  reserved namespace conflicts, additive wire compatibility, `#memory/foo`
  completion/navigation, and catalog invalidation. Run the core formatter, lints, and
  full test suite.

## Phase 2: Python discovery and expansion integration

Work in the `sase` repository after the shared core contract is available.

- Update the Python content-layout wire adapter for the new schema/source records and
  provide a typed memory-source resolver. Reuse `memory_read_root()` and the canonical
  memory parser rather than independently deriving paths or reparsing the memory
  contract in the xprompt package.
- Add a focused xprompt-memory loader that creates no-argument `XPrompt` objects from
  valid `MemoryNote` bodies, derives `memory/<stem>`, sets `memory_type`, and preserves
  source and description provenance. Load the selected project's notes ahead of home
  notes and resolve an explicitly requested registered project from its known workspace
  without mixing another project's notes into the catalog.
- Integrate memory entries into `get_all_xprompts()`, `get_all_prompts()`,
  `get_all_project_local_prompts()`, direct lookup, recursive expansion, unresolved
  reference suggestions, expansion traces, and workflow conversion. Keep `#glossary`
  unresolved unless a separate ordinary xprompt actually defines it; only
  `#memory/glossary` addresses the memory note.
- Centralize reserved `memory/` namespace validation across Markdown, YAML, config,
  plugin, skill, and memory loaders. Report load issues consistently from
  `sase validate`, `sase doctor`, `sase xprompt list/show`, and helper catalogs; do not
  let load order silently decide whether a colliding ordinary definition or the memory
  note wins.
- Keep prompt expansion explicit and side-effect free with respect to the memory audit
  ledger. Add regression coverage proving that expansion strips memory frontmatter, does
  not append generated child listings, does not create audit events, and does not
  reintroduce keyword/dynamic-memory matching.
- Cover project/home precedence, home fallback, selected-project context, invalid memory
  metadata, filename grammar, collisions, no bare alias, recursive body expansion, and
  xprompt-to-workflow metadata propagation. Run `just install` before repository
  verification.

## Phase 3: CLI, ACE, helper, and editor presentation

- Carry `memory_type` through the Python structured catalog, mobile helper bridge, ACE
  assist models, catalog PDF/JSON, xprompt browser, completion rows, preview,
  jump/definition targets, syntax highlighting, and test fixtures. Present these entries
  as `memory` (optionally `memory · short` or `memory · long`) rather than as an
  ordinary xprompt or a skill.
- Update `sase xprompt list` and the schema-versioned `sase xprompt show` model to emit
  the canonical `#memory/foo` reference, `kind: memory`, `memory_type`, source path, and
  editable definition. Bump stable output schemas only where the repository's
  additive-field policy requires it, and preserve deserialization of older helper
  results.
- Include the selected project and home memory roots in ACE prompt-catalog source
  tokens/watch paths and in LSP watched-file invalidation. Editing, creating, renaming,
  or deleting a memory note should refresh completion without restarting ACE or the
  editor.
- Keep memory authoring on the existing `sase memory write` plus human review path. The
  generic xprompt/skill save modal must identify memory entries for navigation but must
  not offer `sase/memory/` as an ordinary xprompt destination or bypass memory
  proposal/review rules.
- Add unit and integration tests for helper/native parity, completion insertion, badges,
  show/list JSON, source refresh, definition navigation, filtering, and absence from
  slash completion. Update visual snapshots only after inspecting actual/expected diffs,
  and run `just test-visual` if rendered ACE output changes.

## Phase 4: Memory documentation and glossary regeneration

- Update the public memory, xprompt, content-layout, editor, initialization, and
  architecture documentation to define xprompt memories, the `#memory/` namespace,
  contextual project/home precedence, and the distinction between explicit prompt
  composition and audited agent-side long-memory reads. Explicitly state that this
  feature does not restore dynamic keyword matching.
- Add this concise entry to the canonical `sase/memory/glossary.md` note near the other
  xprompt terms:

  **XPrompt Memory**  
  A flat SASE memory note exposed as a namespaced xprompt: `sase/memory/foo.md` expands
  with `#memory/foo`, and the `memory/` prefix is required.

- Update the canonical long-term `sase/memory/xprompts.md` note so future agents see the
  memory source and naming rule in the discovery contract. The user's explicit request
  to update SASE memory supplies the required permission for these canonical edits.
- Run `sase memory init` after the canonical edits, as required, to regenerate
  `AGENTS.md`, provider instruction shims, and memory README outputs. Review the
  generated diff and verify a second `sase memory init --check` is clean; do not
  hand-edit generated instruction files.

## Phase 5: Cross-runtime verification

- Exercise the public contract with representative project and home fixtures:
  `#memory/glossary` expands the frontmatter-free glossary body, `#glossary` remains
  unresolved, a project note shadows a same-stem home note, home fallback works, and
  selected-project catalogs never leak another project's memory.
- Compare Python-helper and native-Rust catalog results for name, kind, tier,
  description, provenance, completion, hover, definition, source refresh, invalid source
  diagnostics, and collision behavior. Test a helper outage so the native fallback
  cannot regress to an ordinary-xprompt view.
- Re-run a negative dynamic-memory sweep: no keyword matching, prompt scanning,
  generated dynamic cache, automatic injection, historical `memory/long/<name>`
  reference, or memory tag is restored.
- Run the full `sase-core` checks. In `sase`, run `just install` and `just check-full`
  because the change touches the shared Rust binding, discovery broadening set, stable
  catalog wires, and managed instruction files; run `just test-visual` when applicable.
  Finish with `sase doctor`, `sase validate`, and `sase memory init --check`.

## Acceptance criteria

- A valid flat `sase/memory/foo.md` or home equivalent is represented as a typed xprompt
  memory and expands only through `#memory/foo`; no bare-name alias exists.
- The selected project's note wins over home, canonical/legacy memory collisions remain
  errors, and unrelated registered projects' memory is not merged into the active
  catalog.
- Python and native Rust loaders, CLI, ACE, LSP/editor, mobile/helper catalogs, and
  definition navigation agree on `kind: memory`, `memory_type`, source, and reference
  name; memory entries never appear as slash skills.
- Expansion includes the memory body without YAML frontmatter or synthetic children,
  creates no audited-read event, and does not restore any dynamic-memory behavior.
- The glossary and xprompt memory notes are updated, all generated instruction files are
  refreshed through `sase memory init`, a check run is idempotent, and all full
  verification lanes pass.
