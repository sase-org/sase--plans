---
tier: tale
title: Stop paying a full protection-ledger read for every published file
goal:
  Apollo's bob-cli research-hood publication drains fit well inside the 120s bound
  because protection-ledger lookups no longer re-read and re-validate the ledger once
  per published file.
size: medium
decisions:
  quiet_retryable_timeouts:
    ask:
      Should a drain timeout that leaves its requests retryable stop marking the
      publication tick an error?
    choices:
      keep:
        Every drain timeout still marks the tick check_error and lands in the error
        digest
      quiet:
        Retryable hood timeouts log a warning; only ones that quarantine or retire a
        request are errors
    default: keep
    why:
      After the fix a timeout signals a real regression and still spends one of three
      attempts
    answer: keep
proposed_by: bbugyi200.athena.1bb
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 10:29:06
status: wip
---

# Stop paying a full protection-ledger read for every published file

## Problem

The 2026-11-11 10:14 error digest on apollo reports two `agent_publication` errors
(09:53 and 09:56 EST) for `gh_bobs-org__bob-cli`:
`agent-hood publication did not complete within 120s; publication cleanup also failed: publication cleanup did not complete within 1s`.

Evidence gathered while planning:

- **Drain log.** On apollo, `~/.sase/logs/agent_publication_drains.jsonl` shows both
  alarms fired while draining bob-cli's `research` hood, which has 937 runs.
  - At 14:51Z, `inventory_build` took 16.9s and `hood_publish:research` took 75.3s. The
    alarm fired during `commit`.
  - At 14:56Z, the push lost a race with another owner. `retry_prepare` re-planned for
    36.8s, and the alarm fired during `retry_push`.
  - The next tick (14:57Z) published all eight requests, and no row was quarantined.
  - Normal single-request drains of this hood on apollo take 35–100s today. Athena's
    bob-cli drains take about 15s.
- **Profile.** A read-only
  `tools/agents_publication_profile --project gh_bobs-org__bob-cli --hood research` run
  on apollo (16 cores, load 8–11) breaks one targeted publish down as follows:
  - Inventory takes 14.8s. Planning the hood takes 26.3s for 6,425 payload files (24.8
    MB).
  - Under cProfile, `build_hood_snapshot` took 16.7s. Of that, 12.9s was
    `lookup_publication_content_record`, called 5,386 times (about 2.4 ms per call).
  - Every call takes the ledger flock and re-reads and json-parses
    `agents-publication-protection.json`. It validates the ledger through Rust
    (`publication_protection_ledger_validate`, 4.4s in total). It then calls Rust
    `publication_protection_lookup_enforced` (4.0s in total), which converts and
    validates the whole ledger again.
  - The ledger holds only 16 records. The cost comes from per-call overhead multiplied
    by the file count, not from the ledger's size.
- **Why now.** The lookup has existed since Oct 27, but it returns early when the
  project has no ledger file. bob-cli's apollo ledger was written later, so every
  research-hood publish now pays the cost. The `sase-1qh.3` scan-reuse measurement
  (about 11s of hood time on athena) predates that.
- **Secondary cost.** `load_validated_publication` decodes all ~1,076 owner snapshots
  (9.3s under the profiler). About 4.3s of that is 237k `validate_component` calls,
  mostly on repeated values.
- **Risk.** Each timeout charges one attempt, and requests quarantine after three
  (`DEFAULT_PUBLICATION_MAX_ATTEMPTS = 3`). Seventeen `research.28.*`/`research.29.*`
  rows on apollo were quarantined by this same 120s alarm on Oct 27.

The 120s budget is fine. Most of the avoidable work sits in one function that every
publisher path shares: snapshot gating (two call sites), payload staging, state repair,
and prompt-archive protection.

## Out of scope

- The ~640 older bob-cli quarantined rows (rebase conflicts and deferred prompt
  archives) are owned by `sase-1nf.11` and `sase-1qh.7`.
- Re-planning after a push race.
- Incremental hood rendering and snapshot caching across drains.
- Changing `SASE_AGENTS_PUBLICATION_DRAIN_TIMEOUT` or the one-second cleanup budget.

## Changes

### 1. Memoize the validated ledger and skip FFI for absent identities

File: `src/sase/agents_sync/_publication_protection_ledger.py`

- Add a small process-local cache keyed by ledger path. Each entry holds:
  - a stat signature `(st_dev, st_ino, st_size, st_mtime_ns)`,
  - the validated ledger dict returned by `_read_ledger`,
  - a `frozenset` of exact `(repository_identity, relative_destination, sha256)` triples
    taken from the validated ledger's `records`.

  Confirm the record field names against an existing ledger fixture in
  `tests/agents_sync/test_publication_protection/`. Give the cache a cache-like private
  name and a `_clear_publication_protection_ledger_cache()` helper. Run the opt-in
  global-state leak detector's focused tests (`tests/test_global_state_leak_detector.py`
  and `tests/_global_state_leaks/`) to confirm the new global is classified as a cache.
  Register it with whatever reset hook those tests expect if it is not.

- Update `lookup_publication_content_record`:
  - Keep every existing early return unchanged. An invalid project key, a symlink, or a
    missing file still returns `None`, and the lookup still never creates directories.
  - Stat the ledger. When the signature matches the cached entry, reuse it. On a miss,
    read under `_ledger_lock` through `_read_ledger` and store the new entry.
  - A `RuntimeError` (unreadable or invalid ledger) still returns `None` and is **not**
    cached, so a later valid file is read.
  - When the requested triple is not in the cached set, return `None` without calling
    Rust. This is exactly equivalent to today's result. `publication_protection_lookup`
    in the linked sase-core repo
    (`crates/sase_core/src/agent_publication_protection.rs`) returns the first record
    whose repository identity, relative destination, and sha256 all equal the query.
    `publication_protection_lookup_enforced` only filters that one result.
  - When the triple is present, call `publication_protection_lookup_enforced` with the
    cached validated ledger, as today. Stale-rule enforcement stays in Rust; no policy
    moves into Python.
- After writing, `_upsert_record` and `_resolve_verified_clean` (including its `unlink`
  branch) drop the cache entry for their path. The stat signature already catches
  replacements made by other processes. Confirm that `atomic_write_json` replaces the
  file through rename, so each write gets a new inode. The explicit drop covers
  in-process writes on coarse-mtime filesystems.
- Leave `publication_content_retry_allowed` and
  `read_publication_protection_ledger_from_path` unchanged. They are cold paths (retry
  checks and doctor).
- This is a Python-side read cache in front of the Rust API. No sase-core change is
  needed, and the `sase-core-revision.txt` pin does not move.

### 2. Memoize successful path-component validation

File: `src/sase/agents_sync/v2_validation.py`

`validate_component` is pure. Keep a bounded memo of values that already passed (for
example, an `lru_cache`-backed private predicate on the string, or a capped set). A hit
returns at once. Misses and failures follow today's code path, so error messages still
carry the caller's `label`. Do not cache failures.

## Tests

Add `tests/agents_sync/test_publication_protection/test_ledger_lookup_cache.py`, reusing
that package's `_helpers.py` and the fixture style of `test_ledger_rescan.py`:

1. Many lookups against an unchanged ledger call `_read_ledger` once (use a spy). They
   never call `publication_protection_lookup_enforced` for an absent triple (monkeypatch
   it to fail if called).
2. A present, enforced `blocked` record is returned. A stale scanner-hit block is still
   hidden. Both results must match the uncached Rust path called directly on the same
   ledger.
3. After a cached read, `record_publication_bytes_scan` adds a record and the next
   lookup returns it. A clean resolve that removes the last record (the unlink branch)
   makes the next lookup return `None`.
4. Another writer replaces the ledger through write-temp-then-`os.replace`. The next
   lookup sees the new contents.
5. An invalid ledger returns `None`. Replacing it with a valid ledger and looking up
   again returns the record, which proves failures are not cached.
6. A missing ledger returns `None` and creates no directories.

In `tests/agents_sync/test_publication_snapshot_reuse.py`, or next to it, add a
regression guard. Build a hood snapshot for a fixture hood with several runs while a
ledger file exists, and assert that `_read_ledger` runs at most once during the build.

For `validate_component`, add a focused test: an invalid value still raises with the
caller's label after valid values have been memoized, and a value that previously failed
still fails.

## Verification

1. Run `sase tool run check`, after `just fix` or `just fmt`.
2. Measure before and after on athena, read-only, with the workspace venv:
   `.venv/bin/python tools/agents_publication_profile --project gh_sase-org__sase --hood research --json`.
   Athena's sase project has a protection ledger, so the hotspot applies there.
   - Also run it once under `python -m cProfile`. Confirm that
     `lookup_publication_content_record` drops from seconds to well under a second, and
     that `_read_ledger` runs O(1) times.
   - Record the before and after `hood_timings.*.seconds` in the commit message.
   - If the profile cannot resolve that project or hood on this host, record why, and
     rely on the regression guard.
3. Do not update apollo's checkout or reinstall anything (global installs are
   human-only). Include this post-deploy command for the human in the final summary:
   `tools/agents_publication_profile --project gh_bobs-org__bob-cli --hood research` on
   apollo. Expect hood planning to fall from about 26s toward about 13s. Then watch
   apollo's drain log for `hood_publish:research`.

> [!decision] quiet_retryable_timeouts = quiet
>
> This branch adds the following work after changes 1 and 2.
>
> Make a hood-scoped drain timeout that leaves its requests retryable a warning instead
> of an error:
>
> - In `src/sase/agents_sync/publication_job.py` `_drain_project`, inspect the outcome
>   of `drain_selected_agent_publications`. It qualifies as retryable only when all of
>   the following hold:
>   - `failure_scope == HOOD_FAILURE_SCOPE`,
>   - `classify_publication_error(outcome.error)` returns `timeout`,
>   - every undrained key in the batch is still returned by
>     `active_agent_publications(project_key)` and is neither quarantined nor terminal.
>     Confirm which states that function filters.
> - A retryable timeout records its diagnostic in a new `_ProjectPass.retrying` list
>   instead of `result.error`. `_PublicationJobSummary` gains a `retrying` count and its
>   diagnostics.
> - Any timeout that quarantined or retired a key, every repository-scope timeout, and
>   every other error stays an error.
> - The drain-log `outcome` stays `timeout`, so the measurement trail does not change.
> - In `src/sase/scripts/sase_chop_agent_publication.py`, log retrying diagnostics with
>   `runtime.log.warning("[agent_publication] retrying: ...")` and add `retrying` to
>   `emit_summary`. A tick with only retrying entries does not set `check_error`.
> - In `tests/agents_sync/test_publication_job.py` and its `_publication_job_*.py`
>   helpers, add tests for three cases:
>   - A hood timeout that leaves its rows active gives errors 0, retrying 1, and no
>     `check_error`.
>   - A hood timeout that pushes a row to quarantine counts as an error.
>   - A repository-scope timeout counts as an error.
> - Read `docs_writing.md` through `sase memory read` first. Then add one sentence to
>   the `publication` section of `docs/reference/automation/scheduler-housekeeping.md`:
>   a retryable hood timeout is logged and retried on the next tick without counting as
>   an error.
