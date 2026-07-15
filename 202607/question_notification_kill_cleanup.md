---
tier: epic
title: Dismiss open question notifications when agents are killed
goal: 'Every successful agent-kill surface dismisses notifications associated with
  the killed agent, including open UserQuestion notifications routed through a child
  phase and its visible root agent, without broadening matches to unrelated agents
  or allowing notification cleanup failures to invalidate a successful kill.

  '
phases:
- id: core-matching
  title: Root-aware notification matching in sase-core
  depends_on: []
- id: kill-lifecycle
  title: Named-agent kill cleanup and end-to-end regressions
  depends_on:
  - core-matching
create_time: 2026-07-15 09:47:37
status: wip
prompt: 202607/prompts/question_notification_kill_cleanup.md
bead_id: sase-63
---

# Plan: Dismiss open question notifications when agents are killed

## Context and diagnosis

`/sase_questions` writes a pending-question marker for the running phase and emits a `UserQuestion` notification. The
notification carries `agent_cl_name`, the concrete asking phase in `agent_timestamp`, and the visible/aggregate agent in
`agent_root_timestamp`.

ACE's in-process cleanup path already asks the notification store to dismiss notifications for killed agents. Its Rust
cleanup plan emits notification dismissal candidates, and workflow-parent cleanup normally includes loaded children. By
contrast, `kill_named_agent`—used by `sase agent kill` and the mobile kill integrations—releases the workspace or stale
markers and records the agent in the dismissed-agent index, but never calls the notification store. This confirms the
reported behavior for those kill surfaces.

There is a second contract gap: named-agent discovery resolves the root artifact identity, while the Rust
`UserQuestion`/`PlanApproval` matcher only compares an agent key against `agent_timestamp`. It ignores
`agent_root_timestamp`, even though the Python presentation/navigation matcher correctly treats both timestamps as
identities for the same notification. Consequently, merely adding the missing named-kill notification call would still
leave a root-routed question active when its asking child has a different timestamp. The same gap also makes ACE cleanup
depend unnecessarily on the asking child being present in its loaded cleanup snapshot.

The fix belongs in the shared Rust notification backend for identity semantics, followed by the Python shell's external
kill lifecycle. The generated `/sase_questions` skill template is behaving as designed and should not be changed.

## Phase 1: Root-aware notification matching in `sase-core`

Open the linked `sase-core` repository through the audited `/sase_repo` flow before making changes there.

Update the shared agent-notification matcher so `UserQuestion` and `PlanApproval` notifications can match either
normalized `agent_timestamp` or normalized `agent_root_timestamp`, while continuing to require the existing
ChangeSpec/agent name key. Preserve the current safe fallback for legacy notifications without usable timestamps, the
13-digit and 14-digit timestamp normalization behavior, idempotent dismissal, and the existing action-specific rules for
completion/error notifications. Do not weaken matching to timestamp-only or ChangeSpec-only when a valid timestamp is
present.

Add Rust contract tests that exercise the real bulk state update, including:

- a question with distinct child and root timestamps dismissed by the root key;
- the same notification dismissed by the child key;
- legacy timestamp normalization;
- nearby non-matches with the wrong ChangeSpec or timestamp remaining active;
- already-dismissed rows remaining idempotent.

This phase changes matching semantics only; it should not add a new wire type or move host-side filesystem/process work
into Rust.

## Phase 2: Named-agent kill cleanup and end-to-end regressions

After the root-aware backend contract is available, make successful `kill_named_agent` cleanup dismiss all notification
shapes associated with its resolved `(cl_name, root artifact timestamp)` key through the existing Rust-backed bulk
notification API. Apply the cleanup consistently to both the live-process path and the stale/not-running path, so CLI
and mobile kills have the same notification lifecycle as ACE's `x` cleanup.

Keep this side effect best-effort, like the dismissed-agent index write: a notification I/O or decoding failure must not
turn an already-successful process kill into an error. Failed or permission-denied kill attempts must not dismiss a
still-live agent's question. Reuse the bulk API rather than loading and rewriting notification rows in Python, and
retain precise agent identity matching so unrelated questions are untouched.

Add shell-level regression coverage with an isolated notification store that proves:

- killing a named root agent dismisses an open `UserQuestion` whose concrete asking timestamp belongs to a child and
  whose root timestamp belongs to the killed agent;
- unrelated questions remain active;
- stale/not-running cleanup also dismisses the matching question;
- unsuccessful kills leave notifications active;
- notification-store failures do not change a successful `KillResult`;
- repeated cleanup is idempotent.

Add or extend a focused ACE cleanup regression to show that root-only cleanup also dismisses the question through the
same backend semantics, while retaining the existing rule that merely reading/dismissing completed agent rows does not
silently clear live plan/question requests. No cancellation response should be written to the question response
directory: termination dismisses the UI request because the agent is gone; it does not answer the question.

## Validation

Validate the smallest relevant layers first, then the complete repositories:

1. In `sase-core`, run the targeted notification-store tests, then `cargo fmt --all -- --check`,
   `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`.
2. In the SASE shell repository, run `just install` so the editable environment is rebuilt against the changed linked
   Rust core.
3. Run the targeted notification-store, named-agent-kill, and ACE cleanup test modules, including the new root/child
   question cases.
4. Run the required full `just check` gate. Re-run the targeted question-kill regressions after the full gate to guard
   against backend-selection or test isolation differences.

## Boundaries and risks

- Keep notification identity rules centralized in `sase-core`; do not add a divergent Python matcher for the kill path.
- Do not edit generated skill installations or their source template: the question producer already records both
  required timestamps.
- Do not dismiss on signal failure, permission failure, or an unresolved agent name.
- Preserve backward compatibility for older notifications that omit a root timestamp and for non-question notification
  action shapes.
- Keep all notification cleanup atomic through the existing Rust-backed store update so concurrent appends are not lost.
