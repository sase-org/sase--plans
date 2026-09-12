---
tier: tale
title: Finish explicit bead-action integration and recovery
goal:
  Assigned stitch decisions remain explicit, authenticated, and truthful through every
  producer and recovery path.
size: medium
proposed_by: bbugyi200.athena.sase-zq.land
bead: sase-zq
create_time: 2026-09-12 13:46:35
status: wip
---

- **PARENT:**
  [202609/explicit_bead_action.md](https://github.com/sase-org/sase--plans/blob/main/202609/explicit_bead_action.md)
- **BEAD:**
  [sase-zq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zq/README.md)

# Finish explicit bead-action integration and recovery

## Objective

Complete the behavior promised by `sase-zq` but missed by its first three phases: every
non-interactive stitch producer must make an intentional bead decision, accepted
finalizer recovery must remain bound to the authenticated bead rather than ambient
process state, and a landed commit must not be reported as a successful `close` outcome
until the assigned bead is actually closed.

## Context

The landed implementation already provides the Rust-owned `close`/`keep` policy,
requires `-B/--bead-action` for assigned stitches, persists the action through stitch
checkpoints, closes only the primary assigned bead, and carries authenticated assigned-
bead context through finalizer declarations. Preserve those contracts.

The landing audit found these concrete gaps:

1. `src/sase/axe/run_agent_exec_plan_sdd.py` still invokes `sase stitch create` without
   `-B keep`, even though it creates a mechanical plan/publication commit. Audit the
   other in-repo programmatic stitch producers, including ACE restore, and make each
   producer's intended behavior explicit without changing pass-through wrappers that
   execute a user's own stitch arguments.
2. Finalizer stitch subprocesses inherit `SASE_BEAD_ID` from the ambient environment.
   The accepted declaration already contains an authenticated assigned-bead binding;
   dispatch, retry, repair, and resume must use that saved binding (or deliberately
   clear the variable when the saved binding is absent), not silently reassociate work
   with whatever bead happens to be ambient later.
3. `load_accepted_commit_declaration` does not use the exported core
   `validate_finalizer_assigned_bead_binding` policy when loading accepted context.
   Reject a changed or incompatible binding before running recovery.
4. Marker rescue currently treats a matching landed commit as overall success for both
   actions. A `keep` action may retain that behavior. A `close` action must additionally
   prove that its saved assigned bead is closed; if status cannot be read or remains
   open, return an actionable unsuccessful/recoverable result rather than claiming the
   finalizer succeeded.

## Implementation

### Make automated stitch decisions explicit

- Add `-B keep` to the approved-plan SDD commit invocation and assert the exact argv in
  its tests.
- Inventory direct `sase stitch create` and `sase stitch resume` subprocess producers.
  Give mechanical metadata, restore, or publication commits an explicit `keep` action
  where they can execute in an assigned-agent environment. Preserve user-selected
  actions in wrappers and interactive entry points.

### Bind finalizer execution to accepted context

- Thread the accepted `assigned_bead_id` alongside `bead_action` through initial
  dispatch, bounded follow-up, repair, and resume command construction.
- Construct the stitch subprocess environment from the saved binding: set `SASE_BEAD_ID`
  to that exact ID when present and remove it when absent. Do not infer a new
  association from the parent process during recovery.
- Validate the loaded accepted/latest declaration binding using the existing Rust-backed
  finalizer binding policy. Keep declaration authentication and error formatting in the
  current adapter boundaries; do not duplicate the policy in Python.
- Cover legacy accepted declarations deliberately: allow only combinations the core
  policy defines as compatible, and produce a clear recovery error for an absent or
  changed binding that cannot be proven safe.

### Make landed-marker rescue action-aware

- Pass the saved action and assigned bead into timeout/output-cap rescue.
- For `keep`, retain matching-marker rescue semantics.
- For `close`, query bead status through the existing repository-aware bead read
  adapter. Rescue only when the saved assigned bead is confirmed closed. Treat open,
  missing, ambiguous, or unreadable status as an unsuccessful result with enough detail
  to retry or repair safely.
- Preserve checkpoint/marker idempotence and primary-versus-linked-repository behavior;
  never close a linked repository's bead merely because it shares an identifier.

## Verification

Add focused tests that exercise behavior, not only plumbing:

- automated plan acceptance and every other changed producer emit the intended explicit
  action;
- a finalizer accepted under bead A still invokes stitch under bead A after ambient
  `SASE_BEAD_ID` changes to bead B, and an accepted declaration with no binding clears
  ambient association;
- compatible binding loads succeed and changed bindings fail before a stitch command is
  run;
- resume/retry preserves the checkpointed action, rejects conflicting overrides, and
  handles legacy checkpoints according to the core policy;
- marker rescue succeeds for `keep`, fails for `close` while the assigned bead is open,
  and succeeds for `close` only after that exact primary-repository bead is confirmed
  closed;
- multiple-repository tracking still closes the primary assigned bead and keeps linked
  tracking commits open.

Run the focused commit/finalizer/recovery and SDD tests, then the repository's required
`just check` verification. If a failure is unrelated and already has a durable task,
record corroboration rather than expanding this tale.

## Boundaries and landing handoff

- Prefer the existing Rust policy and Python facades. Change `sase-core` only if the
  current API is demonstrably insufficient; if so, update its wire/binding tests and
  dependency floor as one compatible change.
- Do not close `sase-zq`, alter its linked original plan's status, clean epic symbols,
  or run the final landing sequence in this implementation tale. Its land agent owns
  those steps after this child lands.
- The source templates for `sase_final` and `sase_git_commit` already contain the new
  bead-action guidance, but installed generated copies are stale. After this child is
  landed on a clean canonical branch, the resumed land agent must deploy generated
  skills with the project's generated-skill procedure; do not hand-edit managed output
  in this tale.
- The `sase-zq.1` mypy proposal is no longer reproducible: the current code types the
  setting value as `object | None`, and focused mypy passes. Do not create a duplicate
  task for it.

## Done when

- Every automated stitch producer audited here has an intentional action.
- Recovery uses and validates the accepted assigned-bead binding independently of
  ambient environment changes.
- No `close` finalizer path reports success from a commit marker alone while its
  assigned bead remains open.
- Focused tests and `just check` pass, apart from independently owned failures recorded
  through the normal task workflow.
