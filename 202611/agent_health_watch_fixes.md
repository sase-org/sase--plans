---
tier: epic
title: Agent health watch fixes — publication wedge, leaked claims, stranded handoffs,
  bead open misuse
goal: 'Agent publication heals itself from conflicts on regenerable index pages and
  escalates once otherwise; finished monitors and failed staged handoffs stop leaking
  workspace claims and can be re-adopted; agents can no longer demote their own beads
  by running `sase bead open` to view them.

  '
phases:
- id: pub-policy
  title: Core rebase-conflict policy for agent publication
  depends_on: []
  size: small
  description: 'pub-policy: add a sase-core decision that says whether a publication
    rebase''s conflicted paths are regenerable index pages that can take the upstream
    side, or must escalate, and expose it through the Python binding.'
- id: pub-heal
  title: Self-healing and one-shot escalation in the publication job
  depends_on:
  - pub-policy
  size: medium
  description: 'pub-heal: resolve index-page rebase conflicts in the agents clone
    automatically, otherwise park the project with one deduplicated notification and
    a doctor warning, and stop leaving unpushed commits behind.'
- id: retain-scope
  title: Scope monitor retention evidence to the monitor's own workspace
  depends_on: []
  size: medium
  description: 'retain-scope: stop epic-launch monitors from probing the plan agent''s
    former workspace during terminal retention, and fix the swallowed Workspace held
    notification.'
- id: decisions-commit
  title: Commit the Decision Sheet sibling on bead-work plan launches
  depends_on: []
  size: small
  description: 'decisions-commit: make `sase bead work <plan>` commit the archived
    plan-decisions sheet together with the plan, as plan approval already does.'
- id: stale-claims
  title: Release leaked and dead claims in stale-running cleanup
  depends_on:
  - retain-scope
  - decisions-commit
  size: medium
  description: 'stale-claims: re-settle already-leaked epic-launch monitor claims
    with the corrected scope after publishing their stray decision sheets, and apply
    the held-claim TTL to dead gate claims stuck in a non-terminal state.'
- id: staged-join
  title: Keep reserved joined runs alive through staged adoption
  depends_on: []
  size: medium
  description: 'staged-join: stop the end-of-turn cleanup and starter watchdog from
    stopping tool runs that a staged monitor has reserved, and fall back to a fresh
    owned run when a joined run is gone at adoption.'
- id: staged-response
  title: Make settled stages adoptable after a provider error, with muse parity
  depends_on: []
  size: medium
  description: 'staged-response: persist a staged response when the provider fails
    after a valid stage, let adopters fall back to the provider failure record, and
    give muse the staged-handoff wait guard and directive Claude and Codex already
    have.'
- id: staged-readopt
  title: Make sase agent readopt work for held monitor handoffs
  depends_on:
  - staged-join
  - staged-response
  size: medium
  description: 'staged-readopt: keep the staged monitor''s owner lookup alive across
    held cleanup so explicit re-adoption can find the member, and make failed-adoption
    notices name the remedy that actually works.'
- id: open-guard
  title: Stop agents from demoting beads with sase bead open
  depends_on: []
  size: small
  description: 'open-guard: fix the worker prompts that tell agents to run `sase bead
    open` on their own bead, refuse agent reopen of non-closed beads with a `sase
    bead read` hint, and clarify help text.'
- id: open-actor
  title: Record the real actor and skip no-op reopen events in sase-core
  depends_on:
  - open-guard
  size: medium
  description: 'open-actor: thread the mutating actor through the core reopen mutation
    instead of the bead creator, make reopening an already-open bead a no-op, and
    move the sase-core pin.'
proposed_by: bbugyi200.apollo.cp--5
proposed_with: claude/opus
create_time: 2026-11-06 05:46:42
status: wip
bead_id: sase-1rg
---

- **PROMPT:** [prompts/202611/agent_health_watch_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/agent_health_watch_fixes.md)
- **BEAD:** [sase-1rg](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rg/README.md)

# Plan: Agent health watch fixes

## Context

A five-check health watch (2026-11-05 23:49 UTC to 2026-11-06 10:25 UTC) of every SASE
agent on apollo found the problems below. Throughout the watch the bob-cli-hg epic kept
advancing and no agent failed, but these issues kept recurring or holding state:

- **F1 — agent publication wedged for a week.** Axe job `agent_publication` fails about
  once a minute since 2026-10-30. Each pass, `git pull --rebase` in the hidden agents
  clone (`~/.sase/projects/<key>/repos/agents`) conflicts on `README.md` and
  `users/bbugyi200/README.md` for two projects: sase (local commit `7b4d2fef9c`, 663
  behind) and bob-cli (local `ce3ca4df9d`, 722 behind). The job aborts and retries the
  same commit forever. That is 80–88 errors an hour, and every error in every hourly
  digest comes from this one job, crowding everything else out of the 100-entry
  `recent_errors.json` window. `active_remaining` grew 790 → 802 over the watch. Each
  stuck commit is mostly `agents/*/README.md` plus this machine's own `manifest.json` /
  `snapshot.json` / `state.json` / `meta.json`. Only the two index READMEs conflict.
- **F2 / F2b — failed staged-monitor adoption strands workspaces.** bob-cli-ha.land
  (muse) was held with "ordinary provider response was not persisted for this staged
  handoff". cd--code was held with "tool run … already has a stop request; nothing to
  join". Both workspaces stay pinned, and both monitors show `staged` forever.
  `sase agent readopt` cannot recover either one today.
- **F5 — epic-launch monitors leak workspace claims.** 23 claims (22 bob-cli, 1 sase)
  since 2026-11-01 are pinned by `ace-monitor` with dead pids. No "Workspace held"
  notification was sent. The bob-cli pool has grown to 34 checkouts, at 250–700 MB each.
- **F5b — an unpinned `ace-gate` claim with a dead pid** has held a sase workspace since
  2026-09-19.
- **F6 — agents run `sase bead open <own-bead>`** expecting a viewer, which demotes
  their `in_progress` phase bead to `open`. That was 62+ calls in November, the latest
  at 07:38Z on 2026-11-06. No work has been lost yet.

Rust core boundary: status rules, the bead event actor, the retention policy and the
publication path classifier live in sase-core. Probing, adoption dispatch, prompts, CLI
guards and job orchestration stay in Python. A phase that changes a sase-core binding
the Python side calls must also move `sase-core-revision.txt` past its sase-core commit,
as `docs/rust_backend.md` describes. When two phases move the pin, keep the newest
revision. Workers should read the reference memory that matches what they touch: `ace`,
`cli_rules`, `docs_writing`, `sase_beads`, and `lint_and_test` before finishing.

## Core rebase-conflict policy for agent publication

`crates/sase_core/src/agent_publication_recovery/replay.rs` already sorts agents-repo
paths into three classes: BROWSING (regenerable), OWNER_CONTENT, and ARCHIVE_OBJECT.
Next to that classifier, add a pure decision such as
`decide_publication_rebase_conflict(conflicted_paths)` that returns one of two results:

- **`resolve_upstream { paths }`** when every conflicted path is a regenerable index or
  browsing page that the next pass rewrites from manifests. That covers root
  `README.md`, `users/*/README.md`, machine, hood and session pages,
  `agents/*/README.md`, and `schema.json`.
- **`escalate { paths, reasons }`** for anything else: `prompts/**`, `artifacts/**`,
  owner content (snapshots, manifests, `state.json`, `meta.json`, `agents/*/chat.md`,
  `prompt.md`), `files/objects/**`, or unknown paths.

Expose it through `sase_core_py` and the Python adapter
`src/sase/core/agent_publication_recovery_replay.py`. Add Rust unit tests for both
results, including a mixed set that must escalate. Pin the exact paths that conflict on
apollo today as a `resolve_upstream` case.

## Self-healing and one-shot escalation in the publication job

**Resolve regenerable conflicts.** The pull step in
`src/sase/agents_sync/_commit_publication_transaction_engine.py` (pull/abort around
lines 63–80) currently aborts on any `pull --rebase` failure. Replace that with a shared
helper. `src/sase/agents_sync/git_sync_transaction.py` (around lines 141–147) should use
the same helper, so `sase agent sync` behaves the same way. When a rebase stops on
conflicts:

1. List the unmerged paths (`git diff --name-only --diff-filter=U`) and ask the
   pub-policy decision.
2. On `resolve_upstream`, first pin the pre-rebase local HEAD under a recovery ref. Use
   the pattern in `src/sase/sdd/_repository_recovery_git.py` (`refs/sase/recovery/...`),
   so the local commit stays reachable. Then take the **upstream** side for each
   conflicted path, add it and continue the rebase. Remember that during a rebase
   `--ours` is the upstream side. Repeat for a bounded number of replayed commits. The
   pass then renders as usual, which rewrites the index pages from current manifests.
   Log one info line per healed project.
3. On `escalate`, or if resolution fails, abort the rebase exactly as today and return a
   distinct `failure_stage="diverged_replay"`.

**Park instead of retrying every pass.** In `src/sase/agents_sync/publication_job.py`, a
`diverged_replay` project does the following:

- It writes a per-project marker keyed on (HEAD, upstream). Pulls for that project are
  skipped, and its doorbell re-ring (around lines 201–211) is suppressed, until HEAD or
  upstream changes or a one-hour cooldown passes.
- It is counted in a parked or deferred counter after its first error, not in `errors`.
- It emits one `upsert_notification` with
  `dedup_key="agent_publication_diverged:<project_key>:<head>"` **and** a
  `plus_one_note`, which the notification store requires whenever a dedup key is set.
  The notification body names the stuck commit and the conflicted paths. It also names
  the documented `tools/agents_publication_recover_replay` report and apply commands
  (`docs/contribute/agents-sidecar-sync.md`, "Stale-replay recovery").

Automatic stale-replay apply is intentionally out of scope; escalation stays manual.

**Prevent new wedges.**

- Today, when a pass commits and then the push fails for a non-race reason, the commit
  is left behind. That happens in the retry path of
  `_commit_publication_transaction_retry.py` (around lines 215–261) and in the timeout
  or kill paths of `_commit_publication_drain.py` and `_commit_publication_locked.py`.
  In those cases, drop the pass's own commit, the way the retry path already drops
  `HEAD^` when this pass committed. Preserve any `files/objects` first.
- Only drop a commit created in the same pass whose rows are still unacknowledged. The
  rows stay queued and are rebuilt next pass.

**Doctor.** `src/sase/doctor/checks_agent_publication_sidecar.py` already collects the
ahead/behind/rebase state but only stores it. Make it WARN, with the recovery next step,
when the hidden clone is both ahead and behind upstream or the diverged marker is set.

**Tests** (`tests/agents_sync/`):

- Two-clone fixtures where only index pages conflict: the pass heals, pushes and drains.
- An owner-content conflict escalates once: exactly one notification, and no repeated
  pulls inside the cooldown.
- The marker clears when HEAD changes.
- Update `test_git_sync.py` (around line 443) to match.
- Doctor WARN.

Move the sase-core pin.

**Acceptance on apollo after deploy.** The two stuck commits conflict only on
regenerable READMEs. The first pass should rebase and push them, and `active_remaining`
should start falling from about 800.

## Scope monitor retention evidence to the monitor's own workspace

**Root cause.**

- `start_epic_launch_monitor` (`src/sase/bead/epic_launch.py`) runs the launch monitor
  in its own leased workspace (`inherit_lane_workspace_claim=False`). Its
  `monitor_starter_artifacts_dir` is still the plan agent's artifacts dir.
- At terminal settlement, `collect_monitor_repo_observations`
  (`src/sase/monitor/settlement_fresh_owner.py`, around lines 86–156) folds the
  starter's evidence in with the monitor's own:
  - `discover_workspace_retention_repos`
    (`src/sase/turns/_workspace_retention/discover.py`) takes repo paths verbatim from
    the starter's `finalizer_baseline.json` and `opened_linked_workspaces.json`, forcing
    positive custody, and marks the starter's checkout as primary.
  - `_collect_dirt` (`settle.py` around line 166) prefers the starter's evidence dir
    over the monitor's own.
- Probing a path in the plan agent's former, possibly reused, workspace fails. sase-core
  `decide_workspace_retention` turns that failure into `retain` /
  `evidence_unavailable`. Older records show `missing_baseline_with_custody`, because
  the baseline was looked up only in the starter dir.

**Fix.**

- Fold starter evidence in only when the starter ran in the same workspace, matching
  workspace number or checkout dir. Otherwise drop starter-derived observations whose
  paths are outside the monitor's checkout.
- Merge baselines from both dirs instead of choosing one.
- Set `is_primary` only when the path is the monitor's own checkout.
- Make sibling targets (`sibling:beads`, `sibling:plans`) resolve against the monitor's
  own checkout. Today they come from the inherited linked-repos environment in
  `commit_finalizer_state/_sibling_targets.py`.
- Key per-repo diff snapshots by repo id plus path, so two repos with the same id no
  longer overwrite each other.
- Keep the sase-1pj.7.1.5.1 rule: a known repo inside the monitor's own checkout whose
  probe fails still retains as uncertainty. Leave
  `test_unavailable_owned_checkout_retains_as_uncertainty` green.

**Notification.**

- `notify_retained_workspace` (`src/sase/turns/workspace_retention_surface.py`, around
  lines 90–138) passes a `dedup_key` without a `plus_one_note`. Every call raises
  `ValueError: dedup_key requires plus_one_note`, which is logged and swallowed, so no
  monitor, gate or staged retention notice has ever been stored.
- Pass `plus_one_note` and `plus_one_timestamp`.
- Replace the mocked `upsert_notification` in
  `tests/turns/test_workspace_retention_policy.py` with a real-store test.

**Tests.**

- A monitor with `monitor_starter_artifacts_dir` pointing at a starter whose baseline
  and opened-repo records name repos in a deleted foreign workspace releases its
  workspace.
- A same-workspace starter still contributes evidence.
- A retained monitor stores exactly one notification.

## Commit the Decision Sheet sibling on bead-work plan launches

`archive_plan_file` (`src/sase/sdd/plan_archive.py`, around lines 150–176) copies
`<plan>.plan-decisions.json` next to the archived plan. The `sase bead work <plan>` path
then commits only the plan: `cli_work_from_plan_launch.py` around lines 116–121 calls
`commit_plan_file(paths=[plan_path])` in `cli_work_from_plan_store.py`.

The approval path commits both files (`_plan_archive_approval.py`,
`_archive_commit_paths`). Because the bead-work path doesn't, the decisions sheet never
reaches the plans repo, and it stays as untracked dirt in the launching monitor's plans
checkout. 22 of the 23 leaked claims have that file.

Commit the same path set as approval by reusing `_archive_commit_paths`. Add a test that
a bead-work plan launch with a decisions sheet leaves a clean plans checkout and commits
both files.

## Release leaked and dead claims in stale-running cleanup

Extend `run_stale_running_cleanup` / `cleanup_stale_running_entries`
(`src/sase/axe/hook_jobs.py`, stale-running cleanup module). Today the existing
`held_claim_ttl_days` (default 14) would only release these claims about two weeks after
each leak.

**Already-leaked epic-launch monitor claims.**

- These are pinned `ace-monitor` claims with a dead pid whose
  `terminal_workspace_retention` record probed paths outside the claimed checkout.
- If the claimed checkout's plans repo holds an untracked
  `<YYYYMM>/<plan>.plan-decisions.json` for a plan already committed to the plans store,
  publish those bytes through the same SDD plans-store commit API decisions-commit uses.
  Do not hand-run git in the retained checkout.
- Remove the untracked copy only after verifying the committed bytes match.
- Then re-run settlement with the corrected retain-scope logic. Release on a clean
  result, and keep retaining (now with a working notification) when other dirt remains.
- Settle the record through `_settle_stale_retention_record`.
- Never delete anything else. Be idempotent and log each claim.

**Dead gate claims (F5b).**

- `gate_claim_is_releasable` (`src/sase/gate_turn/claims.py`) releases only with a
  terminal marker. An `accepted_failed` gate turn defers forever: sase-core
  `gate_decision/policy.rs` classifies it, and `gate_turn/_reclaim_pending.py` (around
  lines 227–228) skips it.
- So a gate whose settler died mid-`settling`, with no `done.json` and no decision
  record, keeps its claim.
- Apply `held_claim_ttl_days` to gate claims whose pid is dead and whose marker is not
  terminal. Release the claim and keep the gate record.

**Tests.**

- Cover the leaked-monitor shape: stray decisions sheet, foreign starter evidence.
- Cover the 2026-09-19 gate shape: `gate_state=settling`, no `done.json`, no decision
  path.
- Test that a real dirty checkout still retains.

## Keep reserved joined runs alive through staged adoption

**Root cause.**

- A staged `sase monitor start -J` only writes `staged_join_reservation` on the tool
  run; the join itself is written at adoption.
- `stop_unjoined_detached_runs` (`src/sase/tool/detach_cleanup.py`, around lines
  115–147) runs from `invoke_agent`'s `finally` in `src/sase/llm_provider/_invoke.py`.
  It and the starter watchdog `_watch_starter` (`src/sase/tool/adopt.py`, around lines
  172–214) only check `run["join"]`. So they stop the reserved run ("starter agent …
  ended without joining") seconds before adoption.
- sase-core `activate_staged_join` (`tool_run/store/handoff.rs`, around lines 729–735)
  then refuses `StopRequested`, and the handoff is held.
- This happens whenever a staged join targets a run that is still running when the turn
  ends.

**Fix.**

- Skip runs with a live `staged_join_reservation` in both places. Held cleanup and
  discard already release the reservation, so the watchdog remains a backstop.
- At adoption, when `_activate_staged_join` (`src/sase/monitor/start_launch.py`, around
  lines 246–297) finds the joined run stopped or settled without a usable result,
  reserve a fresh owned ToolRun for the frozen `execution_argv`. Extend
  `_reacquire_settled_tool_run` (`src/sase/monitor/_start_stage_adopt_flow.py`, around
  lines 123–170), which explicitly skips joins today.
- Drop "(joined)" from the label, and record the replacement run id on the monitor
  member, so follow-up evidence points at the new run.

**Tests.** Add a reserved-run case to `tests/tool/test_detach.py`, and an
adoption-fallback case to `tests/monitor/test_monitor_start_stage_adopt.py`.

## Make settled stages adoptable after a provider error, with muse parity

**Root cause.**

- `.sase_handoff_response.json` is written only when the provider succeeds
  (`src/sase/llm_provider/_invoke.py`, around lines 282–295).
- `finalize_staged_after_provider_error` (`_invoke_errors.py`, around lines 56–104)
  writes `.sase_provider_failure.json` and settles finalizers, but writes no response.
  Nothing reads the failure record.
- Both adopters hold when the response is missing (`run_agent_exec_staged_dispatch.py`,
  around lines 517–529, and `run_agent_exec_staged_recovery_adopt.py`, around lines
  296–307).
- bob-cli-ha.land's muse turn kept polling for about three hours after staging, then hit
  a network error. Its joined check had already finished, so adoption would otherwise
  have worked.

**Fix.**

- On a provider error after a valid stage, persist a best-effort staged response:
  partial provider output, or a synthesized reply that cites the failure record. Mark it
  as partial, so the follow-up prompt says the starter's last reply is incomplete.
- Let both adopters fall back to `.sase_provider_failure.json` when no response exists,
  so stages already stranded this way become re-adoptable.
- Bring muse to parity with commit `fdad1aa7bf`:
  - Call `staged_handoff_blocks_continuation()` from the muse wait guard
    (`src/sase/llm_provider/muse_provider.py`, around lines 475–507; helper in
    `_wait_guard.py`).
  - Add the "when they report Staged, finish through /sase_final … do not wait for the
    monitored command" sentence to `_muse_directive.py`, matching the Claude directive.

**Tests.**

- `tests/test_staged_failure_boundaries.py`: an error after a stage is adoptable, and a
  failure-record-only stage is adoptable.
- `tests/llm_provider/test_muse_wait_guard.py`.

## Make sase agent readopt work for held monitor handoffs

**Root cause.**

- `cleanup_staged_monitor(held=True)` (`src/sase/monitor/start_stage_cleanup.py`, around
  lines 101–104) deletes the staged-owner index, and the staged payload has no member
  dir.
- So `sase agent readopt` holds again with "staged monitor member is unknown"
  (`run_agent_exec_staged_recovery_adopt.py`, around lines 321–343).
- `tests/test_staged_recovery_service.py` (around line 316) monkeypatches the index
  read, which hides this.

**Fix.**

- Keep the owner index on held cleanup, and delete it only on adoption or discard. Also
  persist the member artifacts dir in the staged payload as a fallback lookup.
- Make the failed-adoption notice match reality. Name `sase agent readopt <agent>` when
  re-adoption can succeed; otherwise tell the user to salvage the workspace's work and
  dismiss. The runner path retains with `notify_on_retain=False`
  (`run_agent_exec_staged_dispatch.py`, around lines 138–170).
- Coordinate with in-progress bead sase-1ra.8, which settles held or orphaned staged
  monitors into a terminal state in the same cleanup module. Read it first, build on
  whatever has landed, and keep its custody rules.

**Tests.** An end-to-end readopt of a held monitor handoff without monkeypatching the
owner index. Also cover the two shapes from the health watch: a missing response with a
failure record, and a joined run stopped before adoption.

## Stop agents from demoting beads with sase bead open

**Root cause.**

- The worker prompts `bd/work_phase_bead` and `bd/work_task` in
  `src/sase/default_config.yml` (around lines 2436–2437 and 2466–2468) say "run
  `sase bead open {{ bead_id }}` only if the work is genuinely unfinished".
- That step is meant only for the closed-bead host-notice case, but it reads as a
  general step, and a fresh phase is always unfinished.
- sase-core `run_open` demotes any status, including `in_progress` and `claimed`. A
  demoted bead would also fail a host-driven close with `INELIGIBLE_STATUS`
  (`bead_action/policy.rs`, around lines 144–147).

**Fix.**

1. Reword both prompts so the reopen step sits inside the "host notice says the bead was
   already closed" clause. `src/sase/axe/run_agent_runner_bead.py` (around lines
   160–174) already gives the scoped instruction.
2. Add an agent guard. When an agent identity is present (`acting_agent_name()`, as
   `_refuse_agent_bead_show` in `src/sase/bead/_cli_query_show.py` uses it) and the
   target bead is not closed, refuse with exit 2: "`sase bead open` reopens a closed
   bead; to view a bead run `sase bead read <id> -r '<why>'`".
   - Apply it on both paths: `execute_bead_cli` in `src/sase/main/bead_fast_path.py`,
     next to the assigned near-miss check, and `handle_bead_open` in
     `src/sase/bead/cli_crud_lifecycle.py`.
   - Human use is unchanged, including the bead doctor's advice to reopen stale
     `claimed` beads.
3. Change the help text in `parser_bead_lifecycle_state.py` and `cli_admin_onboard.py`
   to say it reopens a closed bead, sets status=open, and is not a viewer. Follow
   `cli_rules`, and regenerate the CLI reference docs per `docs_writing`.

**Tests.** Extend `tests/test_bead/test_cli_open.py`, and add an agent-guard test
mirroring `test_bead_show_agent_guard.py` for both the fast and slow paths.

## Record the real actor and skip no-op reopen events in sase-core

**Root cause.** `run_open` in `crates/sase_core/src/bead/mutation/close_remove.rs`
(around lines 25–118) records `issue.created_by` as the `IssueOpened` actor. Phase beads
inherit the epic creator, so history blames the wrong session. It also always records an
event and `changed: true`, even when the bead is already open.

**Fix.**

- Add `actor: Option<String>`, falling back to the store owner as close does.
- Make reopening an already-open bead a no-op: no event, `changed: false`.
- Thread the actor through PyO3 `py_bead_open`, `bead_mutation_facade.open_issue`, and
  `handle_bead_open` (via `resolve_mutation_author`).
- Route `open` off the Rust fast path as `create` already is, or add an actor input to
  `bead_cli_execute`.
- Add Rust tests (the close-actor test in `mutation/tests/close.rs` is the template) and
  Python tests.
- Move the sase-core pin.
- The same creator-as-actor pattern exists in update, claims and remove. Record it as a
  `PROPOSED FOLLOW-UP:` note instead of widening this phase.

## Out of scope

- **F3:** zombie staged monitor `5kej4t` is already tracked by sase-1r4 / sase-1ra.8.
- **F4:** the 2026-11-05 axe timeout burst did not recur in five checks.
- **F7:** a single degraded follow-up on 2026-11-05 is a watch item only.
- **Host and user actions:** salvaging and dismissing the two stranded agents, and the
  spinning `pinentry-curses` process.
