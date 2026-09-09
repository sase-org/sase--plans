---
tier: tale
title: Make the commit finalizer's conflict-repair budget per-repository
goal: A first conflict in any declared repository gets its own conflict-repair turn;
  only a conflict that resurfaces in a repo that already consumed its repair turn
  fails the run.
size: medium
proposed_by: bbugyi200.athena.0ht
status: done
---

# Make the commit finalizer's conflict-repair budget per-repository

## Problem

Agent `sase-xe.16.11.6.1` (epic clan `sase-xe.16.11.6`) failed after ~2h51m with:

```
BuiltinCommitFinalizerError: commit finalizer hit a second unresolved conflict in sase-core
```

The failure evidence
(`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/09/20260909122331/`) shows
the agent was never given a chance to resolve the sase-core conflict:

1. The finalizer's first `sase stitch create` for repo `main` hit a rebase conflict (one
   test file, `attempt-1.main.stdout`). The one-shot conflict-repair turn was spawned
   for `main`, succeeded, and the commit landed and pushed (`8c8dfc3f6`, evidence
   `conflict_repair: success`).
2. Roughly 1.5 hours later, the dispatch for the **linked sase-core repo** hit its
   **first-ever** conflict (`attempt-1.sase-core.stdout`: rebase vs `origin/master`,
   five incoming commits from sibling clan workers touching the same files — an expected
   situation for parallel epic phase workers).
3. `resolve_commit_conflict()` immediately raised `second_unresolved_conflict` for
   sase-core without spawning any repair turn, discarding ~3 hours of verified work.

## Root cause

`_conflict_repair_spent()` in `src/sase/finalizers/commit_repair.py` keys the "single
conflict-repair turn" budget on the existence of one **run-global** flag file,
`finalizers/commit/conflict_repair_prompt.md`. The budget is therefore _one repair turn
per run_, not _one repair turn per repository_:

```python
def _conflict_repair_spent(artifacts_dir: str | None) -> bool:
    artifact_dir = instance_artifact_dir(artifacts_dir, "commit")
    if artifact_dir is None:
        return False
    return (artifact_dir / _CONFLICT_PROMPT_FILENAME).is_file()
```

`resolve_commit_conflict()` checks this flag before spawning a repair turn and raises
`"commit finalizer hit a second unresolved conflict in {repo.name}"` when it is set —
even when `{repo.name}` never had a conflict before, which is both a lost repair
opportunity and a misleading error message.

The intended design (commit `980bedfea`, and the harness tests
`test_first_repo_conflict_blocks_later_dispatch` /
`test_reversed_manifest_first_host_repo_conflict_blocks_later` in
`tests/test_finalizers_protocol_harness_multi_repo.py`) only asserts that a conflict
that **resurfaces in the same repo** after its repair turn fails the run. No test
asserts that a successfully repaired conflict in repo A must starve a later first
conflict in repo B; that behavior is an accident of the global flag file.

Secondary defect: `_run_conflict_repair_turn()` writes the repair prompt and response to
the fixed filenames `conflict_repair_prompt.md` / `conflict_repair_response.md` (not
exclusive), so a second repair turn in the same run would silently overwrite the first
repo's repair artifacts.

## Fix

Scope the conflict-repair budget to one repair turn **per repository per run**, keeping
the run bounded (at most one repair turn per repository named in the accepted commit
declaration) while giving every repo's first conflict a fair repair opportunity.

All changes are in the sase repo's Python finalizer host code (no `sase-core` Rust
boundary crossing: this is host orchestration, not shared domain behavior).

### 1. `src/sase/finalizers/commit_repair.py`

- Derive a per-repo artifact label from `repo.name` using the existing
  `_ARTIFACT_LABEL_RE` sanitization (same convention `record_stitch_artifacts` already
  uses for `attempt-N.<label>.*` files).
- `_conflict_repair_spent(artifacts_dir, repo)`: check the per-repo prompt filename
  (e.g. `conflict_repair_prompt.<label>.md`) instead of the global one. No legacy-name
  compatibility shim is needed: the artifacts directory is created fresh per run, so a
  new run can never contain the old filename.
- `_run_conflict_repair_turn()`: write the prompt and response to the per-repo filenames
  (e.g. `conflict_repair_prompt.<label>.md` / `conflict_repair_response.<label>.md`) so
  multiple repos' repair artifacts coexist for diagnosis.
- Update the repair prompt wording "This is the single conflict-repair turn." to make
  the scope explicit, e.g. "This is the single conflict-repair turn for this repository
  during this run." Keep the rest of the prompt intact — its content is asserted by
  `tests/test_finalizers_commit_repair_prompt.py`.
- The early-raise in `resolve_commit_conflict()` now only fires when _this_ repo already
  consumed its repair turn, so the existing `second_unresolved_conflict` message becomes
  accurate; no message change is required there. The post-resume raise (resume still
  exits with `EXIT_CODE_CONFLICT`) is already per-repo-accurate and stays as is.

### 2. Interactions to preserve (verify, no code change expected)

- The mutating-attempt ledger (`InstanceLedger`, default `max_attempts: 2` in
  `src/sase/default_config.yml`) is a separate axis: repair turns do not consume ledger
  attempts, and one dispatch pass consumes one attempt regardless of how many repos it
  stitches. Confirm the per-repo repair budget does not change ledger accounting.
- Multi-cycle runs: `_run_budgeted_commit` retry loop and the `stale_commit_declaration`
  recovery path can re-enter `dispatch_commit_decisions` within one run. The per-repo
  flag files persist across those cycles, preserving once-per-repo-per-run semantics (a
  repo whose conflict resurfaces in a later cycle still fails fast).

### 3. Tests

Update:

- `tests/test_finalizers_commit_repair_prompt.py`: the prompt-capture helper reads
  `finalizers/commit/conflict_repair_prompt.md`; point it at the per-repo filename, and
  extend/adjust assertions for the clarified single-turn-per-repo wording.
- `tests/test_finalizers_protocol_harness_multi_repo.py`: the two existing same-repo
  tests (`...first_repo_conflict_blocks_later_dispatch`, reversed variant) must stay
  green — they assert `seen == ["main", "resume:main"]` and
  `provider.invoke.call_count == 1`, which per-repo budgeting preserves because the
  second conflict there is in the same repo (`main` resume still conflicted).

Add (in the protocol harness style of
`tests/test_finalizers_protocol_harness_multi_repo.py`):

- Repaired-first-repo does not starve a later repo: repo A conflicts, its repair turn
  resolves it (resume succeeds, marker recorded), then repo B conflicts for the first
  time → repo B gets its own repair turn (`provider.invoke.call_count == 2`), and when
  B's repair also resolves, the run succeeds.
- Same repo, second conflict after successful repair still fails: repo A repaired
  successfully, then a later dispatch cycle hits a new conflict in repo A → fails with
  `second_unresolved_conflict` naming repo A, without a second repair turn for A.
- Per-repo artifacts: after a two-repo repair run, both repos' prompt and response
  artifact files exist under `finalizers/commit/` with distinct names.

### 4. Verification

Run the repo's mandatory verification per the `lint_and_test.md` reference memory (read
it first), including the focused finalizer test modules:
`tests/test_finalizers_protocol_harness_multi_repo.py`,
`tests/test_finalizers_protocol_harness_controller.py`,
`tests/test_finalizers_commit_repair_prompt.py`,
`tests/test_commit_dispatch_conflict_repair_followup.py`.

## Out of scope

- Recovering or re-landing the failed run's uncommitted sase-core changes; that is
  operational cleanup handled outside this plan.
- Any change to the mutating-attempt ledger budget or retry policy.
- No new feature flag: this restores the documented intent of the existing budget
  mechanism (bounded repair, accurate second-conflict failure) rather than introducing
  new user-reaching behavior; the old behavior (fail a never-repaired repo's first
  conflict) is a defect with no backward-compatibility constituency.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact | Why | Uses |
| --- | --- | --- | ---: |
| cited-by | [agent:bbugyi200.athena.0ht--code][1] | prompt reference @plan:202609/per_repo_conflict_repair_budget.md | 1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ht.md

<!-- sase:referenced-by:end -->
