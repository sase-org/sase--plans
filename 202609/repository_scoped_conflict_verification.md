---
tier: tale
title: Scope conflict-repair verification to the repository being repaired
goal:
  Sidecar conflict repair validates the resolved content without borrowing an unrelated
  repository's gate, while preserving required verification for code repositories.
size: small
proposed_by: bbugyi200.athena.research.1m.cdx.f0.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.research.1m.cdx.f0.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.research.1m.cdx.f0.f0.md)
- **COMMITS:**
  - [e5106d4](https://github.com/sase-org/sase/commit/e5106d490f89ae28682a3264555f2d8722a4bacc)
    — fix(commit): scope conflict-repair verification to target repo

# Scope conflict-repair verification to the repository being repaired

## Problem and evidence

The built-in commit finalizer currently gives every conflict-repair agent the same
unconditional instruction in
`src/sase/finalizers/commit_repair.py::_run_conflict_repair_turn`:

> Before continuing the paused VCS operation, run the project's verification gate and
> fold every resulting fix into the staged resolution.

The reported incident involved a research sidecar containing a Markdown report and a
JSON artifact-link index. The repair agent interpreted “the project” as the parent SASE
repository and ran its expensive `just check` gate twice. That suite did not validate
the sidecar's resolved index. The current source still contains this wording, and
`tests/test_finalizers_commit_repair_prompt.py` explicitly requires it.

The prompt names `repo.name` but omits the available `repo.path`.
`DirtyRepo.changed_files` is a pre-repair snapshot; it must not be treated as the list
of live unmerged files or as sufficient evidence for selecting verification.

The manual path in `src/sase/xprompts/skills/sase_git_commit.md`, under “On Merge
Conflict,” instead goes directly from staging to continuing. Align these two instruction
surfaces. The standalone sync workflow does not contain the offending project-gate
instruction and is outside this repair's scope.

Changing directory alone is insufficient: `just` can search ancestor directories for its
Justfile. Its installed help documents `--ceiling` and `--justfile`. A sidecar nested
under a code checkout must not accidentally execute that checkout's gate.

## Decision and scope

Implement a **small tale**, suitable for one coding agent. Correct the generated
instructions and their regression coverage in four existing files. This fixes the
observed cause without introducing a gate registry, command-discovery engine, repository
classification heuristic, CLI option, feature flag, or new dependency.

The authoritative rule is: **verification follows the repository and the content being
repaired, subject to that repository's applicable instructions**. Repository kind,
filename extension, and the presence of a build manifest are clues, not policy. A
sidecar can contain executable code and have required checks; a code repository can
require its gate even for documentation edits.

This work changes agent-facing text at the existing Python provider-invocation point and
a shipped skill template. It adds no shared backend decision logic or wire/API behavior,
so no Rust-core implementation is needed. Do not build a Python verification policy
engine as part of this change.

## Required instruction contract

Teach the following in the host prompt and the manual commit skill, in concise prose:

1. Identify the repository whose operation is paused. In the host prompt, supply its
   name and actual checkout path from `DirtyRepo`. Use the existing `/sase_repo` access
   workflow when required, and operate on the checkout holding the paused operation.
   Express the path as context and require an explicit working directory; do not
   construct an unquoted executable `cd` command from it or change the provider
   process's global working directory.
2. Inspect live unmerged files, resolve their semantics, stage the resolution, and
   review the staged result before continuing. Verify the integrated content affected by
   the repair, including relevant automatically merged content; do not select checks
   solely from the old `changed_files` snapshot.
3. Consult applicable repository instructions and locally defined verification commands.
   Run the checks they require for these changes, in the working directory those checks
   specify. A mandatory all-changes gate remains mandatory. Do not skip it merely
   because the repair touches JSON or Markdown. A missing Justfile does not by itself
   establish that no gate exists; instructions may name another script or validation
   tool.
4. Confirm the selected command's definition and working directory belong to the
   target's verification procedure. Do not substitute a parent, launch-workspace, or
   sibling repository's gate just because the target has none. Account explicitly for
   task runners discovering ancestor configuration. A command outside the repository is
   appropriate only when applicable instructions explicitly delegate verification there
   and it actually checks the target content.
5. If there is no applicable gate, validate the resolved files directly. For structured
   data, check parsing and the relevant schema or invariants; for the reported JSON
   index shape, preserve distinct records and their associated counts, detect duplicate
   identities, and maintain the required ordering. Parse success alone is insufficient.
   For prose, review that intended content from both sides survives. Verify that no
   unresolved entries or conflict markers remain. Use available local validators when
   they cover the files; do not invent a full test suite to repair a document.
6. A required gate that fails or cannot run because of missing tools or dependencies is
   not an absent gate. Preserve the existing failure-handling workflow and report the
   actual verification problem. Review and stage fixes relevant to this repair; do not
   sweep unrelated or foreign edits into the resolution.
7. Briefly report the repository, the checks performed and their results, or why no
   applicable gate exists and which direct checks were used. Then follow the existing
   continue/resume sequence. Repeat the resolution and verification steps if continuing
   reveals further conflicts. Preserve final declarations and independent obligations
   for every other repository the turn actually changed.

Replace the existing assertion that duplicates are something “only lint or tests will
catch” with an accurate explanation: removing conflict markers is insufficient, and
semantic mistakes need checks that actually cover the merged content. This retains the
original rationale without insisting on a code test suite for structured data.

## Expected cases

| Situation                                                                          | Required result                                                                                            |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Research sidecar has only a Markdown/JSON repair and no applicable gate            | Direct syntax/content/invariant checks; no SASE gate borrowed as a fallback.                               |
| The same sidecar is nested below a SASE checkout with a Justfile                   | Same result; ancestor task-file discovery does not create a sidecar gate.                                  |
| A linked or external code repository documents its own gate                        | Run that gate for the changes according to its instructions and working directory.                         |
| A sidecar has a required schema, documentation, or code validator                  | Run that validator; sidecar status gives no exemption.                                                     |
| SASE itself is repaired, even for a documentation-only change                      | Preserve its all-changes verification rule, including `just check`.                                        |
| The turn changes both a sidecar and SASE                                           | Verify each under its own rules; direct sidecar validation does not discharge the SASE obligation.         |
| A required target gate is unavailable or fails                                     | Report the actual problem; do not relabel it as no gate or use an unrelated passing suite as evidence.     |
| Target instructions explicitly delegate validation to a tool in another repository | Follow the documented procedure that validates the target; the prohibition concerns fallback substitution. |

## Implementation

### 1. Correct the host conflict-repair prompt

Edit `src/sase/finalizers/commit_repair.py::_run_conflict_repair_turn` to include the
repository path and replace the unconditional gate paragraph with the contract above.
Keep the text compact enough to be actionable in a repair turn.

Retain the existing one-shot repair budget, host-instruction attribution,
paused-operation restrictions, artifact persistence, invocation options, usage
aggregation, and `finalizer_owned_turn()` context. Preserve
`sase stitch create --resume` and the final paragraph covering `/sase_final` and the
single follow-up commit. No changes to dispatch, checkpoint handling, automatic retries,
or runtime gate execution belong in this tale.

### 2. Align the manual skill

Edit only the source template `src/sase/xprompts/skills/sase_git_commit.md`. Add
verification between staging and continuing under “On Merge Conflict,” with the same
repository scope, direct-check fallback, ancestor-discovery caveat, failure distinction,
and reporting requirement. Update numbering and the repeated-step range so later
conflicts are also verified. Keep the manual wrapper's `sase_git_commit --resume`
instruction intact.

Do not create a shared runtime abstraction just to deduplicate two instruction blocks.
Regression tests should protect their common contract while allowing different wording
and the appropriate host/manual command spelling.

Generated provider `SKILL.md` files are deployment outputs. Preview the source rendering
with `sase skill init --diff` or `--dry-run`; do not hand-edit installed skills or
publish from an unlanded checkout. After host-owned landing, deploy from a clean
canonical tree with the documented `sase skill init --force` workflow and its chezmoi
apply step when needed. Record deployment as pending if that canonical-tree step is not
available to the implementing turn; do not claim installed guidance has changed before
it has.

### 3. Add focused regression coverage

Update `tests/test_finalizers_commit_repair_prompt.py`:

- Replace assertions demanding the defective unconditional wording and “only lint or
  tests” rationale with assertions for the new instruction contract.
- Exercise the actual `_run_conflict_repair_turn` provider invocation for representative
  main, linked, external, and sidecar `DirtyRepo` values. Include a nested research
  checkout and a checkout path containing spaces. Assert that the actual target name and
  path reach the provider and that the no-fallback/direct-validation instructions are
  unconditional across repository kinds.
- Preserve existing checks for scoped commit restrictions and finalizer ownership,
  including environment restoration on provider failure. Verify that the saved prompt
  artifact matches the provider's prompt using the existing artifact layout.

Extend `tests/main/test_init_skills_source_content.py` with a focused assertion of the
same verification contract in the shipped Git commit skill, including its position
between staging and continuing, the loop range, and retention of the wrapper resume.
Normalize whitespace where needed; avoid a complete prose snapshot that makes harmless
editing difficult. Existing source-content tests are the appropriate surface because
these instructions are the product behavior being corrected.

These tests validate delivered instructions, not LLM command choices. Review the
expected-case table against both instruction blocks; do not claim that a mocked provider
test proves a real agent never invokes an unrelated command. Live paid-model runs and
broad new integration infrastructure are not required for this text fix.

## Verification and completion

Before implementing, read the current `generated_skills.md` and `lint_and_test.md`
reference memories through `/sase_memory_read`. Bootstrap this checkout with
`just install` if its isolated environment needs it.

Use the changed prompt and skill tests as the focused regression checks:

```bash
.venv/bin/python -m pytest tests/test_finalizers_commit_repair_prompt.py tests/main/test_init_skills_source_content.py
```

Run `just check` in the SASE repository before completing implementation: this time SASE
source and test files really are changed, so its verification obligation applies.
Respect the memory's monitor handoff for a long-running gate and its `just check-full`
escalation/landing rules; do not automatically choose the exhaustive lane for a focused
prompt edit. Preview generated skills without deploying them from the working tree.

Review the final diff against every expected case, verify the old unconditional
instruction is gone, and confirm no repair lifecycle or global configuration changed.
Finish through `/sase_final` with the implementation, verification result, and accurate
skill-deployment status. This planning turn creates only this scratch plan and submits
it through `sase plan propose`; implementation begins after the plan handoff.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact                                                | Why                                                                      | Uses |
| -------- | ------------------------------------------------------- | ------------------------------------------------------------------------ | ---: |
| cited-by | [agent:bbugyi200.athena.research.1m.cdx.f0.f0--code][1] | prompt reference @plan:202609/repository_scoped_conflict_verification.md |    1 |

[1]:
  https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.research.1m.cdx.f0.f0.md

<!-- sase:referenced-by:end -->
