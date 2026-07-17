---
tier: tale
title: Transactional SDD sidecar recovery for approved epic launches
goal: SDD sidecar refreshes never leave a checkout mid-Git operation, writers refuse
  structurally unsafe repositories, and an approved epic can be resumed safely after
  concurrent bead-store updates without corrupting its event streams.
---

# Plan: Transactional SDD sidecar recovery for approved epic launches

## Incident and product contract

The Telegram approval path completed successfully: the neutral `EpicApproval` gate recorded the `epic` choice in
detached launch mode, and the runner only failed afterward while executing the canonical `sase bead work` handoff. The
plans sidecar in the claimed workspace had one local bead commit while its remote had advanced. The sidecar refresh ran
`git pull --rebase`; the event stream and derived issue index conflicted, and the refresh logged the failure but left
the checkout in an active rebase with unmerged files and a detached `HEAD`. Later approval and plan-archive writers
treated that checkout as usable, committed selected paths inside the incomplete rebase, failed to push from detached
`HEAD`, and finally handed conflict-marker text to bead validation. The resulting JSONL parse error was therefore
downstream damage, not the initiating failure and not a Telegram transport problem.

Make repository integrity an invariant at every SDD sidecar boundary:

- A refresh that starts a Git operation must either integrate successfully or restore the exact pre-operation repository
  state before returning. No timeout, network error, unsupported conflict, or semantic-merge failure may strand a
  rebase, merge, cherry-pick, unmerged index, or detached launch checkout.
- Concurrent changes limited to supported bead event/index files use the existing semantic bead conflict resolver so
  both histories survive. Unsupported or mixed conflicts are aborted and reported without guessing.
- Existing best-effort refresh behavior may tolerate a temporarily unavailable remote when the local checkout remains
  structurally healthy; it must never equate “pull failed” with “repository is safe to write.” Strict materialization
  must surface an unrecoverable structural state with an actionable path and preserved data.
- Every SDD writer validates repository structure while holding the store write lock, before staging anything. A
  pre-existing operation, unmerged index, or inappropriate detached `HEAD` fails closed and leaves all files and refs
  unchanged.
- Plan approval remains provider- and surface-neutral. ACE, Telegram, mobile, and CLI approval all continue through the
  same gate result and canonical epic launcher; no Telegram-specific branch or retry is introduced.

## Transactional sidecar integration

Factor the local fetch/rebase transaction used by clone refresh and the managed bead sync worker behind one SDD-owned
integration primitive. It should report typed outcomes for success, remote-unavailable-but-healthy, repaired bead
conflicts, aborted unsupported conflicts, and unrecoverable repository state. Keep network fetch outside the local write
critical section, then hold the existing store lock across repository-state inspection, rebase, semantic resolution,
continuation, and abort/rollback. Reuse the existing bead event reducer rather than adding a textual merge policy.

Record the starting branch/ref, `HEAD`, operation markers, unmerged paths, and worktree/index state needed to prove the
postcondition. When a rebase begun by SASE fails, always attempt `rebase --abort`, verify that the repository is
attached and has no unmerged entries or operation markers, and include both the primary and rollback failures if
restoration cannot be proven. A checkout already mid-operation must not be casually aborted because it may contain work
from another actor: strict callers fail with recovery guidance, while launch-scoped clone preparation can preserve or
replace the managed clone through its existing trash/reclone lifecycle. Preserve intentional local commits and untracked
files in normal refreshes.

Replace the raw best-effort `pull --rebase` paths with this transaction, including sidecar-kind materialization and the
legacy separate-repository workspace refresh. The managed sync worker should retain its fetch/rebase/push behavior and
structured log events while consuming the shared integration result, so foreground refresh and deferred push cannot
drift into different conflict or abort semantics.

## Fail-closed writes and epic handoff

Add one repository-health guard shared by SDD prompt/plan commits and bead-state commits. Run it after acquiring the
store write lock and before `git add`; require a valid worktree on the expected branch, no
rebase/merge/cherry-pick/revert/bisect operation, and no unmerged index entries. Return a diagnostic that names the
repository and blocking state without including sensitive remote credentials. Preserve benign no-op behavior for
local/non-Git stores, but do not let best-effort auto-commit wrappers convert a structural failure into a successful
commit result.

At accepted-plan handoff, distinguish an optional prompt-snapshot push failure from an unusable plans store. The former
may remain a warning because the canonical plan is durable; the latter must stop before `sase bead work` and produce one
actionable `epic_launch_failed` notification/resume command. In plan-file mode, validate repository health before
copying the approved plan into the sidecar or opening `BeadProject`, so conflict-marker JSONL is never parsed and no
partial archive, bead DAG, or `bead_id` link is created. Retain the existing rollback guarantees for failures after epic
creation begins.

## Regression coverage and incident recovery

Build realistic temporary bare remotes and two sidecar clones to reproduce the incident: both mutate different records
in the same bead stream, one pushes, and the other refreshes with a local commit. Verify semantic convergence, attached
`HEAD`, valid reduced JSONL, a clean index, no operation directories, and preservation of both updates. Add unsupported
plan-file conflicts and injected fetch/rebase/continue/abort failures; every case must prove either success or
restoration to the captured starting state. Add a legacy pre-poisoned checkout case to prove strict materialization
fails before any write and gives a usable recovery message.

Cover each writer with an in-progress-operation and unmerged-index test, asserting no staging, commit, push, or file
mutation. Add an end-to-end neutral-gate/runner regression in which an approved epic encounters a concurrent bead
update: the plan is archived once, its epic and phases are created once, phase agents launch once, and the
notification/result metadata remains surface-independent. Also assert that an unrecoverable store stops before the
launcher and reports the canonical home-plan resume command rather than a poisoned workspace path.

After the code is fixed, use the preserved home archive for `multi_parent_fork.md` to exercise a dry run from a healthy
workspace, confirm there is no existing `bead_id`/epic from the failed attempt, then run the canonical resume command
once. Verify the new epic, phase dependencies, launched agents, sidecar Git health, remote convergence, and absence of
duplicate beads. Do not hand-edit or discard the failed workspace clone; let the managed workspace trash/reclone path
preserve it for forensics.

Run `just install`, focused SDD clone/sync/commit tests, focused plan-approval and plan-file epic tests, and
`just check`. Keep unrelated baseline failures separate from regressions. This work does not require changes to
Telegram, Rust editor core, long-term memory, agent instruction shims, or generated skills.
