---
tier: tale
title: Pass the declared bead action to pinned-sibling stitches
goal:
  Bead-assigned agents whose host commit includes the pinned sase-core sibling land that
  sibling with -B keep instead of failing with missing_bead_action, and a test
  exercising the real Rust policy guards the regression.
size: small
proposed_by: bbugyi200.athena.0u9
create_time: 2026-09-30 05:57:35
status: wip
---

# Fix: pinned-sibling stitches drop the declared bead action

## Problem

Since `c257a3f220` ("feat(finalizer): add revision_pin for linked repos with commit
ordering and pin update"), every bead-assigned agent whose host commit includes the
pinned `sase-core` linked repo fails at the commit finalizer with:

```
sase stitch create failed for sase-core: ❌ missing_bead_action: bead_action is
required when a bead is assigned; use -B keep or -B close
```

The retry is then skipped as `stitch_retry_skipped_identical_inputs`, the agent ends
`FAILED`, its `sase-core` changes stay uncommitted in its held workspace, and every
downstream epic phase that waits on it stays blocked. The observed victims were the
first phases of three epics (`sase-1cj.12.1`, `sase-1cx.1`, `sase-1d5.1`) across three
providers (muse, grok, codex). The failure does not depend on the model; it is
deterministic.

## Root cause

`src/sase/finalizers/commit_dispatch.py`, inside `dispatch_commit_decisions`:

```python
bead_action = _decision_bead_action(decision)
if repo.kind != "main" and repo.name in revision_pins:
    # Pinned siblings land first; only the main stitch applies
    # bead_action so the assigned bead closes after the pin follows.
    bead_action = None
```

`run_stitch_create` (`src/sase/finalizers/commit_repair_stitch.py`) only adds `-B` when
`bead_action is not None`, so the pinned sibling's stitch runs with no `-B`. The Rust
bead-action policy (`decide_bead_action`, reached through `sase.core.bead_action_facade`
from `sase.workflows.commit.bead_hooks`) requires an explicit action whenever a bead is
assigned, in every repository scope:

| scope              | no `-B`               | `-B keep` | `-B close`                          |
| ------------------ | --------------------- | --------- | ----------------------------------- |
| primary            | `missing_bead_action` | ok        | ok                                  |
| linked/sdd/unknown | `missing_bead_action` | ok        | `close_requires_primary_repository` |

The override is also unnecessary. `_validate_commit_decision` in
`src/sase/finalizers/declaration_manifest.py` already rejects a declaration unless every
decision carries `keep` or `close` when a bead is assigned, and
`validate_finalizer_bead_decision` rejects `close` on any obligation except the primary
one (`commit_bead_action_invalid` / `close_requires_primary_repository`). An accepted
sibling decision therefore always carries `keep` when a bead is assigned, which is
exactly what the sibling stitch needs, and the "only the main stitch closes the bead"
intent already holds without the override.

The existing test locked in the bug:
`tests/test_commit_revision_pin_dispatch.py::test_dispatch_commits_pinned_sibling_first_and_follows_pin`
declares `"bead_action": "close"` on the sibling (a declaration the validator would
reject), asserts `bead_actions["sase-core"] is None`, and uses a fake `stitch_runner`,
so the real policy never ran and the regression passed CI.

## Changes

### 1. Remove the override (`src/sase/finalizers/commit_dispatch.py`)

Delete the
`if repo.kind != "main" and repo.name in revision_pins: ... bead_action = None` block so
the pinned sibling stitch gets its own declared `bead_action` (`keep` when a bead is
assigned, `None` when none is). Do not replace it with a silent `close` → `keep`
downgrade. The declaration validator already makes a sibling `close` impossible, and if
some future path ever smuggled one through, the stitch's loud
`close_requires_primary_repository` refusal is the right failure.

Nothing else consumes the stripped value. The same local `bead_action` flows into
`stitch_attempt_input_fields` (retry fingerprint), `_call_stitch_runner`,
`_rescue_landed_commit_after_bounds_failure`, and `resolve_commit_conflict`, so fixing
it at this one assignment fixes the create, bounds-rescue, and conflict-repair/resume
paths together. `commit_unpushed_resume.py` already reads the decision's `bead_action`
directly and needs no change. Grep `src/sase/finalizers` for any other
`bead_action = None` to confirm this is the only override. At plan time it was.

### 2. Fix and extend the dispatch tests (`tests/test_commit_revision_pin_dispatch.py`)

- In `test_dispatch_commits_pinned_sibling_first_and_follows_pin`, make the fixture
  realistic: give the `FinalizerExecutionContext` an `assigned_bead_id` (e.g.
  `"sase-x.1"`), declare the sibling `"bead_action": "keep"` and main `"close"`, and
  assert `bead_actions["sase-core"] == "keep"` and `bead_actions["main"] == "close"`.
  Update the stale "Only the main stitch applies bead*action" comment to say that only
  the primary stitch may _close* the bead, while the sibling carries `keep`.
- Close the test gap that let this ship: inside the fake `stitch_runner`, pass each
  dispatched `bead_action` through the real Rust policy via
  `sase.core.bead_action_facade.decide_bead_action` (and
  `bead_action_wire_schema_version`), using `repository_scope="linked"` /
  `primary_repository_identified=False` for the sibling and `"primary"` / `True` for
  main, plus `assigned_bead_id` and (for `close`) `bead_status="in_progress"`. Mirror
  the request shape built by `_bead_action_decision` in
  `src/sase/workflows/commit/bead_hooks.py`. The policy raises `ValueError` on refusal,
  so the test fails the same way production did if an override ever strips or
  mis-assigns `-B` again. Factor this into a small helper in the test module if it reads
  better.
- Add a regression test for the exact production shape, where only the pinned sibling is
  dirty and declared (`keep`) and main is not in the declaration. Assert that the
  sibling stitch receives `"keep"`, that dispatch succeeds, and that the pin write is
  skipped with a `revision_pin_skipped` warning diagnostic (existing behavior when the
  primary decision is not `commit`) rather than failing.
- Verify the result with `-B` present. Add or extend a test in the existing
  stitch-runner test coverage (search `tests/` for `run_stitch_create`) asserting that
  `run_stitch_create(..., bead_action="keep")` puts `-B keep` in `argv`, if no such
  assertion already exists.

### 3. Correct the docs (`docs/commit_workflows.md`)

In the `repos.linked[].revision_pin` paragraph of the commit-finalizer dispatch section,
replace "applying `bead_action` only on the primary stitch" with wording that matches
the fixed behavior. Each stitch passes its own declared `bead_action`: the pinned
sibling carries `keep` (only the primary repository may close the assigned bead), and
the primary stitch applies the declared `keep`/`close`. Check `docs/configuration.md`
(`repos.linked[].revision_pin` row) and `docs/rust_backend.md` ("The CI source revision
pin") for any similar claim and align them if needed.

## Out of scope

- The Rust bead-action policy is correct; no `sase-core` change or
  `sase-core-revision.txt` bump is needed.
- Writing the pin when the primary is not otherwise committing is an existing design
  choice (it records `revision_pin_skipped`). This plan does not change it.
- Recovering the three stranded runs (landing their uncommitted `sase-core` work and
  unblocking their waiting epic phases) is an operator task. Do not touch other
  workspaces or agents.

## Verification

- `just test tests/test_commit_revision_pin_dispatch.py tests/test_commit_revision_pin.py tests/test_commit_revision_pin_write.py tests/test_commit_revision_pin_config.py`
  plus any stitch-runner test module you touched. The updated sibling test must fail
  against the old override (temporarily reintroduce it to confirm, then remove it).
- Follow the `lint_and_test` reference memory for this repo (read it with
  `/sase_memory_read` before finishing) and run `just check`.
