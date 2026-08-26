---
tier: tale
title: Execute the conflict-repair turn's follow-up commit declaration
goal:
  A commit finalizer conflict repair that legitimately dirties the repaired repository
  after `sase stitch create --resume` finishes the run by executing the repair turn's
  own declaration as one follow-up commit, instead of failing the agent with
  `dirty_after_stitch` after the primary commit has already landed.
size: medium
proposed_by: bbugyi200.athena.0dz
---

# Plan: Execute the conflict-repair turn's follow-up commit declaration

## Context: the failure being fixed

Agent `0dx` (`ace(run)-260825_163216`, model `gpt-5.6-sol`) failed after a 4h26m run
with:

```
BuiltinCommitFinalizerError: sase stitch create left uncommitted attributable paths in
main: src/sase/ace/tui/widgets/patch_onboarding.py,
tests/ace/tui/test_artifact_tab_labels.py,
tests/ace/tui/widgets/test_changespec_onboarding.py
```

raised at `src/sase/finalizers/commit_dispatch.py:378` with diagnostic code
`dirty_after_stitch`.

### `just fix` is ruled out

The project configures `commit_hooks.before: "just fix"` in `sase/sase.yml`, so the
suspicion is reasonable, but the evidence denies it:

1. `src/sase/workflows/commit/workflow.py:159-161` runs the before-hook, and the git
   provider then stages with `git add -A`
   (`src/sase/vcs_provider/plugins/_git_commit_dispatch.py:143-145`). Anything
   `just fix` rewrites is staged into the same commit, so the hook cannot strand a
   modified file.
2. `sase stitch create --resume` (`src/sase/workflows/commit/workflow_resume.py`) never
   re-runs the before-hook, and the failure happened on the resume path.
3. `just fix` is `fmt-py`, `fmt-docs`, `fmt-md`, and `fix-keep-sorted` (YAML only). The
   three stranded diffs are semantic, not cosmetic: they delete duplicate `"agents"`
   keys from `_ARTIFACT_DESCRIPTIONS` / `_ARTIFACT_LABELS`, restore an agent description
   string, and reorder a `positions[...]` assertion chain. No formatter produces that.

### What actually happened

The run's workflow log (`sase chat`-adjacent transcript for this run) narrates the whole
sequence, and the committed history corroborates every step:

1. The main turn submitted its declaration; the commit finalizer ran
   `sase stitch create`, which rebased onto upstream `master`. A concurrently landed
   commit had touched the same Artifacts sub-tab structures, so the rebase conflicted
   and `sase stitch create` exited `EXIT_CODE_CONFLICT`.
2. `dispatch_commit_decisions` called `resolve_commit_conflict`
   (`src/sase/finalizers/commit_repair.py:84`), which ran the one-shot conflict-repair
   turn.
3. The repair agent resolved the marker conflicts, regenerated the affected PNG goldens,
   continued the rebase, and ran `sase stitch create --resume`. The primary commit
   **landed successfully** as `feat(artifacts): put agents first in subtabs`.
4. Git had silently auto-merged two _semantic_ conflicts that carry no conflict markers:
   both sides added an `"agents"` entry to the same dictionaries, so the merged file
   contained duplicate keys. Nothing in git's conflict machinery can see that.
5. Per this project's `just check` convention, the repair agent then ran `just check`.
   Ruff's duplicate-key rule and two onboarding tests failed. The agent fixed the three
   files — **after** the commit already existed.
6. The repair agent finished through `/sase_final` and submitted a second declaration
   with the message `fix(artifacts): remove duplicate agent tab leftovers`.
7. Control returned to `dispatch_commit_decisions`, which ran
   `remaining = unexpected_path_resolver(repo.path, protected)` at
   `commit_dispatch.py:365` and raised `dirty_after_stitch`.

The user later had to land those three files by hand as
`fix: Commit changes left over from '0dx' sase agent`. That manual commit is exactly the
commit the finalizer should have made.

### Root cause

The conflict-repair prompt and the post-stitch guard contradict each other.

`_run_conflict_repair_turn` (`src/sase/finalizers/commit_repair.py:456-471`) tells the
repair agent:

> "This does not change what you owe elsewhere. Your standing obligation to declare and
> commit every repository you changed this turn is unaffected: after the resume
> succeeds, finish the turn through `/sase_final` as usual, including any other
> repository that is still dirty."

So the repair turn is explicitly authorized to keep working, and a silent semantic
conflict makes post-resume work effectively mandatory under this project's verification
convention. But:

- `dispatch_commit_decisions` computes `protected` **before** the first stitch attempt
  (`commit_dispatch.py:173`) and reuses that stale set for the post-stitch check at
  `commit_dispatch.py:365`. `unexpected_remaining_paths`
  (`src/sase/finalizers/commit_validation.py:194-197`) is "every currently changed file
  minus `protected`", and this run's baseline record for `main` had no fingerprints, so
  `protected` was empty. Every file the repair turn touched became a fatal
  `dirty_after_stitch`.
- `execute_commit_finalizer` loads the accepted declaration exactly once, at
  `src/sase/finalizers/commit.py:109-115`, before dispatch. The repair turn's second
  `sase final submit` writes a newer submission that **nothing ever reads**. The
  finalizer has no path to honor a follow-up commit for the repaired repository.

The result is the worst possible outcome: the primary commit is already in history, the
follow-up fixes are correctly authored and correctly declared, the run dies anyway, and
a workspace is held for manual repair.

## Implementation

Three coordinated changes. Keep them in one patch — the prompt change is only truthful
once the mechanism exists.

### 1. `src/sase/finalizers/commit_dispatch.py`: execute the post-repair declaration

In the per-repository loop, track whether this repository went through
`resolve_commit_conflict` (the `stitch.returncode == EXIT_CODE_CONFLICT` branch). Then
restructure the `remaining` check at `commit_dispatch.py:365-378`:

- If `remaining` is empty, behave exactly as today.
- If `remaining` is non-empty and the repository did **not** go through conflict repair,
  raise `dirty_after_stitch` exactly as today. This case is unchanged; the existing
  invariant still holds.
- If `remaining` is non-empty and the repository **did** go through conflict repair,
  attempt exactly one follow-up commit:
  1. Re-load the latest accepted declaration for `context.artifacts_dir`. Reuse
     `load_accepted_commit_declaration` and `commit_decisions_for_instance` from
     `sase.finalizers.commit_declaration` — `load_latest_finalizer_submission` already
     returns the newest submission, which is the repair turn's. Guard the load in
     `try/except`; a load or validation failure must degrade to the existing
     `dirty_after_stitch` failure with the new explanatory message, never to a crash.
  2. Look for a `commit` decision keyed by this repository's
     `repository_decision_id(repo)`. If there is none, or its action is not `commit`, or
     its message is empty, fall through to the `dirty_after_stitch` failure.
  3. Otherwise call `stitch_runner(repo, follow_up_message, protected, context)` once,
     record its artifacts through `record_stitch_artifacts` with a distinct label such
     as `f"{repo.name}.post-repair"` so the immutable-artifact guard does not collide
     with the first attempt, and apply the same result handling the primary stitch
     already uses: timeout / output-cap, a second `EXIT_CODE_CONFLICT` (fail with the
     existing `second_unresolved_conflict` code — do **not** start another repair turn),
     and non-zero exit (`stitch_failed`).
  4. Verify a new commit marker for this repository the same way the primary path does
     (`new_commit_markers` + `marker_matches_repo`), extend `evidence` with
     `marker_evidence`, and call `reconcile_commit_file_hooks` for the follow-up marker.
  5. Recompute `remaining`. If it is now empty, continue normally. If not, raise
     `dirty_after_stitch` with the new message.

  Add an `evidence` entry (kind `conflict_repair_followup`, value `success`) so the
  outcome wire records that a follow-up commit ran.

**Bounding.** The follow-up must be attemptable at most once per run.
`resolve_commit_conflict` is already one-shot — it refuses a second repair whenever
`_conflict_repair_spent(context.artifacts_dir)` is true (`commit_repair.py:102-113`,
backed by the `conflict_repair_prompt.md` artifact) — so gating the follow-up on "this
repository took the conflict branch in this dispatch" is sufficient and needs no new
state. Do not loop; one follow-up stitch, then re-check, then decide.

**Attempt budget.** Reuse the already-consumed `attempt_id` for the follow-up rather
than calling `_consume_attempt()` again. The follow-up is the tail of a single repair
sequence, it is structurally bounded to one execution, and consuming the second of the
default `max_attempts: 2` here would silently remove the retry budget that
`stitch_failed` still needs. Record the follow-up under that same attempt via the
distinct artifact label.

**Message.** Do not derive or synthesize a commit message. The message must come from
the repair turn's declaration, because
`sase memory read decisions:host-owned-completion` requires that an agent's declaration
— not the machinery — author what gets committed. If no such declaration exists, fail;
do not invent one.

### 2. `src/sase/finalizers/commit_dispatch.py`: make `dirty_after_stitch` actionable

The current message names only the paths. Extend it, for the conflict-repair case, to
state:

- that the primary commit for this repository already landed (include the marker's
  commit sha, which is already in `evidence`),
- which paths are still dirty,
- and why the follow-up did not happen — one of "the conflict-repair turn submitted no
  commit declaration for this repository", "the declaration could not be loaded", or
  "the follow-up commit still left these paths dirty".

Keep the leading
`sase stitch create left uncommitted attributable paths in {repo.name}:` sentence so
existing operator muscle memory and any log greps keep working. Leave the non-conflict
path's message exactly as it is today.

### 3. `src/sase/finalizers/commit_repair.py`: fix the conflict-repair prompt

Update the prompt in `_run_conflict_repair_turn` (`commit_repair.py:456-471`) so it
matches the mechanism and prevents the avoidable half of this failure:

- **Add a verify-before-continue instruction.** Tell the agent that while the operation
  is paused, after resolving every conflict marker and before continuing the paused VCS
  operation, it should run the project's verification gate and fold every resulting fix
  into the staged resolution, so the resumed commit is internally consistent.
- **Warn about semantic conflicts.** Say explicitly that a clean conflict-marker
  resolution does not mean the merge is correct: when both sides add an entry to the
  same list, dict, tuple, or enum, git merges both and produces a duplicate that only a
  lint or test run will catch. This is precisely what happened here and it is not
  obvious.
- **Make the `/sase_final` sentence truthful.** Keep the standing obligation, and state
  that if the repository is still dirty after the resume, the declaration's commit
  decision for it will be executed as a single follow-up commit — so the message it
  writes is the message that lands.

`tests/test_finalizers_commit_repair_prompt.py` asserts on prompt substrings
(`"second commit"` and `"another commit"` must stay absent;
`"paused operation in {name}"`, `"fresh commit in {name}"`,
`"every repository you changed this turn"`, and `"/sase_final"` must stay present).
Preserve all of them; the two negative assertions are load-bearing and the new wording
must not reintroduce those phrases.

## Testing

Extend `tests/test_commit_dispatch_protection_guard.py` or add a sibling
`tests/test_commit_dispatch_conflict_repair_followup.py`. That existing file already has
the exact harness this needs: a fake `stitch_runner` that writes `commit_results.json`,
a mutable `changed_files` list driving `unexpected_path_resolver`, and injected
`prepare_dirty_state` / `protected_path_resolver` / `baseline_record_resolver`
callables. Drive the conflict path by returning `EXIT_CODE_CONFLICT` from the first
`stitch_runner` call and passing a `MagicMock` provider so `_run_conflict_repair_turn`
returns without a real model.

Cover:

1. **Happy path.** First stitch returns `EXIT_CODE_CONFLICT`; the repair turn writes a
   second accepted declaration; `resume_runner` produces the primary marker; residual
   dirty paths remain. Assert exactly one follow-up `stitch_runner` call, that it used
   the repair turn's declared message, that a second commit marker was recorded, that
   `dispatch_commit_decisions` returns normally, and that the ledger consumed only one
   attempt.
2. **No post-repair declaration.** Same setup but the latest submission carries no
   commit decision for this repository. Assert `dirty_after_stitch`, that the message
   names the already-landed commit sha and says no declaration covered the residue, and
   that no follow-up stitch ran.
3. **Follow-up still dirty.** The follow-up stitch succeeds but paths remain. Assert
   `dirty_after_stitch`, and assert the follow-up ran exactly once (no loop).
4. **Follow-up conflicts again.** The follow-up stitch returns `EXIT_CODE_CONFLICT`.
   Assert `second_unresolved_conflict` and that no second repair turn was invoked
   (`provider.invoke.call_count == 1`).
5. **Non-conflict path is unchanged.** A clean stitch that leaves residual dirt still
   raises `dirty_after_stitch` with today's message, and never loads a declaration or
   runs a follow-up stitch.
6. **Declaration load failure degrades.** Make the declaration load raise; assert
   `dirty_after_stitch` rather than the raw exception.
7. **Prompt wording.** In `tests/test_finalizers_commit_repair_prompt.py`, keep the
   existing assertions and add ones for the verify-before-continue instruction, the
   semantic-conflict warning, and the follow-up-commit promise.

Also add an `error_report.md` rendering assertion in the style of
`test_protection_exhausted_error_report_names_reason_and_paths`, so an operator reading
only the error report can tell that the primary commit landed and why the follow-up did
not.

## Acceptance criteria

- A conflict repair whose post-resume fixes are declared through `/sase_final` completes
  the run: the primary commit and one follow-up commit both land, and the finalizer
  returns success.
- `dirty_after_stitch` still fires, with its current message, for a repository that did
  not go through conflict repair.
- The follow-up commit is attempted at most once per repository per run, and never
  triggers a second conflict-repair turn.
- The follow-up commit's message comes from the repair turn's declaration; the machinery
  never synthesizes one.
- The conflict-repair prompt tells the agent to verify before continuing the paused
  operation, warns about marker-free semantic conflicts, and truthfully describes what
  happens to a post-resume declaration.
- `just check` passes.

## Out of scope

- Changing `just fix`, the `commit_hooks.before` contract, or any formatter. The
  investigation ruled the hook out as a cause.
- Widening `protected` / baseline-fingerprint semantics. The stale `protected` set is
  how the guard reaches the wrong conclusion, but broadening protection would let
  genuinely discarded work pass silently, which is what `dirty_work_discarded` exists to
  prevent.
- Detecting semantic (marker-free) merge conflicts automatically. The prompt change
  tells the agent to catch them with the project's own verification gate; a general
  detector is a separate, much larger problem.
- Allowing more than one conflict-repair turn per run.
