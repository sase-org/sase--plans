---
tier: epic
title: Give the artifact store a lifecycle - report, dry-run pruning, and opt-in retention
goal: 'The artifact-file store can be measured, drained, and bounded: `sase artifact
  stats` reports its economics, `sase artifact prune` and `sase artifact reclaim`
  are dry-run by default and route every removal through a restorable trash, an opt-in
  `artifacts.retention` policy keeps the store bounded going forward, and nothing
  an agent declared or a bead, plan, or ChangeSpec references is ever removed.

  '
phases:
- id: core-lifecycle
  title: 'Rust core: store economics, retention planner, and trash primitives'
  depends_on: []
  size: medium
  description: 'core-lifecycle: add a pure store-economics aggregator, a pure retention
    planner, and clock-free trash store/list/restore/purge primitives to `sase_core`,
    expose all four through `sase_core_rs`, and release the crate.

    '
- id: py-report
  title: Store economics report and the protected-reference scan
  depends_on:
  - core-lifecycle
  size: medium
  description: 'py-report: raise the `sase-core-rs` pin, add the Python bridges for
    economics and retention planning, add the bead/plan/ChangeSpec reference scan
    that produces protected artifact ids, and ship `sase artifact stats` as a read-only
    report with a JSON mode.

    '
- id: py-prune
  title: Dry-run-first pruning and the trash lifecycle
  depends_on:
  - py-report
  size: medium
  description: 'py-prune: add `sase artifact prune` (dry run unless `--apply`) and
    the `sase artifact trash {list,purge,restore}` group, with index-row removal and
    restoration performed under the existing index lock and every removal routed through
    the trash.

    '
- id: py-reclaim
  title: Retroactive version-control reclaim of the pooled bytes
  depends_on:
  - py-prune
  size: medium
  description: 'py-reclaim: add `sase artifact reclaim`, which converts stored automatic
    rows whose exact content is reproducible from a durable ref into byte-free VCS-backed
    rows and trashes the redundant copies, reusing the capture policy''s git probe
    and verifying by digest first.

    '
- id: retention-config
  title: Opt-in retention configuration and enforcement
  depends_on:
  - py-prune
  size: small
  description: 'retention-config: add the `artifacts.retention` configuration block
    defaulting to disabled, its schema entries and accessors, and the bounded, fail-safe
    enforcement pass that runs after automatic capture at finalization and purges
    expired trash.

    '
- id: docs-and-skill
  title: Documentation, skill, and configuration reference
  depends_on:
  - py-reclaim
  - retention-config
  size: small
  description: 'docs-and-skill: document the store lifecycle, the new subcommands,
    the trash contract, and the retention block across the CLI, configuration, and
    artifact docs, and refresh the `sase_artifact_file` skill source and its deployed
    copy.

    '
create_time: 2026-07-30 10:39:40
status: wip
bead_id: sase-ba
---

- **BEAD:** [sase-ba](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ba/README.md)

# Plan: Give the artifact store a lifecycle - report, dry-run pruning, and opt-in retention

This implements recommendation 3 of `research:202607/artifact_capture_and_retention/artifact_capture_and_retention.md`
("Give the store a lifecycle: report -> dry-run -> opt-in retention"), including the retroactive migration that
recommendation assigns to itself.

It is the sequel to the epic that is landing now (`sase-b7`, plan `plans:202607/vcs_backed_artifact_capture.md`), which
closed the tap: automatic capture already writes byte-free rows carrying `vcs_repo`/`vcs_sha`/`vcs_relpath` for content
version control can reproduce. This epic drains the pool that accumulated before that landed, and bounds what pools
next.

## 1. Context and measurements

All figures below were re-derived by direct traversal of the live index and store on 2026-07-30, after `sase-b7`'s
capture wiring landed but before any run had exercised it.

| Measure                                          |                                Value |
| :----------------------------------------------- | -----------------------------------: |
| Indexed rows                                     |                                4,295 |
| Recorded bytes                                   |                             665.1 MB |
| Store on disk                                    |                             669.2 MB |
| Explicit (agent-declared)                        |               222 rows / **5.56 MB** |
| Automatic (heuristic-captured)                   |            4,073 rows / **659.6 MB** |
| VCS-backed (byte-free) rows                      |                                **0** |
| Kinds                                            | image 4,053 · markdown 217 · file 25 |
| Window                                           |   25 days (2026-07-06 -> 2026-07-30) |
| Duplicate-digest groups / redundant rows / bytes |               94 / 107 / **35.8 MB** |
| Distinct automatic labels                        |                                  461 |
| Automatic rows with a dead `source_path`         |                          1,220 (30%) |
| Rows with a missing stored file                  |                                **0** |

Predicate projections over the automatic cohort, which is what a retention policy would act on:

| Predicate                      | Rows selected |    Bytes |
| :----------------------------- | ------------: | -------: |
| Keep newest 1 capture / label  |         3,612 | 541.6 MB |
| Keep newest 3 captures / label |         2,853 | 379.5 MB |
| Keep newest 5 captures / label |         2,188 | 277.9 MB |
| Older than 7 days              |         2,906 | 498.6 MB |
| Older than 14 days             |           529 | 128.8 MB |
| Older than 30 days             |             0 |      0 B |

**The decisive new measurement.** 4,073 automatic rows all carry a `workspace_dir`, and 4,066 of them have a
`source_path` inside a numbered workspace, so `relpath = source_path` relative to `workspace_dir` is derivable for 99.8%
of the cohort. Sampling 60 distinct relpaths (560 rows) and walking `origin/master` history in the primary repo, hashing
each `<commit>:<relpath>` blob:

> **560 of 560 sampled rows (100%) have their exact recorded `sha256` present in `origin/master` history at their
> recorded relative path.** Required scan depth: p50 = 6 commits, p75 = 11, p90 = 16, p95 = 19, max = 25.

That is the whole argument for sequencing reclaim ahead of pruning. Roughly 650 MB of this store is not a retention
problem at all - it is a redundancy problem with a lossless fix. Converting those rows to byte-free VCS-backed records
keeps every row, every label, and every provenance field, and deletes only a duplicate of something git already stores
losslessly. Pruning - which does destroy information - is left with the genuine remainder.

The measured p95 of 19 also says the shipped `artifacts.capture.max_history_scan` default of 20 is correctly sized for
_capture_ (where the commit is fresh) and **too small for reclaim**, where the content is up to 25 days and 25 commits
old. Reclaim therefore uses its own, larger bound.

Two more facts that shape the design:

- **Reference protection is cheap and real.** Scanning the ProjectSpec directory and the plans, beads, and research
  sidecars for `(file:)?(default|explicit):<24 hex>` tokens finds ~15 distinct referenced artifact ids today across ~70
  files. A protection scan is a bounded text sweep, not an index.
- **The claimed prior art is thinner than the research assumed.** The research names
  `delete_agent_artifacts`/`delete_agent_artifact_index_row` as "a mature, Rust-owned GC" to extend. Read directly,
  `delete_agent_artifact_markers()` deletes a run directory's loader markers (`workflow_state.json`, `done.json`, and
  each `prompt_step_*.json`), and the index-row helper deletes one SQLite row. There is no reusable deletion engine
  behind either. What _is_ reusable, and what this plan reuses, is the **shape** of
  `crates/sase_core/src/agent_cleanup/`: a pure planner that returns a plan wire with counts, selected items, skipped
  items with reasons, and summary lines, while the host performs the writes it already owns.

## 2. Invariants

These are non-negotiable and every phase must preserve them.

- **R1 - declaration is permanent.** No surface in this epic ever removes, converts, or rewrites a row with
  `explicit=True`. Declared artifacts are the 5.56 MB anyone actually asked for.
- **R2 - referenced artifacts are protected.** An artifact id that appears in any ProjectSpec, bead, bead page, or plan
  file is protected from pruning. If a protection source cannot be read, `prune --apply` refuses and says which source
  it could not read; the dry run still renders and reports the gap. Under-protecting silently is the one failure mode
  that would make the feature untrustworthy.
- **R3 - no hard deletes.** Every removal moves the stored bytes _and_ the complete index row into the trash. Only
  `sase artifact trash purge` hard-deletes, and only entries past the configured grace period unless explicitly
  overridden.
- **R4 - reclaim is lossless.** A stored row becomes byte-free only after its exact bytes have been reproduced from
  `<commit>:<relpath>` on a durable remote-tracking ref and the reproduction's SHA-256 verified equal to the recorded
  digest. This is `sase-b7` invariant A applied retroactively; the resulting row is readable through the resolver and
  re-verifiable through `sase artifact doctor --verify`. Reclaim also honors R1 and R2: it never touches an explicit or
  a referenced row, because converting a row changes its id (section 6) and that would break an existing reference.
- **R5 - dry run is the default.** `prune` and `reclaim` print a plan and change nothing unless `--apply` is passed.
- **R6 - fail safe.** Any git failure, timeout, absent checkout, unknown repo, or unreadable protection source results
  in _no action_ for the affected row, never in a removal.
- **R7 - the newest generation always survives.** Whatever the predicates say, the newest capture of every label is
  never selected. A label's history can shrink to one; it cannot vanish.

## 3. Rust core: store economics, retention planner, and trash primitives

Open the core repo the sanctioned way and use the path it prints:
`sase repo open sase-core -r "Add artifact-store economics, retention planning, and trash primitives"`. release-plz owns
versions; do not hand-edit `Cargo.toml`. Everything here is **additive** - no existing wire constant changes - so a
normal `feat:` commit yields a minor release. Do not use `feat!:`.

All three modules live under the existing `crates/sase_core/src/artifact_file/` directory beside `vcs.rs`, and are
declared from `crates/sase_core/src/artifact_file.rs`, which already reads the index tolerantly through
`read_artifact_file_index()` and knows `artifact_file_is_vcs_backed()`.

Add one shared constant in `artifact_file.rs`: `ARTIFACT_FILE_LIFECYCLE_WIRE_SCHEMA_VERSION: u64 = 1`. Every request and
result wire below carries it, and every request is rejected on mismatch, following `cleanup_request_from_json_value()`.

**No clocks in core.** Neither the planner nor the trash reads the system clock: the host passes `now` (an RFC3339
string) into planning, `trashed_at` into trashing, and an explicit cutoff into purging. This keeps every result
deterministic and testable, and mirrors how `query_artifact_files_at()` is already factored behind
`query_artifact_files()`.

### `crates/sase_core/src/artifact_file/economics.rs`

`artifact_file_store_economics(index_path, options) -> ArtifactFileEconomicsWire`, a pure aggregation over the rows
`read_artifact_file_index()` returns. Options carry `project: Option<String>`, `top_n: usize` (default 10), and
`generation_projections: Vec<u64>` (default `[1, 3, 5]`). The result reports:

- row counts split explicit / automatic / VCS-backed, and recorded bytes split the same way;
- `by_kind`, `by_project`, and `by_agent` group rows (`key`, `rows`, `bytes`), each sorted by bytes descending, with
  `by_agent` truncated to `top_n` and carrying a `truncated_groups`/`truncated_bytes` remainder so the report never
  silently hides a tail;
- the observed window (`first_created_at`, `last_created_at`, `window_days`) and observed `bytes_per_day` /
  `rows_per_day` over that window. Report what was observed; do not extrapolate to a year - the research's own open
  question 3 notes the index may have been rebuilt on 2026-07-17, so the window is a floor, not a history;
- redundancy: `duplicate_digest_groups`, `redundant_digest_rows`, `redundant_digest_bytes` (rows beyond the first in
  each equal-`sha256` group);
- `distinct_labels` and one `label_generation_projections` entry per requested keep-count, each with the rows and bytes
  that keeping only the newest N captures per `(project, label)` would free;
- `source_inside_workspace_rows`/`_bytes`: automatic byte-backed rows whose `source_path` starts with their
  `workspace_dir`. Name this in the wire and in the rendered report as an **upper bound on reclaimable rows**, not as a
  reproducibility result - reproducibility is only ever established by digest verification in `reclaim`.

Rows with a null `size_bytes` contribute to counts and not to bytes; report `rows_missing_size` so a zero-byte total is
never mistaken for an empty store.

### `crates/sase_core/src/artifact_file/retention.rs`

`plan_artifact_file_retention(index_path, policy) -> Result<ArtifactFileRetentionPlanWire, ArtifactFileQueryError>`.

`ArtifactFileRetentionPolicyWire` carries `schema_version`, `now: String`, `keep_per_label: u64` (0 disables),
`before: Option<String>` (parsed with the existing `parse_since_date_bound()` from `plan::search`, so `2026-06-01`,
`2026-06`, `14d`, `2w`, and `1m` all work exactly as `sase artifact list --since` accepts them),
`kinds: Option<Vec<String>>`, `project: Option<String>`, `min_size_bytes: Option<u64>`, `protected_ids: Vec<String>`,
and `limit: Option<usize>`.

Selection, in order:

1. Start from every row that is **not** `explicit` (R1) and not in `protected_ids` (R2). Every excluded row is recorded
   in `protected` with reason `explicit` or `referenced` so the dry run can show its work.
2. Group the remainder by `(project, label)`, order each group by `created_at` descending with `id` as a stable
   tie-break, and mark generations beyond `keep_per_label` as generation-selected. `keep_per_label` is clamped to a
   minimum of 1 when any predicate is active (R7).
3. Apply `before`, `kinds`, `project`, and `min_size_bytes` as additional required conditions on the selected set.
   `min_size_bytes` never selects a byte-free row: a row with no stored bytes cannot satisfy a size predicate.
4. Truncate to `limit` if set, and record `truncated: u64` on the plan. Never truncate silently.

The plan wire carries `selected: Vec<ArtifactFileRetentionItemWire>` (`id`, `label`, `kind`, `project`, `agent_name`,
`path: Option<String>`, `size_bytes`, `created_at`, `reason`), `protected: Vec<ArtifactFileProtectedItemWire>` (`id`,
`reason`), `counts` (`candidates`, `selected`, `protected`, `byte_backed_selected`, `byte_free_selected`),
`reclaimable_bytes`, `truncated`, and `summary_lines: Vec<String>`. Selecting a byte-free row is legitimate and reclaims
no bytes - it removes index noise, which is the research's _other_ argument for retention - so the counts keep the two
apart and `reclaimable_bytes` only ever sums byte-backed rows.

### `crates/sase_core/src/artifact_file/trash.rs`

The trash is a directory under the store root (`~/.sase/artifacts/trash`), owned entirely by core so that a second
frontend inherits identical semantics. One entry is one directory
`<trash_root>/<trashed_at compact>-<sanitized artifact id>/` containing `entry.json` (schema version, `entry_id`,
`artifact_id`, `trashed_at`, `reason`, `size_bytes`, `stored_filename`, and the **complete original index row**
verbatim) and, when the row had bytes, the moved file under its original file name.

- `trash_artifact_file(request) -> Result<ArtifactFileTrashEntryWire, String>` - request carries `trash_root`, `record`
  (the index row as JSON), `stored_path: Option<String>`, `reason`, and `trashed_at`. Writes `entry.json` atomically
  (temp file plus rename, as `write_cache_entry()` already does in `vcs.rs`), then moves the stored file in, falling
  back to copy-then-unlink across filesystems. A byte-free row produces an entry with no payload file. On an entry-id
  collision, suffix with a counter rather than overwriting.
- `list_artifact_file_trash(trash_root) -> Result<Vec<ArtifactFileTrashEntryWire>, String>` - newest first; skips and
  counts unreadable entries rather than failing the listing.
- `restore_artifact_file_trash(request) -> Result<ArtifactFileTrashRestoreWire, String>` - moves the payload back to the
  recorded stored path (creating parents), returns the record so the host can reinsert the index row, and removes the
  entry directory only after the payload is back in place. Refuses when the destination exists with different content.
- `purge_artifact_file_trash(request) -> Result<ArtifactFileTrashPurgeWire, String>` - request carries `trash_root`,
  `before: String` (RFC3339 cutoff), and `all: bool`; removes entries whose `trashed_at` is at or before the cutoff (or
  every entry when `all`), returning purged entry ids and freed bytes.

Every path is resolved and checked to remain inside `trash_root` before any write or delete, the way `cache_path()` in
`vcs.rs` guards `cache_root`. Never follow a symlink out of the trash.

### `crates/sase_core_py/src/lib.rs`

Expose and register `artifact_file_store_economics`, `artifact_file_retention_plan`, `artifact_file_trash_store`,
`artifact_file_trash_list`, `artifact_file_trash_restore`, and `artifact_file_trash_purge`, and document each in the
header comment list beside `artifact_file_materialize_vcs`. Also expose `artifact_file_lifecycle_wire_schema_version()`
so Python can assert the constant matches, exactly as `artifact_file_query_wire_schema_version()` is asserted today.

**Tests.** Economics: mixed explicit/automatic/byte-free rows, null sizes, single-day windows (no divide-by-zero),
`top_n` truncation with a non-zero remainder, and each generation projection. Planner: every predicate alone and
combined, `keep_per_label = 0`, the R7 clamp, protected ids, byte-free selection accounted separately, `limit`
truncation, and determinism across two runs over the same index. Trash: store/list/restore/purge round trip for a
byte-backed row and a byte-free row, id sanitization for the `default:<hex>` colon, entry-id collision, a payload whose
destination is occupied, an unreadable entry skipped during listing, purge cutoff boundaries, and a refusal for a
`trash_root`-escaping path. Use `tempfile` fixtures as `vcs.rs` does.

Run the core repo's own check target before committing, then land and let release-plz cut the release.

## 4. Store economics report and the protected-reference scan

Start by raising the `sase-core-rs` pin in `pyproject.toml` to the release the previous phase produced, then
`just install`; `sase core health` must pass before anything else in this phase.

**`src/sase/core/artifact_file_economics.py`** -
`artifact_file_store_economics(*, index_path=None, project=None, top_n=10, generation_projections=(1, 3, 5)) -> ArtifactFileEconomics`.
Follow `artifact_file_query_facade.py` exactly: assert the lifecycle wire schema through `require_rust_binding`,
validate the returned mapping field by field, and raise a `RuntimeError` naming the offending field on any mismatch.
Return frozen dataclasses, not raw dicts.

**`src/sase/core/artifact_file_retention.py`** - a `RetentionPolicy` frozen dataclass mirroring the policy wire, plus
`plan_artifact_file_retention(policy, *, index_path=None) -> RetentionPlan`. Same bridge discipline. This module is the
only place that builds the policy wire; `prune` and the retention enforcement pass both go through it.

**`src/sase/core/artifact_file_protection.py`** - `collect_protected_artifact_ids() -> ProtectedArtifactIds`, returning
`ids: frozenset[str]`, `sources_scanned: tuple[str, ...]`, and `sources_unavailable: tuple[str, ...]`. It scans, with
one compiled `(?:file:)?(default|explicit):[0-9a-f]{24}` pattern:

- every `*.sase` file under `sase_projects_dir()`, including `<key>-archive.sase`;
- for each enabled project, the plans and beads sidecar checkouts resolved through `collect_repo_inventory()`, using
  each record's live clone. These two plus the ProjectSpecs are the **required** sources - exactly the "bead, plan, or
  ChangeSpec" set the research names - so a missing clone is recorded in `sources_unavailable` rather than treated as
  empty (R2);
- the research sidecar when a clone exists, as an **opportunistic** source. Its absence is normal (it is cloned in one
  workspace out of twenty-one) and must not be reported as unavailable, or `--apply` would be blocked almost always;
- `.md`, `.sase`, `.json`, `.yml`, and `.txt` files only, skipping any `.git` directory, so the sweep stays bounded.

Normalize every hit to the bare `<prefix>:<hex>` id so `file:default:abc` and `default:abc` protect the same row.

**`src/sase/artifact_cli/stats.py`** - `handle_stats(args) -> int`, rendered with `rich` in the same panel-and-table
idiom as `src/sase/artifact_cli/doctor.py` and `listing.py`, with `--json` emitting one stable envelope. Sections:
totals, window and observed growth, by kind, by project, top agents (with the truncated remainder shown), redundancy,
label-generation projections, the reclaimable upper bound, protections (explicit count, referenced count, and any
unavailable protection source rendered as a warning), trash occupancy, and - last - **what the default policy would
select**, obtained by running the retention planner in dry run.

The configuration block that policy comes from lands in the `retention-config` phase. Until then, define the three
values this epic needs as module constants in `src/sase/core/artifact_file_retention.py` (`DEFAULT_KEEP_PER_LABEL = 3`,
`DEFAULT_MAX_AGE_DAYS = 90`, `DEFAULT_TRASH_GRACE_DAYS = 14`) and read them from there, so the later phase replaces
three call sites with accessors rather than reworking the surfaces. Label the projection as a default, not as
configuration, until it is one.

Register the subcommand in `src/sase/main/parser_artifact.py` and `src/sase/main/artifact_handler.py`, keeping
subcommands and options alphabetical and giving every long option a short alias, per the `cli_rules` long-term memory -
read it before writing the parser. Options: `-j/--json`, `-p/--project`, `-t/--top N`. Update the group's usage string
to `{create,doctor,list,open,path,show,stats}`.

**Divergence from the research, stated deliberately.** The research says "extend `doctor` with economics". This phase
ships a separate `sase artifact stats` instead. `doctor` exits non-zero to mean _the index is unhealthy_; economics is
not a health signal, and folding it in would either corrupt that contract or bury it under a report three times the
size. A separate read-only subcommand also gives the mobile gateway and the Files pane one JSON envelope to consume.
`doctor` is left exactly as `sase-b7` left it.

**Tests.** `tests/artifact_file_facade/test_economics.py` (bridge validation and every reported field, over a temp
index), `tests/artifact_file_facade/test_retention.py` (policy construction and plan projection), and
`tests/test_artifact_protection_scan.py` (ids found in a ProjectSpec, a plan file, and a bead page; `file:`-prefixed and
bare forms unified; a missing sidecar reported as unavailable rather than empty). CLI tests assert the JSON envelope's
shape and that `stats` never writes.

## 5. Dry-run-first pruning and the trash lifecycle

**`src/sase/core/artifact_file_trash.py`** - the Python side of the trash, and the only module that removes or reinserts
index rows:

- `trash_artifact_files(rows, *, reason, now=None, index_path=None) -> TrashResult` - for each row, call the core trash
  binding to move the payload and write the entry, then remove the row from the index. Take
  `artifact_file_index_lock(idx, exclusive=True)` **once** for the whole batch and rewrite through
  `read_artifact_file_index_document_unlocked()` / `write_artifact_file_index_unlocked()`, preserving unparsed lines
  exactly as `_upsert_index_row()` does. Core moves the bytes first; the index row is removed only after its entry
  exists on disk, so a crash mid-batch leaves a restorable entry rather than an orphaned row.
- `list_trashed_artifact_files()`, `restore_trashed_artifact_file(entry_id)` (restore payload, then re-upsert the record
  into the index under the same lock), and `purge_trashed_artifact_files(*, before=None, purge_all=False)`.

**`src/sase/artifact_cli/prune.py`** - `handle_prune(args) -> int`. Build the policy from the CLI options, collect
protected ids, plan, and render the same table in both modes; `--apply` then executes through `trash_artifact_files()`.
Refuse to apply, with exit code 1 and a message naming the source, when `sources_unavailable` is non-empty (R2). Print a
final line with rows trashed, bytes reclaimed, and where the trash lives. `--json` emits the plan envelope plus, on
apply, the execution result.

Options (alphabetical, each with a short alias): `-a/--apply`, `-b/--before DATE` (same grammar as `list --since`),
`-g/--keep-generations N`, `-j/--json`, `-k/--kind KIND` (repeatable), `-l/--limit N`, `-m/--min-size BYTES`,
`-p/--project`.

**`src/sase/artifact_cli/trash.py`** - the `sase artifact trash` group with `list` (default when bare, wired centrally
by `_default_list_subcommands()` in `src/sase/main/parser.py` - do not re-implement it), `purge`, and `restore`:

- `trash list` - `-j/--json`, `-l/--limit N`; columns for entry id, artifact ref, label, size, trashed-at, and whether
  the entry is past its grace period.
- `trash purge` - `-a/--all` (ignore the grace period), `-j/--json`. Without `--all`, purges only entries older than the
  grace period, read from `DEFAULT_TRASH_GRACE_DAYS` in `src/sase/core/artifact_file_retention.py` until the
  `retention-config` phase turns it into a configuration accessor.
- `trash restore REFERENCE` - accepts an entry id or an artifact ref, restores payload and index row, and prints both
  paths.

**Safety while developing this phase.** Never run `prune --apply`, `trash purge`, or any other mutating command against
the real store. Every manual exercise runs with `SASE_HOME` pointed at a scratch directory - `sase_home()` in
`src/sase/core/paths.py` honors it - seeded by copying `~/.sase/artifacts/index.jsonl` and a handful of stored files.

**Tests.** `tests/artifact_file_facade/test_trash.py`: batch trashing takes one lock and preserves unparsed index lines;
a byte-free row round-trips with no payload; restore reinserts the row with identical field values and a stable id;
purge respects the cutoff and `--all`; a core-side failure for one row leaves the remaining rows and the index
consistent. CLI tests: dry run writes nothing at all (assert index mtime and store contents unchanged), `--apply`
trashes exactly the planned ids, an unavailable protection source blocks apply but not the dry run, explicit rows are
never selected, and the newest generation of every label survives.

## 6. Retroactive version-control reclaim of the pooled bytes

This is the phase that returns ~650 MB and the reason the ordering in section 1 matters.

**`src/sase/core/artifact_file_reclaim.py`** - `plan_artifact_file_reclaim(...) -> ReclaimPlan` and
`execute_artifact_file_reclaim(...) -> ReclaimResult`.

Candidate selection: automatic (`explicit=False`, R1), not in the protected-id set (R2, R4), byte-backed (`path` set),
with a `sha256`, a `workspace_dir`, and a `source_path` that starts with that `workspace_dir`. Everything else is
reported in an `unresolved` bucket with a reason - never silently dropped.

Repo and relpath resolution, in order:

1. `relpath = source_path` relative to `workspace_dir`.
2. If `relpath` begins with a nested-checkout prefix (`sase/repos/...`), map that prefix to the matching
   `collect_repo_inventory()` record by comparing the prefix against each record's workspace-local clone path, and
   re-relativize inside that repo. This covers the ~29 sidecar, linked, and external rows measured on 2026-07-30.
3. Otherwise the row belongs to the project's primary repo.
4. Resolve a live checkout for that repo identity from the inventory (any clone whose `exists` is true, current
   workspace first). No live checkout means `unresolved`, never a removal (R6).

Verification reuses `sase-b7`'s probe rather than inventing a second git path: `GitVcsProbe` in
`src/sase/core/artifact_capture_policy.py` already offers `durable_candidate_commits()` (bounded `rev-list` over
remote-tracking refs, satisfying durability) and `blob_content_digests()` (one batched `git cat-file --batch` per repo).
For each candidate, walk its durable commits and take the first whose blob digest equals the row's `sha256`. Because
this reads digests rather than materializing, a reclaim **dry run touches nothing at all** - not even the `vcs-cache`.

The bound: default `max_history_scan` of **100** for reclaim, overridable per invocation. The sampled p95 is 19 and the
sampled max is 25; 100 leaves generous headroom for a one-shot pass while keeping every walk finite. Do not reuse
`artifacts.capture.max_history_scan` (20) - it is tuned for a commit made minutes ago.

Execution, per verified row, in this order: write the byte-free row via `store_default_artifact_file()`'s reference
mode - `path=None`, the three `vcs_*` fields, and the already-recorded `sha256`/`size_bytes`/`mime_type` - then trash
the old stored file through `trash_artifact_files()` with reason `reclaimed`. Note that the reference row's id is
derived from the VCS identity (`artifact_file_id()` substitutes `vcs:<repo>:<relpath>:<sha256>` for the path component),
so it differs from the byte-backed row's id; the old row must be removed and the new one written, and the reclaim report
must print both ids so someone holding the old id can follow it. Preserving the old id is not an option: it would
collide with the id a re-capture of the same content computes, and idempotent capture depends on that. Rows that fail
verification are left completely untouched (R4, R6).

**Reclaimed bytes reach the trash, not free space.** Because reclaim routes removals through the trash (R3), a full pass
moves ~650 MB into `~/.sase/artifacts/trash` and `du` does not drop until those entries pass the grace period or
`sase artifact trash purge` runs. Say so in the command's own summary output as well as in the docs - a user who expects
instant disk savings and sees none will assume the command failed.

**`src/sase/artifact_cli/reclaim.py`** - `handle_reclaim(args) -> int`, dry run by default (R5), rendering per-row
outcome, discovered commit, and the total bytes that would be freed, with a trailing summary of the unresolved buckets.
Options: `-a/--apply`, `-d/--max-history-scan N`, `-j/--json`, `-l/--limit N`, `-p/--project`.

**Verification for this phase is unusual and deliberate.** Automated tests run against temp git repos and a temp
`SASE_HOME`. Beyond that, run `sase artifact reclaim` in **dry run against the real store** - it is read-only - and
report the projected row count and reclaimed bytes in the phase's completion notes. Do **not** run `--apply` against the
real store; that is the user's call, and the docs phase tells them how.

**Tests.** `tests/artifact_file_facade/test_reclaim.py` over `tempfile` git repos (see `init_test_git_repo()` in
`tests/_sdd_commit_helpers.py`): a clean pushed file reclaims and its content re-resolves; a file whose content was
never committed is left alone; a file whose matching commit sits 15 commits back is found within the bound and missed
outside it; an unpushed commit is refused (durability); a missing checkout, an unknown repo, and a probe timeout all
leave the row untouched; an explicit row is never a candidate; a sidecar-nested `source_path` maps to the right repo;
rerunning reclaim is idempotent. Extend `tests/test_artifact_file_e2e.py` with capture -> reclaim ->
`sase artifact path` returning verified content -> `doctor --verify` clean.

## 7. Opt-in retention configuration and enforcement

Add the retention block to `src/sase/default_config.yml` beside the `artifacts.capture` block that `sase-b7` shipped:

```yaml
artifacts:
  retention:
    # Retention removes nothing until this is true. Explicit artifacts, artifacts
    # referenced by a bead, plan, or ChangeSpec, and the newest capture of every
    # label are kept regardless of the settings below.
    enabled: false
    # Automatic captures kept per label; older generations are trashed first.
    # 0 disables the generation predicate.
    keep_per_label: 3
    # Trash automatic captures created more than this many days ago.
    # 0 disables the age predicate.
    max_age_days: 90
    # Days a trashed artifact stays restorable before purging removes it.
    trash_grace_days: 14
```

Ship _disabled_ with generous values pre-filled, which is the research's own answer to its open question 2: nothing is
reclaimed until asked, and enabling it later is a single flag flip rather than a policy design exercise.

Mirror the block in `src/sase/config/sase.schema.json` (`additionalProperties: false`, integer minimums, descriptions
written to read well in `sase config` output), add `DEFAULT_ARTIFACT_RETENTION_*` constants and
`get_artifact_retention_*()` accessors to `src/sase/config/core.py` exported from `src/sase/config/__init__.py`, copying
the fail-open, type-checked shape of `get_task_history_limit()` and the `get_artifact_capture_*()` accessors beside it.
Then replace the three `DEFAULT_*` module constants the `py-report` phase put in
`src/sase/core/artifact_file_retention.py` with these accessors and delete them, so exactly one source of truth remains.

**Enforcement.** In `src/sase/axe/run_agent_exec_finalize.py`, after `_collect_default_artifacts()` returns, run one
bounded retention pass when `enabled` is true: plan with the configured policy plus the protected-id scan, trash what it
selects, then purge trash entries older than `trash_grace_days`. Print one `[artifacts]` summary line - rows trashed,
bytes reclaimed, entries purged - beside the existing capture summary. Wrap it in the same defensive `except Exception`
that already degrades capture failure to a flag: **retention must never fail a run**. When any protection source is
unavailable, skip the pass entirely and say so in the summary line (R2, R6).

**Tests.** Config accessor tests for defaults, overrides, and malformed values (fail open to the default). A
finalization test with retention disabled asserting the index is byte-identical afterwards; one with retention enabled
asserting the expected rows moved to trash and expired entries purged; and one where the protection scan reports an
unavailable source, asserting no rows were touched.

## 8. Documentation, skill, and configuration reference

Sequence this after `sase-b7.5` lands its documentation. Extend the sections that phase creates - do not duplicate them.

- `docs/cli.md`: add rows for `sase artifact prune`, `reclaim`, `stats`, and the `trash` group to the artifact table,
  keeping it alphabetical, and describe the dry-run default and the trash contract in the prose beneath it.
- `docs/configuration.md`: document the `artifacts.retention` block in the same table idiom used for the other top-level
  blocks, and cross-link it from the `artifacts.capture` documentation.
- The artifact documentation `sase-b7.5` creates: add a "Store lifecycle" section covering the report -> dry run ->
  opt-in retention progression, the reclaim workflow (including the fact that reclaim changes a row's id and why), the
  trash layout and grace period, the protection rules as a bulleted contract, and how to diagnose a row that reclaim
  reports unresolved.
- `src/sase/xprompts/skills/` `sase_artifact_file` source: agents must know that automatic captures are subject to
  retention while declared artifacts are not, and that `file:` refs they hand to a human or persist in a bead are
  protected from pruning. Read the `generated_skills` long-term memory first and regenerate the deployed skill the way
  that note prescribes.
- Make each new subcommand's `--help` complete and scannable, per the `cli_rules` long-term memory.

## 9. Explicitly out of scope

- **Recommendation 5, consumption recording at `@`-ref expansion.** The research names "never prune anything with
  recorded consumption" as a protection, but nothing records consumption today. The retention planner takes
  `protected_ids` as an opaque set, so adding consumption later is a new contributor to that set and not a redesign.
- **Recommendation 1, copy-by-default for `sase artifact create`.** Still unfixed and still a two-line change plus docs;
  it belongs with the declaration workflow, not with retention.
- **The 2.49 GB agent-directory tier** (the research's open question 4). Bringing it under the same policy is a real
  follow-up, and the core trash primitives here are deliberately generic enough to serve it, but doing both at once
  would double this epic.
- **The mobile gateway and TUI surfaces** for stats, prune, and trash. This epic ships one JSON envelope per surface so
  those can be wired cheaply later; it wires none of them.
- **Content-addressed blob storage**, bundles, aliases, and attestations - all deferred by the research for reasons this
  epic does not change.

## 10. Risks

- **Deleting something that mattered.** Layered: R1 and R2 exclude by construction, R7 keeps a floor, R3 makes every
  removal reversible, and R5 means the first run of any predicate is a report. The genuinely irreversible operation is
  `trash purge`, which acts only on entries the user already saw in `trash list`.
- **Under-protection through an unread source.** The failure mode that would break trust. Handled explicitly: a
  protection source that cannot be read blocks `--apply` rather than being treated as empty.
- **Reclaiming wrong bytes.** Structurally prevented by R4: the exact content is reproduced and digest-verified before
  the copy is dropped, and `doctor --verify` re-checks every reference row afterwards.
- **Reclaim's id change.** A reference row's id derives from its VCS identity, so reclaiming changes the id. Mitigated
  by printing both ids, by R2 protecting rows anything references, and by reclaim being opt-in per invocation. Note it
  in the docs phase.
- **Reclaim latency on the full backlog.** ~4,000 rows across ~450 relpaths, one batched `cat-file` per repo and a
  bounded `rev-list` per relpath. `-l/--limit` exists so the first real run can be a small one.
- **Effort versus the research's estimate.** The research sized this "M". It is larger, for the same reasons the
  `sase-b7` plan gave: a `sase-core` release boundary, trash semantics that need restore to be worth anything, and the
  retroactive migration the research assigns to this item in one sentence. The phase sizes here reflect the real work.

## 11. Verification

Every phase runs `just install` first - ephemeral workspaces drift - then `just check`. The core phase runs the
`sase-core` repo's own check target before committing. New public symbols should find their consumer inside the same
phase; if a phase genuinely lands one ahead of its consumer, add an `--epic-symbol "<bead id>(<symbol>)"` entry to
`_lint-symvision` in the `Justfile` and delete it in the consuming phase, per the `symvision` long-term memory. Do not
reach for pragmas.

Sanity checks after the prune and reclaim phases, all read-only against the real store:

```bash
sase artifact stats                    # totals, growth, redundancy, projections, protections
sase artifact stats -j                 # stable JSON envelope
sase artifact prune -g 3               # dry run: what keeping 3 generations would trash
sase artifact reclaim                  # dry run: rows verifiably reproducible, bytes recoverable
sase artifact trash                    # empty until something is applied
sase artifact doctor                   # unchanged health contract
```

Mutating exercises (`prune --apply`, `reclaim --apply`, `trash purge`, `trash restore`) run only against a scratch
`SASE_HOME`.
