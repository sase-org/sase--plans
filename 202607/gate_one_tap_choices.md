---
tier: epic
title: One-tap gate choices — collapse redundant plan-approval buttons
goal: 'Every gate surface presents each decision exactly once. Tale plan reviews show
  only Tale / Approve / Reject / Feedback (epic reviews: Epic / Reject / Feedback),
  custom gates resolve in a single tap using their authored default add-ons with full
  customization demoted to one opt-in affordance, and every programmatic surface (CLI
  kinds, %auto, mobile choice ids) plus every in-flight hashed gate bundle keeps its
  exact current semantics.

  '
phases:
- id: gate-core
  title: Collapse the plan gate contract to preset choices
  depends_on: []
  description: '''Collapse the plan gate contract'' section: shrink the authored tale
    gate to the four preset choices, stop authoring approval extras, preserve legacy-bundle
    replay and programmatic kind/auto/mobile semantics, and tighten creation-time
    validation.'
- id: tui
  title: Remove the extras path from ACE TUI plan review
  depends_on:
  - gate-core
  description: '''Remove the extras path from ACE TUI plan review'' section: delete
    the remodeled approval-extras checkbox path from the plan review modal and its
    gate-execution plumbing, keeping preset keys and the Custom modal as the fine-grained
    surface.'
- id: telegram
  title: Telegram keyboards — four-button plan review and one-tap custom gates
  depends_on:
  - gate-core
  description: '''Telegram keyboards'' section: render tale plan reviews as a fixed
    two-row four-button keyboard, resolve custom-gate choices in one tap with authored
    default add-ons, and add a single Customize affordance for the advanced flow,
    while keeping already-sent messages and in-flight bundles working.'
- id: skill-guidance
  title: sase_gate skill guidance for one-tap defaults
  depends_on:
  - telegram
  description: '''sase_gate skill guidance'' section: update the generated skill source
    so agents author extras whose defaults are safe to run from a bare tap, and regenerate
    the deployed skill files.'
- id: smoke
  title: Cross-surface gate resolution smoke exercises
  depends_on:
  - tui
  - telegram
  model: haiku
  description: '''Cross-surface smoke exercises'' section: exercise the new four-choice
    tale gate and a custom gate end to end across CLI, auto, TUI, and Telegram code
    paths, including a pre-change legacy bundle replay.'
create_time: 2026-07-17 19:01:58
status: wip
bead_id: sase-6o
---

# Plan: One-tap gate choices — collapse redundant plan-approval buttons

## Problem and root cause

A tale-tier plan review currently renders **seven** Telegram buttons:

1. `📖 Tale` 2. `✅ Approve` 3. `❌ Reject` 4. `💬 Send Feedback`
2. `☑️ 💾 Commit plan file to the plans sidecar` (toggle)
3. `☑️ ▶️ Run coder follow-up` (toggle)
4. `✅ Approve with selected add-ons`

Buttons 5–7 are pure duplication of buttons 1–2. The plan-approval decision space is two booleans,
`commit_plan × run_coder`, and the preset choice records in `src/sase/plan_approval_choices.py` already name every
meaningful combination:

| commit_plan | run_coder | Preset choice                    | Toggle-path equivalent                  |
| ----------- | --------- | -------------------------------- | --------------------------------------- |
| yes         | yes       | **Tale**                         | submit with default toggles             |
| no          | yes       | **Approve** (and legacy `run`)   | submit with commit toggled off          |
| yes         | no        | legacy `commit` (CLI-only today) | submit with coder toggled off           |
| no          | no        | —                                | submit with both off (no real use case) |

The root cause is that `src/sase/plan_gate.py` advertises **two overlapping models of the same decision** on one gate:
the preset choices (`approve`, `run`, `tale`, `commit`, `reject`, `feedback`) _and_ two ORable extras (`commit_plan`,
`run_coder`) attached to the `approve` choice. Every surface then renders both models. Telegram additionally preselects
`approve` (`default_choice_id="approve"` in `_format_plan_approval` / `_handle_plan_gate_callback`), so the extras
toggles and their submit row are always visible alongside the presets. The ACE TUI has the same duplication in
`plan_approval_modal.py` (preset keybindings _plus_ a `GateExtrasSelectionList` when the gate carries extras).

The same pattern generalizes to custom gates (`sase_gate` skill): a choice that carries extras or optional feedback
forces a select → toggle → submit two-step on Telegram, even when the authored `default_selected` values are exactly
what the user wants. Choices without extras already resolve in one tap.

## Design principles

1. **One decision, one model.** Every visible choice is a complete, self-describing decision. No toggle may replicate
   what another visible choice already expresses.
2. **One-tap resolution.** The default path resolves in a single interaction using authored defaults. For plan gates the
   presets _are_ the decisions, so the extras disappear entirely; for custom gates a bare tap applies the choice's
   `default_selected` extras.
3. **Progressive disclosure.** Fine-grained control (arbitrary extras combos, optional feedback, coder prompt/model)
   lives behind a single, clearly secondary affordance — never co-equal with the terminal choices.
4. **Reliability over purity.** In-flight hashed bundles, already-sent Telegram messages, older mobile clients, `%auto`,
   and every `sase plan approve --kind` invocation keep their exact current semantics.

## Target experience per surface

- **Telegram, tale plan:** exactly two rows — `[📖 Tale] [✅ Approve]` / `[❌ Reject] [💬 Send Feedback]`. No toggles,
  no submit row. Callback toasts state consequences, sourced from the choice records' `consequence_text` so copy never
  drifts (e.g. "Tale approved — committed to sdd/plans; coder follow-up launched").
- **Telegram, epic plan:** `[📋 Epic]` / `[❌ Reject] [💬 Send Feedback]` (already extras-free; unchanged shape).
- **Telegram, custom gate:** one button per choice; a bare tap resolves the choice with its authored `default_selected`
  extras. When any choice carries extras or optional feedback, one trailing `⚙️ Customize` row switches the message into
  the existing select → toggle → submit keyboard (with a small `↩ Back` to return). Required-feedback choices keep the
  two-step text flow untouched.
- **ACE TUI, plan review:** plan body + preset keys (`enter`=tier default, `a`, `t`, `E`, `c`=Custom, `r`, `f`, `e`, …).
  No extras checkbox list. The Custom modal (`ApproveOptionsModal`) remains the fine-grained surface and is unchanged.
- **CLI:** `sase plan approve [--kind approve|tale|epic|commit]` keeps all four kinds, including commit-without-coder,
  with unchanged reporting.
- **Mobile / remote:** advertised quick actions align with the four-choice model (add `tale`, drop the redundant `run`),
  while legacy ids from older clients are still accepted.
- **`%auto`:** automatic resolution of a tale gate still yields `commit_plan=True, run_coder=True`, now by resolving the
  `tale` choice instead of `approve` + default extras.

## Collapse the plan gate contract (`gate-core`)

All in `src/sase/`, plus tests.

- `plan_gate.py`:
  - `plan_gate_choice_ids("tale")` → `("tale", "approve", "reject", "feedback")`, with `tale` first — the authored tier
    is the primary action and defines button order on every surface. Epic stays `("epic", "reject", "feedback")`.
  - Stop authoring extras: `_plan_gate_choice` no longer attaches extras to `approve`; `plan_gate_extra_ids` returns
    `()` for newly authored gates; extra command resources are no longer emitted. Keep `PLAN_COMMIT_EXTRA_ID` /
    `PLAN_RUN_CODER_EXTRA_ID` and keep `execute_plan_gate_extra_command` accepting those two ids so pre-change bundles
    (whose hashed command scripts call back into current code) still replay.
  - `execute_plan_gate_command`: validate the choice against the **bundle envelope's own choice list** (the envelope is
    hash-verified) rather than the freshly authored set, so legacy bundles containing `run`/`commit` still resolve.
  - Keep `approve`'s `input_schema` overrides (`commit_plan`, `run_coder` booleans) as the programmatic escape hatch;
    drop the input/result schema branches for the no-longer-authored `run`/`commit` choices from the authoring path
    only.
  - `plan_gate_preset_extra_ids` becomes legacy-bundle support only (or is deleted once the TUI stops importing it —
    extras never execute host side effects; the commit/run behavior flows from the result flags, so resolving plain
    `tale` / `approve` on a legacy bundle is already semantically complete).
- `notification_gates/registry.py`:
  - `resolve_auto_choice` for kind `plan` prefers `tale`, falling back to `approve` for legacy bundles. This pins
    today's auto semantics `(commit=True, run=True)` exactly — on a legacy bundle, `approve` + `automatic_extra_ids`
    defaults still produces the same flags.
  - `_validate_plan_spec`: tighten to the new closed choice sets and reject extras on any plan choice. Validation runs
    only at creation (`service.create_gate`), so in-flight bundles are unaffected.
- `plan_approval_choices.py` / `plan_approval_actions.py`:
  - Keep the `run` and `commit` records (protocol mapping, response messages, `persist_action`, status labels) but mark
    them as legacy/programmatic-only.
  - In `_execute_neutral_plan_approval_response`, when the requested kind resolves to a choice id the bundle does not
    advertise (`run`/`commit` on a new bundle), execute gate choice `approve` with input overrides taken from the
    requested record's protocol — and keep all _reporting_ (response message, persist_action, status label) keyed to the
    requested kind, not the executed gate choice, so `sase plan approve --kind commit` still records "commit".
  - `PLAN_APPROVAL_REMOTE_CHOICES` → `("tale", "approve", "reject", "epic", "feedback")`; the shared action layer
    continues accepting `run`/`commit` ids from older remote clients via the same mapping.
  - `_plan_gate_has_approval_extras` and `translate_plan_gate_response`'s extras translation stay for legacy bundle
    responses.
- Tests: update `tests/test_plan_gates.py`, `tests/test_notification_gates.py`, `tests/test_plan_approval_choices.py`,
  `tests/test_mobile_notifications_bridge.py`. Add a **legacy bundle fixture** capturing today's six-choice + extras
  envelope and command scripts, and prove `run`, `commit`, `approve`, `tale`, and `%auto` resolution on it produce
  today's exact response flags.

## Remove the extras path from ACE TUI plan review (`tui`)

- `src/sase/ace/tui/modals/plan_approval_modal.py`: delete the remodeled path — `approval_extras` constructor plumbing,
  `GateExtrasSelectionList` usage, `_has_approval_extras`, `_selected_approval_flags`, `_result_for_extra_ids`, and the
  `plan_gate_preset_extra_ids` import. `enter` approves the tier default (`tale` for tale gates), `a`/`t`/`E` presets
  and `c` (Custom modal) unchanged. Legacy bundles render identically — their extras are simply not shown, and the
  preset choices they also carry express the same decisions.
- `src/sase/ace/tui/actions/agents/_notification_gate_execution.py` and callers: stop extracting/passing approval extras
  for plan gates; custom-gate extras submission is untouched.
- `custom_gate_modal.py` needs no structural change — it is already a single-screen progressive-disclosure design.
- Gate debug rendering (`src/sase/notification_gates/debug_rendering.py`) stays generic (custom gates still have
  extras); refresh any plan-gate debug fixtures and re-accept the affected PNG snapshots in `tests/ace/tui/visual/`
  deliberately with `--sase-update-visual-snapshots`.
- Update `tests/ace/tui/test_notification_plan_gate.py` and `tests/ace/tui/test_plan_approval_modal_title.py`
  accordingly. Keybindings do not change, so the help modal stays as is.

## Telegram keyboards (`telegram`)

Work happens in the `sase-telegram` linked repo — open it with the `/sase_repo` skill and use the printed checkout path;
run its own `just check` there.

- `formatting.py`:
  - `render_plan_gate_keyboard`: render only the preset rows — `("tale", "approve", "epic")` then
    `("reject", "feedback")` from the envelope's choices — and never render extras toggles or the submit row, even for
    legacy bundles (their preset choices cover the same semantics). Drop the plan-gate progress preselection
    (`default_choice_id="approve"`) from `_format_plan_approval`.
  - `render_custom_gate_keyboard`: compact mode renders one row per choice plus a trailing `⚙️ Customize` row when any
    choice has extras or optional feedback. Customize mode is the existing select/toggle/submit keyboard plus a `↩ Back`
    button that clears progress and restores the compact keyboard.
- `gate_flow.py`: add the customize-mode flag to the Telegram-private `GateProgress` state (safe-by-default when loading
  stale/malformed progress).
- `scripts/sase_tg_inbound.py`:
  - Custom gates: a choice tap in compact mode resolves immediately with the choice's `default_selected` extras
    (required-feedback choices still enter the text flow; optional feedback is reachable through Customize). In
    customize mode, current behavior is preserved.
  - Plan gates: keep the `x<i>`/`submit` handlers working for **already-sent messages** whose inline keyboards still
    carry those callbacks, but new renders never emit them. Tapping `Tale`/`Approve` keeps resolving through the shared
    choice records so semantics match the records exactly.
  - Callback answer toasts derive from the records' `response_message` + `consequence_text`, and the one-tap custom-gate
    toast lists which add-ons ran.
- Leave the pre-neutral legacy notification path (the hardcoded three-button keyboard for non-neutral requests)
  untouched.
- Update the sase-telegram test suite for both keyboards and the new inbound flows, including: compact→customize→back
  round-trip, one-tap default resolution, and a stale-progress recovery case.

## sase_gate skill guidance (`skill-guidance`)

- `src/sase/xprompts/skills/sase_gate.md`: document the one-tap contract — on Telegram a bare tap runs the choice with
  its `default_selected` extras, so agents must author defaults that are safe to run without further review; risky
  add-ons should default to unselected and be reached through `⚙️ Customize`. Mention that choices requiring
  deliberation should use `feedback: required`.
- Regenerate deployed skills: `sase skill init --force`, then `chezmoi apply` (generated `SKILL.md` files must never be
  edited directly).

## Cross-surface smoke exercises (`smoke`)

Exercise, do not redesign. In a scratch HOME/project:

- Author a tale plan gate; assert the envelope advertises exactly four choices and zero extras; resolve each choice
  through `execute_gate_choice` and assert the response flags match the records' protocol table.
- Resolve `%auto` on a fresh tale gate and assert `(commit_plan=True, run_coder=True)`.
- Replay the legacy six-choice fixture bundle through `run`, `commit`, and auto paths; assert identical flags to today.
- Drive `sase plan approve --kind commit` against a fresh gate; assert the executed response and the recorded
  kind/status reporting.
- Call the Telegram render functions (via the linked repo checkout) on both a fresh tale bundle and the legacy fixture;
  assert the four-button keyboard with no toggle/submit rows, and assert a custom gate with extras renders compact +
  Customize.
- Run the full `just check` in both repos.

## Compatibility and rollout

- **In-flight gate bundles** are hash-verified and never re-validated at load; the executor accepts whatever choices the
  envelope declares, and legacy extras acknowledgment commands remain executable. No migration step needed.
- **Already-sent Telegram messages** keep functional buttons: preset callbacks flow through the unchanged shared path,
  and toggle/submit callbacks remain handled.
- **Phase ordering is the rollout order**: the core contract lands first, then the surfaces. The Telegram renderer reads
  choices from the envelope, so it renders both old and new bundles correctly regardless of which side deploys first.
- **Rust core boundary**: notification gates are implemented in this repo's Python (`src/sase/notification_gates/`); no
  `sase-core` change is required.

## Risks

- _Auto semantics drift_ — pinned by explicit tests asserting `(True, True)` for `%auto` on tale gates, on both fresh
  and legacy bundles.
- _Reporting fidelity for `--kind commit`_ — mitigated by keying reporting to the requested kind while executing the
  `approve` gate choice with overrides; covered by a dedicated test.
- _One-tap custom gates running unreviewed add-ons_ — mitigated three ways: the defaults are authored intent, the skill
  guidance requires safe-by-default authoring, and Customize remains one tap away.
- _Stale Telegram progress files_ (e.g. a customize session abandoned before the upgrade) — progress loading already
  recovers from malformed state; the new flag defaults to compact mode.
