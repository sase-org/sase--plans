---
create_time: 2026-07-11 09:59:39
status: done
prompt: 202607/prompts/fold_init_trigger_changes.md
tier: tale
---
# Fold `sase init` trigger changes into generated commits

## Context

`sase memory init` captures the repository's dirty state before it writes generated memory and provider files. It
currently treats every pre-existing `memory/` edit as foldable, prompts for a subject, adds `docs(memory):` when the
subject is not already conventional, adds the `SASE_TYPE=memory` footer, and commits those edits together with the
generated files. Every dirty path outside `memory/` is treated as foreign and causes the automatic commit to fail.

That path-only rule rejects a valid and common case: an edited `AGENTS.md` is the source of planned `CLAUDE.md`,
`GEMINI.md`, `QWEN.md`, and `OPENCODE.md` updates. If `AGENTS.md` is the only pre-existing worktree change, the
generated shims and their source belong in one user-described commit, but the current deploy path writes the shims and
then refuses to commit because `AGENTS.md` is outside `memory/`.

## Desired behavior

- When every pre-existing dirty path is either an existing foldable `memory/` edit or an explicit source of a generated
  change produced by the selected memory initializer, prompt once for the user's commit subject and commit the source
  edits and generated outputs together.
- Treat an edited root or nested `AGENTS.md` as a foldable source only when that invocation actually plans provider-shim
  writes or deletions derived from it. Do not whitelist `AGENTS.md` globally.
- Preserve the current message policy: accept an already-conventional subject, otherwise add the appropriate
  `docs(memory):` prefix, and always attach SASE-owned commit metadata such as `SASE_TYPE=memory` automatically.
- Preserve the current non-interactive `--message` path and the safe refusal when no message can be obtained.
- If even one dirty path is unrelated to the generated work, refuse the entire automatic commit; do not partially stage
  or commit the foldable subset.
- Preserve `--no-commit`, hook, pull/rebase, push, no-upstream, and no-op behavior.

## Implementation plan

1. Model generated-change provenance in the memory initialization result.
   - Extend the provider-shim planning/application plumbing so each changed shim set retains the `AGENTS.md` path whose
     content produced it.
   - Propagate only sources with an actual dependent write or deletion into the project deployment result. This keeps an
     unrelated dirty `AGENTS.md` from becoming automatically committable and naturally supports both root and nested
     agent documents.
   - Keep generated/managed `AGENTS.md` outputs distinct from untouched source documents so a path that the initializer
     overwrites is not incorrectly treated as a user-authored trigger solely because shims also depend on it.

2. Generalize pre-init dirty-state resolution at deployment time.
   - Combine the existing `memory/` dirty set with dirty paths matching the explicit source paths from step 1 to form
     the fold set.
   - Reclassify every remaining path as foreign and retain the all-or-nothing refusal. Continue handling staged,
     unstaged, untracked, deleted, and rename/copy paths through the existing porcelain parser.
   - Stage the approved source paths alongside the generated writes/deletions and existing memory paths, with the
     existing resolved-path deduplication.

3. Generalize the fold prompt and input plumbing.
   - Update the prompt panel and error text to describe source/trigger edits rather than only `memory/` edits, while
     listing exactly what will be folded.
   - Route the bare `sase init` coordinator's injected input function and stdin/TTY object through the memory handler
     into project deployment. This makes the second commit-message prompt behave consistently after the initial `y/N/d`
     prompt and keeps `sase init --all` and tests from falling back to global `input()`/`sys.stdin`.
   - Retain the direct `sase memory init --message` and `sase init memory --message` behavior, and update their help
     text to cover all foldable source changes.

4. Keep commit-message construction centralized.
   - Reuse the conventional-header validation and automatic `docs(memory):` fallback for a user-supplied subject.
   - Reuse the SASE commit-tag helper so callers cannot omit or duplicate the generated `SASE_TYPE=memory` footer;
     preserve already valid conventional headers.

5. Add focused regression coverage.
   - Cover the reported root `AGENTS.md` case: four changed provider shims, an interactive subject, staging of the
     source plus generated files, and one commit with the conventional prefix and SASE footer.
   - Cover a nested `AGENTS.md` source and its sibling provider shims.
   - Cover `--message` in a non-TTY invocation and the bare `sase init` two-prompt flow using the coordinator's injected
     input/TTY.
   - Cover mixed dirty state (`AGENTS.md` plus an unrelated source file) to prove the command still refuses before
     hooks, staging, or commit.
   - Cover a dirty `AGENTS.md` with no dependent generated action so it is not treated as a trigger, plus preservation
     of the existing dirty-`memory/`, empty/EOF/cancel, conventional-subject, `--no-commit`, and no-op cases.
   - Add or adjust pure model/classification tests for source-path matching, including rename/copy boundary cases, and
     update parser-help assertions.

6. Validate the completed change.
   - Run the targeted init-memory, onboarding, git-state, commit-message, agent-doc, and parser test modules while
     iterating.
   - Run `just install` as required for the ephemeral workspace, then run the repository-mandated `just check` before
     handoff.

## Non-goals

- Do not automatically commit arbitrary dirty files merely because `sase init` also has generated work.
- Do not infer provenance from filename patterns alone or broaden the fold rule to every `AGENTS.md` in the repository.
- Do not change memory-file contents, provider-shim format, commit/push ordering, or the behavior of unrelated
  `sase init` subcommands.
