---
tier: tale
title: Repair Plan Decision acceptance, stamps, and new-note grants
goal:
  Every approval route preserves the reviewed decision vector, author order, caller,
  surface, and revision, and valid grants can name notes that have not been created.
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.10.1
bead: sase-1hi.10.1
create_time: 2026-10-08 05:36:31
status: wip
---

- **PARENT:**
  [202610/plan_decisions_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_landing_repairs.md)
- **BEAD:**
  [sase-1hi.10.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.10.1.md)

# Repair the gate phase of Plan Decisions

Implement the assigned phase **sase-1hi.10.1** of **sase-1hi.10**. This is one bounded
implementation task; keep its existing reservation and assignment. The authoritative
designs are `plan:202610/plan_decisions_landing_repairs.md`, Section 1 (`gate`), and
`plan:202610/plan_decisions.md`, Sections 1.6, 2–4, and 6.3. Read them through
`sase artifact read`, and read the phase with
`sase bead read sase-1hi.10.1 -r "Need the assigned gate scope and design"`.

The result must satisfy reliability contracts 3–7, 9, and 10: omissions take the
effective defaults before receipt hashing, one vector is accepted, stale reviews cannot
act, definitions stay frozen, retries keep accepted answers, agent memory requests
cannot exceed their authorization, and validation failures fail closed.

## Scope and ownership

The implementation belongs in the sase checkout. Reuse the existing Rust bindings
through `sase.sdd.plan_decisions`; grammar, resolution, digests, and sheet behavior
remain Rust owned. The existing edit-freeze adapter demonstrates rebuilding a definition
vector with `validated_to_wire_dict` and `payload_binding` using the review's frozen
host facts. No new core API is expected. If an actual missing domain operation prevents
the required behavior, open `sase-core` with `sase repo open`, read its `AGENTS.md`,
implement and test the domain operation there, and advance `sase-core-revision.txt` past
the host-finalized core commit before relying on it.

Read `lint_and_test.md`, `symvision.md`, and `sase_flags.md` with `sase memory read`
before implementation. Source forwarding in ACE is part of this phase; read `tui.md`
before touching that glue. This task repairs the existing feature and does not revive
its retired beta flag.

The sibling `handoff` phase owns frozen accepted-sheet loading, bead DECISIONS display,
epic inheritance, commit-guard paths, and provenance-chain repairs. The `cli`, `tui`,
and `telegram` phases own their rendering changes, including the visible `new` chip.
Keep changes here limited to gate input plumbing, acceptance, stamping/recovery,
validation, direct-route adaptation, grant records, and their tests. Coordinate shared
`sdd/plan_decisions.py` and adapter edits by preserving those phases' public entry
points and avoiding unrelated refactors.

## Confirmed defects and implementation

### 1. Preserve author order and enforce complete, immutable stamps

`plan_gate_stamp.py` currently builds the stamped map by iterating Rust's resolved
values. Those are a sorted map, and `_plan_gate_command.py` also serializes sorted keys.
Consequently an authored `zeta, alpha` becomes `alpha, zeta` in the archive.

In both `stamp_durable_plan` and `stamp_direct_file`, iterate the authored `decisions:`
map and look up each accepted answer by id. Preserve decision order, choice order, and
the authored fields. A shared stamping helper is appropriate if it removes duplicated
orchestration without moving domain rules into Python. Require the accepted vector to
cover the authored ids and refuse unexpected ids or conflicting existing
answers/decider/surface with the existing actionable stamp error. Never silently drop
accepted values.

Treat an identical, complete stamp as a no-op. If the durable file has compatible
coordinates but is missing one or more answers, finish the stamp rather than returning
because newly computed entries already match. Write the durable plan before archive
commit or epic launch; keep the gate's plan resource pristine. Direct-file stamping must
enforce the same immutability as live-gate stamping.

### 2. Carry the true surface and freeze recovery attribution

`execute_plan_approval_response` and `_plan_approval_response.py` currently submit
`source="plan_response"` for both ACE and CLI. Thread a real source through this shared
API: ACE passes `tui`, `main/plan_approve_handler.py` passes `cli`, and the existing
Telegram/mobile/automatic callers retain their corresponding surface. Update the
gate-turn follow-up stage's source consistently.

Normalize/validate the stamp coordinates at the plan normalization boundary, before
`accept_gate_decision` can create a receipt or dismiss a notification. Reject an
unsupported source or invalid caller at that boundary. Supported surfaces are `tui`,
`cli`, `telegram`, and `mobile`; `auto_resolution` maps to `decided_by: auto` with no
`decided_via`. A classified human maps to `reviewer`; an agent maps to `agent`. If the
legacy `plan_response` alias must remain for existing callers or stored responses,
retain its established TUI meaning only where that meaning is known; it must no longer
substitute for a live CLI source. Decision-free gates and non-plan gate kinds retain
their existing contracts.

The neutral `response.json` already records `source` and `caller`, but the translated
runner response does not provide the `_gate_source`/`_gate_caller` fields that
`plan_approval_actions.py` tries to read. Persist the validated attribution with the
accepted vector in the translated response, or persist the mapped
`decided_by`/`decided_via`. Preserve it on every translation and retry; remove fallbacks
that classify a recovery worker as the original reviewer. For pre-response failures,
retain the original coordinates in durable host acceptance/journal metadata tied to the
receipt's acceptance id, so that replay can recover them before constructing a
replacement response.

### 3. Recover from durable answers, without resolving again

Trace recovery through `notification_gates/executor.py`, the plan adapter's
`prepare_terminal_response` and `apply_side_effects`, `cli_answer.py`'s
`_resume_answered_shell`, and runner/legacy plan-response processing. Currently a
published response takes a shortcut past terminal preparation, and normalization can
still run before that shortcut.

For an answered plan gate, extract the accepted answers and original attribution from
`response.json` and its translated response. If the durable plan is missing answers,
stamp it before resuming archive, launch, or follow-up work. Reuse the same recovery
operation wherever a completed response bypasses ordinary terminal preparation. Do not
re-run the resolver, quote verification, or caller discovery to decide those answers,
and do not re-ask the reviewer. Changed submitted overrides cannot replace an accepted
vector.

Keep response replay and side-effect retries idempotent: identical recovery adds no
second archive commit or coder/epic launch. Reject any conflicting stamp. When failure
precedes publication of `response.json`, preserve the existing receipt/journal replay
contract and stamp from the original accepted execution results on terminal-preparation
retry. A retry in a different process must retain the first acceptance's attribution
rather than adopt the retry process's caller.

### 4. Bind attached and detached answers to the displayed revision

In `notification_gates/cli_answer.py`, a malformed `review_revision` currently becomes
absent, and `_submit_detached_answer` drops both revision and source.

Validate a present revision as an integer using the established operation-request wire
convention. Reject non-integral numbers, booleans, malformed strings, and other
incompatible values instead of silently disabling the check. Preserve a missing revision
for older-client compatibility. Forward the revision and source into the detached
operation payload and through any answered-shell resume route that invokes the executor.
The child `--no-detach` command must receive exactly the review coordinate the
submitting client supplied.

A stale pending review must raise `stale_review` before receipt, response, command,
notification dismissal, archive, launch, or gate-turn settlement. Test attached
execution and the detached request followed by its child execution; checking only the
detached payload is insufficient. Keep stale-review rendering in the sibling surface
phases.

### 5. Make kind validation check independently rebuilt definitions

`kind_validation/plan.py::_validate_plan_decisions` compares a digest with a second
digest of the same input. Replace that tautology with a real core round-trip, following
the existing edit-freeze reconstruction pattern.

Read the adapter-owned plan resource, validate it through the core-backed plan
validator, and rebuild definitions from the validated authored questions and the
supplied frozen host facts (`requested_verified`, `provenance`, and resolved records).
Do not gather fresh environment-dependent facts during kind validation. Compare the
rebuilt digest with the frozen payload digest, resolve effective defaults through core,
and build/validate the corresponding sheet. Any malformed definition, inconsistent
default or kind, duplicate id, invalid provenance, unreadable required resource, or
binding exception must be `invalid_plan_decisions`, not an accepted gate.

Keep the existing sealed query/options/groups/commands/operations checks. Validate each
option against its compiled decision input schema and each acceptance result against the
full resolved result schema, including required fields and rejection of extra decision
keys. Explicitly require `reject` to declare no `decision_*` input property or decision
result. Feedback can carry provisional values but cannot become an acceptance result.
Apply the checks to both tale and epic gates.

### 6. Fail closed and use one direct approval resolver

`main/plan_decide.py::caller_for_decide` currently returns `human` when its
classification fails. Delegate to the same fail-closed classifier used by
`notification_gates.executor.gate_response_caller`, including its unknown-actor and
exception handling. Apply that classifier to hand-run `sase bead work` as well. A human
shell stamps `reviewer` via `cli`; an agent shell stamps `agent` via `cli`. Trusted
automatic gate approval retains its separate auto coordinate.

In `main/plan_validate_handler.py` and `main/plan_propose_handler.py`, convert
unexpected host-check failures into an error `PlanDiagnostic` with a stable code, field,
and useful cause. A validator exception must produce a nonzero result; propose must stop
before formatting, archiving, queue mutation, or handoff. Keep the closest-sentence
diagnostic for ordinary unverified quotes. The sibling CLI phase owns moving the
decision sheet inside the JSON envelope; preserve that work while adding failure
diagnostics.

Use `sdd/plan_decisions.py::resolve_plan_decisions_for_direct_approval` as the one
direct resolution implementation for gateless `sase plan approve <file>`, its `-D`
overrides, and gateless `sase bead work`. `main/plan_decide.py` may remain a thin
parser/card/error adapter: keep duplicate-id detection, boolean spellings,
case-insensitive complete choice keys, allowed-value hints, and dry-run output. Build
host facts once and hand typed overrides to the shared resolver. Retain the rows/sheet
needed by the CLI without a second resolution.

Remove the broad error swallowing in `_stamp_bead_work_decisions`. Propagate resolution
errors and stamp failures into the CLI's normal error reporting before plan-store
mutation or agent launch. Dry run must resolve and validate its vector but never stamp,
archive, or launch. Already stamped plans reuse their accepted answers; partial or
conflicting stamps must not silently bypass validation.

### 7. Resolve valid future memory targets as grant records

`sdd/plan_decisions.py::_resolve_memory_records` uses the read selector resolver, which
rejects nonexistent files, and emits `exists: True` unconditionally. Add a grant-address
resolution path that distinguishes target identity from content reading. Reuse existing
selector classification, memory-layout/root resolution, and web lookup rules; keep
ordinary `sase memory read` missing-target errors.

Existing selectors retain project-over-home lookup and freeze the current web strand
list. A valid missing flat note resolves to its future canonical project memory address,
`kind: note`, `type: reference`, and `exists: false`. A valid missing strand resolves
only within a known existing web, inherits that web's resolved scope, names its future
strand path, uses `kind: strand` and `type: strand`, and has `exists: false`. Do not
create a file or invent a web, scope, alias, or type while constructing a grant.

Keep errors for malformed selectors, unknown webs/scopes, ambiguous existing aliases,
invalid nested flat-note names, traversal, broken/escaping symlinks, layout collisions,
and unreadable or malformed existing targets. Missing-target handling must not turn
those errors into new-note grants. Keep grant identity keys consistent across equivalent
selectors so overlap detection still refuses two decisions granting the same existing or
future target.

Validate the frozen records through the existing payload/sheet bindings and check that
`exists: false` survives to the surface-facing sheet. Renderer changes for the visible
`new` chip stay with their assigned sibling phases. Use temporary memory fixtures for
tests; this task does not create or edit durable memory notes.

## Verification and acceptance evidence

Use real plans and `build_plan_approval_gate_spec` for gate fixtures. Add focused
behavioral tests in the existing decision, gate answer/detach, plan approval, direct
approval, bead-work, and plan-handler suites, splitting oversized test files where
useful. Do not substitute source-text assertions or binding skips for exercising the
required behavior.

The acceptance matrix is:

| Behavior                    | Required coverage                                                                                                                                                                                                                                                                                                            |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Stamp order and coordinates | Authored `zeta, alpha` with a choice and a toggle through tale approve+commit, approve-only, commit-only, epic approval, gateless direct-file approval, and hand-run bead work; assert parsed order, every answer, true surface, and caller.                                                                                 |
| Surfaces                    | Human CLI is `reviewer/cli`, ACE is `reviewer/tui`, Telegram/mobile keep their surfaces, agent CLI is `agent/cli`, and automatic approval is `auto` with absent surface. An unknown source fails before acceptance.                                                                                                          |
| Recovery                    | Remove compatible durable answers after a published response, resume, and recover the original vector/coordinates; make re-resolution fail if called. Verify no duplicate launch/commit, repeated recovery is a no-op, and conflicting stamps fail. Cover terminal-preparation replay and answered-shell follow-up recovery. |
| Revision                    | Matching, omitted, malformed, and stale revisions through direct executor, attached `sase gate answer`, and detached answer plus child execution. Failed pending submissions leave no receipt/response or side effects.                                                                                                      |
| Kind validation             | Tale and epic payload/resource drift, invalid shapes/defaults/kinds/provenance, duplicate ids, changed input/result schema, required decision inputs, and decision properties added to reject all fail. Valid frozen definitions round-trip without fresh fact discovery.                                                    |
| Identity and one vector     | Compare the actual receipt `input_identity` for omitted versus explicit effective defaults on each changed live route, including shared and per-option input and approve+commit. Include a clamped memory default and preserve `decision_conflict` refusal. Direct routes must produce the same canonical vector.            |
| Agent memory boundary       | Through actual `sase gate answer -O` inputs, including attached and detached child execution, refuse switching an unauthorized memory row on before receipt creation. Preserve permitted effective defaults and explicit off, and test the same classifier in live/direct plan approval and bead work.                       |
| Host failures               | Inject host-check exceptions into validate and propose; assert diagnostics, nonzero status, and no handoff/archive mutation. Inject actor-classifier failure and prove it cannot grant human authority.                                                                                                                      |
| Edit freeze                 | Replace the source grep in `tests/test_plan_decisions_gate.py` with actual edit/accept operations. Prose-only edits succeed and advance revision; changes to questions, labels, choices, defaults, why, selectors, or system answers fail with the freeze message and leave the accepted resource/revision unchanged.        |
| Future grants               | New flat note and new strand in an existing web have stable scope/path/type and `exists: false`; existing note/strand/web still resolve; duplicates/aliases overlap; malformed, traversal, ambiguous, broken-symlink, and unknown-web cases fail. The real sheet retains the `new` fact.                                     |

Run the focused suites while implementing, then format/fix through the repository
recipes and run **`sase tool run check`** from each modified repository. Do not run
`just check-full`. If dependencies are stale, use the documented install workflow; hand
long commands to `/sase_monitor`, wait for the handoff itself to exit, and put the
remaining verification and closure instructions in its successor prompt.

The parent audit lists pre-existing failures tracked by sase-1hr (macro terminology),
sase-1hy (hinted raw prompt), sase-1g3 (snippet CPU budget), and sase-1hp (Symvision
unused-public backlog). Do not repair those in this phase. Any unclassified failure
needs evidence: fix failures caused by this change, or prove an identical failure on the
clean base tree and record it as a phase follow-up. Do not leave the phase open solely
for a verified clean-base failure.

## Phase completion

Add no new unused-public Symvision entries. An initial planning read found no
`--epic-symbol` entries keyed to this phase; rerun
`sase bead epic-symbols sase-1hi.10.1` immediately before closure. Resolve any entries
that appeared, or re-key an intentionally future-consumed seam in the Justfile to its
still-open consuming phase or parent epic.

Record discovered work only with
`sase bead note sase-1hi.10.1 'PROPOSED FOLLOW-UP: <summary — detail and evidence>'`; do
not create beads. Include existing tracking ids for known failures. When the
implementation and required verification are complete, close **only** `sase-1hi.10.1`
with
`sase bead close sase-1hi.10.1 --note "<implemented behavior and checks verified; any clean-base failures and tracking ids>"`.
Do not change its status by hand or close sase-1hi.10, sase-1hi, or any ancestor plan
bead. Leave ancestor smoke testing, skill deployment, and landing to their land agents.
Finish through the root `/sase_final` workflow so the host owns commit and publication.
