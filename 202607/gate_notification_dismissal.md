---
tier: tale
title: Dismiss the gate notification whenever any client answers a gate
goal: Answering or cancelling a SASE gate from any surface always dismisses that gate's
  notification row, and rows whose bundle already reached a terminal state self-heal
  on the next notification-modal open.
create_time: 2026-07-25 10:13:25
status: wip
---

- **PROMPT:** [202607/prompts/gate_notification_dismissal.md](prompts/gate_notification_dismissal.md)

# Plan: Dismiss the gate notification whenever any client answers a gate

## Problem

A custom gate (`sase gate create`) was answered from the ACE TUI, but its notification row stayed live in the
notification panel forever.

Reproduced from live host state (notification `8b9ae5e5-58ab-4fc5-a94a-854c83805a69`, sender `sase-96.7-reclaim`):

- `~/.sase/interaction_requests/custom/custom-036e64e8-.../response.json` exists with
  `"selected_option_ids": ["cleanup","verify"], "source": "tui"` — the gate **was** answered.
- `~/.sase/pending_actions/actions.json` has that entry at `"state": "already_handled"` with `"handled_source": "tui"` —
  the pending-action store **was** updated.
- `~/.sase/notifications/notifications.jsonl` still has that row at `"dismissed": false` — the notification row was
  **not** dismissed, so ACE keeps rendering it.

### Root cause

Notification dismissal is implemented per gate kind and per client instead of once at the shared submission choke point.
The neutral executor only touches the pending-action store:

- `src/sase/notification_gates/executor.py` — `execute_gate_selection()` and `cancel_gate()` both call
  `_mark_pending_handled()` (line 563), which calls only `pending_actions.mark_already_handled()`. It never calls
  `notifications.store.mark_dismissed()`.

Every kind that does get dismissed today does it somewhere else:

| Gate kind          | Where the row is dismissed today                                                        |
| ------------------ | --------------------------------------------------------------------------------------- |
| `plan`/`epic_plan` | `adapters.apply_side_effects()` → `plan_approval_actions.run_plan_side_effects()` (336) |
| `question`         | `user_question_actions._run_question_side_effects()` (530)                              |
| `launch`           | `launch_approval_actions._run_launch_side_effects()` (229)                              |
| `custom`           | **nothing** — `GateAdapter.apply_side_effects()` returns early for non-plan kinds       |
| `hitl` (neutral)   | **nothing** — same early return                                                         |

Clients duplicate the same responsibility: `src/sase/integrations/_mobile_notification_actions.py:75` calls
`dismiss_notification_best_effort()` itself right after `execute_gate_selection()`. That is why the same custom gate
answered from Telegram _does_ disappear while the TUI answer does not — the TUI path
(`src/sase/ace/tui/actions/agents/_notification_gate_execution.py`) has no such call.

So the current behavior is: `custom` and neutral `hitl` gates answered from ACE (or any future client that does not
hand-roll its own dismissal) leave a permanently live notification row.

## Design

**Decision 1 — dismiss at the one shared choke point.** Every gate submission from every surface funnels through
`execute_gate_selection()`, and every cancellation through `cancel_gate()`. Both already resolve the owning
`notification_id` from the bundle envelope inside `_mark_pending_handled()`. Adding the dismissal there fixes all six
registered kinds for ACE, Telegram/mobile, CLI-driven producers, and any future client at once, instead of adding a
sixth copy of the same call.

**Decision 2 — cancellation dismisses too.** `_mark_pending_handled()` is shared by `cancel_gate()`, so cancelled and
timed-out gates (requester cancel, `gate_timeout_seconds` expiry via `notification_gates/poller.py`) will also dismiss
their row. This is deliberate and slightly wider than "submitting an option": a cancelled gate can never be answered, so
its notification is dead weight for exactly the same reason. Flagging it explicitly because it is a behavior change
beyond the literal report.

**Decision 3 — dismissal is best-effort.** The response is already durably persisted (write-once) by the time the row is
dismissed. A failing notification-store write must never turn a successful gate answer into a `GateError`, and must
never propagate out of `cancel_gate()`. Log a warning and continue, matching the existing best-effort dismissals in
`plan_approval_actions.py` / `launch_approval_actions.py`.

**Decision 4 — no Rust change.** Per the Rust core backend boundary rule, cross-client domain behavior belongs in the
core. This behavior already lives in the Python gate layer that all clients share (`notification_gates/`), and
`notifications.store.mark_dismissed()` is already a thin wrapper over the Rust-backed store mutation
(`notification_store_facade.apply_notification_state_update`). Nothing new needs to move into `../sase-core`; no wire or
binding change is required.

**Decision 5 — keep the existing per-kind dismissals.** `plan_approval_actions.run_plan_side_effects()`,
`user_question_actions._run_question_side_effects()` and `launch_approval_actions._run_launch_side_effects()` are also
reached from the _legacy_ (non-neutral bundle) response paths — see `plan_approval_actions.py:164` and
`launch_approval_actions.py:118` — which never enter the executor. Removing them would regress legacy bundles. On the
neutral path the second `mark_dismissed()` is a cheap no-op (the Rust state update reports `changed_count == 0` and does
not rewrite the file). Do not delete them in this change.

## Step 1 — dismiss the notification inside the neutral gate executor

File: `src/sase/notification_gates/executor.py`

1. Add a module logger (`import logging`; `log = logging.getLogger(__name__)`) — the module has none today.
2. Rename `_mark_pending_handled` to `_settle_gate_notification` (it now has two effects) and update its four call
   sites: lines 65, 201, 203 in `execute_gate_selection()` and line 249 in `cancel_gate()`.
3. Extend the body to dismiss the row after marking the pending action handled:

```python
def _settle_gate_notification(
    envelope: Mapping[str, Any],
    response: Mapping[str, Any],
    *,
    source: str,
    action: str | None = None,
) -> None:
    """Mark one gate's notification handled and dismiss its inbox row.

    Runs for every terminal transition of every gate kind, from every client,
    so no surface has to remember to dismiss the row itself.
    """
    notification_id = envelope.get("notification_id")
    if not isinstance(notification_id, str) or not notification_id:
        return
    from sase.notifications.pending_actions import mark_already_handled

    raw_selected = response.get("selected_option_ids")
    selected = (
        [str(option_id) for option_id in raw_selected]
        if isinstance(raw_selected, list)
        else []
    )
    mark_already_handled(
        notification_id,
        source=source,
        action=action or "+".join(selected) or "resolved",
    )
    _dismiss_gate_notification_best_effort(notification_id)


def _dismiss_gate_notification_best_effort(notification_id: str) -> None:
    """Hide a settled gate row without ever failing a persisted response."""
    try:
        from sase.notifications.store import mark_dismissed

        mark_dismissed(notification_id)
    except Exception:
        log.warning(
            "Failed to dismiss notification for settled gate", exc_info=True
        )
```

Behavior notes the implementer must preserve:

- Auto-resolved gates carry `notification_id = None` in the envelope (`service._start_gate_creation`), so the existing
  guard already makes this a no-op for them — no notification exists to dismiss.
- The `already_completed` early-returns (executor lines 63-69 and 199-202) call this helper too. That is intentional:
  re-submitting an already-answered gate repairs a row that a previous run failed to dismiss.
- Keep the call ordering — pending-action state first (transports read it), dismissal second — and keep both inside the
  existing `.response.lock` block so a settled gate never races a concurrent submission.

## Step 2 — self-heal rows whose bundle is already terminal

Step 1 fixes every future neutral-gate submission, but it does not retire rows that are already stuck (including the one
in the report), and it does not cover legacy bundles answered by the pre-gate response writer
(`_notification_hitl_modal._handle_legacy_hitl` → `write_workflow_action_response`, still reachable for HITL bundles
created by older versions).

Add a bounded reconcile pass that dismisses any live gate row whose bundle already reached a terminal state.

1. `src/sase/notifications/pending_actions.py`: promote `_externally_handled()` (line 321) to a public
   `gate_notification_is_terminal(notification: Notification) -> bool`, keep `_state_for_notification()` calling it, and
   add it to `__all__`. It already answers exactly the right question — response present, cancellation present,
   `plan_approved.marker` present, or a consumed request — and returns `False` when the notification has no bundle, so
   it can never dismiss an unanswered gate.

2. New module `src/sase/notifications/gate_reconcile.py`:

```python
"""Self-healing dismissal for gate rows whose bundle is already terminal."""


def reconcile_terminal_gate_notifications(
    notifications: Iterable[Notification],
) -> tuple[str, ...]:
    """Dismiss live gate rows whose bundle already reached a terminal state.

    Repairs rows left behind by a client that answered a gate without
    dismissing it (legacy bundles, external response writers, or any row
    created before dismissal moved into the shared executor).
    """
    from sase.notification_gates.registry import PRIVILEGED_GATE_ACTIONS
    from sase.notifications.pending_actions import gate_notification_is_terminal
    from sase.notifications.store import mark_many_dismissed

    stale = tuple(
        notification.id
        for notification in notifications
        if not notification.dismissed
        and notification.action in PRIVILEGED_GATE_ACTIONS
        and gate_notification_is_terminal(notification)
    )
    if stale:
        mark_many_dismissed(stale)
    return stale
```

3. Wire it into the ACE unread page read only —
   `src/sase/ace/tui/actions/agents/_notification_provider_direct.py::direct_unread_notification_page`:

```python
snapshot = notification_snapshot_from_direct(
    read_notification_snapshot(include_dismissed=include_dismissed)
)
if reconcile_terminal_gate_notifications(snapshot.notifications):
    # Re-read so the returned rows and the unread counts agree.
    snapshot = notification_snapshot_from_direct(
        read_notification_snapshot(include_dismissed=include_dismissed)
    )
```

Placement rationale (TUI perf): `direct_unread_notification_page` is reached only from user-initiated actions —
`_notification_modal_flow._show_notification_modal()` and `_jump_to_agent_notification()`. The periodic auto-refresh
path (`_notification_polling._refresh_notification_count`) uses `direct_notification_count_snapshot` /
`_read_notification_snapshot_from_provider` and is deliberately left untouched, so the recurring refresh gains no extra
filesystem work. The added cost on modal open is a couple of `stat()` calls per _gate_ row within the page limit, on top
of a full JSONL parse that path already performs; the extra snapshot re-read happens only when something was actually
stale.

## Step 3 — tests

Add to `tests/test_notification_gate_execution.py` (uses the existing `gate_home` fixture, which redirects
`INTERACTION_REQUESTS_DIR`, the notifications JSONL, and the pending-action store into `tmp_path`, plus the
`custom_gate_spec` / `gate_spec` builders in `tests/_notification_gates_fixtures.py`):

1. Answering a `custom` gate dismisses its row: `create_gate(custom_gate_spec())` →
   `execute_gate_selection(bundle, ["proceed", "audit"])` → the row from `load_notifications(include_dismissed=True)`
   has `dismissed is True`, and it is absent from `load_notifications()`. Assert the pending entry is still
   `already_handled` so Step 1 did not regress it.
2. Same for a neutral `hitl` gate via `gate_spec()` — proves the fix is kind-independent, not a custom-gate patch.
3. `cancel_gate()` dismisses the row (Decision 2).
4. A failing notification store does not break the answer: monkeypatch `sase.notifications.store.mark_dismissed` to
   raise, then assert `execute_gate_selection` still returns a persisted response and does not raise.
5. Auto-resolved gate (`custom_gate_spec(auto=...)` is forbidden for custom, so use a `question`- or `plan`-kind spec
   with `auto` enabled, or reuse the existing auto coverage): no notification is created and nothing raises.

Add to a TUI-facing test (extend an existing ACE notification provider/modal test module rather than creating a new
top-level one if a natural home exists, e.g. `tests/ace/tui/`):

6. Self-heal: create a gate, write `response.json` into the bundle directly (simulating a legacy/external writer that
   bypasses the executor), call `direct_unread_notification_page(include_dismissed=False, limit=50)`, and assert the row
   is gone from the returned page and `dismissed is True` in the store.
7. Negative case: an _unanswered_ gate row is never dismissed by the reconcile pass.

## Step 4 — docs

`docs/notifications.md`, "Command-backed interaction gates" section (around the paragraph that begins "Manual creation
succeeds only after the bundle, notification row, and pending-action registration are durable"): state the new invariant
— a terminal transition (response or cancellation) marks the pending action handled **and** dismisses the notification
row, for every kind and every surface, and ACE repairs rows whose bundle became terminal without a dismissal when the
notification modal is opened.

## Non-goals

- Do **not** delete the per-kind dismissals listed in Decision 5; legacy non-neutral response paths still depend on
  them.
- Do **not** remove `dismiss_notification_best_effort()` from `_mobile_notification_actions.py:75`. It becomes redundant
  on the neutral path but the call is a no-op once the row is dismissed, and it still covers the legacy-bundle mobile
  paths.
- No change to unread/read semantics, notification priority, tags, or the pending-action 24-hour stale threshold.
- No Rust (`../sase-core`) change.

## Verification

```bash
just install     # ephemeral workspace: dependencies may be stale
just check       # fmt, ruff, mypy, symvision, toobig, sase validation, tests
```

Targeted while iterating:

```bash
.venv/bin/pytest tests/test_notification_gate_execution.py tests/test_notification_gate_cli.py \
  tests/test_pending_actions.py tests/test_gate_e2e_smoke.py tests/test_plan_gates.py -q
```

Manual check: create a throwaway custom gate with `sase gate create`, answer it from `sase ace` with `<enter>`, and
confirm the row disappears from the notification panel and shows `"dismissed": true` in
`~/.sase/notifications/notifications.jsonl`.

## Risks

- **Symvision**: `gate_notification_is_terminal` becomes a cross-module public symbol and must be added to
  `pending_actions.__all__`; the new `gate_reconcile` module needs its own `__all__`. Consult `sase/memory/symvision.md`
  through the `/sase_memory_read` skill if symvision flags unused or private-misuse findings.
- **Over-dismissal**: the reconcile pass keys off `gate_notification_is_terminal`, which returns `False` when a
  notification resolves to no bundle, so non-gate rows and unanswered gates are untouched. Test 7 pins this.
- **Duplicate dismissal cost**: plan/question/launch gates now call `mark_dismissed()` twice on the neutral path. The
  second call finds the row already dismissed and the Rust state update reports no change, so no second file rewrite
  occurs. Confirm this while implementing; if the store does rewrite unconditionally, drop the redundant call from the
  _neutral_ branch only (`adapters.apply_side_effects` keeps it for the legacy branch).
