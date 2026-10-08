---
tier: tale
title: ACE compact docked Verdict
goal: "Make the ACE plan review show the compact three-line docked Verdict, tint chosen
  callouts, show the real draft banner, render Carries, and handle settled and stale
  reviews without a render-path stall.

  "
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.10.4
bead: sase-1hi.10.4
create_time: 2026-10-08 10:13:36
status: wip
---

- **PARENT:**
  [202610/plan_decisions_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_landing_repairs.md)
- **BEAD:**
  [sase-1hi.10.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.10.4.md)

# ACE compact docked Verdict

Implement phase bead **sase-1hi.10.4** only. The bead is already `in_progress`. Do not
set its status by hand. Do not close epic **sase-1hi.10**, epic **sase-1hi**, or any
other ancestor. Do not create beads. Record anything outside this tale as
`sase bead note sase-1hi.10.4 'PROPOSED FOLLOW-UP: <summary — detail>'`. This tale edits
no memory notes and does not change sase-core.

The authoritative gap list is section 4 of
`plan:202610/plan_decisions_landing_repairs.md`. Visual language is section 1.6 of
`plan:202610/plan_decisions.md`. The parent design's section 6.6 is the original ACE
spec; this tale lands only the gaps the landing audit still found. `sase bead read`
shows no accepted DECISIONS on this phase or its parent epic.

## Out of scope

- Visual goldens and `just fix-tui-screenshots`. Phase **sase-1hi.10.5** owns the new
  `plan_gate_*` PNGs and the one compact-Verdict update group. `sase tool run check`
  does not collect `tests/ace/tui/visual/**`.
- CLI, Telegram, gate stamping, and accepted-sheet loading. Those are other phases. Call
  the accepted-sheet loader they already shipped; do not reimplement it.
- Keymap, inbox-suffix, and toast work that sase-1hi.6 already landed, except the
  gate-card toggle glyph called out below.
- The known master failures named in the landing epic: the macro-terminology test
  (sase-1hr), the hinted raw-prompt identity test (sase-1hy), the parallel
  `test_candidates_fast_path_child_cpu_budget[snippet]` flake (sase-1g3), and the
  sase-1hp unused-public backlog. Record them as KNOWN if check hits them. Do not fix
  them here. This tale must still clear the four ACE symbols below and add no new
  unused-public symbols.

## 1. Compact docked Verdict

Today `PlanApprovalViewMixin.compose` in
`src/sase/ace/tui/modals/plan_approval_modal_view.py` puts a "Verdict" title and
`GateBranchControls` at the bottom of the scrolling rail. The summary Static renders
only when the plan has decisions. Tale AND members still use the full
`plan_gate_option_label` strings inside `compose_group`, and the Tale submit sits in the
expanded group, apart from Reject and Feedback.

Build three docked lines for the plan review only. Custom gates, sudo, and the generic
`compose_group` / `compose_singleton_row` path stay as they are.
`tests/ace/tui/test_gate_branch_inputs.py` and `tests/ace/tui/test_custom_gate_modal.py`
must keep passing unchanged.

- Dock a `#plan-verdict` container to the bottom of `.gate-review-actions` so the three
  lines stay on screen while Decisions scroll. Always compose it, with or without
  decisions, so the rail does not jump.
- Keep the existing width rule: 42 cells, widening to 50 only with
  `.gate-review-actions--decisions`. Leave the 100-column narrow breakpoint alone.
- Line 1, tale only: the approve and commit AND members as `☑️`/`⬜` toggles with short
  labels `🚀 Launch coder` and `💾 Commit plan`. The full labels (`Launch coder agent`,
  `Commit plan file to the plans sidecar`) are tooltips. Space still flips the focused
  member through the existing `GateBranchControls` selection state.
- Line 2, one row: `1 ✅ Tale  2 ❌ Reject  3 💬 Feedback`. Epic reviews use `1 ✅ Epic`
  and have no commit toggle. Numbered keys, Enter, and mouse still submit the same
  branches.
- Line 3: the full summary sentence on every plan review. With decisions, use
  `PlanDecisionDraft.full_summary`. Without decisions, still render the verdict sentence
  (`→ coder + commit`, `→ coder`, `→ commit`, or `→ epic launch`) so the third line is
  present.

The buttons remain `GateControlButton`s wired to the existing branch indices. Do not add
a second submit path.

## 2. Decision rows

In `src/sase/ace/tui/modals/plan_decision_rows.py`:

- An expanded choice header shows the current value and `●` when the value differs from
  the default. The collapsed header already does this.
- An unverified memory row (`quote_not_found`, including a default-true row whose
  effective default was forced off) keeps
  `⚠ "<quote>" — not in your messages · off until you turn it on` after a human sets it
  to yes. Do not replace that line with `you asked: "..."`.
- The same unverified warning is visible on the collapsed row, not only the expanded
  row.
- A resolved memory record with `exists: false` renders the `new` chip beside the note
  name. Existing `core`, `reference`, and `web` chips stay.

In `src/sase/ace/tui/modals/notification_modal_gate.py`, `_plan_decisions_block` pending
rows use `◉` for every decision. Pending toggles use `☑️` when the value is yes and `⬜`
when it is no. Pending choices keep `◉`. Answered rows stay `id: value` with `★` or `●`.

## 3. Branch tint and the fold cache

`classify_callout` in `src/sase/ace/tui/modals/plan_decision_document.py` is public and
only tests call it. For a boolean value it returns `chosen` whenever the toggle is yes,
so a `= no` callout is tinted chosen too. Confirm against `validate_plan` in launch mode
which span field is `no` for `> [!decision] <id> = no` (the cache stores both `branch`
and `key`). A no-branch span is `chosen` only when the value is false and `dimmed` when
it is true. A bare or yes span is `chosen` only when the value is true. Choice spans
stay a key match. Never return a value that means "hide".

Wire that classifier into the document pane. On first display and whenever draft values
change, tint the folded document from the cached callout spans: the chosen branch's
header line is green and bold, and unchosen branches plus a toggle answered no are
dimmed. Every branch line stays visible. Build the renderable from the cached
frontmatter token stream in `src/sase/ace/tui/util/frontmatter_syntax.py`. Do not call
`validate_plan`, parse YAML, or stat on a keypress.

`_scroll_to_focused_decision` currently calls `fold_plan_decisions_content` on every
move. `render_reviewed_content` and the initial display already store `_fold_map`.
Scroll uses that cache and the folded text. Recompute the fold map and callout spans
only when the reviewed content changes.

`render_plan_document` in `src/sase/sdd/_plan_display_rendering.py` accepts `sheet`,
`decided_by`, and `decided_via`, and no caller passes them. Remove those three
parameters and the dead decisions append. Keep `plan_logical_text`'s sheet arguments;
the PLAN lane uses them.

## 4. Edit freeze

`PlanApprovalModalControls._apply_edit_outcome` returns before
`GateActionRunner._apply_edit_outcome` when the outcome message contains
`Decisions are fixed for this review`. The base method is what calls
`controls.set_draft` and `_sync_submission_block`. The early return shows a toast and
leaves the `#gate-draft-banner` hidden, so submit is not blocked.

Apply the base outcome for that freeze as well, so the existing `⚠ Draft not accepted`
banner is shown and `Accept or discard your draft before submitting` blocks every submit
control. The freeze sentence can stay in the notification. Submit stays blocked until
the draft is accepted or discarded. Add a test that drives this outcome, not a source
grep.

## 5. Feedback Carries line

`_notification_modals.py` stores `PlanFeedbackContext.carries`, and
`PlanDecisionDraft.feedback_carry_lines` already returns one `Carries: <id> → <value>`
line per changed decision. `PromptInputBar` in feedback mode never reads them.

Pass those lines into the feedback bar and render them as read-only text above the
editor. The reviewer cannot edit them. No changed decisions means no extra line.

## 6. Settled elsewhere and stale_review

`_settled_text_for_bundle` in
`src/sase/ace/tui/actions/agents/_notification_plan_gate.py` labels every terminal
response `Approved via {surface}`, including reject and feedback.

- Approve, commit, or epic launch: `Approved via {surface} · {summary}`.
- Reject: `Rejected via {surface}`.
- Feedback: `Feedback via {surface}`.

Settled text is computed only inside `load_neutral_plan_modal_data`, so an open modal
never notices a gate that settles on another surface. On the existing notification poll
(`_notification_polling.py`, whose disk work already runs in `asyncio.to_thread`), if a
`PlanApprovalModal` for that request is open and the bundle now has a response, apply
the truthful banner on the UI thread, hide or disable the verdict controls, and block
submit. Do not stat or parse the bundle on the UI thread.

`stale_review` is a `GateError` code from `execute_gate_selection` when
`expected_review_revision` does not match. Both completion handlers in
`submit_neutral_plan_response` toast any failure as `Plan command failed`. The durable
path already branches on `payload["code"]` for `partial_attempt`. Handle `stale_review`
there and on the session-worker path (the worker's exception `code`): reload the bundle,
store the new `review_revision` and definitions, keep the reviewer's current values by
decision id, refresh the rows and the tint, and notify that the plan changed and the
review was reloaded. Do not reset kept values to the new defaults. Do not use the
generic failure toast for this code.

## 7. PLAN-lane render path

`ResponsivePlanSection` in
`src/sase/ace/tui/widgets/prompt_panel/_agent_plan_section.py` calls `_load_plan_sheet`
from `logical_text` and `__rich_console__`. That helper `os.stat`s on every render and,
on a miss, calls `load_stamped_decisions` on the UI thread.

Load the accepted sheet once where the associated plan is already read off the render
path (`build_associated_plan_summary` / `agent_associated_plan.py`), using
`load_stamped_decisions` from `sase.sdd.plan_decision_handoff`. Store the sheet and
`decided_by` / `decided_via` for the section to read. `_load_plan_sheet` becomes a
lookup. A render-path miss returns no sheet. It does not stat, validate, or load.

## 8. Fixtures and Symvision

`tests/ace/tui/test_plan_decision_ace.py` builds `_definitions()` by hand, and its
memory dict does not match the gate payload. Write a temp plan and take definitions from
`build_plan_approval_gate_spec(plan, "visual-session")["payload"]["decisions"]`. An
unverified memory row is the no-artifacts-dir `quote_not_found` case. Keep the existing
behavior assertions, updated for the row fixes above.

Symvision, in this order:

- Prefix `is_unverified_row`, `collapsed_row_text`, and `expanded_row_text` with `_` and
  update the test imports. They are used in `plan_decision_rows.py`. Test imports of
  private names are allowed.
- Keep `classify_callout` public only because the document pane imports it. That import
  is the non-test consumer. Do not add an `--epic-symbol` row for it.

`sase bead epic-symbols sase-1hi.10.4` must print no leftovers before close. The
Justfile currently has no `--epic-symbol` row for these four names. Do not add one. If a
new public symbol has no non-test consumer, delete it, privatize it, or re-key a
Justfile row to a still-open bead (the parent epic or **sase-1hi.10.5**).
`sase bead close` refuses while leftovers remain.

## Tests

Extend `tests/ace/tui/test_plan_decision_ace.py` and the nearest modal test modules.
Cover:

- the three verdict lines with decisions, without decisions, and for an epic;
- short labels plus tooltips, and unchanged generic-gate toggle labels;
- expanded choice value and `●`, unverified copy after turning a row on, collapsed
  unverified warning, and the `new` chip;
- `= no` callout classification and a tint that dims the unchosen branch without
  dropping its lines;
- scroll using the cached fold map (the fold parser is not called per move);
- freeze banner visible and submit blocked until the draft is resolved;
- the feedback bar showing `Carries:` and not editing it;
- reject and feedback settled labels, an open modal picking up a later response, and
  `stale_review` reloading the revision while keeping values;
- the PLAN section render path performing no stat and no plan validation.

## Verification and close

Run `sase tool run check` in this repo. Do not run `just check-full`. Name any KNOWN
failures from the list above in the close note. A failure that reproduces on the clean
base is a `PROPOSED FOLLOW-UP:` note, not a reason to leave the bead open.

Before closing, run `sase bead epic-symbols sase-1hi.10.4` and resolve any rows. Then:

```bash
sase bead close sase-1hi.10.4 --note "<what you verified>"
```

The note lists each item above with its test, the check result, and that epic-symbols
was empty or what was re-keyed. Close no other bead.
