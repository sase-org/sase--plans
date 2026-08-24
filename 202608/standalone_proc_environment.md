---
tier: tale
title: Restore the executable environment for stand-alone procs
goal:
  Stand-alone procs can run ordinary user-installed tools while SASE caller identity
  remains scrubbed.
size: medium
proposed_by: bbugyi200.athena.0cj
create_time: 2026-08-24 11:35:17
status: wip
---

# Restore the executable environment for stand-alone procs

## Objective

Make an approved stand-alone `%proc` execute ordinary user-installed commands such as
`just`, `uv`, and `cargo` from the same usable host tool environment as the SASE
process, while continuing to remove parent agent, chop, artifact, and stale proc
identity from the child. Preserve the existing proc-native lifecycle, private script,
workspace, timeout, and settlement behavior.

## Diagnosis and current behavior

The reported proc `3fa5986yj0f9` ran the approved Bash source
`just install && just check` through `/bin/bash --noprofile --norc` in the correct
project workspace and failed with exit 127 because `just` was not on its `PATH`. Its
persisted request contained:

```text
PATH=/home/bryan/.local/share/uv/tools/sase/bin:/usr/bin:/bin:/usr/local/bin
HOME=<proc runtime directory>
```

On the host, `just` and the Rust toolchain are under the user's Cargo bin directory and
`uv` is under the user's local bin directory. Giving the proc only the fixed path above
therefore hides all three. Merely adding the Cargo bin directory would be incomplete:
`just install` also needs `uv`, and Rustup-backed `cargo` needs the user's normal home
or toolchain environment.

The root cause is the special `xprompt-proc` branch in
`src/sase/procs/supervisor.py::_child_environment`. Ordinary procs start with the
supervisor environment and scrub SASE-specific caller identity, but this branch discards
the supervisor environment and uses only the map returned by the Rust
`sanitized_proc_env` helper. That Rust helper constructs a hermetic environment with a
fixed system-only `PATH` and proc-private `HOME`. This conflicts with the approved
stand-alone-proc design, which specifies starting from the supervisor's sanitized
environment, removing parent agent/chop/artifact identity, and then adding documented
proc context plus the SASE-interpreter path prefix.

## Implementation

1. In the linked `sase-core` repository, change the Rust-owned `%proc` environment
   contract from a complete hermetic environment to an additive overlay over an
   explicitly supplied executable environment. Keep this policy in
   `crates/sase_core/src/agent_launch/proc_runtime.rs` and expose it through
   `crates/sase_core_py/src/lib.rs` rather than rebuilding it in Python.
   - Preserve the caller/supervisor `PATH`, with the directory containing the selected
     SASE Python executable prepended once so `sase` remains resolvable.
   - Preserve ordinary host variables required by user-installed tools, including the
     real home and toolchain configuration, instead of replacing `HOME`, locale, time
     zone, and temporary-directory policy with proc-private defaults.
   - Override only execution-derived values such as `PWD` and the documented current
     proc/project/workspace context.
   - Keep private scripts and proc runtime files in the existing private work directory;
     changing the process environment must not change their location or permissions.
   - Keep the wire change additive/backward-readable where a persisted request can lack
     the new base-environment input, with a safe fallback that still prefixes the SASE
     interpreter and system executable directories.

2. In `sase`, make the xprompt-proc child environment follow the same inherited-then-
   scrubbed process boundary as ordinary supervised procs.
   - Start from the detached supervisor's environment, run the existing agent-identity
     and chop-context scrubbers, remove `SASE_ARTIFACTS_DIR`, and clear stale inherited
     proc-operation/session identity before adding the current proc's values.
   - Apply the Rust-produced proc overlay after scrubbing so the current `SASE_PROC_ID`,
     log path, project, project file, workspace, and prefixed `PATH` win.
   - Thread the base executable environment through
     `src/sase/agent/launch_proc_runtime.py` and the stable facade in
     `src/sase/core/agent_launch_facade.py`; do not capture an agent's artifact or
     operation-sidecar variables in the persisted request.
   - Retain `/bin/bash --noprofile --norc`, direct argv execution, the SASE interpreter
     for Python procs, the private `0600` script, cleanup, timeouts, cancellation, and
     lease settlement unchanged.

3. Add regression and contract coverage at both boundaries.
   - Rust tests should prove that an inherited path containing user bin directories is
     retained, the SASE interpreter directory is prefixed without destructive
     replacement, current proc/workspace values override stale values, and forbidden
     agent/artifact/proc identity is absent from the produced child contract.
   - Python process-level tests should place a fake executable named `just` in a
     temporary user bin directory, launch it through a real stand-alone Bash `%proc`,
     and assert successful resolution and execution. Also assert that normal home/tool
     configuration survives while `SASE_AGENT`, `SASE_AGENT_*`, `SASE_CHOP_*`,
     `SASE_ARTIFACTS_DIR`, and stale `SASE_PROC_*` sidecars do not leak.
   - Cover both immediate admission and the detached-coordinator path so delayed procs
     retain the same executable environment. Keep tests independent of whether CI has
     real `just`, `uv`, Cargo, or Rustup installations.
   - Preserve existing tests for Python interpreter selection, no agent artifacts,
     duplicate-fingerprint replay, workspace containment, timeout, cleanup, and
     settlement.

4. Update the `%proc` environment description in `docs/xprompt.md` (and any directly
   corresponding architecture wording) to state the actual contract: inherit the
   supervisor's ordinary tool environment, scrub SASE caller identity and stale proc
   sidecars, prefix the SASE interpreter directory, and add only the current documented
   proc context. Clarify that the private script directory is not a replacement user
   home and that environment scrubbing is not a filesystem or network sandbox.

## Verification

1. Open `sase-core` through `/sase_repo`, then run its formatting, Clippy, and complete
   Cargo test gates after the Rust and PyO3 changes.
2. In `sase`, run `just install` before tests so the local Rust binding is rebuilt from
   the linked core checkout.
3. Run the focused Rust-binding and process suites, including
   `tests/test_launch_proc_runtime.py`, the proc supervisor tests, and detached
   admission coordinator coverage.
4. Run `just check`. Because this changes a required Rust binding and the executable
   environment of user-authored proc code, run `just check-full` through `/sase_monitor`
   and inspect the result before declaring completion.
5. Perform a final end-to-end stand-alone `%proc` smoke test with a temporary fake
   `just` executable and verify the proc succeeds, records the expected native proc
   metadata, and leaves no agent artifacts or private script behind.

## Acceptance criteria

- A stand-alone `%proc` resolves and executes a user-installed `just` from the launching
  environment rather than failing with exit 127.
- Commands launched by `just` can see the normal user tool environment needed for `uv`
  and Rustup-backed Cargo; the fix is not a one-directory special case for `just`.
- The SASE interpreter directory remains first in `PATH`, and Python `%proc` units still
  use the exact SASE interpreter.
- Parent agent, chop, artifact, and stale proc-operation identity never appears in the
  child environment; only the current proc/project/workspace context is added.
- Private scripts remain mode `0600`, are executed by direct argv, and are cleaned up;
  workspace leasing, cancellation, timeout, replay, and settlement semantics do not
  regress.
- Core, binding, focused process tests, `just check`, and monitored `just check-full`
  all pass, and the public `%proc` documentation matches the implemented environment
  contract.
