---
tier: tale
title: Self-heal dirty hidden artifact-link clones
goal:
  A stranded hidden-clone write is recovered losslessly, its orphaned link row is
  imported, and subsequent artifact-link writers continue normally.
size: medium
proposed_by: bbugyi200.athena.sase-10y
bead: sase-10y
create_time: 2026-09-18 07:18:35
status: wip
---

- **BEAD:**
  [sase-10y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-10y/README.md)

# Self-heal dirty hidden artifact-link clones

## Objective

Complete `sase-10y` by making a matching, host-owned artifact-link sidecar clone recover
from stranded tracked, staged, untracked, or unpublished state instead of refusing every
later writer; preserve the stranded bytes in the existing audited recovery mechanism;
import the specifically preserved legacy `cites` row into the immutable link-event
store; and verify the live write lane before closing the bead.

## Root cause and constraints

- `resolve_machine_artifact_link_store()` currently calls `_fresh_integration_blocker()`
  before `ensure_sidecar_sdd_clone(..., fresh=True)`. The blocker rejects any dirty
  worktree or unpublished `HEAD`, so the established
  `integrate_machine_managed_sdd_repository()` recovery path never gets a chance to
  snapshot the state, reset the disposable host-owned clone to its upstream, and retry
  integration.
- Recovery must remain limited to a clone whose Git identity and configured remote match
  the expected hidden role. Non-Git paths, missing or mismatched remotes,
  detached/unsafe states, missing upstreams, and lock contention must continue to fail
  closed without discarding data.
- The existing recovery implementation already serializes through the store write lock,
  snapshots tracked/staged/untracked state into a verified stash-backed
  `refs/sase/recovery/...` ref, retains unpublished commit history, resets to the
  upstream, emits the recovery ref in diagnostics, and refuses recovery it cannot prove
  safe. Reuse that mechanism rather than inventing a second quarantine format.
- The row in `file:explicit:49cfb149110acf3f8110d8d7` is not currently returned by
  `sase artifact link list`; recovery alone is insufficient because it preserves
  stranded bytes without converting that legacy index row into an immutable event.

## Implementation

1. Change the hidden machine-store preparation path in
   `src/sase/sdd/_artifact_link_machine_store.py` so a correctly identified hidden clone
   requested with `fresh=True` reaches `ensure_sidecar_sdd_clone()` even when it has
   worktree dirt or unpublished commits. Keep the preflight identity/remote protections
   and deadline handling. Let strict machine-managed integration decide whether recovery
   is safe, and surface its typed failure if recovery, upstream resolution, or locking
   cannot safely complete. Remove only redundant preflight helpers/imports that no
   longer serve a safety check.

2. Update `tests/sdd/test_artifact_link_machine_store.py` and the relevant hidden-clone
   end-to-end coverage to prove the behavior at the actual boundary:
   - a matching plans clone containing tracked, staged, and untracked stranded content
     resolves successfully, becomes clean and upstream-aligned, and retains every
     stranded byte in a verifiable recovery ref/stash;
   - a matching clone with unpublished commits likewise recovers without losing the
     unpublished tree;
   - after recovery, an artifact-link event can be committed through the resolved hidden
     store, demonstrating that the shared lane is no longer wedged;
   - a dirty custom document role recovers independently without blocking plans;
   - wrong-remotes/non-Git identities and unsafe or unavailable recovery conditions
     remain preserved and rejected. Adjust older tests whose expected contract was
     “preserve and wedge” so they instead assert “snapshot, clean, and continue” only
     for matching host-owned clones. Keep publication-retry tests fail-closed where a
     retry worker does not own clone recovery.

3. Recover the live plans lane through the fixed code path and capture the reported
   recovery ref for the currently stranded state. Use the audited artifact payload
   `file:explicit:49cfb149110acf3f8110d8d7` as the source of truth for the earlier
   orphan. Reconstitute that exact schema-v2 legacy index in the host-owned plans clone
   only long enough to run the supported `sase artifact link import-indexes` workflow:
   preview, record the emitted attestation token, apply with that token, and allow it to
   publish the deterministic baseline event/cutover markers. Do not hand-edit the
   immutable event store. If the newly recovered snapshot contains additional valid
   orphaned legacy link indexes, restore and include those exact verified payloads in
   the same import; leave unrelated recovered material only in its recovery ref.

4. Verify the operational result before closing:
   - the targeted test module(s) and a focused end-to-end artifact-link mutation test
     pass;
   - the repository-required `just check` validation passes after consulting the
     lint/test memory;
   - the hidden plans clone is clean and aligned after import;
   - `sase artifact link list plan:202609/prose_if_proc_directive_false_positive.md --json`
     contains the preserved `agent:bbugyi200.athena.0j3--code` `cites` row exactly once;
   - a legitimate typed link tying the implemented plan/artifact to `bead:sase-10y` can
     be added and read back, proving a new write succeeds through the formerly wedged
     lane;
   - the recovery ref still resolves and contains the stranded pre-recovery bytes.

5. Re-read `sase bead show sase-10y`, record any materially distinct out-of-scope
   discovery through `/sase_new_task` with `sase-10y` as causal context, and close the
   bead with
   `sase bead close sase-10y --note "<concise code, test, live-import, live-write, clean-clone, and recovery-ref verification>"`
   only after all checks above succeed.

## Acceptance criteria

- One crashed artifact-link writer can no longer wedge all later writers merely by
  leaving a matching hidden sidecar dirty or locally ahead.
- Every automatic cleanup is serialized, lossless, auditable by recovery ref, and
  restricted to a correctly identified machine-owned hidden clone.
- The preserved `prose_if_proc_directive_false_positive` citation is represented in the
  immutable event-backed link graph exactly once.
- Focused tests and `just check` pass, the live clone is clean, a live typed link
  write/read succeeds, and `sase-10y` is closed with those facts in its note.
