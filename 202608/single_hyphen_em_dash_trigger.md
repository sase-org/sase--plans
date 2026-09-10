---
tier: tale
title: Change the em-dash snippet trigger to one hyphen
goal:
  Bob Ledger Tools expands one isolated hyphen into an em dash and trailing space
  without regressing other snippets or multi-hyphen Markdown syntax.
size: small
proposed_by: bbugyi200.athena.0f2
create_time: 2026-09-09 20:00:29
status: wip
---

# Change the em-dash snippet trigger to one hyphen

## Goal

Change Bob Ledger Tools' existing em-dash snippet so a single-cursor Tab press (or the
**Expand Bob snippet** command) immediately after exactly one ASCII hyphen (`-`)
replaces that one character with an em dash plus one ASCII space (`— `). This replaces
the current `--` trigger so the user types one fewer key; it does not add a second
alias, keybinding, or expansion path.

## Behavioral contract

- Match the exact one-character hyphen immediately before the cursor at column zero or
  adjacent to prose. With `|` marking the cursor, `-|` becomes `— |` and `left-|right`
  becomes `left— |right`.
- Preserve every character outside the trigger and keep the current unconditional
  trailing-space behavior. Existing suffix whitespace is not consumed or deduplicated,
  so `-| suffix` becomes `— | suffix`.
- Leave the cursor immediately after the emitted U+2014 EM DASH and U+0020 SPACE. A
  successful match continues to return the existing handled result so Tab is consumed.
- Recognize only an isolated hyphen. Reject a candidate touching another ASCII hyphen on
  either side, including a cursor at the end of `--` or a cursor between its two
  characters. Consequently, the former `--` trigger and every longer run remain
  unhandled and available to Obsidian, preserving Markdown thematic breaks and YAML
  frontmatter delimiters.
- A bare `-` at column zero is intentionally eligible; once a separating space exists,
  normal cursor placement after that space does not match. Any non-match must continue
  returning false so Obsidian retains its normal Tab behavior.
- Do not reinterpret incomplete negative-offset snippet tokens as em-dash requests. At
  valid snippet boundaries, `d-`, `D-`, `t-`, and `dt-` must remain unhandled while the
  user is composing a numeric offset. Preserve `se-` as its existing valid ledger range
  trigger and preserve all completed `ta`, `se...`, `t...`, `d...`, `D...`, and `dt...`
  expansions.
- Preserve the shared single-cursor/collapsed-selection guard, editor replacement path,
  highest-priority Tab keymap, and command behavior.

## Repository and file scope

The source-of-truth change belongs only in the linked `bob-plugins` repository. Open it
with `sase repo open bob-plugins` and use the path returned by that command. Do not edit
the deployed plugin under the Bob vault directly, and do not change `bob-cli` source.

Expected `bob-plugins` files:

- `plugins/bob-ledger-tools/main.js`
- `scripts/test-ledger-tools-snippets.cjs`
- `plugins/bob-ledger-tools/manifest.json`
- `README.md`

No SASE memory change is needed.

## Implementation

1. In `plugins/bob-ledger-tools/main.js`, update the punctuation-specific
   `parseEmDashTrigger` branch to recognize one trailing `-`, return `trigger: "-"`, and
   report a one-character `[startCh, endCh)` span. Keep this parser separate from the
   generic word-like trigger regular expression so prose adjacency remains allowed
   without weakening identifier boundaries for the other snippets.
2. Preserve the two-sided isolation rule by rejecting a preceding `-` in the parser and
   retaining the em-dash-specific following-`-` check in `isTriggerBoundaryBlocked`. Add
   an explicit fallback guard for boundary-valid incomplete `d-`, `D-`, `t-`, and `dt-`
   tokens before treating the final hyphen as the em-dash trigger; do not disturb
   generic parsing, particularly the already-valid `se-` range form.
3. Leave `EM_DASH_REPLACEMENT`, `computeSnippetExpansion`, cursor-offset derivation,
   `findExpansion`, `expandLineAtCursor`, and `expandFromEditor` on the shared path.
   Their established replacement remains exactly `— `, and only the source trigger and
   source span change.
4. Update `scripts/test-ledger-tools-snippets.cjs` with focused coverage for:
   - parsing isolated `-` at column zero and after prose with exact one-character spans;
   - whole-line, prefix, suffix, suffix-whitespace, and `left-|right` expansion text and
     cursor positions;
   - editor-level replacement of exactly one source character by `— `, a `true` handled
     result, and cursor placement after the added space;
   - rejecting the former `--` trigger at both possible cursor positions and rejecting
     three-or-more-hyphen runs at their ends and between characters;
   - retaining `null`/unhandled behavior for boundary-valid `d-`, `D-`, `t-`, and `dt-`,
     while `se-` and representative completed snippets remain unchanged;
   - ordinary non-matches and the existing selection guards continuing to leave Tab to
     Obsidian.
5. Document the new one-hyphen example and exact cursor/trailing-space behavior in
   `README.md`, and explain that `--` and longer adjacent-hyphen runs no longer expand.
   Release this user-visible trigger change as Bob Ledger Tools `1.4.0`, updating both
   `plugins/bob-ledger-tools/manifest.json` and the README plugin table in accordance
   with the repository's existing feature-version convention.

## Validation and deployment

From the checkout printed by `sase repo open`:

1. Run `node --test scripts/test-ledger-tools-snippets.cjs` and confirm the parser,
   boundary, expansion, cursor, editor, and regression cases pass.
2. Run `npm test` to cover every plugin test and catch shared keymap or helper
   regressions.
3. Run `npm run validate` to syntax-check all plugin sources and validate every
   manifest, including the `1.4.0` version.
4. Review `git diff --check`, the final diff, and repository status. Confirm source
   changes are limited to the four expected files and do not alter selection guards,
   keymap priority, or unrelated plugin behavior.
5. Satisfy the linked repository's deployment requirement with a scoped preview:
   `bob plugins sync --no-pull --dry-run --plugin bob-ledger-tools --repo <opened-bob-plugins-path>`.
   Confirm that only the expected managed `main.js` and `manifest.json` vault copies
   differ.
6. Deploy with the same command without `--dry-run`. Do not use `--force` if the vault
   dirty-file guard refuses an overwrite; investigate and preserve unexpected drift.
7. Run `bob plugins list --no-pull --format json --repo <opened-bob-plugins-path>` and
   confirm `bob-ledger-tools` reports version `1.4.0`, `sync: "synced"`, and
   `vault: "enabled"`.

The current unmodified baseline already passes the focused 12-test snippet suite, the
full 622-test repository suite, and six-plugin manifest/source validation; repeat all
checks after implementation rather than relying on that baseline.

## Acceptance criteria

- Through both supported snippet entry points, an isolated `-` immediately before a
  single collapsed cursor expands to exactly `— ` and positions the cursor after the
  ASCII space, including when the hyphen is adjacent to prose.
- `--`, longer hyphen runs, and partial negative date/time tokens do not invoke the
  em-dash expansion; unhandled Tab presses remain available to Obsidian.
- Existing completed snippets, selection guards, and shared editor handling retain their
  current behavior.
- The manifest and README consistently report Bob Ledger Tools `1.4.0` and describe the
  new trigger without advertising `--` as an alias.
- Focused tests, the full repository suite, manifest/source validation, diff review,
  deployment preview, vault sync, and the final synced/enabled status check all pass.
