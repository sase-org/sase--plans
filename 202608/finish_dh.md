---
tier: epic
title: Finish and land canonical prompt artifact persistence
goal: Complete the missing release, integration, migration-publication, and closeout
  work required for sase-dh so prompt artifacts have one canonical archive, every
  supported command preserves that invariant, clean installs use the shipped Rust
  contract, and the epic can be closed with verified remote state.
phases:
- id: rust-release
  title: Finish the Rust contract integration and publish it
  depends_on: []
  size: medium
  description: 'rust-release: Restrict artifact-header recognition to the leading
    plan header block, verify the existing prompt-artifact wire contract, and publish
    the next sase-core release so a clean Python install contains the feature.'
- id: canonical-interfaces
  title: Make every prompt interface use the canonical agents archive
  depends_on:
  - rust-release
  size: medium
  description: 'canonical-interfaces: Fix plans-sidecar selection and publication
    in the migration command, retire plans-sidecar prompt export/search behavior,
    and update tests and documentation so no supported command can recreate prompt
    snapshots in plans.'
- id: publish-migration
  title: Publish and validate the historical migration
  depends_on:
  - canonical-interfaces
  size: medium
  description: 'publish-migration: Recover and publish the six completed plans-sidecar
    migration commits on top of current remote work, then prove that plans has no
    prompt snapshots and both sidecars validate from their remote tips.'
- id: close-epic
  title: Close sase-dh and complete post-close cleanup
  depends_on:
  - publish-migration
  size: medium
  description: 'close-epic: Re-audit all child notes, disposition every proposed follow-up,
    close the epic without an unjustified force, run post-close Symvision cleanup,
    and mark the linked epic plan done.'
status: wip
proposed_by: bbugyi200.athena.sase-dh.land
parent_bead: sase-dh
create_time: 2026-08-01 16:22:50
bead_id: sase-dh.8
---

- **PROMPT:** [prompts/202608/finish_dh.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_dh.md)
- **PARENT:** [202608/artifact_persistence_sidecars.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_persistence_sidecars.md)
- **BEAD:** [sase-dh.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-dh/sase-dh.8.md)

# Finish and Land `sase-dh`

## Context and verified gap

The seven child beads are closed and their primary commits exist, but the epic is not landable yet:

- `sase-core` commit `f97c7f141` implements the Rust prompt-artifact schema and bindings, but no published release
  contains it. The main repo still allows `sase-core-rs>=0.17.5`; a clean dependency sync selects 0.17.7 and the focused
  artifact suite fails because that wheel still exposes schema version 2 and lacks the new bindings. Release-plz PR #73
  currently proposes 0.17.8 and includes the feature.
- The six monthly plans migrations exist as clean local commits (`fe5174db`, `68dec55f`, `f74d6098`, `fd8bfe49`,
  `2afcd799`, `2a4a2223`) rebased over later plans work, but they were never pushed. The remote plans sidecar therefore
  still contains 2,892 prompt snapshots and validates with 5,765 errors: 2,892 `prompt-in-plans-store` errors and 2,873
  missing-target errors.
- The migrated checkout itself validates cleanly with 3,392 plan files, zero errors, and 519 acknowledged pre-existing
  unpaired-link warnings. The agents archive validates with 2,893 prompts, zero errors, and zero warnings. Preserve
  these as the expected post-migration counts unless concurrent work legitimately changes them.
- `sase agent prompts migrate` discovers the first existing plans clone instead of the active/effective plans sidecar.
  From another numbered workspace it can silently inspect the already-migrated checkout, report no work, and leave that
  workspace's plans sidecar stale. Its write path commits both repositories but does not reliably publish the plans
  commit.
- Legacy `sase prompt export --sdd` still writes `YYYYMM/prompts/*.md` into plans, and `sase prompt search --source sdd`
  still reads those snapshots. Both contradict the new canonical-agents-archive invariant and can immediately recreate
  validation failures.
- The Rust artifact parser scans the entire plan body for known header labels. A later plans commit (`8014060f`) had to
  rename an ordinary `**Artifacts:**` body bullet to `**Artifacts tab:**` to avoid false header detection. Header
  recognition must be limited to the leading header block.
- `src/sase/sdd/assets/plans-directory-map.png.prompt.md` still describes prompt snapshots under plans and its PNG needs
  regeneration after correcting that source.

The audit also verified that the core workspace tests, formatting, and clippy pass with the local feature build; the
focused Python artifact tests pass with that local build; current lint/Symvision pass while the epic remains open;
memory initialization is clean; and later main-repo changes, including the prompt-archive removal of an ACE UI helper
dependency, coexist with the epic code. One full parallel Python run ended with 25,355 passes, 7 skips, and two
xprompt-selector failures that immediately passed in isolation; the landing run must either pass cleanly or attribute
and repair/file that parallel-only failure deliberately.

## Phase 1: Finish the Rust contract integration and publish it

Work in `sase-core` through `/sase_repo`; do not fetch repository files over the web.

1. Change artifact-header parsing so only a contiguous leading header block is eligible. Ordinary known-label bullets
   later in the body must remain body content, including `**Artifacts:**`, `**Artifact:**`, and the other recognized
   aliases.
2. Add focused Rust tests for:
   - a valid leading prompt/artifact header;
   - ordinary known-label bullets later in a plan body;
   - fenced examples containing known labels;
   - malformed or discontiguous entries inside the actual leading header block.
3. Revisit plans commit `8014060f`. Restore natural wording if the parser fix makes the workaround unnecessary, while
   preserving that commit's intended documentation meaning and keeping plan validation clean.
4. Run the complete core verification required by that repository (`cargo fmt --check`, clippy with warnings denied,
   workspace tests, and binding tests).
5. Let release-plz incorporate the parser fix into PR #73 or its replacement. Do not manually edit release-owned
   versions or changelogs. Merge the clean release PR, wait for the GitHub release and PyPI wheel to publish, and record
   the actual released version.
6. In the main repo, raise the `sase-core-rs` lower bound to that release while retaining the `<0.18.0` compatibility
   ceiling, refresh `uv.lock`, and prove a clean dependency sync installs the published wheel rather than a local source
   override.
7. Run the focused artifact, staging, archive, cross-link, and validation tests against the published wheel. Explicitly
   assert schema version 3 and availability of the new prompt-artifact bindings.

Acceptance: the next core release is remotely published, clean installs resolve it, no local editable core masks
packaging errors, and the Rust/Python contract tests pass.

## Phase 2: Make every prompt interface canonical

1. Replace `_plans_repo()`'s first-existing-clone heuristic with the same effective sidecar resolution used for the
   current project/workspace. Add a regression test with multiple plans clones where only a non-current clone has
   already been migrated.
2. Make `sase agent prompts migrate --write` finish with both destination and source changes durably committed and
   published, or fail loudly with exact recovery instructions. Keep dry-run read-only. Preserve idempotence and
   explicitly test restart states where:
   - the archive entry exists but the plans snapshot remains;
   - one sidecar commit exists and the other does not;
   - one sidecar is pushed and the other is not;
   - a second run sees a fully migrated month.
3. Remove or retarget `sase prompt export --sdd` so it cannot write prompt snapshots into plans. Exporting to SDD-backed
   storage must use the canonical agents archive and the artifact contract, or the obsolete option must be deprecated
   with an actionable replacement. Do not silently preserve the split store under a renamed implementation.
4. Retarget `sase prompt search --source sdd` to the canonical archive or replace that source name with a clear
   canonical alternative. Preserve useful filters and result metadata; remove plans-snapshot discovery code once all
   callers migrate.
5. Update CLI help, docs, unit/integration tests, and examples to state that agents is the only prompt archive and plans
   contains links plus generated artifact metadata, never prompt markdown snapshots.
6. Correct `src/sase/sdd/assets/plans-directory-map.png.prompt.md` and regenerate `plans-directory-map.png` using the
   established asset workflow. Verify both the source prompt and rendered image no longer show `plans/.../prompts/`.
7. Run focused prompt CLI, migration, archive, artifact, link-validation, and documentation tests. Then run
   `just install` followed by `just check` in the main repo.

Acceptance: every supported prompt export/search/migration path reads or writes the canonical agents archive, the active
sidecar is selected deterministically, partial publication is recoverable and tested, and the docs/image match the
implementation.

## Phase 3: Publish and validate the historical migration

Use `/sase_repo` for both sidecars. Recover the six already-created plans commits by hash from the registered plans
checkout that contains them; do not mention or depend on an absolute numbered-workspace path. Preserve their
month-by-month history unless a concurrent remote change requires a deliberate rebase.

1. Fetch the plans remote and compare each of the six commit patches with the current migration output. If Phase 2 or
   later remote work changes the expected patch, rebase or regenerate deliberately and document why; do not duplicate
   archived prompts or discard unrelated plan edits.
2. Publish the March through August migration commits to the plans remote. Confirm the remote branch contains all six
   effective patches, not merely that a local checkout is ahead.
3. Refresh the current effective plans sidecar from its remote and verify it has zero `*/prompts/*.md` files.
4. Run strict link validation from the remote tips of both sidecars. Expected baseline, absent legitimate concurrent
   additions:
   - plans: 3,392 files, zero errors, 519 pre-existing unpaired-link warnings;
   - agents archive: 2,893 prompts, zero errors, zero warnings.
5. Verify representative March-August prompt links resolve in both directions, canonical archives preserve full prompt
   contents, agent links contain the migrated agent identity, and no staging or archive code refers to the deleted plans
   snapshots.
6. Run the main full test suite against the published core wheel and record the exact summary. Treat new failures as
   work for this epic; do not dismiss them as an old baseline without reproducing and attributing them.

Acceptance: remote state—not only a private checkout—contains the migration, fresh sidecar checkouts validate, no prompt
snapshot remains in plans, and the full suite passes.

## Phase 4: Close `sase-dh` and complete post-close cleanup

This is the final phase and must follow the user's landing order.

1. Re-run `sase bead show sase-dh` and every child show. Confirm all child requirements against the final source and
   remote commits. Collect all `PROPOSED FOLLOW-UP:` entries again so concurrent notes are not missed.
2. Disposition each proposal before closing:
   - absorb the leading-header parser issue, incomplete migration/link repair, canonical directory-map regeneration, and
     any prompt-interface inconsistencies into Phases 1-3;
   - record the current evidence that the uppercase subtab link pair is repaired, the private-import Symvision failure
     is gone, and memory initialization is clean; require Phase 3's full-suite result before declaring the
     ACE/TUI/default-suite proposals resolved, including deliberate treatment of the parallel-only xprompt-selector
     failure;
   - for any still-worthwhile independent work, invoke `/sase_new_task` first for duplicate/epic triage, choose an
     intentional size, create the task with the proposing bead named in its description, and move it to `ready`;
   - state in the epic close note why every proposal not filed was absorbed, resolved, duplicated, or no longer
     reproducible.
3. Confirm all relevant repositories are clean and synchronized and run `just install` then `just check` in the main
   repo.
4. Close with `sase bead close sase-dh --note "..."`. The note must summarize child/commit verification, post-start
   integration, published core version, sidecar remote validation counts, full-test result, and follow-up disposition.
   Do not use `--force` merely to succeed. If closure names incomplete phases, finish or reopen them; use forced
   canceled/superseded resolution only when that outcome is deliberate and fully explained.
5. Only after the close succeeds, run `just symvision`. Remove expired `sase-dh` whitelist entries and any newly exposed
   unused code, then rerun Symvision and the proportionate main checks. Commit and publish those cleanup changes through
   the required SASE commit workflow.
6. Set `status: done` in the frontmatter of `@plan:202608/artifact_persistence_sidecars.md`, preserving the rest of the
   plan. Publish the plans-sidecar edit through its normal workflow.
7. Finish by showing the closed epic, reading the linked plan from a fresh remote-backed checkout, and reporting
   clean/synchronized status for main, core, plans, and agents.

Acceptance: `sase-dh` is closed without an unjustified force, post-close Symvision is clean, the linked plan says
`status: done`, every follow-up is explicitly dispositioned, and all four repositories reflect the landed result
remotely.
