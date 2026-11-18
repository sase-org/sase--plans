---
tier: tale
title: TUI updates wait for an in-flight epic launch instead of failing
goal:
  SASE updates started from the TUI (,U, ,E, Admin Center Updates tab) queue behind a
  running `sase bead work` (an approved epic launch) and proceed automatically once it
  releases the code-swap lock, with a visible waiting phase and a bounded timeout,
  instead of refusing with a "Re-run the update after it finishes" toast.
size: medium
proposed_by: bbugyi200.athena.1e1
proposed_with: claude/opus
create_time: 2026-11-18 11:23:45
status: wip
---

# Plan: TUI updates wait for an in-flight epic launch instead of failing

## Problem

Every SASE TUI update entry point refuses to run while a `sase bead work` holds the
code-swap reader lock. That includes the leader `,U` Update panel, `,E` Update
everything, and the Admin Center Updates tab. Approved epics launch through exactly that
kind of reader: the guarded epic-launch bootstrap hands its shared lock to
`sase bead work <plan.md> --yes-to-all ...`, which holds it for the whole launch
orchestration. So pressing `,U` mid-launch produces an error toast:

```text
A sase bead work is running against this checkout (sase bead work (pid 21121); running
`sase bead work ~/.sase/plans/202611/record_walk_ring.md --yes-to-all ...`; started ...).
Re-run the update after it finishes.
```

The user then has to watch for the launch to end and retry by hand. They want the update
to queue and wait for the launch to finish. That matches how ACE already treats TUI
procs before an update restart: `restart_after_update_when_ready` shows "restart queued
until N TUI tasks finish" and proceeds once they clear.

### Where the refusal comes from today

- **Preview gate.**
  `src/sase/ace/tui/modals/plugins_browser_dev_update.py::dev_update_blocking_reason`
  returns the toast above whenever `code_swap_readers_active()` finds a live blocking
  reader. It has three callers:
  - `plugins_browser_comprehensive_update_preview._plan_sase_leg` (`,U` / `,E`): this
    turns the leg into `sase_blocker`, so the preview is not runnable and
    `handle_comprehensive_noop` toasts it as an error. This is the screencast case.
  - `plugins_browser_sase_update._on_sase_update_preview` (Admin Center "update sase").
  - `plugins_browser_update._on_dev_update_preview` (Admin Center editable plugin
    update).
- **Execution gate.** `src/sase/dev_update/execute.py::execute_dev_update` takes
  `code_swap_writer_lock()` with a non-blocking `LOCK_EX`. If any reader holds the lock,
  it returns a failed result whose reason starts with `deferred:`. Every TUI execution
  path goes through `plugins_browser_dev_update.execute_tui_dev_update(plan, run=...)`,
  which calls `execute_dev_update`. The callers are the comprehensive
  `_perform_sase_leg`, and `_submit_dev_update_proc` and `_submit_combined_update_proc`
  via `plugins_browser_operations._execute_dev_update`.

The documented reason for failing fast (`docs/reference/integrations/plugins-update.md`,
"The code-swap lock") is: "a waiting writer would stall sase's TUI". That concern does
not apply if the wait runs inside the update's existing background session worker and
polls the lock rather than blocking in `flock`.

## Goal

When a TUI-initiated SASE update finds a blocking code-swap reader:

1. **Planning does not refuse.** The confirm preview says the update will wait, and
   names the holder.
2. **Execution waits.** The update proc waits in its worker thread, with a visible
   `Waiting for <holder> to finish` phase, until the reader releases. Then it
   fast-forwards and reconciles as usual.
3. **The wait is bounded.** It times out after 30 minutes. On timeout it records a
   `deferred:` failure without having changed anything, the same safe outcome as today,
   only later.

The CLI `sase update` path is explicitly out of scope and stays fail-fast. It passes no
wait policy, so its behavior and output are byte-for-byte unchanged.

## Design

### 1. Bounded, polling writer wait in the lock module

In `src/sase/dev_update/code_swap_lock.py` (562 lines today; the `toobig` warn line is
700):

- Add a public frozen dataclass `CodeSwapWait` with these fields:
  - `timeout_seconds: float`
  - `poll_seconds: float = 1.0`
  - `on_wait: Callable[[str], None] | None = None`, which receives a human description
    of the current blocker
  - `cancel_check: Callable[[], bool] | None = None`
  - injectable `sleep` (default `time.sleep`) and `clock` (default `time.monotonic`) for
    tests
- Extend `code_swap_writer_lock()` with an optional `wait: CodeSwapWait | None = None`
  and a way to share a deadline with the pre-wait in §2, such as an optional absolute
  `deadline`.
  - **With `wait=None`, behavior is exactly today's.** It makes one non-blocking
    attempt.
  - **With a wait policy, it retries the non-blocking `LOCK_EX | LOCK_NB` attempt every
    `poll_seconds`.** Never use a blocking `flock`. The waiting writer must hold nothing
    between attempts, so readers are never refused because of it, and timeout and
    cancellation stay possible.
  - **A legacy reader counts as a failed attempt.** When `_legacy_reader_blocked_by()`
    reports a live legacy reader after `LOCK_EX` succeeds, unlock and keep waiting
    instead of yielding `acquired=False`.
  - **Stop conditions.** Stop when the deadline passes or `cancel_check()` is true.
    Yield `acquired=False` with the last `blocked_by` and a field that says why the wait
    stopped: timeout or cancelled.
  - **Report waiting to the caller.** Add `waited: bool` to `_CodeSwapLockResult`. It is
    true when at least one attempt was refused before the lock was acquired, so the
    caller can re-validate.
  - **Call `on_wait` sparingly.** Call it on the first refused attempt and again only
    when the blocker description changes, never once per poll.
- Add a holder-file pre-wait helper, for example
  `wait_for_code_swap_readers(wait, *, deadline) -> outcome`. It polls
  `code_swap_readers_active()` until no live blocking readers remain, or until timeout
  or cancellation. It returns which of those happened plus the last blocker description,
  and it uses the same `on_wait` semantics.
  - **Advisory readers never cause a wait.** These are agent runners, and
    `code_swap_readers_active()` already excludes them.
  - **`SASE_DISABLE_CODE_SWAP_LOCK=1` makes it a no-op,** like the rest of the module.
- Update the module docstring. Writers stay non-blocking for direct callers. A caller
  that passes a `CodeSwapWait` (the TUI) polls with a bound and holds nothing while
  waiting.

### 2. `execute_dev_update(..., wait: CodeSwapWait | None = None)`

In `src/sase/dev_update/execute.py`:

- **Default: unchanged.** With `wait=None` the flow and all messages stay the same, so
  the CLI is untouched.
- **With a wait policy:**
  1. **Compute one shared deadline,** `clock() + wait.timeout_seconds`, so the total
     wait never exceeds the timeout.
  2. **Pre-wait first.** Before `_declare_plan_steps` and the fetch, run the holder-file
     pre-wait. The fetch then happens after the epic launch finishes and picks up
     anything that landed meanwhile. If the pre-wait times out or is cancelled, return a
     `failed_result(..., changed=False)` with no git commands run.
  3. **Fetch and preflight as today.**
  4. **Take the lock with the wait.** Call
     `code_swap_writer_lock(wait=wait, deadline=...)`. This is the authoritative wait:
     it covers the race where a new reader starts after the pre-wait, and readers whose
     holder record is missing.
  5. **Re-validate after waiting.** If `lock.acquired` and `lock.waited`, re-run
     `preflight_actionable_roots` while holding the lock and before merging. The tree
     may have changed while you waited. A failure returns a failed result with
     `changed=False`.
- **Deferral wording.** The reason keeps the `deferred:` prefix, and the TUI failure
  toast shows it.
  - Timeout:
    `deferred: <holder> was still running against this checkout after waiting 30m; re-run the update when it finishes`.
  - Cancelled: `deferred: update cancelled while waiting for <holder>`.
  - The immediate (no-wait) message stays as-is.
- `execute.py` is 207 lines; keep the new branch compact. Format durations locally;
  don't import TUI or plugin rendering helpers into `sase.dev_update`.

### 3. TUI adapter: always wait, with a visible phase

In `src/sase/ace/tui/modals/plugins_browser_dev_update.py`:

- **Timeout constant.** Add `TUI_CODE_SWAP_WAIT_TIMEOUT_SECONDS = 30 * 60`.
- **Policy factory.** Add
  `tui_code_swap_wait(*, on_wait=None, cancel_check=None) -> CodeSwapWait` using that
  timeout.
- **Executor waits by default.** `execute_tui_dev_update(plan, *, run=None, wait=None)`
  passes `wait=wait or tui_code_swap_wait()`, so every TUI execution waits even when a
  caller passes no reporter-wired policy.
- **Readers no longer block planning.** In `dev_update_blocking_reason`, remove the
  `code_swap_readers_active()` clause and update its docstring. Readers are now a wait,
  not a blocker. Drop the `code_swap_readers_active` import if it becomes unused there.
- **Wait notice.** Add `dev_update_wait_notice(plan: DevUpdatePlan) -> str | None`.
  - It returns `None` when the plan won't take the writer lock, meaning no
    `actionable_roots` and no `reconcile_steps`, mirroring `execute_dev_update`'s no-op
    check.
  - It also returns `None` when `code_swap_readers_active()` is `None`.
  - Otherwise it returns a line like
    `Waits up to 30m for <holder> to finish before swapping checkouts.`
- **Show the notice in previews.** Have `dev_update_preview_details(plan)` include the
  notice, before the advisory-runner line. Both Admin Center confirm modals,
  `plugins_browser_update._open_dev_update_modal` and
  `plugins_browser_sase_update._open_sase_dev_update_modal`, then show it with no
  further changes.
- **Exports.** Update `__all__`.

In `src/sase/ace/tui/session_proc_reporter.py`:

- Add `SessionProcReporter.code_swap_wait() -> CodeSwapWait`.
  - It lazily imports `tui_code_swap_wait`, following the lazy-import style of
    `dev_command_runner`, to keep the TUI import budget.
  - Its `on_wait` calls `self.phase(f"Waiting for {holder} to finish")`. That updates
    the proc row phase and appends a `==> ...` progress line.
  - Its `cancel_check` is `self.cancel_event.is_set`.

Wire the reporter policy into every TUI executor call:

- **Comprehensive leg.** In
  `plugins_browser_comprehensive_update_execution._perform_sase_leg`, pass
  `wait=reporter.code_swap_wait()` when a reporter is present.
- **Admin Center executor.** In
  `plugins_browser_operations._execute_dev_update(plan, *, run=None, wait=None)`,
  forward `wait` only when the executor accepts it, using
  `callable_accepts_keyword(execute, "wait")` like the existing `run` guard. Existing
  test fakes such as `lambda plan_arg: ...` keep working.
- **Admin Center procs.** `plugins_browser_sase_update_procs._submit_dev_update_proc`
  and `_submit_combined_update_proc` pass `wait=reporter.code_swap_wait()`.
- **Catch stragglers.** Grep for any other `_execute_tui_dev_update` or
  `_execute_dev_update(` callers and wire them the same way.

Thread safety: the wait runs inside Textual thread workers that `_submit_session_worker`
already uses. Nothing new touches the UI thread. ACE quits and restarts through
`os._exit` / `os.execv`, so a waiting worker cannot hang exit. The quit confirmation
already counts it as an interrupted session task, and nothing has been mutated while
waiting.

### 4. Queue notice in the `,U` / `,E` flow

- **New preview field.** In `plugins_browser_comprehensive_update_models.py`, add
  `sase_wait_notice: str | None = None` to `ComprehensiveUpdatePreview`.
- **Compute it while planning.** In `plugins_browser_comprehensive_update_preview.py`
  (559 lines), compute it in `_plan_sase_leg` (worker thread) with
  `dev_update_wait_notice(plan)` for runnable dev plans, and thread it into the preview.
  How you extend the return shape is up to you.
- **Confirm modal.** Have `_sase_preview_section` add the notice to the section's
  `details`, so the confirm modal states that the update will wait.
- **Queue toast.** In
  `src/sase/ace/tui/actions/update_run.py::_submit_scoped_update_task`, toast after a
  successful submit when the SASE leg is selected and `preview.sase_wait_notice` is set.
  Use `information` severity, worded like
  `SASE update queued — waiting for <holder> to finish before swapping checkouts.` This
  covers the auto-approved `,E` path, which has no modal. Use only preview data, with no
  new I/O on the UI thread.

### 5. Docs

Read the `docs_writing.md` reference memory first. Then, in
`docs/reference/integrations/plugins-update.md` "The code-swap lock":

- **Replace the Admin Center refusal text** (the "disables the update instead" block)
  with the new behavior. TUI updates (`,U`, `,E`, and the Admin Center Updates tab) show
  a wait notice, then wait up to 30 minutes in their background proc for blocking
  readers such as an approved epic launch. The proc phase reads
  `Waiting for <holder> to finish`. After the timeout they defer with nothing changed.
- **Rewrite the "Ordinary readers and the writer remain non-blocking" paragraph.**
  Direct readers and `sase update` stay fail-fast. The TUI writer polls with a bound
  from a background worker, so it never stalls the UI and never holds the lock while
  waiting.

No keymap, config, or help-modal changes are needed. `,U` and `,E` keep their bindings
and labels, so `default_config.yml` is untouched.

## Tests

Put new tests in new focused modules. `tests/dev_update/test_execute.py` (415) and
`tests/ace/tui/test_comprehensive_update_preview.py` (477) are near the split threshold.

- **`tests/dev_update/test_code_swap_lock_writer_wait.py` (new).** Use an in-process
  reader held through `code_swap_reader_lock` or a raw `LOCK_SH` fd, plus injected
  `sleep` and `clock`. Cover these cases:
  - The waiting writer acquires after the reader releases, for example from inside the
    fake `sleep`, and reports `waited=True`.
  - A deadline timeout yields `acquired=False` with timeout as the stop reason and the
    holder in `blocked_by`.
  - `cancel_check` stops the wait.
  - `on_wait` fires once per distinct blocker, not per poll.
  - While the writer waits, a new `code_swap_reader_lock` can still be acquired. This
    proves nothing is held between attempts.
  - `wait=None` is unchanged: an immediate `acquired=False`.
  - The pre-wait helper clears, times out, ignores advisory holders, and is a no-op
    under `SASE_DISABLE_CODE_SWAP_LOCK=1`.
- **`tests/dev_update/test_execute_code_swap_wait.py` (new).** Reuse
  `tests/dev_update/_execute_helpers.py`.
  - A reader active at start, then released, leads to a successful merge and reconcile.
  - A pre-wait timeout returns a `deferred:` failure with `changed=False` and no `git`
    commands recorded.
  - A holder-less `LOCK_SH` (the race path) that releases during the lock wait causes
    preflight `git status` to run twice before the merge.
  - The existing no-wait deferral test in `test_execute.py` still passes unchanged.
- **TUI tests.**
  - Update
    `tests/ace/tui/test_plugins_browser_pane_sase_update_dev.py::test_dev_update_blocking_reason_reports_active_bead_work`.
    With a live reader, `dev_update_blocking_reason` is now `None`.
    `dev_update_wait_notice` names `sase bead work` and `plan.md`, and
    `dev_update_preview_details` includes it. A no-op plan yields no notice.
  - New `tests/ace/tui/test_comprehensive_update_wait_notice.py`:
    - With a live reader, the comprehensive preview is runnable, `sase_blocker` is
      `None`, `sase_wait_notice` is set, and `_sase_preview_section(...).details`
      contains it.
    - `_submit_scoped_update_task` emits the queue toast only when the notice is set.
    - `SessionProcReporter.code_swap_wait()` sets the phase on `on_wait`, and its
      `cancel_check` mirrors `cancel_event`.
  - Extend the runner-wiring tests to assert that `_perform_sase_leg` passes a `wait`
    kwarg alongside `run`. These are in
    `tests/ace/tui/test_plugins_browser_git_lock_runner_wiring.py` and
    `test_plugins_browser_pane_comprehensive_update_execution.py`.
  - Assert that `plugins_browser_operations._execute_dev_update` forwards `wait` only to
    executors that accept it.

## Verification

- Read the `lint_and_test.md` reference memory.
- Run `sase tool run check`. Do not run `check-full` unless explicitly asked.
- Keep every touched source file under the 700-line `toobig` warning. Keep symvision
  clean: every new public symbol is used and exported deliberately.
- Optional manual check, for a human with a dev install. Hold a fake blocking reader for
  about 90 seconds:

  ```bash
  python -c 'import time; from sase.dev_update.code_swap_lock import code_swap_reader_lock as r
  with r(op="bead.work", command=["sase","bead","work","demo.md"]): time.sleep(90)'
  ```

  While it runs, press `,U` and choose SASE. Confirm three things: the modal shows the
  wait line, the proc row shows `Waiting for sase bead work ...`, and the update
  proceeds once the reader exits.

## Non-goals

- **CLI `sase update`** keeps today's immediate `deferred:` result.
- **Direct `sase bead work` readers** stay fail-fast when a swap is in progress.
- **The guarded epic-launch bootstrap** is unchanged.
- **No new gear or indicator state.** The waiting update is an ordinary running update
  proc whose phase names the wait.
- **No change to advisory agent-runner semantics.** They never block or delay a swap.
