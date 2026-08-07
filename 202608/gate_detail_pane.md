---
tier: tale
title: Always render a gate detail card in the notification panel and rename the HITL tab to Gates
goal:
  Every notification the panel can highlight renders meaningful content in the right pane — gate-backed rows show a live
  decision card (status, context, branch/command preview, attachments, errors) built from the durable bundle, and every
  other row shows a compact summary card instead of an empty pane — while gate-backed rows group under a tab named
  `Gates` and custom gates are required to declare the title, icon, and notes that make that card readable.
proposed_by: bbugyi200.athena.ui
create_time: 2026-08-07 08:39:44
status: done
---

- **PROMPT:**
  [prompts/202608/gate_detail_pane.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gate_detail_pane.md)

# Plan: Always render a gate detail card in the notification panel and rename the HITL tab to Gates

## Why this is a tale

This is one cohesive change to a single subsystem — notification gate presentation — built almost entirely from
primitives that already exist and are already tested: the gate bundle resolver, the hash-verifying envelope loader, the
branch projection the gate modals already consume, the debug snapshot's status derivation, the notification modal's
existing question/report pane contract, and the existing pump-free task and debounce helpers. The work is a serial chain
— request contract, domain projection, pane renderer, pane wiring, tab label, docs — where each step consumes the
previous step's type, and every test needs the whole chain visible to assert anything meaningful. Splitting it into
phases would add handoffs without unlocking parallelism. There is no Rust core API change and no cross-repository
dependency to sequence.

## Current state and constraints

### The defect

`NotificationAttachmentMixin._display_file` (`src/sase/ace/tui/modals/notification_modal_attachments.py:34`) chooses the
right pane's content in exactly this order:

1. `_render_question_pane` — `UserQuestion` gates only.
2. `_render_report_pane` — report notifications only.
3. `notification.files` — syntax-highlighted file, image preview, or video placeholder.
4. otherwise: the title becomes `… · No files attached` and the content widget is updated with the empty string.

A custom gate declares no `presentation.files` and no `presentation.preview` in the common case, so it lands on branch
4: the pane is blank, exactly as the screenshot shows. Every non-question gate kind with no attachment has the same hole
(`HITL`, `LaunchApproval`, `TaskTriage`, `BeadSnooze`, and `CustomGate`), and so does every ordinary attachment-less
notification.

Everything a reviewer needs is already on disk in the gate bundle — it is simply never projected into the pane. Today it
is reachable only by pressing `Enter` (which opens the review modal) or `d` (which opens the debug modal).

### What already exists (reuse it; do not reimplement it)

- **Bundle resolution.** `resolve_notification_bundle(notification)` (`src/sase/notification_gates/paths.py:104`)
  returns a `ResolvedGateBundle` with `kind`, `root`, `request`, `response`, `cancellation`, and a `legacy` flag, or
  `None` when the notification is not gate-backed. Every neutral bundle uses the same filenames — `request.json`,
  `response.json`, `cancellation.json` (`paths.py:24-26`) — regardless of kind; the adapters' `request_filename` /
  `response_filename` fields describe legacy directories only.
- **Verified envelope loading.** `load_and_verify_bundle(bundle_root)` (`src/sase/notification_gates/hashing.py:27`)
  returns `(envelope, adapter)` after reverifying every recorded resource hash, raising `GateError` on tampering or
  corruption.
- **Branch projection.** `GateBranchData.from_envelope(envelope, default_feedback=…)`
  (`src/sase/ace/tui/modals/gate_branch_controls.py:24`) already turns a verified envelope into `query`, `options`,
  `groups`, `branches`, and `primary_branch`, validating that branches, groups, and options agree. It is a pure domain
  projection that imports only `sase.notification_gates.models`; it is parked in the TUI layer by history, not by
  design.
- **Status derivation.** `_derive_status` (`src/sase/notification_gates/debug.py:271`) already encodes the one correct
  rule for `PENDING` / `ANSWERED` / `CANCELLED` / `TIMED OUT` / `OVERDUE` / `UNKNOWN` from the terminal artifact kind,
  its `reason`, the request's readability, and the creation time plus `gate_timeout_seconds`.
  `build_gate_debug_snapshot` (`debug.py:62`) documents and guarantees the contract this pane needs: _"malformed or
  missing input never escapes"_.
- **Adapter registry.** `adapter_for_action(action)` (`src/sase/notification_gates/adapters.py`) maps a notification
  action to its `GateAdapter`, which carries `kind`, `display_title`, `default_feedback`, `generic_form`, and
  `neutral_only`. `PRIVILEGED_GATE_ACTIONS` is the frozen set of gate actions.
- **Two pane renderers to copy the shape of.** `NotificationQuestionMixin._render_question_pane`
  (`src/sase/ace/tui/modals/notification_modal_question.py:45`) and `NotificationReportMixin._render_report_pane`
  (`notification_modal_report.py:26`) both return `tuple[str, RenderableType] | None` — pane title plus content, `None`
  to decline. The question pane already establishes the visual grammar this plan extends: an identity header, a
  `Table.grid(expand=True)` status row with a right-aligned call to action, `rich.rule.Rule` separators, and `● / ○ / ✓`
  option markers.
- **Primary-branch badge.** `primary_action_badge(data, submit_key)` (`gate_primary_footer.py:34`) renders the bounded
  `key → action` badge the gate modals already show in their footers.
- **Presentation normalizers.** `src/sase/notification_gates/presentation.py` owns `normalize_gate_panel`,
  `normalize_gate_origin_agent`, `normalize_gate_snooze_until`, `RESERVED_GATE_PANELS`, and the `panel` / `origin_agent`
  `action_data` keys. `validate_gate_spec` (`validation.py:31`) is the one place cross-field gate presentation rules are
  enforced, and `_build_notification` (`service.py:313`) is the one place a gate's notification row is built.
- **Tab classification is Rust-owned; tab labels are Python-owned.** The core assigns each row exactly one tab key
  (`crates/sase_core/src/notifications/tabs.rs`), pins the `hitl` key first in display order, and never sends a label
  over the wire — `NotificationTabWire` carries `key`, `kind`, `count`, `oldest_activity_at`, `next_wake_at`, and
  `color` only. Every label the user sees comes from `_notification_tab_label` /`_SYNTHETIC_TAB_LABELS` in
  `src/sase/ace/tui/modals/notification_modal_tags.py:43`, used by both the modal tag strip and the top-bar indicator
  (`actions/lifecycle.py:114`, `actions/agents/_notification_provider_direct.py:27`).
- **TUI concurrency helpers.** `spawn_pump_free_task(owner, coro, name=…, registry_attr=…)` and
  `cancel_pump_free_tasks(owner)` (`src/sase/ace/tui/util/pump_tasks.py`), and `DetailPanelDebouncer(app, delay_s=…)`
  (`src/sase/ace/tui/util/debounce.py`), which coalesces a burst of `j`/`k` into one detail paint.

### Constraints

- **The pane must never block the event loop and never stat on the render path.** Highlight moves paint immediately;
  detail work is debounced and runs off the pump (`sase/memory/tui_perf.md` rules 1, 2, 7, 8). Any state captured before
  an `await` must be re-read after it (rule 4) — a summary that lands after the user moved on is discarded, not painted.
- **The pane must never raise.** A missing, legacy, tampered, half-written, or hand-deleted bundle must degrade to a
  readable card that points at `d` for debug. The notification modal is the user's only inbox; a traceback there is
  worse than a blank pane.
- **Layering.** `sase/notification_gates/` is backend and must not import from `sase.ace.tui`. Presentation-only
  styling, widget layout, and Rich renderables stay in `sase/ace/tui/`.
- **No Rust core change is in scope.** The tab label is Python-owned (see above), so the rename ships without a
  `sase-core-rs` release or floor bump. The core's own `tab_label("hitl") == "HITL"` is used only to sort _panel and
  tag_ tabs among themselves — the gates tab is pinned first and never sorted by label — so no behavior depends on it.
- **Symvision.** Every new public symbol needs a real non-test consumer; test-only references do not count. Do not
  import `_`-prefixed symbols across files.
- **Skill sources are generated artifacts' inputs.** Edit `src/sase/xprompts/skills/sase_gate.md` in this repo only. Do
  **not** run `sase skill init --force` or `chezmoi apply`: deployment happens from a clean, landed tree, after this
  work merges.
- **`just install` first, then `just check`.** This workspace may have drifted dependencies.

## What the user sees

Highlighting a pending custom gate in the panel:

```
🚀 sase-gn.10.2 · Pin sase-core-rs to >=0.19.0
sent today 00:57:22 · 7h ago · filed by @sase-gn.10.2

  ● Awaiting your decision                              press Enter to review
  expires in 12m                                       custom gate · pin-core-0.19

  Bead sase-gn.10.2 must pin sase-core-rs to >=0.19.0 before the snooze close
  path can be covered against a real store.
  #beads  #release

  ─────────────────────────────────────────────────────────────────────────────
  Decision                                (bump AND verify) OR reject

  ▸ 1  🚀 Bump the floor and verify                                      Enter
       ☑ bump      commands/bump
       ☑ verify    commands/verify                          ✎ note optional
    2  ✋ Leave the floor alone
       commands/reject                                      ✎ note required

  Attachments  1/2 · request.diff                                  C-n · C-p
```

An answered gate replaces the status row with `✓ Answered · you chose Bump the floor and verify`, marks the chosen
options `●` green and dims the rest, and prints the recorded feedback below the decision block. A cancelled gate shows
`⊘ Cancelled`, a timed-out gate `⧗ Timed out`, and a gate whose bundle cannot be read shows
`▲ Gate details unavailable — <reason>` with `press d to debug` on the right, above whatever the notification row itself
still carries. An attachment-less non-gate notification shows the same card shape minus the decision block.

## 1. Gate request contract: what a custom gate must declare

The card is only as good as the data behind it. Today a custom gate can be created with no icon, no notes, and no title
of any kind: the list row falls back to `(no message)`, and `CustomGateModal` titles every custom gate with the
adapter's generic `"Custom Gate"`. Fix the contract at the source.

### `presentation.title`

Add a new optional-everywhere, **required-for-`custom`** presentation field.

- In `src/sase/notification_gates/presentation.py`: add `GATE_TITLE_ACTION_DATA_KEY = "gate_title"` and
  `normalize_gate_title(value) -> str | None`, following the shape of `normalize_gate_origin_agent`. Accept `None`;
  otherwise require a string, strip it, and reject: empty after stripping, longer than `120` characters, any Unicode
  `Cc` control character, and any embedded newline (a title is exactly one line). Failures raise
  `GateError("invalid_presentation", "presentation.title", …)`. Export both new names from `__all__`.
- In `src/sase/notification_gates/validation.py`: call `normalize_gate_title(presentation.get("title"))` beside the
  existing `normalize_gate_panel` / `normalize_gate_origin_agent` calls, and add `GATE_TITLE_ACTION_DATA_KEY` to the
  `protected` set so a producer cannot write `gate_title` through `presentation.action_data`.
- In `src/sase/notification_gates/service.py::_build_notification`: project the normalized title into
  `action_data[GATE_TITLE_ACTION_DATA_KEY]` when present, exactly as `panel` and `origin_agent` are projected. This is
  what lets the pane render a real headline with **zero disk I/O** on the highlight path.

### Required fields for `kind: "custom"`

In `validate_gate_spec`, after the existing presentation checks, add a `custom`-only block. Each failure raises
`GateError("missing_presentation", "presentation.<field>", …)` with a message that names the field, says what it is for,
and shows the fix:

| Field                | Rule                                             | Message shape                                                                                             |
| -------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| `presentation.title` | present and non-empty                            | `custom gates require presentation.title: the one-line decision headline shown in the notification panel` |
| `presentation.icon`  | present (existing `validate_icon` still applies) | `custom gates require presentation.icon: one emoji or glyph identifying the decision at a glance`         |
| `presentation.notes` | at least one non-blank note                      | `custom gates require presentation.notes: at least one line of context explaining the decision`           |

Only `kind == "custom"` is gated. `plan`, `epic_plan`, `question`, `launch`, and `hitl` keep their current contracts;
their cards fall back to `adapter.display_title`.

This is a deliberate breaking change to the custom gate request contract. It cannot affect a gate that already exists
(validation runs only at creation), and the two in-repo generic-form producers are updated in the same change:

- `src/sase/bead/snooze_gate.py` — add `"title"` to the presentation dict it already builds (`snooze_gate.py:150`).
- `src/sase/bead/task_gate.py` / `src/sase/bead/_task_gate_spec.py` — add the matching `"title"` for task triage.

Both already declare `icon`, `notes`, and `panel`, so only the title is new. They are not `custom` kind, so this is a
quality improvement rather than a requirement — do it anyway so every built-in gate card reads well.

No new `sase gate create` CLI flag: `--origin-agent`, `--panel`, `--sender`, and `--tag` exist because a _wrapper_
injects host context, while `title` is authored content that belongs in the request JSON next to `notes`.

### Reserved panel names

Add `"gates"` to `RESERVED_GATE_PANELS` in `presentation.py`, keeping `"hitl"` reserved. After section 4 the synthetic
tab is labeled `Gates`; a gate declaring `panel: "gates"` would otherwise produce a second, indistinguishable tab with
the same label. `"hitl"` stays reserved because the core still keys the synthetic tab that way.

## 2. Domain projection: `GateSummary`

Add `src/sase/notification_gates/summary.py`. This is backend code — no Rich, no Textual, no `sase.ace` import — so the
same projection can back the mobile gateway or a future `sase gate show` without being re-derived.

### Move `GateBranchData` down a layer

`GateBranchData` is a pure envelope projection but lives in the TUI, and `summary.py` may not import from
`sase.ace.tui`. Move the dataclass (only the dataclass, not `GateBranchControls`) into
`src/sase/notification_gates/branches.py` and re-export it from `sase/notification_gates/__init__.py`. In
`src/sase/ace/tui/modals/gate_branch_controls.py`, replace the class definition with
`from sase.notification_gates.branches import GateBranchData`; `GateBranchControls` annotates its own `__init__` with
the type, so the import stays live and Symvision-clean. Every existing importer — `modals/__init__.py`,
`gate_primary_footer.py`, `custom_gate_modal.py`, `plan_approval_modal.py`,
`actions/agents/_notification_custom_gate.py`, `_notification_plan_gate.py`, `_notification_modals.py`, and
`tests/ace/tui/test_custom_gate_modal.py` — keeps its current import path unchanged.

### One home for the status rule

Extract `debug.py::_derive_status` into `summary.py` as a public, artifact-free function:

```python
GateSummaryStatus = Literal[
    "pending", "answered", "cancelled", "timed_out", "overdue", "unavailable"
]

def derive_gate_status(
    *,
    request_readable: bool,
    terminal_kind: str | None,      # "response" | "cancellation" | None
    terminal_readable: bool,
    cancellation_reason: str | None,
    deadline: float | None,
    now: float,
) -> GateSummaryStatus: ...
```

The rule is unchanged: an unreadable terminal artifact or unreadable request is `unavailable`; a response is `answered`;
a cancellation is `timed_out` when its `reason` is `"timeout"` and `cancelled` otherwise; a readable request past its
deadline is `overdue`; anything else is `pending`. `debug.py::_derive_status` becomes a thin adapter that calls it and
maps the result to the existing uppercase `GateDebugStatus` display vocabulary, so `tests/test_gate_debug_snapshot.py`
keeps passing untouched.

### The projection

```python
@dataclass(frozen=True)
class GateSummaryOption:
    id: str
    label: str
    icon: str | None
    argv: tuple[str, ...]
    feedback: GateFeedbackMode
    default_selected: bool
    selected: bool          # recorded in a terminal response

@dataclass(frozen=True)
class GateSummaryBranch:
    option_ids: tuple[str, ...]
    label: str              # the matching group's label, else the first option's label
    icon: str | None
    is_primary: bool
    options: tuple[GateSummaryOption, ...]

@dataclass(frozen=True)
class GateSummary:
    kind: str
    display_title: str            # adapter.display_title, e.g. "Custom Gate"
    title: str                    # presentation title, else display_title
    request_id: str
    status: GateSummaryStatus
    deadline_at: str | None       # ISO-8601, for "expires in 12m"
    query: str
    branches: tuple[GateSummaryBranch, ...]
    selected_option_ids: tuple[str, ...]
    feedback: str | None          # the reviewer's recorded note
    attachments: tuple[str, ...]  # basenames, in notification.files order
    error_count: int
    bundle_path: Path | None
    unavailable_reason: str | None
```

Two entry points, both of which **never raise**:

- `gate_summary_from_notification(notification) -> GateSummary | None` — returns `None` when
  `adapter_for_action(notification.action)` is `None` (not a gate row). Otherwise it builds the **instant tier** from
  the notification row alone, with no disk access at all: `kind` and `display_title` from the adapter, `title` from
  `action_data[GATE_TITLE_ACTION_DATA_KEY]` falling back to `display_title`, `request_id` from
  `action_data["request_id"]` falling back to `notification.id`, `attachments` from `notification.files`, `status`
  `"pending"`, empty branches, and `unavailable_reason=None`.
- `load_gate_summary(notification) -> GateSummary | None` — the **bundle tier**, safe to call only off the UI thread.
  `None` for non-gate rows. Otherwise start from the instant tier and enrich it:
  1. `resolve_notification_bundle(notification)`; `None` or `bundle.legacy` → return the instant tier with
     `unavailable_reason` set (`"no gate bundle is attached"` / `"this gate uses a legacy bundle layout"`).
  2. `load_and_verify_bundle(bundle.root)` → `(envelope, adapter)`. Any `GateError` or `OSError` → instant tier with
     `unavailable_reason=str(exc)`.
  3. `GateBranchData.from_envelope(envelope, default_feedback=adapter.default_feedback)` → fold `branches`, `groups`,
     `options`, and `primary_branch` into `GateSummaryBranch` values **in canonical query order**, with `is_primary` set
     on the branch whose option ids equal `primary_branch`. A `GateError` here degrades the same way, keeping whatever
     scalar fields were already recovered.
  4. Read `response.json` and `cancellation.json` when present (best effort, bounded) for `selected_option_ids`,
     top-level `feedback`, and the cancellation `reason`; mark `GateSummaryOption.selected` accordingly.
  5. Compute `deadline_at` from the envelope's `created_at` / `created_at_unix` plus `gate_timeout_seconds`, and
     `status` via `derive_gate_status`.
  6. `error_count` = number of entries in `<bundle>/errors`, `0` when the directory is absent.

Wrap the whole body in a `try/except Exception` that returns the instant tier with
`unavailable_reason=f"{type(exc).__name__}: {exc}"`. The "never raises" contract is the point; make it structural, not
aspirational.

## 3. The detail pane

### Shared palette

Add `src/sase/ace/tui/modals/notification_modal_palette.py` exporting the colors the panes share:
`PANE_ACCENT = "#00D7FF"`, `PANE_AWAITING = "#FFD700"`, `PANE_ANSWERED = "#5FD787"`, `PANE_WARN = "#FFAF5F"`,
`PANE_ERROR = "#FF5F5F"`, `PANE_RULE = "#3A526B"`, `PANE_KEY = "#87D7FF"`, `PANE_MUTED = "dim"`. Replace the private
duplicates in `notification_modal_question.py` (`_QUESTION_CYAN`, `_AWAITING_GOLD`, `_ANSWERED_GREEN`, `_RULE_COLOR`)
with imports from this module so both panes are provably the same visual family. Every constant gains a real non-test
consumer in this change.

### `NotificationGateMixin`

Add `src/sase/ace/tui/modals/notification_modal_gate.py` with `NotificationGateMixin`, following the question/report
mixin contract exactly:

```python
def _render_gate_pane(self, notification: Notification) -> tuple[str, RenderableType] | None
```

Returns `None` when `gate_summary_from_notification(notification)` is `None`. Otherwise it renders, top to bottom, from
the best summary currently available (see caching below):

1. **Status row** — a `Table.grid(expand=True, padding=(0, 1))` with two columns, mirroring
   `notification_modal_question._status_line`: | status | left | right | | ------------- |
   ------------------------------- | ---------------------------- | | `pending` | `● Awaiting your decision`
   (`PANE_AWAITING`, bold) | `press Enter to review` (`PANE_KEY`, bold) | | `answered` | `✓ Answered` (`PANE_ANSWERED`,
   bold) | `you chose <labels>` (dim) | | `cancelled` | `⊘ Cancelled` (bold dim) | `no longer actionable` (dim) | |
   `timed_out` | `⧗ Timed out` (`PANE_WARN`, bold) | `the deadline passed` (dim) | | `overdue` |
   `● Awaiting your decision` (`PANE_WARN`, bold) | `past its deadline` (dim) | | `unavailable` |
   `▲ Gate details unavailable` (`PANE_ERROR`, bold) | `press d to debug` (`PANE_KEY`) | A second dim grid row carries
   `expires in <relative>` (from `deadline_at`, only while `pending`) on the left and `<kind> gate · <request_id>` on
   the right; when `unavailable_reason` is set it replaces the left cell.
2. **Context** — `notification.notes` joined by newlines as plain `Text`, or `No context was provided.` dim italic when
   empty. Then the row's tags as `#tag` in `PANE_KEY` dim, reusing `notification_display_tags` and
   `shorten_notification_tag`.
3. **Decision** — skipped entirely when `branches` is empty (an unavailable or not-yet-loaded summary). Otherwise a
   `Rule(style=PANE_RULE)`, then a header grid with `Decision` bold left and the `query` dim right, then one block per
   branch in canonical order:
   - Branch header: `▸ ` for the primary branch (else two spaces), the 1-based branch number matching the `1`–`9` keys
     the gate modals already bind, the branch icon, and the branch label (bold; dim when the gate is terminal and this
     branch was not chosen). The primary branch's row carries `primary_action_badge(...)`'s `Enter` badge right-aligned.
   - Member rows, indented five spaces, one per option: a marker — `●` `PANE_ANSWERED` when the option was selected in a
     terminal response, `☑`/`☐` from `default_selected` for a multi-option branch while pending, nothing for a singleton
     branch — then the option label, then `·` and the command as `argv[0]` plus any extra arguments, dim. Options whose
     `feedback` is not `disabled` append a right-aligned `✎ note required` / `✎ note optional` dim italic.
   - After the last branch, when `feedback` is set: `Note` bold on its own line and the recorded feedback below it.
4. **Attachments** — only when `attachments` is non-empty: `Attachments` bold, then `<i>/<n> · <basename>   C-n · C-p`
   dim, where `<i>` is `self._current_file_index + 1`.
5. **Errors** — only when `error_count > 0`: `▲ <n> command error(s) — press d to debug` in `PANE_ERROR`.

The returned pane title is `f"{icon} {sender} · {summary.title}"` via the existing `_detail_title` helper, so a gate row
never again reads `No files attached`.

### `NotificationSummaryMixin` — the last hole

Add `_render_summary_pane(self, notification) -> tuple[str, RenderableType]` to the same module (it always renders; no
`None` return). It reuses the status row's grid shape for an action badge line, then the notes body, then the tags, then
a dim `no attachments` line. This is what an ordinary attachment-less notification gets instead of a blank pane.

### Wiring in `_display_file`

Restructure `NotificationAttachmentMixin._display_file` so the pane is a **header card plus an optional body**, rather
than one of five mutually exclusive things:

1. `notification is None` → title `No notification selected`, empty body. (Only reachable with an empty inbox.)
2. `_render_question_pane` → return as today. Question gates keep their specialized pane unchanged.
3. `_render_report_pane` → return as today.
4. `header = _render_gate_pane(notification)`, else `header = _render_summary_pane(notification)`.
5. `body` = today's attachment rendering when `notification.files` is non-empty, otherwise nothing.
6. `content_widget.update(Group(header, Text(""), Rule(style=PANE_RULE), body))` when a body exists, else
   `Group(header)`.

This preserves every existing attachment behavior — syntax highlighting, image preview mode, the video placeholder,
`C-n`/`C-p` cycling, `Y`, `e`, `V`, and the scroll actions — and adds the card above it. A `PlanApproval` gate therefore
keeps showing its plan, now under a header that states whether the gate is still pending.

Add `GATE_HINT_TEXT` to `notification_modal_constants.py` beside `QUESTION_HINT_TEXT`
(`Enter: review  d: debug  C-n/C-p: file  C-d/C-u: scroll  m: mark  x: dismiss  M: mute  s: snooze  []: tags  q: close`)
and select it in `notification_modal_options.py:287` for rows whose action is in `PRIVILEGED_GATE_ACTIONS` minus
`UserQuestion` (which keeps `QUESTION_HINT_TEXT`).

Register `NotificationGateMixin` in `NotificationModal`'s bases in `notification_modal.py` ahead of
`NotificationAttachmentMixin`.

### Two-tier loading, caching, and cancellation

The render path must not touch the disk. Two tiers, one cache:

- `NotificationModal.__init__` creates `self._gate_summary_cache: dict[str, tuple[tuple[int, ...], GateSummary]]`
  (notification id → bundle fingerprint, summary) and `self._gate_summary_debouncer: DetailPanelDebouncer | None = None`
  (built in `on_mount`, which has an `app`).
- `_render_gate_pane` reads `self._gate_summary_cache` and falls back to `gate_summary_from_notification(notification)`
  — the zero-I/O instant tier — on a miss. It **never** stats and never loads. On a miss it also appends a dim
  `loading decision…` line under the status row, so the card visibly settles rather than silently changing shape.
- `_display_file` schedules the enrichment through `self._gate_summary_debouncer.schedule(...)` (150 ms), whose callback
  is thin and synchronous and does nothing but call
  `spawn_pump_free_task(self, self._load_gate_summary(nid), name=f"gate-summary:{nid}", registry_attr="_gate_summary_tasks")`.
- `_load_gate_summary(notification_id)` runs `asyncio.to_thread` over a worker that computes the fingerprint —
  `(st_mtime_ns of request.json, of response.json or -1, of cancellation.json or -1)` — returns the cached summary
  unchanged when the fingerprint matches, and otherwise calls `load_gate_summary(notification)`. Back on the UI thread
  it stores the result in the cache and repaints **only if** `self._get_highlighted_notification()` still has the same
  id (tui_perf rule 4).
- `NotificationModal.on_unmount` calls `cancel_pump_free_tasks(self)`.

The cache is per-modal-instance, so it dies with the modal and no test needs to reset global state.

## 4. Tab rename: `HITL` → `Gates`

In `src/sase/ace/tui/modals/notification_modal_tags.py`:

- Rename the constant `HITL_TAB_KEY` to `GATES_TAB_KEY`, keeping its value `"hitl"`, with a comment recording that the
  Rust core still keys the synthetic tab `hitl` and only the label is Python-owned.
- Rename `_HITL_ACTIONS` to `_GATE_TAB_ACTIONS`. **Do not change its membership** — it mirrors the core's `HITL_ACTIONS`
  list exactly and a parity test depends on that. `BeadSnooze` deliberately stays out: it declares `panel: "beads"` and
  routes there by the higher-precedence panel rule.
- Change `_SYNTHETIC_TAB_LABELS[GATES_TAB_KEY]` to `"Gates"`.

Nothing else changes: the tab key, the classification precedence, the pinned-first ordering, and the wire schema are all
untouched, so the top-bar indicator, the modal strip, and the mobile snapshot stay in agreement.

## 5. Tests

New:

- `tests/test_notification_gate_summary.py` — `gate_summary_from_notification` is `None` for a non-gate row, reads the
  title from `action_data`, and performs no disk access (assert with a `monkeypatch` that fails any
  `resolve_notification_bundle` call). `load_gate_summary` against real bundles built through `create_gate` (reuse the
  `gate_home` fixture pattern from `tests/ace/tui/test_notification_custom_gate.py`) for: pending; pending with a
  deadline in the past → `overdue`; answered with a recorded selection and feedback; cancelled; cancelled with
  `reason="timeout"` → `timed_out`; a bundle whose directory was deleted; a bundle whose `request.json` was truncated;
  and a legacy bundle. Every failing case returns a `GateSummary` with `unavailable_reason` set and never raises.
- `tests/test_notification_gate_presentation.py` — `normalize_gate_title` accepts, strips, and rejects (empty, >120
  chars, control characters, embedded newline); `gate_title` is projected into `action_data`; writing `gate_title`
  through `presentation.action_data` is rejected as reserved; `panel: "gates"` is rejected as reserved; a `custom` gate
  missing `title`, `icon`, or a non-blank `notes` fails with `missing_presentation` and the field named in `target`; the
  same request succeeds for `kind: "hitl"`.
- `tests/ace/tui/test_notification_gate_pane.py` — for each of `CustomGate`, `HITL`, `LaunchApproval`, `TaskTriage`,
  `PlanApproval`, and `BeadSnooze`: the pane title is not `No files attached` and the rendered card text contains the
  status line, the gate title, and the notes. Plus: the decision block lists every branch in canonical order with the
  primary branch marked; an answered summary marks the chosen options and prints the feedback; an `unavailable` summary
  renders the degraded card with `press d to debug`; a gate with attachments renders the card **and** the file body; a
  non-gate attachment-less notification renders the summary card instead of an empty pane; and a summary that lands
  after the highlight moved is not painted.

Extend:

- `tests/test_notification_modal_sections.py` — the synthetic tab label is `Gates` (update
  `test_tag_tabs_order_counts_and_capitalized_labels`, `test_mixed_tab_order_places_muted_last`, and
  `test_declared_panels_sort_after_hitl_and_before_other_tabs`); the pinned-first ordering is unchanged.
- `tests/test_notification_gates.py` / `tests/test_notification_gate_cli.py` — existing custom-gate fixtures need the
  new required fields; add one negative case proving the CLI prints `Error [missing_presentation] presentation.title: …`
  and exits `1`.
- `tests/ace/tui/test_notification_custom_gate.py` — `CustomGateModal`'s header shows the gate title with the kind as a
  dim badge, rather than the generic `"Custom Gate"`. Update `custom_gate_modal.py::_title` and `CustomGateModalData`
  (add `gate_title`, populated in `_load_custom_gate_modal_data` from the summary's title) accordingly.

Visual:

- `tests/ace/tui/visual/test_ace_png_snapshots_notification_gates.py` — two PNG snapshots at `120x40`, one pending
  custom gate card and one answered card, following the existing helper pattern in
  `tests/ace/tui/visual/test_ace_png_snapshots_notification_beads.py`. Generate the goldens with
  `just test-visual --sase-update-visual-snapshots` and inspect the produced PNGs before accepting them — these are the
  "beautiful" acceptance criterion, not just a regression net.

## 6. Documentation and skill

- `docs/notifications.md`:
  - Tabs table (line ~53): rename the `HITL` row to `Gates`, and update the `Panel` row's "sorted alphabetically after
    `HITL`" phrasing.
  - Tag precedence list (line ~457): `HITL` → `Gates`.
  - Gate presentation prose (line ~592): document `presentation.title`, its normalization rules, its projection into
    `action_data.gate_title`, and the fields `custom` gates must declare.
  - Reserved panel names: add `gates` alongside `errors`, `general`, `hitl`, `muted`, `snoozed`.
  - Add a short **Gate detail pane** subsection under "Viewing Notifications" describing what the right pane shows for a
    gate row, that it is always populated, and that `d` remains the escape hatch when a bundle cannot be read.
- `src/sase/xprompts/skills/sase_gate.md`:
  - Add `title` to the "Design The Gate" bullets and to the worked example's `presentation` block, and state that
    `title`, `icon`, and at least one `notes` line are required for custom gates.
  - **Fix the stale claim** that `hitl` is an allowed panel name — it is and has been reserved. List the full reserved
    set (`errors`, `gates`, `general`, `hitl`, `muted`, `snoozed`, and any `__`-prefixed name).
  - Add one sentence telling the author that everything they put in `title`, `icon`, `notes`, option labels, option
    icons, and command paths is rendered directly in the panel's detail pane, so those fields are the user's whole view
    of the decision.
  - Do not deploy; see Constraints.

## Out of scope / follow-ups

- **Rust core label parity.** `crates/sase_core/src/notifications/tabs.rs::tab_label` still returns `"HITL"`, and its
  `RESERVED_PANELS` still omits `"gates"`. Neither is user-visible through this repo (the label is Python-owned, and
  gate creation — the only writer of `panel` — is Python-side and strictly stricter), so shipping this change needs no
  core release. File a task bead through `/sase_new_task` to bring the core's label and reserved set in line on its next
  release.
- **Moving `BeadSnooze` into `_GATE_TAB_ACTIONS`.** That is a core classification change, not a label change, and it
  would pull woken snooze gates out of the `Beads` panel they deliberately live in.
- **A `sase gate show` CLI on top of `GateSummary`.** The projection is deliberately frontend-free so this stays cheap
  later, but no CLI is added here.

## Verification

```bash
just install
just check
just test-visual        # after generating the two new PNG goldens
```

`just check-full` before landing, because this change touches the gate request contract, the notification modal, and the
shared tab-label vocabulary.

Manually confirm in `sase ace`: open the notification panel, highlight a pending custom gate and see the card, press
`C-n` on a gate that has attachments and see the card stay above the file, hold `j` through the whole list and confirm
no stall (`SASE_TUI_PERF=1`, p95 < 16 ms), and delete a pending gate's bundle directory out from under the panel to
confirm the degraded card appears instead of a traceback.
