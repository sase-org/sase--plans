---
tier: epic
title: Make publication drains bounded and recoverable
goal: Fix the measured publication failures and prove safe recovery before bulk backlog
  draining
parent_bead: sase-1nf.10
phases:
- id: retry-policy
  title: Put failure accounting and bounded selection policy in Rust
  depends_on: []
  description: 'retry-policy: extend publication recovery wire types, centralize error
    classes and attempt charging, and implement deterministic bounded request selection
    with binding tests.'
  size: medium
- id: timeout-cleanup
  title: Terminate interrupted Git transfers and restore publication paths
  depends_on: []
  description: 'timeout-cleanup: make streaming Git cancellation kill and reap its
    process group, let the drain alarm escape broad handlers, clean prompts and artifacts,
    and verify existing stale-lock recovery.'
  size: medium
- id: drain-accounting
  title: Apply failure scope and bounded selection to commit drains
  depends_on:
  - retry-policy
  - timeout-cleanup
  description: 'drain-accounting: thread structured failure provenance and selected
    logical keys through the transaction, charge only attempted hood requests, preserve
    timeout stage, and cap background drain batches.'
  size: medium
- id: inventory-scope
  title: Load expensive inventory data only for requested hoods
  depends_on:
  - drain-accounting
  description: 'inventory-scope: filter indexed, dismissed, and historical records
    before payload hydration or per-commit Git calls, reuse commit records within
    a drain, and prove selected-hood equivalence.'
  size: medium
- id: payload-scope
  title: Render selected hoods and their affected browsing indexes
  depends_on:
  - inventory-scope
  description: 'payload-scope: produce targeted page payloads while preserving complete
    manifests and indexes, keep strict validation, reuse unchanged validated metadata
    within a transaction, and extend the read-only profiler.'
  size: medium
- id: bounded-cli
  title: Expose bounded retries without full reconciliation
  depends_on:
  - payload-scope
  description: 'bounded-cli: add retry limit and error-class options, use Rust selection
    under the outbox lock, drain only the bounded request set, and document queue
    isolation and resume behavior.'
  size: medium
- id: manifest-repair
  title: Repair verified stale explicit manifest entries
  depends_on:
  - retry-policy
  description: 'manifest-repair: extend explicit local-owner manifest repair to verified
    stale file lists, retain corrupt-candidate diagnostics, and prove idempotence
    and owner isolation.'
  size: medium
- id: replay-recovery
  title: Build and prove the backup-first stale-replay recovery procedure
  depends_on:
  - bounded-cli
  - manifest-repair
  description: 'replay-recovery: add an explicit resumable recovery helper that preserves
    unpublished owner data and objects, regenerates conflicting browsing pages over
    upstream, and proves lossless replay in real-Git fixtures.'
  size: medium
- id: recovery-proof
  title: Repair Apollo and prove bounded publication on both hosts
  depends_on:
  - replay-recovery
  description: 'recovery-proof: verify installed revisions, preserve unique Apollo
    content before repairing the stale replay and manifest mismatch, run small publication
    canaries on both hosts, and publish acceptance evidence for the land agent.'
  size: medium
proposed_by: bbugyi200.athena.sase-1nf.10
proposed_with: codex/gpt-6-astra
create_time: 2026-10-20 15:40:49
status: wip
bead_id: sase-1nf.10.1
---

- **PROMPT:** [prompts/202610/publication_drain_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/publication_drain_fixes.md)
- **PARENT:** [202610/athena_apollo_audit_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202610/athena_apollo_audit_fixes.md)
- **BEAD:** [sase-1nf.10.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nf/sase-1nf.10.1.md)

# Make publication drains bounded and recoverable

This child epic implements phase `pub-fix`, bead `sase-1nf.10`, of
`plan:202610/athena_apollo_audit_fixes.md`. Its measurement source is
`file:explicit:1332edd16fbe66b228322d62`, produced by closed phase `sase-1nf.9`. Read
both with `sase artifact read` before relying on them. The assigned phase had no
earlier-work notes when this plan was authored. The instrumentation is already landed in
sase commit `1325078a8e`; do not redo it.

This is an epic because the fixes cross a Rust wire contract, subprocess lifetime, two
distinct performance bottlenecks, CLI orchestration, and production recovery. Each
coding phase is bounded to one implementation agent. The final phase is an operational
proof, using monitor and gate continuations when required. Dependencies serialize the
overlapping transaction changes while allowing policy, subprocess work, and later
manifest-repair work to proceed independently where safe.

The ancestor's only reviewer decision is final: `gc_soak = no`. It does not alter this
publication track. No memory edits are needed or authorized by this plan.

## Evidence and intended behavior

The measurement report found these concrete failures:

| Finding                                                                                               | Consequence                                                   | Fix                                                                                  |
| ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Inventory: 209.84 seconds, 19,517 runs, 9,903 Git subprocesses, 452 MB of chat reads                  | A drain cannot finish under its existing 120-second alarm     | Select lightweight metadata before hydrating only requested hoods                    |
| One-hood planning: roughly 14 seconds and 24,000 files / 60 MB for either one or 67 runs              | Every hood renders the repository again                       | Render selected hoods plus affected aggregate indexes                                |
| Athena: 2,905 timeout and 434 lock-busy quarantines                                                   | Repository failures consume every active row's three attempts | Separate repository failure provenance from hood failure provenance                  |
| Broad exception handlers catch the drain alarm; streaming Git lacks an exceptional-exit cleanup guard | A timed-out drain can continue or leave children behind       | Propagate cancellation and terminate/reap the process group                          |
| Commit drain cleans only agents payload                                                               | Dirty `prompts/` and `artifacts/` prevent the next pull       | Use the same regenerable-path cleanup contract as full sync                          |
| Apollo bob-cli: local `2abeff3f9`, 1 ahead / 545 behind, 351 quarantined rows                         | Every pull replays the same conflicting derived pages         | Preserve all unique content, then regenerate from upstream plus preserved owner data |
| Apollo sase: `bbugyi200.apollo.03` explicit-file mismatch, 283 retired rows                           | One stale manifest entry blocks many hoods                    | Explicit, verified local-owner manifest repair                                       |

Counts are observations from 20 October, not assumptions about current production state.
Reinspect before recovery. The existing stale-index-lock reaper already runs from
`clean_agents_payload_worktree`; verify and reuse it. Do not add another reaper based
solely on the report's hypothesis about how the lock was left behind.

The successful path remains enqueue, select, pull, prepare, commit, push, verify,
acknowledge. A repository failure leaves requests queued without consuming their
attempts. An attributable hood failure charges only that hood's attempted rows.
Acknowledgment requires successful publication and the request's page/revision and
deferred-prompt obligations; merely preparing a manifest is insufficient.

## Common implementation rules

- Open sase-core with `/sase_repo` and `sase repo open sase-core -r "<reason>"`; use
  only the printed checkout and read its `AGENTS.md`. Open any other repository the same
  way. Read sidecar artifacts with `sase artifact read`.
- Shared failure classification, retry selection, membership, and repair eligibility
  belong in Rust. Python gathers filesystem/process facts, calls the bindings, and
  performs I/O and Markdown rendering. Reuse existing Rust identity and file-set
  classifiers; do not reproduce name semantics in Python string-prefix tests.
- Wire extensions are additive. Optional fields default to current behavior when absent;
  keep the existing schema version if old readers still accept the wire. Add PyO3
  round-trip tests and thin Python facades for new calls. No fallback or feature flag is
  needed for these fixes, as the ancestor plan already specifies.
- If a phase changes both repos, let the host finalize both and move the core pin. If it
  consumes a previously landed core API, run `just ratchet-core-revision` and rebuild
  the local binding with `just rust-install` before Python tests.
- Read `lint_and_test` before finishing a sase change. Run `sase tool run check` in each
  changed repo; use `/sase_monitor` for long commands. No `check-full`, raw cargo,
  manual commits, branches, or global installs. Use `/sase_final` for host-owned
  completion. Add regressions that fail on the old behavior.
- Read `cli_rules` before CLI changes, `tools` before the profiler, and `docs_writing`
  before documentation. Generate option reference changes through `just fix`; put
  behavior in Reference and recovery internals in Contribute.
- Preserve `files/objects`: it is append-only and not reliably regenerable. The routine
  cleanup path must never delete or reset it. Use existing pending object publication
  and validation where the transaction needs them.
- Keep per-drain diagnostics and the doctor histogram consistent. Error-class
  compatibility matters because recovery filters stored historical errors.
- Phase workers close only their assigned child phase. They record discoveries as
  `PROPOSED FOLLOW-UP:` notes on that phase, not new beads. Before each close run
  `sase bead epic-symbols <phase>` and resolve or re-key any remaining exemption to an
  open phase or this child epic. Do not leave exemptions on closed phases.
- A check failure reproduced identically on the clean base is documented with the
  reproduction and any existing task reference, then the completed phase closes. It does
  not justify leaving the phase open. Do not weaken tests.

## Phase retry-policy

Work in sase-core's `agent_publication_recovery.rs` and its `vcs` bindings, with focused
Python facade tests as necessary. Split new files to respect repo limits.

1. Move the existing publication error taxonomy from `publication_outbox_diagnostics.py`
   into a core classifier callable both by retry selection and by the Python diagnostic
   facade. Preserve the current class names and precedence: `timeout`, `lock_busy`,
   `index_lock`, `unstaged_changes`, `rebase_conflict`, `manifest_mismatch`, `unknown`,
   `other`. Cover the report's actual strings, including "index contains uncommitted
   changes". Keep error class distinct from failure scope: text alone cannot tell
   whether a manifest failure concerns the requested hood or an unrelated owner.
2. Add a pure failure-accounting decision from explicit scope (`repository` or `hood`)
   and the observed error. Repository failures do not mutate request failure history,
   increment attempts, quarantine, or retire rows. Hood failures use the existing
   attempt threshold and repeated-format-error retirement rules. The host supplies the
   exact attempted logical keys; no implicit all-rows key expansion is permitted when
   scope is absent or unknown.
3. Extend `select_publication_retries` with optional positive `limit` and a list of
   accepted error classes. Filter stopped rows before applying the limit; preserve
   terminal-over-quarantined precedence and deduplicate logical keys. Reject
   zero/negative limits and unknown classes. Absent options retain the existing retry
   selection contract. Preserve `prior_class` as the lifecycle class and
   `prior_failure`; expose the error class separately if needed.
4. Add the small shared drain-selection policy needed by the host: select at most N
   distinct logical requests, deterministically oldest first with a key tie break, with
   an optional preferred newly enqueued key placed first. A bounded CLI retry gives
   existing active rows priority and fills the remaining slots from the eligible stopped
   rows. The error-class filter applies to stopped candidates; existing active rows are
   resumed without resetting their attempts. The response distinguishes active rows from
   rows to revive. A no-match result must never mean "drain everything". Keep this
   policy in the same core domain, reusing `select_publication_retries` rather than a
   second eligibility algorithm.
5. Test missing-field wire compatibility, mixed lifecycle states, duplicate keys, stable
   ordering, limit across both stopped classes, active rows exhausting the budget,
   filter precedence, invalid arguments, and scoped charging. Add binding round trips
   for every new API. Run the core check and record exported names and commit evidence
   for downstream phases.

## Phase timeout-cleanup

Work in `src/sase/sdd/_git.py`, the publication transaction boundary, and the existing
Git cleanup helpers. This phase does not change attempt charging.

1. Give streaming Git its own process group/session. Enclose pipe registration, reading,
   and waiting in a cleanup guard. On a drain alarm, interruption, stall, ceiling, or
   other exceptional exit, terminate the whole group, escalate to kill after a short
   bounded grace, reap the direct child, and close pipes. Never leave an unbounded final
   `communicate()` when a descendant can retain a pipe. Preserve normal output and
   re-raise the original cancellation.
2. Move the drain-timeout exception into an importable neutral module if needed.
   Explicitly re-raise it before broad handlers in the per-hood preparation, manifest
   verification, and other reachable publication adapters. Inspect
   optional-history/error handlers too; do not solve this by silently losing errors or
   leaving a worker thread mutating the clone after timeout.
3. Make the commit drain's preflight, finally, and non-fast-forward retry cleanup
   restore both agents payload and regenerable prompt/archive paths using shared
   helpers. Preserve pending `files/objects` and unique committed data. Cleanup should
   attempt each owned path family even if another cleanup operation fails, retain the
   original failure, and report cleanup failures as additional evidence.
4. Reuse `try_remove_stale_index_lock` and its Rust owner/age/identity policy. Prove an
   aged owner-free lock is recovered and a live, young, changed, or uninspectable lock
   is preserved. Such a lock becomes a repository failure.
5. Extend `tests/sdd_store/test_sdd_git_streaming.py` with a real fixture child that
   spawns a descendant holding a pipe; exercise both native timeout and an external
   drain exception. Extend publication tests for alarm escape, lock release, clean
   `prompts/`/`artifacts/`, retained objects, and successful next drain. Use short
   isolated deadlines, not a production timeout.

## Phase drain-accounting

Work in `commit_publication.py`, `commit_publication_transaction.py`, timing, outbox
adapters, and their focused tests. Consume the Rust policy from retry-policy.

1. Thread structured failure provenance through `DrainResult` and the top-level result:
   scope, exact attempted keys, originating stage, and error. Clone, lock, pull/rebase,
   shared inventory, commit, push, and cleanup failures are repository failures even if
   a helper currently returns request keys with them. Validation of unrelated owner
   state is also repository-wide. Per-hood build, render, and request-specific prompt
   failures are attributable hood work.
2. Replace `_record_failure`'s unconditional all-active charging with the core
   accounting decision. Preserve attempts, terminal/quarantine fields, and the previous
   request error for repository failures; report these failures through the drain
   outcome and timing record. A hood error updates only attempted keys once, including
   when cleanup or a push also fails afterward.
3. Capture active-hood keys and alarm stage at the instant the alarm fires. A timeout in
   that hood charges only those keys; an inventory, lock, network, shared manifest, or
   cleanup timeout charges none. Finally cleanup must not overwrite the original alarm
   stage. Restore any prior signal handler/timer correctly and retain the existing
   120-second drain setting.
4. Apply the core drain selector before expensive work. Default commit-triggered batches
   to 25 request rows, preferring the newly enqueued request so a backlog does not
   starve newly completed agents. Pass an explicit selected-key set through the
   transaction; an empty set is a no-op. Group selected rows by hood, publish each
   selected hood once, and leave all unselected rows unchanged.
5. Before acknowledgment after a successful push or verified already-published state,
   use existing completion policy to check the selected request's page, requested
   revision, and prompt obligation. A rejected push or interruption leaves requests
   durable. A cleanup failure after a successful acknowledged push must not resurrect or
   falsely charge completed rows.
6. Update the old charging assertions in `test_commit_publication_bounded_drain.py` and
   `test_commit_publication_queue.py`. Test every repository stage against several
   hoods, a genuine failing hood beside a healthy one, exact-once charging, untouched
   unselected rows, preferred-new-request selection, failed push, completion mismatch,
   and interrupted publication followed by successful replay.

## Phase inventory-scope

Work in `inventory.py`, `inventory_sources.py`, `inventory_history.py`,
`repository_inspection.py`, and their callers/tests. This phase removes the measured
210-second whole-project hydration from a one-hood drain.

1. Add an optional requested-hood set to inventory construction. `None` retains full
   reconciliation; an empty set returns no work. Commit drains pass exactly their
   selected hoods. Direct `publish_agent_hood` callers also build scoped inventory when
   they were not supplied one.
2. Use indexed names or the minimum identity metadata necessary to apply existing Rust
   owner-aware normalization/membership before reading prompts, chats, workflow
   payloads, or commit markers. Do the same for dismissed bundles and commit-only
   synthetic runs. Filter historical run and session lanes without dropping any member
   of a selected hood. Keep session-container commit attribution, legacy names, revival
   inputs, and live/dismissed deduplication.
3. Reuse commit metadata obtained from the batched primary history read. Resolve
   remaining marker SHAs once per repository/SHA within the drain, retaining the
   primary-repository ownership check. Do not introduce a durable cache with a separate
   invalidation problem. Lightweight project-index/history scans may remain; unrelated
   payload hydration and thousands of repeated Git lookups may not.
4. Preserve diagnostics for selected malformed records. Bound unrelated-record
   diagnostics rather than hydrating unrelated records to produce them.
5. Compare selected hood snapshots from full and scoped inventories on fixtures with
   indexed, dismissed, legacy, commit-only, and session-lane history. Use spies or
   sentinel unreadable unrelated payload files to prove those payloads and commit
   markers are never opened. Test repeated SHAs and root aliases. Add a large unrelated
   population without increasing expensive-read or Git-call counts for the selected
   hood; avoid fragile wall-clock unit assertions.

## Phase payload-scope

Work in `publication_planning.py`, `publication_validation.py`, rendering modules, and
`tools/agents_publication_profile`. Python rendering is presentation; reuse the existing
Rust identity/file-set policy for semantic decisions.

1. Split targeted planning from full reconciliation. A targeted payload contains
   selected snapshots/run files, selected hood/agent/session/legacy redirect pages, the
   changed owner manifest, and affected root/user/machine indexes. It must not contain
   unrelated agent, session, or hood pages. Retain complete manifest entries for every
   unselected hood and owner.
2. Preserve aggregate index correctness. Root/user totals can use complete manifests;
   the machine page currently needs snapshots for its states column. Load the metadata
   needed to preserve that column and reuse it once within the locked transaction. Do
   not truncate the machine index to selected hoods or silently change its format to
   make rendering cheaper.
3. Retain snapshot identity/digest checks and selected referenced-file checks. Full
   sync/doctor must retain full validation. Do not ignore a malformed unrelated owner's
   manifest to get a successful canary. Avoid repeated whole-repository
   validation/rendering for several selected hoods by sharing validated metadata within
   the transaction; invalidate/reload after a retry pull because upstream may have
   changed. No persistent-cache schema is needed.
4. Restrict bead-link gathering and kinship rendering to the pages being written. On a
   non-fast-forward retry, rebuild shared indexes from fresh upstream plus the local
   selected snapshots. Never overwrite another machine's new pages from a stale
   whole-repository payload.
5. Test targeted output against the relevant subset of full output, unchanged
   foreign/unselected bytes, complete index counts and states, multiple selected hoods,
   empty selection, legacy redirects, and concurrent remote-owner changes. Assert
   payload file/byte counts scale with selected runs plus aggregate indexes, not every
   run page in the sidecar.
6. Extend the profiler with explicit requested-hood selection, recording both inventory
   and planned payload counters for the selected set. It remains read-only. Do not first
   run the expensive full inventory merely to choose the requested hood. Preserve the
   existing comparison mode for measurement history.

## Phase bounded-cli

Work in `main/parser_agent_sync.py`, parser validation and the command dispatch,
`git_sync.py`, recovery facades/outbox operations, and focused CLI tests.

1. Add `-l/--retry-limit N` and repeatable `-e/--retry-error-class CLASS`, with clear
   help. Require a retry flag; a class filter requires a limit. Reject bounded mode with
   check/refresh, drop, or repair modes, and require exactly one project for a bounded
   operation. Keep existing unbounded invocations compatible. Preserve the existing
   explicit-project rule for retired retries.
2. Under the existing outbox mutation lock, ask Rust for at most N total logical
   requests: active rows first, then stopped rows matching lifecycle flags and error
   classes. Only the selected stopped rows receive a fresh attempt budget. Existing
   active rows keep their budget. Record the selected identities and prior errors in
   diagnostics before those errors are cleared.
3. Route bounded mode into the targeted transaction with that exact selected set, not
   `_sync_project`'s full reconciliation. Selected hoods are complete hood publications,
   so a row limit is not a hard byte/run cap; the 120-second drain bound still applies.
   Do not silently revive or acknowledge additional rows from the same hood, other
   projects, or the Referenced By queue. Existing unbounded behavior for the latter
   remains unchanged.
4. If no eligible rows exist, report a no-op. If active rows consume the cap, report
   that no stopped rows were revived. On interruption those selected rows stay active
   and are prioritized by the next bounded invocation, so repeated retries cannot
   accumulate an ever-larger active batch. A successful drain reports selected, resumed,
   revived, completed, and remaining counts.
5. Test argument validation and aliases, limit across lifecycle classes, a filter with
   no matches, active-first resume, isolation of unselected/foreign queues, genuine
   targeted execution, two bounded batches, failed push then replay, and no duplicate
   acknowledgment. Update the two publication Reference and Contribute pages and
   generated option reference through the documented route.

## Phase manifest-repair

Work in `publication_repair.py`, the Rust session-manifest domain where policy is
needed, and `docs/contribute/agents-sidecar-sync.md`. This phase builds manifest repair
support; production mutation belongs to recovery-proof.

1. Extend explicit `--repair-manifest` so it can repair an existing stale explicit file
   list as well as a missing hood entry. For a local-owner candidate, first verify
   snapshot digest/identity, project/hood identity, path safety, and every referenced
   payload's bytes/size/digest. Then use the Rust policy to normalize the redundant
   explicit list into the already-supported slim entry and derive counts from the
   verified snapshot. Preserve all original content.
2. Distinguish a redundant-list mismatch from missing/corrupt underlying content. Report
   and skip corrupt candidates; do not simply erase `files` on every invalid entry or
   use digest repair to bless unexplained corruption. Never modify a foreign owner's
   subtree. Make repeated repair a no-op. Retain the existing missing-entry recovery
   behavior and test both together.
3. Test the exact stale-list shape reported for `bbugyi200.apollo.03`, missing entries,
   current/slim/legacy entries, corrupt referenced bytes, snapshot identity mismatch,
   foreign-owner isolation, mixed valid/invalid candidates, and a second no-op repair.
   Add a binding round trip for new repair policy. Document the expanded explicit repair
   behavior and run required checks.

## Phase replay-recovery

This phase implements the finite recovery operation for the measured stale local commit;
it does not add automatic history rewriting to normal drains. Work in focused
agents-sync recovery modules, a thin `tools/agents_publication_recover_replay` entry
point, and the Contribute runbook. Put any new conflict/preservation decisions in
sase-core and test its bindings.

1. Provide `-p/--project`, `-H/--expected-head`, `-b/--backup-dir`, and `-a/--apply`.
   Without apply, report the proposed recovery and preservation inventory without
   resetting or publishing. Apply requires the expected local HEAD and a verified
   complete backup; fail before mutation if observed state changed. The backup directory
   also holds atomic progress records so rerunning resumes or reports already complete
   rather than overwriting the original backup.
2. Author and implement a concrete backup-first stale-replay procedure for Apollo
   bob-cli. Discover the hidden clone through SASE's repo interface, acquire its
   existing sync lock, and recheck HEAD/upstream/worktree state while holding that lock.
   Inventory and preserve all unpushed commits in a verified bundle, staged and unstaged
   changes, untracked objects/prompts/artifacts, and the locally unique owner snapshots
   and referenced payloads. Store checksums and source revisions outside the clone
   before any abort/reset/replay recovery.
3. Classify conflicting files into reproducible browsing pages versus canonical owner
   snapshots/content and archive objects. Regenerate browsing pages on top of current
   upstream; do not replay their stale versions. Preserve local-only snapshots and
   blobs. For a same-owner/same-hood divergence, merge through the existing
   `build_hood_snapshot` preservation path, adapting saved verified runs into inventory
   inputs and using upstream as the previous snapshot, and retain both sides' unique
   runs/content; never choose an entire stale subtree over upstream. If identity/content
   cannot be reconciled, stop before destructive mutation with the backup intact.
4. Invoke recovery through an integration callback inside the existing SASE sidecar
   transaction machinery, so recovered data is reapplied after transaction cleanup and
   is not discarded by the next preflight. Reuse the normal commit/push retry
   operations. Do not require a phase worker to run an ad-hoc `git commit`. Resolve
   same-run byte divergences conservatively: use existing richer-content preservation
   only when it proves the other version is retained; otherwise retain both in the
   backup and refuse apply until the conflict has an explicit lossless resolution. Do
   not silently treat longer arbitrary text as a superset.
5. Test with a local bare remote and two owner clones: one local unpublished commit with
   unique chat/session pages, an overlapping remote derived-page update, pending
   objects, and a stale explicit list. Prove backup restoration, retained unique content
   on both sides, validated manifest references, resumed recovery after interruption,
   and idempotent replay. Document exact preconditions, commands/helper invocation,
   postconditions, and rollback evidence for the next phase. A report-only description
   without executable steps is insufficient.

## Phase recovery-proof

This is a small operational canary, not the bulk backlog drain assigned to
`sase-1nf.11`. Read `tailnet` before SSH. Use `sase repo open` for every inspected repo
and audited artifact reads for prior reports. Use `/sase_monitor` for long profiles,
repair commands, and waits; prepare a concrete continuation prompt.

1. Verify the installed sase/core revisions on athena and apollo contain all
   prerequisite phases. If an install is stale, request `sase update` through
   `/sase_gate`, as the ancestor plan requires, and continue through its follow-up. Do
   not run global install recipes directly.
2. Collect fresh doctor histograms and clone state. On Apollo, execute the
   fixture-proven backup/preservation procedure for the bob-cli stale replay
   (historically `2abeff3f9`) and repair the local sase manifest mismatch (historically
   `bbugyi200.apollo.03`). Verify saved revisions and content checksums before any
   destructive step. Hold the sync lock across the state recheck and mutation; use
   hidden machine clones, never primary nested sidecars.
3. Preserve evidence that the unique `bob-cli-7c` session/run content and any pending
   objects are either published intact or restored as pending inputs with durable retry
   obligations. A backup alone is not recovery completion. Push the repaired owner data
   through the normal transaction and validate the remote manifest and its referenced
   files. Repeating the repair must be a no-op.
4. On athena select one known previously timeout-failing hood, profile the scoped path,
   and publish through the real bounded drain with the ordinary 120-second setting. On
   Apollo run a canary for each repaired class/project. Start with limit 1; use at most
   one batch of 25 per project if needed to cover a class. Keep unrelated
   quarantine/retired rows untouched. Record selected logical keys, before/after queue
   fields, elapsed stage timings, payload sizes, Git counts, resulting remote revision,
   and manifest verification.
5. Demonstrate idempotent retry and interrupted-publication resume on the local real-Git
   fixture; production need not be deliberately killed. Confirm a newly completed agent
   after installation publishes automatically, using an existing completion if available
   or the approved `/sase_run` launch path for a minimal canary agent. No unreviewed
   direct SASE agent spawning.
6. Publish a research artifact with the scoped/full baseline comparison, recovery
   backups and checksums, representative remote paths and revision evidence, all five
   acceptance checks, and precise commands/classes suitable for `sase-1nf.11`. Attach it
   to this phase. Note outstanding backlog counts; a large remaining backlog is expected
   here. Close only this child phase, not `sase-1nf.10`, `sase-1nf.11`, `sase-10x`, or
   the ancestor epic.

## Land-agent acceptance and closure

The child epic's land agent checks that every phase has landed and that core pins and
bindings agree. It consolidates the evidence against the ancestor's five acceptance
criteria: a formerly failing request publishes inside 120 seconds, remote manifests
agree with referenced files, replay is idempotent, interrupted publication resumes, and
newly completed agents publish automatically. Required checks run through
`sase tool run check`; baseline-only failures follow the user's document-and-close rule.

The land agent resolves all child-epic and `sase-1nf.10` symbol exemptions, records the
recovery artifact and the handoff commands on `sase-1nf.10`, and completes this child
epic through its normal land workflow. Once descendants are closed, it runs
`sase bead epic-symbols sase-1nf.10` again and closes **only the enclosing assigned
phase** with `sase bead close sase-1nf.10 --note "<verified acceptance and evidence>"`.
This enclosing-phase closure is explicitly delegated to the child land agent by the
user's request. It does not authorize closing `sase-1nf`, any higher ancestor, the
bulk-recovery phase, or `sase-10x`. The existing dependency then releases `sase-1nf.11`
to drain the remaining backlog in its own bounded batches.
