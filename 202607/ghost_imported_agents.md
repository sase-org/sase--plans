---
tier: epic
title: Stop ghost imported agents from flooding the Agents tab
goal: 'Agents-sync imports never mint future-dated agent artifacts, never re-publish
  themselves, and are consistently hidden by every Agents-tab load tier, so previously
  dismissed agents stop appearing and disappearing in `sase ace`.

  '
phases:
- id: dismissal_invariant
  title: Make Agents-tab dismissal tier-independent
  depends_on: []
  size: medium
  description: '''Phase 1 — Make Agents-tab dismissal tier-independent'' section:
    fold dismissed-bundle identities into the TUI loader''s effective dismissed set
    behind a signature-cached snapshot so index-backed and source-scan loads hide
    the same rows.

    '
- id: import_dismissal_record
  title: Record imported dismissals in dismissed_agents.json
  depends_on: []
  size: small
  description: '''Phase 2 — Record imported dismissals in dismissed_agents.json''
    section: extend the v2 import transaction so finalizing an import adds each imported
    run identity to the dismissed-agents state file alongside the bundle it already
    writes.

    '
- id: timestamp_provenance
  title: Stop minting future-dated imported timestamps
  depends_on: []
  size: medium
  description: '''Phase 3 — Stop minting future-dated imported timestamps'' section:
    carry real commit timestamps through commit-only publication records and make
    the import destination-timestamp fallback provably never exceed the current time.

    '
- id: import_loop_break
  title: Break the publish/import/re-publish amplification loop
  depends_on: []
  size: medium
  description: '''Phase 4 — Break the publish/import/re-publish amplification loop''
    section: stamp import provenance onto dismissed bundles, teach the inventory import
    detector to recognize it, and stop self-imports from re-materializing history
    the local machine already owns.

    '
- id: state_repair
  title: Repair existing ghost artifacts, bundles, and registry rows
  depends_on:
  - import_dismissal_record
  - timestamp_provenance
  - import_loop_break
  size: medium
  description: '''Phase 5 — Repair existing ghost artifacts, bundles, and registry
    rows'' section: add an idempotent, dry-run-by-default repair command that purges
    already-written future-dated imported state, plus a round-trip regression test
    proving publish/import/re-publish converges.

    '
create_time: 2026-07-25 15:10:47
status: done
bead_id: sase-9o
---

- **PROMPT:** [202607/prompts/ghost_imported_agents.md](prompts/ghost_imported_agents.md)

# Plan: Stop ghost imported agents from flooding the Agents tab

## Problem

Screenshot `20260725_143902.png` shows the Agents tab with `215 [10/10 running · 4 queued · 10 waiting · 191 done]` and
an `@default` tribe of `198 [R4 Q1 W2 D191]`. Every visible DONE row is an agent the user had already dismissed, grouped
under nonsensical dates (`Fri Nov 29`, `Fri Mar 8`, `Sun Mar 3`, …). The detail pane reads
`Timestamps: START | 2080-08-06 10:49:01` with neighbors starting `2128-12-10 19:27:19` and `2129-08-07 22:10:19`. Two
hours earlier (`20260725_123931.png`) the same tab looked normal: `76` agents, `@default 21 [R2 W2 D17]`, real names and
real times. The rows come back and go away again, and sometimes only a TUI restart clears them.

## Root cause

Four defects compound. The evidence for each is recorded below so the phase agents can re-verify against live state.

### D1 — Commit-only publication records drop the timestamps they already have

`_add_commit_only_runs()` (`src/sase/agents_sync/inventory.py:501`) synthesizes a publication record for an agent that
exists only in primary git history — its local artifact was already cleaned up. It emits `state="completed"` with
`started_at=None`, `finished_at=None`, and `dismissed_at=None`, even though `commits[-1].committed_at` in the very same
record is a real epoch. It stashes that epoch in the record's `timestamp` field (as a bare epoch string, not a 14-digit
stamp), where the import never looks.

### D2 — The import's timestamp fallback spans the years 2000–2136

With no timestamps to anchor on, `preflight_hood()` calls `preferred_timestamp(run.source_run_id, run.started_at)`
(`src/sase/agents_sync/v2_import_history.py:246`). `source_run_id` is `run-<sha256[:32]>`, so the 14-digit-embedded
branch never matches, `started_at` is `None`, and control reaches the fallback:

```python
seconds = int(hashlib.sha256(source_run_id.encode()).hexdigest()[:8], 16)
return (datetime(2000, 1, 1, tzinfo=UTC) + timedelta(seconds=seconds)).strftime("%Y%m%d%H%M%S")
```

`hexdigest()[:8]` is 32 bits, so `seconds` reaches 4 294 967 295 — about 136 years. The fallback is therefore uniform
over **2000-01-01 … 2136-02-07**. The comment calls this "a stable fallback in a historical range"; it is not
historical.

Verified byte-exact against live state: for the artifact
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/212804/28/21280428200542/`,
`preferred_timestamp("run-8678ecb9276f71ab202bdc355ebf66e2", None)` returns `"21280428200542"`.

This value becomes the artifact directory name, which becomes the agent's `raw_suffix`, which
`parse_timestamp_14_digit()` turns into `Agent.start_time`
(`src/sase/ace/tui/models/_loaders/_done_loaders.py:435-436`). So one bad fallback poisons identity, sort order, and
date grouping at once.

On disk today: `~/.sase/dismissed_bundles/` holds 512 shard directories, 508 of which are non-2026 (`200003`, `201*`,
`2043*`, `2076*`, `2128*`, `2133*`, …); 600 of 12 025 bundles are future- or past-dated; the artifact index holds 437
such artifacts out of 5 191.

### D3 — Imports never write to `dismissed_agents.json`, so the two load tiers disagree

`prepare_transaction()` / `_finalize_transaction()` (`src/sase/agents_sync/v2_import_transactions.py:77,272`) write, for
each imported run: artifact markers, a dismissed bundle under `dismissed_bundles/<ts[:6]>/<ts>.json`, saved family
groups, and then `sync_dismissed_agent_artifact_index(force=True)`. That projection **does** include bundle identities
(`src/sase/core/agent_artifact_index_lifecycle_projection.py:65`), so the SQLite artifact index marks every imported row
dismissed.

Nothing ever adds those identities to `~/.sase/dismissed_agents.json`.

The Agents tab has two artifact sources (`src/sase/ace/tui/models/_agent_loader_artifacts.py`):

- **Index-backed**: `query_agent_artifact_index(...)`. The index applies its own dismissed projection, so imported rows
  are excluded upstream. This is the steady state, and it is why the 12:39 screenshot looks clean.
- **Source scan**: `scan_agent_artifacts()` — used for `full_history=True` Tier 2 reconciles and, critically, for the
  bounded Tier 1 fallback (`_TIER1_FALLBACK_SCAN_OPTIONS`, `max_records=200`, `newest_first=True`) taken when the index
  is missing, when its query fails, or when `_artifact_index_schema_bypass` is set. This path walks the filesystem and
  knows nothing about the index projection.

For source-scan loads the only remaining filter is the in-memory dismissed set, which `compute_apply_loaded_agents()`
(`src/sase/ace/tui/actions/agents/_loading_compute.py:232-282`) builds from `dismissed_snapshot` (loaded from
`dismissed_agents.json`) plus `recovered` — and `recovered` only ever contains rows with
`_loaded_from_dismissed_bundle`, which artifact-loaded rows never have. So the imported rows survive the filter.

Verified on live state: of the 437 non-2026 artifacts in the index, **437/437** appear in the index's `dismissed_agents`
table and **0/437** appear in `dismissed_agents.json`.

This is the direct cause of the reported symptom, and it explains the exact shape of it. With `newest_first=True` and a
200-record cap, future-dated ghosts sort ahead of every real agent and consume nearly the whole budget — which is why
the broken screenshot shows 191 done rows that are _all_ ghosts and no real completed agents at all.
`_run_startup_agents_load()` (`src/sase/ace/tui/actions/_startup_loads.py:150-200`) sets
`_artifact_index_schema_bypass = True` when the index schema is stale, serves that bounded fallback for first paint,
rebuilds in the background, and clears the bypass only if the rebuild succeeds — so the ghosts appear, then vanish on
the follow-up index-backed refresh, and persist for the whole session when the rebuild fails. That matches "show up and
then disappear (sometimes only after I restarted the TUI)" precisely.

### D4 — Imported bundles are not recognizable as imported, so they get re-published

`bundle_payload()` (`src/sase/agents_sync/v2_import_rendering.py:187-213`) writes `imported_transaction_key` and
`step_output.imported_source_run_id`, but **not** `imported_source_owner`. `inventory_io.is_imported()`
(`src/sase/agents_sync/inventory_io.py:237`) only checks `imported_from_machine`, `imported_digest`,
`imported_source_owner`, and `source_owner`. The artifact's `agent_meta.json` does carry `imported_source_owner`, so
`_run_from_artifact()` correctly skips it — but `_run_from_dismissed()` reads the _bundle_, sees no import marker, and
re-publishes the imported run as though it were local work.

Proven against the `agents` sidecar, hood `02v`, on two consecutive 2026-07-24 sync commits:

|                      | `state`     | `started_at`                | `source_run_id`                        |
| -------------------- | ----------- | --------------------------- | -------------------------------------- |
| round 1 (`785c0044`) | `completed` | `null`                      | `run-3b781910b256d60910ddb0612286e4f2` |
| round 2 (`8b6a6782`) | `dismissed` | `2089-06-02T09:50:55+00:00` | `run-0363e8a3299df47236e51951fc4c68ed` |

and the two ids are exactly reproducible:

- `_source_run_id("gh_sase-org__sase", "primary-commit-history", "02v")` → `run-3b78…` (round 1 came from
  `_add_commit_only_runs`)
- `_source_run_id("gh_sase-org__sase", "ace-run", "20890602095055")` → `run-0363…` (round 2 came from
  `_run_from_dismissed` reading the _imported_ bundle `dismissed_bundles/208906/20890602095055.json`)

Because the re-published `source_run_id` differs every round, the next import treats it as a brand-new run and mints
another future-dated artifact. The ghost population grows monotonically with every `sase commit`
(`publish_committed_agent_hood`, `src/sase/workflows/commit/workflow.py:405`). On 2026-07-24 between 22:10 and 23:xx
this produced 304 import journals covering 442 artifacts, **437 of which landed on bogus timestamps**.

Note that all of this is a _self_-import: `imported_source_owner` on every ghost artifact is
`{"machine_name": "athena", "username": "bbugyi200"}` — the local owner. `find_exact_local_observation()` cannot
recognize these as already-owned runs because the local artifacts were deliberately cleaned when the agents were
dismissed, so the import re-materializes history the machine itself retired.

### Why the two screenshots differ

Nothing about the ghosts changed between 12:39 and 14:39; only the load tier did. The 437 ghost artifacts had been on
disk since 2026-07-24 22:00. The 12:39 view was index-backed (ghosts excluded by the index projection). The 14:39 view
was a source scan (ghosts unfiltered, future-dated, and monopolizing the 200-record budget).

## Scope note

Phases 1–4 are independent and may run in parallel. Phase 5 depends on 2, 3, and 4 because it must not repair state that
the still-broken producers would immediately recreate. Phase 1 is deliberately independent of the rest: it fixes the
tier divergence for _any_ producer, not just agents-sync.

All work is in this repo. No `sase-core` change is required: the artifact index's dismissed projection is already
correct, and every fix below is either Python-side loader logic or agents-sync rendering.

---

## Phase 1 — Make Agents-tab dismissal tier-independent

**Files**: `src/sase/ace/tui/actions/agents/_loading_compute.py`, `src/sase/ace/tui/actions/agents/_loading_helpers.py`,
`src/sase/ace/tui/actions/agents/_loading_disk.py`, `src/sase/ace/dismissed_agents.py`

Today "dismissed" means one thing to the SQLite artifact index (`dismissed_agents.json` ∪ dismissed-bundle identities)
and a weaker thing to the TUI's in-memory filter (`dismissed_agents.json` only). Make both use the same definition.

1. Add a signature-cached accessor in `sase.ace.dismissed_agents` — e.g. `dismissed_bundle_identities_snapshot()` — that
   returns the bundle-identity set from `dismissed_bundle_index.query_summary_identities()` and reuses the cached result
   while `dismissed_bundle_index_signature()` is unchanged. `index_signature()` is a cheap four-tuple, so the
   steady-state cost is one stat-like call per refresh, not a 12 000-row scan. Both helpers already exist in
   `src/sase/ace/dismissed_agents.py:126,227`.
2. Thread that set into the loader as a third input beside `dismissed_snapshot`, and union it into `effective_dismissed`
   in `compute_apply_loaded_agents()`. Keep the existing `recovered` mechanism intact — it still serves genuine
   bundle-loaded rows.
3. Load the snapshot on the worker thread, not the UI thread. `_load_agents_async()` and
   `_load_agent_artifact_delta_async()` already wrap their disk work in `asyncio.to_thread`; the snapshot fetch belongs
   in the same hop. Do not add a new refresh path (tui_perf rule 5) and do not introduce disk work into a pump/timer
   callback (rule 2).
4. Also union it into the `dismissed_from_loader` computation in `_apply_loaded_agent_disk_projections()`
   (`src/sase/ace/tui/actions/agents/_loading_helpers.py:364-383`) so revive and self-healing see a consistent picture.

**Tests**: a loader test that feeds a source-scan snapshot containing an agent whose identity is only in the bundle
index (not in `dismissed_agents.json`) and asserts it is filtered out; the mirror case for an index-backed snapshot; and
a test that the cached snapshot is not re-read when the bundle-index signature is unchanged.

**Acceptance**: with the current corrupted `~/.sase` state, a `full_history=True` Agents load and a
`use_artifact_index=False` bounded load both return zero future-dated rows.

## Phase 2 — Record imported dismissals in `dismissed_agents.json`

**Files**: `src/sase/agents_sync/v2_import_transactions.py`

The import already decides that every imported run is dismissed — that is what the bundle write means. Make that
decision durable in the state file too.

1. In `prepare_transaction()`, record the dismissed identity triple `(agent_type, cl_name, raw_suffix)` for each
   rendered bundle into the journal (a new `dismissed_identities` list). Deriving it in `prepare` keeps
   `apply_and_finalize_transaction()` replay-safe and keeps recovery of an interrupted transaction correct.
2. In `_finalize_transaction()`, before `sync_dismissed_agent_artifact_index()`, load the current dismissed set, union
   the journal's identities, and persist with `sase.ace.dismissed_agents.save_dismissed_agents()`. Then pass the added
   identities to `sync_dismissed_agent_artifact_index(added=...)` rather than relying solely on `force=True`.
3. Make the step idempotent: re-running finalize on an already-complete journal must not duplicate or reorder entries,
   and must not fail if `dismissed_agents.json` is missing.
4. Journal schema changed — bump `JOURNAL_SCHEMA_VERSION` (`src/sase/agents_sync/v2_import_storage.py`) and tolerate
   journals written by the previous version during `recover_v2_import_transactions()`, which still runs against the 304
   journals already on disk.

**Tests**: an import-transaction test asserting the identities land in `dismissed_agents.json`; a recovery test that
resumes an `applied` journal and still writes them; an idempotence test over a repeated finalize; and a compatibility
test that a pre-bump journal recovers without error.

## Phase 3 — Stop minting future-dated imported timestamps

**Files**: `src/sase/agents_sync/inventory.py`, `src/sase/agents_sync/v2_import_history.py`

Fix the producer and harden the consumer.

1. `_add_commit_only_runs()`: set `started_at` (and `finished_at`) from `commits[-1].committed_at`, rendered as a UTC
   ISO-8601 string so `preferred_timestamp()`'s `_parse_datetime()` branch accepts it. Use the earliest commit for
   `started_at` and the latest for `finished_at` — the commit range is a strictly better estimate than either bound
   alone, and both are real. Keep `dismissed_at` as-is. This alone removes the fallback for the overwhelming majority of
   records.
2. `preferred_timestamp()`: replace the fallback so it can never produce a future date. Anchor it in the past and bound
   the span — e.g. hash into a window that ends at "now" rather than starting at 2000 and running 136 years forward —
   and clamp the result to `<= now` unconditionally. Update the docstring/comment, which currently claims a historical
   range it does not deliver.
3. Add a guard in `reserve_timestamp()` (or at its call site in `preflight_hood()`): a destination timestamp later than
   the current time is a bug, not data. Reject or clamp it, and surface a diagnostic through the existing import
   diagnostics channel so a future regression is visible instead of silent.
4. Keep `_source_datetime()`'s fallback-to-`destination_id` behavior in `v2_import_rendering.py` — once (1)–(3) hold,
   the destination is always sane.

**Tests**: a unit test that `preferred_timestamp()` never returns a timestamp in the future across a large sample of
synthetic `source_run_id`s (this is the test that would have caught the bug); a test that a commit-only inventory record
carries commit-derived `started_at`/`finished_at`; and a test that `reserve_timestamp()` refuses a future preferred
value.

## Phase 4 — Break the publish/import/re-publish amplification loop

**Files**: `src/sase/agents_sync/v2_import_rendering.py`, `src/sase/agents_sync/inventory.py`,
`src/sase/agents_sync/inventory_io.py`, `src/sase/agents_sync/v2_import_planning.py`

1. `bundle_payload()`: add `imported_source_owner` (and `imported_snapshot_digest`, for parity with the artifact meta)
   to the rendered bundle. This is the minimal change that makes `inventory_io.is_imported()` return `True` for imported
   bundles. Confirm `Agent.from_bundle_dict()` — already called at the end of `bundle_payload()` — tolerates the new
   keys.
2. `inventory_io.is_imported()`: also treat `imported_transaction_key` as import provenance. The ~12 000 bundles already
   on disk lack `imported_source_owner` but do carry `imported_transaction_key`, so without this the loop keeps running
   for existing data until Phase 5 finishes.
3. `_run_from_dismissed()` (`src/sase/agents_sync/inventory.py:307`): as a belt-and-braces check, also skip bundles
   whose `step_output.imported_source_run_id` is set.
4. Self-import: decide and implement what an `EXACT_OWNER` import should do when `find_exact_local_observation()` finds
   nothing. The recommendation is to skip materialization — the local machine is the authority on its own history, and
   the absence of a local artifact means the user deliberately cleaned it, so re-creating it is wrong by construction.
   Implement this in `preflight_hood()` by dropping such runs from `planned_runs` (they still participate in
   relationship rewriting, as the existing evidence-only branch at `v2_import_history.py:160-168` already does). If the
   phase agent concludes that cross-machine restore requires materialization, document why in the phase's commit message
   and leave (1)–(3) as the loop fix.

**Tests**: a round-trip test that publishing an imported bundle yields no inventory record; a test that `is_imported()`
accepts each of the three provenance markers; and, for (4), a test that an exact-owner hood with no local observation
plans zero new artifacts.

## Phase 5 — Repair existing ghost artifacts, bundles, and registry rows

**Files**: `src/sase/agents/cli_index.py` (or a sibling repair module), `src/sase/main/parser_agent.py`, plus tests

Phases 2–4 stop the bleeding; this phase cleans the wound. Live counts to repair, as measured while writing this plan
(re-measure before acting — the numbers grow with every sync):

- 437 future-dated artifact directories under `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/`
- 600 future-/past-dated bundles across 508 non-2026 shard directories under `~/.sase/dismissed_bundles/`
- corresponding rows in `~/.sase/agent_artifact_index.sqlite` (`agent_artifacts` + `dismissed_agents`) and in the 7.4 MB
  `~/.sase/agent_name_registry.json`
- 304 import journals under `~/.sase/projects/<key>/agents_sync_imports/journals/`

Requirements:

1. Add a repair subcommand — `sase agent index repair` is the natural home given `gc`/`rebuild`/`status`/`verify`
   already live in `src/sase/agents/cli_index.py` (dispatched from `handle_agents_index()`, and registered in
   `src/sase/main/parser_agent.py:295`). Follow the CLI conventions: keep the subcommand list alphabetical (`gc`,
   `rebuild`, `repair`, `status`, `verify`), sort options alphabetically, give every public long option a short alias,
   make `-h` output excellent, and use color where it aids readability.
2. **Dry-run by default.** Print a categorized summary (artifacts, bundles, index rows, registry entries, journals) and
   require an explicit `-a|--apply` to mutate anything.
3. Selection criterion: an imported artifact or bundle whose 14-digit timestamp is in the future. Use "in the future"
   rather than "not 2026" — it is the actual invariant violation, it is stable over time, and it cannot catch legitimate
   old history. Only touch records that carry import provenance; never delete a locally-produced artifact.
4. Prune the corresponding registry reservations in `agent_name_registry.json` (see `src/sase/agent/names/_wipe.py` and
   `_registry_scan.py` for the existing removal helpers) and the matching `agent_artifacts` / `dismissed_agents` index
   rows, then rebuild the dismissed-bundle index and re-sync the artifact-index projection.
5. Idempotent: a second run reports zero work.
6. Also drop the now-meaningless import journals for repaired transactions so `recover_v2_import_transactions()` does
   not resurrect them.

Finally, add the end-to-end regression test that ties the epic together: a publish → import → re-publish → import round
trip over a fixture project must converge — the second import plans **zero** new artifacts, and no artifact timestamp is
in the future.

**Acceptance for the epic**: after this phase, on the real `~/.sase`, a `full_history=True` Agents load, a bounded
`use_artifact_index=False` load, and an index-backed load all return the same set of visible agents, none of them
future-dated.

## Verification

Per repo policy, run `just install` before `just check` in a fresh workspace, then `just check` for every phase that
touches Python. Phase 1 additionally warrants a `SASE_TUI_TRACE=1` capture across an Agents refresh to confirm the
bundle-identity snapshot did not add measurable time to the loader span; the signature fast path should make it
unmeasurable.
