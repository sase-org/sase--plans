---
tier: tale
title: Snapshot sudo executables on the execution host and surface macOS TCC blocks
goal:
  Remote sudo requests snapshot executables on their target, and macOS Full Disk Access
  blocks are visible to reviewers and successors.
size: medium
decisions:
  unreachable_target:
    ask:
      Should `sase sudo request` fail when it cannot snapshot executables on the remote
      target?
    choices:
      fail:
        Refuse to create the request; the agent retries once the target is online with
        upgraded sase
      unverified:
        Create it with null hashes marked unverified; reviewers approve without a target
        snapshot
    default: fail
    why:
      approval needs the target online anyway; unverified snapshots bring back the false
      assurance
    answer: fail
  tcc_surfaces:
    ask:
      Should sase show macOS Full Disk Access warnings and failure hints, not just
      document them?
    choices:
      full:
        Probe FDA, warn in preview and review modal, and hint the successor in the
        approve result
      docs_only:
        Only update the /sase_sudo skill and sudo docs; no automatic reviewer or
        successor signal
    default: full
    why:
      1ar's successor blamed SIP; only an automatic signal reaches reviewer and
      successor
    answer: full
proposed_by: bbugyi200.athena.1au
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 07:51:15
status: wip
---

# Snapshot Sudo Executables On The Execution Host And Surface macOS TCC Blocks

## Background

Session `1ar` (bob-cli, plan `202611/mac_cron_mail_relay.md`) asked for root work on the
`mac` tailnet machine through `sase sudo request` with `machine: mac`. Its third sudo
gate settled as approved, but the first command failed and the rest were skipped:

```
backup: /bin/cp -p /etc/postfix/main.cf /etc/postfix/main.cf.bak-20261110
exit 1: "cp: /etc/postfix/main.cf.bak-20261110: Operation not permitted"
```

The Mac's unified log (macOS 26.5) shows the real cause:

```
tccd  AUTHREQ_ATTRIBUTION responsible={identifier=com.apple.sshd-keygen-wrapper ...}
      accessing={identifier=com.apple.cp, euid=0, binary_path=/bin/cp}
      service=kTCCServiceSystemPolicyAllFiles -> authValue=0 (denied)
kernel (Sandbox) System Policy: cp(48871) deny(1) file-write-create /private/etc/postfix/main.cf.bak-20261110
```

So sudo worked and `cp` ran as root. macOS TCC refused the write because SSH sessions on
that Mac lack Full Disk Access ("Allow full disk access for remote users" is off under
Remote Login). SIP is not involved: `/private/etc/postfix` has no `restricted` flag. A
non-root process in the same SSH session can detect the condition. Reading the
world-readable `/Library/Application Support/com.apple.TCC/TCC.db` fails with `EPERM`
when FDA is missing.

The EPERM is a host policy, not a sase bug. While investigating it, two real sase
defects turned up, and both made the agent's job harder:

1. **Executables are snapshotted on the wrong host.** `_executable_sha256` in
   `src/sase/sudo/models.py` hashes `argv[0]` on the machine that runs
   `sase sudo request` (athena), even when `machine` names a remote target. As a result:
   - Mac-only binaries such as `/usr/sbin/scutil` are rejected with
     `unsnapshotshotted_executable`. The agent fell back to running
     `/usr/bin/python3 <agent-writable script>` wrappers, which reviewers cannot
     inspect.
   - Every "Executable SHA-256" shown for a remote command is athena's binary, not the
     target's. That is false assurance.
   - The `agent_writable_executable` guard checks athena. A path that is root-owned on
     athena but user-writable on the target, such as `/usr/local/bin/*` from Homebrew on
     an Intel Mac, passes the check. The agent could then swap that binary after review.
   - `validate_sudo_spec` (`src/sase/notification_gates/kind_validation/sudo.py`) calls
     `normalize_sudo_request` again, so the athena-side rehash also runs at staging and
     creation.
2. **Nothing explains a TCC denial.** The ledger shows only "Operation not permitted".
   The successor agent (`1ar--3`) concluded "SIP blocked /etc/postfix" and began
   planning workarounds.

## Goal

- Remote sudo requests snapshot and guard executables on the host that runs them.
- They re-verify those snapshots on the target just before running.
- Reviewers and successor agents learn when macOS TCC (missing Full Disk Access) will
  block or did block a command.

The Rust runner and the sudo manifest wire stay unchanged: the manifest carries no
executable hashes, and nothing here moves domain logic across the sase-core boundary.
All changes are in sase's existing Python sudo request, SSH, and presentation layers.

## Design

### Target-side snapshot entrypoint

- Add a hidden internal option to the hidden target-side `sase sudo exec` command:
  `--snapshot PATH`, repeatable. It is internal subprocess plumbing, so it needs no
  short alias.
  - It needs no TTY and prints one JSON object:
    `{"schema_version": 1, "kind": "sase_sudo_executable_snapshot", "host_facts": {...}, "executables": [{"path", "sha256", "error_code", "message"}]}`.
  - Each entry carries either a `sha256` or an `error_code` from the existing codes:
    `unsnapshotshotted_executable` (relative, missing, or not a regular file) or
    `agent_writable_executable` (writable by the SSH user).
- Move the existing checks out of `models.py` into a new focused module,
  `src/sase/sudo/snapshot.py`, with the same semantics: absolute path, `stat`,
  `S_ISREG`, `os.access(W_OK)`, chunked SHA-256. One function returns a per-path result
  instead of raising.
  - Local requests and the target-side entrypoint both use it.
  - `models.py` keeps raising the same `GateError`s for local requests.
- Add `host_facts()` to the same module. It returns
  `{"platform": sys.platform, "full_disk_access": bool | None}`.
  - On `darwin`, `full_disk_access` comes from a read of
    `/Library/Application Support/com.apple.TCC/TCC.db`. Success means `True`. A
    `PermissionError` whose `errno` is `EPERM` means `False`. Anything else means
    `None`.
  - On other platforms it is `None`.
- Add `"executable_snapshot"` to the capabilities that `sase sudo exec --contract`
  returns. Keep `schema_version: 1`; the change is additive.

### Controller-side remote snapshot

- In `src/sase/sudo/ssh_exec.py` (or a sibling `ssh_snapshot.py` if `ssh_exec.py` grows
  too big), add `snapshot_remote_executables(host, paths, *, command_runner=None)`.
  1. Probe the contract with the existing `_probe_contract` login-shell path.
  2. If the target does not advertise `executable_snapshot`, raise
     `GateError("remote_snapshot_unavailable", ...)`. The message names the host and
     says "upgrade sase on <host>".
  3. Run
     `encode_target_sase_command("sase", "sudo", "exec", "--snapshot", p1, "--snapshot", p2, ...)`
     with no TTY and `REMOTE_POLL_SSH_TIMEOUT_SECONDS`-scale timeouts.
  4. Parse and validate the JSON and return the per-path results plus `host_facts`.
- Run request-time SSH calls non-interactively. They run inside agents, so add an opt-in
  `batch=True` to `ssh_argv` that inserts `-o BatchMode=yes`, and use it here.
- If SSH is unreachable or a transport error occurs, raise `remote_snapshot_unavailable`
  with a message that the target must be online when the request is created.
- Add `remote_snapshot_unavailable` and `snapshot_mismatch` to
  `PRE_SPAWN_REMOTE_ERROR_CODES` so either failure during approval leaves the gate
  pending.

### Request normalization

- Restructure `normalize_sudo_request` in `src/sase/sudo/models.py` so the snapshot
  depends on the target. `_command` stops hashing inline and resolves argv0 snapshots in
  one batch after parsing.
- **Build path.** `build_sudo_gate_request` in `gate.py` asks for a fresh snapshot.
  - Local targets keep today's behavior.
  - Remote targets (`resolve_sudo_target(machine).remote`) make one
    `snapshot_remote_executables` call for the unique argv0 set.
    - Any per-path `error_code` raises the matching `GateError` at
      `commands[i].argv[0]`.
    - A declared `executable_sha256` that differs raises `snapshot_mismatch`, as today.
- **Validation path.** `validate_sudo_spec` makes no SSH calls.
  - Local targets keep today's rehash-and-compare.
  - Remote targets require every command to carry a declared 64-hex `executable_sha256`
    and accept it without touching the local filesystem.
  - This path must never make an SSH call. Validation runs at staging and creation, and
    the target may be offline then.
- `_reject_unsafe_argv` stays name-based and host-independent.

> [!decision] unreachable_target = unverified Do not raise when the remote snapshot is
> unavailable. Record `executable_sha256: null` for each command and set the target
> snapshot marker to `"unverified"`.
>
> - Validation accepts null hashes only for remote requests with that marker.
> - The preview and the modal show "not snapshotted (target unreachable at request
>   time)" in a warning style.
> - At approval, a snapshot is still taken on the target and checked for
>   `agent_writable_executable`. There is no reviewed hash to compare against, so a hash
>   mismatch cannot be detected.

### Snapshot provenance in the payload

- Extend `_target_payload` in `gate.py` with a `snapshot` object:
  `{"taken_on": "target" | "local", "host_facts": {...}}`. Remote requests get the facts
  from the target response. Local requests run `host_facts()` locally; a local macOS
  terminal can lack FDA too.
- Nothing validates `payload.sudo.target` strictly. Keep the change additive and make
  readers tolerate its absence, because gates created before this change are still
  pending.

### Approval-time re-verification (remote)

- In `_approve` (`src/sase/sudo/cli.py`), before the first `claim_execution_record` and
  before any detached or synchronous dispatch, run `snapshot_remote_executables` for the
  selected commands' argv0 paths. Do this only when `target.remote` is true and
  `target.snapshot.taken_on == "target"`.
  - Compare against `payload.sudo.request.commands[*].executable_sha256`.
  - A mismatch or a target-side error raises `snapshot_mismatch`, or the entry's own
    code. Either way the gate stays pending.
  - If the fresh `host_facts` show `platform == "darwin"` and
    `full_disk_access is False`, print a one-line warning to stderr. It must not block
    approval.
- Legacy remote gates, which have no `snapshot` marker, skip this step. Their hashes
  came from the controller and would always mismatch.
- Local approval behavior is unchanged.

### TCC visibility

Gated by the `tcc_surfaces` decision.

- Put one shared helper, for example `tcc_warning(target_payload) -> str | None` in
  `src/sase/sudo/snapshot.py`, behind every surface so the wording cannot drift. It
  returns text only when `snapshot.host_facts.platform == "darwin"` and
  `full_disk_access is False`, roughly:

  > macOS TCC: SSH sessions on `<host>` lack Full Disk Access. Root writes to protected
  > paths (for example `/etc/postfix`) will fail with "Operation not permitted". Enable
  > System Settings → General → Sharing → Remote Login ⓘ → "Allow full disk access for
  > remote users" before approving.

  Local requests use "this terminal" wording instead.

- **Preview.** `_preview` in `gate.py` adds a `Target facts:` line (platform and FDA)
  and, when the helper returns text, a warning block above `Commands:`. Telegram,
  mobile, and `sase gate show` all render `sudo-request.md`, so they get the warning
  too.
- **Review modal.** `src/sase/ace/tui/actions/agents/_notification_sudo.py` parses the
  facts, and `src/sase/ace/tui/modals/sudo_request_modal.py` shows the warning in the
  Context section, styled like the existing risk text. Missing facts render nothing.
- **Successor hint.** `sudo_approve_entrypoint` in `src/sase/sudo/commands.py` adds an
  optional `hints` list to the approve result. A hint is added when both of these hold:
  - The request's `target.snapshot.host_facts.platform == "darwin"`.
  - At least one ledger entry is `failed` with `"Operation not permitted"` in its
    `output_tail`.

  The hint names the failed command IDs and says the denial is macOS TCC, not SIP and
  not a sase fault. It says how to enable FDA for remote users. It tells the agent to
  ask Bryan and not to work around the denial. When `full_disk_access` was `False`, the
  hint is definite; otherwise it says "likely". `result_schema` does not forbid
  additional properties, so `hints` is purely additive. Do not add it to `required`.

> [!decision] tcc_surfaces = docs_only Skip `host_facts()`, the preview and modal
> warning, the approve-result `hints`, and the approval-time stderr warning. Do the
> skill and docs updates below.

### Skill and docs

- Update the generated skill source `src/sase/macros/skills/sase_sudo.md`:
  - Executables are snapshotted and guarded on the target host. Use paths that exist on
    the target, and do not wrap target-only binaries in interpreter scripts.
  - The target must be reachable over SSH, with sase upgraded, when the request is
    created.
  - On macOS targets, "Operation not permitted" from a root command means TCC (SSH lacks
    Full Disk Access), not SIP. Stop and ask Bryan to enable "Allow full disk access for
    remote users" rather than working around it.
  - Pre-flight check:
    `ssh -o BatchMode=yes <host> head -c1 '/Library/Application Support/com.apple.TCC/TCC.db'`.
    `Operation not permitted` means FDA is missing.
  - Follow the generated-skills memory: edit only the source template, and do not deploy
    with `sase skill init` from the unlanded tree.
- Update `docs/reference/automation/sudo.md`. Read `docs_writing.md` through
  `/sase_memory_read` first.
  - **Overview / Agent Contract:** executables are snapshotted on the execution host.
  - **Remote Flow:** add the request-time snapshot step and the approval-time
    re-verification step. The target needs the `executable_snapshot` capability.
  - New **macOS targets and Full Disk Access** subsection: the symptom, the log
    signature (`System Policy: ... deny(1) file-write-create`), and the remedy.
  - **Policy Rationale:** say precisely where hashes are verified.

## Tests

Use `tests/_sudo_ssh_fake.py` and extend its fake remote `sase`:

- Answer `sudo exec --snapshot`, driven by environment variables such as
  `SASE_FAKE_SNAPSHOT_JSON` and `SASE_FAKE_REMOTE_CAPABILITIES`.

Add or extend tests in the existing files: `tests/test_sudo_gate_request.py`,
`tests/test_sudo_ssh.py`, `tests/test_sudo_acceptance_remote.py`,
`tests/test_sudo_gate_approval.py`, and `tests/test_sudo_parser.py`.

**Target-side entrypoint**

- `sase sudo exec --snapshot` reports a hash for a regular file.
- It reports `unsnapshotshotted_executable` for a missing, relative, or directory path.
- It reports `agent_writable_executable` for a user-writable file.
- `host_facts`, with the platform and probe monkeypatched:
  - darwin with an EPERM read gives `False`;
  - darwin with a successful read gives `True`;
  - darwin with any other error gives `None`;
  - linux gives `None`.
- `--contract` lists `executable_snapshot`.

**Request build**

- A remote request whose argv0 is absent locally (for example `/usr/sbin/scutil` on
  Linux CI) succeeds, and the hash comes from the fake target.
- A remote path that is root-owned locally but writable on the target is rejected with
  `agent_writable_executable`.
- An unreachable target, or one without the capability, raises
  `remote_snapshot_unavailable`. Under `unreachable_target = unverified`, the request is
  instead created with null hashes and the `unverified` marker.
- A declared hash that differs raises `snapshot_mismatch`.
- Local requests behave exactly as before.

**Validation**

- `validate_sudo_spec` on a remote payload with declared hashes passes with no SSH call
  (assert the fake runner is never invoked) and no local file access.

**Approval**

- A remote gate with `taken_on: target` runs the snapshot before staging. A hash
  mismatch leaves the gate pending, and the stage and exec SSH calls never happen.
- A legacy remote payload with no `snapshot` marker skips verification.
- A darwin target without FDA prints the stderr warning and still proceeds.

**TCC surfaces**

- The preview contains the warning for darwin with FDA `False`. The preview does not
  contain it for linux or when facts are missing.
- The modal context data carries the warning.
- The approve entrypoint adds `hints` for darwin plus a failed EPERM entry. It adds none
  for linux, for `output_to_agent: none` with empty tails, or for successful ledgers.

Some existing remote-request tests currently rely on local snapshots of local paths.
Update their fakes to answer the snapshot call; do not weaken their assertions.

## Verification

- Run `just install-venv` if the venv is stale, then `just fmt` and
  `sase tool run check`. Do not run `just check-full`.
- `src/sase/sudo/models.py` is about 510 lines. Keep every touched file under the
  `toobig` limit, which is why the snapshot logic moves into `snapshot.py`.
- If the modal's rendered output changes for an existing PNG golden, run
  `just fix-tui-screenshots` for the sudo modal selectors and inspect the report. Facts
  are absent in existing fixtures, so no golden should change.

## Out Of Scope

- Changing macOS settings on the Mac. Enabling "Allow full disk access for remote users"
  is Bryan's call and needs the GUI.
- Bypassing TCC, for example with launchd jobs.
- Rust runner or manifest wire changes, including hash verification inside
  `sase_sudo_runner`.
- Reviewing the contents of interpreter script arguments, such as
  `python3 /path/script.py`.
- Renaming the misspelled `unsnapshotshotted_executable` error code.
