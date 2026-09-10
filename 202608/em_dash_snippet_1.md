---
tier: tale
title: Add an em-dash snippet to Bob Ledger Tools
goal:
  Obsidian expands an exact two-hyphen trigger into one em dash through the existing Bob
  snippet Tab path without disturbing Markdown hyphen runs or existing snippets.
size: small
proposed_by: bbugyi200.athena.0du
create_time: 2026-09-09 20:00:02
status: wip
---

# Add a Tab-expanded em-dash snippet to Bob Ledger Tools

## Goal

Extend the existing `bob-ledger-tools` Obsidian snippet system so a single-cursor Tab
press immediately after exactly two ASCII hyphens (`--`) replaces those two characters
with one Unicode em dash (`—`, U+2014). Reuse the existing highest-priority Tab keymap
and **Expand Bob snippet** command rather than registering another keybinding or
creating a second expansion path.

## Behavioral contract

- Treat the exact two-character run immediately before the cursor as the trigger, even
  when it is adjacent to prose: `left--|right` becomes `left—|right`, where `|` marks
  the cursor. Preserve all text outside the trigger and leave the cursor immediately
  after the one-character replacement.
- Do not recognize two hyphens that touch another ASCII hyphen on either side. This
  keeps Markdown thematic breaks, YAML frontmatter delimiters, and other `---` or longer
  runs available to Obsidian's normal Tab behavior, including when the cursor is inside
  rather than only at the end of the run.
- On a successful match, return the existing editor keymap's handled result so Tab is
  consumed. On a non-match, continue returning false so Obsidian can indent or perform
  its normal Tab action.
- Preserve the current single-cursor/collapsed-selection guard and every existing `ta`,
  `se...`, `t...`, `d...`, `D...`, and `dt...` snippet contract.

## Repository and file scope

The source change belongs only in the linked `bob-plugins` repository. Open it with
`sase repo open bob-plugins` and use the returned checkout path; do not edit the vault's
installed plugin files directly. No `bob-cli` source change is needed: its existing
`bob plugins` command is used only to deploy and verify the finished plugin.

Expected files in `bob-plugins`:

- `plugins/bob-ledger-tools/main.js`
- `scripts/test-ledger-tools-snippets.cjs`
- `plugins/bob-ledger-tools/manifest.json`
- `README.md`

## Implementation

1. In `plugins/bob-ledger-tools/main.js`, add a punctuation-specific trigger parser for
   an exact `--` immediately before the cursor. Keep it separate from the generic
   word-like trigger regular expression so accepting prose-adjacent em dashes does not
   relax identifier boundaries for existing snippets. Return the same trigger span
   metadata as the other parser branches, with a distinct kind such as `emDash`.
2. Teach the expansion computation to map that trigger kind to exactly `—` (U+2014) and
   a terminal cursor position. Adjust the post-cursor boundary check only for this
   punctuation trigger: allow a word or punctuation suffix, but reject an adjacent ASCII
   hyphen. Combined with the parser's preceding-hyphen rejection, no pair inside a
   three-or-more-hyphen run may expand. Leave the shared `findExpansion`,
   `expandLineAtCursor`, editor replacement, command, and Tab-keymap flow in control of
   the actual edit.
3. Extend `scripts/test-ledger-tools-snippets.cjs` with focused coverage for:
   - parsing `--` at column zero and after prose, with exact start/end spans;
   - computing exactly one U+2014 replacement and positioning the cursor after it;
   - whole-line and inline edits, including `left--|right`, with prefix/suffix text
     preserved;
   - refusing `---` and longer runs when the cursor is at the end or between hyphens;
   - an editor-level successful replacement that returns true, replaces only the two
     trigger characters, and sets the cursor correctly, demonstrating that the existing
     Tab handler consumes the key;
   - regression assertions that representative existing snippet triggers still parse and
     expand unchanged and ordinary non-matches remain unhandled.
4. Treat the new snippet as a backward-compatible feature release: bump
   `plugins/bob-ledger-tools/manifest.json` from `1.2.0` to `1.3.0`, update the Bob
   Ledger Tools version in `README.md`, and document `--` -> `—` alongside the current
   Tab/command snippet examples and exact cursor behavior.

## Validation and deployment

From the opened `bob-plugins` checkout:

1. Run the focused suite with `node --test scripts/test-ledger-tools-snippets.cjs`.
2. Run the complete repository suite with `npm test`.
3. Run `npm run validate` to syntax-check every `main.js` and validate all manifests,
   including the new `1.3.0` semver.

After all source checks pass, satisfy the linked repository's deployment requirement
using the exact checkout returned by `sase repo open`:

1. Preview only this plugin with
   `bob plugins sync --no-pull --dry-run --plugin bob-ledger-tools --repo <opened-bob-plugins-path>`
   and confirm only the expected managed `main.js` and `manifest.json` copies differ.
2. Deploy it with the same command without `--dry-run`; do not use `--force` if the
   dirty-vault guard refuses an overwrite. Resolve unexpected vault drift instead of
   discarding it.
3. Run `bob plugins list --no-pull --format json --repo <opened-bob-plugins-path>` and
   confirm `bob-ledger-tools` reports version `1.3.0`, `sync: "synced"`, and
   `vault: "enabled"`.

## Acceptance criteria

- In Bob's Obsidian Markdown editor, a single-cursor Tab press after an exact `--`
  follows the shared snippet path and produces one `—` with the cursor immediately after
  it, including in prose-adjacent use.
- No two-hyphen subset of `---` or a longer hyphen run expands, and a failed match
  leaves Tab available to Obsidian.
- Existing snippets and selection guards retain their current behavior.
- Focused tests, the full plugin test suite, manifest/source validation, deployment, and
  the final source-to-vault sync-state check all pass.
