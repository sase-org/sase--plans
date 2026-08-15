---
tier: tale
title: Finish the proc-shell repair integration and landing
goal:
  The published core floor supports the proc lifecycle and every post-start required
  binding, and epic sase-m9.2.1.6 is verified, closed, cleaned up, and marked done.
size: medium
proposed_by: bbugyi200.athena.sase-m9.2.1.6.land
bead: sase-m9.2.1.6
create_time: 2026-08-15 12:44:07
status: wip
---

- **PROMPT:**
  [prompts/202608/finish_proc_shell_repair_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_proc_shell_repair_landing.md)
- **PARENT:**
  [202608/finish_unified_proc_shell_platform.md](finish_unified_proc_shell_platform.md)
- **BEAD:**
  [sase-m9.2.1.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.2.1.6.md)

# Plan: Finish the proc-shell repair integration and land sase-m9.2.1.6

## Goal

Integrate the only post-start core capability that the completed proc-shell repair could
not account for, prove the published dependency floor supports both the unified proc
lifecycle and the newer provider-disable facade, then close epic `sase-m9.2.1.6`, run
its post-close Symvision cleanup, and mark its linked plan done.

## Verified starting point

- All three children of `sase-m9.2.1.6` are closed. Source and commit inspection
  confirmed `ca93686a6` raised the core floor to `0.27.4` and added a schema-v3 proc
  lifecycle probe, while `ffce3c842` made `wait_for_proc` periodically retry
  reconciliation and added repeated coverage for every settlement crash checkpoint.
- Independent verification on clean `master` `8902cb5e5` passed the three settlement
  regressions, 52 core-validator/proc-facade tests, 22 monitor-facade tests, all 307
  bindings against linked core `0.27.5`, the lifecycle validator, 14 Rust proc-store
  tests, and the PyO3 proc-store binding test.
- Phase `sase-m9.2.1.6.2`'s settlement-flake proposal is epic-caused work already
  completed by phase `.1`. Phase `.3`'s unrelated TUI cost-budget proposal was
  corroborated on exact duplicate task `sase-j0` with proposing-bead detail and evidence
  `file:explicit:bfd8e739aa810bf321521c79`; do not create another task.
- The earlier `just check-full` monitor `ssc7fa54rekp` passed all 30,373 tests and
  failed only the already-tracked `sase-j0` cost gate.

## Remaining integration

The only commit after this repair epic's first commit is `8902cb5e5`, which added the
Rust-backed provider-disable facade. It requires `provider_disable_clear`,
`provider_disable_get`, `provider_disable_set_relative`, and
`provider_disable_set_until`. Those bindings first ship in published
`sase-core-rs==0.27.5`, but `pyproject.toml` still declares
`sase-core-rs>=0.27.4,<0.28.0` and `uv.lock` still resolves `0.27.4`. Consequently,
`tools/probe_core_floor` reports `stale_actionable`. The later commit otherwise touches
only `src/sase/llm_provider/**` and its tests, so it neither duplicates nor conflicts
with the unified proc service.

1. Raise the `sase-core-rs` lower bound in `pyproject.toml` from `0.27.4` to `0.27.5`,
   preserve the `<0.28.0` ceiling, and refresh `uv.lock` from published artifacts so its
   requirement and resolved package are both `0.27.5`. Do not rely on the linked
   editable checkout as proof of the declared minimum.
2. Run `just install`, then verify `tools/check_sase_core_rs_bindings`,
   `tools/validate_sase_core_rs`, and `tools/probe_core_floor`; the floor probe must be
   green and must cover both the five proc lifecycle bindings and all four
   provider-disable bindings. Run the focused proc settlement/facade, provider-disable,
   validator, and monitor-facade tests. Re-audit commits newer than `8902cb5e5` before
   landing and integrate any additional required core capability rather than merely
   ratcheting to a now-stale version.
3. Commit the dependency-floor integration through the required SASE commit workflow.
   Run `just check`; because dependency metadata is a broad verification surface, run
   `just check-full` only through `/sase_monitor` with a concrete `--next` action when
   the scoped gate escalates or exhaustive revalidation is required. Treat a recurrence
   of the already-corroborated `sase-j0` post-pytest cost-budget failure as that task's
   evidence, not as permission to ignore a functional test failure.

## Final phase: land the epic

1. Re-show `sase-m9.2.1.6` and its three children and confirm no new notes or open
   descendants appeared. Close exactly this epic without force using
   `sase bead close sase-m9.2.1.6 --note "<verification, integration, test, and follow-up disposition summary>"`.
   The close note must identify both repair commits, the post-start `8902cb5e5`
   integration and final core floor, the focused/full-suite evidence, the addressed
   settlement proposal, and the `sase-j0` duplicate outcome.
2. After the close, read `symvision.md` through `/sase_memory_read`, run
   `just symvision`, and remove only stale `sase-m9.2.1.6` epic-symbol whitelist entries
   and genuinely unused code it reports. If this changes repository files, run
   proportionate verification and commit the cleanup through the required SASE commit
   workflow.
3. Add `status: done` to the YAML frontmatter of the epic's linked plan,
   `/home/bryan/.sase/plans/202608/finish_unified_proc_shell_platform.md`, and verify
   `sase bead show sase-m9.2.1.6` reports it closed. Do not force-close unfinished
   descendants, and do not close parent `sase-m9.2.1`; its own land agent becomes
   unblocked after this child epic closes.
