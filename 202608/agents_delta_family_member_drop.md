---
tier: tale
title:
  Stop the artifact delta from dropping family-member rows, and stop monitor start from
  destroying shell quoting
goal:
  A family member that completes (a monitor starter, a monitor shell, a planner, a
  feedback round) updates in the Agents tab on the next incremental refresh instead of
  leaving a stale row that renders as FAILED in the default tribe, and `sase monitor
  start -- <argv>` runs exactly the command the caller quoted.
size: medium
proposed_by: bbugyi200.athena.0fa
create_time: 2026-08-27 19:33:28
status: wip
---

# Plan

## Problem

Two independent defects combined into one confusing incident on 2026-08-27. Both are
reproduced below with concrete measurements; fix both.

### Symptom the user reported

In the ACE Agents tab (build `v0.16.0+1660.g794fbd3db`, screenshot at 19:01 local):

- `sase-ud.13.1.3.1.4--1` — an agent that **completed successfully** at 18:45:49 after
  handing its turn to a monitor — rendered under the `@default` tribe's `Failed` group
  as `sase (FAILED)`, outside its own clan.
- Its monitor member `sase-ud.13.1.3.1.4--mon-0` rendered as a **direct child of the
  clan row** `sase-ud.13.1.3.1`, a sibling of the `.1` / `.2` / `.3` / `.4` / `.land`
  family rows, instead of nesting under family `sase-ud.13.1.3.1.4`.

Both on-disk records were healthy at that moment:

- `--1`'s `done.json` had `outcome: "completed"`, and its runner log ended with
  `Agent completed with status: SUCCESS`.
- `--mon-0`'s `agent_meta.json` had `agent_family: sase-ud.13.1.3.1.4`,
  `agent_clan: sase-ud.13.1.3.1`, and `parent_timestamp` pointing at `--1`'s artifact
  timestamp.

So nothing failed. The Agents tab was showing a stale row.

### Defect 1 — the artifact delta silently drops every family-member row

`sort_and_reorder()` in `src/sase/ace/tui/models/_agent_ordering.py` (the
`followups_by_parent` split near the top of the function) pulls every
`is_family_member_child` row out of the top-level list into a bucket keyed by
`parent_timestamp`, and re-emits it only via `_append_followup_subtree()` when the
parent row happens to be present in the **same batch**. Buckets that are never consumed
are discarded — there is no leftover pass, so the rows vanish.

`load_artifact_delta_agents()` (`src/sase/ace/tui/models/agent_loader.py`) scans an
exact set of artifact dirs. The watcher derives that set one-for-one from changed paths
in `_agent_artifact_delta_dirs_for_paths()`
(`src/sase/ace/tui/actions/event_refresh/_artifact_delta.py`) with no family closure.
When a family member writes its `done.json`, only that member's dir changed, so the
batch contains no family container — and the member row is dropped on the floor.

Measured on the current tree against real artifacts from the sibling family
`sase-ud.13.1.3.1.1` (`load_artifact_delta_agents([...], update_index=False)`):

| delta batch                | rows returned      |
| -------------------------- | ------------------ |
| family container dir alone | 7                  |
| member `--1` dir alone     | **0**              |
| member `--mon-0` dir alone | **0**              |
| container + member         | 8 (member present) |

Instrumenting the pipeline for the member-alone case shows the record loads fine and is
only lost at the very end:

```
start           2 rows  [(--1, run, DONE), (--1, workflow, DONE)]
after dead-pid  2 rows
after dedup     1 row   [(--1, workflow, DONE)]
after status    1 row   [(--1, workflow, DONE)]   # agent_family + agent_clan both set
after sort      0 rows
```

Because the delta returns no row for the requested dir,
`merge_incomplete_load_after_complete_history()` (reached from
`_prepare_loaded_agents_worker_prep()` in
`src/sase/ace/tui/actions/agents/_loading_compute.py`) keeps the previously **cached**
row for that member. The Agents tab therefore never observes the member's completion:
the stale pre-handoff row survives, is later projected as `FAILED` because its runner
PID is dead with no terminal marker on the row, and its degraded attribution drops it
into `@default` instead of the clan's `@epic` tribe. With no fresh family-member row for
`--1`, `--mon-0`'s parent lookup fails and the monitor row is re-parented onto the clan
container — exactly the two anomalies in the screenshot.

This was a rare path until commit `794fbd3db` ("perf: Make the artifact delta the
default refresh, not the 2% exception"), which is the newest commit in the build the
screenshot was taken on. That commit made the broken path the default, so family-member
rows stopped updating on incremental refreshes.

### Defect 2 — `sase monitor start -- <argv>` destroys the caller's shell quoting

`_start_command()` in `src/sase/main/monitor_handler.py` rebuilds the monitored command
from the post-`--` argv remainder with `" ".join(words)`. `compile_monitor_argv()` in
`src/sase/monitor/proc_adapter.py` then executes that joined string as
`/bin/sh -c <string>`. Every word boundary the caller established with quotes is lost in
the round-trip.

In the same incident the starter agent invoked, in effect:

```
sase monitor start ... -- sh -c 'just check-full; check_status=$?; just test-visual; \
  visual_status=$?; if [ "$check_status" -ne 0 ]; then exit "$check_status"; fi; \
  exit "$visual_status"'
```

and the durable proc row recorded the executed argv as:

```json
[
  "/bin/sh",
  "-c",
  "sh -c just check-full; check_status=$?; just test-visual; visual_status=$?; if [ \"$check_status\" -ne 0 ]; then exit \"$check_status\"; fi; exit \"$visual_status\""
]
```

The inner `sh -c just` therefore ran `just` with `check-full;` as `$0`. The monitor's
captured output opens with `just`'s `Available recipes:` listing, not a `check-full`
run. `check_status` captured that listing's exit status, the guard
`if [ "$check_status" -ne 0 ]` passed vacuously, and only `just test-visual` actually
ran (it failed on three PNG snapshots, which is the exit 1 the monitor reported).

The dangerous part is not the failure, it is the near-miss: had `test-visual` passed,
the follow-up agent would have been handed a green verdict for a combined `check-full` +
`test-visual` verification in which `just check-full` never executed, and the phase bead
would have been closed on it. The follow-up agent burned a turn rediscovering this from
the log.

This is specific to the `--` remainder path. The hidden `-c/--command '<script>'` alias
passes its string through untouched, which is why the same family's other two monitors
(`--mon` and `--mon-1`) ran the intended command correctly.

## Non-goals

- Do not change the "monitored commands run under `sh -c` on Unix" contract.
- Do not change agent dismissal, cleanup, or artifact-retention behavior.
- Do not attempt to reduce the number of artifact-delta refreshes; the delta path is
  correct as a design, it is just not family-closed.

## Implementation

### 1. Make row ordering total (never silently drop a row)

In `sort_and_reorder()` (`src/sase/ace/tui/models/_agent_ordering.py`), add a final pass
that appends any `followups_by_parent` buckets still unconsumed after the parent walk,
in chronological order, before `project_clan_tree()` runs. An orphaned family member
(parent filtered out, dismissed, or simply outside the batch) must still reach the
output as a top-level row rather than disappearing.

Keep the ordering guarantee explicit: the set of rows out of `sort_and_reorder()` is the
set of rows in, with each row emitted exactly once. `_append_followup_subtree()` already
pops each bucket as it is consumed, so the leftover pass is a straight drain of what
remains.

### 2. Make the artifact delta family-closed

In `load_artifact_delta_agents()` (`src/sase/ace/tui/models/agent_loader.py`), expand
the requested artifact-dir set to include the family container (and the intervening
parent chain) of every scanned record that is a family member, before projecting. This
is what makes family, clan, tribe, and status projection correct rather than merely
present.

Resolve the parent chain from the scanned record's own `agent_meta.parent_timestamp` /
`agent_family` plus the persistent artifact index, which already stores
`parent_timestamp` and `agent_family` per record (`agent_artifacts` table) — do not fall
back to a broad rescan. Keep the expansion bounded: parent chain only, deduplicated,
with a small hop cap and a cycle guard, so a malformed `parent_timestamp` chain cannot
loop or blow up the batch.

### 3. Escalate instead of going stale

`_load_agent_artifact_delta_async()`
(`src/sase/ace/tui/actions/agents/_loading_disk.py`) already returns `False` and falls
back to a broad refresh when `load_state.repair_recommended` is set. Extend the delta
load state so that a requested, non-deleted artifact dir that produced **no visible
row** sets that signal. A delta that cannot project a record it was asked about must
escalate to a broad refresh rather than silently leaving the cached row in place. This
is the backstop that would have surfaced the bug immediately instead of showing a
phantom `FAILED` row for sixteen minutes.

### 4. Preserve shell quoting in `sase monitor start`

In `_start_command()` (`src/sase/main/monitor_handler.py`):

- A **single-word** remainder stays verbatim. That is the documented canonical form
  (`-- 'just check-full; just test-visual'`) and it is already a shell script string.
- A **multi-word** remainder is joined with `shlex.join()` so the caller's quoting
  survives into `/bin/sh -c`. `-- sh -c '<script>'` then executes the script the caller
  wrote, and `-- echo "a b"` stops becoming `echo a b`.
- `-c/--command` keeps its current pass-through semantics.

This is an intentional behavior change for one shape: callers who pass unquoted shell
operators as separate remainder words (`-- a '&&' b`) currently get them interpreted and
will stop. Before landing, grep `src/`, `tests/`, `smoke/`, and
`src/sase/xprompts/skills/` for that shape and fix any occurrences. Record the change in
`CHANGELOG.md`.

Update the docs so the contract is stated once and correctly:

- `src/sase/main/parser_monitor.py`: the `start` subparser epilog and the
  `monitor_command_words` help should say that the remainder is re-quoted with shell
  quoting and run under `sh -c`, and that a whole script belongs in a single quoted
  argument.
- `src/sase/xprompts/skills/sase_monitor.md`: the "Canonical Invocation" and "Sleep Or
  Wait" examples currently use `--command`, which `parser_monitor.py` marks
  `argparse.SUPPRESS` as a hidden compatibility alias. Make the skill and the CLI agree
  on one recommended spelling, and extend the existing "Hazards" bullet about quoting to
  say explicitly that a multi-command script must be one quoted argument.

## Tests

All new tests must build their own artifact fixtures under a temporary SASE home. None
of them may read the developer's real `~/.sase` tree.

1. `tests/ace/tui/models/` — ordering totality: `sort_and_reorder()` given a family
   member whose parent row is absent returns that member (a top-level row), and for a
   mixed batch returns exactly the input row set with no duplicates.
2. `tests/ace/tui/models/` — delta projection: `load_artifact_delta_agents()` over a
   synthetic family (container + `--plan` + `--mon` + `--1` + `--mon-0`) scoped to
   **only the `--1` member dir** returns a row for `--1` that carries `agent_family`,
   `agent_clan`, and status `DONE`. Extend
   `tests/test_agent_loader_epic_created_status.py` only if it is the natural home;
   otherwise add a focused module.
3. `tests/ace/tui/models/` — tree parenting: a monitor member whose starter is present
   in the same projection nests under its family container, not under the clan
   container. This pins the `--mon-0` half of the symptom.
4. Escalation: a delta whose requested, non-deleted dir yields no visible row sets
   `repair_recommended` (or the equivalent new signal) so
   `_load_agent_artifact_delta_async()` falls back to a broad refresh.
5. `tests/main/test_parser_monitor.py` (or a sibling handler test): `_start_command()`
   round-trips `["sh", "-c", "a; b"]` to `sh -c 'a; b'`, leaves a single-word remainder
   untouched, and preserves embedded spaces in `["echo", "a b"]`. Add an end-to-end
   assertion that the argv reaching `compile_monitor_argv()` re-parses under `sh -c` to
   the caller's original command.
6. A regression test that the recorded `monitor_command` for a `--`-remainder start is
   the same string that would be executed, so the value shown by `sase monitor show`
   cannot drift from what ran.

## Verification

- Run `just check` after the change (whole-repo lint gates plus the diff-scoped test
  lane).
- This change touches the Agents-tab loader and ordering, which the ACE PNG snapshot
  suite covers, so run `just check-full` and `just test-visual` before landing. Hand
  those to a monitor with the `/sase_monitor` skill rather than running them inline, and
  pass the whole script as a **single quoted argument** — the bug in section 4 is
  exactly what happens if you write `-- sh -c '<script>'` on an unfixed tree.
- Sanity-check the real behavior by hand: start a monitor whose command is a multi-part
  script, then confirm with `sase monitor show <id>` that the recorded command and the
  captured output correspond to the script that was requested.

## Risks

- Section 2's parent-chain expansion runs on every incremental refresh; keep it index-
  backed and bounded so it does not reintroduce the cost that commit `794fbd3db` was
  removing. Compare the Agents-tab refresh traces before and after.
- Section 1 changes `sort_and_reorder()`, which the full-load path shares. Orphaned
  family members that were previously invisible will now render. Check the ACE PNG
  goldens and the family/clan roster tests for expected diffs, and only update goldens
  that are genuinely the intended new behavior.
- Section 4 is a user-visible CLI behavior change; the compatibility grep and changelog
  entry are part of the work, not optional.
