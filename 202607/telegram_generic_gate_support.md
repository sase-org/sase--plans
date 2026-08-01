---
tier: epic
title: Registry-driven Telegram support for every SASE gate kind
goal: 'Telegram renders and resolves every registered notification-gate kind — including
  TaskTriage and any kind added later — from the shared gate adapter registry, with
  the full option keyboard, command execution, and feedback contract that ACE already
  provides.

  '
phases:
- id: core-registry
  title: Adapter-owned gate capabilities and in-repo adoption
  depends_on: []
  size: medium
  description: 'core-registry: add default_feedback, generic_form, and branch_actionable
    to GateAdapter, collapse the duplicated `kind == "custom"` feedback derivations
    onto the adapter, and replace ACE''s hardcoded gate-action and gate-kind literals
    with registry lookups.

    '
- id: telegram-gates
  title: Registry-driven gate rendering and resolution in sase-telegram
  depends_on:
  - core-registry
  size: medium
  description: 'telegram-gates: replace the six hardcoded action/kind allowlists in
    the sase-telegram plugin with registry lookups and rename the custom-gate formatter
    into an adapter-driven generic gate formatter, so TaskTriage and every future
    kind render with buttons, attachments, and a working callback path.

    '
- id: telegram-optional-feedback
  title: Optional-feedback affordance for Telegram gate branches
  depends_on:
  - telegram-gates
  size: small
  description: 'telegram-optional-feedback: add an `f<branch>` callback and per-branch
    feedback button so Telegram can attach optional feedback to a gate selection,
    matching the ACE gate form.'
proposed_by: bbugyi200.athena.qh
create_time: 2026-07-31 12:13:10
status: done
bead_id: sase-ci
---

- **PROMPT:** [prompts/202607/telegram_generic_gate_support.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/telegram_generic_gate_support.md)
- **BEAD:** [sase-ci](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ci/README.md)

# Plan: Registry-driven Telegram support for every SASE gate kind

## Problem

A `bead_task_triage` gate arrived in Telegram as a plain text message with no buttons:

```
🔔 bead-task-triage

Task ready for triage: sase-ch — Regenerate stale sase_beads provider skills
```

That is `_format_generic()` output. The notification is a fully-formed v2 gate — verified against the live record in
`~/.sase/notifications/notifications.jsonl`:

```json
{
  "sender": "bead-task-triage",
  "icon": "✦",
  "files": [".../task_triage/bead-task-triage-sase-ch-98f47f93d5c8-g1/task.md"],
  "action": "TaskTriage",
  "action_data": {
    "bundle_path": ".../bead-task-triage-sase-ch-98f47f93d5c8-g1",
    "preview_path": ".../task.md",
    "request_kind": "task_triage",
    "request_id": "bead-task-triage-sase-ch-98f47f93d5c8-g1"
  }
}
```

`docs/notifications.md` already documents the intended behavior — "ACE, Telegram, and mobile render branches in query
order from the same normalized envelope structure", and for `TaskTriage` specifically "all client surfaces use the same
host-side side effects". The Telegram implementation does not match its own documented contract.

## Root cause

`sase.notification_gates.adapters` is the registry of gate kinds. `GateAdapter` already carries everything a surface
needs (`kind`, `action`, `sender`, `neutral_only`, …) and core code such as `src/sase/notifications/pending_actions.py`
already derives its tables from `registered_gate_kinds()`. The Telegram plugin predates that registry and instead
carries **six independent hardcoded allowlists**, each enumerating the same five legacy actions. `TaskTriage` — added to
the registry later — is absent from all six, and so is every kind added in the future:

| #   | Repo / file                                                   | Symbol                                                        | Effect of the omission                                                                                                |
| --- | ------------------------------------------------------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| 1   | sase-telegram `src/sase_telegram/scripts/sase_tg_outbound.py` | `_ACTIONABLE_ACTIONS`                                         | No pending-action record and no shared transport record are written, so even a correct keyboard could not be answered |
| 2   | sase-telegram `src/sase_telegram/formatting.py`               | `format_notification()` `match`                               | Falls through to `_format_generic()`: no keyboard **and** the `task.md` preview attachment is dropped                 |
| 3   | sase-telegram `src/sase_telegram/formatting.py`               | `_GATE_DEFAULT_ICONS`                                         | No default icon for unregistered actions                                                                              |
| 4   | sase-telegram `src/sase_telegram/gate_flow.py`                | `load_gate_view()` `action_by_kind`                           | `GateError("invalid_request", "kind", "unsupported Telegram gate kind")` for any unlisted kind                        |
| 5   | sase-telegram `src/sase_telegram/scripts/sase_tg_inbound.py`  | `_GATE_KIND_BY_ACTION`                                        | Button taps answer "This request has expired"                                                                         |
| 6   | sase-telegram `src/sase_telegram/inbound.py`                  | `resolve_gate_response()` allowlist and `_ACTIONABLE_ACTIONS` | Selections are refused, and externally-handled gates never have their keyboards cleaned up                            |

Two secondary defects share the same shape:

- **Duplicated feedback default.** The expression `"optional" if kind == "custom" else "disabled"` is written out four
  times in the sase repo (`notification_gates/hashing.py`, `notification_gates/executor.py`,
  `notification_gates/model_request.py`, `ace/tui/actions/agents/_notification_custom_gate.py`) and a fifth time in
  sase-telegram `gate_flow.py`. A surface that drifts from the executor renders a gate under one feedback contract and
  resolves it under another.
- **Optional feedback is unreachable from Telegram.** `_handle_gate_callback()` only starts the two-step text flow when
  `feedback_mode(...) == "required"`. ACE's gate form exposes a feedback input whenever the mode is not `disabled`, so
  `TaskTriage`'s `launch` option (declared `feedback: optional`) can carry launch feedback in ACE but never in Telegram.

ACE is not free of the same class of bug — `_notification_custom_gate.py` hardcodes `{"custom", "task_triage"}` and
`_notification_modal_flow.py` hardcodes `{"CustomGate", "TaskTriage"}` — but ACE happens to list `TaskTriage`, so it
works today and would only break on the _next_ new kind.

## Approach

Make the registry the single source of truth for "which gate kinds exist and how surfaces should treat them", then
delete every hardcoded allowlist that duplicates it. New kinds registered in `_ADAPTERS` then light up in Telegram with
no plugin change.

Three capability fields are added to the existing `GateAdapter` dataclass rather than new module-level helpers. This is
deliberate: `symvision` flags unused public _functions and classes_, and a helper consumed only by the sase-telegram
repo would need a URI pragma, which CI resolves against sase-telegram's published `main` and therefore fails until that
repo lands. Dataclass fields on an already-public class carry no such hazard, and every field added here also gains an
in-repo consumer inside the `core-registry` phase.

---

## Phase `core-registry`: Adapter-owned gate capabilities and in-repo adoption

Repo: **sase** (this repo).

### Extend `GateAdapter`

In `src/sase/notification_gates/adapters.py`, add three fields to `GateAdapter` and populate them in `_ADAPTERS`:

- `default_feedback: GateFeedbackMode = "disabled"` — the feedback mode an option inherits when its envelope entry omits
  `feedback`. Set `"optional"` on the `custom` adapter only; every other adapter keeps `"disabled"`. This must reproduce
  today's behavior exactly.
- `generic_form: bool = False` — the gate is rendered by a surface's shared generic gate form, driven only by its
  envelope, with no kind-specific body. Set `True` on `custom` and `task_triage`.
- `branch_actionable: bool = True` — the gate is resolved by selecting branches and submitting through
  `execute_gate_selection`. Set `False` on `question` only, which owns a bespoke multi-question form on every surface.

Do not overload the existing `neutral_only` flag for `generic_form`. `neutral_only` means "has no legacy bundle layout"
(`paths.py` uses it to refuse legacy resolution); the two sets coincide today by accident, not by contract.

Keep `GateFeedbackMode` imported from `sase.notification_gates.models`; watch for an import cycle between `adapters` and
`models` and use the existing import direction (`adapters` already imports from `models`).

### Collapse the duplicated feedback derivation

Replace `"optional" if <kind> == "custom" else "disabled"` with the adapter's `default_feedback` at all four sase-repo
sites:

- `src/sase/notification_gates/hashing.py` — `adapter` is already in scope.
- `src/sase/notification_gates/executor.py` — `_options_from_envelope()` reads `envelope["kind"]`; resolve the adapter
  with `adapter_for_kind()` and take its `default_feedback`.
- `src/sase/notification_gates/model_request.py` — same treatment for its local `kind`.
- `src/sase/ace/tui/actions/agents/_notification_custom_gate.py` — `load_and_verify_bundle()` already returns the
  adapter as its second element, which the current code discards as `_adapter`; use it.

### Replace ACE's hardcoded gate literals

- `src/sase/ace/tui/actions/agents/_notification_custom_gate.py`: the guard
  `bundle.kind not in {"custom", "task_triage"}` becomes a `generic_form` check on the resolved adapter.
- `src/sase/ace/tui/actions/agents/_notification_modal_flow.py`: the dispatch arm
  `result.action in {"CustomGate", "TaskTriage"}` becomes a `generic_form` check.
- `src/sase/ace/tui/actions/agents/_notification_modal_flow.py` and
  `src/sase/ace/tui/modals/notification_modal_actions.py` contain several six-element action tuples. Check each one
  individually before replacing it — they are **not** all the same set. The two in `notification_modal_actions.py` and
  the one around `_notification_modal_flow.py:182` enumerate every gate action and should become
  `PRIVILEGED_GATE_ACTIONS`; the tuple around `_notification_modal_flow.py:164` deliberately omits `HITL`, so leave it
  alone or express the exclusion explicitly rather than silently widening it.

`ACTION_BADGES` and `ACTION_ICONS` in `src/sase/ace/tui/modals/notification_modal_constants.py` also cover non-gate
actions (`JumpToAgent`, `ViewReport`, `memory_review`, …). Leave them as presentation tables; only make sure the
`ACTION_ICONS[None]` fallback still applies to a gate action that is absent from the table.

### Tests and docs

- Add a registry test asserting every entry in `_ADAPTERS` declares the three fields, that `custom` is the only adapter
  with `default_feedback == "optional"`, that `question` is the only adapter with `branch_actionable is False`, and that
  `generic_form` is `True` exactly for `custom` and `task_triage`.
- Add a regression test pinning the collapsed derivation: an envelope of each kind resolves to the same option feedback
  modes as before the change.
- Update `docs/notifications.md` where it describes the typed projections table and the shared branch-rendering
  contract, stating that surfaces derive kind capabilities from the adapter registry and that a newly registered kind is
  actionable on every surface without per-surface changes.

### Verification

Run `just install` first (this repo's workspaces have drifted dependencies), then `just check`. Pay attention to the
`symvision` stage: it should stay clean because every new field has an in-repo consumer added in this same phase. If a
new public _function_ turns out to be unavoidable and its only consumer is sase-telegram, prefer restructuring to avoid
it; `--epic-symbol <bead_id>(<symbol>)` in the `Justfile` is the sanctioned fallback while this epic is in flight.

---

## Phase `telegram-gates`: Registry-driven gate rendering and resolution in sase-telegram

Repo: **sase-telegram**. Open it with `/sase_repo` (`sase repo open sase-telegram -r "<reason>"`) and use the printed
path as the only path for reads and writes. Build and test with that checkout's own `just install && just check`.

This phase must land TaskTriage working end to end — rendering a keyboard whose buttons do not resolve would be a worse
state than today's plain message.

### `src/sase_telegram/gate_flow.py`

- `load_gate_view()`: delete the `action_by_kind` literal. Resolve
  `adapter_for_kind(expected_kind or action_data["request_kind"])` from `sase.notification_gates.registry` and use
  `adapter.action` for `resolve_action_bundle()`. `adapter_for_kind()` raises `GateError("unknown_gate_kind", ...)` for
  an unregistered kind; either let that propagate or translate it to the existing `invalid_request` shape — the callers
  already treat any `GateError` as "controls unavailable", so pick one and keep it consistent.
- `load_gate_view()`: replace the local `"optional" if adapter.kind == "custom" else "disabled"` with
  `adapter.default_feedback`.
- Reject a non-`branch_actionable` adapter (i.e. `question`) here, so the generic path can never be pointed at a
  question bundle.

### `src/sase_telegram/formatting.py`

- Rename `_format_custom_gate()` to `_format_gate()` and make it adapter-driven: resolve the adapter from
  `notification.action`, pass `expected_kind=adapter.kind` to `load_gate_view()`, and keep the existing "Controls
  unavailable" fallback for an unreadable bundle.
- Title: keep the header shape `{icon} *{title}*\n*From:* {sender}`. Derive the title in the plugin — presentation
  belongs on the presentation side — from a small table that preserves today's text plus a generic fallback that splits
  the `CamelCase` action into words. `CustomGate` must keep rendering **Custom Request** so existing assertions in
  `tests/test_custom_gates.py` and `tests/test_formatting.py` stay meaningful; `TaskTriage` then renders **Task
  Triage**, and a future `FooBarGate` renders **Foo Bar Gate** with no code change.
- `format_notification()`: keep the explicit `match` arms for the kinds that have kind-specific Telegram bodies
  (`PlanApproval`/`EpicApproval`, `LaunchApproval`, `HITL`, `UserQuestion`). In the `case _` arm, before the existing
  sender-based dispatch, route any notification whose action resolves to a registered adapter through `_format_gate()`.
  That covers `CustomGate`, `TaskTriage`, and every kind added later.
- `_GATE_DEFAULT_ICONS`: keep the table (a notification-authored `icon` already wins for every gate created through
  `create_gate`) but make `_notification_gate_icon()` fall back to a gate-appropriate default for a registered action
  that is absent from the table, instead of the generic `🔔`.
- Inline preview: `_format_gate()` currently shows only `notes` and defers the body to the attachment. Render
  `action_data["preview_path"]`, when present and readable, as a truncating expandable blockquote so the bead
  description is visible in the message itself. `_format_launch_approval()` already implements exactly this
  truncate-and-wrap loop over `MAX_MESSAGE_LENGTH`; factor that loop into a shared private helper and call it from both
  rather than copying it. Keep returning `list(n.files)` so the attachment still flows to `_run_outbound`, which
  converts `task.md` to PDF.

### `src/sase_telegram/inbound.py`

- `resolve_gate_response()`: replace the five-element `action_name not in {...}` allowlist with a registry check — the
  action must resolve to an adapter and that adapter must be `branch_actionable`.
- `_ACTIONABLE_ACTIONS` (used by `find_externally_handled()`): replace with `PRIVILEGED_GATE_ACTIONS`. The
  `UserQuestion` special case inside the loop stays as is.

### `src/sase_telegram/scripts/sase_tg_inbound.py`

- Delete the `_GATE_KIND_BY_ACTION` literal and resolve the expected kind through `adapter_for_action()`, rejecting an
  adapter that is not `branch_actionable`.

### `src/sase_telegram/scripts/sase_tg_outbound.py`

- Replace `_ACTIONABLE_ACTIONS` with `PRIVILEGED_GATE_ACTIONS` so every gate notification gets a pending-action record
  and a shared transport record. The `PlanApproval`/`EpicApproval` and `LaunchApproval` `entry` enrichments below it are
  kind-specific and stay as they are.

### Tests

Extend `tests/test_custom_gates.py`, which already has the right harness (`gate_home` fixture, real `create_gate`,
`_run_outbound`, `_handle_callback`):

- A TaskTriage end-to-end test: build the gate with `sase.bead.task_gate.create_task_triage_gate`, run `_run_outbound`,
  and assert the message carries a two-button keyboard (🚀 Launch, ✕ Close), that `task.md` is attached, and that a
  pending action was written. Then tap Launch and assert the shared executor ran the option command.
  `apply_side_effects` for `task_triage` calls `launch_task_triage()`, which resolves a real project and submits a
  background task — monkeypatch `sase.bead.task_gate.launch_task_triage` (the adapter imports it lazily inside the
  function, so patching the module attribute works). Also cover Close, whose `feedback: required` must route through the
  two-step text flow via `_handle_text_message`.
- A registry-coverage regression test — the guard that makes this fix durable. Iterate `registered_gate_kinds()`, and
  for every adapter that is `branch_actionable`, assert the action is accepted by `resolve_gate_response()`'s guard and
  resolves to a kind through the inbound script's lookup; for every `generic_form` adapter, assert
  `format_notification()` returns a non-`None` keyboard. A kind added to `_ADAPTERS` later without Telegram support then
  fails this test instead of silently degrading to a plain message.

### Docs

Update `docs/inbound.md` and `docs/outbound.md` to describe gate handling as registry-driven rather than listing the
supported actions. Do not hand-edit `CHANGELOG.md` — that repo uses release-please.

---

## Phase `telegram-optional-feedback`: Optional-feedback affordance for Telegram gate branches

Repo: **sase-telegram**, opened with `/sase_repo` as above.

ACE shows a feedback input whenever the selected options' strongest feedback mode is not `disabled`. Telegram only ever
collects feedback for `required`. Close that gap without changing the default one-tap path.

- `render_gate_keyboard()` in `src/sase_telegram/formatting.py`: for a branch whose current selection resolves to
  `feedback_mode(...) == "optional"`, emit one extra button encoded as `f<branch_index>`, labelled with the branch's own
  label (for example `💬 Launch with feedback`). Put it on its own row beneath a singleton branch, and beside the submit
  control for an expanded AND branch. Branches whose mode is `required` already collect feedback through their primary
  control and get no extra button; `disabled` branches get none either.
- `branch_for_token()` in `src/sase_telegram/gate_flow.py` is already prefix-generic (`_token_index` takes the prefix as
  a parameter), so `f` needs no new parsing helper.
- `_handle_gate_callback()` in `src/sase_telegram/scripts/sase_tg_inbound.py`: add an explicit `f`-prefix arm before the
  final `else` (which currently assumes an `s` submit token and would answer "Invalid gate callback"). Resolve the
  branch, compute the same `selected_option_ids` the primary control would submit, and call the existing
  `_begin_gate_feedback()`. That already persists progress, records the awaiting-feedback entry keyed by message id,
  clears the keyboard, and prompts for text; `process_text_message()` then resolves the gate with the typed feedback.
  Nothing new is needed on the resolution side.
- Keep `feedback_is_command_input()` in the path so an option that declares `feedback` in its `input_schema.required`
  still receives it as command input.

### Tests

In `tests/test_custom_gates.py`, drive a gate whose selected option declares `feedback: optional` (TaskTriage's `launch`
is exactly this) through the `f<branch>` callback, send a text message, and assert the recorded response carries the
feedback while a plain tap on the primary button still resolves with no feedback. Add a keyboard-shape assertion in
`tests/test_formatting.py` covering all three feedback modes.

---

## Rollout

`sase-telegram` is a separately versioned package installed into the runtime environment. After the plugin phases land,
the deployed chop must be reinstalled before the fix takes effect; a Telegram message that still arrives without buttons
after the merge means a stale install, not a code defect.

The existing `bead-task-triage` gates in `~/.sase/interaction_requests/task_triage/` remain pending and are picked up by
the reconcile loop in `src/sase/scripts/sase_chop_bead_task_triage.py`, so no data migration is required — the next
outbound pass renders them correctly.

## Out of scope

- Gate `operations` (the plan tier's `edit_file` review step). Telegram has no editing surface; plan gates keep their
  existing singleton feedback branch.
- The mobile surface. It is named alongside ACE and Telegram in `docs/notifications.md` but was not part of the report
  and is not audited here.
- Moving the notification-gate registry into the Rust core. The registry is Python in this repo today; this plan makes
  the existing registry authoritative rather than relocating it.
