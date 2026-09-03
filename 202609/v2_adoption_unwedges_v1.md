---
tier: tale
title: Evidence-backed v1-to-v2 adoption unwedges blocked machines
goal:
  "A machine that imported the lossy legacy v1 agents payload heals on its next `sase
  agent sync`: v2 claims supersede matching v1 registry state, matched v1 artifacts are
  refreshed in place with full v2 data (family, parent linkage, prompt, commits, owner
  provenance, dismissed bundle, saved family group), the forged local-ownership rows
  those v1 records derived disappear, already-dismissed rows stay dismissed, and
  unmigratable edges have a dry-run-first `sase agent names forget-import` escape hatch
  instead of no supported exit."
size: medium
proposed_by: bbugyi200.apollo.sase-w2.4
bead: sase-w2.4
create_time: 2026-09-03 16:03:49
status: wip
---

- **PARENT:** [202609/athena_agent_sync_repair.md](athena_agent_sync_repair.md)
- **BEAD:**
  [sase-w2.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-w2/sase-w2.4.md)

# Evidence-Backed v1-to-v2 Adoption Unwedges Blocked Machines

This is phase `v2-adoption` of epic `sase-w2`
(`plan:202609/athena_agent_sync_repair.md`, bead `sase-w2.4`). Phases
`cleanup-serializer` (`sase-w2.1`), `prompt-capture-at-launch` (`sase-w2.2`), and
`batched-sidecar-reads` (`sase-w2.3`) have landed; this phase builds on them.

## Problem

A destination machine that imported the legacy v1 agents-sidecar payload is permanently
wedged. On the observed machine: 338 `origin: import_v1` artifacts, 651 `import_v1`
registry entries, 1,948 pending v2 hoods that can never land, and 338 forged registry
rows claiming another machine's runs ran locally.

Three verified mechanisms, all in this repo:

1. **The namespace guard rejects every v2 claim over v1 state.**
   `ensure_import_namespace_available` in
   `src/sase/agent/names/_registry_mutation_support.py` iterates every registry entry at
   or under the incoming owner root. Its `legacy_source_machine` tolerances are
   reachable only in the `source_owner is None` (v1-over-v1) branches. A v2 claim
   (`source_owner is not None`) therefore collides with (a) the synthetic
   `owner_namespace` root the v1 path created via
   `owner_namespace_entry(source_machine, namespace_kind="legacy_source_machine")`, and
   (b) every unrelated `origin: import_v1` sibling under that root. Every v2 hood from
   that machine raises `ImportedNameCollisionError` and quarantines.

2. **Even with the root unblocked, the per-name claim collides.** For a
   same-user/other-machine import, `localize_imported_agent_name` maps
   `bbugyi200.athena.06--code` to `athena.06--code`, which is exactly the v1 spelling
   already in the registry. In `_apply_imported_v2_registry_claim`
   (`src/sase/agent/names/_registry_import_mutations.py`) the existing entry satisfies
   none of `exact_local_refresh` / `same_claim` / `exact_local_container` /
   `same_family_root`, so the batch raises. There is no v1-to-v2 upgrade rule anywhere.

3. **The planner has no notion of a v1 predecessor.** `preflight_hood` in
   `src/sase/agents_sync/v2_import_planning.py` only looks for prior _v2_ imports
   (`existing_project_imports`, keyed on `imported_source_owner` +
   `imported_source_run_id`) and, for `EXACT_OWNER` only, local observations. A v1
   artifact matches neither, so the plan would allocate a brand-new destination
   timestamp and leave the lossy v1 artifact, its forged dismissed bundle, and its
   registry rows behind forever.

The forged rows are derived state, not hand-written: the registry is rebuilt from
artifacts and dismissed bundles (`_registry_scan_collectors.py`), and a manual dismissal
of a v1 row wrote a dismissed bundle with no import provenance, so `_entry_provenance`
stamped it `origin: local` with a globalized `canonical_global_name`. Repairing the
_source_ record therefore repairs the registry.

All 338 v1 imports on the observed machine map deterministically to v2 runs: the source
run id is `sha256(project_key \0 workflow \0 durable)` (`source_run_id` in
`src/sase/agents_sync/inventory_io.py`), `durable` is the source artifact's timestamp,
and the v1 import writes the _source_ timestamp as the local artifact directory name
(`_available_artifact_path` in `src/sase/agents_sync/bundles.py`). So the destination
can recompute the source run id from local state alone.

## Fix Design

Five pieces, all Python in this repo. Nothing here is shared backend semantics that
another frontend would reimplement — this is local import planning, local registry
mutation, and a CLI. No `sase-core` change and no feature flag (this is a bug fix; the
old wedged branch is not worth preserving, and `v1-retirement`, the next phase, owns the
sunset flag).

### 1. Namespace guard: accept a v2 claim over legacy v1 state

`src/sase/agent/names/_registry_mutation_support.py`,
`ensure_import_namespace_available`:

- In the `container_kind == "owner_namespace"` branch, add a tolerance: when
  `source_owner is not None`, `existing_owner is None`,
  `raw_entry.get("namespace_kind") == "legacy_source_machine"`, and
  `source_owner.machine_name == source_root`, `continue`. This is the typed owner root
  the v1 path reserved for the same machine the v2 claim comes from.
- In the `elif source_owner is not None:` branch (ordinary entries under the root), add
  a tolerance: when `raw_entry.get("origin") == "import_v1"` and
  `raw_entry.get("legacy_source_machine") == source_owner.machine_name`, `continue`.
  Unrelated v1 siblings under the root must not block a v2 claim for a different run;
  per-name conflicts are decided by `_apply_imported_v2_registry_claim`, not by the root
  sweep.
- Everything else is unchanged. In particular a v1 entry whose `legacy_source_machine`
  is a _different_ machine still collides, and the `ImportedNamespaceOwnedLocallyError`
  selection is untouched.

### 2. Evidence-backed v1 match in the import planner

New module `src/sase/agents_sync/v2_import_v1_adoption.py`:

- `LegacyV1Artifact` (frozen dataclass): `path`, `source_machine`, `v1_name`,
  `source_run_id`, `chat_path: Path | None`, `commit_shas: frozenset[str]`.
- `V1AdoptionAmbiguityError(AgentsSyncFormatError)`.
- `LegacyV1AdoptionIndex`: `by_key: dict[tuple[str, str], tuple[LegacyV1Artifact, ...]]`
  keyed by `(source_machine, source_run_id)`, plus a mutable `consumed: set[Path]`.
- `legacy_v1_adoption_index(project_key, rows) -> LegacyV1AdoptionIndex` — pure over
  supplied `(artifact_dir, meta, done)` rows so the caller keeps the existing
  `iter_agent_artifact_dirs` test seam. A row qualifies only when
  `meta["imported_owner_kind"] == "username_unknown_v1"`,
  `meta["imported_from_machine"]` is a non-empty string, `meta["name"]` is a non-empty
  string, and `meta.get("imported_source_owner") is None`. The recomputed id is
  `source_run_id(project_key, ACE_RUN_WORKFLOW_DIR, durable)` with
  `durable = meta["artifact_agent_id"] or done["artifacts_timestamp"] or artifact_dir.name`
  (`done.json` is unlinked by dismissal, so the directory name is the normal fallback).
  `commit_shas` comes from `commit_markers` if present; today's v1 import writes none,
  so it is normally empty.
- `find_v1_adoption(index, owner, payload) -> LegacyV1Artifact | None`:
  - candidates = `index.by_key[(owner.machine_name, payload.record.source_run_id)]`
    minus `index.consumed`.
  - keep only candidates whose `v1_name` equals the v2 global name minus the newly-known
    username: `payload.record.global_name` must start with `f"{owner.username}."` and
    the remainder must equal `v1_name`.
  - zero survivors -> return `None` (ordinary new import).
  - more than one survivor -> raise `V1AdoptionAmbiguityError` naming the global name
    and every candidate path. Never guess.
  - exactly one survivor -> corroborate with shared digests, and raise
    `V1AdoptionAmbiguityError` on contradiction (not silent rejection, so the hood
    quarantines with a diagnostic instead of silently duplicating the run):
    - chat: when the v2 record carries a `chat` file reference _and_ the v1 `chat_path`
      file exists, `content_digest(bytes)` must equal the reference digest.
    - commits: every sha in `commit_shas` must appear in
      `{c.sha.lower() for c in payload.commits.commits}`. An empty `commit_shas` is not
      evidence against.
  - on success, add the path to `index.consumed` so two runs in one pass cannot adopt
    the same artifact.

`src/sase/agents_sync/v2_import_history.py`:

- Replace `existing_project_imports` with
  `scan_local_import_state(target, identity) -> LocalImportState` (frozen dataclass:
  `imports` — today's
  `dict[(username, machine, source_run_id, global_name), (Path, digest|None)]` — and
  `legacy_v1_rows: tuple[tuple[Path, dict, dict], ...]`). One pass over
  `iter_agent_artifact_dirs`; it already reads every `agent_meta.json`, so the only new
  IO is a `done.json` read for rows that look like v1 imports. Update `__all__`
  (symvision fails on an orphaned export).

`src/sase/agents_sync/v2_import_planning.py`:

- `ImportPreflightContext` gains `legacy_v1: LegacyV1AdoptionIndex`;
  `build_import_preflight_context` builds it from
  `scan_local_import_state(...).legacy_v1_rows`.
- In `preflight_hood`, when `match is None` and `observed is None` and
  `classification is not AgentOwnershipClassification.EXACT_OWNER`, call
  `find_v1_adoption`. On a hit: `artifact = adoption.path`,
  `destination = adoption.path.name`, `disposition = "adopted"`,
  `previous_digest = None`, `reserved.add(destination)`, and record
  `superseded_v1_name=adoption.v1_name` on the `PlannedRun`. Adoption is never attempted
  for `EXACT_OWNER` (this machine is authoritative for its own history) and never for a
  run that already has a v2 import or a local observation.
- Pass the plan's adoption authority into the preflight:
  `preflight_imported_registered_names_v2(claims, identity=identity, adopted_v1_artifact_dirs=preliminary_plan.adopted_v1_artifact_dirs)`.

`src/sase/agents_sync/v2_import_types.py`:

- `PlannedRun` gains `superseded_v1_name: str | None = None`.
- `HoodPlan` gains `adopted_runs` and `adopted_v1_artifact_dirs` properties;
  `is_refresh` becomes true for `"adopted"` as well as `"refresh"`; `changed_runs` and
  `is_unchanged` keep their current semantics (`"adopted"` is a changed, non-unchanged
  disposition).

### 3. Atomic in-place refresh, forgery repair, and preserved chat

`src/sase/agents_sync/v2_import_transactions.py`:

- `prepare_transaction` stages the full v2 marker set over the adopted artifact
  directory exactly as it does for a new import. Because `artifact_payload` renders
  `agent_meta.json` and `done.json` from scratch and `_apply_staged_files` writes whole
  files, the v1 keys (`imported_owner_kind: "username_unknown_v1"`,
  `imported_from_machine`, `imported_digest`) are replaced by `imported_source_owner` /
  `imported_source_run_id` / `imported_snapshot_digest`, and localized `agent_family` /
  `agent_clan`, `parent_timestamp`, prompt, and `imported_commits.json` land in the same
  write set.
- **Chat preservation.** Today `artifact_payload` receives
  `chat_path if chat_payload is not None else None`. For an adopted run whose v2 payload
  has no chat, pass the existing v1 chat path from
  `_existing_chat_path(run.artifact_dir)` (the artifact still holds v1 markers at
  prepare time) so adoption never _loses_ a chat the v1 record had. Same for
  `bundle_payload`.
- **Dismissed bundle.** The v2 bundle relative path is
  `f"{run.destination_id[:6]}/{run.destination_id}.json"`, and `bundle_shard_dir`
  (`src/sase/ace/dismissed_agents_paths.py`) shards a manual dismissal's
  `<raw_suffix>.json` into the same `YYYYMM` directory. Adopting in place therefore
  overwrites the forged bundle at its canonical path, which is what durably removes the
  forged `origin: local` registry row on the next rebuild. Also record, in a new journal
  list `stale_bundle_relatives`, any _legacy pre-migration_ duplicate at
  `dismissed_bundles/<destination_id>.json` (the root, not the shard) for an adopted
  run, and unlink those in `_finalize_transaction` after the canonical bundle is
  written. Nothing else is deleted.
- Journal: add `adopted_v1_artifact_relatives` and `stale_bundle_relatives`; bump
  `JOURNAL_SCHEMA_VERSION` to `3` in `src/sase/agents_sync/v2_import_storage.py` and
  default both lists to `[]` when reading schema 1 or 2, mirroring the existing
  `dismissed_identities` upgrade.
- `apply_and_finalize_transaction` passes the journal's adopted artifact dirs to
  `claim_imported_registered_names_v2(..., adopted_v1_artifact_dirs=...)`, so recovery
  of an interrupted transaction replays the same adoption authority.

`src/sase/agent/names/_registry_import_mutations.py`:

- `preflight_imported_registered_names_v2`, `claim_imported_registered_names_v2`, and
  `_apply_imported_v2_registry_claim` take
  `adopted_v1_artifact_dirs: frozenset[Path] = frozenset()` — one hood-scoped adoption
  authority rather than a field duplicated on every claim.
- Candidate lookup: when `adopted_v1_artifact_dirs` is non-empty, also consider the
  legacy spelling `canonical_global_name` minus the leading
  `f"{source_owner.username}."`, so a foreign-username adoption can still find and pop
  the machine-rooted v1 row (the existing `existing_name != claim.localized_name` branch
  already removes it).
- Add an acceptance case alongside `exact_local_refresh` / `same_claim` /
  `exact_local_container` / `same_family_root`:

  ```
  adopts_v1_import = (
      existing.get("origin") == "import_v1"
      and existing.get("legacy_source_machine") == claim.source_owner.machine_name
      and _entry_artifact_dir(existing) in adopted_v1_artifact_dirs
  )
  ```

  where `_entry_artifact_dir` resolves `existing["artifacts_dir"]` with
  `Path(...).expanduser().resolve(strict=False)`. When true, fall through and write the
  fresh v2 entry. This covers both the run claim (whose `artifacts_dir` is the adopted
  directory) and a family/clan container claim whose registry row is an `auto_prefix`
  derived from an adopted member. Because the replacement entry is built fresh by
  `owner_from_artifact_name` + `imported_v2_entry_provenance`, it carries no
  `collision_owners`, so the forged `origin: local` sub-row for that artifact is dropped
  in the same registry write. Any v1 row **not** proved by `adopted_v1_artifact_dirs`
  still raises `ImportedNameCollisionError` and quarantines its hood — migration never
  guesses.

- Thread the new keyword through the facade wrappers in
  `src/sase/agent/names/_registry.py`.

### 4. Preserved dismissed state and honest counts

- Dismissal preservation is a consequence of the design, and must be asserted, not
  assumed: a v2 import always registers its runs as dismissed
  (`_record_imported_dismissed_agents` unions identities and never removes), and
  adoption reuses the v1 `raw_suffix` as `destination_id`, so the adopted run's
  dismissed identity keeps the same suffix. Pre-existing identities for that suffix are
  retained.
- `src/sase/agents_sync/v2_importer.py`: count an adopting hood as `hoods_refreshed` (it
  is not a new import), and append one informational diagnostic per adopting hood:
  `f"{label}: adopted {n} legacy v1 run(s) in place"`. Diagnostics already flow through
  `IntegrationCounts` and `CachedIntegrationResult`; no count model gains a field, so no
  wire schema changes.

### 5. `sase agent names forget-import` — the dry-run-first fallback

New module `src/sase/agents_sync/v1_forget_import.py`, mirroring the shape of
`src/sase/agents_sync/v1_retirement.py` (typed outcome dataclass with `to_json_dict()`,
`dry_run` default, `apply` flag):

- `forget_v1_import(machine, *, apply=False) -> V1ForgetImportOutcome`.
- Validate `machine` with `validate_machine` from `src/sase/agents_sync/io.py`.
- Scan every project under `sase_projects_dir()` (this is local state only; no
  target/remote resolution) for `ace-run` artifacts whose `agent_meta.json` has
  `imported_owner_kind == "username_unknown_v1"` and `imported_from_machine == machine`.
  Nothing else can match: no local run and no v2 import carries that pair.
- For each match collect the closure:
  - the artifact directory;
  - its chat file from `meta["chat_path"]`, only when the expanded path exists and its
    name starts with `imported-` (never a live local chat);
  - dismissed bundle files found by `find_bundle`/`iter_bundle_paths`
    (`src/sase/ace/dismissed_agents_paths.py`) for that `raw_suffix` whose payload's
    `agent_name` equals the v1 name or whose `artifacts_dir` equals the artifact
    directory;
  - the dismissed identities `(AgentType, cl_name, raw_suffix)` derived from those
    bundles.
- Also collect import receipts with `source_owner_kind == "username_unknown_v1"` and
  `source_machine == machine` across projects (`read_project_receipts`,
  `src/sase/agents_sync/incoming_cache_receipts.py`); add a
  `remove_project_receipts(project_key, keys)` helper there that rewrites the receipts
  document minus the given `source_hood_key`s, reusing the existing atomic writer.
- Dry run (default): mutate nothing; return the closure.
- `--apply`: remove artifact directories, call `delete_agent_artifact_index_artifacts`
  for them (`src/sase/core/agent_artifact_index_lifecycle.py`), unlink chat files and
  bundle files, drop the dismissed identities via `save_dismissed_agents`, call
  `sync_dismissed_agent_artifact_index(force=True)`, drop the receipts, then
  `rebuild_name_registry()` and report the `origin: import_v1` rows for that machine
  that survive (expected: zero). Every filesystem failure is captured into
  `outcome.errors` rather than aborting the sweep.

CLI wiring:

- `src/sase/main/parser_agent_storage.py`, `register_agent_names_parser`: register
  `forget-import` **before** `migrate-auto` (subcommands stay alphabetical). Per
  `sase memory read cli_rules.md`, the required machine is a **positional** argument,
  not an option; every long option gets a short alias; options are listed
  alphabetically: `-a/--apply`, `-j/--json`, `-t/--transport` (`choices=("v1",)`,
  `default="v1"`). Write a full `description=` explaining that the command is a dry run
  unless `--apply` is given, and what it removes.
- `src/sase/agents/cli_names.py`: add the `forget-import` branch, JSON output under
  `--json` and a `rich`-rendered summary otherwise (follow
  `src/sase/agents/cli_retire_v1.py`: `Would remove` vs `Removed`, a
  `Run again with --apply` hint on a dry run, errors in red). Update the usage line to
  `Usage: sase agent names {forget-import,migrate-auto}`. Return a non-zero exit code
  when the outcome carries errors.
- Regenerate the checked-in completion snapshot with `just sync-completion-spec`
  (`tests/completion/test_snapshot.py` is a drift gate on
  `tests/completion/snapshots/cli_spec.json`).
- Document the subcommand in `docs/cli.md` (the `sase agent names` table row) and in the
  `names` row of the CLI options table in `docs/configuration.md`.

## Steps

1. Namespace guard tolerances in `_registry_mutation_support.py` (piece 1) with their
   registry tests — this alone unblocks the 1,948 pending hoods whose names do not
   collide.
2. `v2_import_v1_adoption.py` plus the `scan_local_import_state` single-pass scan in
   `v2_import_history.py` (piece 2, matcher half), with pure unit tests for the matcher.
3. Planner integration: `ImportPreflightContext`, `preflight_hood`, `PlannedRun` /
   `HoodPlan` (piece 2, planner half).
4. Registry supersede + forgery repair in `_registry_import_mutations.py` and the
   `_registry.py` facade (piece 3, registry half).
5. Transaction half of piece 3: chat preservation, journal fields, schema bump to 3,
   stale legacy-root bundle cleanup, adoption authority threaded through
   `apply_and_finalize_transaction`.
6. Importer counts and diagnostics (piece 4).
7. `v1_forget_import.py`, the receipts removal helper, parser, CLI renderer, docs, and
   `just sync-completion-spec` (piece 5).
8. Tests (below), then `just install` if the workspace virtualenv is stale, then
   `just check`. Hand `just check-full` to `/sase_monitor` if the scoped run escalates
   or reports an unusual selection.

## Tests

Reuse `tests/agents_sync/v2_importer_fixtures.py` (`published_package`,
`isolate_local_state`, `SOURCE_OWNER = bob@zeus`, `LOCAL_OWNER = alice@athena`). Add
`tests/agents_sync/v1_adoption_fixtures.py` with a `wedged_machine(...)` helper that
seeds, for a published v2 hood, the v1 state a wedged destination would hold: an
artifact directory named with the source timestamp containing an `agent_meta.json` with
`name: "zeus.crew--plan"`, `imported_owner_kind: "username_unknown_v1"`,
`imported_from_machine: "zeus"`, `imported_digest`, and a `chat_path` pointing at a
written chat file; no `done.json` (dismissal unlinks it); a dismissed bundle at
`dismissed_bundles/<YYYYMM>/<raw_suffix>.json` with no import provenance and a
`dismissed_agents.json` identity for it. The helper must derive the artifact directory
name from the timestamp the fixture publishes, so the recomputed `source_run_id`
genuinely matches rather than being hand-forced.

Because `isolate_local_state` stubs out the real registry, split the suites:

- `tests/agents_sync/test_v2_import_v1_adoption.py` (uses `isolate_local_state`):
  - **Unique match promotes in place.** After `integrate_v2_hoods`, no new artifact
    directory is created, the adopted directory's `agent_meta.json` has
    `imported_source_owner`, `imported_source_run_id`, localized `agent_family`, and no
    `imported_owner_kind`; `raw_xprompt.md` and `imported_commits.json` exist; counts
    report `hoods_refreshed == 1`; a diagnostic names the adoption.
  - **Idempotent.** Running the same import twice leaves the second run
    `hoods_unchanged == 1` with no further writes (the artifact is now discoverable
    through the v2 `imports` map).
  - **Ambiguity quarantines without mutation.** Two v1 artifacts recomputing to the same
    `(machine, source_run_id)` and name -> `hoods_quarantined == 1`, the diagnostic
    names both paths, and both artifacts still hold their v1 `agent_meta.json`
    byte-for-byte.
  - **Contradicting chat digest quarantines.** Same shape, with the single candidate's
    chat bytes altered.
  - **Dismissed state preserved.** The pre-seeded dismissed identity survives, the
    adopted run's identity is present after import, and the canonical bundle at the
    adopted `raw_suffix` is now the v2 bundle (has `imported_source_owner`) rather than
    the forged one. A legacy-root duplicate bundle seeded at
    `dismissed_bundles/<raw_suffix>.json` is gone.
  - **Chat preservation.** With a v2 payload that carries no chat, the adopted
    artifact's `agent_meta.json` still points at the v1 chat file and the file still
    exists.
  - **Crash matrix.** Reusing the `_finalize_transaction` fault injection from
    `tests/agents_sync/test_v2_importer_integration.py`, and additionally failing inside
    `_apply_staged_files`: after the injected failure nothing is loader-visible and no
    partial family or saved group escapes; after `recover_v2_import_transactions` the
    adopted record is complete and the dismissed identity is registered exactly once.
- `tests/test_agent_name_registry_claims.py` (real registry, real `sase_home`
  isolation):
  - A v2 claim for `bob@zeus` succeeds when the registry holds a `legacy_source_machine`
    owner-namespace root `zeus` plus an unrelated `origin: import_v1` sibling
    `zeus.other`, and the sibling is left untouched.
  - With `adopted_v1_artifact_dirs` naming the artifact, a v2 claim replaces the
    `origin: import_v1` entry for the same name: the resulting entry is
    `origin: import_v2` with the right `source_owner` and `canonical_global_name`, and
    carries no `collision_owners` (the forged `origin: local` sub-row is gone).
  - Without that authority, the same claim still raises `ImportedNameCollisionError` and
    the batch persists nothing.
  - A v1 entry whose `legacy_source_machine` is a _different_ machine still collides.
  - Foreign-username adoption pops the machine-rooted v1 spelling and writes the
    username-rooted localized name.
- `tests/agents_sync/test_v1_forget_import.py`:
  - Dry run reports the full closure (artifacts, chats, bundles, identities, receipts)
    and mutates nothing on disk.
  - `--apply` removes exactly that closure, leaves a local run and a v2-imported run
    with the same timestamp shape untouched, drops the matching receipts only, and the
    rebuilt registry has no `import_v1` entry for the machine and no
    `legacy_source_machine` root for it.
  - An unwritable artifact directory is reported in `errors` without aborting the rest
    of the sweep.
- CLI: extend `tests/agents_sync/test_cli.py` (or the nearest `sase agent names` CLI
  test) to assert the parser accepts
  `sase agent names forget-import zeus --transport v1 --json`, that omitting `--apply`
  is a dry run, and that the bare `sase agent names` usage line lists both subcommands.
  `tests/completion/test_snapshot.py` must pass against the regenerated snapshot.

## Out of Scope

- **Deleting the legacy v1 import leg.** Phase `v1-retirement` (`sase-w2.5`) gates it
  behind a sunset flag and writes the decision record. This phase leaves the v1 leg
  working; v1 payloads must stay readable as evidence for this matcher.
- **Typed owner identity in `sase-core`.** Phase `typed-owner-identity` (`sase-w2.6`)
  fixes `agent_local_hood` / `foreign_agent_owner_root` / `globalize_owned_agent_name`.
  This phase deliberately keeps the current string-projection localization rule and does
  not touch `sase-core`.
- **Archive/visibility/capability modelling** (`sase-w2.7`) and **imported family UI**
  (`sase-w2.8`).
- **Orphaned v1 chat files after adoption.** When the v2 payload carries its own chat,
  the old `~/chats/<shard>/imported-<name>-<ts>.md` file is left in place rather than
  deleted inside the transaction. It is unreferenced and harmless, and `forget-import`
  is not the right tool because the record was adopted, not forgotten. Already recorded
  as a `PROPOSED FOLLOW-UP:` note on `sase-w2.4`; do not widen this change to cover it.
- **New count fields on `IntegrationCounts` / `CachedIntegrationResult`.** Adoption is
  reported through `hoods_refreshed` plus a diagnostic so no serialized cache schema
  changes here.
