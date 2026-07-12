---
create_time: 2026-07-09 01:08:32
status: done
prompt: 202607/prompts/telegram_plan_fork_button.md
tier: tale
---
# Telegram Plan Approval Fork Button

## Problem

After a Telegram plan approval callback succeeds, `sase-telegram` sends a small confirmation message such as
`Plan approved` with a `📋 Plan` copy button. That button copies the archived or SDD-relative plan path. The requested
product behavior is to send a `🍴 Fork` copy button instead, matching the existing Telegram fork buttons used for
completed agents and launch confirmations.

The relevant current paths are:

- `sase-telegram/src/sase_telegram/scripts/sase_tg_inbound.py`
  - `_send_plan_confirmation()` builds the current `📋 Plan` `CopyTextButton`.
  - `_handle_callback()` calls `_send_plan_confirmation()` for `approve`, `commit`, `epic`, and `legend` plan responses.
- `sase-telegram/src/sase_telegram/formatting.py`
  - `_format_workflow_complete()` already builds a `🍴 Fork` `CopyTextButton` that copies `#fork:<agent_name> `,
    optionally prefixed by a VCS workflow tag adjusted to the ChangeSpec.
- `sase/src/sase/notifications/senders.py` and `sase/src/sase/llm_provider/_plan_utils.py`
  - PlanApproval notifications already carry `agent_name`, `agent_cl_name`, project, timestamp, model, provider, and
    runtime metadata, but they do not currently carry the original prompt or extracted VCS tag needed for exact parity
    with completion fork buttons.

## Desired Behavior

For successful Telegram plan approval actions, the follow-up confirmation message should keep the same confirmation text
but replace the `📋 Plan` path-copy button with a `🍴 Fork` copy button.

The copied text should be:

- `#fork:<planner-agent-name> ` when only the planner agent name is known.
- `<vcs-tag-for-current-changespec> #fork:<planner-agent-name> ` when VCS context is available, matching the
  completed-agent fork button pattern.

If a legacy pending action has no usable `agent_name`, send only the confirmation text rather than falling back to the
old Plan-path button. This keeps the new behavior unambiguous and avoids copying a misleading non-fork value.

The plan review keyboard itself should not change. The attached plan PDF/document behavior on the original review
message should not change.

## Implementation Plan

1. Add a shared Telegram helper for fork copy text.

   In `sase-telegram`, factor the existing fork-copy construction out of `_format_workflow_complete()` into a small
   helper, probably in `formatting.py` unless a new focused helper module is clearer. The helper should accept
   `agent_name`, optional raw `prompt`, optional `vcs_tag`, and optional ChangeSpec/CL name. It should preserve the
   existing behavior:
   - Start with `#fork:<agent_name> `.
   - If a prompt is provided, use `sase.xprompt.extract_vcs_workflow_tag()`.
   - If a VCS tag and CL name are available, use `replace_ref_in_vcs_tag()` before `display_vcs_refs_in_text()`.
   - Return `None` when there is no non-empty agent name.

2. Propagate VCS context into PlanApproval action data.

   In `sase`, extend `notify_plan_approval()` and `handle_plan_approval()` with an optional `agent_vcs_tag` or
   equivalent field. In `src/sase/axe/run_agent_exec_plan.py`, pass `ctx.vcs_tag` when a planner submits a plan.

   This is intentionally additive and backward-compatible: older `sase-telegram` versions can ignore the field, and
   newer `sase-telegram` versions still work when the field is absent.

3. Replace `_send_plan_confirmation()` button behavior.

   In `sase-telegram/src/sase_telegram/scripts/sase_tg_inbound.py`, change `_send_plan_confirmation()` so it:
   - Keeps the current confirmation text mapping for `approve`, `commit`, `epic`, and `legend`.
   - Reads `agent_name`, `agent_cl_name` or `cl_name`, optional `prompt`, and optional `agent_vcs_tag`/`vcs_tag` from
     the pending action data.
   - Builds a `🍴 Fork` `CopyTextButton` using the shared helper.
   - Uses no keyboard when the helper returns `None`.
   - No longer computes or exposes the relative plan-file path for this confirmation button.

4. Keep callback and pending-action semantics unchanged.

   Do not change the response JSON protocol, action removal, shared pending-action handling, stale callback handling,
   feedback flow, or plan rejection kill behavior. This is a presentation/copy-text change after a response has already
   been written.

5. Update tests.

   In `sase-telegram`:
   - Update `tests/test_integration.py` expectations from `📋 Plan` / `sdd/tales/plan.md` to `🍴 Fork` / fork prompt
     text.
   - Add coverage for VCS-aware fork text when `agent_vcs_tag` and `agent_cl_name` are present.
   - Add or adjust coverage for missing `agent_name` so the confirmation sends without a copy button.
   - Update any direct formatter tests if `_format_workflow_complete()` is refactored through the shared helper.

   In `sase`:
   - Add/update `tests/test_plan_utils.py` coverage proving `handle_plan_approval()` forwards the VCS tag into
     `notify_plan_approval()`.
   - Add/update notification sender tests if there is existing coverage for PlanApproval `action_data`.

6. Verify.

   For `sase`:
   - Run `just install`.
   - Run focused tests around plan approval metadata.
   - Run `just check` if files were changed.

   For `sase-telegram`:
   - Run `just install`.
   - Run focused tests: `tests/test_formatting.py` and `tests/test_integration.py`.
   - Run `just check`.

## Risks And Decisions

- A pure `#fork:<agent>` fallback is useful even without VCS context, and it matches existing fork button behavior when
  no prompt-derived VCS tag exists.
- The VCS-aware path requires a small host-side metadata addition; trying to infer VCS context in `sase-telegram` from
  only `project_dir` and `agent_cl_name` would be more brittle than passing the already-extracted `ctx.vcs_tag`.
- Keeping the old Plan button as a fallback would conflict with the requested behavior, so the fallback should be no
  button when an agent cannot be identified.
- The `CopyTextButton` value should remain short. The normal fork text is well below Telegram's copy-text limit; if a
  future agent naming pattern makes it too long, the helper can omit the button rather than sending an invalid keyboard.
