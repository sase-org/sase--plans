---
tier: epic
title: Fix the commit and monitor failures behind the 1gf agent session
goal: 'Commits from linked repositories outside the host checkout finish publication,
  a failed finalizer is never re-run or reported as a provider failure, and monitors
  started in scratch directories hand their follow-up back to the starter''s workspace.

  '
decisions:
  scratch_claim:
    ask: Should a scratch-directory monitor hold the starter's workspace claim until
      its follow-up launches?
    choices:
      hold: Supervisor holds the starter's claim; the follow-up always returns to
        the same workspace
      remember: Only record the starter's workspace; a retaken workspace still degrades
        to pool or 0
    default: hold
    why: Matches in-workspace monitors and guarantees the follow-up sees the starter's
      files
    answer: hold
phases:
- id: release_evidence_project
  title: Key commit release evidence to the publication project
  depends_on: []
  size: medium
  description: 'release_evidence_project: resolve the commit''s host project via the
    repo inventory, record release evidence and enqueue publication under that one
    key, and skip instead of failing when no project maps.'
- id: finalizer_failure_not_provider_failure
  title: Do not re-run finalizers after they fail
  depends_on: []
  size: medium
  description: 'finalizer_failure_not_provider_failure: keep success-path finalizer
    exceptions out of provider-failure finalization, and seed finalizer attempt numbers
    from attempts already on disk.'
- id: monitor_scratch_workspace
  title: Keep the starter's workspace for scratch-directory monitors
  depends_on: []
  size: medium
  description: 'monitor_scratch_workspace: resolve a monitor cwd outside every managed
    checkout to the starter''s workspace so the follow-up returns there; update tests
    and monitor docs.'
proposed_by: bbugyi200.athena.1gj
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 07:38:23
status: wip
bead_id: sase-1vc
---

- **PROMPT:** [prompts/202611/linked_repo_commit_finalization_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/linked_repo_commit_finalization_fixes.md)
- **BEAD:** [sase-1vc](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vc/README.md)

# Fix the commit and monitor failures behind the `1gf` agent session

## Context

The `1gf` agent session (project `gh_bobs-org__bob-cli`) ended with its last turn,
`1gf--2`, marked failed. The model work itself succeeded: it regenerated
`plugins/bob-ledger-tools/main.js` in the linked `bob-plugins` repository, and commit
`4ac198f9` landed. SASE's own completion machinery then failed in three separate ways,
and the staged monitor that should have continued the session was never adopted.
Evidence lives under
`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202611/26/20261126071246/`
(`finalizer_result.json`, `finalizers/progress.jsonl`, `finalizers/commit/*`,
`.sase_provider_failure.json`, `.sase_handoff_adopted.json`).

### Failure 1: release evidence cannot resolve the project for a sibling linked repo (fatal)

`sase stitch create` ran in `bob-plugins` at its primary directory
(`/home/bryan/projects/github/bobs-org/bob-plugins`), next to the `bob-cli` primary
checkout rather than nested inside a numbered workspace. The background publication path
(`_enqueue_agent_publication_background` in
`src/sase/workflows/commit/workflow_publication.py`) calls
`_record_real_change_release_evidence`. That function calls
`resolve_artifact_link_store()` with no `cwd`, so it resolves the project from the
process cwd through a workspace marker or by guessing the project name from the path.
Neither works for a sibling linked checkout, so it raises
`could not resolve the current project for artifact links`. The function only needs
`store.project_key` from that store. The stitch then returns "primary commit succeeded,
but release evidence ... could not be recorded; run `sase stitch create --resume`". The
error is deterministic, so every resume fails the same way. The conflict-repair turn of
`1gf--1` hit the same error twice for the same repository.

Verified with the installed package:

- `resolve_artifact_link_project_key(Path(".../bob-plugins"))` returns `None`.
- `_publication_inventory_match(".../bob-plugins")` in
  `src/sase/agents_sync/_commit_publication_queue.py` returns
  `("gh_bobs-org__bob-cli", "linked")`.
- The same linked repo nested under a numbered workspace
  (`.../bob-cli_11/sase/repos/linked/bob-plugins`) resolves correctly through the
  marker.

So agent publication and release evidence resolve the same commit to different projects.
The publication queue uses the repo inventory and finds the host project. Release
evidence uses only cwd markers and finds nothing. The inline path,
`_drain_artifact_link_outbox_for_agent`, hits the same resolver but silently ignores
exactly this error. That means inline-mode commits from sibling linked repos never drain
their link outbox either. The bug became fatal when background publication
(`e69360f49d`) added the evidence call without that tolerance. Release evidence has to
be keyed to the same project as the publication hint, because the publisher drains links
for `(project_key, run_id, agent)` taken from that hint.

### Failure 2: a failed finalizer is re-run as if the provider had failed

In `src/sase/llm_provider/_invoke.py`, the success branch calls `run_finalizers(...)`
inside the same `try` whose generic `except Exception` handler calls
`finalize_staged_after_provider_error` (`src/sase/llm_provider/_invoke_errors.py`). When
the finalizer phase raised, that handler did three things:

- It wrote `.sase_provider_failure.json`, blaming the `codex` provider, which had
  actually succeeded.
- Because a staged handoff existed, it ran `run_finalizers` a second time, which shows
  up as a second `phase_started` in `finalizers/progress.jsonl`.
- That second phase built a fresh `InstanceLedger` (`src/sase/finalizers/ledger.py`,
  `next_attempt = 1`). Its second resume attempt collided with the first phase's
  `attempt-2.bob-plugins-checkpoint-resume.*` files. That raised
  `immutable finalizer artifact already exists` (`src/sase/finalizers/artifacts.py`) and
  buried the real error under a misleading diagnostic.

The fallback in `finalize_staged_after_provider_error`
(`0f73623591 feat(handoff): finalize staged turns after provider failures`) is meant for
provider failures. It was never meant to retry a finalizer phase that already ran.

### Failure 3: monitors started in scratch directories send the follow-up to workspace 0 (upstream trigger)

`1gf--code` ran in workspace 11. It started a monitor whose command ran in
`/tmp/bob-ledger-tools-cursor-red.WighUC`. `_resolve_monitor_workspace` in
`src/sase/monitor/start_lane.py` returns `(0, cwd)` for any cwd outside every managed
checkout, even when the starter is in a positive workspace. The follow-up launcher
(`launch_turn_followup` in `src/sase/turns/followup_launch.py`) then can't pair
`/tmp/...` with any workspace and falls back to workspace 0 with the "monitor member's
own metadata did not record a claimed workspace number" warning. `1gf--1` therefore ran
in the primary checkout, where the `bob-plugins` checkout had not pulled `1gf--code`'s
commit. It rebuilt the already-landed change from the plan and spent 33 minutes in
conflict repair against its own earlier commit. It then committed from the sibling
`bob-plugins` primary directory, which is exactly the layout that triggers failure 1. In
November this fallback happened eight times across `gh_bobs-org__bob-cli` and
`gh_bbugyi200__tailnet` (`/tmp/...` and `~/.sase/perf/...` scratch directories). The
behavior is documented in `docs/reference/automation/monitors.md` ("A directory outside
every managed checkout uses workspace `0`.") and pinned by
`tests/monitor/test_monitor_start_nested_cwd.py::test_unmanaged_cwd_still_degrades_to_workspace_zero_with_raw_cwd`.

### Governing decisions

- `decisions:handoffs-end-through-host-completion`: adoption is fail-closed, so a failed
  finalizer correctly held the staged monitor. These fixes keep that rule. They stop a
  deterministic bookkeeping error from causing the failure, and they keep a finalizer
  failure from being reported as a provider failure.
- `decisions:host-owned-completion`: commits stay host-owned. Nothing here lets agents
  commit directly.
- `decisions:machine-link-writes-off-primary`: the inline drain must keep using
  `resolve_machine_artifact_link_store` (the hidden clone). The fix changes only which
  project key is passed to it.
- `decisions:rust-core-required`: all three changes are in Python-owned orchestration
  (commit publication, invocation error handling, and monitor lane resolution). No
  `sase-core` wire changes are expected. If a phase finds that the behavior it needs is
  owned by `sase_core`, it must stop and record a `PROPOSED FOLLOW-UP:` note instead of
  reimplementing that behavior in Python.

The three phases are independent and can run in parallel.

## Phase `release_evidence_project` — key commit release evidence to the publication project

- size: medium
- depends on: none

Goal: committing from a linked repository whose checkout sits outside the host workspace
(such as `bob-plugins` at its primary directory) records artifact-link release evidence
and enqueues agent publication under the host project. A project that can't be resolved
must not fail an already-landed commit.

Work:

1. Add one resolver for the project key of a commit's publication, for example
   `resolve_commit_publication_project_key(cp)` beside the publication helpers. Try
   these candidates in order and return the first canonical project key:
   1. the repo-inventory match for `cp.cwd`, which is the source
      `_resolve_sidecar_publication_target` already uses (`_publication_inventory_match`
      / `_publication_project_key_from_path`; expose a public wrapper from
      `sase.agents_sync.commit_publication` instead of importing a private name across
      packages);
   2. `resolve_artifact_link_project_key(Path(cp.cwd))` (the workspace marker);
   3. the project named by the checkpoint's `project_file`
      (`~/.sase/projects/<key>/<key>.sase`), using an existing project-file helper such
      as the one in `src/sase/main/patch_common.py` instead of a new path parser.
2. Change `_record_real_change_release_evidence` to take that project key and stop
   calling `resolve_artifact_link_store()`. It only ever used `store.project_key`, and
   it should no longer depend on the process cwd. If no key resolves, print a warning
   that names the repository and skip recording evidence. Agent publication for an
   unmappable repository is already skipped with a warning, so this skip is consistent.
   Any other evidence-store exception must keep its current resumable error.
3. In `_enqueue_agent_publication_background`, resolve the key once, record evidence
   under it, and pass it as `project=` to `enqueue_committed_agent_publication`. The
   durable row and `append_publication_hint` must then use the same key as the evidence.
   Add a regression assertion that the two keys are the same.
4. In `_drain_artifact_link_outbox_for_agent` (the inline path), use the same resolver
   instead of `resolve_artifact_link_store(cwd=...)`, and still pass the result to
   `resolve_machine_artifact_link_store`. A sibling linked repo should now drain inline
   instead of silently skipping. Keep the existing quiet skip only for the case where
   nothing resolves.
5. Leave `resolve_artifact_link_store()` and the general
   `resolve_artifact_link_project_key` contract unchanged for other callers.

Tests, in the existing commit-publication test modules under `tests/`:

- Background mode, with a linked repo registered in the repo inventory at a path outside
  the host checkout and no workspace marker: the publication step succeeds, release
  evidence is recorded under the host project key, and the enqueued item and hint use
  that same key.
- Inline mode, same layout: the artifact-link outbox drain runs against the machine
  store for the host project.
- A repository that maps to no project: evidence is skipped with a warning, and the step
  does not return a resumable failure for that reason alone.
- An evidence-store write error still returns `False` with the `--resume` guidance.
- Resuming a checkpoint (`sase stitch create --resume`) that previously failed this way
  finishes publication without creating another primary commit.

## Phase `finalizer_failure_not_provider_failure` — don't re-run finalizers after they fail

- size: medium
- depends on: none

Goal: when the provider succeeds but the finalizer phase raises, the turn fails with the
finalizer's own error. SASE does not record a provider failure and does not start a
second finalizer phase. Separately, attempt artifacts never collide when an artifacts
directory legitimately sees more than one controller phase.

Work:

1. In `src/sase/llm_provider/_invoke.py`, mark when the success-branch finalizer call
   has started (for example with a local flag set before `run_finalizers`, or by
   wrapping its exceptions in a dedicated type). Skip
   `finalize_staged_after_provider_error` in every `except` branch when that mark is
   set. Still run the normal error metrics and `postprocess_error`, and still raise
   `LLMInvocationError` with the finalizer's message. Do not write
   `.sase_provider_failure.json` in this case. Check whether any consumer of that file,
   such as the staged completion controller or failure-facts capture, needs a distinct
   finalizer-failure marker. Add one only if a consumer requires it.
2. Make attempt numbering in `src/sase/finalizers/ledger.py` start above the attempts
   already on disk. When a ledger is created for an instance that already has
   `finalizers/<instance_id>/attempt-<n>.*` files, or attempt events in
   `finalizers/progress.jsonl`, start `next_attempt` at the next unused number. Do not
   change the per-phase `max_attempts` budget semantics. Keep the exclusive-write guard
   in `src/sase/finalizers/artifacts.py`; it is the right protection, it was just
   triggered by a numbering bug.
3. Check the other `run_finalizers` callers (`src/sase/macro/workflow_completion.py`,
   `src/sase/monitor/host_completion_execute.py`) for the same "rerun after failure"
   pattern. Fix any that share the success-then-generic-except shape. Otherwise note in
   the phase result that they are unaffected.

Tests:

- The provider returns success, the finalizer raises, and a staged handoff exists:
  `run_finalizers` runs exactly once, no provider-failure file is written, the raised
  error carries the finalizer message, and the handoff is not adopted.
- A genuine provider failure with a staged handoff still finalizes once through
  `finalize_staged_after_provider_error`, so existing behavior is preserved.
- Two controller phases in one artifacts directory that each execute a
  `*-checkpoint-resume` operation produce distinct attempt numbers, with no
  `immutable_attempt_artifact` diagnostic.

## Phase `monitor_scratch_workspace` — keep the starter's workspace for scratch-directory monitors

- size: medium
- depends on: none

Goal: when an agent in workspace N starts a monitor whose cwd is outside every managed
checkout (such as `/tmp/...` or `~/.sase/perf/...`), the follow-up returns to workspace
N instead of degrading to workspace 0. Behavior is unchanged for a cwd inside the
caller's workspace, a cwd inside a different managed workspace, and starters in
workspace 0.

> [!decision] scratch_claim = hold In `_resolve_monitor_workspace` /
> `resolve_lane_start` (`src/sase/monitor/start_lane.py`), when no workspace owns the
> cwd and the lane has a positive `workspace_num`, return the lane's own
> `(workspace_num, workspace_dir)`. With `inherit_lane_workspace_claim` and a live
> runner pid, set `transfer_from_pid` exactly as the cwd-in-lane case does, so the
> supervisor holds workspace N. `monitor_cwd` keeps the scratch directory, and
> `supervise_run.py` already executes there. Audit every reader that treats
> `workspace_dir` / `workspace_num` as the command's checkout, especially
> `observe_checkout_dir`, the starter-match check, and
> `_successor_inherited_obligations` in `src/sase/monitor/settlement_fresh_owner.py`.
> Settlement should then observe workspace N, as it does for in-workspace monitors, and
> the scratch directory must not be treated as an owned checkout.

> [!decision] scratch_claim = remember Leave the monitor member's `workspace_num` /
> `workspace_dir` as `(0, cwd)` and do not transfer a claim. Record the starter's
> positive pair in new member fields, such as `followup_workspace_num` /
> `followup_workspace_dir`, and have the monitor follow-up
> (`src/sase/monitor/followup.py` → `launch_turn_followup`) use them as the original
> workspace. A busy workspace N then degrades through the existing fresh-claim → pool →
> workspace-0 chain, with its existing degraded-workspace notes.

Work common to both choices:

1. Keep the "cwd owned by some managed checkout" branch first. A starter in workspace 0
   (`lane_workspace_num == 0`) should resolve to `(0, <primary checkout>)` instead of
   `(0, <raw cwd>)`, so its follow-up launches in workspace 0 without the false
   "metadata did not record a claimed workspace number" warning. This covers
   `1gf--mon-0`, whose cwd was the sibling `bob-plugins` primary directory.
2. Update
   `tests/monitor/test_monitor_start_nested_cwd.py::test_unmanaged_cwd_still_degrades_to_workspace_zero_with_raw_cwd`
   to the new contract. Add tests for:
   - a scratch cwd with a workspace-N starter, where the follow-up launches in N (and,
     with `hold`, the claim is held by the supervisor and returned at follow-up);
   - a scratch cwd with a workspace-0 starter, where there is no degraded warning;
   - a scratch cwd when workspace N was retaken (`remember` only), which degrades as
     today;
   - an unchanged cwd in a different managed workspace (fresh claim on that workspace).
     Cover the follow-up side in `tests/monitor/test_monitor_followup_workspace.py`.
3. Update `docs/reference/automation/monitors.md` (the `--cwd` claim rules near "A
   directory outside every managed checkout uses workspace `0`", and the follow-up
   fallback paragraph) to describe the new rule. Follow the `docs_writing.md` reference
   memory.

## Out of scope

- Whether a post-commit publication failure should fail the whole finalizer at all.
  Under `decisions:handoffs-end-through-host-completion` it stays fail-closed. This epic
  only removes the deterministic cause.
- Recovering the stopped `1gf` session (the pending `bob-plugins` commit checkpoint and
  the never-adopted `npm test` / `npm run validate` monitor). That is a manual operator
  step once `release_evidence_project` lands.
- `bob-plugins` test fixtures that look for `../../tests` relative to the primary
  checkout layout. That issue belongs to the bob projects, not SASE.

## Verification

Each phase runs `just check` per the `lint_and_test.md` reference memory. A phase that
changes CLI behavior or docs also follows `cli_rules.md` / `docs_writing.md`.
