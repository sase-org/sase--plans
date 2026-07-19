---
tier: tale
title: Finish generated skill deployment and land sase-6x
goal: 'The committed tribe wait/fork documentation is deployed to every managed provider
  skill, the integrated feature passes repository validation, and epic sase-6x is
  closed with post-close Symvision cleanup and its canonical plan marked done.

  '
create_time: 2026-07-18 19:47:34
status: done
prompt: 202607/prompts/finish_sase_6x_landing.md
---

# Plan: Finish generated skill deployment and land sase-6x

## Context and verified baseline

Epic `sase-6x` adds next-entity `@tribe` targeting for `%wait` and `#fork`, plus lean prompt-and-reply-statistics
context blocks when a fork target resolves to an agent clan. Its four implementation commits are present on `master`:

- `cee14d438` — tribe wait targeting (`sase-6x.1`)
- `7495f7033` — clan fork context (`sase-6x.2`)
- `175972194` — tribe fork targets (`sase-6x.3`)
- `bebd7cf85` — TUI completion and generated-skill source documentation (`sase-6x.4`)

The land audit read the plan, all child beads, current source, commit diffs, and post-start commits. The focused
wait/fork/completion suite passes 492 tests. The later axe policy/targeting, TUI, memory, glossary, and statistics
commits do not duplicate or conflict with the epic. Child `sase-6x.3` had stale status even though its commit and tests
were present; it has now been closed, so all four children are closed.

The remaining gap is concrete: `just check` passes format, Ruff, mypy, Symvision, and size checks, then fails
`sase validate` because the five managed provider copies of `sase_run/SKILL.md` have not been regenerated from the
committed source at `src/sase/xprompts/skills/sase_run.md`.

## Phase 1: Regenerate and deploy the managed skill copies

Follow the audited `generated_skills.md` memory procedure. Open the `chezmoi` linked repo with the `/sase_repo` workflow
before reading or modifying it. Do not hand-edit generated `SKILL.md` files. From the SASE workspace, run the
workspace-installed generator with force and without its automatic commit/push sequence (for example,
`sase skill init --force --no-commit`), then run `chezmoi apply` as required by the memory. Scope application to the
five generated `sase_run` provider targets if the chezmoi tree contains unrelated pending changes. Do not commit or push
unless separately authorized.

Verify regeneration with `sase skill init --check` and inspect the chezmoi diff to confirm it contains only the expected
`sase_run` documentation updates for the managed providers. Preserve all unrelated user changes.

## Phase 2: Revalidate the integrated feature

Run `just check` from the SASE workspace. The prior audit already ran `just install`, but rerun it first if the
workspace dependency state has changed. Diagnose any failure rather than weakening checks or accepting unrelated visual
snapshots. Re-run the focused epic tests when a remediation touches wait/fork/history/completion code.

Before changing anything in the Symvision domain, use the audited `sase memory read symvision.md` procedure. Before
changing the generated skill source or deployment logic, reread `generated_skills.md`. Do not edit canonical memory
files or generated provider instruction shims.

## Phase 3: Land and finalize the epic

This is the final phase and must run only after Phases 1 and 2 pass.

1. Confirm `sase bead show sase-6x` reports every child closed, then close the parent with `sase bead close sase-6x`.
2. After the close, run `just symvision`. Remove only stale epic-symbol entries or genuinely unused code that Symvision
   reports, following the documented decision hierarchy. If this changes SASE source or configuration, run `just check`
   again.
3. Open the `plans` sidecar through `/sase_repo`, then change only the frontmatter status in
   `202607/tribe_wait_fork_targets.md` from `wip` to `done` using `apply_patch`.
4. Verify the parent bead is closed, the plan reports `status: done`, Symvision passes post-close, and the
   primary/chezmoi/plans worktrees contain no unexpected changes. Report any expected uncommitted generated or bead/plan
   files explicitly; do not commit unless separately authorized.

## Risks and constraints

- Leading `@` wait/fork targets are deliberately tribe references; suffix or middle `@` remains the agent-name template
  form.
- Wait and fork must continue to share the same cutoff-based earliest-complete entity selection, including clan
  aggregation and self-exclusion.
- Clan fork blocks must never inline member reply bodies; they include sanitized prompts, reply statistics, metadata,
  and transcript paths only.
- Linked and sidecar repositories must be accessed only through `/sase_repo`.
- Do not modify SASE long-term memory or generated instruction shims.
