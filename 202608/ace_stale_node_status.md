---
tier: tale
title: Repair ACE agent-status latency by de-starving the artifact watcher
goal:
  An ACE agent node flips out of RUNNING within ~1s of its done marker landing, instead
  of waiting on the 60s sanity refresh, and the watcher can no longer be starved by junk
  date shards.
size: medium
proposed_by: bbugyi200.athena.0fl
create_time: 2026-08-28 12:44:50
status: wip
---

# Plan: Repair ACE agent-status latency by de-starving the artifact watcher

## Symptom

ACE Agents-tab node statuses lag reality by tens of seconds. A `sase` agent that has
already written its done marker keeps rendering as `RUNNING` — in the observed case an
agent-family shell (`sase-ud.13.1.4--5`) wrote `done.json` at `12:26:29`, while the ACE
agent list still rendered its family node `sase-ud.13.1.4` as `RUNNING` at roughly
`12:26:56`. Agents in _other_ projects (for example `gh_bobs-org__bob-cli`) updated
promptly in the same ACE session, which is the tell that this is per-project watcher
coverage rather than a global loader problem.

## Root cause (confirmed on the live machine)

ACE's Agents surface is event-driven. `EventAutoRefreshMixin._run_auto_refresh_body`
(`src/sase/ace/tui/actions/event_refresh/_auto_refresh.py`) only runs the agents loader
when `_dirty_agents` is set by the inotify watcher, or when the
`FULL_SANITY_REFRESH_SECONDS = 60` sanity window elapses
(`src/sase/ace/tui/actions/event_refresh/_constants.py`). So if no inotify event ever
fires for a project's live agent artifact directories, that project's node statuses
converge _only_ on the 60-second sanity pass — a uniform 0–60s lag, mean ~30s. That is
exactly the observed ~27s.

No inotify event fires because the watcher's **startup shard budget picks month shards
by lexicographic name order**:

- `ArtifactWatcher._iter_startup_watch_paths` in `src/sase/ace/tui/util/fs_watcher.py`
  watches each `<project>/artifacts` dir and its direct children (so `artifacts/ace-run`
  is watched), then delegates to `_iter_recent_ace_run_shard_watch_paths` for date
  shards.
- `_iter_recent_ace_run_shard_watch_paths` sorts month shards with
  `sorted(..., key=lambda path: path.name, reverse=True)` and keeps only
  `MAX_STARTUP_ACE_RUN_MONTH_WATCHES = 2`, then walks up to
  `MAX_STARTUP_ACE_RUN_DAY_WATCHES = 14` day shards from those months.
- `_is_month_shard` accepts _any_ six digits and `_is_day_shard` any `01`–`31`, with no
  clamp to a plausible date.

The `gh_sase-org__sase` project's `artifacts/ace-run/` currently holds **385 month
shards, 313 of which are future-dated** (`213601`, `213510`, `213509`, … down through
`200003`). They are completely empty (empty day dirs only) and were all created in a
~30-minute burst on 2026-07-24 22:27–22:58, i.e. almost certainly leaked into the real
`~/.sase` by a test or tool run. Because `213601` and `213510` sort highest, they win
both month watch slots, and the live month shard `202608` — plus day shard `28`, plus
every per-agent artifact directory beneath it — gets **no watch at all**.

`_add_watch_tree` cannot recover: it only installs watches for directories whose
`IN_CREATE | IN_ISDIR` event arrives on an _already watched_ parent. With `202608`
unwatched, the creation of `202608/28` and of each `202608/28/<timestamp>` agent
directory is never seen. Every `done.json`, `agent_meta.json`, and `waiting.json` write
in the sase project is therefore invisible to ACE.

Verified live against the running ACE process (`python3 -m sase ace`, started 10:03:52,
124 inotify watches installed, far below the `MAX_INOTIFY_WATCHES = 4096` cap — so this
is not the watch cap):

| path                                             | watched |
| ------------------------------------------------ | ------- |
| `.../gh_sase-org__sase/artifacts`                | yes     |
| `.../gh_sase-org__sase/artifacts/ace-run`        | yes     |
| `.../artifacts/ace-run/202608`                   | **no**  |
| `.../artifacts/ace-run/202608/28`                | **no**  |
| `.../artifacts/ace-run/202608/28/20260828121920` | **no**  |

Ruled out during diagnosis, so the implementer does not need to re-investigate:

- **Not the SQLite artifact index.** `write_done_marker_and_update_index`
  (`src/sase/axe/run_agent_exec_markers.py`) upserts the row synchronously; the row for
  the observed shell was reindexed 3 seconds after `done.json` landed, and a full sweep
  of `~/.sase/agent_artifact_index.sqlite` found **zero** rows claiming
  `has_done_marker = 0` while a `done.json` existed on disk.
- **Not the Tier 1 `freshness="cached"` index query** or the 300s
  `TIER1_INDEX_REVALIDATE_MIN_INTERVAL_S` revalidate: the index row was already correct,
  so a cached read would have returned the right status.
- **Not the family/container status derivation** in
  `src/sase/ace/tui/models/_agent_status_apply.py`: the container faithfully mirrors its
  active shell, and the shell row itself was the stale input.
- **Not the inotify watch cap** and not the kernel sysctl limits
  (`max_user_watches = 505837`).

There are two contributing defects that make the failure total rather than partial:

1. **The month-level refresh pulse is dead code.**
   `_touch_agent_artifacts_refresh_pulse` in `src/sase/shells/handoff.py` and the
   equivalent write in `src/sase/main/plan_propose_handler.py` write
   `Path(artifacts_dir).parents[1] / ".ace_refresh_pulse"`, i.e.
   `ace-run/<month>/.ace_refresh_pulse`. That path is usually unwatched, _and_
   `artifact_path_affects_agents` in
   `src/sase/ace/tui/actions/event_refresh/_artifact_paths.py` classifies it as
   irrelevant: relative parts `("ace-run", "<month>", ".ace_refresh_pulse")` have length
   3, so `_is_sharded_ace_run_dir_path` requires the third component to be a day shard
   and returns `False`. Only the project-level pulse written by
   `touch_shell_refresh_pulse` (`src/sase/shells/settlement.py`), which lands in the
   always-watched `<project>/artifacts/`, actually reaches `_dirty_agents`.
2. **Nothing pulses on agent completion.** `touch_shell_refresh_pulse` fires on shell,
   gate, and monitor settlement, but the agent done-marker write path does not, so the
   single most user-visible transition has no watcher-independent signal.

## Work

### 1. Make startup shard selection date-aware (`src/sase/ace/tui/util/fs_watcher.py`)

Rewrite `_iter_recent_ace_run_shard_watch_paths` so the watch budget is spent on shards
that can plausibly hold live agents:

- Clamp month shards to `<= ` the current month and day shards to `<=` today's date,
  using `sase.core.time.local_now()` (the repo's timezone-display guard test forbids
  naive `datetime.now()` in display/logic paths — follow the existing convention).
  Discard anything dated in the future.
- Keep the descending-by-value ordering for the surviving shards, and keep
  `MAX_STARTUP_ACE_RUN_MONTH_WATCHES` / `MAX_STARTUP_ACE_RUN_DAY_WATCHES` as the budget.
- Guarantee that when the current month shard exists it is always yielded, and when
  today's day shard exists it is always yielded, regardless of how many other shards
  compete for the budget. Spend the day budget newest-first across the selected months
  rather than letting a single month silently consume all of it.
- Keep the function lazy and OSError-tolerant exactly as it is today; a missing or
  unreadable shard must not abort startup.

Also tighten `_add_watch_tree` so recursive installation **stops at the per-agent
artifact directory boundary** (a 14-digit timestamp directory). Every loader-visible
marker lives at the top level of that directory, and deep paths such as
`commit_diffs/001.diff` are already discarded by `artifact_path_affects_agents`, so
watching `commit_diffs/`, `finalizers/`, and `markdown_pdfs/` is pure waste. This
matters because fix (1) turns the live tree from "never watched" into "always watched",
so watch growth becomes real for the first time; cutting per-agent watches to one keeps
a multi-day ACE session well clear of `MAX_INOTIFY_WATCHES`, whose only current
behaviour on exhaustion is to silently stop installing watches (which reproduces this
same bug).

### 2. Accept the refresh pulse at any depth (`.../event_refresh/_artifact_paths.py`)

Teach `artifact_path_affects_agents` to return `True` for a file named
`.ace_refresh_pulse` anywhere under an `artifacts` tree, so the month-level pulse from
`src/sase/shells/handoff.py` and `src/sase/main/plan_propose_handler.py` stops being
silently dropped. Leave `artifact_dir_from_known_marker_path` alone — a pulse is a broad
nudge, not a specific artifact dir, so it must fall through to the broad Tier 1 load
rather than being mistaken for an exact artifact-delta target.

### 3. Add a watcher-independent completion signal

Have the agent done-marker write path pulse the always-watched project-level path.
Concretely, after `write_done_marker_and_update_index`
(`src/sase/axe/run_agent_exec_markers.py`) writes `done.json` and upserts the index,
call `touch_shell_refresh_pulse` with the project derived from the artifacts dir
(`project_name_from_artifacts_dir` in `src/sase/shells/settlement.py` already does that
derivation). Keep it best-effort — a failing pulse must never fail the agent run — and
watch for an import cycle between `sase.axe` and `sase.shells`; use a function-local
import if needed, matching how `write_done_marker` already imports the index lifecycle
helper lazily.

This is the belt to fix (1)'s braces: it makes the completion transition converge
promptly even when deep-tree watch coverage is unavailable (watch cap reached, non-Linux
host, NFS/bind-mount, or a shard layout nobody anticipated).

### 4. Surface the failure in `sase doctor`

Extend the `resources.inotify` check (`src/sase/doctor/checks_resources_inotify.py`,
registered in `src/sase/doctor/checks_resources.py`) — or add a sibling check next to it
if that keeps the check focused — so it reports, per enabled project, whether the shard
that would receive today's agent artifacts is inside the startup watch budget. Reuse the
selection helper from work item 1 rather than duplicating its ranking logic, so the
check cannot drift from the watcher.

Emit `WARN` when a project's live shard would be starved, and include in `next_steps`
the count of future-dated month shards plus the exact command the user can run to remove
the empty ones. **Do not delete anything automatically** — these are directories in the
user's real `~/.sase`, so removal stays an explicit user action.

### 5. Tests

- `tests/ace/tui/test_fs_watcher.py`: build a fake `ace-run` tree containing the live
  month/day shard plus future-dated junk months that sort higher, and assert the startup
  paths include the live month and today's day shard and exclude the future-dated ones.
  Cover the "no junk" case too, so ordinary projects keep their existing behaviour.
- `tests/ace/tui/test_fs_watcher.py`: assert `_add_watch_tree` does not descend below a
  14-digit agent artifact directory.
- A test for `artifact_path_affects_agents` on
  `artifacts/ace-run/<month>/.ace_refresh_pulse` and on `artifacts/.ace_refresh_pulse`
  (there is no existing test module for `_artifact_paths.py`; add one under
  `tests/ace/tui/`).
- A test that writing a done marker touches the project-level
  `<project>/artifacts/.ace_refresh_pulse`.
  `tests/test_run_agent_artifact_index_updates.py` and
  `tests/test_agent_artifact_marker_mutation_audit.py` already exercise the done-marker
  write path and are the natural neighbours.
- `tests/doctor/test_checks_resources.py`: a starved-project fixture yields `WARN` with
  the junk-shard count, and a healthy project yields `OK`.

Use `tmp_path` / the repo's existing SASE-home fixtures for every filesystem test. Never
let a test touch the real `~/.sase` — a test that did exactly that is the likely origin
of the 313 junk shards, and repeating it would recreate the bug.

## Out of scope

- Do not change `FULL_SANITY_REFRESH_SECONDS`, `AGENTS_LOAD_MIN_INTERVAL_SECONDS`, or
  `TIER1_INDEX_REVALIDATE_MIN_INTERVAL_S`. The sanity pass is the safety net that masked
  this bug; it is not the bug.
- Do not touch the SQLite artifact index, the Tier 1/Tier 2 loader tiering, or the
  family status derivation. All three were checked and are correct.
- Identifying which tool created the 313 future-dated shards on 2026-07-24 is a separate
  investigation. If it is not obvious in passing, file a task bead through
  `/sase_new_task` rather than expanding this plan.

## Verification

1. Unit tests above, then `just check` (this touches `src/sase/ace/`, `src/sase/axe/`,
   `src/sase/shells/`, and `src/sase/doctor/`, so run `just check-full` through
   `/sase_monitor` before landing).
2. End-to-end on the live machine, since the junk shards make it directly reproducible:
   - Before: confirm the live shard is unwatched. Find the ACE pid, then for its inotify
     fd compare the `ino:` values in `/proc/<pid>/fdinfo/<fd>` against `os.stat()` of
     `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/<YYYYMM>` and
     `.../<YYYYMM>/<DD>`.
   - Restart ACE with the fix and repeat: both shards, and each live per-agent
     directory, must now be watched.
   - Launch a short agent and confirm its node leaves `RUNNING` within about a second of
     its `done.json` mtime rather than up to 60 seconds later.
3. `sase doctor` reports the starved-project warning before the junk shards are removed
   and `OK` after.
