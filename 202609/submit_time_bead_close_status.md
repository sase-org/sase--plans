---
tier: tale
title: Read the live bead status when validating close at final submit
goal:
  A primary-repository /sase_final declaration with bead_action close is accepted when
  the assigned bead is in_progress or already closed, because the host reads the live
  status instead of refusing every close as unreadable_bead_status.
size: medium
proposed_by: bbugyi200.athena.0qe.f0
create_time: 2026-09-23 20:57:06
status: wip
---

# Supply the live bead status when validating `close` at `sase final submit`

## Problem

Every `sase final submit` whose repository decision says `"bead_action": "close"` is
refused with:

```
commit_bead_action_invalid: unreadable_bead_status: the assigned bead status could not be read; close is refused
```

The bead itself can be read fine. The refusal happens because nothing collects the
status fact at submit time:

1. `sase.finalizers.declaration_manifest._validate_commit_decision` copies the
   agent-authored decision keys (`_BEAD_DECISION_KEYS`) into `bead_decision` and passes
   it to the Rust binding `validate_finalizer_bead_decision`.
2. In `sase-core` (`crates/sase_core/src/bead_action/policy.rs`),
   `bead_action_request_for_decision` copies `decision.bead_status` into the policy
   request. `decide_close` allows `close` only for `InProgress` (close) or `Closed`
   (idempotent) and maps `None`, `Unreadable`, and `Unchecked` to
   `unreadable_bead_status`.
3. The `/sase_final` manifest template has no `bead_status` field, and no Python code on
   the submit path calls `sase.workflows.commit.bead_hooks.bead_status_fact`. So
   `bead_status` is always `None`, and `close` is always refused.

The stitch side is fine. When the host runs `sase stitch create -B close`,
`bead_hooks._bead_action_decision` reads the live status with `bead_status_fact` and
applies the same Rust policy. The accepted runs that reached it all printed the expected
idempotent-close messages (for example run `20260920131342`, bead `sase-14d.4`). Only
the submit-time validator never gets that fact.

Impact: 370 refusals since 2026-09-12, right after the explicit-bead-action epic
(`sase-zq`) landed. The only `close` declarations ever accepted hand-wrote
`"bead_status": "closed"` after the agent had already closed the bead manually. The
documented `/sase_final` close path is effectively dead: agents close beads by hand or
leave them open, and the host's close step never runs. The root cause is recorded in
notes #2 and #3 on epic `sase-zq`.

## Design decision

The fix is host-side Python only. The Rust policy is correct as designed: it decides
from facts the host collects. Reading the live status of a bead is host fact collection,
which the `sase-zq` contract ("Python retains environment/filesystem reads, process
execution") keeps in Python. No `sase-core` change, binding change, or
`sase-core-revision.txt` pin move is needed.

- **The host reads the status fact itself, and only the host's value counts.** Keep
  `bead_status` in `_COMMIT_DECISION_KEYS` so already-accepted historical declarations
  that hand-wrote it still load. Never forward the agent-authored value to Rust: always
  drop it from `bead_decision`, then set the host-read value where it applies.
- **Read only when the policy needs it:** the context has an `assigned_bead`, the
  decision's `bead_action` is `close`, and the decision's `repo_id` equals
  `assigned_bead.primary_repo_obligation_id`. Every other case is decided by Rust
  without a status (`keep`, no assigned bead, close on a linked/external repo →
  `close_requires_primary_repository`), so skip the subprocess there.
- **Reuse the stitch path's status reader:** call `bead_status_fact(bead_id, cwd)` from
  `sase.workflows.commit.bead_hooks` so submit and stitch map statuses identically
  (`in_progress` / `closed` / `other` / `unreadable`). Import it lazily inside a small
  module-level helper in `declaration_manifest.py` (same pattern as
  `commit_dispatch_followup._default_assigned_bead_status`). The helper is also the test
  seam.
- **The `cwd` is the host-recorded path of the primary repository:** look up the
  `HostRepositoryRecord` whose `obligation_id` equals the primary obligation ID and use
  its `path`. This is the same directory the host later runs the stitch in. Fail closed:
  if there is no matching record, or the reader raises, use `"unreadable"` so Rust
  refuses the close. Never fall back to the ambient cwd.
- **All three validation paths get the same fact.** `validate_provider_payloads` runs at
  submit (`declaration.submit_final_manifest`), in the currency check
  (`declaration.final_submission_is_current`), and when the host loads an accepted
  declaration (`commit_declaration.load_accepted_commit_declaration`). All three must
  read the live status, or an accepted `close` would fail its own reload. Re-reading on
  load is intended, as the `sase-zq` plan specified. If the host has already closed the
  bead, the reload sees `closed` and accepts it as idempotent. If the bead was moved to
  an ineligible status in the meantime, the reload refuses, just as the stitch would.

## Implementation

### 1. `src/sase/finalizers/declaration_manifest.py`

- Add a required keyword-only parameter `host_records: Sequence[HostRepositoryRecord]`
  to `validate_provider_payloads` and thread it through `_validate_commit_payload` into
  `_validate_commit_decision`. Import `HostRepositoryRecord` from
  `sase.finalizers.declaration_store`, which this module already imports from. Make it
  required rather than defaulting to `()`, so a future caller cannot silently
  reintroduce the bug.
- Add a module-level helper, for example
  `_assigned_bead_status(bead_id: str, cwd: str) -> str`, that lazily imports and calls
  `bead_status_fact`. Add a second helper that resolves the primary record path from
  `host_records` and returns `"unreadable"` when the record is missing or the reader
  raises.
- In `_validate_commit_decision`, after building `bead_decision`:
  - `bead_decision.pop("bead_status", None)` unconditionally;
  - if `assigned is not None`, `decision.get("bead_action") == "close"`, and
    `repo_id == assigned.primary_repo_obligation_id`, set `bead_decision["bead_status"]`
    to the host-read fact.
- Leave the existing error wrapping (`code="commit_bead_action_invalid"`) and the
  template (`bead_action: None` placeholder) unchanged.

### 2. Callers in `src/sase/finalizers/declaration.py`

- `submit_final_manifest`: read
  `_read_host_repository_file(root / FINAL_CONTEXT_HOST_FILENAME)` before the first
  validation `try` block, and pass it as `host_records=`. Keep the later staleness
  comparison working. Either reuse the same variable or keep the existing second read
  for that comparison, but do not change its semantics: it still compares the records
  published with the context against a freshly rebuilt live context.
- `final_submission_is_current`: move the existing
  `load_accepted_host_repositories(root)` call above `validate_provider_payloads`, bind
  it to a local, and pass it both to `validate_provider_payloads` and to
  `adjudicate_commit_deferrals`.
- Keep the `validate_provider_payloads` re-export in `__all__` and the
  `_symvision_static_refs.py` references working. Nothing is renamed, only a keyword is
  added.

### 3. `src/sase/finalizers/commit_declaration.py`

- `load_accepted_commit_declaration`: move
  `finalizer_declaration.load_accepted_host_repositories(root)` above
  `validate_provider_payloads`, pass it as `host_records=`, and return that same tuple
  (no second read).

### 4. Other direct callers

Search the repo for all callers (`rg -n "validate_provider_payloads" src tests`) and
update each one. Known test callers:
`tests/monitor/test_monitor_host_completion_controller.py` and
`tests/test_finalizer_declaration_channel_providers.py`. Pass `host_records=()` where
the test doesn't involve an assigned-bead close, or the real records where it does.

### 5. Documentation

In `docs/commit_workflows.md`, in the paragraph that starts "If `SASE_BEAD_ID` is set,
every commit attempt must say what happens to that assigned bead", add one or two
sentences. They should say that on `sase final submit` the host reads the assigned
bead's live status for a primary-repository `close`, accepts it when the bead is
`in_progress` (close after commit) or already `closed` (idempotent), and otherwise
refuses the declaration so the agent can choose `keep`. Also say that agents never
supply the status themselves. Keep it short and consistent with the "Explicit Bead
Action" section's refusal list. Do not edit generated skill sources for this fix:
`src/sase/xprompts/skills/sase_final.md` already describes the correct behavior ("On
green the host commits, closes the bead when `bead_action` is `"close"`").

## Tests

Use the existing declaration-channel fixtures (`prepare_dirty_declaration`,
`valid_manifest` in `tests/finalizer_declaration_channel_test_helpers.py`) with
`monkeypatch.setenv("SASE_BEAD_ID", ...)`. Patch the new status helper seam so no test
spawns the real `sase` CLI or touches a real bead store. Put the new cases in a focused
new file (for example `tests/test_finalizer_declaration_bead_close_status.py`) rather
than growing an existing large test module.

1. **The regression:** `close` on the primary repository with a live status of
   `in_progress` is accepted by `submit_final_manifest`. The helper is called once with
   the assigned bead ID and the primary repository's host-recorded path (the fixture's
   repo path).
2. A live status of `closed` is accepted (idempotent). Cover the case from `sase-17d.1`,
   where the bead was already closed by hand.
3. A live status of `other` is refused with `commit_bead_action_invalid` and the
   `close_ineligible_status` message. `unreadable`, and a helper that raises, are
   refused with `unreadable_bead_status`. Each refusal is appended to
   `final_submission_attempts.jsonl` with `accepted: false`.
4. The host fact wins over agent-authored values. A hand-written
   `"bead_status": "closed"` is still refused when the live read returns `other` or
   `unreadable`. A hand-written `"bead_status": "unreadable"` is still accepted when the
   live read returns `in_progress`.
5. `keep` never calls the status helper. With no assigned bead, the status helper is
   never called, and `close` is still refused with `close_without_assigned_bead`.
6. `close` on a non-primary repository obligation (two dirty repos: `main` plus a
   `linked` one) is refused with `close_requires_primary_repository`, and the status
   helper is not called for it.
7. A missing primary host record fails closed: the declaration is refused as unreadable
   and the helper is never called with an ambient cwd.
8. After an accepted `close`, `final_submission_is_current()` returns `True` and
   `load_accepted_commit_declaration()` succeeds while the helper reports `in_progress`
   or `closed`. The load raises `FinalizerDeclarationError` with
   `commit_bead_action_invalid` when it reports `other`.
9. **Live end-to-end** in `tests/test_finalizers_live_e2e.py`: add a close counterpart
   to `test_live_assigned_bead_keep_is_authored_and_threaded_to_stitch_runner`. Set an
   assigned bead, patch the status seam to `in_progress`, call
   `submit_from_context(artifacts, bead_action="close")`, run the controller, and assert
   that the stitch runner received `bead_action == "close"`, the result is `success`,
   and `attempt-1.main.inputs.json` records `"bead_action": "close"`. Keep the stitch
   runner as the existing `real_git_stitch` wrapper; it does not run bead hooks, so no
   real bead is closed.

Update any existing test whose expectations depended on `close` being refused at submit
without a status. Current searches show none. The existing e2e test only asserts the
`None` placeholder refusal, which stays unchanged.

## Verification

- Read the current `lint_and_test.md` guidance via `/sase_memory_read` first; run
  `just install` if the workspace virtualenv is stale.
- Run the new test file, `tests/test_finalizer_declaration_channel_*.py`,
  `tests/test_finalizers_live_e2e.py`,
  `tests/monitor/test_monitor_host_completion_controller.py`,
  `tests/test_commit_dispatch_stitch_timeout_rescue.py`, and
  `tests/test_commit_bead_hooks.py` directly with pytest while iterating.
- Finish with `sase tool run check` (the `just check` gate). Do not run
  `just check-full` unless explicitly asked.

## Out of scope

- `sase-10l`, where `test_stitch_create_requires_keep_then_closes_only_assigned_phase`
  fails with `unreadable_bead_status` inside the stitch path. That is a separate,
  possibly environment-dependent failure of the stitch-side `bead_status_fact` read. Do
  not fold it into this change. If the new tests reveal it is the same defect, record
  that as a note on `sase-10l` instead.
- Changing the Rust policy, the finalizer wire contract, or the manifest template.
- Repairing past bead statuses or re-running historical runs.

## Completion

When the implementation and `just check` pass, add a short note to epic `sase-zq`. The
note should say that the submit-time `unreadable_bead_status` defect from notes #2/#3 is
fixed by having the host read the live bead status in `declaration_manifest`, and should
name the new tests. Do not change `sase-zq`'s status. Its land assignee owns that.
Commits are host-owned: declare the work through `/sase_final`.
