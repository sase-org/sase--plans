---
tier: tale
title: Preserve command outcomes through monitor settlement crashes
goal:
  Recover interrupted monitor bookkeeping without losing the recorded command result or
  duplicating execution and continuation.
size: medium
proposed_by: bbugyi200.athena.0ze
create_time: 2026-10-10 09:27:04
status: wip
---

# Preserve command outcomes when monitor settlement crashes

## Problem and evidence

Agent `sase-1jm.2.1.1` was stranded after a successful build. Its monitor `c7a4eg5x09ax`
ran `just rust-install`, completed the extension and LSP installations, and returned
exit code 0. The failure was in SASE's bookkeeping after the command.

The completed evidence is available in:

- Transcript:
  `~/.sase/chats/202610/gh_sase_org__sase-ace_run-sase_1jm_2_1_1__mon-20261010080232.md`.
- Supervisor exception: `~/.sase/procs/runtime/c7a4eg5x09ax/supervisor.log`.
- Proc checkpoint state: `~/.sase/procs/runtime/c7a4eg5x09ax/settlement.json`.
- Monitor artifacts:
  `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202610/10/20261010080232/`. Its
  `continuation/records/monitor_result/` directory contains both results below.

The sequence on 2026-10-10, in UTC:

1. At 12:12:19, the proc entered settlement with exit code 0.
2. At 12:12:21, it persisted `result:c7a4eg5x09ax:c77c7e4a89bd01f1`, with
   `outcome: completed`, exit code 0, and complete retained output.
3. `settle_monitor_artifacts` then crashed in `save_chat_history` → `sharded_path` →
   `get_timezone` → `load_merged_config`. The traceback ends with
   `ImportError: cannot import name 'legacy_xprompt_syntax_enabled'`. The loaded caller
   still used `_normalization_cache_policy`, which the current source no longer has.
   This is consistent with an interpreter mixing code across a live update; commit
   `8a2f344626` removed that policy helper and its import.
4. At 12:12:30, recovery persisted `result:c7a4eg5x09ax:90b74aa2066a1f7a`, with
   `outcome: lost` and exit code 0.
5. The terminal proc became `error / supervisor-loss`; the monitor became `lost` and its
   follow-up became `not-launchable`. The diagnostic incorrectly attributed this to a
   reboot. Recorded and current boot IDs match, and the host boot predates the run.

The defect is directly visible in the current source:

- `src/sase/procs/submission.py::_reconcile_named_proc` passes a new loss outcome even
  when the proc was already settling a known command result.
- `src/sase/procs/supervisor.py::run_supervisor` does the same when resuming a
  `settling` row.
- `src/sase/procs/settlement.py::settle_named_proc` loads the saved checkpoint state and
  unconditionally overwrites `status`, `message`, `termination_reason`, and `exit_code`
  with those fallback arguments. Its final publication also uses the caller's arguments.
- `src/sase/monitor/proc_adapter.py` maps `supervisor-loss` to `lost`, and
  `src/sase/monitor/settlement.py::LOST_FOLLOWUP_ERROR` assumes every loss was a reboot.
- The crash tests in `tests/test_procs_service.py` accept either `success` or `error`
  for successful commands, so this regression satisfies their assertions.

## Intended behavior and scope

A durably observed command result survives failure of transcript writing, artifact
publication, or another settlement step. Recovery finishes the outstanding bookkeeping
and applies the original continuation policy exactly once, without rerunning the
command. For this incident's sequence, the final result must be `success / completed`,
exit 0, with one continuation delivery and the original command-result identity.

This is one bounded repair across sase-core and sase. Shared outcome validation,
precedence, and persistence belong in Rust; Python keeps process control and settlement
side effects. Open sase-core with `sase repo open sase-core` and read its `AGENTS.md`.

Do not reintroduce retired macro flags or compatibility helpers to mask the example
ImportError. Any post-command exception can expose this defect. Immutable runtime
installations and agent auto-restart are separate work; `sase-1j6` already owns the
pre-provider agent restart work. Preserve genuine unknown-outcome handling and explicit
stop behavior. Do not infer success from exit code 0 alone or text in a build log.

## 1. Persist the settlement outcome atomically in sase-core

Extend the existing proc lifecycle under `crates/sase_core/src/procs/`, with the binding
in `crates/sase_core_py/src/procs/`. Add a typed, optional persisted settlement outcome
to the proc record and the begin-settlement request. It must capture the observed
status, termination reason, exit code, message, and command-ended timestamp. Keep it
distinct from the final proc result, which may also contain operation validation and
follow-up information.

`begin_proc_settlement` must freeze that outcome under the same store lock that moves
the proc into `settling`. This closes the current window between the store transition
and the Python sidecar write. A replay returns the frozen outcome and original
timestamp. A later supervisor-loss/reboot fallback cannot replace it. An inconsistent
second authoritative observation must be rejected with a useful diagnostic, not silently
accepted. Retain supervisor ownership checks and make repeated begin calls idempotent.

Support existing rows without this optional field. Provide Rust-owned validation and
selection for an existing settlement sidecar's outcome, tied to its proc and supervisor
identity. A valid persisted pre-crash outcome can be adopted once. Missing, malformed,
or mismatched evidence must retain the genuine-loss fallback; a bare store exit code or
an unrelated result cannot supply the missing outcome. Do not convert already terminal
historical rows during this compatibility path.

Add round-trip coverage for the new field and legacy requests/records. Keep the change
additive and preserve supported old wire versions; synchronize any required schema
version changes and validators across both repositories. New logic goes in a focused
module rather than expanding the already large `procs/store.rs` unnecessarily.

## 2. Resume settlement from the saved outcome

Update the Python proc models and thin store adapter to carry the Rust contract. In
`settlement.py`, select/freeze the effective outcome before performing configuration,
chat, artifact, claim, or follow-up work. Use the returned values throughout checkpoint
execution, `_finalize_operation_result`, result-envelope writing, and `finish_proc`. Do
not retain the recovery caller's fallback values in the final publication.

Route both `_reconcile_named_proc` and the supervisor's `settling` branch through this
same path. Preserve the saved outcome even if recovery occurs after a host reboot:
reboot makes an unobserved command uncertain, but does not invalidate an already durable
observation. Do not signal a stale/recycled child process group when valid evidence
already establishes the command has exited. Keep current cleanup for truly unobserved
commands and keep explicit cancellation of continuation effective.

Serialize settlement replay per proc while updating checkpoint sidecars. Re-read state
after acquiring ownership so concurrent recovery observers do not race through the same
side effects. Reuse the existing continuation reservation/adoption protocol for
delivery; do not invent a second launcher or hold the global proc-store lock across side
effects.

In `monitor/proc_adapter.py`, build the monitor outcome and command timing from the
frozen observation. Reuse an already persisted matching monitor result and its evidence
references when retrying artifact settlement. Do not recalculate command-end time from
recovery time and thereby mint another result/delivery identity. Honor existing
acknowledged-delivery and ambiguous-dispatch handling and host-completion eligibility.

A settlement exception should leave the proc resumable, with the command evidence
intact. Preserve a bounded diagnostic identifying the failed settlement step and the
supervisor-log location separately from the command result. Do not mark a failed step
complete or swallow its exception and report full success. Existing reconciliation can
retry from a fresh interpreter; no hot-import retry loop or new background service is
needed.

## 3. Report loss accurately and document the boundary

Change the unconditional reboot wording in `monitor/settlement.py`. Use a neutral
unknown-outcome explanation unless persisted evidence specifically identifies a reboot;
retain the actual supervisor-loss or settlement exception as a separate diagnostic.
Update the corresponding assertions and `docs/monitors.md` to distinguish a command
whose outcome is unknown from a known command result with incomplete bookkeeping. The
existing proc-backed mapping for genuinely unknown supervisor loss can remain `lost`;
this repair must not broaden automatic continuation of unknown commands.

Document that this fixes active/resumable settlement. The incident's record has already
been terminalized incorrectly. `sase monitor resume` currently rejects `lost`, even with
a checkpoint; do not claim that a bare resume will recover it, hand-edit its markers, or
loosen all lost-monitor eligibility in this change. Report the retained successful
result and unfinished bead work for a deliberate subsequent recovery. No automatic scan,
restart, or mutation of historical monitor runs is part of this plan.

## 4. Regression coverage and verification

Use isolated state directories and injected failures; do not run builds or launch real
model agents for these tests.

- Rust: atomic first observation, identical replay, conflicting observation rejection,
  unchanged command-end timestamp, legacy sidecar adoption, invalid/mismatched evidence,
  and normal ownership enforcement. Bindings must round-trip the optional field.
- Strengthen successful-command cases in `tests/test_procs_service.py` to require
  `success`, exit 0, and reason `success` after every injected settlement crash
  checkpoint, rather than accepting either success or error.
- Add the incident regression to proc-backed monitor tests: raise the observed
  `ImportError` once from `save_chat_history`, after the completed monitor result is
  persisted. Recover through real proc reconciliation with the failure removed. Assert
  completed state in the proc, metadata, done marker, and continuation result; unchanged
  command result ID/end time; one command invocation; and one acknowledged follow-up.
  Repeat reconciliation and prove no additional delivery occurs.
- Exercise the supervisor's resumed-settlement branch as well as reconciliation. Cover
  recovery after nonzero command exit, total/idle timeout, and explicit stop: their
  original reasons remain intact and stopped continuation stays suppressed.
- Test a crash between the Rust begin transition and the first sidecar write, two
  recovery observers, and an already acknowledged or uncertain continuation delivery.
  Include typed operations so missing/invalid required results still fail; a successful
  subprocess must not bypass `_finalize_operation_result`.
- Negative cases: absent/corrupt evidence and a genuine pre-reboot unobserved command
  remain unknown; no follow-up is manufactured. Verify the same-boot diagnostic does not
  claim a reboot. A saved known outcome survives reboot without changing its facts.

Run focused tests during implementation, then `sase tool run check` in each changed
repository. Follow `lint_and_test.md` via `sase memory read` in sase. Rebuild the local
binding when needed; use `/sase_monitor` for a long `just rust-install`, and join any
escalated ToolRun instead of rerunning it. Do not run `just check-full`.

Declare both changed repositories through `/sase_final`; the host commits sase-core
first and updates `sase-core-revision.txt` for the primary commit, as documented in
`docs/rust_backend.md`. Do not change package release versions or the published
compatibility window for this repair.
