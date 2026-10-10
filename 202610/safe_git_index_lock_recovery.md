---
tier: tale
title: Restore safe Git index-lock recovery to TUI updates
goal:
  TUI and CLI updates automatically recover abandoned Git index locks through one
  bounded policy while preserving active or uncertain locks and reporting recovery
  decisions.
size: medium
proposed_by: bbugyi200.athena.0zf
create_time: 2026-10-10 09:32:27
status: wip
---

# Restore safe Git index-lock recovery to TUI updates

## Scope and sizing

This is a medium tale: one implementation agent can fix a demonstrated runner bypass,
strengthen its existing shared recovery policy, and verify the resulting behavior in
`sase` and the linked `sase-core` repository. These changes form one bounded fix rather
than independent features. Open `sase-core` through `sase repo open sase-core` and read
its instructions before working there. All paths below are relative to the named repo;
do not use the planning agent's checkout paths.

Automatically finish an update when its obstacle is an abandoned index lock. If a live
writer still owns the lock, or safety checks cannot establish that removal is safe,
preserve it and finish the bounded attempt with a specific explanation. Never report
success without a successful Git command, and never wait indefinitely or kill an
unrelated process to force an update through.

No new CLI command, configuration switch, feature flag, memory change, or TUI layout is
needed. Do not broadly wrap arbitrary reporter commands in retries, restart whole
multi-step updates, change merge/reset semantics, reinstall the global tool, or delete
locks in the user's working repositories during implementation. Use temporary test
repositories for destructive regression fixtures.

## Findings and evidence

The supplied `~/tmp/git_index_lock_failure.txt` records this sequence against the
editable `sase-core` checkout: fetch succeeds, status is clean, ancestry is `0 6`, and
`git merge --ff-only origin/master` fails with:

```text
error: Unable to create '.../sase-core/.git/index.lock': File exists.
Another git process seems to be running in this repository...
Updating 6df3bed3..6ed9c3d8
```

The implementation already has retry/recovery, but the TUI bypasses it:

1. `src/sase/dev_update/command.py::run_dev_update_command` wraps Git subprocesses in
   `run_with_git_lock_retry`, supplies a default timeout, and prepares a noninteractive
   Git environment.
2. `src/sase/ace/tui/modals/plugins_browser_comprehensive_update_execution.py` injects
   `reporter.dev_command_runner()` into `_execute_tui_dev_update`. The individual and
   combined editable-update paths in `plugins_browser_sase_update_procs.py` do the same.
3. `src/sase/ace/tui/session_proc_reporter.py::dev_command_runner` independently calls
   `self.run` and converts its result. It never invokes the dev-update command policy.
   Its phase markers match the supplied log. `self.run` combines stderr into stdout,
   which must remain usable as lock-error evidence.

The suspected exit-code gap is not the explanation: Git 2.47.3's fast-forward merge path
returns 1 on checkout failure, and the current classifier already accepts an explicit
lock-creation error with exit 1. The existing tests exercise a fake exit-128 `git add`
through the CLI runner and basic reporter streaming, but do not cover a real locked
fast-forward through the injected TUI runner.

There are also safety gaps in `src/sase/git_lock_retry.py`: any positive unchanged retry
window permits deletion, the 15-second age alternative can override previously observed
identity changes, `stat`/path resolution follows a lock symlink, and there is no
live-owner check. Two eager age-only cleanup paths exist in
`axe/runner_workspace_prepare.py::clear_stale_git_index_lock` and
`agents_sync/git_sync_ops.py::_clear_stale_agents_index_lock`.

The log does not identify the process that created this particular orphan. Do not claim
a proven crash/termination cause. Git normally cleans locks on exit and handled signals,
but its lock is also a pending index update, and it may close the descriptor while
retaining the lock. Thus age, an unchanged file, or a negative open-file probe alone is
insufficient evidence of abandonment.
[Git lockfile API](https://git-scm.com/docs/api-lockfile)

## Implementation

### 1. Put recovery policy in the shared backend

Add a focused `git_lock` domain in `sase-core/crates/sase_core` with explicit serde wire
types, deterministic policy functions, and reasoned outcomes. Bind it in the VCS domain
of `crates/sase_core_py/src/vcs/`, register the bindings, and add binding round-trip
tests. Follow the existing retryability facade pattern, but keep index-lock recovery
separate from network retry classification.

The core owns failure classification, observation-state transitions, eligibility for
removal, and stop/retry/remove decisions. Python may collect filesystem/process facts,
run subprocesses, wait, and render diagnostics; it must not duplicate the core's safety
rules. Keep `src/sase/git_lock_retry.py` as the compatible callback executor/thin
adapter so its many existing callers retain native result objects and the current
attempt-count/lock-path/lock-removed outcome attributes.

Implement the following policy:

- Retry only genuine lock-contention failures. Preserve support for exit 1 and 128,
  bytes and text, and combined stdout/stderr. A success, signal termination, timeout,
  permission-denied error, or incidental mention of `index.lock` is not grounds for
  deletion. Other lock types may retain bounded retries, but are never deleted.
- Keep the existing six-delay backoff schedule and environment override behavior.
  Validate delays as finite and nonnegative, including the SDD override/explicit-delay
  path; NaN and infinity must not produce invalid sleeps or unbounded waits.
- Attempt the command first. Resolve and inspect a lock only after an actual failure.
  Resolve the repository/index using the effective Git context, including the supplied
  working directory, `git -C`, and `.git` files for linked worktrees. Use Git plumbing
  for the index path. Never trust a path copied from stderr as deletion authority; an
  explicit relative or absolute error path must match that independently resolved index
  lock. Unsupported or contradictory `GIT_DIR`/`GIT_INDEX_FILE` context must decline
  deletion, not fall back to a different repository's default index.
- Snapshot the lock using `lstat`: regular file only; include device, inode, size,
  nanosecond mtime and ctime. Canonicalize parent paths consistently while refusing a
  symlink at the lock leaf. A changed path or file identity at any observation
  disqualifies removal for this invocation, even if the replacement has an old mtime.
- Removal requires the unchanged lock to be at least 15 seconds old, exhausted normal
  retries, and affirmative evidence that no relevant owner is active. For a younger
  unchanged owner-free lock, permit a bounded final wait/re-observation until that age,
  so a newly abandoned file can still recover in the same invocation. Cap default
  automatic recovery waiting at 30 seconds and honor any shorter caller deadline. Do not
  repeat backoff schedules or reset the removal budget when a file changes.
- Probe possible owners only on the failure/recovery path. Use a tri-state observation
  (`active`, `absent`, `unknown`) with a reason. Check open descriptors referring to the
  exact lock and live Git operations associated with that effective index/repo,
  including a Git parent waiting on an editor or hook with its lock descriptor closed.
  Associate processes using executable/argv, cwd, relevant Git environment, and parent
  information as needed; account for `-C` and git-dir/worktree layouts. Unrelated Git
  work in another repository must not automatically block recovery. A lock's content is
  index data, not a trustworthy PID record.
- Use bounded host probes: Linux `/proc` facts and a macOS `ps`/`lsof` adapter are
  acceptable; reuse applicable existing process-observation utilities without their
  failure-to-empty behavior. Unavailable tools, permission errors on relevant processes,
  ambiguous repository association, and incomplete inspection yield `unknown`, never
  `absent`. Do not add a global dependency on `psutil` or require elevated permissions.
  Test the observation adapters and policy separately.
- Immediately before unlinking, recheck identity and owner observations. Remove at most
  once, then retry the original command once. If another actor already removed the lock,
  allow that final retry without claiming this caller removed it. A newly
  appearing/replaced lock is not eligible. Keep filesystem/deletion errors secondary to
  the original failed Git result and report why recovery was declined.

The extra age wait is independent of the configurable backoff list: a zero-delay
override is not permission to remove a young file immediately. Inject time and wait
facts for tests instead of weakening the production age/ownership checks. Probe timeouts
consume the same recovery budget, and an exhausted budget forbids deletion.

No portable observation/unlink sequence can eliminate every race with arbitrary external
Git writers. Minimize the interval, test injected races, and document this as
conservative recovery rather than claiming a proof of ownership. Do not introduce a
SASE-only mutex and treat it as exclusion against unrelated Git processes.

Route the two eager age-only cleanup helpers through the same guarded backend, or remove
eager deletion where the subsequent Git command already uses shared recovery. Preserve
existing caller contracts. Do not leave a second route that deletes an old active lock
before the new checks run.

### 2. Make reporter execution use the dev-update command policy

Refactor `src/sase/dev_update/command.py` to support an injected single-attempt
subprocess transport while retaining the default captured/streaming implementations.
Keep timeout selection, environment merging, noninteractive Git settings, error
conversion, Git context resolution, and the retry wrapper in one place.

Have `SessionProcReporter.dev_command_runner()` delegate to that implementation with
`self.run` as its transport. Continue to stream lines, record actual command attempts,
retain bounded output, and publish the final command exit status. Preserve default
300-second command timeouts and explicit longer build timeouts; maintain the existing
`on_output` compatibility behavior without double-delivering lines.

For dev-update commands, establish one monotonic deadline at entry and pass only its
remaining allowance to each attempt/probe/wait. Retries must not multiply the command
timeout. The 30-second recovery-wait cap limits contention handling, not a successful
long-running reconcile/build command. Preserve the final timeout result mapping and
captured diagnostic output.

Add optional recovery-event and cancellation/wait hooks to the executor as needed.
Forward retry delays, recovery/removal, and refusal reasons to the existing reporter
log/progress stream; ordinary logging remains available for other callers. The final
returned stdout/stderr describes the final attempt, while the log retains previous
attempts. Do not replace Git output with recovery messages or emit a terminal failure
for an intermediate attempt that eventually succeeds.

Check cancellation before the initial subprocess, between attempts, during any extra
wait, and before unlinking. A cancelled operation must neither remove a lock nor start
another child. Keep all probing and waiting inside the existing worker body, outside
Textual's event loop and serial message pump. Do not add retry policy to generic
`SessionProcReporter.run()` or nest a second retry wrapper around `execute_dev_update`.

### 3. Add regressions that exercise the failing route

Use deterministic policy/clock/process fixtures for the safety matrix, and real Git
repositories for command integration. Do not use wall-clock sleeps to age a file; set
fixture timestamps. Subprocess ownership tests should signal readiness explicitly and
always reap their children.

- Add a real fast-forward fixture with two commits and a checkout behind its target.
  Plant an old owner-free canonical index lock, call
  `session_reporter().dev_command_runner()` with the actual
  `git -C <repo> merge --ff-only <target>` command, and assert success, HEAD/worktree
  advancement, lock removal, and visible retry/recovery messages. The regression must
  fail on the original reporter implementation. Exercise CLI capture and streaming
  transports against the same scenario.
- Exercise `execute_dev_update` or the comprehensive-update execution seam with a real
  reporter runner and the lock fixture; stub only install/reconcile/network operations
  that would affect the host. Assert a successful final update outcome, continued
  reconcile work, and no stale failed exit status after recovery. Cover the individual
  and combined TUI paths' runner wiring without duplicating whole workflows.
- Cover a transient owner releasing its lock during backoff (retry succeeds without
  deletion), an owner-free fresh lock reaching the age threshold within the bounded
  window, and a long-lived owner (no deletion). Include a real Git process paused in a
  hook/editor with a closed lock descriptor, not just a synthetic open file handle.
- Cover identity/path churn, an old replacement, a symlink leaf and directory leaf,
  deletion failure, disappearance just before cleanup, concurrent replacement before
  unlink, mismatched relative/absolute error paths, linked-worktree gitdirs, and
  ambiguous alternate-index context. Preserve index contents for all refused cases.
- Cover unknown owner probes, missing platform tools, unrelated-repository Git work,
  success/non-lock errors, cancellation during backoff, timeout, disabled removal,
  zero/empty custom schedules, and invalid nonfinite delays. Assert bounded attempts and
  at most one removal/final retry. Tests formerly expecting a freshly planted lock to be
  removed after milliseconds must use an aged orphan or test the new wait path.

Start from `tests/test_git_lock_retry.py`,
`tests/test_git_lock_recovery_integration.py`,
`tests/dev_update/test_execute_command.py`, `tests/dev_update/test_stream_command.py`,
`tests/ace/tui/test_session_proc_reporter.py`, and
`tests/ace/tui/test_plugins_browser_pane_comprehensive_update_execution.py`. Update
existing cleanup regressions in `tests/test_axe_runner_utils.py` and
`tests/agents_sync/test_git_sync.py` for the guarded policy. Split tests by behavior
where useful instead of making already broad files much larger.

### 4. Document and verify the shipped behavior

Update `docs/vcs.md`'s index-lock section to explain shared TUI/CLI recovery, bounded
waiting, owner checks, the minimum age, and why an active/unknown lock is preserved.
Document the existing general and SDD retry environment variables accurately in
`docs/configuration.md`, including finite-delay validation. Note that an already-running
editable TUI must restart to load a landed fix; advancing its checkout does not replace
already-imported Python code.

Build/test the bindings in the isolated development environment. Make
`sase-core-revision.txt` include the new core binding commit: when both repos are in the
final declaration, the host's linked-repo revision-pin finalizer publishes core first
and supplies that pin. Do not invent a SHA, hand-create commits, or ship a Python caller
whose pinned core lacks the binding. Follow `docs/rust_backend.md` and the linked repo's
build instructions; never invoke bare Cargo or reinstall the global tool for testing.

Run focused Rust domain/binding tests and the Python suites above. Read applicable
lint/test and TUI guidance before implementation. Format both repos, then run
`sase tool run check` from each changed repository. Use the SASE monitor workflow for
long final verification commands and wait for its handoff command to exit. Do not run
`just check-full` without a separate explicit instruction. Run targeted visual
maintenance only if rendered UI output or snapshot coverage actually changes.

Acceptance requires the formerly failing reporter-backed fast-forward to recover an
abandoned lock and complete normally, live/unknown/changing locks to survive unchanged,
retry decisions to be visible in the update log, cancellation to remain responsive, and
both repositories' required verification to pass. Report the original orphan's creator
as unknown unless new independent evidence identifies it.
