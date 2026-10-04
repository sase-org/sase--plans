---
tier: tale
title: Finish landing epic sase-1fv by resolving macro-loader source IDs in the mini-macro
  catalog
goal: The Ctrl+G x existing-definition finder lists each config, built-in, and plugin
  macro exactly once with its real path and correct active/shadowed status. Legacy
  xprompts-keyed config macros are editable in place, the name-step warnings name
  the real winner, and epic sase-1fv is closed with its plan marked done.
size: small
proposed_by: bbugyi200.athena.sase-1fv.land
bead: sase-1fv
status: done
---

- **PARENT:**
  [202610/existing_macro_snippet_editing.md](https://github.com/sase-org/sase--plans/blob/main/202610/existing_macro_snippet_editing.md)
- **BEAD:**
  [sase-1fv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1fv/README.md)

# Finish landing epic sase-1fv: resolve macro-loader source IDs in the mini-macro catalog

Epic `sase-1fv` ("Edit existing macros and snippets from the Ctrl+G x / Ctrl+G t
location picker", plan `plan:202610/existing_macro_snippet_editing.md`) has all six
phases closed. Its land agent verified every phase commit (a963c0d3da, 763cc9fca3,
2608a2439e, 4d38e39702, d6e856f4d9, 807e0fa107) and the docs, and found no
`--epic-symbol` leftovers. It then drove the real TUI with `sase screenshot`. The
snippet path works end to end (picker `e` row → finder → `● active` row → in-place
pane). The macro path has one defect introduced by phase `sase-1fv.1`
(macro-redefinition). This tale fixes that defect and then closes the epic.

## The defect (root cause is known)

`sase.macro.get_all_macros()` does not report a file path in `Macro.source_path` for
config-defined and plugin macros. It reports a **loader source ID**: `config`,
`default_config`, `local_config`, `config_overlay:<file>`, `project_local_config:<p>`,
`plugin_config:<module>`, `plugin:<module>/<file>.md`, or `builtin:sase/<rel>`.
`sase.macro.macro_sources.definition_file_for_source(source_id)` already maps each of
these IDs to its file. For the user config, it returns the deployed
`~/.config/sase/sase.yml`, and `resolve_macro_write_target(...).write_path` then maps
that to the chezmoi source path that the save rows use.

`src/sase/ace/tui/modals/mini_macro_target_catalog.py` compares these IDs as if they
were paths:

- `_load_catalog_only_definitions` dedupes on `(name, _normalized_path(source_path))`.
  `_normalized_path("config")` is a bogus cwd-relative path, so every config-defined and
  plugin-markdown macro that already has a row definition is added a second time as a
  catalog-only `read_only` definition. Its `display_path` is the bare label `config` or
  `default_config`, and its preview fails, because `_load_definition_markdown` reads the
  label as a file.
- `_loader_matching_definition` (the runtime-winner oracle) then matches the bogus
  duplicate rather than the real row definition. So the duplicate becomes `effective`,
  and the real definition is marked `shadowed_by="default_config"` / `"config"`.

Observed live on this host (project `sase`):

- The `#review` finder query lists
  `#review 🔒 built-in ./src/sase/default_config.yml:review` with preview
  `shadowed by default_config`, and a second row, `#review 🔒 read-only default_config`.
- Overriding built-in `#bd/work_task` into User config shows the wrong name-step copy:
  `⚠ #bd/work_task already exists in default_config — saving to ~/…/sase/sase.yml adds another definition`.
  Because the active definition is in a row, the correct copy is the "will override it"
  row of the shared copy table.
- The research plugin's `plugin:sase_research_artifacts/*.md` macros are duplicated the
  same way.

A second, related gap is in `_load_config_definitions`, which only reads
`payload["macros"]`. A config that still uses the retired `xprompts:` key (accepted
while the `legacy_xprompt_syntax` flag is on, as on this host's user config: 28 macros)
produces **no** row definitions. Its macros therefore show only as the bogus
`🔒 read-only config` rows, so the user's main config macros cannot be edited in place
from the finder. The rest of the stack already normalizes this key:
`sase.macro.save_index._config_names` (which fills `row.names`) and
`sase.macro.save.load_config_macro_markdown` both call
`sase.legacy_xprompt_syntax.normalize_frontmatter_macros`. The config writer
(`insert_macro_into_config`) migrates a legacy `xprompts:` header to `macros:`, so
in-place saves are safe.

## Changes

1. **Resolve loader source IDs once per catalog load** in
   `src/sase/ace/tui/modals/mini_macro_target_catalog.py`:
   - Add a helper that returns `source_path` unchanged when it is empty or an absolute
     path. Otherwise it returns `str(definition_file_for_source(source_path))`, keeping
     the original ID when the resolver returns `None`.
   - In `load_mini_macro_target_catalog`, right after `get_all_macros(...)`, build the
     resolved mapping once, for example
     `{name: dataclasses.replace(macro, source_path=<resolved>)}`. `Macro` is a plain
     `@dataclass`. Pass that mapping to both `_load_catalog_only_definitions` and
     `_annotate_precedence`, so the dedupe and the oracle both compare real files. Do
     not call the resolver per definition or per comparison: plugin resolution is the
     slow part. The catalog already loads off the event loop, so do not add any
     event-loop I/O.
   - For a catalog-only definition whose resolved source is a YAML file (`.yml` /
     `.yaml`), set `storage_format=SaveTargetFormat.CONFIG`, `entry_name=<name>`, and
     `display_path=f"{_short_path(path)}:{name}"` (the same shape row config definitions
     use). The finder preview (`_load_definition_markdown`) then loads that entry. A
     namespaced name missing from the file only degrades to the finder's existing dim
     preview error line. Keep markdown sources as they are, now with a real path.
2. **Normalize the retired config key** in `_load_config_definitions`. Replace the
   `payload.get("macros")` check with
   `normalize_frontmatter_macros(payload, source=row.location.path)` from
   `sase.legacy_xprompt_syntax`, mirroring `save_index._config_names`. On `ValueError`
   (both spellings present, or legacy while the flag is off), yield nothing. Feed the
   normalized mapping to `parse_macro_entries`.
3. **Keep the file under 700 lines.** `mini_macro_target_catalog.py` is at 673 lines,
   and the plan's file-size rule (toobig thresholds `1000 850 700`) applies. If the
   changes would pass 700, move the loader-source helpers (`_normalized_path`,
   `_loader_matching_definition`, and the new resolver) into a new private sibling
   module under `src/sase/ace/tui/modals/`. Do not add new public symbols, since
   Symvision would flag them.
4. **Integration cleanup** in `src/sase/ace/tui/modals/existing_definition_entries.py`:
   replace both `site.kind == "xprompt"` comparisons (in `_snippet_origin` and
   `_snippet_chip`) with `is_macro_derived_kind(site.kind)` from `sase.snippet.models`.
   This matches `snippet_name_analysis.py` and `sase.snippet.redefinition`, which phase
   `sase-1fv.2` deliberately kept clear of `xprompt` literals.

## Tests

Extend `tests/ace/tui/modals/test_mini_macro_target_catalog.py`. It already has the
`_row`, `_write_config`, and `catalog_mod` monkeypatch helpers. The existing tests only
use absolute loader paths, which is why this slipped through. Monkeypatch the resolver
name as imported into whichever module ends up using it.

- The loader reports `source_path="config"` and the resolver maps it to a config row's
  file. The catalog then has exactly one definition for that name: the editable row
  definition, `effective is True`, with no `shadowed_by`.
- A `default_config` label against a `Built-in (dev)` config row yields one definition
  with `origin_label == "built-in"`.
- A `plugin:<module>/<file>.md` label against a plugin directory row yields one
  definition.
- When the resolver returns `None` for an unknown label, the definition stays a
  catalog-only `read_only` definition, with no crash.
- A label that resolves to a YAML file that is not a row gives a catalog-only definition
  with `storage_format is SaveTargetFormat.CONFIG`, `entry_name == name`, and a
  `<short path>:<name>` display path.
- A config row whose file uses the legacy `xprompts:` key, with legacy syntax accepted,
  yields editable row definitions. Look at how existing tests toggle
  `legacy_xprompt_syntax` (`grep -rn legacy_xprompt_syntax_enabled tests/`). With both
  `macros:` and `xprompts:` present, the row yields nothing and does not raise.
- Regression for the reported copy (in
  `tests/ace/tui/modals/test_mini_macro_name_modal.py` or next to the
  `mini_macro_redefinition` tests): the loader's active source is the `default_config`
  label for a name defined in a built-in config row, and the destination is a writable
  User config row. `macro_redefinition_warning` must use the "already exists in
  <built-in path> — saving to <dest> will override it" copy, not "adds another
  definition".
- In `tests/ace/tui/modals/test_existing_definition_entries.py`, a user-config macro
  reported via the `config` label produces one `active` entry and no `read_only`
  duplicate.

## Verification

- Run `sase tool run check` (the wrapped `just check`). Hand it to `/sase_monitor` if it
  may exceed the synchronous limit. Do not run `just check-full`. A KNOWN/FLAKY-only
  result on files this tale did not touch is not epic work.
- Run targeted
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_existing_finder.py tests/ace/tui/visual/test_ace_png_snapshots_mini_macro.py tests/ace/tui/visual/test_ace_png_snapshots_save_location_picker.py`
  (through `/sase_monitor` when long). The fixtures use absolute paths, so the goldens
  are expected to be unchanged. Read the report and inspect any golden that changed.
- Do a live check in the real app. Run the workspace's own binary
  (`.venv/bin/sase screenshot --keep -o /tmp/land0.png`), because `sase screenshot`
  launches the TUI with its own interpreter, so the global `sase` would show the
  installed build. Drive the printed `sase_tmux_target` with `tmux send-keys`:
  1. Send a literal `+` with `tmux send-keys -t <target> -l '+'` (the tmux key name
     `plus` does not work), type `sase`, and press `Enter` to open the prompt bar.
  2. Press `C-g` then `x`, then `e`, then type `review`. The finder must show one
     `#review` row with a real `…default_config.yml:review` path and no
     `🔒 read-only default_config` duplicate.
  3. Query a macro from the user config (for example `bd/review_tasks`). It must be
     `● active` (or `◐ shadowed` with a real path), not `🔒 read-only config`, and
     `Enter` must open the mini-macro pane with its body. Discard with `Ctrl+C` and do
     not save.
  4. Inspect the PNG captures with `--window <target>`, then
     `tmux kill-window -t <target>`.

## Closeout of epic sase-1fv (final step: do it in this same turn)

The follow-up triage is already recorded on the epic (note "LAND FOLLOW-UP TRIAGE"):
every phase `PROPOSED FOLLOW-UP` was either already resolved on HEAD or already
corroborated on `sase-18t`, `sase-1fn`, `sase-o7`, or `sase-1g0`. No new task beads are
needed. Do not wait for, or order anything after, this tale's own commit; the host
commits after the turn.

1. Run `sase bead epic-symbols sase-1fv`. It listed none at landing time. If any entry
   appears, resolve it (wire it up, privatize it, add a non-test pragma, or delete it
   per the Symvision epic-whitelist policy). No later bead remains open to re-key it
   onto.
2. Close the epic with `sase bead close sase-1fv --note "<verification>"`. Base the note
   on this text, extended with your own results (check ToolRun id, test counts,
   live-check outcome): "Verified all six phases against their commits (a963c0d3da,
   763cc9fca3, 2608a2439e, 4d38e39702, d6e856f4d9, 807e0fa107): picker Existing row +
   override mode, finder modal with chips/preview, macro and snippet in-place edit,
   override detour, replace_draft + dirty guard, hint labels, docs/ace.md +
   docs/prompt.md. Live sase screenshot confirmed snippet picker → finder → in-place
   pane and the macro override/refusal paths. Landing fixed the phase-1 catalog defect
   where loader source IDs (config, default_config, plugin:…) were compared as paths,
   duplicating config/plugin macros as read-only rows and inverting the
   active-definition oracle, and taught the catalog the retired xprompts: config key;
   replaced finder xprompt literals with is_macro_derived_kind. Drift since the epic
   started (781bb0e7ae terminology rename, ed87435ed3 test split, aa8b98ffd5 epic-symbol
   re-key/allowlist, others) needed no further integration. Follow-up triage: see the
   LAND FOLLOW-UP TRIAGE note; no new beads." Never use `--force` to make the close
   succeed. If the close is rejected for leftover epic-symbols, clean them up and close
   again.
3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file (the PLAN path that
   `sase bead read sase-1fv -r "Need the plan path to mark it done"` prints,
   `plan:202610/existing_macro_snippet_editing.md`; it currently reads `status: wip`).
5. `sase-1fv` has no `parent_bead`, so nothing more needs closing after the epic.
