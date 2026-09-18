---
tier: tale
title: Restore notification access beyond the first 100 rows
size: medium
goal:
  Keep every eligible unread notification accessible in the notification modal and
  through the Agents notification shortcut, regardless of backlog size.
proposed_by: bbugyi200.athena.0mp
create_time: 2026-09-18 05:38:27
status: wip
---

# Restore notification access beyond the first 100 rows

## Diagnosis

The user's missing Gates tab and failing Agents `,n` shortcut have the same cause: the
TUI provider silently truncates the unread dataset before either consumer sees it. This
reproduces with only two notification tabs, so a maximum tab count is not the cause of
this failure.

The relevant call chain is:

1. `src/sase/ace/tui/actions/agents/_notification_provider.py` defines
   `DEFAULT_NOTIFICATION_PAGE_LIMIT = 100`. Both the standalone reader and
   `AgentNotificationProviderMixin._read_unread_notification_page_from_provider()`
   impose that default; the mixin also substitutes it for an explicit zero using
   `limit or DEFAULT_NOTIFICATION_PAGE_LIMIT`.
2. `src/sase/ace/tui/actions/agents/_notification_provider_direct.py` loads the complete
   Rust-backed snapshot, reconciles terminal gate notifications, filters
   unread/non-silent/non-dismissed rows, and then slices them with `[: max(0, limit)]`.
   The returned `AceNotificationPage` nevertheless has `next_cursor=None`,
   `bounded=False`, and `truncated=False`.
3. `_show_notification_modal()` in
   `src/sase/ace/tui/actions/agents/_notification_modal_flow.py` supplies that truncated
   list to `NotificationModal`. The modal builds its tabs exclusively from the supplied
   rows in `_tag_tabs()`, so a tab vanishes when all its rows are older than the cutoff;
   counts for partially represented tabs are also incomplete.
4. `_jump_to_agent_notification()` in the same flow module searches the same truncated
   list for `PlanApproval`, `EpicApproval`, or `UserQuestion`. A missing plan/epic match
   silently returns. Questions alone have an existing marker-file fallback. The leader
   dispatcher in `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py` does not
   consult tab visibility: it calls this lookup directly.

A read-only, in-memory reproduction exercised the real provider mixin, modal
classification, agent identity matcher, and shortcut method. Only the store read,
terminal-gate reconciliation, configuration, and final approval handler were stubbed.
The selected agent and older plan shared both name and timestamp identity:

| Newer Done rows | Total eligible rows | Returned rows | Modal tabs  | Plan handler calls |
| --------------- | ------------------- | ------------- | ----------- | ------------------ |
| 99              | 100                 | 100           | Gates, Done | 1                  |
| 100             | 101                 | 100           | Done        | 0                  |
| 101             | 102                 | 100           | Done        | 0                  |

There is a separate, documented visual limit of four chips by default in
`src/sase/ace/tui/widgets/notification_indicator.py`, controlled by
`ace.notification_indicator_max_counts`. Its overflow chip and tooltip operate on the
complete tab summaries and do not control either lookup or modal membership. The modal's
`NotificationTagStrip` renders all supplied tabs, reducing inactive labels when its
measured width is narrow. Neither indicator overflow nor label compaction should remove
notification data.

## Scope and design

This is one medium tale: one coding agent can change the TUI adapter contract, add
consumer regressions, and update documentation together. Shared notification storage,
classification, ordering, gate state, and identity semantics already exist; reuse them.
The erroneous cutoff belongs to the Python TUI adapter, so this fix needs no new Rust
backend behavior or wire API.

The default unread read must return the complete eligible dataset. Preserve explicit
bounded reads for existing internal callers, with accurate metadata. Do not replace 100
with a larger finite constant, prioritize Gates ahead of the cutoff, or invent a second
notification lookup/filter implementation. Those approaches leave other tabs, older
notifications, or future backlog sizes broken.

Keep the indicator chip budget, classification priorities, single-owner tab rules,
mute/snooze routing, keymaps, and existing approval handlers intact. There is no
configuration migration or new CLI option. This repairs the existing access contract.

## Implementation

1. **Make complete reads the default throughout the provider chain.**
   - In `_notification_provider_direct.py`, accept `limit: int | None = None`. Build the
     eligible list using the existing predicates and preserve snapshot order. Slice only
     when a numeric limit is explicitly supplied; preserve the existing direct-reader
     treatment of negative limits as zero.
   - In `_notification_provider.py`, make the standalone reader and mixin pass `None`
     through unchanged. Remove the implicit 100-row default and the truthiness
     substitution, updating stale docstrings to explain complete default reads and
     optional bounded reads. Check all references before removing the constant.
   - For explicit numeric limits, set `bounded=True` and set `truncated` exactly when
     eligible rows were omitted. For unlimited reads both flags are false. Keep counts
     as snapshot-wide counts. Preserve the existing metadata adapter's propagation of
     these flags and its row handles. Direct reads have no pagination capability; do not
     invent a cursor or pagination loop for this repair.
   - Keep terminal-gate reconciliation and the post-reconciliation snapshot reread. Keep
     `include_dismissed` behavior explicit and preserve filtering of read and silent
     rows. Muted and snoozed unread rows remain eligible for their own tabs.

2. **Keep both user entry points on the complete read path.**
   - Verify `_show_notification_modal()` and `_jump_to_agent_notification()` use the
     unlimited default and do not add a consumer-specific cap.
   - Preserve supported approval/question actions, exact identity matching, existing
     hidden-agent reveal behavior, and the dismissed-question marker fallback. No gate
     is answered, dismissed, or marked read simply by finding it.
   - The underlying store is already read in full before today's slice. Reuse that
     single snapshot path; add no per-tab store reads, synchronous I/O, or full-store
     reads inside row rendering. Preserve existing refresh and cache mechanisms. Assess
     modal opening with a larger fixture as described below; do not restore a silent cap
     to handle rendering cost.

3. **Add regressions at the provider-to-consumer boundary.**
   - Add a focused provider regression module under `tests/ace/tui/`. Use an isolated
     store fixture or stub only the underlying snapshot and reconciliation boundary;
     exercise the actual direct reader, standalone wrapper, and mixin. A test that
     supplies an already-complete mock page cannot catch this bug.
   - Cover 100, 101, and a comfortably larger eligible backlog (for example 250 rows),
     including an older gate and an older non-gate custom tab. Assert every eligible ID
     and the source ordering survive the default read, and metadata/row handles describe
     the complete result. Include empty input and rows excluded by read/silent/dismissed
     state, plus included muted/snoozed rows.
   - Cover explicit `None`, zero, negative, smaller positive, and sufficiently large
     limits through the wrapper and mixin as well as the direct reader. Assert truthful
     bounded/truncated flags and snapshot-wide counts, including the include-dismissed
     case. Reuse existing terminal-gate reconciliation coverage in
     `tests/ace/tui/test_notification_custom_gate.py` and add a beyond-cutoff case if
     needed to prove an old live gate survives while a terminal gate is repaired.
   - Exercise `_show_notification_modal()` using the real provider chain and capture the
     pushed modal. Assert Gates and the older custom tab exist, their counts are
     complete, and selecting those tabs exposes the expected rows. Include more than
     four populated tabs to demonstrate independence from the indicator chip budget.
   - Exercise the agent shortcut with 100 or more newer unrelated rows and an older
     matching `PlanApproval`, `EpicApproval`, and `UserQuestion` (parameterized).
     Include matching name plus timestamp/root timestamp, a nearer nonmatching
     notification, and no-match/marker-fallback cases. Patch the final handlers to
     record dispatch without opening or answering live gates. Each supported match must
     dispatch exactly once; unrelated agents must never dispatch.
   - Include a small headless Textual/ACE test that presses comma then `n` on a selected
     asking agent, so the regression covers key dispatch as well as the helper. Use
     existing test fixtures and await meaningful UI state rather than fixed sleeps.
     Exercise the real provider boundary in this test too.

4. **Document the restored behavior.**
   - In `docs/notifications.md`, distinguish the complete modal dataset from the top-bar
     chip budget and describe backlog-independent agent notification jumps. Keep the
     existing documented indicator overflow behavior accurate.
   - Update the relevant `?` help descriptions under
     `src/sase/ace/tui/modals/help_modal/` to reflect access to all unread
     notifications, respecting the existing help box widths. Reuse configured key
     labels; the shipped `,n`, `i`, and default configuration need no changes.

## Validation and acceptance

Read the current TUI and lint/test memories through `/sase_memory_read` before
implementation. Begin with the boundary regressions above and verify they fail on the
original cutoff, then pass with the fix. Use the project's installed Rust binding for
tab classification rather than duplicating classification in fixtures.

Run the focused new tests and relevant existing suites:

- `tests/ace/tui/test_notification_custom_gate.py`
- `tests/test_notification_modal_tab_routing.py`
- `tests/test_notification_modal_tab_order.py`
- `tests/test_notification_modal_tag_strip.py`
- `tests/test_notification_indicator.py`
- Existing notification plan/question and hidden-agent navigation tests selected by the
  changed imports, plus the new actual-keypress regression.

Use a deterministic headless fixture with an older plan behind at least 100 newer
notifications to verify the modal's Gates tab and selected row visibly render and the
shortcut opens the expected gate. Inspect the rendered result at a normal terminal size;
a count assertion alone is insufficient. For a live workflow check, use
`sase screenshot` per the TUI memory with an isolated fixture, without adding test
notifications to the user's live inbox or answering real gates. Also exercise opening
and navigating a moderately larger backlog (for example 1,000 rows); report any observed
responsiveness regression and address it through the existing TUI mechanisms.

Run `just fmt` (or `just fix`) and then `just check`. If verification needs a long
command, use `/sase_monitor`; use `just check-full` only when the repository's
escalation rules require it. Run the dedicated visual lane if visual fixtures or goldens
change, and inspect every intentional visual diff before accepting it.

The fix is complete when:

- No default notification read loses rows at 100 or any other arbitrary backlog size.
- Every populated tab and its complete eligible contents reach the modal.
- Agents `,n` dispatches the correct older plan, epic, or question notification
  independently of indicator overflow and modal tab visibility.
- Explicit bounded reads advertise omissions accurately and retain their documented
  filtering semantics.
- Existing filtering, gate reconciliation, identity matching, indicator behavior, and
  required checks pass without introducing additional store I/O in interactive paths.
