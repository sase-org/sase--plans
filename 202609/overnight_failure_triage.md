---
tier: tale
title: Fix the two live agent-runner defects from the 2026-09-14 overnight failure sweep
goal:
  Usage-limit drain relaunches carry each stranded agent's own bead association, and a
  sidecar clone whose upstream cannot be resolved no longer wedges every launch on its
  workspace, so the two failure classes still live after the overnight triage cannot
  recur.
size: medium
proposed_by: bbugyi200.athena.0l1
create_time: 2026-09-15 07:33:14
status: wip
---

# Fix The Two Live Agent-Runner Defects From The 2026-09-14 Overnight Failure Sweep

## Incident review

This plan came out of a full triage of the agent failures on athena and apollo from
2026-09-12 through 2026-09-14. Most failure classes are already fixed or were transient
version skew; they are recorded here so nobody re-diagnoses them. Only the two defects
in the Problem section below still need code changes.

Already fixed (verified healthy on 2026-09-15, no action):

- **Telegram tg_inbound storm, athena, Sep 14 13:59-15:29 (~420 errors).** Root cause
  one: the proc supervisor spawned the receiver with the bare console-script name
  `sase_chop_tg_inbound`, which is not on the host services' PATH — fixed by
  sase-telegram `8586f91` (anchor telegram proc executables). Root cause two: the
  failure-notification helper passed `dedup_key` without `plus_one_note`, so every tick
  died with `ValueError: dedup_key requires plus_one_note` before reaching the fixed
  launch path — fixed by sase-telegram `4a7b9f0` (preserve telegram retry
  notifications), released in 0.4.14. The telegram lumberjack now runs with zero errors
  and one live receiver proc. Evidence: digests `digest_20260914_1359*`-`152953` under
  `~/.sase/axe/error_digests/`, plans `202609/telegram_receiver_launch_fix.md` and
  `202609/telegram_receiver_notification_fix.md`.
- **Circular-import crash loop, athena, Sep 12 05:41-15:49.** `run_every`,
  `refresh_docs`, and `orchestrator_restart` chops died on
  `ImportError: cannot import name 'ADMISSION_DIRNAME' ... (circular import)`; the
  `run_every` lumberjack crash-looped. Fixed by sase `e1a2f78395` (break monitor import
  cycle), landed Sep 12 15:40 and deployed.
- **artifact_run_prune overflow, athena, Sep 13 15:13-18:07.** The pruner refused its
  own store with `validation: runs has ~11k entries; maximum is 10000`, so it could
  never prune. Fixed by sase `347e53beab` (route agent artifact run retention pruning
  through Rust owner), landed Sep 14 09:51; the chop now succeeds hourly.

Transient version skew during upgrades (no action):

- **`agent scan wire schema mismatch` waves.** Athena Sep 13 08:23-10:23 saw
  `got 9, expected 8`; apollo Sep 13 09:34 saw the inverse `got 8, expected 9`. Both are
  the install-refresh window around the queue/capacity wire bump (`bb68af0fe5`, landed
  Sep 13 10:13); both stopped on their own once installs refreshed.
- **`invalid queue fields: unknown field 'capacity'`**, athena Sep 12 00:01-04:57
  (`toobig_split[sase]`): same wire-transition skew; the chop succeeds today.
- **`ModuleNotFoundError: sase.continuation_capture._constants`**, athena Sep 13 14:13
  (`tg_outbound`, single occurrence): stale editable-install import mid-checkout-update;
  the module exists and the chop is healthy.

Environmental / by-design / downstream (no code action):

- **Codex usage limit, both machines, Sep 14 ~17:00.** All in-flight Codex runs died at
  once ("Error running LLM provider command (exit code 1)"; the
  `codex_rollout::list: state db returned stale rollout path` stderr lines are
  incidental noise), followed by `llm.usage_limit` notifications: Codex disabled until
  Sep 19. Athena: `@sase-zw.8.7.1`, `@sase-113.1`, `@sase-110.6`, `@0ky.w0`. Apollo:
  `@sase-xe.16.11.7.16.2/.3/.4`, `@sase-10j.land`. The provider-drain machinery (closed
  epic sase-su) relaunched some of this work successfully on Claude — but also exposed
  the live Defect 1 below.
- **`wait_checks: Wait dependency can never self-resolve`** storms and the
  `Gate follow-up failed` notifications on both machines are downstream of the failed
  runs above, not independent bugs.
- **Commit-finalizer second-conflict failures, athena** (`@sase-zw.8.3` 09:48,
  `@sase-10w.3` 11:14): by-design escalation under concurrent landing; both agents'
  stitches landed on master afterwards (`347e53beab`, `d0a849df74`).
- **ENOSPC on apollo, Sep 14 01:48-03:28.** Disk hit 0.1% free (`disk_pressure`
  notifications; `waits`, `hooks` chops died with
  `OSError: [Errno 28] No space left on device`; axe self-healed twice). The disk is
  back to 48% used and the retention/reaper hardening continues under in-progress bead
  sase-zw.8.7. The ENOSPC window is also the likely origin of the broken plans sidecar
  clone behind Defect 2.
- **One-offs, no recurrence:** athena `RunnerSlotAdmissionError` ("live snapshot is
  invalid", Sep 14 09:06, before the capacity fixes finished landing); apollo
  `comment_checks` exit 1 with `[output absent]` (Sep 13 21:40).
- **Plans-clone write-lane wedging** is already tracked as ready bead **sase-10y**
  (large): do not duplicate it here. Related athena launch failures (`@0kn`,
  `@research.1y.image`, `@0ku`, Sep 14 11:40-13:56) were eviction refusals caused by a
  real publish rebase conflict on plan commit `3ee13f86` ("Refresh plan provenance for
  telegram_receiver_launch_fix"); the conflicting sidecars are clean again and the
  eviction protection behaved as designed there.

## Problem

Two defects observed in the sweep are still live in the sase repo and will recur.

### Defect 1: drain relaunches carry the wrong bead association

On apollo, ~10 minutes after the Codex runs died, four relaunch attempts (ace runs
`260914_170116`, `170131`, `170137`, `170146`) all failed at directive processing with:

```
RuntimeError: %id bead association 'sase-xe.16.11.7.16.4' does not match SASE_BEAD_ID='sase-xe.16.11.7.16.3'
RuntimeError: %id bead association 'sase-xe.16.11.7.15'   does not match SASE_BEAD_ID='sase-xe.16.11.7.16.3'
RuntimeError: %id bead association 'sase-xe.16.11.7.16'   does not match SASE_BEAD_ID='sase-xe.16.11.7.16.3'
RuntimeError: %id bead association 'sase-xe.16.11.7.16.2' does not match SASE_BEAD_ID='sase-xe.16.11.7.16.3'
```

Every relaunch carried the same `SASE_BEAD_ID` (`sase-xe.16.11.7.16.3`) while each
prompt's `%id(...)` association was correct for its own bead — so a batch relaunch of
stranded agents is propagating one member's environment to the whole batch. The guard
that fires is `src/sase/axe/run_agent_directives.py:155`; `SASE_BEAD_ID` capture flows
through `src/sase/agent/launch_request_planning.py` (~line 263) and the drain planning
and execution modules `src/sase/agent/provider_drain.py`, `_drain_planning.py`,
`_drain_execute.py` (automatic drain was built by closed epic sase-su). Two of the four
failed relaunches also targeted workspace #0 — the host master checkout — which a
relaunch should never hold.

Related same-day evidence to cover in the same investigation: on athena at 18:38 (ace
run `260914_183841`, `@sase-10w.5.f0.f0`), a follow-up relaunch launched with a model
still pinned to the disabled Codex provider more than an hour after the disable, burned
a workspace hold, and failed with "LLM provider 'codex' is temporarily disabled". A
relaunch created while the target provider is disabled should re-route per drain policy
or be deferred, not launched into certain failure.

### Defect 2: unresolvable sidecar upstream wedges every launch on the workspace

On apollo at 15:51, launch `260914_155129` (`@00`) failed with
`_WorkspaceBeadEvictionRefused`, and the paired `sidecar-protection` notification shows
why:

```
Failed to publish sidecar before workspace cleanup: plans
could not count unpublished sidecar commits: fatal: no such branch: 'master'
```

`_unpushed_sidecar_commit_count` in `src/sase/axe/runner_workspace_sidecar.py`
(~line 167) runs `git rev-list --count @{upstream}..HEAD` and treats _any_ git failure
as "cannot prove there are no unpublished commits", which escalates to an eviction
refusal that fails the launch. When a sidecar clone is broken — here the plans clone's
HEAD pointed at an unborn/missing `master` branch, most plausibly damage from the
same-night ENOSPC window — that state never self-heals: every launch that lands on the
workspace fails the same way until a human intervenes. The protection is right for "real
commits exist"; it is wrong for "the clone is too broken to have publishable commits at
all".

## Implementation

All changes are in the sase primary repo. No sase-core wire/API change is expected: both
defects are Python agent-runner orchestration. Do not touch the memory files or the
plans archive.

### Workstream 1: correct per-agent environment on drain and follow-up relaunches

1. Reproduce the batch bug in a unit test: plan a drain over 3+ stranded agents with
   distinct bead assignments and assert each generated relaunch request carries its own
   bead's `SASE_BEAD_ID` (and any sibling per-agent env captured the same way). Find the
   actual capture point — likely a launch-context or env snapshot computed once and
   shared across the batch instead of per member — and fix it so each relaunch derives
   its environment from its own stranded agent's record.
2. Keep the `run_agent_directives` guard exactly as strict as today: it correctly
   refused to run work against the wrong bead. The fix is upstream of the guard, not a
   relaxation of it.
3. Relaunch targeting: assert (and fix if needed) that a drain/follow-up relaunch never
   selects workspace #0 (the host checkout); it must go through the normal workspace
   allocation path.
4. Disabled-provider relaunches: at relaunch-request construction (drain and follow-up
   shells alike), when the resolved provider is disabled, re-route according to the
   existing drain policy (as sase-su does for stranded work) or defer with a visible
   notification instead of launching a run that can only fail. Add a regression test:
   constructing a relaunch for a model whose provider is disabled does not produce a
   doomed launch.

### Workstream 2: make sidecar-count failures diagnosable and recoverable

In `src/sase/axe/runner_workspace_sidecar.py`:

1. Split `_unpushed_sidecar_commit_count` failures into two cases:
   - _Countable but nonzero_: unchanged — publish, and on publish failure retain the
     recovery ref and refuse eviction exactly as today.
   - _Uncountable because the clone has no resolvable branch/upstream_ (unborn HEAD,
     missing branch, missing upstream config — distinguish via explicit
     `git symbolic-ref -q HEAD` / `git rev-parse --verify` probes rather than parsing
     `@{upstream}` error text): treat the clone as damaged. Quarantine it (rename aside
     under the workspace with a timestamped suffix, mirroring the existing "preserved so
     local-only commits are not lost" convention) and let workspace preparation
     re-create a fresh sidecar clone, so the launch proceeds. Emit one
     `sidecar-protection` notification naming the quarantined path and the git error so
     the damage stays visible and recoverable.
2. Never delete the damaged clone; quarantine only. If quarantine itself fails, keep
   today's refusal behavior (fail closed).
3. Regression tests: an unborn-HEAD sidecar clone quarantines, re-clones, and the launch
   path proceeds; a clone with genuine unpublished commits still refuses eviction after
   a failed publish; quarantine failure falls back to refusal.

## Verification

- `just check` in the sase repo; follow the repo's `lint_and_test` memory obligations
  for every tracked file changed.
- Workstream 1: the new drain-batch test fails against the current code (one shared
  `SASE_BEAD_ID` for the batch) and passes after the fix; the disabled-provider relaunch
  test shows re-route/defer instead of a doomed launch.
- Workstream 2: the new unborn-HEAD test fails against the current code (eviction
  refusal) and passes after the fix (quarantine + fresh clone + successful preparation).

## Out of scope

- Everything in the Incident review marked fixed, transient, environmental, or
  downstream.
- Bead sase-10y (hidden plans-clone write lane) — separate, already triaged as ready.
- The ENOSPC retention hardening — continues under in-progress bead sase-zw.8.7.
- Host recovery actions, which are operator-owned and should not wait for this plan:
  - Athena holds workspaces for failed runs `260914_183841` (#19), `260914_164926`
    (#20), `260914_153557` (#25), `260914_113437` (#15), `260914_162918` (#16); apollo
    holds #11, #12, #17, and #0 for its Sep 14 failures. Inspect and dismiss them in
    ACE. **Before dismissing athena's #20:** its linked sase-core checkout still
    contains `@sase-zw.8.7.1`'s uncommitted in-flight work (modified
    `crates/sase_core/src/lib.rs`, `crates/sase_core/src/managed_tmp.rs`,
    `crates/sase_core_py/src/lib.rs`) — recover or commit it first, then rerun
    `sase bead work sase-zw.8.7` on a non-Codex model.
  - Relaunch apollo's `sase-10j.land` on a non-Codex model (Codex re-enables
    2026-09-19).
