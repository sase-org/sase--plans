---
tier: epic
title: 'Backlog Epic 1: landing gates, test cost, and epic liveness'
goal: 'The five most important open task beads are fixed and closed. The land agent
  then starts Backlog Epic 2, and the chain continues five beads at a time until no
  open task beads remain.

  '
phases:
- id: landing-safety
  title: After-rebase import gate and screenshot --window safety
  depends_on: []
  size: medium
  description: 'landing-safety: make the after-rebase import gate ignore virtualenv
    files so it certifies again (sase-1un), and refuse sase screenshot --window targets
    it did not launch (sase-1d9).'
- id: test-cost
  title: Green, meaningful test-cost budgets
  depends_on:
  - landing-safety
  size: large
  description: 'test-cost: make just test-cost, check-full, and the CI Telemetry test-cost
    job green and meaningful on master again, per the cost_budgets decision (sase-j0).'
- id: failed-turn-handoffs
  title: Held staged handoffs from failed turns strand visibly
  depends_on:
  - test-cost
  size: large
  description: 'failed-turn-handoffs: stop treating a never-started staged handoff
    from an ended, failed turn as a live owner in sase-core, so its bead strands visibly
    (sase-1sh).'
- id: zombie-epics
  title: Host-owned liveness for zombie epics
  depends_on:
  - failed-turn-handoffs
  size: large
  description: 'zombie-epics: detect in_progress plan beads whose children are all
    closed and whose land agent is gone, report today''s zombies, and recover per
    zombie_recovery (sase-12u).'
decisions:
  cost_budgets:
    ask: Re-baseline test-cost budgets whose misses trace to suite growth rather than
      a named regression?
    choices:
      rebaseline: Re-measure on a quiet host; reset stale budgets with headroom; fix
        only named regressions
      cut: Keep every committed budget and cut suite cost until it passes; may need
        a child epic
      retire: Drop the summary-level budgets from the gate and keep only the per-cause
        budgets
    default: rebaseline
    why: The suite grew from ~28k to ~62k tests since the budgets were set; a forever-red
      gate gets ignored
    answer: rebaseline
  zombie_recovery:
    ask: Should the host relaunch the land agent of an epic whose children all closed
      while no land agent is live?
    choices:
      relaunch: Host relaunches the land agent of newly zombied epics; existing zombies
        are only reported
      notify: Host sends one inbox notification per new zombie epic and launches nothing
      report: No live mechanism; ship only the report command for existing zombies
    default: relaunch
    why: Nothing surfaces zombie epics today, and the land agent still verifies before
      closing anything
    answer: relaunch
proposed_by: bbugyi200.athena.1h5
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 10:21:00
status: done
bead_id: sase-1vn
---

- **PROMPT:** [prompts/202611/backlog_epic_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/backlog_epic_1.md)
- **BEAD:** [sase-1vn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vn/README.md)

# Backlog Epic 1: Landing Gates, Test Cost, and Epic Liveness

This is the first in a chain of backlog epics. Each one works at most five open task
beads, runs its phases one at a time, and ends with a land agent that plans the next
epic. The chain stops when no open task beads remain.

The backlog was triaged on 2026-11-27 against master `6b58ad0fe2`. All 464 open task
beads were checked for duplicates and for work that is already done:

- 21 were closed with evidence: 20 were already fixed or no longer applied, and sase-1tn
  was closed as a duplicate of sase-1q8.
- 443 open task beads remain.
- One lost change was found and recorded as a `DISCOVERED ISSUE:` note on active epic
  sase-1ra: phase sase-1ra.7 is closed but its change never reached master.

## Why These Five Beads

- **sase-j0** (+38): `just check-full` and the CI Telemetry `test-cost` job have been
  red on clean master since August. Two other beads were already closed as its
  duplicates.
- **sase-1un**: the after-rebase import gate never certifies a real workspace, so rename
  and import conflicts can reach master again.
- **sase-1d9**: `sase screenshot --window` sends SIGUSR2 to any tmux pane. That has
  already killed a user's long-running TUI.
- **sase-1sh**: a phase worker that stages a monitor and then fails leaves its epic
  stranded with no signal. The backlog chain depends on epics landing reliably.
- **sase-12u**: closing a follow-up epic never closes, or relaunches the land agent of,
  its parent plan. About 30 zombie epics now bury the real stalls among the more than
  120 in-progress plan beads.

Governing decisions: `decisions:check-full-is-explicit`,
`decisions:handoffs-end-through-host-completion`, `decisions:rust-core-required`,
`decisions:ci-two-speed-split`, and `decisions:host-owned-completion`.

## Phase `landing-safety` (medium): sase-1un and sase-1d9

### sase-1un: after-rebase import gate

- **Problem.** In `tools/_check_rebase_imports.py`, `_python_paths_not_matching_tree`
  unions four git queries. One of them is `git ls-files -o -i --exclude-standard`, which
  lists every gitignored `.py` file under `.venv/` and `.venv-*`. Thousands of
  virtualenv files therefore count as "dirty", and `_verify_committed_tree` raises
  `CheckerError`. Stitch then treats that as an infrastructure error and prints
  "allowing publication", so the gate never certifies anything.
- **Fix.** Limit the dirty and untracked queries to first-party import roots (`src/`,
  `tools/`, `tests/`, and any other roots the checker resolves imports from), or exclude
  virtualenv directories. Do not weaken the check for real first-party shadowing files.
- **Tests.** Add a regression test with an ignored `.venv/lib/.../x.py` file: the
  checker must certify a clean tree. Also confirm a genuinely dirty first-party `.py`
  file still fails the check.

### sase-1d9: screenshot --window signals panes it does not own

- **Problem.** `src/sase/screenshot/local.py` reads `#{pane_pid}` in
  `_inspect_tmux_window` and runs `kill -USR2 <pane_pid>` in
  `_request_export_with_retries`. When the window's `@sase_screenshot_dir` option is
  empty, it falls back to `screenshot_request_dir()` instead of refusing. Meanwhile,
  `src/sase/ace/tui/actions/screenshot_export.py` installs its SIGUSR2 handler only when
  `SASE_TUI_SCREENSHOT_DIR` is set, so any other `sase tui` is terminated by the signal.
- **Fix.** Before any signal is sent, refuse a `--window` target that lacks the
  `@sase_screenshot_dir` option. Exit non-zero with a clear error that names
  `sase screenshot --keep` as the way to get an owned target.
- **Tests.** Cover both cases: a refused unowned target, where no signal is sent, and
  the existing owned-target recapture path.
- `tui_screenshot.md` already describes `--window` with an owned `--keep` target, so no
  memory edit is needed.

### Verify and close

Run `just check` through `sase tool run check`. Close sase-1un and sase-1d9 with
verification notes, then close the phase bead.

## Phase `test-cost` (large): sase-j0

This phase's assigned bead explicitly authorizes `just test-cost` and `just check-full`,
as `decisions:check-full-is-explicit` requires. Run them only through `/sase_monitor`.
Read sase-j0 and all of its notes first.

### State on 2026-11-27

- `tests/perf/baselines/test_cost_budgets.json` last changed on 2026-09-18
  (`932e6ffae2`), and `tools/check_test_cost_budgets` on 2026-08-23.
- The newest full cost recording (`timings/cost/20260919T182806Z`) still misses four
  hard budgets: `ace_settle_pilot.cpu`, `parser_create.cpu`, `pilot_pause_delay.cpu`,
  and `subprocess_run.count`. Every later recording covers a partial run.
- The CI Telemetry `test-cost` job (`.github/workflows/telemetry.yml`) is cancelled at
  its 180-minute cap on every run.
- Notes from 2026-08-17 show that wall-derived keys (`idle_seconds`,
  `total_file_wall_seconds`, `subprocess_run`, `pilot_pause_delay`) move with host load
  and worker count. Judge those keys per worker or under a controlled worker count.

### Work

1. Take one fresh full recording on a quiet host.
2. Classify every missed key as one of: a regression with a named cause, suite growth,
   or host-load noise.
3. Apply the `cost_budgets` decision:

> [!decision] cost_budgets = rebaseline Reset the stale summary and cause budgets from
> the fresh recording with documented headroom. Make the wall-derived keys robust to
> load and worker count. Fix any miss that traces to a specific regression.

> [!decision] cost_budgets = cut Leave the committed budgets unchanged and reduce suite
> cost until they pass. If that does not fit one agent, author a child epic whose land
> agent closes this phase.

> [!decision] cost_budgets = retire Remove the summary-level budgets from
> `tools/check_test_cost_budgets` and keep the per-cause budgets. Update any docs or
> test that describes the summary gate.

4. Make the Telemetry `test-cost` job finish inside its cap, for example by selecting a
   bounded worker count or the same inputs the local gate uses. If it cannot, record the
   reason and the remaining work on the phase as a `PROPOSED FOLLOW-UP:`.

### Verify and close

- `just check-full` passes end to end on the phase's tree.
- `tools/check_test_cost_budgets` still fails when you inject a synthetic regression,
  for example a doctored recording in a unit test.
- `just check` passes.

Then close sase-j0 and the phase bead.

## Phase `failed-turn-handoffs` (large): sase-1sh

### Problem

A phase fork staged monitor `3w69pb9ezmfy`, then its turn ended `DONE (failed)`. The
monitor stayed `? staged` for five days with no pid, and the phase bead stayed
in_progress. The land agent stayed parked on it, nothing reported it as STRANDED, and
`sase questions` escalation was blocked.

Under `decisions:handoffs-end-through-host-completion`, adoption is fail-closed, so
holding the handoff is correct. Two things are wrong:

- the held handoff is silent;
- `crates/sase_core/src/stranded_work/classify.rs` (sase-core; see `owner_kind()` and
  lines 135-152 at triage time) still treats a staged or held `handoff_not_adopted`
  member as the bead's owner.

Epic sase-1ux only made such members visible as NOT ADOPTED in the TUI.

### Work

1. Confirm the root cause: does host completion hold, or skip, staged handoffs when the
   turn outcome is `failed`? Read the host-completion epilogue before changing anything.
2. In sase-core, stop counting a staged, never-started handoff from a turn that has
   ended as a live owner. The bead then classifies as stranded, and the existing
   stranded-work notification and recovery paths take over.
3. Settle or expire the held staged record so that later turns and `sase questions`
   escalation no longer treat it as a pending handoff.
4. Ship the sase-core change with tests. Move sase's `sase-core-revision.txt` pin past
   it (see `docs/rust_backend.md`), and update any Python adapter or callers.

Related open beads (do not fold them in; mention them in the close note):

- sase-1r4 (owned by sase-1ra) is a different trigger that also leaves zombie staged
  monitors.
- sase-1si and sase-1sj come from the same incident but need different fixes.

### Verify and close

Run `sase tool run check` in sase-core and in sase. Add a regression test that
reproduces the failed-turn staged-monitor shape. Then close sase-1sh and the phase bead.

## Phase `zombie-epics` (large): sase-12u

### Problem

In sase-core, `close_one_and_delegated_parent` (bead mutation, `close_remove.rs` at
triage time) cascades a close upward only when the parent is a phase. When the parent is
a plan, the parent's landing depends on the child land agent's LLM ancestor walk in the
`bd/land_epic` prompt (`src/sase/default_config.yml`). When that walk is lost, the
parent stays in_progress forever. Live examples: sase-zu and sase-zn have every child
closed and are still in_progress.

### Work

1. In sase-core, add detection for plan beads that are in_progress, have every
   descendant closed, and have no live land agent. This is the shared predicate that the
   CLI, the TUI, and AXE all consume.
2. Add a one-time, report-only command, or an option on an existing bead command, that
   lists today's zombie epics with their last child close time. It never closes or
   launches anything.
3. Apply the `zombie_recovery` decision:

> [!decision] zombie_recovery = relaunch Add a host-owned AXE job, or extend the
> existing stranded-work routine, that relaunches the parent's land agent through the
> same path as `sase bead work <epic-id>`. It acts only for epics whose last child
> closed after the job ships, and it is rate-limited and idempotent: one relaunch per
> epic per close event. Existing zombies appear only in the report.

> [!decision] zombie_recovery = notify The same detection sends one inbox notification
> per newly detected zombie epic, deduplicated per epic, and launches nothing.

> [!decision] zombie_recovery = report Ship only steps 1-2.

4. Never close a plan bead automatically. The land agent stays the only closer, which
   keeps `sase_beads.md` ("Never close the parent epic bead; its land agent does that")
   true.
5. Move sase's `sase-core-revision.txt` pin past the sase-core change.

### Verify and close

Run `sase tool run check` in both repos. Add tests that cover these cases:

- detection on a fixture store;
- a plan with an open child is not a zombie;
- a plan with a live land agent is not a zombie;
- for `relaunch`: one relaunch per close event, and none for epics that predate the job.

Then close sase-12u and the phase bead.

## Land: Close This Epic, Then Plan Backlog Epic 2

The land agent first does its normal verification, integration, and follow-up triage,
and then closes this epic. Only after the close does it plan the next backlog epic.

**Order matters.** Propose the next epic only after this epic's bead is closed.
`sase plan propose` makes an epic top-level when the proposing agent's active bead is
already closed. Proposing earlier would make the next epic a child of this one, which
nests the chain and recreates the sase-12u zombie problem.

If landing needs remaining work and the land agent proposes a child plan for it, that
child plan owns these steps. It runs them after it closes this epic, and this land agent
does not also plan Backlog Epic 2.

Steps after closing this epic:

1. **Count.** Run
   `sase bead list -t task -s open -s ready -s snoozed -s claimed -s in_progress -n 0`.
   If it lists no task beads, the chain is finished: say so and finish normally.
2. **Close duplicates and obsolete beads.** This is a scoped sweep, not a full
   re-triage. Check:
   - beads this epic's work may have resolved (beads linked to the five above, and beads
     in the same subsystems);
   - beads created or +1'd since the previous sweep (2026-11-27 for this epic);
   - every candidate you are about to pick.

   Close only with concrete evidence: a fixing commit plus current code that no longer
   has the defect, or a target that no longer exists. Use
   `sase bead close <id> -R done|canceled|superseded -r "<reason>" -n "<evidence>"`. A
   newer duplicate closes as `superseded` in favor of the older bead, and its evidence
   is carried over as a note. A flake passing once is not evidence.

3. **Pick and plan.** Choose the five most important remaining task beads. Rank them by
   severity: red master or CI gates first, then data loss or corruption, then
   user-facing breakage, then corroboration (+1 count and recency). Skip beads that an
   active epic already owns.

   Use `/sase_plan` to author "Backlog Epic N+1" with these rules:
   - at most five phases and at most five task beads;
   - phases run strictly in sequence (each depends on the previous one);
   - a phase may work more than one bead;
   - few `decisions:`, used mainly to let the user opt out of questionable feature or
     memory work;
   - a copy of this whole "Land" section, with the sweep date and epic number updated,
     so the chain continues.

   Validate the plan, propose it, and finish through `/sase_final`. Never wait on the
   proposal.

Starting queue for Backlog Epic 2, ranked at triage time (re-verify each one before
picking it):

- **Flakes and test isolation:**
  - sase-1kr (+7): stale index-lock recovery flake. It pairs with sase-1s4, a test that
    leaves a stale lock in a real checkout.
  - sase-1em (+6).
  - sase-1oo (+5), sase-1on, sase-1pr: one broad-lane isolation cluster, together with
    sase-1tp, 1to, 1tq, 1tr, 1ts, and 1sg.
  - sase-r2 (+4).
  - sase-1f9 (+4).
  - sase-1v2 (+3).
  - sase-1q8 (+2): now also carries sase-1tn's evidence. It forms a fakey runner-slot
    cluster with sase-1u0, sase-1u1, sase-n6, and sase-v0.
- **Bugs:**
  - sase-199 (+4): the Axe j/k bench collects samples from other tabs.
  - sase-1aw: federation reports a stale snapshot as ok.
  - sase-1j2: remote fork launches a local agent.
  - sase-1nl: the sudo receipt check rejects command output that contains `token=`.
  - sase-1fw: `$` hook reset is not durable.
  - sase-1u6: the TUI does I/O on its main thread.
  - sase-1vg: the leak-detector thread poisons the CI Telemetry gate.
- **Human-gated, not for an agent phase:** sase-1ot (secret rotation) needs the owner.

Clusters worth one epic each later:

- bead-event actor attribution: sase-159, sase-15a, sase-1ri;
- bead-scale performance: sase-1iu, sase-1iv, sase-1iw, sase-1ix, sase-1j5;
- stale memory notes, batched by note: tui_perf.md (sase-1f2, 1fd, 1fq, 1jr), macros.md
  (sase-1hu, 1jq), lint_and_test.md (sase-1dk, 1hv), and glossary strands (sase-1bx,
  1c9, 1cw, 1kc).

Close candidates to re-check first: sase-1uv (its guard has been green since 2026-11-22)
and sase-1kq (the code documents the 33-cell spacer as intended).
