---
tier: epic
title: Repair question timezone conversions and finish the 0.18.1 publication watch
goal:
  Question timestamp fallbacks and their tests honor the configured timezone even when
  the system differs, a complete published core contains the current source pin, and
  sase 0.18.1 is published with retained release and clean-install evidence.
parent_bead: sase-1nn.8.4.3.5
phases:
  - id: timezone
    title: Correct question timestamp fallbacks and fixtures under timezone divergence
    size: medium
    depends_on: []
    description:
      "timezone: normalize question model timestamps through the existing
      configured-time adapter, repair fixtures, and verify divergent clocks."
  - id: publish-watch
    title:
      Correct the release watcher and observe the committed repair through publication
    size: medium
    depends_on:
      - timezone
    description:
      "publish-watch: correct cut eligibility and terminal closure, prove the
      transitions, and monitor publication with bounded phase closure."
proposed_by: bbugyi200.athena.sase-1nn.8.4.3.5.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-25 03:37:11
status: wip
---

- **PROMPT:**
  [prompts/202610/v0_18_1_timezone_and_watch.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/v0_18_1_timezone_and_watch.md)
- **PARENT:**
  [202610/v0_18_1_release_integration.md](https://github.com/sase-org/sase--plans/blob/main/202610/v0_18_1_release_integration.md)

# Finish the remaining release work from sase-1nn.8.4.3.5

The target is **sase 0.18.1**. The parent is incomplete: its watch phase
**sase-1nn.8.4.3.5.3** closed canceled with CORE_LAG. Do not interpret a closed watch as
successful publication or mark any ancestor plan done yet.

Two phases are necessary because the publication must contain the timezone repair's
host-created commit and pass remote CI. A tale cannot observe its own end-of-turn
commit. The first phase makes the bounded timestamp repair; the second operates the
watch after that commit. This child epic's lander owns release evidence and parent
resumption. No phase closes this epic, its parent, or any ancestor.

## Verified starting evidence

Landing audit on 2026-10-25 UTC at master `369ac34168e6ee3787c40f42dc015422141b2286`:

- Read every parent phase and every note, its implementation commit, and subsequent
  master commits. `3d13864e76` deletes only the 99 archived `.sase-evidence/` paths. All
  99 blobs match their members in `file:explicit:ddc1ff91f98de4e168df0d63` byte for
  byte; MANIFEST.txt exists. Archive SHA256 is
  `1c82bbdbda694988d4feb824c74c25fa651fdbf7600cf74a9b6f228aaeeda14e`. No removed
  evidence remains tracked or referenced by live source/tests/tools. Preserve this
  result; do not repeat the evidence removal or weaken terminology checks.
- Post-integration commits `2066212dfa`, `113d799b2c`, and `369ac34168` add the
  QUESTIONS Context section, its docs, and legitimate consumers/private symbols. The
  latest commit removes all obsolete question epic-symbol rows. Current Master Gate lint
  passes. Older Full CI lint fails on the old rows; that is already repaired.
- Master Gate **40163651103**, attempt 2, at current master fails only test shard 3, job
  **121082333087**, on
  `tests/ace/tui/widgets/test_agent_questions_render.py::test_expanded_pending_round_lists_options_with_recommended`.
  Expected `asked 14:02` / `23h left`; actual `asked 09:02` / `18h left`. Full CI
  **40163212161**, attempt 1, at `113d799b2c` has the same test failure on Python
  3.12/3.13/3.14 (jobs **121090355932**, **121090355898**, **121090356017**). Its visual
  job **121090355889** passes. Full CI **40159378002**, attempt 2, was still running;
  refresh all run state before editing or acting.
- The exact failing test reproduces on the unchanged master tree with `TZ=UTC` and
  passes with `TZ=America/New_York`. `_unix()` calls naive `datetime.timestamp()` using
  the system clock; `_as_local()` correctly uses `get_timezone()`, and
  `tests/_conftest_runtime.py::_pin_configured_timezone` fixes configured time to
  America/New_York. The failed assertion is a fixture error; the renderer's
  configured-time conversion is correct.
- Source review of `518717cdac` also found the same naive `.timestamp()` mistake in
  production model-to-epoch fallbacks: `agent_question_rounds.py::_asked_at_unix`,
  `_agent_display_header_summary.py::_resolve_question_rounds`, and
  `_agent_gate_section.py::_question_numbered_round`. Agent suffix parsers return naive
  configured-local datetimes. With system UTC and configured New_York, an actual
  `question_round_stubs()` call for a naive 2026-01-15 14:02 row gives epoch 1768485720
  instead of 1768503720, an error of -18000 seconds. These newer question features must
  use the existing configured-time adapter consistently.
- PyPI `sase/0.18.1` returns 404. PR **321** is OPEN, target 0.18.1, head
  `2228be2b3dac20ed2e7a6a6bbf2cb3c8bd492e9d`. Publish **40162818486** failed metadata
  sync in job **121077964207**: no complete core contains the pin.
- Source pin `3156d11f33d5a5277ff806a78b92a9490f78d6be` is after published core v0.42.5.
  PyPI 0.42.5 has the five required distributions, all unyanked. Core PR **337**,
  v0.42.6, head `e6779f6c73a57774cb8da991b8718f9fe5e63cdd`, contains the pin (local
  ancestry proof), is OPEN/MERGEABLE/CLEAN, and has green/skipped checks. A PR is not an
  active cut. Core workflow `release-plz.yml` merges only on daily **07:41 UTC** or
  `workflow_dispatch dry_run=false`; the six-hourly heal does not merge. Successful
  scheduled run **40163770188** at 04:57 UTC skipped Merge release PR and all
  distribution builds/publishing. The next daily cut was imminent at audit time, so
  refresh rather than issuing a redundant dispatch.
- Parent core-cut phase .2 deferred merely because an eligible PR existed, and the
  watcher only dispatches when `eligible` is empty. That reverses the intended guard and
  caused it to sit on a green PR until CORE_LAG. It also calls any containing open PR
  eligible without checking title, mergeability, or checks.
- Audit-read watcher `file:explicit:b73638746f44786733a4be74` and wrapper
  `file:explicit:fa83f28666b71af2e27fd64f`. The wrapper searches compact bead output for
  `[OPEN]`, missing an in-progress phase. The watcher writes `closed_by_watcher` even
  when all close attempts failed. These are remaining operational defects. Other audit
  findings: a file count is not proof of the expected wheel set; fetch failures continue
  with stale refs; deadline checks can be skipped by unknown-state `continue` paths;
  restart resets the absolute deadline.

Retained audit/CI excerpts: `file:explicit:133becdc381754837787779d`. Retained local
timezone fail/pass and singleton serial pass: `file:explicit:c855286573bb6fa4ceb034f1`.

## Follow-ups already triaged by the parent lander

The parent phase **sase-1nn.8.4.3.5.1** has exactly two proposals:

- Note 2: created **sase-1od**, large flake, ready, for
  `test_epic_plan_modal_renders_no_raw_editor_for_capacity` missing `#gate-singleton-0`
  during mount. Correct old CI job ID is **121024209959** (the proposal's 121044209959
  is a typo). One observed CI failure; phase reports isolated and 17-test file passes,
  and the lander also gets serial passes. The cause remains unknown; a Textual
  mount/teardown race is only a hypothesis, with no evidence of process-global leakage.
  No duplicate or causally owning active epic was found. It is absent from the current
  Full CI failure summaries, so keep it out of the timestamp fix unless a fresh
  inventory proves recurrence.
- Note 3: corroborated **sase-1ic**, the existing startup-budget task, with phase check
  **3655c5cefc27e058d22059a60bed98c7**, 3538 versus strict 3537, KNOWN witness
  **0807aefecc59861aee177979c8a6e2be**, and the phase's clean-base proof. Declined a
  duplicate task. This is relayed phase evidence, not a new local import-budget
  measurement. The removal commit adds no import edge.

Earlier ancestor proposals already have dispositions in **sase-1nn.8.4.3 note 1**
(sase-1o6 through sase-1oa, sase-1jz, sase-1f9). Do not duplicate them.

## Rules

1. Re-read your phase, this child epic, the parent **sase-1nn.8.4.3.5**, and its newest
   landing note. Fetch/re-inventory before edits or operational actions. Use
   `/sase_repo` before all other checkouts and `sase artifact read` for plans and
   retained evidence. Discover repo paths dynamically; never reuse the old watcher's
   hardcoded paths.
2. Read `lint_and_test.md` and `sase_beads.md` through `/sase_memory_read`; read
   `ace.md` and `tui.md` before ACE/TUI changes, `symvision.md` for symbol cleanup, and
   `tools.md` before changing tools. Shared domain behavior belongs in Rust; this work
   should need no domain or binding change.
3. File-change verification is `just fix` then `sase tool run check` (guarded
   `just check`). **Do not run `just check-full`.** A scoped pass with a full-check
   failure is an infrastructure follow-up, not remaining product work. Remote release
   policy still governs publication. Keep assertions, pixel equality, and deadlines; no
   tolerance changes, skips, arbitrary sleeps, or assertion retries to make CI green.
4. The inherited decisions remain final: `gate_rerun=yes`, `full_ci_rerun=yes`,
   `publish_dispatch=yes`. `ci_watch` owns sase merges; `publish.yml` publishes.
   **sase-1n6.9** is still in progress; the release steward remains observe-only. No
   hand commits, pushes, merges, new PRs, release.yml dispatches, live steward, ci_watch
   configuration changes, manual release versions/changelogs, or global installs.
5. Use `/sase_monitor` for long checks and operational waits, with a concrete
   continuation or self-closing phase as specified below. Wait for the handoff command
   itself to exit. Every phase reports its actual result. Canceled is not release
   success. Phase workers append PROPOSED FOLLOW-UP notes instead of creating tasks.

## Phase timezone

1. Reproduce the exact node with `TZ=UTC` and `TZ=America/New_York`. Confirm the current
   configured-time fixture and renderer still match the diagnosis above.
2. Make `_unix()` interpret naive fixture values as configured-timezone wall time before
   converting them to epochs (for example attach `get_timezone()` explicitly). Preserve
   the renderer's configured timezone semantics and the explicit expected clock/deadline
   strings. Audit other timestamp fixtures in this presenter test file and directly
   related question-section tests for the same helper mistake.
3. Correct all three production fallback sites named above. Reuse the existing
   `sase.core.time.parse_local` configured-time adapter (or an established equivalent)
   to interpret naive model values as configured-local while preserving aware instants.
   Compute timeout deadlines from the normalized epoch plus duration. This is thin
   model/presentation glue; do not invent a second timezone policy or new backend. Cover
   the stub, resolved lane, and gate Reply paths with real naive model timestamps under
   divergent clocks, and ensure aware timestamps retain their instant. Preserve use-site
   imports and the existing missing-time/missing-bundle behavior.
4. Use the existing `tz_divergence` fixture to cover system UTC versus configured
   Eastern in the affected test; keep its automatic restoration of the system clock
   state. Prove the whole presenter test file passes in both process timezones and run
   the directly affected model/lane/gate/section/projection tests. Run targeted question
   screenshot verification because fallback rendering can change, following the
   screenshot memory and inspecting any changed PNG. Do not broadly rebaseline.
5. Re-inventory the latest exact-head Master Gate and latest five completed Full CI /
   Master Gate runs. Confirm old Symvision rows are already removed and do not spend
   time rerunning stale source snapshots. If a new deterministic or recurring blocker is
   actually present, diagnose its scope; keep unrelated one-off flake tasks separate.
6. Run `just fix` and `sase tool run check`, retain outcomes (including any
   independently witnessed KNOWN import-budget issue), resolve your phase's epic-symbol
   entries, and close normally with verification. `/sase_final` makes the commit. Do not
   wait for that future commit in this phase. If the repair cannot be completed, record
   the exact blocker and close canceled so the lander is not stranded.

## Phase publish-watch

This phase changes no tracked source. Its dependent phase must have closed done and its
host commit must be present on fetched master before publication observation.

1. Re-inventory PyPI, PR321, current source pin, core tags, recent cut runs and PR337
   (or its successor), exact-head Master Gate, latest Full CI, and Publish. Read
   **sase-1n6.9** and **sase-1n6.5.1.5.2.1.3.1**, whose agent may already be waiting on
   the daily core cut and advancing master's exact dependency floor. Reuse its result
   rather than competing. If already published, record version/files and close done; the
   lander still performs full release/clean-install verification. Failed predecessor,
   changed release target, or changed merge authority must end canceled with evidence
   for the lander.
2. Build a corrected one-off watcher and wrapper in `/tmp/sase-release-watch-<phase-id>`
   (not agent-managed temporary storage). Audit-read the two old script artifacts, but
   replace the diagnosed logic rather than executing them unchanged. Obtain checkouts
   through `/sase_repo` and pass their returned paths at runtime.
3. Implement these core decisions with fresh observations:
   - A containing published core requires all five expected unyanked suffixes from the
     opened core workflow: manylinux_2_28_x86_64, manylinux_2_28_aarch64, universal2,
     win_amd64 wheels and sdist. Prove tag ancestry locally; distinguish merge-base exit
     1 from an operational error. Do not equate a count of five or an HTTP 200 with
     completeness. Prefer the newest complete containing version.
   - Only a real cut (daily 07:41 run or non-dry workflow dispatch whose merge job can
     run), or its resulting build/publish, counts as an in-flight cut. An open PR or a
     successful push/heal that skipped merge does not. Inspect job facts where event
     identity alone is ambiguous. Never duplicate an active cut or one scheduled within
     90 minutes; after the scheduled time, re-observe rather than postponing forever on
     an assumed run.
   - After 30 minutes of confirmed core lag, an eligible release-plz PR is a positive
     dispatch prerequisite: release title, open, mergeable, checks success/skipped, and
     head containing the pin. If no cut is active or imminent, dispatch at most once per
     pin using
     `gh workflow run release-plz.yml -R sase-org/sase-core --ref master -f dry_run=false`.
     Record the resulting run ID; let the workflow merge. Missing/red/unrelated PRs are
     blockers, never reasons to dispatch blindly. Preserve state across restarts.
4. Preserve the approved release policy and bounded observation:
   - Every five minutes observe the current head/pin, core, PR/checks, Full CI, and
     Publish. Reset core-lag duration on pin change and successful containment; reset
     Master Gate red duration on head/run change or success. Keep a persisted absolute
     12-hour overall deadline checked even on unknown/error paths, a 4-hour
     confirmed-lag terminal outcome per stable pin, and a 3-hour terminal Master Gate
     red duration after the allowed rerun.
   - Fetch/API/PyPI transport failures are unknown. Do not mutate using stale refs or
     infer a release, missing release, or failed gate from read errors. Bound each
     subprocess so the outer deadline remains enforceable. Preflight must terminate
     after one observation, without remote mutations or bead closure, even on errors.
   - Persist Master Gate and Full CI rerun IDs. At most one failed-job rerun per run;
     recognize already-rerun attempts from GitHub as well as local state. Count only
     observed terminal final attempts. An API error or ambiguous dispatch result is not
     an invitation to retry the action. The two newest final red Full CI runs yield
     FULL_CI_RED; a newer in-flight run defers that conclusion.
   - ci_watch requires exact-head green Master Gate, newest completed Full CI green and
     at most six hours old, and every PR check success/skipped. It triggers stale Full
     CI itself; do not create another merge driver. Once core is ready, refresh release
     metadata through `publish.yml -f publish_existing=false` only when needed, no
     Publish is in flight, at most twice, at least an hour apart. After merge permit one
     dispatch if no Publish has started since the merge. Preserve all budgets on restart
     and account for already-issued actions.
   - Only actual PyPI sase 0.18.1 distributions produce RELEASED. Metadata-only Publish
     success does not. Post-merge Publish failure yields PUBLISH_RED.
5. Make terminal closure robust and testable:
   - Use audited `sase bead read <phase> -N -f json` when checking status; handle
     in_progress/claimed/open and already-closed states explicitly. Do not parse compact
     display glyphs or search for `[OPEN]`.
   - Watcher completion, INT/TERM, unexpected exit, and outer timeout all attempt
     bounded idempotent closure via the CLI. Close done only with actual PyPI release
     evidence; otherwise canceled with outcome, run IDs and log artifact. A completion
     marker is written only after closure is confirmed. Failed close attempts leave the
     fallback armed. Never mask exhausted closure attempts as successful closure; retain
     a visible failure and recovery instruction.
   - Preserve nested timing: watcher deadline 12h, outer `timeout -k 5m 750m`, monitor
     timeout 13h, leaving room for bounded close retries and fallback. The wrapper must
     also close an in-progress phase after a crash before a marker exists.
     Reboot/SIGKILL cannot be guaranteed to run traps: document recovery with
     `sase bead work <this-child-epic>` rather than promising automatic recovery.
6. Before launch, exercise the state transitions with stubbed command outputs: green
   eligible PR/no active cut, daily cut imminent, heal-only success, red PR, pin change,
   missing/wrong wheel suffixes, unknown fetch/API response, restarted deadline/rerun
   budget, successful close, all close retries failing, in-progress wrapper fallback,
   and timeout/signal. Stub mutations; these checks must never merge, dispatch, or close
   real beads. Retain script and proof artifacts. Syntax checks alone are insufficient.
   Run a read-only live preflight with known fields.
7. Run your phase's epic-symbol guard, append preflight evidence and exact command, and
   use `/sase_monitor` to start one zero-weight bounded watch with no model follow-up.
   Read that skill for the installed CLI; wait for monitor start to exit. The watcher
   closes this phase and the epic's lander resumes mechanically. On preflight/start
   failure, close canceled with evidence and use `/sase_final`.

## Lander closeout and parent continuation

This is a lander responsibility, not a phase. Audit both children and notes, commits,
linked plans, and subsequent master drift. A canceled watch requires further work.
Retain an artifact with PyPI 0.18.1 filenames, SHA256 hashes, upload times and exact
core dependency; GitHub v0.18.1 tag/release commit; PR321 merge commit/time/actor;
qualifying Master Gate and Full CI run/SHA/attempt; Publish job results; exact core
floor, its five distributions and ancestry; and available watcher/ci_watch evidence. Do
not infer ci_watch authorship solely from the GitHub merge account.

Perform a fresh Python 3.12 installation of `sase==0.18.1` from PyPI with
`UV_NO_CACHE=1` in an isolated temporary directory outside the repo, isolated HOME
passed only to the child process, the matching exact core dependency, and successful
`sase core health --json`. Do not change shell HOME or the global installation.

Resolve/re-key all `sase bead epic-symbols <this-child-epic>` entries to actual
consumers or legitimately still-open later beads. Close this child normally with
verification, run `just symvision`, and set its linked plan `status: done` through the
plans checkout opened by `/sase_repo`.

Then resume parent **sase-1nn.8.4.3.5**: re-read its latest landing note, every
descendant/note and `plan:202610/v0_18_1_release_integration.md`, and recheck drift and
linked-plan/descendant readiness. Record that the missed eligible-core dispatch was
fulfilled by the observed actual cut; preserve original phase .2 notes. Reopen its
canceled watch **sase-1nn.8.4.3.5.3** and close it normally with the real release
evidence if readiness requires a done phase. Historical cancellation notes remain.
Resolve/re-key every `sase bead epic-symbols sase-1nn.8.4.3.5` entry, then:

`sase bead close sase-1nn.8.4.3.5 --note "<verification, follow-up dispositions, release evidence ref>"`

Run `just symvision` and set `status: done` in
`plan:202610/v0_18_1_release_integration.md` through the opened plans repo. Repeat full
readiness checks through these directly parented plan ancestors:

| Parent         | Linked plan                            | Canceled watch to re-evaluate |
| -------------- | -------------------------------------- | ----------------------------- |
| sase-1nn.8.4.3 | plan:202610/v0_18_1_release_unblock.md | sase-1nn.8.4.3.4              |
| sase-1nn.8.4   | plan:202610/v0_18_1_final_goldens.md   | sase-1nn.8.4.2                |
| sase-1nn.8     | plan:202610/v0_18_1_release_landing.md | sase-1nn.8.3                  |
| sase-1nn       | plan:202610/v0_18_1_release.md         | sase-1nn.7                    |

For each, read all descendants/notes, its earlier landing note and linked plan; recheck
drift/readiness, correct canceled watch resolution via reopen and normal close when
actual completion is proven, retire epic-symbol entries, close normally, run
`just symvision`, and mark that plan done. Never force merely to make close succeed or
to advance successful nested landing. Stop at the first incomplete or ambiguous
ancestor, note the blocker there, and report it. If a parent link has become a phase
instead, verify and close only that phase, leaving its containing epic to its waiting
lander.
