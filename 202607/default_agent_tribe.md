---
tier: tale
title: Reserved default tribe for Agents-tab grouping
goal: 'Agents whose effective tribe cannot be resolved appear consistently in one
  reserved @default tribe panel, with no user-facing no-tribe bucket and no regression
  to panel-scoped actions or persisted agent metadata.

  '
create_time: 2026-07-19 18:32:03
status: wip
prompt: 202607/prompts/default_agent_tribe.md
---

# Plan: Reserved `@default` tribe for Agents-tab grouping

## Context and behavioral contract

The Agents tab currently uses `None` as the internal panel key for agents whose presentation root has no effective tribe
and exposes that implementation state as `(no tribe)`. Replace the user-facing bucket with the reserved `@default` tribe
while keeping a clear distinction between an agent's stored assignment and its effective panel membership:

- Resolve the panel tribe from the outer presentation root exactly as today, so workflow, family, and clan subtrees
  remain atomic and inherit an available root tribe.
- When that resolution yields no tribe, use the reserved `default` identity for display and merged-panel annotations.
  Treat an explicit stored `default` value as the same bucket so implicit and explicit defaults can never produce
  duplicate `@default` panels.
- Do not backfill `agent_meta.json`, rewrite prompts, or add synthetic entries to `agent_tribes.json`. Clearing a
  user-managed tribe therefore returns the agent to `@default`, while CLI and persistence APIs can continue to
  distinguish an absent assignment from an explicit user-managed tribe.
- Retain the existing internal sentinel where it is part of stable split/merged panel state or backend wire behavior.
  The reserved default panel keeps the former fallback panel's cleanup scoping, fold persistence, ordering, and
  non-targetable wait/fork behavior; only its effective identity and visible vocabulary change. This avoids making
  `@default` a partially supported next-entity dependency whose artifact metadata does not actually enroll it.

## Canonicalize effective panel membership and labels

Introduce one canonical default-tribe constant and shared panel normalization / label helpers in the Agents-tab model
layer. Route panel-key discovery, per-agent effective tribe projection, merged-panel tribe annotations, panel titles,
whole-panel summaries, neighbor rows, cleanup/kill descriptions, and detail/modal headings through those helpers instead
of open-coding `None` as `(no tribe)`.

Keep the hot path purely in-memory and preserve the existing panel index and selective-refresh paths: the change must
not add filesystem access, subprocess work, or a second panel-membership computation during render. Preserve the current
deterministic ordering and collapse partition, with the reserved default bucket occupying the fallback panel's existing
canonical position.

Update cached runtime-statistics label matching to accept both the new `@default` spelling and historical `(no tribe)` /
`(untagged)` spellings. This keeps old cached snapshots readable while all newly rendered text uses `@default`.

## Preserve reserved-panel interaction semantics

Replace checks and messages that infer fallback-panel behavior directly from its old label with an explicit
reserved-default predicate. Whole-panel wait and fork actions, command availability, and keybinding footers should
continue to exclude this synthesized bucket, but warnings and help text should name the reserved `@default` panel rather
than claiming the agents have no tribe.

Verify that panel-scoped kill, dismiss, cleanup, isolation, navigation, marking, selection restoration, and fold
persistence still select exactly the agents whose stored/effective tribe previously mapped to the fallback bucket. Avoid
a fold-state schema migration unless the internal serialized sentinel actually changes; existing user fold state should
survive the visible rename.

## Documentation and regression coverage

Update the ACE and agent-family documentation to describe `@default` as the reserved fallback panel, explain that it is
derived rather than persisted, and state the retained wait/fork restriction. Remove current user-facing references to
`(no tribe)` without rewriting unrelated uses of “untagged” in other domains.

Extend focused model/action tests to cover:

- unassigned standalone agents, inherited workflow children, indeterminate multi-tribe containers, and explicitly
  `default` agents converging on one effective panel;
- empty and STARTING-only lists retaining their existing visibility/focus behavior;
- split-panel titles, collapsed and selected titles, whole-panel summaries, merged-panel row annotations, neighbor
  labels, and cleanup/kill copy using `@default` consistently;
- the reserved panel remaining excluded from wait/fork target affordances while ordinary named tribes remain valid
  targets;
- legacy runtime-statistics labels and persisted fold state continuing to load; and
- no extra full-list rebuild or panel-index invalidation beyond the membership change already required by an agent tribe
  update.

Update the affected ACE PNG goldens only after inspecting the generated diffs and confirming that changes are limited to
the intended `@default` text/style. Run `just install` first as required for an ephemeral workspace, then the focused
model/action tests, `just test-visual`, and the mandatory `just check`.

## Acceptance criteria

- Every rendered agent has an effective Agents-tab tribe; unresolved agents are shown in one `@default` panel and never
  in `(no tribe)` or a duplicate default panel.
- Stored tribe metadata remains unchanged unless the user explicitly edits it.
- Existing panel-scoped actions, fold state, navigation, and performance paths behave as before for the fallback
  population.
- Documentation, textual assertions, and reviewed visual snapshots consistently describe the reserved `@default`
  behavior, and the full repository check passes.
