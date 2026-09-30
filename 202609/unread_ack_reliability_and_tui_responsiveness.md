---
tier: epic
title: Make unread acks stick and keep the Agents TUI responsive
goal: "Unread acknowledgments (`,u`, `,j`/`,J`, row-select) are never reverted by
  another notification-store writer or by an older snapshot, and unread actions paint
  within budget: no UI-thread store reads, no full Agents rebuilds for unread-only
  changes, and no multi-second main-loop freezes from the 1 Hz runtime tick or fleet
  reprojection.

  "
phases:
  - id: reconciler-delta-write
    title: Remote-attention reconciler writes only the rows it changed
    depends_on: []
    size: small
    description: "reconciler-delta-write: stop reconcile_remote_attention_inbox from
      handing every store row back to rewrite_notifications, and ignore per-poll
      observed_at churn in change detection, with a lost-update interleave regression
      test.

      "
  - id: core-reconcile-upsert
    title: Atomic field-scoped reconcile write in sase-core
    depends_on:
      - reconciler-delta-write
    size: medium
    description: "core-reconcile-upsert: add a lock-held, field-scoped notification
      reconcile write plus empty raw_suffix matcher parity to sase_core, bind it with
      the GIL released, switch the attention reconciler to it, and move the core pin.

      "
  - id: unread-instrumentation
    title: Trace spans, leader-key perf capture, and unread/idle benches
    depends_on: []
    size: small
    description: "unread-instrumentation: add tui_trace spans and SASE_TUI_PERF
      key-to-paint capture for leader unread keys, plus unread and idle-tick bench
      scenarios that record the baseline later phases compare against.

      "
  - id: roster-generation
    title: Roster generation counter and cached projection index
    depends_on:
      - unread-instrumentation
    size: medium
    description: "roster-generation: introduce one app-wide roster generation bumped on
      every roster assignment and in-place status mutation, cache
      agent_node_projection_index per generation, and remove its quadratic dedupe.

      "
  - id: pending-ack-fence
    title: Sequence-fenced pending-ack overlay and monotonic snapshot cache
    depends_on:
      - roster-generation
    size: medium
    description: "pending-ack-fence: stamp snapshot reads with a read sequence, keep
      in-flight acks as a pending overlay every reconcile path honors, reject stale
      snapshots in the cache, narrow failure restore to owned identities, and stop
      re-confirmations from invalidating undo.

      "
  - id: unread-chrome-helper
    title: One batched unread chrome helper with no full rebuilds
    depends_on:
      - pending-ack-fence
    size: medium
    description: "unread-chrome-helper: route every unread change through one helper
      that patches only visible changed rows via an identity map, skips collapsed
      panels, never falls back to a full rebuild, and refreshes titles, info panel,
      machine chip/tab strip and tribe summary once each.

      "
  - id: bulk-ack-scope-and-undo
    title: Precise bulk-ack scope and a time-bound explicit undo
    depends_on:
      - unread-chrome-helper
    size: small
    description: "bulk-ack-scope-and-undo: make the bulk-ack target set share one
      predicate with the header unread count across tabs, collapsed clans and tribes,
      and replace the silent toggle with a toast-announced 10 second undo window.

      "
  - id: ack-pipeline
    title: Read-free ack completion and a coalescing ack writer
    depends_on:
      - pending-ack-fence
      - unread-chrome-helper
      - bulk-ack-scope-and-undo
    size: medium
    description: "ack-pipeline: delete the synchronous post-ack store read, apply ack
      outcomes to the cached snapshot by id, schedule only the guarded async resync, and
      drain queued acks through one coalescing worker that issues one Rust call per
      batch.

      "
  - id: unread-jump-fast-path
    title: Cheap unread jumps and footer probe
    depends_on:
      - roster-generation
      - unread-chrome-helper
      - ack-pipeline
    size: medium
    description: "unread-jump-fast-path: drop the unconditional trailing tab refresh
      from the unread jump keys, keep detail behind the debounce, reveal only the target
      panel, make the footer probe O(1), and key the jump-candidate cache by generations
      with remove-on-ack.

      "
  - id: runtime-tick-caches
    title: Cached wait-status maps and change-only runtime patching
    depends_on:
      - unread-instrumentation
      - roster-generation
    size: medium
    description: "runtime-tick-caches: cache collect_agent_wait_status_maps per roster
      generation, patch runtime rows only when their rendered runtime text changes, and
      cache clan runtime aggregation so the 1 Hz tick stops freezing the loop.

      "
  - id: fleet-signature-cheap
    title: Cheap fleet reprojection signature computed before projection
    depends_on:
      - unread-instrumentation
      - roster-generation
    size: medium
    description: "fleet-signature-cheap: replace the recursive deep-freeze projection
      signature with a structural signature from the roster generation and fleet wire
      revisions, checked before project_clan_tree runs, with host-level freshness fields
      patched in the header only.

      "
  - id: core-unread-ack-index
    title: Rust ack API, lean unread index, and store generations
    depends_on:
      - core-reconcile-upsert
      - pending-ack-fence
      - ack-pipeline
      - unread-jump-fast-path
    size: large
    description: "core-unread-ack-index: move completion acks and the unread completion
      index into sase_core as lean GIL-released calls that return dismissed ids and a
      store generation, then replace the Python read-sequence fence with store
      generations.

      "
  - id: notification-store-diet
    title: Notification store retention and wait-check payload diet
    depends_on:
      - core-reconcile-upsert
      - core-unread-ack-index
    size: large
    description:
      "notification-store-diet: shorten live retention of dismissed rows and bound
      wait_checks plus_ones and action_data so every full read, rewrite and lock window
      scales with a much smaller store."
proposed_by: bbugyi200.athena.0uc
create_time: 2026-09-30 07:18:03
status: wip
---

- **PROMPT:**
  [prompts/202609/unread_ack_reliability_and_tui_responsiveness.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/unread_ack_reliability_and_tui_responsiveness.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                                                            | Why                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| derives-from | [research:202609/unread_ack_reliability_and_tui_responsiveness/unread_ack_reliability_and_tui_responsiveness.md][1] | Consolidated research whose Phase 0-3 recommendations this epic implements |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202609/unread_ack_reliability_and_tui_responsiveness/unread_ack_reliability_and_tui_responsiveness.md

<!-- sase:links:end -->

# Plan: Make unread acks stick and keep the Agents TUI responsive

## Background

The user pressed `,u` (mark every unread completed agent read) on the Agents tab and the
27 unread markers stayed: header `27 unread`, machine chip `athena 204 F5 U27`, `@epic`
`U21` (unread hidden inside collapsed clans of an expanded tribe) and collapsed `@job`
`U6`. Unread indicators have felt flaky and always slow, including `,j` (jump to next
unread).

The linked research report establishes, and code reading for this plan confirms:

1. **`,u` worked, and then the notification store was reverted.** The optimistic clear
   ran and the background write succeeded, but `reconcile_remote_attention_inbox`
   (`src/sase/dispatch/attention_inbox.py`) loads the whole store with
   `load_notifications(include_dismissed=True)` outside the store lock, copies every row
   into `output`, and calls `rewrite_notifications(output)`. In `sase-core`,
   `merge_and_rewrite_notifications_unlocked`
   (`crates/sase_core/src/notifications/store.rs`) lets the caller's row win on id
   collision, so any dismissal committed between that load and write is silently undone.
   It runs about every 60 s from the TUI (`_remote_attention.py`, via
   `asyncio.to_thread`) and rewrites nearly every time because `_coerce_pending_entry`
   stamps the per-poll `observed_at_unix` into `action_data`
   (`remote_attention_entry_json`), so `refreshed != existing` on almost every poll.
2. **TUI-side flakiness is real but secondary.** Three paths re-project unread from a
   snapshot with no knowledge of in-flight acks (poll `_poll_agent_completions_once`,
   finalize `_sync_unread_completed_agents` in `_loading_finalize.py`, and
   `_reconcile_unread_from_cached_notifications`); `_set_notification_snapshot_cache`
   accepts stale snapshots; the unread patch path never refreshes the machine chip / tab
   strip (`_update_agents_header` in `_fleet_header.py`); and an empty `raw_suffix`
   matches differently in Python (`_notification_matching.py`, `or None`) and Rust
   (`matches_agent_completion_notification_for_agents`, literal `""`).
3. **Most of the slowness is not unread code.** The live watchdog logged about 20
   minutes frozen in 4.1 hours (358 hitches, p50 3.4 s). The top sources are the 1 Hz
   countdown tick (`_patch_agent_runtime_rows` → `patch_row`, which recomputes the
   uncached whole-roster `collect_agent_wait_status_maps` for every clan container) and
   fleet reprojection (`_agents_projection_signature` deep-freezes every field of every
   agent after `project_clan_tree` already ran). Unread paths add their own costs: a
   synchronous full-store read on the UI thread after every ack
   (`_complete_unread_notification_dismissal` → `_refresh_notification_count`), an
   O(K·N) per-row patch loop (`self._agents.index(agent)`, per-row panel title and info
   panel refresh), a full Agents rebuild whenever a changed row sits in a collapsed
   panel (`render_collapsed` empties `widget._agents`, so `_try_patch_agent_row` bails
   and the caller calls `_refresh_agents_display(list_changed=True)`), an undebounced
   trailing `_refresh_current_tab()` on every `,j`/`,J`, an O(N) jump-candidate cache
   key, and `agent_node_projection_index` rebuilt about four times per action with an
   O(owned²) dedupe.

### Why not a proc for the expensive work

The instinct (nothing slow on the UI thread) is right and matches `tui_perf.md`, but a
proc is the wrong mechanism for acks, and this plan does not use one:

- The store write already runs in a Textual thread worker (`agents-unread-ack`). What
  blocks the UI is the work around it (the synchronous post-ack read, per-row patching,
  full rebuilds) and unrelated periodic freezes. A proc's result still has to be applied
  to widgets on the UI thread.
- Procs cost about 1 s end-to-end (a recorded `sase notify apply-state-many` took 2.6
  s). A slower write widens both the store lost-update window and the TUI's optimistic
  window, which are the races that broke `,u`.
- Notification-state procs share one concurrency key that rejects overlapping submits
  (rapid `,j` would be refused), and the default durable-proc completion reloads Agents.

The placement rule this epic follows instead: nothing data-scaled runs on the UI thread
or the Textual pump; I/O and parsing go to in-process workers, `spawn_pump_free_task`,
or Rust with the GIL released; paint, selection and chrome stay synchronous and cheap;
procs are only for durable, audit-worthy, or long-running operations. Moving the
remote-attention inventory sync into a service proc is explicitly deferred (see
Non-goals).

## Requirements every phase honors

- **R1: correctness before speed.** An acknowledgment must never be reverted by another
  writer or an older snapshot; only a genuinely new completion may re-mark a node
  unread. No notification-store writer may read the whole store and write it back.
- **R2: placement rule** as stated above.
- **R4: two latency budgets.** First paint (key → optimistic paint): UI-thread work ≤ 16
  ms p95 for `,u` and for a `,j` to a visible row, ≤ 50 ms for a `,j` that must reveal a
  collapsed panel or clan. Settlement (persist + authoritative reconcile): asynchronous,
  no pump callback over 16 ms, typically under 1 s. Zero synchronous store reads in any
  ack path.
- **R7: no feature flag.** This is a correctness bug and a hitch fix on the default tab;
  every change lands unconditionally.

## Cross-phase rules

- Read the TUI perf rules before any TUI phase:
  `sase memory read tui.md tui_perf.md -r "<why>"`. In particular: rule 2 (off the event
  loop is not off the pump), rule 4 (re-capture selection/tab after every await), rule 6
  (selective patches over rebuilds), rule 7 (debounce detail, never the highlight).
- Shared backend behavior belongs in `sase_core`. Open the linked repo with
  `sase repo open sase-core -r "<why>"`, read its `AGENTS.md` (binding recipe: wire
  types
  - tests in `sase_core`, `#[pyfunction]` in the domain's `sase_core_py` module with
    `py.allow_threads`, registration in `register_notifications`, a round-trip test in
    the domain `tests.rs`), and never edit core versions or `CHANGELOG.md`. On the sase
    side, call bindings only through `require_rust_binding("<string literal>")` in
    `src/sase/core/notification_store_facade.py` so `tools/check_sase_core_rs_bindings`
    can see them; move `sase-core-revision.txt` past the core commit per
    `docs/rust_backend.md` (a declaration that commits both repos gets the pin written
    by the host). Do not bump the `sase-core-rs` range in `pyproject.toml`.
- Session-local presentation state (optimistic unread overlay, chrome patching, jump
  cursor) stays in Python; store semantics, key matching and the unread completion index
  belong in Rust (final phases).
- Line-count gate: `src/sase/ace/tui/actions/agents/_display.py` is exactly at the
  700-line info limit and must not grow; `_loading_finalize.py` (611),
  `_display_panel_collection.py` (602) and
  `tests/ace/tui/test_leader_keymap_dispatch.py` (612) are close. Put new logic in new
  focused modules rather than growing these.
- Every phase keeps existing tests green, updates tests that pin the old behavior, and
  runs `sase tool run check` in each repo it changed (in `sase-core` too). A phase that
  changes rendered TUI output also runs `just fix-tui-screenshots` and inspects the
  report.
- Phases that change a hot path record before/after numbers from the
  `unread-instrumentation` benches in their bead notes.

## Phase `reconciler-delta-write`

Smallest change that stops the durable revert of completion rows. Only
`src/sase/dispatch/attention_inbox.py` and its tests change.

1. In `reconcile_remote_attention_inbox`, track the rows this call created, refreshed,
   or auto-dismissed, and pass only those rows to `rewrite_notifications`. The Rust
   merge keeps every row absent from its input, so completion notifications and every
   other untouched row can no longer be overwritten with a stale copy. Keep `outcome`
   counts and `outcome.changed` semantics unchanged for the TUI caller.
2. Exclude volatile per-poll inventory fields from change detection. Compare `refreshed`
   with `existing` after removing `observed_at_unix` from the decoded
   `remote_attention_entry_json`, and when only volatile fields differ keep `existing`
   untouched (not written, not counted as updated). Hold the volatile key set in one
   named module constant; inspect the inventory entry payload for any other per-poll
   noise (for example cache ages) and include it if it has the same property. Confirm
   the remote-attention modal does not depend on a fresh `observed_at_unix`; a slightly
   stale display-only timestamp is acceptable.
3. Add a comment at the write naming the residual risk this phase leaves: a user
   dismissal of a remote-attention row that the same poll also refreshed can still be
   clobbered; `core-reconcile-upsert` closes it.

Tests (`tests/test_dispatch_attention_inbox.py`):

- Lost-update interleave: seed an undismissed agent-completion notification plus
  remote-attention rows; wrap the reconciler's `load_notifications` so it returns the
  real snapshot and then dismisses the completion row on disk before returning; run a
  reconcile that creates or updates a remote-attention row; assert the completion row is
  still dismissed on disk.
- Two consecutive polls that differ only in `observed_at_unix` report `changed == False`
  on the second and never call `rewrite_notifications` (spy).
- The existing reconciler tests stay green (adjust any that assumed full-list order).

## Phase `core-reconcile-upsert`

Close the lost-update class for the reconciler with an atomic, field-scoped write.

`sase-core` (`crates/sase_core/src/notifications/`):

1. Add a lock-held reconcile write, for example
   `reconcile_notification_rows(path, request) -> counts`, whose request wire carries
   the input rows, the closed set of fields the caller owns, and an optional
   reversible-dismiss marker key (the Python constant
   `REMOTE_ATTENTION_AUTO_DISMISSED_ACTION_DATA_KEY` is passed in, not hardcoded). Under
   the exclusive store lock it re-reads the file and, per input row:
   - absent on disk → append it (created);
   - present → start from the on-disk row and copy only the owned fields (icon, color,
     notes, tags, action, action_data, silent). `read`/`dismissed` come from disk,
     except: (a) resurface when the on-disk row is dismissed and its action_data carries
     the marker `"true"` while the input row is not dismissed → `read=false`,
     `dismissed=false`; (b) auto-dismiss when the input row is dismissed with the marker
     and the on-disk row is not dismissed → `dismissed=true` with the marker. If the
     on-disk row is already dismissed without the marker (a user dismissal), it stays
     dismissed and the marker is never added, so a user dismissal never becomes
     reversible;
   - `id`, `timestamp`, `sender`, `files`, `muted`, `snooze_until`, `resurfaced_at`,
     `plus_ones`, `plus_ones_dropped` and `dedup_key` always come from disk.

   Rows absent from the input are untouched; skip the file write when nothing changed;
   return created / updated / dismissed / resurfaced counts. These semantics must match
   the documented intent of `_refresh_existing_notification` in `attention_inbox.py`.

2. Empty `raw_suffix` parity: in `matches_agent_completion_notification_for_agents` and
   `matches_agent_notification`, treat a notification `raw_suffix` of `""` exactly like
   a missing key (match on `cl_name` alone), the way Python's
   `agent_completion_notification_matches_agent` already does.
3. Tests in `crates/sase_core/tests/notification_store_parity/`: a lost-update
   interleave (snapshot, dismiss a completion row and a remote row, reconcile with the
   stale rows → both stay dismissed), resurface of a marker row, no marker on a
   user-dismissed row, owned-field-only refresh, no write when unchanged, and the
   empty-`raw_suffix` match. Add the binding (GIL released) with a round-trip test in
   `crates/sase_core_py/src/notifications/tests.rs`.

`sase`:

4. Add the facade wrapper and a public `src/sase/notifications/store.py` function, and
   switch `reconcile_remote_attention_inbox` to it (send created, refreshed and
   auto-dismissed rows only). Python keeps using `_refresh_existing_notification` only
   for change detection, if at all.
5. `rewrite_notifications` then has no production caller; add a docstring warning that
   it must never be used for read-modify-write reconciliation.
6. Python integration tests: the reconciler refreshing a remote-attention row that the
   user dismissed between load and write leaves it dismissed; a completion row with
   `raw_suffix: ""` is dismissed on disk by
   `dismiss_agent_completion_notifications_matching_agents` and removed by the Python
   cache-removal predicate for the same agent.
7. Move `sase-core-revision.txt` past the core commit.

## Phase `unread-instrumentation`

Measure before changing hot paths.

1. `tui_trace` spans (`src/sase/ace/tui/util/trace.py`, `tui_trace(span, **counters)`):
   `leader.unread_bulk_ack`, `leader.unread_jump`, `unread.ack_complete`,
   `unread.reconcile`, and `agent_nodes.projection_index`, with workload counters
   (loaded agents, targets, collapsed-panel and off-tab targets, rows patched, full
   rebuilds). Store size comes from the worker thread, never from a UI-thread stat.
2. Extend the `SASE_TUI_PERF` key-to-paint JSONL (`~/.sase/perf/tui_jk.jsonl`) to leader
   keys `,u`, `,j` and `,J`, labelled by key.
3. Benches: a new `tests/ace/tui/bench_tui_jk_unread.py`, re-exported from
   `tests/ace/tui/bench_tui_jk.py`, covering `,u` and `,j` for four branches: target
   visible, inside a collapsed panel, inside a collapsed clan, and on another Agents
   tab, at a roster shaped like the screenshot (about 200 agents, clans, three tribes).
4. Idle scenario in `tests/perf/bench_tui_trace.py` (implementation under
   `tests/perf/tui_trace/`) that times one 1 Hz countdown tick
   (`_patch_agent_runtime_rows`) and one `fleet_refresh` reprojection
   (`_reproject_agents_from_current_mode`) at the same roster size. The j/k benches
   cannot see these by design: the tick and fleet apply are skipped while navigating.
5. Add the capture recipe to `docs/perf_runbook.md`; record baseline p50/p95/max in the
   phase bead note.

## Phase `roster-generation`

Shared cache key for the later phases.

1. Add an app-wide `_agents_roster_generation: int` (initialised in
   `actions/_state_init_agents.py`) that bumps whenever roster content feeding
   projections changes: every assignment of `_agents` / `_agents_with_children` (today
   about a dozen sites, including `_loading_apply.py`, `_loading_filter.py`,
   `_fleet_projection.py`, `_kill_identity.py`, `_dismiss_memory.py`,
   `_named_proc_dismiss.py`, `_proc_action_completion.py`, `_loading_finalize.py`) and
   in-place status mutation (`_loading_helpers.py`, `_wait_actions.py`,
   `_loading_finalize.py`). Route assignments through one setter helper (or a property)
   so new sites cannot forget, and add a guard test that fails when a raw
   `self._agents_with_children =` / `self._agents =` assignment appears outside it.
   Agents are mutated in place, so list identity alone is not a safe cache key.
2. Cache `agent_node_projection_index` (`src/sase/ace/tui/models/agent_nodes.py`) per
   (roster generation, input roster identity) for UI-thread callers (`_unread_state.py`,
   `_notification_unread_projection.py`, `_loading_finalize.py`). The worker-thread
   caller in `_notification_completion_arrival.py` keeps computing its own index.
3. Replace the O(owned²) `all(row.identity != agent.identity ...)` dedupe with a seen
   identity set that preserves order.

Tests: `,u` builds the projection index once instead of about four times; the cache
invalidates on roster assignment and on in-place status mutation; dedupe results and
order are unchanged (`tests/ace/tui/models/test_agent_nodes.py`).

## Phase `pending-ack-fence`

Make the TUI never resurrect an acknowledged node from an older snapshot. Files:
`_notification_provider.py`, `_notification_polling.py`,
`_notification_unread_projection.py`, `_unread_state.py`, the finalize unread sync, and
`_state_init_agents.py`.

1. **Read sequence.** Keep `_notif_read_seq: int`. Every snapshot read (the guarded
   async read `_read_notification_snapshot_guarded` and the provider read) captures the
   sequence value at the moment the read starts and carries it with its result.
2. **Monotonic cache.** `_set_notification_snapshot_cache` ignores a snapshot whose
   read-start sequence is older than the cached one's. Reuse or replace the unused
   `_notification_snapshot_version`.
3. **Pending overlay.** Each ack (bulk `,u`, single `,j`/row-select via
   `_clear_agent_unread_and_dismiss_notification`) registers
   `pending[identity] = (op_id, done_seq=None)` before scheduling its worker. When the
   worker succeeds, record `done_seq` as the current read sequence. Every reconcile path
   (`_reconcile_unread_from_completion_notifications` from the poll, the finalize sync,
   and the cached reconcile) treats pending identities as read. An entry retires only
   after applying a snapshot whose read-start sequence is greater than its `done_seq` (a
   read that began after the write landed). A genuinely new completion for the same
   identity therefore resurfaces at most one poll later, which is acceptable.
4. **Narrow restore.** On write failure, `_restore_unread_notification_dismissal`
   restores only identities whose pending entry this op still owns; a later op on the
   same identity takes ownership.
5. **Undo stability.** A reconcile that only re-confirms pending identities must not
   call `_invalidate_bulk_read_undo`; only genuinely new unread identities invalidate
   it.

Tests: a poll applying a pre-write snapshot while an ack is pending keeps the chrome
cleared; the same through finalize; a stale snapshot landing after a fresh one is
ignored; the pending entry retires only after a post-write read; a failed write restores
only owned identities; re-confirmation keeps undo armed while a genuinely new unread
still invalidates it.

## Phase `unread-chrome-helper`

One way to paint unread changes, used by bulk mark, single ack, undo restore, failure
rollback, and every reconcile path. It replaces `_patch_unread_completed_agent_changes`
plus the `_repaint_changed_unread_rows` compatibility fallback.

1. Compute `changed = before ^ after`, keep the existing ancestor-aware expansion to
   clan containers, and resolve rows through an identity → (panel widget, local index)
   map built with the panel index and invalidated with `_invalidate_agent_panel_cache`,
   instead of `self._agents.index(agent)` (also use the map inside
   `_try_patch_agent_row`).
2. Skip rows in collapsed panels (their `widget._agents` is empty) and rows on other
   tabs: only counts change there. Never call
   `_refresh_agents_display(list_changed=True)` for an unread-only diff; if a
   visible-row patch fails for a real reason (such as width growth), rebuild only that
   panel, adding a per-panel rebuild entry point if none exists.
3. Per-row patches pass `refresh_info=False` and skip per-row panel titles. After the
   loop refresh each affected panel title once, `_update_agents_info_panel()` once,
   `_update_agents_header()` (machine chip and tab strip) once, and
   `_refresh_tribe_summary_only()` once, so header, machine chip, tab strip and tribe
   titles change on the same paint. This closes the `,u` success-branch chrome gap.
4. Emit the `unread.chrome_apply` span with changed, visible-patched, collapsed-skipped
   and panel-rebuild counters.

Tests use the screenshot shape (unread inside collapsed clans of an expanded tribe, a
collapsed tribe panel, off-tab rows): after `,u` the header, machine chip and every
tribe title reach 0 on the same paint, no panel expands, no full display rebuild runs,
and patched rows are bounded by visible changed rows. Update
`test_agent_unread_projection.py`, `test_agent_unread_toggle.py`, and helpers that stub
`_try_patch_agent_row`.

## Phase `bulk-ack-scope-and-undo`

1. "All" for `,u` means every loaded unread terminal agent node across all Agents tabs,
   including collapsed clans and tribes and off-tab query rows. Extract one predicate /
   target function shared by `_toggle_all_unread_done_agents_read` and the header unread
   count (`_agent_info_metrics` / `sase_agent_status_counts`) so the toast count always
   equals the header count. `,u` never expands a panel.
2. Replace the silent toggle with an explicit, time-bound undo: after marking, the toast
   reads like `Marked 27 completed agents read · press ,u within 10s to undo`, rendering
   the configured leader key rather than a hardcoded one. The window is a named 10 s
   constant. Within it, `,u` restores the same identities and says so; after it, `,u`
   with nothing unread is a plain NOOP ("No unread completed agents"). Keep the
   restore's existing session-local semantics.
3. Update any user-facing help text for the action; the keymap itself does not change.

Tests: toast count equals header count for mixed collapsed/off-tab rosters; undo inside
the window restores the same N; after the window (monkeypatched clock) it does not. Put
new tests in a new module if `test_leader_keymap_dispatch.py` would pass 700 lines.

## Phase `ack-pipeline`

1. `_complete_unread_notification_dismissal` never reads the store on the UI thread.
   Delete the synchronous `_refresh_notification_count()` call. The worker computes the
   ids of the cached snapshot's matching notifications off-thread before writing and
   returns them; completion removes those ids from the cached snapshot by id set, not by
   the notifications × keys scan in `_remove_agent_completion_notifications_from_cache`.
   If a resync is wanted, schedule the guarded, coalesced
   `_refresh_notification_count_async`, never the synchronous variant.
2. Coalescing ack writer: acks enqueue `(op_id, keys, identities)`; one worker drains
   the queue and issues one `dismiss_agent_completion_notifications_matching_agents`
   call per batch, handing per-op outcomes back through `call_from_thread`. Keep the
   `agents` worker group, `exit_on_error=False`, and teardown cancellation. Each op in a
   failed batch restores only its own still-owned identities (the fence's ownership),
   and `done_seq` is recorded per op when its batch lands.
3. Completion paints through the chrome helper.

Tests: a guard test fails if `_read_notification_snapshot_from_provider` runs on the
main thread during an ack; five rapid acks coalesce into at most two writes; a failed
batch restores only its ops' owned identities; cache removal is by id.

## Phase `unread-jump-fast-path`

1. In `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`, `,j` and `,J` drop the
   unconditional trailing `_refresh_current_tab()`. On a miss they only toast; on a hit
   the jump's own refresh paints highlight and chips immediately and the detail pane
   stays behind `DetailPanelDebouncer` (150 ms). Update the tests that pin refresh
   counts (`test_agent_unread_done_navigation.py`, `test_leader_keymap_dispatch.py`).
2. A jump that must reveal a collapsed panel or clan rebuilds only that panel.
3. The footer probe `_has_unread_completed_agent` becomes an O(1) cached boolean keyed
   by generations; it never builds jump candidates.
4. Key the `_unread_timed_jump_candidates` cache by generations (roster generation, an
   unread-set generation bumped on every unread-set mutation, fold generation, active
   tab and committed query) instead of the O(N) status tuple plus
   `frozenset(unread_ids)`. When `,j` acknowledges its target, remove that identity from
   the cached ordered list and advance the cached key in step, so the next `,j` hits the
   cache. A fully incremental navigation index is a follow-up only if traces still show
   cost.

Tests: consecutive `,j` presses reuse the cached list; the footer probe builds no
candidates; `,j` into a collapsed tribe rebuilds only that panel; detail render is
deferred; a `,j` to a visible row does no artifact-directory stat on that tick.

## Phase `runtime-tick-caches`

1. Cache `collect_agent_wait_status_maps` (`src/sase/ace/tui/_agent_completion_wait.py`)
   app-wide, keyed by the roster generation plus any other state it reads (for example a
   tribe-assignment generation). `agent_wait_status_maps_for_app` and
   `agent_wait_status_maps_for_build` return the cached maps, so `patch_row` for clan
   containers stops rescanning the whole roster.
2. Change-only runtime patching: `AgentList.patch_active_runtime_rows` patches a row
   only when its rendered runtime text actually changed, and a runtime-only change must
   not invalidate the whole row render cache entry when the suffix can be patched alone.
3. Cache per-clan runtime aggregation (`_aggregate_runtime`, which calls the Rust
   `aggregate_clan_runtime`) keyed by roster generation and member runtime inputs,
   recomputing only the `now`-dependent part per tick.

Acceptance: the idle bench's 1 Hz tick at the screenshot roster stays under 16 ms on the
UI thread; existing runtime patching, clan rendering and unknown-wait tests stay green;
caches invalidate on roster assignment and status mutation.

## Phase `fleet-signature-cheap`

1. Replace `_freeze_projection_value` / `_agents_projection_signature` in
   `src/sase/ace/tui/actions/agents/_fleet_projection.py` with a structural signature
   built from the local roster generation, per-row `fleet_revision` /
   `fleet_row_revision`, and the host snapshot identity already parsed in
   `_fleet_agents_payload.py` (`_snapshot_identity`; carry it on the projection result).
   Compute and compare it before `_agents_source_for_current_mode` runs
   `project_clan_tree`, so an unchanged refresh skips the projection entirely.
2. Host-level per-refresh fields (observed time, cache age, host counts,
   freshness/health, attention) update the header or banner through the cheap patch path
   without reprojecting. Verify which of them rows render; any that a row renders must
   either join the signature or be patched on that row.
3. Forced sources (`remote_attention`, `remote_mutation`) still force a reprojection.

Tests (`tests/ace/tui/test_agents_fleet_refresh_laziness.py`): an unchanged payload
skips both the signature deep walk and `project_clan_tree`; a revision bump repaints;
host-freshness-only changes update the banner without reprojecting; forced sources still
reproject; a local roster change reprojects. Record idle-bench before/after.

## Phase `core-unread-ack-index`

`large`: the worker plans before implementing. Scope:

1. In `sase_core`: `ack_agent_completions(keys) → {dismissed_ids, generation}` and a
   lean
   `read_unread_completion_index() → {generation, rows: [id, agent key, read, dismissed]}`,
   both GIL-released, backed by a store generation that bumps on every store write.
   Decide whether the "which agent nodes are unread" projection also moves into Rust
   (the Rust-core litmus test says other frontends need the same counts), and record the
   decision in the phase plan.
2. In `sase`: the ack pipeline calls `ack_agent_completions` and removes returned ids;
   unread reconciles use the lean index instead of a full snapshot parse where only
   unread state is needed; store generations replace the Python read-sequence fence (a
   pending entry retires once an applied index generation reaches the ack's returned
   generation). Move the core pin.

## Phase `notification-store-diet`

`large`: the worker plans before implementing. Every full read, rewrite and lock window
scales with the live store (about 19 MB and 3,900 rows, 12.7 MB of them dismissed rows
still inside the 14-day archive threshold and 4.7 MB of the active rows `wait_checks`).
Compaction alone frees nothing today.

1. Shorten live retention of dismissed rows in the core compaction policy
   (`maybe_compact_notifications_unlocked` / `notification_should_archive`); archived
   rows stay in the archive file. Choose the threshold from measured data, and make it a
   config value only if users should tune it.
2. Bound `wait_checks` payloads: cap `plus_ones` (with `plus_ones_dropped` accounting)
   and slim `action_data` from the chop wait-check producers
   (`src/sase/scripts/sase_chop_wait_checks.py` and `_chop_wait_checks_*.py`) without
   losing what the notification UI renders.
3. Measure store size before and after on a copy of a realistic store and record it.

## Acceptance checks for the epic

| Action                                                                                     | Pass                                                                                                                            |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| `,u` with 21 unread inside collapsed clans of an expanded tribe and 6 in a collapsed tribe | Header, machine chip and tribe titles go from 27 to 0 on the same paint; no tribe expands; no full rebuild; UI work ≤ 16 ms p95 |
| Poll or finalize 50 ms later with a pre-write snapshot                                     | Chrome stays cleared                                                                                                            |
| Attention reconciler runs between an ack's write and the next poll                         | Rows stay dismissed in the store                                                                                                |
| Second `,u` within the undo window                                                         | Restores the same 27 and the toast says so                                                                                      |
| `,j` to a visible unread row                                                               | Highlight and chips update immediately; detail waits for the debounce                                                           |
| `,j` into a collapsed tribe                                                                | Only that panel rebuilds                                                                                                        |
| Entering leader mode with 27 unread                                                        | No O(N) candidate or status signature built                                                                                     |
| Persist failure                                                                            | Only that operation's identities restore, with an error toast                                                                   |
| Idle 1 Hz tick and unchanged fleet refresh at about 200 agents                             | No watchdog hitch; each under 16 ms of UI-thread work                                                                           |

A long-running TUI keeps executing the code it imported at start, so live verification
needs a TUI restart after landing.

## Non-goals

- No procs for acks. Moving the remote-attention inventory sync into a service proc is
  deferred; reconsider only after `core-reconcile-upsert` lands and only if measurements
  show the in-TUI thread still costs UI time.
- No SQLite store, warm helper daemon, or free-threaded CPython.
- Remote-attention inventory paging (`sase-1av`) stays out of scope.
- No keymap changes and no feature flag.
