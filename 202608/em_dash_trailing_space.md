---
tier: tale
title: Add a trailing space to the em-dash snippet expansion
goal:
  The Obsidian double-hyphen snippet expands to an em dash plus a trailing space with
  the cursor after that space.
size: small
proposed_by: bbugyi200.athena.0e9
create_time: 2026-09-09 20:00:02
status: wip
---

# Add a trailing space to the em-dash snippet expansion

## Goal

Adjust the existing `bob-ledger-tools` Obsidian snippet so expanding an exact `--`
trigger through Tab or the **Expand Bob snippet** command replaces it with an em dash
followed by one ASCII space (`— `) and leaves the cursor after that added space.

Keep the existing parser, shared expansion path, single-cursor guard, and Markdown-safe
rejection of `---` and longer hyphen runs intact. This is a focused behavior update to
the recently added em-dash snippet, not a new trigger or keybinding.

## Behavioral contract

- The replacement owned by the `emDash` snippet is exactly two JavaScript UTF-16 code
  units: U+2014 EM DASH followed by U+0020 SPACE. For example, `--|` becomes `— |`,
  where `|` marks the cursor.
- The terminal cursor position is after the emitted ASCII space. The shared default
  cursor-offset calculation should continue deriving this from the replacement length;
  do not introduce a special editor path solely for this trigger.
- Preserve all text outside the two-character trigger exactly. In particular,
  `left--|right` becomes `left— |right`, and `--|suffix` becomes `— |suffix`.
- Add the trailing space unconditionally, matching the existing `D[-]<N>` dated-entry
  snippet convention. Do not inspect, consume, or deduplicate an existing suffix:
  `--| suffix` therefore becomes `— | suffix` with the authored suffix space still
  present.
- Continue refusing any `--` pair adjacent to another ASCII hyphen on either side, so
  `---`, longer runs, Markdown thematic breaks, and YAML frontmatter delimiters remain
  untouched and leave Tab available to Obsidian.
- Preserve the current single-cursor/collapsed-selection guard, handled/unhandled Tab
  semantics, and every existing `ta`, `se...`, `t...`, `d...`, `D...`, and `dt...`
  snippet contract.

## Repository and file scope

The implementation belongs only in the linked `bob-plugins` source-of-truth repository.
Open it with
`sase repo open bob-plugins -r "Implement the approved em-dash trailing-space plan"` and
use the checkout path printed by that command. Do not edit the installed plugin under
the Bob vault directly, and do not change `bob-cli` source.

Expected files in `bob-plugins`:

- `plugins/bob-ledger-tools/main.js`
- `scripts/test-ledger-tools-snippets.cjs`
- `plugins/bob-ledger-tools/manifest.json`
- `README.md`

No SASE memory changes are needed.

## Implementation

1. In `plugins/bob-ledger-tools/main.js`, change the `emDash` snippet's replacement
   value from `—` to `— ` while keeping its trigger parsing and boundary checks
   unchanged. Continue routing the result through `computeSnippetExpansion`,
   `findExpansion`, `expansionCursorCh`, `expandLineAtCursor`, and `expandFromEditor`;
   the shared replacement-length fallback should naturally put the cursor after the new
   trailing space.
2. Update the focused em-dash coverage in `scripts/test-ledger-tools-snippets.cjs` to
   assert the exact U+2014/U+0020 replacement and its length, plus the new line text and
   cursor columns for whole-line, prefix, suffix, and inline expansions. At editor
   level, assert that only the `--` source range is replaced by `— `, the handler still
   returns `true`, and the cursor lands after the space.
3. Add or retain focused regression cases demonstrating both boundary invariants:
   multi-hyphen runs remain unhandled, and pre-existing suffix text/whitespace is not
   consumed or normalized. Keep representative assertions that all other snippet kinds
   continue to parse and expand as before.
4. Release the behavior adjustment as patch version `1.3.1`: update
   `plugins/bob-ledger-tools/manifest.json` and the Bob Ledger Tools version in the
   README table. Revise the README's `--` example and cursor description to document
   `— `, including that the cursor is placed after the trailing space; retain the
   explanation that three-or-more-hyphen runs are excluded.

## Validation and deployment

From the checkout printed by `sase repo open`:

1. Run `node --test scripts/test-ledger-tools-snippets.cjs` and confirm all focused
   parsing, expansion, cursor, editor, and regression tests pass.
2. Run `npm test` to cover the full plugin repository and detect shared snippet or
   keymap regressions.
3. Run `npm run validate` to syntax-check plugin sources and validate all manifests,
   including the `1.3.1` semver.
4. Review the final diff and confirm it is limited to the four expected files and does
   not alter trigger recognition, selection guards, Tab priority, or unrelated plugin
   behavior.
5. Satisfy `bob-plugins`' mandatory vault deployment rule by previewing only this
   plugin:
   `bob plugins sync --no-pull --dry-run --plugin bob-ledger-tools --repo <opened-bob-plugins-path>`.
   Confirm that only the expected managed `main.js` and `manifest.json` copies differ.
6. Deploy with the same command without `--dry-run`. Do not add `--force` if the vault's
   dirty-file guard refuses an overwrite; investigate and preserve unexpected vault
   drift instead.
7. Run `bob plugins list --no-pull --format json --repo <opened-bob-plugins-path>` and
   confirm `bob-ledger-tools` reports version `1.3.1`, `sync: "synced"`, and
   `vault: "enabled"`.

## Acceptance criteria

- Expanding exact `--` through either supported snippet entry point produces exactly
  `— ` and positions the cursor after the ASCII space.
- Prefix and suffix content remain byte-for-byte unchanged; the added space is always
  part of the replacement and existing suffix whitespace is not deduplicated.
- No pair within `---` or a longer hyphen run expands, and failed matches preserve
  Obsidian's normal Tab behavior.
- Existing snippet triggers, selection guards, and shared editor handling remain
  unchanged.
- The manifest and README consistently report Bob Ledger Tools `1.3.1` and document the
  new output/cursor contract.
- Focused tests, the full repository suite, source/manifest validation, deployment
  preview, vault sync, and the final synced/enabled status check all pass.
