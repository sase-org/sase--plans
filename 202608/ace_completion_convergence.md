---
tier: tale
title: Converge ACE agent rows with the completion notification that announces them
goal:
  An ACE Agents-tab row (and its family container) leaves the Running bucket and shows
  as done and unread within one auto-refresh tick of the agent's completion
  notification, including for agents that were already running when the watcher started,
  without adding broad Tier 1 loads or growing the inotify watch set.
size: medium
proposed_by: bbugyi200.athena.0fp
---

- **AGENTS:**
  - [bbugyi200.athena.research.1c.cdx](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1c.cdx/README.md)
  - [bbugyi200.athena.research.1c.cld](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1c.cld/README.md)
  - [bbugyi200.athena.research.1c.final](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1c.final/README.md)
  - [bbugyi200.athena.research.1c.image](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.research.1c.image/README.md)
- **COMMITS:**
  - [50167df](https://github.com/sase-org/sase--research/commit/50167dfcff1077023b1f0601be02c09caaf9018f)
    — docs(research): assess memory as artifacts
  - [a82e179](https://github.com/sase-org/sase--research/commit/a82e1798dce1e3c81b47eb4862933ed71500cfe7)
    — docs(research): assess migrating sase memory into artifacts
  - [59fe226](https://github.com/sase-org/sase--research/commit/59fe22602f15de519d99df9a53265f8da87e9fab)
    — docs(research): consolidate memory-as-artifacts research into
    memory_as_source_artifacts
  - [25cd282](https://github.com/sase-org/sase--research/commit/25cd28279b58aa849e3adc74a31ea675d7e1ac93)
    — feat(research): add memory-as-source-artifacts infographic

# Plan: Converge ACE agent rows with the completion notification that announces them

## Symptom

A `sase` agent finishes, the user gets the completion notification (toast + bell +
notification-inbox row), and the ACE Agents tab keeps rendering that agent — or its
family container — as still running for tens of seconds afterwards.

Observed instance (`bbugyi200.athena.0fn`, `gh_sase-org__sase`):

| when           | what                                                                                                                                                                                          |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `14:25:04.057` | `0fn--code` writes `workflow_state.json` with `status: completed`                                                                                                                             |
| `14:25:10.213` | `0fn--code` writes `done.json` (`outcome: completed`)                                                                                                                                         |
| `14:25:13`     | SQLite artifact index reindexes that row as `completed`                                                                                                                                       |
| `14:25:10.276` | completion notification appended (`@0fn completed`, `raw_suffix: 20260828140403`)                                                                                                             |
| `14:26:44`     | screenshot `20260828_142644.png`: Agents tab still shows family root `0fn` as `WORKING TALE`, 🏃, in the **Running** bucket — **94 s** after the done marker, and past the 60 s sanity window |

The contract the user wants: the completion notification and the row flipping to
done/unread happen at roughly the same time (one auto-refresh tick of slack is fine).

## What is NOT the cause

Each of these was checked against the live machine during diagnosis. Do not re-open
them.

- **Not the status derivation.** Running `load_artifact_delta_agents()` over the three
  `0fn` family artifact dirs (root `20260828135111`, gate `20260828140149`, coder
  `20260828140403`) yields `0fn--code = TALE DONE` and family root `0fn = TALE DONE`.
  `apply_status_overrides` in `src/sase/ace/tui/models/_agent_status_apply.py` is
  correct; the container faithfully mirrors its newest child.
- **Not the SQLite artifact index.** The `20260828140403` row was
  `status='completed', has_done_marker=1, done_outcome='completed'` with `indexed_at`
  three seconds after the `done.json` write.
- **Not the Tier 1 / Tier 2 loader tiering** or the `freshness="cached"` index query.
  The index row was already correct, so a cached read returns the right status.
- **Not the shard-selection starvation fixed in `dde1b22d8`.** That fix is working: the
  live ACE process currently watches
  `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608` and `.../202608/15` …
  `.../202608/28`, and none of the 313 future-dated junk month shards.

## Root cause

Two independent gaps, both of which have to be closed for the transition to converge
promptly. Either one alone leaves the 60-second `FULL_SANITY_REFRESH_SECONDS` pass as
the only thing that clears the stale row.

### Gap 1 — the watcher is blind to every agent that was already running when it started

`ArtifactWatcher.start()` installs watches from `_iter_startup_watch_paths()`
(`src/sase/ace/tui/util/fs_watcher.py`). That generator yields `<project>/artifacts`,
its direct children, and (for `ace-run`) bounded month/day shards. **It never yields a
per-agent 14-digit artifact directory.** The only way such a directory ever gets watched
is `_add_watch_tree`, which runs from `_collect_relevant_events` on an
`IN_CREATE | IN_ISDIR` event — i.e. only for agent directories created _after_ the
watcher is already running.

inotify on a directory reports events for its direct children only. A watch on
`.../202608/28` therefore sees the _creation_ of `.../28/20260828140403`, but sees
nothing at all for `.../28/20260828140403/done.json`. So for any agent whose artifact
directory predates the watcher, **no marker write in that directory ever reaches ACE** —
not `done.json`, not `agent_meta.json`, not `waiting.json`, not `workflow_state.json`,
not `pending_question.json`.

This is not an edge case on this machine, it is the steady state. ACE hot-restarts
itself in place via `os.execvp` in `src/sase/dev_update/code_swap_guarded_exec.py`
whenever dev-update swaps in new code, which in this repo happens every time an agent
lands a commit — several times an hour. The PID and `ps` start time survive the exec, so
the restart is invisible in process listings, but the inotify fd set is rebuilt from
scratch. Every agent that was mid-run at that moment becomes permanently unwatched for
the rest of its life.

Live confirmation, taken twice minutes apart on the running ACE process:

- probe 1: watched `.../202608/28` plus exactly five 14-digit dirs, all created after
  the most recent exec; `.../28/20260828140403` (the `0fn` coder, created `14:04:03`,
  which is before that exec) **not watched**.
- probe 2 (after another exec — the inotify fds had changed): watched `.../202608/28`
  and **zero** 14-digit dirs, while five agent runners were live.

The `.ace_refresh_pulse` belt added in `dde1b22d8` did not cover this instance either:
`0fn--code`'s runner process started at `14:04:03`, before that commit landed at
`14:17:51`, so it was executing the pre-pulse `write_done_marker_and_update_index`.

### Gap 2 — the completion notification is the ideal trigger, and it is throttled

`~/.sase/notifications` **is** watched (added in `_start_artifact_watcher`,
`src/sase/ace/tui/actions/_startup_watchers.py`), the notification lands ~60 ms after
`done.json`, and its `action_data` already carries the exact `(cl_name, raw_suffix)` of
the finished agent. That is the cheapest and most direct convergence signal available:
one bounded exact artifact-dir delta, no broad load, no watch budget, no polling.

Three things stop it from being reliable:

1. **Tab gate.** `_run_scheduled_notification_poll`
   (`src/sase/ace/tui/actions/agents/_notification_deadlines.py`) only calls
   `request_notification_agents_refresh` when `current_tab == "agents"`. Off-tab, the
   toast fires and the row is not reconciled. This directly contradicts the comment
   already in `_run_auto_refresh_body`
   (`src/sase/ace/tui/actions/event_refresh/_auto_refresh.py`), which says a queued
   bounded exact delta "stays live off-tab -- that is what lets completion/unread state
   converge promptly while the user is on Artifacts or Axe".

2. **The auto-refresh fallback is dead in practice.** `_run_auto_refresh_body` has
   `if new_agent_notification and not agents_due: request_notification_agents_refresh(self)`.
   But when the watcher is active, the notifications watch already ran
   `_schedule_notification_poll(source="watcher")`, and that poll consumed the "new"
   signal (`_delivered_notification_activity_cursors` in
   `src/sase/ace/tui/actions/agents/_notification_polling.py`) and cleared
   `_dirty_notifications`. The tick's own `_poll_agent_completions()` therefore returns
   `False`. The tab-gated call above is the only live trigger.

   Separately, `not agents_due` is now wrong on its own terms: `agents_due` can be
   satisfied by consuming a queued exact delta for _unrelated_ artifact dirs, in which
   case the notified agent's dir is silently never reloaded.

3. **Roster-only targeting.** `_completion_notification_delta_dirs`
   (`src/sase/ace/tui/actions/agents/_notification_utils.py`) resolves dirs by
   intersecting active completion keys with the currently loaded roster, and it uses
   _every_ not-yet-dismissed completion notification. So a completion for a row the list
   has not loaded yet degrades to a broad Tier 1 load, while a busy notification inbox
   makes each single completion rescan every previously-completed agent's dir.

The "and unread" half of the symptom has the same cause:
`_reconcile_unread_from_completion_notifications`
(`src/sase/ace/tui/actions/agents/_notification_unread_projection.py`) projects unread
state onto loaded rows and only marks rows whose status is already a completed status.
While the row is stale-RUNNING it cannot be marked unread either.

## Work

### 1. Give `ArtifactWatcher` a thread-safe incremental watch API

`src/sase/ace/tui/util/fs_watcher.py`

- Add a reverse index `_wd_by_path: dict[str, int]` beside `_watch_paths_by_wd`, and
  guard both with a dedicated watch lock (`self._lock` today only guards
  `_last_event_mono` / `_pending_paths`; `_watch_paths_by_wd` is currently
  watcher-thread-only and is about to become cross-thread). Keep `_add_watch_path` and
  `_remove_watch` as the only mutators so the two indexes cannot drift.
- `_add_watch_path` must treat an already-watched path as a cheap no-op _before_ the
  `MAX_INOTIFY_WATCHES` check, so re-arming near the cap cannot spuriously fail.
- Add `ensure_watches(paths: Iterable[Path]) -> int`, callable from the UI thread:
  install a watch for each not-already-watched existing path, honour
  `MAX_INOTIFY_WATCHES`, and return how many were installed. It must be a no-op (return
  `0`) when the watcher never started or has been stopped.
- Add `prune_agent_dir_watches(terminal_dirs: Iterable[Path]) -> int`: drop watches on
  14-digit agent artifact directories (`_is_agent_artifact_dir_name`) that the caller
  reports as terminal. Deliberately conservative — never prune a directory merely
  because it is absent from the caller's set, only when the caller positively knows the
  row is finished. This reclaims the watch that `_add_watch_tree` installs for every
  agent directory created during the session, which is what currently makes the watch
  set grow by roughly one per launched agent per day.

Do not change `_iter_startup_watch_paths` to walk 14-digit dirs. Scanning the selected
shards on this machine finds 3,665 agent directories, 2,365 of them without a
`done.json` (dismissed, killed, or crashed runs), so "has no done marker" is not a
usable liveness predicate at startup and would blow the watch budget.

### 2. Re-arm live-agent watch coverage from the loaded roster

New small module under `src/sase/ace/tui/actions/agents/` (for example
`_live_watch_coverage.py`), called from `_apply_loaded_agents_prepared_inner` in
`src/sase/ace/tui/actions/agents/_loading_apply.py` immediately after
`self._agents_with_children = boundary.fold.unfiltered_agents`.

- Partition `_agents_with_children` with the existing `agent_row_is_in_flight` predicate
  (`src/sase/ace/tui/models/agent_family_members.py`) — it is already the
  liveness-filtered projection the list renders, so no new disk I/O and no new liveness
  rules.
- Call `watcher.ensure_watches(...)` with the in-flight rows' `get_artifacts_dir()`
  paths, and `watcher.prune_agent_dir_watches(...)` with the rows that are loaded and
  _not_ in flight.
- Cap the ensure set with a new module constant (`MAX_LIVE_AGENT_WATCHES = 256`),
  newest-first, and `log.warning` once when the cap bites. The real number here is
  ~10–20 rows; the cap exists so a pathological roster cannot starve the shard watches.
- Skip the whole pass when `_fs_watcher` is `None`, and keep it out of the measured hot
  path: it is a set difference plus a handful of `inotify_add_watch` syscalls, but it
  must not read the filesystem or touch `Agent` fields that trigger lazy disk lookups
  beyond `get_artifacts_dir()`.

This is what makes coverage self-healing: any ACE start, any `os.execvp` hot-restart,
any missed `IN_CREATE`, and any watch installed before the cap was hit gets repaired on
the very next agents load rather than never.

### 3. Make the completion notification an unconditional exact reconcile

`src/sase/ace/tui/actions/agents/_notification_deadlines.py`,
`src/sase/ace/tui/actions/agents/_notification_utils.py`,
`src/sase/ace/tui/actions/agents/_notification_polling.py`,
`src/sase/ace/tui/actions/event_refresh/_auto_refresh.py`

- Thread the _newly observed_ completion notifications out of
  `_poll_agent_completions_once` (it already computes `new_non_resurface_notifications`)
  and hand them to `request_notification_agents_refresh`, instead of re-deriving targets
  from every active completion notification in the snapshot. One completion must cost
  one artifact-dir scan, not one per unread completion in the inbox.
- In `request_notification_agents_refresh` / `_completion_notification_delta_dirs`, keep
  the loaded-roster lookup as the fast path, then fall back to resolving the
  notification's own `action_data["raw_suffix"]` to an artifact dir via
  `_artifact_dirs_for_normalized_timestamps`
  (`src/sase/ace/tui/models/_agent_loader_artifacts.py`, already re-exported by
  `agent_loader`) so a completion for a not-yet-loaded row still gets an exact delta.
  Only fall through to the broad `_call_schedule_agents_refresh` when neither resolves.
- In `_run_scheduled_notification_poll`, drop the `current_tab == "agents"` gate for the
  **exact-delta** outcome. Keep the broad fallback tab-gated so an unresolvable
  notification cannot trigger an off-tab broad Tier 1 load; off-tab, that case is
  correctly left to the sanity pass.
- In `_run_auto_refresh_body`, replace `if new_agent_notification and not agents_due`
  with a condition that only suppresses the notification-driven reconcile when this tick
  is actually going to run a broad/full agents load. A tick that satisfies `agents_due`
  by consuming a bounded delta for unrelated dirs must still reconcile the notified
  agent.

### 4. Pulse parity for the remaining done-marker writer

`src/sase/axe/runner_artifacts.py`

`write_done_marker` (used by the axe fix-hook, summarize-hook, CRS, and mentor runners)
writes `done.json` and updates the index but does not call `touch_shell_refresh_pulse`,
unlike `write_done_marker_and_update_index` in `src/sase/axe/run_agent_exec_markers.py`.
Add the same best-effort pulse there, derived with `project_name_from_artifacts_dir`, so
every done-marker writer has the watcher-independent belt.

Leave `.ace_refresh_pulse` classification alone. It maps to no artifact dir, so
`_agent_artifact_delta_dir_for_path` reports `unknown_watcher_path` and the tick escapes
to a broad load — which is the correct behaviour for a signal that does not identify a
row. With work items 1–3 landed, the pulse and the exact `done.json` event coalesce into
the same watcher batch, and `_agent_artifact_delta_dirs_for_paths` already prefers the
exact dirs whenever any are present, so the broad load only happens when there is
genuinely nothing exact to reload.

### 5. Tests

- `tests/ace/tui/test_fs_watcher.py`: `ensure_watches` installs a watch for a directory
  created before `start()`; re-calling it for an already-watched path installs nothing
  and does not consume budget; it is a no-op after `stop()`; `prune_agent_dir_watches`
  removes only 14-digit agent dirs it was given and leaves shard/project watches intact.
- New test module beside the existing agents-action tests: after an agents load whose
  roster contains an in-flight row whose artifact dir was created before the watcher
  started, the app calls `ensure_watches` with that dir; after the same row loads
  terminal, it calls `prune_agent_dir_watches` with it. Use a fake watcher object
  recording its calls; do not spin a real inotify fd.
- `tests/test_notification_toast_polling_agent_refresh.py` (extend): a completion
  notification for a row that is **not** in the loaded roster resolves to an exact
  artifact-dir delta via its `raw_suffix` rather than a broad refresh; an unresolvable
  completion still falls back to broad; the exact-delta path fires while
  `current_tab != "agents"` and the broad fallback does not; one new completion arriving
  alongside twenty already-unread completions schedules a delta for one dir.
- `tests/ace/tui/test_event_handlers_auto_refresh_dirty_flags.py` (extend): a tick with
  `new_agent_notification=True` that consumes a queued exact delta for unrelated dirs
  still requests the notification-driven reconcile; a tick that runs a broad load does
  not double-request it.
- `tests/test_run_agent_artifact_index_updates.py` (or its neighbour
  `tests/test_agent_artifact_marker_mutation_audit.py`):
  `runner_artifacts.write_done_marker` touches `<project>/artifacts/.ace_refresh_pulse`.
- Regression guard for the reported shape, using the repo's SASE-home fixtures: a
  plan-workflow family root whose approved-tale coder child flips from `RUNNING` to a
  `done.json` `completed` outcome renders the root as `TALE DONE` after an exact
  artifact delta for the child dir alone. This asserts the merge path
  (`merge_incomplete_load_after_complete_history` →
  `_normalize_relationships_after_merge` → `apply_status_overrides`) re-mirrors the
  container, so a future change cannot reintroduce the symptom from the loader side.

Every filesystem test must use `tmp_path` or the repo's SASE-home fixtures. Never touch
the real `~/.sase` — a test that did exactly that is the likely origin of the 313
future-dated junk shards still sitting in this machine's `ace-run` tree.

## Performance notes

The user's explicit constraint is that this must not cost TUI responsiveness. The design
is net-negative on cost:

- Work item 2 adds a set difference over the already-loaded roster plus, typically, zero
  to two `inotify_add_watch` syscalls per agents load. No new disk reads.
- Work item 1's pruning _shrinks_ the steady-state watch set. Today the day-shard watch
  installs a permanent watch for every agent directory created during the session
  (~100/day/project on this machine, never reclaimed). After pruning, the set converges
  to `startup shards + live agents`, which also removes the silent `MAX_INOTIFY_WATCHES`
  exhaustion failure mode that reproduces this exact bug.
- Work item 3 replaces broad Tier 1 loads with bounded exact deltas on the completion
  path, and stops one new completion from rescanning every unread completion's artifact
  dir.

## Out of scope

- Do not change `FULL_SANITY_REFRESH_SECONDS`, `AGENTS_LOAD_MIN_INTERVAL_SECONDS`, or
  `TIER1_INDEX_REVALIDATE_MIN_INTERVAL_S`. The sanity pass is the safety net that has
  been masking this; it is not the bug, and lowering it would trade correctness for
  cost.
- Do not touch the SQLite artifact index, the Tier 1/Tier 2 loader tiering, or the
  family status derivation in `_agent_status_apply.py`. All three were verified correct
  during diagnosis.
- Do not delete the 313 future-dated junk month shards under
  `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/`. They are directories in the
  user's real `~/.sase`; removal stays an explicit user action, and
  `sase doctor resources.ace_run_watches` already reports them.
- Do not add pulses for non-completion markers (`waiting.json`, `pending_question.json`,
  `agent_meta.json`). Work items 1 and 2 already restore watcher coverage for those
  writes, which is the general fix; per-marker pulses would be redundant write traffic.

## Verification

1. The unit tests above, then `just check`. This touches `src/sase/ace/` and
   `src/sase/axe/`, so run `just check-full` through `/sase_monitor` before landing.
2. End-to-end on the live machine, which reproduces this directly:
   - Launch a short `sase` agent and let it start.
   - While it runs, force an ACE hot-restart (land a commit so dev-update execs it, or
     restart ACE by hand) so the agent's artifact directory predates the watcher.
   - Confirm coverage was re-armed: find the ACE pid, read the `ino:` values from
     `/proc/<pid>/fdinfo/<inotify-fd>`, and check that the agent's 14-digit artifact
     directory inode is now present — before this change it is absent.
   - When the agent finishes, its Agents-tab row (and its family container) must leave
     the Running bucket and show as done and unread within one auto-refresh tick of the
     completion toast, not up to 60 seconds later.
   - Repeat with the Agents tab **not** focused, then switch to it: the row must already
     be terminal rather than converging on the tab switch.
3. Sanity-check the watch set does not grow across a long session: sample the watch
   count from `/proc/<pid>/fdinfo/<inotify-fd>` after several agents complete and
   confirm it stays flat rather than climbing by one per completed agent.
