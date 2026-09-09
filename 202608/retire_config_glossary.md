---
status: done
tier: epic
title: Retire the config glossary
goal: "The strand-backed glossary is the only glossary implementation, ACE exposes it
  through MemoryPane, stale config and CLI surfaces are gone, and documentation and
  generated skills describe the final memory-web model.

  "
phases:
  - id: retire-core
    title: Remove config glossary and legacy command infrastructure
    depends_on: []
    size: medium
    description:
      "retire-core: remove config-backed glossary, CLI, completion, migration, and
      generated-note code while preserving the strand matcher and legacy read-history
      compatibility under memory modules."
  - id: unify-ace
    title: Consolidate glossary browsing and mutation into MemoryPane
    depends_on:
      - retire-core
    size: medium
    description:
      "unify-ace: remove the standalone glossary pane, route glossary navigation through
      MemoryPane, add strand mutation and closure travel there, and retain inert keymap
      compatibility with a doctor warning."
  - id: finish-docs
    title: Finish memory-web documentation and generated skill source
    depends_on:
      - retire-core
      - unify-ace
    size: small
    description:
      "finish-docs: document the final strand-only glossary and two-axis memory model,
      update the generated sase_memory_read skill source, regenerate managed memory
      output, and prepare final verification."
proposed_by: bbugyi200.athena.sase-sq.8
parent_bead: sase-sq.8
bead_id: sase-sq.8.1
create_time: 2026-09-09 19:51:26
---

- **PROMPT:**
  [prompts/202608/retire_config_glossary.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/retire_config_glossary.md)
- **PARENT:**
  [202608/memory_webs.md](https://github.com/sase-org/sase--plans/blob/main/202608/memory_webs.md)
- **BEAD:**
  [sase-sq.8.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-sq/sase-sq.8.1.md)

# Plan: Retire the config glossary

## Context and invariants

The preceding memory-web phases have migrated the project glossary to a user-owned
`sase/memory/glossary.md` descriptor with sibling strand files. This plan removes the
one-release compatibility implementation now that no supported project should read
`memory.glossary` or invoke `sase glossary`.

The following invariants apply across all phases:

- Keep the Rust-backed phrase matcher and editor highlighting behavior. The Rust module
  and the thin `sase.core.glossary_facade` adapter remain valid implementation names;
  only the config-backed Python package and user-facing glossary product surface retire.
- Preserve read access to historical `glossary_reads.jsonl` events through
  `sase memory log --include glossary` and ACE's historical agent-context display, but
  relocate that compatibility code out of `sase.glossary`.
- A core web still inlines only the descriptor body. Strand bodies must never enter
  generated agent instruction documents.
- All TUI disk reads and writes remain worker-backed and off the Textual event loop.
  Selection is restored by stable `web:strand` identity after mutation or refresh.
- `ace.keymaps.glossary` is accepted but inert for one release. The bundled defaults and
  active Glossary-pane bindings disappear, and `sase doctor` tells users to move any
  remaining customization to `ace.keymaps.memory`.
- Do not deploy the generated skill from a dirty or unlanded source tree. The source
  phase previews its render; the child epic's land agent deploys only after the
  host-owned phase commit is present on the canonical branch.

## Phase 1: Remove config glossary and legacy command infrastructure

Delete the config-backed implementation and make the memory web the sole catalog source.

### Domain and compatibility relocation

- Delete `src/sase/glossary/` and `src/sase/glossary_config.py`.
- Move the generic normalization and closure records still used by memory-web lookup,
  mention closure, prompt highlighting, and selector rendering into focused modules
  under `src/sase/memory/web/`; update all consumers to import from the memory domain.
- Move the v1 glossary-read event parser and report materializer into explicitly named
  legacy modules under `src/sase/memory/`. Keep historical JSONL parsing and ACE report
  links working, but do not retain append/write APIs for the retired command.
- Simplify `src/sase/xprompt/glossary_catalog.py` to discover and compile only the
  `glossary` memory web. Delete `_glossary_catalog_config.py` and
  `_glossary_catalog_ranges.py`; keep file-backed source ranges so LSP go-to-definition
  still opens the strand file.
- Remove the dual-source diagnostic and migration hint from the web catalog and doctor
  checks. Delete the one-shot `sase memory web migrate glossary` handler and parser
  surface now that the legacy schema is gone.

### CLI, config, completion, and memory init

- Remove `parser_glossary.py`, `glossary_handler.py`, parser registration, entry
  dispatch, completion command-path overrides, the `ValueKind.GLOSSARY` provider, and
  its config-mtime candidate source. Extend memory-selector completion as needed so
  `glossary:<strand>` is completed from the memory directory rather than `sase.yml`.
- Drop `glossaryEntry` and `memory.glossary` from `src/sase/config/sase.schema.json`,
  and remove `glossary` from the project-only config-key set. Preserve the separate
  `ace.keymaps.glossary` compatibility object for one release.
- Remove `main/init_memory/glossary.py`, the packaged generated-glossary template, and
  every `ProjectGlossaryTerms`, generated-glossary-body, retired-note, or reserved-path
  branch in memory init. `glossary.md` must be treated as a user-owned web descriptor,
  not a generated note.
- Replace any generic import of `MEMORY_CONFIG_KEY` from the retired module with a
  memory-owned constant or local literal so non-glossary memory configuration remains
  unchanged.
- Update or replace affected tests to prove: `sase glossary` and the migration command
  are absent; config schema rejects `memory.glossary`; memory init preserves the
  descriptor and roster; LSP sources point to strand files; memory selector completion
  invalidates from memory files; v1 glossary events remain readable.

## Phase 2: Consolidate glossary browsing and mutation into MemoryPane

Remove the duplicate ACE product surface and finish the existing web-aware MemoryPane.

### Pane consolidation

- Delete `ace/tui/glossary_catalog.py`, `glossary_panel_catalog.py`,
  `glossary_reads.py`, the standalone `modals/glossary_*` pane/preview family, their
  modal exports, TCSS blocks, Config-hub Glossary subtab, and glossary visual snapshots.
- Route the existing prompt glossary shortcut/highlight action to the Config hub's
  Memory subtab, seeded with the resolved `glossary:<slug>` identity. Reuse the Memory
  prompt-focus restore path and remove the duplicate Glossary prompt-bar mixin where
  practical.
- Keep prompt-time semantic highlighting and go-to-definition backed by the strand-only
  editor catalog; those are editor features, not a separate pane.
- Use MemoryPane's existing web expansion, strand preview, audited strand read, scope
  ring, next/previous strand, and link-trail mechanics for glossary terms. For a
  `closure: mentions` web, relation chips must navigate the same closure graph the CLI
  reads.

### Strand writes and keymap compatibility

- Add a generic memory-web strand mutation engine under `sase.memory.web` with atomic
  create/delete, containment checks, digest conflict protection, frontmatter validation,
  catalog ambiguity validation, and descriptor-roster refresh through the normal
  memory-init/publish path.
- Branch MemoryPane's add/delete actions on the selected row: a web row can add a strand
  and a strand row can be deleted after showing aliases, body, source path, and reverse
  mention references. Keep ordinary note add/edit/delete behavior unchanged. Run writes
  through tracked session workers, invalidate the mtime cache, reload the snapshot, and
  restore the nearest stable identity.
- Fold all still-useful Glossary-pane defaults into the existing Memory keymap actions;
  remove the bundled `glossary` defaults and active binding builders/types when they are
  no longer referenced. Continue accepting the schema block as inert compatibility.
- Add a `sase doctor` warning when any loaded config explicitly sets
  `ace.keymaps.glossary`, with a next step naming `ace.keymaps.memory`.
- Replace Glossary-pane tests with MemoryPane coverage for prompt seeding, web
  expansion, mention-relation travel, add/delete success, validation/conflict handling,
  cache invalidation, and event-loop-safe worker execution. Refresh MemoryPane PNG
  goldens only if its rendered UI intentionally changes.

## Phase 3: Finish documentation and generated skill source

Make every supported explanation match the final system.

- Rewrite the relevant sections of `docs/memory.md`, `docs/cli.md`, `docs/editor.md`,
  `docs/completion.md`, and `docs/ace.md`; also sweep `docs/init.md`,
  `docs/configuration.md`, examples, and command snapshots for remaining claims that
  `memory.glossary`, `sase glossary`, the migration command, or a standalone Glossary
  pane still exists.
- State the two independent axes clearly: note/web/strand is memory kind, while
  core/reference is rendering. Explain that descriptors render, strands never inline,
  and `keyword:` is an explicit read-time addressing alias rather than a runtime
  trigger.
- Update `src/sase/xprompts/skills/sase_memory_read.md` to document implemented
  `<web>:<keyword>` reads, batch resolution, whole-web reads, closure behavior, and the
  current core-descriptor/strand distinction. Preview provider output with
  `sase skill init --diff`; do not force-deploy from the phase's dirty tree.
- Run `sase memory init` for this repository to regenerate `sase/memory/README.md`,
  `AGENTS.md`, and provider shims from the updated sources. Confirm the generated README
  describes both axes and the instruction documents contain the glossary roster but no
  strand body.
- Remove obsolete glossary-only tests, fixtures, and CLI completion snapshots; update
  retained snapshots and documentation links deterministically.

## Integration and landing verification

The child epic's land agent must verify the aggregate tree rather than trusting phase
notes:

1. Confirm `rg` finds no importable `sase.glossary`, no config glossary resolver, no
   `sase glossary` parser registration, no migration command, and no standalone
   `GlossaryPane`. Allow only deliberate historical terminology such as the Rust
   matcher, legacy read-log compatibility, and prose that labels an old event format.
2. On the clean, canonical landed tree, run `sase skill init --force` and verify the
   deployed `sase_memory_read` skill provenance matches that commit. Do not bypass dirty
   source or provenance guards.
3. Open the configured `bob-cli` and home/chezmoi roots through the required repository
   workflow. Run `sase memory init --check` for this repository, `bob-cli`, and home;
   resolve all drift in scope and record any truly external problem as a proposed
   follow-up on `sase-sq.8`.
4. Run focused unit and TUI tests while integrating, `just test-visual` when MemoryPane
   rendering changed, and the required exhaustive `just check-full` through
   `/sase_monitor`. Inspect actual visual diffs before accepting golden updates.
5. Run `sase bead epic-symbols sase-sq.8`; resolve every listed symbol or re-key it to a
   still-open bead. Then close only the parent phase with
   `sase bead close sase-sq.8 --note "<what was verified>"`. Do not close `sase-sq` or
   any ancestor plan bead, and run `just symvision` afterward to confirm the whitelist
   is clean.
