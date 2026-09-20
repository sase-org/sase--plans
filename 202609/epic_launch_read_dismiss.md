---
tier: epic
title: Agent-row settlement notifications clear when the row is read
goal: 'An `epic-launch` or `monitor-settlement` notification that names one exact
  agent row is dismissed automatically the moment that row goes unread to read in
  the Agents tab, through the same Rust-owned store operation every surface already
  uses, so the notification inbox stops accumulating launch rows the user has already
  acknowledged on the agent.

  '
phases:
- id: core-match
  title: Rust store matches row-owned settlement notifications
  depends_on: []
  size: small
  description: 'core-match: teach the notification store''s agent-keyed dismissal
    to match host-owned settlement rows by exact (cl_name, raw_suffix), with parity
    tests that pin the exact-match requirement.

    '
- id: host-ack
  title: Read acknowledgment dismisses the settlement row
  depends_on:
  - core-match
  size: medium
  description: 'host-ack: ratchet the core revision pin, widen the TUI''s cached-snapshot
    predicate so acknowledged settlement rows leave the cache and the indicator with
    the completion row, and document the widened contract.

    '
- id: unread-projection
  title: An active settlement row keeps its agent row unread
  depends_on:
  - host-ack
  size: small
  description: 'unread-projection: project active row-owned settlement notifications
    into agent row unread state so a settlement that arrives after the completion
    was read still has a read event to clear it.'
proposed_by: bbugyi200.apollo.17
create_time: 2026-09-20 16:56:50
status: wip
bead_id: sase-14l
---

- **PROMPT:** [prompts/202609/epic_launch_read_dismiss.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/epic_launch_read_dismiss.md)
- **BEAD:** [sase-14l](https://github.com/sase-org/sase--beads/blob/main/pages/sase-14l/README.md)

# Plan: Agent-row settlement notifications clear when the row is read

## 1. The bug

When a `sase bead work` epic launch settles, SASE sends a notification like:

```
[epic-launch] Epic sase-14j launched from agent_bead_touches.md
```

That notification names exactly one agent row in its `action_data`, but nothing ever
clears it from the row's side. Selecting the launching agent in the Agents tab marks the
agent read and dismisses the agent's _completion_ notification, while the `epic-launch`
row stays unread in the inbox until the user dismisses it by hand in the Notifications
modal.

The reported case: selecting the `0oa` agent marked it read, and the `epic-launch`
notification for that same agent survived.

### 1.1 Why it survives today

`src/sase/bead/epic_launch.py` has two settlement paths in `finish_epic_launch`:

- When a deferred planner completion exists, `fold_epic_launch_outcome`
  (`src/sase/bead/epic_launch_handoff_payload.py`) folds the launch result into the
  planner's completion payload and stamps `action="JumpToAgent"`. That row **is** an
  agent completion notification, so it already auto-dismisses on read.
- Otherwise it calls `notify_workflow_complete("epic-launch", cl_name, ...)` with
  `action=None` and `action_data=settlement_notification_action_data(...)`. That is the
  row in the screenshot.

A live example of the second shape, from
`sase notify list -j --all --sender epic-launch`:

```json
{
  "sender": "epic-launch",
  "action": null,
  "action_data": {
    "cl_name": "gh_sase-org__sase",
    "raw_suffix": "20260920082910",
    "family_root_suffix": "20260920073438",
    "agent_root_timestamp": "20260920073438"
  },
  "tags": ["epic", "launch"]
}
```

Every read-acknowledgment path in the Agents tab funnels into one Rust-owned store
operation:

- `_clear_agent_unread_and_dismiss_notification` — row selection, unread-agent jump, and
  toggling a manually unread row back to read
  (`src/sase/ace/tui/actions/agents/_unread_state.py`)
- `_mark_current_unread_done_agents_read` — the bulk unread/read toggle (same file)
- `_dismiss_agent_completion_notifications_for_dismissed_agents` — row dismissal (same
  file)
- `_persist_marked_agent_group_save` — marked-agent group dismissal
  (`src/sase/ace/tui/actions/agents/_marking.py`)

All four call
`sase.notifications.dismiss_agent_completion_notifications_matching_agents`
(`src/sase/notifications/store.py`), which submits the
`DismissAgentCompletionsMatchingAgents` wire update. In the core repo that update is
served by `matches_agent_completion_notification_for_agents` in
`crates/sase_core/src/notifications/store.rs`, which requires `sender == "user-agent"`
and `action` in `{JumpToAgent, ViewErrorReport}`. An `epic-launch` row has neither, so
it never matches.

There is also no existing store operation that can reach it: `DismissMatchingAgents`
routes through `matches_agent_notification`, which switches on `action`, and an
`epic-launch` row has `action: null`.

### 1.2 Why the fix belongs in the Rust core

Notification JSONL store state updates are Rust-owned (see `docs/rust_backend.md`), and
dismiss-by-agent-key is shared backend behavior: the mobile gateway and any other
frontend that acknowledges an agent must land the same rows as the TUI. Matching in
Python — loading the snapshot and calling `mark_dismissed(id)` per row — would put the
rule in one frontend only. So the matcher is widened in `sase-core`, and the host keeps
only its cached-snapshot mirror of that same rule.

## 2. The rule this epic lands

> A notification whose sender is a host-owned settlement sender (`epic-launch`,
> `monitor-settlement`) and whose `action_data` names **both** a non-empty `cl_name` and
> a non-empty `raw_suffix` is owned by that one agent row. It is dismissed whenever that
> row is acknowledged — read, dismissed, or marked — exactly like the row's completion
> notification.

`monitor-settlement` is included because it is the sibling sender in the TUI's existing
`_SETTLEMENT_NOTIFICATION_SENDERS` set in
`src/sase/ace/tui/actions/agents/_notification_utils.py`, carries the identical
`settlement_notification_action_data` shape, and has the identical lifecycle gap.
Leaving it out would encode an arbitrary split between two rows the host already treats
as one class.

### 2.1 The exact-match requirement is load-bearing

`cl_name` on these rows is the **patch name** (`gh_sase-org__sase`), which every agent
in the project shares. The existing completion matcher falls back to matching on
`cl_name` alone when the notification carries no `raw_suffix`; the settlement matcher
**must not** do that, or acknowledging any single agent would dismiss every project-wide
settlement row at once. Settlement rows always carry `raw_suffix`, so requiring it costs
nothing.

Family reach is already handled on the host side and must not be re-added in Rust:
`_notification_keys_for_agents` expands an agent node to its owned rows'
`(cl_name, raw_suffix)` keys before calling the store, so acknowledging a family node
already covers a settlement row that names a descendant.

## 3. Rust store matches row-owned settlement notifications

Work in the checkout printed by
`sase repo open sase-core -r "Match row-owned settlement notifications in the notification store"`;
do not use a hard-coded path.

In `crates/sase_core/src/notifications/store.rs`:

1. Add a `matches_agent_settlement_notification(notification) -> bool` predicate beside
   the existing `matches_agent_completion_notification`. It returns true only when the
   sender is `epic-launch` or `monitor-settlement` **and** `action_data` holds a
   non-empty `cl_name` **and** a non-empty `raw_suffix`. Do not inspect `action`:
   settlement rows carry `action: null`, and the folded variant that does carry
   `JumpToAgent` is already a `user-agent` completion row.
2. Add `matches_agent_settlement_notification_for_agents(notification, agents)`. It
   requires an exact `(cl_name, raw_suffix)` match against a supplied key, with **no**
   `cl_name`-only fallback branch. Add a short comment naming the reason from section
   2.1 so a later reader does not "fix" the asymmetry with the completion matcher.
3. In the `NotificationStateUpdateWire::DismissAgentCompletionsMatchingAgents` arm,
   treat a row as a match when either `matches_agent_completion_notification_for_agents`
   or the new settlement matcher accepts it. Keep the already-dismissed skip, the
   `snooze_until = None` clear, and the `matched_count` / `changed_count` accounting
   exactly as they are.

Do **not** change `matches_agent_completion_notification` itself, and do **not** change
the agent-less `DismissAgentCompletions` arm. That primitive means "every agent
completion row" to its own callers, and widening it is outside this epic.

Do not add a new wire variant. Widening the existing arm keeps
`crates/sase_core/src/notifications/wire.rs`, the PyO3 bindings, and
`src/sase/core/notification_store_wire.py` untouched, so the host needs no new binding
and no `sase-core-rs` floor bump — only a revision-pin move.

### 3.1 Core tests

Extend `crates/sase_core/tests/notification_store_parity.rs` next to the existing
`DismissAgentCompletionsMatchingAgents` and `DismissAgentCompletions` cases. Seed one
store and assert the dismissed/undismissed split over:

- an `epic-launch` row with `action: null` whose `(cl_name, raw_suffix)` matches a
  supplied key — dismissed;
- a `monitor-settlement` row with a matching key — dismissed;
- an `epic-launch` row with a matching `cl_name` but a **different** `raw_suffix` — not
  dismissed;
- an `epic-launch` row carrying `cl_name` but no `raw_suffix` — not dismissed, proving
  there is no `cl_name`-only fallback;
- a settlement-sender row that matches nothing in `agents` — not dismissed;
- an already-dismissed settlement row — not counted in `matched_count`;
- the existing `user-agent` completion row — still dismissed, proving no regression;
- an unrelated sender (`axe`, `crs`) row with the same `cl_name` — not dismissed.

Also assert the agent-less `DismissAgentCompletions` arm leaves a settlement row alone,
pinning the deliberate asymmetry from section 3.

### 3.2 Phase verification

- `cargo fmt` in the core checkout.
- Targeted: `cargo test -p sase_core --test notification_store_parity`.
- Then the core repository's required `just check`.
- Let the core repo's release tooling own `crates/sase_core/CHANGELOG.md`; do not
  hand-edit it.
- Run `sase bead epic-symbols <this phase's bead id>` before closing the phase.

Publish the core change through the normal host-owned finalization path. Do **not**
touch `sase-core-revision.txt` in this phase; the next phase owns the pin move once this
change is on sase-core's remote default branch.

## 4. Read acknowledgment dismisses the settlement row

This phase works in the primary `sase` repository.

### 4.1 Move the core revision pin first

`sase-core-revision.txt` is the revision CI and Master Gate build the Rust core from, so
the host change is untestable in CI until the pin includes the `core-match` commit.

1. Confirm the core commit is reachable from the pin target:
   `git merge-base --is-ancestor <core-match sha> $(cat sase-core-revision.txt)` — if it
   already succeeds, the scheduled `core-pin-ratchet` workflow
   (`.github/workflows/core-pin-ratchet.yml`, every 6 hours) has already landed the bump
   and there is nothing to move.
2. Otherwise run `python3 tools/ratchet_core_revision` and commit the new pin with the
   rest of this phase.
3. Run `tools/check_sase_core_rs_bindings` and `tools/validate_sase_core_rs` so the
   local extension and the pin agree before touching Python.

If `tools/ratchet_core_revision` exits 3 (cannot determine a safe ratchet) because the
core change is not yet on sase-core's remote HEAD, stop and report that instead of
hand-editing the SHA.

### 4.2 Mirror the rule in the host's cached snapshot

The Rust change alone makes the durable store correct, but the TUI keeps an in-memory
snapshot that feeds the notification indicator and the unread counts. Dismissed
settlement rows must leave that cache in the same frame, or the indicator will keep
counting a row that is already dismissed on disk until the next poll.

In `src/sase/ace/tui/actions/agents/_notification_utils.py`:

- Add
  `agent_settlement_notification_matches_agent(notification, *, cl_name, raw_suffix)`
  beside `agent_completion_notification_matches_agent`. It must encode the same
  exact-match rule as section 2.1: active (not dismissed), sender in
  `_SETTLEMENT_NOTIFICATION_SENDERS`, non-empty `cl_name` equal to the supplied one, and
  a non-empty `raw_suffix` equal to the supplied one — no `cl_name`-only fallback.
- Add `agent_row_notification_matches_agent(...)` as the single
  `completion or settlement` predicate host callers use, so the two rules stay in one
  place.
- Do **not** change `active_completion_agent_keys` or
  `_is_active_agent_settlement_notification` in this phase.
  `active_completion_agent_keys` feeds the unread projection (phase `unread-projection`
  owns that), and `_is_active_agent_settlement_notification` feeds targeted artifact-dir
  refresh, where it deliberately also accepts cross-family completion rows this rule
  must not pick up.

In `src/sase/ace/tui/actions/agents/_unread_state.py`, switch
`_remove_agent_completion_notifications_from_cache` to the combined predicate. Every
read-ack and dismissal path listed in section 1.1 already routes through that helper and
through `dismiss_agent_completion_notifications_matching_agents`, so no call site needs
new plumbing — verify that by reading them rather than by adding any.

Check whether `src/sase/ace/tui/actions/agents/_notification_polling.py` and
`_notification_provider.py` keep any parallel copy of the completion predicate for the
indicator or `_last_unread_ids`; if so, route it through the combined predicate too.

### 4.3 Host tests

Add to the existing suites rather than creating new modules where one fits:

- `tests/ace/tui/test_agent_unread_selection.py` — selecting an unread terminal agent
  dismisses an `epic-launch` row that names that agent's exact `(cl_name, raw_suffix)`,
  and leaves a settlement row that names a different `raw_suffix` under the same
  `cl_name` alone. Note the module's autouse fixture already mocks
  `dismiss_agent_completion_notifications_matching_agents`; assert on the keys it
  receives, and cover the real matching through the cache-side predicate and the Rust
  parity tests from phase `core-match`.
- `tests/ace/tui/test_agent_unread_toggle.py` — the bulk read toggle and the `U`
  toggle-to-read path drop settlement rows from the cached snapshot.
- A focused predicate test for `agent_settlement_notification_matches_agent`: exact
  match, wrong `raw_suffix`, missing `raw_suffix`, dismissed row, and a non-settlement
  sender.

### 4.4 Documentation

Update `docs/notifications.md`:

- The Agents-tab unread paragraph ("Unread state on the Agents tab is projected from the
  active user-agent completion notifications…") gains the new rule: a host-owned
  settlement row that names an exact agent row is acknowledged with that row.
- The host-owned settlement paragraph (senders `epic-launch` and `monitor-settlement`)
  gains the dismissal half of the contract, including the exact-match requirement and
  why `cl_name` alone is not enough.
- Keep the existing statement that plan approvals and user questions still require an
  explicit `y`/`n` response and are never auto-dismissed by row navigation — this epic
  does not change that.

Note in `docs/rust_backend.md`'s notification-store bullet that the agent-keyed
completion dismissal also covers row-owned settlement rows, so the boundary doc does not
go stale.

### 4.5 Phase verification

- `just install` if the workspace venv is stale, then `just fix` inline.
- `sase tool run check` (falls back to raw `just check` only if `sase tool` is
  unavailable on a stale install).
- Manual confirmation on a real store, read-only first:
  `sase notify list -j -l 20 --all --sender epic-launch` to capture a live row's
  `action_data`, then confirm in the TUI that reading the named agent row clears it.
- Run `sase bead epic-symbols <this phase's bead id>` before closing the phase.

### 4.6 Confirm the row identity assumption before writing code

One assumption underpins the whole epic and is cheap to verify first: the `raw_suffix`
an `epic-launch` notification carries is the artifact timestamp of an agent row the user
actually sees and reads in the Agents tab (the launching planner or gate-shell agent),
not an internal directory with no row.

Check it with `sase notify list -j -l 20 --all --sender epic-launch` and match a row's
`action_data.raw_suffix` against the agent list. If some settlement rows name a
timestamp with no corresponding Agents-tab row, the Rust matcher and the host predicate
from sections 3 and 4.2 are still correct — they simply will not fire for those rows.
Record that in the phase bead as a `PROPOSED FOLLOW-UP:` note rather than widening the
match to `family_root_suffix`, which would let one read clear settlement rows belonging
to sibling agents.

## 5. An active settlement row keeps its agent row unread

This phase closes the arrival-ordering hole left by phase `host-ack` and works in the
primary `sase` repository.

### 5.1 The hole

Phase `host-ack` hooks dismissal to the unread → read transition, and
`_clear_agent_unread_and_dismiss_notification` returns early — dismissing nothing — when
the agent's identity is not in `_unread_completed_agent_ids`. Agent-row unread state is
projected only from active `user-agent` completion notifications
(`active_completion_agent_keys` → `_reconcile_unread_from_completion_notifications` in
`src/sase/ace/tui/actions/agents/_notification_unread_projection.py`).

So when a settlement notification arrives _after_ the user already read that agent's
completion, the row is read, selecting it is a no-op, and the settlement row is again
unreachable from the row. The reported case is fixed without this phase; this phase
makes the rule hold in every ordering, and restores the one-to-one row/notification
contract `docs/notifications.md` already claims.

### 5.2 The change

In `src/sase/ace/tui/actions/agents/_notification_utils.py`, add
`active_row_owned_notification_keys(notifications)` returning the union of
`active_completion_agent_keys(notifications)` and the exact `(cl_name, raw_suffix)` keys
of active settlement rows, using the phase `host-ack` predicate. Leave
`active_completion_agent_keys` itself unchanged: `_notification_utils` also uses it to
compute artifact dirs for targeted refresh, where settlement rows are already handled by
a separate branch and a union would double-count.

In `_notification_unread_projection.py`, have
`_reconcile_unread_from_completion_notifications` build its `active_keys` from the new
union helper. Everything downstream is unchanged: the `is_unread_completed_status` gate
still means only terminal rows can go unread, the manual-`U` guard still wins, and
`projection_has_active_completion` still resolves a key to its owning agent node.

Two consequences to keep, not fight:

- An agent row that was already read can go back to unread when its settlement row
  arrives. That is the intended signal — new information landed about that agent — and
  it is exactly what makes the dismissal reachable.
- The unread-agent count, the unread-agent jump, and the bulk read toggle all now
  include those rows, because they all read `_unread_completed_agent_ids`.

### 5.3 Tests and verification

- `tests/ace/tui/test_agent_unread_projection.py` — an active settlement row naming a
  terminal agent's exact key marks that row unread; a settlement row naming a different
  `raw_suffix` does not; a settlement row naming a **running** agent does not (the
  terminal-status gate still holds); dismissing the settlement row clears the marker on
  the next reconcile; a row manually marked unread with `U` is still not re-cleared.
- One end-to-end ordering test: completion read first, settlement arrives second, row
  returns to unread, selecting it dismisses the settlement row.
- `just fix` inline, then `sase tool run check`.
- This phase changes which rows render an unread marker. If `just check` or CI reports a
  TUI PNG golden diff, run `just fix-tui-screenshots` for the affected selectors; do not
  run `just check-full` unless a gate explicitly demands it.
- Run `sase bead epic-symbols <this phase's bead id>` before closing the phase.

## 6. Out of scope

- Changing which notifications `epic_launch.py` sends, or folding the standalone
  `notify_workflow_complete("epic-launch", …)` path into the completion payload. The
  standalone row exists because there is no deferred planner completion to fold into;
  removing it would lose the launch result entirely.
- `PlanApproval`, `EpicApproval`, and `UserQuestion` rows. Those stay explicit `y`/`n`
  response workflows and must never be dismissed by row navigation.
- Widening the agent-less `DismissAgentCompletions` primitive, or the `cl_name`-only
  fallback in the existing completion matcher.
- Matching settlement rows by `family_root_suffix` / `agent_root_timestamp`. Host-side
  key expansion already gives family nodes their descendants' keys; matching on the root
  suffix in Rust would let one agent's read clear a sibling's settlement row.
- A feature flag. This is a straightforward behavior improvement, not a deprecation, and
  no old branch needs to stay reachable — see `sase/memory/sase_flags.md`.

## 7. Done when

- Reading an agent row in the Agents tab — by selection, unread-agent jump, `U` toggle
  back to read, or the bulk read toggle — dismisses any active `epic-launch` or
  `monitor-settlement` notification naming that row's exact `(cl_name, raw_suffix)`, and
  the notification indicator drops it in the same frame.
- Dismissing or marking an agent row dismisses those rows too, through the same store
  operation.
- A settlement row naming a different agent under the same `cl_name` is untouched.
- A settlement row that arrives after its agent was read re-flags the row unread, and
  reading it again clears both.
- `sase notify list --sender epic-launch --unread` no longer accumulates rows for agents
  the user has already acknowledged.
