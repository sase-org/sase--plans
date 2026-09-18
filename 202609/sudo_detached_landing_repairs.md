---
tier: epic
title: Complete detached sudo execution after the landing audit
goal:
  Make authenticated detached sudo runs settle without a TTY, prevent duplicate
  execution, and complete the local and remote supervision contracts.
parent_bead: sase-12w
phases:
  - id: runner
    title: Preserve executor ownership and stream command output
    depends_on: []
    description:
      "runner: repair post-spawn failure ownership, provide live bounded output, and
      advertise only supported detach capabilities in sase-core."
    size: large
  - id: completion
    title: Authorize headless completion and protect every answer path
    depends_on:
      - runner
    description:
      "completion: implement durable attempt ownership and narrowly authorized headless
      receipt finalization, including foreground fallback and remote-aware recovery
      state."
    size: large
  - id: remote
    title: Complete SSH transport and integrated detached acceptance
    depends_on:
      - completion
    description:
      "remote: fix SSH script encoding and root liveness, preserve uncertain remote
      attempts, stream remote output, and prove local/remote completion through
      realistic acceptance tests."
    size: large
proposed_by: bbugyi200.athena.sase-12w.land
create_time: 2026-09-18 13:56:19
status: wip
---

- **PROMPT:**
  [prompts/202609/sudo_detached_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sudo_detached_landing_repairs.md)
- **PARENT:**
  [202609/sudo_proc_execution.md](https://github.com/sase-org/sase--plans/blob/main/202609/sudo_proc_execution.md)

# Complete detached sudo execution after the landing audit

This is remaining work from `sase-12w`, whose original plan is
`plan:202609/sudo_proc_execution.md`. Its five phases are closed, but its acceptance
criteria are not met. The `parent_bead` relationship deliberately returns the completed
repair to the interrupted parent landing. Do not recreate the original feature phases or
add parent close-out operations as implementation phases.

## Evidence and existing implementation

Read the parent epic's landing note and audited report
`file:explicit:d53fcb4482373f40e163990f` before implementing. The reviewed Python tree
was `9243c0bdd7`, equal to freshly fetched `origin/master`. Relevant commits are
`af8b7ec140`, `e98ef5b1ce`, `e0f1a8d43a`, and `a26bfc839d`. The Rust runner landed in
`b70e64d`, was released in `56615fb` (v0.34.52), and is unchanged through the inspected
core tree `8d5341a`. Both the installed runner capability and Python handshake binding
were present during the audit.

What already works and should be retained: sealed selected-command manifests, TTY
authentication, detached root workers, handshake validation, the finalize operation
sidecar, local stop files, pending/executing TUI projection, detached default with
`--no-detach`, and old-runner/old-target fallback. Existing focused coverage reports 115
passing sudo, TUI, and completion tests.

The audit nevertheless confirmed:

1. The real headless completion path rejects a valid completed receipt with
   `tty_required`. `detach._run_finalize` calls `answer_ops.apply_approved_receipt`,
   which calls the generic executor. Its `requires_tty` guard runs before receipt
   application. The positive acceptance test forces TTY availability for both
   authentication and finalization, masking the failure.
2. `cli._approve(detach=False)` reaches the foreground runner despite a live detached
   record. This covers `--no-detach` and capability fallback. The detached record is
   checked only inside detached branches, after mode selection.
3. Remote execution records omit the remote host and handoff paths. With the local
   finalize proc dead or unsubmitted, recovery checks the remote PID/token against local
   `/proc` and clears the record. A direct probe confirmed deletion of such a handoff.
   Remote reachability uncertainty must not authorize replay.
4. Remote scripts are passed as separate arguments after `ssh host sh -c`, losing
   quoting when OpenSSH constructs the remote command string. Reproducing the transport
   using `/bin/sh -c ' '.join(argv[2:])` makes manifest staging fail and a live harness
   PID appear dead. Separately, the inner liveness script reports root PID 1 dead for a
   UID-1000 caller because `kill -0` returns EPERM.
5. Rust `run_direct_command` writes captured stdout/stderr to `output.log` only after
   command exit. The remote finalizer does not fetch output.log at all.
6. A root worker can already exist when sentinel handling, final `sudo -k`, or handshake
   transport fails. The Python record may still lack its PID, and cleanup may remove the
   handoff while commands continue. The runner's unconditional capability advertisement
   also does not match its Linux-only `/proc` identity implementation on other
   platforms.

These are completion and integration defects in the promised feature. The SSH staging
bug predates this epic but directly blocks its new remote acceptance.

## Architecture and constraints

Open `sase-core` through `sase repo open`; use only the returned checkout. Shared
attempt-state schemas, transition/authorization policy, and process identity contracts
belong in Rust core with Python bindings. Keep Python process, file, SSH, and Textual
adapters thin. Do not create another Python-only policy engine. Read the relevant CLI,
TUI, flag, and verification memories when touching those domains; do not edit memory
files as part of this repair.

Retain the manifest and ledger version-1 contracts. Additive internal attempt metadata
must have explicit validation and handling for existing in-flight version-1 records. No
credential values enter argv, environment, receipts, logs, or recovery metadata. Keep
authentication on the trusted terminal. Completion authorization must never become a
generic headless approval bypass.

## Phase runner: executor ownership and live output

Work primarily in `sase-core`: `crates/sase_gateway/src/sudo_runner.rs`,
`crates/sase_core/src/sudo.rs`, and the Python binding registration/tests when the
contract needs additions.

- Stream stdout and stderr into the user-readable output.log while each command runs.
  Drain both pipes concurrently and keep only a bounded tail for ledger construction;
  avoid collecting the entire output in memory. Preserve timeout, stop-file
  cancellation, process-group termination, and credential filtering.
- Make startup ownership explicit across root launcher, worker, and foreground runner.
  The existing `started.json` witness must survive once a worker may execute. A
  post-spawn failure must either terminate/reap that worker before reporting that no
  execution started, or retain verifiable identity and handoff evidence for controller
  recovery. In particular, final timestamp invalidation failure must not be represented
  as an ordinary pre-spawn failure while work continues. Do not leave an untracked
  worker when identity/sentinel creation fails in the privileged launcher.
- Keep normal authentication failure ledger-shaped. Document and test how the controller
  distinguishes definitely-not-started from started/uncertain errors; the next phase
  consumes that contract and must not infer death from missing stdout. Preserve
  sealed-manifest revalidation and symlink-safe output writes.
- Advertise detached execution only on platforms where identity and spawning are
  supported. Implement portable identity if an existing core facility supports it;
  otherwise report no capability so the established synchronous fallback is used before
  authentication/spawn on unsupported platforms.
- Extend fake-sudo/nonprivileged worker tests for output visible before command exit,
  bounded tails under large output, cancellation/timeouts with descendant pipes, and
  post-spawn sentinel/cleanup failures. Include capability tests for supported and
  unsupported identity backends.

Acceptance: command progress is visible during execution; every possible worker has
durable recoverable ownership or is proved stopped before failure cleanup; existing
synchronous runner and wire tests still pass. Run core's full required `just check`,
including binding tests. Do not manually edit release versions.

## Phase completion: durable execution ownership and headless settlement

Relevant Python adapters: `src/sase/sudo/{cli,detach,execution,answer_ops}.py`,
`src/sase/notification_gates/executor.py`, and the corresponding Rust gate/sudo policy
and bindings. Integrate with the existing generic decision and settlement contracts
rather than bypassing durable acceptance or writing response.json by hand.

- Add a narrowly scoped internal completed-receipt finalization path. Require the
  matched persisted attempt, operation request, selected IDs/digest, handshake, and
  validated ledger before permitting headless settlement. Normal CLI, generic gate,
  mobile, Telegram, fleet, and arbitrary detached callers must retain their current
  restrictions. A caller-controlled source string alone is insufficient authorization.
  Replays settle the same result exactly once.
- Check and claim execution ownership before selecting detached, foreground, or fallback
  execution. Under the gate's existing locking/decision discipline, reject live attempts
  and already answered/cancelled/conflicting decisions before authentication or command
  execution. Serialize denial/cancellation with this ownership so they cannot settle the
  opposite answer while reviewed commands continue; a running attempt is controlled
  through its stop channel. Cover the generic deny route as well as the sudo CLI.
- Persist attempt identity, controller/auth-start owner, local/remote target and handoff
  paths before the relevant side effects. There must be no gap in which a record with no
  handshake/proc is automatically considered dead while startup is active or uncertain.
  Reconcile the runner's durable started witness when stdout, validation, or proc
  submission fails.
- Distinguish live, definitely dead, and unknown executor state. Never compare a remote
  process identity with local `/proc`; use target-side evidence through a bounded
  adapter. Preserve remote recovery metadata when proc submission fails. Keep TUI
  projection fast: do not run blocking SSH probes during row rendering; project
  cached/durable uncertainty conservatively and verify at recovery time.
- Finalizer failure cleanup must distinguish its own currently-running proc from the
  executor's liveness, and must not delete handoff evidence while the worker may still
  run. Preserve recoverable receipts after settlement errors. Clean both local and
  remote handoffs on successful or idempotent completion once execution is safe to
  retire. Keep failure journaling and pending status honest.
- Add regressions for a genuinely headless finalizer, forged/stale/mismatched attempt
  requests, `--no-detach` and capability fallback during a live attempt, already
  answered/cancelled gates, denial races, lost startup output, proc submission failure,
  remote unknown liveness, and exactly-once continuation.

Acceptance: a real detached finalize process without `/dev/tty` settles a valid
completed attempt, and no other transport gains headless approval. No answer mode
repeats commands while an existing attempt is live or unresolved. Update focused tests
and run required Rust/Python verification for changed repositories.

## Phase remote: SSH transport and integrated acceptance

Work in `src/sase/sudo/ssh.py` and its thin target-side CLI/finalizer adapters, with
shared policy kept in core. Adopt the preceding attempt contract.

- Centralize remote argv encoding: send one correctly shell-quoted command string, or
  use a safely parameterized stdin script protocol where stdin is not already carrying
  the manifest. Apply it to staging, exec, liveness, stop, output reads, and cleanup.
  Staging failure must prevent command launch; check mkdir/chmod/write failures rather
  than accepting only the final shell status.
- Inspect root-owned process existence and boot/start identity without requiring signal
  permission. Permission denied and unreachable hosts are not proof of death. Bound
  every recovery/cleanup SSH operation and apply backoff; transport failures count
  toward the overall deadline without discarding ownership.
- Fetch output incrementally with an offset and bounded chunks into the local finalize
  proc log. Stop requests use the recorded target and handoff; retain them when the
  target is temporarily unreachable. If the interactive SSH or handshake fetch fails
  after a possible spawn, retain/reconcile the remote started witness before cleanup or
  allowing another approval.
- Ensure successful, idempotent, failed, cancelled, and timed-out completion share the
  local validated-ledger/decision contract. Keep old target and old runner fallback
  working, subject to the same duplicate protection.
- Replace argv-only success mocks with a transport test that exercises OpenSSH's actual
  shell-string semantics using an isolated fake endpoint. Test paths with spaces/quotes,
  live root-owned process inspection without signal rights, remote identity differences,
  network loss during and after spawn, live output, stop requests, cleanup retry, and
  target capability skew. No real privileged command or live SSH account is needed for
  these deterministic regressions.
- Extend acceptance to a detached finalizer in a new session with no controlling TTY.
  Assert auth-only terminal occupancy, early output while a command runs, one
  receipt/continuation, pending on nonterminal errors, and preserved recovery state
  under the races above. Keep TUI running/status projection and the default/foreground
  documentation accurate; read TUI memory before UI changes.
- Recheck post-parent history for new consumers and run focused sudo/TUI/parser/
  completion coverage plus the required combined verification. Full verification must
  use `sase_monitor` per the repository memory. Record unrelated failures with exact
  nodes and their existing task, rather than claiming green or expanding this repair
  into unrelated sidecar work.

Acceptance: local and remote default approval return after authentication and finish
through the real headless supervision path; live output, cancellation, duplicate
refusal, fallback, and recovery behave consistently across surfaces.

## Independent follow-up already triaged by the parent land agent

The proposals in `sase-12w.4` note 1 and `sase-12w.5` note 1 describe the same unrelated
atomic sidecar clone regression from `cd7b9f9fd8`, tracked by the single large CI task
`sase-130`. The focused reproduction yields 14 failures and one pass: intentional empty
remotes are rejected before initialization seeds them, plus one stale assertion about
the clone destination. That separate task must remain discoverable to the future land
agent; it is not a phase here.
