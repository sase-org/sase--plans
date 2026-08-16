---
tier: tale
title: Release the prompt context after launch so Ctrl+Space stops dying mid-session
goal:
  Ctrl+Space keeps working for a whole ACE TUI session across any number of agent
  launches, because a stale `_prompt_context` can no longer report a phantom active
  prompt and latch the action-availability and refresh gates off.
size: medium
proposed_by: bbugyi200.athena.03a
create_time: 2026-08-16 09:20:25
status: done
---

- **PROMPT:**
  [prompts/202608/ctrl_space_stale_prompt_context.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ctrl_space_stale_prompt_context.md)
- **AGENTS:**
  - [bbugyi200.athena.03a](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.03a.md)
- **COMMITS:**
  - [2aa8ba2](https://github.com/sase-org/sase/commit/2aa8ba26f7efc6522a7cc28e969c688a3f870b18)
    — fix(tui): release stale prompt context after launch

# Plan: Fix `<ctrl+space>` dying mid-session (stale `_prompt_context` after every launch)

## Problem

`<ctrl+space>` (`start_agent_from_patch`, "repeat last +/Ctrl+Space selection") stops
working part-way through an ACE TUI session and starts working again after a restart.

The user's suspicion is **confirmed but understated**: the key really does go dead and a
restart really does fix it, but the failure is not random. It is deterministic — the key
dies on the **first successful agent launch of the session**, and it comes back whenever
the user happens to open a prompt bar and _cancel_ it instead of launching. That
cancel-revives-it coincidence is what makes it feel intermittent.

The same stale flag silently disables several other surfaces (see "Blast radius").

## Root cause

`src/sase/ace/tui/actions/_event_base.py:88` `_prompt_input_active()` reports "a prompt
surface is live" when **any** of `_prompt_context`, `_approve_prompt_context`,
`_plan_feedback_context` is non-`None`, _or_ a `PromptInputBar` is mounted:

```python
def _prompt_input_active(self) -> bool:
    if getattr(self, "_prompt_context", None) is not None:
        return True
    ...
    return bool(query(PromptInputBar))
```

`src/sase/ace/tui/_app_action_availability.py:66-69` disables the launch action whenever
that returns True:

```python
if action == "start_agent_from_patch" and (
    bool(getattr(app, "_screen_stack", ())) and app._prompt_input_active()
):
    return False
```

That guard is correct and intentional (added in `18d0d9241`, 2026-07-28, "fix(tui):
preserve active prompts on Ctrl+Space"). The defect is that `_prompt_context` is now
**never released on a successful launch**, so the guard latches on permanently.

### Why the release stopped happening

`_launch_resolved_prompt`
(`src/sase/ace/tui/actions/agent_workflow/_launch_start.py:129`) hands the context to
the launch worker and relies on the worker to clear it:

```python
ctx = self._prompt_context            # not keep_bar: same object, no copy
...
if not keep_bar:
    self._unmount_prompt_bar_after_submit()
launch_ctx = ctx if keep_bar else None
self._submit_launch_proc(
    ...,
    proc_callable=lambda: self._run_agent_launch_body(prompt, launch_ctx),
)
```

Passing `launch_ctx=None` is the "you own the context, clear it when you consume it"
signal — `run_agent_launch_body` sets `owns_context = ctx is None` and does
`app._prompt_context = None` on every exit (`_launch_body_impl.py:78,108,164,279,296`
and `_launch_body_single.py:166,184,250,276`).

Commit `0835b38d2` (2026-08-15, "feat(ace): migrate Patch and agent producers to durable
argv") rewrote `_submit_launch_proc` to submit argv-only `sase run` through the durable
supervisor, and **discards the callable**
(`src/sase/ace/tui/actions/agent_workflow/_launch_procs.py:81`):

```python
    ``proc_callable`` is accepted so existing test doubles can still record
    the launch body. Production submits argv-only ``sase run``.
    """
    del proc_callable
```

So `run_agent_launch_body` — the only production code that released `_prompt_context`
after a launch — became unreachable. `_run_agent_launch_body` and
`_run_agent_launch_body_async` now have **no production callers at all**; the real
launch runs out-of-process in `sase run` (`src/sase/main/query_handler/_launch.py`).

Net effect: after any successful submit the bar is unmounted but `_prompt_context` stays
set. `_prompt_input_active()` then reports a phantom active prompt forever.

Contrast the prompt-stash path, which gets this right and is the pattern to copy
(`_prompt_bar_stash.py:96-97`):

```python
self._unmount_prompt_bar_after_submit()
self._prompt_context = None
```

### Verification already performed

Driving the **real** `_launch_resolved_prompt` + `_submit_launch_proc` against a stub
that only supplies `_submit_durable_proc` reproduces it directly:

```
bar unmounted after submit  : True
_prompt_context after launch: True     <-- leaked
_prompt_input_active()      : True     <-- phantom active prompt
```

and feeding that through the real availability policy:

```
prompt_input_active=False -> start_agent_from_patch: True
prompt_input_active=True  -> start_agent_from_patch: False   <-- key is dead
agents tab, leaked ctx    -> search_forward: False
```

### Why tests did not catch it

Every launch test replaces `_submit_launch_proc` with a double that **captures and
invokes `proc_callable`** (`tests/ace/tui/_agent_launch_helpers.py:65`,
`tests/ace/tui/test_agent_launch_non_blocking.py:50`,
`tests/ace/tui/_launch_fan_out_helpers.py:123`, `tests/test_agent_launch_repeat.py:43`).
The doubles keep exercising the in-process body — and its context clearing — that
production no longer runs. No test drives the real `_submit_launch_proc`.

The one test that names this invariant,
`tests/ace/tui/widgets/test_vim_normal_key_containment.py:210`
`test_ctrl_space_action_is_gated_only_while_prompt_is_mounted`, only checks "fresh
session → ungated" and "bar mounted → gated". It never checks "after a launch →
ungated".

## Blast radius

Every consumer of `_prompt_input_active()` latches on the same stale flag after the
first launch, until a cancel path clears it or the TUI restarts:

- `_app_action_availability.py:66` — `<ctrl+space>` / `start_agent_from_patch` dead (the
  reported symptom).
- `_app_action_availability.py:52` — `/` and `?` (`search_forward` / `search_reverse`)
  on the Agents tab dead.
- `actions/agents/_metadata_search.py:46` — `_agent_metadata_search_can_start()` always
  False, so the inline metadata search never claims the keyboard.
- `actions/event_refresh/_auto_refresh.py:72` — `_on_auto_refresh()` returns early every
  tick, so periodic auto-refresh **and** the `FULL_SANITY_REFRESH_SECONDS` reconcile are
  starved.
- `actions/event_refresh/_watcher.py:71` — watcher events are pushed into the defer
  branch forever; `_on_artifact_change_deferred` re-enters, re-checks the same stale
  flag, and re-defers, accumulating `_artifact_change_deferred_paths`.

(Tab switches and the notification poll still refresh, which is why the refresh
starvation is less obvious than the dead keymap.)

## Secondary leak paths (same symptom, predate `0835b38d2`)

`_finish_agent_launch` has terminal paths that return without clearing the context. They
are harmless while a prompt bar is mounted (the user can see the bar and cancel it), but
`_select_and_open_editor_for_home` (`_prompt_bar_mount.py:486`) sets up a prompt context
and opens `$EDITOR` **without ever mounting a bar**, so the same paths leak invisibly:

- `_launch_start.py:75` — `_collect_prompt_inputs_then_launch(...)`, then the user
  cancels `InputCollectionModal`: `_after(None)` notifies and returns
  (`_launch_start.py:112-114`), context retained.
- `_launch_start.py:83` — `render_prompt_with_inputs` raises `PromptInputError`: notify
  and return, context retained.
- `_launch_start.py:120` — `apply_prompt_input_values` raises `PromptInputError`: notify
  and return, context retained.

These reach the user through `+` → "open in editor", the Ctrl+Space replay with
`open_in_editor`, and `action_start_last_vcs_xprompt_in_editor`.

## Scope

In scope: release `_prompt_context` at the correct owner boundary, make
`_prompt_input_active()` structurally unable to latch on a dead context, close the
bar-less leak paths, and add regression coverage that drives the real submit path.

Out of scope (propose as a follow-up task bead, do not do it here): retiring the now
production-dead in-process launch body (`run_agent_launch_body`,
`run_single_agent_launch_body`, `_launch_repeat_agents`, `_launch_bulk_agents`,
`_launch_multi_prompt_agents`, `_launch_multi_model_agents`) and the vestigial
`proc_callable` parameter. That is a large removal with wide test fallout and deserves
its own bead; this plan must not depend on it.

## Implementation

### Step 1 — Release the context at the launch hand-off (root cause)

In `src/sase/ace/tui/actions/agent_workflow/_launch_start.py`,
`_launch_resolved_prompt`: clear `self._prompt_context` on the UI thread in the
`not keep_bar` branch, paired with the unmount, exactly as `_prompt_bar_stash.py:96-97`
already does.

Constraints:

- Clear it **after** the `ctx.timestamp` / `ctx.workflow_name` reassignment. In the
  `not keep_bar` branch `ctx is self._prompt_context`, so the local `ctx` keeps the
  object alive and the submitted launch is unaffected.
- Do **not** clear it when `keep_bar=True`. There the bar stays mounted, `ctx` is a
  `dataclasses.replace` copy, and `self._prompt_context` is the still-live base for the
  remaining panes.
- Update the block comment above the unmount: it currently explains the worker-owns-it
  contract, which is no longer true. State that the launch is submitted as
  out-of-process `sase run` argv, so the UI thread is the only owner and must release
  the context here.

Leave the `owns_context` clearing inside `run_agent_launch_body` /
`run_single_agent_launch_body` alone — it is unreachable in production but still
exercised by the test doubles, and removing it belongs to the out-of-scope follow-up.

### Step 2 — Stop `_prompt_input_active()` latching on a dead context (containment)

In `src/sase/ace/tui/actions/_event_base.py`, make a non-`None` context count as
"active" only while a prompt surface is genuinely live. A context alone must never be
able to wedge the app again, whatever future path forgets to clear it.

Approach:

- Treat a mounted `PromptInputBar` as the primary liveness signal. All three contexts
  are set immediately before their bar is mounted (`_prompt_bar_mount.py:115,315`,
  `actions/agents/_notification_modals.py:201,217`), so the mounted-bar check already
  covers the normal case.
- Add one explicit flag for the genuinely bar-less window: the external-editor suspend.
  Set it around the `with self.suspend()` block in
  `src/sase/ace/tui/actions/agent_workflow/_editor.py` `_open_editor_for_agent_prompt`,
  in a `try/finally` so an editor crash cannot leak it. Honor that flag in
  `_prompt_input_active()`.
- Keep the helper's docstring honest about the new contract: "a prompt surface is
  _mounted_, or the external editor is _currently_ suspended over the TUI".

Preserve today's behavior for the cases the guard was written for: with a bar mounted
(`test_ctrl_space_leaves_focused_prompt_intact` and its frontmatter/vim variants) the
result must be unchanged.

### Step 3 — Close the bar-less leak paths

In `src/sase/ace/tui/actions/agent_workflow/_launch_start.py`, release the context on
the three terminal paths listed under "Secondary leak paths" — but only when no prompt
bar is mounted, so the bar-mounted case keeps its context and the user's prompt survives
the cancel (that is exactly what `test_ctrl_space_leaves_focused_prompt_intact`
protects).

Add one small private helper on the mixin rather than repeating the query three times,
and route all three paths through it. Give it a docstring saying why the mounted-bar
case must be left alone.

### Step 4 — Regression coverage

The coverage gap is that no test drives the real `_submit_launch_proc`. Fix that first,
then cover the rest.

1. **Invariant test against the real submit path.** Add a test that exercises the real
   `LaunchProcMixin._submit_launch_proc` (stub only `_submit_durable_proc`, do not
   override `_submit_launch_proc`) through `_launch_resolved_prompt`, and asserts that
   after the launch `_prompt_context is None` and `_prompt_input_active()` is False.
   This test must fail on `master` before Step 1 and pass after.

2. **Extend the action-gate test.** In
   `tests/ace/tui/widgets/test_vim_normal_key_containment.py`, extend
   `test_ctrl_space_action_is_gated_only_while_prompt_is_mounted` (or add a sibling)
   with the missing third case: mount a home prompt, submit it, and assert
   `check_action("start_agent_from_patch", ())` is not `False` once the bar is gone.
   This is the end-to-end assertion of the reported bug.

3. **`keep_bar` must not regress.** Assert that a single-pane submit from a multi-pane
   stack leaves the bar mounted, leaves `self._prompt_context` set, and still gates
   `<ctrl+space>`. `tests/ace/tui/test_prompt_bar_stack_submit_handlers.py` is the
   natural home.

4. **Bar-less leak paths.** Cover the `InputCollectionModal` cancel and the
   `PromptInputError` paths reached with no bar mounted: assert the context is released
   and `<ctrl+space>` still works afterwards. Also assert the mounted-bar counterpart is
   untouched.

5. **Editor-suspend window.** Assert `_prompt_input_active()` is True while
   `_open_editor_for_agent_prompt` is suspended and False once it returns, including
   when the editor subprocess raises.

6. **Blast-radius spot check.** One test asserting `search_forward` on the Agents tab
   and `_on_auto_refresh()` are both live again after a launch, so a future regression
   here is caught by more than the keymap test.

## Verification

- `just check` for the iteration loop.
- `just check-full` before landing — this touches the launch path, the
  action-availability policy, and the refresh gates, which is squarely the "touches the
  broadening set" case. Run it through `/sase_monitor` with a `--next` action, never
  inline.
- Manual confirmation in a real TUI, since the reported symptom is interactive:
  1. Start `sase ace`, press `<ctrl+space>` — the previous selection replays (or the "No
     previous +/Ctrl+Space selection" warning appears).
  2. Launch any agent from the prompt bar and let the bar close.
  3. Press `<ctrl+space>` again — on `master` nothing happens; after the fix it replays.
  4. Confirm `/` on the Agents tab still works and the Agents list still auto-refreshes
     after that launch.

## Acceptance criteria

- `<ctrl+space>` keeps working for the whole session, across any number of launches.
- After a successful non-`keep_bar` launch: `_prompt_context is None`,
  `_prompt_input_active()` is False, and `check_action("start_agent_from_patch", ())` is
  not `False`.
- A `keep_bar` single-pane submit still keeps the bar mounted, keeps the base context,
  and still gates `<ctrl+space>`.
- With a prompt bar mounted, `<ctrl+space>` is still suppressed and the in-progress
  prompt text is still intact — `18d0d9241`'s behavior is preserved.
- A context that is somehow left set with no mounted bar and no editor suspend no longer
  disables any keymap or refresh gate.
- Agents-tab `/` and `?`, `_on_auto_refresh()`, and watcher-driven refresh are all live
  after a launch.
- New tests fail on `master` and pass after the change; `just check-full` is green.

## Proposed follow-up (do not implement here)

File a task bead via `/sase_new_task`: the in-process TUI launch body
(`run_agent_launch_body`, `run_single_agent_launch_body`, and the
repeat/bulk/multi-prompt/multi-model fan-out helpers) has had no production callers
since `0835b38d2` — `sase run` does that work out-of-process — yet it is still fully
covered by test doubles that invoke the vestigial `proc_callable` parameter. That
arrangement is what hid this bug, and the same shape can hide the next one. The bead
should propose retiring the dead body and the `proc_callable` parameter together, so
test doubles cannot diverge from production again.
