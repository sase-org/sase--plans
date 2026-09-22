---
tier: epic
title:
  Make remote-attention notifications dismissable and unfreeze apollo's attention feed
goal: "A user dismissal of a remote-attention notification sticks for that revision; the
  gateway attention inventory works on hosts with more than 200 visible notifications;
  and on athena the fix is installed, the 8 stuck apollo rows are dismissed and stay
  dismissed, and a fresh sase screenshot no longer shows `?8` at the top right.

  "
phases:
  - id: inbox-dismissal
    title: Honor local dismissal in the remote-attention reconciler
    depends_on: []
    size: small
    description:
      "inbox-dismissal: stop reconcile_remote_attention_inbox from un-dismissing
      same-revision rows, mark and reverse only its own auto-dismissals, and treat
      has_more pages as incomplete, with regression tests."
  - id: inventory-row-cap
    title: Gateway attention inventory must not fail on busy hosts
    depends_on: []
    size: small
    description:
      "inventory-row-cap: in sase-core, drop the 200 raw-row cap from
      project_fleet_attention_inventory (paging bounds output) and pre-filter the
      gateway handler to actionable rows, with core and route tests."
  - id: deploy-dismiss-verify
    title: Install on athena, dismiss the 8 rows, and verify
    depends_on:
      - inbox-dismissal
      - inventory-row-cap
    size: small
    description:
      "deploy-dismiss-verify: run sase update -y, restart pre-update TUIs through their
      Quit/Restart panel, dismiss the remote-attention rows, confirm they stay
      dismissed, and verify via sase screenshot that ?8 is gone."
proposed_by: bbugyi200.athena.0pa
create_time: 2026-09-22 10:05:54
status: wip
---

- **PROMPT:**
  [prompts/202609/remote_attention_dismissal_fix.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/remote_attention_dismissal_fix.md)

# Make remote-attention notifications dismissable and unfreeze apollo's attention feed

## Problem

On athena the TUI top bar shows `?8`: eight `remote-attention` notification rows (sender
`remote-attention`, action `RemoteAttention`, tab `attention`) for apollo `TaskTriage`
gates. Dismissing them — in the TUI or with `sase notify apply-state <id> dismiss` —
never sticks. This was reproduced during planning: a CLI dismiss of
`f3d66550-03af-5212-8a16-624a9971fb3f` went back to `dismissed=false` within about 15 s.

## Root cause (verified)

Two defects combine.

1. **The athena-side reconciler undoes local dismissals.** In
   `src/sase/dispatch/attention_inbox.py`, `reconcile_remote_attention_inbox` calls
   `_refresh_existing_notification` for every inventory entry whose dedup key (origin,
   request id, revision) already has a row. That function forces `read=False` and
   `dismissed=False`, and the row is rewritten whenever that differs from the stored
   row. So any user dismissal of a still-"pending" entry is reverted on the next poll.
   The TUI runs a cache-only inventory poll on nearly every auto-refresh tick, and a
   network poll every 60 s (`FLEET_ATTENTION_INVENTORY_NETWORK_REFRESH_SECONDS`). The
   test `test_attention_inventory_resurfaces_dismissed_pending_request` in
   `tests/test_dispatch_attention_inbox.py` enshrines this behavior. The originating
   epic plan (`plan:202609/unified_agents_across_machines.md`, section
   `attention-inbox`) does not require it. That plan says "query/fold/dismiss changes
   browsing only" and "a changed revision requires renewed review". Dismissal must stay
   a local browsing operation that never answers the remote request, and a **new
   revision** should resurface. A same-revision re-poll must not resurface.

2. **Apollo's inventory endpoint hard-fails, so athena replays a 6-day-old cache.**
   `sase_gateway`'s `fleet_attention_inventory` handler
   (`crates/sase_gateway/src/routes/fleet_attention_handlers.rs` in sase-core) passes
   **every non-dismissed notification row of any kind** into
   `sase_core::project_fleet_attention_inventory`. That function calls
   `project_fleet_attention`, which rejects inputs over `MAX_ATTENTION_ROWS = 200`
   (`crates/sase_core/src/fleet_attention.rs`). Apollo has 251 visible rows (115
   `TaskTriage`, 87 `ViewErrorReport`, 48 plain, 1 `GateExecutionFailed`). So every
   network read returns
   `invalid_request: fleet attention notification rows exceeds 200 entries`. The
   federation worker then serves its last good payload, observed 2026-09-16 (about 499k
   seconds old). Cache-only reads report it as
   `status: "ok", cached: true, error: null`. `_inventory_entries` accepts it and
   re-asserts the same 8 entries as pending forever. Four of the 8 are already dismissed
   or settled on apollo: `sase-10y`, `sase-10d`, `sase-x5`, `sase-xc`. Four are still
   visible there: `sase-10x`, `sase-11m`, `sase-ni`, `sase-11a`. Paging already bounds
   the response, and the input cap counts rows that can never become attention entries.

A latent sibling defect also turns visible as soon as (2) is fixed. The TUI fetches only
the first inventory page (`limit=100`). `_payload_is_fresh_complete` ignores
`page.has_more`, so a fresh first page with `has_more=true` would be treated as
complete: requests on later pages would count as settled and be auto-dismissed, or never
created.

## Goal

- A user dismissal (and read mark) of a remote-attention row sticks for that revision. A
  genuinely new revision of the request still resurfaces as a new visible row.
- The reconciler can still reverse its **own** automatic dismissals: superseded
  revision, or absent from a fresh complete inventory. It does so if the same revision
  later comes back pending.
- An incomplete page (`has_more`) never settles absences.
- The gateway attention inventory works on hosts with more than 200 visible
  notifications.
- On athena: the fix is installed with `sase update -y`, pre-update TUIs are restarted,
  and the 8 rows are dismissed and stay dismissed. A fresh `sase screenshot` shows no
  `?8` at the top right.

No feature flag: this is a bug fix, and no old branch needs to stay reachable.

## Phases

### Phase `inbox-dismissal` — honor local dismissal in the remote-attention reconciler

- slug: `inbox-dismissal`
- size: small
- depends on: none
- repo: sase (primary)

Changes in `src/sase/dispatch/attention_inbox.py`:

1. Add a module constant
   `REMOTE_ATTENTION_AUTO_DISMISSED_ACTION_DATA_KEY = "remote_attention_auto_dismissed"`
   and export it in `__all__`.
2. Wherever `reconcile_remote_attention_inbox` itself dismisses a row, set
   `dismissed=True` **and** add the marker to a copy of `action_data`
   (`{**row.action_data, KEY: "true"}`). This covers the superseded-revision branch and
   the `_covered_by_settling_host` branch. Never mutate the existing dict in place.
3. Rewrite `_refresh_existing_notification(existing, incoming)` so that the reconciler
   owns only the remote-derived fields: `icon`, `color`, `notes`, `tags`, `action`,
   `action_data` (from `incoming`, which carries no marker), and `silent=False`.
   - If `existing.dismissed` is true and `existing.action_data` carries the auto marker,
     the reconciler dismissed it itself. The same revision is pending again, so
     resurface it: `dismissed=False`, `read=False`. The marker drops because
     `action_data` comes from `incoming`.
   - Otherwise preserve `existing.read` and `existing.dismissed`. Keep preserving
     `muted` and `snooze_until`, as today.
   - Keep the existing `if refreshed != existing` guard, so a user-dismissed row with
     unchanged remote content causes no rewrite and does not count as `updated`.
   - Legacy dismissed rows without the marker count as user-dismissed. Document this in
     the docstring. It is the safe direction: a legacy auto-dismissed row that comes
     back with the same revision stays hidden, which is rare because settled requests do
     not un-settle.
4. In `_payload_is_fresh_complete(payload, host)`: if `payload["page"]` is a mapping
   whose `has_more` is truthy, or whose `next_cursor` is a non-empty string, return
   `False`. Do not follow cursors in this phase; that is a follow-up.
5. Update the module and function docstrings to state the policy: dismissal and read are
   local browsing state, never an answer. A new revision is a new row and needs renewed
   review. Only reconciler auto-dismissals are reversible.

Tests in `tests/test_dispatch_attention_inbox.py`, reusing the existing `_response`,
`_entry`, and `notification_store_file` helpers:

- Replace `test_attention_inventory_resurfaces_dismissed_pending_request` with
  `test_attention_inventory_keeps_user_dismissed_pending_request`. Use the same setup
  (user sets read, dismissed, muted, snooze). Re-reconcile the same response and assert
  `outcome.updated == 0`, `outcome.changed is False`, and that the row is still
  `dismissed` and `read`, with muted and snooze preserved and
  `load_notifications() == []`.
- Add a regression test for this incident: a host result with `status: "ok"` and
  `cached: true` that repeats the same pending entry across many reconciles never
  resurfaces a user-dismissed row.
- Add a test that a user-dismissed revision 1 plus an inventory carrying revision 2
  creates a new visible revision-2 row (`created == 1`). The revision-1 row stays
  dismissed and carries no auto marker.
- Add a test that a row auto-dismissed by absence from a fresh complete host carries the
  marker. When the same revision reappears pending, it is resurfaced (`dismissed=False`)
  and the marker is gone.
- Add a test that the superseded-revision auto-dismissal carries the marker.
- Add a test that a fresh payload whose page has `has_more=True` (with a `next_cursor`)
  does not dismiss an existing row that is absent from that page.
- Keep all other existing tests passing. Check
  `tests/ace/tui/test_remote_lifecycle_actions.py` for any assertion that depends on
  resurfacing, and update it the same way if needed.

Before finishing, read the `lint_and_test` reference memory
(`sase memory read lint_and_test.md -r "<why>"`) and run the verification it prescribes
(`just check`).

### Phase `inventory-row-cap` — gateway attention inventory must not fail on busy hosts

- slug: `inventory-row-cap`
- size: small
- depends on: none
- repo: sase-core (linked; open it with `sase repo open sase-core -r "<why>"` and commit
  it through the final declaration)

Changes in `crates/sase_core/src/fleet_attention.rs`:

1. Move the body of `project_fleet_attention` (everything after the `MAX_ATTENTION_ROWS`
   check) into a private helper, e.g.
   `project_attention_entries(origin, rows, resolved, observed_at_unix)`. The helper
   keeps the installation-id, timestamp, and `MAX_ATTENTION_IDENTITIES` validation and
   has no raw-row cap. `project_fleet_attention` keeps its public contract: it rejects
   `rows.len() > MAX_ATTENTION_ROWS`, then delegates to the helper. Its row-scoped
   callers rely on that bound.
2. `project_fleet_attention_inventory` calls the helper directly, so the raw row count
   no longer bounds it. The response stays bounded by `FLEET_ATTENTION_MAX_PAGE_ROWS`
   paging and `validate_fleet_attention_inventory_response`. Update its doc comment to
   say so.

Changes in `crates/sase_gateway/src/routes/fleet_attention_handlers.rs`:

3. In `fleet_attention_inventory`, pre-filter notifications to actionable kinds before
   building row wires. Keep a row only when
   `MobileActionKindWire::from_notification_action(n.action.as_deref())` is not
   `NonAction` or `Unsupported`. This avoids cloning hundreds of non-actionable rows and
   computing their action state. Leave `attention_notification_rows` as the row builder.

Tests:

- `sase_core` unit tests in `fleet_attention.rs`. Build an inventory over 250 rows: 245
  non-actionable plus 5 pending gates. It succeeds and returns exactly the 5 pending
  entries. Build an inventory over 150 pending gates with `limit: Some(100)`. It returns
  100 entries, `has_more: true`, `next_cursor: Some("off:100")`, and
  `total_matching_entries == 150`. Keep a test proving `project_fleet_attention` still
  rejects 201 rows.
- A gateway route test in `crates/sase_gateway/src/routes/tests/fleet_attention.rs`:
  with more than 200 visible notifications, the inventory route returns success and the
  pending gate entries. Follow the existing inventory route tests there.
- Run the sase-core repo's standard checks (its `justfile` or the CI recipe it
  documents; at minimum `cargo test -p sase_core fleet_attention` and
  `cargo test -p sase_gateway fleet_attention`, plus `cargo clippy` on both crates).
  Also make sure the `sase_core_py` crate still builds.

No release is cut in this phase. Deploying to apollo (update plus gateway restart) is
deliberately out of scope. See the notes below.

### Phase `deploy-dismiss-verify` — install on athena, dismiss the 8 rows, verify

- slug: `deploy-dismiss-verify`
- size: small
- depends on: `inbox-dismissal`, `inventory-row-cap`
- repo: none. This is an operational phase; it makes no repository changes and declares
  no commit.

Run every command in the foreground with generous timeouts. `sase update` can take
several minutes when it rebuilds Rust dev artifacts.

1. **Confirm the fixes landed.** Run `sase update -n -j`. The `sase` and `sase-core-rs`
   packages must be `actionable`, behind their `origin/master`, and the incoming range
   must include the two phase commits, or already current with them. If either phase
   commit is not on its `origin/master`, stop and report; do not update.
2. **Install.** Run `sase update -y`. Then confirm the editable `sase` checkout reported
   as `git_root` contains `REMOTE_ATTENTION_AUTO_DISMISSED_ACTION_DATA_KEY` in
   `src/sase/dispatch/attention_inbox.py`.
3. **Restart every pre-update TUI.** A still-running TUI keeps the old reconciler in
   memory and re-undismisses the rows within seconds.
   - List TUIs with `ps -eo pid,ppid,lstart,args | grep -E 'sase tui' | grep -v grep`.
     At planning time the user's TUI ran in tmux window `sase:5` (window name `tui`),
     launched as `python -m sase tui --restart-service` from a zsh pane.
   - For each TUI started before step 2, map it to its pane:
     `tmux list-panes -a -F '#{session_name}:#{window_index}.#{pane_index} #{pane_pid}'`,
     matching the TUI's parent pid.
   - Restart it through its own Quit/Restart panel. Never kill the process, the pane, or
     the pane's shell.
     - Run `tmux capture-pane -p -t <pane>` and confirm the TUI's normal screen is
       showing (no prompt-bar input focus, no open modal).
     - Send `tmux send-keys -t <pane> Q`, then re-capture until the Quit/Restart panel
       is visible. It shows "restart" choices, including `2/r restart`.
     - Send `tmux send-keys -t <pane> r`. That is "Restart TUI", which re-execs the same
       argv and keeps any prompt draft.
     - Confirm a new `sase tui` process started after step 2 is running in that pane.
   - If the pane is not in a safe state, or the panel does not appear, do not force it.
     Stop before step 4 and report that the user must restart that TUI (`Q`, then `r`)
     for the dismissal to hold.
4. **Dismiss the rows.** Run `sase notify list -j -l 500 --sender remote-attention` and
   dismiss every row with `action == "RemoteAttention"` and `dismissed == false`, using
   `sase notify apply-state <id> dismiss`. At planning time these were the 8 apollo
   rows: `f3d66550-03af-5212-8a16-624a9971fb3f`, `a7f0dc6c-57e1-50e7-8081-c499a8e151d1`,
   `44722f65-2cb5-5300-bcc4-d798920b00f6`, `e5511f45-225e-5a6b-b343-468cdc6e4430`,
   `150caeac-9385-5e4c-ab13-3f1191d3b834`, `0b073892-3286-5af7-bfca-a8ab2634a4c8`,
   `59e8a676-0c1e-5a25-918c-9e6c90afa41b`, `5d2f2de4-d4cb-582d-b419-ae3f3f81b726`. Use
   the live query, not only this list.
5. **Prove it sticks.** Run `sleep 120` in the foreground. That covers more than one 60
   s network poll and many cache ticks. Then re-run the list with `--all` and confirm
   every one of those rows is still `dismissed: true`. If any flipped back, find the
   process still running old code (another TUI, or a TUI in another tmux session) and
   fix that cause. Do not re-dismiss in a loop.
6. **Screenshot verification.** Run
   `sase screenshot -o <tmp>/remote_attention_after.png -d 3000 -t 120` and view the
   PNG. The top-right notification indicator must no longer show `?8`, and must show no
   `?` attention chip unless a genuinely new remote request arrived. Explain any chip
   that remains. Also run `tmux capture-pane -p -t <user TUI pane>` and confirm `?8` is
   absent from the user's restarted TUI.
7. **Report:** the before/after state, the screenshot path, and which of the 8 were
   still pending on apollo at planning time (listed above). Also state the apollo caveat
   from the notes below.

## Notes and follow-ups (not in scope)

- **Apollo deployment:** apollo's gateway keeps failing its inventory until apollo runs
  a build containing `inventory-row-cap`, via `sase update` on apollo and a gateway
  restart. Once it does, athena will show apollo's **live** pending gates. That could be
  many more than 8: apollo had 115 visible `TaskTriage` rows at planning time, and only
  the first page of 100 is fetched. Those are real pending decisions; dismiss, snooze,
  mute, or answer them. This epic does not update apollo, because that would flood
  athena's attention tab with apollo's backlog and needs the user's call.
- **Follow-up (propose, do not implement here):** follow inventory `next_cursor` per
  host, so hosts with more than 100 pending requests are fully mirrored.
- **Follow-up (propose, do not implement here):** after a failed network read, the
  federation worker's cache-only reads still report `status: "ok"`, so stale snapshots
  look fresh. Once the network fails they should carry the stale status and error, as
  the network path already does.
- Phase workers must record these as `PROPOSED FOLLOW-UP:` notes on their own phase
  bead, not create beads.
