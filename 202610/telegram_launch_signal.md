---
tier: tale
size: medium
title: Land the Telegram coder-launch signal and close sase-1hi.10.7.6
goal:
  Make Telegram claim a coder launch failure only from sase's real gate-turn
  followup_error, keep each decision's choices with its question, add the missing tests,
  and close epic sase-1hi.10.7.6 plus any fully complete plan ancestors.
proposed_by: bbugyi200.apollo.sase-1hi.10.7.6.land
bead: sase-1hi.10.7.6
create_time: 2026-10-09 08:44:22
status: wip
---

- **PARENT:**
  [202610/plan_decisions_finish_gaps.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_finish_gaps.md)
- **BEAD:**
  [sase-1hi.10.7.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.10.7.6.md)

# Land the Telegram coder-launch signal

## Outcome

Phase `sase-1hi.10.7.6.4` closed as done, but its Telegram work is not on
`sase-telegram` `origin/master`. That checkout is still `04f93ad` (release 0.4.28,
2026-10-08 20:29 EDT). No commit message mentions `sase-1hi.10.7.6.4`, and
`NEW_NOTE_MEMORY_PLAN` is absent. Implement the phase contract below in the linked
`sase-telegram` checkout, then close epic `sase-1hi.10.7.6` and walk complete plan
ancestors.

Do not redo the sase-repo phases. They are already on master and survived later commits:

- `1820636212` tint, stale reopen, and 42-cell generic rails. `6bc2a18bbc` moved stale
  handling to `_notification_plan_gate_stale.py` and kept `handle_plan_approval`, the
  off-pump reload, and `esc_drafts` filtering. Generic `.gate-review-actions` is
  `width: 42`. Decision-free `PlanApprovalModal` rails stay `width: 44`. Decisions rails
  stay 50.
- `6bc2a18bbc` replaced the inline-pause-wait loops that `tools/check_test_wait_helpers`
  reported at `tests/ace/tui/test_plan_decision_ace_stale.py:183` and `:310` with
  `sase.ace.testing.wait.wait_for`. The checker exits 0 on that file. Leave the
  remaining `_asyncio.sleep` loop at line 99 alone.
- `96dd8ed270` added the gate route tests and the single restamp record.
  `adapter_plan.py` logs a restamp failure and re-raises; `executor_side_effects.py`
  writes the one durable record. Those files are untouched since that commit.
- `b8480a9997` refreshed the 14 plan and generic gate goldens. No later commit changed
  them. `974d44aa9b` deleted only the updates-tab scope-strip CSS.

## Where to work

Open the linked checkout with `sase repo open sase-telegram` and read its `AGENTS.md`
before editing. Work only there for the code changes. Decision imports stay on
`sase.sdd.plan_decisions` with the existing feature-detect fallback. Any other sase
import is feature-detected the same way. Do not change sase-core. Do not edit sase
memory notes.

Run `sase tool run check` in that checkout. Do not run bare `just check` (the recipe is
guarded) and do not run `just check-full`.

## 1. Coder launch failure

Today `src/sase_telegram/inbound_handlers/gate_completions.py` treats any
`errors/*.json` whose `stage` is in `_LAUNCH_FAILURE_STAGES` (`side_effects`,
`follow_up`, `coder`, `launch`) as "coder could not start". `side_effects` also covers
restamp, archive, and commit failures. `_host_launch_failure` reads
`bundle_path/"meta.json"` and `response["meta"]`, which current bundles do not have, and
it passes `response.json` to
`sase.notification_gates.journal.current_post_response_failure`. That reader matches
`acceptance_id`. `response.json` has none. `decision_receipt.json` does. Journal events
carry the id from the receipt.

sase records a tale coder launch failure on the gate-turn member. Callers read it as
`followup_error` from
`sase.gate_turn.store.find_gate_turn_by_gate_id(project_name, gate_id)`. sase's own
consumer is `_gate_turn_followup_fields` in `src/sase/_plan_approval_response.py`, which
maps `followup_error` to `coder_error`.

Change the claim so the receipt line "Approved with these choices · coder could not
start · retry" appears only when that gate-turn record's `followup_error` is set, or a
recorded error is explicitly a coder launch failure (`stage` of `coder` or `launch`). An
archive, commit, or restamp `side_effects` error must be reported as that failure, not
as a coder launch failure. A clean acceptance must not claim it. Keep the one later
re-edit when the failure lands after the first receipt edit.

Pass `decision_receipt.json` (the mapping that carries `acceptance_id`) into any
`current_post_response_failure` call. Do not pass `response.json`.

Replace
`tests/test_plan_decisions.py::test_launch_failure_claim_only_for_recorded_coder_failure`.
It hand-writes `code: "coder_launch_failed"` with `stage: "side_effects"` and expects
the claim. Drive these shapes instead:

- a stubbed `find_gate_turn_by_gate_id` record with `followup_error` set claims the
  launch failure;
- an archive or commit `side_effects` error does not claim it;
- a clean acceptance does not claim it.

Feature-detect `find_gate_turn_by_gate_id`. A missing sase binding must not raise during
import.

## 2. One blockquote per decision

`_split_expandable_parts` in `src/sase_telegram/decision_sheet.py` puts every decision's
choice lines into one blockquote after all of the asks. Emit one expandable blockquote
per choice decision, directly under that decision's ask, `★` line, and memory lines.
Keep asks, `★` lines, and memory lines outside the quotes. Do not slice a MarkdownV2
escape.

`test_sheet_three_degradations_only` currently partitions on a single `**>` blockquote
and asserts every choice line lives in that one quote. Update it for one quote per
choice decision.

Add fixtures sized so `render_decision_sheet` at the real `DECISION_SHEET_BUDGET` of
1,800 characters picks stage 1 (drop non-default labels), then stage 2 (drop remaining
labels), then stage 3 (per-decision quotes). One fixture per stage. Assert that no
MarkdownV2 escape is split anywhere in the output: every `\` is followed by an escapable
character, not only at the end of the string.

## 3. Tests the phase named and did not land

Add these in `tests/test_plan_decisions.py` or a focused sibling that stays under the
toobig limit. Reuse the existing fixtures.

- Keyboard. `test_keyboard_exact_rows_tokens_and_style` pins rows 0–1, then uses
  `primary_button()` for `style: success`. Pin every row of the decision keyboard,
  including each button's label and callback token: decision rows, toggles, primary,
  `↺ Reset`, Reject, and Feedback. Assert `style: success` on the primary button of the
  rendered keyboard, not on the helper.
- External settle. `keyboard_cleanup._settle_externally_resolved_decision` is only
  tested negatively (`test_external_settlement_surfaces_and_feedback_null_source`
  asserts `False`). Add positive tests: a gate answered in ACE and one answered from the
  CLI, with no Telegram submit. Each edits the card once into the receipt and removes
  the keyboard.
- PDF and the `new` chip. `test_pdf_accepted_values_callouts_and_cleanup` checks
  `"mode" in content` and looks for `🧠` on `format_notification`. Drive the
  accepted-values path through `load_stamped_decisions` on a stamped plan
  (`decision_pdf._stamped_for_source` already calls it). Assert the accepted values and
  the `new` memory chip text for a grant that creates a new note. Do not leave an unused
  temp dir.
- Stale refresh. In the stale-then-Refresh test, bump the bundle's review revision
  before Refresh. Assert that the re-rendered card and its callback tokens carry the new
  revision, and the draft is kept.
- Keyboard-removal retry. `_disable_decision_controls` in
  `inbound_handlers/gate_response.py` saves the retry before the edit and clears it only
  on success. Test that a failed removal leaves the retry record and that the sweep
  retries it. In `_deliver_stale_recovery`, a sent message followed by a failed keyboard
  restore returns False, so the next poll sends the message again. Make the retry
  restore only the keyboard, and test it.

## 4. Close epic sase-1hi.10.7.6

This tale has no land agent. Finish the landing in this same turn, after the Telegram
tests pass. Do not wait for this turn's own commit, its SHA, a push, or CI.

1. Run `sase bead epic-symbols sase-1hi.10.7.6`. It was empty at landing review. If any
   `--epic-symbol` entry is listed, resolve it (wire it up, privatize it, add a non-test
   pragma, or delete it per the Symvision epic-whitelist policy) or, only when a
   still-open later bead still needs the exemption, re-key that Justfile line to that
   open bead. Do not leave the judgment. `sase bead close` refuses while any entry
   remains.
2. Close with:

   ```bash
   sase bead close sase-1hi.10.7.6 --note "<what you verified: Telegram followup_error claim, per-decision blockquotes, the five test groups, and that ace/gate/goldens were already on master>"
   ```

   Never use `--force` merely to make the close succeed, and never use `--force` to
   advance a successful nested landing. If the close is rejected for leftover
   `--epic-symbol` entries, finish that cleanup and close again. If it is rejected
   because named phases were never completed, finish or reopen them, or record the
   outcome deliberately with `--force --reason ... --resolution canceled|superseded`
   only when that outcome is true.

3. Run `just symvision` in the sase repo when it is available, and confirm the whitelist
   is clean.
4. Set `status: done` in the YAML frontmatter of
   `plan:202610/plan_decisions_finish_gaps.md`. Resolve the file with
   `sase artifact path plan:202610/plan_decisions_finish_gaps.md` or
   `sase repo open plans`. Do not hard-code a workspace path.

## 5. Parent plans

After `sase-1hi.10.7.6` is closed, walk these plan ancestors. At landing review each
one's only incomplete descendant was the child below it. Re-read before closing. Stop at
the first incomplete or ambiguous parent, record a note on that parent describing the
blocker, and do not close it or anything above it.

1. `sase-1hi.10.7` (epic, parent of this epic). Phases `.1` through `.5` were closed.
   Linked plan: `plan:202610/plan_decisions_landing_finish.md`. Read the bead, its
   notes, every descendant, and that plan. Check post-child drift. When it is still
   complete, retire leftover `--epic-symbol` entries
   (`sase bead epic-symbols sase-1hi.10.7`), close it with
   `sase bead close sase-1hi.10.7 --note "<what you rechecked>"`, run `just symvision`,
   and set `status: done` on its plan file.
2. `sase-1hi.10` (epic, parent of `sase-1hi.10.7`). Phases `.1` through `.6` were
   closed. Linked plan: `plan:202610/plan_decisions_landing_repairs.md`. Same checks,
   then close, `just symvision`, and `status: done`.
3. `sase-1hi` (epic, parent of `sase-1hi.10`). Phases `.1` through `.9` were closed.
   Linked plan: `plan:202610/plan_decisions.md`. It blocks ready bead `sase-1i3`.
   Closing `sase-1hi` does not close `sase-1i3`. Same checks, then close,
   `just symvision`, and `status: done`.

Do not close a parent that still has an open descendant, an unfinished linked plan, or
an unresolved `--epic-symbol` entry you have not retired or re-keyed.
