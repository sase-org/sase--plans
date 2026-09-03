---
tier: tale
title: Stop failing agent runs whose commit landed before a stitch timeout
goal:
  A `sase stitch create` subprocess that is killed at the finalizer's hard timeout no
  longer fails the agent run when the repo's commit already landed, and the hard cap
  itself is raised so slower machines stop hitting it in the first place.
size: medium
proposed_by: bbugyi200.kellys_mbp.j.f0
---

- **AGENTS:**
  - [bbugyi200.kellys_mbp.research.3.cdx](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.kellys_mbp.research.3.cdx/README.md)
  - [bbugyi200.kellys_mbp.research.3.cld](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.kellys_mbp.research.3.cld/README.md)
  - [bbugyi200.kellys_mbp.research.3.final](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.kellys_mbp.research.3.final/README.md)
  - [bbugyi200.kellys_mbp.research.4.cdx](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.kellys_mbp.research.4.cdx/README.md)
  - [bbugyi200.kellys_mbp.research.4.cld](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.kellys_mbp.research.4.cld/README.md)
  - [bbugyi200.kellys_mbp.research.4.final](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.kellys_mbp.research.4.final/README.md)
- **COMMITS:**
  - [4aba47c](https://github.com/sase-org/sase--research/commit/4aba47cde707987998fb776b3e8f57cc08f9b1ec)
    — docs(research): analyze a ,X kill-and-edit for the last launched agent
  - [05aa21f](https://github.com/sase-org/sase--research/commit/05aa21fd939ca983f56e9bad762668bca125b8b3)
    — docs(research): assess last-launch kill-and-edit
  - [643120e](https://github.com/sase-org/sase--research/commit/643120e383b37a7622b08acbc803e2f515fcab29)
    — docs(research): consolidate agents-tab kill-last-launch keymap research
  - [72ac1a6](https://github.com/sase-org/sase--research/commit/72ac1a66b0ca00be2ba9ed96ddcbfaea1b53ca0b)
    — docs(research): recommend bulk project onboarding UX
  - [b48d9fa](https://github.com/sase-org/sase--research/commit/b48d9fa4d6e9ab3713f8f3ce6196edf964788bc7)
    — docs(research): add bulk project add UX research for the Projects tab
  - [e98353f](https://github.com/sase-org/sase--research/commit/e98353fa72759a27df21379b5ed2daae5ae98f13)
    — docs(research): consolidate bulk project add UX research into
    projects_tab_bulk_add

# Stop failing agent runs whose commit landed before a stitch timeout

## Problem

Agent run `20260903143316` (agent `j`, claude/sonnet, implementing the approved
`202609/config_token_cwd_invalidation.md` plan) was reported as **failed** with:

```
WorkflowExecutionError: Step 'main' failed: Error: sase stitch create stitch_timeout for main
```

Yet its commit `3dd10267c fix(config): invalidate config token cache on cwd change` was
created, recorded in the run's `commit_results.json`, and is now the tip of
`origin/master`. The user got a failure notification, a held workspace (#12), and an
error report for a run whose deliverable had actually landed.

## Root cause (confirmed from run artifacts)

The host-owned commit finalizer runs `sase stitch create` as a bounded subprocess with a
hard 600-second cap:

- `HARD_MAX_SUBPROCESS_TIMEOUT_SECONDS = 600.0` in
  `src/sase/finalizers/bounded_subprocess.py:17`; `run_bounded_subprocess` clamps every
  caller to it and SIGKILLs the whole process group at the deadline
  (`bounded_subprocess.py:136-153`, `:175-184`).
- `_run_stitch_argv` (`src/sase/finalizers/commit_repair.py:404`) passes exactly this
  cap for `sase stitch create` (and `--resume` at `commit_repair.py:417`).

Timeline reconstructed from
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/03/20260903143316/`:

1. Stitch started ~14:43:20 (commit-message file timestamp `1788461000`).
2. The repo's before-commit hook `just fix` consumed roughly the first 8 minutes.
3. `create_commit` succeeded at 14:51:20 (`committed_at: 1788461480`, sha `3dd10267c`),
   and the marker was written to the run's `commit_results.json`.
4. Post-commit publication continued (its stdout shows prompt-archive publication was
   deferred after hitting transient `index.lock` contention in the agents mirror, with
   "will retry with agent publication").
5. At 14:53:20 the 600s deadline expired; the subprocess group was SIGKILLed.
6. `dispatch_commit_decisions` (`src/sase/finalizers/commit_dispatch.py:295-314`) saw
   `stitch.timed_out`, raised
   `BuiltinCommitFinalizerError("sase stitch create stitch_timeout for main")` **without
   ever reaching the commit-marker check at `commit_dispatch.py:358`**, and the whole
   run was classified failed. `stitch_timeout` is deliberately not in
   `RETRYABLE_DIAGNOSTIC_CODES` (`src/sase/finalizers/ledger.py:17`), so one timeout is
   terminal.

So the user's "timeout on slower machines" suspicion is correct, with a twist: the
deeper defect is that the timeout branch never checks whether the commit already landed.
On this MacBook, `just fix` alone eats ~80% of the 600s budget; any machine or cache
state slightly slower guarantees a false failure.

## Changes

All work is in this repo's Python finalizer layer. No finalizer wire schema change is
needed: `FinalizerDiagnosticWire.code` is a free string and warning-severity diagnostics
already flow through (see `commit_deferred` in
`src/sase/finalizers/commit_types.py:88-116`). Do not touch `sase-core`.

### 1. Rescue landed commits when stitch is killed at its bounds

In `src/sase/finalizers/commit_dispatch.py`, in both bounds-failure branches —
`dispatch_commit_decisions` (`:295-314`) and `_attempt_post_repair_follow_up`
(`:494-513`) — stop raising unconditionally when
`stitch.timed_out or stitch.stdout_truncated or stitch.stderr_truncated`:

- First compute the repo's newly landed markers exactly like the success path does:
  `new_commit_markers(before_markers, load_commit_results(artifacts))` filtered through
  `marker_matches_repo` (helpers already imported from `commit_repair.py`).
- **No matching marker** → keep today's behavior byte-for-byte: raise
  `BuiltinCommitFinalizerError` with diagnostic code `stitch_timeout` /
  `stitch_output_cap`.
- **Matching marker** → do not raise. Record a warning-severity
  `FinalizerDiagnosticWire` (code `stitch_timeout_after_commit` or
  `stitch_output_cap_after_commit`, message naming the repo and that the commit landed
  before the kill) and fall through to the existing marker-verification / evidence /
  `reconcile_commit_file_hooks` / dirty-tree logic starting at `commit_dispatch.py:358`
  so the normal success invariants still run. If the kill left genuinely uncommitted
  attributable paths, the existing `dirty_after_stitch` failure still fires — that is
  correct.
- Plumb the warning diagnostics out of dispatch: `_CommitDispatchResult` carries
  `attempts`/`evidence` today; extend it (and `execute_commit_finalizer` in
  `src/sase/finalizers/commit.py`, which flips `attempts[0]` to `success` at
  `commit.py:353`) so the warning lands in the final
  `FinalizerInstanceResultWire.diagnostics` while the instance status stays `success`.
  The `InstanceLedger.record()` merge in `src/sase/finalizers/ledger.py` already
  preserves diagnostics.
- Deliberately keep `stitch_timeout` out of `RETRYABLE_DIAGNOSTIC_CODES`: retrying a
  timed-out stitch re-runs the multi-minute `just fix` hook and, when nothing landed,
  would usually just time out again.

### 2. Raise the hard subprocess cap for slower machines

In `src/sase/finalizers/bounded_subprocess.py:17`, raise
`HARD_MAX_SUBPROCESS_TIMEOUT_SECONDS` from `600.0` to `1800.0`.

- Sizing evidence: `just fix` alone took ~8 minutes on `kellys_mbp`; commit, push, and
  publication need several more; slower hosts (e.g. `athena`) and cold uv/cargo caches
  need real headroom. 30 minutes keeps a hard safety net while making the cap an anomaly
  detector instead of a routine ceiling.
- This constant is both the clamp ceiling for configured command/plugin finalizers
  (`executor_command.py`, `executor_plugin.py` — their defaults stay 120s; only explicit
  configs may now ask for more) and the direct timeout for stitch create/resume
  (`commit_repair.py:404-427`, `:417`). No other call sites need edits; verify with a
  grep for `HARD_MAX_SUBPROCESS_TIMEOUT_SECONDS`.
- Considered and rejected for now: a per-machine config knob in
  `src/sase/default_config.yml`. No demonstrated need for per-host tuning once the cap
  is 3x and landed commits are rescued; add the knob later if 1800s is ever hit
  legitimately.

### 3. Terminate gracefully before killing

In `src/sase/finalizers/bounded_subprocess.py`, escalate instead of jumping straight to
SIGKILL when the deadline or output cap is hit: send SIGTERM to the process group, wait
a short grace period (~5s), then SIGKILL whatever remains.

- Motivation from the same run: the SIGKILL path is why a killed stitch can leave
  `.git/index.lock` files behind in shared repos (the observed prompt-archive
  `index.lock` contention). git cleans its locks on SIGTERM.
- Keep `BoundedCompletedProcess.timed_out` semantics unchanged (a TERM-exited process
  after deadline still reports `timed_out=True`), and keep the reader threads draining
  during the grace window so late output is captured within the caps.

### 4. Tests

Follow the fixture style of the existing dispatch tests
(`tests/test_commit_dispatch_protection_guard.py`,
`tests/test_commit_dispatch_conflict_repair_followup.py`):

- Timed-out stitch **with** a matching new marker in `commit_results.json` → dispatch
  does not raise; instance result is `success` with a `stitch_timeout_after_commit`
  warning diagnostic and the marker's evidence.
- Timed-out stitch **without** a marker → `BuiltinCommitFinalizerError` with
  `stitch_timeout`, unchanged.
- Output-cap variant of the rescue (`stitch_output_cap_after_commit`).
- Rescue in the `_attempt_post_repair_follow_up` path.
- `run_bounded_subprocess` escalation: a child that traps SIGTERM and exits promptly is
  reaped within the grace window with `timed_out=True`; a child that ignores SIGTERM is
  SIGKILLed after the grace period. Extend `tests/test_finalizers_execution_ledger.py`'s
  bounded-subprocess coverage (see the `timed_out` assertion near line 386) rather than
  duplicating setup.

## Verification

- Run `just install` first if this workspace's virtualenv is stale, then run
  `just check` (whole-repo lint gates + diff-scoped tests). Hand it to `/sase_monitor`
  if it runs long; use `just check-full` only via a monitor if the scoped selection
  escalates.
- Targeted:
  `tools/run_pytest tests/test_commit_dispatch*.py tests/test_finalizers_execution_ledger.py`.

## Out of scope

- Making `stitch_timeout` retryable.
- A config knob for the cap (revisit only if 1800s is hit legitimately).
- The unrelated `exit code -15` provider failure from run `20260903070859` and the
  transient agents-mirror `index.lock` contention (the deferral machinery already
  retried it; the lock is gone).
