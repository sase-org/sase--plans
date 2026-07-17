---
tier: epic
title: First-class custom notification gates with ORed commands, feedback, and icons
goal: 'Agents can mint beautiful, robust custom notification gates on the fly through
  a new /sase_gate skill. Every gate kind, including ones defined in the future, inherits
  graceful free-text user feedback, a per-notification icon, and an ORed command model
  in which the user picks one terminal choice and any subset of proposed add-on commands.
  Plan approval is remodeled onto that ORed model for its sidecar plan-file commit,
  all interactive-surface gate commands run as tracked background tasks visible in
  the ACE tasks tab, and Telegram renders and resolves custom gates completely.

  '
phases:
- id: rust_wire
  title: Core wire support for icons and the CustomGate action
  depends_on: []
- id: gate_model
  title: Custom gate adapter, ORed extras, and shared feedback in the gate service
  depends_on:
  - rust_wire
- id: notify_cli
  title: notify CLI icon support and mechanical gate wait
  depends_on:
  - gate_model
- id: ace_tui
  title: ACE custom-gate modal, icons, and tracked background execution
  depends_on:
  - gate_model
- id: plan_oring
  title: Remodel plan approval onto ORed add-on commands
  depends_on:
  - ace_tui
- id: telegram
  title: Telegram custom-gate rendering, toggles, feedback, and executor routing
  depends_on:
  - plan_oring
- id: gate_skill
  title: Author and deploy the /sase_gate generated skill
  depends_on:
  - notify_cli
- id: rollout
  title: Documentation, end-to-end fixtures, and integrated verification
  depends_on:
  - telegram
  - gate_skill
create_time: 2026-07-16 23:00:15
status: wip
bead_id: sase-6i
---

# Plan: First-class custom notification gates with ORed commands, feedback, and icons

## Context

Epic `sase-6e` landed the unified notification-gate service: neutral bundles under
`interaction_requests/<kind>/<request-id>/`, a versioned `GateSpec` envelope whose terminal choices each carry one
hash-verified argv command, a shared executor with write-once `response.json`, a mechanical producer poller, and
adapter-owned `%auto`. That initiative deliberately kept transport choices a closed typed set. The result today:

- The adapter registry (`src/sase/notification_gates/registry.py`) is a hardcoded five-kind tuple (`plan`, `epic_plan`,
  `question`, `launch`, `hitl`); every kind pins an exact command contract, so agents cannot mint a bespoke gate. `hitl`
  is registered but nothing creates neutral `hitl` bundles, and if one is created via `sase notify create --gate`, both
  ACE and Telegram would answer it through the legacy `hitl_response.json` writers instead of the shared executor — the
  neutral poller would never see the response. That is a real bug.
- One choice = exactly one command. The only way to offer combinations is to multiply terminal choices: plan approval
  ships `approve`/`run`/`tale`/`commit` as four hard bundles of the same two effects (commit the plan file to the plans
  sidecar; run the coder follow-up). Commands within a choice are effectively ANDed; there is no way for the user to
  pick a subset of proposed commands (ORed).
- Feedback is a per-adapter convention (a dedicated choice plus a bespoke input schema and translator) rather than an
  envelope primitive, so a new gate kind gets no feedback support for free.
- No icon exists anywhere: not on `GateSpec.presentation`, not on `Notification` (Python or the Rust
  `NotificationWire`), not on choices. ACE shows text badges from `ACTION_BADGES`; Telegram hardcodes per-formatter
  emoji.
- ACE runs plan/question/launch gate commands as tracked background tasks (`_submit_tracked_task`), which is correct and
  surfaces live output in the tasks tab — but the notification dispatch is a fixed if/elif on `action`, so an unknown
  gate action silently does nothing, and the HITL modal bypasses both the executor and the task queue.
- There is no CLI for a producer to wait mechanically on an arbitrary gate (`sase notify` has only `create`, `list`,
  `show`); only plan/launch/question have built-in waiting call sites.

This epic makes custom gates first-class. Design pillars, in the order the phases build them:

1. **A `custom` gate kind** (action `CustomGate`) with an open, safety-preserving validator, so agents create arbitrary
   gates through the existing trusted machinery instead of a new subsystem.
2. **ORed commands via choice extras**: a gate still resolves with exactly one terminal choice, but a choice may carry
   independently toggleable add-on commands ("extras") that the user opts into before submitting. Extras are
   hash-verified commands like any other; plan approval becomes the flagship consumer.
3. **Feedback as a shared envelope primitive**: every choice declares a feedback mode (`disabled`/`optional`/
   `required`); the executor accepts normalized free text and persists it at a fixed place in the response, so every
   present and future kind inherits graceful feedback on every surface.
4. **Icons everywhere**: an optional single-glyph icon on the notification presentation, on each choice, and on each
   extra, threaded from the envelope through the Rust wire to ACE rows/modals and Telegram messages/buttons, with
   per-kind defaults when unset.
5. **Tracked background execution**: every interactive-surface gate command (terminal + selected extras) runs through
   the tracked task queue, never on the Textual event loop, with status and live output in the ACE tasks tab.

Keep the boundary established in `sase-6e`: envelope, validation, and execution stay Python-first in the main repo;
`sase-core` owns wire shapes, classification, and typed projections. Do not introduce a plugin API for adapters or any
new lifecycle subsystem; `custom` is one more registered adapter whose payload is open.

## Phase 1: Core wire support for icons and the CustomGate action (`rust_wire`)

In `sase-core` (`crates/sase_core`):

- Add an optional `icon` field to `NotificationWire` with a serde default so all existing rows and old readers remain
  valid; extend the notification store contract fixtures and the Python binding validation for round-tripping. This is
  the only `NotificationWire` change; keep it additive.
- Add a `CustomGate` action kind through the centralized gate-action conversion, classified everywhere a typed gate is
  classified: priority/actionability, missing-target state, pending-action conversion, labels, prefix resolution, and
  agent dismissal. Custom gates are agent-dismissable and non-priority by default.
- Add a data-driven generic action detail for custom gates instead of another closed choice enum: choices carry id,
  label, optional icon, feedback mode, and a list of extras (id, label, optional icon, default-selected). Existing typed
  kinds keep their closed detail wires unchanged.
- In pending-action staleness/handled logic, the `custom` kind resolves exclusively via the neutral
  `response.json`/`cancellation.json`; it has no legacy filename.
- Update exports, serde/parity tests, and mobile contract fixtures with a custom-gate example. Let release-plz own crate
  versions.

## Phase 2: Custom gate adapter, ORed extras, and shared feedback (`gate_model`)

In the main repo's `src/sase/notification_gates/` package and its adapters:

- Register the `custom` adapter: action `CustomGate`, pending-action kind `custom_gate`, `auto_policy="forbidden"` (an
  agent must never auto-resolve its own confirmation gate — cover with a creation-time rejection test), neutral bundles
  only. Its validator enforces structural safety (at least one choice, every command an executable bundle `command`
  resource, sane label/icon lengths, valid feedback modes and schemas) while leaving `payload`, choice count, and
  presentation open. Add `CustomGate` to `PRIVILEGED_GATE_ACTIONS` so raw `sase notify create` cannot spoof it.
- Extend the envelope additively (keep `schema_version` 1; readers must accept specs without the new fields):
  - `presentation.icon` and a matching optional `Notification.icon`, validated as a single emoji/glyph grapheme cluster
    of bounded length, stored through the Rust-backed store.
  - `GateChoice.icon` and `GateChoice.feedback` mode (`disabled` default for existing typed kinds, `optional` the
    documented default for custom gates).
  - `GateChoice.extras`: an ordered tuple of extras, each with id, label, optional icon, `default_selected`, and an argv
    command referencing a `command` resource — hashed at creation exactly like terminal commands.
- Extend the executor: the caller submits choice id, validated input, the selected extra ids, and optional feedback text
  (rejected when the choice's mode is `disabled`, required when `required`). After the terminal command succeeds, run
  each selected extra with the same hash re-verification and stdin/stdout contract, then write the single write-once
  response recording the choice result, per-extra results or failures, and the feedback text at fixed top-level keys. A
  terminal-command failure still leaves the gate answerable; an extra failure never unresolves the gate but is recorded
  in the response and the bundle's auditable error log.
- Auto resolution: adapters that permit `%auto` also own their default extras selection so automatic and manual paths
  share one executor call shape.
- Poller: `poll_gate`/`wait_for_gate` results expose choice id, selected extras, and feedback so any producer can
  consume them without a bespoke translator. Migrate the plan/launch/question feedback translators to read the shared
  feedback field first with their legacy in-schema shapes as fallback, without changing their choice semantics or remote
  contracts.
- Route mobile-originated custom-gate responses through the shared executor via the existing mobile notification actions
  integration; mobile transports only carry choice id, extra ids, and feedback text.

## Phase 3: notify CLI icon support and mechanical gate wait (`notify_cli`)

- Raw `sase notify create` accepts an optional `icon` in its stdin JSON with the same validation as gate presentation;
  `sase notify list`/`show` JSON projections include it.
- Add `sase notify wait`: waits mechanically on a gate via the shared poller and prints a stable JSON result (status
  `answered`/`cancelled`/`timeout`, choice id, selected extras, feedback, response path) plus a colored human summary.
  Accept the request by id and kind (matching the descriptor `sase notify create --gate` already prints), honor the
  request's own timeout plus an optional CLI override, and use distinct exit codes per terminal status. Follow the CLI
  rules memory: alphabetized subcommands/options, a short alias for every public long option, excellent `-h` output, and
  no disturbance of the notify group's bare-invocation `list` delegation.
- Update `create` help and the gate-mode descriptor docs for icons/extras/feedback. Cover parser and handler behavior
  with focused tests, including invalid icons and waiting on cancelled/timed-out gates.

## Phase 4: ACE custom-gate modal, icons, and tracked background execution (`ace_tui`)

Follow the repository's long-memory procedure for `sase/memory/tui_perf.md` before TUI work.

- Render `Notification.icon` at the start of inbox rows (falling back to a per-action default icon table that covers
  every registered kind) and in the notification detail header; keep `ACTION_BADGES` text badges as the secondary label.
  Regenerate affected PNG snapshots intentionally.
- Build the generic `CustomGate` modal — this is the beauty centerpiece, so match the polish of the existing plan review
  modal: icon + sender title, notes body with preview/attachment support, one button per choice with its icon, extras
  rendered as checkboxes honoring `default_selected` with per-extra consequence labels, and a feedback input whose
  affordance follows the choice's feedback mode (required feedback blocks submit until text is present). Keyboard
  navigation must match existing modal conventions; if any new keymap is configurable, update
  `src/sase/default_config.yml`.
- Wire the notification dispatch: `CustomGate` routes to the new handler; an unrecognized action now surfaces a graceful
  notice instead of silently doing nothing.
- Execute the submission through `_submit_tracked_task` with reporter phases for the terminal command and each selected
  extra, so status, outcome, and live output appear in the tasks tab; completion raises a toast and refreshes the inbox.
  Nothing runs on the Textual event loop.
- Fix the neutral-HITL bypass: when a notification's `action_data` carries a neutral request id/kind, ACE resolves it
  through the shared executor as tracked work; the legacy direct-write path remains only for legacy bundles.
- Add modal/dispatch/task tests plus PNG snapshots for the new modal states (choices only, extras, required feedback).

## Phase 5: Remodel plan approval onto ORed add-on commands (`plan_oring`)

- Audit the current choice matrix first: `approve`/`run`/`tale`/`commit` differ only in the `commit_plan`/`run_coder`
  protocol pair and archive side effects. Remodel the plan gate so the primary decision is a single **Approve** terminal
  choice carrying two extras — "Commit plan file to the plans sidecar" and "Run coder follow-up" — with defaults
  matching today's recommended `tale` flow; `reject` and `feedback` are unchanged, and `epic` keeps its bundled
  commit-and-launch semantics.
- Preserve compatibility: the envelope keeps the legacy choice ids as presets that map onto Approve plus a fixed extras
  selection, so Telegram/mobile hard-coded remote ids and in-flight gates keep resolving during the compatibility
  window. `%auto` aliases (bare, `plan`, `tale`) resolve to Approve with the adapter's default extras; `%auto:epic` and
  authored-tier conflict rejection are unchanged. Derive the runner-facing `commit_plan`/`run_coder` response fields
  from the selected extras so waiting runners need no change.
- Update the ACE plan review modal to present the extras as checkboxes using the phase-4 component, replacing the
  preset-multiplication of approval buttons; keep single-key bindings for the common presets.
- Update plan gate, auto-approval, response-golden, and side-effect tests; verify the archive-to-sidecar behavior for
  every extras combination matches the audited legacy matrix.

## Phase 6: Telegram custom-gate rendering, toggles, feedback, and executor routing (`telegram`)

In `sase-telegram`, tested against the local SASE/core builds:

- Add a `CustomGate` formatter: icon-led header (per-kind default when unset), sender/notes card in MarkdownV2 with
  expandable blockquotes for long content, one inline button per choice with its icon, and extras rendered as toggleable
  checkbox buttons plus a submit button, reusing the multi-select question interaction pattern with toggle state
  persisted server-side so the 64-byte callback-data limit holds.
- Route callbacks through the shared executor with choice id, selected extras, and feedback; feedback uses the existing
  two-step await-text flow, and a `required` feedback mode goes straight to await-text. Register the action in the
  actionable-action sets on both outbound and inbound sides, and support the externally-handled race guard so a gate
  answered in ACE dismisses the Telegram keyboard.
- Surface the plan remodel: keep the preset buttons stable for the compatibility window, and render the approve extras
  as toggles where the envelope advertises them.
- Align neutral HITL with the executor exactly as ACE does; legacy bundles keep the direct writer.
- Extend formatting/inbound/integration/callback tests, including toggle round-trips, required feedback, duplicate
  callbacks, and unreadable-envelope fallbacks.

## Phase 7: Author and deploy the /sase_gate generated skill (`gate_skill`)

- Add `src/sase/xprompts/skills/sase_gate.md` with `skill: true` (all runtimes uniformly, default `log_skill_use`),
  modeled structurally on the launch-approval skill. The description must do the feature justice: creating beautiful,
  robust, and powerful custom notification gates on the fly, and explicitly positioning it as the easy way to propose
  commands for user confirmation — for example dangerous commands, or commands the user asked to confirm before use.
- The body teaches: when to open a gate; choosing a fitting icon for the notification and per-choice icons (with
  concrete examples); authoring the full gate JSON (choices, extras for ORable follow-ups, feedback modes, resources,
  timeout); creating it with `sase notify create --gate`; waiting with `sase notify wait` or proceeding detached;
  handling the result (choice, extras, feedback). Include the standing warnings: never poll bundle files directly, never
  run bundle commands by hand, and automatic resolution is forbidden for custom gates.
- Add the parametrized entry with expected body phrases to the shipped-skill discovery test, cross-reference custom
  gates from the notify skill source where it lists gate bundles, and regenerate/deploy managed skills through
  `sase skill init --force` and the chezmoi flow.

## Phase 8: Documentation, end-to-end fixtures, and integrated verification (`rollout`)

- Documentation: rewrite the custom-notification sections of `docs/notifications.md` (gate JSON shapes, icons, extras,
  feedback, `wait`, task-tracked execution), update the CLI table in `docs/cli.md`, ACE docs for the new modal and
  tasks-tab behavior, and mobile/integration docs for the `CustomGate` projection. Touch `docs/xprompt.md` only if
  `%auto` alias wording changed in phase 5.
- End-to-end fixtures: create a custom gate and answer it from the ACE, Telegram, and mobile projections; cover extras
  subsets (none, some, all), required feedback, cancellation, gate timeout, duplicate/racing responses, legacy HITL
  fallback, and confirm interactive-surface execution stays in tracked background work rather than the Textual event
  loop.
- Run each repository's complete checks: `just install` + `just check` in SASE including the visual snapshot suite, the
  `sase-core` workspace tests and parity fixtures, and the full `sase-telegram` check against the local SASE/core
  builds. Keep Symvision green, using the epic's allowance mechanism only for symbols that later phases consume.

## Risks and mitigations

- **`NotificationWire` change**: one additive optional field with serde defaults; store contract fixtures prove old rows
  and old readers survive.
- **Write-once response vs extras**: extras execute before response persistence and their failures are recorded, so the
  response stays a single atomic truth and a failed extra cannot wedge the gate.
- **Remote transport compatibility**: plan preset choice ids stay in the envelope through the compatibility window;
  Telegram/mobile resolve choice ids against the envelope, so stale hard-coded ids are rejected exactly as today.
- **Telegram callback limits**: extras toggle state lives server-side keyed by the notification prefix, mirroring the
  proven multi-select question flow.
- **Mobile parity**: custom gates never rely on the Rust-side response-JSON rebuilding path; responses route through the
  Python executor via the mobile actions integration, and the generic detail wire is covered by parity fixtures.

## Acceptance criteria

- An agent can create a custom gate (icon, choices with icons, extras, feedback modes, timeout) via
  `sase notify create --gate`, wait mechanically with `sase notify wait`, and deterministically receive the chosen
  choice, selected extras, and feedback; `%auto` on a custom gate is rejected at creation.
- Feedback and icons are envelope primitives: every registered kind — and any future adapter — inherits them on ACE,
  Telegram, and mobile without transport-specific code.
- Plan approval presents the sidecar plan-file commit and coder follow-up as user-selectable ORed add-ons; legacy choice
  ids, in-flight gates, `%auto` aliases, and runner response fields keep working unchanged.
- Every gate command triggered from an interactive surface (terminal choice and each selected extra) runs as a tracked
  background task with status and live output in the ACE tasks tab; neutral custom/HITL responses always go through the
  hash-verified executor on every surface.
- Telegram renders custom gates beautifully (icon header, per-choice buttons, extras toggles, two-step feedback) and
  resolves them through the shared executor with race-guard support.
- `/sase_gate` is generated for every provider runtime uniformly, its description highlights confirm-before-run command
  proposals, and it instructs agents to select fitting icons.
- All three repositories pass their complete checks, including visual snapshots and core parity fixtures.
