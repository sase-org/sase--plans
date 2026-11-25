---
tier: tale
title: Stop large-hood publication cost from outgrowing the 120s drain alarm
goal:
  Targeted agent-hood publication no longer decodes every owner snapshot per publish and
  renders large hoods in near-linear time with byte-identical output, so apollo's
  bob-cli research-hood drains stay well inside the drain alarm.
size: medium
proposed_by: bbugyi200.athena.1g5
proposed_with: claude/opus
create_time: 2026-11-25 14:47:14
status: wip
---

# Plan: Stop large-hood publication cost from outgrowing the 120s drain alarm

## Problem

Apollo's error digests keep reporting `agent_publication` drain timeouts for
`gh_bobs-org__bob-cli`. On 2026-11-25 alone there were three: 07:22
(`inventory_build did not complete within 30s`), 12:47 (the same), and 14:16
(`agent-hood publication did not complete within 120s`). Each one opens a standing alert
at `attempt1`. The next pass then publishes successfully, so the alert closes and opens
again later. Nothing was lost, but the noise will keep coming, because the cause grows
every day.

Evidence, from apollo's `~/.sase/logs/agent_publication_drains.jsonl` (Nov 18–25):

- **The `research` hood is the outlier.** Every bob-cli research agent (named
  `research.*`) lands in one top-level hood. That hood held 937 runs on Nov 11, about
  1,180 on Nov 18, and 1,500 today, growing about 40 runs a day. Most other bob-cli
  hoods have under 200 runs.
- **Research-hood drains on apollo are slow and getting slower.** Successful drains take
  about 47s wall time at p50, 74s at p90, and up to 114s. `prepare` takes 24.5s at p50
  and `inventory_build` 9.3s. The drain's work alarm fires at about 105s: the configured
  120s minus the 15s cleanup reserve. All 7 research-hood drains that timed out this
  week scanned 1,250–1,515 runs.
- **There is also a floor of about 10s that is not tied to hood size.** Hoods with fewer
  than 30 runs still take about 10s in `prepare`, because every targeted publish decodes
  every owner snapshot. bob-cli on apollo has about 1,230 hood snapshots.

A read-only pyinstrument profile on apollo (load average 8, current master `df9ce4ef55`)
gives this breakdown:

```
.venv/bin/pyinstrument tools/agents_publication_profile --project gh_bobs-org__bob-cli --hood research
inventory 14.1s; plan_targeted_hoods(research) 43.0s; 1,419 runs; 10,176 files; 42 MB payload
```

| Cost inside `plan_targeted_hoods` (profiled)                                                                         |  Time | Scales with              |
| -------------------------------------------------------------------------------------------------------------------- | ----: | ------------------------ |
| `load_validated_publication` → `read_hood_snapshot`/`decode_hood_snapshot` for ~1,230 snapshots                      | 13.0s | all hoods in the project |
| `build_hood_kinship` → `_node_projection` (1.65M generator steps for 1,283 nodes)                                    |  6.5s | hood size **squared**    |
| `render_neighbors_section` → `relative_page_url` → `posixpath.relpath` (101k calls, each calling `abspath`/`getcwd`) |  6.2s | hood size × neighbors    |
| `build_hood_snapshot` (gating, merge, verify)                                                                        |  7.3s | hood size                |
| Other rendering, bead links, `previous_snapshot`                                                                     |  ~10s | mixed                    |

Earlier fixes each removed a constant factor:

- `sase-1qh.3` stopped re-scanning bytes that were already published.
- The Nov 11 change cached protection-ledger lookups.
- `sase-1tz.2.1.5` added inventory reuse.

None of them touched these three hot spots. Two of the three grow with the hood, and the
kinship one grows quadratically, so the research hood will keep pushing drains past the
alarm.

## Goal

Make the publish cost of a large hood roughly linear in hood size with small constants.
Remove the per-publish full-project snapshot decode. Every published byte must stay the
same, and validation must stay as strong as today.

## Changes

### 1. Linear kinship projection (`src/sase/agents_sync/rendering_kinship.py`)

`_node_projection` scans every node once for descendants, then once more for each
ancestor hood in the node's chain. That is O(N²) per hood render.

- In `build_hood_kinship`, build the indexes once from `ordered_nodes`, keeping their
  existing sort order:
  - **`members_by_ancestor`:** maps a hood name to the nodes whose `chain` contains it.
  - **`strict_descendants`:** maps a name to the nodes that have it in `chain[:-1]`.
- Rewrite `_node_projection` to read from those indexes:
  - Collect visible rows by walking the pre-sorted member list and skipping names in
    `assigned`, until `NEIGHBOR_GROUP_LIMIT` rows are collected.
  - Compute `overflow_count` arithmetically: the member count, minus the assigned
    members, minus the visible rows. Do not materialize the filtered list.
  - Per-node cost must be proportional to its visible rows plus its `assigned` set,
    never to the hood size.
- Keep the semantics exactly as they are today:
  - the ancestor → descendant → nearest-to-farthest hood group order;
  - the `assigned` exclusion across groups;
  - the `_node_sort_key` ordering;
  - de-duplication when a name appears in a chain more than once.
- Output must be **byte-identical**. Keep the current algorithm as a private reference
  inside the test module only, and compare against it.

### 2. String-only relative page URLs (`src/sase/agents_sync/rendering_markdown.py`)

`relative_page_url` calls `posixpath.relpath`, which resolves both paths through
`abspath` and `os.getcwd()` on every link.

- Compute the relative path with pure string and component arithmetic when both paths
  are already normalized repo-relative POSIX paths, meaning none of the following:
  - a leading `/`;
  - an empty component, `.`, or `..`.
- The result must equal `posixpath.relpath(path, posixpath.dirname(source))` for every
  such input, including:
  - a root-level `source` (`README.md`, which has an empty dirname);
  - the same directory, which returns the file name;
  - `path` equal to the source directory, which returns `.`.
- For any other input, fall back to the existing `posixpath.relpath` call, so behavior
  never changes.
- Keep the `#anchor` handling and `page_url` encoding exactly as they are.
- Leave callers unchanged: `rendering_kinship`, `rendering_agent_page`,
  `rendering_agent_session_page`, `rendering_index_pages`, and `rendering_variables` all
  benefit.
  - `src/sase/bead_pages/rendering_tables.py` keeps its own helper. Do not touch it in
    this change.

### 3. Digest-keyed roster summaries for unselected hoods (targeted planning only)

In targeted mode, `plan_targeted_hoods` needs full snapshots only for the selected
hoods. For every other hood, `render_targeted_browsing_payload` reads only the machine
roster fields that `render_machine_page` prints: `local_hood`, run count, session count,
and state counts. Root and user pages already render from manifests alone. Yet
`load_validated_publication` fully decodes every snapshot of every owner on every
publish, and each publish runs in a fresh `sase_job_agent_publication` subprocess, so
nothing carries over between ticks.

- **Add a summary record.** Add a small frozen `HoodRosterSummary` (`local_hood`,
  `run_count`, `session_count`, state-count mapping).
  - Add a `from_snapshot` constructor.
  - Change `render_machine_page` (in `rendering_index_pages.py`) to render from
    summaries, and keep a formatter that produces exactly today's `state_counts` string.
  - Full-mode `render_browsing_payload` and every other caller convert their snapshots
    to summaries, so their output stays byte-identical.
- **Opt in from targeted planning.** Give `load_validated_publication` an opt-in keyword
  (for example `summarize_unselected=True`). Only `plan_targeted_hoods` passes it. For
  each manifest entry that is not in `override_snapshots`:
  1. Read the raw bytes and run the existing `content_digest(raw) != entry.digest`
     check, exactly as today. Bytes on disk are always checked against the manifest.
  2. Look the summary up in a persisted cache. The key is (owner username, machine,
     project identity, hood, `entry.digest`).
  3. On a hit, use the cached summary and skip decoding.
  4. On a miss, run today's full path: `read_hood_snapshot` decode, the identity check,
     and `_check_explicit_file_set` when `entry.files` is set. Then record the summary.
  5. Leave entries with an explicit `entry.files` set (legacy) out of the cache and
     always take the full path for them.
  6. Selected and override snapshots, and scoped `verify_run_files`, behave exactly as
     today.
- **Return both views.** The function returns full snapshots for overrides and base
  snapshots, plus summaries for every key. Adjust `plan_targeted_hoods` and
  `render_targeted_browsing_payload` to use the summaries for machine rosters.
  - Rework the `shared_targeted_planning` per-prepare cache so that a multi-hood batch
    still re-reads nothing within one prepare.
  - Its existing rule still holds: never serve a selected hood's unapplied bytes to a
    later plan.
- **Cache storage.** Mirror the `inventory_history.py` history-cache pattern:
  - Store under
    `sase_home() / "cache" / "agent-publication-hood-summaries" / <sha256(project_key + sidecar path)>.json`.
  - Include a `schema_version` and an identity header.
  - Enforce a byte cap.
  - On write, prune keys that are absent from the current manifests.
  - Write atomically with temp-file-and-rename, and only when something changed.
  - Treat any unreadable, oversized, or malformed file as empty. Cache problems must
    never fail planning.
- **Timeout gotcha.** `PublicationDrainTimedOut` subclasses `OSError`. Every `OSError`
  handler on the cache's read and write paths must re-raise it first, as
  `_publication_protection_ledger.py` does. Add a regression test for this.
- **Why this keeps validation equivalent.** The manifest digest pins the exact bytes.
  Every distinct digest still gets one full decode plus the identity and file-set
  validation on this machine before its summary is trusted. A changed or corrupted file
  fails the digest check exactly as it does today.
- **Leave the full-mode callers alone:** `plan_hoods`, repair, doctor, and reconcile.
  Their behavior is unchanged.

### 4. Profiling tool visibility (`tools/agents_publication_profile`)

- Report how many summary-cache hits and misses targeted planning had.
- Add a `--cold` flag that runs the plan against an empty temporary cache, so
  before-and-after comparisons stay honest.
- The tool still writes nothing to the sidecar clone. Update its docstring to say it may
  warm the local summary cache unless `--cold` is passed.

## Tests

- **`tests/agents_sync/test_rendering_kinship.py`:** randomized and hand-built hoods,
  including nested chains, sessions, duplicate-depth names, and groups over
  `NEIGHBOR_GROUP_LIMIT`. Assert projections byte-identical to the in-test reference
  algorithm.
  - Add a scaling guard: a synthetic hood of about 3,000 nodes under one top-level name
    must build its kinship with a near-linear call count. Bound the number of node
    visits, not wall time.
- **`relative_page_url`:** an equivalence test against `posixpath.relpath` over a
  generated set of normalized repo-relative pairs. Add explicit cases for a root-level
  source, the same directory, a target equal to the source directory, anchors, and
  fallback inputs that contain `..`, `.`, `//`, or a leading `/`.
- **`tests/agents_sync/test_publication_targeted.py`** (or a sibling module):
  - A targeted payload with a warm cache is byte-identical to one with a cold cache and
    to the pre-change output.
  - With a warm cache, unselected snapshots are not decoded. Count
    `decode_hood_snapshot` calls.
  - A tampered unselected snapshot file still raises the digest mismatch, whether or not
    a cached summary exists.
  - A new digest is a miss and is fully validated.
  - An identity-mismatched snapshot is never cached.
  - Explicit-`files` entries bypass the cache.
  - A corrupt or oversized cache file is ignored.
  - Pruning drops hoods that are no longer in any manifest.
  - `PublicationDrainTimedOut` raised inside the cache's read and write paths
    propagates.
- **Machine page:** rendering from summaries matches rendering from full snapshots byte
  for byte.

## Verification

- Follow the repo's lint and test rules, using `sase tool run check`.
- **Measure on athena.** On athena, run `tools/agents_publication_profile` before and
  after the change on the same host, in both `--cold` and warm runs:
  - on `gh_sase-org__sase` with one small `--hood`, which shows the O(project) floor
    with 2,000+ hoods;
  - on its largest hood, for the kinship and URL savings.
  - Record those numbers in the commit message.
- **Leave apollo alone.** The coding agent must not install or update apollo.
- **Hand the user an apollo command.** Give the user this command to run after they
  deploy:

  ```bash
  ssh apollo 'cd ~/projects/github/sase-org/sase && .venv/bin/python tools/agents_publication_profile --project gh_bobs-org__bob-cli --hood research --cold && .venv/bin/python tools/agents_publication_profile --project gh_bobs-org__bob-cli --hood research'
  ```

  - Expected result: warm `plan_targeted_hoods(research)` falls from about 26s (about
    43s under the profiler) to well under half that.
  - After a day, apollo's research-hood drains in `agent_publication_drains.jsonl`
    should show p90 wall time well under the about 105s work alarm.

## Acceptance criteria

- Rendered payload bytes are unchanged for both targeted and full publication.
- Targeted planning with a warm cache decodes only the selected hoods' snapshots. The
  number of unselected-snapshot decodes no longer grows with project size.
- Kinship projection cost is near-linear in hood size, enforced by the scaling test.
- `relative_page_url` makes no `getcwd`/`abspath` calls for normalized inputs.
- Validation is unchanged in two respects:
  - every on-disk snapshot's digest is still checked on every targeted publish;
  - every new digest is fully decoded and validated once.
- `sase tool run check` passes.

## Out of scope

- **Hood retention and pruning.** Growth of the manifest and snapshot corpus belongs to
  `sase-11n`. This change removes only the per-publish latency side of that task's
  "O(total-hoods)" symptom; storage growth and the decode cap remain.
- **Incremental inventory and gating for unchanged runs.** That means skipping
  `run_from_artifact` and `gate_run_files` for runs whose sources are unchanged. Do it
  next if the research hood keeps growing after this lands.
- **Error-digest noise policy for retryable timeouts.** The reviewer chose `keep` on
  2026-11-11 (`plan:202611/publication_ledger_lookup_cost.md`).
- **The 120s and 240s budgets**, multi-hood batching, and artifact-link drain speed.
- **Changes to sidecar layout or schema**, and anything in `sase-core`. Publication
  planning and rendering already live in this repo's Python, and no new domain rule is
  introduced.
