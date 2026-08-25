---
tier: tale
title: Show GitHub owners for community plugins in the Admin Center
goal: "Community plugin rows in SASE Admin Center Updates > Plugins always show their
  GitHub owner and repository, such as bbugyi200/bugyi-chops, while built-in rows and
  existing plugin selection and action behavior remain unchanged.

  "
size: small
proposed_by: bbugyi200.athena.0d7
create_time: 2026-08-25 07:02:03
status: wip
---

# Plan: Show GitHub owners for community plugins

## Current behavior and constraints

- `PluginCatalogEntry` already carries the GitHub `owner`, repository `repo`, and
  canonical `full_name`; the GitHub source normalizes `full_name` to `owner/repo`, and
  the plugin detail panel already renders it.
- The Updates > Plugins list loses that ownership context because
  `PluginsBrowserRenderingMixin._row_text()` always renders the short `entry.name`.
- The list's precomputed filter haystack omits `owner` and `full_name`, so an owner
  shown in a row would otherwise not be searchable.
- Grouping, sorting, option IDs, session bookmarks, lookup maps, marks, and plugin
  operations currently use the short name. This plan changes presentation, not those
  identities or action inputs.
- Keep rendering pure and constant-time: do not add filesystem, subprocess, or network
  work to the UI path, and retain the catalog-load-time haystack cache.

## Implementation

1. In `src/sase/ace/tui/modals/plugins_browser_rendering.py`, add a small pure
   display-label helper used by `_row_text()`:
   - Built-in plugins continue to show `entry.name`.
   - Community plugins show the canonical `entry.full_name` (`owner/repo`) in every row
     state, including installed, available, update-available, marked, verbose, and
     jump-hint-decorated rows.
   - Provide a deterministic compatibility fallback from the already-loaded owner and
     repository fields, then the existing short name, so an incomplete or older cached
     entry never renders a blank label and includes its owner whenever that metadata is
     available.
   - Keep the existing status glyphs, version/update text, styles, section ordering, and
     short-name-backed option IDs/bookmarks/actions unchanged.

2. Extend `_plugin_haystack()` using the same already-loaded metadata so filtering
   matches the newly visible community owner/full-name text. Continue building the
   joined, case-folded haystack once per catalog load rather than adding work per
   keystroke.

3. Add focused regression coverage under `tests/ace/tui/` that proves:
   - a normal community row renders `acme-corp/sase-acme` rather than only `acme`;
   - built-in rows still render their short names rather than `sase-org/sase-github`;
   - owner/full-name filtering selects the community plugin; and
   - incomplete community metadata follows the documented fallback without changing the
     short-name identity used for selection and actions. Retain the existing grouping,
     version, update-marker, verbose, and jump-hint assertions as regression coverage
     for every row decoration path.

4. Regenerate only the intentional Updates > Plugins PNG goldens affected by the longer
   community row label, including the highlighted community-detail view, and inspect the
   image diffs to confirm the owner/repository text remains legible in the master list
   without unrelated layout or detail-panel changes.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral SASE
   workspace.
2. Run the focused non-visual plugin-browser tests, including loading/filtering, detail,
   verbose, and jump-hint coverage.
3. Run the Config Center Plugins visual snapshot file with
   `--sase-update-visual-snapshots`, inspect the changed actual/expected/diff images,
   and rerun it without the update flag.
4. Run `just check` for the repository-wide lint gates and diff-scoped test lane.

## Out of scope

- Changing the CLI `sase plugin list` table, which is not the requested Admin Center
  sub-tab.
- Renaming plugin identities, changing catalog sorting/grouping, or redesigning
  duplicate-short-name handling.
- Fetching additional GitHub metadata or changing the catalog/cache schema.
