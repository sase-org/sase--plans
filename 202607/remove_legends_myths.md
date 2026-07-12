---
create_time: 2026-07-09 15:40:17
status: wip
prompt: 202607/prompts/remove_legends_myths.md
tier: tale
---
# Remove SDD Legend And Myth Support

## Context

SASE currently exposes four SDD planning nouns in several layers: tales, epics, legends, and myths. The target end state
is narrower:

- Keep tales, epics, prompts, research, and bead-backed epic work.
- Remove legends as a supported plan tier, approval action, bead tier, xprompt, family role, mobile action, and SDD
  directory.
- Remove myths as an SDD artifact kind, directory, init/readme output, plan-search/list filter, and documentation topic.
- Delete current legend/myth artifacts and legend-tier bead state rather than migrating it.

Initial inspection found current effective SDD artifacts under `.sase/sdd`: two legend plan files, a legend README, an
empty myths directory with README, and one closed legend-tier bead hierarchy (`sase-26`, "SASE Mobile MVP Legend"). The
linked Rust core also contains shared legend/myth support in bead wire/schema/work planning, plan search, mobile
notification choices, and gateway contracts, so this is a cross-repo change.

## Compatibility Stance

This is an intentional product-surface removal. New SASE code should not create, approve, list, search, launch, or
document legends or myths. Existing legend/myth artifacts in the current SDD store will be deleted, and the old content
will remain available through git history.

Storage code should not preserve a hidden legend workflow. If legacy compatibility is needed while rebuilding existing
stores, keep only narrowly scoped parsing/migration needed to delete or ignore old data, and do not expose it through
CLI, UI, xprompts, or docs.

## Plan

1. Remove live SDD artifacts and legend bead records.
   - Use the bead CLI to remove the existing legend hierarchy (`sase bead rm sase-26`) so the projection, event streams,
     and child records stay internally consistent.
   - Delete `.sase/sdd/legends/` and `.sase/sdd/myths/`, including generated README files and the two legend plan files.
   - Verify the effective SDD store no longer contains legend-tier beads, `sdd/legends`, or `sdd/myths` references.

2. Narrow SDD artifact kinds in the Python repo.
   - Remove `legends` and `myths` from canonical SDD directories, init/readme generation, directory map output, SDD list
     choices, SDD link validation/repair, plan write helpers, plan search kinds/rendering, and commit-finalizer
     dirty-plan exceptions.
   - Update tests around `sase sdd init`, `sase sdd list`, SDD path lookup, SDD validation, and plan search.

3. Remove legend from plan approval and agent-family flow.
   - Delete the central `legend` approval choice and update modal choices, CLI `sase plan approve --kind`, remote/mobile
     action choices, status labels, persisted plan-action handling, notification handlers, and approval option model
     defaults.
   - Remove the legend family suffix/role mapping and any special `LEGEND APPROVED` display or loader behavior.
   - Update plan-chain, TUI, mobile bridge, parser, and golden tests to use approve/tale/epic/commit only.

4. Remove legend bead tier and work execution.
   - In Python, remove `BeadTier.LEGEND`, `--tier legend`, `--epic-count`, linked-legend epic creation, legend readiness
     rollback, legend work rendering, and legend-specific `sase bead work` branches. `sase bead work` should apply only
     to epic-tier plan beads.
   - In Rust core, remove the shared `Legend` bead tier, legend epic-count validation, legend work planner, mutation
     branches that allow legend readiness, search/list tier handling, JSONL/storage fixtures, and schema checks for
     newly created stores.
   - Update Python/Rust binding callers so both layers agree on the remaining tier vocabulary.

5. Remove xprompts and generated skill references.
   - Delete `#legend`, `#bd/new_legend`, and `#bd/land_legend` from the default xprompt config.
   - Remove legend parent arguments and `legend_bead_id` instructions from `#bd/new_epic`.
   - Update `src/sase/xprompts/skills/sase_beads.md` to document only plan, epic, and phase bead workflows, then run the
     normal skill regeneration step if required by the repo workflow.

6. Update docs, examples, and assets.
   - Rewrite README and docs pages that present legends/myths as SDD concepts, bead tiers, approval modes, CLI choices,
     mobile actions, or xprompts.
   - Remove or update fixtures, diagrams, generated SDD README text, and the SDD directory-map asset so the public docs
     describe only the retained SDD model.
   - Leave unrelated generic English uses of "legend" only when they are clearly about chart/map legends rather than SDD
     legend support.

7. Clean up tests and fixtures.
   - Delete legend-only test modules and fixtures.
   - Rewrite mixed tests so they assert the narrower behavior rather than parametrizing over legends/myths.
   - Refresh CLI/golden/mobile contract outputs after removing choices and routes.

8. Verify.
   - Run targeted Python tests for plan approval, SDD, plan search, bead CLI/work, xprompt tags, and mobile
     notifications while iterating.
   - Run Rust core formatting/tests for changed crates.
   - Run `just install` and `just check` in the Python repo before finishing, as required after repo file changes.
   - Finish with contextual searches for `legend`, `legends`, `myth`, and `myths` to confirm no remaining SDD support
     references are present.

## Risks

- The largest risk is leaving Python and Rust core with different tier/action vocabularies. The implementation should
  change both sides in the same pass and verify through binding/cross-repo tests.
- Existing bead event stores may still contain historical legend JSON if CLI deletion leaves tombstones rather than
  removing streams. Handle that before removing parser support.
- Documentation contains many old examples and generated contract snippets; stale docs are likely unless search cleanup
  is treated as part of the implementation rather than as a final polish step.
