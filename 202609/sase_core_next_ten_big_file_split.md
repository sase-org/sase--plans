---
tier: epic
title: Split The Next Ten Largest sase-core Rust Files Into <=1500 Line Modules
goal: 'Each of the ten largest Rust files remaining in the sase-core repo after epic
  sase-14s is decomposed into a module tree whose every file is at most 1500 lines,
  with no behavior change, no public API change, and `just check` green after each
  phase.

  '
phases:
- id: bead_cli
  title: Split crates/sase_core/src/bead/cli.rs
  depends_on: []
  size: medium
  description: 'bead_cli: decompose the 4,276-line bead CLI module into dispatch,
    per-command handler, argument parsing, and rendering submodules under bead/cli/.'
- id: federation_worker
  title: Split crates/sase_gateway/src/federation_worker.rs
  depends_on:
  - bead_cli
  size: medium
  description: 'federation_worker: decompose the 3,953-line gateway federation worker,
    including its 1,950-line cfg(unix) imp module, into a federation_worker/ tree
    without disturbing its platform gating.'
- id: agent_stats_run
  title: Split crates/sase_core/src/agent_stats/run.rs
  depends_on:
  - federation_worker
  size: medium
  description: 'agent_stats_run: decompose the 3,732-line run-stats aggregation module
    into query, per-dimension fold, attribution, and finishing submodules, and split
    its 2,250-line test block.'
- id: fleet_reads
  title: Split crates/sase_gateway/src/fleet_reads.rs
  depends_on:
  - agent_stats_run
  size: medium
  description: 'fleet_reads: decompose the 3,371-line gateway fleet read service into
    service, snapshot, record resolution, content, and invalidation-hub submodules.'
- id: bead_events
  title: Split crates/sase_core/src/bead/events.rs
  depends_on:
  - fleet_reads
  size: medium
  description: 'bead_events: decompose the 3,314-line bead event-stream module into
    wire, import, reduction, and merge/relocation submodules under bead/events/.'
- id: tool_run_store
  title: Split crates/sase_core/src/tool_run/store.rs
  depends_on:
  - bead_events
  size: medium
  description: 'tool_run_store: decompose the 3,229-line SQLite tool-run store into
    connection/schema, run lifecycle, query, and retention submodules under tool_run/store/.'
- id: provider_usage_tests
  title: Split crates/sase_core/src/provider_usage/tests.rs
  depends_on:
  - tool_run_store
  size: medium
  description: 'provider_usage_tests: split the 3,176-line provider_usage test file
    into a provider_usage/tests/ directory keyed by the production module each group
    covers.'
- id: editor_directive
  title: Split crates/sase_core/src/editor/directive.rs
  depends_on:
  - provider_usage_tests
  size: medium
  description: 'editor_directive: decompose the 2,952-line editor directive module
    into metadata tables, contract lookup, completion candidates, and context detection
    submodules under editor/directive/.'
- id: runner_capacity
  title: Split crates/sase_core/src/runner_capacity.rs
  depends_on:
  - editor_directive
  size: medium
  description: 'runner_capacity: decompose the 2,938-line runner capacity policy module
    into wire, claims/lineage, waiters, candidate decision, holds, and capacity math
    submodules under runner_capacity/.'
- id: notification_store_parity
  title: Split crates/sase_core/tests/notification_store_parity.rs
  depends_on:
  - runner_capacity
  size: medium
  description: 'notification_store_parity: split the 2,825-line notification store
    integration test into a single-binary tests/notification_store_parity/ directory
    and close out the epic''s file-size invariant.'
proposed_by: bbugyi200.athena.0oh.r0
create_time: 2026-09-21 11:31:32
status: wip
bead_id: sase-15b
---

- **PROMPT:** [prompts/202609/sase_core_next_ten_big_file_split.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sase_core_next_ten_big_file_split.md)
- **BEAD:** [sase-15b](https://github.com/sase-org/sase--beads/blob/main/pages/sase-15b/README.md)

# Plan: Split The Next Ten Largest sase-core Rust Files Into <=1500 Line Modules

This epic continues the work of epic `sase-14s`
(`plan:202609/sase_core_big_file_split.md`), which split the ten largest sase-core Rust
files at the time. After that epic, the repo still holds many Rust files well over 1500
lines. This epic takes the next ten.

Every phase does the same kind of work on a different file: take one oversized Rust
source file in the sase-core repo and split it into a module tree in which no file is
longer than 1500 lines. The phases run strictly in sequence (each `depends_on` the one
before it). That way only one agent edits the sase-core working tree at a time, and each
phase starts from a tree that already passed `just check`.

## How The Targets Were Chosen

The targets are the ten largest git-tracked `.rs` files in sase-core by `wc -l`, as of
sase-core `origin/master` at `3e346b0`:

| #   | File                                                  | Lines |
| --- | ----------------------------------------------------- | ----- |
| 1   | `crates/sase_core/src/bead/cli.rs`                    | 4,276 |
| 2   | `crates/sase_gateway/src/federation_worker.rs`        | 3,953 |
| 3   | `crates/sase_core/src/agent_stats/run.rs`             | 3,732 |
| 4   | `crates/sase_gateway/src/fleet_reads.rs`              | 3,371 |
| 5   | `crates/sase_core/src/bead/events.rs`                 | 3,314 |
| 6   | `crates/sase_core/src/tool_run/store.rs`              | 3,229 |
| 7   | `crates/sase_core/src/provider_usage/tests.rs`        | 3,176 |
| 8   | `crates/sase_core/src/editor/directive.rs`            | 2,952 |
| 9   | `crates/sase_core/src/runner_capacity.rs`             | 2,938 |
| 10  | `crates/sase_core/tests/notification_store_parity.rs` | 2,825 |

One larger file is left out on purpose: `crates/sase_gateway/src/sudo_runner.rs` (4,829
lines at audit time). Phase `sase-14s.10` of the previous epic already split it into a
`sudo_runner/` tree (largest file 701 lines) and closed. That epic's land step had not
reached `origin/master` when this plan was written. Do not touch `sudo_runner.rs` in
this epic.

Test-only files count. Two targets are pure test code (#7 and #10). The goal is that
every Rust file stays at or below 1500 lines, and test files are Rust files too.

Line counts drift. Re-measure your target when you start. If it has shrunk below 1500
lines since this audit, say so in your final response and stop without splitting it.

## Shared Rules For Every Phase

Every phase agent MUST follow all of the rules in this section. The per-phase sections
below do not repeat them.

### Opening The Repository

sase-core is a linked repo, not your workspace checkout. Use the `/sase_repo` skill
first:

```bash
sase repo open sase-core -r "Split <target file> into <=1500 line modules"
```

Use the path it prints as the only path for reads and writes. Do not clone, locate, or
web-fetch sase-core any other way. You will modify it, so sase-core becomes a repository
obligation in your `/sase_final` declaration and needs a `commit` decision.

### Think Hard About The Split

This is the core of the work, and it is a design task, not a mechanical one. **Think
hard about the best way to split your file before you move a single line.** A careless
split turns one hard-to-navigate file into several that are just as hard. A good split
leaves a tree where the file names alone tell a new reader where each concept lives.
Before editing:

1. Read the entire target file. List its top-level items (types, free functions, `impl`
   blocks, constants, statics, nested modules, test modules) with their line ranges.
2. Find the _conceptual seams_: which items form a cohesive unit that a reader would
   expect to find together, and which items only sit next to each other because code was
   appended in that order. Prefer seams that follow the file's own domain vocabulary
   over arbitrary "part 1 / part 2" cuts.
3. Map the internal call graph and shared state. A good seam minimizes the number of
   items whose visibility must widen from private to `pub(super)`/`pub(crate)` just to
   keep compiling. If a candidate seam forces dozens of visibility changes, that is
   evidence it cuts a tightly coupled cluster in half.
4. Sketch at least two candidate decompositions. Weigh them on cohesion, the size of the
   visibility-widening diff, how evenly they spread lines, and how well each name
   predicts its contents. In your final response, say which one you chose and why you
   rejected the other. Also name any candidate split you considered and dropped because
   it would have cut a tightly coupled cluster in half.
5. Decide where the tests go before you move production code (see the size invariant
   below), so tests and production code can share one seam map.
6. Only then start moving code.

The per-phase sections below list seams visible at audit time. They are a starting point
for your own analysis, not the answer. If your analysis finds a better decomposition,
use it and explain why.

Numbered or generic module names (`part1.rs`, `impl_a.rs`, `misc.rs`, `helpers.rs`,
`utils.rs`, `common.rs`, `other.rs`) are a sign the seam analysis was skipped. Do not
use them. Every new module's name should say what lives in it. The one accepted
exception is `support.rs`, and only inside a `tests/` directory, for shared test fixture
builders. The prior epic established that convention (see
`editor/completion/tests/support.rs` and `sase_gateway/src/routes/tests/support.rs`).

### The Size Invariant

- Every `.rs` file in the target's module tree must end at **1500 lines or fewer** after
  the phase, including test modules.
- Test code counts. If the file's test block is itself larger than 1500 lines, split it
  along the same seams as the production code, so each test file sits beside the code it
  covers.
- Aim comfortably below the ceiling (roughly 400-1200 lines per file) so the next
  routine edit does not breach it again right away. Do not create a swarm of 50-line
  files either. Cohesion beats line-count optimization.
- If a genuinely indivisible unit cannot get under 1500 lines without harming the code
  (for example one very long `match` or one huge function), leave it. Say explicitly in
  your final response which file is still over and why.

### Follow The Repository's Existing Module Convention

sase-core already has the convention this epic wants. Do not invent a new one. The
previous epic produced several fresh examples: `crates/sase_core/src/agent_scan/index/`,
`crates/sase_core/src/xprompt_catalog/`, `crates/sase_core/src/editor/completion/`, and
`crates/sase_gateway/src/routes/`. Older examples include `crates/sase_core/src/query/`
and `crates/sase_core/src/gate_decision/`. The shape is:

- A `foo.rs` becomes `foo/mod.rs` plus sibling submodules.
- `mod.rs` keeps the module's doc comment, declares the submodules, and re-exports the
  module's public surface with `pub use`. Keep `mod.rs` thin where practical.
- File-based test modules are declared as `#[cfg(test)] mod tests;`, with the body in
  `tests.rs`, or in a `tests/` directory with its own `mod.rs` when the tests need more
  than one file.

Do not use `#[path = "..."]` attributes to fake a flat layout.

### Pure Refactor, No Behavior Change

- **No public API change.** Every path external callers use today must still resolve.
  That includes items reachable only through a `pub mod` path (for example
  `sase_core::bead::events::<fn>`) and not re-exported at a parent. Re-export moved
  items from the original module path with `pub use`. Other crates in the workspace
  (`sase_gateway`, `sase_xprompt_lsp`, `sase_core_py`), integration tests under
  `crates/*/tests/`, and the Python `sase_core_rs` binding must keep compiling
  unchanged.
- Move code **verbatim** wherever possible. Change `use` statements, visibility
  (`pub(crate)`, `pub(super)`), and nothing else. Do not rename, reorder statements
  inside a function, simplify, or "improve" the code you move. That makes the diff
  unreviewable and hides real regressions. Genuine cleanups belong in a follow-up, not
  here.
- Keep serde attributes, field order, enum variant names, `#[serde(default = ...)]`
  functions, and doc comments intact on every moved type. For `*Wire` types, any change
  there breaks the wire format.
- Keep every `#[cfg(...)]` gate on the items it guards. When code moves out of a
  cfg-gated module, the new file or `mod` declaration must carry the same gate.
- No test may be deleted, skipped, weakened, or left behind. The test count before and
  after the phase must match. When a test moves, it moves whole. Each phase section
  gives the `#[test]`/`#[tokio::test]` attribute count found in the target at audit time
  as a sanity baseline. Measure your own before/after numbers; a
  `cargo test ... -- --list` count for the affected test target is the most reliable.
- Do not edit `[workspace.package].version`, crate `[package].version`, or local
  path-dependency version pins. release-plz owns those (see sase-core `AGENTS.md`).
- Stay in scope. Touch only your target, the new files you create from it, and the
  minimal `use`/`mod` lines elsewhere needed to keep things compiling. Do not split
  other oversized files, even neighbors in the same directory.

### Verification

Run the repo's own gate from the sase-core repo root:

```bash
just check
```

`just check` runs `./scripts/check.sh all` (fmt, clippy, test), the same gate CI runs.
Per sase-core `AGENTS.md`, never verify with `cargo test -p sase_core` alone, because it
skips the `sase_core_py` binding tests. The phase is not done until `just check` passes.
If the run is slow enough to block your turn, use `/sase_monitor`.

If a test that has nothing to do with your file fails once and passes on an unchanged
re-run, it is a flake, not your regression. Record it as a `PROPOSED FOLLOW-UP:` note on
your own phase bead (see below) and continue.

Before finishing, also confirm the invariant directly from the sase-core repo root:

```bash
find crates -name '*.rs' -not -path '*/target/*' -exec wc -l {} + \
  | sort -rn | awk '$1 > 1500' | head -40
```

Your target file, and every file you created from it, must be absent from that list
unless you explicitly justify it as an unavoidable exception.

### Follow-Ups

Phase workers must not create task beads. If you find follow-up work (a cleanup you
deliberately skipped to keep the move verbatim, a flake, a latent bug), record it as a
`PROPOSED FOLLOW-UP:` note on your own phase bead.

### Reporting

In your final response, state:

- the chosen decomposition and the rejected alternative(s), with the reasoning;
- the resulting file list with line counts;
- the before/after test count;
- the `just check` result;
- any file still over 1500 lines, with the reason.

## Split crates/sase_core/src/bead/cli.rs

**Target:** `crates/sase_core/src/bead/cli.rs`, 4,276 lines. Production code runs to
about line 2,677; the `mod tests` block (~1,600 lines, 47 test attributes) runs from
there to the end. Both halves need splitting.

This is the Rust implementation behind the `sase bead` CLI. `bead/mod.rs` declares it as
`pub mod cli` and re-exports only `execute_bead_cli`, `BeadCliOutcomeWire`,
`BeadCliMutationSummaryWire`, and `BeadCliStatusTransitionWire`. Nearly everything else
is private, so most moves need only `pub(super)`.

Seams visible at audit time:

- Entry and outcome plumbing: `execute_bead_cli`, the three `BeadCli*Wire` types,
  `success`/`success_with_mutation`/`error`/`usage_error`/`defer`, and
  `mutation_summary`.
- Read-only command handlers: `handle_list`, `handle_show` (~185 lines),
  `handle_search`, `handle_ready`, `handle_blocked`, `handle_stats`, plus
  `blocking_issue_ids`, `has_active_blocker`, `active_blocker_ids`, and
  `stats_for_issues`.
- Create: `CreateArgs`, `handle_create`, `parse_create_args`, `ParsedCreateType`,
  `parse_create_type`, and the design-storage path resolution (`storage_design_path`,
  `design_plan_roots`, `design_storage_root`).
- Mutating command handlers: `handle_open`, `handle_update`, `handle_close`,
  `handle_dep`, `handle_ref`, `handle_ref_list`, `handle_rm`.
- Argument parsing: `ListFilters`, `SearchFormat`, `ColorMode`, `SearchArgs`,
  `SearchParseOutcome`, the `parse_*` family, `ParsedCloseArgs`, and
  `close_note_author`.
- Issue resolution: `read_issues`, `find_issue`, the `resolve_cli_*` and
  `*_resolution_outcome` functions.
- Rendering and presentation: search renderers (compact/json/full), snippet and
  character-range helpers, the `ANSI_*` constants, `CliPresentation`, and the
  status/type/id colorizers, `highlight_matches`, `status_*`/`issue_type_value`/`tier_*`
  formatters.
- Plan/design reference display: `PLAN_REFERENCE_*` labels, `display_design_path`,
  `DesignResolution`, `resolve_design_reference`, `legacy_path_below_cwd`,
  `display_plan_path`.

Weigh a split by **command** (one file per handler group, each carrying its own parsing)
against a split by **layer** (handlers vs. parsing vs. rendering). Choose deliberately.
Split the test block along whichever axis you pick. The target becomes `bead/cli/mod.rs`
plus siblings. `bead/` already holds `search.rs`, `read.rs`, and `wire.rs`, so avoid new
names inside `bead/cli/` that would read as duplicates of those modules.

## Split crates/sase_gateway/src/federation_worker.rs

**Target:** `crates/sase_gateway/src/federation_worker.rs`, 3,953 lines, 16 test
attributes.

Structure at audit time:

- Lines 1-365: the platform-neutral public surface. Constants, `FederationWorkerConfig`
  (+ `Default`), `default_federation_socket_path`, the `run_federation_worker*` entry
  points, `FederationWorkerError`, `parse_federation_worker_args`, and the IPC wire
  types (`FederationErrorWire`, request/response envelopes, `FederationIpcRequestWire`,
  host config, catalog query, health, read response, host result).
- Lines 366-3,822: `#[cfg(unix)] mod imp`, the whole real implementation. About 1,950
  lines of production code plus its own nested `#[cfg(test)] mod tests` (lines
  ~2,318-3,820, ~1,500 lines). Inside it:
  - listener and socket lifecycle: `run`, `PreparedListener` (+ `Drop`),
    `prepare_listener`, `secure_dir`, `recover_socket_target`, and the three
    per-platform `peer_is_same_user` variants (Linux / macOS / other);
  - connection handling and framing: `handle_connection`, `read_request`,
    `write_response`, `handle_request`, `handle_operation`, `RequestDeadline`,
    `with_deadline`;
  - `FederationWorkerState` (~470 lines): config replacement, `read_all`,
    `read_catalog_hosts`, `launch_one`, `mutate_one`, `resolve_attention_one`,
    `resolve_launch_target`;
  - remote hosts: `ValidatedHostConfig`/`validate_host_config`, `RemoteHost` (two `impl`
    blocks), `RemoteHostRuntime`, `build_http_client`, `managed_trust_ref_path`,
    `ReadOperation`, `read_one_host`, `fleet_url`, `fleet_http_error`;
  - the response cache: `FederationCacheFile`, `FederationCacheEntry`,
    `FederationCache`, `cache_path`, `compound_cache_key`, `entry_age`,
    `cached_or_error`, `OwnedCachePathLimits`.
- Lines 3,823-3,831: the `#[cfg(not(unix))] mod imp` stub returning
  `UnsupportedPlatform`.
- Lines 3,833-3,926: shared response helpers (`federation_capabilities`,
  `success_response`, `error_response`, `federation_error`, `status_from_error`,
  `unix_now_ms`, `to_json`), then a small outer `mod tests`.

The central design question is how to turn the inline `imp` module into files while
keeping the platform gating exact. For example, a file-based `#[cfg(unix)] mod imp;`
beside an inline `#[cfg(not(unix))] mod imp { ... }` stub, or a descriptively named unix
module aliased into place. Pick one, explain it, and do not use `#[path]`. You cannot
build non-unix targets locally, so reason carefully that the non-unix build still sees
exactly the items it saw before. Also keep the Linux/macOS/other `peer_is_same_user` cfg
ladder intact.

This worker enforces local IPC security: socket directory permissions, peer-UID checks,
frame size limits, and request deadlines. Move that code verbatim, and reject any seam
that would reorder a check. `sase_gateway/src/lib.rs` re-exports parts of this module,
and `federation_worker_main.rs` calls its CLI entry point. Both must keep compiling
unchanged.

## Split crates/sase_core/src/agent_stats/run.rs

**Target:** `crates/sase_core/src/agent_stats/run.rs`, 3,732 lines. Production code is
about lines 1-1,479. The `mod tests` block (~2,250 lines, 21 test attributes) is the
larger half and must itself become several files.

Seams visible at audit time:

- Accumulator and intermediate types at the top of the file (`IndexRunRow`,
  `ProviderKey`, the `*Accumulator` structs, `RunXPrompt`, `AttributedPatch`,
  `RunAttribution`, `PatchMetadata`).
- The query entry point: `query_run_stats` and `query_run_stats_with_liveness` (~245
  lines, one function; leave it whole), plus `resolved_question_answer_times`,
  `validate_request`, and the bucket helpers.
- Lifecycle and runner-overlap logic: `launch_timestamp`, `runner_overlap_candidate`,
  `runner_handoff_carry_candidate`, `record_is_user_hidden`, timestamp parsing,
  `fold_lifecycle`, `run_duration_seconds`.
- Per-dimension folds: `fold_retries`, `fold_commits`, `fold_plans`, `fold_questions`,
  `fold_workspace`, `fold_xprompts`/`finish_xprompts`, and `fold_work`/`finish_work`.
- Attribution: `resolve_run_attribution`, `real_patch_name`,
  `is_project_identity_placeholder`, `load_patch_metadata`.
- Finishing and ranking: `ranked_counts`, `finish_providers`, `effort_rank`,
  `finish_workspaces`, `finish_runtime_groups`, `percentile`, `ratio`, `normalized`,
  `resolved_agent_name`, `provider_key`, `runtime_group_value(s)`.

Decide deliberately whether each accumulator type lives with the fold that fills it or
in one shared types module; that choice drives most of the visibility diff. The target
becomes `agent_stats/run/mod.rs` plus siblings. `agent_stats/` already contains
`activity.rs`, `gate_bundles.rs`, `runner.rs`, and `wire.rs`. Do not create a submodule
whose name collides confusingly with those, `runner` especially. Split the tests into a
`run/tests/` directory along the same seams.

## Split crates/sase_gateway/src/fleet_reads.rs

**Target:** `crates/sase_gateway/src/fleet_reads.rs`, 3,371 lines. Production code is
about lines 1-1,771. The `mod tests` block (~1,600 lines, 23 test attributes) runs to
the end.

Seams visible at audit time:

- `FleetReadService` and its inner state. Its single `impl` block (~590 lines) mixes the
  public request API (`summary`, `catalog`, `batch_lookup`, `detail`, `content`,
  `project_eligibility`, `authoritative_snapshot`, `reconcile`, event
  cursor/subscribe/publish), the `*_for_test` hooks, and the snapshot and
  history-snapshot caching machinery (`stamped_*`, `current_*`, `cached_*`,
  `unexpired_*`, `retain_previous_*`). Rust allows several `impl FleetReadService`
  blocks in different submodules of the same crate. Consider splitting the public API
  from the cache machinery that way instead of forcing one file.
- Snapshot building: `CachedFleetSnapshot`, `BuildSnapshotRequest`,
  `build_snapshot_blocking` (~210 lines).
- Record resolution and projection: `ResolvedRecord`, `resolve_record`, logical and
  exact locator construction, `lifecycle_and_content_capabilities`,
  `project_display_labels`, `stopped_at_unix_for_record`, `workspace_num_for_record`,
  freshness and timestamp parsing, `safe_identifier`, `safe_reason`, `first_non_empty`.
- Content handles and reads: `FleetContentSource`, `content_handles_for_record`,
  `ContentPathCandidate`, `content_path_candidates`, `push_path`,
  `canonical_content_path`, `content_handle_id`, `read_content_range`.
- Project eligibility: `project_eligibility_blocking`.
- Errors: `FleetReadError` and its `From<FleetContractError>`.
- Invalidation and events: `FleetInvalidationHub` (+ inner), `FleetEventSubscription`,
  `new_generation`, `resync_item`.

Content-path canonicalization is a filesystem security boundary. Move it verbatim. The
`*_for_test` methods are `pub` and are used by the gateway's `routes/tests/`, so they
must stay reachable at the same path. `sase_gateway/src/lib.rs` re-exports items from
this module. The target becomes `fleet_reads/mod.rs` plus siblings. The gateway crate
already has `fleet_attention.rs`, `fleet_auth.rs`, `fleet_launch.rs`, and
`fleet_mutations.rs`. Do not move code into those modules in this phase.

## Split crates/sase_core/src/bead/events.rs

**Target:** `crates/sase_core/src/bead/events.rs`, 3,314 lines. Production code is about
lines 1-2,150; the `mod tests` block (~1,160 lines, 26 test attributes) runs to the end.

Seams visible at audit time:

- Event wire types and validation: `BEAD_EVENT_SCHEMA_VERSION`,
  `BeadEventStoreManifestWire`, `BeadEventStreamWire`, `BeadEventRecordWire`,
  `BeadEventOperationWire`, `BeadSnoozeWakeCauseWire`, `BeadEventPayloadWire` and its
  ~165-line `impl`, the link-payload validators, `BeadIssueUpdateEventFieldsWire`.
- Import: `import_issues_to_event_streams`.
- Reduction: `reduce_event_streams` and its `pub` variants,
  `reduce_event_streams_inner`, `apply_link_provenance`, external-ref collapsing, the
  large `pub` event-application function around line 1,514 (~280 lines; leave it whole),
  `apply_update_event_fields`, `existing_issue_mut`, `root_issue_ids`, `PendingEvent`,
  `apply_link_added`, `event_timestamp`.
- Merge and ID relocation: `BeadEventStreamMergeWire`, `BeadIdRelocationWire`,
  `BeadIdRelocationKindWire`, `BranchTag`, `merge_bead_event_streams*`,
  duplicate-creation splitting, subtree extraction/remapping, `next_sibling_issue_id`,
  `validate_append_only_branch`, event union keys, and the `StreamHead` ordering
  (`PartialEq`/`Eq`/`PartialOrd`/`Ord`) with `event_operation_priority`.

`bead/mod.rs` declares this as `pub mod events` and re-exports only part of its `pub`
surface. Several `pub fn`s here are reachable only as `sase_core::bead::events::<name>`
(check the Python binding crate and `crates/sase_core/tests/bead_event_parity.rs`), so
`bead/events/mod.rs` must re-export every currently-`pub` item. Bead reduction and merge
semantics must not change at all: event ordering, tie-breaking in `StreamHead`, and
relocation decisions are persisted-data behavior. The target becomes
`bead/events/mod.rs` plus siblings. Avoid names that clash with the existing
`bead/history.rs`, `bead/jsonl.rs`, and `bead/mutation/`.

## Split crates/sase_core/src/tool_run/store.rs

**Target:** `crates/sase_core/src/tool_run/store.rs`, 3,229 lines. Production code is
about lines 1-2,139; the `mod tests` block (~1,090 lines, 20 test attributes) runs to
the end.

This is the SQLite-backed store for sase tool runs. Seams visible at audit time:

- Schema and connection management: `SCHEMA_SQL`, `validate_schema`, `open_write_store`,
  `open_read_store`, `enforce_schema_version`, `enable_wal_mode`, `restrict_mode`,
  `bounded_busy_timeout`, `with_write_store`, `with_read_store`, meta read/touch,
  `count_rows`.
- Corruption handling: `should_quarantine`, `is_sqlite_corruption_error`,
  `quarantine_corrupt_store`, `corrupt_store_quarantine_path`, `sqlite_sidecar_path`.
- Run lifecycle writes: `begin`, `append_event`, `finish`, `reconcile`,
  `identity_for_begin`, `insert_run`, `insert_lifecycle_event`, `insert_event_row`,
  `ingest_event`, `apply_event_projection`, `can_transition`, `canonical_event`, and
  fingerprint persistence (`persist_optional_fingerprint`,
  `mutated_input_from_fingerprints`, `evidence_from_fingerprints`).
- Queries and loaders: `list_runs`, `show_run`, `summarize`, `store_stats`, `load_run`,
  `load_attempt`, `load_events`, `load_stages`, `load_samples`, `parse_optional_json`,
  `empty_summary`, `parse_cursor`.
- Retention: `retention_preview`, `retention_apply`, `retention` (~220 lines),
  `validate_retention_policy`, `SettledRunLogs`, `AggregateLogUsage`,
  `retained_file_bytes`, `select_aggregate_log_candidates`, `QuarantinedStoreUsage`,
  `select_quarantined_stores`, `quarantine_timestamp_secs`.

Keep the SQL text, transaction boundaries, and statement order inside each transaction
exactly as they are. The store's concurrency and busy-timeout behavior is tested
(`concurrent_begins_do_not_livelock`, `busy_failure_is_bounded`) and must not shift.
`tool_run/` already has `fingerprint.rs`, `canonical.rs`, `catalog.rs`, and `wire.rs`.
Store-side fingerprint persistence is not the same thing as the `fingerprint` module, so
pick names that keep that distinction clear, and do not move store code into those
existing siblings. The target becomes `tool_run/store/mod.rs` plus siblings. The test
block fits under the ceiling as one `tests.rs`, but consider splitting it along the
production seams if that makes it clearer.

## Split crates/sase_core/src/provider_usage/tests.rs

**Target:** `crates/sase_core/src/provider_usage/tests.rs`, 3,176 lines of pure test
code (57 test attributes). `provider_usage/mod.rs` declares it as
`#[cfg(test)] mod tests;` near its end, and the file starts with `use super::*;`.

This phase splits tests, not production code. The file becomes a `provider_usage/tests/`
directory with a `mod.rs` and one file per area. The natural seams line up with the
production siblings (`compatibility.rs`, `grok.rs`, `indicator.rs`, `muse.rs`,
`refresh.rs`, `store.rs`) plus snapshot-level health and validation. Groups visible at
audit time:

- Shared fixtures and builders: `FixtureCase`, `load_fixture`, `normalize`,
  `assert_json_eq`, `project_fixture`, `assert_fixture`, the `*_window`,
  `*_observation`, and `indicator_*` builders, and the time constants.
- Fixture-driven snapshot projection cases (used percent, shared plus model-specific
  limits, unknown scope, mixed ages, reset expiry, deterministic ties).
- Indicator policy, thresholds, diagnostics, window classification, and projection
  order.
- Muse (`muse_payload`, `normalized_muse_observation`, and the muse indicator tests).
- Freshness, applicability, and health (freshness bands, reset passing, empty and
  not-applicable snapshots, unauthenticated).
- Observation validation and rejection (schema, duplicates, vendor drift codes,
  controls/secret redaction, public-JSON unknown fields, missing quantities).
- Claude Fable alias canonicalization.
- Usage store (partial merges, tombstones, legacy cache recovery, generations, bad
  records, old schedule decoding, private file permissions).
- Refresh scheduling (reservations, backoff, failure streaks, collector health, due
  reasons, future markers, admission, mark-due).
- Attention (collection-problem attention and rank order).

Watch the `include_str!("fixtures/...")` calls in `load_fixture`. `include_str!` paths
resolve relative to the file containing the macro, so moving it into `tests/<file>.rs`
requires `../fixtures/...`. Test builders currently rely on `use super::*;` reaching
into `provider_usage`'s private items. Keep that working through `mod.rs` (for example
`use super::super::*` or explicit imports), and make sure every test still sees the same
items. `provider_usage/store.rs` (1,906 lines) and `provider_usage/mod.rs` (1,464 lines)
are not targets of this epic. Do not split them here.

## Split crates/sase_core/src/editor/directive.rs

**Target:** `crates/sase_core/src/editor/directive.rs`, 2,952 lines. Production code is
about lines 1-1,833; the `mod tests` block (~1,120 lines, 29 test attributes) runs to
the end.

Seams visible at audit time:

- Static directive metadata tables (~800 lines): the `*_SUGGESTIONS` constants, the
  `DirectiveSyntaxForm` slices (`BARE_PLUS`, `COLON`, `PAREN`, and so on), the
  `*_KEYWORDS` keyword specs (id, clan, proc, if, queue, queue budget, hold, wait),
  `IF_DIRECTIVE_OFF/ON`, `QUEUE_DIRECTIVE_OFF/ON`, the public `DIRECTIVES` table (~200
  lines), `HIDDEN_COMPLETION_DIRECTIVES`, `BEAD_COMPLETION_LIMIT`, and
  `BEAD_STATUS_RANK`.
- Contract lookup API: `directive_is_hidden_from_name_completion*`,
  `canonical_directive_name`, `directive_metadata*`, `if_directive_metadata`,
  `queue_directive_metadata`, `directive_contract*`, `directive_allows_keywords`, and
  the `pub` functions right after it.
- Completion candidates: `build_directive_completion_candidates*`,
  `directive_argument_candidates`, keyword and static-value candidates,
  `rank_and_filter_bead_entries`, `build_bead_completion_candidates` and its bead
  detail/documentation helpers, `argument_candidate`, `keyword_candidate`,
  `selected_keyword_set`, `keyword_is_available`.
- Context detection and clause parsing: `is_directive_like_token`,
  `detect_directive_context_at_position`, `directive_name_token`,
  `DirectiveArgCompletionTarget`, colon/parenthesized/comma-clause contexts,
  `classify_active_clause`, clause splitting, `unterminated_body_end`,
  `find_top_level_equals`, `find_matching_paren_quoted`, `QuoteState`.

Several functions come in flag-aware `_with_flags` pairs. Keep each pair together. Keep
the static tables in declaration order: the directive order in `DIRECTIVES` is
observable in completion output and in the contract-matrix test. `editor/completion/`
already has a `directive_candidates.rs` from the previous epic. Pick names inside the
new `editor/directive/` tree that do not read as duplicates of it, and do not move code
across into `editor/completion/`. Note that `editor/frontmatter.rs` (2,699 lines) is
next in line after this epic's targets; it is not a target here.

## Split crates/sase_core/src/runner_capacity.rs

**Target:** `crates/sase_core/src/runner_capacity.rs`, 2,938 lines. Production code is
about lines 1-1,631; the `mod tests` block (~1,300 lines, 43 test attributes) runs to
the end.

This is the runner capacity admission policy (who may run, who waits, and why). Seams
visible at audit time:

- Wire types and schema version: `RUNNER_CAPACITY_POLICY_SCHEMA_VERSION`,
  `DEFAULT_WAIT_PRIORITY`, the `RunnerCapacity*Wire` request, record, diagnostic, claim,
  blocker, waiter, candidate-decision, and snapshot types,
  `RunnerCapacityRecordWire::normalize_queue_capacity_aliases`, `default_true`,
  `default_workflow_dir`.
- Snapshot entry: `runner_capacity_snapshot`, `records_excluding_candidate`,
  `request_for_waiters`, `candidate_record_with_effective_weight`.
- Claims and lineage: `build_claims`, `ClaimLineage`, `ClaimAccumulator`,
  `active_claim_keys`, `claim_is_reusable`, `claim_lineage*`, explicit/parallel/serial
  lineage, `inherited_lineage_weight`, `lineage_record_weight*`.
- Waiters: `WaiterEvaluation`, `WaiterDraft`, `build_waiters`, `waiter_admission_limit`,
  `waiter_blockers` (~150 lines), comparison and display bucket, capacity shortfall,
  priority normalization, deference window.
- Candidate decision: `build_candidate_decision` (~145 lines).
- Hold barriers: `hold_barrier_blockers`, `hold_candidate`, `hold_barrier_blocker`,
  `epoch_seconds_from_rfc3339`, `format_hold_expires_in`.
- Record predicates and weights: `is_user_agent_record`, `is_occupying_record`,
  `is_waiting_record`, `is_pending_gate`, `is_real_monitor_member`, `effective_weight`,
  `record_weight_is_valid`, `explicit_queue_capacity`.
- Floating-point capacity math: `compensated_sum`, `capacity_fits`, `ulp_at`, `next_up`,
  `weights_equal`, `compare_f64`, `occupied_capacity_exceeds_threshold`. This code is
  numerically sensitive (decimal-boundary tests depend on it). Move it byte-for-byte.

The `*Wire` types reach the Python binding and cross-process records, so their serde
shape, including the legacy-alias deserialization covered by
`legacy_wait_runners_aliases_deserialize_to_queue_capacity`, must be preserved. The
target becomes `runner_capacity/mod.rs` plus siblings. The separate top-level
`runner_limit_override.rs` module is not part of this target. Leave it where it is. The
test block (~1,300 lines) is under the ceiling but close to it. Split it along the
production seams so the next test addition does not breach it.

## Split crates/sase_core/tests/notification_store_parity.rs

**Target:** `crates/sase_core/tests/notification_store_parity.rs`, 2,825 lines. It is a
Cargo integration test (one test binary) with 63 `#[test]` functions.

This target has a different shape from the others: it is an integration test crate, not
a library module. Cargo auto-discovers both `tests/<name>.rs` and `tests/<name>/main.rs`
as a test target named `<name>` (sase_core's `Cargo.toml` has no `[[test]]` entries). So
the split is `tests/notification_store_parity/main.rs` plus sibling modules declared
from `main.rs`. That keeps a single test binary with the same name, so
`cargo test --test notification_store_parity` keeps working. Do **not** split it into
several top-level `tests/*.rs` files. Each extra integration-test binary re-links
`sase_core` and slows every `just check`. Leave the shared `tests/fixtures/` directory
in place.

Test-area seams visible at audit time:

- Shared builders: `store_path`, `archive_path`, `notification`,
  `timestamp_days_from_now`, `write_jsonl`, `CONTRACT_FIXTURE`, plus the settlement
  builders (`settlement_notification`, `settlement_fixture_rows`) and plus-one helpers
  (`plus_one_by_id`, `plus_one_by_key`, `assert_state_flags_untouched`) if more than one
  area uses them.
- Loading, legacy defaults, and the phase-1 contract fixture.
- Append/rewrite round trips, temp-sibling reaping, tags/icon round trips, counts
  variants, byte-identical JSONL, and unseen-row preservation.
- State updates: priority counts, state updates, mark-tab-read, batch dismiss,
  undismiss.
- Mute, snooze, and expiry (bulk mute/unmute/snooze, offset normalization, atomic
  validation, legacy-state recovery, dismissal cancelling snooze, activity cursor).
- Compaction to the archive and retention thresholds.
- Dismiss-matching-agents and agent-completion dismissal (settlement rows, question
  root/child identity, custom gates, user-agent view error reports).
- Plus-one and upsert/supersede semantics.
- Concurrency (append plus rewrite, rewrite counts, upsert, and expiry convergence).
  Decide whether these belong beside the behavior they stress or in one file of their
  own, and justify it.

`include_str!` paths resolve relative to the file containing the macro. Once
`CONTRACT_FIXTURE` moves into `tests/notification_store_parity/`, its path becomes
`../fixtures/notifications/store_contract.jsonl`. Verify the before/after test count
with `cargo test -p sase_core --test notification_store_parity -- --list` in addition to
`just check`.

### Epic Close-Out

This is the last phase, so also re-run the repo-wide audit from the sase-core repo root:

```bash
find crates -name '*.rs' -not -path '*/target/*' -exec wc -l {} + \
  | sort -rn | awk '$1 > 1500' | head -40
```

Report the remaining over-1500-line files. All ten of this epic's targets, and every
file created from them, should be absent. Anything still listed is either a file this
epic never targeted (expected; the repo has more below the top ten, such as
`editor/frontmatter.rs`, `agent_ownership/planner.rs`, and `agent_scan/scanner.rs`), or
a justified exception from an earlier phase. Say which. If
`crates/sase_gateway/src/sudo_runner.rs` is still present as a single over-1500-line
file, the previous epic's landing did not bring in its split. Call that out explicitly,
but do not split it yourself. Do not start splitting untargeted files.
