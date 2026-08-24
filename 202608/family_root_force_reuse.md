---
tier: tale
title: Make family-root kill-and-edit name reuse deterministic
goal: "Killing or dismissing an agent family and relaunching its explicit name from ACE
  succeeds deterministically, without racing cleanup persistence, resurrecting stale
  reservations, widening exact-member cleanup, or weakening clan-container safety.

  "
size: medium
proposed_by: bbugyi200.athena.0cf
---

# Plan: Make family-root kill-and-edit name reuse deterministic

## Diagnosis

ACE's family-root kill-and-edit contract and the forced-reuse cleanup contract disagree.
`prepare_kill_and_edit_prompt()` correctly turns an explicit epic phase root such as
`sase-sq.1--plan` back into the new-family launch form `%id(!1, clan=sase-sq, ...)`;
`force_reuse_owner_names()` therefore identifies `sase-sq.1` as the owner to release.
Once the original planning shell has handed off to a coding shell, however, the durable
name registry represents `sase-sq.1` as a family container. The low-level
`wipe_agent_name_for_reuse()` deliberately refuses all populated containers, and
`wipe_names_for_forced_reuse()` turns that refusal into
`Cannot force-reuse family container 'sase-sq.1'`.

The failure is intermittent because kill-and-edit updates ACE optimistically while
submitting bundle-save and artifact-deletion work through a tracked cleanup proc. A fast
prompt submission can reach forced reuse while the old family artifacts still derive the
container. Waiting longer can change what the registry sees, but launch-side cleanup
must also be safe against the cleanup proc still saving dismissed bundles: a late bundle
write could reintroduce the old family reservation after a replacement has launched.

The repository already has the missing family semantic in
`sase.bead.cli_work_name_cleanup`: deterministic bead relaunch resolves the newest
family generation, wipes every concrete shell, rebuilds the registry, and rejects
residual reservations. That behavior is private to bead work. Existing tests separately
prove family-root prompt rewriting, exact family-member wiping, generic container
refusal, and the mocked durable launch seam, but no test seeds a real family registry
and carries a family-root kill-and-edit prompt through authorized forced reuse.

## Invariants

- Keep `wipe_agent_name_for_reuse()` as the low-level, scoped primitive: asking it to
  wipe a container directly must continue to be non-destructive by default.
- A confirmed forced reuse of a family base means replacing the newest generation of
  that SASE agent, so every concrete shell and dismissed bundle in that generation must
  be gone before the bare family name is relaunched.
- A forced family-member attachment such as `%id(!code, family=foo)` continues to target
  only `foo--code` and its descendants; it must not erase unrelated siblings.
- A populated clan is a rootless parallel group, not one replaceable agent. Direct
  forced reuse of the clan container must remain rejected; explicit clan members retain
  their current supported path.
- Cleanup and relaunch remain off the Textual event loop as tracked procs. The UI may
  mount the prepared prompt after cleanup settles, but it must never block the message
  pump waiting for filesystem or subprocess work.
- Overlap and partial prior cleanup are normal. Missing artifacts, bundles, or registry
  entries must be idempotent success, while real removal errors and residual
  reservations must still abort before spawn.

## Implementation

### 1. Promote family-aware forced-reuse cleanup to the shared agent-name layer

Extract the high-level family cleanup currently embedded in
`src/sase/bead/cli_work_name_cleanup.py` into the shared agent-name/forced-reuse
boundary under `src/sase/agent/names/`, with an exported operation that distinguishes a
concrete owner, a family container, and an unsupported clan container.

For a concrete owner, retain the current low-level wipe and error/residual checks. For a
family container, resolve `find_agent_family()` once to capture the newest generation,
wipe its concrete member names, rebuild the registry, and verify that the family base
and captured members are all absent. If the family has no discoverable members, prove it
is stale before using the existing `allow_stale_container=True` escape hatch. Tolerate a
member becoming absent while cleanup is in progress, because the ACE persistence proc
may have removed it first. Harden artifact-directory deletion so a concurrent
`FileNotFoundError` is treated like the already-supported missing bundle case, without
hiding permission or other I/O failures.

Route `wipe_names_for_forced_reuse()` through this shared operation so the authorized
ACE launch and `sase agent restart` execution boundary can replace a family root. Keep
its explicit clan-container error. Refactor bead-work cleanup to call the same family
primitive while preserving its existing `allow_container_skip` policy for populated
clans and its `ForcedReuseCleanupError` surface. This removes the duplicate algorithm
and prevents bead and ACE behavior from drifting again.

### 2. Serialize the kill-and-edit continuation after cleanup persistence

Extend the existing tracked cleanup-proc completion plumbing with a lightweight settled
continuation. Focused DONE dismissal, focused live kill, and marked/bulk kill-and-edit
should prepare and capture their prompts first, perform the optimistic in-memory
removal, submit the normal durable cleanup proc, and mount the prompt bar or prompt
stack only from the proc's settled callback. If proc submission itself is rejected,
invoke the continuation immediately after the failure path has released its in-flight
guard. If the proc exits with an error, retain the current error notification/recovery
refresh and still offer the already-prepared prompt only after the proc is no longer
capable of writing a late bundle; the launch-side family cleanup remains the final
fail-closed check.

Compose this continuation with the existing in-flight-set release callbacks rather than
adding a second cleanup mechanism or awaiting on the Textual message pump. Revalidate
any captured UI identity before destructive work as today; after the confirmed row has
been removed, the continuation should use the captured prompt and project context and
avoid a full agents-list rebuild merely to mount the editor.

### 3. Add registry-backed race and safety regressions

Add tests at three levels:

1. Agent-name cleanup tests seed plan/code family artifacts and dismissed bundles, then
   prove family-base forced reuse removes the newest family generation and its registry
   container while preserving unrelated agents and the enclosing clan. Cover partial
   prior deletion and a concurrent missing-directory removal. Retain tests showing that
   exact `--code` cleanup preserves siblings and direct clan-container reuse is refused.
2. The forced-reuse launch seam constructs the actual family-root kill-and-edit prompt,
   seeds a real registry family for its base name, submits an authorized durable launch,
   and asserts that the rewritten prompt reaches spawn only after the family reservation
   is gone. This test must not mock away the name-wipe operation that failed in
   production. Also assert cleanup failure prevents spawn and records the failed prompt
   as before.
3. Focused and marked ACE action tests use deferred tracked-cleanup callbacks to prove
   no prompt bar is mounted before settlement, exactly one prompt/pane per selected
   agent is mounted afterward, cancellation remains non-destructive, submission failure
   cannot strand the prepared prompt, and ordinary kill/dismiss actions without relaunch
   keep their current behavior.

Update affected bead cleanup tests to exercise the shared family primitive and retain
their current failure and populated-clan policies. No feature flag is needed: this
restores the already-confirmed kill-and-edit contract and remains guarded by the
existing trusted `allow_force_reuse` request authorization.

## Verification

Run `just install` before repository checks. Exercise the focused suites for agent-name
wiping, forced-reuse launch planning/seams, family-member and family-root relaunch,
marked kill-and-edit, cleanup-proc completion, and bead-work launch cleanup. Then run
`just check` to satisfy the repository-wide lint gates and diff-scoped tests. Use
`just check-full` through `/sase_monitor` only if scoped selection escalates, the shared
cleanup changes broaden beyond these modules, or the focused results expose cross-suite
coupling.

Acceptance requires the production-shaped `%id(!1, clan=sase-sq, ...)` family-root case
to pass whether old family state is fully present, partly removed, or already absent; no
replacement may spawn while a related cleanup proc can still write a dismissed bundle;
and exact-member, clan-container, error-reporting, prompt-stash, and event-loop
responsiveness contracts must remain intact.
