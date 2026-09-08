---
tier: tale
title: Require real changes before publishing automatic artifact links
goal: Read-only runs create no artifact-link commits, while runs with real changes
  retain their links.
size: medium
proposed_by: bbugyi200.athena.0ao
status: done
---

# Publish automatic artifact links only for turns that change files

## Objective and scope

An agent that only consumes artifacts must finish without a VCS commit caused by its
artifact links. Preserve its audited reads locally. Publish its automatic links only
when the host can attribute real file changes to that same run.

This is a medium tale: one implementation agent can carry a bounded policy change
through the Rust contract, Python orchestration, and integration tests. It needs one
coordinated implementation, rather than separately landed phases.

Real changes include authored additions, edits, deletions, and renames in the primary
repository, an opened linked/external repository, or an artifact document repository. A
report or newly authored plan counts. Unchanged baseline dirt, read-generated link
indexes, lock files, generated link tables, automatic archive publication, and the
bookkeeping necessary to publish those links do not qualify. An explicit artifact-link
add/remove remains an intentional mutation with its existing persistence contract.
Preserve normal bead mutations and document lineage operations as well.

Limit this change to automatic consumer links and their commit/publication eligibility.
Preserve existing historical commits and links. No history rewrite, general
artifact-store migration, new CLI command, or memory edit is needed.

## Diagnosis and evidence

The suspicion is confirmed for audited artifact reads. The path is:

1. `src/sase/artifact_cli/read.py:handle_read` records the audit/consumption event and
   calls `_record_read_link` for an identified SASE agent.
2. `_record_read_link` immediately calls `ArtifactLinkStore.upsert_row`, then appends to
   the machine-local read-link outbox. The store writes document `links/**/*.json`
   indexes and lock sentinels. Bead endpoints can also write bead link events. Thus
   consuming context creates VCS dirt before any authored edit.
3. `src/sase/llm_provider/commit_finalizer_state/__init__.py:collect_dirty_state`
   includes that dirt after subtracting unchanged baseline fingerprints.
   `src/sase/finalizers/declaration.py:_build_live_context` uses it to create ordinary
   repository obligations.
4. `src/sase/finalizers/reconciliation.py:prepare_commit_dirty_state` invokes
   `_auto_commit_artifact_link_indexes_if_possible` unconditionally. The helper checks
   the validity/location of indexes, but never requires real changes.
   `src/sase/sdd/_artifact_link_commit.py` then creates
   `chore(artifact-links): persist link indexes`, potentially adding `.gitignore` too.
   In `src/sase/finalizers/commit.py`, this reconciliation occurs before the accepted
   declaration is processed.
5. `src/sase/sdd/artifact_link_outbox.py:drain_artifact_link_outbox` is a second
   persistence path. Its publishability test is whether the agent is published, not
   whether this run changed files. It runs from commit publication and the artifact-link
   background sweep. Publication of an agent or its family is not sufficient evidence of
   a real change by every consuming run.

The behavior was explicitly introduced in commit `0a1aef7d8` on 2026-08-21,
`feat(artifact-links): persist sidecar indexes on mutate and finalize`.

The unchanged-tree investigation ran this existing test successfully:

```sh
.venv/bin/python -m pytest -q -p no:cacheprovider \
  tests/llm_provider/test_commit_finalizer_auto_artifact_links.py::test_two_implicit_plan_reads_commit_once_without_declaration
```

The test creates a clean primary repo and plans sidecar, records two reads, then asserts
a new link-index commit with no declaration and no other edits. Its assertion describes
the bug and must be reversed. The same test module also contains a full controller case
that asserts the link-only sidecar is a commit obligation. Preserve its
evidence/integrity coverage while changing that premise.

Two additional existing CLI/outbox tests passed on the unchanged tree:
`test_drain_published_agent_commits_dirty_index_without_double_counting` and
`test_drain_unpublished_agent_leaves_entry_queued_and_uncommitted` in
`tests/main/test_artifact_link_outbox.py`. They confirm that actual `handle_read` calls
dirty the sidecar and that agent publication alone permits the drain to commit it.
Investigation total: three targeted tests passed, reproducing the current undesired
behavior; no implementation changes were made.

Simply removing the auto-commit helper is insufficient: the generic declaration path
would still require a commit, and the replay queue could publish the links later.
Ignoring all `links/` files would hide intentional or malformed changes.

## Implementation

### 1. Establish one host-owned eligibility contract

Open `sase-core` using `/sase_repo` before accessing it. Its
`crates/sase_core/src/finalizer/` already owns deterministic completion policy;
`crates/sase_core/src/artifact_link/` owns link row semantics. Put the new shared
eligibility rules and versioned data contract there, expose them through
`crates/sase_core_py`, and call them through a thin `src/sase/core/` adapter. Python
should collect VCS evidence and perform I/O; it must not independently reimplement the
policy. Follow the opened repository's instructions.

Bind pending automatic links and their release evidence to the consuming run's stable
identity. An agent name or family publication alone must not release a different run's
links. Reuse existing run metadata, dirty baselines, authenticated finalizer evidence,
and commit checkpoint identity rather than trusting a prompt claim or commit subject.
Recovery/retry for the same run must keep that identity.

Use host-verified commits containing qualifying file changes to authorize durable
publication. This covers changes already committed earlier in the run and changes that
the finalizer commits subsequently. Inspect attributable paths/content and
commit/checkpoint evidence; an old marker, a metadata-only auto-commit, or an unrelated
repository's dirty state is insufficient. Include opened linked and external
repositories and authored artifact documents, not just the primary repo.

Persist enough run-scoped release evidence for a background retry after the workspace
has gone away. The host records that evidence after it verifies the qualifying change.
Keep the contract narrow: pending links plus verifiable release evidence, not a
replacement completion protocol.

### 2. Keep automatic reads local until eligible

Change `_record_read_link` to record pending read rows locally without writing versioned
sidecar indexes, bead link events, or generated Markdown. Extend/reuse the existing
read-link outbox, with explicit run attribution, rather than adding a second queue.
Preserve mandatory audit and consumption logging, reasons, timestamps, and read counts.
Local consumers that need pending context must still be able to see it without promoting
it into VCS-backed state.

Pending-only recording changes the old outbox's count assumptions: it currently stores
cumulative values returned by an immediate store upsert. Define and test event
deduplication and count accumulation so two reads count twice, a retried drain counts
neither twice, and pre-existing durable counts remain intact.

Audit the automatic prompt-reference write-back paths in
`src/sase/agents_sync/referenced_by_planning.py`, its outbox/publication callers, and
automatic consumer-link derivation. Apply the same run eligibility wherever these paths
would publish a no-change run's consumer links. A family member's real commit must not
qualify every read-only neighbor. Preserve authored plan links and document lineage; do
not disable all link derivation or agent archive publication. Ensure plan proposal and
other mechanical handoffs can still consume recorded context, including pending reads,
when authoring a real deliverable.

### 3. Integrate finalization and replay without creating new obligations

Remove the assumption in reconciliation that valid link-index dirt from reads must be
committed. Normal reads should now produce no VCS dirt and therefore no repository
obligation, declaration recovery, stitch, `.gitignore` commit, or publication attempt.
Keep the generic finalizer's normal handling of authored changes, later-finalizer
changes, and explicit link mutations.

Integrate release with existing host commit success/publication paths, including
`src/sase/workflows/commit/workflow_publication.py`, the verified SDD/document commit
path, and finalizer reconciliation where needed. Promote eligible pending rows only
after real-change evidence exists; then reuse scoped index/bead commits and existing
publication verification. Avoid materializing new dirty files between declaration
publication and its validation. Record attributable markers for commits created during
promotion so subsequent scans accept the transition.

Both direct post-commit draining and
`src/sase/scripts/sase_chop_artifact_link_backfill.py` must enforce the same
eligibility. Published-agent status by itself no longer makes an outbox entry
publishable. Unqualified rows remain local and are subject to the existing local
retention policy; they must not be automatically replayable merely because a later turn
commits. Treat legacy queue rows without sufficient run evidence as unqualified rather
than assuming permission to publish them.

Handle already-materialized automatic link dirt conservatively. Only changes proven to
be this run's automatic consumer metadata may be moved to local pending storage and
removed from VCS state. Before changing a tracked index, preserve its pre-existing
bytes/staged state and other agents' or manual rows; use existing baseline/provenance
plus a checked snapshot. If that proof is unavailable, retain the data and report the
ambiguity through existing finalizer mechanisms. Do not blanket-reset repositories,
stash unrelated files, or classify every valid JSON index as disposable. New
pending-only reads should avoid needing this repair.

Keep no-work retirement outside accepted authored-file obligations. Do not weaken
`reject_unproven_reconciliation_transition`, stale declaration checks, baseline
protection, or the requirement for attributable commit evidence when authored dirty work
disappears. Cleanup must be idempotent and must not create its own commit obligation.
Leave uncertain or malformed files visible.

### 4. Add meaningful regression coverage

Use disposable repositories and isolated SASE state. Cover the public read path through
final context construction and the full controller, not only mocked dirty state. Update
old tests that intentionally asserted the unwanted auto-commit.

Required scenarios:

- Read one/two plans, a research document, and representative non-document artifacts
  with no edits: audited reads succeed; all repository HEADs and versioned contents stay
  unchanged; no commit obligation, recovery prompt, stitch, or link publication appears.
  Pending context remains locally recorded.
- A published reader, or a reader published as part of a family, still has no eligible
  links without its own real changes. Running the background drain and a later unrelated
  committing turn cannot publish that reader's pending rows.
- Read then author a source change: the actual change commits and this run's links
  publish. Repeat with only a linked/external repo change and with an authored artifact
  document. Source-only work needs no link commit.
- Real changes already committed within this run still qualify when the final worktree
  is clean. Old/foreign markers, reverted edits, unchanged baseline dirt, and
  link/lock/projection-only bookkeeping do not qualify.
- Explicit link add/remove, authored plan links, and normal bead changes retain their
  persistence behavior. Audited reads of bead endpoints alone do not manufacture
  bead-state commits.
- Existing/staged/manual link rows and foreign changes survive unqualified read
  handling. Malformed, symlinked, and unproven candidates remain visible.
- Repeated reads, interrupted promotion, repeated finalizer cycles, and retrying a
  failed publication are idempotent and preserve counts. Keep existing missing-marker,
  stale-context, mixed-repository, and mixed-sidecar integrity regressions meaningful.

Primary test locations are
`tests/llm_provider/test_commit_finalizer_auto_artifact_links.py`,
`tests/main/test_artifact_link_outbox.py`,
`tests/test_finalizers_commit_reconciliation_multi_repo.py`,
`tests/test_finalizers_commit_reconciliation_mixed_sidecar.py`, commit publication
tests, and the relevant agents-sync tests. Add Rust policy and PyO3 binding
round-trip/error tests for the shared contract.

## Verification and completion

Before implementation, read the current `lint_and_test.md` reference memory and any
other memory required by the files being changed. Refresh the workspace's editable
installation with `just install` if needed, and ensure it loads the updated Rust binding
rather than a stale installed wheel.

Run focused regressions, then `just check` in the SASE repo. Run the opened `sase-core`
repo's required `just check`/`scripts/check.sh`, including PyO3 tests;
`cargo test -p sase_core` alone is explicitly insufficient. If broad test selection
requires `just check-full`, use `/sase_monitor` as required by project policy. Use a
monitor for any other check that becomes long-running as well.

Completion means the public read-to-finalization scenario has zero new commits and zero
authored-work obligations for read-only runs, while a run with real changes retains its
links and passes existing publication and commit-integrity checks. Submit the normal
host-owned final declaration for each changed repo; implementation agents do not create
branches, commits, or PRs themselves.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact | Why | Uses |
| --- | --- | --- | ---: |
| cited-by | [agent:bbugyi200.athena.0ao--code][1] | prompt reference @plan:202609/artifact_links_require_real_changes.md | 1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ao.md

<!-- sase:referenced-by:end -->
