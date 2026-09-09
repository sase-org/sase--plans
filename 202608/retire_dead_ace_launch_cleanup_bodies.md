---
tier: epic
title: Retire dead ACE in-process launch and cleanup bodies
goal: "ACE's launch and cleanup procs have exactly one implementation each — the durable
  argv path production actually runs — with no orphaned in-process body family, no
  vestigial `proc_callable` parameters, no test double that reaches a code path
  production cannot reach, and no user-facing capability silently dropped on the way.

  "
phases:
  - id: force_reuse
    title: Restore forced name reuse on the durable launch path
    depends_on: []
    size: medium
    description: "force_reuse: extract the `%id:!name` force-reuse launch pipeline out
      of the orphaned TUI body into a shared `sase.agent.force_reuse_launch` module,
      carry ACE's confirmation through the `RUN_LAUNCH` request payload, and consume it
      in the `sase run` child so kill-and-edit relaunches work again before the orphaned
      copy is deleted.

      "
  - id: feedback
    title: Restore MRU and unresolved-reference feedback on the durable launch path
    depends_on: []
    size: small
    description: "feedback: move VCS-xprompt MRU recording and the
      unresolved-xprompt-reference warning toast onto the durable path, so `<ctrl+p>`
      cycling keeps being fed and ACE keeps warning about unknown `#refs` once the
      orphaned body is deleted.

      "
  - id: cleanup_retire
    title: Retire the cleanup worker bodies and their proc_callable seam
    depends_on: []
    size: medium
    description: "cleanup_retire: delete the dead `_worker` closures behind kill,
      dismiss, and save persistence, drop `proc_callable` from `_submit_cleanup_proc`,
      and re-point the shared cleanup test harness at the durable persist-cleanup
      payload seam.

      "
  - id: launch_retire
    title: Retire the in-process launch body and fan-out dispatchers
    depends_on:
      - force_reuse
      - feedback
    size: medium
    description: "launch_retire: delete `run_agent_launch_body`,
      `run_single_agent_launch_body`, the four fan-out dispatcher mixins, and
      `proc_callable` on `_submit_launch_proc`, then delete or re-point every test that
      reached them through the discarded callable.

      "
  - id: support_retire
    title: Retire the launch-body support modules the deletion orphans
    depends_on:
      - launch_retire
    size: medium
    description: "support_retire: delete the second-order orphans the body deletion
      leaves behind — the background-spawn bridge, launch-history helpers, TUI workflow
      executor, and the ref-resolution mixin — resolving each against symvision rather
      than by assumption.

      "
  - id: sweep
    title: Final orphan sweep, full verification, and follow-ups
    depends_on:
      - cleanup_retire
      - support_retire
    size: small
    description:
      "sweep: re-run the whole-repo lint and full test suite, resolve any remaining
      orphan reported by symvision, and file the follow-up beads for the capabilities
      this epic deliberately leaves broken rather than restores."
proposed_by: bbugyi200.athena.sase-ng
parent_bead: sase-ng
status: done
bead_id: sase-ng.1
create_time: 2026-09-09 19:51:28
---

- **PROMPT:**
  [prompts/202608/retire_dead_ace_launch_cleanup_bodies.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/retire_dead_ace_launch_cleanup_bodies.md)
- **BEAD:**
  [sase-ng.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ng/sase-ng.1.md)

# Plan: Retire dead ACE in-process launch and cleanup bodies

## Problem

Commit `0835b38d2` migrated ACE's launch and cleanup producers to durable argv
submission. Both submission helpers still accept a `proc_callable` and both delete it
before submitting:

- `LaunchProcMixin._submit_launch_proc()` — `del proc_callable`, then
  `submit_agent_launch()` (`src/sase/ace/tui/actions/agent_workflow/_launch_procs.py`).
- `CleanupProcMixin._submit_cleanup_proc()` — `del proc_callable`, then
  `submit_agent_cleanup()` (`src/sase/ace/tui/actions/agents/_cleanup_procs.py`).

Everything reachable only through those two discarded callables is dead in production:

**Launch family.** `_launch_start.py` is the sole production caller and passes
`proc_callable=lambda: self._run_agent_launch_body(prompt, ctx)`. That lambda is never
invoked, so `run_agent_launch_body()` (`_launch_body_impl.py`),
`run_single_agent_launch_body()` (`_launch_body_single.py`), and the four fan-out
dispatchers they reach by `app.call_later` — `_launch_bulk_agents`,
`_launch_multi_prompt_agents`, `_launch_multi_model_agents`, `_launch_repeat_agents` —
never run. What runs instead is the child `sase run` process:
`handle_run_special_cases([])` reads the prompt from the `RUN_LAUNCH` request sidecar,
calls `launch_query()`, and `launch_agents_from_cwd_impl()`
(`src/sase/agent/launch_cwd_agents.py`) re-implements single, multi-prompt, fan-out, and
repeat dispatch.

**Cleanup family.** Every `_submit_cleanup_proc()` call site in `_kill_procs.py`,
`_dismissing.py`, and `_marking.py` builds a `_worker` closure that is discarded. The
serialized payload travels to `sase agent persist-cleanup`, whose
`_apply_cleanup_payload()` (`src/sase/ops/commands/agent.py`) calls the same
`persist_*_transaction` functions the closures call.

The tests are the problem this bead exists to remove: they capture the discarded
`proc_callable` from a test double and call it by hand, so they assert against the
orphaned copy rather than against what production submits. That is exactly how the stale
`_prompt_context` bug (fixed by `2aa8ba26f`) hid, and it is how forced name reuse
regressed while `tests/ace/tui/test_agent_launch_non_blocking.py` stayed green.

## Scope and the two preconditions

Deleting the orphaned subtree is not a pure subtraction. Three capabilities live
**only** inside it, and symvision will force their supporting symbols out of the tree
along with it. Two of them are still consumed by live code and must be re-pointed at the
durable path **before** the deletion; the rest are already-broken behavior this epic
records rather than repairs.

Must be preserved (phases `force_reuse` and `feedback`):

1. **Forced name reuse (`%id:!name`, ACE's `,x` kill-and-edit).** The pipeline lives in
   `run_agent_launch_body()` and is the only in-repo consumer of
   `preflight_launch_name_requests`, `wipe_names_for_forced_reuse`,
   `force_reuse_bead_associations_by_prompt`, and `force_reuse_bead_env`. Deleting the
   body without re-pointing them deletes forced reuse from the product. This is already
   a live production regression, and the written plan for it
   (`sase/repos/plans/202608/kill_and_edit_force_reuse.md`, steps 1–4 and 6) has **not**
   landed on master — there is no `src/sase/agent/force_reuse_launch.py` and no
   `allow_force_reuse` field in the `RUN_LAUNCH` payload. That plan's step 7 is this
   bead, so its steps 1–4 and 6 are this epic's first phase.
2. **VCS-xprompt MRU and the unresolved-reference toast.** `record_vcs_xprompt_usage()`
   has exactly one writer, `_launch_history.record_launched_vcs_xprompt_usage()`, inside
   the dead body — while `load_launchable_vcs_xprompt_mru()` is still read by `<ctrl+p>`
   cycling (`_vcs_mru_cycling.py`) and the custom-agent picker (`_entry_custom.py`).
   `LaunchProcOutcome.warning_messages` likewise has no producer outside the dead body.

Deliberately **not** restored here (recorded as follow-ups in `sweep`):

- **Bulk launch for marked Patches.** `_start_agents_from_marked()` (leader-mode `A`)
  fills `_bulk_patches`, whose only reader is the dead body. Production submits one
  `sase run` for the shared prompt, so N marked Patches already launch one agent. Making
  the durable path fan out per Patch is new feature work.
- **Replayable VCS selection updated from the submitted prompt.**
  `save_replayable_vcs_selection()` refreshes `Ctrl+Space`'s target from the ref the
  user actually cycled to. It mutates in-memory ACE state, so it cannot move into the
  child process; `save_last_agent_selection_if_launchable()` keeps its other live
  callers, so only the launch-time refresh is lost.
- **TUI-side standalone workflow execution.**
  `WorkflowExecMixin._try_execute_workflow()` is reachable only from
  `run_single_agent_launch_body()`. `sase run "#!name"` routes through `launch_query()`
  for both CLI and ACE today, so the workflow runs inside the spawned agent instead of
  in a TUI daemon thread. This is superseded, not lost.

No feature flag: this removes unreachable code and restores behavior that shipped and
regressed. It is not a new beta, an early-landed path, or a deprecation whose old branch
must stay reachable.

## Restore forced name reuse on the durable launch path

Implements steps 1–4 and 6 of `sase/repos/plans/202608/kill_and_edit_force_reuse.md`.
Read that plan first; it carries the reproduction evidence and the failure-log forensics
and this section does not repeat them.

Add `src/sase/agent/force_reuse_launch.py` with the pipeline split into a pure half and
a mutating half, preserving today's ordering guarantee that all parsing and syntax
validation completes before anything destructive runs:

- `plan_force_reuse_launch(prompt) -> ForceReuseLaunchPlan | None` returns `None` when
  `rewrite_force_reuse_name_directives(prompt) == prompt`, so the common case costs
  nothing. Otherwise it runs
  `preflight_launch_name_requests(segments, allow_force_reuse=True)` on the
  `parse_multi_prompt()` segments and returns a frozen dataclass carrying the
  `!`-stripped `rewritten_prompt`, the `owner_names` from `force_reuse_owner_names()`,
  and per-segment envs built from `force_reuse_bead_associations_by_prompt()` and
  `force_reuse_bead_env()`. `RuntimeError` subclasses from `launch_validation` propagate
  unchanged.
- `apply_force_reuse_launch(plan)` calls
  `wipe_names_for_forced_reuse(plan.owner_names)`.

Handle the contradiction case explicitly: when the rewrite changes the prompt but
`force_reuse_owner_names()` resolves nothing (reachable with alt fan-out, because
`_iter_explicit_name_directives()` bails out via `_prompt_has_launch_fanout()`), no name
is ever wiped and the `!` survives into the child. Raise a clear, actionable error
naming the limitation rather than letting it fall through to a confusing
`confirmation is required` rejection.

Carry ACE's confirmation across the process boundary by adding an explicit authorization
field (`allow_force_reuse: true`) to the `RUN_LAUNCH` payload built in
`submit_agent_launch()` (`src/sase/ace/tui/actions/agent_durable.py`). A plain
`sase run` from a shell or an agent skill has no request sidecar, so it keeps today's
behavior and still raises `AgentNameReuseConfirmationRequiredError`. The field
participates in `operation_fingerprint()`, so keep it stable for identical launches.

Consume it in `launch_query()` (`src/sase/main/query_handler/_launch.py`), which already
reads the payload for `prompt`: plan, apply the wipe, then launch the rewritten prompt,
passing per-segment envs through `launch_agents_from_cwd`'s existing `segment_extra_env`
parameter. Two details:

- `launch_agents_from_cwd_impl()` requires
  `len(segment_extra_env) == len(multi.segments)` where segments are parsed _after_
  `canonicalize_project_aliases_in_prompt()`. Verify that alias canonicalization does
  not change `---` segmentation rather than assuming it.
- A force-reuse error raised before `launch_agents_from_cwd()` would skip the
  `record_failed_launch_prompt()` inside it. Record the submitted prompt on this new
  failure path and emit `emit_run_launch_result(success=False, ...)` so ACE surfaces a
  real message instead of a bare "exited with code 1".

`SASE_AGENT_FORCE_REUSE_BEAD` is a one-shot authorization consumed by
`sase/axe/run_agent_runner_bootstrap.py`. The orphaned body applied it to the first
expanded slot only (`env if slot_index == 0 else None`), while
`launch_agents_from_cwd_impl()` copies each segment env to every xprompt-swarm
expansion. First-slot-only is the behavior that shipped: encode it deliberately and pin
it with a test.

Test at the boundary that actually runs, not through `proc_callable`:

- A `%id(!…)` prompt from `prepare_kill_and_edit_prompt()` submitted through
  `_launch_resolved_prompt()` produces a `RUN_LAUNCH` payload carrying the authorization
  field; feeding that payload to `launch_query()` wipes the name, launches the
  `!`-stripped prompt, and passes the expected `segment_extra_env`. Cover both shapes
  `prepare_kill_and_edit_prompt()` emits: the clan form
  (`%id(!2, clan=sase-op, bead=sase-op.2)`) and the family form
  (`%id(!plan, family=sase-oc.4, bead=sase-oc.4)`).
- Multi-prompt `,x`: each segment relaunches under its own forced name, with the
  per-segment bead markers threaded correctly.
- Negative: `sase run` with no request sidecar, and with a sidecar that does not
  authorize, still raises `AgentNameReuseConfirmationRequiredError`.
- The alt-fan-out contradiction raises the new explicit error.

Do **not** delete `run_agent_launch_body()` in this phase — re-point its force-reuse
block at the shared helper so exactly one implementation exists. `launch_retire` deletes
the body wholesale.

## Restore MRU and unresolved-reference feedback on the durable launch path

Two small pieces of ACE-visible feedback exist only inside the dead body. Move both onto
the durable path so `launch_retire` and `support_retire` can delete their old homes
without silently degrading the UI.

**VCS-xprompt MRU.** After a successful launch in `launch_query()`, record the leading
VCS prefix so `<ctrl+p>` cycling and the custom-agent picker keep being fed. Extract the
prefix lexically from the submitted query with `get_ref_patterns()` — the same match the
orphaned multi-prompt path used — and hand `#<workflow_type>:<ref>` to
`record_vcs_xprompt_usage()`. Do not re-resolve the ref: `record_vcs_xprompt_usage()`
already drops the implicit default prefix, stale known-project prefixes, and
provider-mismatched prefixes internally, and `load_launchable_vcs_xprompt_mru()` prunes
non-launchable entries again at read time, so the `_is_launchable_replay_project()`
guard the orphaned wrapper added is redundant. Recording after the launch succeeds
(rather than before the spawn, as the dead code did) is the intended behavior. Cover
that a plain prompt with no ref records nothing, that `#git:home` is not recorded, and
that an explicit ref is recorded once.

**Unresolved-reference warnings.** `launch_query()` already computes
`scan_query_for_unresolved_references(query)` and prints each warning to the proc log.
Also put `format_unresolved_references_toast(names)` into the `emit_run_launch_result()`
payload under `warning_messages`, and have `_launch_outcome_from_completion()`
(`_launch_procs.py`) read that key into `LaunchProcOutcome.warning_messages`.
`_on_launch_proc_complete()` already surfaces those as warning toasts, so this restores
the toast that `test_launch_task_completion_emits_warning_messages` guards while leaving
the `with_warning_messages()` builder free to die in `support_retire`.

## Retire the cleanup worker bodies and their proc_callable seam

Independent of the launch phases; can land in parallel with them.

Delete the discarded `_worker` closures and pass no callable:

- `src/sase/ace/tui/actions/agents/_kill_procs.py` — both closures
  (`_submit_bulk_kill_persistence_proc`, `_submit_kill_persistence_proc`).
- `src/sase/ace/tui/actions/agents/_dismissing.py` — both closures (bulk dismiss and
  single dismiss).
- `src/sase/ace/tui/actions/agents/_marking.py` — the group-save closure.
- `src/sase/ace/tui/actions/agents/_cleanup_procs.py` — drop the `proc_callable`
  parameter and its `del`, and the now-stale `Callable` import if nothing else needs it.

Each closure's `on_settled` release, its `_*_inflight` bookkeeping, its `log.debug`
timing line, and its error-path `CleanupProcOutcome` must be re-examined rather than
deleted by reflex: `on_settled` is already wired through `submit_agent_cleanup()` and
stays, but the closure's `finally:` block was a second release path and its `except`
branch produced the `severity="error"` outcome that the durable path now produces from
`_run_persist_cleanup()`'s result payload. Confirm the payload already carries an
equivalent failure surface before dropping each one.

One behavioral difference is real and must be recorded, not papered over: the closures
passed `register_expected_deletion=self._register_expected_agent_artifact_deletion` into
`persist_single_kill_transaction` / `persist_bulk_kill_transaction`, and the
out-of-process `_apply_cleanup_payload()` cannot. If a test asserts on that
registration, it is asserting on a code path production does not run; delete or rewrite
it against what the durable path does, and note the difference in the phase result.

Migrate the shared harness so every downstream cleanup test exercises the real payload:

- `tests/_agent_cleanup_proc_helpers.py` — delete
  `TrackedProcRecorderMixin._submit_cleanup_proc` so the real
  `CleanupProcMixin._submit_cleanup_proc` runs, building the request payload and calling
  `submit_agent_cleanup()` into the harness's existing `_submit_durable_proc`. Give that
  stub a `live_body` that applies the payload through
  `sase.ops.commands.agent._apply_cleanup_payload`, i.e. the exact function
  `sase agent persist-cleanup` runs. This makes the payload serialize/deserialize round
  trip — `serialize_agents`, `json_identities`, `agent_cleanup_wire_to_json_dict`,
  `saved_agent_group_wire_to_json_dict` and their inverses — part of the assertion for
  the first time.
- The ~13 files that use `TrackedProcRecorderMixin` / `run_tracked_proc`
  (`tests/test_agent_kill_*.py`, `tests/test_agent_dismiss_persistence.py`,
  `tests/test_dismissed_agent_lifecycle.py`, `tests/ace/tui/test_agent_marking_save.py`,
  `tests/ace/tui/test_agents_tab_completion_dismiss_e2e.py`,
  `tests/_agent_dismiss_helpers.py`, `tests/ace/tui/_agent_marking_helpers.py`, and
  peers) should then need little or no change. Where one does, prefer fixing the
  assertion over reintroducing a bypass.
- `tests/ace/tui/test_agent_cleanup_procs.py` — drop the `proc_callable` doubles and
  assert on the submitted payload and the completion effects.
- `tests/test_agent_group_revival_e2e.py` — `_patch_local_cleanup_submit` monkeypatches
  `_submit_cleanup_proc` with a `proc_callable`-invoking stub; re-point it at the same
  payload seam.

Leave `_submit_session_worker`'s `proc_callable` alone. It is a live parameter with real
production callers, and the doubles in `tests/_plan_approval_tui_helpers.py`,
`tests/test_tui_plan_epic_approval.py`, `tests/ace/tui/_agent_wait_resume_helpers.py`,
and `tests/test_agent_launch_validation.py` legitimately drive it.

## Retire the in-process launch body and fan-out dispatchers

Runs after `force_reuse` and `feedback`, so no capability is deleted before its
replacement exists.

Delete these modules outright — every one is reachable only from the discarded
`proc_callable`:

- `_launch_body.py` (`AgentLaunchBodyMixin`, `_run_agent_launch_body`,
  `_run_agent_launch_body_async`)
- `_launch_body_impl.py` (`run_agent_launch_body`, `_merge_extra_env`)
- `_launch_body_single.py` (`run_single_agent_launch_body`)
- `_launch_bulk.py` (`BulkLaunchMixin`, `_run_bulk_launch`, `_log_bulk_item_failure`)
- `_launch_multi_prompt.py` (`MultiPromptLaunchMixin` and its failure helpers)
- `_launch_multi_model.py` (`MultiModelLaunchMixin`, the fan-out failure report writer,
  and its `ViewErrorReport` notification)
- `_launch_repeat.py` (`RepeatLaunchMixin`, `_log_repeat_failure`,
  `_REPEAT_SPAWN_SLEEP`)

Then:

- `_agent_launch.py` — drop the deleted mixins from `AgentLaunchMixin`'s bases and the
  imports; keep `AgentLaunchStartMixin`, `LaunchDeltaMixin`, and `LaunchProcMixin`.
- `_launch_procs.py` — delete the `proc_callable` parameter, its `del`, and the
  docstring paragraph that explains it.
- `_launch_start.py` — drop the `proc_callable=` argument and rewrite the comment block
  that still describes "the vestigial in-process test callable" operating on the `ctx`
  snapshot. The `keep_bar` snapshot itself stays: it is why `ctx` exists.
- `_launch_background.py`, `_launch_history.py`, `_workflow_exec.py` and the
  ref-resolution mixin become orphans here but are `support_retire`'s job; leaving them
  briefly unreferenced between phases is expected.

Tests. Delete the files whose entire subject is the deleted code:

- `tests/ace/tui/test_agent_launch_dispatch.py`
- `tests/ace/tui/test_launch_repeat_bulk.py`
- `tests/ace/tui/test_launch_multi_prompt.py`
- `tests/ace/tui/test_launch_multi_model.py`
- `tests/ace/tui/test_prompt_stack_launch_integration.py`
- `tests/test_agent_launch_repeat.py`
- `tests/ace/tui/agent_launch_vcs/` — delete the package, **except** the direct
  `resolve_ref_from_prompt()` unit tests in `test_resolution.py`, which exercise a
  function `sase.agent.launch_cwd_agents` and `sase.agent.multi_prompt_vcs` still call.
  Move those into a module that does not import the deleted harness.

Trim rather than delete:

- `tests/ace/tui/_agent_launch_helpers.py` — drop `_LaunchBodyApp`,
  `_run_launch_body_with_common_patches`, `_launch_body_context`, and `_FakeApp`'s
  `_run_agent_launch_body` / `proc_callable` recording; keep the `_FakeApp` submit
  recorder the still-live prompt-submit tests use.
- `tests/ace/tui/_launch_fan_out_helpers.py` — drop `_MultiPromptApp`, `_MultiModelApp`,
  `_RepeatApp`, `_BulkApp`, `_FanOutHarness`, and `_FakeMultiPrompt`; keep
  `_CoalesceApp`, `_LaunchDeltaApp`, `_ctx`, and `_launch_result` if other tests still
  use them.
- `tests/ace/tui/test_agent_launch_non_blocking.py` — keep
  `test_launch_task_completion_emits_warning_messages` (now guarding the payload-fed
  toast from `feedback`) and delete the `_run_agent_launch_body_async` tests. The seven
  force-reuse tests here are `force_reuse`'s replacement coverage: confirm the new
  boundary tests assert the same behavior before deleting each, and say so explicitly in
  the phase result rather than dropping them silently.
- `tests/ace/tui/test_launch_failure_logging.py` — remove the `_run_*_launch` and
  `_run_agent_launch_body_async` cases; whatever remains that tests
  `sase.logs.log_launch_failure` for live paths stays.
- `tests/ace/tui/test_prompt_bar_stack_submit_handlers.py` and
  `tests/ace/tui/test_prompt_input_collection_launch.py` — these test live submit
  behavior (pane stacking, input collection) and only reach through
  `task["proc_callable"]()` to observe the effect. Re-point each assertion at the
  submitted prompt and payload recorded by `_submit_launch_proc`.

Every deletion must be justified by what production does instead. Before deleting a
test, state which durable-path test covers the same behavior, or that the behavior is
one of the three capabilities `sweep` files a follow-up for.

## Retire the launch-body support modules the deletion orphans

Deleting the body orphans a second ring of modules. Resolve each against
`just _lint-symvision` output rather than by assumption, following the decision
hierarchy in `sase/memory/symvision.md`: delete first, make private second, pragma only
for a real consumer symvision cannot see, and never whitelist to silence.

Expected orphans, each to be confirmed:

- `_launch_background.py` (`BackgroundAgentLaunchMixin._launch_background_agent`) — a
  thin bridge to `sase.agent.launcher.spawn_agent_subprocess`, which keeps its live
  callers. Delete the mixin and drop it from `AgentLaunchMixin`.
- `_launch_history.py` — after `feedback` moves MRU recording,
  `record_launched_vcs_xprompt_usage`, `record_resolved_vcs_xprompt_usage`,
  `record_prompt_file_references`, and `save_replayable_vcs_selection` all lose their
  callers. `record_file_references` keeps its live caller in `_prompt_bar_mount.py`, and
  `save_last_agent_selection_if_launchable` keeps its callers in
  `_entry_quick_launch.py`, `_entry_relaunch.py`, and `_entry_custom.py`, so only this
  module dies.
- `_workflow_exec.py` (`WorkflowExecMixin`) — drop it from `AgentWorkflowMixin` in
  `agent_workflow/__init__.py` and delete
  `tests/ace/tui/test_try_execute_workflow_vcs_ref.py` plus the workflow-exec cases in
  `tests/ace/tui/test_failed_launch_stash.py` and
  `tests/ace/tui/test_launch_failure_logging.py`. Check whether
  `src/sase/axe/run_workflow_runner.py` still has a non-test caller once
  `_launch_workflow_subprocess()` is gone; if it does not, delete it and
  `tests/test_run_workflow_visibility.py` with it, and drop its entry from
  `tests/test_agent_artifact_marker_mutation_audit.py`.
- `_ref_resolution.py` — `RefResolutionMixin._resolve_vcs_from_prompt` and
  `strip_all_vcs_refs()` lose their only callers. Delete both and drop
  `RefResolutionMixin` from `AgentWorkflowMixin`; **keep** the module-level
  `resolve_ref_from_prompt()`, which `sase.agent.launch_cwd_agents` and
  `sase.agent.multi_prompt_vcs` call.
- `_launch_procs.py` — `LaunchProcOutcome.with_warning_messages()` loses its producer.
  Delete the builder; keep the `warning_messages` field, which `feedback` now populates
  from the result payload.
- `_launch_bulk`'s consumers of `_bulk_patches` are gone, so
  `AgentLaunchMixin._bulk_patches`, `_agent_launch.py`'s declaration, and
  `_entry_points.py`'s `_bulk_changespecs` compatibility property have no reader. Leave
  `_entry_bulk._start_agents_from_marked()` and its leader-mode binding in place —
  removing a bound user-facing action is out of scope — and file the follow-up in
  `sweep`.

Do not chase orphans into `sase/agent/` on reflex: `validate_launch_name_requests`,
`force_reuse_owner_names`, `rewrite_force_reuse_name_directives`,
`spawn_agent_subprocess`, `launch_multi_prompt_agents`, `execute_launch_plan`, and
`plan_fake_fanout` all keep live production callers, and `force_reuse`'s new module
keeps `preflight_launch_name_requests`, `wipe_names_for_forced_reuse`,
`force_reuse_bead_associations_by_prompt`, and `force_reuse_bead_env` alive. If
symvision reports one of those as unused, that is a signal `force_reuse` did not land
its wiring — investigate before deleting.

## Final orphan sweep, full verification, and follow-ups

Run `just install` first — ephemeral workspaces drift. Then:

- `just check` for the whole-repo lint gates, and `just _lint-symvision` on its own
  while iterating on the orphan set.
- `just check-full` through `/sase_monitor` (never inline) with a `--next` action. This
  change touches the launch and cleanup producer contracts and deletes across ten
  modules, so the scoped test lane is not sufficient evidence.
- Confirm no `proc_callable` remains on `_submit_launch_proc` or `_submit_cleanup_proc`
  in either `src/` or `tests/`, and that the only surviving `proc_callable` references
  belong to `_submit_session_worker`.
- Manual smoke from ACE, since the whole point is that tests could not see these paths:
  submit a plain prompt, a multi-prompt, a `%r:2` repeat, a `%{a | b}` fan-out, and a
  `,x` kill-and-edit relaunch of a named agent; kill one agent and dismiss another.
  Confirm each lands, and that `<ctrl+p>` offers the ref just launched.

Then file follow-up task beads with `/sase_new_task`, each naming `sase-ng` as the
originating bead and the commit that removed the code so it is recoverable from history:

- Bulk launch for marked Patches launches one agent instead of N, because
  `_bulk_patches` lost its only reader; the durable path needs a per-Patch fan-out.
- `Ctrl+Space` replay target is no longer refreshed from the submitted prompt, so
  cycling with `<ctrl+p>` before submitting does not update it.

## Verification

Per phase: `just check` after any file change, plus the phase's own targeted tests.

Before landing the epic's combined tree: `just check-full` through `/sase_monitor`, and
the manual ACE smoke list above. The combined tree touches the broadening set — durable
operation payloads, launch validation, and agent cleanup persistence — so the scoped
lane alone is explicitly not enough here.
