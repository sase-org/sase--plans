---
tier: tale
title: Rename `sase stitch log` to `sase stitch list`
goal:
  The cross-repository stitch timeline is the only `sase stitch list` subcommand, `sase
  stitch log` is gone, a bare `sase stitch` delegates to it, and the old
  repository-constellation listing plus its now-dead provider repo-stats stack are
  deleted.
size: medium
proposed_by: bbugyi200.athena.xt
create_time: 2026-08-11 06:40:40
status: done
---

- **PROMPT:**
  [prompts/202608/stitch_list_rename.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/stitch_list_rename.md)
- **AGENTS:**
  - [bbugyi200.athena.xt](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.xt.md)
- **COMMITS:**
  - [3306e09](https://github.com/sase-org/sase/commit/3306e093cb2cf866be3da097167abb85b79705a9)
    — feat(cli)\!: rename stitch log to list

# Rename `sase stitch log` to `sase stitch list` and delete the old repo-listing command

## Goal

Make the cross-repository stitch timeline the one and only `sase stitch list`
subcommand, and delete the current repository-constellation listing that owns that name
today. A bare `sase stitch` must keep delegating to `list`, which now means the
timeline.

After this change:

- `sase stitch list` shows the day-grouped, cross-repository commit timeline (today's
  `sase stitch log`) with all of its options unchanged.
- `sase stitch log` no longer exists.
- `sase stitch` (and the legacy `sase vcs`) prints the existing delegation notice and
  then the timeline.
- The repository-constellation listing (`sase.vcs_list`) and the provider `repo_stats`
  capability that only it consumed are deleted.

## Current State (verified in this repo)

- `src/sase/main/parser_stitch.py` registers three subcommands: `create`, `list`
  (constellation listing, via `_add_list_options()`), and `log` (timeline, via
  `_add_log_options()`).
- `src/sase/main/stitch_handler.py` dispatches `_handle_list` (calls
  `sase.vcs_list.collect.run_vcs_list` + `sase.vcs_list.render.render`) and
  `_handle_log` (calls `sase.vcs_log.collect.run_vcs_log` +
  `sase.vcs_log.render.render`).
- **Bare-group defaulting already works generically.** `_default_list_subcommands()` in
  `src/sase/main/parser.py` defaults any group with an exact `list` child to that child,
  copies the `list` child's defaults onto the parent, and records the marker that
  `default_list_delegation_notice()` renders. Verified at runtime:
  `parse_args(["stitch"])` yields `stitch_subcommand == "list"` and the notice
  `No subcommand provided for 'sase stitch'; delegating to 'sase stitch list'.`. Because
  the renamed child is still named exactly `list`, **no new defaulting machinery is
  needed** — the requirement is satisfied by the rename itself, and the work is to prove
  it with tests rather than to build it.
- `parser_stitch.py:238` also calls
  `stitch_parser.set_defaults(stitch_subcommand="list")`. This is redundant with the
  central mechanism but matches `parser_plan.py:46`; leave it alone (out of scope).
- `sase.vcs_list` is the **only** consumer of the provider `repo_stats` capability,
  which spans `VCSProvider.repo_stats`, `PluginManagerProvider.repo_stats`, the
  `vcs_repo_stats` hookspec, the bare-git `vcs_repo_stats` hookimpl,
  `sase.core.vcs_repo_stats_facade`, and `sase.core.vcs_repo_stats_wire`.
- No external consumers exist. Verified by grepping the opened checkouts of
  `sase-github`, `sase-core`, `sase-nvim`, and `chezmoi`: none reference `repo_stats`,
  `vcs_list`, `sase stitch list`, or `sase stitch log`.
- `sase.core.git_query_facade.parse_git_branch_name` / `parse_git_local_changes` are
  also used by `_git_revision_ops.py` and `_git_query_ops.py`, so they survive the
  deletion.

## Decisions

**D1 — Hard rename; `log` is NOT kept as an alias.** The plan removes `sase stitch log`
outright, so it fails with argparse's `invalid choice: 'log'`. Rationale: the request is
a rename plus a removal, not a deprecation; leaving a `log` alias next to a `list` that
is now also the bare-`sase stitch` default invites the reader to assume they differ.

_If the reviewer prefers to keep `log` working_, the only change is:
`stitch_sub.add_parser("list", aliases=["log"], ...)` plus normalizing `"log"` to
`"list"` in `handle_stitch_command`, plus a compatibility test. The rest of the plan is
unaffected. Note
`tests/main/test_parser_command_defaults.py::test_all_subparser_choices_are_sorted`
still passes either way, since `["create", "list", "log"]` is sorted.

**D2 — Delete the orphaned `repo_stats` stack rather than keep it.** Once `vcs_list` is
gone, every symbol in that stack is dead. Per `sase/memory/symvision.md`, a symbol
exercised only by tests is not "used", pragmas require a real non-test consumer, and the
first choice for genuinely dead code is deletion together with its tests. Keeping the
stack would fail the `symvision` stage of `just check`.

**D3 — Keep the `src/sase/vcs_log/` package name.** Renaming it to `vcs_list` would
collide with the package being deleted, churn ~20 source modules and ~10 test modules,
and drag in the provider hook name `vcs_log`, which is public plugin API and describes
commit-log collection rather than the CLI verb. Only the docstrings that name the CLI
command change. Explicit non-goal.

**D4 — Behavior differences the docs must call out.** The new `sase stitch list` is not
a drop-in for the old one:

- Sidecar repos are now **excluded** by default (`--sdd` opts in); the old `list` always
  included them.
- `-N`/`--no-fetch` now means "skip the remote fetch", not "skip description lookups".
- `-s` is now `--since`, not `--sort`; `--sort` no longer exists.
- New options: `-a/--all`, `-A/--author`, `-b/--branch`, `-F/--fetch`, `-n/--limit`,
  `-m/--merges`, `-T/--no-tags`, `-R/--reverse`, `-S/--sdd`, `-u/--until`, and `full` as
  a `--format` choice.

## Implementation Steps

### Step 1 — Parser: `src/sase/main/parser_stitch.py`

1. Delete the existing `_add_list_options()` (the
   `--color/--current-only/--format/ --no-fetch/--repo/--sort` constellation options).
2. Rename `_add_log_options()` to `_add_list_options()`. Its body is unchanged.
3. Delete the old `list` subparser registration block and the `log` subparser
   registration block; register one `list` subparser that carries the timeline help
   text, description, `epilog=f"DATE grammar: {DATE_HELP}."`,
   `formatter_class=argparse.RawDescriptionHelpFormatter`, and `_add_list_options`. Keep
   it registered after `create` so `action.choices` stays alphabetically sorted.
4. Change the subparsers `metavar` from `"{create,list,log}"` to `"{create,list}"`.
5. Update the `stitch` group `help=`/`description=` so they describe dispatch + timeline
   (no longer "inspect repositories"), and keep the sentence documenting that a bare
   `sase stitch` defaults to `sase stitch list`.
6. Update the `register_stitch_parser` docstring, which currently says `sase stitch`
   lists "the resolved repository constellation".
7. Leave untouched: `_DEFAULT_LIMIT`, `_NoOpAction`, the `nonnegative_int` /
   `add_commit_create_options` / `DATE_HELP` imports, the
   `stitch_parser.set_defaults(stitch_subcommand="list")` line, and the
   `register_vcs_parser` legacy alias.

Per `sase/memory/cli_rules.md`: keep options sorted, keep every long option's short
alias, and keep `-h` output clean. The renamed option set already satisfies this
(`tests/main/test_stitch_parser.py` asserts the sorted long-option order).

### Step 2 — Handler: `src/sase/main/stitch_handler.py`

1. Delete `_handle_list` (the `run_vcs_list` implementation).
2. Rename `_handle_log` to `_handle_list`; keep the body, but change both stderr
   prefixes from `sase stitch log:` to `sase stitch list:`, and update its docstring.
3. Reduce `_HANDLERS` to `{"list": _handle_list}`.
4. Change the fallback usage string to `Usage: sase stitch {create,list}`.
5. Update `__all__` to drop `_handle_log`.
6. Leave `handle_stitch_command`'s `create` branch and the `handle_vcs_command` alias
   untouched.

### Step 3 — Delete the dead repo-listing and repo-stats code

1. Delete the package `src/sase/vcs_list/` (`__init__.py`, `collect.py`, `models.py`,
   `render.py`).
2. Delete `src/sase/core/vcs_repo_stats_facade.py` and
   `src/sase/core/vcs_repo_stats_wire.py`.
3. `src/sase/vcs_provider/_base.py`: remove `VCSProvider.repo_stats` and the now-unused
   `VcsRepoStatsWire` `TYPE_CHECKING` import.
4. `src/sase/vcs_provider/_plugin_manager.py`: remove `PluginManagerProvider.repo_stats`
   and the now-unused `VcsRepoStatsWire` `TYPE_CHECKING` import.
5. `src/sase/vcs_provider/_hookspec.py`: remove the `vcs_repo_stats` hookspec and the
   now-unused `VcsRepoStatsWire` `TYPE_CHECKING` import.
6. `src/sase/vcs_provider/plugins/_git_query_ops.py`: remove the `vcs_repo_stats`
   hookimpl and the `build_vcs_repo_stats` / `VcsRepoStatsWire` imports. Keep
   `VCS_LOG_GIT_FORMAT`, `parse_git_branch_name`, and `parse_git_local_changes`, which
   other methods in the file still use.

After this step, re-run `just symvision` on its own and resolve any newly surfaced dead
symbol using the `sase/memory/symvision.md` hierarchy (delete first; never add a pragma
without a real non-test consumer).

### Step 4 — Tests

Delete outright:

- `tests/test_vcs_list_collect.py`
- `tests/test_vcs_list_render.py`
- `tests/test_core_vcs_repo_stats.py`

Edit `tests/test_vcs_provider_vcs_log.py`:

- Remove `test_vcs_repo_stats_empty_repo_returns_zero`,
  `test_vcs_repo_stats_collects_counts_branch_dirty_and_last_commit`, and
  `test_provider_repo_stats_delegates_to_hook`.
- Remove the `from sase.core.vcs_repo_stats_wire import VcsRepoStatsWire` import.

Edit `tests/main/test_stitch_parser.py` (the bulk of the test work):

- Delete `test_list_defaults` and `test_list_options` (they cover the removed
  constellation options).
- Repoint every `["stitch", "log", ...]` argv to `["stitch", "list", ...]` and rename
  the `test_log_*` methods to `test_list_*` (including
  `test_log_help_shows_sorted_current_tag_and_fetch_options` and
  `test_legacy_vcs_alias_resolves_log_defaults`).
- Rewrite `test_bare_stitch_defaults_to_list` and
  `test_legacy_vcs_alias_defaults_to_list` to assert the timeline defaults that
  `_default_list_subcommands()` copies onto the parent: `stitch_subcommand == "list"`,
  `limit == 40`, `merges == "hide"`, `sdd is False`, `all is False`, `repos == []`,
  `format == "pretty"`, `color == "auto"`, `show_tags is True`, `no_fetch is False`,
  `force_fetch is False`, `reverse is False`, `since is None`, `until is None`,
  `remote_ref is None`, `authors == []`, and that `sort` is absent.
- Update `test_create_short_flags_do_not_disturb_log_or_list_short_flags`: keep the
  `create` half, replace the `log`/`list` halves with a single `list` case
  (`["stitch", "list", "-r", "sase-core", "-b", "main"]`).
- Update `test_unknown_subcommand_usage_string_names_create` to expect
  `Usage: sase stitch {create,list}`.
- Update the `TestStitchHandlerDispatch` date tests to import `_handle_list` instead of
  `_handle_log`, and assert the `sase stitch list:` stderr prefix.
- Add an explicit regression test that a bare `sase stitch` and an explicit
  `sase stitch list` parse to equal namespaces, and that
  `default_list_delegation_notice()` returns the notice for the bare form and `None` for
  the explicit form. This is the direct test of the "default subcommand" requirement,
  alongside the generic coverage already in
  `tests/main/test_parser_command_defaults.py::test_exact_list_subcommands_default_when_group_is_omitted`.
- Add a test asserting `sase stitch log` is rejected (`SystemExit` with
  `invalid choice`) — this is the executable statement of decision **D1**.

Edit `tests/main/test_parser_narrowing.py`:

- `test_stitch_parser_supports_canonical_command_and_legacy_alias` parses
  `[command, "log"]`; change to `"list"` and update the assertion.

Sweep the remaining test modules:

- `tests/test_vcs_log_*.py` module docstrings say `sase stitch log`; update to
  `sase stitch list` (module filenames stay, matching decision **D3**).
- `grep -rn '"stitch", "log"\|stitch log\|stitch list' tests/` afterwards to confirm no
  stale references remain.

### Step 5 — Documentation

`docs/vcs.md`:

- Delete the `### sase stitch list` constellation section entirely (prose, examples, and
  its option table).
- Rename `### sase stitch log` to `### sase stitch list` and rewrite every
  `sase stitch log` occurrence in its prose, example block, and option table.
- Add the bare-invocation note: `sase stitch` prints
  `No subcommand provided for 'sase stitch'; delegating to 'sase stitch list'.` and then
  the timeline; `sase vcs` remains a deprecated alias.
- Record the **D4** behavior differences so the removal is discoverable (notably that
  sidecars are now opt-in via `--sdd`).
- Remove `vcs_repo_stats` from the "Optional core" hook list (currently line 53).

`docs/cli.md`:

- Collapse the two `sase stitch list` / `sase stitch log` table rows into one
  `sase stitch list` row describing the timeline, pointing at `vcs.md#sase-stitch-list`.
- Verify no other row or anchor still links to `vcs.md#sase-stitch-log`.

`docs/configuration.md`:

- Line ~763: the `sase stitch log` reference becomes `sase stitch list`.
- Lines ~3253-3264: rewrite the paragraph that describes `sase stitch` defaulting to a
  constellation listing so it describes the timeline, and update the date-filter
  sentence's command name.

Do **not** touch `CHANGELOG.md` — it is release-please generated and
`just _lint-changelog` (`tools/validate_changelog`) rejects hand-written sections. Do
**not** touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims; no user
permission for a memory edit exists for this task, and none is needed (no memory file
documents these subcommands).

### Step 6 — Source docstring sweep

Replace `sase stitch log` with `sase stitch list` in docstrings only (no module or
symbol renames) across:

- `src/sase/vcs_log/__init__.py`, `collect.py`, `dates.py`, `fetch_cache.py`,
  `models.py`, `progress.py`, `render.py`, `resolve.py`, `_render_console.py`,
  `_render_plain.py`, `_render_util.py`
- `src/sase/core/vcs_log_wire.py`, `src/sase/core/vcs_log_facade.py`
- `src/sase/vcs_provider/_base.py`, `src/sase/vcs_provider/plugins/_git_query_ops.py`

Finish with `grep -rn "stitch log" src/ docs/ tests/` returning nothing.

## Verification

1. `just install` (workspace venvs are ephemeral; required before anything else).
2. `just symvision` on its own right after Step 3, to catch newly orphaned symbols
   early.
3. `just check-full` — mandatory rather than `just check`, because this change touches
   the parser, the VCS provider base/hookspec/plugin surface, and `src/sase/core/`, i.e.
   the broadening set.
4. Manual smoke test from the workspace:
   - `sase stitch` — prints the delegation notice, then the timeline.
   - `sase stitch list -n 3` — same output shape as today's `sase stitch log -n 3`.
   - `sase stitch list --help` — timeline options, sorted, with the DATE grammar epilog.
   - `sase stitch --help` — subcommand list reads `{create,list}`.
   - `sase stitch log` — fails with `invalid choice: 'log'`.
   - `sase vcs` — same output as `sase stitch`.
   - `sase stitch create --help` — unchanged.

## Risks and Mitigations

- **Newly dead code after the deletion.** Mitigated by running `just symvision` as its
  own step and by the `sase/memory/symvision.md` decision hierarchy. Already checked:
  `parse_git_branch_name` / `parse_git_local_changes` retain other callers, and no tool
  or golden fixture registry enumerates `src/sase/core/` modules.
- **Silent behavior change for `sase stitch list` callers.** Any script passing the old
  `--sort` fails loudly with an argparse error rather than silently changing meaning;
  the **D4** doc note exists so the change is discoverable.
- **Missed `stitch log` references.** Mitigated by the closing greps in Steps 4-6. The
  linked repos were already checked and are clean, so no cross-repo follow-up is needed.

## Out of Scope

- Renaming the `src/sase/vcs_log/` package or the `vcs_log` provider hook (decision D3).
- Removing the redundant `stitch_parser.set_defaults(stitch_subcommand="list")` line, or
  the identical one in `parser_plan.py`.
- Any change to `sase stitch create` / `sase commit`, or to the `sase vcs` legacy alias.
- Restoring repository-constellation output under a new name. If that view is still
  wanted, it should be a separate task bead; this plan deletes it.
