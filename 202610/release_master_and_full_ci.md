---
tier: epic
title: Repair sase release gates and prove a fresh green tip
goal: Master Gate is green on the release tip and Full CI is green on the same SHA
  within six hours.
parent_bead: sase-1i5.9.1.2
phases:
- id: cli-beads
  title: Repair CLI contracts, completion drift, terminology, and bead test doubles
  size: medium
  depends_on: []
  description: 'cli-beads: repair live CLI and bead failures against landed contracts,
    preserve write guards and read-only resolution, and regenerate the completion
    snapshot after checking the active owner.'
- id: host-contracts
  title: Repair host provenance fixtures, foreign-commit recovery, and detached-run
    isolation
  size: medium
  depends_on: []
  description: 'host-contracts: make gate and launch provenance assertions exact,
    resolve foreign-commit recovery without relaxing publication guards, and investigate
    the detached-run database lock with isolated evidence.'
- id: plan-tui
  title: Restore associated-plan cache guarantees and current Verdict copy
  size: small
  depends_on: []
  description: 'plan-tui: apply the active Plan Decisions epic''s recorded signature-cache
    and short-label fixes where still missing, preserve tooltip and invalidation coverage,
    and note the owner.'
- id: tui-functional
  title: Repair timezone-dependent and asynchronous TUI failures
  size: medium
  depends_on: []
  description: 'tui-functional: make wait-lane clocks timezone-stable and fix reproducible
    prompt, context-bar, archived-plan, and deck readiness failures from Full CI without
    longer waits or weaker assertions.'
- id: symvision
  title: Resolve the live unused-public backlog and any newly exposed lint failures
  size: medium
  depends_on:
  - cli-beads
  - host-contracts
  - plan-tui
  - tui-functional
  description: 'symvision: re-inventory lint after functional repairs, privatize in-file-only
    definitions, remove dead definitions, preserve real external consumers, and resolve
    every release-blocking lint error without new pragmas or suppressions.'
- id: visual
  title: Repair visual state failures and inspect complete screenshot verification
  size: medium
  depends_on:
  - symvision
  description: 'visual: fix the four Reply-card navigation failures and snippet finder
    failure, reconcile active Plan Decisions golden work, and verify the full visual
    lane with every changed image inspected.'
- id: ci-proof
  title: Prove Master Gate and a fresh Full CI on the release tip
  size: medium
  depends_on:
  - cli-beads
  - host-contracts
  - plan-tui
  - tui-functional
  - symvision
  - visual
  description: 'ci-proof: observe the integrated master SHA, repair remaining deterministic
    CI failures, dispatch and monitor Full CI, and record successful same-SHA run
    URLs and freshness for the assigned phase''s resumed owner.'
proposed_by: bbugyi200.athena.sase-1i5.9.1.2
create_time: 2026-10-08 14:54:10
status: wip
bead_id: sase-1i5.9.1.2.1
---

- **PROMPT:** [prompts/202610/release_master_and_full_ci.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/release_master_and_full_ci.md)
- **PARENT:** [202610/release_sase_core_and_plugin_floors.md](https://github.com/sase-org/sase--plans/blob/main/202610/release_sase_core_and_plugin_floors.md)
- **BEAD:** [sase-1i5.9.1.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1i5/sase-1i5.9.1.2.1.md)

# Repair the release gates for sase-1i5.9.1.2

This is the child implementation epic for the assigned `sase-gates` phase. Its
authoritative parent design is `plan:202610/release_sase_core_and_plugin_floors.md`.
Read that artifact and this worker's own bead with audited commands before acting. This
plan implements only the parent's final `release_red_master = fix_master` choice. The
parent also selected `plugin_floors = yes`; publishing and plugin floors remain the
other release phases' work. There are no new reviewer decisions and no memory edits.

This is an epic because the live failures cross independent CLI, host, TUI, static lint,
and visual contracts, and the final proof requires landed changes plus external CI
waits. The first four phases can run independently. Symbol cleanup follows their landed
changes to avoid changing the same definitions concurrently. The visual phase uses the
integrated functional tree. CI proof follows every repair.

## Rules that apply to every phase

- Work from the checkout SASE assigns. Use repository-relative paths in all evidence.
  Open any other repo through `/sase_repo`, then read its `AGENTS.md`; read sidecar
  artifacts only with `sase artifact read`.
- Read `lint_and_test.md`, `sase_beads.md`, and, for lint fixes, `symvision.md` through
  `/sase_memory_read`. TUI workers also read `tui.md` and its relevant performance or
  screenshot child note. Read `cli_rules.md` before adding or changing an option.
- Re-inventory current master before editing; the observations below are evidence, not a
  permanent whitelist. Every deterministic Master Gate or Full CI red is in scope under
  `fix_master`. Do not merely label a release-blocking CI failure KNOWN and leave it
  red.
- For an active epic's failure, first read its bead/design and check whether its
  recorded fix has landed. Apply only the missing minimal correction and note that
  owner. Do not duplicate its unfinished architecture. The specific overlap map below is
  mandatory context, not a dependency on completing that whole epic.
- Preserve assertions and the behavior they protect. Do not skip tests, add xfails,
  raise timeouts or performance caps, add Symvision pragmas, disable lint, weaken
  publication/write guards, or change CI schedules, shards, and release policy to get
  green. Re-key a moved exact terminology allowance only with its existing rationale;
  never add a blanket allowance.
- Shared domain behavior remains Rust-owned in `sase-core`; Python may fix test
  fixtures, Textual presentation, and thin adapters. A necessary Rust correction
  requires `/sase_repo`, that repo's checks, bindings, and a sase pin update beyond the
  landed core commit. Notify the release parent because the parallel `core-release`
  phase must then prove the new pin is published. Do not silently move a pin while the
  release proof still refers to the old one.
- Land changes through `/sase_final` and host completion, with `bead_action: keep` for
  intermediate work. Do not create commits, branches, or PRs manually. Run `just fix`,
  focused reproductions, and `sase tool run check` for each changed checkout. Do not run
  `just check-full`; GitHub Full CI is the required exhaustive signal. If a named check
  escalates with exit 124, join its exact run with `sase monitor start -J RUN`; never
  restart it.
- A local check failure proven identical on the clean base is recorded as
  `PROPOSED FOLLOW-UP:` on the worker's own phase bead, citing existing task IDs. It
  does not prevent a repair phase from closing. This exception does not satisfy the
  final Master Gate / Full CI acceptance.
- Phase workers create no beads and close only their own assigned child phase. Before
  every phase close, run `sase bead epic-symbols <own-phase-id>` and resolve entries or
  re-key only onto an actually open bead with an intended future consumer. Use
  `sase bead close <own-phase-id> --note '<verified evidence>'`. Do not close existing
  tasks, other epics, or ancestor beads.
- Use `/sase_monitor` for CI waits, sleeps, and long screenshot commands. Include a
  precise `--next` continuation, use `TESTING` / `TESTED` for verification, and wait for
  the start command itself to exit before ending the turn. A prepared completion must
  not skip inspection of mutating screenshot results.

## Evidence rechecked on 2026-10-08

The planning checkout and observed master tip were
`c70ee9af3d4812a23777c11ef77b2ac75dea4fa9`. Its Master Gate run
[37825898841](https://github.com/sase-org/sase/actions/runs/37825898841) was still
running; lint had already failed at the latest observation. Re-read its final result
before fixing. The latest completed Master Gate was
[37823201800](https://github.com/sase-org/sase/actions/runs/37823201800) on
`af117b598e141370da52565a0af89bf6223af843`: 19 failing test nodes plus Symvision.

Latest Full CI was
[37785681210](https://github.com/sase-org/sase/actions/runs/37785681210) on
`56fcf92447cb351d3adf42e2f286a185575b9ac2`. It failed lint, visual-test, and Python
3.12, 3.13, and 3.14 tests. There was no green Full CI in the six-hour window. Its older
failure set must be checked against current master, because several fixes have landed
since it ran.

The visual run's artifact `ace-visual-artifacts` (GitHub artifact ID 11555963707)
contains `runs/b67f618a34714317be1080c19a3ec3fc/capture.log` and `manifest.json`. The
manifest is `complete: false`, `mode: check`, `child_exit_code: 1`; comparison never
completed. It reports zero image changes because five tests failed to reach their
intended state. This is not evidence that the goldens match. The retained capture log
identifies all five failures listed under `visual` below; the main GitHub job log stops
inside the first enormous SVG assertion.

Existing owners consulted:

- `sase-1h8`, still active: bead read-model symbols and the missing `list_issue_page`
  test-double method (note 2). Note 3's core doctor expectation belongs to the parallel
  release-core phase, not this plan.
- `sase-1hi.10.7`, still active, with design
  `plan:202610/plan_decisions_landing_finish.md`: `.1` owns gate provenance cleanup and
  `write_acceptance_meta`; `.2` owns completion snapshot regeneration; `.3` owns the two
  associated-plan cache failures and the two old Verdict-label assertions; `.4` owns
  Plan Decisions and plan-gate goldens. Read the design's Sections 1, 2, 3 item 5, and 4
  before the overlapping fixes.
- `sase-1hp` already tracks unused-public lint; `sase-1hr` identifies the exact moved
  legacy-string allowance; `sase-1hs` tracks foreign-commit recovery. Preserve their IDs
  in evidence, without closing or claiming them.
- `sase-1i5.8` notes prove cache and other test failures predate its import-budget work.
  Keep the measured TUI import cap intact.

`sase bead epic-symbols sase-1i5.9.1.2` returned no entries during planning; repeat it
at actual completion.

## Phase cli-beads

Own the CLI/bead tests and their minimal adapter changes. Do not own generic gate
response handling, plan UI, finalizer dispatch, or Symvision renames.

1. Read `sase-1h8` again. Reproduce
   `tests/test_bead/test_claimed_status.py::test_default_list_includes_claimed_with_shared_glyph`.
   The fake `_ReadView` lacks the landed indexed `list_issue_page` API. Implement that
   API faithfully in the fake, using the correct page/filter/count shape; retain the
   claimed-status row and shared-glyph assertions. Do not add a Python replay fallback
   to production. Note `sase-1h8` with the correction and verification.
2. Repair the three failures in `tests/main/test_bead_fast_path.py`:
   `test_fast_path_guards_mutations_but_not_reads`,
   `test_fast_path_refuses_unsafe_resolved_location_before_rust`, and
   `test_fast_path_refuses_mutation_from_plain_checkout_sidecar_record`. The code now
   deliberately defers `+1` to Python; the first test still expects the fast-path guard
   for it. Separate deferred-verb coverage from the actual Rust mutation case, retaining
   a pre-binding write-guard assertion on `rm`. The latter two tests patch the old
   `cli_common.resolve_beads_location` seam, but `_resolve_fast_path_context` uses
   `operation_context` routing. Exercise the current seam or the real route with
   isolated project snapshots. Keep explicit assertions that unsafe/read-only mutations
   never reach the executor, reads remain available, and no unwanted store is
   initialized. If reproduction shows a production guard defect, fix the thin routing
   layer, not the assertions.
3. Repair
   `tests/main/test_completion_handler.py::test_candidates_handler_prints_provider_output`:
   its `fake_candidates_for` omits the new keyword-only `selector`. Match the real
   signature and assert which selector the handler forwards, including the default.
   Preserve printed candidate content.
4. Diagnose the help assertions in
   `tests/main/test_parser_command_help.py::test_memory_help_marks_primary_command_and_init_alias`
   and `tests/instructions/test_verify_cli.py::test_verify_help_documents_flags`. Their
   expected `-a, --agent` does not match the rendered metavar form
   `-a NAME, --agent NAME`. Use the existing metavar-aware helper and verify both
   spellings and the documented argument, rather than dropping flag checks.
5. Check `sase-1hi.10.7.2` and current completion drift. If still unfixed, use
   `just sync-completion-spec` to regenerate `tests/completion/snapshots/cli_spec.json`
   from the real parser; inspect the added `-S/--selector` and other legitimate drift,
   and note the active owner. Do not manually edit the generated tree or duplicate its
   shell-scoping work.
6. Apply the exact `sase-1hr` correction: re-key the moved `%xprompts_enabled` fixture
   strings to `tests/history/test_continuation_replay_hydration_basic.py`, preserving
   their compatibility rationale and removing stale old-path rows only when verified. If
   Full CI's `test_sase_turn_terminology` still fails on
   `src/sase/bead/cli_work_from_plan.py`, change current prose to the accepted turn
   vocabulary after consulting its active owner; preserve legacy readers.
7. Run the whole affected test files, completion drift tests, terminology guards, and
   `sase tool run check`. Record node results, exact remaining local failures, and the
   active-owner notes before closing this child phase.

## Phase host-contracts

Own generic gate/launch test contracts, finalizer recovery, and detached-run isolation.
Read the active Plan Decisions design before changing its response code.

1. Reproduce
   `tests/test_plan_gates_execution.py::test_shared_host_executor_handles_feedback_rejection_and_races`.
   The actual option result includes `_gate_source`, `_gate_caller`, `decided_by`, and
   `decided_via`. The approved active owner explicitly requires private gate keys to be
   stripped always, including plans without decisions. Reuse its landed correction, or
   apply it minimally if absent and note `sase-1hi.10.7`. Public provenance stays in the
   expected payload. Keep exact payload equality with public provenance; do not
   normalize away the leak.
2. Recheck the older Full CI failures in `tests/test_gate_cli_answer_detach.py`,
   `tests/test_plan_approval_actions_archive.py`, and
   `tests/test_plan_gates_action_api.py`. Align the exact public payloads with their
   real route's CLI/TUI provenance and preserve rejection, retry, and option-filtering
   behavior. Shared production fixes stay with the active owner's accepted design and
   are noted there.
3. Recheck Full CI's atomic agent-meta and macro-group launcher tests:
   `tests/axe/test_agent_meta_atomic.py` and
   `tests/test_multi_prompt_launcher_macro_groups.py`. Their old expectations omit
   `prompt_origin` and `prompt_source_surface`. Use current launch provenance contracts;
   retain atomic-publication, shared-counter, and first-slot-only force-reuse
   assertions. Make environment setup explicit so test results do not depend on the
   worker's own launch metadata.
4. Read `sase-1hs` and reproduce
   `tests/test_finalizers_discard_guard_before_head.py::test_post_dispatch_foreign_race_on_external_is_exempt`.
   The fixture creates a foreign commit, its resume fake returns success, and the
   foreign commit remains ahead of a real bare upstream. The hardened
   `commit_repair_conflict._verify_settled_or_raise` correctly requires publication
   evidence before accepting a clean tree without a result marker. Verify the intended
   foreign-race contract against neighboring publication and discarded-work tests.
   Prefer making the successful fixture publish the foreign commit through its bare
   remote so the exemption is proven for published work, and cover the unpushed
   counterpart's refusal. If a production route is wrong, fix the precise
   ordering/classification while preserving rejection of every genuinely unpushed or
   discarded payload. Do not turn a successful fake exit into sufficient proof. Record
   the result citing `sase-1hs`; do not close that task.
5. Investigate the latest Master Gate's
   `tests/tool/test_detach.py::test_watchdog_reports_ended_join_monitor` failure:
   `RuntimeError: tool run store is busy: database is locked`. Its fixture sets an
   isolated `SASE_HOME` and launches a live child/watchdog. Inspect the actual traceback
   and resolved database before classifying it. Confirm clean-base behavior and child
   environment isolation, and ensure all launched processes settle before the fixture
   removes its store. Fix demonstrated deterministic fixture lifetime/environment
   defects; a real concurrent store behavior fix belongs in Rust with binding coverage.
   Never patch an installed wheel, retry the assertion until it passes, or raise the
   busy timeout. If it is a true intermittent failure, record reproducible evidence and
   the existing owner; final release acceptance still requires an actually green CI run.
6. Run affected files, neighboring finalizer unpublished/discarded-work tests, and
   `sase tool run check`. Exact response provenance, atomic publication, published-race
   success, and unpushed/revert refusal must survive.

## Phase plan-tui

This is a bounded implementation of the active owner's already-recorded repairs. Check
master and `sase-1hi.10.7.3` first; do not race a landed fix or redo its Verdict
layout/tint architecture. Note `sase-1hi.10.7` when applying a missing correction.

1. The cache root cause is
   `models/_agent_associated_plan_summary.py::_cache_associated_plan_sheet`: every
   enrichment calls `load_stamped_decisions`, re-reading the same plan despite the
   metadata cache. Follow the owner's signature-based cache design keyed by both plan
   file and `.plan-decisions.json` sibling, including absent siblings, invalidation, and
   cached misses. Preserve the render lane's lookup-only behavior and keep file reads
   off the message pump. Coordinate reuse of already cached plan content if needed so
   one unchanged file is read only once.
2. Make both existing tests in
   `tests/ace/tui/models/test_agent_associated_plan_cache.py` pass with their original
   read-count and mtime/signature guarantees. Verify sibling creation, change, and
   removal refresh the accepted sheet without stale provenance.
3. Repair the two old-label assertions in `tests/ace/tui/test_notification_plan_gate.py`
   and `tests/test_plan_approval_modal_title.py`: the approved UI is
   `☑️ 🚀 Launch coder`, with the full explanation in its tooltip. Assert the short
   visible label, icon, and meaningful tooltip, while retaining branch selection and
   asynchronous loading assertions. Do not widen the Verdict or revive the full label to
   satisfy a stale test.
4. Run the affected cache/modal files and `sase tool run check`. No PNG generation here;
   `visual` owns this plan's screenshot reconciliation.

## Phase tui-functional

Own wait-lane tests and non-plan TUI behavior. The older Full CI failures may have been
repaired already; prove their current status before editing. Read `tui_perf.md` and keep
the import budget and prompt latency contracts intact.

1. Fix the two `since 14:32` assertions in
   `tests/ace/tui/test_agent_wait_epic_follow_tui.py`. The fixed timestamp is displayed
   as `since 10:32` in UTC CI. Use a timezone-explicit fixture/clock and calculate the
   expected displayed time consistently with the presentation contract. Verify the two
   nodes under UTC and America/New_York; do not hardcode a replacement hour or discard
   the timestamp assertion.
2. Recheck the earlier Master Gate midword node
   `tests/ace/tui/widgets/test_prompt_next_word_midword.py::test_deferred_midword_defers_then_applies`
   and Full CI's Python 3.14
   `tests/ace/tui/widgets/test_prompt_next_word.py::test_auto_space_shows_ghost_without_separator`.
   Preserve ghost rendering, separator semantics, stale-result cancellation, and event
   ordering. Fix fixture isolation or the actual deterministic deferred application
   behavior, rather than increasing sleeps.
3. Reproduce Full CI-only failures on the current tree in
   `tests/ace/tui/test_prompt_key_perf_smoke.py` (missing `prompt_space`),
   `tests/ace/tui/test_launch_context_bar.py` (density readiness and stale
   provider/model state), `tests/ace/tui/test_link_follow_bounded_panes.py` (archived
   plan selection), and `tests/ace/tui/widgets/decks/test_deck_block_spread_pilot.py`
   (spread alignment and streaming cursor readiness). Trace the actual ready-state
   transition, background worker completion, and retained state. Repair concrete
   ordering, isolation, or presentation defects while keeping bounded selection and
   geometry assertions. Use semantic readiness instead of fixed delay guesses.
4. Verify affected files under Python 3.12 and the extra version that exposed a defect
   where available. CI will provide the full 3.12/3.13/3.14 proof. Record which old
   failures already pass on master and which were actually fixed. Run
   `sase tool run check` and document any still-owned failures precisely.

## Phase symvision

Run after the functional phases to avoid redundant renames and integration conflicts.
Read `symvision.md` and `sase-1hp`; read active owners before touching their symbols.
Start with current Master Gate lint and `just _lint-symvision`, not the historical list
alone. Re-run the exact lint stage after each coherent batch because fixing one rule can
reveal the next. Run any later lint gates Master Gate reveals, including toobig;
preserve their limits.

At planning, the lint list included these groups:

- Instruction internals: `CacheKeyInputs`, both `InstructionManifestError` definitions,
  `CoreMemoryUnit`, `ReferenceMemoryUnit`, `WebMemoryUnit`, `MemoryIntroTexts`,
  `ParityIssue`, `ParityReport`, `RunManifest`, `aggregate_rows`, `cache_entry_path`,
  `prune_cache_entries`, `default_provider`, `detect_host`, run-index root helpers,
  JSON/render helpers, the local render and verify command implementations, observation
  helpers, and doctor helpers.
- Bead-owner seams: `BeadBoardSnapshot`, `BeadStoreFingerprint`,
  `bead_push_log_retention_config`, `hidden_sidecar_clone_dirs`,
  `maybe_gc_hidden_sidecar_clone`, and `route_bead_targets`.
- Other internal seams: `advertised_config_type_names`, `validate_config_input_type`,
  `macro_input_choice_to_wire`, `controller_failure_for_handoff`, `fetch_worker_argv`,
  `finalizer_owned_monitor_refusal`, `finalizer_reports_failure`, `git_fetch_origin`,
  `git_is_ahead_of_upstream`, `git_remote_tracking_ref`,
  `instruction_shadow_render_enabled`, `staged_sdd_files`, and `write_acceptance_meta`.

Many are live only within their defining module, e.g. `CacheKeyInputs` and
`run_instructions_render`; privatize those and update their in-file references, test
imports, and `__all__` where appropriate. Never delete a whole live subsystem because
its internal types were incorrectly public. For genuinely dead definitions, first check
non-test consumers including linked repos opened through `/sase_repo`, then remove only
dead definitions/helpers/tests. Test use alone is not a public consumer. Keep externally
consumed APIs with actual evidence and an existing valid consumer declaration; do not
manufacture consumers or add pragmas.

The active bead/read-model and Plan Decisions owners may now have consumed or privatized
their seams. Note their minimal corrections when still necessary. A future-use
`--epic-symbol` exception is permissible only for a proven intended consumer on an
actually open owner bead, never as a release workaround. Prefer resolving every finding.
Audit all Justfile exceptions, verify no closed phase is referenced, run the whole lint
path and `sase tool run check`, and leave an empty unused-public report with no weakened
checks.

## Phase visual

Read `tui_screenshot.md` and the active Plan Decisions golden phase `sase-1hi.10.7.4`.
Reconcile its already-landed results rather than blindly regenerating the same PNGs
while it edits the Verdict. This phase owns this plan's remaining visual failures and
full-lane proof.

1. Reproduce the four deck visual failures from the retained CI capture log:
   - `test_ace_png_snapshots_agents_deck_blocks.py::test_agents_deck_blocks_paged_older_png_snapshot`
   - `test_ace_png_snapshots_agents_deck_views.py::test_agents_deck_view_auto_page_blocks_png_snapshot`
   - the same file's `::test_agents_deck_view_fixed_page_cards_png_snapshot`
   - the same file's `::test_agents_deck_view_split_narrow_png_snapshot` All reach
     Context/Reply card inventory, press `ctrl+j`, then never select Reply. Inspect
     current focus and keymap behavior and the captured wrong frame. Fix the real
     navigation or drive the correct documented action from a deterministic focus state.
     Preserve Reply selection, older-block paging, fixed-card behavior, and split focus
     assertions. Do not snapshot the wrong card or lengthen the 15-second wait. Update
     `default_config.yml` if a real keymap correction requires it.
2. Reproduce
   `test_ace_png_snapshots_snippet_name.py::test_snippet_location_flow_finder_png_snapshot`.
   It never reaches the `2 snippets` sentinel. Verify the complete location-to- finder
   path, fixture config/catalog isolation, and current displayed count. Keep the
   count/flow contract and avoid accepting a different screen.
3. Use targeted check-only visual captures during diagnosis. Once functional states are
   correct, run full `just fix-tui-screenshots` through `/sase_monitor` with a generous
   timeout and inspection in `--next`. Read its manifest's `complete`, `status`,
   `skipped`, and pruning reason; `partial` or zero changes is not proof of a green
   visual lane. Repair skipped deterministic nodes before accepting the run.
4. Inspect every creation/removal and each update group, expanding unexpected groups.
   Keep legitimate goldens, including every dirty golden produced by verification, in
   host completion. Preserve exact pixel equality, bundled fonts, and renderer versions.
   Confirm current Plan Decisions controls, tint, memory warnings, and generic-gate
   layouts against their owner's design; do not accept a regression as a new baseline.
5. Prove full `just fix-tui-screenshots --check` passes after the inspected updates, and
   run `sase tool run check` for source changes. Record the full visual manifest/run
   evidence and inspected groups before closing.

## Phase ci-proof

This phase must finish actual external verification; a local KNOWN verdict is not
release readiness. It owns any newly revealed deterministic reds on the integrated tip
and keeps the repair loop bounded to the existing accepted scope.

1. Confirm every repair phase's commit is on master through host completion. Re-read the
   current master SHA and Master Gate run list. Use
   `gh run list --repo sase-org/sase --workflow master-gate.yml` and inspect jobs/logs
   for that exact SHA. Do not confuse a green ancestor or another branch with the
   release tip.
2. Monitor an unfinished run, for example with `/sase_monitor` around
   `gh run watch RUN --repo sase-org/sase --exit-status`, a `verify` profile, sufficient
   timeout, and `--next` naming the run/SHA and the repair action on red. Wait for the
   monitor-start command itself to exit. If a run failed, inventory all jobs, fix each
   deterministic cause with focused coverage and `sase tool run check`, and land through
   host completion. Preserve existing CI gates. A genuinely infrastructure-only incident
   needs evidence before a rerun, and acceptance still requires green.
3. Once Master Gate is green on master, dispatch one fresh Full CI:
   `gh workflow run "Full CI" --repo sase-org/sase --ref master`. Record the dispatch's
   actual resulting run ID and head SHA. Observe an already-running same-SHA Full CI
   rather than piling up duplicate runs.
4. Monitor that exact run with a timeout appropriate for the observed nearly two-hour
   suite. Inspect every required job: lint, visual-test, all Python tests, performance
   floors, and other enabled jobs. If the heavy lane finds a new deterministic
   visual/extra-Python failure, repair and verify it, land, re-establish Master Gate
   green, and dispatch Full CI on the new SHA. An obsolete red run is retained as
   evidence, not treated as current success.
5. Immediately before recording completion, re-read master. Both successful run heads
   must equal the release tip, both conclusions must be `success`, and Full CI's
   freshness must be inside six hours under ci_watch's actual freshness calculation.
   Record creation/completion UTC times as well as the evaluated age. If master moved,
   evaluate its actual gates and obtain the same-SHA proof again; no bypass, policy
   change, or hand merge.
6. Note `sase-1i5.9.1.2` with the exact SHA, both run URLs/IDs, conclusions, timestamps,
   freshness, and any already-tracked local-only follow-ups. Close only this child phase
   after its own epic-symbol audit. Keep the release parent aware that `publish-sase`
   must recheck if the tip moves.

## Child-epic landing and assigned-phase completion

This child epic's land agent audits that every repair is landed, no assertions or limits
were weakened, no memory was edited, local verification evidence is recorded, and the
actual same-tip Master Gate / Full CI proof remains fresh. Recheck the live SHA and gate
evidence, rather than trusting an earlier phase's age calculation. Audit its own epic
symbols and the assigned phase's symbols, and close only its own child epic when its
descendants are complete. It must not close `sase-1i5.9.1.2`, `sase-1i5.9.1`,
`sase-1i5.9`, or `sase-1i5` as an ancestor.

The host's resumed owner of the originally assigned phase `sase-1i5.9.1.2` receives the
completion evidence and performs the final closure. That owner re-reads the phase and
this design, confirms the integrated child epic is done and the two live CI signals
still meet acceptance, then runs:

```bash
sase bead epic-symbols sase-1i5.9.1.2
sase bead close sase-1i5.9.1.2 --note '<release SHA, green Master Gate URL, green same-SHA Full CI URL, UTC timestamps and age, verified repairs and checks>'
```

Resolve or legitimately re-key any remaining symbols before that close. Close no
ancestor. Use `/sase_final` for the normal final response; the host owns commits.
