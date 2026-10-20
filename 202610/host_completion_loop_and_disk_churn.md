---
tier: epic
title: Stop unsatisfiable host-completion loops and the disk churn they caused
goal: 'A verified agent can never loop on a host completion that cannot succeed, its
  recovery prompts say why completion refused, monitor cycles no longer pile up cargo
  targets, and a full disk yields a truthful, recoverable monitor outcome instead
  of a silent lost turn.

  '
decisions:
  no_stage_source:
    ask: How should `sase final prepare` treat a `just check` that reports no stage
      evidence?
    choices:
      refuse: Refuse at prepare with the inline-verify fallback; stage-reporting checks
        only
      exit_code: Seal an exit-code-only contract; a zero exit lands the work with
        no stage evidence
    default: refuse
    why: Refusing fails closed and keeps the evidence bar the verify contract enforces.
    answer: refuse
  repeat_guard:
    ask: Should prepare refuse to re-seal an unchanged tree whose passing check was
      already refused?
    choices:
      guard: Add the guard, so an identical refused cycle costs at most one extra
        check run
      skip: Skip the guard and rely on the surfaced recovery reason alone
    default: guard
    why: Only a mechanical guard bounds the loop when a model ignores the reason text.
    answer: guard
phases:
- id: core-completion-seal-guards
  title: 'sase-core: seal-time guards for conditional completion'
  depends_on: []
  size: medium
  description: 'core-completion-seal-guards: in sase-core, refuse to seal a just check
    contract whose stages the project cannot report, and refuse re-sealing an identical
    tree whose passing check was already refused.'
- id: prepare-stage-guard
  title: 'sase: feed the seal guards from sase final prepare'
  depends_on:
  - core-completion-seal-guards
  size: medium
  description: 'prepare-stage-guard: resolve the check''s stage source from the tool
    catalog, pass the predecessor''s refusal, fix creator.project, move the core pin,
    and update docs and skills.'
- id: recovery-reason-prompt
  title: 'sase: show the host-completion recovery reason to the follow-up agent'
  depends_on: []
  size: small
  description: 'recovery-reason-prompt: persist every recovery reason and render a
    Host completion section with glosses in monitor follow-up prompts.'
- id: monitor-scratch-reclaim
  title: 'sase: reclaim the starter''s cargo target when its monitor settles'
  depends_on: []
  size: medium
  description: 'monitor-scratch-reclaim: during monitor settlement, remove the unheld
    starter cargo-targets dir through the Rust-owned reaper, leaving agent-tmp to
    the backstop.'
- id: core-pressure-emergency-age
  title: 'sase-core: honor the low-free-space pressure age for unheld entries'
  depends_on: []
  size: small
  description: 'core-pressure-emergency-age: let observed-unheld pressure candidates
    use the shorter of the dead-launch grace and the emergency min age.'
- id: supervisor-enospc-resilience
  title: 'sase: make the proc supervisor survive a full disk truthfully'
  depends_on: []
  size: medium
  description: 'supervisor-enospc-resilience: keep draining after log write errors,
    keep known exits, retry settlement on ENOSPC, and name the last supervisor error
    on loss.'
proposed_by: bbugyi200.apollo.96
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 06:52:53
status: done
bead_id: sase-1n8
---

- **PROMPT:** [prompts/202610/host_completion_loop_and_disk_churn.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/host_completion_loop_and_disk_churn.md)
- **BEAD:** [sase-1n8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1n8/README.md)

# Stop unsatisfiable host-completion loops and the disk churn they caused

## Incident

`bob-cli-a0.5.1.5` (project `gh_bobs-org__bob-cli`, a Rust repo) ran for about 3.5 hours
as an agent session of 19 LLM turns, each followed by a `verify` monitor running
`just check`. The final monitor (`--mon-i`) ended `lost` ("supervisor exited without
reporting"), which has no continuation, so the session dead-ended as Failed with phase
bead `bob-cli-a0.5.1.5` still open.

The evidence chain:

1. **The completion contract could never pass.** Each turn sealed a conditional
   completion with `sase final prepare` (`verification.command: [just, check]`) and
   bound it with `sase monitor start -p verify -f <ref> -- just check`. sase-core
   hardcodes the sase repo's own stage list for every `just check` contract:
   `crates/sase_core/src/continuation/completion.rs:25`
   `CHECK_STAGES = ["formatting", "ruff", "mypy", "validation", "scoped_tests"]`
   (`CHECK_FULL_STAGES` likewise), applied in `verification_contract()`. bob-cli has no
   `tools:` catalog in its `sase/sase.yml`, so its `just check` is an ad-hoc tool run
   with no stage producer (`diagnostics/diagnostic_manifest.json` has `"stages": []`
   even on green runs). Every green check (exit 0, 5–7 min) therefore evaluated
   `recover` with `missing_required_stage:formatting` (`completion_eval.rs`
   `stage_reasons`). The prepare preview still said `eligible: true`.
2. **The agent was never told why.** `recover_host_completion`
   (`src/sase/monitor/host_completion_complete.py:94`) records the reason in
   `host_completion_receipt.json` and meta, but the follow-up prompt
   (`src/sase/monitor/followup_prompt.py`) only shows "Outcome COMPLETED — exit 0" plus
   the generic `DEFAULT_RECOVERY_ACTION` ("Diagnose failures or stale verification, then
   finish the requested change.", `src/sase/monitor/_host_completion_shared.py:16`). The
   agent re-sealed the identical tree and reran `just check` about 16 times
   (`main_diff.txt` was byte-identical across turns `--2`…`--a`, `--b`…`--f`, and
   `--g`…`--j`).
3. **Every cycle minted a fresh 2.5 GB cargo target that outlived it by hours.**
   `spawn_agent_subprocess` gives every launch a new scratch key and points
   `CARGO_TARGET_DIR` at `~/.cache/sase/tmp/cargo-targets/<key>`
   (`src/sase/agent/launch_spawn.py:152-187`). The monitored command inherits the
   starter's env, so it builds into the starter's dir from scratch. The starter's runner
   skips cleanup on a monitor handoff
   (`src/sase/axe/run_agent_runner_scratch.py:60-66`), nothing in monitor or proc
   settlement reclaims it, and the next turn gets a new empty dir. Only the hourly
   `managed_tmp_reap` dead-launch backstop frees it, two or more hours after its last
   write. The housekeeping log shows the backstop reclaiming 11.9 GB, then 13.3 GB, then
   22.3 GB in three consecutive hourly runs around the failure.
4. **The emergency pressure age did not apply.** At 07:47 UTC the pressure pass ran with
   2.68 GB free, `pressure_trigger=size_and_free_space` and
   `pressure_min_age_seconds=3600`, yet `pressure_selected=0`. `apply_pressure_liveness`
   (`crates/sase_core/src/managed_tmp.rs:1340-1400`) applies only the 2h dead-launch
   grace to observed-unheld entries, so the configured 1h low-free-space age
   (`managed_tmp.pressure.low_free_space_min_age_seconds`) never takes effect when the
   liveness observer is available.
5. **The disk filled and the supervisor crashed instead of reporting.** At 08:36 UTC
   rustc failed with `No space left on device`. The retained log stops at exactly 204800
   bytes. The proc supervisor then died in settlement. Its `supervisor.log` (under
   `~/.sase/procs/runtime/<monitor_id>/`) shows `run_supervisor` → `settle_named_proc` →
   `begin_proc_settlement` →
   `ValueError: failed to serialize proc: No space left on device (os error 28)`.
   Reconcile then settled the proc as `supervisor-loss` with the generic
   `SUPERVISOR_LOSS_MESSAGE` (`src/sase/procs/submission.py:232-267`), which became the
   `lost` outcome with no continuation. Along the way, `BoundedLogPipe._drain`
   (`src/sase/logs/pipe.py:137-159`) stops reading after a log write error, which can
   SIGPIPE the child. `run_supervisor`'s `finally` block
   (`src/sase/procs/supervisor.py:214-218`) also relabels any pipe-close error as
   `supervisor-loss`, even when the command's exit code is already known.

A minor provenance bug showed up too: `prepare_conditional_completion` seals
`creator.project` from `SASE_PROJECT` / `SASE_PROJECT_NAME`, which agents do not have,
so every intent falls back to `"sase"`
(`src/sase/finalizers/_prepare_completion.py:62-66`).

Each phase below fixes one link of this chain, and each fix stands on its own. Phases 1
and 2 remove the loop. Phase 3 gives the agent the reason. Phases 4 and 5 stop the disk
churn. Phase 6 makes a full disk produce a truthful, recoverable outcome instead of a
silent dead end.

## Shared rules for every phase

- Work in the repo the phase names. sase-core changes happen in the linked checkout
  (`sase repo open sase-core -r "<why>"`) and follow its `AGENTS.md`. Agents run
  `sase tool run check` there, never bare `just check`.
- Domain decisions belong in sase-core (`rust_core_backend_boundary`); Python code
  observes, gathers inputs, and renders.
- New wire request fields must be optional with a serde default that reproduces today's
  behavior, so an older sase calling a newer core is unaffected. These are bug fixes and
  wire-compatibility defaults, not user-selectable behavior, so no feature flags.
- Before editing `docs/`, read the `docs_writing` reference memory. Before editing
  generated skill sources under `src/sase/macros/skills/`, read `generated_skills`.
  Before finishing, read `lint_and_test`.
- Never weaken an existing assertion to get a green run. Every behavior change needs a
  regression test that fails on the old code.

## Phase 1 — sase-core: seal-time guards for conditional completion

**Repo:** sase-core

Make an unsatisfiable or already-refused completion contract fail at seal time instead
of at evaluation, in `crates/sase_core/src/continuation/completion.rs` (plus its
`sase_core_py` binding round-trip tests).

1. **Stage source.** Add an optional `stage_source` to the seal request wire
   (`ConditionalCompletionPrepareRequestWire`), as an enum `run_silent | none`. When it
   is `none`, `seal_conditional_completion` returns a typed validation error. The
   message says the bound `just check`/`just check-full` reports no stage evidence (the
   project's tool catalog does not declare `stages: run_silent` for it), so the required
   stages (list them) can never be proven. When it is absent, behave exactly as today
   (legacy callers). Keep `CHECK_STAGES` / `CHECK_FULL_STAGES` and
   `validate_verification` unchanged for `run_silent`, so existing sase intents keep
   sealing and validating the same way.
   > [!decision] no_stage_source = exit_code Instead of refusing, seal
   > `required_stages: []` when `stage_source` is `none`. Teach `validate_verification`
   > to accept an empty list only for an intent sealed with `stage_source: none`, and
   > record `stage_source` in the sealed verification contract so validation can tell a
   > lowered contract from a legitimate one. Add tests that a `run_silent` contract
   > still cannot be lowered to an empty list.
2. **Repeat guard.** Add an optional `prior_refusals` list to the seal request. Each
   entry carries a previously sealed intent's `worktree_fingerprint`, verification
   `command`, `plan_digest`, and the refusal `reason`, as supplied by the host. After
   computing the new seal, refuse with a typed error when an entry matches the new
   seal's worktree fingerprint, command, and plan digest. The message quotes the prior
   reason and says re-running the same verification on the same tree cannot change the
   result. An absent or empty list means no guard.
   > [!decision] repeat_guard = skip Omit `prior_refusals` and its tests entirely.
3. Tests in `completion.rs`'s test module: `none` is refused (or sealed empty under
   `exit_code`), `run_silent` and absent are unchanged, a matching prior refusal is
   refused, and a non-matching one (different fingerprint, command, or plan digest) is
   allowed. Add binding round-trip tests for the new request fields in the domain that
   binds `continuation_seal_conditional_completion`.
4. Run `sase tool run check` in sase-core and land the commit. Phase 2 moves sase's pin
   past it.

## Phase 2 — sase: feed the seal guards from `sase final prepare`

**Repo:** sase

1. In `prepare_conditional_completion` (`src/sase/finalizers/_prepare_completion.py`),
   resolve the wrapper's verification argv against the current project's tool catalog.
   Reuse `sase.tool.argv`'s resolution (see `default_continuation_mode` in
   `src/sase/tool/_executor_shared.py` for how `stages` is read). Pass
   `stage_source: run_silent` only for a named, non-ad-hoc tool whose definition
   declares `stages: run_silent`; otherwise pass `none`. A catalog that cannot be read
   counts as `none` (fail closed).
2. Repeat guard: collect `prior_refusals` from this agent session. Find the monitor turn
   whose result launched the current turn (the current turn's continuation records name
   it). If that monitor exited 0 and its `host_completion_receipt.json` has
   `status: recovery`, load the intent it names (`intent_ref`, resolved against the
   starter's artifacts dir the same way `recover_host_completion` does) and pass its
   sealed `worktree_fingerprint`, verification command, `plan_digest`, and the receipt's
   `reason`. Walking one hop back is enough, because each refused cycle hands its facts
   forward. Any lookup failure means "no prior refusals", never an error.
   > [!decision] repeat_guard = skip Skip this step and its tests; prepare never passes
   > `prior_refusals`.
3. Keep the existing `FinalizerDeclarationError` wrapping, so both new refusals print
   the current `Fallback:` text (run the verification inline, then
   `sase final submit <same-wrapper-file>`).
4. Fix `creator.project`: seal the agent's real project name from the run's own metadata
   (the same source the runner uses for the agent's project) instead of falling back to
   `"sase"`. Keep a fallback only for runs with no project at all.
5. Move `sase-core-revision.txt` past phase 1's commit (`just ratchet-core-revision`;
   see `docs/contribute/rust-backend.md#the-ci-source-revision-pin`).
6. Docs and skills: update `docs/reference/cli/final.md` (`sase final prepare`
   refusals), the conditional-completion section of
   `docs/reference/automation/monitors.md`, and the `sase final prepare` guidance in
   `src/sase/macros/skills/sase_final.md` and `src/sase/macros/skills/sase_monitor.md`.
   They must say that no-model host completion needs a check tool declared with
   `stages: run_silent`, and that an unchanged tree whose passing verification was
   already refused will be refused again. Follow the `generated_skills` note for
   regenerating and deploying skills.
7. Tests (extend `tests/core/test_continuation_facade_conditional_completion.py` and the
   prepare tests next to `src/sase/finalizers/`):
   - a project with no catalog entry gets `none` and is refused with the fallback text;
   - sase's own `check` (with `stages: run_silent`) still prepares;
   - a refused predecessor on the same tree blocks re-preparation;
   - a changed tree still prepares;
   - `creator.project` names the agent's project.

## Phase 3 — sase: show the host-completion recovery reason to the follow-up agent

**Repo:** sase

1. When host completion recovers, persist the evaluator's full `reasons` list next to
   the first `reason` (in `host_completion_receipt.json` and the
   `monitor_host_completion_*` meta written by `record_status`;
   `src/sase/monitor/host_completion_state.py`). The decision dict from
   `_evaluate_intent` already carries `reasons`; thread it through
   `recover_host_completion`'s callers in `src/sase/monitor/host_completion_run.py`.
2. In `render` for monitor follow-up prompts (`src/sase/monitor/followup_prompt.py`),
   add a `## Host completion` section before `## Your next action` whenever the
   monitor's meta records a recovery status. The section states that the verification
   itself finished with its outcome, but host completion did not land the change, then
   lists every reason code. Add a one-line gloss for each reason family:
   `missing_required_stage:*` / `skipped_required_stage:*` mean the check produced no
   matching stage evidence, so re-running the same check cannot fix it; use the inline
   fallback. Also gloss `stale_worktree_fingerprint`, `changed_finalizer_requirements`,
   `new_repository_obligation:*`, `unsupported_executor:*`, `incomplete_observations`,
   `unknown_head`, and the stage contradiction and error codes. Unknown codes render
   bare. Keep the section small and inside the existing disabled-macro region.
3. Tests: a recovered `just check` with exit 0 renders the section with
   `missing_required_stage:formatting` and its gloss; a non-recovery follow-up renders
   no section. Update any golden prompt fixtures this touches.
4. Docs: mention the section where `docs/reference/automation/monitors.md` describes
   host-completion recovery.

## Phase 4 — sase: reclaim the starter's cargo target when its monitor settles

**Repo:** sase

1. After the monitored command is gone, and before the follow-up or host-completion
   launch, remove the starter launch's `cargo-targets/<starter scratch key>` directory
   when no live process holds it. Do this in the monitor-specific settlement path
   (`src/sase/monitor/proc_adapter.py`, which runs inside proc settlement in
   `src/sase/procs/settlement.py`), not in generic proc code.
   - Read the starter's key from its recorded metadata (`read_recorded_scratch_key` on
     the monitor's starter artifacts dir).
   - Go through the existing Rust-owned owner `reap_managed_tmpdir` with a
     `LaunchScratchRequest` whose buckets are only `("cargo-targets",)`, after the same
     `observe_launch_scratch_liveness` check `cleanup_launch_scratch` uses
     (`src/sase/axe/run_agent_runner_scratch.py`). Factor out a shared helper rather
     than copying it.
   - Remove only when the observation is complete and not live; otherwise preserve and
     leave it to the dead-launch backstop.
   - Leave `agent-tmp` alone, because a follow-up turn may legitimately read files the
     previous turn left in `$TMPDIR`.
2. The step is best-effort and idempotent. It never raises into settlement, it is safe
   to re-run on resumed or reconciled settlement, and it does nothing for joined
   monitors or starters without a recorded key. It logs one structured
   `launch_scratch_cleanup` line with a `context: monitor-settlement` field, like the
   runner's.
3. Leave gate handoffs and the runner-exit skip in
   `src/sase/axe/run_agent_runner_scratch.py` unchanged. Note in a code comment that
   monitor settlement, not runner exit, now owns the starter's cargo target.
4. Tests:
   - a settled monitor whose starter target is unheld removes only the cargo target;
   - a held or incomplete observation preserves it;
   - a missing key or dir is a no-op;
   - `agent-tmp` survives;
   - extend `tests/test_run_agent_runner_scratch_cleanup.py` and the monitor proc
     settlement tests (`tests/monitor/test_monitor_proc_settlement.py`).
5. Docs: correct the "a follow-up turn mints a fresh scratch key … the old key goes
   quiet" text in `docs/reference/automation/scheduler-housekeeping.md` and
   `docs/reference/config/runtime-and-storage.md`, and the `managed_tmp_reap` job
   description in `src/sase/default_config.yml`.

Reusing one cargo target across an agent session's turns (so cycles would also rebuild
incrementally) is a separate performance follow-up and out of scope. File it as a task
bead via `/sase_new_task` if it is not already tracked.

## Phase 5 — sase-core: honor the low-free-space pressure age for unheld entries

**Repo:** sase-core

1. In `apply_pressure_liveness` (`crates/sase_core/src/managed_tmp.rs`), an
   observed-unheld (aligned, complete, not live) entry qualifies when it is older than
   the shorter of the dead-launch grace and the effective pressure minimum age. In other
   words, it qualifies when `latest_mtime` is earlier than the later of `grace_cutoff`
   and the pressure `cutoff`. Outside the free-space floor this is unchanged (grace 2h
   beats the 12h default). Below the floor, the configured 1h emergency age now applies.
   Record the cutoff actually used on each candidate, and update the function's doc
   comment.
2. Tests: below the free-space floor, an unheld cargo-target entry quiet for 90 minutes
   is selected. Above the floor it is not. A held entry and an incomplete observation
   are still preserved.
3. Check sase's `tests/test_managed_tmp_reaper_pressure.py` and
   `tests/test_managed_tmp_reaper_dead_launch.py` against the new behavior. If any sase
   expectation changes, move sase's `sase-core-revision.txt` and update those tests in
   the same landing. Otherwise the six-hourly ratchet carries the fix.
4. Coordinate with in-flight epic `sase-zw.8.7.8` (retention landing contracts), which
   is also editing `managed_tmp.rs`. Keep this change confined to
   `apply_pressure_liveness` and its tests.

## Phase 6 — sase: make the proc supervisor survive a full disk truthfully

**Repo:** sase

1. **Keep draining.** In `BoundedLogPipe._drain` / `_ingest_chunk`
   (`src/sase/logs/pipe.py`), after a log write error, keep the error but continue
   reading and discarding output until EOF or the close deadline, so the child never
   hits EPIPE/SIGPIPE because the log disk is full. Count dropped bytes so
   `retention_snapshot()` reports the gap (`complete: false`).
2. **Don't relabel a known exit.** In `run_supervisor`'s `finally` block
   (`src/sase/procs/supervisor.py`), a pipe-close error must not overwrite an outcome
   `_run_command` already produced. Keep its status, exit code, and termination reason,
   and append a note to the message (e.g.
   `exited with code 101; output log incomplete: [Errno 28] No space left on device`).
   Keep today's `supervisor-loss` relabel only when no command outcome exists.
3. **Retry settlement on storage exhaustion.** Wrap the final `settle_named_proc` call
   in a bounded retry for storage-exhaustion failures: `OSError` with `errno` `ENOSPC` /
   `EDQUOT`, or the Rust store binding's `ValueError` whose text contains `os error 28`.
   - Use exponential backoff capped at about a minute, over a total window longer than
     one hourly housekeeping interval (about 70 minutes), so the reaper can free space
     while the supervisor, still alive, keeps reconcile from declaring it lost.
   - Write each retry as one best-effort line to stderr (`supervisor.log`).
   - After the window, re-raise as today.
   - Confirm that a retry after a partial settlement is safe; settlement is checkpointed
     (`src/sase/procs/settlement.py`).
4. **Say why a supervisor was lost.** When `_reconcile_named_proc`
   (`src/sase/procs/submission.py`) settles a `supervisor-loss`, read a bounded tail
   (for example the last 4 KiB) of the proc's runtime `supervisor.log`
   (`proc_runtime_dir(proc_id) / SUPERVISOR_LOG_NAME`), take the last non-empty line,
   and append it to the message as `…; last supervisor error: <line>` (single line,
   length capped). If it shows `os error 28` / `No space left on device`, add a short
   `disk full` hint. Grep for exact-match uses of `SUPERVISOR_LOSS_MESSAGE` and the
   `"supervisor exited without reporting"` text (presentation, done.json, TUI loaders,
   tests) and switch them to prefix matching or reason-based checks where needed. The
   `lost` outcome and its no-continuation policy stay unchanged.
5. Tests:
   - a pipe whose log write raises `ENOSPC` keeps draining and the child exits normally;
   - a supervisor whose close raises after a known exit settles `error` with the real
     exit code, not `supervisor-loss`;
   - settlement that fails twice with `os error 28` and then succeeds settles with the
     original outcome;
   - reconcile appends the last `supervisor.log` line;
   - a missing or unreadable log keeps today's message;
   - start from `tests/logs/test_pipe.py`, `tests/test_procs_service_settlement.py`, and
     `tests/test_procs_runner.py`.
6. Docs: describe the lost-supervisor diagnosis line in
   `docs/reference/automation/monitor-operations.md`.

## Acceptance for the epic

- A `sase final prepare` for a project without a `stages: run_silent` check tool is
  refused at prepare time with the fallback, or, under `no_stage_source = exit_code`,
  sealed with an exit-code-only contract that completes on exit 0. sase's own
  conditional completion behaves exactly as before.
- A recovered host completion's follow-up prompt names every recovery reason.
- After a monitor settles, its starter's cargo target is gone unless something still
  holds it. An agent session cycling LLM turn → `just check` monitor keeps at most one
  launch-sized cargo target on disk at a time.
- Below the free-space floor, unheld cargo targets older than the emergency age are
  reclaimed by the pressure pass.
- A monitored command that hits a full disk ends `failed` with its real exit code once
  settlement succeeds. A truly lost supervisor's `sase monitor show` and done.json say
  why.
