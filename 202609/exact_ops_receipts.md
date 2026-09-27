---
tier: tale
title: Make exact remote stop and retry settle certainly
goal: "Exact remote stop and retry observe a settled same-key receipt, a fresh dispatch
  row is addressable before a slow owner snapshot rebuild finishes, and a killed fleet
  row keeps its retry context for as long as it advertises retry.

  "
size: medium
proposed_by: bbugyi200.apollo.sase-1aq.10.7.5.2
bead: sase-1aq.10.7.5.2
status: done
---

- **PARENT:**
  [202609/1aq_close_original_gates.md](https://github.com/sase-org/sase--plans/blob/main/202609/1aq_close_original_gates.md)
- **BEAD:**
  [sase-1aq.10.7.5.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1aq/sase-1aq.10.7.5.2.md)
- **AGENTS:**
  - [bbugyi200.apollo.sase-1aq.10.7.5.2](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.apollo.sase-1aq.10.7.5.2.md)
- **COMMITS:**
  - [b57cd21](https://github.com/sase-org/sase-core/commit/b57cd21315a24175dcd1bc866d5a2abe526d9f29)
    — feat(fleet): settle exact mutate receipts, overlay fresh launches, retain killed
    rows

# Plan: Make exact remote stop and retry settle certainly

This tale implements phase `exact_ops_receipts` on bead `sase-1aq.10.7.5.2` (parent epic
`sase-1aq.10.7.5`, plan `plan:202609/1aq_close_original_gates.md`). The live Athena
matrix, parity capture, ancestor landing, and dispatch-memory publication belong to
later phases. Do not close `sase-xe.16.11`, `sase-1aq.10.7.5`, or any ancestor. Do not
run `just check-full`. Do not file a duplicate for the clean-base rename/proc failures
owned by epic `sase-1ab` and `sase-th`, or for `test_dispatch_federation`
`AF_UNIX path too long` failures caused by a long workspace tmp path. Record a check
failure that reproduces on the clean base as a `PROPOSED FOLLOW-UP:` note on
`sase-1aq.10.7.5.2` and continue.

Open sase-core with `sase repo open sase-core` before editing it. Read that checkout's
`AGENTS.md`. Shared fleet behavior goes in sase-core. The controller client and owner
kill path stay in this sase checkout. Run checks through `sase tool run`, not a bare
`just` or `cargo`.

## Causes

Three notes on `sase-xe.16.11` (#8 killed rows, #9 uncertain receipts, #10 catalog lag)
are the scope. The code matches those notes:

1. **Uncertain receipts.** `src/sase/dispatch/mutations.py` marks the intent
   `acceptance_uncertain` and calls `mutate_sync` with `config.request_timeout_seconds`
   (default 5.0). The owner handler `fleet_mutate` in
   `crates/sase_gateway/src/routes/fleet_handlers.rs` holds that HTTP call until
   `kill_agent` returns. `request_user_kill(wait=True)` waits up to
   `DEFAULT_TERMINATE_GRACE_SECONDS` (6s) plus a 2s post-kill window. The effect lands
   on Apollo after the controller has already given up. The request already sends
   `acceptance_window_seconds` of at least 30, and `decide_fleet_mutation_replay`
   returns the original receipt for the same key, but the client never waits or polls. A
   later same-key call also loses: `fleet_mutate` runs `evaluate_mutation_precondition`
   before `reserve`, so once the row is terminal the poll dies with `AlreadyTerminal`
   and never reads the stored receipt.
2. **Catalog lag.** `FleetReadService::current_snapshot` serves a cached snapshot until
   `FLEET_SNAPSHOT_STALE_SECONDS` (60). A rebuild that exceeds
   `SNAPSHOT_REFRESH_TIMEOUT` (4s) keeps the previous snapshot via
   `retain_previous_or_error`. On a large owner index that budget is missed, so a row
   that is already in `agent_artifact_index.sqlite` stays out of `catalog` and `detail`.
   Exact stop looks the target up through `catalog_sync`
   (`src/sase/ops/commands/machine.py` `_lookup_snapshots`) and then `detail`, and
   reports no matching agent. The observed ~7 minutes is that stuck cache, not a timer
   to document or preserve. Do not raise the 4s budget and do not tell operators to
   wait.
3. **Reaped killed rows.** `kill_named_agent` always calls `_record_dismissal`. Fleet
   presentation drops `agent_session_root_dismissed` rows immediately
   (`crates/sase_gateway/src/fleet_reads/snapshot.rs`). The owner Agents loader then
   deletes `done.json` and the index row for dismissed artifacts
   (`compute_loader_cleanup` in `src/sase/ace/tui/actions/agents/_loading_compute.py`).
   The ~30 minutes is when that loader next ran, not a retention constant.
   `terminalize_stale_active_agent_artifact_index_rows` (24h) does not delete artifact
   files. Retry is also withheld from every terminal row
   (`lifecycle_and_content_capabilities`), even though `mutation_already_terminal`
   already allows retry and fork of an `AgentTurn`.

## Contract

Keep uncertain operations non-replayable under a new key. Keep cross-project isolation
and stale-locator / stale-revision refusal (`evaluate_mutation_precondition`, exact
locator and `row_revision`). Do not add a fleet wire schema version, a new Python
binding, or a `sase-core-revision.txt` pin move unless an implementation detail forces a
field old readers reject. Prefer the existing mutate response and catalog summary
shapes.

### Settled stop and retry receipts

- In `fleet_mutate`, if an unexpired `FleetMutationStore` record already exists for the
  request key and payload, return that receipt immediately. Do not re-run the live
  precondition and do not execute again. Expired, tombstoned, conflicting, and
  mismatched-target records keep today's decisions.
- An unseen key still validates the live precondition, reserves, executes, and settles
  before the HTTP response, as `fleet_mutate_stop_settles_and_replays` requires. A fast
  bridge still returns `receipt.state=settled` on the first response.
- In `_submit_remote_mutation`, pass the acceptance window
  (`max(timeout or request_timeout, 30)`) as `mutate_sync`'s timeout. Leave the 5s
  default in place for catalog reads, launch, and attention.
- If that call raises `FederationWorkerResponseError`, or the receipt state is not
  `settled` and the decision is not a terminal refusal (`precondition_mismatch`,
  `conflict`, `expired`, capability missing), poll the same request until the receipt is
  `settled` or `expires_at_unix_ms` has passed. Sleep a short interval between polls.
  Never mint a new `operation_id` inside this loop. On window expiry, keep
  `acceptance_uncertain` and do not replay under a new key.
- Treat `return_original_receipt` with `state=settled` as `already_settled`. Do not
  treat a still-`accepted` receipt as success.
- Update
  `tests/test_dispatch_mutations.py::test_lost_reply_reconciles_under_the_same_key` so
  one submit polls the same key and returns `already_settled` when the first worker
  response is lost and a later same-key response is settled. Add a case that stays
  `uncertain` when every poll fails through the window, and assert the key is unchanged.
  Add a core test that a same-key mutate after the row is already terminal returns the
  stored settled receipt instead of `AlreadyTerminal`.

### Address a fresh row before the snapshot rebuild

- When `catalog` or `detail` is served from a cached snapshot that lacks the logical key
  of a settled, unexpired fleet launch receipt, resolve that agent from the artifact
  index with cached freshness and merge the real resolved summary and detail into the
  response. Use the index record's own revision and locator. Do not invent a second
  revision.
- If the index has no record yet, skip the overlay. A later full snapshot that already
  contains the logical key wins, and the overlay drops away.
- Add a gateway test: the index contains the new row, the cached snapshot does not (age
  it, or seed the cache before the row exists, then time out the forced rebuild), and
  both catalog and detail return the row with `lifecycle.stop` while it is alive. A
  missing index record still produces no row.
- Document, in `docs/remote_dispatch.md`, that exact stop and retry address a launched
  agent as soon as the owner index has its record. They do not wait out a snapshot
  rebuild that missed `SNAPSHOT_REFRESH_TIMEOUT`.

### Keep killed-row retry context while retry is advertised

- Fleet mutate's stop path must kill without dismissing. Thread a retain-for-retry flag
  from `execute_fleet_mutation`'s `Stop` arm through the mobile kill request
  (`src/sase/integrations/_mobile_agent_lifecycle.py`) into `kill_named_agent`. When the
  flag is set, skip `_record_dismissal`. Still signal the process, clear running and
  waiting markers, persist mobile kill context, and dismiss notifications. Local TUI
  kill and an ordinary `kill_named_agent` call keep today's dismissal.
- Advertise `lifecycle.retry` and `lifecycle.fork` on a served `AgentTurn`, including a
  terminal or dead one. Keep `lifecycle.stop` limited to `OwnerLivenessWire::Alive`.
  Update capability assertions that required terminal rows to omit retry.
- Because the fleet stop no longer writes the dismissed-agents index, fleet presentation
  keeps the row for the existing recent-terminal window (7 days,
  `FLEET_PRESENTATION_RECENT_TERMINAL_MAX_ROWS` of 200), and `compute_loader_cleanup`
  does not delete its `done.json` or index row. An owner dismissal still excludes the
  row and may clean its artifacts. That is the retention rule: retry context lives
  exactly as long as the row is served and advertises retry.
- Add a test that a fleet stop leaves the dismissed-agents index unchanged, leaves
  `done.json` in place, and leaves catalog and detail able to return the row with
  `lifecycle.retry`. The existing dismissed-identity catalog tests stay green for a real
  dismissal.

## Docs and verification

Update `docs/remote_dispatch.md` in the Launch And Operate section:

- Stop, retry, and fork wait for a settled receipt inside the acceptance window (at
  least 30 seconds). A lost response is recovered with the same operation key. An
  outcome still unknown when the window ends is uncertain and must not be submitted
  again under a new key.
- A fresh remote agent can be exact-stopped once the owner index has its record,
  including while the full fleet snapshot is still the previous cache.
- A remote fleet stop does not dismiss the row. Killed and other served agent turns
  advertise retry and fork until they leave the recent-terminal presentation window.
  Owner dismissal still removes them.

Verify:

- In sase-core, the focused gateway tests for mutate, catalog overlay, and killed-row
  retention, then `sase tool run check`. Give `check` at least 10 minutes. `sase_core`
  edits recompile the whole crate; batch them between `sase tool run fast` runs.
- In sase, `tests/test_dispatch_mutations.py` and any machine-agent kill test you add,
  then `sase tool run check`. Do not run `just check-full`.
- Before closing, run `sase bead epic-symbols sase-1aq.10.7.5.2`. Re-key any
  `--epic-symbol` entry that names this phase onto the parent epic or a later open
  phase. `sase bead close` refuses while leftovers remain.
- Close only `sase-1aq.10.7.5.2`, with a note that names the settled-receipt poll, the
  index-backed catalog merge, and the no-dismiss killed-row retention, plus the checks
  that passed. A clean-base failure that also fails on the unmodified tree is a
  `PROPOSED FOLLOW-UP:` note, not a reason to leave the phase open.
