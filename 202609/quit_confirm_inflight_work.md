---
tier: tale
title: Confirm TUI quit/restart when it would interrupt live work
goal:
  q and the Q quit/restart menu show a y/n confirmation (default No) that lists exactly
  what would be lost whenever an exit would interrupt in-process TUI work or an
  in-flight scheduler launch batch, and behave exactly as today when nothing would be
  lost.
size: medium
proposed_by: bbugyi200.apollo.2o
create_time: 2026-09-28 10:18:37
status: wip
---

# Plan: Confirm before quitting/restarting the TUI when the exit would interrupt live work

## Problem

`q` in sase's TUI (`action_quit`, `src/sase/ace/tui/actions/lifecycle.py`) quits with no
confirmation, and the `Q` quit/restart menu (`QuitOptionsModal`,
`src/sase/ace/tui/modals/quit_options_modal.py`, routed by `action_stop_axe_and_quit` in
`src/sase/ace/tui/actions/axe.py`) runs the chosen option right away. Its only warning
is `N procs will be stopped`, which is wrong. That count (`_count_running_tasks` →
`gear_eligible_count`) is mostly supervisor-backed durable procs, which survive the TUI
exiting. Nothing warns about the two kinds of work an exit really destroys:

1. **Work that lives in this TUI process.** Session workers (`_submit_session_worker`,
   shown as `_session_overlay_rows()`) run on in-process threads. So do durable
   submissions whose submit thread has not yet written the go-barrier
   (`_durable_submit_workers`). All of them die at `os._exit` / `os.execv`. This applies
   to `q`, `Q→1`, `Q→2` and `Q→3`.
2. **Scheduler work that `Q→1` ("Quit & Stop scheduler") and `Q→3` ("Restart TUI &
   service host") kill.** A scheduler job ("chop") that proposes launches runs them one
   at a time inside the routine process (`launch_chop_proposals` →
   `multi_prompt_launch_execution.py` loop). The routine's SIGTERM handler only sets
   `_running = False`. The service host SIGKILLs the scheduler process group after about
   10 s. Nothing records the launches that had not happened yet, and nothing resumes
   them.

`docs/ace.md` ("Quit / Restart Menu", around lines 3575-3591) still says a plain `q`
shows a confirmation dialog when procs are running. That dialog (`QuitConfirmModal`) was
removed in commit 8c4840458 when procs became durable, and nothing replaced it.

### Incident that motivated this (athena, 2026-09-28; already verified, no cleanup needed)

- 09:32:00 EDT: the `toobig_split[sase]` chop run `20260928T093200_850344` (routine
  `run_every`) started launching a 61-member clan `toobig-68` in sequence.
- 09:32:28 and 09:32:37: members 1 and 2 were created (`agent_tabs.0`,
  `tool_runs_pane.0`).
- 09:32:53: the user picked `Q` → `3/a` "Restart TUI & service host". The TUI re-exec'd
  (same pid, new session), and the new TUI ran `systemctl --user restart sase.service`
  at 09:33:00. The old host was SIGKILL-stopped by 09:33:11, which killed the routine
  partway through its launch loop. 59 launches were lost silently.
- The half-launched clan then blocked the chop ("inhibited by active agent clan
  `toobig-68`") until the user killed the leftover members (09:33:42 and 09:58:17). The
  09:58:25 run then launched a complete `toobig-69`.
- Verified state afterwards: no `toobig-68` processes are left, both members are
  `DONE (dismissed)`, there are no workspace or project-file claims, and `toobig-69` is
  launching normally. **Do not touch athena as part of this plan.**

## Goal

Before any exit path would interrupt real work, show a y/n confirmation (default
**No**). The prompt lists exactly what would be lost. When nothing would be lost, `q`
and the `Q` options behave exactly as they do today (no extra prompt). The quit menu's
warning becomes truthful.

## Design

### 1. Scheduler in-flight launch detector (non-TUI, reusable by a future CLI guard)

Add a small read-only helper under `src/sase/axe/`, next to `_state_chops.py`, and
re-export it through `sase.axe.state` like the other state helpers. For example
`find_inflight_chop_launches() -> list[InflightChopLaunch]`, a frozen dataclass with
`lumberjack_name`, `chop_name`, `run_id`, `started_at`, `proposed_count`,
`launched_count` and an optional `clan` (the proposal's clan template, e.g. `toobig-@`,
or the allocated clan name if it is cheap to resolve).

Rules. Keep I/O bounded: newest run only per chop, and no log tails.

- Iterate the routine state dirs (`list_lumberjack_names()`). Skip a routine whose
  recorded pid (`read_lumberjack_pid`) is not alive. Stale `running` rows from dead
  routines are not in-flight.
- For each chop under the routine, read only the newest run id (`read_chop_run_index` is
  newest-first) and its run entry (`read_chop_run`).
- A run is in-flight **only** when all of these hold:
  - `status == "running"` and `finished_at is None`.
  - Its `<run_id>.result.json` (`chop_run_result_path`) has a non-empty
    `proposed_launches`.
  - Fewer launches are recorded than proposed. Count the records for that `run_id` in
    the routine's `agent_chops.json` (see `src/sase/axe/chop_agents.py` for the reader
    and record shape), compared with `len(proposed_launches)`.
- Exclude runs that are still in their job-script phase (no result yet) and `launched`
  runs. Job scripts are short and re-run on the next tick, and launched agents survive a
  scheduler stop because runners are detached. Including them would make the prompt fire
  almost every time.
- Never raise. On an unreadable or corrupt file, skip that entry.
- Before relying on the claim that clan proposals always go through the in-process loop,
  check which launch path they take (`chop_proposal_launch.py`: the typed-admission
  `%if` path versus the multi-prompt loop). The detection rule above is status-based and
  holds either way.

The scheduler's run state is Python-owned in this repo, with no Rust model. So this
stays Python and does not cross the `sase_core` boundary.

### 2. TUI-instance exit impact (in-memory, synchronous, no I/O)

Add a small helper (e.g. `src/sase/ace/tui/quit_impact.py`, or private methods in
`LifecycleMixin`) that returns a `TuiExitImpact`:

- Labels of active session-worker rows (`_session_overlay_rows()` filtered by
  `proc_status_is_active`).
- The number of durable submissions still in flight (`_durable_submit_workers` entries
  whose worker has not finished).
- `is_empty` / `summary_lines(limit=5)` helpers that render at most 5 lines, then
  `…and N more`.

Durable procs, `!` background commands, monitor turns and service procs are deliberately
**not** included, because they outlive the TUI. Pending launches that were never
submitted are already cancelled and stashed for `@` at quit. They may appear as an
informational line, but they must not trigger the prompt on their own.

### 3. `q` (`action_quit`)

Keep the existing artifact-pane toggle short-circuit first. Then:

- Empty impact: `await self._begin_controlled_exit()`, the same as today.
- Otherwise: push `ConfirmActionModal`
  (`src/sase/ace/tui/modals/confirm_action_modal.py`). Use `kind=ConfirmKind.DANGER` so
  Enter defaults to No. Suggested title "Quit sase's TUI?". The message lists the impact
  lines, with confirm label "Quit" and cancel label "Stay".
  - On confirm, call `self._request_controlled_exit()`.
  - On decline or escape, do nothing.
  - `ConfirmDialog` already binds `q` to cancel, so pressing `q` twice cannot quit.
  - Guard re-entry so a second `q` while the dialog is open does not stack dialogs.

### 4. `Q` menu (`action_stop_axe_and_quit` / `QuitOptionsModal`)

- Replace `running_task_count` / `N procs will be stopped` with a truthful line computed
  from the in-memory TUI impact, e.g. `N TUI task(s) will be interrupted`. Say nothing
  about durable procs, since they keep running. Keep the modal otherwise unchanged: keys
  `1/s`, `2/r`, `3/a`, `esc/q`.
- After a choice:
  - `restart_tui`: confirm only if the TUI impact is non-empty.
  - `quit_stop_axe` and `restart_tui_and_axe`:
    - Collect `find_inflight_chop_launches()` off the event loop **and** off the app
      message pump. Use `spawn_pump_free_task` (`src/sase/ace/tui/util/pump_tasks.py`)
      or a thread worker, and marshal the result back to the UI thread (see the
      `tui_perf` memory rules 1-2).
    - Then confirm if either impact is non-empty. Each scheduler line should read like
      `Scheduler is launching toobig_split[sase] (toobig-@): 2/61 agents launched — stopping it drops the remaining 59`.
    - For `restart_tui_and_axe`, word the title and message around restarting the
      service host.
  - Confirmed: run the existing path unchanged (`_stop_axe_and_quit` / `_restart_tui`).
  - Declined: return to the TUI. Do not exit and do not stop anything.
  - Empty impact: behave exactly as today.
- Handle the same re-entry and duplicate-keypress guard as in step 3.

### 5. Clean up the stale count

Remove `_count_running_tasks` if nothing else uses it after the change (grep first).
Otherwise leave it, and repoint its docstring so it no longer claims to be a quit-kill
count. Keep `gear_eligible_count` unchanged; the top-bar gear uses it.

### 6. Docs

Rewrite `docs/ace.md` "Quit / Restart Menu" (around lines 3575-3591) to describe:

- what the new prompt covers (in-process TUI work, and in-flight scheduler launches for
  options 1 and 3);
- that durable procs, `!` background commands and monitors survive quitting;
- that `q` quits immediately when nothing would be interrupted.

Also check that `docs/ace.md` around lines 1154, 1438 and 2975 (bgcmds, persist-cleanup,
pending-launch stash) still agree.

No keymap changes are needed: `y`/`n` come from `ConfirmDialog`, and `q`/`Q` stay as
they are. `src/sase/default_config.yml` needs no edit.

## Tests

- **New:** `tests/axe/` (or wherever `_state_chops` tests live) covering
  `find_inflight_chop_launches`, using a redirected SASE home with fake routine pid,
  index, run, result and `agent_chops.json` files. Cover:
  - an in-flight mid-launch run (2/61);
  - a job-script-phase run with no result (excluded);
  - a fully launched run (excluded);
  - a `launched` status run (excluded);
  - a dead routine pid (excluded);
  - corrupt JSON (skipped, no raise).
- **Update** `tests/ace/tui/actions/test_lifecycle_quit_confirm.py`. It currently
  asserts that `q` never shows a modal. Change it to:
  - no active session workers or in-flight submits → exits with no modal (durable proc
    rows alone must not prompt);
  - an active session worker → `ConfirmActionModal` is pushed; `y` exits and `n` stays;
  - the existing flush-ordering tests keep passing.
- **Update** `tests/ace/tui/actions/test_axe_stop_quit.py` for each option: no impact →
  same as today; TUI impact → confirm for all three options; scheduler impact → confirm
  only for options 1 and 3. Declining never calls `stop_service_proc` or `_restart_tui`.
  Monkeypatch the detector so no real state is read.
- **Update** `tests/ace/tui/test_quit_options_modal.py` for the new warning text.
- Keep `kill_task_calls == 0` style assertions: no exit path may kill durable procs.

Before finishing, follow the `lint_and_test` memory for this repo. Run the repo's check
recipe through `sase tool run check`, and read the `tui` / `tui_perf` memories before
editing TUI code.

## Follow-up to file (use `/sase_new_task` first, which checks for duplicates)

File one task bead (type `bug`, size `large`) describing the backend root cause this
plan only warns about:

- A scheduler or service-host stop gives a chop's in-process proposal launch loop no
  chance to drain. The routine's SIGTERM handler only sets `_running = False`, and the
  host SIGKILLs after about 10 s.
- Unlaunched proposals are neither persisted nor resumed, so a clan ends up partially
  launched and then blocks its own chop.

Evidence to include: the `toobig-68` timeline above. Candidate directions:

- drain or extend the stop grace while a launch batch is active;
- persist remaining proposals for resume;
- add the same guard to `sase scheduler stop`, `sase service restart` and the
  Services-tab stop/restart actions.

## Out of scope

- Changing scheduler or service-host stop semantics (the follow-up bead above).
- CLI-side warnings (`sase scheduler stop`, `sase service restart`) and Services-tab
  stop/restart confirmations.
- The update-restart flow (`restart_after_update_when_ready`).
- A `%wait` on a killed clan member never resolving, which kept
  `toobig-68.tool_runs_pane.0` WAITING until it was killed by hand.
