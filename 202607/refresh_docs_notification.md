---
create_time: 2026-07-09 12:46:20
status: done
prompt: 202607/prompts/refresh_docs_notification.md
tier: tale
---
# Plan: Stop the redundant `refresh_docs` chop completion notification

## Problem

Every time the docs-refresh chop runs (launched with a prompt like
`%n:sase_refresh_docs-@ #gh:sase-org/sase %g:chop #!sase/refresh_docs`), the user gets a notification such as:

```
CODEX(gpt-5.5) @sase_refresh_docs-o completed: ...
```

This notification is noise. When the job actually does work it launches two docs agents
(`refresh_docs.<project>.<head>.update` and `.polish`), and those agents are already visible on the Agents tab — so a
separate "completed" notification for the orchestrator agent that launched them adds nothing. The user wants this
orchestrator notification to stop appearing while keeping the launched docs agents visible as they are today.

## Background: how this notification is produced (and already suppressed elsewhere)

The `#!sase/refresh_docs` reference runs the `refresh_docs` workflow defined in `xprompts/refresh_docs.yml`. It is
invoked as an ordinary user agent (the orchestrator, named `sase_refresh_docs-@` → e.g. `sase_refresh_docs-o`). When
that user agent finishes, its completion notification is built by `send_completion_notification` in
`src/sase/axe/run_agent_runner_finalize.py` (sender `user-agent`) — this is the `PROVIDER(model) @name completed: ...`
line the user sees.

Crucially, SASE **already** has a mechanism to suppress this exact notification. In
`src/sase/axe/run_agent_runner_lifecycle.py`, the completion notification is only sent when:

```python
if (not suppress_completion_notification
        and not was_killed()
        and not all_steps_hidden(current_artifacts_dir)):
    send_completion_notification(...)
```

`all_steps_hidden` (`src/sase/axe/runner_utils.py`) reads the run's `workflow_state.json` and returns `True` when every
step that actually ran is either `hidden: true` or was skipped. In other words, a workflow whose steps are all hidden is
meant to complete silently.

The `refresh_docs` workflow was clearly authored with this quiet-by-design intent: the workflow frontmatter is
`hidden: true`, and its `count_commits` and `launch_docs_agents` steps are both `hidden: true`.

## Root cause

The workflow's third step, `update_marker`, is the **only** step missing `hidden: true`.

`update_marker` runs only when the job actually launches agents (its guard is
`if: count_commits.count >= threshold and launch_docs_agents.launched > 0`). Evidence from a real run confirms the
failure mode:

- No-op runs (commits below threshold): `launch_docs_agents` and `update_marker` are both **skipped**, so
  `all_steps_hidden` is already `True` and **no notification is sent**. This is why the user only sees the notification
  when the job does work.
- Launch runs (commits at/over threshold): `count_commits` (hidden) and `launch_docs_agents` (hidden) run, and then
  `update_marker` runs with `status="completed"` and `hidden=false`. That single visible step makes `all_steps_hidden`
  return `False`, so the orchestrator's completion notification fires — precisely in the case where the two launched
  docs agents are already on the Agents tab.

So the redundant notification appears exactly, and only, when the chop launches its two agents, and it is caused by one
step lacking the `hidden` flag its siblings already carry.

## Chosen solution

Add `hidden: true` to the `update_marker` step in `xprompts/refresh_docs.yml`, matching the `count_commits` and
`launch_docs_agents` steps.

With all three steps hidden, `all_steps_hidden` returns `True` in every run of this workflow, so the existing
suppression gate stops the orchestrator's completion notification — no new code paths, no new config, no new directive.

### Why this is the right fix

- **Uses existing functionality.** It relies on the `all_steps_hidden` suppression path that already exists for exactly
  this purpose, rather than inventing a new suppression mechanism.
- **Completes the author's evident intent.** The workflow and its other two steps are already hidden; the un-hidden
  `update_marker` reads as an oversight.
- **Keeps the launched agents visible.** Step-level `hidden` only affects the completion-notification gate (and the
  step's row in the workflow detail view). It does **not** hide the orchestrator's agent row on the Agents tab (row
  visibility comes from the agent-level `%hide` directive, which is not used here), and it has no effect on the
  separately-launched `.update` / `.polish` docs agents, which remain visible and continue to emit their own
  notifications when they run.
- **Minimal blast radius.** It is a one-line change scoped to this workflow file; it does not alter behavior for any
  other agent or workflow, and it does not touch the Rust core.

### Known trade-off

Once every step is hidden, `all_steps_hidden` also suppresses the notification if the orchestrator _fails_ (a
fully-hidden workflow completes silently whether it succeeds or fails). This already applies to the two currently-hidden
steps, so the change only extends it to `update_marker`. For a low-stakes background docs-refresh trigger this is
acceptable: failures still surface via the agent row on the Agents tab and via the axe error digest. This trade-off
should be called out for the reviewer; if preserving failure notifications is later deemed important, that is a
separate, broader change (see Alternatives) and not required to resolve the reported noise.

## Alternatives considered (and why not)

1. **Add `%hide` / `%h` to the launch prompt.** This is the documented directive for silencing an agent's completion
   notification, and it would work — but it also hides the orchestrator's row on the Agents tab (hidden-by-default,
   revealed with `.`), it must be added to every launch / the saved launch command rather than being fixed durably in
   the repo, and a `%hide` notification is still written to the inbox (merely filtered from the toast, badge, and
   modal). Rejected: it changes row visibility and is not a durable repo-level fix.

2. **Globally suppress completion notifications for "no-op" workflow runs** (in
   `src/sase/axe/run_agent_runner_finalize.py`, treating `outcome == "noop"` like `plan_rejected`). `refresh_docs` is
   always classified `noop` because it launches its agents from a `python` step rather than `agent:` steps, so this
   would also work. Rejected: it has a broad blast radius — it would change behavior for every user-agent workflow that
   launches zero `agent:` steps, including bash/python-only workflows where a completion notice may be wanted — and it
   is unnecessary given the targeted, existing `all_steps_hidden` path already solves the problem.

3. **New directive / config toggle / per-source notification mute.** Investigated and rejected as over-engineering:
   notification mute today is strictly per-notification (by id), there is no config surface for suppressing completion
   notifications, and building a new rule-based mute axis is unjustified when a single existing flag resolves the issue.

## Implementation steps

1. In `xprompts/refresh_docs.yml`, add `hidden: true` to the `update_marker` step so it matches the `count_commits` and
   `launch_docs_agents` steps. (The step's `if:` guard and `python:` body are unchanged; the marker file is still
   written on launch runs.)

2. If a structural/snapshot test exists for the `refresh_docs` workflow's steps, update it to expect the new `hidden`
   flag. (Current tests exercise the generic `all_steps_hidden` helper rather than this workflow's contents, so no test
   change may be required.)

3. Run `just install` then `just check` (this is an `xprompts/` change, not one of the check-exempt categories).

## Verification

- Trigger the `refresh_docs` chop in a launch scenario (commit count at/over threshold) and confirm:
  - the orchestrator's `@sase_refresh_docs-* completed` notification no longer appears (no toast, no unread badge,
    absent from the notifications modal), and
  - the two docs agents (`.update` and `.polish`) still appear on the Agents tab as before.
- Confirm a below-threshold run continues to produce no notification (unchanged behavior).
- Confirm the `refresh_docs_marker` file is still updated on launch runs (the step still executes).
