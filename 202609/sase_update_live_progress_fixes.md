---
tier: epic
title: Finish sase update live progress
goal: '`sase update` live progress holds up on a real terminal. Result panels print
  intact after the live region is torn down, and rows render in execution order. The
  managed uv step streams its output and package rows, interrupts and failures produce
  one accurate final frame, and streaming subprocesses honor their timeouts. The CLI
  completion snapshot matches the parser again.

  '
phases:
- id: stream-and-backend-fixes
  title: Bound streaming waits, fix CRLF, and fix backend step details
  depends_on: []
  size: small
  description: 'stream-and-backend-fixes: always group-kill and bound the pump joins
    in run_streaming (enforcing timeouts after leader exit), keep CRLF lines, isolate
    prebuild stdin, parent the health-check repair row, clear stale transient merge
    details, and regenerate the stale completion snapshot.'
- id: timeline-renderer-fixes
  title: Fix the timeline model, renderers, and session lifecycle
  depends_on: []
  size: medium
  description: 'timeline-renderer-fixes: stop Live before the final frame, and make
    print_final finalize and run once. Add precise interrupt finalization, trailing
    rows, a line counter for verbose output past 200 lines, robust plain fallback,
    full-width titles with an m:ss running clock, and the log mode line; remove dead
    members.'
- id: handler-wiring-fixes
  title: Fix the update handlers, managed rows, and docs
  depends_on:
  - timeline-renderer-fixes
  size: medium
  description: 'handler-wiring-fixes: print stdout only after teardown in the live,
    mode-switch, and dry-run handlers. Stream managed uv output with normalized, parented
    package rows, declare trailing rows in execution order, and make interrupts single-shot.
    Pass argv to the log, dedupe RunUvFn, fix the docs, and add shared-terminal tests.'
proposed_by: bbugyi200.apollo.sase-158.land
parent_bead: sase-158
create_time: 2026-09-21 16:18:51
status: wip
bead_id: sase-158.6
---

- **PROMPT:** [prompts/202609/sase_update_live_progress_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sase_update_live_progress_fixes.md)
- **PARENT:** [202609/sase_update_live_progress.md](https://github.com/sase-org/sase--plans/blob/main/202609/sase_update_live_progress.md)
- **BEAD:** [sase-158.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-158/sase-158.6.md)

# Plan: Finish `sase update` live progress (sase-158 landing fixes)

## Context

Epic sase-158 ("Live, streaming progress for `sase update`", plan
`202609/sase_update_live_progress.md`) landed all five phases, but its land-time review
found defects caused by the epic. The worst one: on an interactive terminal, the success
output is corrupted. This plan covers only that remaining work. Closing sase-158, its
symvision pass, and its plan-file status update belong to sase-158's land agent, not to
these phases.

Main files: `src/sase/dev_update/stream_command.py`, `src/sase/dev_update/prebuild.py`,
`src/sase/dev_update/reconcile.py`, `src/sase/dev_update/roots.py`,
`src/sase/update_progress/` (events, timeline, render_live, render_plain, log_sink,
fanout, session), `src/sase/main/update_handler_live.py`,
`update_handler_mode_switch.py`, `update_handler_dry_run.py`,
`update_handler_support.py`, `src/sase/mode_switch/execute.py`,
`src/sase/main/update_types.py`, `docs/plugins.md`, and
`tests/completion/snapshots/cli_spec.json`.

## Defects found at landing (all reproduced or confirmed in code)

1. **Corrupted terminal output (high).** The live, mode-switch, and dry-run handlers
   print stdout result panels while the transient `rich.live.Live` region is still
   running. `print_final` prints the static frame but never stops Live, and the session
   only exits after the panels print. The `out` console writes straight to the real
   stdout (rich unwraps its own redirect proxy), so Live loses track of its region. Its
   10 Hz refreshes and the transient erase on stop then wipe stdout lines. What you see:
   a stale duplicate timeline, a result panel with its border stuck to the frame, and
   erased restart and completion lines. The dry-run panel's tail also gets erased.
2. **The managed uv step gets no output.** `_ManagedPackageWatcher.sink` only parses
   `+/- name==ver` lines and never calls `progress.output("managed", ...)`. As a result
   there is no live tail, no failure expansion, no `-v` streaming, and no output in the
   log file for the uv leg.
3. **Transitive uv packages become top-level rows.** `start("managed:<pkg>")` on an
   undeclared id auto-appends a top-level row. Only planned packages are declared, as
   children. Ids use the raw uv name, not `normalize_distribution_name`.
4. **Wrong row order.** `restart` and `completions` (and `managed` in mixed mode) are
   declared during inspect, before `check`/`merge`/`reconcile:*` exist. Declaration
   order is display order, so "Restart scheduler" renders above "Check for updates".
   Mode switch declares no restart/completions rows up front at all.
5. **Final frames show pending rows as `○`.** Failure paths call `print_final` before
   `finalize()`, which only runs in `session.__exit__`. In plain mode, the session exit
   then prints `– <step>` lines below the error panel and the `Full log:` line.
6. **Interrupt marking is too broad.** `finalize("interrupted")` also marks
   never-started pending rows as interrupted. A Ctrl-C that lands after the successful
   `print_final` prints the final frame twice. If it lands after the success journal
   write, it also appends a second `interrupted` journal entry.
7. **`-v` goes silent after 200 lines per step.** Both renderers track
   `seen = len(row.tail)` against a `deque(maxlen=200)`. Once the deque is full, nothing
   new prints, and lines that roll off between polls are silently lost.
8. **Unbounded streaming waits.** `run_streaming` waits only for the group leader. After
   a timeout or interrupt it sends SIGKILL only if the leader survives the grace period,
   and `_join_pumps` then waits on pipe EOF with no limit. So a group member that
   ignores SIGTERM or SIGINT, or a backgrounded grandchild that holds the pipes, hangs
   the call past its timeout. With `sh -c "sleep 5 & echo done"` and `timeout=1`, the
   call returns rc 0 after 5 s, while the old `subprocess.run` path raised
   `TimeoutExpired` at 1 s.
9. **CRLF lines reach the sink as empty strings.** `raw_line.split("\r")[-1]` returns
   `""` for `"hello\r"`.
10. **Prebuild streaming drops stdin isolation.** The streaming branch of the prebuild
    probe runner calls `run_streaming` without `stdin=subprocess.DEVNULL`, which the
    non-streaming `run_noninteractive` path guarantees.
11. **The health-check repair row is top-level.** `reconcile:<i>:repair` is started
    without being declared under `reconcile:<i>`, so it renders below completions.
12. **Stale transient details.** A merge child finished `done` keeps `merging…` when
    `format_merge_detail` returns `None`. Auto-skipped children keep
    `fetching…`/`checking…`.
13. **Long titles are always cut off.** The live table caps the title column at 28
    characters, so the plan's own titles (up to 35, e.g. "Rebuild Rust core into uv-tool
    venv", "Upgrade sase + plugins via uv") always render with an ellipsis at 80
    columns.
14. **Fallback to plain is lazy.** `_Frame.__rich__` catches only frame-building errors,
    not errors while rich draws the returned renderable. The plain fallback switch runs
    only from the verbose poll or `print_final`, so a non-verbose degraded run shows
    "progress unavailable" until the end.
15. **Running clock format.** Running rows show `format_duration` (`31.0s`), not the
    plan's ticking `m:ss`.
16. **The log header records no argv or mode.** The default session factories pass
    neither. `UpdateProgressSession.set_header` never reaches the log sink, whose header
    is written before the mode is known.
17. **The completion snapshot is stale (fails CI on master).**
    `tests/completion/snapshots/cli_spec.json` lacks `update -v/--verbose`, so
    `tests/completion/test_snapshot.py` fails 2 tests. Regenerating it adds exactly that
    option plus the `update` description digest.
18. **Small cleanups.**
    - Unused `timezone` import in `log_sink.py`.
    - Unused `FanOutProgress.sinks` and `UpdateProgressSession.degraded` (no consumer
      anywhere).
    - Two identical `RunUvFn` Protocols, in `main/update_types.py` and
      `mode_switch/execute.py`.
    - `docs/plugins.md` says `-v` with `-j`/`-q` "only affects the log file". It has no
      effect, because the log always records everything.
    - The `sase update -j` bullet in `docs/plugins.md` still says `schema_version: 2`;
      `UPDATE_JSON_SCHEMA_VERSION` is 4.

## Phase: stream-and-backend-fixes — Bound streaming waits, fix CRLF, and fix backend step details

Independent of the other phases.

- **Bounded streaming (`src/sase/dev_update/stream_command.py`)** (defect 8):
  - The process group can outlive its leader. After the SIGTERM (timeout) or SIGINT
    (interrupt) grace period, always send SIGKILL to the whole group, ignoring
    `ProcessLookupError`. This happens even when the leader has already exited.
  - Join pump threads with a bounded timeout (a small module constant, about 2 s). Pump
    threads must be daemon threads, so one abandoned on a still-open pipe can't block
    interpreter exit. Return or raise with the output read so far.
  - When a `timeout` is given, enforce the same deadline while joining pumps after the
    leader exits normally. If lingering group members keep the pipes open past the
    deadline, handle it exactly like a timeout: SIGTERM, then SIGKILL the group, bounded
    join, and raise `subprocess.TimeoutExpired` with the partial output. This matches
    `subprocess.run(timeout=...)`. With `timeout=None`, keep waiting for EOF, as
    `subprocess.run` does.
  - The callback-raised `KeyboardInterrupt`/`SystemExit` path gets the same bounded
    group kill.
- **CRLF (defect 9):** strip one trailing `\r` from each raw line before the `\r`
  collapse, for both newline-terminated lines and the final unterminated tail.
  `"hello\r\n"` must deliver `hello`, and a redraw sequence like `"a\rb\r\n"` must still
  deliver `b`.
- **Prebuild stdin (defect 10):** pass `stdin=subprocess.DEVNULL` to `run_streaming` in
  the prebuild probe runner's streaming branch (`src/sase/dev_update/prebuild.py`).
- **Repair row (defect 11):** in `src/sase/dev_update/reconcile.py`, before starting
  `f"{step_id}:repair"`, declare it with
  `progress.declare((StepSpec(repair_step_id, repair_label, parent_id=step_id),))`. Then
  it renders as a child of the health check. The existing `declare` API already creates
  parented rows, so no protocol change is needed.
- **Stale details (defect 12):** in `src/sase/dev_update/roots.py`, when a merge child
  finishes `done` and `format_merge_detail` returns `None`, replace the transient
  `merging…` detail with a final one (e.g. `fast-forwarded`). When a merge or check
  child is finished as `skipped` or `failed`, never leave a
  `fetching…`/`checking…`/`merging…` detail behind: pass the real reason, or clear it.
  Only clear details in backend code that knows they are transient; the model's
  first-write-wins rule stays.
- **Completion snapshot (defect 17):** run `just sync-completion-spec` and commit the
  regenerated `tests/completion/snapshots/cli_spec.json`. The only expected drift is
  `update -v/--verbose` plus the `update` description digest. If other drift appears,
  investigate it instead of blindly committing it.
- **Tests (`tests/dev_update/test_stream_command.py`, the reconcile/progress tests in
  `tests/dev_update/test_progress.py`, `tests/completion/test_snapshot.py`):**
  - The leader exits while a backgrounded grandchild holds the pipes, with `timeout=1`:
    `TimeoutExpired` is raised within a few seconds and the grandchild is gone.
  - A SIGTERM-ignoring group member under a timeout: the call returns within timeout +
    grace + join bound, and the member is killed.
  - `KeyboardInterrupt` raised from the callback: the whole group, grandchild included,
    is gone afterwards.
  - CRLF lines are delivered intact.
  - The health-check repair row is a child of `reconcile:<i>` in a real `TimelineModel`
    snapshot, not just in recorded events.
  - Merge children never finish `done` with `merging…`.
  - Use real tiny `python -c`/`sh -c` children. Polling sleeps need a
    `# sase-test-wait: <reason>` pragma.

## Phase: timeline-renderer-fixes — Fix the timeline model, renderers, and session lifecycle

Independent of stream-and-backend-fixes. The handler phase depends on it.

- **`print_final` tears down Live first (defect 1, renderer half):**
  - `LiveTimelineRenderer.print_final` stops the `Live` region (transient clear) and
    stops its watcher thread before printing the static frame and `slowest:` footer to
    the console.
  - After that the renderer never draws again. `__exit__` stays idempotent.
  - `PlainTimelineRenderer.print_final` stops its ticker. After `print_final`, plain
    mode prints nothing more, including finish lines produced by the session's
    `finalize()` on exit (defect 5, plain half).
  - `UpdateProgressSession.print_final` is idempotent. A second call is a no-op, so a
    late Ctrl-C cannot print a second frame (defect 6).
- **Finalize before the final frame (defect 5):** `UpdateProgressSession.print_final`
  finalizes still-pending/still-running steps as `skipped` before rendering. The failure
  frame then shows `– skipped` rows, matching the original plan.
- **Precise interrupt marking (defect 6):**
  - Extend `finalize` with a keyword-only
    `status_for_running: StepStatus | None = None`. `None` means use
    `status_for_pending`.
  - Apply the keyword across the `UpdateProgress` Protocol, the null sink,
    `TimelineModel`, `FanOutProgress`, `UpdateLogSink`, and any recording fakes in
    tests.
  - Add a small session helper (e.g. `UpdateProgressSession.interrupt()`) that finalizes
    running steps as `interrupted` and pending steps as `skipped`, for the handlers to
    call.
- **Trailing rows (defect 4, model half):**
  - Add `trailing: bool = False` to `StepSpec`. Top-level trailing steps always sort
    after every non-trailing top-level step, in their own declaration order.
  - Steps declared or auto-appended later still land before the trailing block.
  - `TimelineModel.snapshot()` applies this order. Children are unaffected.
- **Verbose streaming beyond 200 lines (defect 7):**
  - `TimelineModel` keeps a per-step monotonically increasing count of lines ever
    appended, exposed on `StepSnapshot` (e.g. `lines_total`).
  - Both renderers track lines emitted per step by that counter, not by `len(tail)`.
  - When lines rolled off the 200-line tail between polls, print one dim
    `… N lines omitted (see full log)` marker before the retained new lines.
- **Robust fallback to plain (defect 14):**
  - The live frame renderable renders its panel to segments inside the `try` (e.g. by
    materializing `console.render(...)` in `__rich_console__`), so drawing errors
    degrade too.
  - The live watcher thread runs in all modes, not only verbose, and switches to the
    plain renderer within one poll interval of a failure.
- **Titles and clock (defects 13 and 15):**
  - At 80 columns, every built-in top-level step title (the longest is 35 characters)
    renders in full. Details stay the column that ellipsizes first, and titles still
    ellipsize, not wrap, at 40 columns.
  - Running rows show a ticking `m:ss` clock. Finished rows keep `format_duration`.
- **Log header (defect 16, sink half):** `UpdateLogSink` records the mode once it is
  known (e.g. a `mode: <mode>` line written by a new `set_mode`, called from
  `UpdateProgressSession.set_header`). Omit the header's empty `mode:` line when no mode
  was given at construction.
- **Cleanups (defect 18, code half):** delete `FanOutProgress.sinks`,
  `UpdateProgressSession.degraded`, and the unused `timezone` import.
- **Tests (`tests/update_progress/`):**
  - `print_final` on a live renderer with a shared `force_terminal` recording stream:
    after the final frame, text printed by a second `Console` on the same stream stays
    intact once the session exits. No cursor-movement or erase sequences appear after
    the final frame.
  - Plain mode prints nothing after `print_final`.
  - Pending rows render `– skipped` in failure frames.
  - `interrupt()` statuses: only the running step is `interrupted`.
  - A second `print_final` is a no-op.
  - Trailing ordering with late declarations and auto-appended rows.
  - Verbose output past 250 lines, including the omitted-lines marker.
  - A draw-time exception degrades to plain without verbose.
  - The 80-column title fit and the `m:ss` running clock.
  - The log sink `mode:` line.

## Phase: handler-wiring-fixes — Fix the update handlers, managed rows, and docs

Depends on timeline-renderer-fixes, which provides the teardown, finalize, interrupt,
and trailing APIs.

- **Stdout only after teardown (defect 1, handler half):**
  - Live update and mode switch: every path that prints to `out` does so only after
    `session.print_final()`, which now stops Live.
  - Dry run: restructure `handle_dry_run` so the transient timeline context exits, with
    the region torn down and no final frame, before the dry-run panel or JSON is
    printed. Failure lines printed while the region is up should go through the live
    console, or be printed after teardown. The dry-run panel stays the only persistent
    output.
- **Managed step output and rows (defects 2 and 3):**
  - `_ManagedPackageWatcher.sink` forwards every line to
    `progress.output("managed", stream, line)`. That gives the tail, failure expansion,
    `-v`, and the log.
  - Then it parses change lines. Package names are normalized with
    `normalize_distribution_name` for child ids (planned packages are declared with the
    same normalization).
  - A package not declared yet is declared under the `managed` parent before its row
    starts, so transitive dependencies render as children.
  - A `- name==old` line records the old version. The matching `+ name==new` line
    finishes the child `done` with detail `old → new` (or `new` when no old line was
    seen), matching the original plan's live `✓` rows.
  - `finish_all` normalizes names the same way and fills in rows the stream never
    touched.
- **Row order (defect 4, handler half):**
  - In `_prepare_live_update`, declare `managed` (with its planned children), `restart`,
    and `completions` with `trailing=True`, in that order. Dev rows (`check`, `merge`,
    `reconcile:*`) declared later then render above them, and mixed mode shows the dev
    rows before the uv row, matching execution order.
  - `handle_mode_switch` declares `restart` and `completions` as trailing rows before
    execution, so they show as pending during the switch.
- **Interrupts (defect 6):**
  - Every `KeyboardInterrupt` path (live pre-session, live in-session, mode switch) uses
    the new session interrupt helper, not `finalize("interrupted")`.
  - `_RunRef` records when the dev journal entry was already appended, and
    `_journal_interrupted` skips it then. A late Ctrl-C writes no second entry.
  - The idempotent `print_final` prevents a second frame.
  - A Ctrl-C at the mode-switch confirmation prompt (`_confirm_mode_switch`, before any
    session exists) prints `Interrupted` to stderr and returns 130 with no traceback,
    matching the epic's "survives Ctrl-C cleanly" goal.
- **Log argv (defect 16, handler half):** the default session factories in the live and
  mode-switch handlers pass the process argv (`sys.argv`) to `UpdateProgressSession`.
  The injectable `ProgressSessionFactory` signature stays unchanged. The mode reaches
  the log through `set_header`.
- **`RunUvFn` dedupe (defect 18):** keep one `RunUvFn` Protocol and import it at the
  other site without creating an import cycle, e.g. `main/update_types.py` importing the
  `mode_switch.execute` one, or both importing from a shared lower-level module. Keep
  `sase.ace.tui.session_proc_reporter`'s type-only import working.
- **Docs (`docs/plugins.md`, defect 18):**
  - Say `-v` has no effect with `-j`/`-q`, because the log always records full output.
  - Fix the `sase update -j` bullet's `schema_version: 2` to the current
    `UPDATE_JSON_SCHEMA_VERSION` (4).
  - Update the managed-install sample if the child-row rendering changed. Keep
    `docs/cli.md` consistent.
- **Tests (`tests/main/test_update_command_live_timeline.py`,
  `tests/main/test_update_command_switch_live.py`, and a dry-run test module):**
  - **Shared terminal.** Build `err` and `out` consoles on one shared `force_terminal`
    stream, run a successful managed update, a dev update, a mode switch, and a dev dry
    run with a fake runner whose steps take nonzero time, then assert:
    - the result panel, restart lines, and completion lines appear intact after the
      final frame;
    - no cursor-up or erase sequences follow the first stdout panel line;
    - dry run leaves no timeline frame behind.
  - **Managed rows and output.** Streamed uv output reaches the `managed` tail and the
    log file. Transitive `+ certifi==...` lines become children of `managed`. `-`/`+`
    pairs render `old → new`.
  - **Row order.** In mixed and dev modes, rows render in execution order: inspect,
    check, merge, reconcile, managed, restart, completions.
  - **Failures.** The failure frame shows `– skipped` rows and expands the failed step's
    tail. Plain mode prints nothing after the `Full log:` line.
  - **Interrupts.** A Ctrl-C mid-run marks only the running step `interrupted`. A Ctrl-C
    raised after the success `print_final` prints one frame and appends one journal
    entry.
  - **Log header.** The log contains `argv` and `mode`.

## Verification (every phase)

Run `just fmt`, then `just check` (prefer `sase tool run check`). Do not run
`just check-full`. No TUI PNG goldens are affected. After the handler phase, manually
run `sase update -n` in a real terminal and piped (`sase update -n 2>&1 | cat`). Confirm
the dry-run panel is intact and no stale timeline remains.
