---
tier: tale
title: Keep a session's repository responsibility across monitor and gate continuations
goal:
  Monitor, gate, and recovery continuations commit the session's linked-repo work, and
  failed-close recovery finds the real project.
size: medium
decisions:
  complete_inherited_baseline:
    ask:
      Should inherited continuation baselines be completed at boot, attributing missing
      checkouts to the session?
    choices:
      complete:
        Missing checkouts get empty-fingerprint records; a late open can never revoke
        session work
      inherit_only:
        Only fix the source; a never-opened, tool-dirtied linked repo can still be
        revoked
    default: complete
    why: Applies the existing inherited=True contract to every checkout, not just main
    answer: complete
proposed_by: bbugyi200.athena.18v
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-05 16:11:13
status: wip
---

# Keep a session's repository responsibility across monitor and gate continuations

## Incident: agent `18o--4` (bob-cli project, workspace #22)

Session `18o` ran
`18o--plan → 18o--code → (monitor) → 18o--1 → (monitor) → 18o--2 → (monitor) → 18o--3 → (monitor) → 18o--4`.
The work touched bob-cli plus three fixture files in the configured linked repo
`bob-plugins`
(`plugins/*/scripts/fixtures/{pomodoro_clock,event_task_close}/vectors.json`).

1. `18o--code` ran `sase repo open bob-plugins` at 18:44Z. Its `finalizer_baseline.json`
   got a clean `linked:bob-plugins` `opened_repo` record (no fingerprints).
2. Each monitor follow-up (`18o--1` through `18o--4`) inherited the baseline of the
   **session root** `18o--plan` instead of the monitor's starter. All four have
   `parent_timestamp = 20261105131252`, which is the plan's run, and their
   `finalizer_baseline.json` is byte-for-byte the plan's 18:12Z run_start set (main plus
   SDD sidecars). The `bob-plugins` record and its opened-repo markers were dropped.
3. `18o--4` declared commits for main (24 paths) and `sibling:bob-plugins` (3 paths).
   The host accepted the declaration at 20:06:35Z. Configured linked repos are always
   obligation candidates (`dirty_configured_sibling_repos`), so no open was needed to
   include `bob-plugins`.
4. The main stitch hit a rebase conflict. The host started the conflict-repair model
   turn, which shares the run's `SASE_ARTIFACTS_DIR`. After resolving the conflict, it
   followed its prompt to finish through `/sase_final` and ran
   `sase repo open bob-plugins -r "inspect dirty fixture files for final commit declaration"`
   at 20:18:14Z.
5. `capture_opened_repo_dirty_baseline` found no record for that path, so it
   fingerprinted the three dirty fixture files as pre-existing dirt. That set them as
   `protected`. `sase final context` then reported no obligations.
6. The remaining-obligation dispatch read the baseline live, found every changed path
   protected, and failed with `protected_paths_exhausted`. The bob-cli commit (16c6356d)
   landed, but the matching bob-plugins fixture changes were stranded.
7. Failed-close recovery also failed:
   `operational workspace lease failed during allocation: project file does not exist: ~/.sase/projects/bob-cli/bob-cli.sase`.
   All three `failed_close_recovery_outcome.json` files on this machine show the same
   error: two for sase (`.../projects/sase/sase.sase`) and this one for bob-cli.

Evidence is in the `18o--4` run artifacts:
`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202611/05/20261105150345/`
(`error_report.md`, `finalizer_baseline.json`, `opened_siblings.json`,
`finalizers/progress.jsonl`, `finalizers/commit/conflict_repair_response.main.md`,
`failed_close_recovery_outcome.json`). Compare with `18o--code`'s baseline at
`.../20261105132442/finalizer_baseline.json`.

## Root causes

1. **Wrong inheritance source.** `monitor/followup.py` and `gate_turn/followup.py`,
   through `turns/followup_records.spawn_turn_agent_session_successor`, resolve the
   successor's attach plan with
   `AgentSessionAttachDirective(parent=<lane>, suffix="@")`. That binds
   `plan.parent_artifacts_dir` to the session root. Bootstrap
   (`axe/run_agent_runner_bootstrap.py`,
   `inherit_commit_finalizer_responsibility(parent_artifacts_dir=plan.parent_artifacts_dir, ...)`)
   then copies the root's baseline and opened-repo markers rather than the starter's.
   Anything the starter or an intermediate member opened is lost. For external repos,
   whose only obligation path is the opened marker, the successor may silently not
   commit that work. Staged handoffs (`run_agent_successor_staged_identity.py`,
   `run_agent_exec_pipe_staged.py`) attach with `parent=ctx.agent_name` and are already
   correct.
2. **Inherited baselines are never completed.** sase-ti.2 made fresh runs baseline every
   checkout that exists before the first turn. That makes a later `sase repo open` of
   the same path a no-op through the path-dedupe in
   `capture_opened_repo_dirty_baseline`. Inherited runs skip that capture.
   `bind_run_start_baseline_to_workspace` returns early when the inherited `main` record
   already matches the workspace, so a checkout materialized after the session's first
   run has no record. The first open then fingerprints session work as foreign and
   revokes an obligation the host already assigned to the run.
3. **Recovery uses the display name as the project identity.**
   `finalizers/_failed_close_recovery_apply._resolve_machine_project` returns the
   checkout marker's `project_name`. That value is the basename of the primary checkout
   (`bob-cli`, `sase`), not the SASE project key (`gh_bobs-org__bob-cli`,
   `gh_sase-org__sase`). This violates the `project-labels-never-keys` decision.
   `finalizers/_prepare_completion._resolve_creator_project` already derives the real
   key from the artifacts path, with project-file and env fallbacks.

## Changes

### 1. Continuations inherit from their actual predecessor

- Add two optional fields to `AgentSessionAttachLaunchPlan`
  (`agent/_agent_session_attach_types.py`):
  `predecessor_artifacts_dir: str | None = None` and
  `predecessor_name: str | None = None`. Serialize them in `agent_session_attach_env`.
  Parse them in `load_agent_session_attach_plan_from_env` as optional keys, so payloads
  from older parents still load. Do **not** change `parent_artifacts_dir` semantics:
  session naming, promotion, continuation-budget persistence, and plan-approval
  placement depend on it meaning the attach parent.
- Add one helper, e.g. `commit_inheritance_source(plan) -> tuple[str, str]`. It returns
  `(predecessor_artifacts_dir or parent_artifacts_dir, predecessor_name or parent_name)`.
  Use it at both bootstrap sites in `run_agent_runner_bootstrap.py`: baseline
  inheritance (`inherit_commit_finalizer_responsibility`) and the strict recovery
  bootstrap observation (`collect_bootstrap_observation(parent_artifacts_dir=...)`).
  Inheritance and the recovery gate must never read different predecessors.
- `agent/detached_child.spawn_agent_session_successor`: accept optional
  `predecessor_artifacts_dir` and `predecessor_name` kwargs and apply them in its
  `replace(plan, ...)`.
- Monitor follow-ups (`monitor/followup.py`): resolve the starter's artifacts dir from
  the monitor's `parent_timestamp` with the existing starter lookup
  (`turns/followup_starter._starter_artifacts_dir` / `starter_identity`). The starter is
  the agent that ran `sase monitor start`, never the monitor member: the monitor
  member's own baseline is a fresh capture with fingerprints. Set it, plus the starter
  name, on `frozen_plan` and pass it into `spawn_agent_session_successor`.
- Gate follow-ups (`gate_turn/followup.py` →
  `turns/followup_records.spawn_turn_agent_session_successor`): thread the same
  starter-derived predecessor through the new kwargs.
- Hijack recovery (`agent/hijack/recover.py`, `recovery_admission.py`,
  `recovery_successor.prepare_recovery_successor`): set the predecessor to the hijacked
  agent's artifacts dir. The attach resolve uses `parent=session_base`, which has the
  same root-binding problem. If recovery already pins the hijacked agent's dir some
  other way, wire that value into the new field instead of adding a second source.
- If no starter can be resolved (`parent_timestamp` missing or its artifacts gone), fall
  back to today's behaviour and record an inheritance diagnostic through the existing
  `opened_repo_inheritance_diagnostics.json`, naming the missing predecessor.

### 2. Complete inherited baselines at boot

> [!decision] complete_inherited_baseline = complete

In `llm_provider/commit_finalizer_baseline.bind_run_start_baseline_to_workspace`, when
`inherited=True` and the `main` record already matches, do not return early. Under the
baseline lock, add a `run_start` record with **empty** fingerprints for every
`collect_baseline_repositories(workspace)` checkout whose normalized path no existing
record (any scope) names. Keep existing records unchanged: first record wins and scope
precedence is preserved. When `inherited=False`, behaviour is unchanged. After this
change, the late open in step 5 of the incident is a path-dedupe no-op even if
inheritance were wrong. The existing call sites already pass `inherited=True` for
continuations, so they need no change:
`run_agent_runner_launch._bind_commit_finalizer_baseline_to_workspace` passes
`bootstrap.inherited_commit_finalizer_baseline` for detached boots, and
`run_agent_helpers_artifacts._seed_followup_commit_finalizer_baseline` handles
in-process follow-ups.

> [!decision] complete_inherited_baseline = inherit_only

Skip this section. Add a code comment at the early return in
`bind_run_start_baseline_to_workspace` that inherited baselines rely entirely on
predecessor inheritance for linked-checkout coverage.

### 3. Failed-close recovery resolves the real project key

Replace `_resolve_machine_project`'s marker lookup with the same resolution
`_resolve_creator_project` uses: the artifacts path relative to `sase_projects_dir()`,
then the `SASE_AGENT_PROJECT_FILE` parent, then a canonicalized marker fallback
(`project_aliases.resolve_project_alias_ref`, then check that the project spec exists,
as `bead/project_name._project_name_from_marker` does). Extract a shared helper so the
two finalizer paths cannot drift; do not copy the logic. Keep returning `None`, which
leads to the existing "recovery store unavailable" outcome, when nothing resolves.

## Tests

- `tests/llm_provider/test_commit_finalizer_inheritance.py` and the bootstrap tests:
  build the incident's chain. Use a root run with a run_start-only baseline, a starter
  with a clean `linked:<repo>` opened_repo record plus opened markers, and a
  monitor-style continuation whose attach plan has the root as `parent_artifacts_dir`
  and the starter as `predecessor_artifacts_dir`. Assert that the continuation's
  baseline and opened markers come from the starter, and that a later
  `record_opened_repo` of that repo (simulating the conflict-repair open) leaves its
  fingerprints empty.
- Attach-plan env round trip with and without the new fields (old payload still loads).
- `tests/monitor/test_monitor_followup.py` and
  `tests/gate_turn/test_followup_launch.py`: the spawned attach env carries the
  starter's artifacts dir and name as the predecessor, not the monitor or gate member's
  dir and not the session root's.
- Recovery bootstrap: inheritance and `collect_bootstrap_observation` receive the same
  predecessor dir.
- `bind_run_start_baseline_to_workspace(inherited=True)` (under
  `complete_inherited_baseline = complete`): a checkout missing from the inherited
  baseline gets an empty-fingerprint run_start record. An existing opened_repo or
  run_start record for the same path is left unchanged. A later
  `capture_opened_repo_dirty_baseline` for that path is a no-op.
- Dispatch-level regression. Declare a commit for a dirty linked checkout absent from an
  inherited baseline, open it after acceptance, then dispatch. Assert that it stitches
  instead of raising `protected_paths_exhausted`.
- Failed-close recovery: an artifacts dir under `<projects>/gh_org__name/artifacts/...`
  with a checkout marker whose `project_name` is `name` resolves to `gh_org__name`. Add
  a case where nothing resolves.

## Docs and memory

Update `docs/contribute/commit-workflow-orchestrator.md`, which documents
`protected_paths_exhausted`, and `docs/commit_workflows.md` wherever it describes
baselines. Add three facts: continuations inherit from their predecessor, inherited
baselines are completed at boot, and a late `sase repo open` cannot revoke an obligation
for session work. No memory note states the old behaviour. The implementer should still
grep `sase/memory/` for "inherit"/"baseline" and route any stale statement through
`/sase_memory_write`.

## Out of scope

- Committing the stranded bob-plugins fixture changes from workspace #22. That is a
  one-off operational recovery, handled outside this plan.
- Changing the conflict-repair prompt. Once these fixes land, the repair turn's
  `sase repo open` is harmless, and that prompt is the designed remaining-obligation
  handoff.

## Acceptance

- Replaying the incident shape (root → starter opens a linked repo → monitor
  continuation → conflict repair re-opens it) commits the linked repo's session changes.
  Recovery outcomes on GitHub-keyed projects no longer fail with
  `project file does not exist`.
- The repo's standard agent verification passes (see the `lint_and_test` memory).
