---
tier: tale
title: Migrate the sase commit CLI command to sase stitch create
goal:
  sase stitch create is the canonical command for dispatching a commit, proposal, or
  pull request, with sase commit still accepted as a silent legacy alias, and the
  sase_git_commit skill, wrapper script, in-repo callers, runtime guidance strings, and
  docs all naming the canonical spelling.
size: medium
proposed_by: bbugyi200.athena.xq
create_time: 2026-08-10 19:19:03
status: wip
---

# Plan: Migrate `sase commit` to `sase stitch create`

## Background

Epic `sase-j8` renamed `sase vcs` to `sase stitch` (phase `sase-j8.1`, commit
`83e3d3c27`). That rename established the house pattern for CLI migrations in this repo:

- The canonical parser module is named for the new spelling (`parser_stitch.py`), and
  the old module (`parser_vcs.py`) becomes a one-line re-export facade.
- The legacy spelling stays accepted **silently** — `aliases=["vcs"]` on the subparser,
  both names in `parser.py`'s `_COMMAND_REGISTRARS`, and `entry.py` dispatching
  `{"stitch", "vcs"}`. There is no runtime deprecation warning.
- Help text and docs name the legacy spelling as a deprecated alias.

This tale extends `sase stitch` with a third subcommand, `create`, which takes over from
the top-level `sase commit` command.

The structural difference from `sase-j8.1` matters: `vcs` -> `stitch` was a flat rename
that argparse expresses directly as a subparser alias. `commit` -> `stitch create` moves
a **top-level command into a nested subcommand position**, which argparse cannot express
as an alias. The two spellings must therefore share their option definitions explicitly.

### Current shape

- `src/sase/main/parser_commit.py` — `register_commit_parser()` registers the top-level
  `commit` parser with 11 flags (`-m/-M/-f/-n/-b/-B/-c/-p/-s/-t/-r`). The same module
  also registers the unrelated `restore` and `revert` parsers.
- `src/sase/main/commit_handler.py` — `handle_commit_command(args) -> NoReturn`.
- `src/sase/main/entry.py:175` — dispatches `args.command == "commit"`.
- `src/sase/main/parser.py:101` — `"commit"` entry in `_COMMAND_REGISTRARS`.
- `src/sase/main/parser_stitch.py` — `stitch` parser with `list` and `log` subparsers,
  `metavar="{list,log}"`, and `set_defaults(stitch_subcommand="list")`.
- `src/sase/main/stitch_handler.py` — `_HANDLERS = {"list": ..., "log": ...}` dispatched
  by `handle_stitch_command()`, which prints `Usage: sase stitch {list,log}` on an
  unknown subcommand.

## Goal

`sase stitch create` is the canonical spelling for dispatching a commit, proposal, or
pull request. `sase commit` keeps working as a silent legacy alias. The
`/sase_git_commit` skill, the `sase_git_commit` wrapper script, in-repo callers, runtime
guidance strings, and the docs all name the canonical spelling.

## Non-goals

- Do **not** rename the `/sase_git_commit` skill or restructure its body. Change only
  the `sase commit` spellings inside it. (The user has separate plans for that skill.)
- Do **not** change any commit flag names, defaults, semantics, or exit codes. This is a
  command-path migration only.
- Do **not** touch `sase restore` or `sase revert`, beyond the one call site inside
  `restore.py` that shells out to the commit command.
- Do **not** rewrite historical records: `CHANGELOG.md`, `docs/blog/posts/*`, and
  `docs/images/*.prompt.md` / `*.critique.md` describe what was true when written.
- Do **not** emit a runtime deprecation warning for `sase commit`. It matches the silent
  `sase vcs` precedent, and a warning on stdout/stderr would pollute output that agent
  hooks and the commit wrapper parse.

## Design decisions

### D1. `sase commit` survives as a top-level legacy alias

`sase commit` is invoked by the `sase_git_commit` wrapper, the `/sase_hg_commit` skill,
`sase axe` SDD plan commits, `sase restore`, and every agent runtime's muscle memory.
Removing it would break in-flight agents mid-run. Keep `register_commit_parser`
registering the top-level `commit` command, with its help text marking it as a legacy
alias for `sase stitch create`.

### D2. One shared option-registration helper

Extract the flag definitions into `add_commit_create_options(parser)` in
`parser_commit.py`. Both the top-level `commit` parser and the new `stitch create`
subparser call it, so the two spellings cannot drift.

### D3. Keep `sase stitch` startup cost flat (important)

`register_commit_parser()` currently does, at registration time:

```python
from sase.workflows.commit.workflow import METHOD_ALIASES, VALID_METHODS
```

`src/sase/workflows/commit/__init__.py` eagerly does
`from .workflow import CommitWorkflow`, so **any** import under
`sase.workflows.commit.*` — including the otherwise-trivial `workflow_types.py` — pulls
the full commit workflow dependency chain. Measured cost: **~55 ms**.

Today only `sase commit` pays that. `sase stitch list` and `sase stitch log` do not,
because `parser.py`'s `parser_only_hint()` narrowing imports `parser_stitch` alone. If
`parser_stitch` naively imports the shared option helper, both stitch read paths inherit
a ~55 ms startup regression.

This repo already has the fix pattern and a test that enforces it:
`src/sase/main/parser_artifact.py` takes its argparse `choices` from the lightweight
`sase.core.artifact_file_types`, guarded by
`tests/main/test_parser_narrowing.py::test_artifact_parser_uses_lightweight_file_types_module`.

Mirror it: add a leaf module `src/sase/commit_methods.py` holding `VALID_METHODS` and
`METHOD_ALIASES`, have `src/sase/workflows/commit/workflow_types.py` re-export from it
for backward compatibility, and import from the leaf in `parser_commit.py`. Then add an
import-isolation test asserting `sase.workflows.commit` stays out of `sys.modules` after
importing `parser_stitch`.

(The alternative — making `sase/workflows/commit/__init__.py` lazy with a PEP 562
`__getattr__` — is more invasive and risks subtle breakage in the many callers doing
`from sase.workflows.commit import CommitWorkflow`. Prefer the leaf module.)

### D4. Dispatch through `stitch_handler`

`stitch_handler._HANDLERS` values return `int` and are invoked as
`sys.exit(handler(args))`. `handle_commit_command` is `NoReturn` (it calls `sys.exit`
itself). Route `create` explicitly rather than forcing it into the `int`-returning
handler table, so mypy stays happy and the control flow stays readable.

`entry.py` needs no new branch: `args.command == "stitch"` already routes to
`handle_stitch_command`.

### D5. Argparse collision check (verified safe)

`stitch create`'s dests are `message`, `message_file`, `files`, `name`, `bug_id`,
`do_not_close_bead`, `checkout_target`, `parent`, `status`, `method`, `resume`.
`stitch list`'s are `color`, `current_only`, `format`, `no_fetch`, `repos`, `sort`. No
overlap, so `parser.py`'s `_default_list_subcommands()` / `_copy_parser_defaults()`
(which copy `list`'s defaults onto the parent `stitch` parser) cannot clobber a `create`
dest.

Short flags that repeat across sibling subparsers are fine because they live on
different subparsers: `-r` is `--resume` on `create` but `--repo` on `log`; `-b` is
`--bug-id` on `create` but `--branch` on `log`; `-c` is `--checkout-target` on `create`
but `--color` on `list`/`log`. Add tests pinning this.

Bare `sase stitch` must still print the delegation notice and run `list`.

## Implementation steps

### 1. Lightweight commit-method constants

- Create `src/sase/commit_methods.py` with `VALID_METHODS` and `METHOD_ALIASES` (moved
  verbatim from `src/sase/workflows/commit/workflow_types.py`), plus a module docstring
  noting it must stay dependency-free so argparse `choices` can import it cheaply.
- In `src/sase/workflows/commit/workflow_types.py`, import and re-export both names so
  existing importers (including
  `from sase.workflows.commit.workflow import METHOD_ALIASES, VALID_METHODS` in
  `commit_handler.py`) keep working unchanged.
- In `src/sase/main/parser_commit.py`, replace the in-function heavy import with a
  module-level `from sase.commit_methods import METHOD_ALIASES, VALID_METHODS`.

### 2. Shared option helper

In `src/sase/main/parser_commit.py`, extract `add_commit_create_options(parser)`
containing all 11 current flags, byte-for-byte identical help strings. Have
`register_commit_parser()` create the top-level `commit` parser and call the helper. Set
its `help=` to mark it a legacy alias for `sase stitch create`.

### 3. `sase stitch create` subparser

In `src/sase/main/parser_stitch.py`:

- Import `add_commit_create_options` from `parser_commit`.
- Add a `create` subparser to `stitch_sub` with help/description explaining it
  dispatches a commit, proposal, or PR through the configured VCS provider, and calls
  the shared helper.
- Update `metavar="{list,log}"` -> `metavar="{create,list,log}"`.
- Extend the `stitch` parser description to mention `create` alongside the existing note
  that `sase vcs` remains a legacy alias.
- Keep `set_defaults(stitch_subcommand="list")` untouched.

### 4. Dispatch

In `src/sase/main/stitch_handler.py`:

- Route `create` to `handle_commit_command` (imported lazily inside the function, so
  `sase stitch list` never imports the commit workflow at dispatch time either).
- Update the unknown-subcommand usage line to `Usage: sase stitch {create,list,log}`.

In `src/sase/main/entry.py`, leave the `commit` branch working and add a short comment
marking it a legacy alias, matching the existing `# legacy command alias` comments on
the `patch`/`changespec` and `stitch`/`vcs` branches.

### 5. Wrapper script

- `src/sase/scripts/sase_git_commit`: change the delegation from `sase commit "$@"` to
  `sase stitch create "$@"`, and update the three header/inline comments that name
  `sase commit`.
- `tests/test_sase_git_commit_wrapper.py`: the fake `sase` stub currently asserts
  `[[ "$1" == "commit" ]]`. Update it to assert `$1 == "stitch"` and `$2 == "create"`,
  and adjust any downstream assertion on the recorded argv.

### 6. In-repo callers

- `src/sase/axe/run_agent_exec_plan_sdd.py`: `cmd = ["sase", "commit", "-M", msg_path]`
  -> `["sase", "stitch", "create", "-M", msg_path]`; update the two docstring mentions
  and the warning log message.
- `src/sase/ace/restore.py`: `["sase", "commit", base_name]` ->
  `["sase", "stitch", "create", base_name]`, plus the console message and the three
  comment/docstring mentions. **Preserve the existing (broken) argument shape** — see
  the follow-up bead below; do not opportunistically fix it here.
- Comment-only mentions that should also be updated for consistency:
  `src/sase/ace/hooks/persistence.py`, `src/sase/ace/scheduler/hook_checks.py`,
  `src/sase/llm_provider/muse.py`, `src/sase/core/shell.py`,
  `src/sase/default_config.yml` (the `commit.message.allowed_types` comment).

### 7. Runtime guidance strings

These strings are printed to agents and tell them what to re-run, so they must name the
canonical spelling:

- `src/sase/workflows/commit/workflow_publication.py` — 6 mentions of
  `sase commit --resume`.
- `src/sase/workflows/commit/workflow.py` — 2 mentions.
- `src/sase/workflows/commit/workflow_resume.py` — 2 mentions (one is
  `re-run sase commit --resume`, one is `Re-run sase commit from scratch.`).
- `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` — 1 mention.
- `src/sase/workflows/commit/bead_hooks.py` — the bead note text
  ``Auto-closed by `sase commit` after {landed}``.

Update the matching assertions in `tests/test_commit_workflow_publication.py` (3) and
`tests/test_commit_bead_hooks.py` (1).

### 8. `/sase_git_commit` skill source — minimal edit

In `src/sase/xprompts/skills/sase_git_commit.md`, change **only** the `sase commit`
spellings to `sase stitch create`. There are 6, at roughly:

- frontmatter `description:` — "Commit changes using sase commit for git-based VCS"
- body intro — "delegates to `sase commit`"
- step 2 — "`sase commit` **rejects** a message whose subject line is not a conventional
  header"
- step 3 — "`sase commit` commits first, rebases automatically"
- step 5 — "`sase_git_commit` delegates to `sase commit`, which normally pushes"

Do not rename the skill, reorder sections, or touch anything else.

`tests/main/test_init_skills_sources.py::test_git_commit_skill_invokes_observable_wrapper`
already asserts `"sase commit -M" not in body` and `"sase commit --resume" not in body`;
those still hold. Add positive assertions that the body names `sase stitch create` and
still routes through the `sase_git_commit` wrapper, and a negative assertion that the
raw `sase stitch create -M` form is not presented as the command an agent should run.

### 9. Docs

Update the canonical spelling in:

- `docs/cli.md` — the `sase commit` row in the "Review And Delivery" table, and the
  command list around line 83.
- `docs/vcs.md` — rename the `### sase commit` section to `### sase stitch create`
  (update the anchor and any in-repo links to it), the three "Common CLI forms"
  examples, and the remaining mentions at the `--resume` / troubleshooting spots.
- `docs/commit_workflows.md` — 15 mentions, including the pipeline diagram
  (`sase commit -> CommitWorkflow -> VCS provider -> tracked output`) and the
  resume-flow section.
- `docs/configuration.md`, `docs/change_spec.md`, `docs/xprompt.md`,
  `docs/agents_sidecar.md`, `docs/project_spec.md`, `docs/init.md`, `docs/llms.md`.

Add a one-line note in `docs/vcs.md` (mirroring the existing `sase vcs` note) that
`sase commit` remains accepted as a deprecated alias.

Optionally refresh the `stitch` summary in `parser.py`'s `_COMPACT_ROOT_COMMANDS` so the
compact root help mentions creating stitches, not only inspecting them.

### 10. Tests for the new surface

- `tests/main/test_stitch_parser.py` — add a `TestStitchCreateParser` class covering:
  `create` defaults; `-m`/`-M` mutual exclusion; `-f` repeatable; `-t` accepting both
  canonical methods and the `commit`/`propose`/`pr` aliases; `-r/--resume`; that
  `create`'s `-r`/`-b`/`-c` do not disturb `log`'s and `list`'s meanings for the same
  short flags; that bare `sase stitch` still defaults to `list`; and that the
  unknown-subcommand usage string names `create`.
- `tests/main/test_parser_narrowing.py` — add an import-isolation test modeled on
  `test_artifact_parser_uses_lightweight_file_types_module`, asserting
  `sase.workflows.commit` is absent from `sys.modules` after importing
  `sase.main.parser_stitch`.
- Add a parity test that `sase.commit_methods.VALID_METHODS` / `METHOD_ALIASES` are the
  same objects/values re-exported by `sase.workflows.commit.workflow_types`.
- Add coverage that `sase commit ...` and `sase stitch create ...` parse to equivalent
  namespaces for a representative flag set.
- Check whether `tests/main/test_parser_root_help.py` and
  `tests/main/test_parser_command_defaults.py` enumerate stitch subcommands and need
  updating.

### 11. Verification

- `just install` first (workspace virtualenvs are ephemeral).
- `just check` during iteration, then `just check-full` before landing — this change
  touches the parser, entry dispatch, docs, and skill sources, which is squarely in the
  broadening set.
- Smoke-test by hand: `sase stitch create --help`, `sase commit --help`,
  `sase stitch --help`, bare `sase stitch` (must still print the delegation notice and
  list repos), and `sase stitch log -n 3`.
- Confirm no startup regression, e.g. compare `python -X importtime` for
  `sase stitch log --help` against the pre-change baseline, and confirm the new
  import-isolation test fails if `parser_commit`'s heavy import is reintroduced.

### 12. Deploy the skill change (last, after landing)

Per `sase/memory/generated_skills.md`, chezmoi skill files are generated and the
destination is global, so deploying from a dirty or unmerged tree deploys content that
exists in no commit:

1. While iterating, preview with `sase skill init --diff` or `--dry-run` (read-only, no
   guard).
2. Commit the skill-source change and land it on the canonical branch.
3. From that clean, merged tree, run `sase skill init --force`, then `chezmoi apply` if
   it was skipped.

## Items that need separate, explicit user permission

Per this repo's `CLAUDE.md`, memory files may not be edited without explicit user
permission granted in conversation, and authorization inside a plan file does **not**
count. The implementer must therefore ask before touching these, or leave them for a
follow-up:

- `sase/memory/glossary.md` — the Stitch glossary entry still reads "The `sase commit`
  command and real Git/Mercurial commits are still called commits." Editing it requires
  running `sase memory init`, which regenerates `sase/sase.yml`, `AGENTS.md`,
  `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, and `QWEN.md`.
- `sase/memory/xprompts.md` — "the provider-neutral finalizer asks the runtime to use
  `/sase_git_commit` -> `sase commit`".
- `sase/memory/generated_skills.md` — "Any change to `sase commit` CLI arguments must
  include same-turn updates to ...".

Also outside the authorized scope, but worth a decision at approval time:

- `src/sase/xprompts/skills/sase_hg_commit.md` invokes `sase commit` **directly** (no
  wrapper; there is no `sase_hg_commit` script in this repo) and has 7 mentions. The
  user authorized updating `/sase_git_commit` only. The hg skill keeps working through
  the legacy alias, so nothing breaks either way — but leaving it on the old spelling is
  inconsistent. Recommend applying the same minimal spelling update in the same change.

## Follow-up bead to file (via `/sase_new_task`)

`src/sase/ace/restore.py` shells out with a positional name —
`["sase", "commit", base_name]` — but the commit parser defines **no positional
argument**, so that call fails with "unrecognized arguments" and `restore_patch` reports
the failure. The same invalid form appears in `docs/vcs.md` as
`SASE_VCS_PROVIDER=git sase commit my_feature`. The likely intent is `-n/--name`. This
is a pre-existing bug, independent of this migration; file it rather than fixing it
inside a rename change.
