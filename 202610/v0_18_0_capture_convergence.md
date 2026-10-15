---
tier: epic
title: Finish the remaining capture races and confirm sase 0.18.0 publication
goal:
  The remaining visual capture defects are repaired, the full visual lane is clean, and
  the committed repairs ship in an installable sase 0.18.0 through the guarded release
  path.
parent_bead: sase-1kx.5.3
phases:
  - id: capture-convergence
    title: Finish the five observed capture states and prove the full visual lane
    depends_on: []
    size: medium
    description:
      "capture-convergence: settle the visible deck and top-bar state, isolate Logs
      telemetry, and await complete Tool Runs detail; verify the repaired fixtures."
  - id: publication-proof
    title: Observe the committed repairs through guarded publication
    depends_on:
      - capture-convergence
    size: small
    description:
      "publication-proof: make no file changes, observe CI and publication on the landed
      repair commit, and always close this phase with an honest release verdict."
proposed_by: bbugyi200.athena.sase-1kx.5.3.land
create_time: 2026-10-14 22:37:08
status: wip
---

- **PROMPT:**
  [prompts/202610/v0_18_0_capture_convergence.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_0_capture_convergence.md)
- **PARENT:**
  [202610/v0_18_0_visual_release_completion.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_0_visual_release_completion.md)

# Finish the remaining release acceptance

This is remaining work for **sase-1kx.5.3**, linked to
`plan:202610/v0_18_0_visual_release_completion.md`. Its directly parented plan ancestors
are **sase-1kx.5** (`plan:202610/v0_18_0_release_finish.md`) and **sase-1kx**
(`plan:202610/v0_18_0_release.md`). Read all three artifacts through
`sase artifact read` and their landing notes before working. Preserve the accepted
decisions **core_release=dispatch** and **stale_epic_1jz=notes**.

The repairs fit one medium coding agent. This is an epic solely because the required
remote CI and publication proof must observe that agent's future landed commit. The
second phase makes no edits and always closes; this plan's `parent_bead` then returns
control to the interrupted landing. Do not put an ancestor close into a coding phase,
and do not wait for a coding phase's own SHA before it ends.

## Audit evidence and completed work

The land audit fetched origin: local HEAD and `origin/master` were both
**2465bc7a1814aff6dcc5232e5218b9a424f843d3**. This is the only commit since sase-1kx.5.3
started and is its repair phase's commit; there is no intervening base integration to
apply. Fetch and review any subsequent commits before implementing.

Both children and every note were reviewed. Phase `.1` is done; phase `.2` is
deliberately **canceled**, reason `release not confirmed: FULL_CI_RED`. Cancellation
released landing and is not evidence that publication succeeded.

The `.1` source changes implement semantic archive-detail waiting, card navigation
readiness, explicit scrollbar treatment, and the Argon model fixture integration. Its
check ToolRun **1f79d9dcb529c985048a5a855d9f2a7a** was independently inspected:
`just check`, exit 0, triage pass, 18m56s overall. All **60 watcher tests** passed
independently in 4.72s. The watcher correctly recognizes the actual prefixed executed
drift markers from the current CI log. The anchored/ANSI/job-aware marker parser, stable
core version parser, and normalized gh environment are complete: do not redo them.
Preserve their stdlib-only implementation and close/verdict contracts.

Full CI **38653006628** on that SHA passed every executed nonvisual job and all **1366
visual tests** (1 skipped), but failed comparison with
`created=0 updated=5 unchanged=1095 stale=0`. All five original scoped images are absent
from this drift inventory. The new five are described below. The complete HTML report,
with embedded expected/candidate PNGs and candidate SVGs, is
**file:explicit:3435087f16e0e38529c2fe7f**. Read it with `sase artifact read`; obtain
its path with `sase artifact path` for image extraction. GitHub source is run
38653006628, artifact **11940791979**, `visual-failure-report.html`; the artifact
download returned HTML bytes directly, not a ZIP.

Earlier Full CI **38650800228** on the same SHA timed out at
`test_agents_deck_blocks_spread_deck_sticky_png_snapshot` waiting 15s for spread mode,
and Python 3.14 failed
`test_reverse_range_cycles_backward_wraps_and_reenters_from_custom` at its existing 5s
wait. Both passed in 38653006628. Keep the exact spread regression in verification; its
existing owner is `sase-1ii` / `sase-1jz.3`.

At audit, Master Gate **38640302111** is green on the repair SHA. PR **#299** remains
open, head **2917ea5529982ef6dc56ab014a43b54e9c81468d**. Publish **38656927396**
succeeded only in release metadata jobs: build, publish, and install smokes were
skipped. PyPI still reports sase **0.17.1**, no 0.18.0; core **0.40.0** is published. Do
not dispatch another core cut for the already satisfied minimum.

Full CI **38669105903** was still running at audit. Its `perf-floors` job had already
failed `test_tool_runs_harness_passes_every_hermetic_case`: `dod-10-concurrent` observed
six distinct stage IDs rather than twelve, and `dod-7-triage-safety-net` had empty
diagnostics. Its earlier same-SHA perf-floors job was green. This distinct unrelated
harness flake is tracked by **sase-1l6**. Read the parent landing note for follow-up
disposition and inspect the run's final result; do not duplicate an in-flight run or
classify this as a visual regression.

## Rules

- Read `lint_and_test.md`, `symvision.md`, `tui.md`, `tui_screenshot.md`, and
  `tui_perf.md` through `/sase_memory_read`. Open other repositories only through
  `/sase_repo` and read sidecar artifacts through `sase artifact read`.
- Preserve assertions, timeouts, exact pixel equality, release guards, and normal close
  semantics. No sleeps, assertion retries, skip/xfail, copied CI candidates, or blanket
  golden updates. Change production behavior only for a demonstrated defect; shared
  domain behavior belongs in Rust core.
- File-change verification is `just fix` then `sase tool run check` (wrapped
  `just check`). **Do not run `just check-full`.** Unrelated check-full-only
  infrastructure failures are follow-ups, not remaining epic implementation.
- Use `/sase_monitor` for long visual runs and external waits. Wait for the start
  command itself to exit. Coding phases record unrelated findings as
  `PROPOSED FOLLOW-UP:` notes; they do not create task beads.
- No manual release PR merge, version edit, global install, primary-checkout deploy, or
  killing the waiting sase-1jz agents.

## Phase capture-convergence

Inspect each expected/candidate pair before editing. These are observed states, not
permission to replace a golden with whichever frame happens to appear locally.

1. **Renamed session header, 3617 changed pixels.**
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_agent_sessions.py::test_renamed_generic_agent_session_root_png_snapshot`
   produces `agents_renamed_generic_session_root_120x40`. Expected shows **Reply 2/2**,
   while CI shows **Context 1/2**; the body and remaining image match. The new
   `select_navigable_main_card(page, "reply")` already runs and its assertions passed,
   so another check of `active_main_card()` alone is insufficient. Trace
   `_ace_agents_png_snapshot_helpers.py`, `panel_interaction.py`, deferred Main view
   application/restoration and the rendered header. Establish the final selected
   identity, view generation, document and visible active-card header before one
   deliberate selection and capture. The existing `_main_view_ready_to_cycle` helper
   demonstrates an applied-generation barrier. Determine whether this is a fixture
   readiness gap or a real stale repaint; add a behavioral regression if production code
   is required. Recheck all callers of the readiness helper, including the original
   gate, weighted-capacity and neighbors cases.

2. **Top-bar density, 389 pixels in each of two images.**
   `test_ace_png_snapshots_artifacts_plans.py::test_plans_filter_completion_png_snapshot`
   and
   `test_ace_png_snapshots_link_reveal_chip.py::test_beads_link_reveal_chip_png_snapshots[size1-link_reveal_chip_beads_60x30]`
   lose only the **inbox:** label. Trace `TopBar.fit_top_bar_indicators`,
   `TabBar.content_cells`, and `TopBarIndicators.density`. The split-layout file has a
   narrow local full-density wait; integrate the appropriate semantic wait across these
   fixtures without blindly assuming every 60-column capture is compact or forcing the
   production density. Prove the final available geometry and intended density for these
   actual tab strips before capture.

3. **Logs capture isolation, 2645 pixels.**
   `test_ace_png_snapshots_config_center_logs.py::test_config_center_logs_tab_toasts_png_snapshot`
   expects **5 active** log sources and empty Agent Load Timing. CI shows **6 active**
   and a new **323B / now** timing file. The seeded logs are not the only writers:
   `_loading_disk_support.py` schedules `sase.logs.log_tui_agent_load`, which appends to
   `tui_agent_loads.jsonl` on slow loads. Isolate this fixture from unseeded live
   telemetry at the narrow writer/path seam; preserve the explicitly seeded toast and
   error content and production telemetry. Check all Logs-tab snapshots using the same
   seed helper. Do not accept a host-load-dependent byte count or wall-clock age into a
   golden.

4. **Tool Runs detail, 47766 pixels.**
   `test_ace_png_snapshots_tool_runs_cards.py::test_tool_runs_card_png_snapshots`
   produces `tool_runs_card_failed_120x40` with **run detail unavailable · summary
   retained**, replacing stages, triage, child run, and log tail. `_CARD_CASES` waits
   only for `3 NEW`, which the summary placeholder already contains.
   `seed_tool_run_surfaces` clears and fills the real detail LRU through its loader; the
   production widget paints cached summaries before an incomplete-detail worker
   finishes. Prove the selected run's loaded detail and visible stages/tail, preserving
   the intentionally pruned-detail case. Verify the cache-completeness/worker path
   rather than simply manufacturing an already-loaded document. If the worker can fail
   to repaint, fix and test that product defect. Recheck every card case and
   `spread_deck_sticky` / `left_right_search_committed`.

These five images are four bounded capture problems and fit one implementation phase.
Preserve the already repaired watcher and original five captures. Regenerate only a
golden whose intended final state genuinely changes, after proving the state and
inspecting the image; a readiness repair normally needs no golden change.

Run the targeted visual checks in the default zone and UTC, repeat to exercise late
repaints, then run full **`TZ=UTC just fix-tui-screenshots --check`** through a monitor.
Require no failed tests and `clean`, `created=0 updated=0 stale=0`; partial or an
isolated repeat of two failed nodes does not establish a clean full run. Inspect and
attribute any new mismatch. The parent already authorizes full visual acceptance;
unrelated defects retain their task ownership and must not be silently rebaselined. Run
the 60 watcher tests if shared helpers affect them, then `just fix` and
`sase tool run check`. Record exact evidence, resolve this phase's epic-symbols, close
this phase normally when complete, and let host finalization commit it.

## Phase publication-proof

This phase changes no files. It always closes with an honest outcome so landing runs.

1. Read the repair phase's notes. Fetch and fast-forward a clean checkout; prove the new
   repair commit and 2465bc7a18 are on origin/master. Use the **new repair SHA** as
   FIXED_SHA so historical deterministic reds cannot terminate this watch. Inspect newer
   core pins using `/sase_repo`; core 0.40.0 already satisfies the audited pin. Only
   repeat the authorized urgent core-release flow if a newer pin actually needs it, and
   never duplicate an equivalent running release.
2. Inspect Full CI and PR #299 without duplicating an in-flight workflow. Resolve this
   phase's `sase bead epic-symbols <phase-id>` before giving the watcher close
   authority. Preflight inline and require exit 0 plus one fully parsed state line:

   ```sh
   NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version 0.40.0 --once --dry-run --close-bead <phase-id>
   ```

   On preflight failure, close this phase canceled with reason
   `release not confirmed: WATCHER_BROKEN (preflight)` and the evidence, then finalize.
   Do not repair code in this phase.

3. Start one `/sase_monitor`, timeout 13h, labels `WATCHING RELEASE` /
   `RELEASE WATCHED`:

   ```sh
   NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 5m 760m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version 0.40.0 --deadline 12h --close-bead <phase-id>
   ```

   The watcher owns bounded deduplicated dispatch; ci_watch owns guarded merge. The
   monitor successor reads the verdict, repairs a mechanical phase-close failure only
   (done for RELEASED, canceled with the precise reason otherwise), and finalizes. It
   must not reopen the phase, edit files, or restart the watch. If monitor start fails
   twice, close canceled with WATCHER_NOT_STARTED and both errors. Wait for any yielded
   monitor-start session to exit.

## Landing and ancestor resumption

Closing both phases is not sufficient. This child epic's lander must verify its
implementation, notes, latest-base integration, clean full visual evidence, fresh green
Full CI and Master Gate, PR #299's merge, GitHub v0.18.0, successful Publish **build and
publish jobs**, and PyPI 0.18.0. Also use a fresh temporary environment outside the
repository for **unpinned `pip install sase`**, proving resolution to 0.18.0 and working
`sase --version`; record the resolved core version. The watcher's pinned smoke does not
replace this unpinned acceptance. If only external waiting remains, use `/sase_monitor`
without epic `--close-bead`; if implementation remains, use the tier-aware
remaining-work flow. Never treat a canceled watch as success.

After this child epic lands, `parent_bead: sase-1kx.5.3` resumes the interrupted
landing. Review that parent's previous landing note, all descendants and notes, linked
plan and post-child drift. Recheck descendant and linked-plan readiness. Resolve or
re-key every `sase bead epic-symbols sase-1kx.5.3` entry, close normally with an
evidence-rich `sase bead close sase-1kx.5.3 --note "<verification>"`, run
`just symvision`, and set `status: done` in
`plan:202610/v0_18_0_visual_release_completion.md` using the sanctioned plans checkout.
These are landing actions, not implementation phases. Never use `--force` to advance a
successful nested landing.

Repeat readiness and drift checks for **sase-1kx.5**, retire its epic-symbols, close
normally, run symvision, and mark `plan:202610/v0_18_0_release_finish.md` done. Then
recheck **sase-1kx** and `plan:202610/v0_18_0_release.md`. Its outstanding closeout
includes evidence-based closure of sase-16p, sase-1gh, sase-19d, sase-1kg, and sase-1ii;
follow the accepted **notes** branch for sase-1jz (verify before closing .3, note .4/.5
with green evidence), and report whether ci_watch's heavy dispatcher is live. Preserve
historical follow-up dispositions from each landing note. Only when fully ready, retire
its epic-symbols, close normally, run symvision, and mark its plan done. Inspect the
next parent link and continue only through ready plan ancestors; stop and note the first
incomplete or ambiguous parent. Do not deploy to the primary.

## Follow-up disposition

The proposing lander completed task triage and recorded final task IDs and evidence in
the parent **sase-1kx.5.3** landing note before submitting this plan. Both new tasks
below are ready. Preserve these dispositions in eventual close notes.

- `.1` note #1: the pager history-frame evidence corroborates **sase-1f9**. The distinct
  cause-frame intermittent drift is **sase-1l5**, a large flake task because the root
  cause is unproved. It is related to sase-1et and sase-1f8, but their known diff-range
  styling and deterministic age issues do not establish duplication.
- `.2` note #2: decline an omnibus FULL_CI_RED task. The five current ACE mismatches
  remain release acceptance work in this plan. Record concrete Tool Runs/deck overlap on
  **sase-1jz**, preserve its waiting agents, and corroborate **sase-1ii** for the spread
  timeout. The unrelated Statistics failure corroborates **sase-1a2**.
- The separate ToolRun smoke failure from run 38669105903 is **sase-1l6**, a large flake
  task with the exact failed cases and a passing same-SHA witness. Its root is unproved;
  it is related to sase-1k8 and sase-1jl without claiming duplication. It is not
  evidence that the visual or watcher changes broke backend recording. Do not weaken
  release acceptance or omit it from the final CI verdict.
