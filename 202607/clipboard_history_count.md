---
tier: tale
goal:
  Positive numeric clipboard markers capture the current clipboard plus the requested
  number of recent history entries as headerless child bullets, with deterministic
  sourcing, collision-safe file planning, atomic writes, and stable human and JSON
  output.
create_time: 2026-09-09 19:53:07
status: wip
---

# Plan: Headerless clipboard-history capture with `%<N>`

## Product context and scope

`bob capture` currently gives a terminal `%...` token two meanings: bare `%` captures
the live clipboard without a header, while `%<header>` captures it under an explicit
formatted header. The token participates in the existing terminal-marker grammar
alongside schedules and routes, and every clipboard payload is independently classified
as inline text, flat lines, attachments, or a snippet before the resulting child lines
are inserted atomically with the new parent task or bullet.

Add a third, numeric form: `%<N>`, where `N` is a positive integer, captures the `N`
most recent clipboard entries. All of those entries are deliberately headerless.
Preserve the current live-clipboard behavior, explicit nonnumeric headers,
route/schedule composition, classification rules, attachment/snippet handling, note
placement, dry-run semantics, and rollback guarantees.

This remains a `tale`: the parser, clipboard-history adapter, multi-entry planner,
output model, documentation, and regressions form one cohesive change and should land
together.

## User-visible grammar and ordering

- Parse a non-empty, all-ASCII-digit `%...` token before the general header branch. A
  representable value greater than zero is a history count, so `%1`, `%2`, and `%20`
  request one, two, and twenty entries respectively. Leading zeroes are accepted when
  the numeric value is positive (`%01` is `%1`).
- `%1` is an exact alias for bare `%`: it reads the live clipboard through the existing
  source resolution, renders it without a header, and does not require a
  clipboard-history manager.
- For `%N` where `N > 1`, make the live clipboard the first entry, then append the next
  `N - 1` recent entries from history in newest-to-oldest order. This makes the sequence
  prefix-stable: increasing `%2` to `%3` only adds an older entry at the end, and the
  first result always agrees with `%` even if the history manager has not observed the
  latest copy yet.
- Require exactly `N` usable entries. If history cannot provide enough older entries,
  fail with the requested and available counts before planning or writing any vault
  content; never silently produce a partial capture.
- Numeric zero and numeric overflow are not positive-count markers and remain literal
  terminal text under the existing forgiving-marker rule. Positive numeric marker
  headers are intentionally reassigned: a numeric label can still be requested
  unambiguously with `--clip=<digits>` (or a nonnumeric marker header).
- Keep nonnumeric `%<header>` validation and rendering unchanged. Bare `--clip` and
  `--clip=HEADER` continue to force a single live capture and keep `%...` tokens
  literal; no new CLI option is needed. `--no-clip` disables bare, headed, and numeric
  marker parsing alike.
- Preserve the current terminal-region and duplicate-marker rules. Numeric history
  markers must compose in either supported order with `s:<N>`, ordinary routes, bullet
  routes, forced routes/sections, and Pomodoro routes. Mid-body numeric tokens, invalid
  `%...` tokens, and the unextracted member of a duplicate clip-marker sequence remain
  text.

Represent these states explicitly in `src/native/capture.rs`, preferably as a
clip-request enum with mutually exclusive `Current { header: Option<String> }` and
`History { count: NonZeroUsize }` variants. Do not reintroduce ambiguity between “no
clipboard request,” “headerless current clipboard,” and “multiple headerless history
entries.”

## Clipboard-history source contract

Continue to use the existing `BOB_CLIPBOARD_CMD` and platform fallback chain for the
live first entry. Add a separate history-candidate reader because the existing command
contract emits one opaque clipboard value and cannot safely frame multiple values
containing newlines.

1. If `BOB_CLIPBOARD_HISTORY_CMD` is set and non-empty, whitespace-split it consistently
   with `BOB_CLIPBOARD_CMD`, append the requested total count as the final argument, and
   run it. Its stdout is a UTF-8 JSON array of strings ordered newest first. Each
   element is one complete clipboard value and may contain newlines. This is the
   portable power-user override and deterministic integration-test hook.
2. Otherwise, on macOS, read Clipy's current SQLite history database in read-only mode.
   The configured environment uses Clipy, but Clipy exposes no supported command-line or
   URL API; a read-only, schema-checked adapter avoids brittle UI automation, never
   changes the current pasteboard, and never mutates Clipy's history. Locate the
   production database under Clipy's bundle-specific Application Support directory,
   query recent records by update time with a deterministic tie-break, and reconstruct
   the text/path representation from stored pasteboard assets in their original order.
   Use full stored assets rather than Clipy's display title, which is truncated. Support
   modern/deprecated UTF-8 string and file/URL representations needed by the existing
   clipboard classifier; reject an entry that only has unsupported binary
   representations with an indexed, actionable error. Verify required tables/columns and
   report an unsupported or unmigrated Clipy database clearly rather than guessing at a
   changed private schema.
3. On other platforms without the override, fail clearly for counts above one, naming
   `BOB_CLIPBOARD_HISTORY_CMD` and its JSON protocol. Do not guess among incompatible
   clipboard-manager databases or add UI automation. `%` and `%1` remain portable
   through the existing single-clipboard providers.

Normalize every candidate with the same CRLF/CR, UTF-8, embedded-NUL,
trailing-empty-line, and empty-content rules as a live clipboard read. Merge candidates
with the live value by removing at most the first normalized candidate equal to the live
clipboard, then take the next `N - 1`; this handles both an up-to-date Clipy history
(which already contains the live value) and a lagging history (which does not) without
duplicating the first entry. Preserve later duplicate entries supplied by a provider
because they are distinct history records. Provider failure, malformed JSON, invalid
entry data, Clipy schema/type errors, or insufficient history aborts the whole capture.

Keep the provider boundary narrow and testable. If a SQLite dependency is introduced,
open databases read-only, keep the history decoding independent from the capture
renderer, and include the lockfile change. Do not add a daemon, record clipboard changes
inside Bob, depend on Clipy's UI, or modify the linked dotfiles repository as part of
this change.

## Multi-entry planning and Markdown layout

Run every selected value through the existing classifier independently with
`header: None`. This preserves the meaning of “do the same thing” for all supported
content shapes:

- inline text, a single attachment, and a snippet link each contribute one direct
  two-space child bullet;
- flat multi-line text and multiple attachments contribute direct sibling child bullets
  using the already-approved headerless layout;
- no entry receives `**CLIP:**`, an index label, an empty container bullet, or an extra
  separator.

Flatten entry child lines in source order beneath the unchanged parent line. Entry
boundaries need not add visible Markdown because the requested presentation is
headerless, but retain them in the structured result so callers can distinguish a
multi-line entry from several one-line entries.

Refactor `src/native/capture_clip.rs` to plan a sequence as one aggregate operation with
a shared destination reservation table. Planning entries independently against only the
filesystem is unsafe: two attachments with the same sanitized name or two snippets with
the same timestamp/slug can select the same not-yet-created path. The aggregate planner
must therefore:

- reserve every planned attachment and snippet path across all entries;
- reuse one destination for identical attachment content and apply the existing
  hash-suffix policy to differing content;
- allocate `-2`, `-3`, and later snippet suffixes against both existing and
  already-reserved paths;
- retain exact per-entry outputs while exposing flattened lines/files for note
  insertion, human confirmation, saving, and cleanup;
- save each unique file once and return every newly created path so a later note-write
  failure can remove all and only artifacts created by this capture.

Plan and validate all entries and the note edit(s) before the first write. `--dry-run`
performs the same history read, normalization, classification, collision allocation, and
Pomodoro validation but writes no note, attachment, snippet, or directory. A failure in
any entry, file save, routed-note validation, or Pomodoro two-file edit must leave the
vault unchanged, with existing best-effort cleanup extended across the aggregate plan.

## Human and JSON output

Human output should print the parent and the flattened headerless child lines exactly as
they will appear, followed by the existing saved/reused/would-save confirmations for
unique files. Do not add entry-number labels to the note or confirmation block.

Preserve the existing `clip` object for all clipboard captures. Single current captures
retain their current serialized shape byte-for-byte. For a numeric history capture, use
an aggregate shape that remains useful to existing consumers:

- `header` is `null`;
- `mode` is a new `"history"` value;
- `lines` is the exact flattened Markdown child-line array written to the note;
- `attachments` aggregates attachment records in entry order;
- a new `entries` array contains one ordinary clip output per requested history entry,
  each with `header: null`, its existing classification mode, exact lines, attachments,
  and optional snippet path;
- the aggregate omits the singular top-level `snippet` field; snippet paths remain on
  their owning entries.

Omit `entries` for existing single captures so their JSON is unchanged. The history
`entries` length is the successfully fulfilled count, and `task_line` remains the
unchanged parent line. JSON errors keep the existing single
`{ "ok": false, "error": ... }` contract.

## Help and README documentation

Update `bob capture --help` and the README to make the three marker forms scannable: `%`
for one headerless live value, `%<positive integer>` for that value plus recent history,
and `%<nonnumeric header>` for one explicitly headed live value. Include an example such
as `bob capture research links %3`, the newest-first/exact-count behavior, `%1`
equivalence, `%0` literal behavior, and the numeric-header escape via `--clip=<digits>`.

Replace the old documentation that describes `%20` as an accepted header quirk, since it
now intentionally means twenty history entries. Document `--no-clip` as the way to keep
a genuine terminal `%N` literal. Add `BOB_CLIPBOARD_HISTORY_CMD` to help/README
environment guidance with its appended count argument, JSON-array framing, ordering, and
failure behavior; describe the read-only Clipy fallback and the lack of an automatic
non-macOS history provider. Keep public options alphabetical and preserve all existing
CLI-rule requirements.

Document the history JSON shape and clarify that classification limits (such as the
attachment cap) apply per history entry, while every requested entry must succeed for
the capture to commit.

## Verification

Add focused unit and integration coverage before running the full repository gate.

- Parser tests: `%1`, larger counts, leading zeroes, `%0`, overflow, positive numeric
  reassignment, nonnumeric headers, mid-body/literal invalid tokens, duplicate clip
  markers, `--no-clip`, forced `--clip`/numeric header, and every
  schedule/route/bullet/Pomodoro terminal ordering already supported.
- History-source tests: current-value merge with up-to-date and lagging histories,
  preservation of later duplicates, newest-first truncation, exact-count failure,
  malformed/failed command output, JSON strings containing newlines, normalization
  errors, and actionable no-provider diagnostics. Exercise the Clipy adapter with
  temporary SQLite fixtures covering ordering, string and file URL assets, insufficient
  rows, unsupported binary-only entries, and schema mismatch; tests must not require a
  running Clipy installation or touch a real user database.
- Aggregate planner tests: mixed inline/lines/attachment/snippet entries, exact
  flattened headerless Markdown, retained entry boundaries, same-run attachment basename
  collisions (identical and different content), same-second snippet collisions, unique
  saves, dry-run, partial-save cleanup, and failure in a later entry leaving no files or
  note edits.
- CLI integration tests: `%1` through `BOB_CLIPBOARD_CMD`; `%3` through a JSON history
  fixture under plain task, routed bullet, and Pomodoro placement; exact note ordering;
  absence of all headers/containers; human output; aggregate JSON and unchanged
  single-clip JSON; insufficient history and malformed provider output leaving the vault
  untouched; and updated help text/option ordering.
- Regression coverage: explicit `%header` and `--clip=header`, bare `%`/`--clip`,
  attachments, snippets, scheduling, route placement, Pomodoro atomicity, non-clipboard
  JSON, and subsequent insertion after the full child block remain unchanged.
- Run `just all` and require formatting, Clippy, unit tests, and CLI integration tests
  to pass. Also run `git diff --check` and review the final diff for unintended changes
  outside capture code, dependency metadata, README/help text, and tests.

## Acceptance criteria and boundaries

- `%N` captures exactly `N` usable values with the live clipboard first and older
  history following newest-first; `%1` is indistinguishable from `%` except for the
  typed marker.
- Every history entry uses the approved headerless rendering and its existing
  classifier; the parent capture and note placement are unchanged.
- Same-run file-name collisions cannot overwrite, alias, or silently drop any history
  entry, and all failure paths preserve the existing no-partial-vault-update guarantee.
- Existing single-clipboard behavior and JSON remain compatible, while history output
  exposes both flattened written lines and per-entry structure.
- Clipy access is read-only and schema-checked, the command override is deterministic
  and portable, and unsupported history environments fail with setup guidance rather
  than falling back to repeated reads of the same live value.
- Raw-image extraction from history, Windows support, a generic clipboard-manager plugin
  system, configuration of history ordering, entry headers/index labels, and migration
  of existing notes are out of scope.
