---
tier: tale
title: Reliable scheduled task-status hooks on the Mac
goal:
  Recover safely from transient vault contention, retain cron diagnostics in logs, and
  stagger and verify the Mac's Bob maintenance jobs.
size: medium
proposed_by: bbugyi200.athena.0kj
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.research.1w.cdx](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1w.cdx/README.md)
  - [bbugyi200.athena.research.1w.cld](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1w.cld/README.md)
- **COMMITS:**
  - [fd31339](https://github.com/sase-org/sase--research/commit/fd31339f7d24160ee80d2f3f0fdb35b5beba2c12)
    — docs(research): add researcher B report on agent-requested sudo password prompts
  - [a42c51a](https://github.com/sase-org/sase--research/commit/a42c51a12656177c2bda737ec1bf4ea86583cac6)
    — docs(research): analyze secure sudo agent requests

# Reliable scheduled task-status hooks on the Mac

## Outcome and scope

Make `bob task-status-hooks` recover automatically from temporary maintenance lock
contention and concurrent vault saves, with bounded jittered backoff. Record retry
diagnostics and scheduled-command output in the Mac's existing log files, and stagger
the three Bob cron jobs to reduce simultaneous writes. Implement, test, document, and
deploy this as one bounded coding task. This is a `tale` of size `medium`: one
command-level retry controller, its CLI/tests, and a targeted update to an existing
crontab. Separate implementation agents or an epic are unnecessary.

The user requested this plan before implementation. Authoring and validating this
scratch plan are the only file changes in the planning turn. Implementation and live
configuration changes begin after this plan's approval handoff.

## Verified evidence

Read-only inspection on 2026-09-14 established:

- `ssh mac 'crontab -l'` showed these three jobs all using `*/15 * * * *`:
  `~/bin/maybe_bob_highlights_sync -w`, `~/.cargo/bin/bob projects sync`, and
  `~/.cargo/bin/bob task-status-hooks`. Each redirects stdout with `>>` to its own
  `/var/tmp` log, but leaves stderr for cron to mail.
- The deployed `com.bbugyi.bob-vault-sync` LaunchAgent runs
  `/Users/bbugyi/.cargo/bin/bob vault-sync -q` at load and every 15 seconds. Its plist
  matches the managed source in the linked `chezmoi` repository at
  `home/Library/LaunchAgents/com.bbugyi.bob-vault-sync.plist`.
- The Mac's latest inspected sync completed successfully in 950 ms; the LaunchAgent's
  last exit status was 0. This is one observation, not a measured upper bound on sync
  duration or proof of which process held each historical lock. The quoted error proves
  maintenance-lock contention.
- `src/native/task_status_hooks.rs::run` calls `sync_task_statuses` once. The latter
  acquires the shared lock before reading/planning, retains it through application, and
  returns immediately when it is busy.
- `src/native/ob.rs::try_acquire_lock` uses an advisory exclusive lock at
  `BOB_VAULT_SYNC_LOCK_FILE`, or `bob_sync.lock` under `XDG_RUNTIME_DIR` when valid,
  otherwise `/tmp`. `vault-sync`, `nightly`, and live status hooks coordinate through
  it. Ordinary editor saves do not take this lock.
- `task_status_hooks_write.rs` separately detects changed/unstable inputs and enforces a
  two-second quiet interval for structural regrouping. It stages guarded writes and
  preserves recovery records. `SyncError` already exposes reason, applied/deferred
  paths, and recovery directory.
- The visible status-hooks log contained successful no-ops; failed attempts were absent
  because their stderr was mailed instead. All three jobs still need their diagnostics
  retained when run unattended.

Relevant files: `src/native/task_status_hooks.rs`,
`src/native/task_status_hooks_write.rs`, `src/native/ob.rs`, `tests/cli.rs`,
`README.md`, `docs/task-status-hooks.md`, and `docs/vault-git-sync.md`. `justfile`
defines the normal format, lint, and test checks.

## 1. Add bounded retries around complete attempts

Keep `sync_task_statuses` as a single complete attempt. Add the retry controller above
it, so returning from an attempt drops its maintenance-lock guard and all snapshots
before any backoff. Every next attempt must reacquire the same lock, rediscover inputs,
select the effective daily sources, and recompute the whole plan. Never reuse staged
output or a stale write plan after a conflict.

Add `-r, --retry-timeout SECONDS`, an integer with default `120`. Zero requests one
immediate attempt and preserves the old fail-fast behavior. Explain this in
alphabetically ordered help with the short alias, examples, and docs. Validate
malformed, negative, and overflowing values without panic. Use overflow-safe
elapsed-duration arithmetic rather than unchecked deadline addition. Existing hidden
command aliases must share the same behavior.

Use a monotonic elapsed budget beginning before the first attempt. On eligible failure,
use exponential delay ceilings of 2, 4, 8, 16, then 30 seconds, capped at 30 thereafter,
with a random delay in the upper half of each ceiling. Use subsecond resolution so
retries do not synchronize with the 15-second sync timer. Production jitter must vary
between invocations; reuse an existing appropriate primitive or a small local
implementation, avoiding a new large dependency for this purpose. Inject clock, sleep,
jitter, and attempt behavior through a small private testing seam.

Clamp any sleep to the remaining budget and check the elapsed budget again before
starting another attempt. Never start an additional attempt once the budget is
exhausted. The initial attempt always runs; allow an active attempt to finish safely
even if it crosses the deadline. This is a bound on retries, not a forced timeout that
can interrupt a note replacement. State that limit clearly. Backoff must never hold the
shared maintenance lock.

Use an explicit allowlist of existing error reason codes, never error-message matching
or a blanket retry on exit code 1:

| Failure                                                                                 | Automatic action                  |
| --------------------------------------------------------------------------------------- | --------------------------------- |
| `lock_contention`                                                                       | Retry within the budget.          |
| `vault_changed`, `quiet_period`, `unstable_read`                                        | Retry only with no applied files. |
| `partial_apply`, or any error listing applied files                                     | Stop and retain recovery details. |
| `recovery_failed`, `unsupported_file`, `io`, validation errors, unknown/missing reasons | Stop immediately.                 |

Keep partial-apply failures terminal even when their applied-files list is empty: the
existing reason can combine different failure stages, and this change must not broaden
replay safety by guessing. Update the documentation's current statement that a partial
apply is "retryable" to distinguish manual inspection/rerun from this automatic retry
policy. Preserve all existing snapshot, quiet-period, staging, and recovery safeguards
and final exit codes.

Dry-run always uses one attempt and never enters the retry controller's sleep/logging
behavior, even when a timeout is supplied. It must still create no lock, recovery
records, staged files, or modified notes.

## 2. Make retry output useful and safe for automation

Emit one concise timestamped stderr event per retry decision with a run identifier,
attempt number, elapsed time, reason, error detail, and next delay. Include any recovery
directory from the failed attempt so a later successful retry does not hide preserved
evidence. After retries, emit a timestamped success, exhausted-budget, or
terminal-failure summary with attempt count and elapsed time. Do not claim that an
exhausted retry budget succeeded.

Render the normal final result/error only once. In JSON mode stdout must remain exactly
one final JSON object with the existing success/error fields and exit status semantics;
intermediate diagnostics belong on stderr. Preserve normal warning output. Uncontended
runs should retain their existing concise output.

The scheduler owns file redirection: use `>> logfile 2>&1`, in that order, to retain
both streams. A new native log-file subsystem or blanket `MAILTO=""` is unnecessary.
Interactive users may still see diagnostics on stderr. The cron invocation must capture
retries, successful completion, warnings, and terminal errors in its file, including
child-command diagnostics.

## 3. Apply a targeted Mac cron change after deployment

Keep each job's 15-minute frequency and separate their start times:

```cron
0,15,30,45 * * * * ~/bin/maybe_bob_highlights_sync -w >> /var/tmp/maybe_bob_highlights_sync.log 2>&1
5,20,35,50 * * * * ~/.cargo/bin/bob projects sync >> /var/tmp/bob_projects.log 2>&1
10,25,40,55 * * * * ~/.cargo/bin/bob task-status-hooks --retry-timeout 120 >> /var/tmp/bob_task_status_hooks.log 2>&1
```

Highlights intake precedes project reconciliation, which precedes task status
reconciliation. The two-minute retry budget fits comfortably within the 15-minute task
cadence under ordinary run durations. Offsets reduce cron job collisions; they cannot
guarantee exclusion from editors, slow jobs, or the 15-second sync LaunchAgent.
Correctness comes from the existing lock/guards and fresh-attempt retry behavior.

Retain the current healthy 15-second vault-sync interval. Offsetting cron by minutes
cannot avoid that timer, and slowing it would increase cross-machine sync latency
without solving save races. There is no need to migrate these jobs to launchd or
introduce a second scheduler/retry wrapper in this change. The source inspection found
no managed crontab for these entries in chezmoi; persist their operating instructions
and exact schedule in the bob-cli runbook. No chezmoi source modification is expected.

Immediately before installation, reread the live crontab. Save a timestamped backup
outside the vault and prepare a candidate replacing only the matching three Bob entries.
Preserve unrelated entries, comments, and environment lines. Review the candidate diff;
if current contents differ from the inspected baseline, merge the relevant changes
rather than overwrite them. Verify log paths are writable by the cron user. Install the
candidate with `crontab` only after the Mac binary recognizes the new option, then read
back the installed table and confirm exactly one entry per intended job.

## 4. Tests and documentation

Add focused tests that establish retry correctness and protect the write boundary, with
simulated time/jitter for long budgets:

- Contention followed by success; exhausted contention retains the final reason and
  failure exit status; zero timeout performs exactly one attempt.
- Backoff increases to its cap, stays in its jitter range, respects the remaining
  budget, and never starts a retry after expiry or spins with zero delay.
- Each allowed transient reason retries; permanent/unknown errors and every
  partial-apply case stop immediately, with recovery information preserved.
- After an actual guarded-write conflict, a successful next attempt observes an
  intervening edit and replans from fresh bytes. Use a controlled fixture and existing
  writer callbacks or a narrow injected seam, not timing races against the real vault.
  Confirm the lock is available to another holder during backoff and the editor's
  changes are preserved.
- A real CLI test holds an isolated lock, waits for the retry diagnostic, releases it,
  and verifies eventual success. Bound the test duration, clean up the child on failure,
  and do not use fixed arbitrary sleep races.
- JSON retries produce one parseable final object and useful stderr diagnostics;
  exhaustion and terminal errors retain the existing structured error fields.
- Dry-run remains immediate and creates no lock/recovery/staging artifacts. Retain the
  existing quiet-period, partial-apply, guarded-write, and idempotence coverage. Change
  the existing held-lock fail-fast CLI test to pass `-r 0` so it does not wait for the
  new production default.
- CLI help lists the new option alphabetically with its short alias; canonical and
  compatibility command names agree; invalid timeout values fail clearly.
- Exercise the planned command through `/bin/sh` using isolated test vault, lock, and
  temporary log paths with cron-like `PATH=/usr/bin:/bin`. Induce a retry and a terminal
  failure separately; assert both parent output streams are empty, the log retains
  diagnostics/final result, and exit status survives redirection. Do not change the real
  vault or use its maintenance lock here.

Run focused Rust tests first, then the repository's required `just all` checks (or the
equivalent commands from `justfile`). Use `/sase_monitor` for long commands per the
environment workflow. Existing fixture tests isolate state and lock paths; new tests
must do the same.

Update README usage and `docs/task-status-hooks.md` with defaults, backoff, eligible
reasons, timeout semantics, dry-run behavior, JSON output, recovery limitations, and the
cron redirection example. Update `docs/vault-git-sync.md` with the observed scheduling
issue, exact staggered crontab, retained sync cadence, log inspection commands,
deployment verification, and rollback.

## 5. Install and verify on the Mac

Use the audited memory skill to reread `cli_rules.md`, `obsidian.md`, and `tailnet.md`
before work in those domains. All external repository access must go through
`/sase_repo`; the linked chezmoi repo was inspected read-only during planning. Do not
directly edit managed installed files or an unrelated Mac source checkout. Source edits
belong in the implementing workspace.

After tests pass, deploy the exact verified bob-cli source to the Mac using an isolated
staged export/package of this checkout and build/install there with locked dependencies.
Follow repository deployment conventions. Do not copy a Linux executable to macOS or
install an unrelated remote HEAD. Preserve the previous installed binaries needed for
rollback and record the source identity of the deployed build. Test the Mac build on an
isolated fixture and verify `~/.cargo/bin/bob task-status-hooks --help` exposes the
retry option before installing the new crontab.

Run an installed-binary fixture check on the Mac with controlled lock contention and
both streams redirected to a temporary log; verify eventual success and no shell output.
Then perform one authorized live status reconciliation using the exact production
command/redirection and inspect its log, exit status, and
`bob vault-sync status --json`. Do not intentionally hold the real vault lock or
manufacture real note conflicts for testing. Confirm the sync LaunchAgent remains
healthy. Inspect the next scheduled run's log when practical through the monitor
workflow, and distinguish an observed cron run from a manual smoke test in the
completion report. A single successful run is not a reliability percentage or a
guarantee against permanently busy/broken vaults.

If the Mac is temporarily unreachable, complete code/tests/docs and retain the concrete
deployment and crontab candidate; clearly report deployment as pending and resume when
access is available instead of claiming the machine was fixed. Do not make the new
crontab depend on an option that is not installed yet.

Rollback order: restore the previous matching cron entries from the saved backup while
preserving any intervening unrelated edits, then restore the previous binaries if
necessary. Keep logs and recovery records. No vault reset, forced synchronization, or
automatic recovery-copy restoration is part of this change.

## Acceptance

Temporary lock contention or a safe pre-apply vault conflict can settle within the
budget and lead to a successful fresh attempt without dropping edits. Persistent
failures stop within the retry policy, retain useful diagnostics, and remain failures.
JSON and dry-run contracts hold. The Mac runs the verified binary with the staggered
schedule, and ordinary retry/error output is retained in its log files without becoming
cron email. Report tests, deployment status, installed schedule, log locations, and any
unobserved scheduled-run verification.
