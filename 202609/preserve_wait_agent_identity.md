---
tier: tale
title: Preserve agent identity through wait edits and runner refreshes
goal:
  Keep assigned agent names stable across wait-driven relaunches and automatic runner
  refreshes so existing downstream dependencies continue to resolve.
size: medium
proposed_by: bbugyi200.athena.0i7
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0i7](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0i7.md)
- **COMMITS:**
  - [a723858](https://github.com/sase-org/sase/commit/a723858f27dc48fe42ea849b71f69527046811a0)
    — fix(agent): preserve wait agent identity

# Preserve agent names through wait edits and runner refreshes

## Outcome and scope

Once a launch has acquired a concrete name, changing its waits or automatically
refreshing its runner must preserve that name. For example, an agent launched as
`builder.w0` remains `builder.w0` when its dependency becomes `reviewer`, when a timer
or runner limit is added, and when the runner reloads updated SASE code after waiting.
An existing agent waiting for `builder.w0` must still resolve that same logical agent
and observe its eventual successful completion.

This is one medium tale: fix two lifecycle paths, reuse the existing naming and relaunch
infrastructure, and add regression coverage. Initial `.w<N>` allocation and deliberate
creation of new agents retain their existing behavior. General kill-and-edit naming,
explicit user renames, and historical repair of agents already renamed are outside this
change. No new CLI option or keymap is needed.

## Diagnosis established before implementation

The dependency-edit suspicion is partly correct. There are two independently reproduced
identity-loss paths; a dependency edit is not necessary for both.

### 1. Wait editing sometimes launches a replacement without its current identity

In `src/sase/ace/tui/actions/agents/_wait_actions.py`:

- `_apply_wait` persists ordinary dependency/bead/priority edits on a WAITING row in
  place through `submit_agent_directive`. This path does not allocate a name.
- `_apply_wait_running` sends STARTING/RUNNING rows to `_apply_wait_relaunch`.
- `_apply_wait` also chooses relaunch for a time condition, an applicable runner limit,
  or an agent dependency edit on a row already holding a runner-slot request.
  Runner-only changes on an existing slot request can instead update in place.
- `_apply_wait_relaunch` reads the stored raw prompt, calls `_do_kill_agent`, edits only
  `%wait`/`%queue`, then calls `_finish_agent_launch`. The row's `agent_name` is never
  carried into the replacement prompt. Its `display_name` only supplies UI context; it
  does not constrain naming.

`PlannedNameAllocator.planned_name_for_prompt` in
`src/sase/agent/multi_prompt_reference_allocator.py` treats that prompt as a new launch.
The child has equivalent allocation paths in
`src/sase/axe/run_agent_directive_identity.py`.

An isolated probe called the real wait handlers and name planner while replacing
kill/launch/persistence with captures and supplying an in-memory reservation set. With
`old.w0` reserved, it observed:

| Edit                                                   | Current behavior                |
| ------------------------------------------------------ | ------------------------------- |
| RUNNING: change `old` dependency to `new`              | Relaunch as `new.w0`            |
| RUNNING: resubmit dependency `old`                     | Relaunch as `old.w1`            |
| WAITING: add a timer or runner limit, retaining `old`  | Relaunch as `old.w1`            |
| WAITING: replace dependency with a timer only          | Relaunch with a plain auto-name |
| WAITING: dependency-only change to `new`               | Update in place; keep `old.w0`  |
| QUEUED with a slot request: change dependency to `new` | Relaunch as `new.w0`            |
| QUEUED with a slot request: runner-only edit           | Update in place; keep `old.w0`  |

The current relaunch also omits cleanup/launch ordering. `_do_kill_agent` starts durable
cleanup asynchronously; this caller ignores its boolean result and does not use its
`on_settled` callback. Existing kill-and-edit code uses `_relaunch_barrier.py` to stop
late cleanup from racing a replacement name claim. This must be addressed when the wait
path starts reusing names.

Existing tests `test_apply_wait_with_time_relaunches_with_replacement_directive` and
`test_apply_wait_running_relaunches_with_canonical_wait` in
`tests/ace/tui/test_agent_wait_resume.py` check prompt editing but omit identity
preservation; the latter even gives the row and stored `%id` different names.

### 2. Automatic post-wait code refresh can allocate the name again

`_run_agent` in `src/sase/axe/run_agent_runner.py` bootstraps the agent, waits, then
calls `refresh_runner_code_after_wait`. When the editable SASE source HEAD changed
during a blocking wait, `run_agent_runner_refresh.py` recreates the submitted prompt and
calls `os.execv`. The same run then bootstraps and resolves identity again.

Names resolved only in the child have no `SASE_AGENT_PLANNED_NAME`; parent planning
deliberately defers prompts with unresolved non-resume xprompts. The environment has
`SASE_AGENT_NAME` after claiming, but `prepare_agent_name_request` does not use it.
`preserved_agent_metadata` also omits `name`. Consequently, a refresh can allocate the
next wait-derived name even though the existing name belongs to this same artifacts
directory.

A second isolated probe exercised the actual identity resolver with a refresh marker,
`SASE_AGENT_NAME=old.w0`, and `old.w0` already reserved. It selected `old.w1` without a
parent-planned name. Supplying `SASE_AGENT_PLANNED_NAME=old.w0` with matching artifact
ownership preserved `old.w0`. The probe mocked metadata writes and claims; it did not
run live agents or an actual exec.

These are code-level reproductions, not attribution of an individual historical
incident. The automatic refresh path explains renames without any `w` edit.

## Implementation

### A. Preserve the current run's identity across automatic refresh

1. Carry the successfully resolved `state.agent_name` from `_run_agent` into the refresh
   handoff. Before exec, bind that concrete name to the same run using the existing
   planned-name transport and artifact-ownership validation. It must work when the
   parent could not preallocate a name and when a stale parent hint differs from the
   name the child actually claimed.
2. Keep `_planned_name_is_reserved_for_artifacts` and the existing same-owner claim
   checks. A name from another artifacts directory must never be adopted merely because
   it is present in the environment. Do not globally prioritize inherited
   `SASE_AGENT_NAME` for new launches.
3. Treat re-exec as continuation of this run, with no forced-reuse wipe. Ensure
   allocation, metadata `name`, registry ownership, and the environment agree; copying
   only the old metadata field over a newly allocated identity is not a fix. Preserve
   already-planned and explicit names as well as child-derived wait names. Re-expansion
   must not reinterpret a known run as a new allocation.
4. If prompt restoration or exec fails, restore any transport environment changed by the
   attempted handoff. Retain the existing one-shot refresh behavior and
   `wait_completed_at` fast path. Ordinary child launches must not inherit a
   continuation identity; verify the launch environment cleanup path.

Primary files: `run_agent_runner.py`, `run_agent_runner_refresh.py`, and, if necessary
for transport consumption, `run_agent_directive_identity.py` and
`run_agent_runner_bootstrap.py` under `src/sase/axe/`.

### B. Preserve identity for wait-driven replacement launches

1. Prepare a replacement prompt from the selected concrete row before killing anything.
   Retain canonical `%wait`/`%queue` editing, but also carry the current resolved name
   rather than deriving one from the edited dependencies. For a standalone `old.w0`, the
   existing `ensure_forced_name_reuse` helper produces `%id:!old.w0` even when the
   original prompt has no named `%id`.
2. Reuse the identity-aware helpers in `src/sase/agent/relaunch_prompt.py` and the
   family metadata adaptation in
   `src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py`. Preserve concrete names
   from templates and `%i` aliases. Keep family-root versus exact-member semantics,
   owner-qualified names, clan membership, tribe, and bead metadata. Never write a full
   `--suffix` shell name as a new standalone user ID. `prepare_kill_and_edit_prompt`
   intentionally leaves unnamed standalone prompts untouched, so calling it alone is
   insufficient; explicitly pin auto-named wait replacements without changing general
   kill-and-edit behavior.
3. Perform prompt loading, rewriting, and preflight before the destructive step, using
   the existing background prompt-resolution helper. Re-resolve the row after
   asynchronous preparation. Missing/invalid prompts or unsupported container/fan-out
   targets must leave the source agent intact with an actionable error, not silently
   allocate a new name.
4. Integrate the existing relaunch cleanup barrier and `RelaunchOperation`/prompt
   session association. Pass settlement into `_do_kill_agent`; honor a false return and
   do not submit a replacement then. The barrier must belong to the replacement
   submission, not be an unassociated barrier it can bypass. Keep durable cleanup and
   launch off the UI thread. Handle cancellation and failed submission without replaying
   the replacement or leaking a barrier.
5. Route the prepared prompt through the existing trusted forced-reuse launch path so
   old reservations cannot cause a collision. Do not delete registry entries or rewrite
   downstream wait strings in the TUI. Check that cleanup leaves independent agents
   waiting on the old name intact, including a waiter whose auto-name is `old.w0.w0`.
6. Leave in-place wait edits and run-now behavior in place and add identity assertions
   for them. Update the Agents help text in
   `src/sase/ace/tui/modals/help_modal/agents_bindings.py` to explain that wait edits
   preserve the agent name, respecting its existing width limits.

### Backend boundary

These repairs should primarily connect existing mechanisms: runtime refresh transport
and Textual relaunch orchestration. Do not introduce a second name allocator, dependency
resolver, or ownership policy in Python. Reuse the existing Rust-backed identity/name
APIs and thin adapters. If implementing a missing shared ownership/continuation decision
proves necessary, put that decision and its tests in `sase-core`, expose it through
`sase_core_rs`, and adapt the Python caller. Open that repo with `/sase_repo` and use
only its returned checkout path. Do not migrate unrelated naming code or alter
release-managed crate versions.

## Regression coverage and acceptance

Use temporary isolated agent stores and mocked process/provider boundaries; do not
launch, stop, or rename the user's real agents while testing.

1. Add a refresh regression that begins without a parent-planned name, resolves
   `%wait:builder` to `builder.w0`, simulates the refresh handoff, and repeats
   bootstrap/identity resolution against the same artifact store. Assert that metadata
   and lookup still expose only `builder.w0` for that run, that no new `.w1` claim
   occurs, and that a completed wait is not repeated. Include the unresolved-xprompt
   parent-planning route that makes this condition realistic.
2. Cover already-planned names, foreign/stale artifact reservations, no refresh,
   prompt-file/exec failure rollback, and a fresh nested launch receiving its own name.
   Extend `tests/test_run_agent_runner_refresh.py`,
   `tests/test_agent_names_extract_naming.py`, and the relevant bootstrap tests.
3. Parameterize wait-action coverage over running/starting, waiting with a timer or
   runner limit, and queued dependency edits. Cover switching dependencies, retaining
   one, replacing one with several, and timer-only waits. Assert the replacement's
   resolved name, not merely the presence of a `%id` string.
4. Exercise an auto-named prompt with no `%id`, an explicit name, a resolved template,
   and representative family/clan identity forms. Preserve the existing unrelated
   metadata. Update the inconsistent row-versus-prompt test fixture.
5. Model asynchronous kill persistence in the wait-action fake or a launch-seam test: no
   replacement is submitted before cleanup settles, one is submitted afterward, and
   cancelled/failed kills do not launch. Verify operation-scoped barrier association and
   that unrelated launches remain independent.
6. Add an integration regression with A, B=`A.w0`, and C waiting on B's exact original
   name. Exercise both wait-relaunch and same-run-refresh cases using real
   registry/lookup or wait-index code. C must remain pending while B is
   running/restarting and resolve after B succeeds. Assert C's artifacts and dependency
   string survive cleanup; include an auto-named downstream C.
7. Retain coverage that fresh anonymous single-agent waits still allocate the next free
   `.w<N>` name, explicit names win on a genuinely new launch, and in-place
   edits/run-now do not change existing identities.

## Verification and delivery

Read `lint_and_test.md`, `tui_perf.md`, and `xprompts.md` through `/sase_memory_read`
before implementing in their domains. Use the existing tests near the changed paths,
including `tests/ace/tui/test_agent_wait_resume.py`, the runner refresh/extraction
tests, `tests/test_launch_planned_agent_name.py`, and relevant
force-reuse/relaunch-barrier tests. Add focused integration tests where the existing
tests mock away the failing seam.

The planning checkout's virtualenv could not import its editable Rust extension. The
read-only probes ran the workspace Python source with the installed SASE interpreter and
Rust wheel instead. The implementer should repair their own workspace environment with
the documented `just install`/`just rust-install` workflow as needed and run
verification against the implementation, not assume these probes replace tests.

Run the focused regressions, then `just check`. Use `/sase_monitor` for a long check;
use `just check-full` under that skill if the documented escalation rules require it. If
Rust changes are needed, also run `just check` from its opened repo so both Rust and
PyO3 binding tests run. Do not use core-only cargo tests as the final Rust verification.

Report the two diagnosed triggers, the identity-preserving behavior, and actual
verification results. Follow host-owned finalization and `/sase_final` for the
implementation turn; do not manually create commits, branches, or PRs.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact | Why | Uses |
| --- | --- | --- | ---: |
| cited-by | [agent:bbugyi200.athena.0i7--code][1] | prompt reference @plan:202609/preserve_wait_agent_identity.md | 1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0i7.md

<!-- sase:referenced-by:end -->
