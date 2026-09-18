---
tier: tale
title: Fix remote screenshot execution and save an Apollo PNG artifact
goal:
  Make sase screenshot --host apollo succeed with accurate diagnostics and store its
  verified PNG as a permanent SASE image artifact.
size: medium
proposed_by: bbugyi200.athena.0my
create_time: 2026-09-18 10:44:14
status: wip
---

# Fix remote screenshot execution and demonstrate an Apollo PNG artifact

## Goal

Make `sase screenshot --host apollo` work with Apollo's existing compatible SASE
installation, preserve safe SSH argument transport and capture cleanup, and save the
resulting PNG as a new permanent SASE image artifact. Explain the observed root causes
and report the actual verification and artifact reference.

This is one bounded implementation task for one coding agent. No implementation files
were changed during planning. The requested plan submission precedes implementation.

## Confirmed diagnosis

The current remote adapter is `src/sase/screenshot/remote.py`. `_probe_contract` sends
`sase screenshot --contract` through `_ssh_argv`, which quotes arguments correctly but
executes them in SSH's default noninteractive, non-login shell. Every nonzero probe
status except SSH transport failure becomes the same misleading upgrade message.

Read-only probes against Apollo established:

- Plain SSH running `sase screenshot --contract` exits 127 with
  `zsh:1: command not found: sase`.
- Its PATH includes system binaries and `.cargo/bin` but omits `.local/bin`.
- The user's `.local/bin/sase` is executable. Both its absolute invocation and
  `"$SHELL" -lc 'sase screenshot --contract'` return `{"schema_version": 1}`.
- A noninteractive login shell resolves SASE and tmux without startup output. An
  interactive shell also resolves SASE but emits an inappropriate-ioctl warning;
  interactive shells and allocated terminals are unsuitable for this protocol.
- `_remote_sase_version` has a second independent bug: it invokes `sase --version`.
  Apollo rejects that with exit 2. The supported command is `sase version --json`. Its
  schema-1 payload contains a `packages` record with `name: sase`, `role: host`, and
  `display_version: 0.17.1+861.g3fb42fa11` at investigation time. This version can
  change; use the value from the eventual capture.
- `tests/main/test_screenshot_remote_command.py` masks both defects. Its fake remote
  runner puts its SASE executable directly on PATH, and the fake executable accepts
  `--version` even though the real parser does not.

Prior SSH quoting and retained-target fixes remain relevant: preserve the single quoted
remote command string, literal key/query arguments, owned tmux target identity, bounded
deadlines, and cleanup after failures.

## Scope and implementation

### 1. Run remote SASE commands in the user's noninteractive login environment

Update the Python SSH subprocess adapter so the contract probe, SVG capture, and version
query use the same noninteractive login-shell invocation. Use the remote `SHELL` with a
`/bin/sh` fallback, and pass a safely quoted `exec` command containing the original
argument vector to `-lc`. Expand the shell choice on the remote side. Preserve both
quoting layers: the SSH command is one argument, and arbitrary screenshot keys, regexes,
paths, and forwarded TUI arguments remain literal arguments inside the login shell. Do
not hard-code Bryan's home directory, require zsh, source an interactive rc file,
allocate a PTY, or change machine configuration to make the test pass.

Keep the SVG fetch and cleanup as minimal shell commands; they do not need SASE's login
environment. Preserve the existing SSH destination validation and timeout/error
handling. Keep the existing send-keys hint behavior unless a directly demonstrated
failure requires changing it.

This work repairs Python CLI/subprocess glue around a Textual screenshot facility; there
is no new shared domain algorithm. The linked `sase-core` was opened through
`sase repo open` and inspected during planning; no existing screenshot or login-shell
command builder was found. Keep the bounded adapter changes here. If implementation
uncovers a necessary shared backend behavior, reopen the linked core through `sase_repo`
and put that behavior and its binding in Rust rather than duplicating it.

### 2. Use the real version command and preserve actionable diagnostics

Replace `sase --version` with `sase version --json`. Parse the existing version
inventory contract (see `src/sase/version/render.py`, `src/sase/version/inventory.py`,
and `tests/main/test_version_command.py`), selecting the `sase` host package's nonempty
`display_version`. Preserve one machine-readable output line,
`remote_sase_version=sase <display_version>`, including its source revision suffix.
Reject malformed JSON, incompatible payloads, and missing/invalid host version data with
an actionable version-query error; do not parse the human table or accept arbitrary
stdout as a version.

Make failed contract probes identify the failing action, host, exit status, and
available remote stderr/stdout. A missing executable should suggest checking the remote
login environment and installation; a nonzero startup/import failure should retain its
real cause. Reserve definite incompatible-contract/upgrade advice for evidence of
incompatible support, such as a schema mismatch. Malformed contract output should
identify the protocol problem and include bounded useful output rather than assert that
an upgrade is necessarily the solution. Preserve the existing distinct SSH-255 and
timeout diagnostics.

Update the remote screenshot paragraph in `docs/ace.md` to explain the login-shell
environment requirement and version reporting. No new CLI options or screenshot contract
version are needed. Do not edit durable memory as part of this fix.

### 3. Add regressions that reproduce the real failures

Extend `tests/main/test_screenshot_remote_command.py`, splitting fixtures into a helper
module if required by repository file-size gates. Retain actual shell interpretation in
transport tests. Build an isolated remote environment where SASE is absent from the
initial PATH and becomes available only through login-shell setup; do not inherit the
developer's dotfiles or accidentally find the installed SASE executable.

Cover:

- Successful probe, capture, and version query through that environment, followed by SVG
  retrieval, local PNG rendering, and remote temporary-file cleanup.
- Literal argument preservation through the added shell layer, including spaces, quotes,
  dollar expressions, semicolons, and pipes; existing kept-window/recapture and
  send-keys regressions must continue to pass.
- A fake SASE CLI that implements `version --json` using the real schema and rejects
  unsupported `--version`. Include a parser-level check against the real SASE parser so
  the transport fixture cannot silently invent the interface again.
- Version payloads with other package records before the host record, plus invalid JSON,
  absent host data, and failed version commands; preserve cleanup on failures.
- Missing SASE after login initialization, runtime/probe failure details, malformed
  contract output, incompatible schemas, SSH authentication/transport failure, and the
  existing single bounded operation budget.

Run the focused screenshot and version suites using the repository environment. Use
controlled fixtures rather than requiring Apollo access for automated tests.

## Verification and requested deliverable

1. Read the current `lint_and_test.md` and `tui_screenshot.md` via `sase_memory_read`
   before implementation/verification. Read `tui_perf.md` only if work reaches the live
   app's export/settle loop; the confirmed failures are in remote invocation, not
   rendering or convergence.
2. Ensure the workspace runtime uses the changed Python source and required
   dependencies; use `just install` if needed. Run the focused tests, format the changed
   files, and run the required `just check`. If a long check needs a handoff, use
   `sase_monitor` after `just fix` or `just fmt`, as the verification memory requires.
   Do not replace whole-repo lint with only focused pytest results.
3. Put the workspace's SASE executable on the local PATH and verify its source
   resolution, then run the exact command `sase screenshot --host apollo` with its
   default PNG output. No shell alias, SSH wrapper, precreated PNG, or SVG-only command
   substitutes for this public entry point. The fixed caller should work with Apollo's
   existing compatible installation; the demonstrated failures do not require an Apollo
   package upgrade or remote source edits.
4. Record exit status and the printed `host=`, `remote_sase_version=`, `png=`, `svg=`,
   and tmux metadata. Open the produced PNG with the image viewer and inspect the actual
   TUI capture; check that it is a valid nonempty image. Use default cleanup rather than
   leaving a new retained tmux window. If another failure appears, use its evidence to
   finish the in-scope pipeline repair rather than claiming the preflight fix alone
   completes the task.
5. Read `sase_artifacts.md` via `sase_memory_read`, then register the exact PNG path
   from the successful command:

   ```bash
   sase artifact create --kind image --path <printed-png-path> --label "Apollo TUI screenshot after remote capture fix"
   ```

   Retain the source file; use the normal copy behavior. Save the returned canonical
   `file:` reference and check it with `sase artifact show` so the final response
   identifies a durable image artifact rather than only a temporary path.

6. Report the PATH/login-shell root cause, the unsupported version invocation, the
   passing checks, the successful real Apollo capture/version, and a link/reference to
   the saved image. Follow `sase_final` for the completion declaration.

## Acceptance criteria

- A compatible SASE installed through the remote login environment is usable even when
  SSH's initial PATH cannot find it.
- Remote preflight failures expose the actual cause and distinguish incompatible
  contract data from transport, executable lookup, and startup errors.
- Version reporting uses the supported runtime inventory API and stays one line.
- Existing quoting, timeout, renderer, target-identity, and cleanup behavior remains
  covered; focused tests and `just check` pass.
- The exact `sase screenshot --host apollo` command succeeds, its PNG is visually
  inspected, and those PNG bytes are stored as a new explicit SASE image artifact.
