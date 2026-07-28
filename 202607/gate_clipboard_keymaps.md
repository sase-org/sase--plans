---
tier: tale
title: Make plan-gate clipboard shortcuts copy the durable plan path and full contents
goal: Swap the plan review clipboard shortcuts so y copies the durable archived plan
  path and Y clearly copies every byte of the reviewed plan text.
create_time: 2026-07-24 21:01:26
status: done
---

- **PROMPT:** [202607/prompts/gate_clipboard_keymaps.md](prompts/gate_clipboard_keymaps.md)
- **AGENTS:**
  - [bbugyi200.athena.k0](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.k0/README.md)
  - [bbugyi200.athena.k0--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.k0.md#member-code)
- **COMMITS:**
  - [9c40093](https://github.com/sase-org/sase/commit/9c400939a9202481f53238ea9410d2442d3632b4) — fix(tui): align plan gate clipboard shortcuts

# Plan

## Context and intended behavior

The `PlanApprovalModal` currently binds `y` to `action_copy_plan` and `Y` to `action_copy_plan_path`. The path action
reads `self._plan_file`, but neutral notification gates pass the owned review resource at
`~/.sase/interaction_requests/<kind>/<request-id>/plan.md` as that value. The same notification already carries the
durable proposal identity in `action_data["original_plan_file"]`; for plans proposed through `sase plan propose`, that
value points at the archived `~/.sase/plans/<YYYYMM>/<name>.md` file.

After this change:

- `y` copies the durable plan path, abbreviated under the user's home directory as `~/.sase/plans/<YYYYMM>/<name>.md`.
- `Y` copies the complete reviewed plan contents. Both the binding description and the visible footer should say
  `Copy all contents` so this is not confused with copying a selection or a path.
- The gate-owned `plan.md` remains the review/edit/hash resource. Only the path placed on the clipboard changes.
- Direct and legacy modal callers that do not provide a distinct durable path continue to copy their existing
  `plan_file` path.

## Implementation

1. In `src/sase/ace/tui/modals/plan_approval_modal.py`, separate the reviewed resource path from the path intended for
   the clipboard.
   - Add an optional, keyword-only durable/copy-path constructor argument and store a fallback to `plan_file` for direct
     and legacy callers.
   - Keep `_plan_file`, `_plan_content`, border-title rendering, file reads, and edit behavior tied to the gate-owned
     reviewed resource.
   - Make the path-copy action use the distinct durable/copy path, retaining the existing home-directory abbreviation
     and success/error notifications.
   - Swap the static bindings so `y` invokes the path-copy action and `Y` invokes the full-content action.
   - Change the binding and footer labels to `y=Copy path` and `Y=Copy all contents`; align the content-copy success
     message with that terminology.

2. In `src/sase/ace/tui/actions/agents/_notification_modals.py`, derive the modal's clipboard path from the
   already-projected `notification.action_data["original_plan_file"]`, falling back to the reviewed `plan_file` when the
   metadata is absent or blank.
   - Pass the same path to both the initial modal and the modal re-opened after editing so an edit round trip cannot
     regress to the interaction-request resource path.
   - Do not replace `plan_file` itself: neutral gate loading, editing, validation, response execution, and resource
     hashes must continue to use the owned gate resource.
   - Keep this lookup to in-memory notification metadata; do not add filesystem I/O to the TUI message-pump path.

3. Add focused coverage in `tests/test_plan_approval_modal_title.py` and `tests/ace/tui/test_notification_plan_gate.py`.
   - Assert the exact swapped binding/action/description pairs and the explicit footer wording.
   - Exercise the shortcuts with preloaded content and a distinct durable plan path, capturing clipboard calls to prove
     `y` copies the abbreviated durable path while `Y` copies the entire content string.
   - Cover the constructor fallback for direct/legacy callers.
   - Extend the neutral notification-gate integration test to prove the `original_plan_file` path reaches the modal
     instead of the bundled `interaction_requests/.../plan.md` path.

4. Refresh the plan-gate PNG goldens whose footer text changes: `plan_gate_tale_five_controls_120x40.png`,
   `plan_gate_frontmatter_120x40.png`, `plan_gate_epic_action_120x40.png`, and `plan_gate_tale_stacked_90x40.png`.
   Inspect the rendered wide and narrow results to confirm both shortcuts remain legible.

## Validation

Run `just install` before repository checks, then run the focused modal and notification-gate tests, the dedicated
visual snapshot suite, and the mandatory full check:

```bash
just install
pytest -q tests/test_plan_approval_modal_title.py tests/ace/tui/test_notification_plan_gate.py
just test-visual
just check
```

The change is complete when the keypress tests prove the clipboard payloads, the neutral gate modal uses the archived
plan identity, the refreshed snapshots show `y=Copy path` and `Y=Copy all contents`, and the full repository check
passes.

## Scope notes

This is presentation-only TUI wiring and does not require a Rust-core change. The copy shortcuts are static modal
bindings rather than configurable shared gate keymaps, so `src/sase/default_config.yml` does not need a corresponding
keymap update.
