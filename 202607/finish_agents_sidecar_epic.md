---
tier: tale
title: Finish and land the hidden agents sidecar epic
goal: Epic sase-8k is fully integrated with intervening work, verified end to end,
  closed, Symvision-clean, and marked done.
bead: sase-8k
parent: sase/repos/plans/202607/agents_sidecar_repo.md
create_time: 2026-07-22 17:12:26
status: wip
---

- **PROMPT:** [202607/prompts/finish_agents_sidecar_epic.md](prompts/finish_agents_sidecar_epic.md)

# Finish and land the hidden agents sidecar epic

## Goal

Complete the integration work discovered while auditing epic bead `sase-8k`, verify the corrected behavior, and then
land the epic. Preserve the epic's core contracts: durable local agent identities are machine-qualified, local UI and
editable prompts hide the local machine hood, foreign hoods remain visible, legacy unqualified commit footers remain
backfillable, and the agents sidecar only publishes completed commit-associated agents.

## Audit context

The epic bead has eight closed children (`sase-8k.1` through `sase-8k.8`). Their implementation commits are present on
the current primary branch; the Rust helper commit `6b39455b8899917df4c92f5c59a765b1286ea91a` is present on the current
`sase-core` branch. The linked epic plan is `$SASE_SDD_PLANS_DIR/202607/agents_sidecar_repo.md` and is still
`status: wip`.

The completed implementation and a 464-test focused baseline verify the main feature, but the landing audit found three
integration gaps:

1. `src/sase/agents_sync/bundles.py::_historical_commit_associations` rejects every footer whose legacy `SASE_MACHINE`
   hostname differs from the new configured `machine_name`. This contradicts the plan's compatibility rule that an
   unqualified legacy `SASE_AGENT` is local and must be qualified with the current machine hood during backfill.
   Existing coverage only tests the coincidental hostname-equals-machine-name case.
2. Agent retry and kill/edit paths added while the epic was in progress pass durable names such as `athena.sase-8k.2`
   into prompt rewriters whose source prompt still uses the local presentation spelling `sase-8k.2`. A qualified
   clan-member probe currently raises `ValueError`, and retry allocation can write the local machine hood into the
   editable prompt. The tribe summary code likewise compares stripped child labels with raw qualified clan/family
   prefixes, preventing its relative-label compaction.
3. The two-machine test proves the transport round trip but uses a plain-text pseudo-transcript, so it never verifies
   that an imported real SASE transcript appears through `sase chat list`, with the local hood stripped on its owning
   machine and the foreign hood preserved on its peer.

Before touching a sidecar checkout, use the `/sase_repo` workflow and retain the path it returns. Do not edit SASE
memory or generated instruction files; the glossary follow-up remains out of scope without explicit user approval.

## Phase 1: Restore legacy-footer backfill compatibility

- Update `src/sase/agents_sync/bundles.py` so historical footer classification distinguishes modern machine-qualified
  provenance from legacy unqualified provenance.
  - An unqualified `SASE_AGENT` must be treated as local and qualified with the current configured `machine_name` even
    when its legacy `SASE_MACHINE` hostname differs.
  - A modern agent name explicitly qualified by a different `SASE_MACHINE` must remain foreign and must not be exported
    by the local machine.
  - Preserve validation, deterministic commit ordering, duplicate-SHA coalescing, and the rule that only a matching
    completed local artifact with a readable transcript becomes a bundle.
- Extend `tests/agents_sync/test_bundles.py` with real trailing-tag parsing and explicit cases for:
  - an unqualified legacy agent with a mismatched legacy hostname that backfills as `<current-machine>.<agent>`;
  - a local modern qualified footer that remains accepted;
  - a foreign modern qualified footer that remains excluded.
- Prefer a small named helper for footer classification if that makes the legacy/modern ambiguity and policy explicit.

## Phase 2: Integrate machine hoods with later prompt and tribe UI paths

- At the retry/kill-and-edit action boundary, derive prompt-facing names with the existing machine-hood facade instead
  of writing durable local-qualified names into source prompts.
  - Strip the local hood before family-base normalization and before passing `current_agent_name` or replacement names
    to clan prompt rewriting.
  - Retry allocation must still consult the durable qualified registry namespace, but strip its returned local hood
    before inserting the allocated name into the editable prompt.
  - Preserve foreign hoods and existing template, family, clan, bead, and forced-reuse semantics.
- Update `src/sase/ace/tui/models/agent_tribe_summary.py::_relative_child_label` to compare presented child names with
  `Agent.presented_clan_reference_name()` / `Agent.presented_family_reference_name()`. These methods were added by the
  epic specifically to avoid config or selector I/O on render paths; do not call `MachineHoodIdentity.current()` from
  the tribe render loop.
- Add regression coverage in `tests/ace/tui/test_retry_edit_agent_name.py` for qualified local standalone, family, and
  clan retry/kill-edit cases, including the currently failing `athena.sase-8k.2` clan case. Add focused tribe-summary
  coverage showing that qualified local clan/family children compact relative to their presented container while a
  foreign hood remains visible.

## Phase 3: Complete the end-to-end user-facing exercise

- Strengthen `tests/agents_sync/test_two_machine_e2e.py` to use a realistic SASE chat transcript with the standard
  `# Chat History - <workflow> (<agent>)` header rather than plain text.
- In the same alpha/beta round trip, exercise the real chat catalog/list projection after export/import:
  - alpha sees its own `alpha.worker` transcript presented as `worker`;
  - beta sees the imported transcript presented as `alpha.worker`;
  - the imported chat remains resolvable through the reconstructed artifact's `chat_path` / `response_path`.
- Keep the existing assertions for portable metadata allowlisting, artifacts, registry claims, reverse-machine round
  trip, and digest idempotence. Do not replace the real Git transport with mocks.
- Confirm the already-covered negative paths remain green: disabled sidecar, private visibility propagation, missing
  machine identity, and declined repo-init consent.

## Phase 4: Validate the integrated epic

- Run the focused suites for agents sync, machine hoods, retry/edit prompt rewriting, tribe summaries, chat listing,
  hidden-sidecar inventory/open/path behavior, repo init, TUI agents-sync actions/indicator, and comprehensive update.
- Run `just test-visual` if any rendered tribe output or PNG expectation changes intentionally; inspect diffs before
  accepting a golden.
- Run `just check` after `just install`, as required for changes in the SASE repo. Fix failures without weakening the
  compatibility, privacy, or no-network-render contracts.
- Re-read the actual diff and verify no memory files, generated instruction shims, unrelated repositories, or temporary
  test artifacts were changed.

## Phase 5: Close and clean up epic `sase-8k`

This phase is last and must not begin until Phases 1-4 are complete and green.

1. Run `sase bead show sase-8k` one final time and confirm every child is closed, then run `sase bead close sase-8k`.
2. After the close, run `just symvision` if the recipe is available. The close expires epic-symbol whitelist entries.
   Before fixing any Symvision diagnostics, follow the required `/sase_memory_read` workflow for
   `sase/memory/symvision.md`; remove stale `sase-8k` whitelist entries and genuinely unused code, then rerun Symvision
   to green.
3. If Symvision cleanup changes SASE source or test files, rerun `just check` after that cleanup.
4. In the already-open plans sidecar, set only the epic plan frontmatter field to `status: done` in
   `$SASE_SDD_PLANS_DIR/202607/agents_sidecar_repo.md`.
5. Verify `sase bead show sase-8k` reports closed, the epic plan reports `status: done`, all relevant worktrees contain
   only the intended landing changes, and no `sase-8k` Symvision whitelist entry remains.

## Handoff notes

- Existing machines require a one-time `sase config init` after this feature lands.
- “Machine Agent Hood” and “Agents Sidecar” are natural glossary/memory additions, but do not make those edits without
  explicit user approval.
