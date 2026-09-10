---
tier: tale
title: Move ACE disabled-provider launch resolution into the launch proc
goal: "Prompt-input launches stay responsive and avoid duplicate launch planning while
  preserving the interactive disabled-provider decisions and fail-closed launch
  guarantees.

  "
size: medium
proposed_by: bbugyi200.athena.09k
create_time: 2026-09-09 20:00:11
status: wip
---

# Plan: Move ACE disabled-provider launch resolution into the launch proc

## Problem and confirmed cause

ACE already submits every agent launch as a tracked, durable `run.launch` proc backed by
`sase run`. That child process expands multi-prompts and xprompt swarms, resolves
fan-out models, applies the hard-disabled-provider guard, and only then claims
workspaces or spawns agents.

The prompt-input path currently performs a second copy of the expensive part first.
Whenever the cheap disable snapshot contains any hard disable,
`_launch_provider_guard.py` calls `plan_launch_units()` in a Textual thread worker,
including xprompt expansion and every candidate/model resolution. Only after that
preflight reports clear does ACE submit the durable launch proc, which repeats expansion
and provider selection. Moving the worker to a different thread prevents a direct event
loop block, but it does not make the work a tracked proc, eliminate the duplicate
planning, or give the user the normal launch-proc lifecycle while the check runs.

The existing child-side guard is correctly placed before workspace allocation and agent
spawn and remains the source of truth. The change should expose its confirmed block as a
typed ACE result instead of maintaining a second TUI-side planner.

## Resulting flow

1. Prompt submission performs only cheap, lexical/input work on the UI thread and
   submits the existing durable `sase run` launch proc. ACE marks the request as an
   interactive provider-guard launch; it does not create a separate preflight proc.
2. The child expands the launch exactly once and runs the canonical hard-disable guard.
   A clear launch continues in that same process to workspace allocation and spawning.
3. A confirmed block produces a successful, typed `provider_decision_required` proc
   result containing the complete expanded unit inputs and the first blocked unit. No
   workspace is claimed, no agent is spawned, and this expected interaction is not
   recorded as a failed launch.
4. ACE rehydrates the strict result, keeps or restores ownership of the submitted
   prompt, and opens the existing `DisabledProviderLaunchModal`. Abort remains local.
   Model and unit changes update the retained session; enable/soft-enable choices are
   sent as an explicit action on the next tracked launch request.
5. Continuing from the modal resubmits the same durable `run.launch` operation with the
   retained `launch_units` bundle and any confirmed provider-state action. The child
   applies the action, re-runs the canonical guard, and either launches or returns the
   next blocked unit. No recheck calls `plan_launch_units()` or `blocked_launch_units()`
   inside the TUI process.

## Implementation

### 1. Define a strict provider-guard wire result at the launch-domain boundary

In `src/sase/agent/launch_guard.py`, add versioned serialization and strict parsing for
the data ACE needs to continue the existing modal workflow:

- the ordered expanded `LaunchUnitInput` values (`prompt`, `template_group`, and
  `swarm_xprompts`);
- the blocked `LaunchUnit`, including its original index/total, candidates,
  availability, resolved provider/model, and blocking disable records; and
- an explicit outcome discriminator/schema version so ordinary launch-result payloads
  cannot be mistaken for an interactive guard response.

Have `guard_launch_units()` plan once when a hard disable exists and attach both the
complete planned inputs and the first blocked unit to `DisabledProviderLaunchError`.
Keep its no-hard-disable lock-free shortcut and its existing fail-open behavior for
unexpected planning errors. Reuse `TemporaryProviderDisable.from_wire()` for strict
disable-record hydration and reject malformed/unknown fields rather than partially
constructing a modal model.

Unit-test round trips, malformed versions/fields, empty plans, candidate details, and
multi-unit ordering in `tests/agent/test_launch_guard.py`.

### 2. Teach the existing `run.launch` proc to return an interaction, not a generic failure

Extend the ACE-authored request in `src/sase/ace/tui/actions/agent_durable.py` with an
explicit interactive-provider-guard capability. Keep plain shell, mobile, and other
programmatic `sase run` callers on their current fail-closed error contract.

In `src/sase/main/query_handler/_launch.py` and the CWD launch facade/implementation:

- allow the ACE request to suppress failed-prompt history only for a confirmed provider
  decision that is being returned to the still-live interactive surface;
- catch `DisabledProviderLaunchError` before the generic `RuntimeError` branch;
- emit a successful `run.launch` operation result whose typed payload is
  `provider_decision_required`, then exit without spawning; and
- accept an optional, strictly validated follow-up action to enable all blocking
  providers, enable one named blocking provider, or convert the current hard records to
  soft records while preserving their expiry. Apply that user-confirmed action inside
  the same tracked child before rechecking and launching.

Do not weaken the final child-side guard: stale disable snapshots, a provider disabled
between UI submission and child execution, malformed interactive payloads, and direct
non-ACE launches must still fail before any workspace claim or spawn. Do not treat a
provider-decision result as a launched or failed prompt; a later successful launch owns
the normal history write, while an unrelated child failure retains the existing failed
launch recovery behavior.

Cover the request/result seam in `tests/agent/test_launch_cwd_guard.py` (or a focused
sibling): interactive block result, ordinary CLI block, action-before-recheck,
soft-enable expiry preservation, malformed action rejection, and proof that blocked
paths never call the spawn/workspace allocators.

### 3. Replace the TUI worker preflight with launch-proc session ownership

Refactor `src/sase/ace/tui/actions/agent_workflow/_launch_provider_guard.py`,
`_launch_start.py`, and `_launch_procs.py` so prompt launches go directly to
`_submit_launch_proc()` and the completion handler recognizes the typed guard result.
Remove the TUI imports/calls for `plan_launch_units()`, `blocked_launch_units()`, and
the `launch-provider-guard` Textual worker group.

Track the prompt-release state by proc ID, alongside the existing submitted-prompt
recovery metadata. The record must contain the immutable launch context, submitted
prompt, `keep_bar` behavior, and any existing guard session. This gives completions a
generation/ownership check instead of consulting whatever prompt happens to be current
when the child returns.

Preserve these UI invariants:

- normal launches still transfer focus and release the prompt context without waiting
  for agent refresh work;
- while an interactive check can still return, the exact submitted bar/stack remains
  recoverable, including placeholder-expanded, multi-pane, relaunch, and `keep_bar`
  submissions;
- a stale completion never closes or overwrites a newer prompt bar; retain the prompt in
  the existing failed-launch stash and show an actionable toast if ownership was lost;
- a malformed guard result fails visibly and preserves the prompt instead of launching
  from partially trusted data; and
- proc completion, quit accounting, deduplication, failure logging, and the Procs pane
  continue to see one ordinary launch proc per launch attempt.

Use the cheap lock-free disable peek only if needed to decide whether prompt release
must be deferred; it must never expand prompts, resolve models, read xprompt
definitions, or replace the authoritative child guard. If release is deferred, move
focus off the prompt widget and prevent a duplicate submission while leaving the rest of
ACE navigable; on success release only the bar generation owned by that proc, and on a
guard result restore editing before opening the modal.

### 4. Drive every modal continuation back through the same proc path

Retain `_ProviderGuardSession` as presentation/session state, but populate it from the
strict child result. Map the blocked unit's relative index back to the session's
surviving original units so sequential multi-agent abort/model decisions keep the
current `original_total` labels and launch the intended subset.

Change modal continuations as follows:

- `abort_unit` marks the current unit locally; aborting the last unit/all units ends the
  session and leaves the prompt editable;
- `pick_model` rewrites only that unit and builds the existing strict `launch_units`
  request bundle;
- enable and soft-enable choices become the validated provider action on the resubmitted
  launch request rather than a TUI-thread store write plus local recheck; and
- every nonterminal continuation submits the retained prompt/unit bundle through
  `_submit_launch_proc()`. Its result either completes the real launch or advances the
  modal to the next blocked unit.

Account explicitly for entry points already sharing `_finish_agent_launch()`—normal
prompt input, history/relaunch, placeholder input collection, single-pane `keep_bar`,
and marked-Patch bulk launch. Bulk children may each race with provider-state changes,
so they must retain the child fail-closed guard; serialize any interactive block
handling so ACE never stacks competing disabled-provider modals or loses the source
prompt.

Update `tests/ace/tui/test_disabled_provider_launch_panel.py`,
`tests/ace/tui/test_agent_launch_non_blocking.py`,
`tests/ace/tui/test_launch_submit_context_release.py`, and focused launch-proc tests to
exercise:

- no in-process planning/worker call even with a hard disable;
- clear, blocked, malformed, unrelated-failure, and stale-completion results;
- enable, soft-enable, enable-one, model replacement, abort-one, and abort-all;
- sequential blocked units with original index preservation;
- exact prompt/context recovery and no duplicate failed-history/stash entry;
- duplicate-submit/dedup behavior while a guard-capable proc is running; and
- normal, `keep_bar`, relaunch, and bulk entry points.

Add a responsiveness regression that holds the child result open while sending a benign
ACE navigation/message event. Assert the event is handled before completion and that no
Textual worker or synchronous launch planner ran on the prompt-submit path. This pins
the reported symptom rather than only checking the final modal.

## Verification

1. Run focused launch-domain, CLI seam, modal, proc-completion, prompt-context, and
   responsiveness tests while iterating.
2. Run `just install` before repository verification, as required for an ephemeral SASE
   workspace.
3. Run `just check` and resolve every lint, type, import-graph-selected test, and plan
   regression failure. If the scoped selector escalates or reports unusual selection,
   use `/sase_monitor` to run `just check-full` with the required `TESTING`/`TESTED`
   statuses and a follow-up action.
4. Confirm with `git diff --check` and a final search that the ACE prompt-launch path no
   longer calls `plan_launch_units()`, `blocked_launch_units()`, or the retired
   `launch-provider-guard` worker, while the child-side guard remains before workspace
   allocation/spawn.

## Non-goals

- Do not remove hard-disabled-provider enforcement from non-ACE launch surfaces.
- Do not add a second preflight command/proc or move provider/model selection into a
  presentation-only widget.
- Do not change the modal's choices, keybindings, or visual design except for minimal
  pending/interaction state needed to make proc ownership clear.
- Do not cache a provider decision across launches; each child must re-read current
  provider-disable state immediately before launch.
