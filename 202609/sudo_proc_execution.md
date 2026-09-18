---
tier: epic
title: Detached proc execution for sase sudo
goal: 'Approving a sudo gate takes over the terminal only long enough to authenticate;
  the reviewed commands then run under a detached, supervised proc so the TUI is usable
  again immediately, and the gate still settles with a validated ledger when execution
  finishes.

  '
phases:
- id: runner
  title: Runner auth-then-spawn mode and handshake wire
  depends_on: []
  size: large
  description: 'runner: teach sase_sudo_runner (sase-core) a detach mode that authenticates
    on the TTY, spawns a detached root executor for the sealed manifest, invalidates
    the sudo timestamp, and prints a started-handshake; add the handshake wire type,
    a --capabilities probe, and sase_core_py bindings.'
- id: cli
  title: Detached sudo answer path and finalize proc
  depends_on:
  - runner
  size: large
  description: 'cli: add an opt-in --detach path to `sase sudo answer` that runs the
    runner in auth-then-spawn mode, records a durable in-flight execution record,
    and submits a supervised finalize proc; add the internal `sase sudo finalize`
    subcommand that waits for the executor, validates the ledger, answers the gate,
    and settles the gate shell.'
- id: tui
  title: TUI handoff returns after authentication
  depends_on:
  - cli
  size: medium
  description: 'tui: make the ACE sudo terminal handoff pass --detach, handle the
    new execution_started payload, toast the background handoff, and surface the executing
    state on the sudo gate and its finalize proc.'
- id: remote
  title: Detached execution for remote sudo targets
  depends_on:
  - cli
  size: medium
  description: 'remote: extend `sase sudo exec` and the SSH relay so a remote target
    authenticates interactively, spawns its own detached executor, and the local finalize
    proc polls for and fetches the remote ledger; gate the path on an additive contract
    capability with a synchronous fallback.'
- id: default
  title: Detach becomes the default answer mode
  depends_on:
  - tui
  - remote
  size: small
  description: 'default: flip `sase sudo answer --run` to detach by default with --no-detach
    keeping the synchronous path, matching the shell-backed `sase gate answer` convention;
    update help text and tests.'
proposed_by: bbugyi200.athena.0ms
create_time: 2026-09-18 08:50:35
status: wip
bead_id: sase-12w
---

- **PROMPT:** [prompts/202609/sudo_proc_execution.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sudo_proc_execution.md)
- **BEAD:** [sase-12w](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12w/README.md)

# Plan: Detached proc execution for sase sudo

## Problem

`sase sudo answer <id> --run` performs sudo authentication **and** full command
execution while attached to the terminal. The ACE TUI approves a sudo gate by suspending
itself (`suspend_for_external_tool`) and running that command synchronously
(`src/sase/ace/tui/actions/agents/_notification_sudo.py`), so a long command such as
`apt-get install` freezes the TUI for its entire runtime. Only the password entry
actually needs the terminal.

## Current architecture (verified)

- **Gate**: `sase sudo request` builds a gate-shell-backed `sudo` gate
  (`src/sase/sudo/gate.py`). The `approve` option declares `requires_tty: true` and
  takes `{command_ids, receipt}` as input. The generic detached gate answer
  (`sase gate answer --detach`, `src/sase/notification_gates/cli_answer.py`) explicitly
  rejects `requires_tty` options, so sudo gates cannot use it as-is.
- **Approve path** (`src/sase/sudo/cli.py` `_approve`): resolves the bundle, seals the
  reviewed command subset (`selected_sudo_manifest`), takes the host-wide auth lease
  (`src/sase/sudo/lease.py`), then synchronously invokes the trusted external runner
  (`src/sase/sudo/runner.py` writes the sealed manifest to a temp file that is deleted
  on return), validates the ledger (`validate_sudo_receipt`), executes the approve
  option via `execute_gate_selection` with the receipt, and settles the gate shell.
- **Runner** (`crates/sase_gateway/src/sudo_runner.rs` in the **sase-core** linked repo;
  installed separately as `sase_sudo_runner`): verifies the manifest SHA-256, then
  `sudo -k` → interactive `sudo -v` on the TTY → per command `sudo -n -v` +
  `sudo -n -u <run_as> -- <argv>` riding the cached timestamp → final `sudo -k`. It
  prints one JSON ledger on stdout. Because the timestamp is TTY-keyed, a detached
  process cannot reuse it — privilege must be handed off while still attached to the
  terminal.
- **Remote targets** (`src/sase/sudo/ssh.py`): stage manifest over SSH, run
  `ssh -t <host> sase sudo exec --manifest … --ledger …` interactively for the full
  duration, then fetch the ledger file.
- **Manifest/ledger wire** contracts live in sase-core (`crates/sase_core/src/ sudo.rs`)
  and reach Python only through `sase_core_rs` bindings (`src/sase/sudo/core.py`). Any
  new wire schema belongs there.
- **Procs**: durable supervised background processes with logs, stop requests,
  reconciliation, and an operation-request sidecar submission contract
  (`src/sase/procs/submission.py`, `ProcSubmitRequest.operation`). A gate shell's
  execution-phase proc keeps `shell_kind: "gate"`.

## Design decisions

1. **Privilege handoff: auth-then-spawn, not timestamp reuse.** In detach mode the
   runner authenticates on the TTY as today (`sudo -k`, `sudo -v`), then — still
   attached to the terminal, so the timestamp is valid — launches one
   `sudo -n`-escalated re-invocation of itself as a **root executor** for the sealed
   manifest. The executor immediately detaches (setsid + stdio to files). The foreground
   runner then invalidates the timestamp (`sudo -k`) and exits with a started-handshake
   JSON. No process ever needs the sudo timestamp after the terminal is released, so
   `timestamp_type=tty` is a non-issue. The executor runs commands directly (setting
   uid/gid for `run_as`) instead of re-invoking `sudo -n` per command.
2. **A supervised finalize proc owns completion.** An unprivileged proc
   (`sase sudo finalize`, internal) waits for the detached executor, streams its output
   into the proc log, validates the ledger, answers the gate with the receipt, and
   settles the gate shell — so the agent's continuation still launches exactly as it
   does today, host-owned. The proc is visible and stoppable from the Procs surfaces.
3. **Cancellation via stop file, not signals.** The executor runs as root, so the
   unprivileged supervisor cannot signal it. The executor polls a `stop` file inside the
   user-owned handoff directory between commands and while a command runs; on stop it
   terminates the current command's process group and records the `cancelled` ledger
   outcome (mirroring the existing SIGINT handling).
4. **No feature flag.** Each phase lands complete for its scope: `--detach` is opt-in
   and fully functional for local gates when the `cli` phase lands (remote targets get a
   clear "not supported yet, run without --detach" error until the `remote` phase). The
   final phase flips only the default; `--no-detach` remains forever as a user choice,
   which per the flag rules is an option, not a flag.
5. **Auth lease covers only the terminal phase.** The lease's purpose is "one
   interactive sudo authentication at a time". It is released once the handshake is
   validated and the finalize proc is submitted. Double-answer protection for a specific
   gate comes from a durable in-flight execution record instead (decision 6).
6. **In-flight execution record.** The detached answer writes an `execution-state` JSON
   record co-located with the gate bundle (like the execution journal), under the
   bundle's `.response.lock`, recording the finalize proc id, executor pid +
   process-identity token, handoff dir, and selected command ids. `sase sudo answer`
   refuses a second approve while a live record exists; a record whose proc and executor
   are both dead is stale and is cleared (with a notice) so the user can
   re-authenticate. `sase sudo show`/`list` and ACE render the executing state from it.

## Handoff contract (pinned so phases agree)

- Python creates a **handoff directory** per attempt, `0700`, user-owned, under the sase
  state tree (e.g. `sase_subdir("sudo") / "exec" / <gate_id>`), containing:
  `manifest.json` (the sealed subset — note the current temp-file delete-on-return
  behavior in `run_sudo_runner` cannot be reused, the manifest must outlive the
  foreground call), later `ledger.json`, `output.log`, and `stop` (created only to
  request cancellation). The directory is deleted by the finalize proc after the gate is
  answered, and best-effort by stale-record cleanup.
- Runner detach mode:
  `sase_sudo_runner --manifest PATH --expected-sha256 SHA --detach-dir DIR`. On success
  it prints one handshake JSON on stdout and exits 0; auth
  failure/cancel/tty-unavailable keep today's ledger-shaped output and exit codes so
  existing error mapping in `src/sase/sudo/cli.py` keeps working.
- Handshake wire (new, in sase-core `sudo.rs`, schema_version 1):
  `kind: "sudo_exec_started"`, `manifest_sha256`, `executor_pid`, `executor_identity`
  (boot-id:start-ticks token compatible with `sase.core.process_identity`),
  `ledger_path`, `log_path`, `started_at`. Exposed to Python as a
  `sudo_validate_handshake` binding via `sase_core_py`.
- `sase_sudo_runner --capabilities` prints
  `{"schema_version":1,"capabilities":["detached_execution"]}`; Python probes it and
  falls back to the synchronous path (with a notice) when the deployed runner predates
  it.
- The ledger schema is unchanged: the executor writes the same `SudoLedgerWire` JSON to
  `ledger.json`, and `validate_sudo_receipt` + `execute_gate_selection` consume it
  exactly as the synchronous path does.

## Phase: Runner auth-then-spawn mode and handshake wire

Work in the **sase-core** linked repo — open it with `sase repo open sase-core -r "…"`
and work only in the printed path.

- `crates/sase_core/src/sudo.rs`: add the `sudo_exec_started` handshake wire type +
  validation and re-export it; bump nothing on manifest/ledger.
- `crates/sase_core_py`: bind `sudo_validate_handshake`.
- `crates/sase_gateway/src/sudo_runner.rs`:
  - `--capabilities` probe flag and `--detach-dir DIR` mode per the pinned contract;
    keep `--help` text complete and scannable.
  - Detach mode: after the existing `-k`/`-v` auth steps, spawn
    `sudo -n -u root -- <self> <internal-exec-args>` for the sealed manifest, wait only
    for a spawn-success sentinel (e.g. the executor writes `ledger.json.pending` or a
    pidfile before detaching, so auth-phase failures surface synchronously), run
    `sudo -k`, print the handshake, exit.
  - Executor (root, internal re-invocation): re-read and re-verify the manifest SHA-256
    (defense against TOCTOU swaps), setsid/double-fork, redirect stdout+stderr to
    `output.log`, run each command directly with `run_as` uid/gid, per-command timeouts,
    `stop`-file polling with process-group kill and `cancelled` outcome, output-tail
    truncation into ledger entries as today, then write `ledger.json` symlink-safely
    (`O_NOFOLLOW`, write-then-rename inside the handoff dir) with ownership that leaves
    it readable by the invoking user.
  - Preserve current behavior exactly when `--detach-dir` is absent.
- Tests: extend the existing fake-sudo fixture pattern in `sudo_runner.rs` (the
  shell-script sudo stub) to cover detach-mode auth failure, spawn failure, handshake
  shape, stop-file cancellation, `run_as` resolution failure, and symlink refusal. Run
  the executor in a non-detaching test mode where needed.

Acceptance: `--capabilities` and `--detach-dir` behave per the pinned contract; all
existing runner tests still pass; new tests cover the detach paths; `sase_core_py`
exposes handshake validation.

## Phase: Detached sudo answer path and finalize proc

Work in the sase repo (Python).

- `src/sase/main/parser_sudo.py`: add `-D/--detach` and `-N/--no-detach` to
  `sase sudo answer` (mutually exclusive; default synchronous for now), and register an
  internal `finalize` subcommand hidden like `exec` (`help=argparse.SUPPRESS`; internal
  args are exempt from the short-alias rule). Keep options alphabetized and help text
  excellent.
- `src/sase/sudo/cli.py` `_approve` with detach requested:
  1. Probe runner capabilities; no `detached_execution` → notice + fall back to the
     synchronous path. Remote target → `GateError` telling the user to run without
     `--detach` (until the `remote` phase).
  2. Under the auth lease: create the handoff dir, write the sealed `manifest.json`,
     check/claim the in-flight execution record, invoke the runner with `--detach-dir`,
     validate the handshake via the new binding.
  3. Submit the finalize proc through `submit_proc_request` using the operation-request
     sidecar (mirror `_submit_detached_answer` and `GATE_ANSWER_DETACH_ORIGIN` in
     `src/sase/notification_gates/cli_answer.py`): argv
     `sase sudo finalize <gate_id> --json`, sidecar payload carrying the handoff dir,
     handshake, command ids, and manifest sha; label like
     `Sudo run: <first argv> (<gate_id>)`; `timeout_seconds` from the summed per-command
     timeouts (today's `_runner_timeout_seconds`); record the proc id in the execution
     record; release the lease.
  4. Emit `{"status": "execution_started", "proc_id": …, "request_id": …}` (JSON and
     human forms); the gate remains pending.
- `sase sudo finalize` (new handler): re-resolve the bundle; wait for the executor using
  pid + identity token (reuse `sase.core.process_identity` liveness helpers) and the
  appearance of `ledger.json`; stream `output.log` into the proc's own stdout so
  `sase proc` log surfaces show live command output; on SIGTERM write the `stop` file,
  wait a bounded grace, and exit as killed. On ledger present: `validate_sudo_receipt` →
  `execute_gate_selection([approve], option_inputs={command_ids, receipt})` →
  `_settle_shell`, then clear the execution record and delete the handoff dir. On
  executor death without a ledger or overall timeout: record the failure in the gate's
  execution journal, clear the execution record, leave the gate pending, and exit
  non-zero so the proc row shows the error.
- Non-terminal ledger outcomes (`auth_failed`, `cancelled`, `tty_unavailable`,
  `runner_error`) keep today's semantics: the gate stays pending
  (`_reject_non_terminal_auth` equivalent at finalize; auth-phase failures already
  surface synchronously in the terminal step).
- `sase sudo answer` on a gate with a live execution record → clear `GateError`
  (`execution_in_progress`, naming the proc id); with a stale record (proc and executor
  both dead) → clear it, notice, proceed.
- `sase sudo show`/`list`: include the executing state and finalize proc id.
- Idempotency: if the gate's `response.json` already exists when finalize runs (e.g.
  after a crash-retry), skip straight to shell settlement, mirroring the existing
  gate-shell reclaim semantics.
- Tests: extend `tests/test_sudo_acceptance.py` (fake runner script that prints a
  handshake and forks a fake executor writing a delayed ledger) plus unit tests for the
  execution record, stale-record recovery, capability fallback, finalize idempotency,
  and the remote-target refusal.

Acceptance: with a new-enough runner, `sase sudo answer <id> --run --detach --json`
returns within the auth phase, a proc supervises execution, and the gate settles with
the same receipt shape as the synchronous path; without `--detach` behavior is
byte-for-byte unchanged.

## Phase: TUI handoff returns after authentication

Read the `tui.md` reference memory before implementing.

- `src/sase/ace/tui/actions/agents/_notification_sudo.py`: pass `--detach` in
  `_sudo_answer_argv`; handle the `execution_started` payload in `_sudo_cli_message`
  ("Sudo authenticated; commands running in background proc <id>"), keep all existing
  failure mappings, and fall back gracefully when the CLI reports the synchronous
  fallback (capability or remote fallback still blocks the terminal — say so in the
  toast).
- Surface the executing state: the sudo notification/gate row should render as running
  (not re-promptable) while the execution record is live, and the finalize proc should
  be findable in the Procs tab under its readable label. Re-opening the sudo request
  while executing shows status instead of a second approve.
- Update the help popup / footer conditional keymaps if any sudo-related bindings change
  (per `src/sase/ace/CLAUDE.md` rules).
- Tests: TUI-level tests for the new payload mapping and the executing-state rendering,
  following existing `_notification_sudo` test patterns.

Acceptance: approving a sudo gate in ACE suspends only for password entry; the TUI is
interactive again while `apt-get`-style commands run; completion (or failure) is
reflected on the gate row and proc row without restarting ACE.

## Phase: Detached execution for remote sudo targets

- `sase sudo exec` (target side, `src/sase/sudo/cli.py` + parser): add a detach mode
  that runs the target's local runner in `--detach-dir` mode under the SSH TTY and
  writes the handshake to a `--handshake PATH` file (stdout is shared with the
  interactive sudo prompt over the pty, so the handshake must travel by file exactly
  like the ledger does today). Extend `sase sudo exec --contract` with an additive
  `capabilities` list including `detached_execution`; `schema_version` stays 1.
- `src/sase/sudo/ssh.py`: probe the contract; without the capability fall back to
  today's synchronous relay (notice). With it: stage the manifest, run the interactive
  `ssh -t … sase sudo exec --detach …` (short, auth-only), fetch and validate the
  handshake file.
- Local finalize proc for remote targets: poll `ssh <host> cat <ledger>` with backoff
  until the ledger parses or the overall timeout lapses, then validate/answer/settle
  exactly like the local flow; clean up the remote handoff files best-effort. Executor
  liveness on the target is checked over SSH (pid + identity from the handshake);
  repeated SSH failures count against the timeout rather than failing immediately.
- Stop request: finalize proc writes the remote `stop` file over SSH.
- Tests: extend the fake `command_runner` patterns in the existing ssh tests for the
  probe, handshake fetch, poll loop, stop, and fallback.

Acceptance: a remote sudo approval holds the terminal only for SSH + password; polling
completes the gate; an old target CLI transparently falls back to the synchronous relay.

## Phase: Detach becomes the default answer mode

- Default `sase sudo answer --run` to detached, mirroring the shell-backed default in
  `sase gate answer` (`_submit_detached_answer` precedent); `--no-detach` keeps the
  synchronous path. Capability/remote fallbacks from earlier phases keep old runners and
  old targets working with a notice.
- Update parser help/examples, `sase sudo` group description, and the TUI argv (drop the
  now-redundant explicit `--detach`).
- Flip/extend tests that assumed the synchronous default; add one test that
  `--no-detach` still runs synchronously end-to-end.

Acceptance: bare `--run` detaches on capable hosts and falls back with a notice
otherwise; `--no-detach` preserves the fully synchronous behavior.

## Risks and mitigations

- **Root executor lifetime**: an orphaned root executor after a finalize-proc crash
  keeps running its sealed manifest only (bounded by per-command timeouts); stale-record
  recovery reports it, and the stop file remains the cancellation channel. Never kill it
  blindly from reconciliation — record and surface instead.
- **Security surface**: the executor re-verifies the manifest SHA-256 as root before
  executing and writes into the user-owned dir symlink-safely; the handshake/ledger
  never carry credential-shaped data (`contains_credential_shape` still guards
  receipts).
- **Runner/CLI skew**: capability probe + synchronous fallback keeps every phase safe to
  land independently of runner deployment timing.
- **Two-repo coordination**: the `runner` phase changes sase-core (wire + bindings +
  binary) and must land and be released/installed before the `default` phase flips; the
  `cli` phase's capability probe makes the ordering safe rather than atomic.
