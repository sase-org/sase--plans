---
tier: tale
title: Reconcile memory, plan publication, and flag policy contracts
goal:
  Complete phase sase-rm.3 with deterministic memory checks, single-writer plan
  publication, frozen flag-triage evidence, and explicit task-type refusal copy.
size: medium
proposed_by: bbugyi200.athena.sase-rm.3
bead: sase-rm.3
create_time: 2026-08-20 15:05:14
status: wip
---

- **PROMPT:**
  [prompts/202608/policy_publication.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/policy_publication.md)
- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/sase-rm.3.md)

# Reconcile memory, plan publication, and flag policy contracts

## Goal

Complete phase `sase-rm.3` against the five assigned task contracts without editing
canonical memory files or closing any task or ancestor bead. Make approval-time plan
publication single-writer and recoverable, freeze feature-flag call-site evidence into
strictly validated triage gates, preserve the current global-only flag-scope contract,
and carry explicit task-type create-refusal copy through the Rust wire and Python
catalog.

## Current evidence and constraints

- `sase-n0` currently reproduces on a clean freshly installed tree: both
  `.venv/bin/sase validate` and `.venv/bin/sase init memory --check --diff` agree that
  `~/.local/share/chezmoi/home/sase/memory/README.md` is stale by `+2/-1`. The user has
  not authorized a canonical memory edit, so this phase may add hermetic precedence and
  drift coverage but must leave the task open with the exact required memory action.
- `sase-n3` has two canonical plan writers: the approval side effect publishes through
  `archive_approved_plan`, while the resumed runner writes and commits the same tale
  path in `handle_accepted_plan`. The approval response already persists
  `saved_plan_path`, but `PlanApprovalResult` discards it.
- `sase-o2`'s project-scope premise was retired by `98b27e849`: current definitions have
  no scope field, local config is rejected, and `sase flag new` exposes no scope option.
  Do not reintroduce project-scoped flags; pin the explicit global-only rejection
  contract and record cancellation-ready evidence.
- `sase-o3` can reuse `find_flag_call_sites`, but the scan must occur once when the gate
  is created. The persisted payload, preview renderer, and strict validator must all use
  the frozen value, including an explicit zero-call-site state.
- `sase-qz` needs an optional `create_refusal` field. Missing values must retain the
  existing `when_to_use` fallback, while the builtin feature type supplies refusal copy
  that survives the machine-global `agent_creatable: false` override. Shared wire and
  digest behavior belongs in the opened `sase-core` linked repository.

## Implementation

1. Extend the generated-memory hermetic tests to cover project-versus-home/chezmoi
   rendering precedence and the complete generated-note inventory in the home README. Do
   not edit or initialize canonical memory. Re-run both pinned checks at the end and
   capture their matching result plus the remaining unauthorized README action.
2. Thread the approval response's `saved_plan_path` through `PlanApprovalResult`. In
   `handle_accepted_plan`, treat that already-published path as the canonical tale plan,
   remove the runner's plan write/commit path and its competing `Add SDD files` push,
   preserve prompt publication and legacy fallback when an old response lacks a saved
   path, and update focused tests to prove one commit policy plus idempotent repeated
   and recovery behavior.
3. Keep feature flags global-only. Add focused model/resolver/env tests that project
   scope cannot be registered or locally resolved while global snapshots still pin child
   processes, documenting why the old `sase-o2` reproduction is inapplicable on the
   current architecture.
4. Capture `find_flag_call_sites` output at FlagTriage creation, normalize it into the
   persisted payload, parse it strictly, render a deterministic `Call sites` section
   (with an explicit empty state), and rebuild previews from only frozen payload data.
   Cover populated, empty, malformed, and post-creation source-mutation cases.
5. In `sase-core`, add optional `create_refusal` support to task-type spec/snapshot wire
   structs, validation, stable digest, serialization, and PyO3 round trips without
   changing digests for specs that omit the field. In the Python repository, use the
   field before the compatibility fallback, give builtin `feature` non-contradictory
   refusal copy, update catalog snapshots, and test explicit and fallback messages.
6. Run focused Python suites for memory init, approval publication, feature flags,
   FlagTriage, and task types. Run `just check` in `sase-core`, rebuild the editable
   Rust binding with `just install`, then run primary `just check`; use the long-check
   monitor workflow if a full lane becomes necessary. Inspect `git diff` and both
   repository statuses, record one close-ready evidence block per assigned task on
   `sase-rm.3`, and record any discovery only as a `PROPOSED FOLLOW-UP:` phase note.
7. Run `sase bead epic-symbols sase-rm.3` and resolve every leftover symbol or re-key it
   to a still-open bead. Close only `sase-rm.3` with a verification note; do not close
   the five task beads, parent epic, or any ancestor.

## Verification

- The two pinned generated-memory checks report the same status and diagnostics; no
  canonical memory file is changed without approval.
- Approval/recovery tests show the approval archive is the only canonical tale-plan
  writer and publisher, and runner recovery consumes its saved path without a second
  commit.
- Feature-flag tests prove global snapshot transport and explicit rejection of local or
  project scope; FlagTriage tests prove frozen populated/empty call-site evidence and
  byte-comparable validation.
- Rust and Python task-type tests agree on validation, digest, snapshots, explicit
  refusal copy, and fallback compatibility.
- `sase-core` `just check` and primary `just check` pass after the binding rebuild, or
  unrelated failures are recorded as phase follow-ups under the prescribed protocol.
