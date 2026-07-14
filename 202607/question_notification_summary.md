---
create_time: 2026-07-14 06:50:24
status: done
prompt: 202607/prompts/question_notification_summary.md
tier: tale
---
# Plan: Question Notification Summary in the Notifications Right Pane

## 1. Problem & Product Goal

In the `sase ace` **Notifications** panel, highlighting a `[question]` notification shows nothing useful on the right —
just **"No files attached"** and an empty body. Question notifications are the one notification type the user most needs
to _understand at a glance_ before deciding whether to drop what they're doing and answer, yet the right pane is blank.

**Goal:** When a `[question]` notification is highlighted, the right pane should render a clear, reliable, and beautiful
**summary of the question the agent asked**, making it unmistakable **which agent asked it**, what it wants, what the
options are, and whether it still needs an answer.

This is a **read-only preview**. Pressing `Enter` still opens the existing interactive answer modal — this plan does not
change the answering flow, only the at-a-glance summary.

### Success criteria

- Highlighting a `[question]` notification replaces "No files attached" with a summary that shows:
  1. **Which agent asked** — friendly agent name, provider/model, ChangeSpec, and when.
  2. **What was asked** — every question's header + full text (multi-question aware).
  3. **The choices offered** — each option's label + description, and single- vs multi-select.
  4. **Answer state** — Awaiting / Answered / Expired, with a clear call to action.
- Non-question notifications are completely unaffected (file attachments still render as today).
- Every failure mode (missing/expired/corrupt request file, no live agent row) degrades to a graceful, still-informative
  view — never a stack trace and never a blank pane.
- The result is visually polished and covered by a PNG snapshot so it stays that way.

## 2. Background: How Question Notifications Work Today

Verified against the current code so the design rests on real data, not assumptions.

- **Creation** (`src/sase/notifications/senders.py` → `notify_user_question`): a question notification is
  `action="UserQuestion"`, `sender="question"` (a constant string — **not** the agent name), `files=[]`,
  `notes=[<first-3-question-texts joined by "; ">]`, and `action_data` carrying `response_dir`, `session_id`, and (when
  available) `agent_cl_name`, `agent_project_file`, `agent_timestamp`, `agent_root_timestamp`.
- **The actual Q&A payload is NOT on the notification.** It lives on disk at `<response_dir>/question_request.json` =
  `{"questions": [...], "session_id", "timestamp"}`, where each question is
  `{question, header?, options: [{label, description?}], multiSelect?}` (schema per the `sase_questions` skill; written
  by `src/sase/axe/run_agent_helpers_questions.py`).
- **Answer state is derived from the filesystem**, not a field: a question is "answered" once
  `<response_dir>/question_response.json` exists (or the request file is gone) — this is exactly the rule
  `src/sase/notifications/pending_actions.py::_externally_handled` already uses. `question_response.json` =
  `{"answers": [{question, selected, custom_feedback}], "global_note"}`.
- **The asking agent is not named on the notification.** Only `agent_cl_name` (ChangeSpec) + timestamps are stored. A
  friendly agent name/provider/model must be resolved by matching the notification against the app's loaded agent rows
  via the existing `src/sase/notifications/agent_matching.py::agent_matches_notification_identity` /
  `src/sase/ace/tui/actions/agents/_notification_navigation.py::find_agent_for_notification`.
- **The right pane** is `NotificationAttachmentMixin._display_file()`
  (`src/sase/ace/tui/modals/notification_modal_attachments.py`), which updates `Label#notification-file-title` +
  `Static#notification-file-content` with Rich renderables. It short-circuits to "No files attached" whenever
  `not notification.files` — which is always true for questions.

### There is already a Python precedent for reading this file

The interactive answer modal (`src/sase/ace/tui/modals/user_question_modal.py`, opened via
`_notification_question_modal.py::_open_user_question_modal`) already reads `question_request.json` **in Python** and
renders it. Our summary is the read-only twin of that existing behavior.

## 3. Rust Core Boundary Decision (please confirm)

Per the repo's `rust_core_backend_boundary` rule, shared backend/domain behavior belongs in `sase-core`, while
presentation stays here. I investigated the Rust core (`sase-core`, `crates/sase_core/src/notifications/mobile.rs`) to
place this correctly:

- Rust core does **not** build a display summary for questions. Its mobile `UserQuestion` card is thin — `response_dir`,
  a hardcoded `[Answer, Custom]` choice set, and a `question_count` read straight from `action_data` (which Python does
  not even populate today, so it is `0`).
- Rust parses `question_request.json` **only at answer-validation time** (`plan_question_action_response`), to resolve
  one selected option when a mobile client submits an answer. It never extracts the full multi-question / options /
  answer-state view a rich display needs.
- The **answer-state** rule is already dual-maintained (Python `pending_actions.py` ↔ Rust `pending_actions.rs`), and
  the display-time request parsing already exists **in Python** (the answer modal).

**Recommendation (what this plan implements):** put the summary _extraction_ in a small, pure, presentation-free
**Python** module in the existing notification domain (`src/sase/notifications/`), and keep only Rich rendering in the
TUI. Rationale: there is no core summary to reuse, an in-repo Python precedent already parses this exact file for
display, and standing up a new Rust projection + PyO3 bindings + wire dataclasses + parity tests is disproportionate to
a right-pane UI improvement. The module will be deliberately pure so it can be **promoted to `sase-core` later** if/when
mobile or a web client wants byte-identical summaries.

**Decision point for review:** if you'd rather I build the summary projection in `sase-core` now (so mobile/CLI share
it), say so and I'll re-scope Phase 1 as a cross-repo change (`sase-core` wire + builder + `sase_core_rs` binding +
parity test, consumed via a Python facade). Everything else in this plan stays the same.

## 4. Design

### 4.1 Right-pane visual design (the "beautiful" part)

Rendered into the existing scrollable content `Static` (so `Ctrl-d`/`Ctrl-u` already scroll long multi-question
requests). Illustrative mock for an awaiting question:

```
 Question from  codex--fix-hook   CODEX(gpt-5.6-sol)
 memory-protection · asked 4m ago · session 3f2a…

 ⏳ Awaiting your answer                         press Enter ↵ to answer
 ────────────────────────────────────────────────────────────────────
 Q1  Protected memory regenerated
 The commit hook regenerated protected memory/pr files. How should I
 proceed with the pending commit?

   ○ Amend and keep — fold the regenerated files into this commit
   ○ Revert regeneration — restore the protected versions
   ○ Abort — let me inspect manually
   single-select
 ────────────────────────────────────────────────────────────────────
 Q2  Run tests?  …
```

Design principles, all consistent with existing TUI idioms (`rich.text.Text.append(style=...)`, provider palette in
`provider_styles.py`):

- **Identity header ties the question to its agent.** "Question from" (dim) + the resolved **agent display name in its
  provider color** (reusing `provider_styles`) + `PROVIDER(model)` badge. Second line: humanized ChangeSpec · relative
  age · short session id. Coloring the asker by provider is the primary "which agent asked" signal and matches how
  agents are colored elsewhere in the TUI.
- **Status pill** with a right-aligned call to action:
  - `⏳ Awaiting your answer` (gold `#FFD700`) + `press Enter ↵ to answer`
  - `✅ Answered` (green) + a compact `You chose: <labels>` line (+ global note if present)
  - `⌛ Expired or already handled` (dim) when the request file is gone with no response
- **Per-question sections** separated by a dim horizontal rule:
  - `Q<n>` badge (cyan `#00D7FF`, matching the existing `[question]` badge) + bold header.
  - Full question text, word-wrapped.
  - Options as an aligned list: `○` marker + bold label + `—` + dim description. In the answered state the chosen
    option(s) render with a green `●`/`✓`.
  - A dim italic `single-select` / `multi-select` tag.
- **Title label** (`#notification-file-title`) becomes a concise section label (e.g. `Agent Question`); the rich,
  colorful detail lives in the body.

### 4.2 Module structure

**New — pure extraction (no Textual/Rich imports):** `src/sase/notifications/question_summary.py`

- `QuestionOption(label, description, selected=False)`
- `QuestionEntry(prompt, header, options, multi_select, custom_answer=None)`
- `QuestionSummary(asker_cl_name, session_id, answer_state, questions, global_note=None, detail_available=True, fallback_note=None)`
- `AnswerState = Literal["awaiting", "answered", "expired", "unavailable"]`
- `is_question_notification(notification) -> bool`
- `question_answer_state(notification) -> AnswerState` — mirrors the `_externally_handled` semantics (kept consistent
  with `pending_actions.py`), from `response_dir` file existence.
- `build_question_summary(notification) -> QuestionSummary | None` — returns `None` for non-questions; otherwise reads
  `question_request.json` (and `question_response.json` when answered) defensively. Any missing/corrupt input yields a
  summary with `detail_available=False` and `fallback_note` set from `notification.notes[0]`, so the pane is never
  blank.

Exported from `src/sase/notifications/__init__.py`.

**New — TUI rendering mixin:** `src/sase/ace/tui/modals/notification_modal_question.py`

- `NotificationQuestionMixin._render_question_pane(notification) -> tuple[str, RenderableType] | None` — returns
  `(title_text, content_renderable)` or `None` when not a question.
- `_resolve_asker(notification, summary)` — best-effort friendly identity: try
  `find_agent_for_notification(self.app, notification)` for `display_name` + `llm_provider` + `model`; fall back to
  `humanize_cl_name(summary.asker_cl_name)`; then to a generic label. All wrapped in `try/except` so a missing app or
  unmatched row never breaks rendering.

**Edited — wire it in:**

- `src/sase/ace/tui/modals/notification_modal.py`: add `NotificationQuestionMixin` to `NotificationModal`'s bases +
  import.
- `src/sase/ace/tui/modals/notification_modal_attachments.py`: at the top of `_display_file`, before the
  `not notification.files` branch, add a single dispatch: if `notification is not None` and
  `self._render_question_pane(notification)` returns a result, set `_set_image_preview_mode(False)`, update title +
  content, reset scroll, and return. This keeps one dispatch point and is future-proof for other action-typed summaries
  (plan/launch) later, without over-building now.

**Optional polish (same PR):**

- Context-aware bottom hint: when the highlighted row is a question, swap the generic hint for a question-focused one
  emphasizing `Enter: answer` (in `on_option_list_option_highlighted` / `_update_hint_footer`, mirroring the existing
  jump-mode hint swap). No new keybinding is added (Enter already answers; `Ctrl-d/u` already scroll), so per
  `src/sase/ace/CLAUDE.md` the help modal and keybinding footer should not need changes — this will be re-verified.

### 4.3 Reliability / edge cases (explicit)

1. Not a `UserQuestion` → `_render_question_pane` returns `None`; existing file logic runs unchanged.
2. `response_dir` absent from `action_data` → `unavailable` state; render header + fallback note.
3. `question_request.json` missing (expired/consumed) with no response → `expired`; header + note.
4. Corrupt/invalid JSON → `detail_available=False`; header + fallback note (never raises).
5. `question_response.json` present → `answered`; flag selected options, show choices + global note.
6. Multiple questions / very long text → all rendered, scrollable via existing `VerticalScroll`.
7. Agent row not currently loaded / `self.app` unavailable (e.g. isolated tests) → fall back to humanized
   `agent_cl_name`; summary builder itself is fully app-independent.
8. Switching from an image/file notification to a question and back → mirror the existing no-file branch
   (`_set_image_preview_mode(False)`, consume image cleanup) to avoid stale state.

## 5. Testing Strategy

- **Unit — pure builder** (`tests/test_notification_question_summary.py`): non-question → None; single-question with
  options; multi-question + `multiSelect`; missing `response_dir`; missing request file; corrupt JSON; answered
  (response present) with selected options + global note. Uses `tmp_path`; no TUI. This is the correctness backbone.
- **Unit — rendering mixin** (`tests/test_notification_modal_question_pane.py`): drive `_render_question_pane` with
  synthetic notifications (reusing `tests/_notification_modal_helpers.py` patterns) and assert the rendered plain text
  contains the agent name, each question's text, the option labels, the select-mode tag, and the correct status label —
  across awaiting / answered / expired.
- **Visual PNG snapshot** (`tests/ace/tui/visual/test_ace_png_snapshots_notification_question.py`): build a temp
  `response_dir` + `question_request.json`, create a `UserQuestion` notification, open `NotificationModal`, highlight
  it, and snapshot the pane (following `test_ace_png_snapshots_help_panel.py` + `_ace_png_snapshot_helpers.py`). Assert
  the SVG contains the agent name, a question text, and "Awaiting"; add golden(s) under
  `tests/ace/tui/visual/snapshots/png/` (an awaiting snapshot; an answered one if cheap). This satisfies both the
  "beautiful" bar and the repo's visual-snapshot conventions.
- **Regression:** existing notification-modal suites must stay green — the new branch only triggers for
  `action == "UserQuestion"`, leaving file/image/video rendering untouched.

## 6. Out of Scope / Non-goals

- No change to the interactive answer modal (`UserQuestionModal`) or the answer-write flow — `Enter` still opens it.
- No new notification fields and no change to the notification store / Rust wire.
- No `sase-core` changes (unless the Section 3 decision point flips to "build it in core").
- Optional, noted-not-done: populating `action_data["question_count"]` in `notify_user_question` would also fix the
  mobile card's `question_count: 0`; and `sase notify show` could reuse the new pure builder for a richer question view.
  Both are natural, low-risk follow-ups enabled by the pure module, but are not required for this feature.

## 7. Rollout Checklist

- Implement the pure module + exports; add the TUI mixin; wire the single `_display_file` dispatch; add the class base.
- Add unit + rendering + PNG snapshot tests; generate goldens with `--sase-update-visual-snapshots` and eyeball them.
- Re-verify no keybinding/help drift (`src/sase/ace/CLAUDE.md`).
- Run `just install` then `just check` (and `just test-visual`) before handing back.

## 8. Files Touched (summary)

- **New:** `src/sase/notifications/question_summary.py`
- **New:** `src/sase/ace/tui/modals/notification_modal_question.py`
- **Edit:** `src/sase/notifications/__init__.py` (exports)
- **Edit:** `src/sase/ace/tui/modals/notification_modal.py` (mixin base + import; optional hint swap)
- **Edit:** `src/sase/ace/tui/modals/notification_modal_attachments.py` (dispatch in `_display_file`)
- **New tests:** `tests/test_notification_question_summary.py`, `tests/test_notification_modal_question_pane.py`,
  `tests/ace/tui/visual/test_ace_png_snapshots_notification_question.py` (+ golden PNG[s]) </content> </invoke>
