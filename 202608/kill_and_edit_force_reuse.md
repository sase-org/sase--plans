---
tier: tale
title: Restore forced-name-reuse launches for Agents-tab `,x` kill-and-edit
goal:
  A prompt pre-filled by the Agents-tab `,x` kill-and-edit keymap launches successfully
  again, reclaiming the killed agent's exact name instead of failing with "uses forced
  reuse; confirmation is required".
size: medium
proposed_by: bbugyi200.athena.054
create_time: 2026-08-17 13:33:48
status: wip
---

# Plan: Restore forced-name-reuse launches for Agents-tab `,x` kill-and-edit

## Symptom

Pressing `,x` on the Agents tab kills the selected (or marked) agent and pre-fills the
prompt input widget with the agent's prompt, rewritten so the relaunch reclaims the dead
agent's exact name. The rewritten `%id` directive carries the forced-reuse `!` marker,
e.g.:

```text
%id(!2, clan=sase-op, bead=sase-op.2)
#gh:gh_sase-org__sase
%model:@medium
%auto
#bd/work_phase_bead:sase-op.2
```

Submitting that pre-filled prompt always fails. The launch proc exits 1 and its log
contains only:

```text
Error: Agent name 'sase-op.2' uses forced reuse; confirmation is required.
```

The prompt _text_ is correct — the `!` spelling is exactly what the relaunch is supposed
to produce. What is broken is that nothing consumes the `!` any more.

## Root cause

`sase.agent.launch_validation._preflight_launch_name_requests()` raises
`AgentNameReuseConfirmationRequiredError` whenever a launch prompt requests forced reuse
and the caller did not pass `allow_force_reuse=True`
(`src/sase/agent/launch_validation.py:378`). ACE is the surface that supplies that
confirmation.

ACE used to supply it inside `run_agent_launch_body()`
(`src/sase/ace/tui/actions/agent_workflow/_launch_body_impl.py:52`-`121`), which:

1. computed `rewrite_force_reuse_name_directives(prompt)` (strips the `!`),
2. when that differed from the submitted prompt, ran
   `preflight_launch_name_requests(..., allow_force_reuse=True)` on the multi-prompt
   segments (non-mutating validation _before_ any destructive step),
3. collected `force_reuse_owner_names()` and
   `force_reuse_bead_associations_by_prompt()`,
4. called `wipe_names_for_forced_reuse()` to release the reserved names,
5. swapped in the `!`-stripped prompt, and
6. built per-segment `force_reuse_segment_envs` from
   `sase.agent.force_reuse_bead.force_reuse_bead_env()` so the spawned runner receives
   the one-shot `SASE_AGENT_FORCE_REUSE_BEAD` marker.

Commit `0835b38d2` ("feat(ace): migrate Patch and agent producers to durable argv",
2026-08-15) changed `LaunchProcMixin._submit_launch_proc()` to submit argv-only
`python -m sase run` through the durable proc supervisor and to discard the in-process
worker body: `del proc_callable`
(`src/sase/ace/tui/actions/agent_workflow/_launch_procs.py:81`). The sole production
caller, `_launch_resolved_prompt()`, still passes
`proc_callable=lambda: self._run_agent_launch_body(prompt, ctx)`
(`src/sase/ace/tui/actions/agent_workflow/_launch_start.py:209`), but that lambda is
never invoked.

So the whole force-reuse pipeline above became unreachable in production. The raw
submitted prompt — `!` intact — is written into the `RUN_LAUNCH` durable request payload
by `submit_agent_launch()` (`src/sase/ace/tui/actions/agent_durable.py:94`-`120`) and
handed to the child `sase run`. The child
(`sase.main.query_handler._launch.launch_query()` → `launch_agents_from_cwd()`) has no
force-reuse handling at all and validates with the default `allow_force_reuse=False`, so
it rejects every relaunch.

### Evidence

- Reproduced directly with the exact failing prompt from history:
  `validate_launch_name_requests(["%id(!2, clan=sase-op, bead=sase-op.2)\n..."])` raises
  `AgentNameReuseConfirmationRequiredError: Agent name 'sase-op.2' uses forced reuse; confirmation is required.`
- Failed proc logs under `~/.sase/procs/logs/` (`razxvr2v8j6y`, `0h6j1ddpjq9t`,
  `ha7dkypahrcx`, `xq1q7ggmdghd` → `sase-op.2`; `cd5rmekt9kpr` → `research.0o.final`)
  contain exactly that error and nothing else.
- The failure signature in `~/.sase/logs/launch_failures.jsonl` flips at the migration
  date: the last pre-migration force-reuse failure (`260814_170938`) has
  `stage=force_reuse_wipe`, a stage only `run_agent_launch_body()` emits; every failure
  from `260816_084447` onward is `stage=launch_proc` / `"exited with code 1"`.
- Every `%id(!...)` entry in `~/.sase/prompt_history/2608.json` dated `260815_195441` or
  later is recorded `cancelled: true`.
- The prompt-preparation side is healthy: replaying all 20 historical `%id(!...)`
  prompts through `rewrite_force_reuse_name_directives()` + `force_reuse_owner_names()`
  resolves the correct owner name for every one of them. Nothing needs to change in
  `prepare_kill_and_edit_prompt()`
  (`src/sase/ace/tui/actions/agent_workflow/_entry_name_prompts.py:62`) or in
  `sase.xprompt._directive_edit_identity`.

### Why CI stayed green

`tests/ace/tui/test_agent_launch_non_blocking.py:231`
(`test_finish_agent_launch_force_reuse_schedules_original_prompt_and_worker_rewrites`)
captures the discarded `proc_callable` from a test double and calls it by hand. It
therefore asserts against the orphaned in-process body, not against what production
submits. The regression is invisible to it.

## Scope

In scope: forced name reuse works end to end again for every ACE launch surface that
submits `RUN_LAUNCH`, most importantly focused `,x`, marked/bulk `,x`, and any prompt
the user hand-writes with `%id:!name` / `%id(!name, ...)`.

Out of scope: the broader dead-code question. `run_agent_launch_body()`,
`run_single_agent_launch_body()`, and the four TUI-side fan-out dispatchers
(`_launch_bulk.py`, `_launch_multi_prompt.py`, `_launch_multi_model.py`,
`_launch_repeat.py`) are all reachable only through the same discarded `proc_callable`.
Their behavior is duplicated inside `launch_agents_from_cwd_impl()`
(`src/sase/agent/launch_cwd_agents.py`), which is what actually runs, so nothing else is
user-visibly broken — but that whole subtree is orphaned and still tested. Retiring it
is a separate change; this plan only removes the _force-reuse_ duplication (step 5) and
files a follow-up for the rest (step 7).

No feature flag. This restores behavior that shipped and regressed; it is not a new
beta, an early-landed path, or a deprecation.

## Implementation

### 1. Extract the force-reuse launch pipeline into a shared helper

Add `src/sase/agent/force_reuse_launch.py` holding the pipeline in two clearly separated
halves, preserving today's ordering guarantee that _all_ parsing and syntax validation
completes before anything destructive runs (see the comment at
`src/sase/ace/tui/actions/agent_workflow/_launch_body_impl.py:64`):

- A pure planning function, e.g.
  `plan_force_reuse_launch(prompt: str) -> ForceReuseLaunchPlan | None`, returning
  `None` when `rewrite_force_reuse_name_directives(prompt) == prompt` (the common
  no-force-reuse case, so callers pay nothing). Otherwise it runs
  `preflight_launch_name_requests(segments, allow_force_reuse=True)` and returns a
  frozen dataclass carrying the `!`-stripped `rewritten_prompt`, the `owner_names`, and
  the per-segment `segment_envs` built from
  `force_reuse_bead_associations_by_prompt()` + `force_reuse_bead_env()`. Raising
  behavior on invalid prompts must match today: `RuntimeError` subclasses from
  `launch_validation` propagate to the caller.
- A mutating function, e.g. `apply_force_reuse_launch(plan)`, which calls
  `wipe_names_for_forced_reuse(plan.owner_names)`.

Handle the contradiction case explicitly. When the rewrite changes the prompt but
`force_reuse_owner_names()` resolves nothing, no name is ever wiped and the `!` silently
survives into the child, producing the same confusing `confirmation is required`
rejection. This is reachable today with alt fan-out — `_iter_explicit_name_directives()`
bails out via `_prompt_has_launch_fanout()` (`src/sase/agent/launch_validation.py:533`),
so `%id(!2, clan=sase-op)\n%{%m:claude/opus | %m:claude/sonnet}` rewrites but yields no
owner names. Verified: this combination has never worked. Raise a clear, actionable
error naming the limitation instead of letting it fall through.

### 2. Carry the ACE confirmation across the process boundary

Add an explicit authorization field (e.g. `allow_force_reuse: true`) to the `RUN_LAUNCH`
request payload built in `submit_agent_launch()`
(`src/sase/ace/tui/actions/agent_durable.py:106`). The payload reaches the child as the
versioned request sidecar that the proc supervisor writes and `load_request(RUN_LAUNCH)`
reads via `$SASE_PROC_REQUEST_PATH` (`src/sase/ops/cli.py:50`-`85`,
`src/sase/ops/models.py:12`).

This is the right trust boundary: a plain `sase run` from a shell or from an agent skill
has no request sidecar, so it keeps today's behavior and still raises
`AgentNameReuseConfirmationRequiredError`. Note the field participates in
`operation_fingerprint()`, so keep it stable for identical launches.

### 3. Consume the authorization in the child

In `sase.main.query_handler._launch.launch_query()`
(`src/sase/main/query_handler/_launch.py:21`), the payload is already read for `prompt`.
When the payload authorizes forced reuse, call the step-1 helper before
`launch_agents_from_cwd(query)`: plan, apply the wipe, then launch the rewritten prompt
passing the per-segment envs through the existing `segment_extra_env` parameter
(`src/sase/agent/launch_cwd.py:20`-`33`).

Two details:

- `launch_agents_from_cwd_impl()` requires
  `len(segment_extra_env) == len(multi.segments)`
  (`src/sase/agent/launch_cwd_agents.py:113`), where segments are parsed _after_
  `canonicalize_project_aliases_in_prompt()`. Alias canonicalization does not change
  `---` segmentation, so the counts line up, but assert/verify this rather than assuming
  it.
- Failures must stay recoverable. Today a rejected launch reaches
  `record_failed_launch_prompt()` inside `launch_agents_from_cwd_impl()`; a force-reuse
  error raised _before_ that call would skip it. Record the submitted prompt on this new
  failure path too, and keep emitting a typed result via
  `emit_run_launch_result(success=False, ...)` so ACE surfaces a real message instead of
  a bare "exited with code 1".

### 4. Preserve the one-shot bead marker semantics

`SASE_AGENT_FORCE_REUSE_BEAD` is a one-shot authorization consumed by
`sase/axe/run_agent_runner_bootstrap.py:185`. The two segment-env expanders disagree
about fan-out:

- the orphaned body applied the env to the first expanded slot only
  (`env if slot_index == 0 else None`,
  `src/sase/ace/tui/actions/agent_workflow/_launch_body_impl.py:218`);
- `launch_agents_from_cwd_impl()` copies it to every expansion
  (`expanded_segment_extra_env.extend([env] * len(segment_expansions))`,
  `src/sase/agent/launch_cwd_agents.py:133`).

Routing force-reuse envs through `segment_extra_env` therefore hands the same one-shot
marker to every xprompt-swarm slot of a segment. Decide the intended semantics
deliberately — first-slot-only is the behavior that shipped — and encode it, with a test
that pins the choice.

### 5. Remove the duplicate

Re-point the force-reuse block in `run_agent_launch_body()`
(`src/sase/ace/tui/actions/agent_workflow/_launch_body_impl.py:52`-`121`) at the shared
helper so exactly one implementation of the pipeline exists, keeping its existing
`stage="force_reuse_wipe"` failure logging and `LaunchProcOutcome` error surface. Do not
delete the body wholesale here; that is step 7's follow-up.

### 6. Test at the boundary that actually runs

Add regression coverage that would have caught this, exercising the production path
rather than the discarded `proc_callable`:

- `,x` end to end at the seam: a `%id(!…)` prompt from `prepare_kill_and_edit_prompt()`
  submitted through `_launch_resolved_prompt()` produces a `RUN_LAUNCH` payload that
  carries the authorization field; feeding that payload to `launch_query()` wipes the
  name, launches the `!`-stripped prompt, and passes the expected `segment_extra_env`.
  Cover both the clan form (`%id(!2, clan=sase-op, bead=sase-op.2)`) and the family form
  (`%id(!plan, family=sase-oc.4, bead=sase-oc.4)`) — the two shapes
  `prepare_kill_and_edit_prompt()` emits.
- Marked/bulk `,x`: each pane relaunches under its own forced name.
- Negative: `sase run` with no request sidecar, and with a sidecar that does not
  authorize, still raises `AgentNameReuseConfirmationRequiredError`.
- The fan-out contradiction from step 1 raises the new explicit error.
- Fix `tests/ace/tui/test_agent_launch_non_blocking.py:231` so it no longer asserts
  through `task["proc_callable"]()`; it should assert on what `_submit_launch_proc()`
  actually submits.

### 7. File the follow-up

Use `/sase_new_task` to file a task bead for retiring the orphaned TUI-side launch body
subtree (`run_agent_launch_body`, `run_single_agent_launch_body`, `_launch_bulk.py`,
`_launch_multi_prompt.py`, `_launch_multi_model.py`, `_launch_repeat.py`, and the tests
that reach them through `proc_callable`), noting that it is dead in production since
`0835b38d2` and that its live tests are a false-confidence source. Do not do that work
in this change.

## Verification

- `just check` after the change. Because this touches launch validation and the durable
  ops payload contract, also run `just check-full` through `/sase_monitor` before
  landing.
- Manual: from ACE, `,x` a finished agent that owns a clan-member name, submit the
  pre-filled prompt unedited, and confirm the agent relaunches under the same name with
  no proc failure. Repeat with two marked agents. Confirm the relaunch's phase-bead
  association survives (the `bead=` argument in the rewritten `%id`).
- Confirm `~/.sase/logs/launch_failures.jsonl` gains no new `stage=launch_proc` entry
  for the relaunch, and that the corresponding `~/.sase/procs/logs/<proc_id>.log` is
  free of the `confirmation is required` error.
