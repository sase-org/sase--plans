---
tier: tale
title: Complete sase-74 integration and landing
goal: 'The clan-scoped Agent Cleanup feature is reflected in documentation that changed
  while the epic was in flight, the epic is closed only after final verification,
  post-close Symvision findings are resolved, and the canonical epic plan is marked
  done.

  '
create_time: 2026-07-19 10:06:27
status: wip
prompt: 202607/prompts/sase_74_integration_and_landing.md
---

# Plan: Complete sase-74 integration and landing

## Context and verified baseline

Epic `sase-74` adds first-class clan cleanup to the Agents-tab `X` panel: `C` opens a planner-backed chooser for whole
clans or individual members in the focused tribe, and every result converges on the existing bulk confirmation and
execution funnel. Its three phase beads are closed.

The implementation audit already confirmed the promised work in source and history:

- `sase-core` commit `9d561ea` adds the additive Rust cleanup wire fields, clan selection semantics, generation
  filtering, workflow cascade behavior, and Rust/PyO3 tests. The focused Rust verification passed 31 cleanup tests, the
  cleanup-target wire-defaulting parity test, and the PyO3 clan-request round-trip test.
- SASE commits `dc0fa09f9`, `b14df5461`, and `d4087b08e` add the Python wire and fallback parity, clan
  chooser/panel/routing implementation, e2e behavior, help/footer coverage, and the PNG snapshot. The focused Python
  verification passed 216 tests, and the exact clan-modal PNG snapshot passed separately.
- Later SASE commit `286b992e4` rewrote help access around leader chords after the epic's polish commit. Its final help
  model and tests retain `Open cleanup panel (C: clan)`, so no help integration change is needed.
- Later commits in `sase-core` touch editor completion and prompt literals, not cleanup planning. Other SASE commits
  after the first epic commit do not duplicate or conflict with clan cleanup.

One integration gap remains. Commit `19db28c9e` refreshed `docs/ace.md` after the epic began, while the UI feature was
not complete. Three current passages still describe the old cleanup scopes and omit the clan chooser.

## Phase 1: Integrate clan cleanup into current ACE documentation

Update `docs/ace.md` wherever it enumerates the Agents-tab cleanup panel:

- The Agents keybinding table's `X` row must include clan cleanup alongside panel, global, tribe, marked, group, and
  custom cleanup.
- The detailed cleanup-panel paragraph must document `C` as the clan/member chooser scoped to the focused tribe, while
  preserving the existing meanings of lowercase `c` (custom) and `t` (tribe/tag chooser).
- The completed-agent dismissal paragraph must include clan selections in its list of planner-backed cleanup scopes.

Keep the wording consistent with the implemented behavior rather than adding a parallel conceptual model: whole-clan
selections use clan name plus generation, member subsets and mixed selections use explicit identities, and confirmation
still uses the shared bulk-cleanup flow. Format the Markdown, run the focused documentation/help tests as useful, then
run `just install` followed by the required `just check` gate for the SASE repository. Treat the local linked-core
published-version-window warning as informational if the dev build succeeds; do not change release-owned version
metadata as part of this plan.

## Final phase: Close and land epic sase-74

Perform this phase only after the documentation integration and repository checks pass.

1. Re-read `sase bead show sase-74` and each child to confirm the epic remains ready to land, then close it with exactly
   `sase bead close sase-74`.
2. Only after the close succeeds, use the `sase_memory_read` skill to review the required Symvision guidance, confirm
   that the `symvision` Just target is available, and run `just symvision`. Remove stale `sase-74` epic-symbol whitelist
   entries and any newly exposed unused code it reports. If this changes Python or configuration files, run
   `just install`, the relevant focused tests, and `just check`; rerun `just symvision` until it is clean.
3. Resolve the canonical plan path through `SASE_SDD_PLANS_DIR` (or `sase repo path plans` when needed) and, as the
   final state mutation, change the frontmatter of `202607/agent_cleanup_clan_scope.md` from `status: wip` to
   `status: done`. Do not alter the plan body or memory files.
4. Finish with read-only checks: `sase bead show sase-74` must report closed, every child must remain closed, the epic
   plan must report `status: done`, Symvision must be clean, and the relevant worktrees must contain only the
   intentional landing changes.

## Risks and guardrails

- Closing the epic expires its Symvision whitelist entries, so the close must precede the Symvision run; do not
  pre-emptively hide findings.
- The canonical plan and bead store may live in the plans sidecar. Use SASE's resolved paths and bead commands instead
  of assuming they are ordinary in-tree files.
- Preserve unrelated concurrent changes. Do not edit SASE memory files, release-owned Cargo versions, or unrelated
  documentation.
