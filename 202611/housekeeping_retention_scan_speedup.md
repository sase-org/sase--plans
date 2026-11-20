---
tier: tale
title: Stop the hourly artifact-retention housekeeping jobs from timing out
goal:
  artifact_retention and artifact_run_prune finish well under their 300s timeout under
  heavy host load, with identical protection sets.
size: medium
decisions:
  recent_window_short_circuit:
    ask:
      Should artifact_run_prune skip per-run scans for keep-recent-window runs, which
      can never be selected?
    choices:
      skip:
        About 8s faster per tick; JSON output lists only Rust-added reasons for recent
        runs
      keep:
        Recent runs keep full reason lists; job stays about 8s slower scanning 11.6k
        unselectable runs
    default: skip
    why:
      Selection, counts, and quarantine are unchanged; no review surface shows
      recent-run reasons
    answer: skip
proposed_by: bbugyi200.athena.1eq
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 18:14:17
status: wip
---

# Stop the hourly artifact-retention housekeeping jobs from timing out

## Problem

The axe error digest `digest_20261120_175619_e5de8115` reports three jobs killed at
their 300s timeout around 17:29. Two of them are a recurring pattern, not a one-off:

- `housekeeping/artifact_retention` and `housekeeping/artifact_run_prune` both run
  hourly, start at the same moment, and have hit the 300s timeout about ten times since
  2026-11-19 (for example 2026-11-19 10:18, 12:22, 13:51, 14:25, 19:36 and 2026-11-20
  10:20, 15:05, 17:13, 17:29). Their successful runs on 2026-11-20 took 37s to 265s
  while doing nothing (`trashed 0 rows`, `selected=0`). The spread follows host load:
  athena ran at load average around 40 with 10-23% iowait from 15:00 to 17:40.
- `publication/agent_publication` timed out once (17:24:32 to 17:29:33, during the same
  load window, right after a routine restart). It printed nothing, and its run log was
  pruned within minutes by the 10-run history cap, so it cannot be root-caused from the
  evidence available.

### Measured cause (athena, load about 13, warm cache, installed sase)

Both jobs are single-threaded, CPU-bound Python scans that read the same corpus every
hour. On a quiet host they already take 35-65s, so a 4-8x slowdown under load crosses
300s.

`artifact_retention`: `collect_protected_artifact_ids()` in
`src/sase/core/artifact_file_protection.py` takes about 35s.

- About 20s is `_ARTIFACT_ID_RE.finditer` over about 70k sidecar text files (about 522
  MiB). The leading optional `(?:file:)?` group stops Python's regex engine from using a
  literal-prefix search. A benchmark on the real corpus returned identical ids (1,145)
  in 20.2s with the current regex, 4.6s with `(?:default|explicit):[0-9a-f]{24}`, and
  1.6s with a `"default:" in text or "explicit:" in text` prefilter plus the simpler
  regex. Only about 5.9k of the 70k files contain a candidate.
- About 9.5s is the `.sase` walk of the whole `~/.sase/projects` tree (1.2M files, 208k
  dirs) to find about 20 top-level `<project>/<project>.sase` spec files. That walk
  found 0 ids. The run-prune twin already prunes `.git`, `artifacts`, and `repos` for
  the same walk, which leaves about 21k files.
- The remaining time is per-entry `pathlib.Path` construction (`Path(directory) / name`,
  `.suffix`, `.is_symlink()` on every child dir; about 3M `Path.__init__` calls).

`artifact_run_prune`: about 16s of protections plus about 27-47s of planning.

- `collect_ace_run_retention_protections()` in
  `src/sase/core/agent_artifact_run_protection.py` spends about 8.7s in
  `_AGENT_REF_RE.finditer` (the leading lookbehind defeats the literal prefix). A
  benchmark showed an `"agent:" in text` prefilter returns the same 16,905 matches in
  1.05s.
- `_collect_candidates()` in `src/sase/core/agent_artifact_run_retention.py` takes about
  27s: `_tree_size` on all 24,111 run dirs (about 1M `stat` calls, 7-9s, but sizes are
  only used for `selected` items, and there were 0), `_scan_records` (about 10.8s),
  continuation planning (about 5.5s), and repeated `_normalized_path` → `Path.resolve()`
  calls (about 97k calls, about 5s).
- About 11.6k of the 24.1k runs sit in the 2-month keep-recent window (`202610`,
  `202611`). The Rust owner (`classify_run_candidate` in sase-core
  `crates/sase_core/src/agent_artifact_run_retention.rs`) adds `recent_month` itself, so
  those runs can never be selected whatever reasons Python sends.

No existing bead covers this. `sase-1tz.4` deals with log and state retention horizons,
not scan cost.

## Goal

Make both housekeeping jobs finish in well under a minute on a quiet host, with enough
headroom to stay under 300s at load average 40+. Keep every protection set exactly the
same: retention protection guards against deleting user data, so no change may drop an
id, dir, agent name, or timestamp that the current code protects. Also make the next
`agent_publication` timeout diagnosable.

Non-goals: porting the scanners to Rust (do it only if the targets below are missed),
merging the two jobs' corpus walks into one shared pass, raising job timeouts, and
changing retention policy or what gets selected.

## Changes

### 1. Artifact-file protection scan (`src/sase/core/artifact_file_protection.py`)

1. **Project-spec walk.** When `collect_protected_artifact_ids()` walks
   `sase_projects_dir()` for `.sase` files, prune `.git`, `artifacts`, and `repos`
   directories, matching `agent_artifact_run_protection.py`. Put the skip set in one
   place both modules import (a small shared constant or helper next to the existing
   `_sidecar_protection_label` helpers), so the two walks cannot drift. Sidecar roots
   keep their current coverage: change only the projects-tree walk.
2. **Id extraction.** Add one helper, for example
   `_extract_artifact_ids(text) -> Iterable[str]`, that returns early unless
   `"default:" in text or "explicit:" in text`, then runs
   `(?:default|explicit):[0-9a-f]{24}` (`findall`). Use it in `_scan_root`,
   `_collect_linked_ids`, and the `issues.jsonl` fallback in `_collect_bead_store_ids`.
   Leave `_CONSUMED_FILE_REF_RE`, a `fullmatch`, unchanged. The optional `file:` prefix
   is non-capturing and never changes the captured id, so the id set is identical.
3. **Walk overhead.** In `_scan_root`, test the suffix with string ops
   (`os.path.splitext(filename)[1].lower()`, plus the `extra_filenames` check) before
   building any path. Drop the per-child-dir `is_symlink()` filter:
   `os.walk(..., followlinks=False)` already refuses to descend into symlinked dirs.
   Keep the per-file symlink skip for files that pass the suffix filter, and keep the
   `.git` prune, the `onerror` collection, and the required/unavailable semantics
   exactly as they are.

### 2. ACE-run protection scan (`src/sase/core/agent_artifact_run_protection.py`)

1. In `_collect_text_refs`, gate `_AGENT_REF_RE.finditer` on `"agent:" in text` and gate
   `_ACE_RUN_REF_RE.finditer` on `"artifacts/ace-run/" in text`. The regexes themselves
   do not change, so the lookbehind and trailing-punctuation behavior stays the same.
2. Give `_scan_text_root` the same string-op suffix test and the same removal of the
   redundant child-dir `is_symlink()` filter as change 1.3. Keep the per-file symlink
   skip, the size-limit check, and the unavailable reporting.

### 3. ACE-run retention planning (`src/sase/core/agent_artifact_run_retention.py`)

1. **Size only selected runs.** Stop calling `_tree_size` in `_collect_candidates`.
   After `_call_rust` returns, compute `_tree_size` only for `run_items` whose outcome
   is `selected` (in `_plan_from_result` or a helper it calls) and feed `size_bytes` and
   `reclaimable_bytes` exactly as now. Drop the `size_by_dir` return value from
   `_run_owner` and `_collect_candidates`, and update callers. `apply_ace_run_retention`
   does not use it. Check the `disk_footprint*` callers of `plan_ace_run_retention`
   still get the same selected sizes.
2. **Normalize once.** In `_collect_candidates`, compute `normalized_dir` once per dir
   and pass it into `_protection_reasons` (add a parameter instead of recomputing).
   Compute the normalized `SASE_ARTIFACTS_DIR` once per call, not once per candidate.
   `protection_reasons_for_dirs` must keep producing identical reasons.
3. **Skip runs in the recent window** (decision `recent_window_short_circuit`).

   > [!decision] recent_window_short_circuit = skip Split `all_dirs` by whether
   > `info.timestamp[:6]` is in
   > `_recent_month_names(policy.now, policy.keep_recent_months)`. Pass recent-window
   > dirs to Rust as candidates with `protected_reasons: []`, and leave them out of
   > `plan_continuation_run_retention` and `_scan_records`. Only out-of-window dirs get
   > the record scan, continuation planning, and `_protection_reasons`. This requires
   > threading `now` and `keep_recent_months` into `_collect_candidates`. Rust still
   > reports them as `protected` with `recent_month`, so `counts.candidates`,
   > `counts.protected`, `selected`, and the trash quarantine (which only re-verifies
   > reviewed, out-of-window dirs through `protection_reasons_for_dirs`) do not change.

   > [!decision] recent_window_short_circuit = keep Skip this step. Changes 3.1 and 3.2
   > still apply.

### 4. Phase progress in the two housekeeping chops

When a job is killed for timing out, its final summary never prints, which is why both
digests say `[output absent]` or show only the first line. Emit one line per completed
phase with elapsed seconds, so the subprocess excerpt in the next digest shows where the
time went:

- `artifact_retention`: add an optional `progress: Callable[[str], None] | None = None`
  parameter to `enforce_artifact_retention` and `_run_locked_pass` in
  `src/sase/core/artifact_file_retention_pass.py`, called after protections, plan,
  trash, and purge (for example `artifact_retention phase protections: 4.2s ids=1931`).
  `src/sase/scripts/sase_chop_artifact_retention.py` passes `runtime.log`. The
  `maybe_enforce_artifact_retention` finalization path keeps passing nothing and stays
  silent.
- `artifact_run_prune`: in `src/sase/scripts/sase_chop_artifact_run_prune.py`, log one
  elapsed-time line after protections, trash purge, plan, and publish.

Follow the existing `gate turn reclaim progress:` pattern in
`src/sase/scripts/sase_chop_gate_turn_reclaim.py`. Do not change the summary key=value
grammar or structured result fields.

### 5. Make the `agent_publication` timeout diagnosable

The pass already threads its 240s `deadline` into the drains, so a 300s kill means some
single step overran without evidence. Do not guess a fix. Add a progress line that only
appears on slow passes, so normal 15s ticks stay quiet in `lumberjack-publication.log`:

- Give `run_publication_job` (in `src/sase/agents_sync/_publication_job_run.py`) an
  optional `progress: Callable[[str], None] | None = None`. When a pass has already used
  more than 60s (a module constant), emit a line before each later step: project
  discovery, `resolve_sync_targets`, each `drain_project(<key>)`, the standing-alert
  apply, and the heartbeat write. Each line carries the elapsed seconds and the project
  key.
- `src/sase/scripts/sase_chop_agent_publication.py` passes `runtime.log.info`.

## Tests

Add or extend tests next to the existing suites:

- `tests/test_artifact_protection_scan.py`:
  - The projects-tree walk still finds ids in `<project>/<project>.sase` and ignores
    `.sase` files under `artifacts/`, `repos/`, and `.git/`.
  - Symlinked dirs are not descended.
  - Extraction returns the same ids as the old regex for `file:default:<24hex>`,
    `explicit:<24hex>`, ids next to punctuation, uppercase-hex non-matches, and text
    with no candidate tokens.
- `tests/core/test_agent_artifact_run_protection_unconnected.py` (or a sibling): the
  `agent:` and `artifacts/ace-run/` prefilters preserve current matches, including the
  lookbehind rejecting `xagent:foo` and trailing punctuation being stripped.
- Run-retention planning (extend the suites that already call `plan_ace_run_retention`,
  such as `tests/main/test_artifact_cli_prune_runs.py` or `tests/core/`):
  - With a monkeypatched counting `_tree_size`, only selected dirs are sized, and
    `reclaimable_bytes` and per-item `size_bytes` are unchanged.
  - Under `skip`: `scan_agent_artifact_dirs` and continuation planning never receive
    recent-window dirs, and those dirs are still counted and reported `protected` with
    `recent_month`.
  - `counts` match the pre-change values for a fixture mixing recent and old runs.
- `tests/test_axe_chop_artifact_retention.py` and
  `tests/test_axe_chop_output_contract_retention_prune*.py`: phase lines appear and the
  summary/result contract is unchanged.
- `tests/agents_sync/`: with a fake clock past the threshold, progress lines are emitted
  per project, and none are emitted on a fast pass.

## Verification

1. **Protection parity on real data (required, read-only).** Before editing, use the
   workspace `.venv` Python to dump to a JSON file outside the repo:
   `sorted(collect_protected_artifact_ids().ids)`; the four sets from
   `collect_ace_run_retention_protections()` (`protected_dirs`, `protected_agent_names`,
   `protected_timestamps`, `non_closed_bead_ids_by_project`); and
   `plan_ace_run_retention(...)` `counts`, the selected dirs, and `reclaimable_bytes`.
   Repeat after the change and assert the protection sets and selection are identical.
   Counts must match too, apart from runs or files created in between. These functions
   only read; do not call trash, purge, or `publish_run_prune_preview` for this.
2. **Timing.** Record wall time for the same three calls before and after, on the same
   host. Targets on a quiet athena: `collect_protected_artifact_ids` ≤ 8s (from about
   35s), `collect_ace_run_retention_protections` ≤ 8s (from about 16s), and
   `plan_ace_run_retention` ≤ 15s (from about 27-47s). Record the before/after numbers
   in the final summary. If a target is missed by a wide margin, report the profile
   instead of adding further rewrites, and file a follow-up task bead for a Rust-side
   scanner.
3. Run `just check` per the lint-and-test memory note.

## Done when

- The protection sets and selection are identical on real data before and after.
- The timing targets are met, or the miss is reported with a profile.
- Both housekeeping chops log phase progress, and `agent_publication` logs progress once
  a pass passes 60s.
- `just check` passes.
