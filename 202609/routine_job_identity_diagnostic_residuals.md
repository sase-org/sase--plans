---
tier: epic
title: Close the residual job-tribe identity and routine/job diagnostic gaps
goal: Every persisted, targeted, and displayed job-tribe identity comes from one contextual
  resolution, every live routine/job diagnostic uses canonical wording at its owner
  without rewriting user data, and the combined upgrade passes the published-floor
  and full landing gates.
parent_bead: sase-11e.8.6.5
phases:
- id: tribe_writes
  title: Resolve job-tribe identity before any metadata or store write
  size: medium
  depends_on: []
  description: 'tribe_writes: %id, %clan, TUI, and proposal tribe writes persist the
    contextually resolved identity and reject collisions before touching any state.'
- id: tribe_evidence
  title: Share one stored-tribe evidence source across wait, fork, and display
  size: medium
  depends_on: []
  description: 'tribe_evidence: runner wait fast path, fork, completion, clan tribes,
    colors, and panel collapse resolve @job from the same evidence.'
- id: python_diagnostics
  title: Canonicalize the remaining live Python routine/job text at its owners
  size: medium
  depends_on: []
  description: 'python_diagnostics: replace the doctor blanket rewrite and fix residual
    chop/lumberjack wording in digests, doctor, SDK, logs, help, and editor.'
- id: core_diagnostics
  title: Canonicalize the remaining live Rust job validation text
  size: small
  depends_on: []
  description: 'core_diagnostics: job wording for sase-core axe_chop validation, target,
    and PyO3 request-label messages without changing codes or wire keys.'
- id: acceptance
  title: Prove the repaired contract and pass published-floor and full landing gates
  size: medium
  depends_on:
  - tribe_writes
  - tribe_evidence
  - python_diagnostics
  - core_diagnostics
  description: 'acceptance: extend the upgrade fixture, ratchet the core floor and
    pin after a published release, and run just check-full through a monitor.'
proposed_by: bbugyi200.athena.sase-11e.8.6.5.land
create_time: 2026-09-16 14:58:07
status: wip
bead_id: sase-11e.8.6.5.4
---

- **PROMPT:** [prompts/202609/routine_job_identity_diagnostic_residuals.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/routine_job_identity_diagnostic_residuals.md)
- **PARENT:** [202609/routine_job_identity_diagnostic_completion.md](https://github.com/sase-org/sase--plans/blob/main/202609/routine_job_identity_diagnostic_completion.md)
- **BEAD:** [sase-11e.8.6.5.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11e/sase-11e.8.6.5.4.md)

# Close the residual job-tribe identity and routine/job diagnostic gaps

## Scope and handoff

This plan contains only work found incomplete while landing `sase-11e.8.6.5`
(`plan:202609/routine_job_identity_diagnostic_completion.md`). Its parent plans
(`plan:202609/routine_job_final_contract_repairs.md`,
`plan:202609/axe_routine_job_landing_repairs.md`, `plan:202609/axe_routines_jobs.md`)
and their compatibility decisions remain in force. Read them and the landing notes on
`sase-11e.8`, `sase-11e.8.6`, and `sase-11e.8.6.5` through `sase artifact read` and
`sase bead show` before starting.

The `parent_bead: sase-11e.8.6.5` relationship is the mechanical handoff back to the
interrupted landing. Do not add closing that bead, its Symvision pass, or its plan
status update as work here.

Keep AXE, `sase axe`, the AXE tab, `sase tui`, scheduling/admission semantics, private
wire fields, schema-v1 keys, `.chop.` agent names, state directories, environment
variable names, graph identities, historical assignments, accepted legacy inputs,
internal diagnostic codes/ids, exact user names/paths/script names, and historical log
bytes intact. Keep the sunset flag `axe_routine_job_contract` and its removal bead
`sase-11f`. Shared backend/domain behavior belongs in Rust (`sase-core`); Python owns
file I/O, cached adapters, and presentation. Use `/sase_repo` before touching
`sase-core`. Before any TUI change read `tui_perf.md` and `src/sase/ace/AGENTS.md`;
never put configuration discovery, artifact scans, or assignment-file reads on a render,
completion, or event-loop path.

No live AXE start/stop, real job execution, Telegram send, deployment, or operator
configuration change is required. Tests use temporary config/state and mocked launches.

## Verified landing baseline

The landing audit used SASE `fb1e7f576b` and `sase-core` `f822ebd` (release v0.34.37).
`sase-core-revision.txt` pins `f822ebd5839cca0b41627d3c0294203aafa4401d` and
`pyproject.toml` requires `sase-core-rs>=0.34.37,<0.35.0` (published wheels exist). All
three child phases of `sase-11e.8.6.5` are closed with one note each and no
`PROPOSED FOLLOW-UP:` entries; `sase bead epic-symbols sase-11e.8.6.5` is empty.
Post-start commits (agent hold, gate intents, monitor recovery, agents-sync manifests,
prompt search counts, skill text) do not conflict with the feature. Their changes must
be preserved.

What landed and was verified: `ace/agent_tribes.set_tribe` and the `sase agent tribe`
CLI pass config layers and reject a same-/cross-layer `ace.tribes.chop`/`ace.tribes.job`
collision without writing the store; the fork path, TUI completion wait binding, and
query filter are context-aware; the listed Python AXE templates and TUI job-run
not-found/ambiguous errors are canonical; the upgrade fixture in `tests/test_axe_cli.py`
covers both flag states for list/doctor, the hidden `chop` alias, ambiguous run, target
expansion, env alias propagation and scrubbing, history, agent linkage, and status.

The audit then found these gaps, each reproduced against the current binding.

### Identity gaps

1. **Raw public `job` persisted as metadata.** `%id(tribe=job)` flows from
   `src/sase/axe/run_agent_directive_identity.py` (`agent_tribe = directives.tribe`)
   into `src/sase/axe/run_agent_directive_metadata.py`, which writes
   `agent_meta["tribe"] = "job"`. Meanwhile `update_agent_tribe` stores `chop`. The TUI
   tribe modal (`src/sase/ace/tui/actions/agents/_tribe_assignment.py`) writes
   `meta_set {"tribe": after}` raw as well. AXE proposal launch text
   (`src/sase/axe/chop_proposal_models.py`) emits `%id(..., tribe={proposal.tribe})`.
   The wait/fork index treats meta tribes as stored evidence. So after one public-alias
   launch, the index sees an "independent" `job` identity, and every later `@job` wait
   or fork stops matching the real `chop` agents. `%clan(..., tribe=job)` is only
   name-validated (`src/sase/xprompt/_directive_values.py`) and written raw as
   `clan_tribe`.
2. **Partial state on rejection.** In `src/sase/axe/run_agent_directives.py` the
   `%id tribe=` collision check (`update_agent_tribe(..., layers=...)`) runs only after
   `write_agent_meta` and name reservation, leaving `tribe: job` in `agent_meta.json`.
   The existing test `test_id_tribe_job_alias_collision_fails_the_launch` does not check
   the meta file. In the TUI N-key path,
   `src/sase/ace/tui/actions/agents/_directive_persistence.py` rewrites the prompt and
   patches meta before `_patch_agent_tribe_store` raises. `%clan(tribe=job)` and the TUI
   clan-tribe assignment never check for collisions.
3. **Wait and fork use different evidence.** The fork path
   (`src/sase/scripts/_agent_chat_from_name_tribe.py`) and the wait-check job
   (`src/sase/scripts/sase_chop_wait_checks.py`) use all-projects
   `WaitDependencyIndex.tribes`. The runner fast path
   (`src/sase/axe/run_agent_wait_deps.py`, `build_wait_dependency_index(project_name)`)
   sees one project only, so an independent `job` identity in another project gives
   different answers. Clan-level tribes (`clan_tribe`) are missing from the stored
   evidence: a clan with only `clan_tribe: job` is not bound by `@job` even though
   `tribe_candidate("job")` finds it, and adding any unrelated direct `job` agent flips
   the result. `set_tribe` sees only `agent_tribes.json` values, not a meta-only `job`.
4. **Display regressions.** `named_tribe_identity_colors` in
   `src/sase/ace/tui/models/tribe_display.py` now uses its own input as `stored_tribes`.
   Its caller `src/sase/ace/tui/modals/agent_neighbor_modal.py` passes public panel
   labels (`job` for the built-in `chop` panel), so with only `ace.tribes.chop.color`
   configured the neighbor modal now shows the fallback color. That is a regression from
   `9759e5afe8`. `effective_collapsed_panel_keys` (same file, `panel_keys is None`
   branch) canonicalizes configured names with no context, and `_tribe_target` in
   `src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py` uses context-free
   `parse_tribe_reference`, so a `@job` wait bound to an independent `job` agent is
   drawn in `chop`'s color even though the binding itself is correct.

### Diagnostic gaps

5. **Blanket rewrite in doctor.** `_public_axe_text` in `src/sase/doctor/checks_axe.py`
   (added in `d2d30944dc`) does whole-string `chop→job` / `lumberjack→routine`
   replacement on `axe.jobs` details and next steps. That rewrites diagnostic ids
   (`configured_chop:`), user script names (`sase_chop_foo`), user routine/job names,
   and legacy config paths (`axe.lumberjacks`). The parent plan forbids exactly this.
6. **Residual live Python text** still using the old vocabulary on canonical surfaces in
   both flag states:
   - AXE error digest label `Lumberjack:` in `src/sase/notifications/senders.py`; the
     dict key stays.
   - `no fresh chop probe:` in `src/sase/doctor/checks_external_mirror.py`.
   - `per-chop env` in `src/sase/axe/chop_doctor.py` (surfaces in `sase axe job doctor`
     text and public JSON `summary`).
   - "past chop budget" / "chop budget exceeded" in
     `src/sase/scripts/sase_chop_artifact_link_backfill.py`,
     `src/sase/sdd/_artifact_link_machine_store.py`, and
     `src/sase/sdd/_artifact_link_publication_retry.py` (job run log).
   - `ValueError` messages in `src/sase/chops/report.py`, reached through the public
     `sase.jobs.JobReport`.
   - `src/sase/logs/launch_log.py` renders `kind="chop"` from
     `src/sase/ace/tui/actions/axe_chop_run.py` as `chop launch failure:` plus
     `chop:`/`lumberjack:` context labels in `launch_failures.log`, which the TUI Logs
     pane shows. The stored JSONL `kind` value is a contract and must stay.
   - Help text in `src/sase/main/parser_bead_store.py` (`sync-external`) and
     `src/sase/main/parser_patch.py` (`set-origin`).
   - Newly generated mirrored-issue description text in
     `src/sase/external_mirror/_issue_planning.py`; never rewrite existing beads.
   - The AXE routine editor (`src/sase/ace/tui/modals/axe_entry_editor_types.py`) offers
     deprecated `chop_timeout` as a basic field while hiding `job_timeout`, and its
     invariant errors ("chop identity requires a chop name", "schema does not contain
     AXE lumberjack definitions", "an AXE {kind} definition") reach users as "Could not
     edit AXE config: …". The unused transaction title in `axe_entry_editor_modal.py`
     still builds `AXE {kind}`.
   - Classify, and change only if they are human text: Prometheus help strings in
     `src/sase/telemetry/metrics.py`, the `builtin chop` invariants in
     `src/sase/chops/builtin.py`, and `log.exception` text in
     `src/sase/agent/launch_spawn.py`. Leave workspace-lease workflow tags such as
     `chop:external_issue_mirror`, `.chop.` names, category/source tags, and legacy-only
     `sase.chops` SDK/`sase_chop_*` program help unchanged.
7. **Residual live Rust text** in `sase-core`:
   - `crates/sase_core/src/axe_chop/validation.rs`: "chop result is not valid JSON…",
     "chop result has an invalid field…", "chop result must be a JSON object", "only an
     `ok` chop result may propose agent launches", "…inside the chop run directory",
     "…not allowed in chop proposals…", "chop name must contain at least one letter or
     digit".
   - `crates/sase_core/src/axe_chop/targets.rs`: "chop name must not be blank", "chop
     name must not contain target-instance brackets". The second surfaces through
     `src/sase/axe/_config_targets.py` on every `sase axe job`/`routine` command and in
     the TUI's "Could not load AXE config" notice.
   - The `"chop result"` request label in `crates/sase_core_py/src/lib.rs`, which
     reaches `sase.jobs.write_job_result`/`validate_job_report` and the notification
     report view. The other internal request labels there ("chop decision request",
     "chop launch proposal", …) surface only on malformed internal requests: classify
     them and change only if reachable.

## Phase `tribe_writes`

Make every tribe write persist the identity the contextual resolver chooses, and make
rejection happen before any write.

- Resolve `%id(tribe=...)` once, at directive-identity time, through
  `canonicalize_public_tribe_name` / the Rust resolver, using config layers from
  `discover_layer_inputs()` and stored-tribe evidence from the same cached source the
  wait/fork index uses (coordinate with `tribe_evidence`: take evidence through a small
  shared adapter rather than a new scan). Write that resolved identity to both
  `agent_meta.json` and `agent_tribes.json`. An ordinary launch with no independent
  `job` identity stores `chop` in both. A same-name update of an existing historical
  `job` assignment keeps `job`. The raw prompt text stays byte-faithful.
- Move the collision check ahead of name reservation and `write_agent_meta`, so a
  rejected launch leaves no `agent_meta.json` tribe, no store entry, and no reserved
  name side effects beyond what an ordinary validation failure leaves.
- Apply the same resolution and early rejection to `%clan(..., tribe=...)`
  (`clan_tribe`), the TUI N-key tribe modal (`_tribe_assignment.py` and
  `_directive_persistence.py`: resolve and validate in the existing worker before the
  prompt rewrite or meta patch; report the diagnostic without optimistic persistence),
  TUI clan-tribe assignment, and AXE proposal launch text.
- Keep reserved-tribe behavior, clan aggregation, and `.chop.` agent names.
- Tests (production paths, temporary stores): `%id(tribe=job)` with no independent
  identity writes `chop` to meta and store; a later `@job` wait and fork still bind the
  real `chop` agents; a historical `job` stays `job`; `%id` and `%clan` collisions leave
  meta, store, and prompt files unchanged (extend
  `test_id_tribe_job_alias_collision_fails_the_launch` to assert the meta file); TUI
  N-key and clan-tribe collision paths leave the prompt, meta, and store unchanged.

## Phase `tribe_evidence`

Give wait, fork, completion, and display one definition of stored-tribe evidence.

- Define the evidence set once, preferably in Rust or a thin cached Python adapter over
  the existing wait index: direct meta tribes, `agent_tribes.json` assignments, and
  `clan_tribe` values, across all projects. Make the runner fast path in
  `src/sase/axe/run_agent_wait_deps.py` resolve the `@job` identity from that same
  evidence (for example a cached all-projects stored-tribe-name set, or the identity
  recorded when the wait was registered). Keep its per-project dependency scan for
  everything else, and do not add an uncached all-projects artifact scan per wait tick.
  Measure before and after if you touch a hot path.
- The fork path, the wait-check job, and TUI completion keep resolving from that shared
  definition. A clan with only `clan_tribe: job` must behave the same whether or not an
  unrelated direct `job` agent exists.
- Fix display: `named_tribe_identity_colors` must take the stored panel keys or explicit
  stored-tribe context from callers, not its own public labels. Update
  `agent_neighbor_modal.py` to pass stored identities, and restore the built-in `chop`
  color for the public `@job` label when no independent `job` exists. Make
  `_agent_wait_section._tribe_target` use the binding's resolved tribe (already in
  `tribe_wait_bindings`) or the loaded rows' stored tribes. Give
  `effective_collapsed_panel_keys`'s `panel_keys is None` branch the same context as the
  other callers (the caller in `_panel_hint_folding.py` already has the loaded agents).
  All display inputs must be already in memory.
- Tests: wait (runner fast path and wait-check job) and fork agree for an independent
  `job` identity in another project; clan-only `job` binding is stable; neighbor-modal
  color for `@job` with only `ace.tribes.chop.color` configured; wait-lane color for an
  independent `job` binding; collapse settings for `ace.tribes.job` with and without an
  independent `job` panel. Cover both the cheap syntactic path and the contextual
  semantic path.

## Phase `python_diagnostics`

Finish the Python reachable-message audit at each string's owner.

- Delete `_public_axe_text` from `src/sase/doctor/checks_axe.py`. Render canonical text
  at the owners (`src/sase/axe/chop_doctor.py` and its render helpers), using the
  existing surface/flag-aware rendering so that `sase doctor` and `sase axe job doctor`
  show canonical wording. Ids, script names, user names, and config paths must pass
  through unchanged. Legacy-only v1 output stays as it is.
- Fix every item under gap 6 at its owner. Where the backing key is a contract (digest
  dict key `lumberjack`, launch-log JSONL `kind`), change only the rendered label. In
  the routine editor, show `job_timeout` as the basic field while still editing an
  existing `chop_timeout` source key in place (source-preserving edits from
  `config/plan.rs`), and make the invariant errors canonical.
- Classify the remaining ambiguous strings listed in gap 6 and record the classification
  in the phase note.
- Tests: include routine/job/script names and config paths containing
  `chop`/`lumberjack` to prove they are not rewritten. Cover `sase doctor` and
  `sase axe job doctor` in both flag states, the hidden legacy alias, the error digest
  file, the launch-failure log renderer, `JobReport` errors, backfill warnings, help
  text, and editor fields and errors. Update visual goldens only after inspecting the
  actual/expected/diff output.

## Phase `core_diagnostics`

In `sase-core`, change the live human messages under gap 7 to job wording. Keep error
codes, paths, wire keys, and function/module names. Update Rust unit tests and PyO3
tests. Search the SASE repo for tests or goldens that assert the old strings, and update
them in the same change set against the rebuilt linked binding. Run core `just check`
(Python >=3.12 PyO3 coverage included) and SASE `just check`. Land the core change so
release-plz can cut the next patch release. Leave the floor and pin ratchet to
`acceptance`, and record the core commit in the phase note.

## Phase `acceptance`

- Extend or split the isolated upgrade fixture in `tests/test_axe_cli.py` (or a sibling
  file) to cover the repaired contract end to end:
  - `%id(tribe=job)` metadata/store identity, then the `@job` wait (fast path and
    wait-check) and the fork agreeing.
  - A clan-only `job`.
  - Collision rejection leaving files unchanged.
  - Doctor details preserving ids, script names, and legacy config paths.
  - A job result validation error and a bracketed target-name config error with
    canonical wording.
  - Both flag states with inherited `SASE_FEATURE_FLAGS` removed, asserting the resolved
    state.
- Recheck drift since the first `sase-11e.8.6.5` commit (`66e20c1c24`) on SASE and core,
  excluding this plan's commits, and integrate anything that should now use the repaired
  paths.
- Once the core change from `core_diagnostics` is in a complete published release, use
  the supported tools: `tools/ratchet_core_window --report-only`,
  `tools/probe_core_floor --advisory`, then the dependency-window ratchet and
  `just ratchet-core-revision`. Validate with `validate_sase_core_rs` and
  `validate_sase_core_rs_version`. Until the release is published, the missing wheel is
  a real landing blocker. A local editable build is not release proof.
- Run the focused suites, core `just check`, SASE `just check`, and schema, default, and
  completion validation. Run the affected narrow/wide visual tests and inspect any
  golden diff. Then run `just fix` inline, and run `just check-full` only through
  `/sase_monitor` with the `TESTING`/`TESTED` status pair and a mechanical continuation.
  Record the exact SASE/core revisions and the published-floor evidence. Do not claim
  success while the floor or any epic-caused failure remains.

## Follow-up disposition

No child of `sase-11e.8.6.5` proposed a follow-up. Every gap above is caused by the
routine/job contract work and stays epic work here. Keep the ancestor dispositions:
object-sharing wire alignment is resolved; missing chezmoi `busted` is tracked by
`sase-11m`; hidden-plan artifact attachment trouble is corroborated on `sase-10y`. Do
not create duplicates.
