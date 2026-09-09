---
tier: epic
title: Finish and land the unified proc-shell platform
status: done
goal:
  Repair the two epic-caused landing blockers, integrate the published Rust core
  lifecycle into the Python dependency floor, exhaustively verify the unified proc
  service, and close sase-m9.2.1 with every proposed follow-up durably dispositioned.
phases:
  - id: stabilize-settlement-recovery
    title: Make crash-boundary settlement recovery deterministic
    depends_on: []
    size: medium
    description:
      "stabilize-settlement-recovery: root-cause and fix the reproducible race in
      tests/test_procs_service.py::test_settlement_resumes_after_an_injected_crash.
      Current master 6683d4bcc fails on the second focused invocation after the injected
      supervisor exits at output_closed: reconcile_running_procs can leave the row
      nonterminal until wait_for_proc times out. Inspect supervisor process identity,
      zombie/reparenting behavior, settling ownership, and reconciliation timing rather
      than weakening the timeout. Preserve the contract that a dead supervisor is
      detected, every settlement checkpoint resumes idempotently, and the row reaches a
      durable terminal state. Add a deterministic regression that exercises the actual
      race and stress every injected settlement checkpoint repeatedly. Run just install,
      focused proc service/supervisor/settlement suites, and just check. Record any
      unrelated failure as PROPOSED FOLLOW-UP on this phase instead of creating a task."
  - id: integrate-published-core-floor
    title: Require the published proc lifecycle bindings
    depends_on: []
    size: small
    description:
      "integrate-published-core-floor: raise the sase-core-rs dependency floor from
      0.27.2 to the first published release containing reserve_proc,
      claim_proc_supervisor, request_proc_stop, begin_proc_settlement, and finish_proc
      (phase notes identify 0.27.3), retain the <0.28.0 ceiling, and refresh uv.lock
      against published packages rather than relying on the linked editable 0.27.4
      checkout. Verify the core-floor probe, binding validation, package metadata, and
      Python wire/facade tests against the declared minimum. Audit all schema-v3
      capabilities used by the Python proc service so the floor covers the complete
      runtime API, not only importable names. Run just install and just check. Record
      unrelated failures as PROPOSED FOLLOW-UP on this phase."
  - id: land-unified-proc-platform
    title: Re-audit, verify, and close sase-m9.2.1
    depends_on:
      - stabilize-settlement-recovery
      - integrate-published-core-floor
    size: medium
    description:
      "land-unified-proc-platform: perform the final land-agent audit for epic
      sase-m9.2.1. Re-read the epic, all five original child beads, every note, the
      original plan at plan:202608/unified_proc_shell_platform_1.md, the Rust and Python
      implementations, and every bead-tagged commit (Rust 6d7000a; Python 11072ba5d,
      152268b59, 1e242aa8b, 8b4635ad1, and 6683d4bcc), plus the commits from the new
      repair phases. Re-run the since-start integration audit and integrate any newer
      default-branch changes that should consume or conflict with the unified proc
      service; the initial audit found no proc integration needed in unrelated commits
      66145e553, 2f9b59cad, a14f22809, 718357102, 682cc31b3, 368e8f664, 545cb8e70,
      8f6c7eccb, and 41977629d, while the latter two independently exposed the
      settlement race. Confirm all acceptance criteria across named shells, monitor
      facade, legacy rows, ids/logs/family projection, concurrent reservation,
      replay/conflicts, stop/timeouts/reboot/pid reuse, every settlement crash boundary,
      follow-up exactly-once behavior, claim transfer/release, and retention ownership.
      Run just install, focused Rust and Python suites, then just check-full only
      through /sase_monitor with a --next action as required; run visual tests only if
      ACE rendering changed. Collect every new PROPOSED FOLLOW-UP and use /sase_new_task
      for each genuinely distinct issue not caused by this epic. Preserve the already
      recorded outcomes: Rich FORCE_COLOR assertion failures from phases .2/.3/.4
      corroborate ready task sase-m7, and the implicit phase-agent monitor targeting
      proposal from phase .5 corroborates ready task sase-ll; the dependency-floor and
      settlement proposals are epic work completed by the preceding phases. Write all
      outcomes and verification evidence into the close note, then close with `sase bead
      close sase-m9.2.1 --note ...` without force unless a deliberate canceled or
      superseded resolution is genuinely required. After close, run just symvision,
      remove only stale sase-m9.2.1 epic-symbol entries and unused code it reports, run
      the proportionate verification again, and add `status: done` to the frontmatter of
      /home/bryan/.sase/plans/202608/unified_proc_shell_platform_1.md."
proposed_by: bbugyi200.athena.sase-m9.2.1.land
parent_bead: sase-m9.2.1
bead_id: sase-m9.2.1.6
create_time: 2026-09-09 19:50:24
---

- **PROMPT:**
  [prompts/202608/finish_unified_proc_shell_platform.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_unified_proc_shell_platform.md)
- **PARENT:**
  [202608/unified_proc_shell_platform_1.md](unified_proc_shell_platform_1.md)
- **BEAD:**
  [sase-m9.2.1.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m9/sase-m9.2.1.6.md)

# Plan: Finish and land the unified proc-shell platform

## Audit findings

All five original phases are closed and their reported implementation commits are
present. The Rust lifecycle landed in `sase-core` commit `6d7000a`; the Python facade,
supervisor service, named-shell CLI, monitor facade, and documentation landed in commits
`11072ba5d`, `152268b59`, `1e242aa8b`, `8b4635ad1`, and `6683d4bcc`. Source inspection
confirms the intended architecture exists: schema-v3 reserve/claim/stop/settle/finish
operations are Rust-backed, ordinary proc submission uses `ProcSubmitRequest`, monitor
submission projects onto the same proc id and settlement path, named shells have their
own qualification and resolution layer, and legacy supervisors remain a compatibility
path.

The epic is not yet landable. First, `pyproject.toml` and `uv.lock` still declare and
lock `sase-core-rs` 0.27.2 even though the new lifecycle functions first ship in 0.27.3;
the local editable install succeeds only because it builds linked core 0.27.4. Second,
the crash-recovery test is genuinely flaky even in isolation: during this audit it
passed once and then failed on the next invocation with a ten-second `wait_for_proc`
timeout, matching three phase/epic reports and proving the settlement recovery contract
is not deterministic.

## Follow-up disposition already completed

The repeated Rich/ANSI failures are unrelated to this epic and semantically duplicate
ready task `sase-m7`; this land agent added independent evidence identifying proposing
phases `sase-m9.2.1.2`, `.3`, and `.4`. The implicit monitor-target failure proposed by
`sase-m9.2.1.5` is also pre-existing and semantically duplicates ready task `sase-ll`;
the landing audit reproduced `agent_family_base("sase-m9.2.1.5") == "sase-m9.2.1"` and
added that evidence. No new task is warranted for either proposal.

## Integration boundary

The post-start commits listed in the final phase are query-profile, model-selection,
test-organization, memory, and documentation work. None adds a proc/monitor writer or
duplicates the unified service, so the initial source and diff audit found no direct
integration edit. Two of those unrelated commits did provide independent full-suite
reproductions of the settlement race. The final phase repeats this audit because more
commits may land while the repair phases run.

## Landing standard

The epic closes only after the dependency contract works against published artifacts,
settlement recovery is deterministic under repeated crash-boundary stress, focused Rust
and Python coverage passes, and the governed exhaustive verification succeeds. Closing,
post-close Symvision cleanup, and marking the original epic plan done are one atomic
landing responsibility in the final phase.
