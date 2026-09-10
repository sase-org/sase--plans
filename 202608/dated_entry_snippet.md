---
tier: tale
title: Add the uppercase dated-entry snippet
goal:
  D[-]<N> expands to an emphasized local offset date, an em dash, and a trailing cursor
  stop while every existing Bob Ledger Tools snippet keeps its current behavior.
size: small
proposed_by: bbugyi200.athena.03h
create_time: 2026-09-09 19:59:59
status: wip
---

# Plan: Add the `D[-]<N>` dated-entry snippet

## Goal and exact user-facing contract

Extend the source-of-truth `bob-ledger-tools` Obsidian plugin with an uppercase
`D[-]<N>` snippet alongside its existing lowercase `d[-]<N>` date snippet. Expanding the
new trigger with Tab or the existing **Expand Bob snippet** command must replace it with
this exact Markdown shape:

```text
_YYYY-mm-dd_ — <cursor>
```

For example, with a local current date of 2026-08-16:

- `D0` becomes `_2026-08-16_ — `;
- `D1` becomes `_2026-08-17_ — `; and
- `D-1` becomes `_2026-08-15_ — `.

The underscores are literal Markdown emphasis delimiters, the separator is one Unicode
em dash (`U+2014`) with one ordinary space on each side, and the replacement ends with
one trailing space. The user's cursor lands immediately after that trailing space. The
`$1` from the requested snippet notation describes that cursor stop and must never be
inserted literally.

`N` retains the existing date-snippet grammar and semantics: at least one decimal digit
is required; the optional `-` makes the offset negative; `0` means today; and date
calculation uses local calendar days. Preserve all current trigger-boundary behavior,
including rejecting a trigger embedded in an identifier or followed immediately by a
word character.

This is a `small` tale because the new behavior is an isolated extension of the shared
snippet parser/formatter, with bounded tests, release metadata, documentation, and
deployment in one plugin.

## Existing implementation to preserve

Work in the linked `bob-plugins` repository opened through `sase repo`; do not edit the
deployed vault copy directly. The relevant source is `plugins/bob-ledger-tools/main.js`:

- `TRIGGER_RE`, `DATE_TIME_TRIGGER_RE`, and `parseTrigger()` currently distinguish
  lowercase `d`, `t`, and `dt`, along with `se...` and `ta` snippets.
- `computeSnippetExpansion()` formats a lowercase date trigger with
  `formatOffsetDate(now, offset)`, which already performs local-calendar arithmetic via
  `addLocalDays()` and `formatLocalDate()`.
- `findExpansion()` owns the identifier-boundary guards and preserves surrounding line
  content.
- `expansionCursorCh()` uses an explicit `cursorOffset` when present and otherwise
  places the cursor at the end of the replacement. `expandLineAtCursor()` and the live
  editor path both converge on this calculation; the existing `ta` snippet proves the
  explicit cursor-stop mechanism.

The uppercase trigger must be added narrowly. Do not make the date/time regular
expressions globally case-insensitive, because that would silently introduce unrequested
uppercase `T` and `DT` snippets. Do not alter the meaning or output of lowercase `d`,
`t`, `dt`, `se`, or `ta`.

## Implementation design

1. Extend the trigger grammar to accept only uppercase `D` in addition to the existing
   prefixes, and have `parseTrigger()` return a distinct, clearly named kind (for
   example, `datedEntry`) for it. Keep the parsed integer offset, trigger span, and
   word-boundary rules identical to lowercase `d`.
2. Add a `computeSnippetExpansion()` branch for that distinct kind. Reuse
   `formatOffsetDate(now, trigger.offset)` so month/year rollover, zero padding, local
   timezone behavior, and negative offsets cannot drift from `d[-]<N>`.
3. Build the exact replacement as `_<formatted date>_ — ` using a literal UTF-8 em dash.
   Return an explicit cursor offset equal to the complete replacement length so the `$1`
   contract is visible in the expansion metadata and remains correct through both helper
   and live-editor expansion paths. Do not add selection/tab-stop machinery or emit a
   literal `$1`; this snippet has only one terminal cursor position.
4. Retain `findExpansion()`'s existing surrounding-text behavior. A valid trigger at
   line start or after punctuation/whitespace should expand in place; unchanged suffix
   text must remain after the replacement, with the cursor positioned between the new
   trailing space and that suffix.

Expected implementation files in `bob-plugins`:

- `plugins/bob-ledger-tools/main.js` for parsing, formatted replacement, and cursor
  metadata;
- a focused ledger-snippet test file under `scripts/` plus `package.json` test-script
  registration (prefer a dedicated `scripts/test-ledger-tools-snippets.cjs` rather than
  mixing snippet behavior into the navigation- or Vim-mapping-specific suites);
- `plugins/bob-ledger-tools/manifest.json` for the feature release; and
- `README.md` for the synchronized version and concise trigger/output documentation.

## Regression coverage

Use the plugin's exported pure helpers with a fixed local `Date` so tests are
deterministic in every timezone. Cover:

1. Parsing `D0`, positive `D1`/multi-digit offsets, and negative `D-1`; assert the
   distinct kind, exact offset, and source span.
2. Exact expansion text for today, tomorrow, and yesterday, including literal
   underscores, single spaces, `U+2014`, and the final trailing space. Include a
   month/year boundary to prove the uppercase form reuses local-day date arithmetic.
3. Cursor placement from `expandLineAtCursor()`: at end of the new replacement for a
   terminal trigger, and immediately before an unchanged non-word suffix when expanding
   in surrounding text. Assert that the output contains no `$1` text.
4. Boundary and malformed inputs: reject bare `D`, bare `D-`, embedded `xD0`, and `D0x`,
   while allowing the same line-start/whitespace/punctuation contexts as `d0`.
5. Compatibility assertions showing lowercase `d0` still expands to bare `YYYY-mm-dd`,
   while representative `t`, `dt`, `se`, and `ta` parsing/expansion remains unchanged.
   In particular, uppercase `T` and `DT` must remain unsupported.
6. A thin editor-path assertion, if needed beyond the pure expansion test, that the
   existing `expandFromEditor()` replacement and `setCursor()` calls honor the returned
   explicit cursor offset without disturbing the line's untouched prefix or suffix.

Keep the test stubs consistent with the existing CommonJS/Obsidian harnesses and avoid
wall-clock expectations that can flake at midnight.

## Documentation, release, and deployment

- Treat the new user-visible snippet as a minor feature: bump `bob-ledger-tools` from
  `1.1.3` to `1.2.0` in `plugins/bob-ledger-tools/manifest.json` and keep the README
  plugin table synchronized.
- Add concise README documentation for both related forms so their contrast is clear:
  `d[-]<N>` emits the bare local ISO date, while `D[-]<N>` emits the emphasized date, em
  dash, trailing space, and cursor stop. Use examples with `0`, positive, and negative
  offsets; do not describe `$1` as literal output.
- Run the dedicated snippet test while iterating, then the complete repository checks:

  ```bash
  node --test scripts/test-ledger-tools-snippets.cjs
  npm test
  npm run validate
  ```

- After source and tests pass, follow the linked repository's mandatory deployment
  workflow from that opened checkout. Preview without pulling over the tested work,
  deploy only `bob-ledger-tools`, and verify a final preview is clean:

  ```bash
  bob plugins sync --no-pull --dry-run --plugin bob-ledger-tools --repo .
  bob plugins sync --no-pull --plugin bob-ledger-tools --repo .
  bob plugins sync --no-pull --dry-run --plugin bob-ledger-tools --repo .
  bob plugins list --no-pull --repo . --format json
  ```

  Do not use `--force` to overwrite protected vault edits; report that condition as a
  deployment blocker. Confirm the deployed managed files match the tested source and
  that version `1.2.0` is enabled/synced. Note that an already-running Obsidian instance
  may need the plugin reloaded before the JavaScript change is active.

## Boundaries and acceptance criteria

- Do not change date formats, UTC/local-time policy, trigger activation keys, command
  IDs, multi-cursor rejection, ledger-range handling, Pomodoro behavior, or any other
  plugin.
- Do not interpret `$1` as content or add a general snippet-placeholder engine; an
  explicit cursor offset is sufficient and matches the current architecture.
- The work is complete when every valid `D[-]<N>` expands to the exact emphasized local
  date plus em dash and trailing space, the cursor lands at the requested stop, invalid
  and identifier-embedded forms remain inert, all old snippets are unchanged, focused
  and full tests plus manifest validation pass, version/docs agree, and the plugin is
  synced to the vault with a clean post-deployment dry run.
