---
create_time: 2026-07-10 11:19:54
status: done
prompt: 202607/prompts/vcs_log_opt_in_sdd.md
tier: tale
---
# Make SDD Commits Opt-In for `sase vcs log`

## Goal

Change `sase vcs log` so its default merged timeline includes primary and linked repositories but excludes commits from
separate SDD repositories. Add `-S|--sdd` as the explicit opt-in for including those SDD histories.

The option must work in both supported discovery scopes:

- `sase vcs log --sdd` includes the current project's existing separate SDD repository alongside its primary and linked
  repositories.
- `sase vcs log --all --sdd` includes existing separate SDD repositories discovered across all registered projects.

This is a log-scope change, not a change to SDD storage or repository inventory. In-tree SDD files already appear in the
primary repository's history, local SDD storage is unversioned, and `sase vcs list` should continue to show the full
repository constellation, including a separate SDD repository when present.

## Current Behavior and Boundaries

The CLI path is entirely in the existing Python command and repository-resolution layer:

- `src/sase/main/parser_vcs.py` defines the sorted `vcs log` options and currently describes SDD history as part of the
  default timeline.
- `src/sase/main/vcs_handler.py` translates parsed options into the `run_vcs_log()` service call.
- `src/sase/vcs_log/collect.py` resolves the repository set before fetching and aggregating commits.
- `src/sase/vcs_log/resolve.py` currently discovers a separate SDD repository for both current-project and all-project
  scopes by default.
- `src/sase/vcs_list/collect.py` reuses that resolver and expects SDD repositories to remain part of the inventory.

Commit fetching, remote comparison, aggregation, renderers, and the Rust core wire/API do not need new behavior. The new
flag changes which repositories reach those existing layers, so no `sase-core` change is warranted.

## User-Facing Semantics

- `sase vcs log` and `sase vcs log --all` exclude separate SDD repositories by default.
- `-S` and `--sdd` are equivalent boolean opt-ins and default to false.
- `--sdd` composes with `--all`, date/author filters, fetch controls, output formats, and ordering exactly like the
  current implicit SDD inclusion.
- `--repo` remains a narrowing filter over the eligible repository set; it does not silently expand scope. To show only
  SDD commits, use `sase vcs log --sdd --repo sdd`. Without `--sdd`, `--repo sdd` should follow the existing unmatched
  repository warning path.
- `--current-only` retains its established meaning: only the current/primary repository is eligible, so it continues to
  exclude linked and SDD repositories even if other scope-selection input is supplied.
- Only existing, separate SDD repositories contribute an additional history. No command path should materialize an SDD
  checkout merely because `--sdd` was passed.

## Implementation Plan

1. Add the public CLI option and update command help.

   Add `-S|--sdd` to `_add_log_options()` in `src/sase/main/parser_vcs.py` as a `store_true` option with concise help
   explaining that it includes separate SDD repository commits. Place it in alphabetical long-option order between
   `--reverse` and `--since`, preserving the repository's CLI-help conventions. Rewrite the log description so primary
   and linked repositories are the default and SDD history is explicitly opt-in.

2. Thread the opt-in through the log service boundary.

   Pass the parsed boolean from `_handle_log()` to `run_vcs_log()`. Add an `include_sdd` input to the log entry point
   and repository resolver, defaulting to false for log callers so direct service use matches the new command behavior.
   Keep this as repository-selection state rather than a commit filter: excluded SDD repositories should never be
   opened, fetched, or represented in remote state.

3. Gate SDD discovery in every resolver scope.

   Update current-project resolution to call `_resolve_sdd_repo()` only when `include_sdd` is true and `current_only` is
   false. Update all-project candidate expansion similarly so the default global scan does not probe SDD stores or emit
   warnings about histories the user did not request. Preserve the existing rules for separate, existing repositories,
   label/alias assignment, canonical-path deduplication, and stable primary/linked/SDD ordering when SDD inclusion is
   enabled.

   Apply `--repo` filtering after this eligible set is constructed. This makes the new inclusion contract explicit and
   avoids a repo filter accidentally restoring a scope that was disabled by default.

4. Preserve `sase vcs list` inventory behavior.

   Have `run_vcs_list()` explicitly request SDD inclusion when it calls the shared resolver. This prevents the log
   default from unintentionally removing SDD rows, stats, aliases, or descriptions from `vcs list`. Update misleading
   list help/documentation that says it always lists exactly the default log set; describe it instead as the available
   repository constellation and point users to `vcs log --sdd` for the corresponding SDD commit history.

5. Add focused regression coverage.

   Extend `tests/main/test_vcs_parser.py` to verify:
   - the new option defaults to false;
   - both `-S` and `--sdd` set it true;
   - help includes the option in sorted order and accurately describes the opt-in.

   Extend `tests/test_vcs_log_collect.py` to verify `run_vcs_log()` passes the default false value and an explicit true
   value into repository resolution while preserving warning ordering and collection behavior.

   Extend `tests/test_vcs_log_resolve.py` to cover:
   - a current project's separate SDD repo is absent by default and present with `include_sdd=True`;
   - excluded SDD stores are not probed and cannot produce discovery warnings;
   - all-project scope excludes every SDD candidate by default and restores them, including existing deduplication and
     qualified-label behavior, when enabled;
   - in-tree, local, and missing separate stores remain absent even when enabled;
   - `--repo sdd` requires the SDD inclusion scope and `--sdd --repo sdd` selects only the SDD repo;
   - `current_only` continues to win over SDD inclusion.

   Update `tests/test_vcs_list_collect.py` so the shared-resolver seam asserts that list collection explicitly includes
   SDD repositories.

6. Update user documentation and examples.

   Revise `README.md`, `docs/vcs.md`, and the `sase vcs` reference in `docs/configuration.md` so they no longer imply
   that the default timeline matches the full list inventory. Document `-S|--sdd`, show current-project and all-project
   examples, and explain that `--repo` narrows after SDD opt-in. Update the concise command overview in `docs/cli.md`
   only where its wording or flag inventory would otherwise be stale.

## Validation

After implementation, install/update the workspace environment before running checks as required by the project:

```bash
just install
```

Run the focused behavior tests first:

```bash
.venv/bin/pytest tests/main/test_vcs_parser.py tests/test_vcs_log_collect.py tests/test_vcs_log_resolve.py tests/test_vcs_list_collect.py
```

Check the rendered help and documentation build:

```bash
.venv/bin/sase vcs log -h
just docs-check
```

Finally run the repository's required comprehensive verification:

```bash
just check
```

## Risks and Mitigations

- Shared resolver defaults could accidentally change `vcs list`. Make list's SDD inclusion explicit at its call site and
  pin it with a seam test.
- Filtering after an unconditional SDD scan would still fetch metadata or surface irrelevant warnings. Gate SDD
  discovery before candidate construction and before `--repo` matching.
- Global scope can expose several repositories with the `sdd` alias. Retain the existing qualification and ambiguity
  rules, and test opt-in alongside all-project deduplication.
- Existing scripts that relied on implicit SDD commits will observe the intentional breaking default. Make migration
  obvious in help and docs: add `--sdd`, and add `--repo sdd` as well when only SDD history is desired.

## Non-Goals

- Do not change SDD storage policy, initialize or materialize missing SDD repositories, or alter SDD commit creation.
- Do not change commit aggregation, remote fetch/cache behavior, rendering formats, JSON schemas, or SASE tag parsing.
- Do not remove SDD repositories from `sase vcs list`.
- Do not add configuration that restores implicit SDD inclusion; the command-line opt-in is the new contract.
