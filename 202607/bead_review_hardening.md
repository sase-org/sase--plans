---
tier: epic
title: Harden the bead subsystem against the verified gaps from the post-sase-9r/9s
  review
goal: 'The verified bugs and inconsistencies from the five-reviewer bead-subsystem
  audit are fixed: every runner bead mutation runs under the store write lock, retained
  claims are always committed and publishable, claim residue is reconciled and visible
  to doctor, git probe failures fail closed, the managed sync worker clears cooldowns
  and stops leaking process state, the bead_store_refresh chop fits its own time budget,
  the `sase bead work` CLI honors its confirmation/JSON/reporting contracts, legacy
  epic approvals stop blocking the ACE event loop, sase-core bead mutations are atomic
  against each other, dead code left by the recent refactors is removed, and docs/beads.md
  matches current behavior.

  '
phases:
- id: launch_claim_lock
  title: Serialize the launch-claim mutation under the store write lock
  depends_on: []
  size: small
  description: 'launch_claim_lock: hold one bead_store_write_lock span across the
    runner launch-claim mutation and its SDD commit in claim_bead_for_agent_launch,
    matching the sase-9r.1 contract already enforced on the wait-claim and release
    paths.

    '
- id: claim_persistence
  title: Always commit retained claims and make release outcomes tri-state
  depends_on: []
  size: medium
  description: 'claim_persistence: commit and publish a wait claim that is held but
    still uncommitted after a retry, return distinct released/no-op/error outcomes
    from claim release so shutdown can clear markers on no-ops, guard home-mode and
    in-tree claim publication, and enforce the push path''s never-raises contract.

    '
- id: residue_diagnostics
  title: Reconcile claim residue and surface stuck beads in doctor
  depends_on:
  - claim_persistence
  size: small
  description: 'residue_diagnostics: give the bead_claim_checks chop a terminal tombstone
    for dead-owner records so it regains its zero-read steady state, and add report-only
    doctor advisories for dead-owner claimed/in_progress beads and dangling dependency
    edges.

    '
- id: probe_fail_closed
  title: Git probe failures must never read as clean
  depends_on: []
  size: small
  description: 'probe_fail_closed: make bead_state_is_clean fail closed on probe errors,
    treat unexpected `git diff --cached` exit codes as errors in commit_sdd_files,
    and include unmerged_error in repository blockers and rollback-mismatch verification,
    finishing the sase-9r.3 sweep.

    '
- id: sync_worker_hygiene
  title: Managed sync worker cooldown, environment, redaction, and log fixes
  depends_on: []
  size: small
  description: 'sync_worker_hygiene: clear the failed-integration cooldown marker
    on every successful integration path, stop mutating os.environ in-process, redact
    credentials in sync worker git errors via the shared formatter, give detached
    sync logs a unique collision-free writer, and drop the worker''s redundant freshness
    marking.

    '
- id: refresh_chop_budget
  title: Fit the bead_store_refresh chop inside its own time budget
  depends_on: []
  size: small
  description: 'refresh_chop_budget: bound the per-project lock wait of the chop''s
    blocking refresh below the chop timeout, and persist backoff state so a timeout
    kill does not erase it; scoped to compose with the separately planned sase-9u
    refresh_cooldown phase.

    '
- id: bead_work_cli
  title: Restore the bead work CLI confirmation, JSON, and reporting contracts
  depends_on: []
  size: medium
  description: 'bead_work_cli: honor --yes/--yes-to-all on epic resume instead of
    hardcoding force-approval, restore the epic-launch task origin, emit JSON envelopes
    on every bead-id error path, report only real phases as phase beads, diagnose
    plan-file-only flags on bead-id targets, correct misleading error/hint texts,
    and serialize bead-id launches under the epic launch lock.

    '
- id: tui_epic_approval_offload
  title: Move legacy epic-approval preflight off the ACE event loop
  depends_on: []
  size: small
  description: 'tui_epic_approval_offload: run the legacy plan-approval modal''s epic-launch
    preflight and submission through the tracked-task pattern the modern gate path
    already uses, so store materialization and lock waits never block the Textual
    event loop.

    '
- id: core_mutation_atomicity
  title: Make every sase-core bead mutation atomic against concurrent writers
  depends_on: []
  size: medium
  description: 'core_mutation_atomicity: extend with_bead_mutation_lock to all bead
    store mutations (not just the three claim functions), write JSONL through per-process
    temp files, align missing-dependency semantics between ready/blocked reads and
    the epic work planner, and remove the dead OperationOutcomeWire type. Implemented
    in the sase-core repo.

    '
- id: dead_code_cleanup
  title: Remove dead code and duplicated helpers left by the recent bead refactors
  depends_on: []
  size: small
  description: 'dead_code_cleanup: delete the orphaned imports, aliases, unreachable
    branches, duplicate exception arms, and dead parameters identified by the review,
    deduplicate the sync worker''s git runner, and label the last unnamed store-lock
    call sites.

    '
- id: docs_truth
  title: Align docs/beads.md and CLI help with current behavior
  depends_on: []
  size: small
  description: 'docs_truth: rewrite the stale claim-publication and Plan Approval
    Flow sections of docs/beads.md, add the missing sase bead work flags to its flag
    table, and document that --json implies skipped confirmations in the parser help.

    '
create_time: 2026-07-26 11:32:00
status: done
bead_id: sase-9v
---

# Harden the bead subsystem against the verified gaps from the post-sase-9r/9s review

## Context

A five-reviewer audit of the recent bead work (the sase-94, sase-95, sase-9l, sase-9r, and sase-9s series plus the
follow-up fixes through `7ba445a45`) verified every finding below against HEAD before it entered this plan. The focused
bead/sdd-store pytest subset (738 tests) passes at HEAD, so none of these gaps are covered by existing tests.

This epic is disjoint from the open **sase-9u** epic (bead store merge convergence). Two review findings were
deliberately left to sase-9u rather than planned here: last-writer-wins reduction of competing wait claims from two
clones (sase-9u `claim_lifecycle`), and colliding-creation merges that fail only at the reduce step and surface as
uncaught exceptions (sase-9u `merge_integration` / `reader_resilience` — the conflict resolver's reduce-before-write
ordering in `src/sase/bead/conflict_resolver.py` is the safety net and must be preserved there). The
`refresh_chop_budget` phase here deliberately scopes itself to the chop's _own_ time budget and backoff durability so it
composes with sase-9u `refresh_cooldown` (cooldown/TTL gating of doomed integrations) without overlap. A sase-core audit
also confirmed sase-9u's assumption that no Rust changes are needed for one-shot merge: `merge_bead_event_streams`
already handles true-merge-base three-way merges of append-only branches, with parity tests.

## Phase 1 — Serialize the launch-claim mutation under the store write lock

`launch_claim_lock`. sase-9r.1 (`a89f4c059`) established that a bead worktree mutation and its git commit must sit under
one `bead_store_write_lock` span, and applied that to the wait-claim and release paths (`src/sase/bead/claims.py`,
regression-tested by `test_wait_claim_holds_store_lock_from_materialization_through_commit`). The launch-promotion path
was missed: `claim_bead_for_agent_launch` (`src/sase/axe/run_agent_runner_bead.py:25-49`) runs
`project.claim_for_agent_launch(...)` — which writes stream JSONL immediately — with **no** store lock, and only the
later `commit_sdd_store_files` locks internally. In the unlocked window a detached sync worker's integration on the same
clone can rebase over the dirty mutation; the post-verify porcelain comparison then routes to `_abort_and_verify`, whose
`reset --hard` discards the uncommitted claim (the exact incident class the lock comment in
`src/sase/sdd/_git_contention.py:26-30` describes), after which the commit step raises "produced no local SDD commit".

- Wrap the mutation + commit in one `bead_store_write_lock(beads_dir)` span, passing `already_locked` through to
  `commit_sdd_store_files` the same way `claims.py` does (after the `claim_persistence` phase lands its branch collapse,
  simply pass the flag — but do not depend on that phase; write whichever form is current).
- The in-tree branch (mutation only, no commit) should also hold the lock across the mutation for symmetry.
- Audit `commit_epic_graph_checkpoint` (`src/sase/bead/cli_work_from_plan.py`) for the same pattern: epic-creation
  mutations followed by a separately locked commit. If closing that span is a contained change, do it here; if it
  cascades, record the follow-up in the phase notes instead.
- Tests: mirror the existing wait-claim lock-span regression test for the launch path (assert the lock is held from
  mutation through commit, e.g. by injecting a lock factory that records acquisition ordering).

## Phase 2 — Always commit retained claims and make release outcomes tri-state

`claim_persistence`. Four verified gaps in `src/sase/bead/claims.py` and its shutdown caller:

1. **Retained claims are never committed.** In `claim_bead_for_waiting_agent` (`claims.py:147-183`), attempt 1 can
   persist the Rust claim mutation and then fail the commit (e.g. a lock-timeout-flavored error); the retry sees
   claimed-by-same, gets `changed=False`, skips the commit block entirely, prints "Retained claim", and returns success
   — leaving the claim as dirty uncommitted store state that no later path sweeps in (the chop's acquire pass hits the
   same `changed=False` short-circuit). Fix: when `held_by_us` and the store has uncommitted bead state
   (`bead_state_is_clean` from `_sync_git`), run the commit + publish even when `changed` is False.
2. **Release conflates no-op with failure.** `release_bead_claim_for_agent` returns `changed`, and returns False on
   exceptions too; `finalize_runner_shutdown` (`src/sase/axe/run_agent_runner_lifecycle.py:217-236`) clears
   `bead_claim.json` only on True, so a benign no-op (bead already open, closed, or stolen) strands the marker forever.
   Fix: return a small enum/named result (released / nothing-to-release / error); shutdown clears the marker for both
   non-error outcomes.
3. **Home-mode noise.** `project_name` is the truthy string `"home"` in home mode, so every bead-carrying home agent
   with a wait warns "no canonical bead store for project 'home'". Skip the claim attempt for home mode explicitly
   (silent, debug-level at most).
4. **In-tree publication guard + never-raises contract.** `refresh_bead_store` refuses in-tree dirs
   (`src/sase/bead/sync.py:241-243`) but `publish_bead_claim`/`push_bead_work_launch` have no such guard, so an in-tree
   canonical store would have its _primary project checkout_ fetch/rebased/pushed (including unrelated local commits) as
   a claim side effect. Add the same in-tree guard to the publication path (in-tree claim commits ride normal project
   pushes). Separately, `push_bead_work_launch`'s "Never raises" docstring is not enforced — `_find_git_root` can raise
   `SddGitCommandTimeout` and the lock/log setup can raise `OSError` before the internal try. Make the function actually
   never raise (wrap the whole body), since `push_sdd_store_after_commit` and `commit_successful_work_launch` rely on
   it.

Also collapse the duplicated `if already_locked / else` commit branches (`claims.py:159-170` and `:251-262`) into single
calls passing `already_locked=already_locked`.

Tests: retained-claim-after-failed-commit ends committed and published; release tri-state drives marker clearing on
no-op; home mode attempts no claim; in-tree store publication is a no-op; `push_bead_work_launch` returns an error
outcome instead of raising when the git root probe times out.

## Phase 3 — Reconcile claim residue and surface stuck beads in doctor

`residue_diagnostics`. Depends on `claim_persistence` for the tri-state release outcome.

- **Chop steady state.** `sase_chop_bead_claim_checks.py`'s pre-pass keeps every dead, unpromoted, bead-carrying
  artifact record as a `release_candidate` forever — after a successful or no-op release nothing marks the record
  terminal, so the chop opens bead stores every 10s indefinitely, defeating the "cheap pre-pass / zero-read steady
  state" intent of sase-94.3. Write a small tombstone (e.g. `bead_claim_reconciled.json` beside the artifact record, or
  a field the pre-pass already reads) once a release completes with a non-error outcome, and filter tombstoned records
  out in the pre-pass.
- **Doctor visibility for post-promotion residue.** Today a bead whose owner died _after_ promotion is invisible to
  every layer: shutdown skips promoted claims, the chop only considers unpromoted records and only reads
  `Status.CLAIMED`, and `checks_beads.py` queries only `[CLAIMED, OPEN]` with the claimed advisory keyed on missing
  artifacts. Add report-only doctor advisories (no automatic mutation — promotion is documented as permanent):
  - a `claimed` bead whose assignee resolves to a **dead** agent (artifacts present or not),
  - an `in_progress` bead whose assignee matches a runner-promoted agent (`bead_claim_promoted` in agent meta) that is
    dead and whose bead never closed. Each advisory names the bead and prints the manual remedy (`sase bead open <id>`
    or a retry of the owning epic).
- **Dangling dependency advisory.** `sase bead rm` can leave dependents with edges pointing at removed beads; the read
  side treats those as satisfied while the epic work planner errors (see `core_mutation_atomicity`, which aligns those
  semantics). Add a doctor advisory listing dependency edges whose target no longer exists.

Tests: chop reaches zero-store-read steady state after one reconciliation cycle; each new doctor advisory fires on a
synthetic store and stays silent on a healthy one.

## Phase 4 — Git probe failures must never read as clean

`probe_fail_closed`. sase-9r.3 (`a4b9515b5`) established "a failed probe must never read as clean", but three residues
remain:

- `bead_state_is_clean` (`src/sase/bead/_sync_git.py:274-285`) swallows `BeadWorkLaunchCommitError` from the
  `git ls-files` probe and returns True. It is used as the post-commit verification barrier in
  `_checkpoint_and_publish_graph` (`src/sase/bead/cli_work_from_plan.py`) and by `BeadProject.doctor()`. Return False on
  probe failure (fail closed: callers treat "dirty" as the unsafe state); audit both callers for the flipped semantics —
  `checks_beads.py` only expects `OSError` degradation today.
- `commit_sdd_files` (`src/sase/sdd/_commit_store.py:76-84`) treats any non-zero `git diff --cached --quiet` exit as
  "has staged changes", including exit 128. Align with its two siblings (`_commit_bare_git.py:151-154`,
  `_sync_git.py:253-260`), which raise on non-{0,1}.
- `unmerged_error` (added by sase-9r.3) is excluded from `SddRepositoryState.blockers`
  (`src/sase/sdd/_repository_types.py:57-70`) and from `sdd_rollback_mismatch`
  (`src/sase/sdd/_repository_health.py:209-238`), so `_abort_and_verify` can declare a rollback "verified" when the
  final unmerged probe actually failed, and preflight can pass on an unproven index. Include it in both, mirroring how a
  failed status probe flips `valid_worktree`.

Tests: each probe failure produces the fail-closed result; the rollback-verification path reports mismatch when the
unmerged probe errors.

## Phase 5 — Managed sync worker cooldown, environment, redaction, and log fixes

`sync_worker_hygiene`. Five contained fixes in `src/sase/bead/sync_worker.py` and the async push path in
`src/sase/bead/sync.py`:

- **Cooldown clearing.** `refresh_bead_store` documents the principle "any successful integration ends the clone's
  failed-integration cooldown, not only the pull path that recorded it" (`sync.py:279-284`), but the most frequent
  successful path — `_run_locked_sync`, which integrates after every claim/commit publish — and
  `refresh_materialized_store` (`src/sase/sdd/_store_materialization.py`) never call `clear_failed_integration_marker`.
  Call it on success from both, so one failed pull no longer suppresses launch-time integration for the full cooldown
  window on a clone that has since integrated fine.
- **Redundant marking.** `integrate_sdd_repository` already marks freshness on success; drop the worker's duplicate
  `mark_bead_integration` call (`sync_worker.py:94-98`).
- **Process-wide env mutation.** `_run_locked_sync` sets `os.environ["GIT_TERMINAL_PROMPT"]="0"` (`sync_worker.py:62`),
  which now runs in-process from `push_bead_work_launch` — permanently disabling git prompting for every later
  subprocess in a long-lived host (runner, TUI). Pass the variable via the git runner's subprocess env instead.
- **Credential redaction.** `_git_error` (`sync_worker.py:129-133`) duplicates `format_git_error` minus
  `_redact_credentials`, so push/fetch stderr with `https://user:token@host` URLs reaches sync logs and stderr warnings
  unredacted. Use the shared redacting formatter from `_repository_health` (and fold the third copy in
  `_bead_manifest_repair.py` into the same shared helper).
- **Detached log writers.** `push_bead_work_launch_async` opens the log with mode `"w"` as the child's stdout/stderr
  while the worker appends JSON records to the same file, so child stderr (e.g. a traceback) overwrites the earliest
  records; `_new_sync_log_path` is second-granularity, so two pushes in one second truncate each other. Use a unique log
  name (pid + monotonic or uuid suffix) and open the child's handle in append mode.
- **Backoff pruning.** In `sase_chop_bead_store_refresh.py`, backoff-state entries for projects that no longer have bead
  waiters are only deleted on a successful refresh; prune entries for projects absent from the current waiter set.

Tests: cooldown cleared after a successful worker sync; `os.environ` unchanged after an in-process publish; a
token-bearing remote URL is redacted in the logged error; two same-second async pushes write to distinct logs.

## Phase 6 — Fit the bead_store_refresh chop inside its own time budget

`refresh_chop_budget`. The chop runs every 30s with `timeout: "2m"` (`src/sase/default_config.yml:460-464`) and serially
refreshes every project with bead waiters, but each blocking refresh can wait up to the 180s worktree-mutation lock
bound (`DEFAULT_WORKTREE_MUTATION_LOCK_TIMEOUT_SECONDS`, `src/sase/sdd/_git_contention.py:30`) — one contended project
already exceeds the whole chop budget, and a timeout kill (`chop_runner_script` SIGKILLs at the deadline) bypasses the
in-process backoff recording, so the next run 30s later re-attacks the same contended lock with no backoff and the chop
surfaces raw "timed out after 120s" failures.

- Give the chop's refresh call a bounded, chop-appropriate lock wait (well under the chop timeout divided by the
  waiter-project count, floor ~10s — the chop reruns every 30s, so declining a contended refresh is cheap). Thread the
  bound through `refresh_bead_store`'s lock acquisition rather than inventing a parallel path.
- Record the backoff-state entry for a project _before_ starting its refresh attempt (mark in-flight, finalize on
  completion), so a timeout kill leaves a backoff record instead of erasing the attempt.
- Keep this scoped to budget/backoff mechanics: do not implement the failed-integration cooldown or freshness-TTL gating
  here — sase-9u `refresh_cooldown` owns that, and both changes must compose (note the interaction in the phase notes
  for whoever lands second).

Tests: a lock held by another process makes the chop decline quickly and record backoff instead of blocking past its
budget; a simulated kill mid-refresh still leaves a backoff entry.

## Phase 7 — Restore the bead work CLI confirmation, JSON, and reporting contracts

`bead_work_cli`. Verified contract breaks in the `sase bead work` pipeline:

- **Resume force-approves destructive cleanup.** `resume_linked_epic` (`src/sase/bead/cli_work_from_plan_resume.py:103`)
  passes `yes_to_all=True` to `launch_epic_bead_work` while its own `yes_to_all` parameter is never used — so a bare
  `sase bead work plan.md` on an already-linked plan (exactly the relaunch case with live agents to kill) skips both the
  destructive-cleanup prompt and the launch prompt, contradicting the documented `-y`/`-Y` split. Thread the real flag
  through. Then audit `src/sase/bead/epic_launch.py:209` (`success = error is None and bool(epic_id)`): once resume can
  decline, a declined launch must not send a success notification — include `result.launched`.
- **Origin regression.** `submit_epic_launch_task` declares `origin: str = "api"` and the only production caller
  (`src/sase/_plan_approval_epic.py`) never passes it, so every epic approval records origin `api` — sase-95.7
  explicitly introduced `epic-launch` as "the only record of where the work came from". Pass `origin="epic-launch"` at
  the call site (or change the default) and pin it with a test.
- **`--json` on bead-id errors.** `handle_bead_work`'s early bead-id error paths
  (`src/sase/bead/cli_work_handler.py:363-374` issue-not-found and non-plan type, `:425-432` non-epic tier) print plain
  stderr and exit even under `--json`, while sibling paths emit JSON envelopes. Emit the same
  `{"ok": false, "mode": "bead_id", ...}` envelope on all three.
- **Misleading error text.** `:369-372` says "is_ready_to_work only applies to plan beads" — leaking an internal flag
  name at the user who ran `bead work`; reword to match the tier check's phrasing.
- **Child epics reported as phases.** `phase_bead_ids` (handler `:410`) and the resume path's "Phase beads" rendering
  (`cli_work_from_plan_resume.py:78`, `cli_work_from_plan.py:334`) use `get_epic_children`, which returns _all_ children
  including child epics. Filter to `IssueType.PHASE` wherever the output means phases.
- **Silently ignored flags.** `-a/--artifacts-dir` and `-c/--cl-name` only take effect in plan-file mode; on bead-id
  targets they are accepted and dropped, while `--parent` gets an explicit error. Give them the same explicit rejection.
- **Post-launch failure hint kills live agents.** The failure notification's resume hint always includes `--yes-to-all`
  (`epic_launch.py:229-238`), including for the launched-agents commit failure where docs promise the agents keep
  running; pasting it force-wipes the healthy clan. Use `--yes` there, matching the CLI-side hint.
- **Bead-id mode bypasses the epic launch lock.** Only plan-file mode serializes under `epic_plan_launch_lock`; wrap the
  bead-id launch branch in the same lock keyed by epic so a manual retry cannot interleave with a plan-file launch of
  the same epic.
- **Manual-push hint path.** `cli_work_commit.py`'s hint `cd {beads_dir.parent.parent}` is right only for in-tree
  stores; compute the repo root (`find_git_root`) instead.

Tests: resume prompts without `-Y` and skips with it; origin pinned; JSON envelope asserted for each early error; phase
filtering; flag rejection; hint contents.

## Phase 8 — Move legacy epic-approval preflight off the ACE event loop

`tui_epic_approval_offload`. In the legacy plan-approval modal path, `on_dismiss`
(`src/sase/ace/tui/actions/agents/_notification_modals.py:335-356`) calls `prepare_epic_launch` inline on the Textual
event loop. That call chain runs `require_epic_launch_store_health` → `resolve_beads_location(materialize=True)` (can
clone/pull the plans sidecar over the network) → `store_git_write_lock(..., mutates_worktree=True)` (waits up to 180s).
Approving an epic from a legacy notification while anything holds the store lock freezes the whole TUI. The modern gate
path already does this correctly (`_notification_plan_gate.py` runs `execute_plan_approval_response` in a tracked task
worker).

Per the TUI performance rules (never block the event loop; slow user-initiated operations run as tracked background
tasks with optimistic UI → thread worker → UI-thread completion effects): route the legacy path's `prepare_epic_launch`
through `_submit_tracked_task()` / the `LaunchTaskMixin` shape used by the modern path, with the error notify moved to
the completion callback. Read `sase/memory/tui_perf.md` via `sase memory read` before implementing.

Tests: existing modal tests updated; add one asserting the dismiss callback returns without blocking (e.g. the preflight
stub records it ran off the UI thread / via the task-submission seam).

## Phase 9 — Make every sase-core bead mutation atomic against concurrent writers

`core_mutation_atomicity`. This phase is implemented in the sibling **sase-core** repository (open it with the
`/sase_repo` skill; follow that repo's own build/test/release conventions and `just rust-check` + `just bead-perf-smoke`
back in this repo afterwards). Python code in this repo should need no changes.

- **Lock coverage.** `with_bead_mutation_lock` (crates/sase_core/src/bead/mutation.rs) guards only
  `claim_for_agent_launch`, `claim_for_agent_wait`, and `release_agent_claim`. Every other mutation — `create_issue`,
  `update_issue`, `preclaim_epic_work_plan`, `open_issue`, `close_issues`, `remove_issues`, `add_dependency`,
  `set_ready_to_work` — is an unlocked load → mutate → whole-store `save()` that rewrites **every** stream file plus
  `issues.jsonl`. A concurrent CAS claim and an unrelated `sase bead update` can therefore erase each other's events
  (verified failure mode: an update loaded pre-claim rewrites the stream from its stale snapshot, silently dropping the
  committed claim event). Wrap all mutations in the same lock. Keep the lock file identity unchanged (the reused
  `beads.db` path) unless a dedicated lock file can be introduced compatibly in one release — if not, document the known
  caveat that deleting `beads.db` mid-flock splits the lock across inodes.
- **Torn writes.** `write_file_atomic` (crates/sase_core/src/bead/jsonl.rs:367) uses the single fixed temp path
  `path.with_extension("tmp")`; two concurrent writers truncate each other's in-flight temp file and can publish a torn
  rename. Use a per-process unique suffix (pid + counter).
- **Missing-dependency semantics.** `has_active_blocker` (src/bead/read.rs:283-299) treats a dependency whose target is
  absent as satisfied, while `build_epic_work_plan` (src/bead/work.rs:129-142) treats a missing out-of-epic blocker as
  active and hard-errors — so `sase bead ready` unblocks a bead the work planner refuses to schedule. Align them: treat
  a missing blocker as inactive (satisfied) in the work planner too, but include a warning in the plan result naming the
  dangling edge (the Python-side doctor advisory from `residue_diagnostics` gives it a durable diagnostic home).
- **Dead code.** `OperationOutcomeWire` (src/bead/wire.rs:297-305) is constructed and consumed nowhere — only
  re-exported. Remove it.

Tests (Rust): concurrent update + claim property/loom-style or serialized-interleaving test asserting no event loss;
unique-temp-path unit test; work-planner dangling-blocker warning; plus the existing parity suites stay green.

## Phase 10 — Remove dead code and duplicated helpers left by the recent bead refactors

`dead_code_cleanup`. All verified as reference-free at HEAD (re-verify each with grep before deleting; update Symvision
whitelists per `sase/memory/symvision.md` if lint requires):

- `src/sase/sdd/store.py`: 15 underscore-aliased imports plus `_STORAGE_VALUES` imported and never used or re-exported;
  `SddPushAfterCommit` (`_store_types.py`) has zero consumers.
- `find_live_name_collisions` (`src/sase/bead/cli_work_plan.py:37-45`) and its re-export chain — orphaned by the
  force-reuse redesign; keep `_legacy_land_agent_name` (still used).
- `commit_plan_file`'s unreachable in-tree branch (`src/sase/bead/cli_work_from_plan_store.py:173-183`): both callers
  pass `push_after_commit=False`, so the `commit_sdd_files_for_exec_plan` branch and its `Literal["async"]` type are
  dead.
- `cli_work_handler.py:666-677`: byte-identical `except BeadWorkLaunchCommitError` and `except Exception` arms; keep
  one.
- `commit_bead_work_launch`'s dead `title` parameter (`_sync_git.py:108-121`); `db.py:480` `_get_dependencies` alias;
  `epic_launch.py`'s test-only `started_at` parameter.
- `snapshot_managed_changes` never returns a `None` ref — tighten the return type to `str` and drop the caller's dead
  `snapshot_ref or recovery_ref` fallback (`_repository_recovery_snapshot.py`, `_repository_recovery.py:270`).
- `sync_worker._git` duplicates `_repository_health.default_git_runner` byte-for-byte — reuse it (coordinate with
  `sync_worker_hygiene`, which touches neighboring lines; whichever lands second rebases trivially).
- Pass `op=` labels at the last bare `store_git_write_lock` call sites (the recovery-marker helpers and
  `_repository_recovery_reaper.py:80-83`) so timeouts stop logging "an unnamed store write".
- `_sdd_store_record_label` (`_commit_store.py:256-266`) derives `sdd_dir.parent.parent`, wrong for sidecar layouts —
  compute the workspace dir per storage kind or fall back explicitly.
- Evaluate `preclaim_epic_work` (`project.py:220-238`, facade + binding): no production caller since the JIT-claim
  redesign; if the binding is deliberately retained API, correct its stale docstring, otherwise remove the Python
  wrapper (leave the Rust binding to the core phase's judgment).

Tests: the suite stays green; no new tests beyond adjusting any that referenced removed symbols.

## Phase 11 — Align docs/beads.md and CLI help with current behavior

`docs_truth`. Three verified doc/help drifts:

- `docs/beads.md:109` still says wait claims are "committed locally, but never pushed — a claim is host-local runtime
  state", but `publish_bead_claim` has synchronously published claim transitions since `7ba445a45`. Rewrite the
  claim-lifecycle bullet (commit locally, then best-effort synchronous publish; failures warn and never roll back).
- The "Plan Approval Flow" section (`docs/beads.md:556-567`) describes the removed tracked-task submission, the
  planner-subprocess fallback, and foreground `sase plan approve --kind epic` — all deleted by sase-9s.5/9s.6. Rewrite
  to match the detached-task reality (align with the already-correct `docs/sdd.md:103-110`).
- The `sase bead work` flag table (`docs/beads.md:461-467`) omits `-Y/--yes-to-all`, `-a/--artifacts-dir`, and
  `-c/--cl-name`; add them. Conversely `--json` implying skipped confirmations is documented but absent from `--help` —
  add it to the parser help text (`src/sase/main/parser_bead.py`).
- While editing, sweep the rest of docs/beads.md against the `bead_work_cli` phase's changes (JSON envelopes on bead-id
  errors, `-a`/`-c` rejection) — coordinate wording if that phase has landed; otherwise document current behavior only.
