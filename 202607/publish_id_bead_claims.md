---
tier: tale
title: Publish `%id(bead=...)` claim transitions
goal: Runner-owned bead claims and releases are promptly published to their Git remote
  without sacrificing local lifecycle state on push failure.
create_time: 2026-07-26 10:24:27
status: wip
---

- **PROMPT:** [202607/prompts/publish_id_bead_claims.md](prompts/publish_id_bead_claims.md)

# Publish `%id(bead=...)` claim transitions

## Goal

Make every runner-owned bead transition initiated by the `%id(..., bead=<id>)` directive attempt to publish its
committed bead state to the configured Git remote, so GitHub promptly reflects both the pre-execution `claimed`
reservation and the `in_progress` promotion. Preserve local claim state when a remote is absent or a push fails, and
report publication failures clearly.

## Current behavior

- `bootstrap_agent_run()` extracts the directive's `bead=` value and, when the runner must wait, calls
  `claim_bead_for_waiting_agent()` before dependency or runner-slot admission.
- `claim_bead_for_waiting_agent()` mutates the canonical store under the shared store-write lock and calls
  `commit_bead_claim()`. That helper explicitly makes a local commit without pushing it. The `bead_claim_checks`
  reconciler uses the same path, so reconciled claims are also only local.
- Immediately before model execution, `_promote_bead_claim()` calls `claim_bead_for_agent_launch()`. For managed
  standalone SDD stores this uses `commit_sdd_store_files()`, whose push is controlled by the general
  `sdd.push_after_commit` setting; the setting may be disabled or asynchronous. In-tree stores deliberately remain
  ordinary workspace edits.
- The open `sase-9u` convergence epic adds periodic publication as a fallback, but it does not make the directive-owned
  claim transition itself publish immediately. This change should reuse the managed sync API so it remains compatible
  with `sase-9u.1`'s planned merge-integration implementation and complements `sase-9u.3` rather than replacing its
  periodic safety net.

## Implementation

1. Add a small claim-publication helper in the bead synchronization layer that runs the existing managed
   fetch/integrate/push operation for a bead-store path. Treat "not in a Git repository" and "no configured remote" as
   benign no-ops. Return or expose the push outcome so callers can distinguish a successful push, a benign skip, and a
   real publication failure without discarding the already-created local commit.

2. In `claim_bead_for_waiting_agent()`, retain whether `commit_bead_claim()` actually created a commit. Exit
   `bead_store_write_lock()` before invoking remote synchronization because the managed worker acquires the
   repository/store locks itself. When a commit was created, synchronously publish it before reporting that the claim
   was acquired. A same-agent no-op re-claim must not create another commit; it may skip publication because the
   periodic canonical sync remains the retry path for a previously reported push failure.

3. Apply the same post-lock publication behavior to `release_bead_claim_for_agent()`. Although release is the inverse
   transition, publishing it is required to keep GitHub from retaining a stale `claimed` owner after a runner dies
   before model execution.

4. In `claim_bead_for_agent_launch()`, separate the managed-store commit from its push: call `commit_sdd_store_files()`
   with its generic post-commit push disabled, then invoke the same synchronous claim-publication helper used by the
   waiting path. This makes the runner lifecycle invariant independent of the general `sdd.push_after_commit` preference
   and gives claim-specific diagnostics rather than relying on the generic SDD warning. Keep the existing in-tree
   policy: the mutation remains part of the agent's normal workspace commit rather than creating and pushing an
   unsolicited implementation-branch commit before the model runs.

5. Keep the lifecycle's established failure boundaries:
   - A local mutation or commit failure still prevents the launch-time promotion and is wrapped as a bead-claim failure.
   - A missing remote is a successful local-only outcome.
   - A remote integration/push failure preserves the local commit, emits a warning containing the bead ID, agent name,
     and managed-sync error/log detail, and does not release or roll back a claim that other local runners already
     observe.
   - Publication happens outside the foreground mutation lock to avoid deadlocking the managed sync worker and to let
     its existing conflict reconciliation handle concurrent remote bead commits.

6. Update comments/docstrings around `commit_bead_claim()`, the waiting claim lifecycle, and the SDD push configuration
   so they describe the split accurately: local persistence is performed under the mutation lock, while `%id(bead=...)`
   lifecycle callers synchronously publish afterward and periodic canonical sync provides retry/convergence.

## Tests

- Extend `tests/test_bead/test_claims.py` to assert that a changed waiting claim publishes exactly once after its local
  commit and after the store lock has been released; declined and same-owner no-op claims do not create or publish
  duplicate commits.
- Cover claim-release publication and verify that push errors warn while both helpers retain their current boolean
  lifecycle result and local state.
- Extend `tests/test_run_agent_runner_bead.py` to assert that managed launch-time promotions disable the generic commit
  helper's push and invoke the synchronous claim publisher exactly once, while in-tree claims retain their existing
  no-auto-commit behavior.
- Add or extend a fixture-repository integration test with a bare remote: acquire a waiting claim, verify a fresh clone
  observes `claimed` and its assignee, promote it through the launch helper, and verify the remote then exposes
  `in_progress`. Include release-to-`open` coverage for a pre-execution shutdown.
- Run the focused claim, runner-bead, managed-sync, and concurrent-writer tests, then run the repository-required
  `just check`.

## Acceptance criteria

- A successful `%id(..., bead=...)` waiting claim is committed locally and synchronously attempted against the remote
  before the helper returns.
- The corresponding `in_progress` promotion explicitly requests a synchronous managed-sync push for remote-backed
  standalone SDD stores, independent of the general SDD push preference.
- A released waiting claim is also published, preventing a stale remote owner.
- No remote or a failed push never destroys the local bead transition, and a real failure is visible in a targeted
  warning.
- Claim publication remains serialized safely with concurrent bead writers and managed integration, with regression
  coverage spanning a real Git remote.
