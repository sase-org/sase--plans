---
tier: tale
title: Load the `,x` relaunch prompt without waiting for cleanup
goal:
  Pressing `,x` mounts the killed agent's rewritten prompt immediately, while the
  replacement launch still waits for the kill/dismiss persistence proc to settle.
size: medium
proposed_by: bbugyi200.athena.0de
create_time: 2026-08-25 08:36:16
status: wip
---

# Plan: Make `,x` load the relaunch prompt immediately

## Problem

Pressing `,x` (kill & edit) on the Agents tab now has a noticeable pause between the
agent disappearing from the list and its prompt appearing in the prompt input widget.

The pause was introduced deliberately by `1b2381366` ("fix(ace): make family-root
kill-and-edit name reuse deterministic"), which changed both `,x` branches to defer
mounting the prompt bar/stack until the kill/dismiss persistence proc settles:

- `src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py:271`
  (`_finish_kill_and_edit_agent`) passes `on_settled=mount_prompt_bar` to
  `_dismiss_done_agent` / `_do_kill_agent`.
- `src/sase/ace/tui/actions/agents/_marking_kill.py:125`
  (`_bulk_kill_marked_agents_and_edit`) passes `on_settled=mount_prompt_stack` to
  `_do_bulk_kill_agents`.

That callback only fires after this whole chain completes:

1. `_submit_cleanup_proc` → `submit_agent_cleanup` → `_submit_durable_proc` submits
   `python -m sase agent persist-cleanup` to the durable proc supervisor from a thread
   worker.
2. The supervisor spawns a fresh Python interpreter. Measured in this workspace,
   `python -m sase --version` alone costs ~0.42s before any cleanup work runs.
3. The cleanup transaction runs (bundle save, dismissed-index sync, notification
   dismissal, workspace release).
4. `ProcObserver` notices the terminal row on its next poll — `POLL_SECONDS = 0.5` in
   `src/sase/ace/tui/proc_observer.py:55`, so up to another half second.
5. `_deliver_tracked_completion` finally invokes `on_settled`
   (`src/sase/ace/tui/actions/_proc_action_completion.py:263`).

The user pays that entire round trip (roughly 1.5–3s) before they can even see the
prompt they asked to edit.

## What the fix actually has to protect

The commit message states the invariant precisely: a late bundle write from the old
cleanup must not "resurrect a name a replacement agent is about to reuse".

Concretely:

- `,x` rewrites the stored prompt into a forced-name-reuse prompt (`%id:!name`) via
  `prepare_kill_and_edit_prompt` (`src/sase/agent/relaunch_prompt.py:101`).
- When that prompt is launched, the child `sase run` runs `apply_force_reuse_launch` →
  `wipe_names_for_forced_reuse` → `wipe_force_reuse_owner`
  (`src/sase/agent/names/_forced_reuse.py:21`), which deletes the old owner's artifact
  dirs, dismissed bundles, index rows, and registry entries.
- The cleanup proc writes a dismissed bundle for the killed agent
  (`persist_cleanup_side_effect_intents` → `save_dismissed_bundle`,
  `src/sase/ace/tui/actions/agents/_dismiss_persistence.py:86`).
- `rebuild_name_registry` (`src/sase/agent/names/_registry.py:406`) re-derives
  reservations from artifact dirs **and dismissed bundles**. So a bundle write that
  lands _after_ the wipe re-registers the name the replacement just claimed.

The hazardous operation is therefore the **launch**, not the mount. Mounting the prompt
bar writes nothing that the name registry reads: `_mount_edit_relaunch_prompt_bar`
(`src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py:396`) only unmounts any
prior bar, builds a `PromptContext`, and mounts a `PromptInputBar` widget.

Gating the mount is a _proxy_ for gating the launch — it is strictly earlier than
necessary, and it makes the human pay a machine-speed wait they would otherwise have
absorbed while typing.

## Approach

Move the ordering barrier from "prompt bar mount" to "launch submit".

1. `,x` mounts the prompt bar/stack as soon as the rewritten prompt is resolved and the
   kill/dismiss has been applied optimistically — i.e. restore the pre-`1b2381366`
   ordering, which is the latency floor for this flow.
2. Opening a `,x` cleanup registers a **relaunch cleanup barrier** on the app. The
   barrier settles from the same `on_settled` callback the mount used to consume.
3. ACE's single launch funnel, `AgentLaunchStartMixin._submit_resolved_launch`
   (`src/sase/ace/tui/actions/agent_workflow/_launch_start.py:216`), refuses to submit
   while any barrier is unsettled: it parks a resume thunk and replays it once every
   barrier settles.

This keeps the exact guarantee the original fix bought — no durable `sase run` is
submitted before the cleanup proc that could clobber its name has settled — while
removing the wait from the path the user actually watches. In practice the barrier is
already settled by the time a human finishes reading and editing the prompt, so the hold
is usually a no-op.

`_submit_resolved_launch` is the right choke point: every ACE prompt-driven launch
(prompt bar submit, prompt history replay, quick launch, `,r` retry-edit, the bulk-Patch
fan-out) reaches `_submit_launch_proc` only through it, and it runs before the bar is
unmounted, so a held launch leaves the user's prompt intact and editable.

## Work

### 1. Add the relaunch cleanup barrier

New module `src/sase/ace/tui/actions/agent_workflow/_relaunch_barrier.py`.

Module-level helper functions that take the app object (matching the existing style of
`schedule_relaunch_prompt_resolution` in `_entry_relaunch.py`, which already operates on
an untyped `owner`). Free functions rather than a mixin, because the three call sites
live in two different action packages (`agent_workflow/` and `agents/`) and the narrow
mixin unit tests do not run `_init_app_state`; every accessor must lazily create its
state with `getattr(app, ..., None)`.

Public surface:

```python
RELAUNCH_CLEANUP_BARRIER_TIMEOUT_SECONDS = 30.0

@dataclass
class RelaunchCleanupBarrier:
    """One in-flight kill-and-edit cleanup a replacement launch must follow."""
    label: str
    settled: bool = False

def open_relaunch_cleanup_barrier(app, label: str) -> RelaunchCleanupBarrier: ...
def settle_relaunch_cleanup_barrier(app, barrier: RelaunchCleanupBarrier) -> None: ...
def relaunch_cleanup_is_pending(app) -> bool: ...
def hold_launch_for_relaunch_cleanup(app, resume: Callable[[], None]) -> bool: ...
```

Behavior:

- `open_relaunch_cleanup_barrier` appends to `app._relaunch_cleanup_barriers` and arms a
  timeout via `app.set_timer` when available.
- `settle_relaunch_cleanup_barrier` is idempotent (`barrier.settled` guard — the barrier
  can be settled by both `on_settled` and the timer). It drops the barrier, cancels its
  timer, and when no barriers remain, drains `app._relaunch_cleanup_launch_waiters` by
  calling each parked thunk exactly once. Drain by swapping the list out first so a
  thunk that re-parks (a second `,x` mid-drain) cannot loop.
- The timeout path logs a warning, settles the barrier, and emits one
  `severity="warning"` toast making clear the ordering guarantee was skipped, so a hung
  supervisor degrades to the pre-`1b2381366` behavior instead of wedging every launch in
  the session. 30s is far beyond the ~1.5–3s a healthy cleanup proc takes, so a real
  cleanup never trips it.
- `hold_launch_for_relaunch_cleanup` returns `False` (caller proceeds) when nothing is
  pending; otherwise parks `resume`, emits one informational toast, and returns `True`.

Initialize `_relaunch_cleanup_barriers = []` and `_relaunch_cleanup_launch_waiters = []`
in `src/sase/ace/tui/actions/_state_init_agents.py` next to
`_dismiss_persistence_inflight` / `_kill_persistence_inflight` (line 316), keeping the
lazy `getattr` fallbacks for narrow test apps.

### 2. Report whether the cleanup actually started

`_finish_kill_and_edit_agent` must mount the prompt bar only when a cleanup was really
initiated. Today the "nothing happened" case is silent and already buggy:
`_dismiss_done_agent` (`src/sase/ace/tui/actions/agents/_dismissing.py:266`) returns
early when `agent.raw_suffix is None` **without calling `on_settled`**, so the current
code leaves the prompt bar permanently unmounted; with a barrier it would instead leave
a barrier pending until the timeout.

Change `_dismiss_done_agent`, `_do_kill_agent`
(`src/sase/ace/tui/actions/agents/_kill_flow.py:32`) and `_do_bulk_kill_agents`
(`_kill_flow.py:142`) to return `bool` — whether a cleanup was initiated and therefore
whether `on_settled` will fire (immediately or via the proc). Existing call sites that
ignore the return value stay valid.

Concretely:

- `_dismiss_done_agent`: `return False` on the `raw_suffix is None` guard, `True`
  otherwise.
- `_do_kill_agent`: `False` on the unknown-kind and `signal_failed` early returns (both
  already call `on_settled`), `True` otherwise.
- `_do_bulk_kill_agents`: `False` when neither `kill_items` nor `dismiss_candidates`
  survive filtering, `True` otherwise.

Update the test doubles that shadow these methods to return `True`:
`tests/ace/tui/_retry_edit_agent_name_helpers.py`,
`tests/ace/tui/test_agent_bulk_kill_edit.py`,
`tests/ace/tui/test_family_member_relaunch.py`.

### 3. Mount immediately in both `,x` branches

`src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py`,
`_finish_kill_and_edit_agent`:

```python
barrier = open_relaunch_cleanup_barrier(self, f"kill-and-edit {agent.display_name}")
settle = lambda: settle_relaunch_cleanup_barrier(self, barrier)

if agent.status in DISMISSABLE_STATUSES or agent.pid is None:
    if not self._dismiss_done_agent(agent, on_settled=settle):
        settle()
        return
    mount_prompt_bar()
    return
```

and the same shape inside the `ConfirmKillModal`'s `on_dismiss` for `_do_kill_agent`.
Open the barrier only after the user confirms, so a cancelled kill leaves no barrier
(the existing `test_focused_kill_and_edit_cancel_stays_non_destructive_and_unsubmitted`
covers that).

`src/sase/ace/tui/actions/agents/_marking_kill.py`, `_bulk_kill_marked_agents_and_edit`:
same shape around `_do_bulk_kill_agents` and `mount_prompt_stack`.

Update the docstrings on both — they currently assert the mount-after-settle contract
explicitly and would become wrong.

### 4. Gate the launch

`src/sase/ace/tui/actions/agent_workflow/_launch_start.py`, at the top of
`_submit_resolved_launch` (before the `_prompt_context is None` check, so a held launch
does not consume the context):

```python
if hold_launch_for_relaunch_cleanup(
    self,
    lambda: self._submit_resolved_launch(
        prompt, keep_bar=keep_bar, extra_payload=extra_payload
    ),
):
    return
```

The resume thunk re-enters the same method, so on drain it takes the normal path with a
freshly reserved launch timestamp. Nothing else in `_submit_resolved_launch` moves.

Two edge cases the resume thunk must handle, both already handled by re-entry:

- The user cancelled the prompt bar during the hold: `self._prompt_context` is `None`,
  so the re-entered call notifies "No prompt context - cannot launch". Suppress that by
  having `hold_launch_for_relaunch_cleanup`'s drain skip a thunk when
  `app._prompt_context is None` (the cancel path already saved the text to prompt
  history, so nothing is lost). Log at debug.
- A multi-pane stack submitting pane 1 with `keep_bar=True` while the bulk barrier is
  still pending: the pane is parked and replayed; the base context is untouched, which
  is exactly what `keep_bar` already guarantees.

### 5. Tests

Rewrite `tests/ace/tui/test_kill_and_edit_deferred_settlement.py` (rename to
`tests/ace/tui/test_kill_and_edit_launch_barrier.py`, and update its module docstring,
which currently states the old contract). It already drives the real
`_dismiss_done_agent` / `_do_kill_agent` / `_do_bulk_kill_agents` persistence chain
through `TrackedProcRecorderMixin`, so it is the right harness for the new contract:

- Focused dismiss `,x`: the prompt bar is ready **before**
  `tracked_procs[-1]["proc_callable"]()` runs.
- Focused kill `,x` (confirm modal): same.
- Marked bulk `,x`: the prompt stack is ready before settlement, with the same
  `["%id:!live\nFirst", "%id:!done\nSecond"]` pane assertion the current test makes.
- Rejected submission (`_dismiss_persistence_inflight` collision): mounts, and leaves no
  barrier pending.
- Cancelled kill: nothing mounted, no proc, no barrier.
- `raw_suffix is None`: no mount, no barrier left pending.

New tests for the barrier itself — this is where the original bug's regression coverage
now lives, stated as _relative proc order_ rather than as a mount assertion:

- Submitting a launch while the barrier is pending submits **no** launch proc; after the
  cleanup proc's callable runs, exactly one launch proc is submitted with the same
  prompt. This is the direct regression test for `1b2381366`.
- After settlement, a launch submits with no hold and no toast.
- Cancelling the prompt bar during a hold drops the parked launch: no launch proc after
  settlement.
- Timeout fires → barrier drops, parked launch proceeds, warning toast recorded.
- Two overlapping `,x` barriers: the parked launch replays only once, after the second
  settles.

Check whether `tests/test_force_reuse_launch_seam.py` (added by the same commit) encodes
the mount-ordering contract; if it does, retarget it at the launch ordering instead.

## Verification

Run `just install` first (ephemeral workspace), then:

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED \
  --next 'Report just check-full results for the ,x prompt-latency tale'
```

`just check-full` rather than `just check`: this touches the shared kill/dismiss
persistence signatures (`_do_kill_agent`, `_dismiss_done_agent`,
`_do_bulk_kill_agents`), which many agent tests exercise, so the diff-scoped lane is not
a safe backstop here.

Manual check in ACE: `,x` a DONE agent and a RUNNING agent, and a marked set of both.
The prompt must appear essentially instantly; the Procs indicator should show the
cleanup proc still running behind it. Submitting immediately should either launch
straight away or show the waiting toast once and then launch.

## Out of scope

- **Durable proc dependencies.** Chaining the launch behind the cleanup proc in the
  supervisor would be the most robust fix and would survive an ACE restart, but
  `ProcSubmitRequest.followup` (`src/sase/procs/request.py:53`) is monitor-specific, not
  a general dependency graph, and ACE needs the launch result back on the UI thread for
  `_handle_launch_results_delta`. That is a much larger change across the proc
  supervisor and the Rust core boundary.
- **Making `save_dismissed_bundle` refuse a write for a just-wiped suffix.** A tombstone
  at the destination would remove the ordering requirement entirely, but it changes
  bundle/name-registry semantics that live behind the sase-core boundary. Worth a
  follow-up task bead if this race shows up again from a non-ACE writer.
- **Shortening `ProcObserver.POLL_SECONDS` or making `request_poll()` interrupt the poll
  sleep.** A real general responsiveness win (`_thread_main` currently sleeps a fixed
  0.5s regardless of `request_poll()`), but it is not on this critical path once the
  mount no longer waits for settlement.
- **Restructuring `,x` to show `ConfirmKillModal` concurrently with prompt resolution.**
  `prepare_kill_edit_agent_prompt` is one artifact read plus pure string rewriting; add
  a `log.debug` timing line alongside the existing one in `_do_kill_agent`
  (`_kill_flow.py:135`) and only pursue this if it measures material.

## Notes

- No feature flag. This restores previously shipped behavior behind a stricter guard
  rather than introducing a new user-facing choice or an unfinished path.
- Keep `schedule_relaunch_prompt_resolution` and the `resolve_agent_identity`
  re-resolution exactly as they are; both are unrelated to this latency and still
  needed.
- `on_settled` composition in `_kill_procs.py` / `_dismissing.py` already guarantees the
  callback fires exactly once (proc completion via the `finally` in
  `_deliver_tracked_completion`, or immediately on a rejected submission). The barrier
  relies on that; `settle_relaunch_cleanup_barrier` stays idempotent anyway because the
  timeout is a second possible caller.
