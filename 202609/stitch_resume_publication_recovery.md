---
tier: epic
status: done
title: Repair stitch recovery and retry unpublished artifact links
goal:
  Prevent stale workspace origins from breaking stitch resume, finish run-owned pending
  stitch steps without duplicating commits or completed tracking, retry stranded
  artifact-link publications, and complete and close sase-yg, sase-xi, and sase-ye with
  verified evidence.
phases:
  - id: origins
    title: Validate managed origins at stitch execution boundaries
    size: medium
    depends_on: []
    description:
      "origins: reconcile a proven stale managed-clone origin before provider selection
      on create and resume, cover retained workspaces, and preserve unrelated remote
      configuration and local work."
  - id: stitch-recovery
    title: Resume owned checkpoints and preserve unpushed evidence
    size: medium
    depends_on:
      - origins
    description:
      "stitch-recovery: complete sase-yg and sase-xi by recording resumed push failures,
      validating checkpoint ownership, resuming pending hooks and tracking before new
      work, and preserving bounded retries and accurate marker identity."
  - id: publication
    title: Retry and report aging artifact-link publications
    size: medium
    depends_on: []
    description:
      "publication: complete sase-ye by persisting retry state for hidden document
      sidecars and sweeping unpublished work without requiring new mutations, with
      bounded backoff, ownership checks, and aging diagnostics."
  - id: verification
    title: Verify recovery end to end and close the three tasks
    size: medium
    depends_on:
      - origins
      - stitch-recovery
      - publication
    description:
      "verification: reconcile the incident against current published history, test the
      combined tree, close sase-yg, sase-xi, and sase-ye with evidence, and leave the
      already-closed sase-y6 epic intact."
proposed_by: bbugyi200.athena.08g
bead_id: sase-yh
create_time: 2026-09-09 19:52:49
---

- **PROMPT:**
  [prompts/202609/stitch_resume_publication_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/stitch_resume_publication_recovery.md)
- **BEAD:**
  [sase-yh](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yh/README.md)

# Repair stitch recovery and retry unpublished artifact links

## Scope and tier

This is an epic because managed-workspace repair, resumable finalizer execution, and
scheduled sidecar publication have distinct state, failure modes, and regression suites.
Each implementation phase is bounded medium work. Origins precedes stitch recovery;
publication can proceed independently; verification consumes all three implementations.
The existing task beads remain the work's acceptance checklist. Do not create duplicate
tasks or close them merely because this plan was accepted.

Implementation repositories are `sase`, linked `sase-core`, and, only where its retained
workspace setup needs changes, linked `sase-github`. Open linked repositories using
`sase repo open` and use the returned paths. New deterministic recovery/authorization
decisions and retry/aging policy belong in Rust core, with PyO3 binding tests and thin
Python adapters. Filesystem inspection, Git/process execution, and host orchestration
remain Python. Extend narrow core contracts; do not port unrelated finalizer machinery.

## Evidence and diagnosis

The completed run behind family `sase-y6.land` is shell `sase-y6.land--2`, artifacts at
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/08/20260908110601`. Resolve
it again with `sase agent show sase-y6.land`; names and workspace claims can change. The
bare family chat selector returned no transcript. Stable evidence came from `done.json`,
`commit_state.json`, `commit_results.json`, `finalizers/commit/attempt-1.main.stdout`,
and `finalizers/commit/conflict_repair_response.md` in that run. Its published agent
page was stale (reported waiting), so the completed local run is authoritative.

The failure sequence was:

1. Create made commit `e2010dd9f`, then encountered a real `Justfile` rebase conflict.
   The repair completed the rebase and verified its resolution. This was a recoverable
   integration conflict, not the final blocker.
2. Resume selected the bare-Git provider from a legacy origin pointing to the primary
   non-bare checkout. Push of `master` to that checkout failed with
   `refusing to update checked out branch: refs/heads/master`. The local repaired commit
   was reported as `9207ca284`.
3. Hardening commit `77b7e5ea0` had already added reusable-clone origin repair to
   `workspace_provider/utils.py:ensure_git_clone_at`. That preparation-time fix does not
   ensure a long-lived/retained workspace is checked when a stitch resumes:
   `workflow_resume.py` goes directly from checkpoint to `get_vcs_provider(cp.cwd)`. The
   land family began before that hardening commit. The GitHub preallocated setup branch
   also bypasses `ensure_workspace_checkout`. Validate these entry paths with
   regressions; do not assume every continuation rematerializes its clone.
4. `vcs_finalize_commit` returns plain `(False, err)` after push failure, while
   `vcs_create_commit` uses `_format_unpushed_commit_failure`. The surviving checkpoint
   has no completed dispatch and null commit SHA/tree; its ledger contains bead commits
   but no unpushed marker for the main repository. This is `sase-yg`.

There has already been later recovery: the investigation checkout's HEAD and fetched
`origin/master` include `7b934722f87355adbe97e4ea22da5ffb353be847`, with the same
intended subject and `SASE_AGENT`/`SASE_BEAD` provenance. Its final diff is only a
flake-baseline adjustment. Thus verify current history and tracking before any
operational replay; matching subjects alone do not establish that every intended change
or hook completed. Do not manufacture another landing commit or rewrite the historical
failed outcome.

The requested beads were all READY when inspected with
`sase bead show --no-links sase-yg sase-xi sase-ye`:

| Bead      | Remaining behavior                                                                                                                              |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `sase-yg` | Preserve local-commit evidence when a resumed stitch cannot push; coordinate marker and checkpoint recovery.                                    |
| `sase-xi` | On finalizer retry, resume the same run's pending hook/tracking checkpoint before considering the repository finished or starting a new stitch. |
| `sase-ye` | Retry unpublished machine artifact-link commits even when no subsequent mutation occurs; persist backoff and report aging failures.             |

Audited reads of these bead references failed because their published pages were absent;
their CLI records supplied the descriptions. Use the IDs directly to reread their live
state and attach implementation/verification evidence through supported bead commands.
Audited context also includes `agent:sase-y6.land` and
`plan:202609/fix_commit_finalizer_retry_loop.md`. The older plan explains why the
controller's no-progress guard, fresh-result retry classification, and proof against
discarded dirty work must remain intact. Its deferred hook-resume work is `sase-xi`.
`sase-y6` and its five children are already closed; this plan does not reopen them.

## Origins

Starting points: `src/sase/workspace_provider/utils.py`, checkout marker/registry and
ownership APIs, `src/sase/workflows/commit/workflow.py`,
`src/sase/workflows/commit/workflow_resume.py`, and the finalizer's provider-selection
boundary. Check `sase-github/src/sase_github/scripts/gh_setup.py` for preallocated
setup.

Introduce a non-destructive origin preflight usable by fresh create, resume, and
retained workspace setup. Run it before origin-based provider selection and Git
synchronization. Resume must inspect the checkpoint's repository, not the process's
incidental cwd. Keep this distinct from destructive clone materialization/checkout
preparation.

Use trusted managed-checkout/project identity and live ownership/claim rules to resolve
the canonical remote. Repair only an origin proven to be the managed clone's primary
checkout/clone seed, using the same project's authoritative remote. Normalize relative
paths, file URLs, and symlinks before comparison. Preserve legitimate local bare remotes
and unrelated intentional remote configuration. Check the effective push destination,
including explicit push URLs; never claim healing succeeded when pushes still resolve to
the stale checkout. Ambiguous/missing identity, an unavailable canonical remote, or a
failed rewrite must produce an actionable failure before a commit or push. An unmanaged
repository retains its supported behavior. Do not change the primary's branch or
`receive.denyCurrentBranch`, force-push, reset, or discard local edits to repair this.

Put any new frontend-neutral reconciliation decision in `sase-core`; Python gathers
facts and performs the authorized remote change. Reuse existing identity APIs rather
than adding a second project locator. Avoid adding CLI options for this fix.

Acceptance tests use disposable Git repositories: a canonical bare remote, a non-bare
primary left on its default branch, and a managed worker with a stale primary origin.
Exercise fresh create, checkpoint resume, and retained/preallocated entry paths. Assert
the canonical remote receives the commit, provider classification sees the healed
origin, and the primary HEAD/index/worktree remain unchanged. Cover correct origins,
local bare remotes, explicit push URLs, relative/symlink aliases, lookup/set-url errors,
missing ownership evidence, and dirty/conflicted workers whose work is preserved. Extend
`tests/workspace_provider/test_utils.py`, managed workspace tests, and the GitHub
plugin's setup tests only where their paths change.

## Stitch recovery

Starting points: `vcs_provider/plugins/_git_commit_dispatch.py`,
`workflows/commit/{checkpoint,workflow,workflow_resume,commit_tracking}.py`,
`finalizers/{commit,commit_dispatch,commit_repair,controller,ledger}.py`, and their
existing recovery/integration tests.

### Preserve and distinguish local, pushed, and fully tracked results

Route a resumed push failure through the same durable evidence path as fresh create.
Record actual post-amend/rebase SHA/tree, canonical repository identity, run identity,
original method/payload provenance, `pushed: false`, and useful push diagnostics. Keep
the checkpoint and do not publish a success singleton, advance dispatch, or delete
recovery state when the push failed. Avoid hardcoding `create_commit` for another
supported checkpoint method. Preserve the original SASE footer tags and rewritten body.
Surface persistence failure rather than silently representing missing evidence as
success.

Update consumers in the same phase. In particular,
`commit_repair.resolve_commit_conflict` currently treats any new matching marker as
completed repair. A new unpushed marker must not skip required resume/push/hook work.
Also, `_upsert_commit_results_marker` keys by `(cwd, result)`, so a SHA rewritten by
retry/rebase can leave an old `pushed: false` row that is selected forever. Give the
checkpointed operation stable identity and explicitly supersede/settle its earlier
marker when verified recovery completes. Preserve audit evidence and unrelated commits;
do not delete every marker for a repository. Only completed tracking may become the
ordinary success marker. Repeated recovery must produce one stitch entry for this work.

### Resume owned pending steps before fresh dispatch or clean acceptance

Add a Rust-backed decision for pending-checkpoint eligibility and required recovery
action, based on host-supplied facts. Bind newly saved checkpoints to the original run,
repository, operation, and accepted work/payload, not merely a subject or a file path.
Reuse authenticated finalizer context and baseline evidence where available. Define
versioned compatibility explicitly: legacy checkpoints remain readable for supported
manual recovery; automatic legacy recovery requires equivalent independently verified
ownership evidence. Missing/foreign/malformed/unknown-version/conflicting checkpoints
must fail closed with diagnostics and leave evidence intact.

Inspect pending checkpoints for all accepted repositories before the already-clean
shortcut and before a new create can overwrite the run's checkpoint. The current
checkpoint is a singleton across repositories: finish its matching operation first and
preserve ordered dispatch of remaining repositories. Do not execute the same checkpoint
once through the unpushed lane and again through the generic retry lane.

For a run-owned checkpoint whose dispatch/file hooks completed, call `run_stitch_resume`
to execute only the pending after-hook and tracking steps. A pushed commit or clean
worktree alone is insufficient while those steps remain pending. Recompute dirty state
after recovery before considering follow-up changes. Keep true conflicts in the existing
one-shot conflict-repair lane. Consume attempt budget before executing recovery and
preserve subprocess timeout/output bounds, fresh-result retry classification, and
no-progress termination. Repeated hook/push failures remain visible failures once the
budget is exhausted; they neither spin nor create duplicate commits.

A checkpoint's expected commit may be rewritten by supported rebase/amend operations;
use proven operation lineage and Git evidence, never arbitrary matching HEAD subjects.
Do not reinterpret a mismatched/unproven checkpoint as a successful no-op merely because
the working tree is clean. Preserve supported no-commit-needed semantics only where
their existing ownership and upstream evidence justify them.

Completed checkpoint steps are skipped. Pending external hooks may be retried after a
crash between their side effect and checkpoint persistence; preserve/document their
idempotency requirements rather than promising transactional exactly-once execution of
arbitrary shell commands. Ambiguous non-idempotent external work needs explicit recovery
diagnostics. Commit creation and stitch tracking must still remain deduplicated.

Required regressions:

- Real conflict, repair, rejected resumed push: retained checkpoint and unpushed marker;
  then a successful retry settles the same operation and writes one completed stitch.
- Retry that rebases to a new SHA retires the old pending marker; a third invocation
  does not push again or duplicate ledger/tracking identities.
- After-hook fails once on a committed clean repository and then succeeds: one commit,
  no rerun of completed file hooks/dispatch, pending hook retried, tracking completed.
- Hook failure with additional dirty paths and multi-repository obligations finishes the
  checkpoint before deciding what newly attributable work still needs a commit.
- Foreign run/repository, payload mismatch, malformed/legacy checkpoint without proof,
  absent commit proof, persistent failure, timeout, and exhausted-budget cases refuse
  safely. Existing no-progress-loop and conflict-repair tests continue to pass.

Extend the real-Git tests in `tests/test_vcs_provider_commit_first_rebase.py`,
`tests/test_finalizers_live_e2e_cycles.py`, the commit-workflow resume suites, marker
tests, and execution-ledger tests. Add focused Rust/PyO3 tests for the new decisions.

## Publication

Starting points: `src/sase/sdd/_artifact_link_commit.py`,
`src/sase/sdd/_artifact_link_machine_store.py`,
`src/sase/scripts/sase_chop_artifact_link_backfill.py`, and
`src/sase/bead/{_sync_publication,sync_worker}.py`.

The current publication check calls `push_bead_work_launch` once and returns an error
string. The worker itself has bounded immediate retries; the missing feature is durable
retry across later housekeeping ticks. Reuse that worker's integration, semantic bead
repair, lock ordering, noninteractive Git behavior, and preservation guarantees rather
than introducing raw push/reset logic or another daemon.

Persist a small per-root publication record outside tracked sidecar content, keyed by
project/role/repository and intended remote/upstream. Include first-pending time,
attempt count, last attempt/error/log, and next due time; track the pending revision
without letting a rebase or additional local commit reset its age. Atomic writes and
bounded existing locking must protect concurrent writers/ticks. Register failed machine
publication and also discover legacy unpublished heads without a record.

For legacy discovery, use the oldest verifiable local-only commit time as aging evidence
when available, otherwise record first observation explicitly; never invent an earlier
failure time. A change of configured remote/upstream requires revalidation before retry.

Add publication retry before the backfill sweep's early-return paths, for each enabled
project's eligible hidden document sidecars (`plans` and configured custom roles). Do
not require `committed=True`, new links, an outbox entry, or successful derivation.
Resolve inventory/ownership without first forcing network integration for every role:
the current store resolver integrates all roles before returning and can throw before
the sweep reaches pending work. One offline/broken role must not prevent another role or
project from being retried. Never scan arbitrary working directories or mutate a human
primary/nested sidecar. Retain the separate agents/beads publication mechanisms.

Implement pure due/aging/state transitions in Rust with bindings. Initial policy: retry
at the next hourly tick, then exponential backoff capped at six hours; warn when pending
for six hours. Keep retrying after warnings rather than silently abandoning work. Use
injected time in tests and stable first-pending timestamps across process restarts. Do
not add a configuration surface unless an existing scheduling policy can be reused; if a
setting is added, document and update `default_config.yml` together.

Each due root gets a bounded worker attempt, including a deadline propagated to locks
and subprocesses so Git's existing long timeouts cannot exhaust the chop's 240-second
work budget unnoticed. Preserve lock order; contention defers work. Persist fair
progress so early projects cannot permanently starve later roots. Confirm remote
publication with fresh upstream/remote evidence after the attempt, then clear pending
state. Missing upstream and network failure are diagnostics, not false success. Retain
pending work through unsupported conflicts and recovery; do not allow an integration
reset to turn an unpublished local commit into an apparent completed publication.

Report attempted/published/deferred/failed/aged counts plus project, role, age, last
error, next due time, and worker log in the existing chop result/log. Correct the stale
claim that hidden host-owned clones are lost on workspace eviction. Keep immediate
foreground publication errors visible while explaining scheduled recovery where it
actually applies.

Test a committed link index whose push fails, restart the worker state, make no further
mutations, advance the clock, and run the chop: the remote must eventually receive the
same work. Cover legacy discovery, custom roles, backoff/aging, repeated ticks,
concurrent locks, already-published state, divergent remote history, dirty hidden roots,
missing upstream, time budgets/fairness, resolver failure, and refusal of primary roots.
Extend `tests/test_axe_chop_artifact_link_backfill.py`, machine-store/ownership tests,
and artifact-link persistence tests; include a disposable-remote integration test.

## Verification and closure

Each phase rereads its repository instructions and `lint_and_test.md`. Use
`just install` when needed, focused regressions, then `just check` in every changed
repository. For `sase-core`, its complete check must include the PyO3 crate and
Python >=3.12; core-only Cargo tests are insufficient. Build/test against the changed
binding locally, and use the supported release/dependency workflow so installed SASE
receives the API before callers require it. Do not hand-edit Rust release versions;
release-plz owns them.

The verification phase integrates the combined tree and runs `just check-full` only
through `/sase_monitor` with TESTING/TESTED statuses. Preserve exact command outcomes;
fix failures caused by this work. Handle unrelated failures with the existing evidence
and task-deduplication procedures. Test helpers must isolate repositories, environment,
hooks, and runtime state from the host. No production fault injection is required.

Perform a final incident audit using the original run's completed artifacts and freshly
verified canonical remote history. Establish whether the completion snapshot, whitelist
resolution, and flake-baseline change are present even if the final commit diff became
smaller during rebase. Compare operation provenance, actual content and ancestry,
pending checkpoint state, publication evidence, and Patch/stitch tracking. If anything
remains, finish it through supported host-owned recovery in an authorized checkout under
the original operation's attribution. Resolve paths through `sase repo`, not by opening
a sibling workspace directly. Do not automatically adopt another run's checkpoint into
the new implementation agent or hand-edit `done.json`, checkpoints, or STITCHES to claim
success. If the historical operation already completed, record that fact and avoid
replay.

Close each requested task only after its acceptance tests and combined verification
pass, using `sase bead close <id> --note "<fix, commit/artifact evidence, tests>"`:

- `sase-yg`: resumed push failure is recorded and recoverable with correct settlement.
- `sase-xi`: pending hook/tracking work is resumed once within the retry budget.
- `sase-ye`: an unchanged unpublished sidecar is retried after restart and aging is
  visible, while machine ownership and time bounds are preserved.

Confirm the three close results and their publication through supported bead status
commands. Re-read their state before closure in case another worker has completed one;
use an evidence note for already-closed tasks rather than reopening them. Leave
`sase-y6` closed. Record a concise root-cause and recovery report with links to the
implementation and verification evidence. Use `/sase_final` for host-owned completion of
all changed repositories; this plan grants no need for manual commits or PR creation.
The new epic's land agent performs its normal final verification and closes its own
parent epic only after these task obligations are satisfied.
