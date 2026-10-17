---
tier: epic
title:
  Green master's release gates and ship sase v0.18.0 through a watch that always
  releases its lander
goal: "Master Gate and Full CI are green on master, ci_watch merges release-please PR
  #299, and sase 0.18.0 installs from PyPI. The release-watch phase's bead and agent
  session both settle without any LLM follow-up turn, so the land agent always runs,
  confirms the release before landing, and then resumes the stalled sase-1lh landing
  chain.

  "
parent_bead: sase-1lh.7.5.4.3
phases:
  - id: clock
    title: Fix the aged-out auto-restart fixture stamps (sase-1ma)
    depends_on: []
    size: small
    description:
      "clock: stop the hung previous watch turn and its parked lander, derive the
      sweep-safety fixture row stamps from the test clock, and sweep for other aged-out
      calendar fixtures."
  - id: ci-reds
    title: Fix the intermittent tutorial gate and any other current gate red
    depends_on: []
    size: medium
    description:
      'ci-reds: root-cause and fix the intermittent "no agents appeared after sase run"
      tutorial failure, make that failure self-diagnosing, and fix any other
      deterministic Master Gate or Full CI red except the clock phase''s test.'
  - id: watch
    title: Watch v0.18.0 to PyPI with a self-closing, follow-up-free monitor
    depends_on:
      - clock
      - ci-reds
    size: small
    model: claude/sonnet
    description:
      "watch: make no edits; preflight tools/release_watch, then start one monitor with
      no follow-up whose watcher closes this bead and whose command always exits 0."
proposed_by: bbugyi200.athena.113
proposed_with: claude/opus
create_time: 2026-10-17 14:27:18
status: wip
---

- **PROMPT:**
  [prompts/202610/green_gates_ship_0_18_0.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/green_gates_ship_0_18_0.md)

# Green master's release gates and ship sase v0.18.0

This epic is a child of **sase-1lh.7.5.4.3**, the deepest open epic in the v0.18.0
release chain: sase-1lh → .7 → .7.5 → .7.5.4 → .7.5.4.3. Earlier attempts (sase-1io.7.6,
sase-1kx, and that chain) fixed the watcher, the visual lane, and the parseable
`Release-As: 0.18.0` declaration. What remains is small, but the last attempt also
exposed a structural flaw in how the release-wait phase ends. This plan fixes both.

The inherited accepted decisions stay in force: **dispatch = yes** (the watcher may
dispatch Full CI, Publish, and one Master Gate rerun per SHA when due), **visual_flakes
= recapture**, and **stale_epics = supersede**. This plan proposes no memory changes. It
is auto-approved, so the planner made two calls itself. The clock phase stops the hung
previous watch turn and its parked lander. The recurring flakes sase-1ht and sase-1lr
are not edited here, because the watcher already tolerates one red Full CI run and
reruns a red Master Gate once per SHA.

## Evidence at planning time (2026-10-17 ~18:15Z, master a4465e18db)

- **PR #299** is OPEN, titled `chore(master): release 0.18.0`, with head 1a6dee4de2 and
  merge state CLEAN. Its `release-core-floor-smoke` passes. The 0.18.0 target is
  restored, so do not touch the release declaration again.
- **PyPI** serves sase 0.17.1 (0.18.0 absent) and sase-core-rs 0.41.0. CI builds the
  pinned core wheel 0.40.1 from `sase-core-revision.txt`. Task sase-16p records that
  0.17.1 is uninstallable today, which 0.18.0 should cure.
- **ci_watch** (chezmoi `home/dot_config/sase/sase_athena.yml`, job `ci_watch`, every 5
  min) merges the release PR only when Master Gate is green at master HEAD, a green Full
  CI is at most 6 h old, and the PR's checks are green. It dispatches a stale Full CI
  every 2 h, and Full CI also runs on a 2-hourly cron.
- **Master Gate** has been red on every master push since 69e0cd7540. Each of the last
  six runs (39021309094 … 39047428622) fails exactly one node, the date bomb below. Run
  39037235499 also hit the existing flake sase-1lr.
- **Full CI** has been red on every run. Each `test (3.12/3.13/3.14)` leg fails only the
  date bomb, and run 39020878025 also hit the existing 3.14 flake sase-1ht. Visual,
  perf, lint, and isolation now pass. `docs-build` and `release-core-floor-smoke` run
  only on PRs, so they are expected to be skipped here.
- **The date bomb (sase-1ma):**
  `tests/test_agent_auto_restart_sweep_safety.py::test_disabled_path_resurfaces_silenced_once_and_ignores_others`
  asserts `[1, 0, 0]` and gets `[0, 0, 0]`. Its fixture row is stamped
  `20261009T120000`, and healer enumeration
  (`src/sase/agent/auto_restart/healer_targets.py`,
  `_HISTORY_LOOKBACK_SECONDS = 7 * 86400`) stopped seeing it on 2026-10-16. Other tests
  in the same module also use `20261009T*` rows. The ones that assert "ignored" or
  "idle" now pass vacuously, because the row is out of window rather than for the reason
  the test names.
- **The intermittent tutorial gate:** Full CI's `tutorial-tests` job fails
  `tests/test_tutorials.py::test_start_tutorials_pass_on_fakey` at
  `docs/start/first-change.md` step 1. The step runs
  `sase run "+home create welcome.txt containing exactly this line: My first SASE change. Do not edit other files."`
  and fails with "no agents appeared after `sase run` within 60s" (Python 3.12, `CI`
  set, prebuilt core wheel). It failed in runs 39009012994 and 39011976162 (both at
  315092afeb) and 39046170656 (at a80f2891e8). It passed in the other runs since
  28ced2a125 and in every run of the two days before. The first failures follow
  04490cabb1 (sase-1le docs/CLI-usage landing) and 315092afeb (release-integration:
  config-loader fixtures, `identity.py` typing, `_sdd_role_available` research
  availability). Locally at a4465e18db the module passes (5 passed, 22 s).
- **Why the last watch failed to release its lander.** Watch phase sase-1lh.7.5.4.3.2
  ran `tools/release_watch --close-bead`, and the watcher did close the bead at 07:44Z
  (`MASTER_GATE_RED`). Its `--next` follow-up turn `sase-1lh.7.5.4.3.2--2` ran on
  grok-4.6. That turn hit "submission plan_digest is stale", re-ran
  `sase final context`, and the grok harness moved the command to the background after
  30 s. The provider has made no tool call since 07:58Z and was still RUNNING about 10 h
  later. The epic land agent waits on each phase **agent session** as well as each phase
  bead (`%w(<phase agents>, for_epic=false)` plus `%w(bead=…)`, built in
  `src/sase/bead/work_prompt.py`). So `sase-1lh.7.5.4.3.land` is still WAITING. Closing
  the bead was necessary but not sufficient.

## Rules for every phase

- Fetch and fast-forward before work, then re-read the current Master Gate and Full CI
  runs. Other agents push to master constantly, so check for concurrent fixes before
  editing and never redo or overwrite them. If PyPI already serves sase 0.18.0, a fix
  phase finishes any genuinely red gate it owns and otherwise closes with that evidence.
- No manual commits, branches, pushes, merges, workflow dispatches, or release edits.
  Host completion commits each fix phase. Only the watcher started by the watch phase
  dispatches workflows, and ci_watch owns the merge.
- Use `just fix` and `sase tool run check` for file changes. Never run
  `just check-full`. Read `lint_and_test.md`, plus `tui.md` or `tools.md` when you touch
  those areas, through `/sase_memory_read` before finishing. Shared backend behavior
  belongs in sase-core, opened with `/sase_repo`. No change here is expected to need it.
- No skips, xfails, retries of a failing step, wider time budgets, bare sleeps, or
  weakened assertions. A test must wait for the behavior it asserts.
- Some failures reproduce locally but have never failed CI, for example
  `test_deferred_wait_gates_before_claim_and_prepares_claimed_workspace` and
  `test_home_mode_running_marker_cleanup_updates_artifact_index`. They are not release
  gates. Record newly seen ones as `PROPOSED FOLLOW-UP:` notes, not repairs.
- Phase workers never create beads. They record discoveries as `PROPOSED FOLLOW-UP:`
  notes on their own phase bead.
- **Never add or re-key a Justfile `--epic-symbol` entry to the watch phase bead.** Key
  any entry you need to this epic's bead. `sase bead close` refuses unconditionally
  while an entry is keyed to the bead being closed, and the watcher must be able to
  close the watch bead unattended.

## Phase clock

**First, stop the stale agents from the previous attempt.** Run `sase agent list -j` and
confirm two things before acting. First, `sase-1lh.7.5.4.3.land` is still WAITING.
Second, `sase-1lh.7.5.4.3.2--2` is still RUNNING and its artifacts dir shows no tool
call after 2026-10-17T07:58Z (`tool_calls.jsonl`). Then run
`sase agent kill -n sase-1lh.7.5.4.3.land` first, so that lander never wakes on the
other kill. Then run `sase agent kill -n sase-1lh.7.5.4.3.2--2`. Wait about a minute,
confirm with `sase agent list -j` that neither has respawned, and record what you
stopped in a phase note. Touch no other agent. If either has already exited or changed
state, leave it alone and note what you saw. This frees the runner slot and workspace
the hung turn holds. This epic's land agent performs sase-1lh.7.5.4.3's landing through
`parent_bead`.

**Repair the date bomb (sase-1ma).** In `tests/test_agent_auto_restart_sweep_safety.py`,
replace every hardcoded calendar row stamp (`20261009T120000`, `20261009T121000`, the
matching `2026-10-09T12:00:00+00:00` ledger fields, and the `20261009120000_…` bundle
name) with values derived from the same `now` each test passes to the code under test.
Format them in `sase.core.time.get_timezone()` with the exact stamp formats the healer
parses; a small module helper is appropriate.

- Rows a test means to be recent go inside `_HISTORY_LOOKBACK_SECONDS` of that `now`.
- Rows a test means to be stale (`test_stale_legacy_row_is_ignored`) go explicitly
  outside it. Derive this from the constant, not from a calendar date.
- For every test that asserts a row is ignored, untouched, or idle, make the row
  in-window, so the assertion proves the named rule (non-skew, dismissed bundle, handled
  doorbell) rather than passing because the row aged out.
- Keep the production lookback and the disabled-path contract unchanged: silenced
  pending rows resurface exactly once while auto-restart is off.

**Sweep for siblings.** Search `tests/` for other literal calendar stamps (for example
`2026-10-`, `202610`, `2026101`) that feed wall-clock-relative logic: lookback, age,
TTL, retention, expiry, staleness, or "recent" windows. Fix any that has already aged
out or will age out before 2026-11-17, using the same derive-from-`now` approach. Leave
stamps used only for parsing or formatting. Bound the sweep to those comparisons, and
record any you judged safe in a phase note.

Verify the module and any swept modules, then run `sase tool run check`. Append a
`sase bead note sase-1ma` naming the fix. Leave closing sase-1ma to the land agent.

## Phase ci-reds

**Collect the current reds.** List every failing node in the latest Master Gate runs and
the latest completed Full CI run on master (`gh run view <id> --log-failed`, or
`gh api --allow-escape-sequences repos/sase-org/sase/actions/jobs/<job>/logs` for a
single job). The date-bomb node belongs to clock. Everything else deterministic is this
phase's work.

**Root-cause the tutorial failure.** Read `tests/test_tutorials.py` to see what "agents
appeared" polls, how the tutorial HOME is prepared, and which provider runs (fakey).
Then trace `sase run "+home …"` through `+home` project resolution, launch, and agent
registration in that fresh HOME. Start with what changed around the onset: 04490cabb1,
315092afeb, and the commits between 28ced2a125 and a80f2891e8. Look especially at
config-loader, SDD-role-availability, identity, and launch or registration paths a fresh
tutorial HOME exercises.

Reproduce under CI-like conditions: Python 3.12, a fresh venv where practical, `CI=1`,
repeated runs (at least 10), and CPU contention, since CI runners are small. The failure
is intermittent there and has not reproduced locally yet. When an attempt fails, capture
`sase run`'s exit status, stdout, stderr, and the tutorial HOME's agent artifacts or
logs.

Fix the race or regression you find, in product code when the product is wrong and in
the harness only when the harness is wrong. Do not lengthen the 60 s window or retry the
step. Wait on a real readiness signal only if evidence shows that legitimately slow
startup is the documented behavior.

**Make the failure self-diagnosing, whether or not you find the cause.** When this step
fails, `TutorialStepError` should carry the `sase run` exit code and bounded
stdout/stderr, plus the agent and artifact state found in the tutorial HOME. Then the
next CI failure explains itself. Run `tests/test_tutorials.py` repeatedly after the fix.
If the cause stays unproven after a bounded investigation, land the diagnostics and any
proven partial fix, and record the evidence and remaining hypotheses as a
`PROPOSED FOLLOW-UP:`. Do not weaken the test.

Fix any other deterministic red from the collection step. For reds that are flakes, add
the run IDs as a `PROPOSED FOLLOW-UP:` naming the existing owner bead.

Do not edit the two recurring flakes:

- sase-1ht:
  `tests/llm_provider/test_subprocess_utf8_decode.py::test_stream_json_lines_dispatches_stderr_during_a_large_stdout_burst`
- sase-1lr:
  `tests/ace/tui/test_link_follow_seam.py::test_beads_sync_reports_visible_and_missing`

Both are large flakes with unknown causes, and the watcher tolerates one red of each.
Only add fresh run IDs as notes when you observe them.

Run `sase tool run check` before finishing. Also run `just test-tutorials` through
`sase tool run` if the catalog offers it, otherwise run the module directly;
`just check` does not run that module.

## Phase watch

This phase makes no file edits. The exception is the epic-symbol fallback in step 3. It
ends its turn by handing off to a monitor that has **no `--next`**. No LLM turn runs
after the watch, and no follow-up can hang. The watcher process closes this bead, and
the monitored command always exits 0. A monitor member whose state is `completed` (or
`stopped`) and has no follow-up satisfies the land agent's agent-session wait. See
`effective_done_outcome` in `src/sase/core/dismissed_agent_completion.py` and
`test_completed_monitor_member_releases_plan_agent_session_waiter`. The phase is pinned
to `claude/sonnet` because its only risky moment is `sase monitor start`, which can take
up to a minute. The last attempt's harness backgrounded commands after 30 s and then
hung.

1. **Idempotent start.** `sase bead read` this phase bead. If it is already closed (a
   relaunch after recovery), run `/sase_final` and stop.
2. **Fix SHA and core.** Fetch and fast-forward. Confirm each sibling phase's commit is
   on `origin/master`, for example
   `git log origin/master --grep 'SASE_BEAD=\[<phase-id>\]'`. Re-fetch for up to about 5
   minutes if a host push is still landing. A sibling that closed with "no commit
   needed" evidence is fine. If an expected commit is still missing, close this bead
   with
   `sase bead close <phase-id> -R canceled -r "release watch not started: sibling commit missing"`
   and the evidence, then run `/sase_final`.

   Set FIXED_SHA to the newest `origin/master` SHA. Set CORE to the newest sase-core-rs
   on PyPI that passes `tools/check_sase_core_rs_bindings` against FIXED_SHA (0.41.0 at
   planning time; the previous watch used 0.40.1). If PyPI already serves sase 0.18.0,
   close this bead with that evidence as a note and run `/sase_final` without watching.

3. **Epic-symbol guard.** Run `sase bead epic-symbols <phase-id>`. It must list nothing.
   If it lists entries, re-key exactly those Justfile lines to this epic's bead. That is
   the only edit this phase may make. Then close this bead with
   `-R canceled -r "release watch not started: re-keyed epic-symbol entries"` and run
   `/sase_final` so the host commits the re-key. The land agent then starts the watch.
4. **Preflight inline.**

   ```sh
   PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --once --dry-run --close-bead <phase-id>
   ```

   It must exit 0 and print a fully parsed state line. If it fails, close this bead with
   `-R canceled -r "release not confirmed: WATCHER_BROKEN (preflight)"` and the output,
   then run `/sase_final`. Do not repair code in this phase.

5. **Record, then hand off.** First append a phase note containing FIXED_SHA, CORE, the
   preflight state line, and the exact monitored command. Then start exactly one monitor
   from the workspace root and run it in the foreground until it exits. Pass the whole
   command as one quoted string. The host runs it under `/bin/sh -c`, so do not add a
   `bash -c` wrapper.

   ```sh
   sase monitor start \
     -C "$PWD" \
     -r 'Watch sase v0.18.0 through ci_watch, Publish, and PyPI; the watcher closes <phase-id>' \
     -t 14h \
     -s 'WATCHING RELEASE' -S 'RELEASE WATCHED' \
     -L 'sase 0.18.0 watch' \
     -- 'PYTHONUNBUFFERED=1 NO_COLOR=1 GH_FORCE_TTY=0 timeout -k 10m 750m tools/release_watch --repo sase-org/sase --version 0.18.0 --fixed-sha <FIXED_SHA> --await-core-version <CORE> --deadline 12h --close-bead <phase-id>; rc=$?; echo "RELEASE_WATCH_EXIT=$rc"; sleep 120; sase bead close <phase-id> -R canceled -r "release not confirmed: watcher exit $rc left the bead open" || true; exit 0'
   ```

   Do **not** pass `--next`, `-m`, `--idle-timeout`, or `-p`. A successful start ends
   this turn; that handoff is exempt from `/sase_final`. If `sase monitor start` exits
   nonzero twice, close this bead with
   `-R canceled -r "release not confirmed: WATCHER_NOT_STARTED"` and both errors, then
   run `/sase_final`.

### Why the land agent always runs

The land agent needs two things: this bead closed, and this phase's agent session
settled successfully. The layers nest so that each ends inside the next one out. The
watcher stops at 12 h. The outer `timeout` stops at 12.5 h, plus 10 min of SIGTERM
grace. The 120 s pause plus the fallback close add a few more minutes. The monitor's own
timeout is 14 h, which therefore never fires on a live host.

| Path                                         | Who closes the bead                                                           | What settles the agent session                 |
| -------------------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------- |
| Any verdict, including 12 h `DEADLINE`       | Watcher, with in-process retries                                              | Wrapper exits 0, monitor `completed`           |
| Outer `timeout` at 12.5 h, or a watcher hang | Watcher signal path or detached closer; else the wrapper fallback after 120 s | Wrapper exits 0                                |
| Watcher crash, or its close fails            | Wrapper fallback close                                                        | Wrapper exits 0                                |
| Already closed with any reason               | Nobody needed; the fallback is refused as a conflicting re-close              | Wrapper exits 0                                |
| Missing commit, epic-symbol, or preflight    | This agent, inline                                                            | This agent's `/sase_final`                     |
| `sase monitor start` fails twice             | This agent, inline                                                            | This agent's `/sase_final`                     |
| A human stops the monitor                    | Watcher's detached closer (`STOPPED`)                                         | Monitor `stopped` counts as success            |
| Reboot (`lost`) or an external SIGKILL       | Nobody                                                                        | Needs a human: `sase bead work <this-epic-id>` |

Only the last row needs a human. In that case the waiting land agent stays parked, and
`wait_checks` reports it. Step 1 makes recovery safe: `sase bead work <epic-id>`
relaunches only the land agent if this bead is closed, and relaunches the watch if it is
open.

## Landing

Inherited **dispatch = yes** applies. Never merge PR #299 by hand.

1. **Confirm the release before landing anything.** Verify all six facts:
   - ci_watch merged PR #299;
   - GitHub release v0.18.0 exists;
   - Publish's build and publish jobs succeeded;
   - PyPI lists sase 0.18.0;
   - a fresh, isolated, unpinned install resolves 0.18.0 and reports its resolved
     sase-core-rs version;
   - the Master Gate and Full CI the release relied on are green.

   A canceled watch is not success. Read the watch bead's close note (the watcher's
   evidence) and the monitor's retained output, which you can find with
   `sase monitor list --all`, label `sase 0.18.0 watch`, line `RELEASE_WATCH_EXIT=`.

2. **If the release is not confirmed, repair it.** The land agent has the widest view,
   so this is its job.
   - When the cause is transient and needs no code change, rerun the same watcher
     command yourself under `/sase_monitor`. Transient means, for example, `DEADLINE`
     while CI was still converging, or a flaky red that is green on rerun. Omit
     `--close-bead`, since the phase bead is already closed, and give it a `--next` that
     resumes this landing.
   - When code repair is needed, plan the remaining work as a child epic with this
     plan's shape: parallel fix phases, then a Claude-pinned watch phase that uses the
     same follow-up-free, always-exit-0 monitor and this guarantee table.
   - Never end a turn promising to resume, and never claim success on a canceled watch.
3. **Land.** Follow the normal land procedure. Verify each phase's commits and notes and
   integrate with concurrent master changes. Triage every `PROPOSED FOLLOW-UP:` through
   `/sase_new_task`. Explicitly cover:
   - close sase-1ma once its fix is verified on master;
   - recheck sase-16p against the 0.18.0 install;
   - file a task for the grok follow-up hang in "Why the last watch failed" (stale
     `plan_digest`, a backgrounded `sase final context`, then 10 h with no tool call)
     unless an existing bead already owns it;
   - if anything differs on the `ubuntu-latest` → Ubuntu 26 runner migration that GitHub
     announced for 2026-10-19, record it.
4. **Resume the chain.** After this epic closes, `parent_bead` hands you
   **sase-1lh.7.5.4.3**. Its own land agent never ran (the clock phase stopped it), so
   audit it in full. Then continue up through sase-1lh.7.5.4, sase-1lh.7.5, sase-1lh.7,
   and sase-1lh while each remains complete, per the land procedure. Honor every saved
   post-release duty in their landing notes. For sase-1lh these include:
   - triage of the Install smoke heal and the unstable visual nodes in the release's
     Full CI manifest;
   - **stale_epics = supersede** for the sase-1kx tree and sase-1jz;
   - marking each linked plan done.

   If the clock phase found either stale agent still live and left it alone, report it
   in your final response. Never `--force` a successful nested landing. Stop at the
   first incomplete parent and record a note on it.
