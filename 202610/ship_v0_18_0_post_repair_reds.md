---
tier: epic
title: Clear the post-repair Master Gate and Full CI reds and ship sase v0.18.0 through
  ci_watch
goal: 'sase 0.18.0 is on PyPI, published from release-please PR #299 after ci_watch
  merged it on a green Master Gate and a fresh green Full CI, with sase-core-rs 0.42.0
  published as its core floor. The release is confirmed before landing and recorded
  in an evidence artifact by the final epic lander, and parent epic sase-1mk is resumed
  and closed.

  '
parent_bead: sase-1mk
phases:
- id: research-readme
  title: Restore the generated research sidecar README
  depends_on: []
  size: small
  description: 'research-readme: regenerate the research sidecar README that commit
    dcbfd0dc hand-edited (sase-1mm), so init repo --check and both lint lanes pass.'
- id: jump-golden
  title: Resolve the agents JUMP panel golden drift from the clan Enter change
  depends_on: []
  size: medium
  description: 'jump-golden: decide whether the db14df00e6 JUMP title change is intended,
    then fix the product or update the golden so Full CI visual-test passes.'
- id: link-follow-kind
  title: Stop inventory-miss completion from overriding kind-corrected link follows
  depends_on: []
  size: medium
  description: 'link-follow-kind: fix the sase-1mk.7 regression that completes a kind-mismatched
    follow as MISSING during a pending query, keeping sase-1lr fixed.'
- id: new-project-owner
  title: Fix the New Project create-form readiness regression on owner switch
  depends_on: []
  size: medium
  description: 'new-project-owner: fix the sase-1mk.6 RepoCreationFormWidget readiness
    regression that strands the create form after an owner switch in every Full CI
    leg.'
- id: runtime-flakes
  title: Fix the known-continuation and provider-drain flakes
  depends_on: []
  size: medium
  description: 'runtime-flakes: root-cause test_agent_run_stops_on_generic_output
    and the provider-drain JSONDecodeError (sase-1k2) outside the TUI.'
- id: watch
  title: Watch sase 0.18.0 to PyPI and always close
  depends_on:
  - research-readme
  - jump-golden
  - link-follow-kind
  - new-project-owner
  - runtime-flakes
  size: small
  description: 'watch: run one bounded tools/release_watch monitor that closes this
    phase bead on every outcome, so the lander always runs.'
proposed_by: bbugyi200.athena.sase-1mk.land
proposed_with: claude/opus
create_time: 2026-10-18 16:47:56
status: wip
bead_id: sase-1mk.9
---

- **PROMPT:** [prompts/202610/ship_v0_18_0_post_repair_reds.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/ship_v0_18_0_post_repair_reds.md)
- **PARENT:** [202610/ship_v0_18_0_owned_blockers.md](https://github.com/sase-org/sase--plans/blob/main/202610/ship_v0_18_0_owned_blockers.md)
- **BEAD:** [sase-1mk.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1mk/sase-1mk.9.md)

# Plan: Clear the post-repair Master Gate and Full CI reds and ship sase v0.18.0 through ci_watch

## Why this child epic exists

Parent epic **sase-1mk** owned every v0.18.0 release blocker it could see at 13:45Z. All
seven of its repair phases landed on sase master (`7dd39ff95f`, `f1c20a9f6b`,
`3b78c03441`, `f27d630c82`, `551a0f7555`, `dabd231e82`). The sase-core fixture
`80dc704a` landed too. The parent lander verified each change is on master and none was
reverted. The core side is healed:

- sase-core-rs **0.42.0** was published at 2026-10-18 17:18:53Z with five distributions,
  and its `v0.42.0` tag (`07bfb83f`) contains `80dc704a`.
- Publish run 39149037943 ratcheted PR #299's core window. PR #299 head `280ebad298`
  passes `release-core-floor-smoke`. It is `MERGEABLE`/`CLEAN`, with every check SUCCESS
  or SKIPPED.

The parent's watch phase sase-1mk.8 ended **MASTER_GATE_RED** (watcher exit 17) at
19:29Z. Master Gate at `dabd231e82` failed lint. Since then, Master Gate at the new tip
`457dd728d5` (run 39159243248) is red too. Full CI has not been green since the parent
started.

**This epic owns every remaining release blocker outright.** That includes reds caused
by concurrent commits and reds caused by the parent's own phases. No phase or lander may
label a red "externally owned" and wait.

### Blockers as of 2026-10-18 ~20:25Z (re-derive before acting)

1. **Lint, both lanes: research sidecar README drift (task sase-1mm).** The
   `SASE validation` stage fails at `init repo --check` with
   `~ update sase/repos/research/README.md −5 research sidecar README.md`. CI's
   `tools/ci_bootstrap_sidecars` clones `sase-org/sase--research`, and research commit
   `dcbfd0dc` (2026-10-18 15:18Z, "docs(research): tag recent PyPI release reliability
   research") hand-added a `## Topic Tags` section to that exact-match generated file.
   The file is rendered from `src/sase/sdd/templates/sidecar-research-README.md` by
   `expected_sdd_sidecar_files` in `src/sase/sdd/_init_files.py`.
   - Every Master Gate since `3b78c03441` (15:53Z) has a red lint job, in runs
     39142532111, 39142737163, 39142812537, 39143295454, 39143297524 and 39159243248.
   - The Full CI lint job is red in runs 39141460003, 39142620455 and 39155173153.
   - It also fails every agent's local `sase validate` / `just check`. The parent lander
     reproduced it locally.
2. **Full CI visual-test: JUMP panel golden drift (concurrent commit).** Full CI
   39155173153 `visual-test` reports
   `agents_jump_panel_collapsed_in_flight_mix_120x40: updated | changed pixels 91123/1520532`.
   CI renders `JUMP · CLAN MEMBERS 0-5 · 4 WAVES +…` where the golden shows
   `JUMP · CLAN MEMBERS 0-5`.
   - The drift reproduces locally at `457dd728d5` with
     `just fix-tui-screenshots --check -- tests/ace/tui/visual/test_ace_png_snapshots_agents_jump_panel.py::test_jump_panel_collapsed_in_flight_mix_png_snapshot`.
   - Reverting only the `src/` changes of concurrent commit `db14df00e6` ("feat(tui):
     act on a clan's lone stopped member from Enter", agent 11c, not this epic chain)
     makes it match again.
   - `visual-test` was green in Full CI 39141460003 and 39142620455, which ran before
     that commit.
3. **Master Gate and every Full CI test leg: New Project owner switch (regression from
   sase-1mk.6).** In Master Gate 39159243248 `test (8)`, and in all three test legs of
   Full CI 39155173153 (jobs 117770020902, 117770020903 and 117770020936),
   `tests/ace/tui/modals/test_new_project_create_flow.py::test_owner_switch_preserves_per_owner_drafts`
   timed out at line 175, waiting 5 s for `modal._create_form_status == "ready"` after
   switching the owner back to `sase-org`.
   - This node had not failed in any of the previous 105 failed Master Gate runs.
   - Every leg logs `pump-free task new-project-create-form:gh:bbugyi200 failed` with
     `textual.css.query.NoMatches: No nodes match '#repo-creation-form-errors' on RepoCreationFormWidget()`.
     The traceback runs from `_observe_create_form` (`new_project_create_checks.py:212`)
     through `_schedule_create_check`, `_paint_create_configure`, `set_context`,
     `_sync_form_slot` and `RepoCreationFormWidget.set_values`, into
     `_recompute_and_notify` and `_refresh_evaluation_view`
     (`repo_creation_form.py:509`).
   - Cause: `dabd231e82` (sase-1mk.6) replaced the widget's `is_mounted` guards with a
     `_controls_ready` flag. `_mark_controls_ready` sets that flag once and nothing ever
     resets it. When the owner switch detaches and remounts the form widget, the flag
     stays `True` before the new controls are composed. The refresh then queries
     controls that do not exist yet. The exception kills the form task, and the create
     form never reaches `ready` for the switched-back owner.
   - That commit also added a project-name pending-edit branch to
     `NewProjectCreateConfigure._rename_target`; audit it in the same phase.
4. **Master Gate: link-follow kind correction (regression from sase-1mk.7).** In Master
   Gate 39142737163 `test (4)`,
   `tests/ace/tui/test_link_follow_planner.py::test_epic_context_selects_and_expands`
   failed at line 101: `assert 'id:alpha-open' == 'id:alpha-1 id:alpha-1.*'`.
   - Its first failure was on `f27d630c82` (sase-1mk.7).
   - That commit added `_complete_pending_inventory_miss` in
     `src/sase/ace/tui/widgets/artifacts/beads_options.py`, plus an analog in
     `plans_options.py`. It is called while a query worker is pending.
     `_unfiltered_snapshot_has_target` matches the exact chip kind. The test follows a
     `task` chip for `alpha-1`, which is really an epic, so the follow can complete
     MISSING before the follow planner corrects the kind and widens the host query.
   - The same commit dropped `not self._loading` from `_loaded_current_snapshot`, which
     has other callers.
5. **Master Gate: known-continuation triage.** In Master Gate 39142737163 `test (5)`,
   `tests/tool/test_known_continuation.py::test_agent_run_stops_on_generic_output`
   failed with `assert 'helper_error' == 'unknown_item'`. It was seen once. Lander
   sase-1jm.2.1 earlier declined a report of the same node because it passed on current
   master.
6. **Full CI: provider drain (task sase-1k2).** In Full CI 39141460003 `test (3.12)`,
   `tests/fakey/test_provider_drain_e2e.py::test_provider_drain_e2e_relaunches_stranded_agent`
   raised `json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)` at
   line 333. It also failed in Master Gate 38935192508 and 38935262082 on 2026-10-16.
7. **Core release evidence: settled, no phase.** On the released commit `07bfb83f`,
   attempt 1 of sase-core CI push run 39146142422 failed
   `cargo fmt + clippy + test (macos-latest)` on a concurrency test,
   `crates/sase_core/tests/bead_read_model_parity.rs:934`
   (`concurrent_readers_see_consistent_snapshots`), not the fixture. Core master later
   hardened that path in the non-epic commit `0834d844`. The parent lander re-ran only
   the failed job: attempt 2's macOS job 117779108636 passed at 20:37:33Z. Release PR
   head `fd3b362f` CI 39144909733 was green too. This epic's lander cites attempt 2 as
   the Landing step 2 macOS proof.

Full CI 39155173153 (on `dabd231e82`, completed about 20:43Z) failed only these jobs:
`lint` (blocker 1), `visual-test` (blocker 2), and `test (3.12)`, `test (3.13)` and
`test (3.14)`, each failing only blocker 3's node (about 56335 passed per leg). Its
`perf-floors`, `tutorial-tests`, `ace-page-group-isolation` and `build-core` jobs
passed. Each repair phase still re-derives its area from the newest runs.

### Integration notes from the parent landing

- Concurrent commit `457dd728d5` ("fix(publish): build release artifacts from the
  release-please tagged commit") changes Publish's `build` checkout to release-please's
  `sha` output. Publish's job names are unchanged, so the watcher and Landing step 2's
  Publish proof need no change. The Publish run after ci_watch's merge must still show
  `release`, `build`, `install-smoke`, `install-smoke-core-floor` and `publish`
  succeeding.
- The parent lander triaged every parent `PROPOSED FOLLOW-UP:`. It corroborated
  sase-13a, sase-12f, sase-19u and sase-18t, and filed sase-1mq. This epic's Landing
  triages only its own phases' proposals.
- Parent epic sase-1mk has no `--epic-symbol` entries and no `parent_bead`. Its own
  Landing resumes at step 4 once this epic confirms the release.

### What ci_watch needs, all in the same 5-minute tick

The logic and config are unchanged from the parent. To read the logic, open
`gh:bbugyi200/bugyi-chops` with `/sase_repo`. All three of these must hold together:

1. **Master Gate** has completed green at the exact master HEAD SHA.
2. The **newest completed Full CI** run on master is green and finished within
   `heavy_max_age_hours: 6`. A newer red run hides an older green one.
3. **PR #299** is the only open release-please PR. It is `MERGEABLE`/`CLEAN` with every
   check SUCCESS, SKIPPED or NEUTRAL, and no release-please or Publish run is in flight.

### Concurrent work this epic must respect

- Other agents push to sase master. Every phase starts from a fetched, fast-forwarded
  `origin/master` and preserves concurrent changes.
- Other agents also write research reports into the research sidecar. Phase
  research-readme changes only `README.md` there.
- The stale release chains (**sase-1lh**, **sase-1kx**, **sase-1c1**) have no live
  agents. Do not relaunch them. Only the final lander touches them.

## Rules for every phase

- Re-derive your area's current failures from fresh CI (`gh run list`,
  `gh run view --log-failed`) before editing. The snapshot above is a starting point.
- Fix root causes. Never skip, xfail, quarantine or delete a test or assertion. Never
  add rerun-on-failure, and never widen ci_watch or watcher gates. Raise a timeout only
  when the wait is already condition-based and you have evidence the latency is
  legitimate. The expected shape of a fix is a deterministic settle condition: wait for
  the actual state, drain the worker or timer that owns it, or inject the clock or
  scheduler. If the race is a real product bug, fix the product code.
- Prove each flake fix with a stress loop that mimics the CI lane. For example, run the
  node plus its module under `-n 8` for 20–30 iterations, with other test modules
  loading the host. Run the loop before and after the fix where it reproduces. Record
  iterations and failures on your phase bead. If it never reproduces, explain the race
  from the code and test the fixed condition directly.
- A failure in your area that fresh CI shows and the snapshot lacks is yours. For a
  failure outside every phase's area, append a `PROPOSED FOLLOW-UP:` note with run and
  job IDs to your phase bead. Phase workers never create beads and never close task
  beads. The watch phase is the only phase that closes a bead itself: its own.
- Touch shared helpers (`tests/conftest.py`, common Textual pilot wait helpers) only
  when the root cause lives there. Keep that change minimal and backward compatible, and
  note it on your phase bead, since sibling phases run in parallel.
- After editing sase files, read the lint_and_test memory, run `just fix`, then run
  `sase tool run check`. Never run `just check-full`. Until research-readme lands, the
  `SASE validation` stage fails on the research README drift. Treat that one failure as
  known (cite sase-1mm), and require every other stage to pass.
- Never merge a PR by hand, edit versions, changelogs or dependency windows, or change
  ci_watch's code or config. Dispatch release workflows only where this plan says so.
- End with a phase note that names the task beads your fix addresses and the stress
  evidence, so the lander can close those beads with release evidence.

## Phase research-readme

1. Use `/sase_repo` to open the `research` sidecar. Read its `AGENTS.md` if one exists,
   then fetch and fast-forward. If its `README.md` already matches the generated
   template, verify that with step 3 and go to step 4.
2. From the sase checkout, run `.venv/bin/sase init repo --check --diff`. Confirm the
   only planned change is the research `README.md` hunk that removes the hand-added
   `## Topic Tags` section. Restore the generated content with `sase init repo`, or by
   applying exactly that hunk. Then confirm no other sidecar file changed.
   - Keep `TAGS.md` and the `tags:` frontmatter that `dcbfd0dc` added. Only the README
     pointer goes.
   - Do not copy the section into `sidecar-research-README.md`. That template is shared
     by every project's research sidecar, and `TAGS.md` exists only in sase's, so the
     link would break elsewhere.
   - If a supported research-tags convention (a template section plus a generated or
     optional `TAGS.md`) is worth building, record it as a `PROPOSED FOLLOW-UP:`.
3. Verify from the sase checkout that `.venv/bin/sase init repo --check` reports nothing
   to do and that `.venv/bin/sase validate` passes. No sase file changes, so
   `sase tool run check` is not required.
4. Append a phase note that names sase-1mm as resolved by this change and gives the
   verification output. Finalize normally; `/sase_final` lists the research sidecar as a
   repository obligation, and the host commits and pushes it. A sase master push or a
   Master Gate rerun after that push then picks up the fixed README.

## Phase jump-golden

1. Read the tui memory and its tui_screenshot child, and the PNG section of the
   lint_and_test memory. Reproduce the drift with the targeted `--check` command from
   blocker 2, and inspect the report's expected, actual and diff images.
2. Trace how `db14df00e6` changes the JUMP header of the clan members panel. That commit
   touched `_agent_enter_action.py`, `_agent_enter_models.py`,
   `_agent_enter_resolver.py`, `_agent_clan.py` and `_keybinding_bindings_agents.py`.
   The title suffix comes from `_agent_display_clan_roster.py` (`<n> WAVES + LAND`), and
   `b971b55fd6` fits grouped JUMP targets to the rendered space. Explain why the
   Enter-target change makes the suffix appear, and why it renders truncated as `+…`.
3. Decide:
   - If the new title is the intended rendering, update the golden with
     `just fix-tui-screenshots -- <selector>` and inspect the result as the memory
     requires.
   - If it is an unintended side effect, such as Enter resolution or footer bindings
     changing the JUMP layout or width budget, fix the product code so the golden
     matches again.
   - Either way, a half-truncated suffix that should have been fitted or omitted is a
     fit bug. Fix it rather than bless it.
4. Run `just fix-tui-screenshots --check` over
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_jump_panel.py`, and over every
   other visual module covering clan rows, the agents footer or the Act-on chooser.
   Confirm no other golden drifts. Re-derive Full CI `visual-test` from the newest run.
   A golden that drifts there for any other reason is yours too.

## Phase link-follow-kind

1. Reproduce `test_epic_context_selects_and_expands` with a stress loop over
   `tests/ace/tui/test_link_follow_planner.py`, `tests/ace/tui/test_link_follow_seam.py`
   and `tests/ace/tui/test_link_follow_bounded_panes.py` under `-n 8`. Also write a
   deterministic driver that holds a query worker pending while a kind-mismatched chip
   is followed.
2. Fix the completion path in `beads_options.py` and `plans_options.py`:
   - An inventory miss may complete MISSING only when no row with the target's identity
     exists under any kind the follow planner can resolve.
   - A follow the planner can still correct must stay pending until the planner settles.
     One way is to match by project plus bead ID across epics, phases, tasks and flags.
     Another is to let the planner's resolution run before the miss check.
   - Audit every caller of `_loaded_current_snapshot`, since dropping
     `not self._loading` changed its meaning. Restore a narrower predicate where a
     caller needs it.
3. Keep sase-1lr fixed: `test_beads_sync_reports_visible_and_missing` must still report
   MISSING for a truly missing bead. Keep the truncated-archive plans case PENDING until
   the deep scan finishes. Add a regression test for the kind-mismatch race, then stress
   all three modules again.

## Phase new-project-owner

1. Reproduce `test_owner_switch_preserves_per_owner_drafts` with a stress loop over
   `tests/ace/tui/modals/test_new_project_*` under `-n 8` with other modules loading the
   host. It failed in every Full CI leg but passed in isolation for the parent lander (1
   passed in 4.16 s at `457dd728d5`), so also write a deterministic driver that
   refreshes the form after a detach and remount but before the controls compose.
   Confirm the `NoMatches('#repo-creation-form-errors')` traceback from blocker 3.
2. Fix `RepoCreationFormWidget`'s control-readiness lifecycle in
   `src/sase/ace/tui/widgets/repo_creation_form.py`:
   - `_controls_ready` must be `False` whenever the composed controls are absent. Reset
     it on unmount, and re-arm `_mark_controls_ready` on every mount, or derive
     readiness from `is_mounted` plus the controls actually being present.
   - Keep `dabd231e82`'s intent: no evaluation-view refresh before the controls exist.
   - Check every `_controls_ready` guard (`set_values`, `_recompute_and_notify`,
     `_refresh_evaluation_view`, `_apply_advanced_visibility`).
3. Make the create-form task robust. A paint failure after `_create_form_status` is
   assigned must not silently strand the switched-back owner's form. Also audit the
   project-name pending-edit branch that `dabd231e82` added to `_rename_target`, so a
   programmatic input restore during an owner switch is never read as a user edit. After
   any owner switch, the form status must settle to `ready` or `unavailable` for the
   current owner. Add a regression test that switches owners twice and asserts no pump
   task failed. Tests wait on settled state. Re-run sase-1mi's and sase-1m9's nodes and
   every `test_new_project_*` module under the stress loop, and confirm `dabd231e82`'s
   race fixes still hold.

## Phase runtime-flakes

1. **Known continuation:** `test_agent_run_stops_on_generic_output` returned
   `helper_error` instead of `unknown_item`. Find which helper failed under load, such
   as a triage or selector subprocess timeout, a missing fixture binary, or store lock
   contention. Make the test exercise the generic-output path deterministically. Fix the
   product code if a loaded helper failure masks the real classification.
2. **sase-1k2:** `test_provider_drain_e2e_relaunches_stranded_agent` raised a
   `JSONDecodeError` on an empty read at line 333. Find the file it reads and its
   writer. If the writer is non-atomic, write through a temp file plus `os.replace`. If
   the test reads before the record exists, wait on the record's completed state. Prefer
   the product fix when a real reader could see the same partial file.
3. Stress both nodes with their modules under `-n 8` as the rules require.

## Phase watch

This is the release-watch section. Nested epics copy it verbatim, close table included,
substituting only their own bead IDs. Make no source edits, and read `/sase_monitor`
first. **Every path through this phase ends with this phase bead closed**, so the lander
always runs.

1. If this phase bead is already closed, use `/sase_final` and stop.
2. Fetch and fast-forward the sase checkout. Confirm that every sibling repair phase's
   change is on `origin/master`, using its closing note and `git log`. If a change is
   missing, start one `/sase_monitor` 10-minute `sleep` whose `--next` resumes this
   step. If the change is still missing after that, close this bead with
   `sase bead close <watch-bead> -R canceled -r "WATCH_PRECONDITION: <what is missing>" --note "<evidence>"`
   and finalize. Otherwise set `FIXED_SHA` to the fetched `origin/master` tip.
3. Open sase-core with `/sase_repo` and fetch it. Set `CORE` as follows:
   - If the containing release is already published, use that version.
   - Otherwise use the version the open release-plz PR proposes (currently 0.42.0).
   - The chosen version must contain the core-fixture change and `mobile_pairing_issue`.
     Never use 0.41.0.

   If `CORE` is not yet published, confirm the open release PR's head contains the
   core-fixture commit. If every check on that head has completed and one failed, close
   this bead canceled with `-r "CORE_CI_RED: <failing checks>"` and the job URLs as the
   note, then finalize.

4. Run `sase bead epic-symbols <watch-bead>`; it must be empty. Never key an exemption
   to a watch bead. If an entry needs a Justfile edit, close this bead canceled with
   `-r "WATCH_PRECONDITION: epic symbols"` and an explanatory note, then finalize; the
   lander handles it.
5. If PyPI already lists sase 0.18.0, prove it yourself: install `sase==0.18.0` in a
   fresh temporary venv and run `sase --version`. Close this bead normally with that
   output and finalize.
6. Preflight with the concrete values:

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --dispatch-core-cut --once --dry-run --close-bead <watch-bead>
   ```

   The preflight must exit 0 and print a fully parsed state line. If it does not, close
   this bead canceled with `-r "WATCHER_BROKEN (preflight)"` and the output as the note,
   then finalize. Record `FIXED_SHA`, `CORE`, the preflight output and the exact monitor
   command in a note on this bead.

7. Start exactly one monitor, with placeholders substituted:

   ```sh
   sase monitor start -C "$PWD" -r 'Watch sase-core <CORE> and sase 0.18.0 to PyPI; the watcher closes this phase bead' -t 14h -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' -L 'sase 0.18.0 release watch' -m '@small' -n '<NEXT>' -- 'PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --dispatch-core-cut --deadline 12h --close-bead <watch-bead>; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 120; sase bead close <watch-bead> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   `<NEXT>` is this text, with its IDs substituted: "Release-watch closeout for
   `<watch-bead>` of epic `<epic-id>`. Do not restart the watcher, dispatch anything,
   merge, or edit files. Read the monitor result and the verdict file it names. Run
   `sase bead read <watch-bead>`. If the bead is not closed, close it: normally with the
   verdict file as the note only if the verdict is RELEASED; otherwise with
   `-R canceled -r "release not confirmed: <verdict or monitor outcome>"`. Append a
   one-paragraph summary to `<epic-id>` with `sase bead note`, giving the verdict, exit
   code, verdict-file path and last state line. Then use /sase_final."

   Do not pass `--idle-timeout`. Let `sase monitor start` itself run to completion. If
   it fails twice, close this bead canceled with `-r "WATCHER_NOT_STARTED"` and both
   errors as the note, then finalize.

Three layers close this bead: the watcher's own `--close-bead` on every verdict,
including signals; the shell fallback after the watcher exits; and the follow-up's
check. The fallback's canceled close is refused harmlessly when the watcher already
closed the bead as done. A canceled watch is not a release, and the lander decides what
follows.

| Watcher verdict                                                                                                | How this bead closes        | What the lander does                                     |
| -------------------------------------------------------------------------------------------------------------- | --------------------------- | -------------------------------------------------------- |
| RELEASED                                                                                                       | done, verdict file as note  | Confirms the full proof, then lands                      |
| FULL_CI_RED, MASTER_GATE_RED, CORE_CI_RED, CORE_FLOOR_BLOCKED, INSTALL_FAILED, PUBLISH_FAILED, RELEASE_PR_GONE | canceled, verdict as reason | Repairs through a nested epic                            |
| DEADLINE, STOPPED, WATCHER_BROKEN, WATCH_PRECONDITION, WATCHER_NOT_STARTED                                     | canceled, verdict as reason | Runs its own watcher if only waiting remains, else nests |

## Landing

The land agent confirms the release before landing. Closed phases are not proof. Nested
epics copy this section verbatim.

1. **Integration audit.** Read this epic, every phase and note, and this plan. Fetch
   sase and sase-core. Verify each repair phase's change is on master and that no
   concurrent commit undid it. Run `sase bead epic-symbols <epic>` and retire each entry
   under Symvision policy: wire it, privatize it, add a justified pragma, or delete it.
   Re-key an entry only to a still-open nested child bead. If you edit sase files, run
   `just fix` and `sase tool run check`. Never run `just check-full`.
2. **Confirm the release.** Every one of these must hold:
   - sase-core-rs `<CORE>` is on PyPI with every expected distribution. Its `v<CORE>`
     tag contains the core-fixture change, and the macOS CI job for the released commit
     passed.
   - PR #299 was merged by ci_watch. Show the merge commit, `mergedBy`, and the matching
     row in `~/.sase/axe/lumberjacks/ci_watch/ci_watch_releases.json`.
   - The GitHub release v0.18.0 exists.
   - The Publish run after the merge succeeded in `release`, `build`, `install-smoke`,
     `install-smoke-core-floor` and `publish`. A skipped job is not proof.
   - `https://pypi.org/pypi/sase/json` lists 0.18.0.
   - In a fresh temporary directory outside every repo, `uv venv` followed by an
     unpinned `uv pip install --refresh sase` resolves sase 0.18.0, and `sase --version`
     prints it. Record the resolved sase-core-rs version, which must be at least `CORE`.
   - Master Gate was green at the tip ci_watch merged, and the Full CI run it relied on
     was green and less than 6 h old.
3. **If the release is not confirmed,** read the watcher verdict and its evidence, then:
   - **Only external waiting remains** (core wheels still publishing, Full CI in flight,
     Publish not yet run, ci_watch between ticks): start your own `/sase_monitor`
     running the watch phase's watcher command **without** `--close-bead`, with `-t 14h`
     and a `--next` that resumes this Landing at step 2. Do this at most once for an
     unchanged state.
   - **GitHub release v0.18.0 exists but PyPI lacks 0.18.0,** and Publish's build or
     publish failed transiently: run
     `gh workflow run publish.yml -R sase-org/sase --ref master -f publish_existing=true`
     once, then take the waiting path.
   - **Repair is needed** (FULL_CI_RED, MASTER_GATE_RED, CORE_CI_RED,
     CORE_FLOOR_BLOCKED, INSTALL_FAILED, PUBLISH_FAILED, RELEASE_PR_GONE, or a waiting
     path that already ran once without progress): create a nested child epic with
     `/sase_plan`, setting `parent_bead` to this epic. The child gets:
     - one repair phase for every observed blocker, owned outright and never deferred to
       another epic;
     - a watch phase that copies "Phase watch" and its close table verbatim;
     - this Landing section verbatim, so its lander keeps nesting until the release is
       observed, creates the artifact, and resumes its ancestors.

     Record on this epic what the child owns, then propose the plan. That ends your
     turn, and the child's lander resumes this landing through `parent_bead`.

   - **ci_watch is not ticking, or it refuses to merge while all three gates are
     green:** ask through `/sase_questions`. Never merge by hand.
   - Never close this epic, or force-close it, without a confirmed release.

4. **Evidence artifact.** The first lander to confirm the release is the final epic
   lander. It writes `sase_v0_18_0_release_evidence.md` in a fresh temporary directory,
   not inside a repo. The file carries every proof from step 2:
   - run, job and PR IDs, SHAs, URLs and times;
   - PyPI file names with their sha256 for sase 0.18.0 and core `<CORE>`;
   - the unpinned install output;
   - the watcher verdict file's content;
   - the flake beads closed in step 5.

   Then run
   `sase artifact create -p <file> -l "sase v0.18.0 PyPI release evidence" --bead <this-epic>`.
   For each ancestor epic in this chain, add
   `sase artifact link add <artifact> related bead:<ancestor> "v0.18.0 release evidence"`.
   Ancestor landings reuse this artifact and never create a second one.

5. **Follow-ups.**
   - Triage every `PROPOSED FOLLOW-UP:` note through `/sase_new_task`, corroborating
     existing tasks instead of duplicating them.
   - Close each flake task bead that a phase note says it fixed (sase-1a7, sase-1mj,
     sase-1k9, sase-1j4, sase-1ht, sase-1a2, sase-1lr, sase-1mi, sase-1m9, sase-1l7,
     sase-1l8, sase-1ky, plus every task bead a phase of this epic names, including
     sase-1mm and sase-1k2) only when the release's green Full CI and Master Gate runs
     plus the phase's stress evidence support it. Otherwise append the evidence as a
     note.
   - Note the outcome on sase-1m3.7.1. Close sase-10d if the evidence proves it.
   - Triage Install smoke, which should heal after the release, and the 3.12
     coverage-leg time margin (sase-v8).
6. **Stale epics.**

   Parent epic sase-1mk decided `stale_epics = supersede`; that choice is final here.

   After confirming the release, close the **sase-1lh** chain (deepest open descendant
   first, from sase-1lh.7.5.4.3.3.4.4 up to sase-1lh), the **sase-1kx** chain (from
   sase-1kx.5.3.3.3 up to sase-1kx) and **sase-1c1**. Close each with
   `-R superseded --force -r "superseded by <this-epic>: sase v0.18.0 released"`, with
   the artifact ref and a one-line summary as the note. **sase-1i5.9.1** and
   **sase-1jz** still have non-release scope (plugin floors and goldens), so give them
   an evidence note only.

7. **Close and resume.**
   - Close this epic with
     `sase bead close <epic> --note "<proof summary, artifact ref, follow-up dispositions>"`.
     If the close is refused, fix the cited readiness issue and retry. Never use
     `--force` for a successful landing.
   - Run `just symvision`, through the tool wrapper if it is guarded.
   - Open the plans repo with `/sase_repo`, read this plan with `sase artifact read`,
     and set `status: done` in it.
   - If this epic has a `parent_bead`, resume that parent's Landing at step 4 and reuse
     the existing artifact. Repeat up the chain, and stop with a blocker note at the
     first ancestor that is incomplete.
   - Use `/sase_final`, and include every repository you changed.

## Out of scope

- Changing ci_watch's logic or config, or merging any PR by hand.
- Releasing sase-github or sase-telegram, and raising plugin floors (sase-1i5.9.1.4).
- CI Telemetry's leak gate, Deploy Docs, and feature work beyond the listed repairs.
