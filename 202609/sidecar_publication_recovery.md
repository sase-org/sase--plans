---
tier: tale
title: Recover concurrent sidecar publication before agent startup
goal:
  Start agents after safely publishing compatible sidecar divergence while preserving
  unpublished work when recovery fails.
size: medium
proposed_by: bbugyi200.athena.0jb
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0jb](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0jb.md)
  - [bbugyi200.athena.sase-xe.16.11.7.14.6.7.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.16.11.7.14.6.7.2/README.md)
- **COMMITS:**
  - [3153478](https://github.com/sase-org/sase-core/commit/3153478b832db52ec92ea64c3b988e47d1d6bd23)
    — fix: add sidecar publication policy
  - [37588f6](https://github.com/sase-org/sase-core/commit/37588f67f9a42297adcf3e589c8a8fe61a5077f4)
    — feat(continuation): validate launch requester continuations

# Recover concurrent sidecar publication before agent startup

Allow a new SASE agent to start when a previous workspace occupant left unpublished
sidecar commits and the remote has advanced with compatible changes. Fetch, integrate,
and publish those commits before evicting the old clone. If publication cannot be
verified, preserve the clone and its recoverable work and report the concrete cause.

This is a medium tale: one coding agent can deliver a small Rust policy/binding
addition, Python integration with existing Git transaction helpers, and focused
regression tests. It does not require separate feature phases or replacement of the SDD
synchronization engine. The investigation and this proposal constitute the large
planning work.

## Investigation and current status

The failed execution was `sase-zl.3`, workflow `ace(run)-260911_085039`, on 2026-09-11.
These retained sources establish what happened:

- Notification `a609433d-60e4-424c-8648-16480f16778a`, sender `user-agent`, timestamp
  `2026-09-11T08:51:43.550624-04:00`, reports `_WorkspaceBeadEvictionRefused`.
- The completed workflow log is
  `~/.sase/workflows/202609/gh_sase-org__sase_ace-run-260911_085039.txt`. It records one
  unpublished commit in the plans sidecar, two rejected ordinary pushes to
  `sase-org/sase--plans` (`main -> main`, `non-fast-forward`), and the exception from
  `prepare_launch_workspace_repos`. The runner reports 49 seconds total. It failed
  before invoking the coding provider.
- Sidecar-protection notifications `818ec5d5-083d-47b1-a1e7-f90ee25779a4` and
  `799ef822-043e-4c0b-a6fc-777eae9bf006` record the same failures at 08:51:39 and
  08:51:40 EDT and recovery refs `refs/sase/recovery/20260911T125139Z-main-bd845dfdf6`
  and `refs/sase/recovery/20260911T125140Z-main-1e1da005ef`.
- The original transcript and referenced `error_report.md` were unavailable after the
  restart. The retained workflow log and notifications are completed evidence; they do
  not establish the contents or mergeability of the unpublished commit.
- The replacement workflow `ace(run)-260911_085927` passed workspace preparation, reused
  fresh sidecars, and invoked its provider. Its log is
  `~/.sase/workflows/202609/gh_sase-org__sase_ace-run-260911_085927.txt`.
  `sase agent list -a -j` reported `sase-zl.3` RUNNING with start time
  `2026-09-11T13:00:10.488118+00:00`. This proves startup recovered; it does not prove
  completion of the phase or establish how the earlier unpublished work was resolved.

Audited context: `bead:sase-zl.3` and `plan:202609/monitor_continuations.md`. The phase
implements continuation capture; the failure occurred before that work and is outside
the monitor epic's implementation. Do not modify its phase assignment, status, running
process, or plan.

At inspected SASE revision `8f6e65361d1803160446bb3bdff16c061c7fa050`, the underlying
limitation remains:

1. `src/sase/axe/runner_workspace.py::_protect_sidecar_repo` counts unpublished commits,
   calls `_push_sidecar_repo`, and counts again.
2. `_push_sidecar_repo` executes only `git push`. It performs no integration or retry
   after a non-fast-forward rejection.
3. The first workspace-cleaning pass preserves a recovery ref and warns. The later
   strict eviction pass repeats the push and raises `_WorkspaceBeadEvictionRefused`.
4. Commit `54d9c112a` introduced generic sidecar protection. Its refusal prevents loss
   of unpublished work and must remain. The missing behavior is reconciliation of
   ordinary remote divergence, not permission to bypass that protection.

Existing building blocks include the non-destructive
`sdd._repository_transaction.integrate_sdd_repository`, its rollback-verifying
transaction, the bounded retry example in `bead/sync_worker.py`, and the
integration-then-retry pattern in `sdd/_store_adoption.py::_push_sidecar_store`. The
latter has adoption-specific push arguments and must not simply be called from workspace
preparation. Existing plans-sidecar eviction tests use an ahead-only clone or an
injected push failure; they miss concurrent remote advancement.

## Implementation

### 1. Add the missing publication decision in the Rust core

Open `sase-core` with `sase repo open sase-core -r '<specific reason>'`, falling back to
`sase repo open gh:sase-org/sase-core` if it is not configured. Use only the returned
checkout and its instructions. Paths beginning `crates/` below are relative to it; all
other source paths are relative to the SASE checkout.

Add a narrow deterministic sidecar publication policy in `crates/sase_core/src/`, using
the existing `git_query` module or a small adjacent module. It accepts observed Git push
results and attempt count and decides success, integrate-and-retry, or stop with
preserved work. Recognize explicit non-fast-forward/fetch-first rejections;
authentication, transport, timeout, hook rejection, and unknown failures are terminal. A
generic “failed to push some refs” message alone is insufficient to retry. Use at most
three push attempts, including the initial push. Unknown or failed publication
verification cannot authorize eviction.

Expose and test the policy through `crates/sase_core_py` and a thin typed adapter in
`src/sase/core/`. Follow the established binding/wire conventions and validate inputs.
Keep the classification and retry decision in Rust with no Python fallback. Python
continues to own Git subprocesses, timeouts, lock acquisition, repository observations,
recovery refs, and notifications. Do not port the existing Git integration engine or
change unrelated publication callers as part of this fix.

### 2. Wire transactional reconciliation into workspace sidecar publication

Extract the host publication operation into a focused SDD helper if necessary to keep
`runner_workspace.py` within the repository's size limits. Preserve the existing
ahead-only fast path and configured upstream; do not hard-code `main`, change tracking
configuration, or use force push.

After a retryable rejection, invoke `integrate_sdd_repository` to fetch and rebase
against the intended upstream, then retry the push under the Rust policy's bound. Honor
the configured upstream remote; if the existing transaction's hard-coded `origin` fetch
prevents this, narrowly parameterize that fetch while preserving existing callers'
default. Do not use `integrate_machine_managed_sdd_repository`: its reset-based recovery
can move unpublished work off HEAD and make an ahead count of zero insufficient evidence
for deleting the clone.

Use the existing cooperative store write-lock and Git timeout mechanisms. Reuse the
transaction's lock ownership rather than nesting incompatible locks; an unavailable lock
is a preservation outcome. Preserve pre-existing dirty files and in-progress Git
operations. A failed rebase must restore the transaction's starting branch, HEAD, and
index and leave no new rebase markers. Do not add manual textual conflict resolution,
implicit staging/commits, or a reset-to-upstream shortcut.

After integration and publication, verify the resulting local history is published to
the intended upstream. An equivalent change already on the remote may be dropped by a
successful rebase; prove the artifact contents survive rather than requiring the
original commit SHA to survive rebasing. A failed or malformed verification probe is
unknown, never zero unpublished commits. A remote advancing again between integration
and push consumes another bounded attempt.

On exhaustion or any non-retryable failure, keep unpublished work reachable and return
the final cause to the existing recovery-ref/refusal path. A successful reconciliation
must avoid failure notifications and allow normal clone eviction/recreation. Failure
diagnostics must identify the sidecar, failed operation, attempts, retained recovery ref
when available, and why preparation stopped. Do not broaden this into notification
deduplication or agent-retry UI work.

Retain the specialized semantic bead-store sync path, combined plans/beads layouts,
machine-shared agents-sidecar guard, primary-workspace behavior, and retry-handoff
workspace preservation. Both preparation passes should benefit from the helper; after
the first succeeds, the strict pass should see no unpublished commits and need no
additional publication attempt.

### 3. Add regression coverage and verify the integrated result

Extend `tests/test_bead/test_workspace_sidecar_bead_eviction.py`, with additional
focused helper tests if needed. Use disposable local bare remotes and clones; never run
workspace cleanup, Git fixtures, or trial publication against live agent checkouts.
First demonstrate the missing case fails on the pre-fix implementation, then make it
pass through the real helper and strict eviction entry point.

Required cases:

| Scenario                                                                                                  | Required result                                                                                                 |
| --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| One local plan commit and a disjoint remote commit, with both fresh and stale tracking refs               | Fetch/rebase/push publishes both artifacts; eviction/recreation proceeds; no protection-failure notification.   |
| Equivalent local change already published remotely                                                        | Integration converges safely; no repeated rejection or data loss.                                               |
| Another remote writer wins after the first integration                                                    | A subsequent bounded retry succeeds; no lost remote or local artifact.                                          |
| Remote keeps advancing through all three push attempts                                                    | Stop within the bound, preserve the clone and all unpublished work, report exhaustion, and perform no eviction. |
| Genuine conflicting edits to the same plan                                                                | Abort/verify restoration, preserve the clone and recovery ref, and report the conflict.                         |
| Fetch/push timeout, authentication failure, hook rejection, unavailable lock, or failed publication probe | No false success, unsafe reset, eviction, or unbounded retries; concrete diagnostics.                           |
| Reconciliation needed with a dirty worktree, detached HEAD, missing upstream, or existing Git operation   | Preserve the pre-existing state and fail safely when publication cannot be established.                         |
| Non-default upstream branch and a configured non-origin upstream remote                                   | Integrate and verify exactly the publication target, without changing tracking configuration.                   |

Retain the existing ahead-only successful publication test and the failed-publication
preservation tests. Add a runner/preparation integration regression covering both passes
so a recovered sidecar permits provider launch, while an unrecovered sidecar prevents
it. Re-run relevant bead protection and shared-agents-sidecar lock tests. Cover the Rust
policy and actual PyO3/Python adapter together, including rejection classification and
the attempt limit.

Read `lint_and_test.md` with `/sase_memory_read` before implementation verification. Run
the required `just check` in SASE after changes and focused tests above. In sase-core
run its `just check` or `./scripts/check.sh`, with Python >=3.12 available;
`cargo test -p sase_core` alone omits required binding tests. Build the modified wheel
for isolated cross-repository verification. If SASE's test selection escalates or the
change touches the broadening set, run `just check-full` through `/sase_monitor` with
TESTING/TESTED and finish its continuation. Fix attributable failures and rerun the
relevant checks until they pass.

## Release and completion

Recheck the implementing checkout before editing: another agent may have fixed this path
after this investigation. If so, demonstrate the required concurrent-publication and
preservation behavior and add only missing coverage. Recheck `sase-zl.3` with the status
CLI to report its latest state, without restarting or modifying it.

Release core/bindings before Python requires the new API. The inspected dependency
window is `sase-core-rs>=0.34.0,<0.35.0`; ratchet the dependency and lockfile only to an
actually available compatible release containing the binding. Follow release-plz's
version ownership and host-owned finalization; do not hand-edit Cargo package versions
or manually commit, branch, or publish. Do not leave Python calling an unavailable
binding or claim local-wheel verification proves deployment.

Completion requires a demonstrated fix for compatible remote divergence, preservation on
unrecoverable publication, passing Rust/binding/Python checks, and a usable paired
dependency. Report the historical failure, current restarted-agent status, verified
behavior, and any remaining deployment limitation separately. No repair of the live
plans clone is currently justified: the restart passed that preparation step. Do not
alter canonical memory, generated skills, the monitor epic, or unrelated retry behavior.

The planning turn creates only this scratch proposal. Validate it first with
`sase plan validate sase_plan_sidecar_publication_recovery.md --explain`, edit as
needed, revalidate without `--explain` until successful, and submit with
`sase plan propose sase_plan_sidecar_publication_recovery.md` before implementation.
