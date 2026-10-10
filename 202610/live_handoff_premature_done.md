---
tier: tale
title: Keep live in-process handoffs from reading as finished
goal:
  A %wait never releases while its dependency's runner is still alive without done.json,
  and a session node never shows DONE mid-handoff.
size: medium
decisions:
  handoff_label:
    ask:
      Should live plan handoffs show the gate's upcoming label (TALE APPROVED, ...)
      instead of RUNNING?
    choices:
      running: "No: every live handoff shows RUNNING; uniform, no plan-file reads"
      plan_aware:
        "Yes: plan handoffs show the upcoming gate label; question/pipe handoffs show
        RUNNING"
    default: running
    why:
      RUNNING is accurate for every handoff kind; the gate shows its own label seconds
      later
    answer: running
proposed_by: bbugyi200.apollo.6h
decided_by: auto
create_time: 2026-10-10 15:18:59
status: wip
---

# Plan: Stop live in-process handoffs from reading as finished (early `%wait` release and `DONE` flicker)

## Verdict on the reported suspicion

The suspicion is **confirmed, with one nuance**. Both symptoms have the same root cause.
They appear in two different sub-windows of the same `%auto` plan handoff, and two
different consumers misread them.

| Window (2026-10-10, UTC)  | Session `6g.w1.w0.f0` state                                                                                                                                                                                                                      | ACE node                                                     | Wait resolver                                                                                                   |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| W1: 18:31:38 to ~18:31:46 | `sase plan propose` SIGTERMed the `--plan` provider. The runner already flipped the planner's `workflow_state.json` / `prompt_step_*.json` to `completed`. No gate member exists yet.                                                            | **`DONE`** (TUI load lag of 3-17 s kept it on screen longer) | unresolved, because no member is `is_done`                                                                      |
| W2: 18:32:01 to 18:32:15  | `--gate` auto-settled inside the still-live creator runner (pid 254775). Its `done.json` says `outcome=gated`, `gate_state=answered`, and its meta says `gate_followup_outcome=suppressed` with no follow-up agent. `--code` does not exist yet. | `TALE APPROVED` (Running bucket)                             | **resolved**, so `6g.w1.w0.f0.w0` was released (`wait_release_source=ready_json`, `wait_completed_at` 18:32:05) |
| 18:32:15 onward           | `continue_as_successor()` created `6g.w1.w0.f0--code` in the same runner                                                                                                                                                                         | `WORKING TALE`                                               | unresolved                                                                                                      |

The waiter's `wait_dependencies_satisfied_at` (1791657121.157304) is exactly the gate
`done.json` `finished_at`.

Replaying `WaitDependencyIndex.is_resolved("6g.w1.w0.f0")` on copies of the W2-era
artifacts (`20261010142546` planner + `20261010143145` gate) returns `True`. Adding the
`20261010143215` `--code` dir makes it return `False`.

A later reconciler rewrote the gate's meta at 19:01:55 to
`gate_followup_outcome=launched` and `gate_followup_agent=6g.w1.w0.f0--code`. That was
long after the release. Any replay must reset those two fields to reproduce W2.

### The same bug is broader than this incident

I scanned this host's bob-cli artifacts from the last two days, comparing each waiter's
real release time (`wait_completed_at`) with when its dependency session actually
finished. That found **8 premature releases**:

- **Shape A (W2 above), 2 cases:** `6g.w1.w0--plan` on `6g.w1`, and
  `6g.w1.w0.f0.w0--plan` on `6g.w1.w0.f0`. Each was released 17-18 minutes early, before
  `--code` existed.
- **Shape C, 6 cases:** a coder's `workflow_state.json` turns `completed` about 40-50 s
  before its runner writes `done.json`, because host-owned finalizers such as `commit`
  run in between. Waiters were released in that gap: `6c--plan`, `6a.f0.f0--plan`,
  `6c.w1--plan`, and `6i`, confirmed by marker timing. `61.w0` and `6g.w1--plan` were
  released 43 s and 50 s early, matching the same signature. So `%w` waiters can start
  before the dependency's commit lands.

## Root cause

`artifact_is_resolved()` is at
`src/sase/core/wait_dependency_resolution/_artifact_state.py:234`. Its `outcome is None`
branch treats any plan-chain member **without `done.json`** as resolved once
`_completed_handoff_workflow_state()` (`:288`) sees completed workflow and prompt-step
markers. That rule came from aaf3ec8a7d. It was meant for superseded feedback and
planner handoffs that never get a `done.json`.

The rule assumes "workflow markers say completed" means "member is finished". The runner
writes those markers well before the member is finished:

1. **Every runner handoff finalizes first and creates the next member later.**
   `handle_plan_marker` (`src/sase/axe/run_agent_exec_plan.py:95`),
   `handle_questions_marker` and `handle_pipe_marker` each call two helpers from
   `src/sase/axe/run_agent_helpers_handoff.py`, `normalize_handoff_interruption_state`
   and `finalize_handoff_artifacts_as_completed`. They do this before the gate member or
   in-process successor exists.
   - Under `%auto`, `_resolve_auto_gate`
     (`src/sase/notification_gates/service_creation.py`) settles the gate with
     `settle_gate_turn(creator_live=True)`.
   - `suppress_live_creator_followup` (`src/sase/gate_turn/handoff_launch.py:224`) then
     records `suppressed` and names no successor.
   - The runner keeps going in-process: `_continue_after_plan_result` →
     `handle_accepted_plan` → `continue_as_successor`. Here that took 14 s.
2. **Every coder's workflow completes before its `done.json`.** The finalizers run in
   between.

During those windows the member counts as "resolved" even though its runner is alive.
The session aggregates then release as soon as any other member is `is_done`:

- The aggregates are `_agent_session_entity` / `_workflow_entity`
  (`_index_entities.py:96`, `:149`) and `agent_session_candidate_for_root`
  (`_index_queries.py:281`). Each computes `all(is_resolved)` and `any(is_done)`.
- In W2 the `is_done` member is the auto-settled gate. `effective_done_outcome` maps
  `gated` + `answered` to `completed`.
- Nothing holds the session open. `turn_followup_handoff_agent` returns `None` for
  `suppressed`, so no follow-up is declared.

**The TUI makes the same inference in W1:**

1. The planner root's base row comes from `workflow_state.json`, where `completed` maps
   to `DONE` (`src/sase/ace/tui/models/_loaders/_workflow_snapshot_loaders.py:96`).
2. `dedup_running_vs_workflow` (`src/sase/ace/tui/models/_dedup.py:356`) folds the live
   RUNNING-claim row into it and keeps `DONE`. Only `runner_is_live` carries over
   (`:63`).
3. Root mirroring in `apply_status_overrides`
   (`src/sase/ace/tui/models/_agent_status_apply.py:145`) falls back to the newest child
   (`:411`). That child is the `DONE` main workflow step.

`pending_review_window_active` (`_loaders/_meta_enrichment_status.py:116`) was meant to
cover this window but cannot:

- It is gated on `meta.plan`, which is only set for `%auto:tale|epic` launches. Since
  73f593a3a5 it comes from the autonomy record's legacy projection. Manual and bare
  `%auto` launches never set it.
- It returns `False` whenever auto-approval covers the plan.

**The invariant this plan restores:** a member whose runner is still alive is not
finished until it writes `done.json`. Completed workflow markers without `done.json`
count as resolved only after the runner that wrote them has exited. That covers
superseded handoffs and crashes, which keep today's fail-open behavior.

## Rejected alternatives

- **Move `finalize_handoff_artifacts_as_completed` after successor creation.** The
  SIGTERM-induced `failed` markers would flash `FAILED`. Also, 4f635a90e4 added the
  early finalize specifically to stop permanently stuck `RUNNING` planner rows.
- **Make the creator-live gate declare the in-process successor** (`suppressed` →
  `launched` + agent). This fixes W2 only. It misses W1 when the session already has a
  done member, and it misses shape C. It would also need suffix pre-allocation inside
  gate settlement.
- **New wire fields or `sase-core` changes.** Not needed. `pid` and `process_identity`
  are already on `AgentMetaWire`, and session wait resolution and TUI status derivation
  both live in this repo.

## Implementation

### 1. Wait resolver: markerless completion resolves only after the runner exits

In `src/sase/core/wait_dependency_resolution/_artifact_state.py`:

- In `artifact_is_resolved`, once `_completed_handoff_workflow_state(artifact_dir)`
  passes, return `False` when the member's runner is still live.
- Add a private helper `_member_runner_is_live(artifact_dir, meta) -> bool`. It returns
  `True` only when all of these hold:
  - `meta["pid"]` is an `int`.
  - `meta["process_identity"]` is a verifiable `"<boot_id>:<start_ticks>"` token.
  - `sase.agent.names.is_process_alive(meta, artifact_dir)` is true. That helper already
    honors `stopped_at`, thread ids, and PID reuse via identity and boot time.

  Any exception returns `False`. Artifacts with no recorded identity (legacy) keep
  today's behavior, so old intermediate handoffs and crashed runners still resolve.

- Keep the import function-local, as `src/sase/axe/wait_marker_scan.py` does.
  `sase.agent.names` lazily imports `sase.ace.hooks.processes`.
- Rewrite the docstrings of `artifact_is_resolved` and
  `_completed_handoff_workflow_state` to state the invariant above.

Both index-build paths carry `pid` and `process_identity`, so the one change covers
both:

- on-disk dicts: `WaitDependencyIndex.build` / `add_many`
- `asdict(AgentMetaWire)` snapshot rows: `add_scan_record`

Every consumer reads this index, so the change also covers:

- the `wait_checks` chop and its confirmation pass
- the runner fallback and the startup fast path (`resolve_initial_wait_release`)
- `sase agent wait`, hold liveness, and the TUI wait display
- `_member_is_pending` / terminal-blocker detection

Do **not** change:

- the planner-row named candidate (`submitted_plan_artifact`, `%wait` on `<base>--plan`)
- gate settlement
- the runner's handoff ordering

Reference shape (an out-of-tree prototype of exactly this was verified):

```python
def _member_runner_is_live(artifact_dir: Path, meta: Mapping[str, Any]) -> bool:
    """Return whether the runner that wrote *meta* is provably still alive."""
    try:
        if not isinstance(meta.get("pid"), int):
            return False
        if not _verifiable_process_identity(meta.get("process_identity")):
            return False
        from sase.agent.names import is_process_alive

        return is_process_alive(dict(meta), artifact_dir)
    except Exception:  # liveness uncertainty keeps legacy resolution
        return False
```

Prototype results on copies of the real artifacts:

| Scenario                  | Prototype result     |
| ------------------------- | -------------------- |
| W2, runner live           | unresolved           |
| W2, runner dead           | resolved (fail open) |
| Shape C, runner live      | unresolved           |
| Shape C, runner dead      | resolved             |
| Coder `done.json` written | resolved             |

All 37 existing wait-resolution test modules (299 tests) passed with the prototype
patched in.

On cost, the probe runs only for markerless plan-chain members whose workflow already
reads `completed` and that carry an identity token. It costs one `/proc` read and one
`agent_meta.json` read. That is comparable to the workflow and prompt-step reads
`_completed_handoff_workflow_state` already does.

### 2. TUI: a live handoff never mirrors `DONE` onto its session node

In `apply_status_overrides` (`src/sase/ace/tui/models/_agent_status_apply.py`), after
the per-root loop picks a status, override it when all of these hold:

- **(a)** The picked status lands in the `Done` bucket. That covers the newest-child
  fallback at `:411` and the `if not children: continue` early exit at `:325` for
  session roots.
- **(b)** The session **frontier** shows a markerless `DONE`, meaning the `DONE` came
  from workflow markers and not `done.json`.
  - The frontier is the newest non-turn agent row among the root and its agent-session
    member children.
  - Gate and monitor turn rows don't count.
  - The root's `main` workflow-step child represents the root's own run.
- **(c)** The frontier row's runner is live (`runner_is_live`). For the root, the
  RUNNING-claim merge in `_dedup.py` and `mark_live_artifact_delta_runners` in
  `_agent_loader_normalization.py` already set it.
- **(d)** The frontier row is not in a finalizer phase. `row_status_is_finalizing`
  (`src/sase/ace/tui/models/finalizer_row_state.py:151`) must keep owning the coder's
  finalizing window.

When all four hold, present the node as in flight per decision `handoff_label`. With
`running` (the default), that is `RUNNING` in the `Running` bucket.

Constraints:

- Add no filesystem I/O in this pass. Loader normalization must stay cheap (tui_perf
  rules). Use only fields already loaded on `Agent`.
- Determine how a `done.json`-backed row is distinguished on `Agent` (agent type or
  loaded done-marker evidence). Rows backed by `done.json` must never be overridden. A
  row whose runner is not live keeps `DONE`.
- Leave `pending_review_window_active` and its `meta.plan` gate as they are. Rule
  (b)-(d) supersedes it for live runners.

> [!decision] handoff_label = plan_aware For a plan root with a submitted plan
> (`plan_times` non-empty) and no gate member yet, show the label the gate will publish
> next:
>
> - `pending_plan_status_for_tier(tier)` (`TALE` / `EPIC` / `PLAN`) when the plan needs
>   manual review.
> - The tier's approved label (`TALE APPROVED` / `EPIC APPROVED` / `PLAN APPROVED`) when
>   the recorded autonomy covers the plan. Use the same coverage test enrichment already
>   uses (`recorded_auto_covers_plan`).
>
> Get the tier from the already-cached `cached_plan_tier`, resolved in the loader's
> enrichment step, not with new reads in `apply_status_overrides`. Question and pipe
> handoffs, and anything without a submitted plan, still show `RUNNING`.

### 3. Tests

**Wait resolver.** Extend the helpers in
`tests/test_axe_chop_wait_checks_plan_agent_sessions_handoffs.py` or
`tests/test_gate_wait_dependency_pending_followup.py`, or add a focused module. Use
`pid=os.getpid()` with `process_identity=process_identity_token(os.getpid())` for a live
runner. For a dead one, use a non-matching token (e.g. `"<boot_id>:1"`) or `stopped_at`.

1. **W2 shape.** Set up two members:
   - A `--plan` root: `plan_chain_root`, completed `workflow_state.json`, no
     `done.json`, live identity.
   - A `--gate` turn member: `done.json` with `outcome=gated` and `gate_state=answered`;
     meta with `gate_followup_outcome=suppressed` and no follow-up agent.

   Assert that a session waiter gets no `ready.json`, and that
   `resolve_initial_wait_release` does not release either.

2. **W2 with a dead runner** releases. This pins today's fail-open.
3. **Adding `--code`** (parent = root) without `done.json` stays unresolved. Writing its
   `done.json` (`completed`) releases.
4. **Shape C.** Root and gate are done; `--code` has completed workflow and prompt-step
   markers, no `done.json`, and a live identity. Assert no release, then add `done.json`
   and assert release.
5. **Legacy.** Same as 1 but without `process_identity`; behavior is unchanged.
6. **Existing regressions stay green.** This includes the 33.r1 superseded-feedback
   handoff test, `test_completed_plan_root_handoff_without_done_does_not_resolve`, and
   the terminal-blocker tests.

**TUI.** Extend `tests/test_agent_loader_status_override_gate_turn_agent_session.py` or
`tests/test_agent_loader_status_override_promoted_plan_agent_session.py`.

7. **W1.** The plan root row shows `DONE` from workflow markers, with
   `runner_is_live=True`, a submitted plan, a `DONE` main-step child, and no gate. The
   node shows `RUNNING` (Running bucket), or the decision's label when `plan_aware` is
   chosen.
8. **W1 with the runner not live** keeps `DONE`.
9. **Unchanged cases:**
   - W2 (settled gate) still shows `TALE APPROVED`.
   - A running `--code` still shows `WORKING TALE`.
   - A root backed by `done.json` still shows `DONE`.
   - A coder in its finalizer phase still renders `FINALIZING`.

### 4. Verification

- Run `just fix` (or at least `just fmt`), then `sase tool run check`. Run
  `just install-venv` first if the workspace venv is stale; `sase_core_rs` bindings must
  match.
- Optional incident replay:
  1. Copy the planner and gate artifact dirs
     (`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202610/10/20261010142546`
     and `.../20261010143145`).
  2. In the gate's meta, reset `gate_followup_outcome` to `suppressed` and drop
     `gate_followup_agent`.
  3. Rewrite the planner meta's `pid` / `process_identity` to the current process.
  4. Assert `is_resolved("6g.w1.w0.f0")` is `False`.
- No PNG goldens are expected to change, since snapshot fixtures don't model live
  runners. If targeted visual tests show drift, run a targeted
  `just fix-tui-screenshots -- <selector>` and inspect its report before accepting.
- No feature flag. This is a bug fix that only tightens resolution and status while a
  runner is provably alive.
