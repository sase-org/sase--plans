---
tier: tale
title: Finish integration and land epic sase-dr
goal:
  Epic sase-dr is integrated with later prompt-archive changes, its follow-ups are deliberately settled, and its bead
  and linked plan are finalized.
proposed_by: bbugyi200.athena.sase-dr.land
bead: sase-dr
create_time: 2026-08-01 16:17:41
status: done
---

- **PROMPT:** [202608/prompts/land_sase_dr.md](prompts/land_sase_dr.md)
- **PARENT:**
  [202608/task_bead_plus_one.md](https://github.com/sase-org/sase--plans/blob/main/202608/task_bead_plus_one.md)
- **BEAD:** [sase-dr](https://github.com/sase-org/sase--beads/blob/main/pages/sase-dr/README.md)

# Finish integration and land epic `sase-dr`

## Context and verified baseline

Epic `sase-dr` implements corroborated task beads and disciplined task creation. Its five phase beads are closed. The
main-repository commits are `c9aed8a6f`, `767852ac9`, `d63a86bfd`, `0f1f28699`, `2ec86131d`, and `c1efe9f93`; linked
`sase-core` commit `e101432e3` owns the Rust domain/event/mutation contract. The land audit already confirmed that the
Python facade delegates to Rust, structured evidence is immutable and reporter-unique, creator/reporter retries are
idempotent, open/closed tasks promote to ready, new tasks require a size while legacy sizeless tasks launch as small,
large/xlarge tasks receive `#plan`, and CLI/pages/ACE/triage/mobile use the shared presentation contract.

The audit also ran:

- `cargo test --workspace` in the linked core checkout: all suites passed, with one debug-only performance test
  intentionally ignored.
- 186 focused Python tests across mutation, CLI, routing, pages, ACE, triage, mobile, and generated skill sources: all
  passed.
- 27 historical-regression tests covering admin-center selection, saved-query navigation, footer fakes, prompt archive
  imports, and bead contention: all passed.
- `just test-visual`: 405 passed and 1 skipped.
- `just check`: formatting, Ruff, mypy, pyscripts, changelog, Symvision, and toobig passed; validation then stopped on
  undeployed provider skills and the separate post-start prompt-archive migration.

## Phase 1: Revalidate concurrent integration state

1. Run `sase bead show sase-dr` and show all five children again. Preserve every `PROPOSED FOLLOW-UP:` disposition below
   in the eventual epic close note.
2. Re-open sidecar and linked repositories through `sase repo open` with audit reasons before reading or changing them:
   `plans`, `agents`, `sase-core`, and `chezmoi`. Re-check their working trees because the concurrently active
   prompt-archive epic `sase-dh` may have landed while this plan waited for approval. Preserve unrelated changes.
3. Re-check main and core history from the first epic commit. The unrelated commits after `c9aed8a6f` are the
   plan/prompt cross-linking and archive migration, Artifacts/TUI refactors, prompt validation, and documentation
   changes. Confirm that current head still has no duplicated task mutation/routing/presentation implementation and that
   phase-5 commit `c1efe9f93` remains the last required source integration.
4. Use the current checkout's installed CLI for post-migration commands. The host-level runtime was observed at
   `2d133a14a` while this checkout was at `77ec838db`; do not mistake missing `sase agent prompts` commands in that old
   host runtime for missing source. Run `just install` as needed and select the checkout runtime without letting a plain
   `uv run` re-sync away the locally built core binding.

## Phase 2: Integrate the prompt migration and settle every follow-up

1. Inspect the canonical prompt with `sase agent prompts show task_bead_plus_one`. If the linked epic plan still has the
   historical relative prompt bullet, update only its `PROMPT` header entry to the canonical agents-sidecar form:

   ```markdown
   - **PROMPT:**
     [prompts/202608/task_bead_plus_one.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/task_bead_plus_one.md)
   ```

   Do not mass-migrate unrelated historical plans as part of `sase-dr`. Active epic `sase-dh` owns the broader prompt
   archive migration and already records the 5,765-error validation state on child `sase-dh.7`.

2. Resolve phase proposals as follows and record every decision in the close note:
   - The repeated Symvision unused-public proposals from `sase-dr.1`, `.2`, and `.3` are resolved: current Symvision
     passed before validation.
   - The full-suite/admin-center/saved-query/footer/import-boundary/contention proposals from `.1`, `.2`, `.3`, and `.4`
     are resolved by `c1efe9f93` and the passing focused and visual suites above.
   - The Artifacts default-subtab and task-triage golden proposals from `.2` and `.3` are resolved by the current exact
     visual suite.
   - The July `uppercase_active_subtabs` reverse-link proposals from `.3` and `.4` are semantic duplicates of ready task
     `sase-dn`. Follow the `sase_new_task` duplicate policy and add one independent `+1` with this land audit's current
     reproduction evidence; do not create another task.
   - The broader canonical prompt migration remains causally owned and already documented by active epic `sase-dh`; do
     not create a task or duplicate its existing note.
   - The provider-skill deployment proposal from `.5` is completed in this phase, not filed as a task.
3. From a clean checkout whose source includes `2ec86131d`, deploy generated provider skills with
   `sase skill init --force`. The command currently refreshes the five `sase_new_task` provider files and five later
   `sase_artifact_file` updates. Because it writes the linked chezmoi repository, use only the path returned by
   `sase repo open chezmoi` and inspect the resulting diff. Verify with `sase skill init --diff` or the equivalent
   `init skills --check`; do not accept drift in the new skill.
4. Re-run focused validation for the canonical prompt, plan header, generated skill sources, and deployed provider
   files. A repository-wide plan-link failure remains outside this epic if `sase-dh` has not landed; report that active
   epic ownership precisely instead of mass-editing its sidecars.

## Phase 3: Land and finalize

1. Ensure all five descendants are still closed and no epic-caused issue remains. Close normally, without force:
   `sase bead close sase-dr --note "<complete verification, integration, test, and follow-up dispositions>"`. Include
   the Rust/main commit audit, later-commit integration review, prompt-link update, generated-skill deployment outcome,
   exact test results, the `sase-dn` corroboration, and why no new task beads were filed.
2. Only after the close succeeds, run `just symvision`. Remove any expired `sase-dr` whitelist entries or newly exposed
   unused code it reports. If this changes the main repository, run `just install` and the mandatory `just check`, while
   distinguishing any still-active `sase-dh` sidecar migration failure from code-quality failures.
3. As the final lifecycle mutation, set `status: done` in the frontmatter of the linked `202608/task_bead_plus_one.md`
   plan. Re-open/read through the audited plans-repository path, preserve the canonical prompt link, and verify the epic
   shows closed with all descendants closed.
4. Report final working-tree state for the main, core, plans, agents, beads, and chezmoi repositories. Do not commit
   ordinary repository changes unless the user or a SASE finalizer explicitly authorizes that commit workflow.
