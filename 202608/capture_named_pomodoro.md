---
tier: epic
status: done
title: Named-Pomodoro capture targeting (`@route:id#pomodoro`)
goal: "A capture can name exactly which Pomodoro in today's daily note receives its task
  link, using a typeable slug that supports multi-word names, with excellent Bob Mac
  Capture completion that also offers unnamed Pomodoros and names them in place.

  "
phases:
  - id: pomodoro_ledger
    title: Pomodoro ledger scanner and `bob capture-pomodoros`
    depends_on: []
    size: medium
    description:
      "pomodoro_ledger: add the shared named-Pomodoro scanner (name, slug, time range,
      current, stale-safe ref, selection) and the read-only discovery command that
      exposes it."
  - id: capture_named_marker
    title: "`@route:id#pomodoro` grammar and capture execution"
    depends_on:
      - pomodoro_ledger
    size: medium
    description:
      "capture_named_marker: extend the capture grammar with the third Pomodoro-name
      component, route it through capture execution's Pomodoro selection, and report it
      in capture-parse spans, needs, and diagnostics."
  - id: pomodoro_name_command
    title: "`bob capture-pomodoro-name` write command"
    depends_on:
      - pomodoro_ledger
    size: medium
    description:
      "pomodoro_name_command: add the single-purpose command that canonicalizes a name
      and appends it to one open, unnamed Pomodoro through an atomic rename."
  - id: complete_pomodoro_name
    title: "`capture-complete` Pomodoro-name context"
    depends_on:
      - pomodoro_ledger
      - capture_named_marker
    size: medium
    description:
      "complete_pomodoro_name: return ranked pomodoro_name candidates, collapse
      duplicate slugs, and always offer never-filtered nameable rows for unnamed
      Pomodoros."
  - id: mac_pomodoro_name_ui
    title: Bob Mac Capture Pomodoro-name completion and naming prompt
    depends_on:
      - pomodoro_name_command
      - complete_pomodoro_name
    size: medium
    description:
      "mac_pomodoro_name_ui: decode the additive candidate fields, render the Pomodoro
      completion rows, and add the inline naming prompt that writes the name before
      splicing the slug."
proposed_by: bbugyi200.athena.0fk
create_time: 2026-09-09 19:59:50
---

- **PROMPT:**
  [prompts/202608/capture_named_pomodoro.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/capture_named_pomodoro.md)

# Named-Pomodoro capture targeting (`@route:id#pomodoro`)

## Goal

`bob capture '@sase:deep-fix' 'Some task.'` creates a next-status task and adds its task
link to _whichever_ Pomodoro capture picks — the current one, else the first future one.
That is the right default and it stays the default. What is missing is the ability to
say **which** Pomodoro:

```bash
bob capture '@sase:deep-fix#bugs' 'Fix the startup regression.'
```

adds `[[sase#^deep-fix]]` under the open Pomodoro named `BUGS` in today's daily note,
leaving the current Pomodoro alone. In Bob Mac Capture, typing `#` right after a
resolved `@route:block-id` marker opens a Pomodoro picker listing today's open Pomodoros
— named ones ready to insert, unnamed ones ready to be named in place.

A named Pomodoro is a ledger entry whose body ends in an em dash plus an ALL-CAPS name
(`- [ ] () — DEEP WORK`). Names contain spaces, so the central design question is how a
whitespace-free capture token addresses them.

## Design decisions

### Names are addressed by slug, exactly like task sections

The capture grammar already solves "address a multi-word ALL-CAPS title from a
whitespace-free token": `@cash+goog-exit#future-work` reaches the `FUTURE WORK` section
of a task. The Pomodoro name component reuses that mechanism unchanged.

- The selector is a **slug**: ASCII-lowercase, internal whitespace collapsed to single
  `-`. `AFTER TUI FIX` is addressed as `after-tui-fix`.
- Matching is a **whole-slug match in document order, else the first slug-prefix
  match**, identical to `capture_task_sections::select_section`. `#bugs` and `#bug` both
  reach `BUGS`; a whole-slug match beats an earlier prefix match, so `#memory` reaches
  `MEMORY` even when `MEMORY WORK` appears first.
- The selector's character set is the existing task-section selector set (`A-Z`, `a-z`,
  `0-9`, and `& ' ( ) , . / -`).

Rejected alternatives, recorded so they are not re-litigated: percent- or
backslash-escaping spaces (no other capture token escapes anything, and the escapes
would have to survive shell quoting, the editor, and the highlighter); quoting the name
(`#"AFTER TUI FIX"`) — every token in this grammar is whitespace-free by construction,
and quoting would break the tokenizer contract that `capture`, `capture-parse`, and
`capture-complete` all share; a numeric index (`#2`) — unstable the moment a Pomodoro is
inserted, and unreadable in a draft.

### Only open Pomodoros are targetable

`@route:id` writes a **next** task and links it into a Pomodoro that is still open;
completed entries are ignored today. Naming a Pomodoro does not change that: a
Pomodoro-linked task is planned work, and planning it into a finished session is
meaningless. So:

- Selection considers only open (unchecked) top-level entries.
- Completion offers only open entries.
- A selector that matches only a completed entry gets its own error naming that fact,
  not a generic "no such Pomodoro".

`bob capture-pomodoros` still _lists_ completed entries under `-a/--all`, because a
discovery command should be able to show the whole ledger.

### An explicit name overrides the multiple-open-timed invariant

Today `insert_pomodoro_child_block` refuses outright when the ledger has more than one
open timed entry, because implicit selection would be ambiguous. That guard protects
implicit selection only. A named selector is explicit and unambiguous, so it resolves
without the guard. This is strictly a relaxation: no existing input changes behavior.

Where "the current Pomodoro" is reported (`is_current` in JSON, the `Current` badge), it
is true only when there is **exactly one** open timed entry; otherwise no entry is
marked current and `capture-pomodoros` emits a warning.

### Duplicate names collapse in completion and resolve to the first

Real daily notes repeat names (two `MEMORY` Pomodoros in one day is normal). A slug
cannot distinguish them, so pretending otherwise would be a lie:

- Capture resolves a duplicated slug to the **first open match in document order** — the
  same "first duplicate wins" rule `select_section` already uses.
- Completion **collapses** open named candidates that share a slug into one row
  describing the first match, badged `2 matches`. Selecting it inserts a slug that
  resolves to exactly the row shown.
- The documented way to target the second one is to give it a distinct name (in Obsidian
  with `<ctrl+shift+m>`, or through the app's naming prompt when it is unnamed). This
  limitation is intentional and documented, not an oversight.

Nameable (unnamed) candidates are never collapsed: each is a distinct target.

### A typed `#` is a commitment

`@route:id#` with an empty name is **incomplete**, exactly like `@route+id#`. It never
degrades to "any Pomodoro" — omitting `#` already means that, and silently falling back
would make the marker unpredictable. `bob capture` reports an actionable error, and
`capture-parse` reports mode `incomplete` with `needs: ["pomodoro_name"]`.

That is also what gives the app its trigger: the moment `#` is typed, the completion
field is a zero-length `pomodoro_name` range with an empty query, so the picker opens
with the full list.

### Naming an unnamed Pomodoro mirrors "Add block ID"

The `@route+` task picker already solves this shape: Bob returns candidates that _cannot
be inserted yet_ (`requires_block_id: true`, empty `replacement`), the client prompts,
calls a small single-purpose write command, and splices the canonical value Bob returns.
The Pomodoro picker reuses that protocol verbatim with `requires_name: true` and
`bob capture-pomodoro-name`.

Names are canonicalized on write (trim, collapse whitespace, ASCII-uppercase) because
the vault's own definition of a named Pomodoro is an ALL-CAPS name, and because
uppercasing cannot change selectability (slugs are lowercased anyway). The command
returns the canonical `name` and `slug` so the client never guesses, and the app shows a
live `Saves as DEEP WORK` hint so the transformation is visible before it happens.

## Wire contracts

All three JSON surfaces stay at `schema_version` 1: every field below is additive, and
every new client field decodes as absent from an older `bob`.

**`bob capture-pomodoros`** (new, read-only):

```json
{
  "ok": true,
  "schema_version": 1,
  "day_file": "/home/bryan/bob/2026/20260828.md",
  "relative_day_file": "2026/20260828.md",
  "count": 2,
  "pomodoros": [
    {
      "ref": "31:1a2b3c4d",
      "line": 31,
      "state": "open",
      "status_symbol": " ",
      "name": "MEMORY",
      "slug": "memory",
      "selectable": true,
      "time_range": "1205-1230",
      "placeholder": false,
      "is_current": true,
      "child_count": 5
    },
    {
      "ref": "38:9f8e7d6c",
      "line": 38,
      "state": "open",
      "status_symbol": " ",
      "name": null,
      "slug": "",
      "selectable": false,
      "time_range": null,
      "placeholder": true,
      "is_current": false,
      "child_count": 0
    }
  ],
  "warnings": []
}
```

**`bob capture-pomodoro-name`** (new, single write):

```json
{
  "ok": true,
  "schema_version": 1,
  "dry_run": false,
  "day_file": "/home/bryan/bob/2026/20260828.md",
  "relative_day_file": "2026/20260828.md",
  "name": "DEEP WORK",
  "slug": "deep-work",
  "line": 38,
  "ref": "38:0b1c2d3e",
  "pomodoro": { "…": "the same entry shape as capture-pomodoros, after the write" }
}
```

**`bob capture-complete`** with `context: "pomodoro_name"`; each candidate:

```json
{
  "replacement": "memory",
  "ref": "31:1a2b3c4d",
  "name": "MEMORY",
  "requires_name": false,
  "line": 31,
  "state": "open",
  "status_symbol": " ",
  "time_range": "1205-1230",
  "placeholder": false,
  "is_current": true,
  "child_count": 5,
  "match_count": 2
}
```

A nameable candidate has `name: null`, `requires_name: true`, `match_count: 1`, and an
empty `replacement` an updated client must never insert — the same contract
`requires_block_id` already has.

Failure JSON for both new commands is `{"ok": false, "error": "..."}` and is write-free.

## Compatibility and safety invariants

- Every existing capture input keeps its exact behavior. `@route:id` with no `#` still
  uses `CurrentOrFuture` selection, including its multiple-open-timed guard.
- A `#` inside a bullet-section prefix, a trailing bare `#` Pomodoro note, and a `#` in
  the middle of body text are all unchanged. `@route#sec:x` stays a bullet marker
  because `#` precedes `:`.
- `@@` declarations are untouched: `@@route:id` and `@@route:id#name` remain
  unsupported-global-destination errors, and `capture-rewrite` still classifies a
  `PomodoroTask` local marker as non-absorbable.
- The routed note and the daily note are both parsed and validated before either is
  replaced; any failure (unknown name, completed-only match, missing daily note, missing
  `Pomodoros` section, duplicate block ID) leaves both files unchanged, and `--dry-run`
  performs the same validation with no writes.
- `bob capture-pomodoro-name` writes through one same-directory temporary file plus
  rename, appends only ` — NAME` to the resolved physical line, and preserves that
  line's terminator and every unrelated byte.
- The daily note is selected exactly as `bob capture` selects it: `BOB_DAY_FILE` when
  set and non-empty, otherwise `<bob-dir>/YYYY/YYYYMMDD.md` from `BOB_NOW` or the local
  date. No new environment variable is introduced.
- Errors and logs never echo draft text or captured task descriptions; they name routes,
  slugs, names, line numbers, and file paths only.

## Phase 1: Pomodoro ledger scanner and `bob capture-pomodoros`

Work in `bob-cli`. This phase owns every rule about what a Pomodoro entry _is_; later
phases only consume it.

1. Move the selector grammar into its single owner. In `src/native/capture_language.rs`,
   move `slug` in from `src/native/capture_task_sections.rs` as
   `pub(crate) fn selector_slug`, and rename
   `is_task_section_selector`/`is_task_section_selector_byte` to
   `is_selector_component`/`is_selector_byte`, making the first `pub(crate)`. Update
   both existing call sites and the doc comments to say these own the third-component
   selector grammar for `@route+id#section` _and_ `@route:id#pomodoro`. Make
   `capture_task_sections::is_section_title` `pub(crate)`; it is the name-validity
   predicate the next phases reuse. Behavior must not change: the existing task-section
   tests are the regression net.
2. Add `src/native/capture_pomodoros.rs` with the scanner. Port the plugin's
   `parsePomodoroEntryLineParts` faithfully (`plugins/task-status-cycler/main.js` in the
   linked `bob-plugins` repo, opened through `sase repo open`) so Bob and Obsidian can
   never disagree about what a name is:
   - Enumerate unindented, non-fenced list items inside the `## Pomodoros` section,
     reusing `pomodoro::pomodoros_section_range`, `pomodoro::open_ledger_task`,
     `pomodoro::completed_ledger_task`, and `markdown::fenced_lines`.
   - The entry body is the text after the checkbox. A **range text** is either a
     placeholder `(` + optional whitespace + `)` or a time range, and it must start at
     offset 0 of the body; reuse `pomodoro::task_time_range` for the timed case but
     require its match to begin the body. `time_range` is the normalized `HHMM-HHMM`
     string, or `null` for a placeholder or an entry with no leading range.
   - After the range text, `^[ \t]*—[ \t]*(.*)$` (U+2014) yields the name; the trimmed
     capture is the name when nonempty, otherwise the entry is unnamed. An entry with no
     leading range text is unnamed and has no time range.
   - `slug` is `capture_language::selector_slug(name)`, empty when unnamed. `selectable`
     is true only when the entry is named _and_ its slug is a valid
     `is_selector_component`; a named entry with an untypeable slug is treated as
     nameable everywhere else, because "I cannot type this" and "this has no name" need
     the same repair.
   - `child_count` is the number of direct child list items; `state` is `open` or
     `completed`; `is_current` is true only for the single open timed entry when exactly
     one exists.
3. Give entries a stale-safe ref. Extract `note_tasks::task_digest` into
   `pub(crate) fn line_digest(line: &str) -> String` (unchanged: first 8 hex characters
   of the SHA-256 of the trailing-whitespace-trimmed physical line) and have the
   existing task path call it. Add `PomodoroRef { line, digest }` to the new module with
   the same `<line>:<digest>` text form and the same resolution order as
   `NoteTaskScan::by_ref`: exact one-based line plus digest, else a unique digest match
   anywhere, else `Stale`, else `Ambiguous`.
4. Add selection to the same module: `select_named(&scan, selector) -> NamedSelection`
   over **open** entries only, returning `Found(&entry)`, `CompletedOnly(&entry)` when
   the selector's whole-slug or prefix match exists only among completed entries, or
   `Missing { suggestion }` where the suggestion is a unique
   `note_tasks::bounded_levenshtein(.., 2)` slug match. Whole-slug match beats the first
   prefix match, both in document order — reuse `select_section`'s exact precedence.
5. Register `bob capture-pomodoros` (`src/runner.rs` `SUBCOMMANDS`, kept alphabetically
   sorted between `capture-parse` and `capture-rewrite`; `src/native.rs` module plus
   `NativeCommand` arm). Options, alphabetized, each with a short alias per the
   project's CLI rules: `-a/--all` (include completed entries; default is open only,
   mirroring `capture-tasks`), `-b/--bob-dir`, `-f/--format human|json`, `-h/--help`.
   Human output is colored through `style::Styler` and plain when piped: one row per
   entry with the name (or a dim `unnamed` placeholder), the dim slug, the time range or
   `planned`, and `current` / `completed` / `N links` / `empty` badges, with a
   `No Pomodoros found.` empty state. A missing daily note or a missing `Pomodoros`
   section is a **successful empty list plus one bounded `warnings` entry** naming the
   file, so a picker degrades to "nothing to choose" instead of an error dialog.
6. Document the command in `docs/capture.md` (a new subsection under "Discovery
   commands", including the daily-note selection rule and the open-by-default behavior)
   and add its row to the `README.md` command table plus the discovery-helper sentence
   beneath it. Add `bob capture-pomodoros --help` to the `install-smoke` recipe in
   `justfile`, keeping that list sorted.
7. Tests. Unit tests in `src/native/capture_pomodoros.rs` for: timed, placeholder, and
   range-less entries; names with and without surrounding whitespace and with double
   spaces before the em dash; an em dash inside the name; completed entries; nested and
   fenced lookalikes ignored; `is_current` for zero, one, and two open timed entries;
   slug/selectable classification including an untypeable name; whole-slug beats earlier
   prefix; completed-only and suggestion outcomes; ref exact, shifted, stale, and
   ambiguous resolution. Integration tests in `tests/cli.rs` for the JSON shape,
   `--all`, plain piped human output, alphabetized `--help` options, and the
   missing-note and missing-section warning paths.

Validation:

- `cargo fmt --check`
- `cargo clippy --all-targets --all-features`
- `cargo test`
- `BOB_DAY_FILE=<a fixture daily note> bob capture-pomodoros -f json | jq` and the same
  with `-a`, confirming names, slugs, `is_current`, and refs match the file.

## Phase 2: `@route:id#pomodoro` grammar and capture execution

Work in `bob-cli`.

1. Extend the marker parser in `src/native/capture_language.rs`. Add a
   `pomodoro_name: Option<String>` component to `CaptureKind::Pomodoro`. In
   `parse_pomodoro_route_token`, split the post-colon remainder on the first `#`:
   - `is_pomodoro_marker_candidate` already accepts a `#` that follows the colon, so
     detection needs no change; confirm with a test that `@route#sec:x` is still a
     bullet marker and `@route+id#sec` is still a sub-bullet.
   - Empty block ID with a name present: "Pomodoro capture requires a block ID before
     the Pomodoro name: `@<route>:<block-id>#<pomodoro>` (run
     `bob capture-tasks -r <route>` to list task block IDs)".
   - Empty name after a typed `#`: "Pomodoro capture requires a Pomodoro name:
     `@<route>:<block-id>#<pomodoro>` (run `bob capture-pomodoros` to list today's
     Pomodoros)".
   - A name that fails `is_selector_component`: a new `POMODORO_NAME_ERROR`, "Pomodoro
     capture name must contain only A-Z, a-z, 0-9 or `& ' ( ) , . / -`".
   - Replace the two "Pomodoro capture markers must use `@<route>:<block-id>`" strings
     with one shared constant that also names the `#<pomodoro>` form. The legacy `@!`
     alias inherits the new component through the same parser; that is intended and
     needs no separate spelling.
2. Extend the editor parse in the same file. Add `SpanKind::PomodoroName`
   (`pomodoro_name`) and `Need::PomodoroName` (`pomodoro_name`, ordered after
   `PomodoroId`). Give `classify_pomodoro_token` a `MarkerThird` exactly as
   `classify_sub_bullet_token` has one, so mode, needs, spans, and the
   interactive-placeholder span for a bare trailing `#` all fall out of the existing
   `marker_parse`. `EditorParse.section` carries the Pomodoro name when the marker is a
   Pomodoro marker — the same "whichever applies" reuse `block_id` already has, and
   `mode` disambiguates. Add the `invalid_pomodoro_name` diagnostic code. Confirm
   `classify_local_marker` still reports `PomodoroTask` as non-absorbable so
   `capture-rewrite` is unchanged.
3. Extend completion field resolution: in `marker_field_at_cursor`, give the Pomodoro
   branch a
   `CompletionThird { separator_len: 1, part, context: CompletionContext::PomodoroName }`
   and add that variant to `CompletionContext`. The name component must resolve even
   when the block ID is still empty (`@route:#bu`), because the Pomodoro list does not
   depend on the block ID; only the route must already be valid.
4. Route the selector through execution in `src/native/capture.rs`. Add
   `PomodoroSelection::Named(&str)`. In `insert_pomodoro_child_block`, apply the
   `timed.len() > 1` invariant only to the two implicit variants, and for `Named`
   resolve through `capture_pomodoros::select_named` over the open entries. Thread the
   selector from `CaptureKind::Pomodoro` through `plan_capture_with_pomodoro_link` and
   `insert_pomodoro_block_link`. Errors, all `CaptureError::io` and all write-free:
   - No open entries at all: keep today's "Bob daily note has no eligible open
     Pomodoro".
   - Open entries exist but none is named: "Bob daily note has no named open Pomodoro;
     name one with `bob capture-pomodoro-name`".
   - Unique close match: "Bob daily note has no open Pomodoro named `<selector>`; did
     you mean `<slug>`? (run `bob capture-pomodoros` to list them)".
   - Otherwise: "Bob daily note has no open Pomodoro named `<selector>` (open Pomodoros:
     `<slug>`, `<slug>`, …; run `bob capture-pomodoros` to list them)", listing at most
     eight slugs before an ellipsis.
   - Completed-only match: "Pomodoro `<name>` is already completed; a Pomodoro-linked
     task needs an open Pomodoro".
5. `src/native/capture_parse.rs` renders span kinds, needs, and severities through their
   generic `label()` methods, so it should need no change; confirm that and add the new
   forms to its tests rather than to its rendering code.
6. Documentation. In `docs/capture.md`: add `@route:id#pomodoro` to the grammar table
   and to the "`#` is not one marker" table; expand "Pomodoro-linked tasks" with the
   slug rule, matching precedence, duplicate-name behavior, open-only rule, the
   explicit-name relaxation of the timed invariant, and every error case; add
   `pomodoro_name` to the `capture-parse` span-kind, `needs`, and diagnostic-code lists
   and note that `section` carries the name for a Pomodoro marker. Mirror the
   grammar-table row in `README.md`. Add a
   `bob capture '@dev:foobar#bugs' 'Some foobar task.'` example to `bob capture --help`.
7. Tests. Unit tests in `capture_language.rs` for leading and trailing marker positions,
   `s:<N>`/`p:<N>`/`%` composition in either order, the `@!` alias, every new error
   message, mode/needs/spans for `@route:`, `@route:id`, `@route:id#`, `@route:#name`,
   `@route:id#name`, and cursor-position completion fields across all four components.
   Unit tests in `capture.rs` for named selection with placeholder and timed entries,
   first-duplicate-wins, completed-only rejection, unknown-name suggestion, two open
   timed entries succeeding under an explicit name while still failing without one, and
   both notes staying unchanged on every failure. Integration tests in `tests/cli.rs`
   for a real named capture (both notes' resulting bytes), `--dry-run`, and JSON output.

Validation:

- `cargo fmt --check`
- `cargo clippy --all-targets --all-features`
- `cargo test`
- Against a fixture daily note with a current Pomodoro and a later `— BUGS` Pomodoro,
  confirm `bob capture --dry-run -f json -- '@dev:some-id#bugs' 'x'` targets `BUGS`
  while `'@dev:some-id'` targets the current one, and that
  `bob capture-parse -f json -- '@dev:some-id#'` reports mode `incomplete` with
  `needs: ["pomodoro_name"]`.

## Phase 3: `bob capture-pomodoro-name` write command

Work in `bob-cli`. Model this command on `src/native/capture_task_id.rs`; it is the same
shape of single-purpose, picker-facing write.

1. Add `src/native/capture_pomodoro_name.rs` and register it (`src/runner.rs`
   `SUBCOMMANDS`, alphabetically before `capture-pomodoros`; `src/native.rs` module and
   `NativeCommand` arm). Options, alphabetized with short aliases: `-b/--bob-dir`,
   `-d/--dry-run`, `-f/--format human|json`, `-h/--help`, `-n/--name NAME` (required),
   `-p/--pomodoro-ref REF` (required).
2. Canonicalize and validate the name: trim, collapse internal whitespace runs to one
   space, ASCII-uppercase, then require `capture_task_sections::is_section_title`.
   Reject with "Pomodoro name must contain only A-Z, 0-9 or `& ' ( ) , . / -` and must
   start with a letter or digit". Uppercasing is deliberate and documented: the vault's
   named-Pomodoro convention is ALL-CAPS, and case cannot affect the slug.
3. Resolve `--pomodoro-ref` through `capture_pomodoros::PomodoroRef`, then refuse — all
   write-free, with the same message text in both formats — when the ref is stale or
   ambiguous, the entry is completed ("Pomodoro on line N is completed; only an open
   Pomodoro can be named"), or the entry already has a selectable name ("Pomodoro on
   line N is already named NAME; target it with `#<slug>`"). A named-but-untypeable
   entry is _not_ refused: naming it is the repair, and it replaces the existing em-dash
   tail rather than appending a second one.
4. Write by appending ` — NAME` to the resolved physical line after trimming that line's
   trailing spaces, preserving its terminator and every other byte in the file, through
   one same-directory temporary file plus rename so the change is observable only after
   the rename completes. Re-scan the written contents and confirm the entry now parses
   with the expected name and slug before reporting success; a mismatch is an error, not
   a silent success. `--dry-run` returns the identical success shape without writing.
5. Success output: the JSON object in "Wire contracts" above, and colored human output
   naming the canonical name, the slug to type, and the relative daily-note path.
6. Document the command in `docs/capture.md` (its own section after
   `bob capture-task-id`, describing canonicalization, every refusal, and the atomic
   write) and add its `README.md` command-table row plus a mention in the
   discovery-helper paragraph. Add `bob capture-pomodoro-name --help` to the
   `install-smoke` recipe in `justfile`, keeping that list sorted.
7. Tests. Unit tests for canonicalization (trimming, collapsing, uppercasing,
   rejection), LF and CRLF notes, byte-exact preservation of unrelated lines, the
   untypeable-name repair path, and every refusal. Integration tests in `tests/cli.rs`
   for JSON success, human success, `--dry-run` leaving the file untouched, and
   stale/ambiguous/completed/already-named failures exiting non-zero with
   `{"ok": false, …}` and no write.

Validation:

- `cargo fmt --check`
- `cargo clippy --all-targets --all-features`
- `cargo test`
- On a scratch copy of a real daily note, name a placeholder Pomodoro and confirm with
  `git diff` (or `diff`) that exactly one line changed, then confirm
  `bob capture-pomodoros -f json` reports the new name and slug.

## Phase 4: `capture-complete` Pomodoro-name context

Work in `bob-cli`, in `src/native/capture_complete.rs`.

1. Add a `PomodoroName(Vec<PomodoroNameCandidate>)` variant to `Candidates` and handle
   `CompletionContext::PomodoroName` in `build_result`, scanning the daily note through
   `capture_pomodoros`. The candidate struct carries the fields listed in "Wire
   contracts"; a missing daily note or missing `Pomodoros` section returns a successful
   empty list plus one bounded warning, never an error.
2. Build candidates from **open** entries only:
   - Named, selectable entries collapse by slug: keep the first in document order and
     set `match_count` to the number of open entries sharing that slug.
   - Every other open entry (unnamed, or named with an untypeable slug) becomes a
     nameable candidate with `name` set to its current name or `null`,
     `requires_name: true`, `match_count: 1`, and an empty `replacement`.
3. Rank: named candidates first — slug-prefix matches, then slug-substring matches,
   document order within each tier, reusing the existing `rank` helper on the slug.
   Nameable candidates always follow every named candidate, in document order, and are
   **never filtered by the query**: browsing for something to name is exactly the case
   where a nonempty query must not empty the list. An empty query keeps every candidate.
4. Add the `pomodoro_name` context label and human-output rows in `context_label` and
   `candidate_lines`, showing name-or-`unnamed`, slug, time range, and the `current` /
   `N matches` / `name it` badges, plain when piped.
5. Document the new context in `docs/capture.md`'s `capture-complete` section: when it
   is active (`@route:id#prefix` and a bare `@route:id#`, and also `@route:#prefix`
   since the name does not depend on the block ID), the candidate fields, the open-only
   rule, slug collapsing, the never-filtered nameable rows, and the `requires_name`
   contract that an updated client must honor.
6. Tests in `src/native/capture_complete.rs` for: an empty query listing everything in
   the documented order; a prefix query keeping nameable rows; duplicate-slug collapsing
   with `match_count`; completed entries excluded; the untypeable-name entry arriving as
   nameable with its current name; `@route:#bu` completing without a block ID; the
   missing-daily-note warning path; and JSON shape stability. Add a `tests/cli.rs` case
   for the end-to-end `--cursor` invocation the app makes.

Validation:

- `cargo fmt --check`
- `cargo clippy --all-targets --all-features`
- `cargo test`
- `bob capture-complete --cursor <byte after #> -f json -- 'x @dev:some-id#'` against a
  fixture daily note, confirming context, replacement range, ordering, and badges.

## Phase 5: Bob Mac Capture Pomodoro-name completion and naming prompt

Work in the linked `bob-mac-capture` repository, opened through `sase repo open`. Model
every piece on the existing **Add block ID** flow, which solves the identical problem
for `@route+` task candidates.

1. `Sources/CaptureCore/CaptureModels.swift`: add the additive optional candidate fields
   `requires_name`, `line`, `state`, `time_range`, `placeholder`, `is_current`, and
   `match_count` (decoding as absent/`false` from an older `bob`). The existing `ref`,
   `name`, and `child_count` fields already cover the rest. Add a
   `CapturePomodoroNameResponse` success/failure model for the new command.
2. `Sources/CaptureCore/CompletionRowContent.swift`: add `.pomodoroName` to
   `CaptureCompletionContext` (`"pomodoro_name"`), map the `pomodoro_name` span kind to
   `.section` in `captureSemanticCategory(forSpanKind:)` — the third component of a
   marker shares the section color, exactly as `sub_bullet_section` does, so the editor
   and the completion list keep agreeing — and add the row case: context label
   `Pomodoro`; symbol `timer` for selectable rows and `square.and.pencil` for nameable
   ones; primary text the name, or the time range (falling back to `Unnamed Pomodoro`)
   when there is none; secondary text the time range or `Planned` for a placeholder;
   badges `Current`, `N matches` when `matchCount > 1`, `N links`/`Empty` from
   `childCount`, and `Name it` for nameable rows. Nameable rows use the `.priority`
   category, matching how missing-ID task rows already signal "one more step".
3. `Sources/CaptureCore/BobProcessClient.swift`: add
   `assignPomodoroName(ref:name:dryRun:)` running
   `bob capture-pomodoro-name --pomodoro-ref REF --name NAME --format json` on its own
   lane, mirroring `assignCaptureTaskID`.
4. `Sources/BobMacCapture/CapturePanelModel.swift`: add `CapturePomodoroNamePromptState`
   and the `.pomodoroNamePromptName` focus target, and the
   `updatePomodoroNamePromptName` / `cancelPomodoroNamePrompt` /
   `submitPomodoroNamePrompt` trio. Branch `acceptSelectedCompletion` on
   `context == "pomodoro_name" && candidate.requiresName` to present the prompt instead
   of splicing, and include the new prompt in `completionVisible` and in every
   `taskIDPrompt`-guarded disable/dismiss/draft-changed path so the two prompts can
   never both be open. **Pre-fill the name field from the in-progress completion query**
   with `-` replaced by spaces and the text uppercased, so typing `#deep-work` and
   picking an unnamed Pomodoro opens the prompt already reading `DEEP WORK`. On success,
   splice the canonical `slug` Bob returned into the saved replacement range, restore
   the caret after it, rerun analysis, and announce
   `Named <NAME> in <relative day file>`. Cancel and every failure leave the draft
   untouched.
5. `Sources/BobMacCapture/`: add a `PomodoroNameField` alongside `BlockIDField` (same
   AppKit first-responder handling and accessibility identifier pattern) and a
   `PomodoroNamePromptCard` alongside `TaskIDPromptCard`, showing the target Pomodoro's
   time range and child count, an em-dash prefix glyph instead of `^`, the live
   `Saves as DEEP WORK` hint, the caption
   `Letters, numbers, spaces, and & ' ( ) , . / -`, and **Cancel** / **Name & Select**
   actions. Register it in `CapturePanelView`'s auxiliary section and `AuxiliarySection`
   enum, and route its keys in `CaptureKeyCommandRouter` the same way the block-ID
   prompt is routed.
6. `README.md`: extend the Runtime Contract completion bullets with the
   `@route:block-id#` trigger, the `pomodoro_name` context, the `requires_name` protocol
   and the `bob capture-pomodoro-name` call, and add a **While Name Pomodoro is open**
   column (or an equivalent row set) to the Keyboard table.
7. Tests. `Tests/CaptureCoreTests`: candidate decoding including absent additive fields
   from an older `bob`, `CapturePomodoroNameResponse` success and failure, row content
   for selectable / nameable / current / duplicate-slug candidates, and the process
   client's argument vector. `Tests/BobMacCaptureTests`: accepting a selectable
   candidate splices the slug; accepting a nameable candidate opens the prompt instead;
   the prompt pre-fills from the query; a draft change while the prompt is open cancels
   it; success splices the returned slug and restores focus to the editor; every failure
   keeps the draft unchanged; the two prompts are mutually exclusive.

Validation:

- `just format-lint`
- `swift build --target CaptureCore` and `swift test --filter CaptureCoreTests` — on the
  Linux host only `CaptureCore` and `CaptureCoreTests` build; the `BobMacCapture` app
  target needs AppKit and therefore macOS.
- `just build` and `just test` when a Mac is available. Otherwise push and verify the
  repository's `macOS 26 SwiftPM` GitHub Actions workflow with `gh run list` /
  `gh run view --job=<id> --log`, and report the app-target result from that run rather
  than as unverified.
- With a real vault: type `Fix startup @sase:some-id#` in the panel, confirm the picker
  lists today's open Pomodoros with named rows first and unnamed rows badged `Name it`,
  name one, and confirm the draft, the daily note, and the resulting capture all agree.
