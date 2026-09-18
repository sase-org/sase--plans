---
tier: tale
title: Make workspace sidecar clone materialization atomic
goal:
  Abrupt process termination can no longer expose a partial Git clone at a canonical SDD
  sidecar path.
size: medium
proposed_by: bbugyi200.athena.0mw
create_time: 2026-09-18 09:41:44
status: wip
---

# Make workspace sidecar clone materialization atomic

## Context

At 2026-09-18 09:26:28 EDT, telemetry recorded `sdd.clone.remote` cloning the `plans`
sidecar into workspace 25. The Git child was terminated by `SIGTERM` (`returncode: -15`)
after 117 ms, while stderr contained only the initial `Cloning into ...` message. Git
had already created the destination and left its temporary clone state behind:
`.git/HEAD` pointed at `refs/heads/.invalid`, and the repository had no refs or objects.

The next refresh correctly rejected that directory as an unrecoverable detached HEAD.
Strict workspace preparation later quarantined it and successfully cloned a fresh
replacement, so no plan data was lost. The quarantine notification was therefore the
recovery mechanism working, not the original fault.

The underlying gap is that fresh SDD/sidecar clones are materialized directly at their
canonical path. The existing retry and failure branches remove partial output only if
the Python caller survives long enough to run cleanup. Process death or process-group
termination can occur between Git creating the directory and that cleanup, exposing an
incomplete clone to subsequent readers. The available evidence proves that `SIGTERM`
interrupted this clone; it does not identify the signal's issuer, and the fix should not
depend on that provenance.

## Implementation

1. Introduce a target-scoped clone transaction in `src/sase/sdd/_store_clone_ops.py`
   that keeps in-progress clones outside the canonical destination. Use a SASE-owned
   staging container adjacent to the target (rather than a direct child that workspace
   sidecar discovery could mistake for another sidecar), a unique attempt directory, and
   an advisory lock keyed by the canonical target. The lock must be released
   automatically on process death and honor any existing materialization deadline.

2. While holding that target lock, remove only abandoned staging entries owned by the
   same target, run the existing reference/fallback/retry policy against a staging path,
   and keep telemetry expressed in terms of the canonical store while adding staging
   information when useful for forensics. Preserve the host-wide remote-clone admission
   limit and all current timeout, retryability, and strict/non-strict error semantics.

3. Before publication, verify that a successful Git command produced a usable attached
   checkout with a resolvable `HEAD`, matching configured remote, and tracking upstream.
   Promote it to the canonical path with one same-filesystem rename only after
   validation. Never overwrite a destination that appeared concurrently: accept it only
   after proving it is the same healthy store; otherwise discard the staged result and
   fail closed.

4. Apply the transaction consistently to both remote clones and clones seeded from the
   primary store. Refactor the legacy replacement path in `src/sase/sdd/_store_link.py`
   to use the same staging ownership rules so an interrupted replacement cannot leave a
   clone-shaped direct child under `sase/repos/`. Preserve its backup/restore guarantees
   for pre-existing content.

5. Extend `tests/sdd_store/test_sidecar_clone_retry.py` and the focused store-link tests
   to cover successful atomic promotion, reference fallback and retry cleanup,
   primary-seeded clones, concurrent destination appearance, and health-validation
   failure. Add a subprocess regression that terminates a materializer after Git creates
   partial staging, proves the canonical sidecar path was never exposed, then verifies a
   later materialization safely reaps the abandoned stage and succeeds without invoking
   damaged-sidecar quarantine.

6. Run the focused SDD clone/store-link tests and the repository's normal `just check`
   verification gate.

## Expected outcome

Abrupt termination may leave an identifiable staging directory, but it can no longer
leave a half-built repository at `sase/repos/<role>`. The next launch can materialize
the canonical sidecar normally, while the existing quarantine path remains available for
genuinely damaged repositories whose local state might need preservation.
