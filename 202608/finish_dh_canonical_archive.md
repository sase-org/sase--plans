---
tier: epic
title: Close the remaining canonical prompt-archive gaps in sase-dh
goal: 'The agents sidecar is the canonical and only home for prompt Markdown in practice
  as well as in principle: no supported command can write prompt Markdown into plans,
  prompt search reads the canonical archive, plan authors can write ordinary body
  bullets named after header labels, the migration command acts on the caller''s own
  sidecar and publishes what it commits, the shipped docs and directory map match
  the implementation, and sase-dh is closed on verified evidence.

  '
phases:
- id: header-block
  title: Restrict plan-header parsing to the leading block
  depends_on: []
  size: medium
  description: 'header-block: stop the Rust plan-header parser from scanning the whole
    document body, so an ordinary known-label bullet no longer invalidates a plan;
    publish the fix in a core release, raise the Python floor to it, and drop the
    plans-sidecar wording workaround.

    '
- id: canonical-interfaces
  title: Make every prompt interface canonical
  depends_on: []
  size: medium
  description: 'canonical-interfaces: retire the export path that writes prompt Markdown
    into the plans store, retarget prompt search at the canonical agents archive,
    delete plans-snapshot discovery once nothing calls it, and add the regression
    test that enforces the invariant.

    '
- id: migrate-durability
  title: Make agent prompts migrate correct and durable
  depends_on: []
  size: medium
  description: 'migrate-durability: resolve the plans sidecar effective for the caller
    instead of the first existing clone, and make a write run either publish both
    sidecars or fail with exact recovery instructions, with restart states tested.

    '
- id: docs-assets
  title: Correct the directory-map asset and the prompt docs
  depends_on:
  - canonical-interfaces
  size: small
  description: 'docs-assets: remove prompt snapshots from the plans directory-map
    source, regenerate its PNG, and bring the prompt documentation in line with the
    canonical archive and the export decision.

    '
- id: closeout
  title: Close out sase-dh
  depends_on:
  - header-block
  - canonical-interfaces
  - migrate-durability
  - docs-assets
  size: medium
  description: 'closeout: re-verify every gap against final source and remote state,
    disposition all proposed follow-ups, close the epic on evidence rather than force,
    run post-close Symvision cleanup, and mark the linked plan done.'
proposed_by: bbugyi200.athena.rt
create_time: 2026-08-02 09:28:20
status: done
bead_id: sase-e7
---

- **BEAD:** [sase-e7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-e7/README.md)

# Close the remaining gaps in `sase-dh`

## Why this plan exists

`sase-dh` asked for one thing above all: the `<project>--agents` sidecar becomes the **canonical and only** location for
prompt Markdown files, with every prompt's artifacts captured at launch and published beside it.

A verification pass confirmed most of that shipped and works:

- launch-time staging writes a content-addressed pool under `.sase/artifacts/pool/`, so two prompts referencing the same
  path with different bytes get distinct pool files;
- both `@<kind>:<payload>` artifact references and plain `@path` file references are staged;
- `.sase/home/` moved to `.sase/artifacts/home/`;
- `sase commit` publishes captured bytes into `artifacts/<YYYYMM>/` beside `prompts/<YYYYMM>/` in the agents sidecar and
  rewrites the archived prompt with inline Markdown links to them;
- the historical migration is published — the plans sidecar has zero `*/prompts/*.md` files and the agents archive holds
  the migrated prompts;
- `sase validate` passes end to end (init memory/repo/skills, plan links validate, agent prompts validate);
- the focused staging/archive/validation/migration suites pass.

But `sase-dh.8` and all four of its phases were force-closed with a `canceled` resolution rather than completed, and the
epic `sase-dh` is still `IN_PROGRESS`. Concrete work those phases owned is still undone, and two items actively break
the headline invariant. This plan finishes exactly that remainder — it does not revisit what already works.

## Verified gaps

Every item below was reproduced against current `master`, not inferred from bead text.

1. **`sase prompt export --sdd` still writes prompt Markdown into the plans sidecar.** `_sdd_snapshot_path()` in
   `src/sase/prompt/cli_export.py` returns `store.kind_root("plans") / <YYYYMM> / "prompts" / <name>.md`. Any use of
   that flag recreates the exact file class that `list_plans_store_prompt_files()` reports as the
   `prompt-in-plans-store` **error** in `src/sase/sdd/_link_validation.py`. One command re-breaks `sase validate` and
   the "only location" guarantee.
2. **`sase prompt search --source sdd` still discovers plans-store prompt snapshots.** `_sdd_prompt_roots()` in
   `src/sase/prompt/search/sources.py` builds on `sdd_prompt_roots()` in `src/sase/sdd/_paths.py`, which globs
   `*/prompts` under each plans root. The canonical agents archive is not searched at all, so the store users are told
   is canonical is the one store `prompt search` cannot see.
3. **Rust plan-header parsing still scans the whole document body.** `first_header_candidate()` in
   `crates/sase_core/src/plan/artifact_link.rs` walks every line outside fences looking for a known header label, and
   `parse_block()` then rejects any later hit with `discontiguous or nested plan header bullets found`. Reproduced:

   ```
   a plan with a valid leading "- **PLAN:** [x](...)" header plus an ordinary
   "- **Artifacts:** ..." bullet deep in the body
     -> PlanHeaderDisposition.INVALID
        reason='discontiguous or nested plan header bullets found'
   ```

   This is why a plans commit had to rename a normal `**Artifacts:**` body bullet to `**Artifacts tab:**`. Plan authors
   currently cannot write ordinary prose bullets named `Artifacts`, `Beads`, `Commits`, `Agents`, `Plan`, `Prompt`, or
   `Parent`.

4. **`sase agent prompts migrate` picks the wrong plans clone.** `_plans_repo()` in `src/sase/agents/cli_prompts.py`
   ends with `next((path for path in candidates if path.is_dir()), None)` — the first existing clone, not the sidecar
   effective for the current project/workspace. From one workspace it can silently inspect another workspace's
   already-migrated checkout, report "no work", and leave the caller's own sidecar stale.
5. **`migrate --write` commits but never publishes.** `_apply_month()` in
   `src/sase/agents_sync/prompt_archive/migration.py` calls `_commit_paths()` for the agents repo and
   `commit_sdd_files()` for plans. `_commit_paths()` stages and commits and then returns — there is no push and no loud
   failure telling the operator how to recover a half-published migration.
6. **The plans directory-map asset still documents prompt snapshots under plans.**
   `src/sase/sdd/assets/plans-directory-map.png.prompt.md` lists `prompts/prompt.md` as monthly plans content (lines 42
   and 56), and `plans-directory-map.png` was never regenerated.
7. **`docs/prompt.md` still teaches the old layout.** The search section presents "exported or historical
   `plans/*/prompts/*.md` snapshots" as a first-class store, and the export section documents `--sdd` writing under
   `<resolved-plans-root>/YYYYMM/prompts/`.
8. **The epic is not closed out.** `sase-dh` is `IN_PROGRESS`, and its linked plan
   `plans:202608/artifact_persistence_sidecars.md` still carries `status: wip`.

Not a gap, checked and clear: the agents sidecar has no `artifacts/` directory yet only because every prompt archived so
far resolved to clean tracked VCS content, which links to hosted URLs by design instead of copying bytes. The copy path
itself is implemented in `_publish_linked_artifacts()` and covered by
`test_prepare_prompt_archive_links_and_copies_all_reference_classes`. No Symvision whitelist entries reference
`sase-dh`.

## Conventions for every phase

- Read `sase/memory/cli_rules.md` through `/sase_memory_read` before changing any CLI surface, and
  `sase/memory/symvision.md` before deleting code.
- Use `/sase_repo` to reach `sase-core`, the plans sidecar, and the agents sidecar. Never fetch their files over the
  web, and never name a numbered workspace path in a commit, doc, or plan edit.
- Run `just install` before `just check`, and `just check` before finishing any phase that touched files.
- Anything you discover that is out of scope goes in a `PROPOSED FOLLOW-UP:` note on your own phase bead.

## Phase 1: Restrict plan-header parsing to the leading block

**id:** header-block · **depends_on:** none · **size:** medium

Work in `sase-core` via `/sase_repo`.

1. Change `parse_block()` / `first_header_candidate()` in `crates/sase_core/src/plan/artifact_link.rs` so only a
   contiguous header block at the top of the body is eligible. A known label appearing later in the body is ordinary
   content and must not make the document `INVALID`.
   - "Top of the body" means the first non-blank content line after the frontmatter. Decide deliberately whether a
     leading `#` title may precede the block, and encode that decision in a test either way.
   - Keep the existing `discontiguous or nested plan header bullets found` diagnostic for genuinely broken blocks — a
     second header bullet group separated from the first _inside_ the leading region.
2. Add focused Rust tests for: a valid leading block; ordinary `**Artifacts:**` / `**Beads:**` / `**Commits:**` bullets
   later in a body; known labels inside fenced code blocks; and malformed or discontiguous entries within the real
   leading block.
3. Run that repo's full verification (`cargo fmt --check`, clippy with warnings denied, workspace tests, binding tests).
4. Let release-plz produce the release PR. Do not hand-edit release-owned versions or changelogs. Merge it, wait for the
   GitHub release and the PyPI wheel, and record the published version.
5. In the main repo, raise the `sase-core-rs` lower bound to that version, keep the `<0.18.0` ceiling, refresh
   `uv.lock`, and prove a clean sync installs the published wheel rather than a local source override.
6. Re-run the reproduction from Gap 3 against the published wheel and confirm it now parses as valid with the body
   bullet left as body content.
7. In the plans sidecar, restore the natural `**Artifacts:**` wording in `202608/artifacts_beads_and_files_subtabs.md`
   (line ~145 currently says `Artifacts tab:` purely to dodge this parser). Preserve the sentence's meaning, keep plan
   validation clean, and publish the edit through the normal plans workflow.

**Acceptance:** an ordinary known-label body bullet no longer invalidates a plan, the fix is in a published release that
a clean install resolves, and the plans-sidecar workaround is gone.

## Phase 2: Make every prompt interface canonical

**id:** canonical-interfaces · **depends_on:** none · **size:** medium

Export and search share `sdd_prompt_roots()`, so they move together.

1. **Export.** Stop `sase prompt export --sdd` from writing under any plans root. Recommended resolution: retire the
   flag. A local-history prompt has no agent run, no artifact manifest, and no captured bytes, so writing it into the
   canonical run archive would forge provenance the archive is supposed to guarantee — while `--out` already covers "put
   this prompt in a file". Make `--sdd` exit non-zero with an actionable message naming `--out` and `sase agent prompts`
   as the replacements, and remove `_sdd_snapshot_path()`.
   - If you instead retarget it into the agents archive, you must carry the full artifact contract and a manifest, and
     say in the phase note why forged provenance is acceptable. Do not preserve the split store under a new name.
2. **Search.** Retarget the SDD source at the canonical agents archive: read `prompts/<YYYYMM>/*.md` from the agents
   sidecar resolved the same way `sase agent prompts` resolves it. Rename the user-facing source accordingly (e.g.
   `--source archive`) and keep `sdd` working as a documented deprecated alias for one release. Preserve the existing
   filters, ranking (archive hits still rank above local history), result metadata, and `-f json` shape; add the
   archive's plan link and artifact count where the current SDD hit carries a plan reference.
3. Remove plans-snapshot prompt discovery once nothing calls it: `_sdd_prompt_roots()` in
   `src/sase/prompt/search/sources.py` and the `*/prompts` globbing in `sdd_prompt_roots()` (`src/sase/sdd/_paths.py`).
   Check every caller first — `find_sdd_file()` uses `sdd_prompt_roots()` for `_SDD_PROMPT_KINDS`, and
   `list_plans_store_prompt_files()` in `src/sase/sdd/_legacy_prompt_files.py` must keep working, because migration and
   validation still need to _detect_ stray plans-store prompts even after nothing can create them.
4. Update CLI help, `PromptSource` docstrings, and the renderers in `src/sase/prompt/render.py` and
   `src/sase/prompt/cli_search.py` so no user-facing string implies plans holds prompt Markdown.
5. Add a regression test that fails if any supported command writes a `*/prompts/*.md` file into a plans store — the
   test that would have caught Gap 1.
6. Update the affected tests (`tests/prompt_command/test_search_sources.py` and neighbors) and run the focused prompt
   CLI, search, archive, and link-validation suites, then `just install` && `just check`.

**Acceptance:** no supported command can write prompt Markdown into plans, `prompt search` reads the canonical archive,
and a regression test enforces both.

## Phase 3: Make `agent prompts migrate` correct and durable

**id:** migrate-durability · **depends_on:** none · **size:** medium

1. Replace the first-existing-clone heuristic in `_plans_repo()` (`src/sase/agents/cli_prompts.py`) with the plans
   sidecar effective for the current project and workspace — the same resolution the rest of the command uses for its
   agents target. Add a regression test with two plans clones where only the non-current one is already migrated, and
   assert the command operates on the current one.
2. Make `migrate --write` finish with both sidecars durably committed **and published**, or fail loudly.
   `_apply_month()` currently returns after `_commit_paths()` with no push. Publish both sides through the established
   sidecar workflows, and on partial failure emit the exact recovery command sequence rather than a bare exception.
3. Keep dry-run strictly read-only, and keep the operation idempotent. Test the restart states explicitly:
   - the archive entry exists but the plans snapshot remains;
   - one sidecar is committed and the other is not;
   - one sidecar is pushed and the other is not;
   - a second run over a fully migrated month reports no work and changes nothing.
4. Run the migration and archive suites, then `just install` && `just check`.

**Acceptance:** migrate always acts on the caller's own plans sidecar, and a `--write` run either publishes both
sidecars or tells the operator exactly how to recover.

## Phase 4: Correct the directory-map asset and the prompt docs

**id:** docs-assets · **depends_on:** canonical-interfaces · **size:** small

1. Fix `src/sase/sdd/assets/plans-directory-map.png.prompt.md` so monthly plans content no longer lists
   `prompts/prompt.md` (lines ~42 and ~56). Plans hold plan Markdown plus links; prompt Markdown lives in the agents
   sidecar.
2. Regenerate `plans-directory-map.png` through the established asset workflow and confirm the rendered image shows no
   `plans/.../prompts/` node. Cross-check it against `agents-directory-map.png`, which should be the one showing
   `prompts/` and `artifacts/`.
3. Update `docs/prompt.md`: the search section must describe the canonical agents archive as the non-local store (not
   `plans/*/prompts/*.md`), and the export section must match whatever Phase 2 decided for `--sdd`.
4. Sweep `docs/` for any remaining text placing prompt Markdown under plans, and reconcile with `docs/sdd.md`,
   `docs/cli.md`, `docs/configuration.md`, and `docs/agents_sidecar.md`, which already describe the new layout.
5. Run `just install` && `just check`.

**Acceptance:** the shipped directory map and the prompt docs match the implementation.

## Phase 5: Close out `sase-dh`

**id:** closeout · **depends_on:** header-block, canonical-interfaces, migrate-durability, docs-assets · **size:**
medium

This phase runs last and must not start until the other four are landed and published.

1. Re-run `sase bead show sase-dh` and every child show. Re-verify each of the eight gaps above against final source and
   remote state, not against bead text. Collect every `PROPOSED FOLLOW-UP:` note across the whole tree, including ones
   added while this epic ran.
2. Note explicitly in the close text that `sase-dh.8` and its four phases were force-closed as `canceled`, and state
   which of their requirements this epic actually completed.
3. Disposition every follow-up before closing: absorbed here, resolved and evidenced, duplicate, or no longer
   reproducible. For anything still worth doing independently, use `/sase_new_task` first for duplicate and active-epic
   triage, choose an intentional `--size`, name the proposing bead in the description, and move it to `ready`.
4. Re-run `sase validate` and the full test suite against the published core wheel. Record the exact counts you observe
   — current plans and agents archive totals, error counts, and the pre-existing unpaired-link warnings — and treat any
   new failure as work for this epic rather than an inherited baseline.
5. Confirm main, `sase-core`, plans, and agents are all clean and synchronized.
6. Close with `sase bead close sase-dh --note "..."`. The note must cover child verification, the force-close history,
   the published core version, sidecar validation counts, the full-suite result, and follow-up disposition. Do not reach
   for `--force` merely to make the close succeed; if a child is genuinely incomplete, finish or reopen it.
7. Only after the close succeeds, run `just symvision`, remove anything newly exposed as unused by the Phase 2
   deletions, rerun Symvision and the proportionate checks, and publish the cleanup through the SASE commit workflow.
8. Set `status: done` in the frontmatter of `plans:202608/artifact_persistence_sidecars.md`, leaving the rest of the
   plan intact, and publish that plans-sidecar edit.
9. Finish by showing the closed epic, reading the linked plan from a fresh remote-backed checkout, and reporting clean
   and synchronized status for all four repositories.

**Acceptance:** `sase-dh` is closed on verified evidence, its plan reads `status: done`, every follow-up has a recorded
disposition, and all four repositories are clean.
