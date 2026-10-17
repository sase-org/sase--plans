---
tier: epic
title: Project statistics core engine for sase-1m3.4
goal:
  Complete the Rust project_stats API, hardened Git execution, cached full-DAG history,
  Output and lifetime aggregation, and Python binding required by sase-1m3.4, leaving
  stable extension points for its sibling facets.
phases:
  - id: wire-seams
    title: Complete wire types and facet extension points
    depends_on: []
    size: medium
    description:
      "wire-seams: define the complete project_stats request and response contracts,
      internal history and directory types, validation, and empty facet seams without
      adding user-facing callers."
  - id: git-runner
    title: Harden shared Git execution and add bounded batch input
    depends_on: []
    size: medium
    description:
      "git-runner: scrub repository-routing environment variables in
      file_history::runner and add concurrent stdin and stdout handling with timeout,
      output-cap, and cleanup tests."
  - id: history-cache
    title: Resolve repositories and cache complete commit history
    depends_on:
      - wire-seams
      - git-runner
    size: medium
    description:
      "history-cache: implement repository identity and ref validation, full-DAG footer
      parsing, incremental history folding, atomic cache persistence, and repository and
      cache regression fixtures."
  - id: query-binding
    title: Aggregate Output and lifetime statistics and expose the binding
    depends_on:
      - history-cache
    size: medium
    description:
      "query-binding: implement footer attribution, deduplicated project and portfolio
      aggregates, coverage and fingerprints, query dispatch, the project_stats_query
      binding, and measured cold and warm history acceptance."
proposed_by: bbugyi200.apollo.sase-1m3.4
parent_bead: sase-1m3.4
create_time: 2026-10-16 22:15:58
status: wip
---

- **PROMPT:**
  [prompts/202610/project_stats_core_engine.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/project_stats_core_engine.md)
- **PARENT:**
  [202610/sase_stats_project.md](https://github.com/sase-org/sase--plans/blob/main/202610/sase_stats_project.md)

# Project statistics core engine for sase-1m3.4

## Outcome and boundaries

This is a child epic for the assigned phase `sase-1m3.4`, implementing the `core-engine`
section of `plan:202610/sase_stats_project.md`. That phase had no previous
implementation or remainder notes when inspected. Neither
`crates/sase_core/src/project_stats/` nor its Python binding exists on the inspected
base. Four bounded coding phases separate the full wire contract, shared subprocess
behavior, persisted history, and aggregation. The first two can run independently; the
remaining dependencies are explicit above.

All implementation changes belong to the linked `sase-core` repository. Each worker must
use `/sase_repo` to open `sase-core` with an audit reason and read its `AGENTS.md`. Use
the returned checkout only. This plan introduces no sase Python callers, CLI, UI, memory
edits, new dependency, or manual pin update. The sibling phases `sase-1m3.5` through
`.8` consume this engine after `.4` closes; their Runs, directory/plan, snapshot, and
loader implementations remain outside this child epic.

The parent decision `stats_decision_record = no` is final. Do not re-ask it or edit any
memory note. A `PROPOSED FOLLOW-UP:` for the skipped decision record is already recorded
on `sase-1m3.4`; do not duplicate it.

Native helpers, phase workers, and the planning agent must not close `sase-1m3.4`
prematurely. The child epic's land agent owns combined verification and closure of that
assigned phase after the child work is complete. Never close `sase-1m3` or any other
ancestor plan bead. Do not create task beads: record discovered work as
`PROPOSED FOLLOW-UP:` notes on the assigned phase, citing existing task IDs when known.
Host finalizers own commits and branch operations.

## Verified building blocks and conventions

- `file_history/runner.rs` already drains stdout concurrently, preserves an output cap,
  kills timed-out children, disables optional locks and prompts, and provides
  checked/unchecked runners and revision validation. It does not scrub ambient
  repository-routing variables or accept batch stdin.
- `memory_history/cache.rs` has versioned SHA-256 cache keys, equal-tip reuse, ancestry
  folds, corrupt-file fallback, an `(mtime, length)` process memo, writer-only `fs2`
  locking, tempfile replacement, and file/directory fsync. Mirror these small helpers
  here; a cross-domain cache refactor is unnecessary scope.
- `commit_footer::parse_commit_footer` returns ordered normalized keys and resolved link
  labels. Its first matching `AGENT`/`MODEL` tags are the source of provenance.
- `chrono`, `chrono-tz`, `fs2`, `sha2`, and `tempfile` are already available.
- `agent_stats/wire.rs` uses integer Unix timestamps and count fields; its run query is
  the future Runs seam's source, not something to reimplement now.
- `sase_core_py/src/memory_history/` demonstrates dict parsing, `py.allow_threads`,
  exception mapping, serialization, registration, and binding tests.

Use free functions over serde `*Wire` types and a `thiserror` error enum. Add the
alphabetical module declaration in core `lib.rs`; do not add root re-exports or prelude
aliases. Keep each new file under 1,500 lines and format to the repository's 80-column
rustfmt policy. New multi-file `mod.rs` files are facades containing module declarations
and re-exports; put binding function bodies in a sibling file.

## Shared contract

Names and fields below implement parent sections 4–6 and 12. Fields can grow additively,
but later phases must not need to replace the response structure. Use typed enums with
snake-case wire values. Counts are unsigned; timestamps are integer Unix seconds;
nullable measurements use `Option`. Never omit an optional facet from JSON:
absent/not-requested values serialize as `null`.

### Request and validation

Define `PROJECT_STATS_WIRE_SCHEMA_VERSION`, `PROJECT_STATS_METRIC_POLICY_VERSION`, and
`PROJECT_STATS_CLASSIFIER_VERSION`, all initially `1`, with `ProjectStatsRequestWire`
containing:

| Area               | Fields                                                                                                                |
| ------------------ | --------------------------------------------------------------------------------------------------------------------- |
| Identity and clock | `schema_version`, `start_ts`, `end_ts` (exclusive), `now_ts`, `timezone` (IANA)                                       |
| Facets and cache   | `facets: [output, code, runs]`, `cache_dir` (empty disables cache)                                                    |
| Agent sources      | optional `run_index_path`, `agents_sidecars: [{project_key, path}]`, `local_agent_name_prefix`, `known_model_aliases` |
| Run policy         | `record_repo_names`, `include_run_only_projects`                                                                      |
| Bounds             | `budget: {timeout_ms, max_output_bytes}` with generous nonzero defaults                                               |
| Selection          | `projects: [ProjectStatsProjectSpecWire]`                                                                             |

Each project spec has `key`, `label`, `state: enabled|disabled`,
`selection: always|if_runs`, `index_project_names`, `code_repos`, `record_repos`,
`not_counted`, and `plan_store`.

`ProjectStatsRepoSpecWire` has `id`, `name`, `kind: primary|linked|sidecar`, `path`,
`identity` (canonical remote or empty), and ordered `ref_candidates`. `not_counted`
entries have `name` and `reason: user_linked|external`. The plan-store descriptor has
`storage: in_tree|local|separate_repo|sidecar_repos`, optional `repo_path`, `plans_dir`,
and optional `project_filter`.

Reject unsupported schema versions, invalid IANA timezones, reversed ranges, invalid
bounds, and structurally invalid request identities as request errors. An empty range is
valid and yields measured zero counts where sources are available. Filesystem absence
and Git failures are per-source issues, not fatal request errors. Do not silently
substitute a different timezone or widen the supplied range.

### Response

`ProjectStatsResponseWire` contains versions (`schema_version`, `metric_policy_version`,
`classifier_version`, `engine_version`), `range: {start_ts, end_ts}`, `fingerprint`,
`projects`, `portfolio`, and `issues`. An issue has optional `project` and `repo`, plus
`code`, `message`, and optional `hint`.

`ProjectStatsProjectWire` contains `key`, `label`,
`state: enabled|disabled|unregistered`, nullable `output`, `lifetime`, `code`, and
`runs`, plus `repos`, `not_counted`, `models`, `patches`, `truncated_patch_rows`, and
`coverage`.

| Value     | Required fields                                                                                                                                                                              |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Output    | `commits`, `agent_commits`, `agent_share`, nullable `plans_committed`, `plans_epic`, `plans_tale`, `models`, `commit_coverage`, `plan_coverage`                                              |
| Lifetime  | `first_commit_ts`, `last_commit_ts`, `age_days`, `active_days`, `commits`, `agent_commits`                                                                                                   |
| Code      | `counter: sloc                                                                                                                                                                               | nonblank`, `code_files`, `code_lines`, `test_lines`, nullable `test_share`, `doc_files`, `doc_lines`, `agent_context_files`, `agent_context_lines`, `languages`, `repos_configured`, `repos_measured` |
| Runs      | `runs`, `completed`, `failed`, `other_terminal`, `in_progress`, `waiting`, nullable `success_rate`, `total_runtime_seconds`, `distinct_patches`, `unattributed_runs`, nullable `last_run_ts` |
| Model row | `kind: resolved                                                                                                                                                                              | unknown                                                                                                                                                                                               | no_agent`, `provider`, `model`, `model_is_alias`, `commits`, `plans`, `plans_epic`, `plans_tale`, `unknown_reasons` |
| Patch row | `name`, `status`, `has_pr`, `runs`, `completed`, `failed`, `run_commits`, `total_runtime_seconds`, `first_run_ts`, `last_run_ts`                                                             |
| Repo row  | `id`, `name`, `kind`, `tier: code                                                                                                                                                            | records`, `status: ok                                                                                                                                                                                 | unavailable                                                                                                         | partial`, optional `issue`, optional `ref`, optional `output`, `lifetime`, and `snapshot` |
| Ref       | `name`, `commit`, `shallow`, optional `fetched_at`                                                                                                                                           |

Define typed language, coverage, repo-output, snapshot, and portfolio structures
alongside these rows. Coverage must expose configured/measured/partial/unavailable
repository counts separately from agent-tagged/resolved/unknown attribution counts and
unknown-reason counts. Include a typed records snapshot extension for the later Code
phase's tracked-files and lifetime-commit statistics. Portfolio carries deduplicated
versions of the relevant aggregates and model rows.

Unrequested facets are `null`. A requested implemented facet with an unavailable source
is present with nullable measurements and an issue; measured zero is `0`. When some
sources succeed, show measured subtotals with explicit incomplete coverage and issues.
Empty repositories are measured zero, not missing repositories; their date endpoints and
shares without a denominator are `null`.

The parent explicitly permits not-yet-implemented seams to return `None`: until their
sibling phases land, Code and Runs and plan measurements are `null`, not fabricated
zeros. Directory is empty. No user-facing caller is introduced in this intermediate
state. Real missing-source reporting for those later facets belongs to their owners.

### Internal seams

Provide core-internal `HistoryRow` (SHA, author/committer timestamps, merge flag,
optional agent and model tag), resolved-repo metadata, and a history outcome carrying
completeness and issues. Cache raw history rows, not attributed aggregates.

Define a `Directory` type capable of name-to-member metadata, family-to-members, and
SHA-to-member lookups; distinguish missing entries from entries with empty model
metadata. Its initial loader returns an empty value. Future directory loading must
remain read-only and must not import agent state. Define facet context/results that can
carry values, issues, and source fingerprint tokens. Later phase workers should be able
to implement `runs.rs`, `directory.rs`, `plans.rs`, and `snapshot.rs` with small query
dispatch changes, preserving this wire.

## Phase wire-seams

Create the complete `project_stats` wire and error types, validation, the four facet
seams, and a query scaffold. Split request and response source files if useful and
re-export from `wire.rs`. Export the domain through its facade without adding a Python
binding yet. Do not implement Git/history/cache logic in this phase.

The scaffold must compile and expose explicit unimplemented-facet `None` values; it must
not assert a successful measured Output before history exists. Establish internal
interfaces for the history and query phases. Put the shared extension contract and field
defaults in rustdoc next to their types.

Verify serde round trips for the complete request/response, explicit JSON nulls,
version/range/timezone validation, enum spellings, all project states, and model bucket
variants. Include representative nonzero Code, Runs, plans, Patch, and records fixtures
so those fields are real typed contracts rather than empty JSON placeholders. Run the
repo gate after formatting.

## Phase git-runner

Change only the shared `file_history` runner, exports, and its tests. Preserve existing
checked/unchecked signatures and their exit-code/truncation behavior. Centralize command
construction so every new and existing runner variant:

- removes `GIT_DIR`, `GIT_WORK_TREE`, `GIT_INDEX_FILE`, `GIT_OBJECT_DIRECTORY`,
  `GIT_ALTERNATE_OBJECT_DIRECTORIES`, `GIT_COMMON_DIR`, and `GIT_NAMESPACE`;
- sets `GIT_NO_LAZY_FETCH=1`, `GIT_OPTIONAL_LOCKS=0`, and `GIT_TERMINAL_PROMPT=0`,
  retaining `--no-optional-locks` and existing output config;
- executes argument vectors without a shell.

Add a bounded batch-input helper using a writer thread concurrently with the stdout
drainer. Close stdin on completion, honor the same whole-command deadline and output
cap, and kill/reap the child and join workers on timeout and error paths. Detect
read/write failure rather than returning plausible successful partial output. Keep
draining/discarding excess output so exceeding the byte cap cannot deadlock. Callers
must still inspect the truncation flag even when Git exits zero.

Tests must cover input and output both exceeding pipe capacity, ordinary batch success,
nonzero Git exit, output truncation, timeout and early exit/BrokenPipe, and all
environment removals. Avoid unsynchronized process-environment mutation in parallel
tests: use an isolated subprocess or injected command environment.

Do not migrate `file_history::read_blobs` or `memory_history::causes::batched_diff_tree`
in this phase. Their protocol-specific parsing makes migration a separate change. Record
that explicit parent-permitted deferral as a `PROPOSED FOLLOW-UP:` on the assigned
phase, with the two paths and their existing synchronous stdin risk. Later project_stats
facets must use the new helper from their first implementation. Run the repo gate after
formatting.

## Phase history-cache

Implement `project_stats/git.rs`, `history.rs`, and `cache.rs` using the agreed wire and
hardened runner. Add hermetic temporary-repo helpers reusable by the final phase.

### Repository resolution

Canonicalize both the requested path and `rev-parse --show-toplevel`; they must match. A
missing nested clone must not cause Git to use an ancestor checkout. Report
`identity_mismatch` or the appropriate unavailable-source issue. Reject unsafe revision
tokens. Resolve the first candidate that verifies to a commit and keep both its supplied
name and full OID; never substitute HEAD for a failed candidate unless HEAD itself is in
the ordered candidates. Preserve valid empty-repo handling.

Collect the canonical Git common directory, shallow state, and the mtime of `FETCH_HEAD`
resolved through Git's path facilities. Identity falls back from the supplied canonical
remote to a deterministic root-commit identity, then realpath; multiple roots must have
a deterministic treatment. Query caches must not conflate different common directories
sharing a remote. Report shallow coverage explicitly.

### Walk and parse

Read the full DAG at the resolved OID with
`git log -z --format=%H%x1f%P%x1f%at%x1f%ct%x1f%B <tip>`. Do not use first-parent or
range/date pruning; non-monotonic commit dates make that incorrect. Parse NUL-delimited
records and split only the first four field separators so message text does not alter
framing. Parse the terminal footer once per new row with `parse_commit_footer`, taking
the first normalized `AGENT` and `MODEL` keys and their resolved labels. Merge detection
comes from parent count. Raw prose that merely mentions a footer key does not count.

A byte or timeout limit keeps only complete records, marks the repo partial, and emits
`history_truncated`. Parse failures are explicit issues, not silently skipped history
advertised as complete. Truncated output must never become an equal-tip complete cache
hit. A real empty repository returns an empty complete history.

### Cache

Store versioned JSON at `{cache_dir}/v{schema}/{sha256(key)}.json`. Include identity,
canonical common dir, and relevant parser/policy versions in keys; hold the walked tip
and completeness metadata in the entry. Empty `cache_dir` bypasses reads and writes.
Equal complete tips reuse rows. If the cached tip is an ancestor, walk
`cached_tip..tip`, merge by SHA, and preserve old rows. Rewinds, rewritten history,
missing old tips, or invalid entries rebuild. A partial entry is never promoted to
complete without a successful walk. Shallow-to-complete history changes must not reuse
stale rows merely because the tip stayed equal.

Use a writer-only sibling lock, atomic tempfile replacement, and fsync. Readers do not
take writer locks. Missing, corrupt, unreadable, or wrong-schema entries rebuild; cache
write failure still returns the in-memory result. Memoize by path, mtime with
nanoseconds, and length. Do not persist attributed model rows: changed directory data
must re-attribute old commits without a new Git walk.

### Verification

Test merge commits and complete DAG traversal, bounded record framing, linked footer
labels and repeated keys, empty and missing/nested repos, canonicalized path aliases,
ref precedence, inherited `GIT_DIR`, shallow clones, equal-tip reuse,
incremental-versus-cold equality, force-push rebuild, corrupt/wrong-schema cache, disk
reload independent of the memo, cache-write failure, and retry after truncation. Set
identity and signing config in every temporary repo that commits. Run the repo gate
after formatting.

## Phase query-binding

Implement `attribution.rs`, `aggregate.rs`, and `query.rs`; replace the query scaffold
with real Output and lifetime behavior and wire the four existing seams.

### Attribution

An absent `AGENT` tag always yields the `no_agent` bucket (`∅`), even if MODEL is
present. With an agent tag, a MODEL value split at the first `/` is authoritative;
validate nonempty components. A malformed tag yields `unknown` (`?`) with
`malformed_model_tag` and does not fall back to another source. Keep recorded
provider/model identities without guessing concrete models; flag known aliases.

Pass the Directory value through the resolver. This phase's loader returns empty, so an
unstamped agent commit resolves to `?` / `page_unpublished`. Leave explicit extension
points for the sibling attribution phase to implement exact-SHA lookup, then
`<agent>--code`, exact agent name, and unambiguous family-model lookup; its other
reserved reasons are `page_metadata_empty` and `session_ambiguous`. Do not implement
sidecar/index loading or plan counting here.

Every model table includes unknown and no-agent rows even at zero. Sort resolved rows by
count with deterministic provider/model ties. Commit, plan, and unknown reason fields
share one model identity; currently unavailable plan measurements must remain
distinguishable from measured plan zeros.

### Aggregation and dispatch

Resolve and walk unique repositories once per query. Project totals union code-repo
SHAs; portfolio totals union all selected code-repo SHAs, rather than summing project
counts. Deduplicate repeated repo identities in coverage and measurement. Record repos
remain visible as records-tier metadata but do not enter code commit totals. Populate
per-repo Output and lifetime as well as project and portfolio aggregates.

Output includes commits with `start_ts <= committer_ts < end_ts`, merges included.
Lifetime always uses all available rows with author timestamps: first/last commit, age
as whole nonnegative days from the earliest author timestamp to `now_ts`, and distinct
local author dates under the supplied `chrono_tz` timezone. Return lifetime with Output
even when Code is not requested. Do not use the Output range to derive lifetime facts.
Preserve empty-denominator nulls and measured-zero rules.

Partial or unavailable repos reduce coverage and carry issues; failed sources must not
erase successful sources. Distinguish unknown attribution on an available commit from
unavailable source data. Enforce, with tests and debug assertions, that model rows sum
to measured commits, provider subtotals sum to the same count, resolved plus unknown
agent counts equal agent-tagged counts, and share numerators never exceed denominators.
Source-incomplete totals must be labeled as such.

Dispatch requested facets through the seam modules. `selection: if_runs` filtering,
unregistered run-only projects, and full run-source semantics are implemented by the
later Runs phase: preserve the inputs and a clear dispatch hook without making up run
counts. The same applies to plan and snapshot results. Build a canonical fingerprint
from the semantic request (range, timezone, now-dependent lifetime, selection, aliases
and facet inputs), engine/policy versions, resolved repo identities and tips/source
health, and source tokens supplied by seams. Reserve integration for plan-store tips,
sidecar HEADs, and run-index `(path, mtime, length)` tokens. Exclude cache hit/miss
status, elapsed timings, and cache location so a cold and warm query have the same
fingerprint and data. Keep ordering stable.

### Binding and acceptance

Add `crates/sase_core_py/src/project_stats/` and register
`project_stats_query(request: dict) -> dict` in the extension's module initialization.
Use the existing JSON conversion helpers, `py.allow_threads`, and core error to Python
exception mapping. Test module registration by Python-visible name and a nontrivial dict
round trip. Include an invalid-request binding test.

Use deterministic temporary repositories to verify both footer spellings, authoritative
and malformed MODEL tags, repeated tags, body prose, absent-agent MODEL tags,
unknown/no-agent zero rows, model aliases, merge inclusion, shared-history SHA dedupe
within and across projects, per-repo totals, half-open range endpoints, non-monotonic
author/committer dates, New York DST author-day bucketing, mixed
available/unavailable/partial sources, all-facet serialization, and stable
cold/warm/cache-disabled results and fingerprints. Reuse history fixtures rather than
writing a second Git harness.

Measure fresh-cache and repeated warm Output/history queries against the current sase
checkout, with a fixed range and `now_ts`, from a temporary local harness using the new
binding or domain query. Record history count, tip, budget, cold and warm timings, cache
state, and whether timing includes extension startup on this phase's bead and summarize
on `sase-1m3.4`. Target warm one-project query under 150 ms; the parent's combined cold
history/directory/plans target is about 1–1.5 seconds, but directory/plans are not
available in this child epic, so do not claim that combined target has been met. Use a
few repetitions and report the median rather than adding a timing assertion to CI.
Investigate repeated history walks on warm requests; record any remaining performance
gap precisely.

Run focused domain and binding tests during iteration, then format and run the repo
gate. No Python sase code calls this new binding yet, so pin advancement belongs to the
later caller phase or the host's automatic cross-repo pin handling.

## Verification and completion ownership

Never run bare cargo. In `sase-core`, use `just test -p sase_core <filter>` and
`just test -p sase_core_py <filter>` for iteration, `just fmt`, then
`sase tool run check` as the required gate. A targeted test does not replace that gate.
Never run `just check-full`. Use `/sase_monitor` for a long-running gate and wait for
the monitor handoff command itself to exit before ending the turn. If a worker changes a
tracked sase file despite this scope, it must also read `lint_and_test.md` through
`/sase_memory_read` and satisfy that repository's gate.

Each child worker closes only its own child phase after validation, checks
`sase bead epic-symbols <its-phase>` first, and returns implementation/verification
evidence. Do not claim `sase-1m3.4` finished from any one child phase. Do not hand-set
bead lifecycle state or create commits.

The child land agent verifies the integrated core query and binding, that all four child
phases and their descendants are complete, that the full gate evidence is current for
the integrated tree, and that measured history performance is recorded. It compares the
implementation with the full contract above and parent section 12, particularly the null
seams later sibling phases require. Then run `sase bead epic-symbols sase-1m3.4`,
resolve any symbols or re-key them to a genuinely open consumer, and close only the
assigned outer phase with
`sase bead close sase-1m3.4 --note "<integrated verification and measurements>"` after
its descendant-close requirements have been met. Child plan completion must not close
the ancestor epic `sase-1m3`.

A gate failure that reproduces identically on the clean base is recorded as a
`PROPOSED FOLLOW-UP:` with the reproduction and any existing tracking bead; it does not
leave the assigned phase open. Fix failures introduced by this work before closure.
Preserve host-owned finalization for every modified repository.
