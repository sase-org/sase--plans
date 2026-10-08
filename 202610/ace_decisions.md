---
tier: tale
title: ACE Decisions accordion and compact plan-review Verdict
goal:
  Plan review in ACE shows each Plan Decision above a compact docked Verdict, submits
  exactly the values on display bound to the displayed review revision, and renders
  those decisions on the toast, inbox, gate card, and PLAN lane.
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.6
bead: sase-1hi.6
create_time: 2026-10-08 02:01:48
status: wip
---

- **PARENT:**
  [202610/plan_decisions.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions.md)
- **BEAD:**
  [sase-1hi.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.6.md)

# ACE Decisions accordion and compact plan-review Verdict

Implement epic phase `tui` of `plan:202610/plan_decisions.md` (bead `sase-1hi.6`). That
plan's sections 1.2, 1.6, 3, and 6.6 are the contract. This tale is the file-level cut.
Do not reopen grammar, resolution, stamping, or the `plan_decisions` flag. Those belong
to phases that already landed or to `sase-1hi.9`.

The product term is **Plan Decision**. In code say `plan_decisions`. The existing
`PlanApprovalDecisionsMixin` is the result protocol (approve, reject, feedback, the `c`
round trip). Leave it as that. Do not turn it into the accordion.

## Read first

With `sase memory read`:

- `tui.md`, `tui_perf.md`, `tui_screenshot.md` before editing ACE.
- `lint_and_test.md` before verification.
- `symvision.md` only if a symvision failure is yours to fix.

Do not open sidecar plan or research files directly. The design you need is in this
tale. Do not edit `sase-core` and do not move `sase-core-revision.txt`.

## What already exists

Use it. Do not reimplement it.

- Frozen definitions are `payload.decisions` on the plan-gate envelope, built only when
  the `plan_decisions` flag is on (`src/sase/plan_gate.py`). Each item has `id`, `kind`
  (`choice` or `toggle`), `ask`, optional `why`, `choices` (`key`, `label`), `default`,
  `effective_default`, and optional `memory` (`selectors`, `provenance`, `quote`,
  `resolved`).
- `sase.sdd.plan_decisions.sheet_binding(definitions, values, review_revision)` returns
  the Decision Sheet (`rows` in author order, plus counts).
- `summary_binding(sheet, verdict, form)` returns the shared sentence. Verdict is
  `coder + commit`, `coder`, `commit`, or `epic launch`. Pass the form the binding
  accepts for the full sentence and for the short form (`defaults` or `N change(s)`,
  plus ` · 🧠` when a memory edit is on). Do not build that sentence in Python.
- Glyphs and chips live in `src/sase/sdd/_plan_display_decisions.py`:
  `pending_decisions_text`, `accepted_decisions_text`, `format_decision_value` (`yes` /
  `no`), `provenance_chip`. PLAN lane and the prompt panel must call these, not a second
  painter.
- `decision_<id>` properties are already host-collected. `HOST_COLLECTED_PROPERTIES` in
  `plan_approval_gate_data.py` treats any `decision_` name as collected, so the input
  panel and `✎ n inputs` must not list them. Reuse that rule everywhere a field list is
  built. `c` and `i` never show decisions.
- `normalize_plan_option_inputs` resolves one vector. Differing `decision_*` values
  across selected options raise `decision_conflict`. Write the same value onto every
  selected option that declares that key.
- `sase gate answer` already reads `review_revision` from the request payload
  (`notification_gates/cli_answer.py`) and `execute_gate_selection` refuses a mismatch
  as `stale_review` before any side effect. ACE does not send it yet.
- Gate notes already include a second line `N decisions · 🧠 M`. The plan toast
  (`_plan_toast` in `src/sase/ace/tui/actions/agents/_toasts.py`) ignores it. Inbox rows
  (`notification_modal_options.py`) show only `notes[0]`.
- Callout spans are on the validated plan (`decision_callouts`: `id`, optional `key`,
  `branch`, `start_line`, `end_line`). They are not copied onto the gate envelope.
- `markdown_document_syntax` caches a token stream in `util/frontmatter_syntax.py`.
- `review_revision` lives on the gate envelope (starts at 1, bumps on an accepted edit).

## Behaviour

**Approve as shown.** The reviewer can change a value without submitting. Enter, the
digits, and the mouse approve exactly the values on display. Accepting every default is
one keystroke. Enter never edits a decision.

**Compact Verdict on every plan review**, with or without decisions, so the layout does
not jump. Drop the Cancel button. `q` and Esc still cancel. Custom gate modals keep
today's layout.

The rail has three parts:

1. **Actions**, one line, unchanged in meaning.
2. **Decisions**, omitted entirely when the envelope has no definitions. Header
   `Decisions  N · 🧠 M`. Rows scroll inside the rail.
3. **Verdict**, docked to the bottom, renamed from the section titled `Decision`.
   - Line 1: AND toggles with short labels (`☑️ 🚀 Launch coder`, `☑️ 💾 Commit plan`).
     The full option label (`Launch coder agent`,
     `Commit plan file to the plans sidecar`) is the tooltip.
   - Line 2: compact numbered buttons (`1 ✅ Tale`, `2 ❌ Reject`, `3 💬 Feedback`; epic
     uses `1 ✅ Epic` and has no commit toggle).
   - Line 3: the full summary sentence from `summary_binding`.

Wide layout: the plan-review rail stays 42 cells with no decisions and becomes 50 only
when decisions exist. Do not widen `CustomGateModal` or the other `width: 42` rules. The
narrow breakpoint stays 100 columns. Under `-gate-review-narrow` the rail still docks to
the bottom at full width (`styles.tcss` already does this).

**Rows** (section 1.2). Author order, never re-sorted. The modal opens with the first
decision focused.

- A collapsed row is two lines: glyph, id, and current value, then the ask truncated or
  the memory chips.
- The focused row expands: the full ask, every choice with its consequence, `★` on the
  default and its `why`, and for a memory row the note chips (`core`, `reference`,
  `web`, `new`), the provenance chip (`you asked`, `not asked`,
  `⚠ quote not found · off`, `approved in epic`), and the quote.
- `☑️` / `⬜` are toggles. `◉` is the selected choice and the choice row. `○` is an
  unselected choice. Do not use `◆`, `⋔`, or `🎛`.
- `★` is the planner default. A gold `●` (`#FFD700`) marks a value changed from
  `effective_default`. Detail text also says the default, so colour is not the only cue.
- Toggle display is `yes` / `no`.
- An unverified memory row (provenance `quote_not_found`, or `effective_default` forced
  off) reads `⚠ "<quote>" — not in your messages · off until you turn it on` in the
  warning colour.
- When a human turns that row on, it shows `yes ●` and the summary names the note. There
  is no confirmation dialog.

**Keys.** Add them under `ace.keymaps.gate` in `src/sase/default_config.yml`, on
`GateModalKeymaps`, in `_GATE_BINDING_META`, and in `src/sase/config/sase.schema.json`
(`additionalProperties` is false). `build_gate_modal_bindings` already sets
`priority=True`. `gate_modal_taken_keys` only reserves keymap values of length 1, so
teach it to split comma-separated alternatives and reserve each single character.
Gate-action fallback keys must skip `h`, `l`, `r`, and `R`.

| Action                              | Default   | When a decision row is focused                                              |
| ----------------------------------- | --------- | --------------------------------------------------------------------------- |
| `toggle_option`                     | `space`   | Next choice, wrapping, or flip a toggle. On an AND member, toggle as today. |
| `decision_next`                     | `l,right` | Next choice, or set yes.                                                    |
| `decision_prev`                     | `h,left`  | Previous choice, or set no.                                                 |
| `decision_reset`                    | `r`       | Reset the focused row to `★`.                                               |
| `decision_reset_all`                | `R`       | Reset every row to `★`.                                                     |
| `next_control` / `previous_control` | `j` / `k` | Move through Actions, then Decisions, then Verdict.                         |

`CustomGateModal` uses the same binding list. Give it no-op handlers for the four new
actions so a custom gate does not crash. Space on a custom gate still toggles AND
members.

Footer hints for `h` / `l`, `r`, and `R` appear only when the plan has decisions. The
Enter badge becomes `Enter=Tale · defaults`, `Enter=Tale · N change(s)`, and the epic
equivalents, using the short summary. Append ` · 🧠` when any memory edit is on.

Update the Plan Approval Keybindings table in `docs/ace.md` and the `ace.keymaps.gate`
table in `docs/configuration.md`, plus the remap example that lists the gate keys.

**Document pane.**

- When the displayed text has a `decisions:` map, fold that map into one dim line:
  `decisions: N · answered in the Decisions panel`. `e` and `Y` still read and write the
  raw file (`_plan_content`), never the folded display.
- Cache callout spans once when content is set (open, and `render_reviewed_content`),
  from `validate_plan` in launch mode. Do not put callouts on the gate envelope. Do not
  validate, lex, or stat on a keypress. Keypress only chooses which cached span is lit
  and scrolls.
- Focusing a decision scrolls to its first callout, or else the first mention of its id.
  Map raw line numbers through the fold (many YAML lines become one).
- Tint the chosen branch green with a bold header. Dim the unchosen branch, or a toggle
  callout answered no. Never hide a branch.
- Reuse `util/frontmatter_syntax.py`. After the rows exist, measure a decision keypress
  with the `tui_perf` guidance. A keypress that re-lexes the document or reads the plan
  file is a defect.

**Every submit path** sends the sheet. Check each one:

- Enter, `ctrl+s`, digits, and mouse on the Verdict.
- `i` and input-panel completion.
- `c` → `ApproveOptionsModal`, including the `PendingApproveState` round trip.
- Programmatic `action_approve`, `action_tale`, `action_epic`, `action_feedback`,
  `action_reject`, and `action_submit_*`.
- `_plan_gate_submission_payload` in
  `src/sase/ace/tui/actions/agents/_notification_plan_gate.py`.

One helper builds the `decision_*` map from the draft and writes that same map into
every selected option whose schema declares the key. `approve` and `commit` therefore
cannot disagree. Reject's schema has no `decision_*` properties, so reject does not send
them. Feedback does. `c` and `i` still submit the hidden sheet values.

ACE sends the `review_revision` it displayed on every live-gate submit, including
reject. `durable_request_payload(...)` in `_submit_durable_neutral_plan_response` is the
durable path. Thread the same field through the non-durable
`execute_plan_approval_response` path when that path answers a live gate. Legacy
notifications with no bundle omit it (absent means unchecked).

**States.**

- Esc stores the draft per request id for the life of the ACE process and restores it
  when that request opens again. `r` / `R` update that store. Not on disk.
- An in-gate edit that touches `decisions:` already fails in the adapter with
  `Decisions are fixed for this review. Change answers in the Decisions panel, or send feedback to change the questions.`
  Show that text on the existing red "Draft not accepted" banner and keep the draft. Do
  not drop the edit silently.
- Feedback (`3`) carries a read-only first line on the feedback prompt:
  `Carries: grouping → mode` for each changed value, author order, toggles as `yes` /
  `no`. Thread it through `PlanFeedbackContext`. Unchanged decisions are omitted. The
  replanner prompt already renders provisional decisions. Do not redo that.
- Settled elsewhere: if this request already has a terminal response (Telegram, CLI, or
  another ACE), replace the controls with `Approved via <surface> · <summary>` using the
  full sentence, and refuse submit. Observe settlement from the existing notification
  refresh. Do not poll the bundle on the UI thread or add a timer that reads disk
  (`tui_perf` rules 1 and 2).
- `stale_review` reloads the bundle's plan text and revision, keeps the draft values,
  and does not approve. Surface the failure from the submit completion the same way
  other gate answer errors are surfaced.

**Around the modal.** Plans with no decisions stay pixel-identical on these surfaces.

- Plan toast (`_plan_toast`): add the existing second note (`N decisions · 🧠 M`) as a
  dim second line, after the epic detail line when that line exists.
- Inbox rows (`notification_modal_options.py`): when that second note is present, append
  ` · ◉N`, and ` 🧠` only when the memory count in that note is greater than zero. Keep
  the current 50-character truncation of `notes[0]`.
- Inbox gate card (`notification_modal_gate.py`): for a plan gate with definitions, add
  one Decisions block. Pending rows read `◉ grouping  pane ★`. Answered rows read
  `grouping: mode ●`. Do not also list `decision_*` as per-option fields, and do not let
  them increment `✎ n inputs`. Leave the shared branch header alone so existing
  notification-gate snapshots that have no decisions stay unchanged.
- PLAN lane and the agent prompt panel: render `pending_decisions_text` or, when
  `decided_by` is set, `accepted_decisions_text`. Prefer extending the shared plan
  document (`render_plan_document` / `plan_logical_text`) with an optional sheet so both
  surfaces pick it up. Load it with the existing mtime-keyed plan summary cache. A
  render with no sheet matches today's text exactly. Confirm the prompt panel uses that
  renderer before painting a second copy.
- `%auto` receipt rows already have `action=None`, `silent=True`, and tag
  `plan_decisions_receipt`. Add a unit test that the inbox renders those notes as a
  plain row and does not toast or open a gate card. No new snapshot unless a current
  snapshot paints the receipt as a broken gate.

## Modules

Keep `plan_approval_modal.py` a composer. Prefer new modules, and a different split is
fine if these boundaries stay:

- `plan_decision_sheet.py` — draft values, edits, reset, the sheet, and both summary
  forms. No Textual imports, so tests do not need a pilot.
- `plan_decision_rows.py` — the accordion widget.
- `plan_decision_document.py` — fold, callout span cache, tint, scroll target.

Pass definitions, callouts-or-content, and `review_revision` into `PlanApprovalModal`
from `PlanGateModalLoad` / `_notification_modals.py`. Direct callers that only pass a
path keep an empty sheet and the compact Verdict.

## Tests

Unit tests, not just snapshots:

- Collapsed versus focused row text, unverified copy, human override `yes ●`, author
  order, and reset.
- `j` / `k` order, space wrap and toggle flip, `h` / `l`, `r` / `R`, and Enter
  submitting rather than editing. A custom gate's new handlers no-op.
- Every submit path listed above attaches identical `decision_*` values and the
  displayed `review_revision`. Reject omits `decision_*`. `✎` counts ignore them.
- Fold line mapping, chosen-branch tint, and "scroll uses the folded coordinate".
- Esc restore, freeze banner text, feedback carry line, settled-elsewhere refusal,
  `stale_review` reload that keeps values.
- Toast second line, inbox suffix, gate-card block, PLAN document with and without a
  sheet, receipt row.

Update tests that assert the old "Decision" title, the Cancel button, the old footer, or
the old gate keymap field set. Known files:

- `tests/test_plan_approval_modal_title.py`
- `tests/ace/tui/test_gate_primary_footer.py`
- `tests/ace/tui/test_notification_plan_gate.py`
- `tests/test_keymaps_defaults_panels.py`
- `tests/test_keymaps_registry_loading_panes.py`
- `tests/test_config_schema_keymaps.py`

## Visual goldens

Fixtures must build a real plan gate (flag on, `payload.decisions` from the gate
builder), not hand-drawn rows. Add:

- `plan_gate_tale_decisions_120x40`
- `plan_gate_tale_decisions_memory_120x40`
- `plan_gate_tale_decisions_unverified_120x40`
- `plan_gate_tale_decisions_stacked_90x40`
- `plan_gate_epic_decisions_120x40`
- the inbox gate card, pending and answered, beside the harness in
  `tests/ace/tui/visual/test_ace_png_snapshots_notification_gates.py`
- the plan toast with decisions, using `stabilize_toast_frame`

Update the existing `plan_gate_*` goldens once for the compact Verdict, as one group:
`plan_gate_tale_five_controls_120x40`, `plan_gate_frontmatter_120x40`,
`plan_gate_epic_action_120x40`, `plan_gate_tale_stacked_90x40`.

Read `/sase_monitor` and run `just fix-tui-screenshots` through it, with those snapshot
modules as selectors after `--`. Do not run a full visual suite and do not run
`just check-full`. Inspect `.pytest_cache/sase-visual/latest-report.json`: every
creation and every update group. `partial` means the skipped goldens are not current.
Generation is not approval. Look at the new shots and confirm the expanded row, the gold
`●`, the folded frontmatter line, the docked three-line Verdict, and the narrow stack.

## Verification and close

Run `sase tool run check` in this repo. Do not run `just check-full`. Do not bypass
`sase tool run` unless the tool is missing, and then record why.

Before closing, run `sase bead epic-symbols sase-1hi.6`. There are none today. Do not
add an `--epic-symbol` entry keyed to `sase-1hi.6`. A symbol that must stay open belongs
on the still-open parent `sase-1hi` or on `sase-1hi.9`. `sase bead close` refuses while
a leftover is keyed here.

Close only this bead:

```bash
sase bead close sase-1hi.6 --note "<what you verified>"
```

Do not close `sase-1hi`, `sase-1hi.9`, or any other ancestor. Do not create beads.
Record discovered follow-up work as
`sase bead note sase-1hi.6 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`.

A `sase tool run check` failure that reproduces identically on the clean base tree does
not keep this bead open. Note it as a `PROPOSED FOLLOW-UP:` (cite a task bead if one
already tracks it) and close anyway.

## Out of scope

- Telegram, the finalizer guard, skill text, and deleting the `plan_decisions` flag.
- New gate wire fields, resolver changes, and sase-core edits.
- Generic `TypedInputForm` toggles and any Android rendering.
- Memory notes, glossary strands, and decision records. This epic edits no memory.
