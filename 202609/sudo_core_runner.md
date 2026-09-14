---
tier: tale
title: Sudo manifest contracts and TTY-attached runner in sase-core
goal:
  Ship hash-bound sudo wires, an unprivileged TTY runner, Python/wheel plumbing, and
  deny-only remote approval enforcement for sase-110.1.
size: medium
proposed_by: bbugyi200.athena.sase-110.1
bead: sase-110.1
status: done
---

- **PARENT:**
  [202609/agent_sudo_requests.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_sudo_requests.md)
- **BEAD:**
  [sase-110.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-110/sase-110.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-110.1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-110.1.md)
- **COMMITS:**
  - [ab68522](https://github.com/sase-org/sase-core/commit/ab68522ac465d11544d48d6881ad1ea0c9f372d3)
    — feat(sudo): add reviewed sudo runner contracts

# Sudo Manifest Contracts And TTY-Attached Runner In `sase-core`

## Goal

Implement phase `sase-110.1` entirely in the `sase-core` repository:

1. Add strict, versioned Rust wire contracts for reviewed sudo manifests, execution
   ledgers, and deterministic per-command risk badges.
2. Ship an unprivileged `sase_sudo_runner` binary and wheel console script that binds
   execution to the reviewed manifest hash, gives `/usr/bin/sudo`/PAM the real
   controlling TTY, runs the approved commands one at a time, and emits a bounded JSON
   ledger without ever collecting or persisting authentication material.
3. Expose manifest validation/hashing, ledger validation, and risk derivation through
   `sase_core_rs` so the Python CLI/TUI phase consumes the same policy.
4. Make `requires_tty` gate options deny-only over fleet attention: a remote attempt to
   choose the privileged branch must settle as `capability_missing`, while display and
   non-TTY branches such as deny keep their current behavior.

Do not change release-plz-owned workspace/crate versions or path-dependency pins. Do not
edit the Python `sase` repository in this phase; the later `core-pin` phase owns
updating its Rust revision and dependency floor.

## Security And Behavioral Invariants

- SASE never accepts a password, passphrase, PAM response, attempt count, or a digest,
  length, or timing derivative of one. The manifest, runner argv/environment, ledger,
  stdout/stderr capture, tests, and error messages contain no credential-shaped field.
- Authentication is a terminal handoff. The runner invokes `/usr/bin/sudo` without `-S`,
  `-A`, `SUDO_ASKPASS`, or `-E`; sudo/PAM alone converses with the controlling TTY.
  `--help` is usable without a TTY, but every execution path refuses before touching
  sudo when no controlling TTY is present.
- The runner is never privileged, setuid, or installed under a root-owned libexec path.
  Sudoers evaluates every real approved command because the runner invokes sudo
  separately for each command.
- Approval is bound to bytes: read the bounded manifest once, compare the SHA-256 of its
  canonical normalized form with a required expected digest, then execute that same
  parsed value. A digest mismatch, malformed manifest, or unsupported schema runs
  nothing.
- Invalidate a pre-existing timestamp with `/usr/bin/sudo -k`, authenticate with
  `/usr/bin/sudo -v`, prefer per-command
  `/usr/bin/sudo -n -u <run_as> -D <cwd> -- <argv...>`, and always run
  `/usr/bin/sudo -k` again on success, failure, cancellation, or timeout. Never use `-K`
  or leave intentional warm credentials.
- Probe cached authentication before executing a command rather than retrying a nonzero
  command: a real command must never run twice merely because its exit status could be
  confused with `sudo -n` authentication refusal. When policy has no usable timestamp
  (for example `timestamp_timeout=0`), select the plain per-command sudo path before
  launching that command so sudo can prompt on the same TTY.
- Privileged command stdin is `/dev/null`. Commands run in manifest order, in separate
  process groups, with per-command timeouts. Default `stop_on_failure=true` marks the
  remaining entries skipped; already-completed mutations are never rolled back or
  retried automatically.
- Output shown to the operator may stream to the runner's diagnostic stream, but JSON
  ledger capture is bounded by the reviewed `none|tail|full` policy and a hard maximum.
  The sudo/PAM conversation is inherited on the TTY and is never redirected into the
  ledger stream.
- Linux applies `PR_SET_DUMPABLE(0)` and disables core dumps before sudo is invoked.
  Other supported Unix targets retain the no-secret architecture and compile cleanly;
  non-Unix targets keep `--help` available and return a typed unsupported-platform error
  for execution.

## Wire Contracts

Add a focused `sudo` module under `crates/sase_core/src/` and re-export its public
surface from `crates/sase_core/src/lib.rs`. Follow existing wire conventions:
`schema_version: 1`, `serde(deny_unknown_fields)`, closed enums, structured error codes,
bounded strings/collections, and validation before normalization or hashing.

### Manifest

The normalized `SudoManifestWire` contains:

- `schema_version`, `request_id`, and `host`;
- an explicit `host_is_remote` boolean used only for deterministic lockout-risk
  classification;
- `run_as`, an absolute `cwd`, a sorted map of reviewed `env` additions,
  `stop_on_failure`, and `output_to_agent: none|tail|full`;
- ordered `commands`, each with unique `id`, non-empty exact `argv`, a bounded `why`,
  optional positive bounded `timeout_seconds`, and `shell` defaulting to false;
- optional `resume_from`, expressed as a command ID that must exist in `commands`.

Validation rejects empty IDs/argv values, duplicate command IDs, relative working
directories, oversized manifests/collections/strings, non-finite or non-positive
durations, unsafe environment names (`PATH`, `LD_*`, `SUDO_*`, and malformed names),
leading nested-elevation programs (`sudo`, `doas`, `su`, `pkexec`), and interactive
first-version exclusions. Shell execution is allowed only when `shell=true` and the
reviewed argv has the explicit shell/program form expected by the caller; the runner
does not infer or concatenate shell strings.

Provide pure entry points to:

- parse and validate a JSON value into its normalized wire;
- serialize the normalized wire as canonical JSON (recursively sorted object keys,
  stable defaults, command/argv order preserved);
- compute the lowercase SHA-256 used by the bundle's existing `hashes` envelope.

Tests must prove that object/map insertion order does not affect canonical bytes or the
digest, while every execution-relevant field and command ordering does. Do not include
provenance-only runtime facts or authentication state in the manifest hash.

### Risk badges

Define a closed badge-kind enum with the plan's exact serialized names: `shell`,
`network`, `package-manager`, `system-path-write`, and `service-restart`. Return
deterministic per-command assessments containing the command ID, badge list in the fixed
order above, and `lockout_prone=true` only for remote-host restarts/reloads of SSH or
Tailscale services.

Keep derivation conservative and pure over the normalized manifest. Centralize the
executable-basename and argv heuristics for network tools, common package managers,
write-oriented commands targeting `/etc`, `/usr`, `/var`, `/opt`, `/boot`, `/sys`,
`/proc`, or root's home, and `systemctl`/`service` restart-like actions. Package-manager
commands also receive `network`; `shell=true` always receives `shell`. Add table-driven
tests for positives, near-miss negatives, stable ordering, duplicate suppression, and
the remote-only lockout flag.

### Ledger

The `SudoLedgerWire` binds `request_id` and `manifest_sha256` to:

- `outcome: completed|auth_failed|cancelled|tty_unavailable|runner_error`;
- exactly one ordered entry per manifest command, with command `id`,
  `status: ran|failed|skipped`, optional `exit_code`, non-negative finite
  `duration_seconds`, and bounded `output_tail`;
- an optional bounded host-generated diagnostic for runner failures only, never raw
  sudo/PAM output.

Successful exit zero maps to `ran`; an executed nonzero command maps to `failed`;
commands not attempted because of `resume_from`, stop-on-failure, authentication loss,
cancellation, or runner failure map to `skipped` with no fabricated exit code. The batch
outcome remains `completed` when execution reached a normal command result; the
per-command statuses describe command failure. Add validation that ledger IDs/order,
status/exit-code combinations, bounds, and manifest hash are internally consistent, plus
JSON round-trip tests.

## Runner Implementation

### Library and binary surface

Add `crates/sase_gateway/src/sudo_runner.rs`, export its public CLI/library entry points
from `crates/sase_gateway/src/lib.rs`, and add a thin
`crates/sase_gateway/src/sudo_runner_main.rs` plus a `[[bin]]` entry named
`sase_sudo_runner`. Mirror the federation-worker split: argument parsing and all logic
live in the library module; `main` only prints a safe error and chooses the documented
exit code.

The CLI accepts one manifest source (`--manifest|-m PATH`, with an fd form on Unix if it
can be implemented without weakening validation) and a required
`--expected-sha256|-e HASH`. `--help|-h` documents the invocation and exit statuses.
Emit exactly one serialized ledger on stdout; reserve stderr/the inherited terminal for
the trusted banner, progress, and streamed command output so machine-readable output is
never corrupted. Use distinct process exit statuses for normal completion,
authentication failure, cancellation, TTY absence, invalid input/hash, and internal
runner failure.

### Execution engine

- Read and bound the manifest before invoking sudo; normalize, hash, constant-time
  compare the expected lowercase digest, and retain the parsed value used for execution.
- Confirm a controlling TTY with Unix `isatty`/`/dev/tty` behavior. Harden the process
  before authentication (`PR_SET_DUMPABLE(0)` on Linux and `RLIMIT_CORE=0` where
  available).
- Call the absolute `/usr/bin/sudo` executable with an empty environment plus only the
  fixed locale/terminal baseline and reviewed environment additions. Reject forbidden
  environment keys in core validation; never honor a PATH-based executable lookup.
- Pre-invalidate, run the interactive `-v`, then execute only the selected range from
  `resume_from`. Before each command, use a noninteractive validation probe to choose
  between the `-n` fast path and a plain sudo invocation on the same TTY; never rerun a
  command based on its own nonzero exit status.
- Keep root-command stdin null. Capture stdout/stderr concurrently to avoid deadlock,
  stream safely to the operator, and maintain the bounded ledger view without splitting
  UTF-8. Start each command in its own process group; on timeout or cancellation,
  terminate the group, escalate to kill after a short grace period, reap it, and record
  an honest failed/skipped ledger.
- Install cancellation handling that forwards termination to the active command group,
  then reaches the timestamp cleanup guard. A cleanup failure may upgrade a successful
  run to `runner_error` but must not erase already-recorded command results.
- Factor process spawning, clock/sleep, TTY detection, and cancellation behind a small
  internal test seam. Production always supplies `/usr/bin/sudo`; tests supply a fake
  executable and must not gain a public/environment-variable escape hatch that could
  replace sudo in normal execution.

Behavior tests use an executable fake-sudo fixture that records argv and supports
initial auth failure, noninteractive timestamp-probe failure, command exit failure,
sleep/timeout, and cleanup observation. Cover exact call order, absolute production path
selection, pre/post `-k`, `-v`, `-n`, `-u`, `-D`, `--`, environment clearing,
`resume_from`, stop/continue behavior, timeout process-group cleanup, output bounds, TTY
refusal, digest mismatch running nothing, and cancellation. No test invokes real sudo or
contains a credential-like canary.

## Fleet Attention Hardening

Carry the generic `requires_tty` option bit through the existing mobile/fleet wire path:

1. Extend `GateOptionWire` and `FleetAttentionOptionWire` with a default-false
   `requires_tty` field, project it from the gate envelope, and include it in attention
   revision hashing so changing the privilege boundary invalidates stale reviews.
2. Recognize the canonical sudo notification action (`SudoRequest`) as a gate-shaped,
   priority action and project its detail through the existing custom-gate branch
   structure. This lets the simultaneously developed Python gate phase publish its
   options without requiring another Rust change.
3. In `fleet_attention_resolve`, derive capabilities from both the current pending row
   and the selected option IDs. Do not advertise `attention.approve_gate` when any
   selected option is marked `requires_tty`; the existing precondition then returns the
   required structured `capability_missing` receipt before `execute_gate_action` can
   run. A deny/request-changes branch without `requires_tty` retains the existing gate
   capability and remains remotely answerable.
4. Keep unknown-option validation authoritative and fail closed if option metadata
   cannot be correlated. Add core projection/precondition tests and gateway route tests
   proving a crafted remote approve never reaches the host bridge, while deny and read
   inventory behavior are unchanged.

Update any hand-maintained mobile/fleet contract snapshots required by these additive
wire fields. Preserve existing schema versions where serde defaults make the addition
backward-compatible; only bump a schema if the repository's contract tests demonstrate
that compatibility requires it.

## PyO3 And Wheel Plumbing

In `crates/sase_core_py/src/lib.rs`, add Python functions following the existing
dict/JSON conversion helpers:

- `sudo_validate_manifest(manifest: dict) -> dict`;
- `sudo_manifest_sha256(manifest: dict) -> str`;
- `sudo_derive_risk_badges(manifest: dict) -> list[dict]`;
- `sudo_validate_ledger(ledger: dict, manifest: dict | None = None) -> dict`;
- `sudo_runner_main(args: list[str]) -> None` delegating outside the GIL to the gateway
  CLI entry point.

Register all functions and add PyO3 tests for callable exports, normalized/error shapes,
hash parity with Rust, risk-badge parity, and ledger validation. Add
`python/sase_core_rs/sudo_runner.py`, mirroring `gateway.py` and `federation_worker.py`,
and add `sase_sudo_runner = "sase_core_rs.sudo_runner:main"` to `pyproject.toml`.

Extend the Linux CI wheel smoke and every release wheel smoke platform to assert that
the installed console script exists, the Python wrapper is callable, and
`sase_sudo_runner --help` succeeds without sudo or a TTY. Keep Windows compilation and
help behavior working even though execution is unsupported there. Update concise crate
or gateway documentation where needed so the manifest/hash/ledger stdout contract is
discoverable.

## Implementation Sequence

1. Add the core wire/error/validation/canonical-hash/risk module and focused Rust tests;
   export it from `sase_core`.
2. Implement the gateway runner behind test seams, add the thin binary and fake-sudo
   behavior tests, and verify help/non-Unix compilation boundaries.
3. Propagate `requires_tty` and `SudoRequest` through mobile/fleet projection, then add
   the gateway capability refusal and its bridge-not-called/deny-still-works tests.
4. Add PyO3 functions, Python wrapper, console-script metadata, registration/parity
   tests, and CI/release wheel smoke coverage.
5. Format and run the repository's complete verification; inspect the final diff for
   credential-shaped fields, executable substitution hooks, accidental version edits,
   and contract-snapshot drift.

## Verification And Phase Completion

From the `sase-core` repository root:

1. Run focused tests while iterating, including core wire/risk tests, runner fake-sudo
   tests, fleet attention projection/precondition tests, gateway route refusal tests,
   and PyO3 binding tests.
2. Run `./scripts/check.sh` (equivalent to `just check`) and fix every fmt, clippy, and
   workspace-test failure. Do not substitute a single-crate `cargo test` because it
   omits the binding tests.
3. Build an abi3 wheel into a temporary directory, install it into a fresh Python 3.12+
   virtual environment, assert the three console scripts resolve, and run
   `sase_sudo_runner --help`. Do not invoke an execution path or real sudo in smoke
   verification.
4. Confirm `git diff` contains no release-plz-owned version/pin edits and no changes in
   the Python `sase` repository.
5. Back in the `sase` project, run `sase bead epic-symbols sase-110.1`. Resolve every
   remaining symbol or re-key its Justfile annotation to the still-open parent/later
   phase that owns it.
6. Close only `sase-110.1` with a note naming the full `scripts/check.sh`, wheel-smoke,
   fake-sudo, and fleet deny-only evidence. Do not close `sase-110` or any other phase.
