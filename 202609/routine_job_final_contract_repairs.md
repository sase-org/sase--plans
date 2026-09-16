---
tier: epic
title: Finish routine/job source edits, tribe safety, and landing integration
goal: Complete the remaining routine/job compatibility contracts with reproducible
  production-path acceptance.
parent_bead: sase-11e.8
phases:
- id: source_edits
  title: Preserve existing AXE source structure for generic edits
  depends_on: []
  description: 'source_edits: repair inherited-leaf and list-form source editing through
    the shared Rust planner and Python apply adapter.'
  size: medium
- id: tribe_safety
  title: Enforce the shared automation tribe collision contract
  depends_on:
  - source_edits
  description: 'tribe_safety: connect provenance-aware Rust resolution to config validation,
    assignment, filtering, references, and display without changing historical identities.'
  size: medium
- id: diagnostics
  title: Finish canonical diagnostic templates without changing user data
  depends_on:
  - tribe_safety
  description: 'diagnostics: update live routine/job diagnostics at their owning templates
    and preserve authored paths and opaque payloads.'
  size: medium
- id: acceptance
  title: Align the CI core pin and prove the combined upgrade contract
  depends_on:
  - diagnostics
  description: 'acceptance: integrate the actual pinned Rust dependency, add missing
    production-path acceptance, review drift, and run complete repository verification.'
  size: medium
proposed_by: bbugyi200.athena.sase-11e.8.land
create_time: 2026-09-16 06:01:32
status: wip
bead_id: sase-11e.8.6
---

- **PROMPT:** [prompts/202609/routine_job_final_contract_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/routine_job_final_contract_repairs.md)
- **PARENT:** [202609/axe_routine_job_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/axe_routine_job_landing_repairs.md)
- **BEAD:** [sase-11e.8.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11e/sase-11e.8.6.md)

# Finish the remaining routine/job contracts

## Scope and handoff

This plan contains only work still incomplete during the landing of `sase-11e.8`. Read
`plan:202609/axe_routine_job_landing_repairs.md`, its parent
`plan:202609/axe_routines_jobs.md`, and the new landing audit note on `sase-11e.8`
through audited artifact/bead commands. Their compatibility decisions remain in force.

The `parent_bead: sase-11e.8` link is the mechanical handoff back to the interrupted
landing. Do not add closing this parent, its Symvision pass, or its plan status update
as an implementation phase. The eventual land agent must recheck this epic and then the
directly parented plan ancestors normally; never force a successful landing.

Keep AXE, `sase axe`, the AXE tab, `sase tui`, scheduling/admission semantics, private
wire fields, `.chop.` agent names, state directories, graph identities, historical
assignments, accepted legacy inputs, and opaque user values intact. Keep the sunset flag
`axe_routine_job_contract` and its independent removal bead `sase-11f`.

Use `/sase_repo` before accessing `sase-core` or another non-primary repository. All
paths here are relative to the named repository, with SASE as the default. Shared
behavior belongs in Rust; Python owns file I/O, adapters, and presentation. Before
changing TUI paths read `tui_perf.md` and `src/sase/ace/AGENTS.md`. Tests must use
temporary config/state and mocked launches. No live daemon mutation, Telegram send,
deployment, or operator-config application is required.

## Verified baseline and remaining failures

The audit used SASE `db48ae56dfb5b5ae955182e84bf36ef057d37db1` and core
`fe1a17bc486ac3474c3b1ae5e10427525cb39f1c`. A fresh fetch found SASE `origin/master`
equal to HEAD. All five `sase-11e.8` child beads are closed with one completion note
each and no `PROPOSED FOLLOW-UP:` entries; the current epic initially had no notes. The
source and commits show substantial implementation, but these counterexamples still
violate explicitly required acceptance:

1. **Generic edit source selection is incomplete.** In core `config/plan.rs`,
   `plan_edit` looks up the full normalized path in `target_normalized.sources`, then
   falls back to a canonical path. With a target containing only
   `axe.lumberjacks.checks.interval: 5`, setting `axe.routines.checks.job_timeout: 30`
   through the public Python `plan_config_edit` and `apply_config_edit` APIs creates a
   parallel `axe.routines.checks` subtree. The actual temporary-file output was:

   ```yaml
   # preserve me
   axe:
     lumberjacks:
       checks:
         interval: 5
     routines:
       checks:
         job_timeout: 30
   ```

   For an existing list-form job
   `axe.lumberjacks.checks.chops: [{name: hook, script: old-script}]`, a generic edit
   using exact segments `axe/routines/checks/jobs/hook/script` raises
   `validation: cannot set ...chops.[0].script: chops is not a mapping` instead of
   updating that contribution. The raw Rust planner reproduces this in both flag states.
   Existing tests cover replacement of an already-present map leaf, which does pass;
   they miss these required additions and list cases.

2. **Tribe collision detection is advisory and disconnected from identity mutation.**
   Core `agent_tribe.rs::resolve_agent_tribe_display_config` reports a same-layer
   `ace.tribes.chop`/`ace.tribes.job` conflict, but its only production Python consumer
   is TUI display logging. With different icons/descriptions for those two entries,
   general `build_config_inventory` returns no collision diagnostics and
   `ace.agent_tribes.set_tribe(..., "job")` still selects stored `chop` silently.
   `canonicalize_public_tribe_name` and `parse_tribe_reference` accept no configuration
   or collision context. A temporary assignment store containing an existing
   `tribe: job` record is rewritten to `chop` by a same-name
   `update_agent_tribe_assignment(..., "job")`. Metadata reads were repaired, but
   mutation and targeting remain unsafe for an independent historical job tribe. The
   resolver checks only simultaneous aliases inside one layer; display chooses keys
   independently and does not use the resolver's `display_keys` result.

3. **Live diagnostic templates remain incomplete.** `axe_config_compose` with canonical
   `axe.routines.chop-watch.jobs: [chop-test, chop-test]` and required descriptions
   still emits `duplicate chop identity`, `chop ... requires ...`, and
   `lumberjack ... requires ...`. Some validation paths are normalized internal
   `axe.lumberjacks...` even though the authored source used `routines`. The original
   child plan explicitly required fixing these templates. Core `config/axe.rs` and
   `axe_chop/config.rs`, plus Python `axe/status_collector.py`, still contain live
   old-word messages. Phase `sase-11e.8.4` correctly removed unsafe whole-string
   replacements and added explicit public status projection; retain those fixes.

4. **CI dependency integration is incomplete.** The intervening commit `7fe9f7d25` moved
   `sase-core-revision.txt` to `a7d588263e5a1c69f49dddbb2f72a138382b53a5`, which
   precedes all three repair core commits: `d4b301f` (general config), `ad13940` (tribe
   bindings), and `fe1a17b` (status projection). Both
   `.github/workflows/master-gate.yml` and `ci.yml` build that exact pinned revision.
   The pin lacks the new tribe and status entrypoints the Python code requires. A local
   build of current linked HEAD does not prove that CI or a published wheel provides
   them. Use the existing revision-ratchet and dependency-floor workflows.

`just install` completed. After rebuilding the local binding, 135 existing tests passed
across `test_config`, `test_config_inventory`, `test_config_edit_apply`, `test_axe_cli`,
`test_axe_chop_doctor`, `test_axe_status_cli`, `test_agent_tribes`, and
`ace/tui/models/test_tribe_display`. The failures above were independently reproduced
through production APIs and isolated temporary files. Those passing tests do not cover
these counterexamples. No full landing gate was claimed by this audit.

## Phase source_edits

Repair source-path selection in `sase-core/crates/sase_core/src/config/plan.rs` using
the existing AXE normalization/provenance machinery in `config/axe.rs`. Reuse the
established AXE editor's source-preserving semantics rather than adding a Python mapping
or another divergent Rust merge algorithm.

For a missing leaf under an existing authored entity, use the real existing ancestor and
appropriate structural field spelling. Canonical paths remain correct for genuinely new
public entries. Do not create a second alias subtree for an existing routine/job. Handle
list-form sources deliberately: either edit a real indexed contribution through an
appropriate plan shape, or use the established lossless list-to-map promotion, with an
apply plan the YAML adapter can execute. Preserve comments, siblings, order where
contractual, exact dotted names, opaque values, and effective semantics.

Test existing and missing leaves, both authored spellings and rollout states, map/list
forms, inherited-only values, set/unset/reset/delete, and equal/conflicting aliases.
Include exact-segment dotted routine/job names. Assert returned write paths, preview
values, exact temporary-file results, and recomposed runtime behavior. Cover both
generic and AXE entry editors. Do not introduce test-only passing paths that bypass
`config/edit.py` or disable the actual public projection.

## Phase tribe_safety

Complete the provenance-aware Rust identity resolver and expose sufficient context
through the binding so config validation, assignment, queries, wait/fork references,
completion, and TUI display use one result. Start with core `agent_tribe.rs`, general
config inventory/validation, Python `core/agent_tribe.py`, `ace/agent_tribes.py`,
`ace/agent_query/evaluator.py`, and `ace/tui/models/tribe_display.py`.

Distinguish customization of the built-in automation identity from conflicting or
independent historical `job` identity. Ordinary later-layer customization of a built-in
default must remain valid; same-layer conflict, distinct authored identities across
layers, and historical stored job assignments must not be silently merged. Specify and
test that decision using source provenance and stored-identity evidence; do not guess
independence solely from an icon difference. If ambiguous, return an actionable
source-aware diagnostic before assignment/targeting can select another identity.

General config validation must surface collision diagnostics, and TUI behavior must
present them usefully rather than merely logging and proceeding with an ambiguous
mapping. Reuse cached/worker-based configuration resolution: avoid new I/O on the UI
event loop and avoid flag/config initialization recursion. Metadata/history reads keep
their exact stored identity. A same-name mutation must not migrate a historical custom
tribe. Preserve non-conflicting built-in `job -> chop` writes and canonical display,
plus color, glyph, grouping, and collapse settings.

Add Rust and PyO3 coverage and production-path Python tests for diagnostic propagation,
config layers, custom stored job assignments, same-name updates, legacy/canonical
filtering, targeting, metadata reads, and built-in customization. Verify stores remain
unchanged when an operation is rejected.

## Phase diagnostics

Update owned, live templates in core `config/axe.rs`, `axe_chop/config.rs`, status
assessment/projection as appropriate, and Python `axe/status_collector.py` plus any
remaining live SDK/context/editor/CLI emitters. Audit reachable messages, not private
symbol names. Human vocabulary is canonical in both flag states and through hidden
legacy invocation aliases. Preserve compatibility JSON envelopes and internal codes
where contractual.

Use source provenance for authored config paths; do not label a canonical authored path
with its internal alias. Preserve names such as `chop-watch`, `chop-test`, exact
old-only executable paths, arbitrary dictionary keys, descriptions, stored diagnostics,
and historical output. Do not restore substring replacements over rendered text.

Cover duplicate identities, generated-instance edit errors, missing descriptions,
invalid routine/job shape, collector failures, and actual source paths under both input
spellings and flag states. Check the human renderer as well as JSON because the new Rust
status projection only runs on selected public JSON paths. Keep current status/doctor
payload-preservation regressions passing.

## Phase acceptance

Once the preceding Rust changes are available in the linked repository, review the
supported `just ratchet-core-revision` workflow and advance the SASE pin to the actual
verified revision containing them. Build and check bindings against that pinned
revision, not an unrelated local HEAD. Run `tools/check_sase_core_rs_bindings` and
appropriate core-floor/release checks. Do not manually edit release-managed Rust
versions or treat an unreleased local wheel as proof of a published dependency floor.

Complete the previously requested reproducible integration fixture, extending existing
tests where appropriate. Exercise equivalent legacy/canonical config, list/doctor,
generic and AXE edits, a harmless fixture-script run, history/public JSON, and job/agent
link navigation. Include both rollout states, old-only plugin entrypoints, collision
cases above, target expansion, exact dotted names, duplicate job names across routines,
env alias propagation/scrubbing, stable deduplication, and timeouts. Prove preserved
bytes and identities as well as canonical envelopes. Clear inherited
`SASE_FEATURE_FLAGS` for isolated flag probes and assert the resolved state; an empty
string is invalid transport, so remove the variable rather than setting it empty.

Recheck post-audit drift in SASE, core, linked integrations, and any PR base branch.
This audit reviewed intervening disk-reap/object-sharing module splits, xprompt
highlighting/LSP work, and the CI pin change. Keep preview-only retention, borrower
protection, and queue/hold admission behavior intact. Telegram, chezmoi, GitHub, and
research-artifacts had no post-start commits; the only newer Neovim change was
`3115d9d`, an xprompt semantic-token smoke test. Edit plugins only for actual new
routine/job integration matches.

Run full core `just check`, including Python >=3.12 PyO3 tests, required SASE
`just check` after changes, relevant integration checks, and affected visual tests
(inspect actual/expected/diff before updating goldens). Recheck docs/schema/defaults,
completions, and generated outputs affected by these fixes. Use memory/skill authoring
procedures only if their sources actually need changing; do not redeploy generated
skills from an unlanded tree.

Before combined landing, run `just check-full` through `/sase_monitor` with the
TESTING/TESTED profile and a mechanical continuation. Verify the entire command exits
successfully, including cost-budget checks, and record reproducible evidence. Keep
unrelated failures explicit and route genuinely independent discovered work through the
proper task procedure; incomplete contracts above remain this epic's work.

## Existing follow-up dispositions

There were no proposed follow-ups in the five `sase-11e.8` child notes. The parent
landing audit already disposed of its original proposals: wire alignment resolved; tribe
collisions remain epic work; missing chezmoi `busted` is task `sase-11m`; artifact
attachment trouble corroborated `sase-10y`. Both tasks were still READY at this audit.
Do not create duplicates or claim chezmoi's full check passed. Carry these dispositions
into the eventual ancestor close note after rechecking the original descendant notes.
`sase bead epic-symbols sase-11e.8` currently has no entries; recheck at actual closure.

For the acceptance note, map each numbered failure above to its regression test and
verified production behavior. Identify the exact SASE/core revisions used by the full
gate so the next landing audit can distinguish passing evidence from subsequent drift.
