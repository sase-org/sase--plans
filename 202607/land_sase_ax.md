---
tier: tale
title: Integrate and land sase-ax
goal:
  The artifact read CLI is integrated with later completion work, deployed and backfilled, verified end to end, and
  closed cleanly.
bead: sase-ax
create_time: 2026-07-29 19:30:34
status: done
---

- **PROMPT:** [202607/prompts/land_sase_ax.md](prompts/land_sase_ax.md)
- **PARENT:**
  [202607/artifact_read_cli.md](https://github.com/sase-org/sase--plans/blob/main/202607/artifact_read_cli.md)
- **BEAD:** [sase-ax](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ax/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ax.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ax.land.md#member-code)
  - [bbugyi200.athena.sase-ax.land--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ax.land.md#member-plan)
- **COMMITS:**
  - [af42951](https://github.com/sase-org/sase/commit/af42951798753ef28a2c73e75bcbef1780dbfb83) — perf: warm artifact
    completion through Rust query

# Integrate and land `sase-ax`

## Goal

Finish the `sase-ax` epic as a verified, integrated feature: route the newer prompt artifact-completion catalog through
the epic's Rust-backed artifact query contract, complete the real-index backfill and generated-skill deployment
intentionally deferred to landing, validate the merged behavior, then close the bead and finalize its durable epic plan.

## Context and verified baseline

- `sase bead show sase-ax` reports four child phases, all closed with resolution `done`, and links
  `plans:202607/artifact_read_cli.md`.
- The phase commits are:
  - `ad900a77` in `sase-core` for `sase-ax.1`, providing the shared tolerant v1/v2 artifact-index reader,
    filtering/sorting query API, PyO3 binding, and wire handshake.
  - `f39b0c40` for `sase-ax.2`, providing the three optional enrichment fields, store-time population, preserving
    writer, and inspect/backfill/verify APIs.
  - `30e2ed37` for `sase-ax.3`, providing the canonical `sase artifact` CLI, its compatibility alias, Rust query facade,
    reference resolution, viewers, and tests.
  - `c40aa7f9` for `sase-ax.4`, providing the read-capable generated-skill source, documentation, and the phase-3
    artifact-reference regression fix.
- The implementation and tests match the child-bead notes. The main and linked-core branches are clean and match their
  canonical base branches.
- The non-epic commits after the first epic commit were reviewed. Preview rendering, preview search, prompt input
  stabilization, punctuation handling, and path-inventory warming do not conflict with the CLI. The newer unified
  `@`-completion work still loads and sorts artifact rows through `read_artifact_file_index`, duplicating the query
  behavior that became available in this epic.
- `sase skill init --diff` confirms that the committed `sase_artifact_file` source has not yet been deployed to the
  managed provider skill files, as the phase note intentionally deferred.
- A read-only `sase artifact doctor` run confirms 4,090 supported rows with no duplicate IDs, unsupported schema
  versions, or malformed rows, but many legacy rows still need the three-field enrichment backfill.

## Implementation

1. Integrate the post-epic prompt-completion work with the artifact query contract.
   - In `src/sase/ace/tui/widgets/artifact_ref_completion.py`, make the existing mtime/size-keyed, off-thread
     artifact-index cache load rows through `sase.core.artifact_file_query_facade.query_artifact_files`.
   - Preserve the cache boundary and the keystroke-path rule: querying and disk work remain confined to catalog warming,
     while warm completion reads only the immutable catalog snapshot.
   - Preserve current project/alias filtering, inclusion of unscoped rows, the 500-candidate bound, and Rust's
     deterministic newest-first ordering. Remove only the now-duplicate Python sorting.
   - Update focused completion tests to prove a cache miss uses the Rust-backed query, an unchanged index token reuses
     cached rows, ordering/filtering remain stable, and keystroke paths never invoke the query provider.

2. Complete the deferred runtime integration.
   - From the clean canonical `sase` tree, run `sase skill init --force`; if generation reports that chezmoi application
     was skipped, run `chezmoi apply`. Re-run `sase skill init --diff` and require no remaining generated-skill drift.
   - Run `sase artifact doctor --fix` against the real artifact index, then run `sase artifact doctor` again and require
     a healthy exit. Missing recycled-workspace source paths remain informational; stored-file, enrichment, duplicate,
     schema, and malformed-row problems must be absent.
   - Smoke the landed CLI on real data: bounded image listing renders durable refs and display names, and
     `sase artifact path plans:202607/artifact_read_cli.md` resolves to the linked epic plan.

3. Validate the integrated tree.
   - Run `just install` before repository checks.
   - Run the focused artifact query/CLI/doctor/completion tests, including the filesystem-free keystroke regression.
   - Run `just check` and resolve any failure caused by this epic or its integration. Diagnose unrelated pre-existing
     failures explicitly rather than masking them.

4. Land the epic last.
   - Close `sase-ax` without force using `sase bead close sase-ax --note "..."`. The note must summarize the four-child
     source/commit audit, the reviewed post-start commits, Rust-query completion integration, generated-skill
     deployment, real-index backfill, smoke results, and validation.
   - After the close, read the required Symvision long-memory guidance and run `just symvision`. Remove stale `sase-ax`
     epic-symbol allowances and any unused code exposed by closure. If cleanup changes repository files, run
     `just check` again.
   - Open the plans sidecar through `sase repo open plans`, change only the epic plan's frontmatter from `status: wip`
     to `status: done`, and verify both the plan and `sase bead show sase-ax` report the terminal state.
   - Do not force the close. If descendant or phase validation rejects it, finish or reopen the named work and repeat
     the relevant validation; use a non-`done` forced resolution only for a deliberate cancellation or supersession.

## Acceptance

- Artifact completion uses the Rust query facade only during off-thread catalog warming and retains its cache, bounds,
  project behavior, and keystroke latency guarantees.
- The managed `sase_artifact_file` skills match the committed template.
- The real artifact index is healthy after idempotent enrichment backfill.
- Focused tests and `just check` pass.
- `sase-ax` is closed with resolution `done` without force, post-close Symvision is clean, and
  `plans:202607/artifact_read_cli.md` has `status: done`.
