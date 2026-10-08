---
tier: epic
title:
  "Plan Decisions landing finish: fix the broken Verdict, tint, receipts, and the
  missing route coverage"
goal:
  Finish the work the sase-1hi.10 land audit found incomplete or broken. ACE shows every
  Verdict control and tints the chosen branch from the first frame. The epic-caused red
  tests on master pass. sase bead work reuses accepted answers without re-resolving
  them. Stale reviews leave a durable record that every surface can recover from. The
  %auto receipt reaches the ACE inbox. Shell completion scopes -D ids to the named
  proposal. Telegram receipts, stale recovery, and the sheet budget match the parent
  plan. The goldens show all of this.
phases:
  - id: gate
    title:
      Bead-work answer reuse, durable stale_review records, one direct resolver, receipt
      inbox, guard strands, and the owed route tests
    depends_on: []
    size: large
    description:
      "gate: stop sase bead work from re-resolving accepted plans, record stale_review
      and other pre-acceptance rejections durably, surface swallowed re-stamp failures,
      keep one direct resolver, strip private gate keys, cover new strands in the memory
      guard, show the %auto receipt in the ACE inbox, clear write_acceptance_meta, and
      add the stamping, stale_review, refusal, and writer-side tests the first pass
      skipped."
  - id: cli
    title:
      Completion snapshot, shell-scoped -D completions, consistent memory chips, and
      handler-level CLI tests
    depends_on:
      - gate
    size: medium
    description:
      "cli: regenerate the completion spec snapshot, make the zsh/bash/fish helpers pass
      the named proposal to plan-decision completions, document -S, make the card and
      pending sheet show one consistent provenance chip, and replace helper-level CLI
      tests with handler and rendered-output tests."
  - id: tui
    title:
      ACE Verdict that fits the rail, first-frame tint with syntax kept, cheap settle
      polling, real stale reload, and the epic-caused red tests
    depends_on: []
    size: large
    description:
      "tui: scope Verdict CSS so all five tale controls and every epic control sit
      inside the rail, keep Decisions visible at 90 columns, tint on first display while
      keeping syntax colours, poll only the open modal's bundle, reopen or rebuild on
      stale_review with values kept, fix the four tests the first pass broke, and clean
      up dead caches."
  - id: goldens
    title:
      Regenerate and inspect the Plan Decisions and plan_gate goldens after the Verdict
      and tint fixes
    depends_on:
      - gate
      - tui
    size: medium
    description:
      "goldens: run the full just fix-tui-screenshots under /sase_monitor after tui and
      gate land, inspect every created or updated PNG, and confirm every Verdict
      control, the 90-column Decisions panel, chosen-branch tint, and unchanged generic
      gate goldens."
  - id: telegram
    title:
      Telegram receipts without doubled words, stale recovery that keeps the card and
      draft, budget order per the parent plan, and flow-level tests
    depends_on:
      - gate
    size: large
    description:
      'telegram: fix the doubled "via" and "auto auto" receipt headers, recover
      stale_review and pre-response errors from the durable gate record while keeping
      the card, the refresh button, and the draft, edit the original card after feedback
      replies, follow the parent plan''s three-step sheet budget, pin the keyboard, and
      add flow-level settle and submit tests.'
parent_bead: sase-1hi.10
proposed_by: bbugyi200.apollo.sase-1hi.10.land
create_time: 2026-10-08 13:17:00
status: wip
---

- **PROMPT:**
  [prompts/202610/plan_decisions_landing_finish.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/plan_decisions_landing_finish.md)
- **PARENT:**
  [202610/plan_decisions_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_landing_repairs.md)

# Plan Decisions landing finish

## 0. Scope, ownership, and ground rules

This child epic finishes **sase-1hi.10** ("Plan Decisions landing repairs"), which
finishes **sase-1hi** ("Plan Decisions").

- The authoritative design is **plan:202610/plan_decisions.md** (the parent plan). Read
  it with `sase artifact read`, especially Sections 1.2–1.6, 6.4.5, 6.6, and 6.7.
- The repair plan this epic completes is
  **plan:202610/plan_decisions_landing_repairs.md**.

The sase-1hi.10 land agent audited every repair phase against both plans on master
`f92bde8abe`. Every item below is a confirmed gap from that audit, reproduced where
noted. Line numbers are from `f92bde8abe` and may drift. Confirm each one before you
change anything.

**Two module splits landed during sase-1hi.10.** Use the new locations:

- `f92bde8abe` split `notification_gates/cli_answer.py` into `cli_answer_submit.py`,
  `cli_answer_handle.py`, `cli_answer_resume.py`, and `cli_answer_inputs.py` behind the
  `cli_answer` facade. Detached submission now lives in
  `cli_answer_submit.submit_detached_answer`.
- `9aa7ff2485` split `notification_gates/adapters.py` into `adapter.py`,
  `adapter_plan.py` (plan side effects and the `%auto` receipt hook), and
  `adapter_registry.py` behind the `adapters` facade.

**Not in this epic.** These belong to the resumed landings and no phase here does them:

- closing sase-1hi.10 and sase-1hi, and marking their plans done;
- the parent plan's Section 7 end-to-end smoke and usability report;
- deploying changed skills.

The audit triaged its non-epic findings into existing beads: sase-1hp (Symvision
unused-public backlog) and sase-1ic (TUI import budget). Older follow-ups are sase-1hy,
sase-1hz, sase-1i0, sase-1i1, sase-1i2, and sase-1i3. Do not re-file any of them.

**Repositories.**

- `gate`, `cli`, `tui`, and `goldens` work in the sase repo.
- `telegram` works in the linked sase-telegram checkout. Open it with
  `sase repo open sase-telegram` and read its `AGENTS.md`.
- No phase is expected to change sase-core. If one must, follow the Rust boundary
  (`sase repo open sase-core`, then move `sase-core-revision.txt` past the new commit).

**Required reading.** Use `sase memory read`. This epic edits no memory notes.

- Every sase phase: the `lint_and_test` and `symvision` notes.
- `tui` and `goldens`: also `tui`.
- `cli`: also `cli_rules`.

**Verification.** Run `sase tool run check` in every repo you change. Never run
`just check-full`.

These failures are on master and are not caused by this epic. Record them as KNOWN and
do not fix them here:

- `tests/test_macro_terminology.py::test_macro_string_literals_avoid_xprompt_terms`
  (sase-1hr)
- `tests/ace/tui/widgets/test_identity_header_raw_prompt.py::test_hinted_raw_prompt_moves_to_identity_and_keeps_its_markers`
  (sase-1hy), plus the related raw-prompt and hint failures sase-1i9 and sase-1ia
- `tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
  (sase-1ic)
- `test_candidates_fast_path_child_cpu_budget[snippet]` under the parallel lane
  (sase-1g3)
- `test_post_dispatch_foreign_race_on_external_is_exempt` (sase-1hs)
- the unused-public Symvision backlog owned by sase-1hp. Its one epic-owned entry,
  `write_acceptance_meta`, is cleared by `gate`.

If any other failure appears, prove it reproduces on a clean base before calling it
pre-existing. Record such failures as a `PROPOSED FOLLOW-UP:` note.

**Symvision.** Each phase adds **no new** Symvision unused-public entries. A seam a
later phase of this epic will consume gets an `--epic-symbol` row keyed to the consuming
phase's bead, per the `symvision` note.

**Phase workers never create beads.** Record anything out of scope as a
`PROPOSED FOLLOW-UP:` note on your phase bead.

## 1. `gate` — answer reuse, durable rejections, one resolver, receipt inbox, guard, tests

1. **`sase bead work` must reuse accepted answers, never re-resolve them.**
   - `_reuse_stamped_bead_work_answers` (`src/sase/bead/cli_work_from_plan.py` ~379-430)
     feeds stamped values back into a fresh `build_definitions` resolution with live
     host facts and the current caller.
   - Reproduced: a reviewer-accepted memory decision `true` re-resolved with caller
     `agent` returns `memory_decision_requires_human`. So `sase bead work <file>` from
     an agent shell fails with "stamped plan answers failed validation".
   - Its conflict check can never fail, because the stamped values are the overrides.
   - A partly stamped plan (one decision answered, another not) is accepted silently,
     against the function's own docstring. Raise a clear error instead.
   - The fresh-stamp branch (~351-356) ignores `resolved["errors"]`. Surface them.
   - Read accepted answers through `load_stamped_decisions`
     (`sdd/plan_decision_handoff.py`).
   - The sibling backfill (~416-430, added by `6828ed3836`) writes definitions resolved
     from the reader's environment into `foo.plan-decisions.json`. Write the sibling
     only from frozen gate data. Otherwise leave it absent, so the neutral accepted
     fallback applies.
   - Tests:
     - an agent caller runs bead work on a reviewer-accepted memory-yes plan and it
       succeeds without re-resolution;
     - a partial stamp errors;
     - resolver errors surface;
     - no sibling is written from the reader's environment.
2. **Record every pre-acceptance rejection durably.**
   - `execute_gate_selection` (`notification_gates/executor.py` ~150-160) raises
     `stale_review` before any `recorded_rejection` scope. So a detached
     `sase gate answer`, the Telegram route, writes no `errors/*.json`, and no surface
     can see why the answer vanished.
   - Record `stale_review` through `command_runner.record_execution_error`, with code
     `stale_review` and the current revision in the message or payload. Do the same for
     the other `GateError`s raised before `accept_gate_decision`: `unknown_source`,
     `decision-resolve-failed`, and `memory_decision_requires_human`.
   - Keep the exception propagating unchanged.
   - Tests: detached `sase gate answer` with a stale revision leaves exactly one
     `errors/*.json` with code `stale_review` and no `response.json`.
   - `telegram` consumes this record. Keep the code string and file shape stable.
3. **Surface swallowed re-stamp failures.**
   - `recover_plan_stamp_from_response` failures are swallowed with
     `except Exception: pass` at `executor.py` ~232-237 and `adapter_plan.py` ~33-38.
     Only `cli_answer_resume.py` ~74-76 raises.
   - Log each failure and record it through `record_execution_error` (stage `restamp`),
     or raise where the route can report it, as the resume route does.
   - Test `recover_plan_stamp_from_response` directly: success, conflicting stamp,
     missing answers, and the recorded failure.
4. **One direct resolver.**
   - Two entry points remain. `main/plan_decide.resolve_direct_decisions` serves
     `plan approve <file>` and `-D`.
     `sdd/plan_decisions.resolve_plan_decisions_for_direct_approval` (~726) serves bead
     work, and its docstring wrongly claims the other two routes.
   - Keep one function, used by all three routes.
   - Remove the second `build_definitions` call in
     `main/plan_direct_approval_resolve.py` (~177-190).
5. **Private gate keys.** `_gate_source`/`_gate_caller` stay in `response.json` option
   results for plans without decisions, because they are stripped only when decisions
   exist. Strip them always, and update `tests/test_plan_gates_action_api.py` (~57-60),
   which pins the leak.
6. **Missing-note grants use structured errors.** Whether a missing note may be granted
   is decided by matching words in error text (`non_grant_markers` in
   `sdd/plan_decisions.py`). Classify by a structured reason instead, and keep the
   existing error codes for bad syntax, unknown scopes, and overlap.
7. **Memory guard covers new strands.**
   - `commit_memory_guard._coverage_from_sheet_rows`
     (`finalizers/commit_memory_guard.py` ~128-179) reads only `kind: note` and
     `kind: web`. So a granted new strand (`kind: "strand"`, for example
     `glossary:plan-decision`) is reported as uncovered when created.
   - Cover strands.
   - Drop the redundant second `_collect_committed_paths(new_markers)` call (~574),
     which reruns `git diff-tree` for every marker for no effect.
   - Tests should drive `memory_guard_for_new_markers` itself:
     - a frozen sibling loaded with cwd `/tmp` covers `sase/memory/tui.md`;
     - a granted new strand is covered;
     - a nested hand-written `AGENTS.md` stays "other".
8. **The `%auto` receipt appears in the ACE inbox** (parent plan 6.4.5).
   - The receipt is posted `silent=True` (`plan_decision_handoff.py` ~571).
     `_notification_provider_direct.py` ~96 (`src/sase/ace/tui/actions/agents/`) drops
     silent rows, so the receipt never shows.
   - Add one predicate next to `RECEIPT_TAG` in `sdd/plan_decision_handoff.py`. It is
     true for a notification that is silent, not muted, tagged `plan_decisions_receipt`,
     and has no action, matching sase-telegram's `outbound._is_quiet_decision_receipt`.
   - Have the direct page keep those rows. Counts, toasts, and the bell must still skip
     them (`lifecycle.py` ~128,149), so there is still no unread bump.
   - Do not switch to `muted=True`: Telegram skips muted rows.
   - Fix `docs/notifications.md` (~985-986): Telegram does deliver the receipt quietly.
     Fix the `post_auto_approval_receipt` docstring, which calls it "a plain inbox row".
   - Test: the receipt appears on the modal's direct page and leaves the unread count
     unchanged.
9. **Symvision.** `write_acceptance_meta` (`notification_gates/decision.py` ~110), added
   by `c929bb176b`, is unused-public. Privatize it or wire it.
10. **Tests the first gate pass owed** (repair plan Section 1 item 9):
    - `stale_review` through `execute_gate_selection` and through `sase gate answer`,
      attached and detached;
    - stamping on every route in authored order: tale approve+commit, approve-only,
      commit-only, epic, direct file, hand-run `sase bead work`, and retry re-stamp;
    - the agent memory refusal end to end through `sase gate answer`;
    - a grant for a new strand of an existing web (`glossary:plan-decision` →
      `exists: false` at `sase/memory/glossary/plan-decision.md`);
    - the `decision-host-check-failed` diagnostic for `sase plan validate` and
      `sase plan propose`;
    - bead-work caller classification: a human shell gives `reviewer`/`cli`, an agent
      gives `agent`/`cli`;
    - handoff writer side: the bundle-to-sibling write, never overwriting an existing
      sibling, and the archive copy plus commit.

## 2. `cli` — completion, chips, handler-level tests

1. **Completion snapshot.** `b470a1b461` added `-S/--selector` to
   `sase completion candidates` but never regenerated
   `tests/completion/snapshots/cli_spec.json`. `tests/completion/test_snapshot.py` (two
   tests) is red on master. Run `just sync-completion-spec` and commit the snapshot.
2. **Shell completion passes the proposal.**
   - The provider and fast path accept `-S`. But the generated helpers call
     `__sase_run completion candidates $kind` with the kind only:
     - `emit_zsh_preamble.py` ~71
     - `emit_bash.py` ~69
     - `emit_fish.py` ~38
   - So interactive `sase plan approve foo -D <TAB>` still offers ids merged from every
     proposal.
   - When completing plan-decision ids, pass the proposal named on the command line as
     `-S <proposal>`. Key the zsh and bash caches by kind plus selector, and never store
     a scoped result under the merged key.
   - In `_scope_rows_to_selector` (`completion/candidates/catalog_plans.py`), an exact
     match must win over prefix ambiguity.
   - Document `-S/--selector` in `docs/completion.md` (~166-167).
   - Test the emitted helper text for each shell and the scoping rules.
3. **One consistent provenance chip** (parent plan Section 1.6).
   - `_build_host_facts` (`sdd/plan_decisions.py` ~499-505) sets provenance `not_asked`
     whenever `default` is false, even when a `requested:` quote exists. The card and
     the pending sheet then print `not asked · you asked: '…'`. Reproduce with the
     `PENDING_TALE` fixture shape in `tests/test_plan_decide_cli.py`.
   - Make each row show exactly one provenance chip on the card, the pending sheet, and
     `sase plan validate`.
   - A `quote_not_found` row that a human sets with `-D id=yes` must not show `yes ●`
     next to `⚠ quote not found · off`. Show the human override consistently.
   - Pad the value column so the source column lines up, as in the Section 1.4 mock.
4. **Handler-level and rendered-output tests.** The first pass tested helpers. Add:
   - `sase plan show` on a plan with decisions: the rendered `DECISIONS` header in text,
     the compact `◉N 🧠M` string, and the handler's `-f json` output. Today
     `tests/main/test_plan_show_render.py` and `test_plan_show_handler.py` have none.
   - live-gate `-D` through `plan_approve_handler` (~331-380): assert
     `execute_plan_approval_response` receives the `decision_*` inputs and
     `expected_review_revision`;
   - direct-file `-D` through `execute_direct_approval`, asserting the stamped file;
   - `test_validate_json_stdout_is_single_document`: clear `SASE_AGENT` and
     `SASE_ARTIFACTS_DIR` (it clears a nonexistent `SASE_AGENT_CONTEXT`), so it really
     covers the outside-agent path.

## 3. `tui` — ACE

Read the `tui` note's performance rules first. Do not update PNG goldens here; `goldens`
owns them. A targeted, non-updating visual run or `sase screenshot` to check layout is
fine.

1. **The compact Verdict must fit the rail** (parent plan 6.6.1 and the Section 1.2
   mock).
   - Under the real ACE stylesheet, two rules push controls outside the rail:
     - `GateBranchControls .gate-option-toggle { width: 100% }` (`styles.tcss` ~1914);
     - the stacked `.gate-singleton-row Button`/`.gate-group-submit { width: 100% }`
       (~1936-1941).
   - The Verdict reuses `gate-branch-controls--stacked` (`plan_approval_modal_view.py`
     ~313). Measured: the rail spans x=5..49, "💾 Commit plan" sits at x=49..93, and "2
     ❌ Reject"/"3 💬 Feedback" sit at 49..81. So a tale review shows only Launch coder
     and Tale, and epic reviews cut off Feedback.
   - The `✎ N inputs` badge widens each toggle to about 30 cells.
   - At 90 columns the docked Verdict (height 10) covers the Decisions header and rows.
   - Add `.plan-verdict`-scoped CSS and layout:
     - Line 1: `☑️ 🚀 Launch coder  ☑️ 💾 Commit plan`, auto widths, on one row;
     - Line 2: `1 ✅ Tale  2 ❌ Reject  3 💬 Feedback` (and the epic row) on one row;
     - Line 3: the summary.
   - Keep the "Verdict" title from the mock. The badge must not push a toggle off its
     row. At 90 columns the Decisions panel must stay visible and scrollable.
   - Generic, non-plan gates must not change.
   - Add a pilot test that loads the real ACE stylesheet. For tale and epic, with and
     without decisions, at 120x40 and 90x40, assert that:
     - every Verdict control's region lies inside the rail's content region;
     - the Decisions header is visible at 90x40.
   - `test_compact_verdict_three_lines_with_without_and_epic` only checks that the
     widgets exist.
2. **Tint on first display, with syntax colours kept** (parent plan 6.6.4).
   - `compose()` sets `_last_fold_content`, so the guard in `on_mount`
     (`plan_approval_modal_controls.py` ~138) skips `cache_callout_spans`.
     `_callout_spans` stays `[]` and the pane renders untinted. No decisions golden
     shows a tint.
   - Fix it so spans are cached once at mount.
   - `tinted_document_text` (`util/frontmatter_syntax.py` ~92-160) throws away the
     cached token stream and returns plain `Text`. Apply the tint as a style overlay on
     the highlighted text: chosen branch green with a bold header, unchosen branches and
     toggles answered no dimmed, nothing hidden.
   - Delete the dead `_chosen` computation (`plan_approval_modal_view.py` ~179-189).
   - Test after mount: spans are non-empty, the chosen branch carries the tint style,
     unchosen lines are dimmed, and syntax token styles survive.
   - Measure keypress cost against the `tui` note.
3. **Settled-elsewhere polling is cheap.**
   - `_notification_polling.py` (~249-265) loads and hash-verifies every
     PlanApproval/EpicApproval bundle on every auto-refresh, even when no plan modal is
     open.
   - Check only the open plan modal's bundle, with a cheap `response.json`
     existence/mtime probe off the UI thread. Load the bundle only when that changes.
   - Test that no bundle is read while no plan modal is open.
4. **`stale_review` really reloads** (parent plan 6.6.6).
   - When the modal has already closed, `_handle_stale_review`
     (`_notification_plan_gate.py` ~595-675) only saves values to `esc_drafts` and
     toasts "reloaded".
   - Reopen the modal on the reloaded bundle and its new revision, with the reviewer's
     values restored by decision id (ids that vanished are dropped, new ids take `★`).
   - When the modal is open, rebuild the rows, the document content, the fold map, and
     the tint.
   - Test both paths.
5. **Fix the red tests the first tui pass caused**:
   - `tests/ace/tui/test_notification_plan_gate.py::test_plan_modal_bundle_loading_stays_off_the_message_pump`
     and
     `tests/test_plan_approval_modal_title.py::test_group_submit_uses_current_branch_selection`.
     Both assert the old full label "Launch coder agent". Assert the short label plus
     its tooltip.
   - `tests/ace/tui/models/test_agent_associated_plan_cache.py::test_frontmatter_cache_reuses_parse_until_mtime_changes`
     and `::test_title_is_normalized_cached_and_invalidated_with_file_signature`.
     `_cache_associated_plan_sheet` (`models/_agent_associated_plan_summary.py`) runs
     `load_stamped_decisions` on every enrichment with no file-signature check. Key it
     by the plan file's signature and its `.plan-decisions.json` sibling's, so unchanged
     plans are never re-read.
6. **Edit freeze.** Add the missing half of the test: submit unblocks once the draft is
   resolved.
7. **Cleanups.**
   - Delete the dead `_PLAN_SHEET_CACHE` (`widgets/prompt_panel/_agent_plan_section.py`
     ~45).
   - Remove the private names from `plan_decision_rows.py`'s `__all__`.
   - Replace the leaking `NamedTemporaryFile(delete=False)` fixtures in
     `tests/ace/tui/test_plan_decision_ace.py` with `tmp_path`.

## 4. `goldens` — regenerate and inspect

1. After `tui` and `gate` land, run the **full** `just fix-tui-screenshots` through
   `/sase_monitor`, using the `TESTING`/`TESTED` pair and a generous timeout. Because
   the Verdict CSS changed, a targeted run is not enough evidence that the generic gate
   goldens are unchanged. If the run reports `partial`, read the WARNING block and the
   manifest's `skipped` list.
2. Expect updates to the eight Plan Decisions goldens and the four `plan_gate_*` goldens
   refreshed by sase-1hi.10.5:
   - `plan_gate_tale_decisions{,_memory,_unverified}_120x40`
   - `plan_gate_tale_decisions_stacked_90x40`
   - `plan_gate_epic_decisions_120x40`
   - `notification_gate_plan_decisions_{pending,answered}_120x40`
   - `plan_toast_tale_decisions_120x40`
   - `plan_gate_tale_five_controls_120x40`, `plan_gate_frontmatter_120x40`,
     `plan_gate_epic_action_120x40`, `plan_gate_tale_stacked_90x40`
3. Open and inspect every created or updated PNG. Confirm each of these:
   - **Tale:** both toggles (Launch coder, Commit plan) and all of Tale, Reject, and
     Feedback are visible inside the rail.
   - **Epic:** the full verdict row, Feedback included, is visible.
   - **Stacked 90x40:** the Decisions panel is visible.
   - **Decisions goldens:** the chosen branch is tinted and unchosen branches are
     dimmed, with syntax colours intact.
   - The unverified `⚠` warning and the memory chips are intact.
4. A changed golden for a generic, non-plan gate is a regression. Fix the CSS scoping (a
   small fix is in scope here) and rerun; do not accept it.
5. Record what each image shows in your phase note.

## 5. `telegram` — sase-telegram

Work only in the linked checkout. Every decision import stays on
`sase.sdd.plan_decisions` with its feature-detect fallback.

1. **Receipt headers.**
   - `decision_receipt.py` ~278 and ~287 build `"{decider} via {surface}"`, but the
     surface already reads "via …".
   - Reproduced:
     - "❌ Rejected · you via via Telegram" (likewise via ACE and via CLI);
     - "💬 Feedback sent · human via via mobile";
     - "✅ Tale approved · auto auto".
   - Fix all three to the Section 1.3 wording.
   - The test at `tests/test_plan_decisions.py` ~564 passes no decider or surface. Cover
     each surface and `%auto`.
2. **Stale and pre-response errors recover from the durable record** (`gate` item 2).
   - Today the stale restore branches (`gate_response.py` ~149-155,
     `gate_completions.py` ~187-191) are reachable only from a synchronous test fake.
   - In production the keyboard is removed at submit (`gate_response.py` ~176-189). Then
     `gate_completions.py` ~197-217 reports "finished without a recorded answer",
     deletes the progress file and the draft values, and leaves a card with no buttons.
   - Read the bundle's `errors/*.json`. On `stale_review`, answer "This plan changed
     since this card was shown.", restore a keyboard with **↻ Refresh review**, and keep
     the progress (parent plan 6.7.4).
   - Report other pre-response errors (for example a schema rejection) with their
     recorded message, not "finished without a recorded answer".
   - Bound the wait when the proc row is missing, so the record never spins forever.
3. **Feedback replies edit the original card.**
   - `text_messages.py` ~176-183 removes the pending action and progress as soon as the
     feedback text arrives. Settle then cannot find the review message id, so it sends a
     new message, and ↻ is gone.
   - Keep what settle needs until settle runs, so the receipt edits the card even when
     `source_message_id` is null.
4. **Launch failure versus acceptance.**
   - `_is_launch_failure` matches `successor_launch_failed`, which no sase code emits,
     plus text heuristics.
   - The receipt is finalized as soon as `response.json` exists, which happens before
     side effects run.
   - Read sase's real post-acceptance launch-failure signal (the journal/side-effects
     outcome in the gate bundle). Show "Approved with these choices · coder could not
     start · retry" only for a real failure, and update the receipt if the failure lands
     after the first edit.
5. **Sheet budget exactly as parent plan 6.7.1.**
   - Today `_render_sheet_expandable` (`decision_sheet.py` ~132-165) wraps the whole
     sheet, asks included, in the blockquote, against its own comment at ~160. It also
     drops every memory line (note names, provenance chips, quote), and adds an extra
     "drop why/quote" step.
   - Implement exactly the three steps: drop the labels of non-default choices, then the
     remaining labels, then wrap only the choice lines in an expandable blockquote.
   - Each `ask`, its `★` line, and the memory note and chip lines always survive.
   - Tests:
     - the degrade order, step by step;
     - the blockquote is present, with the asks outside it;
     - memory lines survive;
     - the 1,800-character budget;
     - no split MarkdownV2 escape.
6. **Pins and flow-level tests.**
   - Pin the decision keyboard layout: exact rows, labels, and callback tokens, per the
     Section 1.3 mock.
   - Assert `style: success` on the primary button.
   - Cover the accepted-plan PDF path and the `new` memory chip.
   - Flow-level submit tests from a Telegram callback: Reject, approve-only, and
     commit-only on a plan with decisions. Fix the
     `test_approve_commit_reject_feedback_epic_vectors` docstring.
   - Flow-level settle tests: settled from Telegram, from ACE, and from the CLI; an
     external reject; a missing `response.json`; and stale-after-submit restoring ↻ with
     the draft kept.
   - Make the `EPIC_DECISIONS_PLAN` test assert the epic-specific content, not only
     header text.
7. **Smaller items.**
   - `_disable_decision_controls` saves a retry record and immediately deletes it
     (`gate_response.py` ~198-199), so a failed keyboard removal is never retried.
   - The Telegram settle path leaves the pending action for a later sweep, which edits
     the card a second time. Settle once.
   - Update the two comments that name `cli_answer._reject_detached_tty_options`
     (`gate_callbacks.py` ~197, `tests/test_custom_gates.py` ~972) to
     `cli_answer_submit.reject_detached_tty_options`.
8. Run `sase tool run check` in sase-telegram.

## 6. Completion evidence

Each phase's closing note lists:

- the items it fixed, each with its test;
- `sase tool run check` results, with any KNOWN failures named;
- confirmation that `sase bead epic-symbols <phase-bead>` is empty, or what remains
  re-keyed to which later phase.

This epic's land agent re-runs the audit checks above. After it closes this epic, it
resumes sase-1hi.10's interrupted landing through the `parent_bead` link: closing
sase-1hi.10 and then resuming sase-1hi.
