---
tier: tale
title: ACE Verdict rail, first-frame tint, and stale reload
goal:
  ACE plan review keeps every Verdict control inside the rail, tints the chosen branch
  on the first frame without dropping syntax colours, polls only the open modal, and
  reloads a stale review with the reviewer's values kept.
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.10.7.3
bead: sase-1hi.10.7.3
create_time: 2026-10-08 13:30:02
status: wip
---

- **PARENT:**
  [202610/plan_decisions_landing_finish.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_landing_finish.md)
- **BEAD:**
  [sase-1hi.10.7.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.10.7.3.md)

# Plan: ACE Verdict rail, first-frame tint, and stale reload

This tale is the implementation of phase bead **sase-1hi.10.7.3** (epic sase-1hi.10.7,
parent plan `plan:202610/plan_decisions_landing_finish.md` section 3, design from
`plan:202610/plan_decisions.md` sections 1.2 and 6.6). The bead is already `in_progress`
and assigned. Do not set its status by hand. Do not close sase-1hi.10.7, sase-1hi.10,
sase-1hi, or any other ancestor. Phase sase-1hi.10.7.4 owns golden regeneration and is
blocked on this work.

Ancestor beads show no accepted `DECISIONS`. This epic edits no memory notes. Do not
edit anything under `sase/memory/`.

## Read first

Use `sase memory read` (not the files directly):

- `tui.md`, `tui_perf.md` — layout, refresh, and keypress cost
- `symvision.md`, `lint_and_test.md` — before `sase tool run check`

Confirm the line numbers below before editing. They were true in the workspace this plan
was written from and can drift.

## Ground rules

- Work only in the sase repo. No sase-core and no sase-telegram changes.
- Leave PNG goldens alone. Do not edit `tests/ace/tui/visual/snapshots/` or run
  `just fix-tui-screenshots`. A Textual pilot that loads the real
  `src/sase/ace/tui/styles.tcss` is the layout evidence. sase-1hi.10.7.4 regenerates
  goldens after this lands.
- Add no new public symbol that only tests call. Keep new helpers private.
  `tinted_document_text` is already public and stays public because the modal calls it.
- Run `sase bead epic-symbols sase-1hi.10.7.3` before closing. It is empty today. If you
  add a public symbol that only a later phase of this epic will call, add one
  `--epic-symbol` row keyed to that phase's bead (sase-1hi.10.7.4 or sase-1hi.10.7.5)
  per `symvision.md`. Re-key any leftover that this phase was supposed to consume.
  `sase bead close` refuses while leftovers remain.
- Do not create beads. Out-of-scope findings go on this phase with
  `sase bead note sase-1hi.10.7.3 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`.
- These failures are already on master and are not this phase's. Treat a
  `sase tool run check` hit on any of them as KNOWN and do not fix them here:
  - `tests/test_macro_terminology.py::test_macro_string_literals_avoid_xprompt_terms`
    (sase-1hr)
  - `tests/ace/tui/widgets/test_identity_header_raw_prompt.py::test_hinted_raw_prompt_moves_to_identity_and_keeps_its_markers`
    (sase-1hy), plus the raw-prompt and hint failures sase-1i9 and sase-1ia
  - `tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
    (sase-1ic)
  - `test_candidates_fast_path_child_cpu_budget[snippet]` under the parallel lane
    (sase-1g3)
  - `test_post_dispatch_foreign_race_on_external_is_exempt` (sase-1hs)
  - the unused-public Symvision backlog owned by sase-1hp (`write_acceptance_meta`
    belongs to the `gate` phase, not this one)
- Any other check failure must be reproduced on a clean base before it is called
  pre-existing. Record that as a `PROPOSED FOLLOW-UP:` note (cite the task bead if one
  already tracks it) and still close this phase. A failure that reproduces on the clean
  base does not keep the phase open.
- Verify with `sase tool run check`. Do not run `just check-full`.

## 1. Compact Verdict fits the rail

Section 1.2 mock, three lines inside the rail:

```
Verdict
☑️ 🚀 Launch coder    ☑️ 💾 Commit plan
1 ✅ Tale   2 ❌ Reject   3 💬 Feedback
→ coder + commit · …
```

Epic line 2 is `1 ✅ Epic  2 ❌ Reject  3 💬 Feedback` and has no Commit toggle. The
same three-line Verdict is used with and without decisions. The rail is 50 cells when
decisions exist (`.gate-review-actions--decisions`) and the narrow breakpoint stays at
100 columns (`PlanApprovalModal.HORIZONTAL_BREAKPOINTS`).

### What is wrong

`GateBranchControls._compose_plan_compact`
(`src/sase/ace/tui/modals/gate_branch_controls.py`) already builds `#plan-verdict-line1`
(tale AND toggles) and `#plan-verdict-line2` (numbered submits). The modal composes them
inside `#plan-verdict` with class `gate-branch-controls--stacked`
(`plan_approval_modal_view.py`).

Two unscoped rules then force every control to a full row, so the docked Verdict grows
to about ten lines and paints outside the rail (measured: rail x=5..49, Commit plan at
x=49..93, Reject and Feedback at x=49..81):

- `GateBranchControls .gate-option-toggle { width: 100%; }` in
  `src/sase/ace/tui/styles.tcss`
- `GateBranchControls.gate-branch-controls--stacked .gate-singleton-row Button`,
  `.gate-group-expand`, and `.gate-group-submit { width: 100%; }`

There is no `height: 10` rule. The height is the stacked buttons. `#plan-verdict` is
only `dock: bottom; height: auto`.

`plan_toggle_label` appends the `✎ N inputs` badge (`option_input_count_label`) onto the
short label. That badge widens each toggle to about 30 cells, so two toggles cannot
share the 50-cell rail.

### Change

Leave the unscoped `width: 100%` rules in place so generic, non-plan gates stay stacked.
Add `#plan-verdict`-scoped rules that win on specificity (an id beats the class
selectors above):

- `#plan-verdict` toggles, `.gate-group-submit`, and `.gate-singleton` use
  `width: auto`, `min-width: 0`, `height: 1`, horizontal margin, and
  `text-wrap: nowrap`.
- `.plan-verdict-toggles` and `.plan-verdict-branches` stay one row tall.
- The Verdict stays `height: auto` and docked to the bottom of the rail. With two
  control rows plus the title and the summary it must stay short enough that, at 90×40,
  `#plan-decisions-header` is inside the rail's visible content region and does not
  intersect `#plan-verdict`.

For the compact approve/commit buttons only, keep the button label as the checkbox plus
the short text (`🚀 Launch coder`, `💾 Commit plan`). Put any `✎ N input(s)` text on the
tooltip after the full `option.label`. `toggle_label` and non-plan buttons keep the
badge on the button. The full label remains the tooltip, which
`test_compact_verdict_three_lines_with_without_and_epic` already requires.

### Test

Add a pilot whose App sets `CSS_PATH` to the real `src/sase/ace/tui/styles.tcss`
(absolute path; see `tests/test_notification_modal_scroll.py`).
`test_compact_verdict_three_lines_with_without_and_epic` only checks that the widgets
exist and does not load that stylesheet, so it cannot see the overflow.

For tale and epic, with and without decisions, at 120×40 and 90×40:

- Every `GateControlButton` inside `#plan-verdict` has a region contained by the rail
  `VerticalScroll` (class `gate-review-actions`) `content_region`. Use
  `content_region.contains_region`, as
  `tests/ace/tui/command_line/test_chrome_layout.py` does.
- Tale line 1 shows both short toggles; line 2 shows Tale, Reject, and Feedback. Epic
  line 2 shows Epic, Reject, and Feedback, and no Commit toggle.
- With decisions, `#plan-decisions-header` is inside that same content region and its
  region does not intersect `#plan-verdict`.

## 2. Tint on the first frame, syntax colours kept

### What is wrong

`compose` calls `_display_content` → `_ensure_fold_cache`, which sets
`_last_fold_content` before `on_mount`. The guard in
`PlanApprovalControlsMixin.on_mount` (`plan_approval_modal_controls.py`) then skips
`cache_callout_spans`. `_callout_spans` stays `[]`, and the pane renders untinted. The
later `on_mount` update calls `_document_renderable` with those empty spans.

`tinted_document_text` (`src/sase/ace/tui/util/frontmatter_syntax.py`) warms
`_cached_frontmatter_tokens` and then builds a plain `rich.text.Text`, one style per
line, which throws away the token stream.

`_document_renderable` (`plan_approval_modal_view.py`) computes `_chosen` and discards
it. That block is dead. `classify_callout` must stay a non-test consumer through
`tinted_document_text`, which already imports it.

### Change

Cache callout spans once for the displayed content, including the first paint:

- Add a private ensure that calls `cache_callout_spans` only when spans are empty or the
  content differs from the content those spans were built for.
- Call it from `compose` before the first `#plan-approval-content` `Static` is yielded,
  so the first frame is tinted.
- In `on_mount`, run that ensure even when `_last_fold_content` already equals the
  content. The fold-cache guard must not gate the span cache. A second call with the
  same content is a no-op.
- `render_reviewed_content` already recaches spans when reviewed content changes. Keep
  that path, and make it share the same ensure so a content change still refreshes spans
  exactly once per content.

`tinted_document_text` must overlay tint on highlighted text:

- Build the text from the cached lexer. `markdown_document_syntax` /
  `FrontmatterMarkdownLexer.get_tokens_unprocessed` already returns
  `_cached_frontmatter_tokens`. Turn that into `rich.text.Text` with `Syntax.highlight`
  (theme `monokai`, the same theme `markdown_document_syntax` uses). Do not append each
  line with a single replacement style.
- Overlay with `Text.stylize`, which adds spans and leaves the token spans in place.
  Chosen header line: `bold green`. Unchosen branch lines, and a toggle answered no:
  `dim`. Chosen continuation lines keep their token styles and are not dimmed. Delete no
  characters. `classify_callout` remains the chosen-versus-dimmed decision.
- Map raw span lines through `fold_map` the way the function already does. Spans are
  1-based; the fold map is 0-based.

Delete the dead `_chosen` block in `_document_renderable`.

### Keypress cost

`tui_perf` rule 8: do not re-read, re-lex, or re-validate on a keypress.
`_refresh_verdict_summary` re-tints from `_folded_text` and `_callout_spans`. After this
change, a draft edit (the path `_apply_draft_edit` already uses) must not call
`validate_plan`, `cache_callout_spans`, or `_lex_frontmatter_markdown`. A cache hit in
`_cached_frontmatter_tokens` is the allowed work. Prove that with a mounted-modal test
that counts those calls across one value change. Do not add a `pytest -m slow` bench.

### Test

After mount, with the decision fixture in `tests/ace/tui/test_plan_decision_ace.py` (it
already has `> [!decision]` callouts):

- `_callout_spans` is non-empty.
- The document renderable's plain text still contains the unchosen branch.
- The chosen header carries a bold-green span.
- An unchosen line carries a dim span.
- At least one syntax token style from the lexer is still present on the same `Text` (a
  span that is not only the tint overlay).

## 3. Settled-elsewhere polling reads only the open modal

### What is wrong

`_poll_agent_completions_once` in
`src/sase/ace/tui/actions/agents/_notification_polling.py` always `await`s
`asyncio.to_thread(prepare_settled_texts_for_notifications, every notification)`. That
function (`_notification_plan_gate.py`) `load_and_verify_bundle`s every `PlanApproval`
and `EpicApproval`, on every auto-refresh, even when no plan modal is open.

The poll callback is already off the UI thread via that `to_thread`. `tui_perf` rules 1
and 2 still apply: do not add a new awaited body on the pump, and do not move the stat
onto the UI thread. Narrow the work inside the existing thread handoff. When no plan
modal is open, do not schedule the settled read at all.

### Change

On the UI thread, before the thread handoff, find the open `PlanApprovalModal` the same
way `apply_settled_text_to_open_modal` does (current screen, else the screen stack). If
there is none, skip `prepare_settled_texts_for_notifications`.

If one is open, pass only the notification whose `action_data["request_id"]` or `id`
equals the modal's `_request_id`.

For that one notification:

- `resolve_notification_bundle` is a path resolve plus `request.is_file()`. Use it to
  find `bundle.response` (`response.json`).
- Stat that path off the UI thread: existence and `st_mtime_ns`. A missing file is a
  stable signature and means "not settled"; do not hash-verify it.
- Keep the last signature and the last settled text on the app, keyed by request id.
  Call `load_and_verify_bundle` only when the signature changes. An unchanged tick
  reuses the cached text and does not read the bundle.
- Drop that cache entry when the modal is no longer open so the next open loads again.
- `apply_settled_text_to_open_modal` stays on the UI thread and stays as it is.

### Test

Drive `_poll_agent_completions_once` the way
`tests/test_notification_completion_arrival.py` does, with `load_and_verify_bundle`
counted:

- No plan modal open, and at least one PlanApproval notification present: the count
  stays 0.
- Modal open, `response.json` missing: the count stays 0.
- Modal open, `response.json` appears: exactly one load. A second poll with the same
  mtime does not load again. A newer mtime loads once more.

## 4. `stale_review` reloads for real

`_handle_stale_review` runs on the UI-thread `on_complete` of the plan response
(`_notification_plan_gate.py`). It already calls `load_neutral_plan_modal_data`. Keep
that call where it is. Do not add a new disk read on a keypress path.

Today, when the modal is gone, the handler only stores `decision_*` values in
`esc_drafts` and toasts that the review was reloaded. When the modal is open and the
request id matches, it swaps the revision and the draft, then calls
`_refresh_verdict_summary`. It does not rebuild rows for an added or removed decision,
and it does not recompute the fold map or the tint from the new plan text.

`PlanDecisionRows.update_rows` only relabels buttons that already exist. A changed id
list needs a rebuild: update `_rows`, `_definitions`, and `_by_id`, and when mounted
replace the children (remove and mount the compose set again). `PlanDecisionDraft`
already drops unknown ids and fills a missing id from `effective_default`, which is the
`★` value.

### Closed modal

- Values come from the stale submit's `option_inputs` keys that start with `decision_`.
  Keep an id only when it is still in the reloaded definitions. Omit ids that vanished.
  Omit brand-new ids so the new modal's `PlanDecisionDraft` applies `★`.
- Store that map in `esc_drafts` under the request id.
- `app.push_screen` a new `PlanApprovalModal` built from the reloaded
  `PlanGateModalLoad`: `plan_file`, `plan_content`, `default_choice`, `gate`, `actions`,
  `decision_definitions`, `review_revision`, `request_id`, `settled_text`. The
  constructor already restores `esc_drafts` for that request id. Match the fields
  `_notification_modals.py` passes into `PlanApprovalModal` for the ones the load
  carries.
- Keep the existing warning toast.

### Open modal

- Do not push a second modal.
- Same value merge, with the live draft as the base and the submit's `decision_*` values
  overlaid, then filtered to the new ids.
- Assign the new definitions, revision, and `PlanDecisionDraft`.
- When the load carries `plan_content`, set `_plan_content`, clear the fold-content
  guard, recompute the fold map, and recache callout spans for that new content (this is
  a reload, not a keypress).
- Rebuild the decision rows from the new sheet and the new definitions.
- Update `#plan-approval-content` through `_document_renderable`, and refresh the
  verdict summary and footer.
- Store the merged values in `esc_drafts` too, so a later Esc reopen matches the screen.

### Tests

Extend `test_stale_review_reloads_revision_keeping_values`. Its reloaded stand-in only
has `review_revision` and `decision_definitions`, and its app has no `push_screen`. Grow
the stand-in and assert the behaviour:

- Closed: `push_screen` receives a `PlanApprovalModal` whose `_review_revision` is the
  reloaded revision, whose draft keeps `grouping=mode`, whose vanished id is absent, and
  whose new id equals that definition's `effective_default`.
- Open: mount a real modal in a pilot, call the handler, assert `push_screen` was not
  used, the same screen shows the new revision, the rows match the new ids, and the
  document widget was updated from the reloaded content (fold map / spans recomputed;
  the new body text is what the pane shows).

## 5. The four red tests this epic's first TUI pass caused

Short labels are already what the buttons render. The tests still expect the old full
label on the button.

- `tests/ace/tui/test_notification_plan_gate.py::test_plan_modal_bundle_loading_stays_off_the_message_pump`
- `tests/test_plan_approval_modal_title.py::test_group_submit_uses_current_branch_selection`

For `#gate-option-0-0` and `#gate-option-0-1`: the button label contains the short text
(`Launch coder`, `Commit plan`) and the rocket icon, and it does not contain
`Launch coder agent` or `Commit plan file to the plans sidecar`. Those full strings are
on the tooltip. Keep the existing numbered-branch assertions (`1 `, Tale, `2 `, `3 `).

The other two tests count `Path.read_text` and expect a single read of the plan across
two `resolve_agent_associated_plan` calls, then a second read only after the plan mtime
changes:

- `tests/ace/tui/models/test_agent_associated_plan_cache.py::test_frontmatter_cache_reuses_parse_until_mtime_changes`
- `::test_title_is_normalized_cached_and_invalidated_with_file_signature`

`_cache_associated_plan_sheet`
(`src/sase/ace/tui/models/_agent_associated_plan_summary.py`) calls
`load_stamped_decisions(path)` with no tier. That calls `_frontmatter_tier`, which
`read_text`s the plan on every enrichment. `validate_plan_file` uses `read_bytes` and is
invisible to these tests; the `read_text` in `_frontmatter_tier` is the extra read. A
signature check that still calls `load_stamped_decisions` once on the first enrichment
still fails `reads == [plan.resolve()]`.

Fix:

- Key a side table by the plan path. The value is the plan file signature
  `(st_mtime_ns, st_size)` (or an absent sentinel) and the same signature for
  `sibling_path_for_plan` (`sase.sdd.plan_decision_freeze`). On a hit, return without
  calling `load_stamped_decisions`.
- On a miss, call `load_stamped_decisions(path, tier=<authored tier>)` so
  `_frontmatter_tier` does not `read_text`. `build_associated_plan_summary` already has
  `metadata.authored_tier`. When that tier is missing, pass `"tale"` rather than letting
  the helper re-read the file.
- Leave `_ASSOCIATED_PLAN_SHEET_CACHE`'s public triple
  `(sheet, decided_by, decided_via)` and `associated_plan_sheet_for` unchanged. The
  render path must not stat. `test_plan_section_render_path_no_stat_no_validate`
  monkeypatches `os.stat` to throw, and `_load_plan_sheet` must stay a dict lookup.
- A sibling mtime change with an unchanged plan still reloads the sheet. Add a focused
  test for that. The two existing tests have no sibling; their `read_text` counts stay
  exactly what they assert.

## 6. Edit freeze, the missing half

`test_freeze_banner_visible_and_submit_blocked` shows the banner and the block.
`_apply_edit_outcome` already calls `set_draft(None)` and `_sync_submission_block` when
`GateEditOutcome.draft` is false.

Extend that test: after the blocked state, apply an accepted outcome with `draft=False`.
Assert `#gate-draft-banner` has `hidden`, `GateBranchControls._submission_block` is
`None`, and `_resolve_branch` does not notify "Accept or discard". If that fails, fix
the clear path. Do not change the blocked behaviour the first half asserts.

## 7. Cleanups

- Delete the unused `_PLAN_SHEET_CACHE` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_plan_section.py`. Nothing reads it.
  `_load_plan_sheet` uses `associated_plan_sheet_for`.
- Remove `_collapsed_row_text`, `_expanded_row_text`, and `_is_unverified_row` from
  `plan_decision_rows.py`'s `__all__`. They stay private. Tests import them by name;
  that does not require `__all__`. Do not rename them.
- `tests/ace/tui/test_plan_decision_ace.py` `_definitions` writes a
  `NamedTemporaryFile(delete=False)` and never deletes it. The returned value is a list
  of dicts; the file only has to exist during `build_plan_approval_gate_spec`. Give
  `_definitions` the test's `tmp_path`, write the plan under that directory, and thread
  `tmp_path` through its callers in that module. Leave no `delete=False` file behind.

## Verification and close

1. `sase tool run check` in the sase repo. Name every KNOWN failure in the closing note.
   Do not run `just check-full`.
2. `sase bead epic-symbols sase-1hi.10.7.3` is empty, or every remaining row is re-keyed
   to a still-open later phase.
3. Close only this phase:

```bash
sase bead close sase-1hi.10.7.3 --note "<items fixed and the test that covers each; check result with KNOWN names; epic-symbols empty or re-keyed>"
```

The closing note lists each item from sections 1–7 with the test that covers it.
