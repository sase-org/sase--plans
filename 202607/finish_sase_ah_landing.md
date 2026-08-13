---
tier: tale
title: Finish and land the agent-publication reliability epic
goal:
  Repair the generated-skill and prompt-provenance landing gaps, restore all gates, close sase-ah normally, clean
  post-close Symvision residue, and mark its epic plan done.
bead: sase-ah
---

- **PARENT:** [202607/agent_publication_reliability.md](https://github.com/sase-org/sase--plans/blob/main/202607/agent_publication_reliability.md)
- **BEAD:** [sase-ah](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ah/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ah.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ah.land.md#member-code)
  - [bbugyi200.athena.sase-ah.land--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ah.land.md#member-plan)
- **COMMITS:**
  - [8d34bc9](https://github.com/sase-org/sase/commit/8d34bc9ae0f093f4170229cf78a7dafe8007a26f) — test: keep suite gate socket paths below Linux limits
  - [7ba8b1c](https://github.com/sase-org/sase/commit/7ba8b1ceab7d6652e011ac4461c1745e69f91997) — test: preserve suite-gate holder status at timeout

# Finish and land the agent-publication reliability epic

## Goal

Finish the two landing gaps discovered while auditing epic bead `sase-ah`, integrate the prompt-provenance requirement
introduced by work that landed after the epic began, restore the full repository gate, and only then close the epic. The
final landing order is fixed: close `sase-ah`, run the post-close Symvision cleanup, and mark
`@plan:202607/agent_publication_reliability.md` done.

## Verified starting point

- `sase bead show sase-ah` reports three closed children: `sase-ah.1`, `sase-ah.2`, and `sase-ah.3`.
- The implementation commits are `4fc555db0` (path-based host-project resolution), `d8afeb7b0` (durable terminal
  publication disposition), and `ee5938a20` (operator surfacing and residue cleanup).
- The current resolver maps this workspace's primary checkout, plans sidecar, beads sidecar, and linked `sase-core`
  checkout to `gh_sase-org__sase`.
- The live `gh_sase-org__sase` publication outbox is empty, confirming the named `k4`, `lt`, and `lz` residue was
  dropped rather than reset.
- The 115 focused agent-publication, doctor, TUI, provenance, commit-workflow, and later bead-page publication tests
  pass.
- Commits after `4fc555db0`, excluding the epic commits, were reviewed. The later bead-page publisher already normalizes
  a sidecar path through `workspace_context_for_plan_resolution`, so it reaches the correct workspace store and does not
  need to duplicate the agent-publication resolver.
- `just check` passes formatting, Ruff, mypy, script lint, Symvision, and toobig, but fails SASE validation for two
  completion gaps:
  1. `sase-ah.3` changed `src/sase/xprompts/skills/sase_chats.md`, while the five generated provider copies in the
     chezmoi source still describe only queued and quarantined publication states.
  2. `202607/agent_publication_reliability.md` and the later `202607/bead_pages.md` plan are each missing their prompt
     provenance link, even though their prompt files already link back to the plans.

## Phase 1 — Deploy the committed `sase_chats` skill source

Use `/sase_memory_read generated_skills.md` before acting and `/sase_repo` before reading or modifying the linked
`chezmoi` repository.

1. Confirm the main checkout is clean, `HEAD` is on the canonical branch, and the committed source at
   `src/sase/xprompts/skills/sase_chats.md` contains the `retired` disposition and `sase agent sync --drop-retired`
   guidance.
2. Preview with `sase skill init --diff`. It should show only the corresponding `sase_chats/SKILL.md` updates for the
   managed Gemini, Claude, Codex, OpenCode, and Qwen providers.
3. From the clean, merged main tree, run `sase skill init --force`; run `chezmoi apply` if deployment reports that it
   was skipped.
4. Verify `sase init skills --check` is clean and the installed/provider copies all contain the retired-state semantics.
5. If the chezmoi source has tracked changes, persist exactly those generated `sase_chats` files through the required
   SASE git-commit workflow, then confirm that repository is clean and synchronized. Do not hand-edit generated
   `SKILL.md` files.

## Phase 2 — Repair prompt provenance introduced during the integration window

Open the plans sidecar with `/sase_repo`; use the path it prints for every read and write.

1. Run the plan-link repair in dry-run mode and confirm it proposes only the two known missing prompt links:
   `202607/agent_publication_reliability.md` ↔ `202607/prompts/agent_publication_reliability.md` and
   `202607/bead_pages.md` ↔ `202607/prompts/bead_pages.md`.
2. Apply the supported repair (`sase plan links repair --write`) rather than manually inventing header syntax. Review
   the resulting plan-sidecar diff and any automatic commit to ensure no unrelated provenance changed.
3. Run `sase plan links validate`; the four missing-link/reverse-link diagnostics from the audit must be gone.

The `bead_pages` pair belongs here because that feature landed after `sase-ah` started and its missing integration now
blocks the same repository-wide gate.

## Phase 3 — Revalidate the completed implementation

1. Re-run the focused tests from the audit:

   ```bash
   .venv/bin/pytest -q \
     tests/agents_sync/test_commit_publication_target_resolution.py \
     tests/agents_sync/test_commit_publication.py \
     tests/agents_sync/test_publication_outbox.py \
     tests/agents_sync/test_git_sync.py \
     tests/agents_sync/test_cli.py \
     tests/doctor/test_checks_agent_publication.py \
     tests/ace/tui/test_artifacts_chats_detail.py \
     tests/history/test_chat_catalog_publication.py \
     tests/test_commit_workflow_checkpointing.py \
     tests/test_commit_workflow_resume.py \
     tests/test_bead/test_bead_page_publication.py
   ```

2. Run `just check`. It must pass, including `init skills --check` and plan-link validation. Diagnose genuine failures;
   do not classify a repeatable failure as unrelated merely to reach the close.
3. Reconfirm the publication outbox for `gh_sase-org__sase` is empty and add an attributed note to `sase-ah` summarizing
   the audit, integration repairs, and successful gates.

## Phase 4 — Land the epic (final phase)

This phase is deliberately last and must preserve the following order.

1. Close normally with `sase bead close sase-ah`. Do not force. If descendant validation rejects the close, finish or
   reopen the named work instead.
2. Only after the close, use `/sase_memory_read symvision.md`, then run `just symvision`. The close expires any
   `sase-ah` epic-symbol exemptions. Remove stale whitelist entries and unused code Symvision reports, following the
   memory's cleanup hierarchy. If this changes the main repository, run the affected tests and the required
   `just check`, then persist the scoped cleanup through the SASE commit workflow.
3. Open the linked plans repository through `/sase_repo` and change only the frontmatter of
   `202607/agent_publication_reliability.md` from `status: wip` to `status: done`. Preserve its repaired provenance
   header. Persist the plan change through the supported plans-sidecar commit workflow.
4. Verify `sase bead show sase-ah` reports `closed` with resolution `done`, `just symvision` is green, the epic plan
   reports `status: done`, the main and plans repositories are clean and synchronized, and the publication outbox
   remains empty.

## Acceptance criteria

- Generated `sase_chats` provider skills describe queued, quarantined, retired, and mixed states with correct retry/drop
  remediation, and skill initialization validation is clean.
- Both missing prompt/plan provenance pairs validate.
- Focused tests and `just check` pass before closure.
- `sase-ah` closes normally with resolution `done`; no force resolution is used.
- Post-close Symvision is clean, and any expired exemption cleanup is tested and persisted.
- `@plan:202607/agent_publication_reliability.md` has `status: done`.
- Main, plans, and any changed chezmoi repository state is durable and clean.
