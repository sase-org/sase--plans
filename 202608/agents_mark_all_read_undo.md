---
tier: tale
title: Reversible agents-tab mark-all-read action
goal:
  Pressing ,u on the Agents tab marks every unread completed lane read, and a second ,u restores that exact batch as
  unread when no lane has become unread in the interim.
proposed_by: bbugyi200.athena.ri
---

- **AGENTS:**
  - [bbugyi200.athena.ri](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ri.md)
- **COMMITS:**
  - [d53f085](https://github.com/sase-org/sase/commit/d53f0856eed0f83076e2afe96ad5b1fb0bee5707) — feat(ace): add undo
    for bulk agent read toggle

# Plan: Make the Agents-tab mark-all-read action reversible

## Outcome and behavioral contract

Keep the existing configurable leader-mode action ID `mark_all_unread_done_agents_read` and its default `,u` binding,
but turn its Agents-tab behavior into a guarded two-state action:

1. When one or more loaded, non-container, terminal agent lanes are unread, `,u` acknowledges that batch exactly as it
   does today, remembers the affected identities for this TUI session, and reports how many lanes were marked read.
2. When there is a remembered batch and no agent lane has been marked unread since that acknowledgment, the next `,u`
   restores the still-loaded, still-terminal members of that batch and reports how many lanes were restored.
3. Whenever notification projection or the manual `U` action adds an unread lane after the bulk acknowledgment,
   invalidate the old undo batch. A later `,u` must mark the new current batch read rather than resurrecting the older
   one. Invalidation must remain effective even if that newly unread lane is individually acknowledged again before `,u`
   is pressed.
4. If there is neither a current unread terminal batch nor a valid restorable batch, retain the current “No unread
   completed agents” behavior.

The undo is deliberately session-local and lane-scoped. The first press keeps the existing notification-dismissal
behavior. Restored identities are added to both the unread set and the manual-unread guard set, using the existing
supported manual-marker path so notification reconciliation cannot immediately clear the restored lane and old
completion notifications do not resurface or alert again. The undo batch is consumed after one restore attempt;
identities that disappeared from the loaded roster, became clan containers, or are no longer terminal are skipped rather
than reintroducing stale state.

## Implementation

### Model the bulk action and pending undo explicitly

- In `src/sase/ace/tui/actions/agents/_unread_state.py`, add a small typed result for the three outcomes (marked read,
  restored unread, no-op) instead of overloading the existing integer return value. Rename the private helper as needed
  so its name describes the toggle semantics, while leaving the keymap configuration action ID unchanged.
- Add session state for the pending bulk-read identity snapshot in `src/sase/ace/tui/actions/_state_init.py`, with
  matching type declarations on the relevant Agents mixins. Keep the snapshot identity-based rather than retaining
  mutable `Agent` instances so agent reloads cannot make the undo use stale row objects.
- Factor a narrow invalidation helper into the unread-state mixin. Call it only when unread identities are newly added,
  not when reconciliation merely preserves existing unread state or removes stale/read identities.

### Implement mark, restore, and invalidation paths

- Preserve the current first-press target selection over the complete loaded roster (`_agents_with_children` when
  present), terminal-status filtering, manual-guard cleanup, completion-notification dismissal/cache cleanup,
  notification-count refresh, metric invalidation, and selective ancestor-aware row repaint. Record the exact target
  identities only after a real batch is selected; a no-op must not arm or overwrite undo state.
- On the guarded second press, resolve the saved identities against the current complete roster, retain only real
  terminal agents, add those identities to both `_unread_completed_agent_ids` and `_manual_unread_agent_ids`, clear the
  pending snapshot, invalidate cached agent-info metrics, and reuse `_repaint_changed_unread_rows` so visible rows, clan
  ancestors, panel titles, summaries, and detail state update through the existing selective fast path. Do not perform
  notification-store I/O during restore.
- In `src/sase/ace/tui/actions/agents/_notification_unread_projection.py`, compare the unread set before and after
  projection and invalidate a pending undo when projection adds any identity. This covers new completion notifications,
  including additions discovered during an agent reload or background notification refresh.
- In the manual-unread add branch of `_toggle_agent_unread`, invalidate pending bulk undo before adding the selected
  identity. Removing or acknowledging an unread identity must not create or revive an undo opportunity.

### Surface the contextual behavior consistently

- Update the `,u` dispatch in `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py` to consume the typed result and
  emit distinct messages for marking read, restoring unread, and a no-op. Retain the current Agents-only behavior and
  leader-repeat bookkeeping.
- Expose a cheap boolean “bulk-read undo available” state to both leader-footer update call sites. Extend
  `src/sase/ace/tui/widgets/_keybinding_modes.py` so leader mode shows `mark all read` while unread completed lanes
  exist and shows an `undo mark all read` label when only the guarded undo is available. Do not add roster scans or I/O
  to rendering; the boolean must come from the session snapshot.
- Update the static command-palette and Agents help descriptions in `src/sase/ace/tui/commands/_mode_commands.py` and
  `src/sase/ace/tui/modals/help_modal/agents_bindings.py` to advertise the reversible behavior. Do not rename the
  configuration key or change `src/sase/default_config.yml`, so existing user overrides remain valid.

## Verification

- Extend `tests/ace/tui/test_agent_unread_toggle.py` to cover the full state machine: first press records and clears an
  automatic/manual mixed batch; second press restores exactly the eligible identities as guarded manual unread rows; a
  third press can mark the restored batch read again; no-op does not arm undo; missing/non-terminal saved identities are
  skipped; and selective repaint/fallback plus metric invalidation remain correct.
- Add invalidation coverage for both production unread writers: a new completion projected after the first press cancels
  the older undo even if that lane is later acknowledged, and manually marking a lane unread with `U` cancels it as
  well. Verify a notification reconciliation after undo preserves the restored lanes because of their manual guards and
  does not recreate completion notifications.
- Update `tests/ace/tui/_leader_keymap_helpers.py` and `tests/ace/tui/test_leader_keymap_dispatch.py` for typed
  mark/restore/no-op outcomes, exact toast text, Agents-only dispatch, and repeat behavior.
- Extend `tests/ace/tui/test_leader_keybinding_footer.py` for the contextual `mark all read` versus `undo mark all read`
  footer labels, and adjust command catalog/help assertions where the human-readable description changes. Keep the
  existing default-keymap tests proving that `,u` and the configuration ID are unchanged.
- Run focused tests for unread toggling/projection, leader dispatch/footer, command metadata, and keymap defaults. Then,
  because implementation changes files in the SASE repository, run `just install` followed by `just check` as required
  by the project instructions.

## Constraints and non-goals

- Do not add notification-store “undismiss” behavior or change the sibling Rust core: this feature restores lane unread
  state without replaying historical notifications across other frontends.
- Do not persist undo state across TUI restarts, stack multiple undo batches, or add a timeout. Only the most recent
  successful bulk acknowledgment is reversible, and any newly unread lane invalidates it.
- Do not replace selective row/panel patching with a full Agents-list rebuild or introduce synchronous I/O in the
  restore path.
