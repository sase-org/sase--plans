---
tier: tale
size: medium
title: Make approved-plan archive handoffs workspace-independent
goal:
  Prevent approved tale agents from failing when the host archives their plan in an
  operational workspace different from the runner workspace.
proposed_by: bbugyi200.athena.0aw
create_time: 2026-08-22 13:53:01
status: wip
---

# Plan: Make approved-plan archive handoffs workspace-independent

## Outcome

An approved and committed tale continues successfully regardless of which operational
workspace the host uses to publish its plan. The approval protocol identifies the
archive with the canonical `plan:YYYYMM/name.md` reference instead of asking the runner
to consume another checkout's absolute path. The runner still rejects incomplete or
malformed current-protocol responses, never writes or commits a duplicate plan, and
hands the coder a stable plan reference.

## Root cause and constraints

- The observed planner completed its plan, validation, and approval successfully. The
  approval host also archived and committed the plan successfully.
- Approval-time publication deliberately runs under `operational_workspace_lease(...)`.
  Its returned filesystem path therefore belongs to the leased checkout, which can
  differ from the agent runner's checkout and can be released or reused immediately
  after publication.
- The newly added `host_v1` consumer in `src/sase/axe/run_agent_exec_plan_accept.py`
  treats `saved_plan_path` as though it must be below the runner's own
  `SddStore.kind_root("plans")`. A valid host archive from a different checkout
  consequently raises `_PlanArchiveProtocolError` before the coder successor can start.
- The absolute leased path is not a durable cross-process identity even if that strict
  containment check is removed. SASE already has a workspace-independent identity for
  this exact artifact: a canonical `plan:` reference backed by the shared Rust plan-ref
  parser/canonicalizer.
- Preserve legacy and already-in-flight response handling. Do not weaken current
  protocol validation into accepting arbitrary external paths, and do not reintroduce a
  second runner-owned plan write or commit after the host has published the archive.

## Implementation

1. **Return a canonical archive identity from the host publication boundary.**
   - Extend the result of `archive_approved_plan(...)` in
     `src/sase/_plan_archive_approval.py` so the caller receives both the concrete path
     used inside the operational lease and a canonical `plan:` reference computed while
     the publishing `SddStore` and its plans root are still known.
   - Use the existing helpers in `sase.sdd.plan_refs` (and therefore the shared Rust
     grammar) rather than deriving a reference by string-splitting a workspace path.
   - Retain the concrete path for user-facing clipboard/reporting behavior, but document
     that it is host-local diagnostic data rather than the runner's plan location.

2. **Version and propagate the workspace-independent approval response.**
   - Update `prepare_plan_terminal_response(...)` and the neutral-gate translation in
     `src/sase/plan_approval_actions.py` and `src/sase/plan_gate.py` to publish the
     canonical reference alongside the archive owner/state before the terminal response
     becomes visible.
   - Extend `PlanApprovalResult` and response parsing in
     `src/sase/llm_provider/_plan_utils.py` to carry that reference and identify the new
     host protocol version explicitly.
   - Keep the old `saved_plan_path` field and `host_v1` parsing only as a compatibility
     lane for legacy/in-flight responses; new responses must not imply that a leased
     checkout path belongs to the runner.

3. **Consume committed tale archives by canonical reference.**
   - Refactor the current-protocol checks in
     `src/sase/axe/run_agent_exec_plan_accept.py` so the new version requires
     `plan_archive_owner == "host"`, `plan_archive_state == "archived"`, and a
     canonical, non-legacy `plan:` reference accepted by the shared parser.
   - Mark the tale committed from that verified host-owned archive result, skip
     `write_sdd_files(...)` and every runner-side SDD commit exactly as today, and
     record the stable reference in plan/follow-up metadata instead of the operational
     checkout's absolute path.
   - Build the coder handoff with `@plan:...`. Use the durable reviewed proposal path
     only where an API still requires a local file (for example size/frontmatter routing
     or `SASE_PLAN`), because the gate synchronizes edits back to that durable proposal
     before archiving it.
   - Preserve the existing strict same-store validation for legacy `host_v1` responses
     and the legacy no-host-archive fallback, so compatibility does not become an
     arbitrary-path escape hatch.

4. **Add cross-workspace protocol regression coverage.**
   - Update approval action, neutral gate, translation, and plan-chain golden tests to
     assert the new archive reference and protocol fields are written atomically with a
     committed tale response.
   - Add an end-to-end runner-unit regression whose host archive path is under one fake
     checkout and whose runner `SddStore` is under another. Assert that acceptance no
     longer raises, the plan is considered committed, no runner write/commit occurs,
     metadata carries the canonical reference, and a coder continuation uses
     `@plan:YYYYMM/name.md` rather than either checkout's absolute path.
   - Add negative cases for a missing reference, a legacy filesystem path presented as a
     canonical reference, a wrong artifact kind, and malformed/traversal-like input.
     Retain coverage showing that old `host_v1` paths outside the runner store are still
     rejected.

## Validation

- Run the focused approval/archive, neutral-gate, plan-chain golden, and accepted-plan
  follow-up test modules that exercise the changed protocol.
- Run `just install` before repository verification, then run `just check`.
- If scoped selection escalates or reports unusual coverage, follow repository guidance
  and run `just check-full` through `/sase_monitor`.

## Acceptance criteria

- A host-owned tale archive published from a different operational checkout cannot fail
  solely because its concrete path is outside the runner's plans root.
- New committed-tale approval responses contain a validated canonical `plan:` reference
  and are not published before archive commit/push succeeds.
- The coder successor and persisted relationship metadata contain no dependency on the
  released operational checkout.
- Host-owned archives still have exactly one writer/commit, and malformed or incomplete
  current-protocol responses fail before a coder is launched.
- Legacy response paths retain their existing compatibility and containment behavior.
