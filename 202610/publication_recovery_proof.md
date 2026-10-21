---
tier: epic
title: Finish Apollo recovery and prove bounded publication
goal: Complete the outstanding operational acceptance for sase-1nf.10.1.9 without
  bulk backlog draining or losing unpublished content.
parent_bead: sase-1nf.10.1.9
phases:
- id: apollo-replay
  title: Recover Apollo bob-cli unpublished owner content
  depends_on: []
  description: 'apollo-replay: execute the backup-first stale-replay helper, prove
    unique content reached the remote, and verify repeat apply is a no-op.'
  size: medium
- id: apollo-manifest
  title: Repair and verify Apollo's stale owner manifest
  depends_on:
  - apollo-replay
  description: 'apollo-manifest: repair the verified local-owner explicit-list mismatch
    through the normal transaction, verify remote payload references, and prove repeat
    repair is a no-op.'
  size: small
- id: athena-canary
  title: Measure and publish a formerly failing Athena request
  depends_on:
  - apollo-manifest
  description: 'athena-canary: profile the selected timeout hood, run the bounded
    production canary, and record exact queue, timing, payload, and remote evidence.'
  size: medium
- id: apollo-canaries
  title: Prove bounded publication for Apollo's repaired classes
  depends_on:
  - athena-canary
  description: 'apollo-canaries: run bounded canaries for bob-cli replay and sase
    manifest failures, preserving unrelated stopped requests and validating remote
    results.'
  size: medium
- id: completion-proof
  title: Prove fixture recovery and automatic publication after installation
  depends_on:
  - apollo-canaries
  description: 'completion-proof: run the existing real-Git recovery fixtures and
    verify a newly completed agent published automatically after installation.'
  size: medium
proposed_by: bbugyi200.athena.sase-1nf.10.1.9
proposed_with: codex/gpt-6-astra
create_time: 2026-10-21 07:38:41
status: wip
bead_id: sase-1nf.10.1.9.1
---

- **PROMPT:** [prompts/202610/publication_recovery_proof.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/publication_recovery_proof.md)
- **BEAD:** [sase-1nf.10.1.9.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nf/sase-1nf.10.1.9.1.md)

# Finish the recovery-proof phase

This child epic completes the remaining work of `sase-1nf.10.1.9`, the `recovery-proof`
phase in `plan:202610/publication_drain_fixes.md`. Read that plan,
`plan:202610/athena_apollo_audit_fixes.md`, the enclosing phase's notes, and
`file:explicit:1332edd16fbe66b228322d62` through audited commands. The continuation
baseline `file:explicit:eaaaade9836ab612d5ce7c78`, attached to the enclosing phase, is
additional planning evidence, not a recovery acceptance report.

The previous agent left four notes describing unfinished work, including two cold
binding builds of roughly nine minutes each. The user explicitly requires a child epic
when that remainder cannot clearly fit one agent. The production recovery, two-host
canaries, fixture verification, and automatic-completion proof are separate bounded
units. All implementation phases of the parent epic are already closed; reuse their
code. Do not repeat their implementation or restart the full measurement investigation.

The ancestor decision `gc_soak = no` remains final. No memory edits are required or
authorized. There are no new reviewer choices.

## Fresh evidence and resolved installation blocker

On 21 October 2026 at approximately 11:31 UTC, `sase version --json` reported these
clean installed revisions on both hosts:

- SASE `5639375bfd0b43bb49dfd761d96a4bf6668fabe6`.
- Core `461e7a1818903ac578c0c0bf0377a9c206bacf32`.

Git ancestry in the local checkout confirms that required SASE `5803ddfa0b` is an
ancestor of `5639375bf`. The installed CLI exposes `--retry-limit` and
`--retry-error-class`. The old notes' installation blocker has therefore been resolved;
do not request another update merely because the notes say the install was stale. Record
revisions again before mutation, including the code and loaded core used by any helper.
A later replay-module split is present in the planning checkout, so do not assume
checkout code and installed code are identical.

Fresh `sase doctor -C state.agent_publication_outbox -j` observations:

| Host/project   | Active | Quarantined | Retired | Clone ahead / behind |
| -------------- | -----: | ----------: | ------: | -------------------- |
| athena/sase    |      3 |        3389 |       0 | 0 / 0                |
| apollo/bob-cli |      3 |         388 |       0 | 1 / 566              |
| apollo/sase    |      3 |         109 |     284 | 0 / 7                |

Athena's totals across projects were 3395 requests, 4 active, 3389 quarantined, and 2
retired. Its error histogram included 2919 timeout, 440 lock-busy, and 19 index-lock
errors. Apollo's totals were 787 requests, 6 active, 497 quarantined, and 284 retired;
its histogram included 390 rebase-conflict and 298 manifest-mismatch errors. Complete
problem records still contained 388 bob-cli errors naming `2abeff3f9` and 298 sase
errors naming `bbugyi200.apollo.03`.

Ahead/behind values use cached upstream refs. Historical row errors are not proof of
current manifest invalidity, and the doctor reporting zero owner-manifest problems is
not remote manifest acceptance. Reinspect the actual targets and record a fetched remote
revision when proving publication. Neither production recovery nor a bounded retry was
performed by the planner.

## Shared execution and evidence contract

1. Read `tailnet`, `sase_artifacts`, and `lint_and_test` through `/sase_memory_read`.
   Reach Apollo with the documented login-shell invocation or an explicit executable
   PATH; a bare noninteractive `ssh apollo 'sase ...'` does not find `sase`. Use
   `/sase_repo` and `sase repo open` for every inspected repository, including the
   appropriate project's `agents` sidecar. For example,
   `sase repo open agents -p bob-cli -r "Inspect the publication recovery target"`. Use
   only the returned path; confirm it is the hidden machine clone. Run helpers from an
   opened checkout, never a primary nested sidecar. Do not manually locate, clone,
   reset, or web-fetch another repository.
2. Use the existing recovery helper, manifest repair, bounded retry, and validation
   APIs. `tools/agents_publication_recover_replay` uses `-H` for expected HEAD;
   `tools/agents_publication_profile` takes `--hood`, not `-H`. The publication commands
   resolve machine targets through `resolve_sync_targets`. Do not hand-edit outboxes,
   manifests, request histories, or progress files.
3. Use `/sase_monitor` for long profiles, repairs, builds, tests, and waits. Prepare the
   continuation prompt before starting the command and wait for the handoff command
   itself to exit. Do not leave an inline background process when the turn ends. Use
   `/sase_gate` for `sase update` only if fresh evidence establishes that an additional
   installed fix is necessary. No global install recipes, raw cargo, manual commits, or
   direct agent spawning.
4. Serialize production work in the dependency order above. Even the two machines share
   the sase agents remote. Keep a canary command ledger per host/project in the
   evidence: begin with limit 1 and allow at most one subsequent batch of at most 25 if
   needed to cover the historical class. This is the entire child epic's budget, not a
   fresh allowance for every continuation. Never retry all, drop requests, increase the
   120-second bound, or deliberately interrupt production.
5. Existing active requests win over stopped-row error filters. A first limit-1 call may
   publish an active row without exercising the historical class. Record actual selected
   keys and prior state. Do not claim repaired-class acceptance unless the evidence
   identifies a previously failing selected row. Do not enlarge a batch to circumvent an
   active-row regression. An interruption may resume the exact durable selection; it
   does not authorize selecting successive batches of new work.
6. For each mutation, capture UTC times, exact command and exit status, host/code/core
   revisions, selected logical keys, and before/after request fields: revision,
   lifecycle, attempts, prior error/class, and prompt obligation. Record the complete
   preexisting key set and compare unselected rows. New unrelated enqueue activity can
   change total counts, so explain concurrent additions separately. Capture timing log
   entries, Git operation counts, payload files/bytes where applicable, remote SHA,
   manifest validation, and representative remote paths. Save the raw evidence as
   explicit artifacts attached to the child phase; do not rely on ephemeral stdout.
7. Validate remote content at a recorded remote revision through the permitted repo
   access path and existing `load_validated_publication` API. Verify the checked tree
   corresponds to that fetched revision; validating a modified local worktree alone does
   not prove publication. Read contextual sidecar artifacts with `sase artifact read`,
   not direct file reads.
8. Stop production retries on a regression. Preserve backups and durable pending
   obligations. Fix a bounded defect in the responsible component when necessary, with
   shared policy in sase-core and a regression test; do not weaken preservation or
   verification. A substantive additional implementation need must be recorded
   explicitly, not hidden behind a claimed passing canary. Record unrelated findings as
   `PROPOSED FOLLOW-UP:` notes on the worker's phase; workers never create beads.
9. No source changes are expected. If needed, read the applicable domain memories and
   run `sase tool run check` in each changed repo, with the core pin/binding updated
   when required. Never run `check-full`. A failure independently reproduced on the
   clean base is recorded with evidence and any known task; it does not keep completed
   work open. Use `/sase_final` for normal turn completion. Before closing each child
   phase, run `sase bead epic-symbols` and resolve or re-key any remaining exemption.

## Phase apollo-replay

This phase owns only Apollo bob-cli stale-replay recovery and its content proof. It does
not revive stopped outbox requests; that belongs to `apollo-canaries`.

1. Read closed bead `sase-1nf.10.1.8` and the current Contribute runbook
   `docs/contribute/agents-sidecar-sync.md#stale-replay-recovery`. Its real-Git fixture
   proof is already recorded; the final fixture phase supplies a current run.
2. Open Apollo's bob-cli agents target, capture full HEAD/upstream and worktree state,
   and verify that the stale unpublished commit is still the problem. If an earlier
   continuation already repaired it, validate that recovery and reuse its backup; never
   reset a healthy clone to the historical SHA.
3. Run the helper without `--apply`, using the observed full HEAD and a durable,
   phase-specific backup directory outside the clone. Do not put the only backup in
   disposable scratch. Preserve the bundle, staged/unstaged patches, owner snapshots,
   chats/prompts, pending archive objects, checksum manifest, revisions, and progress.
   Require `stage=verified`, verified checksums and bundle, and empty `unknown_paths`
   and `refused_divergences`. The helper holds the existing sync lock across fetch,
   recheck, backup, and mutation; do not bypass its HEAD-drift guard.
4. Apply using the identical backup and expected HEAD. The helper regenerates derived
   pages over upstream, merges canonical owner content through existing policy, and uses
   `sync_project_locked` for commit/push. If interrupted after `reset` or `replaying`,
   resume the same backup. Do not run an ad-hoc commit or bare reset.
5. Inventory the locally unique `bob-cli-7c` runs/session members and their content
   digests before apply, then prove their published membership and bytes afterward.
   Verify pending objects are published intact or restored as pending inputs with
   explicit durable retry obligations. A backup by itself is insufficient. Also prove
   upstream-only content survives and remote manifest references validate.
6. Repeat apply with the same backup; require `already_complete` without another
   publication. Record remote SHA, backup identity/checksums, representative paths,
   preserved run/object counts, and remaining outbox counts as the phase deliverable.

## Phase apollo-manifest

This phase owns only Apollo sase's local-owner manifest repair, using the already landed
explicit repair command. It does not consume the canary retry budget.

1. Reinspect `bbugyi200.apollo.03` on the opened hidden sase agents clone. Save the
   original manifest/snapshot/payload digests and identify the redundant explicit-list
   mismatch. If it has already been repaired, verify that state and remote ancestry.
2. Run `sase agent sync --project sase --repair-manifest --json` on Apollo through a
   monitor. The command itself uses the normal locked transaction and push. It must
   verify owner/project/hood identity, snapshot digest, safe paths, and every referenced
   payload's size/digest before normalizing the redundant list. Do not use digest repair
   to bless corrupt data, touch a foreign owner, or invoke full reconciliation.
3. Require successful remote publication, unchanged canonical payload bytes, a validated
   slim owner entry, and a second repair that makes no change. Compare foreign owner
   state at the relevant revisions. Save both repair results and remote validation
   evidence. If corrupt underlying content is found, preserve it and resolve that
   condition before treating this class as repaired.

## Phase athena-canary

This phase owns the scoped/full comparison and Athena's sole canary budget.

1. Refresh the doctor summary and capture the complete local queue state. Use the
   existing Rust selector without mutating the queue to understand the active-first
   selection and identify a stopped timeout row within the permitted batch. Do not
   discard active rows or invent a selection path bypassing the CLI's policy.
2. Profile the exact historical timeout hood with
   `tools/agents_publication_profile --project sase --hood <hood> --json` under a
   monitor. The profile is read-only. Record inventory/planning seconds, runs and bytes
   read, Git calls by operation, and payload files/bytes. Compare with the archived
   baseline: 209.84 seconds inventory, 9903 Git calls, approximately 452 MB chat reads,
   and roughly 14 seconds / 24000 files / 60 MB one-hood planning. State the different
   measurement dates/populations; do not rerun expensive full inventory just to select a
   hood.
3. Run
   `sase agent sync --project sase --retry-quarantined --retry-limit 1 --retry-error-class timeout --json`.
   If successful but the active-first selection did not cover a stopped timeout row, use
   the one allowed batch of at most 25. Profile any different stopped hood actually used
   for the acceptance comparison.
4. Prove at least one identified formerly timeout-failing request completed within the
   ordinary 120-second drain bound, including push, remote manifest validation, exact
   requested revision/page, and prompt obligation. Preserve stage timing and selected
   key diagnostics. Unselected stopped rows must retain lifecycle and attempt history.
   Record final remaining counts without continuing into the bulk drain.

## Phase apollo-canaries

This phase owns both Apollo host/project canary budgets and consumes the repair evidence
from the first two phases.

1. For bob-cli start with
   `sase agent sync --project bob-cli --retry-quarantined --retry-limit 1 --retry-error-class rebase_conflict --json`.
2. For sase start with
   `sase agent sync --project sase --retry-quarantined --retry-retired --retry-limit 1 --retry-error-class manifest_mismatch --json`.
3. Respect active-first selection and the shared limit-1/one-batch-at-most-25 rule.
   Verify one historical request from each repaired class/project, not simply an
   arbitrary successful active row. Never revive a retired class whose cause remains
   unresolved. Treat a no-match result as no-op evidence, not permission for full sync.
4. For each operation record the shared evidence contract, remote validation and
   preservation of unrelated rows, plus timing/Git counters. Obtain scoped payload
   metrics for the hoods used. Confirm bob-cli unique-content preservation still holds
   after normal drain cleanup and that the repaired sase manifest remains valid. Save
   explicit evidence artifacts and precise bulk-drain handoff commands.

## Phase completion-proof

This phase supplies current local fixture proof and a real automatic-publication
observation; it does not perform further manual production retry batches.

1. Use the existing tests in `tests/agents_sync/test_publication_replay_recovery.py`.
   Verify the workspace binding matches its pin. If preparation/build is required, use a
   monitor from the start; do not repeat the earlier abandoned inline cold build. Run
   the focused fixture file through the normal test recipe. In particular, capture
   passes for `test_report_only_writes_a_verified_backup_without_resetting`,
   `test_apply_keeps_both_sides_unique_content_and_pending_objects`, and
   `test_interrupted_apply_resumes_and_second_apply_is_already_complete`. Preserve
   HEAD-drift and same-run divergence refusal coverage. Do not create redundant tests.
2. Identify an existing agent that completed after the verified installation and
   published via its automatic finalizer/commit path. This epic's earlier workers may
   supply a suitable observation only if they actually triggered that path. Correlate
   completion timestamp, requested primary revision, publication timing/acknowledgment,
   and remote run/page/prompt evidence. Mere absence from an outbox or a manual canary
   publication is insufficient.
3. If no qualifying completion is available, use the approved `/sase_run` launch path
   for one minimal publication-eligible canary agent, including a concrete harmless
   deliverable that invokes normal host completion. Do not spoof a marker or run a
   direct provider subprocess. Wait through a monitor and inspect the actual result.
4. Attach fixture command/results and automatic-publication evidence to the phase, with
   fresh per-host backlog summaries. A naturally occurring completion is preferred to a
   new launch, but missing proof must not be labeled passed.

## Land-agent acceptance, artifact, and enclosing-phase closure

The child land agent consolidates the completed phases; it does not repeat production
repairs or spend fresh canary budgets. Open the research sidecar through `/sase_repo`
and publish a research report through host-owned finalization, attaching its canonical
reference to `sase-1nf.10.1.9`. Include links to the immutable raw evidence, the
scoped/full comparison, durable backup locations and checksum evidence, remote SHAs and
representative paths, and this five-check acceptance table:

1. A named formerly failing request publishes within 120 seconds.
2. Remote manifests and all referenced files agree for the repaired/published state.
3. Repeating the recovery is a no-op, with current fixture and production evidence.
4. An interrupted local real-Git publication resumes without content loss.
5. A newly completed agent publishes automatically after installation.

Document per-host/project remaining backlog counts and exact bounded commands/error
classes suitable for `sase-1nf.11`. Bulk draining remains that phase's work. Carry
forward the enclosing phase's existing follow-ups: Apollo job/tailnet remote access
failures, and the known tagged-commit prompt-obligation test failure recorded in
`sase-1nf.10.1.8`; do not duplicate them or treat them as passing product acceptance.
Triage child `PROPOSED FOLLOW-UP:` notes through the usual land workflow.

The user explicitly delegated closure of the enclosing phase to this child epic's land
agent. After all child work is closed and all five checks have evidence, complete this
child epic through its normal land workflow, then run
`sase bead epic-symbols sase-1nf.10.1.9`. Resolve every remaining symbol or re-key it to
a still-open appropriate owner. Close **only the enclosing assigned phase** with
`sase bead close sase-1nf.10.1.9 --note "<verified acceptance and research reference>"`.
Do not close `sase-1nf.10.1`, `sase-1nf.10`, `sase-1nf`, `sase-1nf.11`, or `sase-10x`.
Individual child phase workers close only their own phase.
