---
tier: tale
title: Gate Debug view (`d`) for notification gate panels
goal: 'Every notification gate surface in ACE (custom gate, plan/epic approval, launch
  approval, user question, workflow HITL, and the notification inbox rows that back
  them) gains a `d` keymap that opens a beautiful, reliable Gate Debug modal showing
  the full lifecycle, integrity, and raw artifacts of the gate notification — making
  future gate issues diagnosable in seconds.

  '
create_time: 2026-07-17 09:14:36
status: wip
prompt: 202607/prompts/gate_debug_view.md
---

# Plan: Gate Debug view (`d`) for notification gate panels

## Context

Plan approvals, epic approvals, user questions, launch approvals, workflow HITL prompts, and `/sase_gate` custom gates
are all projections of one durable "notification gate" system:

- A bundle on disk at `~/.sase/interaction_requests/<kind>/<request-id>/` containing a canonical `request.json` envelope
  (choices, resources, hashes, producer, timeout, `created_at`), a write-once `response.json` or `cancellation.json`,
  reviewed previews/attachments, adapter-owned command scripts, and an `errors/` directory of per-failure execution
  records (`src/sase/notification_gates/`: `models.py`, `service.py`, `executor.py`, `paths.py`).
- A notification row (`src/sase/notifications/models.py`) whose `action` + `action_data` (`request_id`, `request_kind`,
  `bundle_path`, `request_path`, `response_path`, `preview_path`, `legacy_directory_key`) point back at that bundle.
- One ACE modal per gate kind under `src/sase/ace/tui/modals/`: `CustomGateModal`, `PlanApprovalModal` (plan + epic),
  `LaunchApprovalModal`, `UserQuestionModal`, `WorkflowHITLModal`, dispatched from `NotificationModal` via
  `src/sase/ace/tui/actions/agents/_notification_modal_flow.py`.

When a gate misbehaves today (modal fails to open, hash mismatch, command failure, timeout that "never fired", a gate
that Telegram/mobile can't see), diagnosing it means hand-spelunking bundle directories and the notifications JSONL.
This plan adds a first-class debugging surface: press `d` on any gate panel (or on a gate-backed inbox row) to open a
**Gate Debug** modal that assembles the entire picture in one place.

`d` is currently unbound on all six modals (they only bind `ctrl+d` for scrolling), so no key conflicts exist. Modal
keys are declared as Textual `BINDINGS` class attributes, not through `src/sase/default_config.yml` (that file only
drives main-tab keymaps), so no config changes are needed.

## UX design

### Entry points

Add a `("d", "debug_view", "Debug")` binding to:

1. `CustomGateModal`, `PlanApprovalModal`, `LaunchApprovalModal`, `UserQuestionModal`, and `WorkflowHITLModal` — opens
   the Gate Debug modal for the gate being reviewed. The debug modal is _pushed on top_ of the gate modal; closing it
   returns to the gate modal with all in-progress state (selected choice, checked extras, typed feedback) intact.
2. `NotificationModal` — opens the Gate Debug modal for the highlighted row. This entry point is essential: when a
   bundle is broken the gate modal may fail to open at all (today that surfaces only as an error toast), and the inbox
   row is then the _only_ place debugging can start. For rows with no gate behind them, the modal still opens and shows
   the raw notification row, with gate sections rendering an explicit "No gate bundle attached to this notification"
   empty state — uniform behavior, no dead key.

When a gate modal fails to load from the inbox (e.g. `GateError` from bundle verification), extend the existing error
toast with a hint: "press d on the notification to debug".

Each gate modal receives an optional `debug_context` at construction (default `None`, so existing call sites and tests
keep working). The dispatch handlers in `_notification_modal_flow.py` / `_notification_custom_gate.py` build it from the
notification they already hold. If a modal is somehow opened without context, `d` shows a quiet warning toast instead of
a broken modal.

### The Gate Debug modal

A new `GateDebugModal` (`src/sase/ace/tui/modals/gate_debug_modal.py`), modeled on the existing scrollable-`Syntax`
modal pattern (`LaunchApprovalModal`) plus the `NotificationModal` tab strip.

**Header (fixed):** gate icon + kind badge (per-kind accent color: plan, question, launch, custom, hitl), `request_id`,
and a status pill — `PENDING` (amber), `ANSWERED` (green), `CANCELLED` / `TIMED OUT` (red), `OVERDUE` (red, deadline
passed but no cancellation written yet — itself a useful diagnostic), or `UNKNOWN` (dim, e.g. unreadable bundle) — plus
created-at age.

**Tabbed body** switched with `[` / `]` (matching `NotificationModal` tab conventions), each tab a `VerticalScroll`:

| Tab          | Contents                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Overview`   | Aligned key/value grid: kind, request id, notification id, sender/producer (agent, machine), continuation mode, timeout + deadline + remaining/overdue, status detail (choice id, selected extras, feedback, responded-at when answered; cancellation reason when cancelled); choice inventory (id, label, icon, feedback mode, extras count, argv); resource integrity table (path, role, executable, size, hash ✓ / ✗ MISMATCH / ⚠ missing — re-verified live); pending-action + notification state (read/dismissed/muted/snoozed, tags, stale-window status). |
| `Request`    | The verified `request.json` envelope, pretty-printed with `Syntax(..., "json", theme="monokai")` (same theme as the existing gate preview rendering).                                                                                                                                                                                                                                                                                                                                                                                                            |
| `Response`   | `response.json` (including per-extra `extra_results` and embedded errors) or `cancellation.json` when present; otherwise a styled "pending" placeholder showing what will be written here.                                                                                                                                                                                                                                                                                                                                                                       |
| `Errors (N)` | Entries from the bundle `errors/` directory, newest first: code, message, source, returncode, and bounded stdout/stderr excerpts. Clean empty state when none. Tab label carries the live count.                                                                                                                                                                                                                                                                                                                                                                 |
| `Row`        | The raw notification row exactly as stored in `notifications.jsonl`, pretty-printed JSON.                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |

**Keybindings inside the debug modal:** `[` / `]` switch tabs; `j`/`k`, `ctrl+d`/`ctrl+u`, `g`/`G` scroll; `y` copies
the current tab's raw text; `Y` copies the bundle path; `e` opens the current tab's backing file (`request.json`,
`response.json`/`cancellation.json`, error record, or the notifications JSONL) in `$EDITOR` via the existing suspend
pattern; `d` / `Esc` / `q` close (the same key that opened it toggles it closed). Footer shows the bindings as on other
modals.

### Visual polish

This modal is a showcase surface: consistent per-kind accent colors, single-glyph icons reused from the gate
presentation, ✓/✗/⚠ glyphs in the integrity table, monokai JSON matching the existing preview styling, and a tidy fixed
header/status pill. PNG visual snapshots (below) lock the look in.

## Data assembly: `GateDebugSnapshot`

New module `src/sase/notification_gates/debug.py` with a typed, TUI-independent builder:

- `build_gate_debug_snapshot(notification_row, action_data) -> GateDebugSnapshot` — resolves the bundle root from
  `action_data` (tolerating legacy HITL bundles and missing paths), reads `request.json` / `response.json` /
  `cancellation.json` / `errors/*.json`, re-verifies resource hashes (reusing the existing verification helpers rather
  than duplicating them), derives the lifecycle status (pending / answered / cancelled / timed-out / overdue / unknown),
  and inspects pending-action registration state.
- The snapshot is a frozen dataclass tree: header fields plus one entry per tab with a status (`ok` / `missing` /
  `error(<message>)`) and bounded body text. Every field is best-effort — a corrupt or missing artifact becomes a
  rendered diagnostic (`✗ request.json: <parse error>`), never an exception. That degradation _is_ the debug
  information.
- Bounded reads: cap each artifact (and each stdout/stderr excerpt) at a fixed size with an explicit truncation banner,
  so a pathological bundle cannot bloat the modal.

Placing assembly in `notification_gates/` (not the TUI) keeps it unit-testable without Textual and reusable by a future
CLI (`sase notify debug`) if ever wanted. Rust-core boundary: this introduces no new domain behavior — it is read-only
inspection of artifacts owned by the existing Python `notification_gates` package, so it stays in this repo per the
core-boundary litmus test.

## Performance and reliability rules (per `tui_perf` memory)

- Pressing `d` pushes the modal immediately with the cheap in-hand context (header renders from `action_data` alone);
  all disk work (bundle reads, hashing, JSONL row lookup) runs off the event loop _and_ off the message pump, following
  the same pattern as the existing off-pump custom-gate loader, with results marshalled back to populate the tabs. Tabs
  show a lightweight loading placeholder until then.
- The load task is cancelled if the modal is dismissed before it completes; no state is applied after dismissal.
- Hash re-verification and `errors/` directory listing are part of the off-thread load, never the render path. No
  stat/glob happens per keypress; tab switching only swaps already-built content.
- The modal never raises on malformed input — property: any bytes on disk produce a rendered debug view.

## Implementation outline

1. `src/sase/notification_gates/debug.py` — snapshot dataclasses + `build_gate_debug_snapshot`, with bounded reads,
   status derivation, and hash re-verification reuse.
2. `src/sase/ace/tui/modals/gate_debug_modal.py` — `GateDebugModal` (header, tab strip, scrollable `Syntax` bodies,
   bindings, copy/editor actions) + its TCSS styling alongside the other modal styles.
3. A small shared `GateDebugContext` (notification row + `action_data` projection) and a
   `debug_context_from_notification(...)` helper; thread an optional `debug_context` parameter through the five gate
   modals and their dispatch call sites (`_notification_modal_flow.py`, `_notification_custom_gate.py`, and the
   plan/launch/question/ HITL handlers).
4. `d` bindings + `action_debug_view` on all six modals (including `NotificationModal` row entry point); off-pump load
   wiring; failure-toast hint when a gate modal cannot open.
5. Help surfaces: update the `?` help popup / relevant guide modal content for the new keys, per
   `src/sase/ace/CLAUDE.md`.
6. Docs: `docs/notifications.md` (modal keybindings table + a short "Debugging a gate" subsection pointing at the bundle
   layout) and `docs/ace.md` (gate modal sections and notification modal keys).

## Testing

- **Unit (snapshot builder):** pending / answered / cancelled / timed-out / overdue gates; missing bundle; corrupt
  `request.json`; resource hash mismatch; populated `errors/` directory; legacy HITL row with no neutral bundle;
  non-gate notification; oversized artifact truncation.
- **TUI interaction:** `d` opens the debug modal from each gate modal and from a `NotificationModal` row; closing
  returns to the gate modal with selection/feedback state intact; `[`/`]` tab switching; `y`/`Y` copy; `d` with no
  context produces the warning toast; load-after-dismiss applies nothing. Extend the existing modal test files
  (`tests/ace/tui/test_custom_gate_modal.py`, `tests/ace/tui/test_notification_custom_gate.py`,
  `tests/test_notification_modal_action_bindings.py`) plus a new `tests/ace/tui/test_gate_debug_modal.py`.
- **PNG visual snapshots:** add Gate Debug goldens to `tests/ace/tui/visual/` following
  `test_ace_png_snapshots_custom_gate.py` — at minimum the Overview tab of a pending custom gate with extras, and an
  answered gate's Response tab with an error record present.
- Run `just check` (with `just install` first in a fresh workspace) before finishing.

## Non-goals

- No `sase notify debug` CLI subcommand (the builder is deliberately reusable for one later).
- No changes to the `/sase_gate` skill template or gate creation/execution behavior.
- No Telegram/mobile debug projections.
- No `default_config.yml` keymap changes (modal bindings are class-level, not config-driven).

## Risks

- **Binding collisions in dynamic choice bindings:** `PlanApprovalModal` composes extra choice bindings via
  `review_modal_choice_bindings()`; the implementer must confirm `d` cannot be generated as a choice shortcut there
  (current presets are `a`/`t`/`c`/`r`/`f`/`E`).
- **Modal-on-modal focus/copy-mode quirks:** the debug modal must play well with `CopyModeForwardingMixin` and restore
  focus to the underlying gate modal on close; cover with an interaction test.
- **Large bundles:** bounded reads and off-thread loading are the mitigation; the truncation banner keeps honesty about
  what was dropped.
