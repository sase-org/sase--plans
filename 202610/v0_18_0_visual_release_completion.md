---
tier: epic
title: Finish the five release-blocking visual drifts and prove v0.18.0 publication
parent_bead: sase-1kx.5
goal: The five observed visual drifts and remaining watcher parsing defects are repaired,
  and committed fixes pass the guarded release path through a confirmed installable
  sase 0.18.0.
phases:
- id: deterministic-captures
  title: Repair the five capture states and remaining watcher parsing gaps
  description: 'deterministic-captures: pin the observed footer, archive, deck, and
    scrollbar states; harden watcher input parsing; verify the integrated changes.'
  depends_on: []
  size: medium
- id: publication-proof
  title: Observe committed fixes through CI and publication without editing files
  description: 'publication-proof: preflight and monitor the release on the repair
    commit, always close the phase with an honest verdict, and hand evidence to landing.'
  depends_on:
  - deterministic-captures
  size: small
proposed_by: bbugyi200.athena.sase-1kx.5.land
create_time: 2026-10-14 15:28:09
status: wip
bead_id: sase-1kx.5.3
---

- **PROMPT:** [prompts/202610/v0_18_0_visual_release_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_0_visual_release_completion.md)
- **PARENT:** [202610/v0_18_0_release_finish.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_0_release_finish.md)
- **BEAD:** [sase-1kx.5.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kx/sase-1kx.5.3.md)

# Complete the remaining v0.18.0 release acceptance

This is remaining work for **sase-1kx.5**, whose plan is
`plan:202610/v0_18_0_release_finish.md`. Its parent is **sase-1kx**, with plan
`plan:202610/v0_18_0_release.md`. Read both through `sase artifact read`, and read the
current landing notes on both beads. The accepted decisions remain
`core_release=dispatch` and `stale_epic_1jz=notes`.

There is one coding phase. A separate no-edit phase is necessary because release
acceptance requires GitHub CI and publication **after that coding phase's commit is
landed**. A tale coder cannot observe its own future commit before ending its turn. The
no-edit phase always closes, and this child epic's `parent_bead` resumes the interrupted
landing. Do not put ancestor closure in an implementation phase.

## Verified starting point

The land audit fetched `origin/master`; both it and local HEAD were
`4d8940d4b7e102c1f6e031f4b59e3642fdbd2931`. The current epic's implementation is
`960f1e1b670a6993ef885506359f9b3166ae318b`. Both child beads and every note were read:

- `sase-1kx.5.1` closed done. Its recorded `just check` ToolRun
  `933abe374ee28172dfa8b09d2c1bd597` was independently inspected: succeeded, exit 0,
  triage pass. It reported a clean full UTC visual run locally.
- `sase-1kx.5.2` closed **canceled**, reason `release not confirmed: FULL_CI_RED`. That
  outcome is deliberate and releases landing; it is not release proof.
- Full CI **38604579828**, SHA `d070e6b2dc17d1229f6d13b2a4d0ab1dd25b0f5d`, passed all
  executed nonvisual jobs and all **1366 visual tests**, but reported `check/drift` with
  `created=0 updated=5 unchanged=1095 stale=0`. Its executed
  `SASE_VISUAL_REPORT_STATUS=drift` markers prove the watcher correctly stopped on this
  run. This is remote release acceptance, not a local check-full-only failure.
- The complete CI HTML report, including expected/candidate images and candidate SVGs,
  is **file:explicit:280388d2d1c4308e7a8bbe18**. Read it with `sase artifact read`; use
  `sase artifact path` when a parser/viewer needs its path. GitHub source is run
  38604579828, artifact `visual-failure-report.html`, artifact ID 11915824409. The
  artifact endpoint returned the HTML bytes directly during audit, so inspect content
  before assuming the response is a ZIP.
- Master Gate **38610666059** is green on `4d8940d4b7e1`. Full CI **38616988827** on
  that SHA was still in progress at audit; inspect its final result before editing.
- PyPI `sase-core-rs` **0.40.0** is published. Publish **38599345547** completed the
  floor ratchet; its build/install/publish jobs were **skipped**, so it did not ship
  sase. PR **#299** remains open at `c5e39b542b58a9f109fe7fa57e8fd61f35891f0f`, its
  `release-core-floor-smoke` is green, PyPI sase remains **0.17.1**, and GitHub release
  `v0.18.0` is absent. Do not dispatch another core release for the already satisfied
  0.40.0 floor.

Concurrent integration was reviewed through that HEAD. `613d4b2acb` / `8671366a27`
updated docs, `de27bceedf` split ACE testing tests, and `d070e6b2dc` finished explicit
instruction delivery. `79c1b5d958` split watcher tests into
`test_release_watch_parse.py`, `test_release_watch_decisions.py`,
`test_release_watch_lifecycle.py`, and `_release_watch_helpers.py`: all 32 test
functions survive; only three changed helper references from `_run_payload` to
`run_payload`. The integrated **46 parametrized watcher tests pass**. `4d8940d4b7` adds
Argon model display and a ToolRun parent-read timeout fix; it does not change these
fixture models or watcher behavior. Put new watcher coverage in the split modules. Fetch
and inspect newer drift before changing anything.

## Rules

- Read `lint_and_test.md`, `symvision.md`, `tui.md`, `tui_screenshot.md`, and
  `tui_perf.md` with `/sase_memory_read`. Open other repositories through `/sase_repo`.
- Preserve assertions, existing timeouts, pixel equality, release guards, and all
  original watcher verdict/close contracts. Do not add assertion retries, skip/xfail,
  fixed sleeps, broad rebaselines, or copy CI candidates into goldens.
- Fix fixtures and semantic waits. Change production code only for a demonstrated
  product defect; shared domain changes belong in Rust core. Keep the operational
  watcher stdlib-only and independent of the sase installation.
- Verification is `just fix` followed by `sase tool run check` (wrapped `just check`).
  **Do not run `just check-full`.** A check-full-only infrastructure failure is a
  follow-up, not remaining release implementation work.
- Use `/sase_monitor` for long checks and waits; wait for the start command itself to
  exit. Do not leave provider-owned commands running at turn end. Phase workers use
  `PROPOSED FOLLOW-UP:` notes for new unrelated findings, never create task beads.
- No manual release-PR merge, version edits, global installation, or changes to the
  user's primary checkout. Do not kill the waiting sase-1jz agents.

## Phase deterministic-captures

### Repair the observed states

The audit inspected all five expected/candidate image pairs, then their fixture and
production paths. Reproduce the following concrete states, establish the intended
settled state, and pin it before capture:

1. **`agents_session_panel_turns_gate_90x40`**, 14707 changed pixels, in
   `test_ace_png_snapshots_agents_agent_session_panel_gate.py`: expected has the
   `(/) blocks` action; candidate lacks it and the whole footer reflows. The current
   `cleanup` sentinel plus `stabilize_toast_frame()` only settles stopwatch/status
   geometry, not block navigation readiness.
2. **`agents_weighted_runner_capacity_expanded_80x32`**, 16318 pixels, in
   `test_ace_png_snapshots_agents.py::test_weighted_runner_capacity_png_snapshots`: the
   same action disappears after expansion. The narrow branch only waits for visual idle.
   Trace both cases through
   `_app_action_availability_agents._focused_card_blocks_navigable`,
   `AgentDetail.deck_area.focused_panel()`, and
   `widgets/decks/panel_blocks.py::card_blocks_navigable`. Pin the actual selected
   card/document and its final predicate; do not simply force the footer visible.
3. **`agents_archive_list_120x40`**, 1919 pixels, in
   `test_ace_png_snapshots_agents_archive_list.py`: expected contains a container
   summary with **0 matching members**; candidate has **2**, `clan.alpha` and
   `clan.beta`. The left list already shows both in each image. The test expands the
   container, waits for `clan.alpha` anywhere, then captures without waiting for the
   detail summary. Trace `_archive_reader._paint_container_summary` and
   `archive_container_members`; wait for the selected container's fully loaded detail,
   including both members. The existing `settle_archive_detail()` helper uses a
   one-second sleep; use a semantic predicate instead of copying that wait.
4. **`agents_session_lane_neighbors_160x50`**, 3570 pixels, in
   `test_ace_png_snapshots_agents_neighbors.py::test_agents_session_lane_neighbors_png_snapshot`:
   expected has Reply active (`2/2`) and a tall overlay scrollbar; candidate has Context
   active (`1/2`) and a small thumb, while reply body text is otherwise present. The
   test re-resolves the container after expansion, then waits only for a jump-panel
   sentinel. Establish the intended active MAIN card and scroll position after selection
   refresh; retain the jump-map assertions.
5. **`artifacts_split_selected_wide_120x40`**, 1903 pixels, in
   `test_ace_png_snapshots_artifacts_split.py::test_artifacts_split_mode_selected_detail_png_snapshot`
   (select the `wide` case; derive the parametrized node with collection): the detail
   scrollbar is tall in expected and a small thumb in candidate. Body content is
   unchanged. Pin final detail layout/scroll state. If the overlay is intentionally
   excluded from this layout fixture, apply the existing explicit test-only
   overlay-hiding convention after proving that diagnosis, preserving scrolling coverage
   elsewhere.

Regenerate only after the correct stable state is established and inspect each image.
Run targeted checks in the default zone and UTC, repeat the repaired nodes to expose
late reflow, and then run full **`TZ=UTC just fix-tui-screenshots --check`** through a
monitor. Require no failed tests and `clean`, `created=0 updated=0 stale=0`; `partial`
is not proof. Inspect any newly recurring mismatch rather than silently accepting it.
The parent already authorizes completing the whole visual lane; these five are the
current inventory, not permission for unrelated product changes.

### Finish watcher parsing and agent-environment integration

The existing startup-signal and blank-terminal-conclusion repairs are correct and
covered. Do not redo them. Independent probes of current code found these remaining
contract gaps in `tools/_release_watch_api.py`:

- `_is_visual_golden_drift("echo SASE_VISUAL_REPORT_STATUS=drift")` and the quoted
  `printf 'SASE_VISUAL_REPORT_STATUS=drift\\n'` source both return true. Parse a
  complete executed marker line, accounting for actual `gh` job/timestamp prefixes and
  ANSI output. Reject echoed commands, headings, quoted source, and trailing junk. The
  current CI run's true executed markers must remain true. When a job-specific log is
  unavailable, do not attribute another job's marker from the combined failed log to
  visual-test; preserve job identity or conservatively leave deterministic drift
  unproved. Add focused tests for true prefixed output, source-only text, test-failure
  status, and mixed-job fallback.
- `core_version_meets("0.41.0rc1", "0.40.0")` and even
  `core_version_meets("0.41.0garbage", "0.40.0")` return true because the greater
  numeric branch bypasses stability/validity. Require a valid stable published version
  before ordering. Test older/equal/newer stable versions and prerelease, dev, and
  malformed suffixes on **higher** numeric versions as well as equal ones. Preserve any
  deliberately supported valid post/local release forms; do not add a packaging
  dependency to the stdlib watcher.
- Phase `.5.2` preflight failed under the normal agent's `CLICOLOR_FORCE=1`; its caller
  had to export `NO_COLOR=1 GH_FORCE_TTY=0`. `GhClient._call` currently inherits that
  environment except for `GH_REPO`. Normalize machine-readable gh output at this
  boundary, with regression coverage proving inherited forced-color settings cannot
  corrupt JSON. Preserve authentication, repo routing, and timeouts. Keep the external
  uncolored wrapper in release commands as defense in depth.

Run the split watcher tests and required `sase tool run check`. Record exact evidence
and any residual triage. Resolve this phase's epic-symbol entries, close the phase
normally when its implementation and checks are complete, and use host finalization to
commit. Do not wait for this phase's own future SHA or CI.

## Phase publication-proof

This phase makes **no file changes**. It observes the coding phase's landed commit, and
always closes itself so landing is released on every outcome.

1. Read predecessor notes. Fetch and fast-forward a clean checkout, prove
   `origin/master` contains the repair commit and `960f1e1b67`, and set FIXED_SHA to the
   **new repair commit**, not the old failure's ancestor. Otherwise historical
   deterministic reds will immediately terminate the watch. Check any newer pin and core
   publication using `/sase_repo` before reading core files. Core 0.40.0 already
   satisfies the audited pin; only repeat the authorized urgent release flow if a newer
   pin genuinely requires it, and never duplicate an equivalent running cut.
2. Inspect current Full CI and #299. Do not duplicate an in-flight run. Check own
   `sase bead epic-symbols <phase-id>` before giving the watcher close authority.
3. Preflight inline with one fully parsed line and exit 0:

   ```sh
   NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version 0.40.0 --once --dry-run --close-bead <phase-id>
   ```

   Adapt the core minimum only if the pin required a newer release. Preflight failure
   closes this phase canceled with reason
   `release not confirmed: WATCHER_BROKEN (preflight)` and the output; then finalize. Do
   not repair code in this phase.

4. Use `/sase_monitor` to start one watcher, with timeout 13h, statuses
   `WATCHING RELEASE` / `RELEASE WATCHED`, and the shell command:

   ```sh
   NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 5m 760m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version 0.40.0 --deadline 12h --close-bead <phase-id>
   ```

   The watcher handles deduplicated Full CI/Publish dispatch; ci_watch owns guarded
   merge. The monitor follow-up reads the verdict and phase, repairs a failed mechanical
   close only (done for RELEASED; canceled with precise reason otherwise), records new
   blockers as notes, then finalizes successfully. It must not reopen the phase, edit
   code, or restart the watch. If monitor start fails, retry once; after two failures
   close canceled with WATCHER_NOT_STARTED and the errors. Poll any yielded start
   session until it exits.

## Landing and ancestor resumption, not implementation phases

Do not close this child epic or any ancestor merely because both phases closed. Verify
the new code and all notes, current-base integration, full visual evidence, fresh green
Full CI and Master Gate, #299's merge, GitHub `v0.18.0`, actual successful Publish
**build and publish jobs**, and PyPI 0.18.0. Separately require a fresh temporary
environment outside the repository with **unpinned `pip install sase`** resolving
0.18.0, working `sase --version`, and the resolved core recorded. The watcher proves a
pinned install; it does not replace that unpinned acceptance check.

If only external waiting remains, hand it to `/sase_monitor` without `--close-bead` on
an epic. If implementation remains, follow the tier-aware remaining-work flow. Never
force a successful nested landing or declare a canceled wait successful.

This child epic's `parent_bead: sase-1kx.5` hands back to its lander after the child
lands. That lander must recheck `sase-1kx.5` descendants, notes, linked plan readiness,
and post-child drift, retire or re-key every `sase bead epic-symbols sase-1kx.5` entry,
then close normally with an evidence-rich note, run `just symvision`, and mark
`plan:202610/v0_18_0_release_finish.md` frontmatter `status: done` using the sanctioned
plans checkout. These are landing actions, never a child implementation phase.

Then resume the directly parented plan **sase-1kx**. Read every descendant and note, the
prior landing audit, and `plan:202610/v0_18_0_release.md`; recheck post-child drift and
all descendant/linked-plan readiness. Its outstanding closeout includes evidence-based
resolution of `sase-16p`, `sase-1gh`, `sase-19d`, `sase-1kg`, and `sase-1ii`; preserve
`stale_epic_1jz=notes` (verify before closing .3, then note .4/.5 with green runs) and
report whether ci_watch's heavy dispatcher is live. Do not deploy to the primary
checkout. Preserve the parent landing note's historical follow-up dispositions. Only if
all requirements hold, retire `sase bead epic-symbols sase-1kx` entries, close normally,
run `just symvision`, and mark `plan:202610/v0_18_0_release.md` done. Recheck its parent
link and continue only through fully ready plan ancestors; stop and note the first
incomplete or ambiguous parent.

## Follow-up disposition already completed by the land audit

The only new proposal was **sase-1kx.5.2 note #3**, “Full CI visual-test red blocked
v0.18.0.” Its five concrete drifts remain required release acceptance work, including
the recurrence of the exact gate-footer drift this epic promised to repair. No new
omnibus task was created. `/sase_new_task` triage searched task statuses, reviewed
`sase-x5` and `sase-10u`, swept last-week tasks, and inspected active epics and
`sase-1jz.3`. The old static title/keymap rebaselines are not evidence of these live
capture-state defects, so they were not corroborated. A `DISCOVERED ISSUE` with the CI
images and concrete causal gate/deck overlap was appended to **sase-1jz**. The archive
summary and scrollbar observations are retained in the existing full-lane acceptance
scope, not filed as speculative separate product defects. The watcher gaps are
unfinished epic implementation. Copy these dispositions into eventual landing close
notes; triage genuinely new unrelated findings independently.
