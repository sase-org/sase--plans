---
tier: tale
title:
  Stage every change by default in `sase stitch create` and replace `-f/--file` with
  `-x/--exclude`
goal:
  "`sase stitch create` (and its `sase commit` alias) stages every change in the
  repository — including new, untracked files — with no per-file flags, and an agent
  narrows that commit only by naming paths to leave out with the new repeatable
  `-x/--exclude` option."
size: medium
proposed_by: bbugyi200.athena.yq
create_time: 2026-08-12 12:34:20
status: wip
---

# Plan: Commit everything by default; exclude instead of include

## Background

`sase stitch create` currently takes a repeatable `-f/--file` allowlist. When the
allowlist is empty the git provider already falls back to `git add -A`, so "stage
everything" is an existing, working code path — it is just not the default that agents
are instructed to use.

The project owner's premise is that SASE agents are the only writers in the repos they
work on, so an allowlist is the wrong default: it silently drops newly created files an
agent forgot to list. This plan inverts the contract. The allowlist disappears from the
agent-facing CLI, staging everything becomes the default, and a new repeatable
`-x/--exclude PATH` subtracts paths from that default.

### Current shape (verified on master)

- `src/sase/main/parser_commit.py` — `add_commit_create_options()` declares `-f/--file`
  (`action="append"`, `dest="files"`). This one function is shared by
  `sase stitch create` (registered in `src/sase/main/parser_stitch.py`) and the legacy
  top-level `sase commit` alias, so the two spellings cannot drift.
- `src/sase/main/commit_handler.py` — maps `args.files` into `payload["files"]`.
- `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` — `vcs_create_commit()` and
  `vcs_create_pull_request()` both do `git add -- <files>` when `payload["files"]` is
  non-empty and `git add -A` otherwise, then unconditionally stage the bead store
  (`_stage_bead_dirs`) and any workflow-owned plan path (`_stage_extra_paths`, from
  `payload["_plan_path"]`). `_amend_bead_changes()` folds post-commit bead-store writes
  in with `--amend`.
- `src/sase/workflows/commit/checkpoint.py` — persists the whole payload dict, so any
  new payload key rides along through `--resume` with no extra work.
- `src/sase/axe/run_agent_exec_plan_sdd.py` — `commit_sdd_files_for_exec_plan()` is the
  only in-repo caller that passes `-f`. It shells out to `sase stitch create` to commit
  _just_ the approved SDD plan file before an epic agent starts. This narrowing is
  load-bearing: the same workspace can hold an untracked retired prompt snapshot
  (`sdd/prompts/<yyyymm>/<name>.md`) that must not be committed, which
  `tests/test_sdd_commit_plan_accept.py` asserts.
- `src/sase/scripts/sase_git_commit` — the wrapper agents actually invoke. It parses
  `-f`/`--file`/`--file=` to record a `files` list in the
  `$SASE_ARTIFACTS_DIR/commit_skill_invoked.json` invocation marker, then forwards every
  argument to `sase stitch create`.
- `src/sase/commit_instructions.py` — `build_commit_instruction_message()` tells
  post-completion finalizer-triggered agents to pass one `-f` per listed file.
- `src/sase/xprompts/skills/sase_git_commit.md` and `sase_hg_commit.md` — the skill
  sources that document `-f`.
- `docs/commit_workflows.md` — flag table, worked examples, and the payload contract.

### Facts established while researching (do not re-derive)

- `git add -A -- ':/' ':(exclude,top)<path>'` was verified by hand in a scratch repo: it
  stages the whole tree even when run from a subdirectory, honors directory excludes,
  and leaves excluded paths dirty. Bare `git add -A -- .` would only stage the current
  subtree, so the `:/` positive pathspec matters.
- A `:(exclude)` pathspec that matches nothing is **silently ignored** by `git add` — no
  error, no non-zero exit. A typo'd `-x` path would therefore be committed anyway. This
  is why this plan requires exclude validation rather than trusting git.
- `.sase/` and `/sase/repos/` are both git-ignored (`.gitignore` lines 61 and 63), so
  staging everything cannot sweep in scratch commit-message files or the nested linked,
  sidecar, and external repo clones.
- `RunResult.FAILED == 1` and `EXIT_CODE_CONFLICT == RunResult.CONFLICT == 2`
  (`src/sase/workflows/commit/workflow_types.py`). Exit code 2 already means "a rebase
  is paused for a real conflict; do not re-run" in the commit skill. `parser.error()`
  also exits 2, so **no new CLI rejection may go through `parser.error()`** — that would
  send an agent down the conflict-recovery path for a bad flag. Every new rejection in
  this plan exits 1.
- The GitHub provider in the `sase-github` linked repo has no commit-dispatch source of
  its own; it composes `GitCommitDispatchMixin` from this repo. Its tests build payloads
  with a `files` key, so keeping the `files` payload semantics intact keeps that repo
  green with no cross-repo change.
- No Rust change is needed. `sase-core` carries only read-only git _parsers_
  (`crates/sase_core/src/git_query/`); all commit dispatch and staging is host-side
  Python, and the nearest existing pure path predicate (`_all_bead_conflicts`) already
  lives in the Python dispatch module.
- The invocation marker's `files` key has exactly one consumer,
  `tests/test_sase_git_commit_wrapper.py`. Nothing in `sase-core` reads it.

## Design decisions

1. **Keep the `files` payload key; remove only the agent-facing flag.** The provider
   allowlist stays exactly as it is today (`payload["files"]` non-empty ⇒
   `git add -- <files>`). Removing it would break the SDD plan commit and would force a
   coordinated change in `sase-github`. What changes is who can reach it.

2. **Expose the allowlist only through a hidden, internal-only long option.** Add
   `--only-file` (`action="append"`, `dest="only_files"`, `help=argparse.SUPPRESS`, no
   short alias — `sase/memory/cli_rules.md` exempts internal subprocess arguments from
   the short-alias rule). `commit_sdd_files_for_exec_plan()` switches to it. It never
   appears in `--help`, in any skill, or in any doc.

3. **`-f/--file` stays registered as a loud removal stub.** Generated skill copies are
   deployed to a global chezmoi destination and are known to lag the in-repo sources, so
   a stale copy will pass `-f`. Registering a suppressed `-f/--file` whose argparse
   action prints an explanation to stderr and exits **1** turns that into an actionable
   message instead of argparse's bare `unrecognized arguments: -f` (which exits 2 and
   reads as a paused rebase). Message text:

   ```
   error: `-f/--file` was removed. `sase stitch create` now stages every change in
   the repository, including untracked files. Use `-x/--exclude PATH` (repeatable)
   to leave a path out of the commit.
   ```

4. **`-x/--exclude` takes repo-root-relative literal paths, not glob pathspecs.** A file
   or a directory. Directories exclude the whole subtree. Absolute paths, paths
   containing `..`, and strings starting with `:` (raw git pathspec magic) are rejected.
   Keeping the surface literal makes validation meaningful and keeps the semantics the
   same shape as the paths `git status` prints.

5. **An exclude that matches nothing is a hard failure, before any commit.** Since git
   ignores unmatched exclusions, a typo would otherwise commit the very file the agent
   meant to keep out — a silent, unrecoverable-by-inspection error. Validation runs
   before staging and fails the workflow with exit 1.

6. **`-x` may not exclude workflow-owned paths.** The bead store and
   `payload["_plan_path"]` are staged unconditionally by `_stage_bead_dirs()`,
   `_stage_extra_paths()`, and `_amend_bead_changes()`; honoring an exclude there would
   either break bead-store consistency or lie about what was excluded. Validation
   rejects such an exclude with a message naming the reason.

7. **A provider that cannot honor excludes must fail loudly, not ignore them.** Only the
   git dispatch mixin implements exclusion. A Mercurial provider (out of tree — the
   `/sase_hg_commit` skill exists, the plugin does not live in this repo) would silently
   commit an excluded file. Add a structural capability probe following the existing
   `_has_hookimpls()` pattern in `src/sase/vcs_provider/_plugin_manager.py`, and refuse
   the commit when `exclude` is non-empty and the resolved provider does not advertise
   support.

8. **Do not re-sort the option block.** `add_commit_create_options()` is not currently
   alphabetized; declare `-x/--exclude` where `-f/--file` was and leave the rest of the
   block alone. A drive-by re-sort would bury this change's diff.

### Rejected alternatives

- _Let the SDD plan commit stage everything._ Rejected: it would sweep the untracked
  retired prompt snapshot into the plan commit, contradicting an existing test and
  changing what an epic agent inherits.
- _Have the SDD caller compute excludes from `git status` and pass `-x` for everything
  else._ Rejected: racy, and it inverts a narrow, well-understood operation.
- _Call `CommitWorkflow` in-process from the SDD path instead of shelling out._
  Rejected: `CommitWorkflow` reads `os.getcwd()`, so an in-process call from axe would
  need a process-wide `chdir` in a threaded runner.

## Implementation

Work through these in order; each step leaves the tree consistent.

### 1. CLI surface — `src/sase/main/parser_commit.py`

In `add_commit_create_options()`:

- Replace the `-f/--file` declaration with a suppressed removal stub. Implement a small
  module-level `argparse.Action` subclass (mirroring the `_NoOpAction` precedent in
  `src/sase/main/parser_stitch.py`) that writes the message from design decision 3 to
  `sys.stderr` and raises `SystemExit(1)`. Give it `nargs="?"` so both `-f path` and a
  bare `-f` are caught and no stray value leaks into positional parsing.
- Add the replacement option in that slot:

  ```python
  parser.add_argument(
      "-x",
      "--exclude",
      action="append",
      default=[],
      dest="exclude",
      metavar="PATH",
      help="Repo-relative file or directory to leave out of the commit (repeatable)",
  )
  ```

- Add the hidden internal allowlist:

  ```python
  parser.add_argument(
      "--only-file",
      action="append",
      default=[],
      dest="only_files",
      help=argparse.SUPPRESS,
  )
  ```

- Update the `-r/--resume` help text: it currently says `-m/-M/-f` are ignored on
  resume; make it say `-m/-M/-x`.

Confirm `-x` does not collide (the block uses `-m -M -f -n -b -B -c -p -s -t -r`), and
that `--file` still resolves to the stub rather than being an ambiguous abbreviation of
`--only-file` (it is not a prefix of it, so it does not).

### 2. Payload mapping — `src/sase/main/commit_handler.py`

- Build the payload with both keys:
  `{"message": ..., "files": args.only_files or [], "exclude": args.exclude or []}`.
- Reject the nonsensical combination before constructing the workflow: if both
  `only_files` and `exclude` are non-empty, print a clear error to stderr and
  `sys.exit(1)`.
- Normalize each exclude entry here (strip whitespace, drop a leading `./`, collapse a
  trailing `/`) so the provider receives canonical repo-relative POSIX paths, and reject
  absolute paths, entries containing a `..` component, and entries beginning with `:`
  with a printed reason and exit 1.

### 3. Provider capability probe

- `src/sase/vcs_provider/_hookspec.py` — add a `vcs_supports_commit_excludes()` hookspec
  returning `bool`.
- `src/sase/vcs_provider/plugins/_git_commit_dispatch.py` — implement it with
  `@hookimpl`, returning `True`.
- `src/sase/vcs_provider/_base.py` — add a concrete
  `supports_commit_excludes(self) -> bool` returning `False` on `VCSProvider`, so every
  existing and out-of-tree provider defaults to "not supported" without being forced to
  change.
- `src/sase/vcs_provider/_plugin_manager.py` — override it as
  `return self._has_hookimpls(("vcs_supports_commit_excludes",))`, matching the existing
  `supports_issues()` / listing-probe style.
- `src/sase/workflows/commit/workflow.py` — in `run()`, immediately after the payload
  validation block and **before** the first side effect (`apply_bead_commit_tag`), fail
  with `print_status(...)` + `RunResult.FAILED` when `self._payload.get("exclude")` is
  non-empty and the provider for `cwd` does not support excludes. Name the provider in
  the message.

### 4. Staging — `src/sase/vcs_provider/plugins/_git_commit_dispatch.py`

Add two helpers next to the existing `_stage_bead_dirs` / `_validate_staged` helpers:

- `_validate_excludes(self, payload: dict, cwd: str) -> tuple[bool, str | None]`
  - Collect the changed set the same way `_changed_bead_files()` already does:
    `git ls-files --modified --others --deleted --exclude-standard -z` (whole repo, no
    pathspec), split on `\0`.
  - An exclude `p` matches a changed path `c` when `c == p` or
    `c.startswith(p.rstrip("/") + "/")`.
  - Unmatched exclude ⇒ `(False, "…")` naming the path and stating that it matches no
    changed file, so the commit was refused before anything was staged.
  - Exclude that covers `BEADS_DIRNAME` (already imported in this module) or the
    `payload["_plan_path"]` value ⇒ `(False, "…")` explaining that the commit workflow
    owns those paths and always stages them.
- `_stage_all_except(self, exclude: list[str], cwd: str)`
  - Empty `exclude` ⇒ run exactly today's `["git", "add", "-A"]` (byte-identical, so the
    common path cannot drift).
  - Otherwise
    `["git", "add", "-A", "--", ":/", *(f":(exclude,top){p}" for p in exclude)]`.

Wire both into `vcs_create_commit()` and `vcs_create_pull_request()`: keep the `files`
allowlist branch untouched, and in the else-branch call `_validate_excludes()` first
(returning its error unchanged on failure) and then `_stage_all_except()`. In
`vcs_create_pull_request()` the validation must run **before** `git checkout -b`, so the
refusal does not strand a new branch.

### 5. Internal SDD caller — `src/sase/axe/run_agent_exec_plan_sdd.py`

In `commit_sdd_files_for_exec_plan()`, change the argument construction from
`cmd.extend(["-f", f])` to `cmd.extend(["--only-file", f])`. Behavior is otherwise
unchanged. Update the docstring line that describes the flag if it names `-f`.

### 6. Wrapper script — `src/sase/scripts/sase_git_commit`

In the embedded Python `_parse_args()`:

- Stop collecting `-f` / `--file` / `--file=`.
- Collect `-x` / `--exclude` / `--exclude=` into an `exclude` list.
- Return it in place of `files` and write `"exclude": exclude` into the
  `commit_skill_invoked.json` marker, dropping the `"files"` key.

Everything else in the wrapper (method resolution, `--resume` detection, jlog events,
argument forwarding) is unchanged.

### 7. Finalizer instructions — `src/sase/commit_instructions.py`

In `build_commit_instruction_message()`, replace the two `-f` sentences with wording
that matches the new default. Suggested replacement, keeping the same
one-sentence-per-`parts.append()` style:

- "The commit skill stages every change in that repository by default, including newly
  created untracked files, so you do not list files."
- "Pass `-x <path>` (repeatable) only when a specific path must be left out of this
  commit."

### 8. Skill sources

`src/sase/xprompts/skills/sase_git_commit.md`:

- Step 1: keep the `git status` / `git diff` review, but reframe the untracked-file
  sentence — untracked files are now committed automatically, so the reason to read
  `git status` is to confirm nothing _unwanted_ is dirty and decide whether any path
  needs `-x`.
- Step 4: the command becomes `sase_git_commit -M .sase/commit_message.md`. Drop the
  paragraph telling finalizer-triggered commits to pass one `-f` per listed file.
- Step 4 flag list: remove the `-f` entry; add `-x`: "Repo-relative path (file or
  directory) to leave out of this commit (repeatable). Everything else that changed,
  including untracked files, is committed. A path that has no pending change is an
  error, so the commit fails loudly rather than quietly committing a mistyped path."
- Exit-code list: no change (1 = failed with a printed reason already covers a rejected
  flag or a bad `-x`).
- `## Example`: `sase_git_commit -M .sase/commit_message.md` plus one exclusion example.

`src/sase/xprompts/skills/sase_hg_commit.md`: the CLI options are shared, so its two
`sase commit -M commit_message.md -f file1.py -f file2.py` lines must lose `-f`. Do not
document `-x` there — the Mercurial provider does not advertise exclude support and step
3 makes it an error.

### 9. Docs — `docs/commit_workflows.md`

Update, in place: the `sase_git_commit -M ... -f src/example.py` example (line ~76), the
`sase stitch create` example (~84), the `-f | --file` flag-table row (~107), the
`-r/--resume` row that lists `-f` (~113), the worked example (~247), the payload JSON
block and the paragraph mapping flags to payload keys (~262-269), and the two "Stage
files (`git add -A` or specific files)" step summaries (~392, ~445). The payload docs
should describe `exclude` as the agent-facing key and should not advertise `files` as
reachable from the CLI.

### 10. Tests

Update:

- `tests/test_commit_cli.py` — `-f` now exits 1 with the removal message (replacing
  `test_basic_commit` / `test_multiple_files` / `test_no_files_stages_all` usage of
  `-f`); `-x a.py -x b/` maps to `payload["exclude"] == ["a.py", "b"]`; the default
  payload carries `"files": []` and `"exclude": []`; `--only-file` maps to
  `payload["files"]`; `--only-file` combined with `-x` exits 1; an absolute or `..`
  exclude exits 1.
- `tests/test_sase_git_commit_wrapper.py` — invoke the wrapper with
  `-x src/foo.py --exclude=tests/test_foo.py`, assert `marker["exclude"]` and that
  `files` is gone, and assert the forwarded argv still matches.
- `tests/test_sdd_commit_plan_accept.py` — the two helpers that scan for `-f` now scan
  for `--only-file`; the assertions that the plan file is included and the prompt
  snapshot is not are unchanged.
- `tests/test_commit_instructions.py` — replace the two `-f` assertions with the new
  wording.
- `tests/test_vcs_provider_bare_git_plugin.py` and the other dispatch tests that pass
  `{"files": []}` — still valid; extend where a real staging assertion is cheap.

Add:

- Real-repo coverage for staging (a temp git repo, in the style of
  `tests/test_vcs_provider_commit_first_rebase.py`): a default commit stages a modified
  file **and** an untracked file; `exclude` of a file leaves it out of the commit and
  still dirty afterward; `exclude` of a directory leaves the whole subtree out; an
  exclude that matches no changed path fails with no commit created and the tree
  untouched; an exclude covering the bead store or `_plan_path` is refused; the
  `create_pull_request` path refuses before creating the branch.
- A test that a provider without `vcs_supports_commit_excludes` plus a non-empty
  `exclude` fails the workflow with `RunResult.FAILED` and never reaches dispatch.

### 11. Verification

- `just install` first (workspaces are ephemeral and dependencies may have moved), then
  `just check-full` — this change touches the CLI parser, the VCS provider hookspec, and
  the shell wrapper, which is broader than the scoped lane's import-graph closure.
- Re-read `src/sase/xprompts/skills/sase_git_commit.md` end to end after editing and
  confirm no `-f` remains anywhere in `src/`, `docs/`, or `tests/` for the commit CLI:
  `grep -rn -- '-f ' src/sase/xprompts/skills/ docs/commit_workflows.md`.
- Do **not** run `sase skill init --force` from the implementing workspace. Per
  `sase/memory/generated_skills.md`, the chezmoi destination is global: the template
  change must be committed and landed on the canonical branch first, and the deploy runs
  from that clean, merged tree afterward. `sase skill init --diff` is read-only and is
  fine for previewing while iterating.

## Out of scope

- Re-sorting or otherwise reorganizing `add_commit_create_options()`.
- Any change in `sase-core` (no commit staging lives there) or `sase-github` (it
  composes this repo's dispatch mixin and its payloads keep working).
- Teaching a Mercurial provider to honor `-x`; step 3 makes the gap an explicit, loud
  failure rather than a silent one.
- Refreshing the deployed chezmoi skill copies, which is a post-land step.
