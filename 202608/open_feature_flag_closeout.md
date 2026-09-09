---
tier: epic
title: Close out the seven unowned SASE feature flags
goal: "The seven in-scope SASE feature flags are retired only after their authored
  acceptance evidence is satisfied, their winning behavior is made unconditional, and
  their dedicated flag beads are closed without disturbing the separately owned
  artifact_links and pluggable_finalizers work.

  "
phases:
  - id: fast_retirements
    title: Retire the mature formatter and plugin catalog paths
    depends_on: []
    size: medium
    description:
      "fast_retirements: verify and retire prettier_enabled and
      plugin_catalog_scoped_latest, including compatibility cleanup, schema
      synchronization, focused coverage, and bead closure."
  - id: completion_soak
    title: Prove update-time completion refresh across supported shells
    depends_on: []
    size: medium
    description:
      "completion_soak: gather reproducible unmanaged-install evidence for
      completion_refresh_on_update across bash, fish, and zsh without changing the flag
      definition."
  - id: epic_resume_soak
    title: Prove EpicResume behavior under real stall and handoff races
    depends_on: []
    size: medium
    description:
      "epic_resume_soak: exercise epic_resume_gate against controlled stalls, recovery,
      retries, and handoff races and record whether its authored removal gate passes."
  - id: planner_chat_trial
    title: Measure inherited planner chat value and cost
    depends_on: []
    size: medium
    description:
      "planner_chat_trial: compare paired coder handoffs with and without
      coder_inherits_planner_chat and produce an evidence-backed product recommendation."
  - id: shared_clone_audit
    title: Audit commit-finalizer shared-clone exemptions
    depends_on: []
    size: medium
    description:
      "shared_clone_audit: establish attributable shared-clone race evidence after the
      separately owned finalizer work lands and determine whether
      commit_finalizer_shared_clone_exempt is safe to retire."
  - id: ref_sync_observation
    title: Complete the two-release ref-sync gesture observation gate
    depends_on: []
    size: small
    description:
      "ref_sync_observation: verify the required two-minor-release incident-free window
      for ref_sync_gesture and keep the phase open until real release evidence satisfies
      it."
  - id: completion_retirement
    title: Make completion refresh unconditional
    depends_on:
      - completion_soak
    size: small
    description:
      "completion_retirement: remove completion_refresh_on_update after its soak passes
      while preserving managed-file skips, update success, failure isolation, and bead
      integrity."
  - id: epic_resume_retirement
    title: Make EpicResume gating unconditional
    depends_on:
      - epic_resume_soak
    size: small
    description:
      "epic_resume_retirement: remove epic_resume_gate after its operational evidence
      passes while preserving settle-window, deduplication, cancellation, and retry
      safeguards."
  - id: planner_chat_retirement
    title: Resolve the planner-chat experiment into a durable behavior
    depends_on:
      - planner_chat_trial
    size: medium
    description:
      "planner_chat_retirement: apply the approved planner-chat disposition, remove
      coder_inherits_planner_chat, and close its bead with the trial evidence and
      decision recorded."
  - id: shared_clone_retirement
    title: Make safe shared-clone race classification unconditional
    depends_on:
      - shared_clone_audit
    size: small
    description:
      "shared_clone_retirement: remove commit_finalizer_shared_clone_exempt only after
      the audit passes and retain fail-closed coverage for genuine discarded work."
  - id: ref_sync_retirement
    title: Make the ref-sync gesture unconditional
    depends_on:
      - ref_sync_observation
    size: small
    description:
      "ref_sync_retirement: remove ref_sync_gesture only after its two-release gate
      passes and retain the gesture's input-safety and responsiveness coverage."
  - id: integrated_closeout
    title: Reconcile the combined registry and verify the closeout
    depends_on:
      - fast_retirements
      - completion_retirement
      - epic_resume_retirement
      - planner_chat_retirement
      - shared_clone_retirement
      - ref_sync_retirement
    size: medium
    description:
      "integrated_closeout: reconcile concurrent registry and schema edits, verify all
      seven in-scope definitions and branches are gone, and run whole-repository
      acceptance without touching the two excluded flags."
proposed_by: bbugyi200.athena.09i
bead_id: sase-ru
create_time: 2026-09-09 19:51:00
status: wip
---

- **PROMPT:**
  [prompts/202608/open_feature_flag_closeout.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/open_feature_flag_closeout.md)
- **BEAD:**
  [sase-ru](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ru/README.md)

# Plan: Close out the seven unowned SASE feature flags

## Scope and outcome

This is a staged closeout epic for the seven open flag beads that do not already have
owners:

| Flag                                   | Bead      | Kind/default | Required outcome                                                                                                                          |
| -------------------------------------- | --------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `prettier_enabled`                     | `sase-qf` | sunset / On  | Keep formatter fallback behavior, remove the formatter escape hatch, and make the On path unconditional.                                  |
| `plugin_catalog_scoped_latest`         | `sase-qq` | beta / Off   | Prove the scoped online/offline behavior and make installed-only eager lookup plus highlighted-row lazy lookup unconditional.             |
| `completion_refresh_on_update`         | `sase-qg` | beta / Off   | Prove unmanaged bash/fish/zsh refreshes and make successful-update refresh unconditional.                                                 |
| `epic_resume_gate`                     | `sase-qh` | beta / Off   | Prove real-stall gating without retry/handoff false positives and make the guarded chop path unconditional.                               |
| `coder_inherits_planner_chat`          | `sase-qe` | beta / Off   | Resolve the experiment using measured plan fidelity and context cost, then replace the temporary flag with the approved durable behavior. |
| `commit_finalizer_shared_clone_exempt` | `sase-qi` | sunset / On  | Prove shared-clone exemptions do not mask discarded work and make the safe classification unconditional.                                  |
| `ref_sync_gesture`                     | `sase-qu` | sunset / On  | Honor the authored two-minor-release, incident-free gate and then make the gesture unconditional.                                         |

`artifact_links` / `sase-rc` and `pluggable_finalizers` / `sase-ro` are explicitly out
of scope because other agents own them. No phase may edit their behavior, remove their
registry entries, or close their beads. Because those owners and this epic all touch the
feature-flag registry and generated schema, every phase must refresh its inventory from
`sase flag list -j` immediately before editing, and the landing work must regenerate the
schema from the finally merged registry rather than accepting any phase's generated
block blindly.

The source tree is currently version `0.16.0`. `ref_sync_gesture` shipped on 2026-08-19,
and its bead requires two minor releases without an accidental-colon report. Therefore
that flag is not eligible for removal today. This plan deliberately keeps its
observation phase incomplete until the v0.16 and v0.17 evidence exists; neither tests
nor a simulated clock may substitute for the release criterion. The other evidence
phases follow the same rule: failed or insufficient evidence is recorded on the existing
flag bead and blocks its dependent removal phase rather than producing a false close.

## Shared execution rules

Before changing a flag, inspect its live definition, call sites, task fields, history,
effective overrides, and relevant source-controlled configuration. Treat the bead's
`remove_when`, `when_enabled`, and `when_disabled` fields as acceptance criteria. Record
the exact commands, versions, dates, event identifiers, and results of operational
checks with `sase bead note`; do not replace existing notes. If another owner already
removed a supposedly in-scope definition, reconcile and verify that work instead of
reintroducing or duplicating it.

For each actual retirement, keep registry, implementation, generated schema, tests,
documentation, compatibility aliases, and the dedicated bead closure in the same phase
change:

1. Delete the disabled branch and make the accepted enabled behavior unconditional,
   except for the explicitly approved planner-chat experiment disposition described
   below.
2. Remove the `FeatureFlag` member and registry definition, then run
   `tools/sync_feature_flags_schema --write`.
3. Remove stale CLI/help/config/docs references and migrate tests from both-state
   coverage to unconditional behavior plus retained failure/fallback invariants.
4. Run `tools/check_feature_flags` before and after
   `sase bead close <id> --note "<verified evidence and resulting behavior>"`; do not
   close a bead while its definition or disabled branch survives.
5. Run `just install`, focused tests for the affected domain, and `just check`. A phase
   that uncovers unrelated work records `PROPOSED FOLLOW-UP:` on its own phase bead
   instead of creating a task bead.

Do not rewrite the feature-flag framework merely because its registry becomes smaller;
the two excluded flags may still need it. Preserve unrelated user changes and avoid
editing machine-wide user configuration unless a checked-in compatibility migration
explicitly requires it.

## Phase: `fast_retirements` — formatter and plugin catalog

Retire `prettier_enabled` first. Confirm no supported workflow still requires
`SASE_DISABLE_PRETTIER`; replace test-suite uses of that environment alias with explicit
prettier availability/failure fakes so deterministic tests do not depend on a production
escape hatch. Remove the legacy environment mapping and runtime guard while retaining
the current no-op fallback when prettier is missing, returns an error, or cannot produce
usable output. Update formatter docstrings, resolver/doctor/CLI diagnostics, and focused
formatter and markdown-width tests. Run the formatter-focused tests and the ACE tests
whose fixtures previously exported the alias; run `just test-visual` if any committed
visual fixture or rendering setup changes.

For `plugin_catalog_scoped_latest`, exercise both online and offline behavior for
`sase plugin list`, `sase plugin show`, and Updates > Plugins before editing. Confirm
that installed plugins receive eager latest-version enrichment, an uninstalled selected
row is fetched lazily through the existing debouncer, offline mode performs no network
work, explicit all-latest behavior remains intentional, and catalog-size work stays
bounded. Then make the scoped path unconditional, remove the full-catalog default and
TUI flag guard, preserve public explicit-scope APIs where they remain useful, and update
focused CLI/TUI tests. Run `just plugin-catalog-scale-check` as an acceptance gate.

After each definition is removed and its checks pass, close `sase-qf` and `sase-qq` with
separate evidence-rich close notes.

## Phase: `completion_soak` — update-time completions

Use disposable, explicitly passed directories rather than the operator's real home.
Install stamped, unmanaged completion scripts for bash, fish, and zsh, then exercise
several genuine successful update cycles. Cover regeneration from the new version,
stale-script replacement, zsh compilation and restamping, idempotence, mixed installed
and absent shells, and a failure in each shell that remains nonfatal to the update and
does not suppress the other shells. Separately verify that chezmoi-managed files are
still recognized and skipped; the current process-level enablement is not evidence for
unmanaged files.

Prefer the production command path with controlled update/version dependencies over
calling only unit-level helpers. Add narrowly scoped observability or a reusable test
harness if the outcome cannot otherwise be attributed, but do not weaken the managed
file protection. Add the complete evidence to `sase-qg`. This phase passes only after
multiple successful cycles on all three supported shells and the failure-isolation case.

## Phase: `completion_retirement` — unconditional refresh

After `completion_soak` passes, remove the early return in
`maybe_refresh_installed_completions` and make the post-success update hook
unconditional. Retain the report model, per-shell isolation, nonfatal top-level
fallback, stamps, zcompile behavior, and chezmoi skips. Convert tests that asserted the
flag's Off state into tests for update eligibility and preserved failure behavior,
retire the registry/schema/help surfaces, and close `sase-qg` with the soak evidence
referenced in the close note.

## Phase: `epic_resume_soak` — stalls, retries, and cancellation

Enable the feature for a controlled disposable SASE project and ensure the Axe process
actually inherits the value. Use the existing fakey/scenario support or another
controlled provider to create at least one genuine failed phase that remains stalled
beyond `bead.epic_resume.settle_seconds`. Verify one `EpicResume` gate per failed
generation, correct preview/action payloads, successful resume through `sase bead work`,
and cancellation or suppression when recovery occurs before the gate settles. Exercise
fast retry, normal handoff, repeated chop ticks, unreadable inventory, and concurrent
recovery so none produce a false or duplicate gate. Record durable gate, agent,
generation, and timing identifiers on `sase-qh`; unit coverage alone does not satisfy
the authored operational gate.

## Phase: `epic_resume_retirement` — unconditional stalled-epic gating

Once the soak passes, delete only the `flag_disabled` early return from the chop and
remove beta-specific help/configuration. Preserve fail-closed inventory/scan handling,
the settle window, state-file deduplication, recovery cancellation, and notification
priority. Update the chop, gate, stall-policy, and checkpoint tests to describe the
unconditional contract, retire the registry/schema entry, and close `sase-qh` with the
operational event evidence.

## Phase: `planner_chat_trial` — product decision evidence

Run several matched plan-to-coder handoffs that vary only whether the coder inherits the
planner chat. Include small and medium plans and at least one long/noisy planner
conversation. Compare prompt/context size, token cost, adherence to the approved plan,
unnecessary repetition, implementation correctness, and reviewer rework. Use stored run
metadata and artifacts rather than subjective recollection, and record the trial matrix
on `sase-qe`.

Recommend one of three durable outcomes:

- Promote On only if inherited context produces a consistent material fidelity benefit
  at an acceptable typical and worst-case context cost.
- Preserve the plan-file-only behavior and abandon the beta if inherited context has no
  material benefit or causes regressions; because this rejects the authored On path,
  require an explicit owner decision and use an accurate canceled/abandoned close
  reason.
- Replace the flag with a normal, documented config field if results vary legitimately
  by user or workflow; use the flag lifecycle's Keep disposition and retain a deliberate
  default.

If the evidence does not yield a clear universal winner, pause through `/sase_questions`
for the product decision. The trial phase must not silently turn a preference into an
unconditional behavior.

## Phase: `planner_chat_retirement` — durable handoff behavior

Implement the approved outcome in `run_agent_exec_plan_accept`: unconditional `#fork`,
unconditional fresh coder context, or a durable config-controlled branch. Remove
`coder_inherits_planner_chat` from the temporary flag registry and schema in all cases,
update follow-up prompt and fork-history tests, document any permanent config field in
the default config/schema, and close `sase-qe` with both the measured trial and the
owner decision. Do not leave two independent controls for the same choice.

## Phase: `shared_clone_audit` — commit-finalizer evidence

Begin only after the separately owned `pluggable_finalizers` work is integrated into the
phase's base; otherwise record the prerequisite and leave the phase pending. Audit
existing finalizer logs for attributable shared-clone exemptions. If they cannot answer
which repository transition was exempted and why, add a narrow structured event/counter
that records repository kind, before/after HEAD, upstream-ahead state, attribution
class, and final classification without leaking sensitive paths or content.

Generate several controlled real shared-clone races covering foreign-agent commits,
already-published transitions, and pending-publication transitions in opened-external
and SDD-kind repositories. For each, prove from git provenance that no current-agent
work was discarded. Also create negative controls for local current-agent work, main
workspace repositories, unattributable commits, and genuinely ahead/unpublished work;
these must remain fail-closed. Record event identifiers and conclusions on `sase-qi`.
Any masked genuine discard fails the gate and blocks retirement.

## Phase: `shared_clone_retirement` — unconditional safe classification

After the audit passes, rebase once more over finalizer work, remove
`_shared_clone_exemption_enabled` and its strict flag-Off branch, and make only the
proven opened-external/SDD race classifications unconditional. Preserve the negative
controls and defensive fallback semantics. Remove temporary diagnostic instrumentation
unless it has lasting operational value, update hidden-agent/publication/resume
provenance coverage, retire the registry/schema entry, and close `sase-qi` with the
audited event evidence.

## Phase: `ref_sync_observation` — real release evidence

Track v0.16 and v0.17 release history, issue reports, bead notes, and support feedback
for accidental consumption of a literal second colon, incorrect kind selection, stale
rows, failed sidecar refresh, or input-path stalls. Exercise the enabled gesture across
clone-if-missing, TTL-expired force-pull, fresh catalog rescan, offline/failure
recovery, unknown kinds, nonempty payloads, non-insert modes, and rapid repeated input.
Preserve timestamps and release versions in `sase-qu` notes.

Do not complete this phase before the project reaches v0.18 eligibility and both prior
minor release windows are genuinely observable. A report of accidental colon consumption
or a responsiveness regression blocks removal; update the existing flag
threshold/evidence deliberately rather than closing it.

## Phase: `ref_sync_retirement` — unconditional gesture

After the release gate passes, remove the feature-flag check from
`_artifact_ref_sync_trigger` while retaining all semantic guards: insert mode, prompt
mode, empty payload, cursor position, and known kind. Remove disabled-state/fallback
prose and tests, but retain trigger, flow, panel, error, and rapid-input responsiveness
coverage. Run focused widget tests and `just test-visual` if rendering changes, retire
the registry/schema entry, and close `sase-qu` with the two-release evidence.

## Phase: `integrated_closeout` — landing and final acceptance

Reconcile phase changes with the latest base and with the two separately owned flag
removals. Resolve registry/schema/test conflicts semantically: keep any correctly landed
external removal, never resurrect a removed flag, and never close an externally owned
bead on this epic's behalf. Regenerate `src/sase/config/sase.schema.json` once from the
final registry and update generated completion/help snapshots only when their source
surface actually changed.

Final acceptance requires:

- `sase flag list -j` contains none of the seven in-scope keys and reports no integrity
  diagnostics; only still-live, separately owned flags may remain.
- `sase bead list -T flag -s all -f json -n 0` shows `sase-qe`, `sase-qf`, `sase-qg`,
  `sase-qh`, `sase-qi`, `sase-qq`, and `sase-qu` closed with evidence-rich resolutions.
- `rg` finds no production `FeatureFlag` member, guard, compatibility alias, beta help,
  or stale schema property for an in-scope key. Historical bead notes and tests of the
  generic feature-flag framework may retain names only when intentionally fixture-local.
- `tools/check_feature_flags`, `just plugin-catalog-scale-check`, and every focused lane
  named above pass on the combined tree.
- After `just install`, run `just check-full` through `/sase_monitor` with
  `TESTING`/`TESTED` statuses and a follow-up that inspects the result. Resolve caused
  failures before landing; record unrelated failures as phase follow-ups under the epic
  rules.

The epic is complete only when the combined code, generated schema, behavioral tests,
operational evidence, and seven bead closures agree. An unmet evidence gate leaves the
relevant phase and flag bead open; elapsed deadlines alone never convert that into a
successful closeout.
