---
create_time: 2026-07-09 15:11:47
status: done
prompt: 202607/prompts/remove_cd_vcs_xprompt.md
tier: tale
---
# Remove the `#cd` VCS XPrompt Workflow

## Goal

Remove the built-in `#cd` directory workspace workflow completely. After this change, SASE should no longer ship,
register, complete, document, or test `#cd:<path>` as a workspace/VCS xprompt. Existing project work should use the
remaining workspace references, primarily `#git:<ref>` or plugin-provided refs such as `#gh:<ref>`.

## Current Shape

`#cd` is not just a memory-note row. It is implemented as a built-in workspace provider and an embeddable workflow:

- `src/sase/xprompts/cd.yml` defines the `tags: vcs` workflow.
- `src/sase/workspace_provider/plugins/cd_workspace.py` implements local directory ref resolution and `SASE_CD_*`
  preallocation metadata.
- `pyproject.toml` registers the `cd` workspace provider entry point.
- Launch code treats `cd` as the only non-workspace workflow, so `#cd` skips numbered workspace allocation while normal
  refs defer allocation to the shared executor.
- Prompt parsing has one hard-coded `{"cd"}` fallback for `#workflow@ref` shorthand recognition.
- User-facing surfaces document and advertise `#cd` in root/run help, README, docs, smoke notes, and the generated
  `sase_run` skill source.
- Tests cover the workflow file, provider, parsing, launch resolution, spawn env, completion, MRU behavior, and
  docs/help examples.
- `memory/xprompts.md` lists `#cd:<path>` in its project-task launch table.

## Implementation Plan

1. Remove the shipped workflow and provider.
   - Delete `src/sase/xprompts/cd.yml`.
   - Delete `src/sase/workspace_provider/plugins/cd_workspace.py`.
   - Remove the `cd = "sase.workspace_provider.plugins.cd_workspace:CdWorkspacePlugin"` entry from
     `[project.entry-points."sase_workspace"]` in `pyproject.toml`.
   - Confirm `sase xprompt list` and xprompt completion no longer discover a `cd` workflow from package defaults.

2. Simplify runtime paths that only existed for `#cd`.
   - Remove `NON_WORKSPACE_WORKFLOW_TYPES = {"cd"}` and the special `is_non_workspace_workflow()` behavior, or reduce
     callers so no `cd` string remains.
   - In ACE launch and CLI launch paths, keep `%wait` deferred-workspace behavior, but remove the `#cd` fixed-workspace
     branch. Normal workspace refs should continue to resolve metadata without pre-claiming a numbered workspace.
   - Remove the hard-coded `{"cd"}` fallback in `src/sase/xprompt/_parsing_vcs_refs.py`.
   - Remove `SASE_CD_WORKSPACE_DIR` and `SASE_CD_WORKSPACE_NUM` from runtime env contracts and provider-specific
     fallback lists unless a check is intentionally preserved as backward-compatible stale-env scrubbing. If any legacy
     scrub remains, document it as legacy cleanup rather than active `#cd` support.

3. Remove user-facing references.
   - Update `memory/xprompts.md` to remove the `#cd:<path>` row and any text implying directory-only launch support.
   - Run `sase memory init --no-commit` after the memory edit so derived memory indexes stay consistent.
   - Replace root/run help examples in `src/sase/main/parser.py` and `src/sase/main/parser_commands.py` with a non-`#cd`
     example, likely `#git:home` or a plain prompt that relies on default `#git:home` normalization.
   - Update `src/sase/xprompts/skills/sase_run.md` to list only remaining workspace refs. Because this is a
     generated-skill source, run `sase skill init --force` afterward, then apply the generated skill update workflow
     required by the repo.
   - Update README, smoke README, docs, and blog posts so no discoverable docs recommend or describe `#cd`. Rewrite
     file/path completion docs to say prompt-selected workspace refs come from registered workspace providers, without a
     `#cd` precedence rule.
   - Update plugin/workspace docs to describe only the remaining built-in `bare_git` workspace provider and optional
     provider plugins.

4. Update or remove tests.
   - Delete tests whose sole purpose is `#cd` feature coverage: `tests/workspace_provider/test_cd_workspace.py`,
     `tests/test_cd_workflow_loader.py`, `tests/test_cd_ref_resolution.py`, `tests/test_cd_spawn_env.py`, and the
     dedicated `test_cd_*launch*` suites.
   - Rewrite parser, completion, MRU, xprompt parsing, prompt-file completion, launch-name, deferred-workspace, and ACE
     launch tests that currently use `#cd` as a convenient fixture to use `#git`/fake registered workspace metadata
     instead.
   - Remove `SASE_CD_*` expectations from runtime env cleanup tests.
   - Keep generic workspace-provider tests intact; they should exercise registered plugin behavior without the deleted
     `cd` provider.
   - Add at least one negative assertion that `#cd` is no longer treated as a workspace reference or built-in xprompt,
     so stale registrations do not silently return.

5. Validate reference cleanup.
   - Run targeted searches before final validation:
     `rg '#cd|SASE_CD|CdWorkspacePlugin|cd_workspace|NON_WORKSPACE_WORKFLOW_TYPES'` should return no active references.
     If a legacy cleanup reference remains, it should be explicitly named and justified in code/docs.
   - Also search docs for phrases such as `plain directory`, `Directory Workflow`, and `directory-backed` to catch
     prose-only remnants.

6. Run required validation.
   - Run `just install` first because numbered workspaces are ephemeral.
   - Run regeneration/formatting in the correct order: `sase memory init --no-commit`, `sase skill init --force`, any
     required generated-skill deployment step, then `just fmt`.
   - Run focused tests while iterating, especially xprompt parsing, launch, workspace provider, CLI help, and completion
     tests.
   - Finish with `just check`, as required for repo file changes.

## Risks and Decisions

- This is a breaking behavior change for prompts that launch directly in an arbitrary local directory. That is
  intentional per the request; the replacement is to use registered workspace refs.
- `#cd` currently doubles as a test fixture for “non-workspace” behavior. Since no other built-in workflow has that
  semantics, the implementation should remove the special path rather than preserve a dead abstraction with an empty
  set.
- Some generated live skill files may live outside this repo. The repo source change should still follow the
  generated-skill workflow so the live instructions do not continue advertising `#cd`.
- The phrase “all references” should mean all feature-specific `#cd`/`SASE_CD` references. Ordinary shell `cd` text is
  unrelated and should not be churned.

## Acceptance Criteria

- `#cd` is absent from package xprompt discovery, completion, help examples, docs, memory, and plugin entry points.
- `#cd` is not recognized as a workspace ref by the built-in runtime unless an external third-party plugin independently
  registers that workflow name.
- No source, test, docs, memory, or generated-skill source references remain for the removed built-in `#cd` workflow.
- `just check` passes.
