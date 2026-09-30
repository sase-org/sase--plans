---
tier: tale
size: medium
title: Rust ack API, lean unread index, and store generations
goal:
  Move completion acks and the unread completion index into sase_core as GIL-released
  calls that return dismissed ids and a store generation, and replace the Python
  read-sequence fence with that generation so an ack cannot be resurrected by an older
  observation.
proposed_by: bbugyi200.athena.sase-1d7.12
bead: sase-1d7.12
create_time: 2026-09-30 15:16:30
status: wip
---

- **PARENT:**
  [202609/unread_ack_reliability_and_tui_responsiveness.md](https://github.com/sase-org/sase--plans/blob/main/202609/unread_ack_reliability_and_tui_responsiveness.md)
- **BEAD:**
  [sase-1d7.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d7/sase-1d7.12.md)

# Rust ack API, lean unread index, and store generations

Implements phase `core-unread-ack-index` on bead `sase-1d7.12` (parent epic `sase-1d7`).
One agent lands both repositories. The host commits `sase-core` first and writes
`sase-core-revision.txt`; do not hand-edit that pin, any `CHANGELOG.md`, any crate
version, or the `sase-core-rs` range in `pyproject.toml`.

## Decision: the agent-node projection stays in Python

The Rust-core litmus test (other frontends need the same answer) applies to the **store
index**: which completion and settlement rows exist, their agent key, `read`,
`dismissed`, and the store generation those rows were observed at. It does not apply to
"which Agents-tab nodes are unread."

That projection needs the loaded roster, manual `U` marks, and the session-local
pending-ack overlay. The epic keeps that presentation state in Python. Mobile and CLI
can project this same index onto their own agent lists. Do not add a Rust API that
returns unread node ids.

## What "done" means

- `ack_agent_completions(path, {agents})` dismisses the same rows as today's
  `DismissAgentCompletionsMatchingAgents` state update, under the store lock, with the
  GIL released. It returns the ids it newly dismissed and the store generation after the
  call.
- `read_unread_completion_index(path)` returns that generation plus lean rows
  `{id, agent: {cl_name, raw_suffix}, read, dismissed}` for completion and settlement
  rows only, GIL released.
- Every successful notification-store write bumps a persisted generation. No-op writes
  do not.
- The TUI ack pipeline calls `ack_agent_completions` and removes the returned ids from
  the cached snapshot. It does not decide those ids by scanning the cache.
- A pending ack retires when an applied index generation is **greater than or equal to**
  the generation the ack returned. The Python read-sequence counter is no longer the
  fence.
- Unread reconciles that already hold a snapshot project it into the same lean rows and
  pass `snapshot.generation`. A reconcile that needs store state and does not already
  hold a snapshot calls `read_unread_completion_index` off the UI thread. Neither path
  reads the store on the UI thread.

## Store generation

Add `crates/sase_core/src/notifications/generation.rs`.

The counter is a sibling of the JSONL, named like the lock file: `notifications.jsonl` →
`notifications.jsonl.generation` (the filename plus `.generation`). Contents are one
canonical decimal `u64` and a trailing newline. Missing file means `0`. Corrupt contents
(extra lines, a non-integer, a value that does not fit in `u64`) are an error, not a
silent `0`. A read must not create the file or its parent directory.

Bump only inside the exclusive store lock, after the JSONL mutation has succeeded and
before the lock is released:

- Call the bump from `write_notifications_atomic` (rewrite, reconcile, state updates
  that write, plus-one, upsert, compaction, snooze-expiry rewrite).
- Call it from `append_notification_unlocked` (append does not go through the atomic
  rewrite).

`checked_add(1)` and return the error on overflow. Do not wrap. Do not bump when the
caller returns without writing: reconcile with no field changes,
`apply_notification_state_update` with `changed_count == 0`, plus-one `no_match`.

Sample the generation **in the same critical section** as the rows it describes, after
any write that section performed. Do not read rows, drop the lock, and then read the
generation file. A missing JSONL still reports the sibling file's value when that file
can be read without creating directories, and `0` otherwise.

`store.rs` is already over the 1,500-line file guideline. Keep the new logic in
`generation.rs` and `ack.rs`. `pub(crate)` only the lock, row-read, write, and matcher
helpers `ack.rs` needs. Do not add names to the crate-root `pub use` list or to
`sase_core_py`'s prelude.

`NotificationStoreSnapshotWire` gains `generation: u64` with `#[serde(default)]`. Set it
on every snapshot return from the value sampled under that read's lock.
`NOTIFICATION_STORE_WIRE_SCHEMA_VERSION` stays `1`. Old Python parsers ignore unknown
keys; new Python treats a missing field as `0`. This is additive, not a breaking wire
change.

## Ack API

New request and outcome wires in `wire.rs`, schema version 1:

- Request: `agents: Vec<NotificationAgentKeyWire>`.
- Outcome: `dismissed_ids: Vec<String>`, `generation: u64`.

`ack_agent_completions` in `notifications/ack.rs`:

1. Take the exclusive store lock.
2. Read every live row.
3. For each row that is not already dismissed, dismiss it when
   `matches_agent_completion_notification_for_agents` **or**
   `matches_agent_settlement_notification_for_agents` matches. Set `dismissed = true`
   and `snooze_until = None`, matching today's `DismissAgentCompletionsMatchingAgents`
   arm. Already-dismissed rows are unchanged and are not returned.
4. Collect ids in file order.
5. If the id list is empty, do not write. Return the current generation.
6. If it is not empty, persist through the existing merge/compaction write
   (`merge_and_rewrite_notifications_unlocked` or the same write it uses) so compaction
   stays on this path. The bump inside that write is the only bump. Return the ids and
   the generation sampled after the write.
7. Release the lock.

Empty `agents` is a no-op read of the current generation: no bump, empty ids. Do not
invent a second matcher. Empty `raw_suffix` on a completion row still matches on
`cl_name` alone; settlement rows still require an exact `(cl_name, raw_suffix)` and have
no `cl_name`-only fallback.

## Lean index

`read_unread_completion_index` returns:

- `schema_version`
- `generation` sampled under the same lock as the rows
- `rows`, in file order, one per live completion row and one per live settlement row,
  **including dismissed rows**

A completion row is whatever `matches_agent_completion_notification` accepts
(`sender == "user-agent"`, action `JumpToAgent` or `ViewErrorReport`, non-empty
`cl_name`). A settlement row is whatever `matches_agent_settlement_notification` accepts
(`epic-launch` or `monitor-settlement`, non-empty `cl_name` and `raw_suffix`).
Pending-gate rows and every other sender stay out.

Each row is `{id, agent, read, dismissed}`. `agent.raw_suffix` is `None` when the stored
suffix is missing or empty, so it matches Python's `or None`. `read` and `dismissed` are
the stored flags. Do not drop silent rows. Do not read the archive file. Do not expire
snoozes on this read. Compaction that the snapshot reader would perform on a large store
still runs, and the returned generation is the post-compaction value.

Reuse `read_rows_unlocked`. Do not add a second JSON parser.

## Python bindings

Follow the `reconcile_notification_rows` binding in
`crates/sase_core_py/src/notifications/mod.rs`:

- `#[pyfunction]` / `#[pyo3(name = "ack_agent_completions")]` and
  `#[pyo3(name = "read_unread_completion_index")]`.
- Parse the ack request dict into the request wire. Malformed input is `ValueError`.
- `py.allow_threads` around the core call.
- Register both in `register_notifications`.
- Add a round-trip test in that module's `tests.rs`: ack dismisses a matching completion
  and a matching settlement, leaves a remote-attention row, returns those two ids and a
  generation one higher than before the call; a second ack with the same keys returns no
  ids and the same generation; the index lists the completion and settlement rows with
  the flags and that generation, and omits the unrelated row.

In sase, call them only through `require_rust_binding("ack_agent_completions")` and
`require_rust_binding("read_unread_completion_index")` in
`src/sase/core/notification_store_facade.py`. Rehydrate in
`src/sase/core/notification_store_wire.py` with `generation: int = 0` added to
`NotificationStoreSnapshotWire` (defaulted, at the end). Add frozen dataclasses for the
ack outcome and the index. `notification_snapshot_from_dict` reads
`int(data.get("generation") or 0)`.

Facade rules:

- Ack goes through `_call_mutating_binding`, which already drops the snapshot memo.
- The index is a read. Memoize it the way `_read_snapshot_cached` memoizes snapshots,
  including the "token changed during the read, so do not cache" race. Key the memo by
  path only. `invalidate_notification_snapshot_cache` drops it too.
- `src/sase/notifications/store.py` exposes `ack_agent_completions(agent_keys)` and
  `read_unread_completion_index()` against `notifications_file_path()`. Export both from
  `src/sase/notifications/__init__.py`.
- Leave `dismiss_agent_completion_notifications_matching_agents` in place. `_marking.py`
  and `_dismiss_agent_completion_notifications_for_dismissed_agents` still call it. Do
  not move those callers in this tale.

## Fence

`src/sase/ace/tui/actions/agents/_pending_ack_fence.py` today stores `(op_id, done_seq)`
and retires when a snapshot's **read-start sequence is greater than** `done_seq`.
Replace that with the store generation:

- The overlay value becomes `(op_id, done_generation: int | None)`.
- `mark_pending_ack_write_complete(app, op_id, identities, generation)` records the
  generation `ack_agent_completions` returned. `None` still means the write has not
  landed.
- `retire_pending_ack_entries(app, applied_generation)` drops an entry only when
  `done_generation` is not `None` and `applied_generation >= done_generation`.
- Equal generations **do** retire. The ack's returned generation already includes its
  write, which is the opposite of the old read-sequence counter (that counter identified
  a read that had started, not a write that had landed). Rewrite
  `test_pending_entry_retires_only_after_post_write_read` accordingly: an observation at
  `generation - 1` keeps the overlay and does not resurrect; an observation at
  `generation` retires, so a still-active row may become unread.
- An entry whose `done_generation` is `None` never retires. The existing "poll before
  the worker finishes" tests keep that behavior.
- Undo still uses `release_pending_ack_entries`.

Stop stamping `_sase_notif_read_seq` in `_read_notification_snapshot_from_provider`. The
cache in `_set_notification_snapshot_cache` becomes monotonic in store generation:
reject an incoming snapshot whose generation is strictly less than the cached
generation. A snapshot with no generation (test doubles, locally derived snapshots)
inherits the cached generation and is accepted. Read generation from the wire field
`generation`, and accept the same value stamped on a non-wire snapshot for tests.

Poll (`_notification_polling.py`), finalize (`_loading_finalize.py`), and
`_reconcile_unread_from_cached_notifications` pass that generation into the reconcile.
Delete the read-sequence helpers once nothing calls them.

## Unread reconcile and the ack pipeline

`_apply_reconciled_unread_from_completion_notifications` keeps its roster, manual-mark,
and pending-overlay behavior. Its notification input becomes lean index rows:

- Add `unread_completion_index_rows_from_notifications` next to the existing predicates
  in `_notification_matching.py`. Same inclusion rules as the Rust index, including
  dismissed rows and literal (not timestamp-normalized) suffixes.
- Active keys for `projection_has_active_completion` are the non-dismissed rows'
  `(cl_name, raw_suffix)`. A completion with no suffix is `(cl_name, None)`, which
  preserves today's `cl_name`-only fallback. Settlement rows contribute only their exact
  pair.
- Callers that already parsed a snapshot (poll, async count refresh, cached reconcile,
  finalize) project those in-memory notifications. They do not start a second store
  read.
- A reconcile that needs disk and has no snapshot calls `read_unread_completion_index`
  from the ack worker thread or `asyncio.to_thread` / the existing pump-free refresh,
  then applies the rows on the UI thread with `call_from_thread`. Do not call it on the
  UI thread. Do not add a proc.

The post-ack path already schedules `_schedule_notification_snapshot_refresh` so the
indicator and plan-lifecycle disappearance logic still run. Keep that schedule. That
refresh's unread reconcile is an applied index because the snapshot now carries
`generation` and is projected to lean rows. Do not add a second full-store parse on the
ack path just to obtain the index. The index disk API is for the no-snapshot case and
for the parity test below.

`_unread_ack_writer.py`:

- One batch still makes one Rust call.
- Call `ack_agent_completions` with the combined keys, including an empty key list (the
  call returns the current generation and writes nothing).
- Delete the cache pre-scan as the source of ids. Remove `outcome.dismissed_ids` from
  the cached snapshot. Pass that id set through the existing per-op completion so each
  op still records the generation and the coalesced resync count stays as it is today
  (five rapid acks still complete five ops and issue one write).
- On exception, do not record a generation. Each op restores only the identities it
  still owns.
- The Rust call stays inside the worker thread. Completion on the UI thread does not
  read the store. `test_ack_completion_never_reads_store_on_ui_thread` must stay green.

Retarget test doubles that stub the ack writer. Patch
`sase.notifications.ack_agent_completions` so a drain cannot touch disk. Return an
object with `dismissed_ids` and `generation` (a `SimpleNamespace` is enough). Assert
call counts on that mock. Leave patches that protect `_marking.py` / row-dismissal on
`dismiss_agent_completion_notifications_matching_agents`. Files that drive unread ack
today and will need the new patch include `test_pending_ack_fence.py`,
`test_unread_ack_pipeline.py`, `test_agent_unread_jump_fast_path.py`,
`test_agent_bulk_ack_scope_undo.py`, `test_agent_panel_entry_unread.py`,
`test_agent_unread_done_navigation.py`, `test_agent_unread_done_navigation_folds.py`,
`test_agent_unread_done_navigation_panels.py`, `test_agent_unread_selection.py`,
`test_agent_unread_toggle.py`, `test_agent_marking_toggle.py`,
`test_agent_panel_first_selection.py`, `test_jump_hints_for_folded_banners_dispatch.py`,
and `bench_tui_jk_unread.py`. `test_agent_marking_save.py` stays on the old function if
it is exercising marked-group save rather than the ack writer.

Do not grow `src/sase/ace/tui/actions/agents/_display.py`, `_loading_finalize.py`,
`_display_panel_collection.py`, or `tests/ace/tui/test_leader_keymap_dispatch.py`. Put
new projection helpers in the matching module, which is not on that list.

## Tests

`sase-core`, new `crates/sase_core/tests/notification_store_parity/ack_index.rs`
(register it in that directory's `main.rs`):

- Append, rewrite, and a real state update each bump the generation by one. A no-op
  state update and a no-op reconcile do not.
- The generation file survives reopening the path.
- Ack dismisses one completion (including an empty-suffix completion matched by
  `cl_name` only) and one settlement, and does not dismiss a settlement on `cl_name`
  only or an unrelated row. Returned ids are exactly the newly dismissed ones.
  Generation bumps once.
- A repeat ack returns empty ids and the same generation.
- The index rows equal the completion and settlement rows still in the live file,
  dismissed included, with the post-ack generation.
- An index read and an ack in two threads only observe generations that are monotonic
  with the rows: an index generation greater than or equal to an ack's returned
  generation shows that ack's ids dismissed.

Python, new `tests/notification_store/test_unread_completion_index.py`:

- Build one fixture through the public store API. The Rust index and
  `unread_completion_index_rows_from_notifications` on a full snapshot of that file
  return the same ids, keys, flags, and generation.

TUI:

- Pre-write reconcile (worker not finished, `done_generation is None`) still keeps the
  optimistic clear.
- Cache rejects a snapshot whose generation is older than the cached one.
- Ack completion removes the returned ids even when the cached snapshot's key scan would
  have chosen a different subset, and it does not remove an id the ack did not return.
- Five rapid acks still coalesce to one `ack_agent_completions` call.
- A failed batch still restores only the op's still-owned identities.

Update `tests/notification_store/test_storage.py` only if a new assertion is needed for
the generation on an existing dismiss. Do not change dismiss behavior.

## Verification

In `sase-core` (`sase repo open` already done this session; use that checkout):
`sase tool run check`. It is the CI gate and takes several minutes; give it a long
timeout. `just test -p sase_core ack_index` and the py binding tests are the inner loop
only. Do not run bare `cargo` or bare `just check`.

In sase: `sase tool run check`. Targeted pytest for
`tests/notification_store/test_unread_completion_index.py`,
`tests/ace/tui/test_pending_ack_fence.py`, and
`tests/ace/tui/test_unread_ack_pipeline.py` while iterating.

No rendered TUI output changes, so do not run `just fix-tui-screenshots`. No new `just`
recipe and no `--epic-symbol`. Before closing the bead, run
`sase bead epic-symbols sase-1d7.12`. If a symbol is still keyed to this phase, re-key
that Justfile line to the parent epic `sase-1d7` or to a later open phase. Close only
`sase-1d7.12`. Do not close `sase-1d7` or any ancestor. Record discovered work as
`sase bead note sase-1d7.12 'PROPOSED FOLLOW-UP: …'`. Do not create beads. A check
failure that reproduces on the clean base tree is a follow-up note, not a reason to
leave this phase open.

## Out of scope

- Notification retention and `wait_checks` diet (`sase-1d7.13`).
- Moving the agent-node unread projection into Rust.
- Replacing `dismiss_agent_completion_notifications_matching_agents` for marked-group
  save or row dismissal.
- A proc, SQLite, a warm helper, a feature flag, or a keymap change.
- A schema-version bump.
