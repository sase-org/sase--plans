---
tier: epic
title: Unblock the sase v0.18.0 PyPI release and give Full CI a reliable two-hour
  clock
goal: 'sase v0.18.0 is on PyPI and `pip install sase` works again. It ships through
  the normal `ci_watch` path: a green Full CI no more than 6h old, plus a green Master
  Gate at master HEAD. Full CI on sase master starts about every two hours, driven
  by a clock that GitHub''s backlogged cron scheduler cannot drop.

  '
decisions:
  cadence_driver:
    ask: Which clock should start Full CI every two hours?
    choices:
      athena: ci_watch on athena dispatches it; reliable 5-min clock, needs a one-time
        checkout pull
      github: A relay workflow in sase re-dispatches it on completion; self-contained
        but unproven
    default: athena
    why: GitHub delivers this repo's crons hours late; an outside clock is GitHub's
      own remedy
    answer: athena
phases:
- id: test-flakes
  title: Fix the test-lane failures blocking Full CI and Master Gate
  depends_on: []
  size: medium
  description: 'test-flakes: fix the New Project source/create debounce races and
    the sudo canary length false positive that redden today''s Full CI test lanes,
    re-inventory newer red runs, and root-cause the recent Master Gate tip flakes
    where the cause is a test wait or fixture.'
- id: visual-lane
  title: Deterministic captures, then rebaseline the Artifacts-strip goldens
  depends_on: []
  size: medium
  description: 'visual-lane: fix the capture races and host differences behind 5 of
    the 57 drifted goldens, regenerate the rest for the intended retired-Agent Artifacts
    strip, and prove a clean full visual check under TZ=UTC.'
- id: full-ci-clock
  title: Dispatch Full CI on a reliable two-hour clock
  depends_on: []
  size: medium
  description: 'full-ci-clock: replace GitHub''s dropped `17 */2` cron with a guarded
    dispatcher that starts Full CI on master when the newest run is at least 2h old
    and none is in flight.'
- id: release
  title: Prove Full CI green, let ci_watch merge PR 299, and verify v0.18.0 on PyPI
  depends_on:
  - test-flakes
  - visual-lane
  - full-ci-clock
  size: medium
  description: 'release: deploy the clock, drive Full CI to green on a tip with the
    fixes, triage any new reds, confirm ci_watch merges #299, dispatch Publish, verify
    the PyPI install, and prove the two-hour cadence.'
proposed_by: bbugyi200.athena.106
decided_by: reviewer
decided_via: tui
create_time: 2026-10-13 14:15:34
status: wip
bead_id: sase-1kl
---

- **PROMPT:** [prompts/202610/v0_18_0_release_unblock.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_0_release_unblock.md)
- **BEAD:** [sase-1kl](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kl/README.md)

# Unblock the sase v0.18.0 PyPI release

PyPI's newest `sase` (0.17.1) cannot be installed today (`sase-16p`: it pins
`sase-core-rs<0.33`, and PyPI no longer has any release in that range). The fix is
v0.18.0, which has been sitting in release-please PR #299 since 2026-08-30. PR #299 is
`CLEAN`/`MERGEABLE`, its checks are green, and it pins `sase-core-rs>=0.39.0,<0.40.0`;
0.39.0 is on PyPI. `ci_watch` (an AXE job on athena) merges it only when **both**
signals hold:

1. Master Gate is green at the exact master HEAD.
2. The newest completed **Full CI** run is green and at most 6h old.

Full CI has not been green since 2026-08-29. After #299 merges, the `Publish` workflow
tags v0.18.0, builds, smoke-installs, and uploads with no manual approval.

Background: `research:202610/sase_v0_18_0_pypi_release_status.md`.

## Evidence (observed 2026-10-13 against master `1a638eac95`)

Full CI run **38427346934** (scheduled, started 16:03 UTC) is the newest run. It is far
closer to green than earlier runs: lint, perf-floors, ace-page-group-isolation, and
build-core all pass. The deterministic reds from earlier runs (meta-priority,
workflow-async, completion snapshots, Node Finder, proc-producer inventory) no longer
reproduce. Four jobs still fail:

| Job         | Result                            | Failure                                                                                                                                                             |
| ----------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| test (3.12) | 1 failed / 55359 passed           | `tests/ace/tui/modals/test_new_project_source.py::test_filter_and_enter_advance_the_modal_without_an_entry_point`                                                   |
| test (3.14) | 1 failed / 55360 passed           | the same node                                                                                                                                                       |
| test (3.13) | 1 failed / 55359 passed           | `tests/test_sudo_acceptance_credentials.py::test_sudo_runner_invocation_keeps_canary_out_of_process_argv_and_env`: "credential length leaked into process metadata" |
| visual-test | 1365 passed, then `--check` drift | `created=0 updated=57 unchanged=1043 stale=0`                                                                                                                       |

All 1365 visual tests now pass, with no timeouts. That includes the `spread_deck_sticky`
and `left_right_search_committed` nodes from `sase-1ii`. The job is red only because of
golden drift. The drift report is in that run's `ace-visual-artifacts` artifact
(`report/summary.md`, `manifest.json`, and
`report/images/updated-*/{expected,actual,diff}.png`). Cropping the differing regions
gives these classes:

| Class                            | Goldens                                                                                                                                                                                                                                                                                                                                                                                                     | What differs                                                                                                                                                                                                                                              |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Intended, never rebaselined (52) | every `command_line_*` (blocks, completion, doc peek, history search, indexing), every `axe_*` editor image incl. `axe_editor_constrained_width_70x36`, `mini_macro_*` (incl. `location_flow_{picker,finder}`, `pane_*`, `name_*`, `save_diff`, `scoped_frontmatter`), `glossary_preview_card_*`, `repo_preview_card`, `notification_report_modal`, `update_failure_modal_*`, `plan_toast_*`, `link_rail_*` | The golden's Artifacts sub-tab strip still starts `1 ⬡ Agent · 2 ⊙ STITCH …`. The candidate starts `1 ⊙ STITCH`. This is the intended change from `95430b58aa` (feat(ace): finish retired Artifacts Agent pane strip …), which refreshed only 34 goldens. |
| Turn-count race                  | `agents_tab_strip_stale_host_120x40`                                                                                                                                                                                                                                                                                                                                                                        | Golden `main 1`, candidate `main 0`. This is the same settle race `4b99e45907` fixed for `test_running_agent_session_current_runtime_png_snapshots`.                                                                                                      |
| Deck-position race               | `agents_retry_e2e_plan_session_countdown_120x40`                                                                                                                                                                                                                                                                                                                                                            | Golden: MAIN deck `S·A ‹ 2/2 › Reply` with agent reply blocks. Candidate: `C·A ‹ 1/2 › Context`, with an empty body.                                                                                                                                      |
| Unclassified                     | `artifacts_files_nested_strip_120x40`, `artifacts_split_selected_narrow_120x40`                                                                                                                                                                                                                                                                                                                             | `95430b58aa` already regenerated both. The candidate drops the top bar's `inbox:` label, and in the split image it also drops the footer's second-row labels (`issue`, `close`, `launch`, `snooze`) while keeping the keys.                               |

**Master Gate.** Master Gate has been mostly green since `0a1bdf5279`. It is green at
`1a638eac95`, `82aa8a7a88`, and `099fcee221`. The reds on the tip since 2026-10-13 11:12
UTC are single hits:

- `test_new_project_create_flow.py::test_typing_starts_no_sync_probe_and_settles_once`
  (run 38436668576, `['sase-org/fo…', 'sase-org/new'] == ['sase-org/new']`)
- `test_prompt_next_word_inline_tail.py::test_typing_space_inside_parens_shows_tail_ghost`
  (owner `sase-1ib`)
- `test_agents_tab_x_row_lifecycle_e2e.py::test_done_row_dismissed_without_signal_and_stays_hidden`
  (xdist worker crash)
- `fakey/test_provider_drain_e2e.py::test_provider_drain_e2e_relaunches_stranded_agent`
  (owner `sase-1k2`)
- `test_agent_archive_remaining_proof.py::test_mounted_arrival_then_clear_name_keeps_selection`
- the `test_new_project_source` node above

Older Full CI single hits that may recur:

- `test_stream_json_lines_dispatches_stderr_during_a_large_stdout_burst`
- `tool/test_wrapper_fidelity.py::test_owner_child_shares_wrapper_group_and_group_kill_leaves_no_survivors`
- `test_models_panel_runner_limit.py::test_title_footer_and_chooser_show_effective_and_configured_limits`
  (owner `sase-si`)
- `test_agent_load_tiering_production_oracle.py::test_production_machine_query_oracle_repairs_owner_after_index`

**The Full CI clock.** The problem is GitHub's scheduler, not `full.yml`. `full.yml`
(`17 */2 * * *`, concurrency `full-ci`, `cancel-in-progress: false`) produced no
cancelled or superseded runs. GitHub simply delivered few events: Full CI started every
5–9h between 10-03 and 10-13. Every scheduled workflow in this repo shows the same lag:

- `Publish` (`17 */3`) fires every 4–14h.
- `Core Pin Ratchet` (`43 */6`) fires every 8–14h.
- The weekly `Shard Timings Ratchet` (`43 4 * * 1`) fires 5–8h late.

GitHub documents cron delivery as best-effort under load. Editing the cron expression
cannot fix this; only an outside clock can.

**Stalled epic `sase-1jz`.** Phase `.3` landed its fixes in `4b99e45907`, but its joined
check timed out and its agent exited without closing the bead. Phases `.4` (rebaseline),
`.5` (CI proof), and `sase-1jz.land` are still WAITING on `.3`. Their scope assumed the
Models-line and dictionary-card drift, which is already gone. Today's drift is the
Artifacts strip, which `.4`'s acceptance rules would refuse. This epic does that
remaining work. It then hands the evidence back so `sase-1jz` can finish (see Landing).
No phase here edits `sase-1jz` beads before then, so `.4`/`.5` cannot start and race
these phases.

## Rules for every phase

- Read `lint_and_test.md` and `sase_beads.md` through `/sase_memory_read`. For TUI and
  visual work, also read `tui.md` and `tui_screenshot.md`. For any wait, refresh, or
  worker change, also read `tui_perf.md`.
- Re-inventory on the current master tip before editing. Other epics land on master
  continuously. The evidence above is a starting point, not a whitelist.
- **Fix waits and fixtures, never tolerance.** Do not raise timeouts, add retries around
  assertions, skip, xfail, loosen pixel comparison, or delete assertions. Wait on the
  semantic state the assertion or capture is about. Do not wait on a fixed
  `pilot.pause(<seconds>)` or on visual idle.
- Never copy CI candidate PNGs into the tree. Regenerate a golden only after its test is
  deterministic.
- Shared domain behavior stays in sase-core (`rust_core_backend_boundary`). None of
  these fixes is expected to need a Rust change.
- Run `just fix`, then `sase tool run check`, for every changed checkout of this repo.
  Use the repo's own check in bugyi-chops and chezmoi. Do not run `just check-full`;
  Full CI is the exhaustive signal. The CI `lint` job enforces `toobig`, so do not grow
  a file past its line limit.
- Run long commands through `/sase_monitor` (`TESTING`/`TESTED` for verification). Use
  it to wait for CI as well.
- Phase workers create no beads. Record `PROPOSED FOLLOW-UP:` notes on your own phase
  bead. In your close note, name every existing task bead your evidence resolves
  (`sase-1gh`, `sase-19d`, `sase-1kg`, `sase-1ib`, …) so the land agent can close it.
  Run `sase bead epic-symbols <own-phase-id>` before closing your phase with
  `sase bead close <id> --note '<evidence>'`, and land through `/sase_final`.

## Phase test-flakes

Must fix (the first two redden today's Full CI in all three test lanes):

1. **New Project source filter race.** In
   `tests/ace/tui/modals/_new_project_source_flows.py`,
   `test_filter_and_enter_advance_the_modal_without_an_entry_point` types
   `sase-org/new`, calls one `pilot.pause()`, and then asserts that
   `#new-project-source-detail` shows the text. That panel updates through
   `DetailPanelDebouncer` (`src/sase/ace/tui/modals/new_project_source.py`), so on a
   loaded runner the assertion runs before the debounce fires.
   - Wait (`sase.ace.testing.wait.wait_for` or the file's existing helper) until the
     detail shows `sase-org/new`.
   - After `enter`, wait for the stage to reach `configure` and for the stepper to show
     `✓ Source`.
   - Audit the sibling flows in the same file for the same pause-then-assert pattern
     against debounced or worker-fed state, and fix each one.
   - If the debouncer can deliver a stale probe after a newer keystroke, that is a
     product bug: fix it in the modal, not the test.
2. **Sudo canary length false positive** (`sase-1gh`; check whether it also explains
   `sase-19d`).
   - The bug: `_sensitive_tokens` in `tests/_sudo_acceptance_helpers.py` treats
     `str(len(canary))` (`8119`) as a bare substring. The 3.13 lane matched it inside
     the child's inherited `/proc/<pid>/environ`, where the runner's long tokens and IDs
     easily contain those four digits.
   - The fix:
     - Keep the value and sha256 checks strict everywhere.
     - Match the length token only as a standalone decimal (digit boundaries on both
       sides).
     - For process environ, inspect only `KEY=VALUE` entries that the code under test
       added or changed relative to the parent's environment.
   - Apply the same digit-boundary rule to the file-scanning helper used by
     `test_sudo_local_flow_never_persists_canary_credentials` (`sase-19d`). Persisted
     timestamps and IDs have the same exposure there.
   - Add a helper unit test:
     - An embedded `…1281199…` passes.
     - A delimited `length=8119` or JSON `8119` fails.
     - An inherited-but-unchanged env entry is ignored.
3. **New Project create settle race.** In
   `tests/ace/tui/modals/test_new_project_create_flow.py`,
   `test_typing_starts_no_sync_probe_and_settles_once` clears `probe_calls` after a
   fixed `pilot.pause(0.6)` that follows `enter_create_configure`. Under load, the
   initial draft name's debounced availability probe lands after the clear. That is why
   the probe list reads `['sase-org/fo…', 'sase-org/new']`.
   - Wait until the initial probe has settled before clearing.
   - Replace the post-typing fixed pause with a wait on the settled `sase-org/new`
     probe.
   - Keep the exactly-one-probe assertion.

Then:

4. Re-inventory every red test job in Full CI and Master Gate runs on master newer than
   `1a638eac95`:
   - `gh run list -R sase-org/sase --workflow full.yml` and `--workflow master-gate.yml`
   - `gh api repos/sase-org/sase/actions/jobs/<id>/logs` (pass
     `--allow-escape-sequences` and strip ANSI)
   - Fix every deterministic failure.
5. Best effort, timeboxed to read-the-test-and-fix: for each tip flake listed in
   Evidence, look for a pause-then-assert or wall-clock race and fix it when the cause
   is identifiable. Otherwise record the failing output and a hypothesis as
   `PROPOSED FOLLOW-UP:` naming the existing owner bead.
6. Proof:
   - Run each repaired node 20 times serially, then its file 3 times under `-n 8`, and
     record the pass counts.
   - Run `sase tool run check`.
   - Confirm `tests/ace/tui/test_proc_producer_inventory.py` passes on the tip. If it
     does, `sase-1kg` is already fixed; say so in the close note.

## Phase visual-lane

Make `just fix-tui-screenshots --check` clean on GitHub's UTC runners.

1. Rebase on current master. Run the targeted check over the drifting test files twice:
   once normally, and once with `TZ=UTC` exported for the whole command.
   - `just fix-tui-screenshots --check -- <files>` through `/sase_monitor`.
   - Drift that appears locally too is universal. Drift that appears only in CI or only
     under `TZ=UTC` is host-dependent.
   - Inspect the report images for every record. The CI artifact from run 38427346934 is
     the reference for what the runners render.
2. **Turn-count race.** Make
   `test_ace_png_snapshots_agent_tab_strip.py::test_agents_tab_strip_stale_host_png_snapshot`
   wait for the settled main-turn count before capturing. Reuse the wait that
   `4b99e45907` added to `test_ace_png_snapshots_agents_agent_sessions.py`.
3. **Deck-position race.** Make
   `test_ace_png_snapshots_agents_retry_e2e.py::test_real_loader_plan_agent_session_retry_countdown_png_snapshot`
   wait until the MAIN deck has landed on the block its golden is meant to show (`2/2`
   Reply with the agent reply blocks), not the initial `1/2` Context frame.
   - If the landing block itself is nondeterministic in production (a selection race),
     fix that instead.
   - Then decide from the test's intent which frame is correct, and regenerate.
4. **`artifacts_files_nested_strip` and `artifacts_split_selected_narrow`.** Find out
   why the CI render drops the `inbox:` label and the footer's second-row labels. The
   possible causes are a width or responsive-compaction input that differs per host
   (font metrics, `HOME`, user name, terminal size, clock), a capture race, or a later
   intended change.
   - Make the test host-independent, or fix the race, before regenerating.
   - If it is an intended change, regenerate and cite the commit.
5. Regenerate the 52 intended Artifacts-strip goldens and every golden repaired in steps
   2–4:
   - Use targeted `just fix-tui-screenshots -- <files>` first.
   - Then run the full `just fix-tui-screenshots` through `/sase_monitor`. It must end
     `applied` or `clean`, never `partial`, with zero failed tests.
   - Inspect every update group. Accept only:
     - the retired `1 ⬡ Agent` sub-tab, and the strip reflow it causes;
     - the frames that steps 2–4 deliberately fixed;
     - changes from other commits landed since `1a638eac95` that you trace to an
       intended commit.
   - Anything else is a determinism bug. That includes time tokens, hostnames,
     `/home/<user>` paths, and loading placeholders. Fix it, or record it.
6. Proof:
   - Run the full `just fix-tui-screenshots --check` with `TZ=UTC` exported, through
     `/sase_monitor`. It must report `clean` with `created=0 updated=0 stale=0` and no
     failed tests.
   - For each repaired test, run the targeted check five times.
   - Run `sase tool run check`.

Owned goldens: everything listed in Evidence plus anything a newer master commit left
stale. Coordinate with `sase-1jz` only through the Landing step.

## Phase full-ci-clock

Root cause (see Evidence): GitHub delivers this repo's scheduled events hours late or
not at all, so `full.yml`'s `17 */2 * * *` cannot keep the 2h cadence that the two-speed
CI decision and `ci_watch`'s 6h freshness window assume. Keep `full.yml`'s cron and
`workflow_dispatch` unchanged. A delivered cron event is a free backstop; a run it
starts while another is in flight queues behind it in the `full-ci` concurrency group.
The dispatcher counts that run, so it does not start an extra one.

Dispatch rule (both choices): once per tick, read Full CI's recent runs on `master` with
a per-workflow query. Do not use the shared all-workflow run window; on sase master, 100
runs span only about 25h (45 Master Gate plus 45 Deploy Docs).

- If any run is in an in-flight status (`queued`, `in_progress`, `waiting`, `pending`,
  `requested`), do nothing.
- Else, if the newest run's `created_at` is less than 2h ago, do nothing and report the
  next due time.
- Else, dispatch Full CI on `master`.
- Never dispatch twice for the same due window.

> [!decision] cadence_driver = athena
>
> Extend `ci_watch` in bugyi-chops (`sase repo open gh:bbugyi200/bugyi-chops`,
> `src/bugyi_chops/ci_watch.py`, tests in `tests/test_ci_watch.py`). It already ticks
> every 5 minutes on athena, owns the Full CI freshness gate, and has bounded `gh`
> plumbing plus dry-run handling.
>
> 1. Add the per-repo var `heavy_dispatch_every_hours`. Parse it like
>    `heavy_max_age_hours` (`_release_setting_values`). If absent or 0, dispatch is off.
>    It applies to that repo's `heavy_workflows` on its default branch.
> 2. Each tick:
>    - Apply the dispatch rule with a bounded per-workflow read, using
>      `repos/{repo}/actions/workflows/{workflow}/runs` filtered to the branch, or
>      `gh run list --workflow`.
>    - In live mode, dispatch with
>      `gh workflow run <workflow> -R <repo> --ref <branch>`. In dry-run, report
>      `would dispatch`.
>    - Persist the last dispatch time per repo and workflow in the job state, and skip
>      redispatch for at least 15 minutes. That covers a run that is slow to appear in
>      the list.
> 3. Keep dispatch independent of release decisions, incident notifications, and merges.
>    A failed read or dispatch is reported in the summary line and the CI WATCH report.
>    It must never flip release eligibility, raise an incident, or abort the tick.
> 4. Add one heavy-dispatch line per repo to the compact stdout summary and the CI WATCH
>    report: newest run age, then in-flight, not-due (with the due time), dispatched, or
>    would-dispatch.
> 5. Update the module docstring and mutation allowlist. The job's GitHub mutations
>    become the guarded release merge and the guarded heavy-workflow dispatch.
> 6. Add tests for:
>    - due → dispatch;
>    - not due;
>    - in flight (each status);
>    - the dedupe window;
>    - dry-run;
>    - disabled;
>    - a dispatch failure that leaves merge evaluation untouched;
>    - a per-workflow read that ignores other workflows' runs.
>
>    Run the bugyi-chops checks.
>
> 7. In chezmoi (`sase repo open chezmoi`, `home/dot_config/sase/sase_athena.yml`):
>    - Add `heavy_dispatch_every_hours: {sase-org/sase: 2}` to the `ci_watch` job vars.
>    - Update the routine and job `description:` text that says the only GitHub mutation
>      is the guarded merge.
>    - Run `sase axe job doctor`. Deployment happens in phase release.

> [!decision] cadence_driver = github
>
> Add `.github/workflows/full-ci-relay.yml` to this repo:
>
> - Triggers: `workflow_run` on `Full CI` `completed`, `workflow_dispatch`, and a
>   frequent off-the-hour cron backstop (for example `7,37 * * * *`).
> - `permissions: {actions: write, contents: read}`.
> - Its own concurrency group, with `cancel-in-progress: true`, so a newer trigger
>   replaces a sleeping one.
>
> One small job applies the dispatch rule with `gh` and `GH_TOKEN: ${{ github.token }}`.
> When the next due time is at most 60 minutes away, it sleeps until then, re-checks for
> in-flight runs, and dispatches with `gh workflow run full.yml --ref master`.
> `GITHUB_TOKEN` is allowed to create `workflow_dispatch` runs. Whether `workflow_run`
> fires after a token-dispatched run is unverified, so the release phase must observe
> the chain. Put any logic beyond a few lines in a `tools/` script with unit tests, and
> extend `tests/test_github_actions_ci_workflow.py` to pin the triggers, permissions,
> concurrency, and the guard. Run `sase tool run check`.

## Phase release

1. **Deploy the clock.**

   > [!decision] cadence_driver = athena
   >
   > After the full-ci-clock commits are on both repos' remotes:
   >
   > - Run `chezmoi update -a --force` (chezmoi's apply-after-commit rule).
   > - The `sase` uv tool env imports `bugyi_chops` editable from the user's primary
   >   checkout (`~/projects/github/bbugyi200/bugyi-chops`). Request a
   >   `git -C <that checkout> pull --ff-only` through `/sase_gate` rather than running
   >   it yourself.
   > - Once the gate is answered, run `ci_watch` once manually
   >   (`sase axe job run ci_watch`). Confirm its summary shows the heavy-dispatch line,
   >   and that it dispatches if Full CI is due.
   >
   > Do not block on the gate: dispatch Full CI by hand meanwhile.

   > [!decision] cadence_driver = github
   >
   > Confirm `Full CI Relay` is active (`gh workflow list -R sase-org/sase`) once it
   > reaches master.

2. **Get a run on the fixed tip.**
   - Wait until Master Gate is green on a master tip that contains the test-flakes and
     visual-lane commits.
   - Make sure a Full CI run starts on that tip: let the clock do it, or run
     `gh workflow run full.yml -R sase-org/sase --ref master`.
   - Monitor it to completion through `/sase_monitor` (about 1h50m).
3. **Triage any red.** Classify every failure.
   - Fix deterministic regressions and identifiable waits or fixtures directly, under
     the same rules.
   - For a single unrootable flake, note it on the owner bead's evidence trail in your
     phase notes, and let the next clock run retry.
   - If four consecutive Full CI runs fail on causes you could not fix, stop. Close the
     phase with the evidence and report.
4. **Confirm the merge.** When Full CI is green, watch `ci_watch`'s CI WATCH report or
   `sase notify` for the guarded merge of #299. The merge needs Master Gate green at the
   exact HEAD and the green Full CI no more than 6h old.
   - If `ci_watch` reports a blocker, fix it in scope, or record it. Possible blockers:
     #299's `release-core-floor-smoke` going red because master raised the core pin past
     0.39 (then follow the core-window ratchet, `tools/ratchet_core_window`),
     `release_generator_busy`, or `gating_workflow_red`.
   - Do not merge #299 by hand.
5. **Publish.** After the merge, run
   `gh workflow run publish.yml -R sase-org/sase -f publish_existing=false` instead of
   waiting for Publish's lagging cron. Monitor the tag, build, both install smokes, and
   the PyPI upload.
6. **Verify the install.**
   - PyPI's JSON API lists `sase` 0.18.0.
   - In a fresh temporary venv, `pip install sase==0.18.0` resolves, pulling a
     `sase-core-rs` 0.39.x.
   - `sase --version` prints 0.18.0.
7. **Prove the cadence.** After deployment, show at least three consecutive Full CI runs
   on master that start no more than 2h15m apart, triggered by the new clock (the event
   and actor show it).
   - > [!decision] cadence_driver = github
     >
     > Confirm that the relay fired from `workflow_run` after a token-dispatched run. If
     > it did not, record that, plus the cron backstop's actual delivery, as
     > `PROPOSED FOLLOW-UP:` (switch to the athena driver).
8. **Close note.** Record:
   - the Full CI run ID and SHA, the Master Gate run ID, the #299 merge commit, the
     Publish run, the PyPI version, the install proof, and the cadence timestamps;
   - the task beads this resolves: `sase-16p`, plus those named by test-flakes and
     visual-lane.

## Landing

The land agent closes the task beads the phases name as resolved, each with that phase's
evidence (at least `sase-16p`, `sase-1gh`, and `sase-1kg` if confirmed). It then
releases the stalled `sase-1jz` epic:

- Close `sase-1jz.3` with a note. `4b99e45907` landed its deck-race fixes. Full CI
  38427346934 and this epic's green Full CI passed every visual test, including both
  `sase-1ii` nodes, so `sase-1ii` is resolved.
- Add a note to `sase-1jz.4` and `sase-1jz.5` that points to this epic's rebaseline and
  same-SHA green Full CI evidence, so their waiting agents can verify and close quickly
  instead of redoing the work.
