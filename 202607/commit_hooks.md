---
create_time: 2026-07-11 08:14:40
status: done
prompt: 202607/prompts/commit_hooks.md
tier: tale
---
# Plan: Before/After Commit Hooks

## Goal

Replace the project-local `precommit_command` setting with a nested `commit_hooks.before` setting and add
`commit_hooks.after`, whose command runs only after `sase commit` has successfully created and, where applicable, pushed
its commit. Configure the chezmoi repository to apply committed source changes automatically with
`chezmoi update -a --force`.

## Configuration contract and migration

- Replace the top-level default and JSON-schema entry for `precommit_command` with a closed `commit_hooks` object
  containing string-valued `before` and `after` fields, both defaulting to the empty string. This lets normal deep-merge
  semantics combine a global value for one phase with a project-local value for the other.
- Treat the rename as an intentional configuration migration rather than retaining a hidden compatibility alias: active
  code, tests, documentation, and maintained repository configs will use only `commit_hooks.before` and
  `commit_hooks.after`.
- Migrate active configuration in every maintained location found by the cross-repository audit:
  - the SASE repository's `just fix` command moves to `commit_hooks.before`;
  - the chezmoi-managed global SASE config's `sase_git_fix` command moves to `commit_hooks.before`;
  - the sase-telegram repository's `just fmt` command moves to `commit_hooks.before`;
  - a new root-level `sase.yml` in the chezmoi repository sets `commit_hooks.after` to `chezmoi update -a --force`.
- Leave archived SDD research, tales, prompts, and bead history unchanged: they document the historical field and are
  not live configuration or current product documentation. Finish with a scoped stale-reference search across executable
  code, tests, current docs, and active config files.

## Commit-hook execution

- Refactor the current precommit-command runner into phase-aware commit-hook helpers that read the nested configuration,
  execute in the repository root, capture output, and report phase-specific progress and failure diagnostics. Preserve
  the existing bounded stdout/stderr tail on failures.
- Keep `commit_hooks.before` at the existing pre-dispatch point, after bead/plan mutations and before diff capture and
  VCS dispatch. Preserve the current before-hook behavior used by project memory initialization under the renamed key.
- Run `commit_hooks.after` for commit-producing workflows (`create_commit` and `create_pull_request`) immediately after
  successful VCS dispatch, because provider success is the point at which the normal Git path has completed its push. Do
  not run it for `create_proposal`, which saves a diff without creating a commit.
- Integrate the after-hook with commit checkpoints. Record it as a completed step only after it succeeds, keep the
  checkpoint when it fails, return a nonzero result with an explicit message that the commit may already be pushed, and
  direct the user to `sase commit --resume` instead of creating a duplicate commit. On conflict resume, run the
  after-hook only after `finalize_commit` successfully completes its push, and skip it if the checkpoint already records
  success.
- Keep post-dispatch tracking and checkpoint deletion ordered after the after-hook so a failed command cannot be
  reported as an entirely successful commit workflow. Account for the narrow crash window with at-least-once semantics:
  a process that dies after the external command succeeds but before checkpoint persistence may rerun that command on
  resume, so configured after-hooks should be safe to repeat; the requested chezmoi command is repeatable.

## Tests and documentation

- Replace the focused precommit runner tests with commit-hook tests covering nested config lookup, empty hooks, phase-
  specific diagnostics, output-tail behavior, and failing commands.
- Extend workflow tests to pin ordering and gating: before runs before dispatch; after runs after successful dispatch;
  proposals and failed dispatches do not run after; after-hook failure preserves the checkpoint and returns failure;
  normal completion records the after step; resume runs after only following successful finalization and does not rerun
  a completed hook.
- Update shared test fixtures and memory-init tests for the renamed before-hook API/config shape, preserving the
  existing guarantee that a failing before-hook aborts before staging or committing.
- Rewrite the configuration reference and current commit-workflow/init documentation to show the nested YAML, explain
  phase timing, proposal behavior, push/resume semantics, failure consequences, and repeatability expectations. Update
  current architecture/blog/infographic source text that still names the precommit stage so it reflects separate before
  and after hooks without regenerating an unembedded stale PNG.

## Verification

- Install the workspace dependencies first, then run focused commit-hook, checkpoint/resume, workflow, config, and
  memory- init tests while iterating.
- Run `just check` in the SASE repository after all SASE file changes, plus the appropriate checks in any linked repo
  whose source/config change requires them (config-only repositories can be validated by parsing their YAML and
  inspecting the merged values).
- Validate the final merged configuration in each affected repository and confirm that the chezmoi root resolves
  `commit_hooks.after` exactly to `chezmoi update -a --force` while inherited/global `commit_hooks.before` values still
  deep-merge correctly.
