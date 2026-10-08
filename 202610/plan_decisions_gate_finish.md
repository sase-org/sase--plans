---
tier: tale
title: Finish Plan Decisions gate reliability and route coverage
goal:
  Complete sase-1hi.10.7.1 with immutable accepted answers, durable failures, correct
  memory coverage, and quiet inbox receipts.
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.10.7.1
bead: sase-1hi.10.7.1
create_time: 2026-10-08 13:28:32
status: wip
---

- **PARENT:**
  [202610/plan_decisions_landing_finish.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_landing_finish.md)
- **BEAD:**
  [sase-1hi.10.7.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.10.7.1.md)

# Finish Plan Decisions gate reliability and route coverage

## Scope and execution contract

Implement the entire `gate` phase of `plan:202610/plan_decisions_landing_finish.md` and
complete the assigned bead `sase-1hi.10.7.1`. This is one bounded coding task: the
defects have identified causes, existing APIs, and concrete acceptance tests. The steps
below form an implementation sequence, not additional phase beads.

Start with `sase bead read sase-1hi.10.7.1 -r "Need the phase scope and design file"`.
Read the authoritative designs using `sase artifact read`: the landing-finish plan,
`plan:202610/plan_decisions_landing_repairs.md`, and `plan:202610/plan_decisions.md`,
especially reliability contracts 3–10 and Sections 6.3, 6.4.5, and 6.8. Honor the epic's
accepted DECISIONS. The original plan wins over research reports when they differ. The
two research references in its Section 0 returned `missing` during planning; its
explicit contracts and the repair plans provide the implementation requirements.

The inspected base is `f92bde8abef2bf0191273e86c30325e12d243fad`. Recheck source
locations against the implementing checkout. Follow repository instructions and read
`lint_and_test.md`, `symvision.md`, `tui.md`, and `tui_perf.md` through
`sase memory read`. Implementation targets code, tests, and notification docs; memory
examples are isolated test fixtures. The sibling CLI, Verdict/layout, golden, and
Telegram phases own their respective changes. Preserve the shared interfaces they
consume, particularly the `stale_review` error record.

Use the existing Rust bindings for plan validation, resolution, sheets, and consent.
Python changes here orchestrate those bindings and supply host I/O facts; they must not
introduce a Python implementation of core decision behavior. No core API change is
expected. If a required domain change emerges, use `sase repo open sase-core`, follow
its instructions, update the Rust wire/binding and tests, and advance
`sase-core-revision.txt` past the resulting host-owned commit before finishing. Every
changed repo needs `sase tool run check`.

## 1. Reuse accepted bead-work answers without live resolution

Change `_stamp_bead_work_decisions` and `_reuse_stamped_bead_work_answers` in
`src/sase/bead/cli_work_from_plan.py`.

- Detect any existing acceptance stamp before the fresh-resolution and dry-run branches.
  Both ordinary and dry-run bead work must reuse accepted answers.
- Validate stamp completeness and attribution using the existing Rust Archived
  validation path: every authored decision needs an answer and a valid acceptance
  attribution. Surface partial stamps, invalid answer types/choices, or incomplete
  metadata as a clear `PlanFileWorkError`. Never fill missing accepted answers from
  defaults. This check precedes the loader, which currently fills omissions.
- Read the accepted vector through `load_stamped_decisions`. Require an accepted result
  covering all authored ids. The reader must not call `build_definitions`, the decision
  resolver, quote verification, or current-caller classification to reinterpret an
  acceptance. Preserve its original answers and attribution.
- Remove the ineffective comparison against freshly resolved overrides. Remove
  reader-environment sibling backfill. A sibling can come only from frozen data
  belonging to the original approval; when none is reachable, keep it absent and use the
  loader's neutral accepted fallback.
- For fresh approvals and fresh dry runs, classify the caller with the existing
  fail-closed classifier. Resolve once and check `resolved["errors"]` before any
  stamp/archive/launch. Propagate resolver exceptions and structured refusal errors. A
  dry run remains read-only. A human shell stamps `reviewer`/`cli`, an agent shell
  stamps `agent`/`cli`; an agent cannot turn memory consent on.

Tests must prove an agent can run bead work on a reviewer-accepted memory-yes epic with
its original grant intact, including a changed reader cwd and absent quote source. Make
any fresh resolution of that accepted plan fail the test. Cover a partial stamp, an
invalid stamp, fresh resolver errors, dry-run errors and accepted reuse, and the absent
sibling remaining absent. Exercise the real bead-work route with archive/launch effects
isolated so the test launches no agents.

## 2. Keep one direct resolver and retain its frozen definitions

The two current paths are `main/plan_decide.resolve_direct_decisions` and
`sdd/plan_decisions.resolve_plan_decisions_for_direct_approval`.

Keep the latter as the canonical host resolution function for direct-file approval,
`-D`, and fresh bead work. The former may remain a thin CLI parser/card adapter: it owns
raw assignment parsing, bool spellings, case-insensitive choice keys, suggestions, and
CLI error presentation. It must delegate semantic resolution to the canonical function
and the Rust binding.

Allow the canonical function to consume already-built definitions so the CLI adapter can
type-check raw input against the same vector without rebuilding host facts. Return or
carry that vector with the resolved values, rows, and sheet. Update the wrapper's result
and `main/plan_direct_approval_resolve.py` together so
`DirectApprovalPlan.decide_definitions` comes from that exact resolution. Delete its
second `build_definitions` call and swallowed-exception fallback. Preserve existing `-D`
hints, refusal behavior, and public APIs used by other frontends. Privatize or remove
any helper that loses its production consumers.

In `main/plan_direct_approval_run.py`, pass those frozen definitions to the stamp
writer. The normal resolved approval path must never need its legacy live rebuild. Keep
any necessary compatibility path explicit and test that the normal route builds host
facts only once. Accepted retries use stored answers instead of this fresh resolver.
Test default and overridden fresh routes, memory refusal, and the same definitions
reaching the durable sibling.

## 3. Durably record pre-acceptance rejections exactly once

In `notification_gates/executor.py`, place the owned-bundle pre-acceptance checks inside
a recording boundary using `command_runner.record_execution_error` or
`recorded_rejection`. Include revision comparison, selection/normalization, and the
transport and adapter preflight checks before `accept_gate_decision`.

Record `stale_review`, `unknown_source`, `decision-resolve-failed`, and
`memory_decision_requires_human` with their existing code strings. The stale message
must name the current revision, and preferably the submitted revision. Keep the error
JSON schema and fields stable. Re-raise the original exception; the durable record
supplements the existing CLI/ACE/Telegram error response. Keep this scope separate from
the scopes inside acceptance and later execution, so one rejected submission writes one
record rather than two.

Tests build real gate bundles and prove each refusal leaves exactly one `errors/*.json`,
no `response.json`, no accepted receipt, no option command, and no plan stamp. For
staleness, cover `execute_gate_selection`, attached `sase gate answer`, and detached
submission followed by execution of its real operation payload through the answer
handler. Capturing the detached payload alone is insufficient. Verify its `source` and
`review_revision` survive and the stored stale record identifies the new revision.
Exercise the agent memory-yes refusal through the answer handler, not only the resolver
helper. Isolate subprocess supervision, gate-turn lookup, and user-home state in
fixtures.

## 4. Make retry re-stamp failures observable

Replace `except Exception: pass` around `recover_plan_stamp_from_response` in
`notification_gates/executor.py` and `notification_gates/adapter_plan.py`. Log
exceptions with context. Record them once via `record_execution_error` with
`stage="restamp"`, preserving a `GateError` or approval-error code when available, and
propagate where the route can report failure. Ensure retry does not report successful
completion or launch a successor after a failed stamp. Integrate with existing
post-response failure tracking instead of nesting scopes that duplicate the same restamp
record. An already accepted answer and `response.json` remain intact for recovery. The
CLI resume path already propagates failures; retain that behavior.

Test `recover_plan_stamp_from_response` itself with a real request and response:
successful recovery, identical recovery, conflicting values/attribution, and missing
answers. It must never invent answers or call live resolution. Distinguish an
intentional non-applicable recovery from a malformed approving stamp. Add route tests
forcing a recovery exception in both formerly swallowing paths; assert the log and
durable `restamp` record and the observable failed outcome.

## 5. Strip private gate coordinates on every plan result

In `plan_approval_actions._stamp_decisions_best_effort`, remove `_gate_source` and
`_gate_caller` before the early return for plans without decisions. Capture their values
first when stamping needs them. Keep the response's supported top-level `source` and
`caller` for retry attribution.

Update both assertions in `tests/test_plan_gates_action_api.py` that currently pop and
expect the leaked keys. Test decision-free and decision-bearing approval results,
including commit-only, and assert the private keys are absent from persisted option
results while retry attribution stays correct.

## 6. Classify missing-target host errors structurally

`sdd/plan_decisions._grant_record_for_missing_selector` currently uses
`non_grant_markers` substrings to decide whether an error authorizes a future target.
Replace that mechanism with an explicit reason carried from the existing host
selector/path failure at its origin. Relevant adapters are `memory/_read_log_models.py`,
`_read_log_paths.py`, `selector_models.py`, `selector_notes.py`, `selector_web.py`, and
`web/lookup.py`.

Represent actual absence distinctly from invalid syntax, unknown web/scope, ambiguous
lookup, traversal, symlink failure/escape, layout collision, unreadable/non-file target,
and an existing non-readable note kind. Preserve the existing exception types and human
messages while propagating the structured reason through wrapper errors. Unclassified
failures remain unresolvable. Host filesystem facts stay in the adapters; grammar and
decision semantics remain in the existing Rust bindings. New-target records are allowed
only for actual absence of a well-formed flat note or a strand in an existing resolved
web.

Preserve `decision-memory-selector-invalid`, `decision-memory-unresolvable`, and
`decision-memory-overlap` behavior. Tests must show that changing diagnostic wording
cannot change grantability; unknown or unsafe errors remain rejected. Cover a missing
flat note, a new `glossary:plan-decision` strand with `exists: false` and canonical path
`sase/memory/glossary/plan-decision.md`, unknown web/scope, ambiguous aliases/prefixes,
malformed selectors, traversal, broken/escaping symlinks, collisions, unreadable
targets, and overlapping grants. Use isolated fixture memory trees, never canonical
memory content.

## 7. Cover granted strands in the actual finalizer guard

In `finalizers/commit_memory_guard._coverage_from_sheet_rows`, handle `kind: strand` by
adding the frozen canonical strand path to exact coverage. An individual strand must not
grant the whole web or unrelated strands. Continue using accepted frozen paths,
including inherited epic sheets.

Remove the redundant `_collect_committed_paths(new_markers)` call from
`memory_guard_for_new_markers`; the grouped collection already supplies the paths.
Delete any now-dead helper and update tests appropriately.

Drive `memory_guard_for_new_markers` in tests, including actual accepted-plan and
frozen-sibling loading with current cwd outside the plan checkout. Prove a frozen grant
covers `sase/memory/tui.md`, a new granted strand is covered, another strand stays
uncovered, and a nested authored `src/sase/ace/AGENTS.md` remains `other`. Keep
generated-root coverage conditional on a covered memory change in the same repo. Assert
one changed-path collection per new commit marker and preserve the advisory,
non-blocking behavior.

## 8. Keep quiet auto receipts in the ACE inbox

Add a single notification predicate beside `RECEIPT_TAG` in
`sdd/plan_decision_handoff.py`: true only for a silent, non-muted notification tagged
`plan_decisions_receipt` whose action is null. Use it from
`ace/tui/actions/agents/_notification_provider_direct.py` to retain these rows on the
modal's direct page while ordinary silent rows remain filtered. Use a lazy import where
appropriate for startup cost. Preserve read/dismissed filtering and bounded-page
behavior.

Counts, unread-id tracking, startup cursors, toasts, and the arrival bell continue to
exclude silent receipts. Preserve `silent=True`, `muted=False`, `action=None`, and
per-request deduplication. In `docs/notifications.md`, document this intentional
exception: visible in the ACE inbox and delivered quietly by Telegram, with no unread
bump, toast, or bell. Correct `post_auto_approval_receipt`'s docstring to describe the
actual inbox exception.

Tests post a real auto receipt, load the direct provider page used by the modal, and
assert it is present with unchanged unread counts and no unread activity cursor or
toast/bell event. Include negative predicate cases and pagination, deduplication on
retry, and decision-free approvals posting no receipt. The goldens phase owns PNG
updates; this phase supplies behavioral coverage.

## 9. Clear the unused-public acceptance helper

`notification_gates/decision.write_acceptance_meta` has an in-module production caller
and is exported unnecessarily. Confirm current consumers, rename it to
`_write_acceptance_meta`, update its internal call, and remove its `__all__` entry.
Adjust imports if needed. Resolve any new Symvision diagnostic produced by the resolver
and guard cleanup using the documented hierarchy.

## 10. Complete the approval and writer acceptance matrix

Extend the existing gate/repair/handoff/action tests or add focused test modules using
shared real-gate fixtures. Use authored decision order `zeta, alpha` and choice order
`b, a` to catch accidental sorting. Cover:

| Route                          | Required evidence                                                                                        |
| ------------------------------ | -------------------------------------------------------------------------------------------------------- |
| Tale approve + commit          | One accepted vector; author/choice order, attribution, durable stamp, and frozen sibling survive archive |
| Tale approve only              | Stamp and frozen sibling exist without a commit option                                                   |
| Tale commit only               | Stamp and archive exist with no coder launch                                                             |
| Epic approval                  | Complete ordered stamp and frozen definitions; isolate epic launch                                       |
| Direct file, defaults and `-D` | Actual resolved execution stamps the same vector/definitions with reviewer/cli attribution               |
| Hand-run bead work             | Human reviewer/cli and agent agent/cli classification, accepted memory reuse, and fresh refusal/errors   |
| Retry recovery                 | Stored vector and original coordinates re-stamp without live host resolution                             |

Check persisted YAML and siblings, not merely helper return values. Exercise real
handlers/executor entry points while isolating unrelated agent launches, push, and user
notification state. On the gate routes touched, omitted values and explicit effective
defaults must still share `input_identity`; differing AND option values must remain a
refusal.

Add writer-side tests for bundle `payload.decisions` reaching the durable sibling,
existing siblings never being overwritten by stamping/recovery, and
`sdd/plan_archive.archive_plan_file` copying a valid sibling. Drive the approval archive
writer with isolated stores and assert the commit includes both the approved Markdown
and its sibling. Mock external publication only at its boundary; tests must not mutate
production sidecars. If these tests expose a missing writer step in a route owned here,
repair it and retain the regression test.

Test `decision-host-check-failed` through both `sase plan validate` and
`sase plan propose` handlers by forcing a host-check exception on a valid
decision-bearing plan. Assert a failing diagnostic before formatting/archive, gate
creation, or handoff; never invoke a real proposal handoff in the test. The existing
helper-level tests are useful but do not satisfy this route coverage.

## Verification and completion

1. Ensure the implementing checkout's dependencies and Rust bindings are current; use
   `just install` if needed and the monitor skill for known-long commands. Required
   binding coverage must run rather than silently skip on a stale wheel.
2. Run the focused suites for bead work, gate rejection/recovery, direct approval,
   handoff/frozen writers, selector errors, finalizer guard, and direct notification
   page/lifecycle behavior. Use the repo's governed test path and inspect the results.
3. Run formatting/fixes, then `sase tool run check` in every changed repo. Use
   `/sase_monitor` with `TESTING`/`TESTED` for long verification and follow through
   until results are reviewed. `just check-full` is outside this phase's verification.
4. Fix NEW/UNKNOWN failures caused by this phase. The landing-finish plan lists the
   known base failures and owners (`sase-1hr`, `sase-1hy`, `sase-1i9`, `sase-1ia`,
   `sase-1ic`, `sase-1g3`, `sase-1hs`, and Symvision backlog `sase-1hp`). Use the tool
   run's classification and independent base evidence. For a newly discovered failure
   that reproduces identically on the clean base, record the reproduction and any
   existing owner as `PROPOSED FOLLOW-UP:` on this phase and proceed to completion.
5. Record all out-of-scope discoveries only with
   `sase bead note sase-1hi.10.7.1 'PROPOSED FOLLOW-UP: <summary — detail>'`. Phase
   workers do not create beads or manually set their status.
6. Before closing, run `sase bead epic-symbols sase-1hi.10.7.1` again. It was empty on
   the inspected base. Resolve every newly present symbol or re-key its Justfile entry
   to a still-open consuming phase or parent; do not leave a row for this bead.
7. Close only `sase-1hi.10.7.1` with
   `sase bead close sase-1hi.10.7.1 --note "<implementation and verification evidence>"`.
   Its evidence should name the accepted-memory reuse test, durable stale record,
   stamp/writer matrix, strand guard coverage, quiet inbox receipt, check result, and
   any proven base failures. Ancestor closures, end-to-end land smoke, skill deployment,
   and parent-plan completion remain the land agents' work.
8. Let host-owned finalization commit the changes through `/sase_final`; no manual
   commits, branches, or PRs are part of this task.
