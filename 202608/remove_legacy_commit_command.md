---
tier: tale
title: Remove the legacy sase commit command
goal:
  Make sase stitch create the only CLI entry point for tracked VCS dispatch while
  preserving its behavior and aligning every active caller, skill, test, and
  documentation surface.
size: medium
proposed_by: bbugyi200.athena.0bt
create_time: 2026-08-23 10:41:28
status: wip
---

# Plan

## Context and boundaries

`sase stitch create` is already the canonical spelling, but the old top-level
`sase commit` command remains registered as an alias. The alias is not isolated: its
parser module owns the shared create flags, its handler module owns the canonical create
implementation, the root entry point has a separate dispatch branch, and tests still
exercise both spellings. Active Mercurial/Patch skills and several current or draft docs
also direct users to the legacy command.

Remove the command rather than warning or redirecting: after this tale, `sase commit`
must be absent from root help, completion/spec output, the lazy command registry, and
runtime dispatch, and invoking it must fail through the normal unknown-command argparse
path. Preserve concepts that merely use the word “commit” and are not this CLI alias,
including `CommitWorkflow`, `create_commit`, `SASE_COMMIT_METHOD`, the `#commit`
xprompt, `builtin@commit`, finalizer commit decisions, commit tags/statistics, and Git
or Mercurial commits.

This is one atomic, medium tale: parser ownership, behavior tests, generated-skill
sources, and user guidance must move together so there is no intermediate contract
mismatch. No feature flag or compatibility window is appropriate because the requested
outcome is complete removal of an already-deprecated alias.

Protected-memory boundary: the audited `sase/memory/generated_skills.md` note still
describes a `sase commit` CLI/skill synchronization contract. The current prompt does
not explicitly authorize editing a SASE memory file, so do not modify it from this plan.
Use `/sase_new_task` to record that memory correction unless the user separately grants
explicit permission in the implementation conversation; with permission, update the
canonical note and run the mandatory `sase memory init` regeneration workflow.

## Implementation

1. Make stitch-create own its parser and execution path.
   - Remove `commit` from `src/sase/main/parser.py`'s lazy command inventory and remove
     the top-level `args.command == "commit"` branch from `src/sase/main/entry.py`.
   - Remove `register_commit_parser` from the full registrar catalog. Keep `restore` and
     `revert` registered and routed normally.
   - Move the create-option builder and removed-`-f/--file` compatibility action out of
     `parser_commit.py` into the canonical stitch parser or a narrowly named
     stitch-create parser helper. Leave `parser_commit.py` responsible only for the
     still-supported restore/revert parsers, or rename/split it if that yields clearer
     ownership without introducing a compatibility import.
   - Move `handle_commit_command` and its create-only helpers out of `commit_handler.py`
     into a canonical stitch-create handler (renamed accordingly), and have
     `handle_stitch_command` dispatch `stitch create` directly to it.
     `commit_handler.py` should retain only restore/revert behavior if the file remains.
   - Update internal comments/docstrings and the lightweight `commit_methods` module
     description so none imply two CLI spellings. Update the SDD plan-commit caller's
     handler comment to name the canonical message-file behavior, without changing its
     `sase stitch create` subprocess contract.

2. Convert the behavioral tests to the sole canonical surface and lock in removal.
   - Migrate the comprehensive flag-to-payload and `--resume` suites from a synthetic
     top-level commit parser to `sase stitch create`, renaming test modules/helpers and
     patch targets where useful so their names describe the public command under test.
     Preserve coverage for message-file lifecycle, method/environment precedence,
     excludes, bead metadata, conflict exit codes, resume idempotence, and workflow
     construction.
   - Delete the alias-equivalence assertion in `tests/main/test_stitch_parser.py` and
     replace it with explicit negative coverage: the root command inventory/help does
     not contain `commit`, parser narrowing does not treat it as a known command, and a
     real `sase commit` parse/invocation exits with the normal unknown-command status.
   - Keep and strengthen the lazy-import assertion that building the stitch parser uses
     only the dependency-free method constants rather than importing the workflow.
     Update incidental test docstrings that describe the removed CLI while leaving
     finalizer IDs, xprompt names, and generic commit-domain fixtures intact.

3. Migrate active callers, generated-skill sources, glossary wording, and docs.
   - Change the Gemini-only `src/sase/xprompts/skills/sase_hg_commit.md` examples,
     conflict recovery, description, and resume command to `sase stitch create`. Update
     the Patch and ChangeSpec skill sources to recommend the canonical command for
     tracked changes and STITCHES management. Extend
     `tests/main/test_init_skills_source_content.py` so both Git and Mercurial skill
     sources reject the legacy spelling and the Mercurial source explicitly exercises
     the canonical spelling. Preview generated output with `sase skill init --diff`; do
     not deploy global generated skills from the dirty/unmerged implementation tree.
   - Update the Stitch glossary source in `sase/sase.yml` to identify
     `sase stitch create` as the tracked VCS path and remove the claim that the old
     command still exists. Do not change the meaning of a Stitch or rename real commits.
   - Replace active or draft user guidance in `docs/agent_images.md`,
     `docs/configuration.md`, `docs/vcs.md`, `docs/xprompt.md`, and the draft workflow
     and Patch blog posts. While touching the stitch-create CLI table, correct its stale
     removed `-f/--file` row to the actual repeatable `-x/--exclude` and hidden
     `--only-file` contract rather than perpetuating an unrelated parser mismatch.
     Reword noun-only phrases such as “no sase commit” when they could be mistaken for
     the removed command.
   - Update the commit-workflow infographic prompt/style brief to route through
     `sase stitch create`, regenerate or relabel the unembedded PNG with the imagegen
     workflow so the bitmap no longer advertises the removed command, and refresh its
     critique/status text against the resulting asset. Preserve genuinely historical
     release notes such as `CHANGELOG.md`; historical provenance may say the command
     existed, but no current instruction or architecture statement may claim it is
     callable.

## Verification and acceptance criteria

1. Run focused tests for the stitch parser/handler, migrated create and resume suites,
   parser narrowing/root help, generated skill source contracts, glossary behavior, and
   any docs/image validation affected by the edits. Exercise both a successful
   `sase stitch create` handler path under mocks and the rejected `sase commit` CLI
   path.
2. Run `sase skill init --diff` to verify rendered provider skill output uses the
   canonical command without writing global destinations. Inspect the regenerated PNG at
   original detail to confirm the visible label is `sase stitch create`, is legible, and
   did not damage the workflow diagram.
3. Audit the tree with targeted `rg` searches for the exact `sase commit` spelling,
   `register_commit_parser`, `handle_commit_command`, and the removed root-registry/
   dispatch shapes. Classify every remaining match: only protected memory awaiting
   separately authorized regeneration and deliberately historical provenance may remain;
   active source, skills, tests, help, docs, and binary-visible labels must be clean.
   Also confirm canonical `sase stitch create` references remain intact.
4. Because this repo uses ephemeral workspaces, run `just install` before verification,
   then run `just check`. If the scoped selector escalates, reports unusual selection,
   or the touched files fall in the broadening set, run `just check-full` only through
   `/sase_monitor` with the required `TESTING`/`TESTED` statuses and a concrete
   follow-up action.

The tale is complete when the old command is neither discoverable nor executable, all
tracked VCS behavior remains available through `sase stitch create`, active automation
and guidance use only that spelling, the canonical path retains its existing payload,
resume, and exit-code behavior, and all required validation passes.
