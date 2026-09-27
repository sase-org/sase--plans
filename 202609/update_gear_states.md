---
tier: epic
title: Three-state updates gear (green updating, yellow restart queued, red last update
  failed)
goal: 'The gear inset at the left edge of the top bar''s `updates:` badge shows at
  most one gear and always tells the truth: green while an update proc runs, yellow
  while an installed update waits for this ACE''s own procs before restarting ACE
  and the SASE service, and red while the user''s most recent update (or update-planning)
  attempt has failed. Each gear has a tooltip that explains it and a click target
  that acts on it. The red state survives ACE restarts and crashes, and every ACE
  instance on the machine shows it.

  '
phases:
- id: yellow-gear
  title: Gear state model, three-hue palette, and the yellow restart-queued gear
  depends_on: []
  size: medium
  description: 'yellow-gear: add the pure gear-state model and precedence, the lime/yellow/red
    palette with contrast guards, and state-driven chip rendering. Make the tracked-restart
    wait publish a coalesced pending-restart record so the badge shows the yellow
    gear with a blocker tooltip and a click that opens Procs on the first blocker.
    Feature-flag restarts opt out.'
- id: attempt-journal
  title: Durable update-attempt journal
  depends_on: []
  size: medium
  description: 'attempt-journal: add a best-effort, flock-guarded, atomically written
    JSON journal under sase_home(). It holds pure reducers for attempt start and settle,
    dismissal, and interrupted-attempt reconciliation (owner liveness survives execv
    restarts). A revisioned view feeds the UI. No UI changes.'
- id: red-gear
  title: Red gear lifecycle and the failure report
  depends_on:
  - yellow-gear
  - attempt-journal
  size: medium
  description: 'red-gear: record every update-lane attempt through the journal. Session
    workers write start and settle markers in the worker thread before on_complete;
    durable update procs settle off-thread. Add an app mixin that applies revisioned
    views at startup, on the 10-minute tick, and after settles. Show the red gear
    with its tooltip, and open a new failure-report modal on click (u open Update,
    d dismiss, y copy).'
- id: panel-failure-row
  title: Update panel failure row and docs polish
  depends_on:
  - red-gear
  size: small
  description: 'panel-failure-row: surface the recorded failure as the first Update
    panel (,U) row. f opens the failure report, and d dismisses in place through a
    panel message. The open panel refreshes when the journal view changes, and the
    Update-panel docs describe the row.'
proposed_by: bbugyi200.apollo.2d
create_time: 2026-09-27 13:22:55
status: done
bead_id: sase-1bd
---

- **PROMPT:** [prompts/202609/update_gear_states.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/update_gear_states.md)
- **BEAD:** [sase-1bd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bd/README.md)

# Plan: Three-state updates gear

## Why and what exists today

The `updates:` top-bar group (`src/sase/ace/tui/widgets/updates_indicator.py`) is the
top bar's only deep chip: a moss surface (`UPDATES_SURFACE`) with lime ink. While an
update-lane proc runs, `UpdatesAvailableIndicator._build_content` puts a lime-filled
`⚙` inset at the badge's left edge. That inset comes from `update_gear_chip(True)` in
`src/sase/ace/tui/proc_gear_chips.py`. Its input is `ProcGearLanes.update_labels`.
`_update_proc_indicator` in `src/sase/ace/tui/actions/_proc_action_observer.py` computes
that value with `proc_gear_lanes()`, and `is_update_row()` in
`src/sase/ace/tui/_proc_observer_models.py` decides which rows count
(`UPDATE_PROC_TYPES` and the update exclusive scopes). Clicking the badge while it is
green opens the Procs tab on the oldest running update (`action_open_update_procs` in
`src/sase/ace/tui/actions/_base_admin.py`).

There are two gaps today:

1. **Update finished, restart still waiting.** When an update changes code, the
   completion handlers (`UpdateRunActionsMixin._on_scoped_update_complete`,
   `SaseUpdateProcMixin._handle_code_update_completion`, and the mode switch) call
   `restart_after_update_when_ready()` in `src/sase/ace/tui/update_restart.py`. That
   function polls once per second for up to 60 s, waiting for this ACE's
   `running_background_procs()` to drain. Then it calls
   `_restart_tui(restart_axe=True)`, which re-execs ACE and restarts the SASE service.
   During this wait the green gear is already gone, because the update worker has
   settled. The only signal is a one-shot toast.
2. **Failures leave nothing behind.** Update procs are session-local workers
   (`_submit_session_worker`), so they exist only in memory. When one completes, its row
   disappears from the projection. A failed plan or apply leaves only an error toast,
   and an update that crashes ACE mid-flight leaves nothing at all. The one piece of
   update state that survives a restart is `pending_update_toast.json`, written by
   `src/sase/ace/update_receipt.py`. This design reuses its atomic-write pattern.

## The design

### The contract (what the user sees)

| Gear   | Fill                                                    | Meaning                                                                                            | Starts                                                                       | Ends                                                                | Click                                              |
| ------ | ------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------- | -------------------------------------------------- |
| green  | lime `#AFFF87` (existing `UPDATES_ACCENT`)              | An update proc of this ACE session is running                                                      | Update-lane proc submitted                                                   | Proc settles                                                        | Procs tab on the oldest running update (unchanged) |
| yellow | amber-yellow `#FFC000`                                  | New code is installed; ACE is waiting for its own procs before restarting ACE and the SASE service | A tracked restart is deferred behind blockers                                | ACE re-execs (blockers drained, or the 60 s wait expired)           | Procs tab on the first blocking proc               |
| red    | red `#FF5F5F` (the codebase's established "failed" red) | The most recent update or update-planning attempt failed or was interrupted                        | An update-lane attempt settles with `success=False`, or ACE died mid-attempt | A later-started attempt succeeds, or the user dismisses the failure | The failure report modal                           |

**Precedence: green > yellow > red.** A resolver guarantees at most one gear, and each
state gets the slot when it is the most time-sensitive:

- Live work (green) always wins, so starting a retry while red immediately turns the
  gear green.
- A queued restart (yellow) lasts at most 60 s and is about to change the process, so it
  outranks a sticky failure. When an update partly fails but still changed code, the
  gear shows yellow during the wait and red after the restart. The yellow tooltip
  mentions the failure.
- Red is sticky and waits.

**Palette and form.** All three gears use the same `⚙` glyph, the same 3-cell width,
and the same bold `#1a1a1a` ink. They sit in the same place, so switching state never
shifts the badge. The fills step down in lightness as urgency rises: lime (relative
luminance ≈ 0.82), then yellow (≈ 0.59), then red (≈ 0.30). The states therefore stay
separable by lightness under red-green color-vision deficiency, matching the rule in
`update_accents.py`: hue is identity, lightness tells neighbors apart. Measured contrast
values:

- Ink contrast: ≥ 5.8 for every fill.
- Contrast against the moss surface: red 3.42, which clears the existing ≥ 3.0 guard.
- Adjacent-state ratios: lime to yellow 1.37, yellow to red 1.81.

`#FFC000` reads as yellow but is darker than the monitor-orange and unread-yellow chips,
so it doesn't collide with them. The monitor gear is `#FFAF5F` and is a standalone
counted chip in `procs:`, while the update gear is an uncounted inset inside the moss
chip.

**Tooltip copy.** Absolute times are used so a tooltip is never stale:

- Green (unchanged): `Update in progress: <label>` /
  `Click to watch it in the Procs tab.`
- Yellow: `New SASE code installed · restart queued` /
  `ACE and the SASE service restart once 2 procs finish: sync, mail.` /
  `If they are still running at 14:32:05, ACE restarts anyway.` /
  `Click to see what it is waiting on.`
  - Label lists are capped at 3 plus `+N more`.
  - When a recorded failure finished at or after the moment the restart was queued, add
    this line: `The update also reported a failure; details after the restart.`
- Red: `Last update failed · today at 14:32` / `<label>: <error>` /
  `Click for the failure report.`
  - An interrupted attempt reads `Last update was interrupted · today at 14:32` /
    `ACE exited before "<label>" finished; the install may be incomplete.`
- Every state keeps the existing availability sentence appended, as the green path does
  today.

### What counts as an update attempt

- **One predicate.** Every proc for which `is_update_row()` is true is an attempt. This
  is exactly the set that lights the green gear, and it includes planning
  (`update-preview`, stage `plan`); every other update proc type is stage `apply`. The
  gear's green and red states therefore can never disagree about what an "update" is.
- **Failure:** the proc settles with `success=False`. Results with `collision=True`
  never ran, so they are ignored.
- **Interrupted:** a session-worker attempt whose owning ACE process died before
  settling. This is detected from a durable in-flight marker, so an update that breaks
  the running environment still leaves red behind.
- **Clearing:**
  - A success clears the recorded failure only if the successful attempt _started at or
    after the failure finished_. In the normal sequential flow this means "the last
    attempt wins". It also keeps a concurrent success that began earlier from hiding a
    newer failure.
  - A successful re-plan counts as a successful attempt. This follows the request
    literally: planning is an attempt.
  - Dismissal is keyed by attempt id, so dismissing in one ACE can never clear a newer
    failure recorded by another.
  - An interrupted marker becomes red only if no attempt that started later has since
    succeeded.
- **Not attempts:**
  - background availability checks (the Update panel's `! check failed` chips already
    cover those)
  - submissions rejected as duplicates before a proc exists
  - CLI `sase update` runs
  - durable procs whose completion this ACE never observed

### Scope of each state

- **Green** stays scoped to this ACE session, unchanged.
- **Yellow** is per ACE instance and in memory only. It ends with the process, because
  the restart re-execs ACE.
- **Red** is per user and machine-wide, persisted in `sase_home()/update_attempts.json`.
  Other ACE instances converge at startup and on the existing 10-minute update-check
  tick.

### Non-goals and boundaries

- **Stale running code** (an editable checkout's HEAD moved) does not light yellow by
  itself. Nothing is queued in that case, and it keeps its toast and the Update panel's
  **Restart ACE** row (`x`). If the user chooses that restart and procs block it, the
  restart goes through the same tracked queue and the gear turns yellow.
- **Feature-flag restarts** (`src/sase/ace/tui/modals/feature_flags_pane.py`) are not
  updates and opt out of the yellow gear.
- **No animation or blinking.** The gear stays a calm, static inset.
- **Rust core boundary.** The journal is ACE presentation state for one frontend's
  badge, the same kind of state as `pending_update_toast.json`. It therefore lives in
  Python beside `update_receipt.py`. Reopen that choice, and promote the journal into
  `sase_core`, if a second frontend (CLI, web, editor) needs "last update failed".
- **Procs tab lanes are unchanged.** `procs_pane_render.py` and
  `procs_pane_selection.py` keep `UPDATE_GEAR_HUE`, which remains the green running hue.

### Cross-cutting rules for every phase

- **Perf rule 1** (`tui_perf.md`): no disk I/O, JSON parsing, or locking on the UI
  thread. Journal calls run only in worker threads, and their results reach the UI
  through `call_from_thread` or worker results.
- **Size and lint limits:** keep modules under the `toobig` limit, and don't add
  speculative public symbols (Symvision flags unused ones). Each phase adds only the API
  its own code uses.
- **Bindings:** the new bindings are modal-local `BINDINGS` (UpdatePanel,
  UpdateFailureModal). Confirm that `src/sase/default_config.yml` has no keymap entries
  for these modals. A search at planning time found none. Update it only if that turns
  out to be wrong.
- **Docs:** each phase updates `docs/ace.md` for what it lands. The relevant parts are
  the top-bar paragraph near "While SASE is updating itself", the "Automatic checks
  publish" paragraph in the updates section, and the Update-panel section.

## Phase yellow-gear: gear model, palette, and the yellow gear

1. **Accents.** In `src/sase/ace/tui/widgets/update_accents.py`, add
   `UPDATE_RESTART_ACCENT = "#FFC000"` and `UPDATE_FAILED_ACCENT = "#FF5F5F"` and export
   both. Rewrite the docstring paragraph about the gear to describe the three states,
   the precedence, and the lightness ladder.
2. **Pure gear module.** Add `src/sase/ace/tui/update_gear.py`:
   - `UpdateGearState = Literal["updating", "restart_pending", "failed"]`.
   - `@dataclass(frozen=True, slots=True) class PendingUpdateRestart` with these fields:
     - `blocker_labels: tuple[str, ...]`
     - `blocker_identities: tuple[str, ...]`, where each entry is
       `row.durable_proc_id or row.proc_id`
     - `queued_at: float`, the wall-clock epoch when the chain started
     - `restart_by: float`, the wall-clock epoch when the wait expires
   - `resolve_update_gear(*, updating: bool, restart_pending: bool, failed: bool) -> UpdateGearState | None`
     implements green > yellow > red.
   - `restart_pending_tooltip(pending) -> str` returns the yellow copy above. The
     local-time `HH:MM:SS` formatting for `restart_by` lives in a small private helper.
3. **Chip.** In `src/sase/ace/tui/proc_gear_chips.py`, change the signature to
   `update_gear_chip(state: UpdateGearState | None) -> Text`:
   - `None` returns an empty `Text`.
   - Every state renders `⚙` with `bold #1a1a1a on <hue>`, taking the hue from a
     `UPDATE_GEAR_HUES` mapping: `updating` → `UPDATES_ACCENT`, `restart_pending` →
     `UPDATE_RESTART_ACCENT`, `failed` → `UPDATE_FAILED_ACCENT`.
   - Keep `UPDATE_GEAR_HUE` as the green running hue for the Procs tab lanes.
   - Update the module docstring.
4. **Indicator.** In `UpdatesAvailableIndicator`:
   - Add `_pending_restart: PendingUpdateRestart | None` and a
     `set_restart_pending(pending)` method, which is a no-op when the value is equal.
   - Add a `gear_state` property. For now it resolves with `failed=False`.
   - `_build_content` takes `gear: UpdateGearState | None` instead of `running: bool`.
   - The tooltip uses `restart_pending_tooltip` plus the availability sentence when the
     gear is yellow.
   - `on_click` routes `updating` to `open_update_procs` and `restart_pending` to
     `open_restart_blockers`. Otherwise it keeps `CLICK_ACTION`, and it keeps the
     existing prevent/stop handling.
   - A gear-only badge with zero counts must stay visible, as green does today.
   - Move tooltip copy into `update_gear.py` so the widget stays small.
5. **Restart queue.** In `src/sase/ace/tui/update_restart.py`:
   - Add a keyword `track_pending: bool = True` to `restart_after_update_when_ready`,
     and forward it from `restart_after_update`.
   - **Publishing.** When a tracked call defers because of blockers, publish a
     `PendingUpdateRestart` through `getattr(app, "_set_pending_update_restart", None)`.
     Republish on every 1 s poll so the blocker list tracks reality; the indicator
     ignores equal values. Compute `queued_at` and `restart_by` once at the start of the
     chain, where `restart_by = time.time() + (deadline - time.monotonic())`, and carry
     them through the timer callbacks.
   - **Coalescing.** Keep at most one tracked chain per app, stored as a small chain
     record on the app with the latest message, purpose, deadline, `queued_at`, and
     `restart_by`. A second tracked `deferred=False` request while a chain is active
     updates the chain's message, emits the usual queued notice, and returns without
     scheduling a second timer chain. When the chain restarts, its final notice uses the
     latest message.
   - **Immediate restarts.** An immediate restart (no blockers on the first check) never
     publishes a pending record.
   - **Ending the chain.** When the chain proceeds to restart, leave the yellow record
     published, because the process is exiting. If the app has no callable
     `_restart_tui`, clear the pending record and drop the chain.
   - **Feature flags.** `feature_flags_pane.py` passes `track_pending=False`. Keep
     untracked chains exactly as they behave today. Every update caller keeps the
     default: the scoped update in `update_run.py`, the
     `_restart_running_code_when_ready` restart row in `_base_updates.py`, and the
     plugins-browser SASE, dev, and mode-switch updates.
6. **App wiring.**
   - Add `_set_pending_update_restart(pending)` to `BaseUpdateActionsMixin`
     (`src/sase/ace/tui/actions/_base_updates.py`). It stores
     `self._pending_update_restart` and pushes the value to `#updates-indicator`,
     guarding the query the way `_update_proc_indicator` does.
   - Initialize the attribute to `None` beside `_running_code_state` in
     `src/sase/ace/tui/actions/_state_init_runtime.py`.
   - Add `action_open_restart_blockers` next to `action_open_update_procs` in
     `_base_admin.py`, sharing one private helper that opens the Procs tab bookmarked on
     an identity. It targets the first blocker, or just opens Procs if there is none.
7. **Tests.**
   - `tests/ace/tui/test_proc_gear_chips.py`: one `⚙` chip per state with the right
     fill and dark ink, `None` renders empty, and every state has the same cell width.
   - `tests/ace/tui/test_top_bar_palette.py`: every gear fill has ≥ 4.5 contrast with
     the dark ink and ≥ 3.0 against `UPDATES_SURFACE`. Luminance strictly descends
     updating > restart_pending > failed, with each adjacent ratio ≥ 1.3.
   - `tests/test_updates_indicator.py`:
     - For all 8 combinations of (running, pending, failed-forced-false for now), the
       rendered body contains at most one `⚙` and follows the precedence.
     - A pending-only badge renders `updates:  ⚙ ` with zero counts.
     - The yellow tooltip lists blockers with `+N more` and the restart time.
     - Clicks route correctly.
   - `tests/ace/tui/test_update_restart.py`:
     - A deferred tracked request publishes a pending record, and it is refreshed on
       each poll as blockers change.
     - An immediate restart publishes nothing.
     - `track_pending=False` never publishes.
     - A second tracked request coalesces into one timer chain, and the restart uses the
       latest message.
     - The pending record is cleared when `_restart_tui` is missing.
   - Adjust `tests/ace/tui/test_feature_flags_pane.py` and
     `tests/ace/tui/test_top_bar_indicators.py` for the new signatures.
   - Add `test_updates_indicator_restart_pending_png_snapshot` to
     `tests/ace/tui/visual/test_ace_png_snapshots_updates_indicator.py`, mirroring the
     updating snapshot. Generate its golden with a targeted
     `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_updates_indicator.py`,
     through `/sase_monitor` if the run may be long, and look at the resulting PNG. If
     golden capture cannot run in the environment, drop the visual test and record a
     `PROPOSED FOLLOW-UP:` note.
8. **Docs.** Describe the yellow gear, its tooltip, and its click in `docs/ace.md`.

## Phase attempt-journal: durable update-attempt journal

This phase makes no UI changes. Split the work between a pure model module,
`src/sase/ace/_update_attempts_model.py`, and a thin I/O facade,
`src/sase/ace/update_attempts.py`. The facade's docstring must state that every call
blocks and belongs off the UI thread.

**Record** (JSON, `schema: 1`):

```json
{
  "schema": 1,
  "revision": 7,
  "last_success_started_at": 1790000000.0,
  "in_flight": [
    {
      "attempt_id": "…",
      "label": "update everything",
      "proc_type": "comprehensive-update",
      "stage": "apply",
      "started_at": 1790000000.0,
      "owner": {
        "pid": 4242,
        "identity": "<boot_id>:<start_ticks>",
        "instance_id": "…"
      }
    }
  ],
  "failure": {
    "attempt_id": "…",
    "label": "plan update",
    "proc_type": "update-preview",
    "stage": "plan",
    "started_at": 1790000000.0,
    "finished_at": 1790000012.5,
    "error": "…",
    "output_tail": "…",
    "interrupted": false
  }
}
```

**Types** (frozen, slotted dataclasses):

- `UpdateAttemptOwner(pid, identity, instance_id)`
- `UpdateAttempt(attempt_id, label, proc_type, stage, started_at)`, created by
  `new_update_attempt(*, label, proc_type, started_at=None)`. `stage` is `"plan"` iff
  `proc_type == "update-preview"`, and `attempt_id` is `uuid4().hex`.
- `UpdateFailure(attempt_id, label, proc_type, stage, started_at, finished_at, error, output_tail, interrupted)`
- `UpdateAttemptsView(revision: int, failure: UpdateFailure | None)`, the only thing the
  UI consumes.

**Pure reducers.** All of them take `now` explicitly:

- **Start:** adds or replaces the in-flight marker for the attempt id.
- **Settle:** removes the marker.
  - On success, it raises `last_success_started_at` to at least the attempt's
    `started_at`, and clears the failure iff
    `attempt.started_at >= failure.finished_at`.
  - On failure, it replaces the failure with the new one: the newest finish wins.
  - Settle is self-contained, so it works even if the start marker was never written.
- **Dismiss:** clears the failure only when the attempt id matches.
- **Reconcile(is_alive):** drops markers whose owner is dead. The newest dead marker
  becomes an `interrupted=True` failure with `finished_at=now` and the error
  `ACE exited before this update finished; the install may be incomplete.`, but only if
  its `started_at > last_success_started_at`.
- **Bounds:**
  - `error` is the first non-empty line of the error, capped at 300 chars with an
    ellipsis.
  - `output_tail` is the last 200 lines, capped at 16 KiB and trimmed on a line
    boundary.
  - `in_flight` is capped at 16 entries, dropping the oldest.

**Codec tolerance:**

- A missing file yields an empty record at revision 0.
- Malformed JSON or wrong types yield an empty view. The next successful write repairs
  the file.
- Unknown keys are ignored.
- A `schema` other than 1 yields an empty view, and writes are skipped so a newer
  writer's file is never clobbered.

**Facade:**

- **Paths.** The journal lives at `sase_home() / "update_attempts.json"`, with a sibling
  `.lock` file. A module-level `_UPDATE_ATTEMPTS_FILE: Path | None` override exists for
  tests, mirroring `update_receipt.py`.
- **Writes.** Each read-modify-write holds `fcntl.flock(LOCK_EX)` on the lock file. The
  wait is bounded: non-blocking retries for about 2 s, then give up. When the reduced
  record changed, the write bumps `revision` and uses tmp + fsync + `os.replace`.
- **Failure handling.** Never raise. On I/O failure, log at debug level and return the
  view of the in-memory reduced record, so the calling ACE still reflects its own
  outcome.
- **Owner identity.** `CURRENT_INSTANCE_ID` is a `uuid4().hex` made at import.
  `current_owner()` uses `os.getpid()`, `process_identity_token()`
  (`src/sase/core/process_identity.py`), and that id.
- **`owner_is_alive(owner)`** returns:
  - true when the instance id matches the current one;
  - false when the pid matches but the instance id does not, which is a previous
    incarnation (the restart uses `os.execv`, which keeps the pid);
  - false when `os.kill(pid, 0)` raises `ProcessLookupError`, and true on
    `PermissionError`;
  - otherwise the result of `process_identity_matches(pid, identity)`.
- **Public API:**
  - `begin_update_attempt(attempt)` returns the view.
  - `settle_update_attempt(attempt, *, success, error, output)` returns the view.
  - `dismiss_update_failure(attempt_id)` returns the view.
  - `load_update_attempts()` reconciles under the lock, writes only if something
    changed, and returns the view.
  - `new_update_attempt(...)`.

**Tests** go in `tests/ace/test_update_attempts.py` and
`tests/ace/test_update_attempts_model.py`:

- the reducer matrix:
  - a sequential fail then success clears;
  - a success that started before the failure finished keeps red;
  - a newer failure replaces an older one;
  - dismissal with a matching id clears, and a stale id does not;
  - the plan stage mapping;
  - bounds trimming;
  - revision bumps only on change.
- reconciliation:
  - a dead owner becomes interrupted;
  - a live foreign owner is kept;
  - this instance's own marker is kept;
  - the same pid with a different instance id counts as dead;
  - a later-started success suppresses the interrupted failure.
- I/O:
  - an atomic write round-trips;
  - a malformed file reads as empty and the next write repairs it;
  - a foreign schema is never overwritten;
  - lock timeout degrades to the in-memory view;
  - writes land under the isolated `SASE_HOME`.

## Phase red-gear: lifecycle wiring, the red gear, and the failure report

1. **Tracking adapter.** Add `src/sase/ace/tui/update_attempt_tracking.py`:
   - `update_attempt_for(proc_info) -> UpdateAttempt | None` returns an attempt iff
     `is_update_row(proc_info)`. It is pure and safe on the UI thread. It takes
     `label=proc_info.label` and `started_at=proc_info.started_at.timestamp()`.
   - A settle helper maps a `TrackedProcResult` to the journal. It skips `collision`,
     and `error` is `result.error or result.message`.
2. **Session workers.** Change `_proc_action_submission.py`, `_proc_action_types.py`,
   and `_proc_action_completion.py`:
   - **Submit:** `_submit_session_worker` computes the attempt on the UI thread and
     keeps it in `self._session_update_attempts[proc_id]`.
   - **Worker thread:** `_wrapped()` calls `begin_update_attempt` before the body. After
     the body returns, and also in its exception branch, it calls
     `settle_update_attempt` with `proc_info.get_live_output()`. The view is returned
     via a new optional `SessionWorkerResult.update_attempts` field. This ordering is
     the reliability guarantee: the outcome is durable before the UI-thread
     `on_complete` runs, so an update that immediately re-execs ACE has already recorded
     itself.
   - **Completion:** `_on_session_worker_completed` pops the attempt and applies
     `result.update_attempts` _before_ calling `on_complete`. The red, yellow, and green
     transitions then land in one UI callback with no flicker.
   - **Worker error:** `_on_session_worker_error`, where the worker died outside the
     body wrapper, pops the attempt and schedules an off-thread settle as a failure with
     the worker error.
3. **Durable update procs** (`plugin.update`). In `_deliver_tracked_completion`, before
   `on_complete`, compute the attempt from the delivered row. Unless the result is a
   collision, schedule an off-thread settle that applies the returned view through
   `call_from_thread`. Durable procs get no in-flight marker, because they outlive ACE.
   Record this limitation in a comment: if ACE restarts mid-run, that attempt is never
   recorded.
4. **App state mixin.** Add `src/sase/ace/tui/actions/_update_attempt_state.py` with
   `UpdateAttemptStateMixin`, and register it in `src/sase/ace/tui/app.py` beside
   `UpdateRunActionsMixin`. Initialize its attributes in `_state_init_runtime.py`.
   - `_apply_update_attempts_view(view)`: drop the view when `view.revision` is below
     the current revision, since an older read must not undo a newer settle. Otherwise
     store it and call `indicator.set_last_failure(view.failure)`.
   - `_schedule_update_attempts_refresh()`: a coalesced thread worker in group
     `startup-loads`, with an in-flight flag released in `finally`, mirroring
     `_schedule_automatic_update_check`. It calls `load_update_attempts()` and applies
     the result via `call_from_thread`. Call it from `_startup_loads_maintenance.py`
     next to `_schedule_startup_update_toast_check()`, and from
     `UpdateToastMixin._on_periodic_update_check`.
   - `_dismiss_update_failure(attempt_id)`: first apply an optimistic view with the same
     revision and no failure, then persist off-thread and apply the written view.
   - `action_open_update_failure()`: push `UpdateFailureModal(failure)`. On the result,
     `"dismiss"` calls `_dismiss_update_failure`, and `"open_update"` calls
     `action_update_sase_shortcut()`.
5. **Indicator.** Add `set_last_failure(failure)` to `UpdatesAvailableIndicator`:
   - `gear_state` now passes `failed=failure is not None`, and a click on a red gear
     runs `open_update_failure`.
   - The red tooltip, and the yellow tooltip's failure line (a failure whose
     `finished_at >= pending.queued_at`), live in `update_gear.py`.
   - Add a date-aware formatter to `update_gear.py` that returns `today at 14:32`,
     `yesterday at 09:10`, or `Sep 26 at 14:32`.
6. **Failure report.** Add `src/sase/ace/tui/modals/update_failure_modal.py`, a
   `ModalScreen[Literal["dismiss", "open_update"] | None]`:
   - **Frame:** a border in `UPDATE_FAILED_ACCENT` titled `✗ Update failed`, or
     `✗ Update interrupted` for interrupted attempts.
   - **Header:** a dim meta line `<label> · plan|apply · failed today at 14:32`,
     followed by the bold red error line. Interrupted attempts get the explanation copy
     instead.
   - **Output:** a dim `last output` divider, then a `VerticalScroll` of `output_tail`
     rendered as plain `Text` so markup in the output is never interpreted. With no
     output, show `No output was captured.`
   - **Keys:** the footer reads `u open Update · d dismiss · y copy · q close`.
     Bindings: `u`, `d`, `y`, `escape`/`q`, `ctrl+d`/`ctrl+u`, and `g`/`G`. `y` copies
     the error and output through `schedule_copy_delivery`, as `GateActionOutputModal`
     does.
   - **Styles:** follow the existing modal CSS conventions (check where
     `GateActionOutputModal` styles live).
7. **Tests.**
   - Adapter: non-update rows return `None`, the stage mapping is right, and collisions
     are skipped.
   - Session workers (`tests/ace/tui/test_proc_actions_session_workers.py`), using a
     fake journal:
     - begin is written before the body, and settle before `on_complete`;
     - a failing body yields a red view, and a later success clears it;
     - non-update workers never touch the journal;
     - the worker-error path settles a failure.
   - Durable delivery schedules a settle for `plugin.update` rows.
   - Mixin:
     - the revision guard drops stale views;
     - the startup and periodic refreshes apply views;
     - dismissal is optimistic and then persisted.
   - Indicator:
     - the full 8-state precedence matrix, now including `failed`;
     - a red badge with zero counts stays visible;
     - the failed and interrupted tooltip copy;
     - the yellow failure line;
     - a click on red opens the report.
   - Modal:
     - it renders the meta line, error, and output, and empty output shows its
       placeholder;
     - markup-like output renders literally;
     - `d`, `u`, and `q` return the right results.
   - Add a red-state PNG snapshot, using the same targeted golden procedure and fallback
     as yellow-gear.
8. **Docs.** Describe the red gear in `docs/ace.md`: what counts as a failure, the
   interrupted case, the clearing rules, the report and its keys, and the
   `update_attempts.json` file.

## Phase panel-failure-row: Update panel integration

1. **Panel state.** In `src/sase/ace/tui/update_panel_state.py`:
   - `build_update_panel_state(..., last_failure: UpdateFailure | None = None)` adds a
     `"failure"` row first, above **Restart ACE**, when a failure exists.
   - Row fields: key `f`, title `Last update failed` (or `Last update interrupted`),
     description `Open the failure report · d dismisses it.`, and one detail line
     `<label>: <error>` truncated to the row width. The row uses the
     `UPDATE_FAILED_ACCENT` accent.
   - Add a new chip kind `"update_failed"` with the text `✗ failed <HH:MM>`, or
     `✗ failed <Mon DD>` when the failure isn't from today.
2. **Panel modal.** In `src/sase/ace/tui/modals/update_panel.py`:
   - `f`/`F` choose the failure row. This mirrors how `x`/`X` both choose the restart
     row.
   - `d` posts a new `UpdatePanel.DismissFailureRequested(attempt_id)` message only when
     the failure row exists, and does nothing otherwise. The panel stays open.
   - `_chip_style` renders `update_failed` as `bold UPDATE_FAILED_ACCENT`.
   - The hint text gains `d dismiss failure` only while the row exists.
3. **App.**
   - `action_update_sase_shortcut` and `_refresh_open_update_panel` pass
     `last_failure=self._update_attempts_view.failure`, guarding against `None`.
   - `on_result` handles `"failure"` by calling `action_open_update_failure()`.
   - An `on_update_panel_dismiss_failure_requested` handler, modeled on
     `on_update_panel_recheck_requested`, dismisses and then refreshes the open panel.
   - `_apply_update_attempts_view` also calls `_refresh_open_update_panel()`. That call
     is a pure projection and costs nothing when the panel isn't open.
4. **Tests.**
   - `tests/ace/tui/test_update_panel_shortcut.py` plus the panel-state tests:
     - the row is present or absent as expected, and ordered first;
     - the chip copy is right for today and older failures;
     - `f` opens the report;
     - `d` posts a message only when there is a failure;
     - the hint text toggles.
   - A panel refresh after dismissal removes the row.
5. **Docs.** Add the failure row and its `f` and `d` keys to the Update-panel section of
   `docs/ace.md`.

## Verification (every phase)

- Run `sase tool run check`. Do not run `just check-full`.
- For visual goldens, use only the targeted `just fix-tui-screenshots -- <file>` run
  described in the phases, and inspect the PNG.
- Re-read the tooltip copy in a real widget render (the unit tests assert it) and keep
  the `⚙` inset 3 cells wide in every state.
