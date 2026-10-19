---
tier: epic
title: Committed code snapshots and records for project statistics
goal: Complete sase-1m3.7 with accurate cached code snapshots, an evidenced line-counter
  choice, and metadata-only records summaries exposed by project_stats_query.
parent_bead: sase-1m3.7
phases:
- id: counter-policy
  title: Role policy, nonblank counter, and measurement contracts
  depends_on: []
  size: medium
  description: 'counter-policy: implement pure nonblank counting and role classification,
    and finish the wire contracts needed by code and records snapshots.'
- id: code-snapshot
  title: Committed-tree snapshots and incremental blob cache
  depends_on:
  - counter-policy
  size: medium
  description: 'code-snapshot: enumerate committed trees, resolve committed attributes,
    measure bounded blob batches, and cache per-path results with correct attribute
    and classifier invalidation.'
- id: records-snapshot
  title: Metadata-only records summaries
  depends_on:
  - counter-policy
  size: small
  description: 'records-snapshot: count record-repository commits and tracked files,
    measure author-time bounds without reading content, and cache complete results
    by tip.'
- id: query-integration
  title: Code facet dispatch, coverage, and deduplicated aggregates
  depends_on:
  - code-snapshot
  - records-snapshot
  size: medium
  description: 'query-integration: populate project, repository, and portfolio code
    results, attach records summaries, avoid unrelated facet work, and verify the
    public Python binding.'
- id: measurement
  title: Tokei dependency gate and performance evidence
  depends_on:
  - query-integration
  size: medium
  description: 'measurement: evaluate tokei against the integrated nonblank baseline,
    measure cold and incremental snapshots on sase, and assemble compatibility and
    completion evidence for sase-1m3.7.'
proposed_by: bbugyi200.apollo.sase-1m3.7
create_time: 2026-10-17 02:56:45
status: done
bead_id: sase-1m3.7.1
---

- **PROMPT:** [prompts/202610/project_stats_code_records.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/project_stats_code_records.md)
- **PARENT:** [202610/sase_stats_project.md](https://github.com/sase-org/sase--plans/blob/main/202610/sase_stats_project.md)
- **BEAD:** [sase-1m3.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1m3/sase-1m3.7.1.md)

# Plan: Committed code snapshots and records for project statistics

## Scope and authority

This child epic implements only the `core-code` phase of
`plan:202610/sase_stats_project.md`, assigned as `sase-1m3.7`. Read that artifact with
`sase artifact read` when consulting its full design. Its sections 4, 6, 15, and 21 are
the governing metric, cache, implementation, and verification contracts. The parent
phase had no previous-work notes when this plan was prepared; its dependency
`sase-1m3.4` is closed and its foundation is present.

The work is split because the dependency experiment, Git tree measurement, cache
invalidation, and multi-facet integration are separate substantial tasks. Each phase is
direct implementation work for one coder. The two snapshot phases can proceed
independently after the policy/contracts phase. Only `query-integration` owns the main
query orchestration changes.

The ancestor's final decision is `stats_decision_record = no`. Do not edit any memory,
including decision strands. The required skipped-memory `PROPOSED FOLLOW-UP:` is already
recorded on `sase-1m3.7`; do not duplicate it. Do not reopen that decision. There are no
new reviewer decisions in this plan.

Implement shared behavior in the linked `sase-core` repository, opened with
`sase repo open sase-core -r "<phase-specific reason>"`; read its `AGENTS.md` before
editing. The plan uses repository-relative paths throughout. Keep Python changes to
necessary wire-consumer compatibility or the core revision pin. The parent epic's
loader, render, CLI, TUI, Runs, and attribution phases retain their own feature work.

## Verified starting point

In `sase-core`, `crates/sase_core/src/project_stats/` contains the foundation:

- `code.rs` and `snapshot.rs` return `FacetResult::unimplemented()`. `snapshot.rs` is
  currently documented as the records seam, despite its broad name. Replace it with the
  code snapshot implementation and put records in an explicit `records.rs` module.
- `wire/response.rs` already defines `ProjectStatsCodeWire`, `ProjectStatsSnapshotWire`,
  language rows, repository coverage, and nullable facet fields. The snapshot has only
  integer line counters, lacks source/config/other role partitions and exclusion counts,
  and has no first or last commit timestamps. Some fields cannot yet express unavailable
  data.
- `query.rs` resolves and walks all code and record repositories and calls Output and
  every facet before checking the requested facets. It then strips unrequested fields.
  It does not attach snapshots, and portfolio Code is always null. `output.rs`
  constructs repository rows and regards an absent history walk as unavailable; that
  assumption must change for code-only requests.
- `cache.rs` is specifically a raw-history cache with writer-only locks, atomic writes,
  and a `(path, mtime, length)` process memo. Reuse its small persistence mechanisms
  mechanically if convenient, or mirror them for separately namespaced typed entries. Do
  not rewrite history policy.
- `file_history::runner::run_git_checked_with_stdin` and its unchecked counterpart
  already drain output while writing input. Use them for blob and attribute batches. The
  older `file_history::blobs::read_blobs` still writes all input first; do not adopt it
  or expand scope to migrate it.
- The Python binding already exists in `crates/sase_core_py/src/project_stats/` and
  releases the GIL. Extend its round-trip tests; no new binding is required.
- There is no tokei dependency. The release profile uses thin LTO and one codegen unit.
  Wheel configuration is `crates/sase_core_py/pyproject.toml`, with stripping and the
  `pyo3/extension-module` feature enabled.

Reinspect these seams on each phase's landed base: the parent epic has other active Rust
phases. Preserve their changes instead of restoring the starting stub versions described
above.

## Shared behavior and wire contract

All measurements refer to the resolved commit, never the working tree or index. Use the
existing selected reference and repository identity logic. No fetches or working-tree
mutation occur during a statistics query.

Code roles are exactly `source`, `tests`, `docs`, `agent_context`, `config_data`, and
`other`. Exclusions are `generated`, `vendored`, and `binary`. Count skipped gitlinks
and symlinks separately; never follow either. An oversized regular blob is a generated
exclusion with an oversize reason. Every regular file belongs to one included role or
one exclusion. Expose per-role file/line counts and exclusion file counts additively on
the snapshot, together with skipped-entry counts, so the partition is auditable. Use
typed serde structs/enums, deterministic ordering, and defaults on new fields for old
fixture/cache deserialization.

`code_files` and `code_lines` are source plus tests. `test_lines` is a subset of
`code_lines`; `test_share` is their ratio or null when code lines are zero. Language
rows describe that same source-plus-tests population, not prose or agent context. Docs
and agent-context lines are nonblank. Retain nonblank counts for config/data and other
files to make their role rows meaningful. The selected counter is explicit on Code and
code snapshots.

Keep existing wire field names. Finish the previously unimplemented payloads so
unavailable is never serialized as measured zero:

- When Code is requested, return a Code object with configured/measured repo counts.
  Make measurement fields nullable where needed: if every configured source failed,
  file/line/language measurements are null and issues explain why. A successfully
  measured empty repository produces zero counts, an empty language list, and null test
  share. With no configured code repos, report the empty measured population, not a
  repository failure.
- With partial source availability, aggregate measured repositories and name the missing
  coverage. A failed repository snapshot is null with an issue. Never turn a truncated
  tree or failed blob batch into a complete zero result.
- Snapshot content-derived fields, including tracked/test/doc/context lines and language
  counts, are nullable for records because their contents are deliberately not read.
  Likewise, lifetime/agent-commit fields not measured by a code-only snapshot are null,
  rather than fabricated or obtained by an unrelated full history walk. Existing
  available numeric values stay numeric.
- Add optional first/last commit timestamps to the snapshot for records. Their clock is
  author time, matching lifetime statistics. Do not populate the existing
  `ProjectStatsLifetimeWire` with invented active-day or agent-attribution values merely
  to transport these two facts.
- Preserve the existing null behavior for an unrequested Code facet and absent
  repository snapshots. Do not change Output or Runs semantics.

These payloads have not yet been emitted by the current query implementation. Complete
their intended schema-1 availability contract now, update Rust fixtures, and verify
compatibility with any newly landed Python consumers. Do not bump the wire schema for
additive fields alone. If a now-landed consumer requires a schema change, follow the
repository's schema-change procedure across both repos instead of allowing silent
disagreement.

For `tracked_files`, retain the total tracked blob-entry count, including excluded blobs
and symlinks but not gitlinks. Distinguish it from included role counts; assert that
included roles plus exclusions plus skipped symlinks equal tracked files.
`tracked_lines` for code is the sum of included role line counts under the declared
policy. Records only measure tracked files, commit count, and author-time bounds.

## Phase counter-policy

Own the measurement contracts in `wire/response.rs` and their fixtures, plus new pure
`classify.rs` and `counter.rs` modules. Keep module facades small.

1. Define the shared nullable fields and role/exclusion rows above before dependent
   phases start. Document their serialized form with a realistic Code fixture, an
   unavailable Code fixture, and a records snapshot fixture.
2. Implement a pure classifier whose inputs are path, mode, size, parsed committed
   attributes, and blob evidence. No filesystem/config discovery belongs in the
   classifier. Its policy is:
   - Safety exclusions (NUL-containing bytes, LFS pointer, size cap) are not bypassed by
     a prose or source role override.
   - Explicit `linguist-generated`, `linguist-vendored`, and `linguist-documentation`
     attributes override corresponding heuristics. Handle `set`/`true`, `unset`/`false`,
     and `unspecified` distinctly; an explicit false suppresses that heuristic. When
     both exclusions are enabled, use generated before vendored, then documentation for
     included text. Record why an entry was excluded.
   - Generated/snapshot/golden/minified and vendored path heuristics precede ordinary
     roles. Agent instruction files (`AGENTS.md`, `CLAUDE.md`, equivalent known names),
     `sase/memory/`, `sase/macros/`, and skill trees are agent context before the
     generic Markdown/prose rule. Recognized prose is docs, data formats/lockfiles are
     config/data, recognizable code under test paths or test filename patterns is tests,
     other recognizable code is source, and remaining text is other. A JSON fixture or
     prose file under a test directory is not source code.
   - Select and document a fixed blob-size cap (8 MiB, consistent with the existing blob
     helper). Unknown extensions may use a bounded, SASE-owned shebang recognizer,
     including common `env` forms.
3. Implement nonblank counting with correct CRLF, whitespace-only, empty, and
   unterminated-line behavior. Keep detection separate from counting so both the
   eventual tokei and fallback paths use the same role policy. Define a small common
   blob-counter interface with explicit policy and language inputs. Integrate initially
   with `counter: nonblank`; this is the measurable baseline, not a decision to reject
   tokei. The final phase performs the dependency gate once real production callers keep
   the counter reachable in a release wheel.

Tests cover override truth states, role precedence and partition, language and shebang
detection, binary/LFS evidence, file cap, comment-only text, CRLF, and final
unterminated lines. This phase need not wire the live query, but its library
implementation must compile and be exercised by tests.

## Phase code-snapshot

Own `snapshot.rs`, committed-tree/attribute/blob helpers, and a dedicated snapshot cache
module. Consume the pure classifier and wire contract without changing the main query
orchestration yet. Provide an internal per-resolved- repository measurement function for
integration.

1. Resolve the committed tree and enumerate with `ls-tree -r -z -l <commit>`. Parse
   NUL-separated paths without whitespace splitting; validate entry framing, modes,
   OIDs, and sizes. Preserve distinct non-UTF-8 paths internally rather than collapsing
   them through lossy strings. Use the existing hardened runner and request budgets for
   every Git subprocess. An empty repository is a valid empty snapshot.
2. Batch `git check-attr --source <commit> -z --stdin` for the three attributes. Dirty
   worktree/index attributes must not affect the result. Prevent user, system, and
   `.git/info/attributes` overrides from contaminating committed policy; use an isolated
   temporary Git metadata view of the same object database if necessary, without editing
   the source repo or bypassing runner environment isolation. Use explicit runner
   arguments for that view, not ambient `GIT_DIR` that the runner scrubs. Report
   unsupported committed-attribute lookup as unavailable with an actionable issue; do
   not silently consult the working tree.
3. Skip gitlinks and symlinks without reading targets. Skip obvious generated/vendored
   and oversized blobs before content reads. Read eligible unique blob OIDs in bounded
   `cat-file --batch` chunks using the concurrent stdin runner. Validate declared sizes
   and headers, account for framing overhead in the output cap, and surface
   missing/truncated/failed reads. Do not request an entire large repo in a single
   unbounded batch.
4. Cache a complete snapshot by identity and local object-store context,
   classifier/counter policy versions, measured tree OID, and per-path blob rows.
   Include a distinct cache-kind discriminator so history and records entries cannot
   collide. Cache reads and writes follow the existing atomic, writer-lock and
   process-memo conventions; disabled or corrupt caches affect speed only. Never persist
   an incomplete snapshot as complete.
5. An identical tree reuses its complete result. For a new tree, re-evaluate path roles
   and committed attributes, remove deleted paths, and reuse blob measurements only when
   all their parsing inputs match. The same OID renamed to another extension or moved
   between source/tests/docs can require different counting or classification. A
   `.gitattributes`-only commit can reclassify unchanged blobs. Retain nonblank and
   selected-code counts as appropriate, keyed by language/counter inputs, to avoid
   unnecessary rereads. Newly included blobs that were previously skipped must be read.
   Classifier changes rebuild; rewritten history/tree changes must produce the same
   result as a cold query.

Tests exercise normal and odd filenames, committed-versus-dirty attributes, local
attribute contamination, explicit false overrides, all roles, binary/LFS, cap, symlink
and gitlink handling, empty and zero-code repos, malformed/truncated batches, and cache
corruption/version invalidation. Compare full cold results with warm results after
content edits, deletions, renames, and attribute-only commits. Count actual requested
blob OIDs through a test seam to prove that unchanged eligible blobs are not reread;
timing alone is not that proof.

## Phase records-snapshot

Own `records.rs` and a separately namespaced records cache. Consume the shared wire and
repo-resolution types; leave query orchestration to the next phase. If the other
snapshot phase extracts common cache helpers, merge that small interface rather than
changing history semantics.

- At the selected committed tip, obtain reachable commits with `rev-list --count`,
  tracked blob entries with NUL-safe `ls-tree`, and minimum/maximum author timestamps
  from a metadata-only Git log. Include merges; do not use the tip's author timestamp as
  a maximum in histories with nonmonotonic author dates.
- Never read record contents or parse footers for this summary: no blob reads, no
  general history/attribution walk. Do not manufacture line counts or lifetime agent
  counts. An unborn empty repo has zero files and commits with null time bounds. An
  unavailable repo has a null summary and an issue.
- Cache successful complete metadata by resolved identity/context, tip, policy, and
  relevant shallow/fetch state. Equal-tip shallow deepening must not reuse incomplete
  counts. Preserve partial coverage for shallow or budget-limited sources, and never
  mark a truncated count complete.
- Test counts, merges and author-time ordering, empty/unavailable/shallow repos,
  corrupted cache recovery, same-tip reuse, and new-tip refresh. Use command-observation
  tests to prove there are no content reads.

## Phase query-integration

Own `code.rs`, the necessary `query.rs` and `output.rs` adjustments, aggregate helpers,
and query/binding tests. Integrate any already-landed Runs and attribution work in
place. Do not rewrite or implement those sibling facets.

1. Separate repo resolution from optional history walks. A code-only request resolves
   the configured sources and measures snapshots without walking code commit history,
   loading the agent directory, or computing Output. Record summaries use their
   metadata-only implementation, not Output's history walker. Output-only requests do
   not read code blobs or records content; Runs-only behavior respects whatever
   dependencies its own landed implementation actually needs. Call expensive facets only
   when requested. Build repo status from the measurements actually requested, so
   missing optional history is not an availability error.
2. Populate each code repo's `snapshot` and each record repo's metadata snapshot only
   for the Code facet. Preserve reference stamps, source issues, and partial/unavailable
   state. In a combined request, merge statuses without overwriting a history or
   snapshot failure with `ok`.
3. Aggregate Code per project and portfolio using semantic repo identity, not
   `(identity, common_dir)`; two clones of the same repository must not double-count.
   Cache storage still distinguishes local object stores. Resolve one measurement per
   identity deterministically for a request and reuse it across projects. Prefer a
   successfully measurable clone over a failed one; if successful clones disagree on the
   selected tip, choose by a stable documented key and emit a disagreement issue instead
   of summing them or choosing according to request order. Preserve the actual chosen
   reference and measurement provenance on displayed rows.
4. Merge role/language counts and calculate shares from totals, never by averaging
   per-repo shares. Records never enter Code denominators or code repo coverage. Missing
   code sources reduce measured/configured coverage; a valid empty code repo still
   counts as measured.
5. Add code and record source tokens, status, and policy versions to the fingerprint.
   Stable queries, dirty files, and cache warmth do not change measurement identity;
   selected tips, source availability, attributes at a changed tip, and classifier
   policy do. Preserve the foundation's existing query-identity behavior and other
   facets' tokens.
6. Extend the public `project_stats_query` binding round-trip with code-only, combined,
   records, unavailable, and zero-code fixtures. Update existing placeholder assertions.
   Check consumers in sase if they have landed, making only necessary
   nullable/additive-wire compatibility changes.

Tests prove facet isolation through observed Git/helper calls, exact project and
portfolio deduplication across projects/clones, request-order stability, mixed
availability, empty repo versus missing repo, role and language sum invariants, nullable
zero denominator, and fingerprint stability/invalidation. Ensure the records tier does
not create extra code commits or model rows.

## Phase measurement

After all implementation is integrated, run the required tokei experiment against the
actual reachable binding path:

1. Record a clean release baseline of the integrated nonblank implementation. Add
   `tokei` with `default-features = false`, use extension lookup plus the SASE shebang
   check and `parse_from_slice` with an explicit `Config`, and plug it into the shared
   counter interface. Do not read ambient `tokei.toml` or walk directories. Regenerate
   the workspace hack with the repository-prescribed hakari commands after dependency
   changes.
2. Measure both baseline and candidate with the same Python >=3.12, Rust toolchain,
   target, release profile, features, strip settings, and fixed build parallelism. Use
   independent empty target directories inside the opened checkout; do not clean another
   agent's or a shared build cache. Warm registry downloads before timing, record the
   exact invocations, compare equivalent actual release wheels, and record absolute
   bytes, MiB delta, clean build seconds, time ratio, and new Cargo.lock packages. Use
   the existing build tooling; never run bare cargo contrary to the repository
   instructions. A small checked build/measurement wrapper is appropriate if needed to
   resolve the same Python/linker environment.
3. Accept tokei only if wheel growth is at most 2 MiB **and** clean release build time
   growth is at most 20%. Otherwise remove tokei and its unused dependency/hack changes
   and retain `counter: nonblank`. Add an implementation policy comment and phase-note
   evidence. A missing measurement is not a passing gate or an invented threshold
   failure; finish the experiment. If load noise makes the threshold outcome ambiguous,
   repeat paired measurements under comparable load and record the spread.
4. Update counter-sensitive tests and assert the selected policy's treatment of comments
   and mixed-language blobs, while docs/context remain nonblank. If tokei survives,
   prove ambient config independence. Verify that a cache created under nonblank cannot
   be reused as a tokei snapshot; policy is part of its identity. Avoid leaving an
   unused alternate implementation or experiment feature switch in production.

Then assemble reproducible performance and compatibility evidence:

- Measure cold Code on the sase checkout's committed tree with an empty isolated cache,
  same-tip warm Code, and warm Code after one new commit. Use a disposable benchmark
  fixture based on the opened checkout's objects for the one-commit case; do not commit
  benchmark changes to project branches or modify the real working tree. Keep all
  temporary data inside the authorized workspace. Record measured ref/OIDs, file count,
  hardware/ toolchain, wall times, cache state, and blob reads. Verify the new-tip warm
  snapshot equals a cold snapshot of that same tip.
- Targets from the parent design are cold LoC 2–5 seconds, warm after one commit below
  0.5 seconds, and warm one-project query below 150 ms. Separate query overhead from
  blob parsing when explaining a miss. Fix local avoidable causes; record any unresolved
  budget miss with concrete evidence as `PROPOSED FOLLOW-UP:` on `sase-1m3.7`, rather
  than concealing it.
- Confirm the counter shown in project/repo/portfolio output matches the recorded tokei
  outcome. If subsequent integration materially changes how the counter is linked or the
  dependency set, refresh the paired release measurements. Do not repeat expensive
  builds without a reason.
- Check the actual Python binding with the updated core and a request over real code
  plus record repos. Cross-check records counts against Git, confirm no contents were
  read for records, and check unknown versus zero.
- Ensure sase's `sase-core-revision.txt` includes the landed implementation before
  downstream features depend on it. Let host finalization update the pin when committing
  both repos; if needed after the core changes have landed, use the documented
  `just ratchet-core-revision` flow. Never hand invent a SHA. Read `docs_writing.md`
  before any documentation edit and `lint_and_test.md` before finishing tracked sase
  changes.

This phase records evidence and closes only its assigned child phase. It does not close
`sase-1m3.7` or any ancestor.

## Verification and completion rules

Each implementation phase runs `just fmt`, meaningful targeted tests through the
repository recipes, and `sase tool run check` in every repository it changes. Never run
raw guarded `just check` or `check-full`. The targeted Rust command is
`just test -p sase_core project_stats`; binding changes also require
`just test -p sase_core_py project_stats`. The full wrapped check remains required.
Rebuild a stale local sase venv through its documented `just install-venv` workflow when
exercising changed Rust bindings.

Use `/sase_monitor` before long builds/checks, with a continuation prompt that states
the exact phase, experiment state, output paths, and remaining verification. Wait for
the monitor handoff command itself to exit; do not end a turn with an unmanaged running
process. Use host-owned finalization for implementation commits, not manual branches,
commits, or PRs.

No worker creates beads or edits memory. Discovered follow-ups go on the assigned child
phase, with parent-wide budget and compatibility evidence also summarized on
`sase-1m3.7` for the outer epic's land agent. A check failure reproduced identically on
a clean base is recorded with evidence and any existing tracking bead; it does not leave
a finished phase open. Before closing any assigned phase, run
`sase bead epic-symbols <phase-id>` and resolve or re-key every listed symbol to a
still-open owner. Close only the assigned phase with a verification note.

The **child epic's land agent**, after all child phases have landed, verifies the
complete scope above, summarizes dependency/performance/check evidence on `sase-1m3.7`,
and checks `sase bead epic-symbols sase-1m3.7`. Its landing and parent-phase completion
follow the host's child-epic lifecycle; if the host has not already completed the
assigned parent phase, close only `sase-1m3.7` with that evidence once the child epic is
complete. Child phase workers must treat this paragraph as land-agent preparation, not
authority to close an ancestor. Nobody in this child epic closes `sase-1m3` or another
ancestor plan bead.
