---
tier: tale
title: Remember the Agents tab query across restarts
goal:
  Restore the last committed Agents query, including an intentional empty query, across
  ACE sessions and machine restarts without slowing the UI or overwriting newer input.
size: medium
proposed_by: bbugyi200.athena.0jn
create_time: 2026-09-11 14:21:50
status: wip
---

# Remember the Agents tab query across restarts

## Outcome and scope

After the user submits a structured query on the top-level Agents tab, the next ACE
session restores that query automatically. Submitting an empty query remembers the
unfiltered view. Persistence belongs to the user's machine-local SASE state, shared by
ACE sessions using the same `SASE_HOME`, regardless of working directory or project. The
default state location is under `~/.sase`, outside disposable workspaces and temp
directories, so successful saves survive process exits and machine restarts.

This is a **medium tale**: one coding agent can implement and verify the bounded
storage, lifecycle, and query-entry integration together. It does not need independently
landed epic phases. Authoring this tale is large planning work under `sase_sizes.md`;
the approved implementation is medium direct work.

## Findings that determine the implementation

- `src/sase/ace/tui/actions/_state_init_agents.py:init_agent_state` always initializes
  `_agent_search_query` to `""` and resets the current-project seed markers.
- `actions/agents/_filter_bar_session.py:_commit_agents_filter_query` commits the
  unified query. Its typing, history navigation, and `#` saved-slot commands operate on
  separate preview state until the user submits. The feature-flag Off path commits
  through the modal callback in `actions/agents/_filter_actions.py`.
- `tui/modals/machines_pane.py:action_show_agents` also directly replaces the live
  Agents query. It must participate in remembering explicit query changes.
- `actions/agents/_search_query_seed.py` seeds from the current project once when
  `ace.current_project.seed_agents_query` is enabled. Both load entry points in
  `actions/agents/_loading_disk.py` consult this seed before capturing the query for
  provider loading. Production startup runs the async loader after first paint.
- `src/sase/ace/saved_queries.py:last_query.txt` and the CLI default in
  `src/sase/main/parser_ace.py` concern Patches. Query history stores previous/next
  entries, not the active Agents query. Neither store can infer the desired last query.
- Existing TUI persistence provides suitable patterns:
  `actions/_admin_center_persistence.py` has a serial, coalescing background writer and
  bounded exit flush; `models/agent_fold_persistence.py` and
  `modals/config_center_state.py` use bounded reads and same-directory atomic writes.
  `src/sase/ace/_query_persistence_io.py` currently writes directly to its target, so
  its helper is not suitable for this restart-sensitive store without modification.

Paths abbreviated as `actions/`, `models/`, or `modals/` above are relative to
`src/sase/ace/tui/`.

## Behavioral contract

1. Remember the **committed source text**, preserving spelling, quoting, and relative
   expressions. Evaluate it against current data and time on restart; do not persist
   results, a parsed AST, or a corpus. A query matching zero agents is still valid and
   must restore.
2. Live typing, invalid submissions, Escape, merely selecting history for preview, and
   saving/deleting a numbered slot do not replace the remembered query. A later
   successful Enter on a recalled query does. Bare `/` metadata search remains a
   separate, session-only operation.
3. An explicit empty/whitespace-only submission persists an empty query record. It must
   be distinguishable from a missing record and must suppress current-project seeding on
   future sessions. Saving the same source as the current startup default still
   establishes this explicit choice.
4. Startup precedence is: a newer explicit choice made in this session; otherwise a
   compatible remembered query (including empty); otherwise the existing optional
   current-project seed; otherwise empty. Automatic project seeding stays session-only
   until the user explicitly submits that query. Restoring a query does not push
   history, change saved slots, or cause an unsolicited persistence write.
5. Support both states of the existing `agents_unified_query` flag. Remember the
   producing dialect and use existing validation/canonicalization APIs. Nonempty records
   from another dialect or a stale unified profile must not be silently reinterpreted.
   Ignore them for this session, preserve their on-disk bytes until a new explicit
   commit, and show one actionable warning. Empty means unfiltered in either dialect and
   remains restorable. Do not add a new flag or configuration field.
6. The state is machine-local, with no cross-machine synchronization or project
   partitioning. Concurrent ACE sessions have atomic last-completed-write-wins behavior.
   An idle session or a session that only restored a query must not overwrite another
   session's changes when it exits. No live synchronization between open TUIs is needed.

## Implementation

### 1. Add a small TUI resume-state store

Add `src/sase/ace/tui/models/agent_query_persistence.py` for synchronous, explicitly
called load/save operations and an immutable record. Use
`sase.core.paths.sase_home() / "ace_agents_last_query.json"`, resolved at call time so
`SASE_HOME` and test isolation work normally. Use a versioned JSON object containing the
dialect and the established `QueryRecord` fields (`source`, `canonical`, and
`profile_digest`). The absence of a usable record is represented separately from a
record whose source and canonical strings are empty. No migration from Patches,
Artifacts -> Agent, saved slots, or query history is appropriate.

Keep this code in the TUI layer: it persists this frontend's navigation preference, like
its Admin Center and fold state. It adds no shared agent selection or query semantics.
Unified parsing and evaluation continue through the existing Rust-backed profile APIs;
no Rust API change or Python replacement of core logic is needed.

Bound the complete serialized file to 64 KiB on both read and write. Validate version,
field types, recognized dialect, and record shape before applying it. Missing files mean
no remembered choice. Truncated JSON, bad UTF-8, oversized data, unsupported versions,
and read errors must leave startup usable and preserve the existing file. For nonempty
compatible records, reparse the source with the active dialect and compare its canonical
form with the saved canonical text before accepting it; apply the unified profile-digest
check used by existing query-history replay. Perform no ad hoc syntax translation.
Normalize whitespace-only committed sources to `""`. Do not truncate an oversized query
into a different query. Report a save failure while keeping the submitted query active
in memory.

Write a complete snapshot to a uniquely named temporary file in the same directory;
flush and fsync the file, atomically replace the target, then fsync the parent directory
on supported platforms. Clean up temporary files on failures. Failures before replace
preserve the old complete record; failures after replace must not attempt a destructive
rollback. Retain a clear failure result for the lifecycle caller. Do not broaden this
change into rewriting the other query stores or building a generic persistence
framework.

### 2. Coordinate committed state, restore, and background writes

Add a focused `actions/agents/_query_persistence.py` mixin and wire it into the Agents
mixin composition and state initialization. Keep in-memory committed-query generation,
one-shot restore state, latest pending immutable snapshot, and one active writer task.
Use the existing `spawn_pump_free_task` lifecycle and `asyncio.to_thread` for all new
disk/JSON work. Worker threads return data; they do not mutate widgets or app state.

Add one shared helper that records an explicit committed query, marks the seed attempt
as settled even for empty text, clears the seeded marker, advances the generation, and
queues persistence. Call it from unified Enter, successful legacy modal submission, and
Machines -> Show Agents. Preserve each entry point's existing history and refresh
behavior; the helper must not accidentally treat the Machines shortcut as a filter-bar
preview commit. Sweep assignments to `_agent_search_query` to ensure the only other
writes are initialization and non-persisting startup restore/seed operations.

Integrate a one-shot restore into the async Agents load before current-project seeding
and before capturing query/provider request keys. Perform its bounded read and stored
query validation off-thread in that already-background workflow. First paint and other
tabs must stay responsive, and normal refreshes must not re-read the store. Coalesce any
overlapping initial-load requests onto one restore operation. Keep synchronous
test/replay callers compatible without adding synchronous disk I/O to the UI path; they
can exercise the shared restore-result application with an explicitly loaded snapshot
rather than introducing a second automatic startup path.

On return from an await, check the current generation and lifecycle state again. A new
explicit commit or clear must win over the older disk read. Also ensure an in-flight
current-project resolution cannot reseed after a user clear. If the editor opened before
restore completed, preserve its draft and focus: a valid restore may establish the
committed baseline, Escape returns to that baseline, and Enter establishes the newer
choice. Do not overwrite editor text or apply stale preview results. Clear stale query
caches as needed and feed the restored query through the existing provider, finalize,
info-panel, unread-jump, and prospective-clan paths; do not add another filter.

Start saving after each explicit commit, rather than waiting for exit. Serialize writes
within an app and coalesce successive pending snapshots so an older worker cannot finish
after a newer one and restore stale text. Deduplicate already durable snapshots, but do
not deduplicate away the first explicit empty choice or a retry after failure. Use a
rate-limited warning/log for failures, preserve dirty state, and permit retry on the
next commit and once during controlled exit; avoid an unbounded retry loop.

### 3. Flush pending work during the existing controlled exit

Add `_flush_agents_query_state` to the flush collection and capability detection in
`src/sase/ace/tui/actions/lifecycle.py`. Use the existing two-second bounded persistence
flush pattern, before `_do_quit` cancels background tasks. Only drain dirty/pending
commits; do not rewrite the currently restored state just because ACE is closing.
Preserve the existing quit/signal routes and failure-tolerant cleanup. Cancellation of
an async wrapper must not create a second concurrent writer while its thread still
writes. Unmount must not leave callbacks updating disposed widgets.

Completed saves survive restart without requiring a graceful application exit. The small
interval before a background save finishes, a forced kill during that interval, and
actual storage failure cannot promise the newest query is durable. Controlled exit
drains that interval within the existing bounded shutdown policy.

### 4. Document the behavior

Update `docs/ace.md` under Agent Search and its current-project discussion, and
`docs/configuration.md` under `ace.current_project`, to explain submitted-query
persistence, clearing, and seed precedence. Clarify the existing introductory last-query
CLI description as Patches-specific. Keep existing keybindings/defaults. Any new
contextual help wording must retain the help popup's width conventions.

## Verification and acceptance

Use the repository's isolated `SASE_HOME` fixtures and temporary directories. No test
should read or overwrite a developer's real last query. Add storage and lifecycle tests
in the established TUI model/action test layout, and extend the existing integration
tests rather than duplicating their entire harness. The restart/first-provider-query
tests must exercise the real startup restore and Agents async-load entry points. Stub
the underlying data sources, not the entire load method: the existing visual startup
helper offers `use_real_agent_loader=True` for this purpose. Update focused fake hosts
to expose the new commit helper where needed.

- **Restart round trip:** submit a nonempty query in one `AcePage`, let the save finish,
  close it, and create a fresh app against the same isolated state. The source, info
  panel, initial provider query, and filtered results agree, including a zero-match
  query. Repeat with an explicit empty query and changed current project. Include a
  fresh-process storage read to establish independence from module/app caches; no real
  machine reboot is required for automated verification.
- **Editing boundaries:** extend `tests/ace/tui/test_agents_filter_bar_session.py` for
  invalid Enter, Escape, valid preview without Enter, history preview, saved-slot
  commands, clear, and resubmitting an unchanged query. Only committed changes persist.
  Exercise legacy submission/cancel and Machines -> Show Agents in
  `tests/ace/tui/test_machines_pane.py` under both flag states.
- **Startup precedence:** extend `tests/ace/tui/test_agents_tab_current_project_seed.py`
  for absent, remembered, explicitly empty, and rejected records with seeding on/off.
  Restore must precede provider filtering, suppress seeding when appropriate, and leave
  history intact. Starting on another tab must still restore before the Agents data is
  used.
- **Deterministic races:** delay the reader/writer with synchronization events. Commit
  or clear while restore/current-project resolution is pending and prove it wins.
  Open/type/cancel or submit while restore is pending and verify draft/baseline
  behavior. Submit A, B, then empty while A is writing; the final disk value must be
  empty. Verify key processing continues while file I/O is blocked and refreshes do not
  repeatedly read the store. Avoid sleep-based timing assertions.
- **Faults and compatibility:** cover missing/invalid/oversized files, bad UTF-8,
  unknown versions, incompatible dialect/profile/canonical data, failed writes, and
  retry. Atomic-write failure injection leaves a complete old/new record, not partial
  JSON. Two app instances must not share temp filenames, and closing an idle instance
  must not replace another instance's newer persisted query.
- **Shutdown:** extend `tests/ace/tui/actions/test_lifecycle_quit_confirm.py` for query
  flushing before quit, rapid commit then quit, and bounded failure/timeout. Retest
  fold/Admin Center flushes. Check restore/save cleanup cannot restart work after
  teardown.
- **Regression:** run the relevant unified/legacy Agents query/filter/refresh tests,
  Machines tests, query history/saved-query tests, current-project seed tests, and
  existing lifecycle tests. Ensure Patches `last_query.txt`, Artifacts -> Agent state,
  saved slots, and metadata search retain their existing behavior.

Before implementation verification, read `lint_and_test.md` through `sase memory read`.
Run formatting as appropriate and **`just check`** on the final tree. If test selection
escalates or is unusual, follow the documented `check-full` policy using
`/sase_monitor`; exhaustive landing verification remains host-owned. If startup or key
responsiveness changes unexpectedly, follow `tui_perf.md` and `docs/perf_runbook.md`
rather than adding synchronous reads or extra reloads.

The tale is complete when an explicit query or clear is restored by a fresh session, all
identified commit routes and startup/shutdown races obey the contract, and the required
checks pass. This plan authoring turn changes no implementation files.
