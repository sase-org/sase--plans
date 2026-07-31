---
tier: tale
title: Finish integration and land epic sase-cp
goal:
  The generated bead memory incorporates concurrent bead-update semantics, the rollout is verified, worthwhile follow-up
  work is filed, and epic sase-cp is closed cleanly.
proposed_by: bbugyi200.athena.sase-cp.land
bead: sase-cp
create_time: 2026-07-31 15:54:35
status: done
---

- **PROMPT:** [202607/prompts/cp_land_integration.md](prompts/cp_land_integration.md)
- **PARENT:**
  [202607/sase_beads_memory.md](https://github.com/sase-org/sase--plans/blob/main/202607/sase_beads_memory.md)
- **BEAD:** [sase-cp](https://github.com/sase-org/sase--beads/blob/main/pages/sase-cp/README.md)

# Plan: Finish and land epic `sase-cp`

## Objective

Finish the integration audit for epic bead `sase-cp`, incorporate bead behavior that landed while the epic was in
progress, verify the complete generated-memory rollout, and then perform the epic's final landing sequence.

## Verified context to preserve

- `sase bead show sase-cp` reports three closed phases and links the canonical plan `plans:202607/sase_beads_memory.md`.
- Phase `sase-cp.1` added the packaged `memory-sase-beads.template.md`, generated-long-note overlays, first-pass Tier 2
  rendering, deployment staging, documentation, and tests in commit `d6a2cce1f`.
- Phase `sase-cp.2` retired the bundled `src/sase/xprompts/skills/sase_beads.md`, its docs row, source fixture, and
  superseded CLI-contract tests in commit `642b4f490`.
- Phase `sase-cp.3` removed six deployed chezmoi skill copies in linked-repo commit `67b58a6f`, and the home-directory
  scan currently finds no deployed `skills/sase_beads` directory outside ephemeral SASE workspaces. Home
  `sase/memory/sase_beads.md`, `AGENTS.md`, and `CLAUDE.md` exist and the two instruction files list the note in Tier 2.
- The epic plan and its prompt currently link to one another, so `sase-cp.1`'s missing-backlink proposal is resolved and
  should not become a task bead.
- Commit `7404e4ab1` repaired the two exact ACE PNG goldens named by `sase-cp.1`; verify the visual suite remains green,
  but do not file that already-resolved proposal as a task bead.
- No existing bead matched `skill init prune`. The stale editable-install concern proposed by `sase-cp.3` is worthwhile
  and overlaps the epic plan's explicit follow-up recommending a pruning path.

## Integration work

Commit `50988fe7f` landed between the epic's two primary-repo commits. It changed `sase bead update` to accept multiple
IDs atomically, report unchanged beads without committing them, and evaluate descendant-close validation against the
whole batch. It updated the old `sase_beads` skill, but phase 2 subsequently deleted that skill; the replacement Tier 2
note therefore missed those non-obvious semantics.

1. Add one concise paragraph to the packaged generated note
   `src/sase/main/init_memory/templates/memory-sase-beads.template.md` explaining that `sase bead update` accepts one or
   more IDs, applies common fields atomically, and treats already-matching beads as no-ops. Mention whole-batch
   descendant validation only if it can be stated without bloating the note. Preserve the note's purpose and concise
   style.
2. Run `just install` before project checks. Regenerate the canonical memory outputs using the workspace-installed SASE
   implementation (not a stale external editable checkout), including `sase/memory/sase_beads.md`, generated memory
   metadata, agent instructions, and provider shims as required by `sase memory init`.
3. Verify the packaged asset and generated note agree, the note remains an audited Tier 2 long-memory read, fresh-root
   initialization remains idempotent, and the CLI examples still parse. Run focused memory/asset tests, then
   `just test-visual` and `just check`.
4. Re-run the home/chezmoi rollout checks from phase 3. The canonical primary branch and linked chezmoi tree must
   contain no `sase_beads` skill source/copy, and the home scan must contain no deployed target. A globally configured
   stale editable checkout may still make `sase skill list` display the retired source with missing targets; do not
   modify an unrelated user checkout merely to hide that diagnostic. Record it as motivation for the task bead below.

## Final landing phase

Perform these steps last, after all integration edits and verification pass:

1. Create one task bead for pruning retired generated skill targets and guarding against stale editable sources that can
   resurrect them. Its description must cite `sase-cp.3` as the proposing bead and explain the observed stale
   editable-install behavior. Mark the new task bead `ready`. Do not file the missing backlink or visual-golden
   proposals because the evidence above shows both are resolved.
2. Close the epic with `sase bead close sase-cp --note "..."`. The note must summarize source/commit verification, the
   `50988fe7f` integration, test results, linked chezmoi/home rollout evidence, the filed task bead ID, and explicit
   reasons the other two proposed follow-ups were not filed. Do not force-close: every child is already closed, and any
   rejection must be investigated deliberately.
3. After the close, run `just symvision`. Remove only stale `sase-cp` epic-symbol whitelist entries and genuinely unused
   code it reports, then rerun Symvision until clean.
4. In the canonical plan path reported by `sase bead show sase-cp`, change the YAML frontmatter from `status: wip` to
   `status: done` without altering unrelated plan content.
5. Run the proportionate final checks required by the primary repository after all file changes, including `just check`,
   and report the final clean/dirty state of both the primary and plans repositories. Do not create a git commit unless
   an authorized completion finalizer explicitly requests one.
