---
tier: epic
title: Finish observation-window hardening and land sase-ix
goal: Every task +1 gets a valid observation-window instant or a safe current-time
  fallback, the canonical bead memory and deployed task-filing skill describe the
  shipped close-boundary behavior, and epic sase-ix is fully verified, integrated
  with later commits, and closed without stale Symvision exemptions.
phases:
- id: harden-observation-metadata
  title: Fall back safely for malformed observation metadata
  depends_on: []
  size: small
  description: 'harden-observation-metadata: validate agent_meta.json run_started_at
    before passing it to the Rust +1 mutation, fall back to a sub-second current instant
    with a debug diagnostic when it is malformed, and add direct and CLI regressions.'
- id: reconcile-contract-guidance
  title: Reconcile canonical docs and deployed plus-one guidance
  depends_on:
  - harden-observation-metadata
  size: medium
  description: 'reconcile-contract-guidance: update the generated canonical bead memory
    and public bead docs to state the observation-window close rule, run sase memory
    init, then deploy the already committed sase_new_task guidance from a clean merged
    tree and verify every active runtime receives it.'
- id: land-sase-ix
  title: Verify, close, and clean up epic sase-ix
  depends_on:
  - reconcile-contract-guidance
  size: medium
  description: 'land-sase-ix: rerun full Python and Rust verification, record every
    child-note and post-start integration outcome in the close note, close sase-ix,
    run Symvision after closure and remove anything it newly exposes, then mark the
    original and follow-up plans done.'
proposed_by: bbugyi200.athena.sase-ix.land
parent_bead: sase-ix
create_time: 2026-08-10 13:26:52
status: wip
bead_id: sase-ix.5
---

- **PROMPT:** [prompts/202608/finish_plus_one_reopen_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_plus_one_reopen_landing.md)
- **PARENT:** [202608/plus_one_post_close_reopen_race.md](https://github.com/sase-org/sase--plans/blob/main/202608/plus_one_post_close_reopen_race.md)
- **BEAD:** [sase-ix.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ix/sase-ix.5.md)

# Plan: Finish observation-window hardening and land sase-ix

Epic `sase-ix` fixes the race where a stale, already-in-flight `sase bead +1` could
reopen a task seconds after its worker closed it. The four original phases are closed,
and landing verification confirmed their main implementation is present and works:

- Rust core commit `d1a19d566` added optional `observed_since` evidence, validates it as
  RFC 3339, withholds stale closed-task reopens, clears the assignee on a real reopen,
  and applies the same decision in the event reducer. Core search commit `4f09d2774`
  made that evidence searchable. The later CI compatibility commit `86e4eb9a` retained
  the expanded PyO3 signature, and release commit `269928f50` published the result as
  `sase-core-rs` 0.24.0.
- Sase commits `47b2a74aa`, `187085a80`, and `43337c3f7` supply the observation window,
  expose `--verified-after-close`, render withheld corroboration across CLI/ACE/pages/
  gates, update the task-filing source skill, and reproduce the race against a real
  temporary store.
- Post-start commit `dcb243b75` correctly integrated the published core release by
  raising the Python dependency window to `>=0.24.0,<0.25.0`; it also refreshed the
  contract manifest and migrated the tale-size tests that earlier phase workers saw
  fail. Post-start commits `ebd3a91bc` and `128b326ea` only stabilize TUI waits and PNG
  fixtures. Commits `012e1a88b` (model-alias routing) and `3eddffba9` (commit-hook
  module split) landed during this audit; neither changes +1 mutation behavior, and the
  hook split preserves bead-close behavior, but `012e1a88b` touched `docs/beads.md`
  without integrating this epic's completed corroboration contract.

The landing audit ran the epic-focused Python suite (50 passed), the formerly failing
contract-manifest and tale-routing suite (10 passed), `just symvision`, `just check`
(which escalated to and passed the full non-visual suite), and
`cargo test -p sase_core -p sase_core_py`; all pass on the integrated tree.

Four epic-caused gaps remain and prevent an honest close:

1. `resolve_observation_window_start()` promises malformed agent metadata falls back to
   now, but any nonblank `run_started_at` is returned unchecked. A value such as
   `not-an-instant` reaches the Rust binding, fails its RFC-3339 validation, and aborts
   `sase bead +1` instead of taking the documented fallback.
2. `sase/memory/sase_beads.md` still states that every +1 on a closed task promotes it
   to ready. That is the old contract and directly contradicts this epic.
3. `src/sase/xprompts/skills/sase_new_task.md` contains the new withheld-reopen
   instructions, but `sase skill init --diff` shows they were never deployed to any of
   the six managed runtime destinations. The active Codex `/sase_new_task` skill is
   therefore stale.
4. `docs/beads.md` repeats the old unconditional rule in the Task Corroboration section
   and command reference, and says every closed-task +1 creates close history. A
   post-start model-alias commit touched this document without integrating the newly
   completed observation-window feature, so the public reference remains inaccurate.

## Fall back safely for malformed observation metadata

Work in the primary sase repository.

1. Tighten `resolve_observation_window_start()` in `src/sase/agent/identity.py`. Treat
   `run_started_at` as usable only when it parses as an ISO/RFC-3339 instant with an
   explicit offset (`Z` is valid). Blank, wrong-type, unparseable, or offset-less values
   are malformed provenance: emit the existing debug-level fallback diagnostic with
   enough detail to distinguish a missing field from an invalid one, then return
   `current_instant()`. Keep the returned valid string unchanged so its original
   precision and offset remain durable evidence.

2. Add direct unit coverage for a valid `Z` value, a valid non-UTC offset, malformed
   text, an offset-less timestamp, missing metadata, invalid JSON, and a human/no-agent
   environment. Freeze or monkeypatch `current_instant()` so fallback assertions are
   exact. Put the tests in a focused identity test module rather than making every case
   travel through the CLI.

3. Add one CLI-level regression in `tests/test_bead/test_cli_plus_one.py`: metadata
   identifies an agent but gives it a malformed `run_started_at`; `sase bead +1` must
   record evidence successfully using the mocked current instant instead of surfacing
   the Rust validation error. Preserve the existing tests for stale withholding,
   explicit `--verified-after-close`, and the human fallback.

4. Run `just install`, the focused identity/CLI tests, and `just check`. Do not change
   Rust validation: rejecting malformed wire data is correct; only the metadata
   resolver's promised best-effort fallback was incomplete.

## Reconcile canonical and deployed plus-one guidance

This phase has two generated-content workflows. Do not hand-edit generated shims or
installed skill files.

### Canonical bead memory

Approval of this plan is explicit owner authorization to update
`sase/memory/sase_beads.md` for this epic and to run the mandatory regeneration. Before
editing, read `sase_beads.md` through `/sase_memory_read` as required by project policy.

1. `sase/memory/sase_beads.md` is generated from
   `src/sase/main/init_memory/templates/memory-sase-beads.template.md`. Update the
   packaged source template; do not hand-edit generated instruction shims. Replace the
   old Task Beads sentence saying the first +1 on any closed task atomically promotes it
   to ready. State the shipped rule precisely:
   - an open draft still promotes;
   - a closed task promotes only when the reporter's observation window starts strictly
     after the current close, while missing provenance preserves legacy reopen behavior;
   - stale-window evidence is recorded but the close remains standing;
   - `--verified-after-close` is only for an actual reproduction on a tree containing
     the close;
   - claimed, ready, in-progress, and snoozed tasks retain their existing behavior.

2. Run `sase memory init` immediately after the template edit so the requested canonical
   `sase/memory/sase_beads.md` change and all derived files are generated together.
   Review the canonical note, `AGENTS.md`, provider instruction shims, and memory README
   diff for only the expected propagation. Never edit those generated outputs by hand.

3. Update every stale public reference in `docs/beads.md`, including Task Corroboration,
   Close History, and the `sase bead +1` command reference. Document
   `--verified-after-close`, the durable withheld-reopen note/output, the post-close
   evidence badge, `observed_since` in JSON, and that only a qualifying fresh +1
   archives close metadata and clears the assignee. Preserve the legacy `None`
   provenance behavior for non-CLI callers. Sweep all tracked Markdown again for the
   unconditional closed-task wording.

4. Add or update focused memory-template/documentation tests so future regeneration
   cannot restore the old sentence. Run `sase memory init --check` if supported in
   addition to the normal generation command; otherwise use the existing init-memory
   tests plus a clean second `sase memory init` as the idempotence check.

### Generated task-filing skill

Before deployment, read `generated_skills.md` through `/sase_memory_read` and open the
configured `chezmoi` repo with `/sase_repo` for the audit trail.

1. Confirm the source paragraph in `src/sase/xprompts/skills/sase_new_task.md` is still
   present and committed (introduced by `187085a80`). Do not rewrite it merely to make
   deployment happen.

2. The generator is global and shared, so deploy only from a clean primary tree whose
   `HEAD` is on the canonical merged branch. Run `sase skill init --diff` first, then
   `sase skill init --force`; run `chezmoi apply` if the generator reports it was
   skipped. Do not use `--allow-dirty` and do not hand-edit files under the chezmoi
   source or runtime skill destinations.

3. Re-run `sase skill init --diff` and verify the withheld-reopen paragraph appears in
   each generated provider destination and in the active Codex
   `/sase_new_task/SKILL.md`. Other already-committed canonical skill updates may deploy
   in the same run; record them rather than reverting them.

4. Run `just install`, focused memory/skill generation tests, and `just check`. Because
   memory regeneration changes files in the primary repo, include all intended generated
   outputs in the phase's normal SASE commit workflow.

## Verify, close, and clean up epic sase-ix

This final phase completes the landing interrupted by this follow-up plan.

1. Re-read `sase bead show sase-ix` and every child. Confirm the four gaps above are
   fixed, all children remain closed, and the original source/commit audit still holds.
   Re-run `git log` from `47b2a74aa^` to the current base and explicitly include
   `012e1a88b` and `3eddffba9` in the integration review so commits landing after this
   plan was authored cannot silently escape the same check. Run `just install`,
   `just check-full`, and in the `/sase_repo`-opened core checkout run
   `cargo test -p sase_core -p sase_core_py`.

2. Record every `PROPOSED FOLLOW-UP` outcome in the close note:
   - `sase-ix.1`'s core-version handoff and `sase-ix.2`'s Python pin proposal are one
     release-process concern. No task is warranted: `release-plz` published 0.24.0 in
     `269928f50`, and later primary commit `dcb243b75` raised the dependency window.
   - `sase-ix.2`'s unused `resolve_notification_tab_icon` finding was already tracked by
     ready task `sase-iz` and was removed by unrelated commit `c49452c47`; current
     Symvision passes. Do not add a +1 because the issue no longer reproduces.
   - `sase-ix.2`/`.4`'s contract-manifest and large/xlarge tale-routing failures were
     already tracked by `sase-iu`, duplicate `sase-iv`, and completed `sase-is`; later
     commit `dcb243b75` fixes the current instances and the exact tests pass. Do not add
     stale corroboration.
   - `sase-ix.3`'s 21 invalid committed tale plans were causally owned by active epic
     `sase-il.7`, whose phase `sase-il.7.2` migrated them in plans-sidecar commit
     `a91c3138`; committed-plan validation now passes. No standalone task is warranted.
   - `sase-ix.4`'s stale memory contract is caused by this epic and is completed by the
     preceding phase, not filed as a task. State explicitly that `/sase_new_task` was
     used during landing to search every task status, sweep the last week, and inspect
     active epics, and explain each no-new-task decision above.

3. Close the epic normally; never force it merely to bypass descendant validation:

   ```bash
   sase bead close sase-ix --note "<complete verification, integration, and follow-up outcomes>"
   ```

   If the close is rejected, finish or reopen the named descendant. Use
   `--force --reason ... --resolution canceled|superseded` only for a deliberately
   canceled or superseded descendant, never for convenience.

4. Only after the close succeeds, read `symvision.md` through `/sase_memory_read`, run
   `just symvision`, and remove expired `sase-ix` epic-symbol whitelist entries or
   unused code it reports. Rerun `just check` after any cleanup.

5. Set `status: done` in the frontmatter of
   `plans:202608/plus_one_post_close_reopen_race.md`, and also mark this follow-up plan
   file done. Verify both sidecar changes are durable.

## Verification expectations

- The observation-window resolver tests prove every malformed metadata shape falls back
  without weakening the Rust wire validator.
- The canonical memory and all generated runtime skills describe the same close-boundary
  rule as the Rust mutation and reducer.
- `just check-full`, the core test suites, and post-close `just symvision` pass on the
  final combined tree.
- The epic close note names the audited commits, later integration commits, live-store
  recommendation for `sase-ct`, and every proposal outcome; it must mention that the
  live-store audit intentionally left `sase-ct` ready because later fresh evidence
  independently justified that status.
