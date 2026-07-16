---
tier: epic
title: Unified command-backed notification gates
goal: 'Plan, epic-plan, question, and launch approvals use one durable, trusted request-envelope
  constructor behind sase notify create, with command-backed responses, shared waiting,
  typed transport projections, and no custom lifecycle-role subsystem.

  '
phases:
- id: family_cleanup
  title: Remove custom lifecycle roles and plan member selection
  depends_on: []
- id: typed_core
  title: Add typed EpicApproval core projection and gate fixes
  depends_on:
  - family_cleanup
- id: gate_foundation
  title: Build the durable gate service and notify create API
  depends_on:
  - typed_core
- id: launch_gate
  title: Migrate launch approval and close its response loop
  depends_on:
  - gate_foundation
- id: question_gate
  title: Generalize questions into command input gates
  depends_on:
  - launch_gate
- id: plan_gates
  title: Migrate tale and epic plan approvals
  depends_on:
  - question_gate
- id: rollout
  title: Complete compatibility, documentation, and end-to-end verification
  depends_on:
  - plan_gates
create_time: 2026-07-16 15:05:00
status: wip
---

# Plan: Unified command-backed notification gates

## Goal and product boundary

Replace the three separately assembled plan, question, and launch request flows with one interaction-gate service. The
service is the in-process source of truth, and `sase notify create` is its low-level public CLI. Existing typed front
doors (`sase plan propose`, `sase questions`, and agent-initiated launch requests) remain the recommended workflows and
call the service directly rather than spawning the CLI.

The common layer owns request IDs, the neutral bundle layout, the versioned envelope, reviewed-content hashes,
notification projection, pending-action registration, compensation after partial failures, structured creation results,
response polling, cancellation, and terminal response persistence. Registered kind adapters own validation, previews,
command construction, input/result schemas, auto-policy interpretation, continuation behavior, and privileged side
effects. HITL must fit this interface but is not migrated in this epic. Navigation, memory review, mentor review, and
error notifications remain outside it.

Keep `NotificationWire` structurally unchanged and retain typed action names as transport tags. `PlanApproval`,
`EpicApproval`, `UserQuestion`, and `LaunchApproval` project the common envelope into ACE, mobile, and Telegram; do not
introduce a generic wire action. Rich definitions remain in request files, while `action_data` carries only string
identifiers and owned paths.

## Request and execution contract

New requests live under the SASE-owned neutral layout `interaction_requests/<kind>/<request-id>/`. Each bundle contains
a canonical `request.json`, an eventual write-once `response.json`, previews/attachments, and any copied or generated
action scripts. The versioned request envelope contains:

- identity, kind, timestamps, producer/agent context, continuation mode, and an optional gate timeout distinct from
  transport staleness;
- typed payload and presentation metadata supplied by the registered adapter;
- terminal choices with stable IDs, labels, an argv command descriptor, an input schema, and a result schema;
- non-terminal operations, initially only `edit_file`, referencing an adapter-approved file in the bundle or
  notification attachments;
- a common `auto` field carrying an enabled state and an optional opaque argument interpreted and validated by the kind
  adapter; and
- SHA-256 hashes for the canonical request, reviewed payloads/previews, command scripts, and editable targets at the
  revision shown to the user.

Commands are argv arrays and are always launched with `shell=False`; raw shell text is not part of the schema. The
shared host executor validates the owned path, adapter registration, choice/input schema, current hashes, symlink and
path-containment rules, and write-once response state before execution. It passes normalized choice and form input as
JSON on stdin, validates JSON on stdout, atomically persists the response, marks the pending action handled, and then
applies adapter-declared host side effects. A failed command leaves the gate answerable and records an auditable error
instead of manufacturing a successful response.

The `edit_file` operation is deliberately not a command and does not resolve the gate. A local surface suspends into
`$EDITOR`, then the adapter revalidates the file, advances the reviewed revision and hashes, regenerates affected
previews, and forces the modal to reload before a terminal choice can run. Remote transports omit operations they cannot
safely implement.

Typed transport choices remain a closed subset during this initiative. Each mobile or Telegram choice ID resolves to a
command present in the envelope; the host rejects a stale hard-coded choice that is absent. ACE may render the full
adapter-approved choice and input model. This supplies data-driven command semantics without changing the
notification-row protocol.

## CLI and durability contract

Enhance `sase notify create` with a gate mode that accepts the versioned request specification from stdin, validates it
through the kind registry, and returns a stable JSON descriptor containing schema version, notification ID, request ID
and kind, bundle/request/response/preview paths, continuation mode, auto-resolution state, and hashes. Preserve raw
creation for ordinary non-actionable rows, including `silent`, but reject attempts to mint a registered privileged
action without the gate path. Keep every public option alphabetized, documented, and paired with a short alias.

For a manual gate, success means the complete bundle, notification row, and pending-action entry are durable. Use atomic
file replacement and fsync where the stores support it, plus a small creation-state journal so retries can repair or
compensate interrupted work. On an ordinary failure, unregister any pending entry, remove an unpublished bundle, and
dismiss or explicitly mark an already-appended notification as failed; never report success for a partial gate. Creation
and response operations are idempotent by request ID.

The pending-action 24-hour threshold remains transport-advisory: it may hide remote buttons but cannot terminate a
producer. The generic poller observes `response.json`, cancellation, and an explicit per-request timeout only. Kind
adapters translate neutral responses into legacy continuation data until the old runner paths are removed.

Automatic resolution uses the same adapter choice, command executor, validation, hashes, and response contract as a
human choice. An auto-resolved gate persists its bundle and response for audit but does not publish a live pending
action. `%auto` parsing must retain an optional raw argument instead of validating against a global plan-only enum; the
adapter that eventually opens a gate supplies the default and valid arguments. A bare `%auto` retains normal plan
auto-approval for tale plans, selects epic approval for a `tier: epic` plan, and retains first-option behavior for
questions. Launch approval continues to reject automatic resolution.

## Phase 1: Remove custom lifecycle roles and plan member selection

Delete all three lifecycle-role layers before reshaping plan requests:

- Remove the inactive `improve_plan`/`tester` definitions and their prompt bodies, generic `kind: agent_family`
  discovery/models/validation, evaluator branches, custom-role execution, visit counts, snapshots, and transition state.
- Remove `role_completed`, custom-role artifact metadata and labels, plan `member_options`/`default_member_ids`,
  response `selected_member_ids`, the plan-approval member UI, `--with`/`--without`, configuration/schema keys, and
  every historical reader. Do not retain dormant compatibility seams.
- Keep a small catalog-boundary tombstone that rejects `kind: agent_family` with a hard migration error; this is
  replacement guidance, not a runtime or historical reader.
- Rewrite agent-family documentation and examples around supported manual `%name(parent, suffix)` attachment, arbitrary
  suffix names, and agent-initiated family launches through `LaunchApproval`. Preserve all ordinary family naming,
  grouping, attachment, and launch behavior.
- Update focused unit/golden tests and intentionally regenerate the ACE PNG snapshots whose lifecycle labels or member
  controls disappear.

This phase must leave plan approval behavior unchanged except for removal of custom member selection.

## Phase 2: Add typed `EpicApproval` core projection and gate fixes

In `sase-core`, add `EpicApproval` as a first-class mobile/pending action kind without adding a `NotificationWire` field
or a generic action. Reuse or split the plan choice wire only as needed to give epic approvals a typed detail and
request path. Include the new action in priority/actionability, missing-target state, pending-action conversion, labels,
prefix resolution, and agent dismissal. Add `LaunchApproval` to agent dismissal at the same time.

Because the new action must be added to every existing string mapping, centralize the gate-action conversion used by the
touched Rust notification modules so `EpicApproval` cannot be recognized by one path and silently lost by another. Do
not expand this cleanup to `memory_review` or unrelated action mappings. Update Rust exports, serde/parity tests, mobile
fixtures, and Python binding validation. Let release-plz own all crate versions.

## Phase 3: Build the durable gate service and `notify create` API

Add a focused notification-gates package in SASE with typed envelope/result models, the adapter registry, neutral path
resolver, bundle/journal writer, hash verifier, strict pending registration, command/input executor, generic poller, and
legacy bundle resolver. Keep filesystem and request-envelope work Python-first as selected, while continuing to use the
Rust-backed notification store and typed core projection.

Implement gate-mode `sase notify create` on this service and preserve the ordinary raw-row mode with privileged-action
rejection. Refactor `append_notification` or add a strict gate-specific store operation so the service can detect
pending-registration failure instead of swallowing it. Fold the repeated pending-action and question-summary filename
tests into the registry/resolver. Register adapter shapes for plan, epic plan, question, launch, and future HITL, but do
not migrate producers yet.

Test schema/version rejection, stable JSON output, idempotent retries, concurrent responses, write-once behavior,
crash/compensation points, transport-only staleness, explicit timeouts, path traversal, symlink swaps, hash mismatch,
command injection, malformed command output, and attempts to create raw privileged rows.

## Phase 4: Migrate launch approval and close its response loop

Move launch request, preview, notification, and response creation into the neutral launch adapter while retaining launch
preflight, slot checks, cwd revalidation, and host-owned dispatch. Express approve, reject, and feedback as hashed
commands in the bundle; approval still performs dispatch on the approver host and returns dispatch status/count in the
neutral response.

Change the agent-side launch command to wait mechanically with the common poller and return a deterministic
approved/rejected/feedback result. Remove the generated-skill instruction that asks an LLM to poll
`launch_response.json`. A non-agent `sase run` remains exempt and launches directly. Make ACE, mobile, and Telegram
resolve the neutral envelope and run the same executor, with legacy `launch_requests/.../launch_request.json` fallback.
Preserve action-specific rendering and callback shapes during the compatibility window.

Cover requester cancellation, rejection, feedback, dispatch failure, requester death after approval, duplicate
callbacks, and the distinction between command completion and host dispatch completion.

## Phase 5: Generalize questions into command input gates

Represent questions as an adapter-defined input form rather than a bespoke response writer. The question adapter must
generate a bundle-local, hashed script for the current SASE continuation: after every question is complete, the executor
supplies normalized answers/global note as JSON stdin, runs the script, validates its JSON result, and writes the
neutral response. The same input model should support future `UserQuestion` notifications that inject validated user
input into another adapter-approved command.

Retain the marker/SIGTERM handoff, slot reacquisition, Q&A history, prompt reconstruction, and first-option auto
behavior in the typed question adapter. Replace the duplicated ACE/mobile/Telegram response writers with the shared
executor while allowing Telegram to persist partial multi-question progress until submission. New consumers read neutral
requests first and legacy `user_question/.../question_request.json` requests second.

Fix the ACE question handled-state bug by marking every successful local question execution handled through the common
completion path. Verify single/multi-select answers, custom text, global notes, incomplete forms, cross-surface
duplicate answers, slot reacquisition, agent cancellation, and legacy in-flight questions.

## Phase 6: Migrate tale and epic plan approvals

Route `sase plan propose` by required frontmatter tier: `tier: tale` opens a `PlanApproval` gate and `tier: epic` opens
an `EpicApproval` gate. Split the typed adapter policies and presentation while sharing plan validation, reviewed-file
hashing, feedback, rejection, persistence, and runner continuation. Every terminal plan/epic choice must point to a
command in the envelope; plan editing uses the non-terminal `edit_file` operation and must refresh validation, hashes,
and previews before approval.

Move `%auto` from the current closed `plan|tale|epic` parser contract to the common raw argument field. Preserve useful
legacy spellings as adapter-level aliases for the compatibility release, but reject a tier-changing auto argument that
conflicts with the file's required authored tier. Bare `%auto` keeps the tale adapter's normal auto-approve behavior;
for an epic-authored plan it selects the EpicApproval command that commits and launches the epic as an epic. Both paths
must use the same command executor as manual approval.

Migrate ACE, CLI approval, mobile, and Telegram through the neutral resolver and executor. Add the `EpicApproval` badge,
modal/detail, priority, dismissal, pending-state, remote keyboard/callback, and host epic-launch behavior. Keep legacy
`plan_approval/.../plan_request.json` requests answerable and ensure an old `PlanApproval` carrying an epic choice still
completes during the grace period; all new epic writers use `EpicApproval`.

Verify validation failures after edit, tale and epic auto modes, feedback and rejection, runner-owned versus host-owned
epic launch, response races, reviewed-file TOCTOU rejection, legacy plan choices, and typed remote choice projection.

## Phase 7: Complete rollout and verification

Coordinate deployment consumers-first: core/bindings and readers, then new writers. Keep neutral-first legacy readers
for at least one full release and beyond the 24-hour pending-action stale window; encode the removal condition in
tests/docs rather than silently deleting fallback in the same release. Do not dual-write old and new request trees.

Update source skill templates for plan, questions, launch, and notification usage, regenerate managed skills with
`sase skill init --force`, and update CLI help, ACE help where behavior changed, notification docs, action lists, and
architecture/migration documentation. Remove stale “via hook” language while accurately documenting marker/SIGTERM
handoffs. Update summaries to use the gate resolver and expose `silent` in raw notification creation. Leave HITL and
non-gate actions behaviorally unchanged, with a follow-up note identifying the adapter work needed for a later HITL
migration.

Run each repository's complete checks after its phase changes. The final verification must include `just install`
followed by `just check` in SASE, the dedicated visual snapshot suite with intentional golden updates, the `sase-core`
workspace tests/parity fixtures, and `just check` in `sase-telegram`. Add end-to-end fixtures that create and answer
each new gate from ACE/mobile/Telegram projections, exercise automatic resolution and legacy fallback, inject failures
at every durability boundary, and confirm that TUI command execution stays in tracked background work rather than the
Textual event loop.

## Acceptance criteria

- All new plan, epic-plan, question, and agent launch requests are created by the shared service in the neutral layout;
  typed front doors contain no private notification/bundle construction.
- Gate-mode `sase notify create` returns its documented stable descriptor and cannot create a successful partial or
  untrusted privileged gate.
- Every terminal choice resolves to a hash-verified argv command, question inputs are delivered to the generated script
  only after the form is complete, and plan editing is the sole special non-terminal operation.
- Manual and automatic decisions share validation, execution, response, and audit behavior. Bare `%auto` approves an
  epic-tier plan as an epic.
- Launch requesters mechanically observe approval, rejection, feedback, and dispatch failure; non-agent direct launches
  remain unchanged.
- ACE, mobile, and Telegram retain typed actions and can answer both neutral bundles and in-flight legacy bundles during
  the compatibility window.
- The custom lifecycle-role subsystem, member-selection protocol, stale metadata/readers, examples, configuration, and
  UI are gone, while manual families, arbitrary suffixes, and family launch approvals still work.
- `NotificationWire` has no new field and no generic interaction action is introduced; `EpicApproval` is fully
  classified everywhere a typed gate is classified.
- Transport staleness never terminates a waiting producer, all gate-critical bugs selected for this initiative are
  covered by regression tests, and all three repositories pass their complete checks.
