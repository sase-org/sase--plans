---
tier: tale
title: Count committed plans again with size-aware blob reads in project stats
goal:
  sase stats project reports the real number of committed plans for sase (about 2,364 in
  the last 90 days, about 5,542 all time) instead of 0, and a plan store that cannot be
  read shows as unmeasured with an issue rather than a fake zero.
size: medium
proposed_by: bbugyi200.athena.13j
proposed_with: claude/opus
create_time: 2026-10-24 05:40:57
status: wip
---

# Count committed plans again with size-aware blob reads in project stats

## Goal

`sase stats project -p sase -r 90d` prints `0 plans committed`, and every PLANS / EPIC /
TALE cell in the By Model table is `0`. The sase plan store actually holds about 2,364
plans first committed in that window (759 epic, 1,605 tale) and about 5,542 all time.
Fix the Rust core so the count is correct. Make every read failure in this path honest:
an unreadable plan store must report `— plans committed` plus an issue, never a measured
`0`.

All code changes are in the linked **sase-core** repo. Open it with
`sase repo open sase-core -r "<reason>"` and work only in the path it prints. No file in
the sase repo needs to change.

## Root cause (verified)

The sase plan store is the `plans` sidecar repo (`storage: sidecar_repos`,
`plans_dir: "."`). Counting happens in `crates/sase_core/src/project_stats/plans.rs`:

1. `load_records` → `build_index(repo, ["."])` walks the whole sidecar history (about
   10.5 MB of `git log --raw` output today, under budget). Then `records_from_index`
   collects the newest blob OID of every `.md` lineage under the store.
2. It reads those blobs with `git::read_blobs(repo, &oids, budget).unwrap_or_default()`.
3. `project_stats::git::read_blobs` (`crates/sase_core/src/project_stats/git.rs`) sizes
   its `cat-file --batch` chunks by assuming **1 KiB per OID**:
   `chunk = (max_output_bytes / 1024).clamp(64, 65_536).min(oids.len())`. With the
   default 32 MiB budget that is 32,768 OIDs per chunk, so all of the store's ~5.5k+
   plan blobs go into **one** `cat-file --batch` call. Plan files average about 6 KB,
   and that single call produces about 74 MB of stdout. That exceeds the 32 MiB
   per-command `max_output_bytes` cap, `result.truncated` is set, and `read_blobs`
   returns `Err("git cat-file --batch exceeded its command budget")`.
4. `.unwrap_or_default()` turns that error into an empty `Vec`. Zipping `wanted` with it
   yields zero records. `measure` then reports `plans_committed: Some(0)`,
   `measured: true`, and **no issue**. `load_records` also persists that empty record
   list to the plans cache (`<cache_dir>/v1/plans/<key>.json`) for the current tip. The
   cache entry for today's tip already holds 0 records.

Reproduced through the binding with the real request and the cache disabled:

| `budget.max_output_bytes`     | plans_committed (all time)      | issues |
| ----------------------------- | ------------------------------- | ------ |
| 32 MiB (shipped default)      | 0                               | none   |
| 256 MiB                       | 5,542 (1,208 epic · 4,334 tale) | none   |
| 32 MiB, `plans_dir: "202610"` | 334                             | none   |

The bug only appears once the store's blob volume passes the cap. That is why small
stores and the existing `sidecar_root_counts_plans_without_a_directory_prefix` test
pass.

The same reader feeds the agents-sidecar directory walk (`project_stats/directory.rs`,
about 35.9k `meta.json`/`commits.json` blobs at about 470 B each, so about 17 MB in one
batch today). It is under the cap now, but it fails the same way once those pages
roughly double. On error it marks the walk `truncated`, so it degrades less silently,
but it would still lose all model attribution. Fixing the shared reader covers both
callers.

A related silent zero exists in the same function. When `build_index` itself fails,
`load_records` returns `(Vec::new(), [plans_unavailable issue], false)`, and `measure`
still reports `plans_committed: Some(0)` with `measured: true`.

## Changes (sase-core)

### 1. Size-aware batching in `project_stats::git::read_blobs`

Keep the signature
`read_blobs(repo, oids, budget) -> Result<Vec<Option<Vec<u8>>>, String>` and its
contract: results are in input order, and `None` means missing or unusable. Replace the
fixed per-OID guess with real sizes:

1. **Probe sizes** with `git cat-file --batch-check` (through
   `run_unchecked_with_stdin`, so it is recorded, timed out, and capped like every other
   project-stats git command). Chunk the probe so its stdout stays under the budget: use
   128 bytes per line as the upper bound for SHA-1 or SHA-256 headers, giving
   `(max_output_bytes / 128).clamp(1, 65_536)` OIDs per call. Parse
   `<oid> <type> <size>`. A `missing` line or a non-blob type maps to `None`.
2. **Pack batches by real size.** The cost of each blob is `size + header overhead` (the
   snapshot module already uses 96 bytes, which covers SHA-256 headers and the trailing
   newline). A batch's total cost must stay at or under `max_output_bytes`, and it holds
   at most 65,536 OIDs. Prefer one shared packer over a third copy: generalize
   `snapshot::blobs::pack_batches` into a parameterized helper (max output bytes, max
   OIDs per batch) that both readers call. Snapshot keeps its 256-OID limit and its
   oversize-is-an-error behavior, and its existing `pack_batches` unit tests keep
   passing. If sharing turns out awkward, a private packer in `git.rs` is acceptable,
   with a comment pointing at the snapshot one.
3. **Oversize blobs.** A blob whose own cost exceeds `max_output_bytes` is never
   requested. Its slot is `None`, and the remaining blobs still read. No realistic plan
   or agent page comes close, and a `None` slot is already skipped by both callers.
4. **Read each batch** with `cat-file --batch` and the existing `parse_cat_file_batch`,
   then scatter the results back to their input indices. A truncated or failed batch is
   still an `Err`. The size probe makes truncation a genuine anomaly rather than an
   expected outcome.
5. Replace the stale "1 KiB per oid covers typical agent page JSON" comment with one
   that states the size-probe and packing rule.

### 2. Never report a measured zero when plan blobs or history could not be read (`plans.rs`)

- `records_from_index` returns `Result<Vec<CachedPlanRecord>, String>` and propagates
  the `read_blobs` error. Remove `.unwrap_or_default()`.
- `load_records` distinguishes three outcomes: complete records (cacheable), truncated
  records (keep today's partial behavior plus the `plans_truncated` issue, not cached),
  and **failed**. A failure covers both a `build_index` error and a blob-read error. A
  failure is never cached.
- `measure` maps a failure to an unmeasured result through the existing
  `unavailable(...)` shape: `plans_committed/epic/tale = None`, `measured: false`,
  coverage `unavailable_repos: 1`, and one issue with code `plans_unavailable` whose
  message carries the underlying error (include the repo the way the current
  `plans_unavailable` issue does). Downstream, the card already renders
  `— plans committed` for `None`, and the issue appears on the report's `Issues:` line,
  so no sase-side change is needed.
- Bump `PLANS_CACHE_SCHEMA_VERSION` from `2` to `3`. This discards the empty record
  lists the broken reader already persisted, including the entry for the plans sidecar's
  current tip, so the first run after upgrading recounts instead of serving `0` until
  the next plan lands.

### 3. Agents-sidecar directory walk

No code change is expected in `directory.rs`. It calls the same `read_blobs` and gains
size-aware batching automatically. Confirm that its existing tests still pass.

## Tests (sase-core)

Match the neighbouring test modules (`plans_tests.rs`, `directory_tests.rs`,
`test_support.rs` fixtures, and `git::capture_git_commands`).

1. **Regression: the many-plans store exceeds one batch.** Build a sidecar-root store
   (`storage: SidecarRepos`, `plans_dir: "."`) with several plans, each padded so the
   plans fit the budget individually but not together. For example, six plans of about 4
   KiB each, with `request.budget.max_output_bytes` lowered to about 16 KiB and tuned so
   the git-log walk and commit facet still fit. Assert that `plans_committed == Some(6)`
   with the correct epic and tale split. Confirm the test fails on the old reader: with
   that budget the old code puts every OID in one chunk and returns 0.
2. **`read_blobs` unit coverage:** order is preserved across several packed batches. A
   missing OID is `None` and does not break framing. A blob larger than the whole budget
   is `None` while its neighbours still read. Assert through `capture_git_commands` that
   more than one `cat-file --batch` ran and that a `--batch-check` probe ran.
3. **Failure is unmeasured, not zero:** a blob-read failure (for example
   `records_from_index` against a path where `git cat-file` cannot run, or a small seam
   that maps the failure to a `PlanMeasurement`) yields `plans_committed == None`,
   `measured == false`, a `plans_unavailable` issue, and **no** plans cache file
   written. Cover the `build_index`-error branch the same way if a deterministic trigger
   exists. Otherwise, unit-test the shared failure-to-measurement mapping.
4. Existing tests stay green. In particular, check
   `sidecar_root_counts_plans_without_a_directory_prefix`, the snapshot `pack_batches`
   tests, the directory tests, and the `code_query_tests` assertions that
   `cat-file --batch` is _not_ invoked on code-only paths. The new `--batch-check` probe
   must run only when `read_blobs` is called with a non-empty OID list.

## Verification

1. In the sase-core checkout, run `sase tool run check` (the guarded `just check`, about
   5 minutes; give it a tool timeout of at least 15 minutes). This is the gate. Targeted
   `just test -p sase_core project_stats` runs are fine while iterating but never
   replace it.
2. **End to end, in the sase workspace's own `.venv`** (never the global install): run
   `just rust-install` from the sase workspace root. It builds `sase_core_rs` from the
   linked sase-core checkout into the workspace `.venv`, and may take several minutes.
   Use a long timeout, and go through `sase tool run` if the recipe is guarded. Then
   run:
   - `.venv/bin/sase stats project -p sase -r 90d` shows about 2,364 plans committed
     (759 epic · 1,605 tale), plus any plans archived since 2026-10-24, and nonzero
     PLANS cells in the By Model table.
   - `.venv/bin/sase stats project -p sase -r all -j` reports `plans_committed` about
     5,542 or more, and no `plans_unavailable` / `plans_truncated` issue.

   If the build is impossible in the agent environment, say so explicitly in the final
   report instead of claiming end-to-end verification.

## Non-goals and follow-ups

- **Shipping to the user's global `sase`:** the user's install is editable, so it picks
  up the fix after a human runs `sase update` once this lands on sase-core master. Wheel
  installs need the next `sase-core-rs` release and sase's exact `sase-core-rs==` pin
  ratchet. That is the normal release flow; do not edit versions, pins, or CHANGELOGs
  here.
- **The history-walk budget:** the full-history `git log --raw` over the plans sidecar
  is about 10.5 MB of the 32 MiB cap today, because `link-events/` JSON churn shares the
  walk. Exceeding the cap already reports `plans_truncated`, a partial result rather
  than a silent zero. If you judge it worth tracking, file a task bead for narrowing the
  walk or making it incremental (use `/sase_new_task`). Do not build it here.
- No wire, binding, or schema change. This is a `fix(project_stats): …` commit, not a
  breaking change.
