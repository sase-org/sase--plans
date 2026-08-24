---
tier: tale
title: Memory web and strand substrate
goal:
  Provider-backed memory webs validate and render safely behind a beta flag without
  exposing strand bodies to agent context.
size: medium
proposed_by: bbugyi200.athena.sase-sq.2
bead: sase-sq.2
---

- **PARENT:** [202608/memory_webs.md](memory_webs.md)
- **BEAD:**
  [sase-sq.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-sq/sase-sq.2.md)

# Memory web and strand substrate

## Goal

Complete phase bead `sase-sq.2` by adding the provider-backed memory-web domain,
fail-closed validation, managed strand rosters, and memory-init/doctor integration
behind the `memory_webs` beta flag. Preserve the existing flat memory-note inventory:
descriptors remain ordinary top-level notes, while strand bodies never enter generated
agent documents.

## Implementation

1. Scaffold the `memory_webs` beta flag through `sase flag new`, add its generated
   registry entry, schema/default synchronization, and a small consumer helper. Keep an
   explicit disabled path where `web: true` is ignored and no roster update is planned,
   and an enabled path where web discovery and validation run. The dedicated flag bead
   is implementation metadata required by the approved design, not discovered follow-up
   work.
2. Add `sase.memory.web` modules for immutable web/strand models, YAML frontmatter
   parsing with the approved defaults and opaque `metadata`, provider-based file
   discovery, project-over-home per-strand scope merging with origin tracking, ordered
   lookup through slug/keyword/alias/normalized-prefix, managed inline/list roster
   rendering, and a shared validation report containing blockers and warnings.
3. Implement all fail-closed validation rules from the epic design: local identity and
   alias ambiguity/collisions, orphan or mismatched descriptors/directories, registered
   artifact-kind plus `assets`/`README` reserved names, nested directories, symlink
   escape, malformed descriptor/strand frontmatter, required list summaries, and
   duplicated/unbalanced managed-region markers. Report project/home keyword collisions
   as warnings while retaining project-wins per-strand resolution.
4. Wire enabled web discovery and validation into `memory_root_context`. Convert valid
   user-owned descriptor roster regions into ordinary `MemoryExpectedFile` updates;
   merge validation blockers with existing memory-init blockers and surface warnings
   through the root/init planning models without changing one-level note discovery.
   Ensure disabled webs remain byte-for-byte ordinary notes and no roster is appended.
5. Add a read-only `sase doctor` memory-web check that calls the same discovery and
   validator implementation and presents blockers as errors and cross-scope collisions
   as warnings. Register it in the stable doctor registry.
6. Add focused unit and integration coverage for parsing/defaults, both provider and
   scope behaviors, lookup precedence, both roster styles/marker failures, every
   validation class, memory-init enabled/disabled behavior, doctor parity, and the
   invariant that strand bodies never appear in generated `AGENTS.md` or provider shims.
   Preserve tests proving `discover_memory_notes` and inventory walkers remain flat.

## Verification

- Run the focused memory-web, memory-init, memory-note/inventory, doctor, and feature-
  flag test modules while iterating.
- Run `just install` as required for the ephemeral workspace, then `just check` for the
  repository-wide lint gates and diff-scoped tests.
- Inspect `git diff`, run `sase bead epic-symbols sase-sq.2`, resolve or re-key any
  phase-owned symbols, and close only `sase-sq.2` with a note describing the verified
  flag-state, validator, rendering-isolation, doctor, and `just check` evidence.
