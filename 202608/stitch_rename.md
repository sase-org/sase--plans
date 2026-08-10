---
tier: epic
title: Rename sase vcs to sase stitch and the ACE Commits sub-tab to Stitches
goal: '`sase stitch` is the CLI command for the repository constellation and cross-repo
  timeline (with `sase vcs` still accepted as a legacy alias), and the ACE Artifacts
  tab''s second pane is named "Stitches" end to end — label, sub-tab identifier, DOM
  ids, keymap action ids, and config keys — with legacy keymap/config names still
  honored and warned about.

  '
phases:
  - id: cli
    title: Rename the sase vcs CLI command to sase stitch
    depends_on: []
    size: medium
    description: "cli: rename the `sase vcs` command tree to `sase stitch`, keep `vcs`
      as a registered legacy alias with facade modules, and update the compact root
      help, parser/handler modules, tests, and CLI/VCS/configuration docs.

      "
  - id: subtab-id
    title: Rename the ACE Artifacts sub-tab identifier to stitches
    depends_on: []
    size: medium
    description: "subtab-id: rename the ACE Artifacts sub-tab identifier from `commits`
      to `stitches` across `artifact_tabs.py` and every consumer, including the pane DOM
      ids and their `styles.tcss` selectors, without changing any displayed text.

      "
  - id: config-keys
    title: Rename user-configurable commits keymap and config keys
    depends_on:
      - subtab-id
    size: medium
    description: "config-keys: rename the `commits_*` app keymap actions to
      `stitches_*`, the `artifacts_commits` copy-mode group to `artifacts_stitches`, and
      the `ace.artifacts.commits` config block to `ace.artifacts.stitches`, each with a
      deprecated legacy name that still loads and warns.

      "
  - id: labels
    title: Flip user-visible Commits text to Stitches and refresh snapshots
    depends_on:
      - config-keys
    size: medium
    description:
      'labels: change every displayed "Commits" string that names the Artifacts pane to
      "Stitches" — tab strip, pane chip, placeholder copy, quickstart, help modal,
      command palette, footer — then update docs and regenerate the affected text and
      PNG snapshot goldens.'
proposed_by: bbugyi200.athena.xn
create_time: 2026-08-10 16:17:49
status: wip
---

- **PROMPT:**
  [prompts/202608/stitch_rename.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/stitch_rename.md)

# Plan: Rename `sase vcs` to `sase stitch` and the ACE Commits sub-tab to Stitches

## Background

SASE recently introduced **Stitch** as the name for its internal representation of a
commit: "the lightweight ordered change record inside a Patch's `STITCHES:` section.
Every VCS commit made through the tracked workflow has an associated numeric stitch, but
a stitch need not have a commit." Today a stitch is close to a commit, but the intent is
for the abstraction to diverge. This epic renames the two user-facing surfaces that
should carry the Stitch name now, before the abstraction diverges:

1. the `sase vcs` CLI command tree, and
2. the "Commits" sub-tab of the ACE TUI's "Artifacts" tab.

Some of this rename already happened: the fold-mode keymaps were renamed `cycle_commits`
-> `cycle_stitches` and `toggle_commits` -> `toggle_stitches`, with legacy names kept in
`_LEGACY_FOLD_KEY_ALIASES` in `src/sase/ace/tui/keymaps/registry.py`. That is the
pattern this epic follows for every other user-configurable name.

### Compatibility precedent to copy

The `changespec` -> `patch` rename is the reference implementation for a SASE rename
that must not break existing invocations or user config. Read it before starting:

- CLI alias: `_COMMAND_REGISTRARS` in `src/sase/main/parser.py` maps both `patch` and
  `changespec` to `register_patch_parser`; `register_patch_parser` in
  `src/sase/main/parser_patch.py` calls `add_parser("patch", aliases=["changespec"])`;
  `src/sase/main/entry.py` dispatches `if args.command in {"patch", "changespec"}`.
- Facade modules: `src/sase/main/parser_changespec.py` and
  `src/sase/main/changespec_handler.py` are two-line star re-export shims that keep the
  legacy import paths importable.
- Symbol aliases: `register_changespec_parser = register_patch_parser` at the bottom of
  `parser_patch.py`, exported through `__all__` with a `# legacy parser alias` comment.
- Keymap aliases: `_LEGACY_APP_KEY_ALIASES` and `_migrate_key_aliases()` in
  `src/sase/ace/tui/keymaps/registry.py`.
- Copy-mode group alias: `_migrate_copy_group_aliases()` in the same module.

### Scope boundaries

These names are **deliberately out of scope**. Do not rename them; a worker that
"helpfully" renames them will blow up the diff and break persisted data.

- `src/sase/vcs_log/`, `src/sase/vcs_list/`, `src/sase/vcs_provider/` and their
  `sase.core` facades/wires (`vcs_log_facade.py`, `vcs_log_wire.py`,
  `vcs_repo_stats_facade.py`, `vcs_repo_stats_wire.py`). These are the VCS-level engines
  behind the command and the pane, they mirror Rust-core names across the `sase-core`
  boundary, and `vcs_provider` is genuinely about version-control providers.
- The `commit:<sha>@<repo>` artifact-reference scheme and everything in
  `src/sase/artifact_ref*.py`. Artifact references appear in prompts and persisted
  artifacts; renaming the scheme is a separate, much larger decision.
- Domain models that describe real commits: `RepoCommit`, `VcsLogResult`,
  `CommitFilterSpec`, `CommitLogFilterValues`, the commit filter query language and its
  tokens, `commit_view_modal.py`, and revert-agent commit discovery.
- Python module, class, and function names inside `src/sase/ace/tui/widgets/artifacts/`
  (`commits_pane.py`, `CommitsPane`, `CommitsTimeline`, `build_commits_info`, ...). The
  pane still renders VCS commits; renaming these is churn without user benefit. Only the
  sub-tab identifier and DOM ids move in this epic.
- `src/sase/xprompt/tags.py`'s `vcs = "vcs"` workflow tag, `--vcs-provider` CLI options,
  and the `vcs` doctor check group. These name version control, not stitches.
- `CHANGELOG.md` (release-please owns it) and historical bead event streams under
  `sase/repos/beads/`.

### Known design wart to flag, not fix

`sase vcs list` renders the repository _constellation_ (primary, linked, sidecar repos
with branch, dirty state, commit counts, activity), not commits. After this rename, bare
`sase stitch` still delegates to `sase stitch list` via the central
`_default_list_subcommands()` convention, so `sase stitch list` will list repos — which
reads oddly under the Stitch name, and overlaps with the already-existing
`sase repo list`. This epic performs the rename the user asked for and does **not**
relocate that subcommand. The `cli` phase records a `PROPOSED FOLLOW-UP:` note on its
own bead proposing that the repo-constellation listing move under `sase repo` so
`sase stitch` can default to the timeline instead.

### Verification for every phase

Workspace directories are ephemeral, so always run `just install` first.

```bash
just install
just check
```

Run `just check-full` before the phase is considered done — this epic touches parsers,
keymaps, config schema, and TUI widgets, which is exactly the broadening set
`just check` warns about. The `labels` phase additionally needs `just test-visual`.

---

## Phase `cli`: Rename the sase vcs CLI command to sase stitch

Rename the command tree so `sase stitch` is canonical and `sase vcs` keeps working.

### Parser

- `git mv src/sase/main/parser_vcs.py src/sase/main/parser_stitch.py`.
- Rename `register_vcs_parser` -> `register_stitch_parser`. In it, call
  `subparsers.add_parser("stitch", aliases=["vcs"], ...)`.
- Rename the subparser dest `vcs_subcommand` -> `stitch_subcommand` (including
  `add_subparsers(dest=...)` and the explicit `set_defaults`). Nothing outside the
  parser/handler/tests reads this dest.
- Add `register_vcs_parser = register_stitch_parser  # legacy parser alias` and export
  both names through `__all__`, mirroring `parser_patch.py`.
- Update the module docstring, the `register_stitch_parser` docstring, the parser
  `help=`/`description=` text, and the `list`/`log` subparser descriptions so every
  example reads `sase stitch ...`. Mention in the group description that `vcs` remains a
  legacy alias, and keep the documented bare-default sentence ("With no subcommand,
  `sase stitch` defaults to `sase stitch list`.").
- Add `src/sase/main/parser_vcs.py` as a facade shim, byte-for-byte in the shape of
  `src/sase/main/parser_changespec.py`:

  ```python
  """Legacy ``sase vcs`` parser facade for :mod:`sase.main.parser_stitch`."""

  from sase.main.parser_stitch import *  # noqa: F403
  from sase.main.parser_stitch import __all__ as __all__
  ```

### Handler

- `git mv src/sase/main/vcs_handler.py src/sase/main/stitch_handler.py`.
- Rename `handle_vcs_command` -> `handle_stitch_command`, read `stitch_subcommand` from
  the namespace, and update the two `sase vcs log:` error prefixes and the
  `Usage: sase vcs {list,log}` fallback to say `sase stitch`.
- Keep `handle_vcs_command = handle_stitch_command  # legacy handler alias`, and add
  `src/sase/main/vcs_handler.py` as a facade shim mirroring
  `src/sase/main/changespec_handler.py`.
- The imports of `sase.vcs_list.collect.run_vcs_list` and
  `sase.vcs_log.collect.run_vcs_log` stay exactly as they are (see scope boundaries).

### Routing

- `src/sase/main/parser.py`:
  - In `_COMMAND_REGISTRARS`, replace the `"vcs"` entry with a `"stitch"` entry pointing
    at `("sase.main.parser_stitch", "register_stitch_parser")`, and add a `"vcs"` entry
    pointing at the _same_ spec with a `# Legacy command alias for the stitch parser.`
    comment. Both keys are required: the dict is deduplicated by spec when building the
    full parser, and `parser_only_hint()` uses it to decide whether the narrow-parser
    fast path is safe for `sase vcs ...`. Keep the dict alphabetically sorted.
  - In `_COMPACT_ROOT_COMMANDS`, rename the `vcs` entry to `stitch` and reword its
    summary, e.g. "Inspect primary, linked, and SDD repos, or show their stitch
    timeline." `_validated_compact_root_commands()` asserts the name exists in the root
    subparser choices, so this must match the new canonical name.
- `src/sase/main/parser_full_registrars.py`: import `register_stitch_parser` from
  `sase.main.parser_stitch` and update the `COMMAND_REGISTRARS_BY_NAME` entry. Keep the
  import block's existing ordering style.
- `src/sase/main/entry.py`: change the `# --- vcs ---` comment to `# --- stitch ---`,
  the guard to `if args.command in {"stitch", "vcs"}:  # legacy command alias`, and the
  import to `from .stitch_handler import handle_stitch_command`.

### Tests

- `git mv tests/main/test_vcs_parser.py tests/main/test_stitch_parser.py`; rewrite it
  against `stitch`/`stitch_subcommand` and rename the test class accordingly.
- Add coverage that the legacy spelling still resolves: `parse_sase_args(["vcs"])` and
  `parse_sase_args(["vcs", "log"])` produce `stitch_subcommand` values and the same
  defaults as the `stitch` spellings, and that `sase.main.parser_vcs` and
  `sase.main.vcs_handler` remain importable and expose the legacy symbol names.
- `tests/main/test_parser_command_defaults.py`: replace `"sase vcs"` with
  `"sase stitch"` in the expected list-group set. Check whether the walk visits the
  alias choice as a separate path; if it does, include `"sase vcs"` as well rather than
  special-casing the walker.
- Update the module docstrings of the `tests/test_vcs_log_*.py` and
  `tests/test_vcs_list_*.py` files that say `sase vcs log` / `sase vcs list` to the new
  command spelling. Leave those files' names and imports alone — they test the
  `vcs_log`/`vcs_list` engines, which are out of scope.

### Docs

- `docs/vcs.md`: rename the `### \`sase vcs list\``/`### \`sase vcs
  log\``headings and every example command to`sase stitch
  ...`. Add one sentence noting `sase vcs` is still accepted as a deprecated alias. Keep
  the file name and the mkdocs nav entry ("VCS Providers") — the page also documents VCS
  providers.
- `docs/cli.md`: update the two table rows and their anchor links
  (`vcs.md#sase-vcs-list` -> `vcs.md#sase-stitch-list`, same for `log`). Verify the
  anchors match the new heading slugs.
- `docs/configuration.md`: update the `### \`sase
  vcs\``section heading and body (around line 3218) and the`sase vcs log` mention near
  line 760.
- Grep the whole repo for `sase vcs` afterward and confirm the only remaining hits are
  `CHANGELOG.md` and `sase/repos/beads/` history.

### Follow-up note

Record on this phase's bead:
`PROPOSED FOLLOW-UP: sase stitch list renders the repository constellation, not stitches, and overlaps sase repo list. Consider moving repo-constellation listing under sase repo so bare sase stitch can default to the commit/stitch timeline.`

---

## Phase `subtab-id`: Rename the ACE Artifacts sub-tab identifier to stitches

Rename the internal identifier only. **No displayed text changes in this phase** — that
keeps the diff reviewable and keeps snapshot goldens untouched until the `labels` phase.
Verify that by confirming `just test-visual` still passes with the existing goldens.

### Canonical constants

`src/sase/ace/tui/artifact_tabs.py` is the spine. Change `commits` -> `stitches` in
`ArtifactsSubTab`, `ArtifactsPaneKey`, `DEFAULT_ARTIFACTS_SUBTAB`,
`ARTIFACTS_SUBTAB_ORDER`, `ARTIFACTS_PANE_IDS`, and `ARTIFACTS_ACCENTS`. Preserve the
existing order and the `#FFD700` accent. `src/sase/ace/tui/widgets/artifacts/types.py`
only re-exports, so it needs no change.

### Consumers

Roughly 87 `"commits"` literal sites in `src/` are the sub-tab identifier; the rest of
the `commits` hits are unrelated (revert-agent plans, update toasts, statistics tables,
plugin browser). Work from `grep -rn '"commits"' --include=*.py src/` and change only
the ones typed as `ArtifactsSubTab` / `ArtifactsPaneKey` / an artifacts pane key. Known
sites include:

- `src/sase/ace/tui/_app_action_availability.py`
- `src/sase/ace/tui/actions/artifacts.py`, `actions/artifacts_navigation.py`,
  `actions/artifacts_commits.py`, `actions/base.py`, `actions/_state_init.py`
- `src/sase/ace/tui/actions/clipboard/_palette.py`, `_palette_artifacts.py`,
  `_palette_artifact_previews.py`, `_artifact_references.py`,
  `_artifact_reference_resolution.py`, `_artifacts.py`
- `src/sase/ace/tui/commands/types.py`, `commands/availability.py`,
  `commands/_app_metadata.py`, `commands/_mode_commands.py`
- `src/sase/ace/tui/widgets/_keybinding_modes.py`, `widgets/patch_onboarding.py`,
  `widgets/artifacts/view.py`, `widgets/artifacts/panes.py`,
  `widgets/artifacts/commit_filter_bar.py`, `widgets/artifacts/commits_rendering.py`,
  `widgets/artifacts/commits_filtering.py`, `widgets/artifacts/commits_detail.py`

Two adapters need care rather than a blind rename:

- `src/sase/artifact_ref_entries.py`'s `expected` map keys the _subtab_ name to an
  artifact-ref kind. Add `"stitches": "commit"` and keep the existing `"commits"` and
  `"commit"` keys — the ref kind itself stays `commit` (scope boundary). The same
  applies to any other subtab-string-to-kind mapping found by grep.
- `src/sase/ace/tui/actions/clipboard/_artifact_references.py` maps subtab -> attribute
  name (`"commits": "_commits_pane"`). Rename the key, not the attribute.

### DOM ids and styles

Rename the pane-scoped Textual ids so they match the new identifier:

- `artifacts-commits-pane` -> `artifacts-stitches-pane`
- `commits-detail-scroll` -> `stitches-detail-scroll` (in `_DETAIL_SCROLL_IDS` in
  `widgets/artifacts/view.py`)
- the pane-internal ids `#commits-info`, `#commits-info-header`, `#commits-legend-row`,
  `#commits-position`, `#commits-legend`, `#commits-main`, `#commits-list-container`,
  `#commits-timeline`, `#commits-footer`, `#commits-detail-container`, `#commits-detail`
  -> the `stitches-` equivalents.

Update `src/sase/ace/tui/styles.tcss` (~15 selectors around lines 79-360) in lockstep.
Leave the unrelated `PluginActionConfirmModal ... commits` selectors near line 6886 and
the revert-agent comment near line 1457 alone. After the rename,
`grep -n 'commits' src/sase/ace/tui/styles.tcss` should only show those unrelated hits.
A missed `query_one("#commits-...")` raises at runtime rather than at import, so rely on
the pane's interaction tests, not just lint.

### Tests

About 54 test files reference the sub-tab identifier, pane ids, or keymap names. Start
from:

```bash
grep -rln '"commits"\|artifacts-commits-pane\|commits-detail-scroll\|#commits-' tests/
```

Update the sub-tab/pane-id usages. Do **not** rename the test files themselves in this
phase (`tests/ace/tui/test_commits_pane_*.py`, `tests/ace/tui/_commits_pane_helpers.py`)
— they test the commits-rendering widgets whose names are out of scope, and renaming
them would churn `tools/select_tests`' import-graph closure for no benefit.

Add a regression test asserting `DEFAULT_ARTIFACTS_SUBTAB == "stitches"` and that
`ARTIFACTS_SUBTAB_ORDER` still leads with it, so a later refactor cannot silently
reorder the Artifacts tab.

---

## Phase `config-keys`: Rename user-configurable commits keymap and config keys

Every name in this phase is something a user can put in `~/.config/sase/sase.yml`, so
each rename ships with a legacy alias that still loads and logs a deprecation warning.

### App keymap actions

Rename the ten actions `commits_next`, `commits_prev`, `commits_view_selected`,
`commits_copy_sha`, `commits_filters`, `commits_toggle_sdd`, `commits_cycle_merges`,
`commits_toggle_all_projects`, `commits_fetch`, `commits_refresh` to their `stitches_*`
spellings across:

- `src/sase/ace/tui/keymaps/app_keymaps.py` (dataclass fields)
- `src/sase/ace/tui/keymaps/metadata.py` (the `(action, label, ...)` tuples; leave the
  human labels for the `labels` phase)
- `src/sase/ace/tui/keymaps/defaults.py` and any scope/binding tables that name them
- `src/sase/default_config.yml` (the `# Commits sub-tab` block, ~line 282)
- `src/sase/ace/tui/bindings.py` (the `Binding(...)` action strings)
- `src/sase/ace/tui/commands/_app_metadata.py` and `commands/availability.py` (including
  the `"app.commits_*"` command ids)
- `src/sase/ace/tui/widgets/artifacts/commits_rendering.py` (`actions.commits_*`
  attribute reads in `build_commits_hints`)
- `src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py`

Add the ten legacy entries to `_LEGACY_APP_KEY_ALIASES` in
`src/sase/ace/tui/keymaps/registry.py` with `# legacy compatibility alias` comments.
`_migrate_key_aliases()` already handles both the rename and the "both configured"
conflict warning. Follow the `next_changespec` precedent in `app_keymaps.py` and add
deprecated property aliases only if something in-tree still reads the old attribute
names; prefer renaming the readers.

### Copy-mode group

Rename the `artifacts_commits` copy-mode group in
`src/sase/ace/tui/keymaps/mode_keymaps.py` (~line 67) to `artifacts_stitches`, and
update `src/sase/ace/tui/commands/_mode_commands.py` (~line 153) plus every reader found
by grep. `_migrate_copy_group_aliases()` in `registry.py` currently hard-codes the
single `changespecs` -> `patches` case; generalize it to a `{legacy: canonical}` mapping
table (keeping the existing entry and its warning text shape) and add
`artifacts_commits` -> `artifacts_stitches`.

### Config block

Rename `ace.artifacts.commits` to `ace.artifacts.stitches`:

- `src/sase/default_config.yml` (~line 124): rename the block, keep the comment.
- `src/sase/config/sase.schema.json` (~line 557): the `artifacts` object has
  `additionalProperties: false`, so add a `stitches` property with the current
  `default_query` schema **and keep** a `commits` property whose description marks it
  deprecated in favor of `stitches`. Removing it would make existing user configs fail
  validation. Update the `default_query` description text to say "Stitches filter".
- `src/sase/ace/tui/widgets/artifacts/commit_config.py`: `_resolve...` currently reads
  `artifacts.get("commits")`. Read `stitches` first; fall back to `commits` when
  `stitches` is absent, logging a deprecation warning. If both are present, use
  `stitches` and warn that `commits` is ignored — same semantics as the keymap alias
  path. Update `_CONFIG_PATH` to the new dotted path and make the fallback's diagnostics
  name whichever key was actually read.
- `docs/configuration.md` (`#### ace.artifacts.commits`, ~line 735) and `docs/ace.md`
  (~line 219): document the new key and note the deprecated alias.

### Tests

- Extend `tests/ace/tui/test_commits_config.py` with cases for the new key, the legacy
  key falling back, and both keys present.
- Add keymap-alias coverage next to the existing legacy-alias tests (grep for
  `next_changespec` in `tests/`) covering a `commits_*` override migrating to
  `stitches_*`, and the copy-group alias.
- Update `tests/test_keymaps_app_bindings.py`, `tests/test_keymaps_e2e.py`,
  `tests/test_command_availability_scope.py`, and the copy-palette helper tests that
  name the old actions/group.

---

## Phase `labels`: Flip user-visible Commits text to Stitches and refresh snapshots

Only now does displayed text change. The rename target is the _pane name_; leave per-row
wording that describes an individual commit ("Copy full commit SHA", "Open full commit
message and diff", "Next Commit") as commit-language for now, since the pane still shows
real commits — but rename anything that names the pane or the sub-tab itself.

### Strings to change

- `src/sase/ace/tui/widgets/artifacts/view.py`: `_ARTIFACT_LABELS["stitches"]` ->
  `"Stitches"`. This is the tab-strip label.
- `src/sase/ace/tui/widgets/artifacts/commits_rendering.py` (~line 64): the
  `" Commits "` accent chip -> `" Stitches "`.
- `src/sase/ace/tui/widgets/artifacts/panes.py`: the `_PLACEHOLDER_COPY` entry title and
  body. Note `_scope_text()` renders `self.subtab.title()`, which now yields "Stitches"
  automatically — confirm that reads correctly.
- `src/sase/ace/tui/widgets/artifacts/commit_filter_bar.py` (~line 127): docstring
  mentioning "the Commits legend".
- `src/sase/ace/tui/widgets/tab_quickstart.py` (~lines 245, 255): the "Jump: Commits ·
  Beads · ..." and "Cycle Artifacts: ..." strings.
- `src/sase/ace/tui/widgets/patch_onboarding.py` (~line 37): the sub-tab blurb.
- `src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py`: the
  `"Commits Pane"` section title and the `"Pick; Commits rewrites project:"` line.
- `src/sase/ace/tui/modals/help_modal/patches_copy_bindings.py` (~line 31):
  `Copy Mode · Commits` -> `Copy Mode · Stitches`.
- `src/sase/ace/tui/commands/_app_metadata.py`: the `"Commits: ..."` palette labels and
  the `("next artifact", "commits", ...)` search-alias tuple (~line 78). Keep
  `"commits"` as an extra alias term there so palette muscle memory keeps working, and
  add `"stitches"`.
- `src/sase/ace/tui/keymaps/metadata.py`: labels such as `"Fetch Commits"` /
  `"Refresh Commits"` that name the pane -> "Stitches"; leave "Next Commit" / "Copy
  Commit SHA" alone.
- `src/sase/ace/tui/bindings.py`: the `# Commits sub-tab actions.` comment and the two
  `Binding(...)` descriptions that name the pane.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_artifacts_lane.py` (~line 65): this
  `"  Commits:\n"` header lists an agent's commits, **not** the Artifacts pane. Decide
  deliberately; the default is to leave it as "Commits".

Re-read `src/sase/ace/CLAUDE.md` before touching the help modal: box sections are pinned
to `_BOX_WIDTH = 57` / `_CONTENT_WIDTH = 50`, and keybinding descriptions truncate at 32
characters. "Stitches" is one character longer than "Commits", so re-check every line
whose length is near a limit.

### Docs

- `docs/ace.md`: rename the Artifacts sub-tab discussion of the Commits pane.
- `docs/configuration.md`: the prose around the ACE keymap and artifacts sections that
  names the "Commits pane".
- `docs/vcs.md` (~line 199): "ACE Artifacts Commits pane's configurable persistent
  query" -> Stitches.

### Snapshots

Displayed text changes, so text-comparing TUI tests and every PNG golden that shows the
Artifacts sub-tab strip will fail until refreshed.

```bash
just test                    # fix text/assertion failures first
just test-visual             # see which goldens moved
just update-visual-snapshots # accept the intentional visual change
just test-visual             # confirm clean
```

Affected goldens include `artifacts_commits_*`, `artifacts_beads_*`, `artifacts_bugs_*`,
`artifacts_chats_*`, `artifacts_files_*`, `artifacts_plans_*` (all render the sub-tab
strip), and `copy_as_commits_*`. Also rename the snapshot ids that name the sub-tab —
the string literals in `tests/ace/tui/visual/test_ace_png_snapshots_commits.py` and
`test_ace_png_snapshots_copy_as_palette.py` — from `artifacts_commits_*` /
`copy_as_commits_*` to `artifacts_stitches_*` / `copy_as_stitches_*`, `git mv` the
corresponding PNGs, and regenerate. Inspect `.pytest_cache/sase-visual/` diffs before
accepting: the only pixel changes should be the renamed labels and the width they
occupy.

### Final sweep

```bash
grep -rn 'Commits' --include=*.py --include=*.md --include=*.yml --include=*.tcss \
  src/ docs/ tests/ | grep -iv 'plugin\|toast\|statistic\|revert\|receipt\|bead_pages'
```

Every surviving hit should be deliberate commit-language, not a stale pane name.
