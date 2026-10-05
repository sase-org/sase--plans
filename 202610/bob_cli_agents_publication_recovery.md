---
tier: epic
title: Restore bob-cli agent publication and publish the missed pages
goal: Restore compatible agent publication and verify that every publication-eligible
  bob-cli agent and session available from its publishing machines is present on GitHub,
  with recovered requests and deferred prompts accounted for.
phases:
- id: manifest_compatibility
  title: Accept intact legacy session manifests through the Rust core
  size: medium
  depends_on: []
  description: 'manifest_compatibility: narrowly accept the historical family-only
    file set, preserve integrity checks, expose the Rust policy to Python, and prove
    cross-owner publication.'
- id: publication_recovery
  title: Add explicit retired-request recovery with correct completion checks
  size: medium
  depends_on:
  - manifest_compatibility
  description: 'publication_recovery: add project-scoped retired-request retry, preserve
    deferred prompts, recognize session pages, and test acknowledgment only after
    successful publication.'
- id: bob_cli_backfill
  title: Run bob-cli recovery and prove remote completeness
  size: medium
  depends_on:
  - manifest_compatibility
  - publication_recovery
  description: 'bob_cli_backfill: use the verified host/core versions, recover bob-cli
    on each relevant owner machine, reconcile missing pages, and record remote and
    repeat-run evidence.'
proposed_by: bbugyi200.apollo.4s.f1
create_time: 2026-10-03 12:41:58
status: done
bead_id: sase-1fs
---

- **PROMPT:** [prompts/202610/bob_cli_agents_publication_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/bob_cli_agents_publication_recovery.md)
- **BEAD:** [sase-1fs](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1fs/README.md)

# Restore bob-cli agent publication and publish the missed pages

## Outcome and scope

Fix the regression tracked by `bead:sase-1fm`, then actually publish the missing
`bob-cli--agents` pages. Shipping a validator change, clearing a queue, or observing Git
status `ready` is insufficient. Completion requires an identity-based comparison of the
expected publication set against a verified remote commit.

Use an epic because the shared Rust/Python compatibility change, durable recovery
operation, and live multi-machine recovery have distinct deliverables and deployment
dependencies. Each phase is bounded direct implementation or operational work. The
dependency order is intentionally serial; no live repair starts on an unverified build.

The product fix applies to the shared publisher. Live mutation is scoped to the
`bob-cli` project (`gh_bobs-org__bob-cli`) and its configured agents sidecar. Preserve
the existing publication eligibility rule: a primary-commit-bearing hood is published
with its available members; a prompt-only agent is not automatically owed an agent page.
Include previously published snapshots and all outstanding publication requests when
reconciling the expected set. Do not expand this into terminal-state refresh triggers
(`sase-1cc`), manifest capacity work (`sase-10x`/`sase-11o`), or agents import.

## Evidence and implementation map

Read these durable sources through `sase artifact read` / `sase bead read`:

- `file:explicit:a9a43980633a6f1d01e63f14`: October 3 diagnosis, owner counts,
  reproduction, and recovery caveats.
- `bead:sase-1fm`: the existing defect and its related reports.
- `bead:sase-11o`: its implementation phase is closed but its separate Athena recovery
  phase remains in progress; avoid overlapping global recovery work.

The September 25 commit `7cb8359534d90a04aa09d012bc4c6b2190cb2ce9` added canonical
`sessions/<global-name>.md` paths to `hood_file_set()`. Older full manifests list only
the corresponding `families/` pages. Their snapshots are intact, but
`load_validated_publication()` compares every owner's recorded file list against the new
list. An untouched legacy entry prevents unrelated publication.

The earlier audit found 56 affected hoods: Apollo 2, Athena 52, and `kellys_mbp` 2. At
this plan's read-only check on October 3, the last sidecar commit touching `agents/` was
still `569173f07a12f29d4aea644ea55197bc108e9c63` on September 25. The status check
reported 371 stopped requests on Apollo: 370 retired and one quarantined, all with the
same manifest mismatch. These are observations, not fixed recovery totals; capture fresh
counts before execution.

Relevant implementation:

- `src/sase/agents_sync/publication_validation.py`: canonical files, snapshot
  identity/digest checks, and scoped run-file verification.
- `publication_planning.py`, `publication_snapshot.py`, and `rendering.py` in that
  package: merge prior runs, plan owner writes, and regenerate browsing pages for all
  validated snapshots, including foreign owners.
- `publication.py` and `inventory*.py`: targeted publication, full reconciliation,
  live/dismissed records, commit-only recovery, and local-owner eligibility.
- `publication_outbox_{models,operations,store,serialization}.py`,
  `commit_publication_transaction.py`, and `git_sync.py`: durable requests, retirement,
  locked transactions, and acknowledgment.
- `prompt_archive/deferred.py`: deferred prompt regeneration. Its current missing source
  path returns false without producing a failure; recovery must account for this case
  explicitly instead of assuming a prompt was restored.
- `src/sase/main/parser_agent_sync.py`, `parser_root_args.py`, and
  `src/sase/agents/cli_sync.py`: CLI declarations, flag routing, and mode checks.
- `src/sase/sase_agent.py`: existing canonical run/session identity and page helpers.

`--repair-manifest` restores omitted hoods, not old file lists. `--repair-digests`
re-signs changed payloads, which is not this failure. `--retry-quarantined` excludes
terminal requests, re-enqueue preserves terminal state, and `--drop-retired` destroys
the request rather than recovering its work. None is a standalone solution.

## Shared constraints

Open `sase-core` with `/sase_repo` and work only in its returned checkout. New shared
path-validation, retry-selection/state-transition, and completion-decision policies
belong in `sase_core` with typed wire APIs and registered `sase_core_rs` bindings.
Python remains the thin adapter and owns existing filesystem, registry, Git, lock, and
CLI orchestration. Keep this a focused publication-policy addition; do not port the
entire publisher or outbox store.

Read applicable repository instructions and reference memory before implementation. Use
existing snapshot/run/name validation, feature flag, and bounded publication batching
contracts. Preserve owner/project/path checks, snapshot digests, payload digests/sizes,
missing-file failures, and existing scoped run verification. Do not resolve this by
accepting arbitrary file subsets, skipping corrupt hoods, dropping the `files` key from
foreign manifests, raising limits, or re-signing payloads.

Declare every changed repository to the host finalizer. The host commits/pushes the
linked core first and updates `sase-core-revision.txt` before the primary commit; verify
the resulting pin includes every newly used binding. For an already-landed core commit,
use the documented pin ratchet. Never invent an uncommitted core SHA or rely on a
locally installed extension that CI's pin cannot build.

## Manifest compatibility

1. Add a small Rust API that derives the canonical explicit file set from typed,
   validated snapshot facts and classifies a manifest as current, supported legacy,
   slim, or invalid. Make `hood_file_set()` and the changed comparison delegate to this
   policy through a thin Python facade. Register and test the binding.

2. Define the compatibility rule exactly. Let `C` be the current canonical set,
   including run payloads, snapshot and README paths, and both page paths for each
   recognized session container. Let `L` be `C` minus only the `sessions/` paths derived
   from those actual containers. Accept an explicit list only when it equals `C` or `L`;
   `L` still requires all matching `families/` paths. A hood with no session containers
   has no exception. Keep the existing sorted/unique and safe path requirements. Reject
   arbitrary partial migrations, unexpected paths, omitted payloads/READMEs, and session
   paths for nonexistent containers. Keep slim entries with no explicit list supported
   under their existing contract.

3. Preserve both historical `family` and current `session` container spellings and
   `family_count` decoding. Do not infer legacy format from machine names or dates.
   Validation is read-only. Normal publication writes the current format (or the
   existing slim format according to `slim_agents_manifest`), regenerates canonical
   session pages and permanent family redirect stubs, and leaves foreign manifests,
   snapshots, and run payloads untouched. Existing foreign derived browsing pages may be
   regenerated by the normal renderer.

4. Follow `sase_flags.md` for the compatibility transition: create the sunset flag
   `agents_session_manifest_compat` through `sase flag new`, not a hand-created bead.
   Enabled accepts the exact supported legacy set; disabled retains the pre-fix strict
   comparison. Author the removal condition around successful mixed-owner recovery,
   integrity regression tests, and verified publisher rollout; eventual removal deletes
   the disabled strict-only branch and retains compatible reading. Test both states and
   the existing slim-manifest states. Do not toggle `slim_agents_manifest` globally as
   an incident workaround.

5. Add Rust/binding tests for the policy and Python integration regressions in
   `tests/agents_sync/test_publication_manifest.py` and adjacent rendering tests. Seed a
   real legacy explicit manifest with valid snapshot and run digests, then publish a
   different hood of the same owner and another owner. Prove canonical pages and
   redirects appear and foreign source bytes remain identical. Cover full/current/slim
   manifests, multiple sessions, no sessions, repeated sync, and independently corrupted
   snapshot digest, identity, payload bytes, and file list. The legitimate legacy
   fixture succeeds; each genuine corruption still fails.

Exit: a current publisher can publish beside every diagnosed legacy shape without
weakening integrity or requiring the other machines to rewrite their manifests first.

## Publication recovery

1. Add `sase agent sync --retry-retired` with short alias `-t`. Require an explicit
   `--project/-p` for this new operation. It revives that project's retired
   agent-publication requests once, then runs full reconciliation and the normal queue
   drain. It may be combined with `--retry-quarantined`; reject combinations with
   `--check`, `--drop-retired`, and repair-only modes before any mutation. Follow CLI
   memory for sorted/helpful options and update flag routing, JSON results, and user
   documentation. Report the selected count and prior failure classes so a retry is
   auditable. Preserve existing commands' semantics.

2. Put retry selection and state transition in Rust; execute the transformation inside
   the existing Python outbox lock and atomic-write operation. Keep logical keys
   `(global_agent, primary_revision)`, creation time, hood identity/digest, ordering,
   and deferred work. Reset attempts, last error, terminal reason, and
   terminal/quarantine markers for the selected terminal rows. Leave active and
   quarantined rows alone unless their separate retry option was selected. Do not touch
   other projects or the Referenced By terminal queue. Do not broadly stop retiring real
   format failures or retry terminal rows automatically on every sync.

3. Route recovered requests through deferred-prompt restoration and the existing locked
   pull/plan/apply/commit/push transaction. Fix completion checks required for recovery:
   `_publication_request_materialized()` currently only checks
   `agents/<name>/README.md`, while session agents publish to `sessions/<name>.md`. Use
   validated snapshot identity and existing canonical helpers to distinguish runs,
   session containers, and historical request spellings. Require the proper page and the
   requested primary revision's run or session association, not merely an unrelated
   preexisting path. Avoid relying solely on the current live registry for historical
   agents. Share the decision through the Rust policy boundary.

4. Account for each deferred prompt as restored, already durably archived, not
   applicable with evidence, or unavailable/failed. A missing local source alone does
   not prove there was no prompt obligation. Consult existing archive records before
   reporting a loss, and preserve a request with unresolved prompt work. Keep any
   recoverable existing chat/prompt bytes when merging historical runs; do not replace
   richer archived content with commit-only placeholders.

5. Acknowledge only fulfilled requests after the transaction has successfully pushed, or
   proved equivalent payloads already exist upstream. Lock contention, planning errors,
   deferred-prompt errors, failed pushes, non-fast-forward retries, and interruption
   must leave outstanding work discoverable and retryable. Keep existing bounded
   rollback and batching rather than adding direct Git or JSON repair writes. Preserve
   pre-recovery failure details in the recovery report before resetting row state.

6. Add meaningful tests around `test_publication_outbox.py`, `test_cli.py`,
   `test_git_sync_outbox.py`, and `test_commit_publication_queue.py`, plus the new
   core/binding policy. Exercise a retired legacy-mismatch request end to end to a local
   bare remote, with a deferred prompt, a session request, and more than one primary
   revision. Verify remote page and archive content, unrelated row/project preservation,
   negative completion evidence, failed push recovery, repeat retry idempotence,
   preservation of concurrently enqueued requests, and safe behavior when the original
   artifact source is unavailable. Bound the retry operation to the selected rows
   observed under lock; do not run an open-ended loop reviving requests that fail again
   while the command is still running.

Exit: there is a supported, tested way to revive the stopped bob-cli requests and
discharge their actual work without deleting evidence or falsely acknowledging pages.

## Live backfill and acceptance

This phase performs the requested publication; it is not merely a runbook handoff.

1. Confirm both code phases have passed checks and that the exact host/core revisions
   are available to the command performing recovery. Use supported update/install
   tooling on each participating machine and verify loaded module versions/paths and
   `sase core health`. Verify ongoing publishers will also use the fixed code; refresh a
   stale publishing process through its supported mechanism if needed. Do not assume
   editing an isolated checkout updates the installed CLI.

2. Open each repository through `/sase_repo`. For a read-only archive inspection from
   another project's workspace, the established resolver is
   `sase repo open agents -p gh_bobs-org__bob-cli -w 0 -r "Audit bob-cli recovery"`. Use
   returned paths only. Use the supported sync target and host-owned hidden clone for
   writes, never hand-edit a human's primary sidecar. Read artifact bodies through
   `sase artifact read`; use metadata and identities for bulk accounting.

3. Establish a baseline before mutation. Record the remote branch and SHA, local
   cleanliness, manifest/snapshot hashes, owner identities, all queue rows and failure
   classes, primary commit-history cutoffs, and source availability. Save a durable
   report/backup of the queue and relevant metadata through the artifact workflow.
   Preserve access to the pre-recovery Git commit; never reset production history to it.
   Do not run `--drop-retired` or prune any source artifacts.

4. Derive expected pages independently of queue size. On each relevant owner, combine
   the normal inventory's eligible hoods and all their members, retained snapshots, and
   outstanding requests, cross-checking primary commit footers so dropped/unindexed
   requests cannot disappear from the accounting. Include local live and dismissed
   sources, prior archived runs, and supported commit-only recovery. Track each run's
   global identity, session, relevant revisions, required page paths, and recoverable
   prompt obligation. Diagnose exclusions and session commit history with no available
   member; never silently remove them from the expected set. Do not fabricate missing
   transcripts or member identities.

5. Start on Apollo. Preflight the fixed validator against the current archive, then run
   `sase agent sync -p bob-cli --retry-retired --retry-quarantined --json` using the
   tested executable. Inspect every outcome and retained failure, not only the exit code
   or `ready`. Let normal reconciliation backfill all eligible hoods, including those
   with no surviving active request. Correct any recovery-path defects through tested
   changes before continuing; do not normalize malformed source data merely to make a
   check pass.

6. Audit all owners observed in the archive and post-outage primary history, at least
   `bbugyi200.apollo`, `bbugyi200.athena`, and `bbugyi200.kellys_mbp`. Read `tailnet.md`
   before bounded SSH to `athena` and `mac`; run repository opens on the relevant
   machine before inspecting its repositories. Determine which have actual missing
   publication obligations rather than assuming each legacy manifest implies a new
   backlog. Run owner-scoped recovery on every machine with missing work, sequentially
   against the shared sidecar and using its real owner identity. Never impersonate
   another owner or copy arbitrary runtime stores to Apollo. Existing foreign snapshots
   can supply derived pages centrally, but new foreign runs may require that machine's
   sources.

7. The Mac may be offline. Attempt bounded access and recover everything already
   available from validated archives/history. If an owner or irreplaceable source
   remains unavailable, list the exact unresolved identities/revisions and required next
   action; retain their requests/evidence and keep the corresponding recovery work open.
   Do not label a partial recovery as "all published." Use SASE's monitor/questions
   mechanisms when a real wait or user action is required.

8. Verify the remote after all successful pushes by fetching through the supported
   repository workflow and recording the verified remote SHA. Compare expected
   identity/path/revision sets against that tree: run READMEs, canonical session pages,
   legacy family redirects, hood/machine/root links, and recoverable prompt archives
   must resolve. Validate manifests/snapshots and referenced payload digests read-only
   across owners. Report any independently preexisting integrity problem instead of
   re-signing it. Check that prior owners' source records and previously richer payloads
   were preserved. Observe remote refs, not only cached ahead/behind counts or local
   files.

9. Run an ordinary second bob-cli sync without recovery flags and compare the same
   frozen expected set. It must not re-retire recovered requests, rewrite stable
   recovered content, or recreate missing pages. Record legitimately new agents since
   the baseline separately and include all eligible work through a clearly stated final
   cutoff in the final reconciliation. A clean queue alone is not a completeness proof;
   a busy system need not have zero newly arriving requests.

10. Record per-owner before/after counts, expected/published identities, recovered
    prompt outcomes, excluded/unavailable sources, remote SHA, repeat-sync result, and
    representative GitHub page links in a durable artifact. Link evidence to `sase-1fm`
    using the supported bead/artifact commands. The land agent closes the defect and
    epic only once the required page set is proven remotely present and no unresolved
    recovery obligation is hidden by retirement, deletion, or an inaccessible owner.
    Follow the feature flag removal workflow when its authored condition is met; leaving
    a sunset flag must have its tracked removal bead.

## Verification and operational safeguards

For each changed code repository, run targeted regressions, then its required
`sase tool run check`. In `sase-core`, use the documented `just test` wrappers for
targeted work rather than bare Cargo; its final check includes binding tests. In `sase`,
use the default diff-scoped check and required whole-repository lint gates; do not
initiate `just check-full`. Run formatting before final verification. Read
`lint_and_test.md`; use `/sase_monitor` for known-long build/check/recovery commands
before starting them, and wait for each monitor handoff command itself to exit.

The operational rollback is to stop explicit recovery on an unexpected invariant
failure, preserve the failure and unacknowledged queue rows, and fix forward. Reuse the
supported transaction's cleanup/retry behavior. Do not force-push, manually reset the
live archive, edit queue JSON, overwrite another owner's manifest, or drop retained
requests to make the dashboard green. Already verified pages can remain published while
an independently failing item is investigated.

Acceptance requires all three: compatibility proven by negative as well as positive
tests, recovery correctness proven through a remote-backed transaction, and actual
bob-cli publication completeness proven at a recorded remote SHA and cutoff.
